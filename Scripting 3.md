# Phase 9, Lesson 3: Go for DevOps

---

## WHY GO FOR DEVOPS — NOT OPINION, FACTS

```
╔══════════════════════════════════════════════════════════════════╗
║  The entire Kubernetes ecosystem is written in Go.              ║
║  kubectl, kubelet, kube-proxy, etcd, containerd, Prometheus,    ║
║  Terraform, Docker, Istio, ArgoCD, Helm, Vault, Consul,        ║
║  CoreDNS, Linkerd2, CRI-O, Buildah, Trivy, Falco...            ║
║                                                                  ║
║  If you can't write Go, you can't extend, debug, or contribute  ║
║  to ANY of these tools. You're a consumer, not an engineer.     ║
╚══════════════════════════════════════════════════════════════════╝
```

**Go vs Python for DevOps — Decision Framework:**

| Dimension | Go | Python |
|-----------|-----|--------|
| Distribution | Single binary. No runtime. Copy and run. | Needs Python installed + venv + pip |
| Startup time | ~5ms | ~200-500ms (import overhead) |
| Concurrency | Native goroutines (millions cheap) | GIL limits threads; asyncio is bolt-on |
| Type safety | Compile-time. Bug caught before deploy. | Runtime. Bug caught in production. |
| K8s ecosystem | Native. client-go, controller-runtime, kubebuilder | kubernetes python client (thin, incomplete) |
| Cross-compile | `GOOS=linux GOARCH=arm64 go build` | Not applicable |
| Memory | Predictable, low footprint | Higher, GC pauses |
| Learning curve | Steeper initially | Lower |
| Prototyping speed | Slower | Faster |
| When to use | CLI tools, K8s controllers, long-running services, performance-critical | API glue, scripting, data processing, prototyping |

**NovaMart Rule:** CLI tools and K8s integration = Go. Automation scripts and API glue = Python. Entrypoints and glue = Bash.

---

## 1. GO FUNDAMENTALS — WHAT DEVOPS ENGINEERS MUST KNOW

I'm not teaching you the entire Go language. I'm teaching you the **parts that matter for DevOps tooling**, with production patterns.

```go
// ═══════════════════════════════════════════════════════════
// BASIC TYPES AND PATTERNS YOU'LL USE DAILY
// ═══════════════════════════════════════════════════════════

package main

import (
    "fmt"
    "strings"
    "time"
)

// ── Structs (your primary data structure) ─────────────────
type Instance struct {
    ID         string            `json:"id" yaml:"id"`
    Type       string            `json:"type" yaml:"type"`
    State      string            `json:"state" yaml:"state"`
    LaunchTime time.Time         `json:"launch_time" yaml:"launchTime"`
    Tags       map[string]string `json:"tags" yaml:"tags"`
    AZ         string            `json:"az" yaml:"az"`
}

// Methods on structs
func (i *Instance) Name() string {
    if name, ok := i.Tags["Name"]; ok {
        return name
    }
    return i.ID
}

func (i *Instance) Age() time.Duration {
    return time.Since(i.LaunchTime)
}

func (i *Instance) AgeDays() int {
    return int(i.Age().Hours() / 24)
}

// ── Interfaces (THE Go power tool) ───────────────────────
// Interfaces are implicit — no "implements" keyword.
// Any type with the right methods satisfies the interface.
// This is how you make testable, mockable code.

type CloudProvider interface {
    ListInstances(region string) ([]Instance, error)
    StopInstance(id string) error
    TerminateInstance(id string) error
}

type NotificationSender interface {
    Send(channel, message string) error
}

// Now you can test with mock implementations:
type MockCloud struct {
    Instances []Instance
    StopErr   error
}

func (m *MockCloud) ListInstances(region string) ([]Instance, error) {
    return m.Instances, nil
}

func (m *MockCloud) StopInstance(id string) error {
    return m.StopErr
}

func (m *MockCloud) TerminateInstance(id string) error {
    return nil
}

// ── Slices and Maps ───────────────────────────────────────
func filterRunning(instances []Instance) []Instance {
    // Pre-allocate with estimated capacity (avoid repeated grow)
    result := make([]Instance, 0, len(instances))
    for _, inst := range instances {
        if inst.State == "running" {
            result = append(result, inst)
        }
    }
    return result
}

func groupByAZ(instances []Instance) map[string][]Instance {
    groups := make(map[string][]Instance)
    for _, inst := range instances {
        groups[inst.AZ] = append(groups[inst.AZ], inst)
    }
    return groups
}

// ── String operations ─────────────────────────────────────
func normalizeTag(key string) string {
    return strings.ToLower(strings.TrimSpace(key))
}

func buildARN(partition, service, region, account, resource string) string {
    // strings.Builder is efficient for multiple concatenations
    var b strings.Builder
    b.WriteString("arn:")
    b.WriteString(partition)
    b.WriteString(":")
    b.WriteString(service)
    b.WriteString(":")
    b.WriteString(region)
    b.WriteString(":")
    b.WriteString(account)
    b.WriteString(":")
    b.WriteString(resource)
    return b.String()
    // For simple cases, fmt.Sprintf is fine:
    // return fmt.Sprintf("arn:%s:%s:%s:%s:%s", partition, service, region, account, resource)
}

// ── Pointers — when and why ───────────────────────────────
// Use pointers to:
// 1. Modify the original (method receivers on structs you want to mutate)
// 2. Avoid copying large structs
// 3. Represent "optional" (nil pointer = not set)
// Don't use pointers for: small structs, read-only access, primitives

func updateState(inst *Instance, newState string) {
    inst.State = newState // Modifies the original
}

// Optional fields pattern (common in K8s API objects)
type DeployConfig struct {
    Replicas    int32  
    MaxSurge    *int32  // nil = use default
    MaxUnavail  *int32  // nil = use default
}

// Helper to create pointer to literal
func int32Ptr(i int32) *int32 { return &i }
func stringPtr(s string) *string { return &s }
func boolPtr(b bool) *bool { return &b }
```

---

## 2. PROJECT STRUCTURE

```
novactl/
├── cmd/                          # Entrypoints
│   └── novactl/
│       └── main.go               # Minimal — just calls root command
├── internal/                     # Private packages (can't be imported)
│   ├── aws/
│   │   ├── ec2.go
│   │   ├── ec2_test.go
│   │   ├── rds.go
│   │   └── session.go
│   ├── k8s/
│   │   ├── client.go
│   │   ├── pods.go
│   │   ├── deployments.go
│   │   └── events.go
│   ├── config/
│   │   ├── config.go             # Viper-based config
│   │   └── config_test.go
│   ├── output/
│   │   ├── table.go              # Table output (tablewriter)
│   │   ├── json.go               # JSON output
│   │   └── format.go             # Format router
│   └── version/
│       └── version.go            # Build-time version injection
├── pkg/                          # Public packages (importable by others)
│   ├── health/
│   │   ├── checker.go
│   │   └── checker_test.go
│   └── retry/
│       ├── retry.go
│       └── retry_test.go
├── api/                          # API definitions (webhook payloads, CRD types)
│   └── v1/
│       └── types.go
├── deploy/                       # K8s manifests for the tool itself
│   ├── deployment.yaml
│   └── rbac.yaml
├── scripts/                      # Build/release scripts
│   └── build.sh
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
├── .goreleaser.yml
└── README.md
```

```go
// cmd/novactl/main.go — MINIMAL entrypoint
package main

import (
    "fmt"
    "os"

    "github.com/novamart/novactl/internal/cmd"
)

func main() {
    if err := cmd.Execute(); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

```makefile
# Makefile — standard targets every Go project needs
BINARY    := novactl
VERSION   := $(shell git describe --tags --always --dirty)
COMMIT    := $(shell git rev-parse --short HEAD)
DATE      := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)
LDFLAGS   := -s -w \
    -X github.com/novamart/novactl/internal/version.Version=$(VERSION) \
    -X github.com/novamart/novactl/internal/version.Commit=$(COMMIT) \
    -X github.com/novamart/novactl/internal/version.Date=$(DATE)

.PHONY: build test lint clean docker release

build:
	CGO_ENABLED=0 go build -ldflags "$(LDFLAGS)" -o bin/$(BINARY) ./cmd/novactl/

test:
	go test -race -coverprofile=coverage.out ./...
	go tool cover -func=coverage.out

lint:
	golangci-lint run ./...

fmt:
	gofumpt -l -w .

vet:
	go vet ./...

# Cross-compile for all platforms
build-all:
	GOOS=linux GOARCH=amd64 go build -ldflags "$(LDFLAGS)" -o bin/$(BINARY)-linux-amd64 ./cmd/novactl/
	GOOS=linux GOARCH=arm64 go build -ldflags "$(LDFLAGS)" -o bin/$(BINARY)-linux-arm64 ./cmd/novactl/
	GOOS=darwin GOARCH=amd64 go build -ldflags "$(LDFLAGS)" -o bin/$(BINARY)-darwin-amd64 ./cmd/novactl/
	GOOS=darwin GOARCH=arm64 go build -ldflags "$(LDFLAGS)" -o bin/$(BINARY)-darwin-arm64 ./cmd/novactl/

docker:
	docker build -t $(BINARY):$(VERSION) .

clean:
	rm -rf bin/ coverage.out

# Install locally
install: build
	cp bin/$(BINARY) $(GOPATH)/bin/
```

```go
// internal/version/version.go
package version

// Set at build time via ldflags
var (
    Version = "dev"
    Commit  = "unknown"
    Date    = "unknown"
)

func String() string {
    return Version + " (commit: " + Commit + ", built: " + Date + ")"
}
```

```dockerfile
# Dockerfile — multi-stage, distroless
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w" \
    -o /novactl ./cmd/novactl/

# Distroless — no shell, no package manager, minimal attack surface
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /novactl /novactl
USER nonroot:nonroot

ENTRYPOINT ["/novactl"]
```

---

## 3. CLI WITH COBRA + VIPER

```go
// internal/cmd/root.go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
    "github.com/spf13/viper"
    "go.uber.org/zap"

    "github.com/novamart/novactl/internal/config"
    "github.com/novamart/novactl/internal/version"
)

var (
    cfgFile string
    logger  *zap.SugaredLogger
)

// rootCmd represents the base command
var rootCmd = &cobra.Command{
    Use:   "novactl",
    Short: "NovaMart Platform CLI",
    Long: `novactl is the unified CLI for NovaMart platform operations.

It provides commands for cluster management, deployments, secrets,
compliance, cost analysis, and incident response.`,
    Version: version.String(),
    // PersistentPreRun runs before ANY subcommand
    PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
        return initConfig()
    },
    // SilenceUsage prevents printing usage on errors (noisy)
    SilenceUsage: true,
}

func Execute() error {
    return rootCmd.Execute()
}

func init() {
    // Persistent flags — available to ALL subcommands
    rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "",
        "config file (default: $HOME/.novactl.yaml)")
    rootCmd.PersistentFlags().String("output", "table",
        "output format: table, json, yaml, csv")
    rootCmd.PersistentFlags().Bool("verbose", false, "verbose output")
    rootCmd.PersistentFlags().Bool("dry-run", false, "preview changes without applying")
    rootCmd.PersistentFlags().String("context", "", "kubernetes context override")
    rootCmd.PersistentFlags().String("region", "us-east-1", "AWS region")

    // Bind flags to Viper (env vars + config file + flags unified)
    viper.BindPFlag("output", rootCmd.PersistentFlags().Lookup("output"))
    viper.BindPFlag("verbose", rootCmd.PersistentFlags().Lookup("verbose"))
    viper.BindPFlag("dry-run", rootCmd.PersistentFlags().Lookup("dry-run"))
    viper.BindPFlag("context", rootCmd.PersistentFlags().Lookup("context"))
    viper.BindPFlag("region", rootCmd.PersistentFlags().Lookup("region"))

    // Env var binding: NOVACTL_REGION → region
    viper.SetEnvPrefix("NOVACTL")
    viper.AutomaticEnv()

    // Register subcommand groups
    rootCmd.AddCommand(clusterCmd)
    rootCmd.AddCommand(deployCmd)
    rootCmd.AddCommand(secretsCmd)
    rootCmd.AddCommand(complianceCmd)
    rootCmd.AddCommand(costCmd)
    rootCmd.AddCommand(incidentCmd)
    rootCmd.AddCommand(completionCmd)
}

func initConfig() error {
    if cfgFile != "" {
        viper.SetConfigFile(cfgFile)
    } else {
        home, err := os.UserHomeDir()
        if err != nil {
            return fmt.Errorf("finding home directory: %w", err)
        }
        viper.AddConfigPath(home)
        viper.AddConfigPath(".")
        viper.SetConfigName(".novactl")
        viper.SetConfigType("yaml")
    }

    // Read config (not an error if missing)
    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return fmt.Errorf("reading config file: %w", err)
        }
    }

    // Initialize logger
    var err error
    if viper.GetBool("verbose") {
        cfg := zap.NewDevelopmentConfig()
        l, err := cfg.Build()
        if err != nil {
            return err
        }
        logger = l.Sugar()
    } else {
        cfg := zap.NewProductionConfig()
        cfg.OutputPaths = []string{"stderr"} // Never pollute stdout
        l, err := cfg.Build()
        if err != nil {
            return err
        }
        logger = l.Sugar()
    }
    _ = err

    return nil
}


// ═══════════════════════════════════════════════════════════
// SUBCOMMAND: cluster
// ═══════════════════════════════════════════════════════════

// internal/cmd/cluster.go
var clusterCmd = &cobra.Command{
    Use:   "cluster",
    Short: "Cluster operations",
    Long:  "Inspect, audit, and manage EKS clusters.",
}

var clusterInfoCmd = &cobra.Command{
    Use:   "info",
    Short: "Show cluster information",
    RunE: func(cmd *cobra.Command, args []string) error {
        ctx := cmd.Context()
        region := viper.GetString("region")
        format := viper.GetString("output")

        // Build dependencies
        k8sClient, err := k8s.NewClient(viper.GetString("context"))
        if err != nil {
            return fmt.Errorf("creating k8s client: %w", err)
        }

        // Execute
        info, err := k8sClient.ClusterInfo(ctx)
        if err != nil {
            return fmt.Errorf("getting cluster info: %w", err)
        }

        // Output
        return output.Render(format, info)
    },
}

var clusterAuditCmd = &cobra.Command{
    Use:   "audit",
    Short: "Audit cluster for issues",
    Long: `Checks for:
  - Pods without resource limits
  - Deployments with single replica
  - Orphaned ConfigMaps/Secrets
  - PVCs not bound
  - Nodes with high utilization
  - Images using :latest tag
  - Missing PodDisruptionBudgets`,
    RunE: func(cmd *cobra.Command, args []string) error {
        ctx := cmd.Context()
        namespace, _ := cmd.Flags().GetString("namespace")
        format := viper.GetString("output")

        k8sClient, err := k8s.NewClient(viper.GetString("context"))
        if err != nil {
            return fmt.Errorf("creating k8s client: %w", err)
        }

        findings, err := k8sClient.Audit(ctx, namespace)
        if err != nil {
            return fmt.Errorf("audit failed: %w", err)
        }

        return output.Render(format, findings)
    },
}

func init() {
    clusterCmd.AddCommand(clusterInfoCmd)
    clusterCmd.AddCommand(clusterAuditCmd)

    clusterAuditCmd.Flags().StringP("namespace", "n", "",
        "namespace to audit (default: all)")
}


// ═══════════════════════════════════════════════════════════
// SUBCOMMAND: deploy
// ═══════════════════════════════════════════════════════════

// internal/cmd/deploy.go
var deployCmd = &cobra.Command{
    Use:   "deploy",
    Short: "Deployment operations",
}

var deployStatusCmd = &cobra.Command{
    Use:   "status [service]",
    Short: "Check deployment status",
    Args:  cobra.ExactArgs(1), // Require exactly 1 argument
    RunE: func(cmd *cobra.Command, args []string) error {
        service := args[0]
        namespace, _ := cmd.Flags().GetString("namespace")
        format := viper.GetString("output")

        k8sClient, err := k8s.NewClient(viper.GetString("context"))
        if err != nil {
            return fmt.Errorf("creating k8s client: %w", err)
        }

        status, err := k8sClient.DeploymentStatus(cmd.Context(), namespace, service)
        if err != nil {
            return fmt.Errorf("getting deploy status: %w", err)
        }

        return output.Render(format, status)
    },
}

var deployRollbackCmd = &cobra.Command{
    Use:   "rollback [service]",
    Short: "Rollback a deployment",
    Args:  cobra.ExactArgs(1),
    RunE: func(cmd *cobra.Command, args []string) error {
        service := args[0]
        namespace, _ := cmd.Flags().GetString("namespace")
        dryRun := viper.GetBool("dry-run")
        revision, _ := cmd.Flags().GetInt64("revision")

        if !dryRun {
            // Confirmation for non-dry-run
            fmt.Fprintf(os.Stderr,
                "⚠️  About to rollback %s in namespace %s\n", service, namespace)
            fmt.Fprint(os.Stderr, "Continue? [y/N]: ")
            var response string
            fmt.Scanln(&response)
            if response != "y" && response != "Y" {
                fmt.Fprintln(os.Stderr, "Aborted.")
                return nil
            }
        }

        k8sClient, err := k8s.NewClient(viper.GetString("context"))
        if err != nil {
            return fmt.Errorf("creating k8s client: %w", err)
        }

        result, err := k8sClient.Rollback(cmd.Context(), namespace, service, revision, dryRun)
        if err != nil {
            return fmt.Errorf("rollback failed: %w", err)
        }

        fmt.Fprintf(os.Stderr, "✅ Rollback %s: %s\n",
            map[bool]string{true: "simulated", false: "completed"}[dryRun],
            result)
        return nil
    },
}

func init() {
    deployCmd.AddCommand(deployStatusCmd)
    deployCmd.AddCommand(deployRollbackCmd)

    deployStatusCmd.Flags().StringP("namespace", "n", "default", "namespace")
    deployRollbackCmd.Flags().StringP("namespace", "n", "default", "namespace")
    deployRollbackCmd.Flags().Int64("revision", 0, "revision to rollback to (0 = previous)")
}


// ═══════════════════════════════════════════════════════════
// SHELL COMPLETIONS (professional touch)
// ═══════════════════════════════════════════════════════════

// internal/cmd/completion.go
var completionCmd = &cobra.Command{
    Use:   "completion [bash|zsh|fish|powershell]",
    Short: "Generate shell completion scripts",
    Long: `To load completions:

Bash:
  $ source <(novactl completion bash)
  # To load completions for each session, add to ~/.bashrc:
  $ echo 'source <(novactl completion bash)' >> ~/.bashrc

Zsh:
  $ novactl completion zsh > "${fpath[1]}/_novactl"

Fish:
  $ novactl completion fish | source`,
    ValidArgs:            []string{"bash", "zsh", "fish", "powershell"},
    Args:                 cobra.ExactValidArgs(1),
    DisableFlagsInUseLine: true,
    RunE: func(cmd *cobra.Command, args []string) error {
        switch args[0] {
        case "bash":
            return rootCmd.GenBashCompletion(os.Stdout)
        case "zsh":
            return rootCmd.GenZshCompletion(os.Stdout)
        case "fish":
            return rootCmd.GenFishCompletion(os.Stdout, true)
        case "powershell":
            return rootCmd.GenPowerShellCompletionWithDesc(os.Stdout)
        default:
            return fmt.Errorf("unsupported shell: %s", args[0])
        }
    },
}
```

**Viper config file example (`~/.novactl.yaml`):**
```yaml
# Default values — overridden by flags and env vars
region: us-east-1
output: table
verbose: false

# Context aliases
contexts:
  prod: arn:aws:eks:us-east-1:123456789012:cluster/novamart-production
  staging: arn:aws:eks:us-east-1:123456789012:cluster/novamart-staging

# Notification defaults
slack:
  channel: "#platform-alerts"
  webhook_url: ""  # Set via NOVACTL_SLACK_WEBHOOK_URL env var

# Cost thresholds
cost:
  daily_alert_threshold: 15000
  anomaly_deviation_pct: 20
```

---

## 4. ERROR HANDLING — THE GO WAY

```go
// ═══════════════════════════════════════════════════════════
// Go error handling is EXPLICIT. No exceptions. No try/catch.
// Every function that can fail returns an error.
// This is a feature, not a burden.
// ═══════════════════════════════════════════════════════════

package errors

import (
    "errors"
    "fmt"
)

// ── Custom error types ────────────────────────────────────

// Sentinel errors — compare with errors.Is()
var (
    ErrNotFound       = errors.New("resource not found")
    ErrAlreadyExists  = errors.New("resource already exists")
    ErrUnauthorized   = errors.New("unauthorized")
    ErrTimeout        = errors.New("operation timed out")
    ErrDryRun         = errors.New("dry run — no changes made")
)

// Structured errors — unwrap with errors.As()
type AWSError struct {
    Service   string
    Operation string
    Code      string
    Message   string
    Err       error
}

func (e *AWSError) Error() string {
    return fmt.Sprintf("aws %s.%s: %s — %s", e.Service, e.Operation, e.Code, e.Message)
}

func (e *AWSError) Unwrap() error {
    return e.Err
}

type K8sError struct {
    Resource  string
    Namespace string
    Name      string
    Operation string
    Err       error
}

func (e *K8sError) Error() string {
    return fmt.Sprintf("k8s %s %s/%s: %s failed: %v",
        e.Resource, e.Namespace, e.Name, e.Operation, e.Err)
}

func (e *K8sError) Unwrap() error {
    return e.Err
}

type ValidationError struct {
    Field   string
    Value   interface{}
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s=%v — %s", e.Field, e.Value, e.Message)
}


// ── Error wrapping — THE critical pattern ─────────────────

func DeployService(ctx context.Context, name, namespace, image string) error {
    // GOOD: wrap errors with context at every level
    dep, err := getDeployment(ctx, namespace, name)
    if err != nil {
        return fmt.Errorf("getting deployment %s/%s: %w", namespace, name, err)
        // %w wraps the error — callers can use errors.Is() and errors.As()
    }

    if err := updateImage(dep, image); err != nil {
        return fmt.Errorf("updating image for %s: %w", name, err)
    }

    if err := applyDeployment(ctx, dep); err != nil {
        return fmt.Errorf("applying deployment %s: %w", name, err)
    }

    return nil  // nil = success
}


// ── Error checking patterns ───────────────────────────────

func HandleError(err error) {
    // Check for sentinel error
    if errors.Is(err, ErrNotFound) {
        // Resource doesn't exist — maybe create it
        fmt.Println("Resource not found, creating...")
        return
    }

    // Check for typed error
    var awsErr *AWSError
    if errors.As(err, &awsErr) {
        // We know it's an AWS error — handle by code
        switch awsErr.Code {
        case "Throttling":
            fmt.Println("AWS rate limit hit, backing off...")
        case "AccessDenied":
            fmt.Println("Permission denied — check IAM role")
        default:
            fmt.Printf("AWS error: %s\n", awsErr.Message)
        }
        return
    }

    var k8sErr *K8sError
    if errors.As(err, &k8sErr) {
        fmt.Printf("K8s error on %s/%s: %v\n",
            k8sErr.Namespace, k8sErr.Name, k8sErr.Err)
        return
    }

    // Unknown error
    fmt.Printf("Unexpected error: %v\n", err)
}


// ═══════════════════════════════════════════════════════════
// ANTI-PATTERNS
// ═══════════════════════════════════════════════════════════

// BAD: Ignoring errors
// result, _ := riskyOperation()  // NEVER in production code

// BAD: panic in library code
// panic("something went wrong")  // Only acceptable in main() or truly unrecoverable

// BAD: Error string comparison
// if err.Error() == "not found" {...}  // Fragile. Use errors.Is() or errors.As()

// BAD: Wrapping without context
// return fmt.Errorf("failed: %w", err)  // "failed" adds zero value

// GOOD: Wrap with what YOU were doing when it failed
// return fmt.Errorf("deploying %s to %s: %w", service, namespace, err)
```

---

## 5. LOGGING — STRUCTURED (zerolog / zap)

```go
// ═══════════════════════════════════════════════════════════
// zerolog — fastest structured logger for Go
// zap is also excellent (Uber). Both produce JSON logs.
// Pick one. NovaMart uses zerolog.
// ═══════════════════════════════════════════════════════════

package logging

import (
    "os"
    "time"

    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func Setup(verbose bool, jsonOutput bool) {
    zerolog.TimeFieldFormat = time.RFC3339

    if jsonOutput {
        // JSON for production (parsed by Loki/Fluentbit)
        log.Logger = zerolog.New(os.Stderr).
            With().
            Timestamp().
            Caller().        // Adds file:line
            Str("app", "novactl").
            Logger()
    } else {
        // Human-readable for development
        log.Logger = zerolog.New(zerolog.ConsoleWriter{
            Out:        os.Stderr,
            TimeFormat: "15:04:05",
        }).
            With().
            Timestamp().
            Logger()
    }

    if verbose {
        zerolog.SetGlobalLevel(zerolog.DebugLevel)
    } else {
        zerolog.SetGlobalLevel(zerolog.InfoLevel)
    }
}

// ── Usage patterns ────────────────────────────────────────

func ExampleLogging() {
    // Simple
    log.Info().Msg("starting cluster audit")

    // With fields (structured data)
    log.Info().
        Str("namespace", "payments").
        Str("deployment", "payment-service").
        Int("replicas", 3).
        Msg("deployment_status_checked")

    // With error
    err := fmt.Errorf("connection refused")
    log.Error().
        Err(err).
        Str("host", "rds-payments.internal").
        Int("port", 5432).
        Msg("database_connection_failed")

    // Sub-logger with context (pass to functions)
    logger := log.With().
        Str("operation", "deploy").
        Str("service", "payment-service").
        Str("environment", "production").
        Logger()

    logger.Info().Msg("starting_deployment")
    logger.Info().Str("image", "payment:v2.3.1").Msg("image_updated")
    logger.Info().Dur("duration", 45*time.Second).Msg("deployment_complete")

    // Timing
    start := time.Now()
    // ... operation ...
    log.Info().
        Dur("duration", time.Since(start)).
        Msg("operation_complete")

    // NEVER log secrets
    // log.Info().Str("password", secret).Msg("connecting")  // ← NEVER
    log.Info().
        Str("host", "rds-payments").
        Str("user", "app_user").
        Msg("connecting_to_database")  // No password
}
```

---

## 6. CONCURRENCY — GO'S SUPERPOWER

```go
// ═══════════════════════════════════════════════════════════
// Goroutines are NOT threads. They're lightweight (2KB stack).
// You can run millions. They're multiplexed onto OS threads.
// Channels are how goroutines communicate.
// context.Context is how you cancel them.
// ═══════════════════════════════════════════════════════════

package concurrency

import (
    "context"
    "fmt"
    "sync"
    "time"

    "golang.org/x/sync/errgroup"
    "github.com/rs/zerolog/log"
)

// ═══════════════════════════════════════════════════════════
// PATTERN 1: errgroup — bounded concurrency with error handling
// THIS IS YOUR GO-TO PATTERN. Use it 90% of the time.
// ═══════════════════════════════════════════════════════════

func CheckEndpointsParallel(
    ctx context.Context,
    endpoints []string,
    maxConcurrency int,
    timeout time.Duration,
) ([]HealthResult, error) {
    results := make([]HealthResult, len(endpoints))
    
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(maxConcurrency) // Bounded concurrency

    for i, endpoint := range endpoints {
        i, endpoint := i, endpoint // Capture loop variables (Go < 1.22)
        // Go 1.22+ fixes this, but be safe for backward compat
        
        g.Go(func() error {
            result, err := checkHealth(ctx, endpoint, timeout)
            if err != nil {
                // Don't return error — we want partial results
                results[i] = HealthResult{
                    Endpoint: endpoint,
                    Status:   "error",
                    Error:    err.Error(),
                }
                return nil // nil = don't cancel other goroutines
            }
            results[i] = result
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return results, fmt.Errorf("health check failed: %w", err)
    }

    return results, nil
}

type HealthResult struct {
    Endpoint   string        `json:"endpoint"`
    Status     string        `json:"status"`
    Latency    time.Duration `json:"latency"`
    StatusCode int           `json:"status_code"`
    Error      string        `json:"error,omitempty"`
}


// ═══════════════════════════════════════════════════════════
// PATTERN 2: Fan-out / Fan-in with channels
// ═══════════════════════════════════════════════════════════

func ScanRegions(
    ctx context.Context,
    regions []string,
    scanFn func(ctx context.Context, region string) ([]Finding, error),
) ([]Finding, error) {
    type regionResult struct {
        region   string
        findings []Finding
        err      error
    }

    resultCh := make(chan regionResult, len(regions))

    // Fan-out: one goroutine per region
    var wg sync.WaitGroup
    for _, region := range regions {
        wg.Add(1)
        go func(r string) {
            defer wg.Done()
            findings, err := scanFn(ctx, r)
            resultCh <- regionResult{region: r, findings: findings, err: err}
        }(region)
    }

    // Close channel when all goroutines complete
    go func() {
        wg.Wait()
        close(resultCh)
    }()

    // Fan-in: collect all results
    var allFindings []Finding
    var errs []error

    for result := range resultCh {
        if result.err != nil {
            log.Error().Err(result.err).Str("region", result.region).Msg("scan_failed")
            errs = append(errs, fmt.Errorf("region %s: %w", result.region, result.err))
            continue
        }
        allFindings = append(allFindings, result.findings...)
        log.Info().
            Str("region", result.region).
            Int("findings", len(result.findings)).
            Msg("region_scan_complete")
    }

    if len(errs) > 0 {
        return allFindings, fmt.Errorf("%d region(s) failed: %v", len(errs), errs[0])
    }
    return allFindings, nil
}


// ═══════════════════════════════════════════════════════════
// PATTERN 3: Worker pool
// ═══════════════════════════════════════════════════════════

func WorkerPool[T any, R any](
    ctx context.Context,
    items []T,
    workers int,
    processFn func(ctx context.Context, item T) (R, error),
) ([]R, []error) {
    jobs := make(chan T, len(items))
    type result struct {
        value R
        err   error
        index int
    }
    resultsCh := make(chan result, len(items))

    // Start workers
    var wg sync.WaitGroup
    for w := 0; w < workers; w++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range jobs {
                select {
                case <-ctx.Done():
                    return
                default:
                    val, err := processFn(ctx, item)
                    resultsCh <- result{value: val, err: err}
                }
            }
        }()
    }

    // Send jobs
    for _, item := range items {
        jobs <- item
    }
    close(jobs)

    // Wait for workers then close results
    go func() {
        wg.Wait()
        close(resultsCh)
    }()

    // Collect results
    var values []R
    var errs []error
    for r := range resultsCh {
        if r.err != nil {
            errs = append(errs, r.err)
        } else {
            values = append(values, r.value)
        }
    }

    return values, errs
}


// ═══════════════════════════════════════════════════════════
// PATTERN 4: context.Context — timeout, cancellation, values
// ═══════════════════════════════════════════════════════════

func ContextPatterns() {
    // Timeout — automatically cancels after duration
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel() // ALWAYS defer cancel to release resources

    // Check if cancelled
    select {
    case <-ctx.Done():
        fmt.Println("Cancelled:", ctx.Err())
        // ctx.Err() is context.Canceled or context.DeadlineExceeded
    default:
        fmt.Println("Still running")
    }

    // Pass to HTTP requests, K8s API calls, DB queries — everything
    // They all respect context cancellation
}

func LongOperation(ctx context.Context) error {
    for i := 0; i < 100; i++ {
        // Check context before expensive work
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }

        // Do work...
        time.Sleep(100 * time.Millisecond)
    }
    return nil
}


// ═══════════════════════════════════════════════════════════
// PATTERN 5: sync.Once — safe lazy initialization
// ═══════════════════════════════════════════════════════════

var (
    k8sClient     *kubernetes.Clientset
    k8sClientOnce sync.Once
    k8sClientErr  error
)

func GetK8sClient() (*kubernetes.Clientset, error) {
    k8sClientOnce.Do(func() {
        config, err := rest.InClusterConfig()
        if err != nil {
            k8sClientErr = err
            return
        }
        k8sClient, k8sClientErr = kubernetes.NewForConfig(config)
    })
    return k8sClient, k8sClientErr
}
```

**Concurrency failure modes:**
```go
// FAILURE 1: Goroutine leak — the #1 Go bug in production
// BAD:
func leaky() {
    ch := make(chan int)
    go func() {
        val := <-ch  // Blocks forever — nobody sends to ch
        _ = val
    }()
    // Function returns, goroutine stays alive forever
    // Do this in a loop = memory leak, goroutine count grows until OOM
}
// FIX: ALWAYS use context for cancellation
func notLeaky(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            _ = val
        case <-ctx.Done():
            return // Goroutine exits cleanly
        }
    }()
}

// FAILURE 2: Race condition
// go test -race ./...  ← RUN THIS IN CI. ALWAYS.
// BAD:
var counter int
func increment() { counter++ }  // Multiple goroutines = data race
// FIX:
var mu sync.Mutex
func safeIncrement() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
// Or use atomic: atomic.AddInt64(&counter, 1)

// FAILURE 3: Channel deadlock
// BAD:
ch := make(chan int) // Unbuffered
ch <- 1              // Blocks forever — no reader yet
val := <-ch          // Never reached
// FIX: Use buffered channel or goroutine
ch := make(chan int, 1) // Buffered
ch <- 1                 // Doesn't block
val := <-ch

// FAILURE 4: nil channel blocks forever
// var ch chan int  // nil channel
// ch <- 1         // Blocks forever
// <-ch            // Blocks forever
// FIX: Always initialize: ch := make(chan int)

// FAILURE 5: Sending to closed channel panics
// close(ch)
// ch <- 1  // PANIC: send on closed channel
// FIX: Only the sender closes. Use sync.Once for close if multiple senders.
```

---

## 7. HTTP SERVER AND CLIENT

```go
// ═══════════════════════════════════════════════════════════
// HTTP CLIENT — for API calls (Slack, PagerDuty, Prometheus)
// ═══════════════════════════════════════════════════════════

package httpclient

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net"
    "net/http"
    "time"
)

// NewHTTPClient creates a production-ready HTTP client.
// DEFAULT http.Client has NO timeout — blocks forever.
func NewHTTPClient() *http.Client {
    return &http.Client{
        Timeout: 30 * time.Second, // Total request timeout
        Transport: &http.Transport{
            DialContext: (&net.Dialer{
                Timeout:   5 * time.Second,  // Connection timeout
                KeepAlive: 30 * time.Second, // TCP keepalive
            }).DialContext,
            MaxIdleConns:          100,
            MaxIdleConnsPerHost:   10,
            IdleConnTimeout:       90 * time.Second,
            TLSHandshakeTimeout:   5 * time.Second,
            ResponseHeaderTimeout: 10 * time.Second,
            // IMPORTANT: Reuse this client. Don't create per-request.
        },
    }
}

// Generic API caller with retry
func DoWithRetry(
    ctx context.Context,
    client *http.Client,
    req *http.Request,
    maxRetries int,
) (*http.Response, error) {
    var lastErr error

    for attempt := 0; attempt <= maxRetries; attempt++ {
        if attempt > 0 {
            backoff := time.Duration(attempt*attempt) * 500 * time.Millisecond
            select {
            case <-ctx.Done():
                return nil, ctx.Err()
            case <-time.After(backoff):
            }
        }

        resp, err := client.Do(req.WithContext(ctx))
        if err != nil {
            lastErr = err
            continue
        }

        // Retry on 429 and 5xx
        if resp.StatusCode == 429 || resp.StatusCode >= 500 {
            io.Copy(io.Discard, resp.Body) // Drain body to reuse connection
            resp.Body.Close()
            lastErr = fmt.Errorf("HTTP %d", resp.StatusCode)
            continue
        }

        return resp, nil
    }

    return nil, fmt.Errorf("request failed after %d retries: %w", maxRetries, lastErr)
}


// ═══════════════════════════════════════════════════════════
// PROMETHEUS QUERY CLIENT
// ═══════════════════════════════════════════════════════════

type PrometheusClient struct {
    baseURL string
    client  *http.Client
}

func NewPrometheusClient(baseURL string) *PrometheusClient {
    return &PrometheusClient{
        baseURL: baseURL,
        client:  NewHTTPClient(),
    }
}

type PromQueryResult struct {
    Status string `json:"status"`
    Data   struct {
        ResultType string `json:"resultType"`
        Result     []struct {
            Metric map[string]string `json:"metric"`
            Value  [2]interface{}    `json:"value"` // [timestamp, "value"]
        } `json:"result"`
    } `json:"data"`
}

func (p *PrometheusClient) Query(
    ctx context.Context,
    query string,
) (*PromQueryResult, error) {
    url := fmt.Sprintf("%s/api/v1/query?query=%s", p.baseURL, 
        neturl.QueryEscape(query))

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, fmt.Errorf("creating request: %w", err)
    }

    resp, err := DoWithRetry(ctx, p.client, req, 3)
    if err != nil {
        return nil, fmt.Errorf("querying prometheus: %w", err)
    }
    defer resp.Body.Close()

    var result PromQueryResult
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, fmt.Errorf("decoding response: %w", err)
    }

    if result.Status != "success" {
        return nil, fmt.Errorf("prometheus query failed: %s", result.Status)
    }

    return &result, nil
}


// ═══════════════════════════════════════════════════════════
// HTTP SERVER — webhook handler, health endpoints, metrics
// ═══════════════════════════════════════════════════════════

package server

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/prometheus/client_golang/prometheus/promhttp"
    "github.com/rs/zerolog/log"
)

type Server struct {
    httpServer *http.Server
    mux        *http.ServeMux
}

func New(addr string) *Server {
    mux := http.NewServeMux()

    s := &Server{
        httpServer: &http.Server{
            Addr:         addr,
            Handler:      mux,
            ReadTimeout:  10 * time.Second,
            WriteTimeout: 30 * time.Second,
            IdleTimeout:  65 * time.Second, // > ALB idle timeout (60s)
        },
        mux: mux,
    }

    // Standard endpoints
    mux.HandleFunc("/healthz", s.healthz)
    mux.HandleFunc("/readyz", s.readyz)
    mux.Handle("/metrics", promhttp.Handler())

    return s
}

func (s *Server) healthz(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}

func (s *Server) readyz(w http.ResponseWriter, r *http.Request) {
    // Check dependencies here
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "ready"})
}

// HandleWebhook registers a webhook handler with signature verification
func (s *Server) HandleWebhook(
    path string,
    secret string,
    handler func(payload []byte) error,
) {
    s.mux.HandleFunc(path, func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodPost {
            http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
            return
        }

        body, err := io.ReadAll(io.LimitReader(r.Body, 1<<20)) // 1MB limit
        if err != nil {
            http.Error(w, "bad request", http.StatusBadRequest)
            return
        }

        // Verify HMAC signature
        if secret != "" {
            sig := r.Header.Get("X-Hub-Signature-256")
            if !verifyHMAC(body, sig, secret) {
                log.Warn().Str("path", path).Msg("invalid_webhook_signature")
                http.Error(w, "unauthorized", http.StatusUnauthorized)
                return
            }
        }

        // Process in background
        go func() {
            if err := handler(body); err != nil {
                log.Error().Err(err).Str("path", path).Msg("webhook_handler_failed")
            }
        }()

        w.WriteHeader(http.StatusAccepted)
        json.NewEncoder(w).Encode(map[string]string{"status": "accepted"})
    })
}

// Run starts the server with graceful shutdown
func (s *Server) Run(ctx context.Context) error {
    // Graceful shutdown on SIGTERM/SIGINT
    ctx, stop := signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)
    defer stop()

    // Start server
    go func() {
        log.Info().Str("addr", s.httpServer.Addr).Msg("server_starting")
        if err := s.httpServer.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal().Err(err).Msg("server_failed")
        }
    }()

    // Wait for shutdown signal
    <-ctx.Done()
    log.Info().Msg("shutdown_signal_received")

    // Graceful shutdown with timeout
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := s.httpServer.Shutdown(shutdownCtx); err != nil {
        return fmt.Errorf("server shutdown failed: %w", err)
    }

    log.Info().Msg("server_stopped_gracefully")
    return nil
}
```

---

## 8. KUBERNETES CLIENT-GO — THE CORE

```go
// ═══════════════════════════════════════════════════════════
// client-go is the official Go client for Kubernetes.
// kubectl is built on it. Every K8s controller uses it.
// This is NON-OPTIONAL for a Senior DevOps Go programmer.
// ═══════════════════════════════════════════════════════════

package k8s

import (
    "context"
    "fmt"
    "path/filepath"
    "sort"
    "time"

    "github.com/rs/zerolog/log"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/labels"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/util/homedir"
)

// Client wraps the Kubernetes clientset with NovaMart operations
type Client struct {
    clientset *kubernetes.Clientset
    config    *rest.Config
}

// NewClient creates a K8s client.
// In-cluster: auto-detects pod service account.
// Out-of-cluster: uses kubeconfig (~/.kube/config or KUBECONFIG).
func NewClient(contextOverride string) (*Client, error) {
    var config *rest.Config
    var err error

    // Try in-cluster first (running inside K8s)
    config, err = rest.InClusterConfig()
    if err != nil {
        // Fall back to kubeconfig
        kubeconfig := filepath.Join(homedir.HomeDir(), ".kube", "config")

        if contextOverride != "" {
            // Use specific context
            config, err = clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
                &clientcmd.ClientConfigLoadingRules{ExplicitPath: kubeconfig},
                &clientcmd.ConfigOverrides{CurrentContext: contextOverride},
            ).ClientConfig()
        } else {
            config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
        }
        if err != nil {
            return nil, fmt.Errorf("building k8s config: %w", err)
        }
    }

    // Tune client for production
    config.QPS = 50          // Queries per second (default: 5)
    config.Burst = 100       // Burst capacity (default: 10)
    config.Timeout = 30 * time.Second

    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        return nil, fmt.Errorf("creating clientset: %w", err)
    }

    return &Client{clientset: clientset, config: config}, nil
}


// ─── Pod operations ───────────────────────────────────────

type PodInfo struct {
    Name       string            `json:"name"`
    Namespace  string            `json:"namespace"`
    Status     string            `json:"status"`
    Node       string            `json:"node"`
    IP         string            `json:"ip"`
    Restarts   int32             `json:"restarts"`
    Age        string            `json:"age"`
    Ready      string            `json:"ready"`
    CPUReq     string            `json:"cpu_request"`
    MemReq     string            `json:"memory_request"`
    CPULim     string            `json:"cpu_limit"`
    MemLim     string            `json:"memory_limit"`
    Labels     map[string]string `json:"labels"`
    Containers []ContainerInfo   `json:"containers"`
}

type ContainerInfo struct {
    Name         string `json:"name"`
    Image        string `json:"image"`
    Ready        bool   `json:"ready"`
    RestartCount int32  `json:"restart_count"`
    State        string `json:"state"`
    Reason       string `json:"reason,omitempty"`
}

func (c *Client) ListPods(
    ctx context.Context,
    namespace string,
    labelSelector string,
) ([]PodInfo, error) {
    opts := metav1.ListOptions{}
    if labelSelector != "" {
        opts.LabelSelector = labelSelector
    }

    var pods *corev1.PodList
    var err error

    if namespace == "" {
        pods, err = c.clientset.CoreV1().Pods("").List(ctx, opts)
    } else {
        pods, err = c.clientset.CoreV1().Pods(namespace).List(ctx, opts)
    }
    if err != nil {
        return nil, fmt.Errorf("listing pods: %w", err)
    }

    result := make([]PodInfo, 0, len(pods.Items))
    for _, pod := range pods.Items {
        result = append(result, podToInfo(&pod))
    }

    return result, nil
}

func podToInfo(pod *corev1.Pod) PodInfo {
    info := PodInfo{
        Name:      pod.Name,
        Namespace: pod.Namespace,
        Status:    string(pod.Status.Phase),
        Node:      pod.Spec.NodeName,
        IP:        pod.Status.PodIP,
        Age:       time.Since(pod.CreationTimestamp.Time).Round(time.Second).String(),
        Labels:    pod.Labels,
    }

    // Calculate ready count
    readyCount := 0
    totalCount := len(pod.Spec.Containers)
    var totalRestarts int32

    for _, cs := range pod.Status.ContainerStatuses {
        ci := ContainerInfo{
            Name:         cs.Name,
            Image:        cs.Image,
            Ready:        cs.Ready,
            RestartCount: cs.RestartCount,
        }

        if cs.State.Running != nil {
            ci.State = "Running"
        } else if cs.State.Waiting != nil {
            ci.State = "Waiting"
            ci.Reason = cs.State.Waiting.Reason
        } else if cs.State.Terminated != nil {
            ci.State = "Terminated"
            ci.Reason = cs.State.Terminated.Reason
        }

        if cs.Ready {
            readyCount++
        }
        totalRestarts += cs.RestartCount

        info.Containers = append(info.Containers, ci)
    }

    info.Ready = fmt.Sprintf("%d/%d", readyCount, totalCount)
    info.Restarts = totalRestarts

    // Resource requests/limits
    var cpuReq, memReq, cpuLim, memLim resource.Quantity
    for _, c := range pod.Spec.Containers {
        if c.Resources.Requests != nil {
            cpuReq.Add(*c.Resources.Requests.Cpu())
            memReq.Add(*c.Resources.Requests.Memory())
        }
        if c.Resources.Limits != nil {
            cpuLim.Add(*c.Resources.Limits.Cpu())
            memLim.Add(*c.Resources.Limits.Memory())
        }
    }
    info.CPUReq = cpuReq.String()
    info.MemReq = memReq.String()
    info.CPULim = cpuLim.String()
    info.MemLim = memLim.String()

    // Override status with more specific condition
    for _, cond := range pod.Status.Conditions {
        if cond.Type == corev1.PodReady && cond.Status != corev1.ConditionTrue {
            info.Status = cond.Reason
        }
    }
    // CrashLoopBackOff shows in container status
    for _, cs := range pod.Status.ContainerStatuses {
        if cs.State.Waiting != nil && cs.State.Waiting.Reason == "CrashLoopBackOff" {
            info.Status = "CrashLoopBackOff"
        }
    }

    return info
}


// ─── Deployment operations ────────────────────────────────

type DeploymentStatus struct {
    Name              string `json:"name"`
    Namespace         string `json:"namespace"`
    Replicas          int32  `json:"replicas"`
    ReadyReplicas     int32  `json:"ready_replicas"`
    UpdatedReplicas   int32  `json:"updated_replicas"`
    AvailableReplicas int32  `json:"available_replicas"`
    Image             string `json:"image"`
    Strategy          string `json:"strategy"`
    Age               string `json:"age"`
    Conditions        []string `json:"conditions"`
}

func (c *Client) DeploymentStatus(
    ctx context.Context,
    namespace, name string,
) (*DeploymentStatus, error) {
    dep, err := c.clientset.AppsV1().Deployments(namespace).Get(ctx, name, metav1.GetOptions{})
    if err != nil {
        return nil, fmt.Errorf("getting deployment %s/%s: %w", namespace, name, err)
    }

    status := &DeploymentStatus{
        Name:              dep.Name,
        Namespace:         dep.Namespace,
        Replicas:          *dep.Spec.Replicas,
        ReadyReplicas:     dep.Status.ReadyReplicas,
        UpdatedReplicas:   dep.Status.UpdatedReplicas,
        AvailableReplicas: dep.Status.AvailableReplicas,
        Strategy:          string(dep.Spec.Strategy.Type),
        Age:               time.Since(dep.CreationTimestamp.Time).Round(time.Second).String(),
    }

    // Get primary container image
    if len(dep.Spec.Template.Spec.Containers) > 0 {
        status.Image = dep.Spec.Template.Spec.Containers[0].Image
    }

    // Extract conditions
    for _, cond := range dep.Status.Conditions {
        status.Conditions = append(status.Conditions,
            fmt.Sprintf("%s=%s (%s)", cond.Type, cond.Status, cond.Reason))
    }

    return status, nil
}

func (c *Client) Rollback(
    ctx context.Context,
    namespace, name string,
    revision int64,
    dryRun bool,
) (string, error) {
    log.Info().
        Str("deployment", name).
        Str("namespace", namespace).
        Int64("revision", revision).
        Bool("dry_run", dryRun).
        Msg("initiating_rollback")

    // Get current deployment
    dep, err := c.clientset.AppsV1().Deployments(namespace).Get(ctx, name, metav1.GetOptions{})
    if err != nil {
        return "", fmt.Errorf("getting deployment: %w", err)
    }

    currentImage := dep.Spec.Template.Spec.Containers[0].Image

    // Get ReplicaSets to find previous revision
    rsList, err := c.clientset.AppsV1().ReplicaSets(namespace).List(ctx, metav1.ListOptions{
        LabelSelector: labels.Set(dep.Spec.Selector.MatchLabels).String(),
    })
    if err != nil {
        return "", fmt.Errorf("listing replicasets: %w", err)
    }

    // Sort by revision (annotation)
    type rsRevision struct {
        rs       appsv1.ReplicaSet
        revision int64
    }
    var revisions []rsRevision
    for _, rs := range rsList.Items {
        rev := int64(0)
        if revStr, ok := rs.Annotations["deployment.kubernetes.io/revision"]; ok {
            fmt.Sscanf(revStr, "%d", &rev)
        }
        revisions = append(revisions, rsRevision{rs: rs, revision: rev})
    }
    sort.Slice(revisions, func(i, j int) bool {
        return revisions[i].revision > revisions[j].revision
    })

    if len(revisions) < 2 {
        return "", fmt.Errorf("no previous revision found for %s", name)
    }

    // Target revision: specific or previous
    var targetRS appsv1.ReplicaSet
    if revision > 0 {
        found := false
        for _, r := range revisions {
            if r.revision == revision {
                targetRS = r.rs
                found = true
                break
            }
        }
        if !found {
            return "", fmt.Errorf("revision %d not found", revision)
        }
    } else {
        // Previous revision (second in sorted list)
        targetRS = revisions[1].rs
    }

    targetImage := targetRS.Spec.Template.Spec.Containers[0].Image

    if dryRun {
        return fmt.Sprintf("would rollback %s from %s to %s",
            name, currentImage, targetImage), nil
    }

    // Perform rollback by updating the deployment with the target RS template
    dep.Spec.Template = targetRS.Spec.Template
    _, err = c.clientset.AppsV1().Deployments(namespace).Update(ctx, dep, metav1.UpdateOptions{})
    if err != nil {
        return "", fmt.Errorf("updating deployment: %w", err)
    }

    return fmt.Sprintf("rolled back %s from %s to %s", name, currentImage, targetImage), nil
}

func (c *Client) ScaleDeployment(
    ctx context.Context,
    namespace, name string,
    replicas int32,
    dryRun bool,
) error {
    log.Info().
        Str("deployment", name).
        Int32("replicas", replicas).
        Bool("dry_run", dryRun).
        Msg("scaling_deployment")

    scale, err := c.clientset.AppsV1().Deployments(namespace).GetScale(
        ctx, name, metav1.GetOptions{})
    if err != nil {
        return fmt.Errorf("getting scale: %w", err)
    }

    oldReplicas := scale.Spec.Replicas
    scale.Spec.Replicas = replicas

    opts := metav1.UpdateOptions{}
    if dryRun {
        opts.DryRun = []string{metav1.DryRunAll}
    }

    _, err = c.clientset.AppsV1().Deployments(namespace).UpdateScale(
        ctx, name, scale, opts)
    if err != nil {
        return fmt.Errorf("updating scale: %w", err)
    }

    log.Info().
        Str("deployment", name).
        Int32("from", oldReplicas).
        Int32("to", replicas).
        Msg("scale_updated")

    return nil
}


// ─── Node operations ──────────────────────────────────────

type NodeInfo struct {
    Name             string            `json:"name"`
    Status           string            `json:"status"`
    Roles            []string          `json:"roles"`
    Version          string            `json:"version"`
    InstanceType     string            `json:"instance_type"`
    AZ               string            `json:"az"`
    Age              string            `json:"age"`
    PodCount         int               `json:"pod_count"`
    CPUCapacity      string            `json:"cpu_capacity"`
    MemCapacity      string            `json:"memory_capacity"`
    CPUAllocatable   string            `json:"cpu_allocatable"`
    MemAllocatable   string            `json:"memory_allocatable"`
    Taints           []string          `json:"taints,omitempty"`
    Unschedulable    bool              `json:"unschedulable"`
    Labels           map[string]string `json:"labels"`
}

func (c *Client) ListNodes(ctx context.Context) ([]NodeInfo, error) {
    nodes, err := c.clientset.CoreV1().Nodes().List(ctx, metav1.ListOptions{})
    if err != nil {
        return nil, fmt.Errorf("listing nodes: %w", err)
    }

    // Get pod counts per node
    pods, err := c.clientset.CoreV1().Pods("").List(ctx, metav1.ListOptions{})
    if err != nil {
        return nil, fmt.Errorf("listing pods: %w", err)
    }

    podCountByNode := make(map[string]int)
    for _, pod := range pods.Items {
        if pod.Spec.NodeName != "" {
            podCountByNode[pod.Spec.NodeName]++
        }
    }

    result := make([]NodeInfo, 0, len(nodes.Items))
    for _, node := range nodes.Items {
        info := NodeInfo{
            Name:           node.Name,
            Version:        node.Status.NodeInfo.KubeletVersion,
            InstanceType:   node.Labels["node.kubernetes.io/instance-type"],
            AZ:             node.Labels["topology.kubernetes.io/zone"],
            Age:            time.Since(node.CreationTimestamp.Time).Round(time.Hour).String(),
            PodCount:       podCountByNode[node.Name],
            CPUCapacity:    node.Status.Capacity.Cpu().String(),
            MemCapacity:    node.Status.Capacity.Memory().String(),
            CPUAllocatable: node.Status.Allocatable.Cpu().String(),
            MemAllocatable: node.Status.Allocatable.Memory().String(),
            Unschedulable:  node.Spec.Unschedulable,
            Labels:         node.Labels,
        }

        // Node status
        for _, cond := range node.Status.Conditions {
            if cond.Type == corev1.NodeReady {
                if cond.Status == corev1.ConditionTrue {
                    info.Status = "Ready"
                } else {
                    info.Status = "NotReady"
                }
            }
        }
        if node.Spec.Unschedulable {
            info.Status += ",SchedulingDisabled"
        }

        // Roles
        for label := range node.Labels {
            if label == "node-role.kubernetes.io/control-plane" ||
                label == "node-role.kubernetes.io/master" {
                info.Roles = append(info.Roles, "control-plane")
            }
        }
        if len(info.Roles) == 0 {
            info.Roles = []string{"worker"}
        }

        // Taints
        for _, taint := range node.Spec.Taints {
            info.Taints = append(info.Taints,
                fmt.Sprintf("%s=%s:%s", taint.Key, taint.Value, taint.Effect))
        }

        result = append(result, info)
    }

    return result, nil
}

func (c *Client) CordonNode(ctx context.Context, nodeName string, dryRun bool) error {
    node, err := c.clientset.CoreV1().Nodes().Get(ctx, nodeName, metav1.GetOptions{})
    if err != nil {
        return fmt.Errorf("getting node %s: %w", nodeName, err)
    }

    if node.Spec.Unschedulable {
        log.Info().Str("node", nodeName).Msg("node_already_cordoned")
        return nil
    }

    node.Spec.Unschedulable = true

    opts := metav1.UpdateOptions{}
    if dryRun {
        opts.DryRun = []string{metav1.DryRunAll}
    }

    _, err = c.clientset.CoreV1().Nodes().Update(ctx, node, opts)
    if err != nil {
        return fmt.Errorf("cordoning node %s: %w", nodeName, err)
    }

    log.Info().Str("node", nodeName).Bool("dry_run", dryRun).Msg("node_cordoned")
    return nil
}


// ─── Events — for debugging ──────────────────────────────

type EventInfo struct {
    Time      string `json:"time"`
    Type      string `json:"type"`
    Reason    string `json:"reason"`
    Object    string `json:"object"`
    Message   string `json:"message"`
    Count     int32  `json:"count"`
    Source    string `json:"source"`
}

func (c *Client) GetEvents(
    ctx context.Context,
    namespace string,
    resourceName string,
    since time.Duration,
) ([]EventInfo, error) {
    opts := metav1.ListOptions{}
    if resourceName != "" {
        opts.FieldSelector = fmt.Sprintf("involvedObject.name=%s", resourceName)
    }

    events, err := c.clientset.CoreV1().Events(namespace).List(ctx, opts)
    if err != nil {
        return nil, fmt.Errorf("listing events: %w", err)
    }

    cutoff := time.Now().Add(-since)
    result := make([]EventInfo, 0)

    for _, ev := range events.Items {
        eventTime := ev.LastTimestamp.Time
        if ev.EventTime.Time.After(eventTime) {
            eventTime = ev.EventTime.Time
        }
        if eventTime.Before(cutoff) {
            continue
        }

        result = append(result, EventInfo{
            Time:    eventTime.Format(time.RFC3339),
            Type:    ev.Type,
            Reason:  ev.Reason,
            Object:  fmt.Sprintf("%s/%s", ev.InvolvedObject.Kind, ev.InvolvedObject.Name),
            Message: ev.Message,
            Count:   ev.Count,
            Source:  ev.Source.Component,
        })
    }

    // Sort by time descending
    sort.Slice(result, func(i, j int) bool {
        return result[i].Time > result[j].Time
    })

    return result, nil
}


// ─── Cluster Audit ────────────────────────────────────────

type AuditFinding struct {
    Severity  string `json:"severity"` // CRITICAL, HIGH, MEDIUM, LOW
    Category  string `json:"category"`
    Resource  string `json:"resource"`
    Namespace string `json:"namespace"`
    Name      string `json:"name"`
    Message   string `json:"message"`
    Fix       string `json:"fix"`
}

func (c *Client) Audit(ctx context.Context, namespace string) ([]AuditFinding, error) {
    var findings []AuditFinding

    // Check 1: Deployments without resource limits
    depFindings, err := c.auditResourceLimits(ctx, namespace)
    if err != nil {
        log.Warn().Err(err).Msg("resource_limits_audit_failed")
    }
    findings = append(findings, depFindings...)

    // Check 2: Single replica deployments in production
    replicaFindings, err := c.auditReplicaCounts(ctx, namespace)
    if err != nil {
        log.Warn().Err(err).Msg("replica_count_audit_failed")
    }
    findings = append(findings, replicaFindings...)

    // Check 3: Pods using :latest tag
    imageFindings, err := c.auditImageTags(ctx, namespace)
    if err != nil {
        log.Warn().Err(err).Msg("image_tag_audit_failed")
    }
    findings = append(findings, imageFindings...)

    // Check 4: Missing PDBs
    pdbFindings, err := c.auditPDBs(ctx, namespace)
    if err != nil {
        log.Warn().Err(err).Msg("pdb_audit_failed")
    }
    findings = append(findings, pdbFindings...)

    // Check 5: Missing probes
    probeFindings, err := c.auditProbes(ctx, namespace)
    if err != nil {
        log.Warn().Err(err).Msg("probe_audit_failed")
    }
    findings = append(findings, probeFindings...)

    // Check 6: Orphaned ConfigMaps/Secrets
    orphanFindings, err := c.auditOrphans(ctx, namespace)
    if err != nil {
        log.Warn().Err(err).Msg("orphan_audit_failed")
    }
    findings = append(findings, orphanFindings...)

    // Sort by severity
    severityOrder := map[string]int{"CRITICAL": 0, "HIGH": 1, "MEDIUM": 2, "LOW": 3}
    sort.Slice(findings, func(i, j int) bool {
        return severityOrder[findings[i].Severity] < severityOrder[findings[j].Severity]
    })

    return findings, nil
}

func (c *Client) auditResourceLimits(ctx context.Context, namespace string) ([]AuditFinding, error) {
    var findings []AuditFinding
    listOpts := metav1.ListOptions{}

    var deps *appsv1.DeploymentList
    var err error
    if namespace == "" {
        deps, err = c.clientset.AppsV1().Deployments("").List(ctx, listOpts)
    } else {
        deps, err = c.clientset.AppsV1().Deployments(namespace).List(ctx, listOpts)
    }
    if err != nil {
        return nil, err
    }

    for _, dep := range deps.Items {
        for _, container := range dep.Spec.Template.Spec.Containers {
            if container.Resources.Limits == nil ||
                container.Resources.Limits.Cpu().IsZero() ||
                container.Resources.Limits.Memory().IsZero() {
                findings = append(findings, AuditFinding{
                    Severity:  "HIGH",
                    Category:  "resources",
                    Resource:  "Deployment",
                    Namespace: dep.Namespace,
                    Name:      dep.Name,
                    Message:   fmt.Sprintf("Container '%s' missing CPU/memory limits", container.Name),
                    Fix:       "Add resources.limits.cpu and resources.limits.memory",
                })
            }
            if container.Resources.Requests == nil ||
                container.Resources.Requests.Cpu().IsZero() ||
                container.Resources.Requests.Memory().IsZero() {
                findings = append(findings, AuditFinding{
                    Severity:  "MEDIUM",
                    Category:  "resources",
                    Resource:  "Deployment",
                    Namespace: dep.Namespace,
                    Name:      dep.Name,
                    Message:   fmt.Sprintf("Container '%s' missing CPU/memory requests", container.Name),
                    Fix:       "Add resources.requests.cpu and resources.requests.memory",
                })
            }
        }
    }
    return findings, nil
}

func (c *Client) auditImageTags(ctx context.Context, namespace string) ([]AuditFinding, error) {
    var findings []AuditFinding

    pods, err := c.ListPods(ctx, namespace, "")
    if err != nil {
        return nil, err
    }

    for _, pod := range pods {
        for _, container := range pod.Containers {
            image := container.Image
            // Check for :latest or no tag
            if strings.HasSuffix(image, ":latest") {
                findings = append(findings, AuditFinding{
                    Severity:  "CRITICAL",
                    Category:  "image",
                    Resource:  "Pod",
                    Namespace: pod.Namespace,
                    Name:      pod.Name,
                    Message:   fmt.Sprintf("Container '%s' uses :latest tag: %s", container.Name, image),
                    Fix:       "Pin to specific version tag or SHA digest",
                })
            } else if !strings.Contains(image, ":") && !strings.Contains(image, "@") {
                findings = append(findings, AuditFinding{
                    Severity:  "CRITICAL",
                    Category:  "image",
                    Resource:  "Pod",
                    Namespace: pod.Namespace,
                    Name:      pod.Name,
                    Message:   fmt.Sprintf("Container '%s' uses untagged image: %s", container.Name, image),
                    Fix:       "Always specify image tag or SHA digest",
                })
            }
        }
    }
    return findings, nil
}

func (c *Client) auditReplicaCounts(ctx context.Context, namespace string) ([]AuditFinding, error) {
    var findings []AuditFinding

    var deps *appsv1.DeploymentList
    var err error
    if namespace == "" {
        deps, err = c.clientset.AppsV1().Deployments("").List(ctx, metav1.ListOptions{})
    } else {
        deps, err = c.clientset.AppsV1().Deployments(namespace).List(ctx, metav1.ListOptions{})
    }
    if err != nil {
        return nil, err
    }

    for _, dep := range deps.Items {
        // Skip system namespaces
        if dep.Namespace == "kube-system" || dep.Namespace == "kube-node-lease" {
            continue
        }
        if dep.Spec.Replicas != nil && *dep.Spec.Replicas < 2 {
            findings = append(findings, AuditFinding{
                Severity:  "HIGH",
                Category:  "availability",
                Resource:  "Deployment",
                Namespace: dep.Namespace,
                Name:      dep.Name,
                Message:   fmt.Sprintf("Only %d replica(s) — no HA", *dep.Spec.Replicas),
                Fix:       "Set replicas >= 2 with PDB for availability",
            })
        }
    }
    return findings, nil
}

func (c *Client) auditPDBs(ctx context.Context, namespace string) ([]AuditFinding, error) {
    var findings []AuditFinding

    var deps *appsv1.DeploymentList
    var err error
    if namespace == "" {
        deps, err = c.clientset.AppsV1().Deployments("").List(ctx, metav1.ListOptions{})
    } else {
        deps, err = c.clientset.AppsV1().Deployments(namespace).List(ctx, metav1.ListOptions{})
    }
    if err != nil {
        return nil, err
    }

    // Get all PDBs
    pdbClient := c.clientset.PolicyV1().PodDisruptionBudgets
    var allPDBs []policyv1.PodDisruptionBudget

    if namespace == "" {
        pdbList, err := pdbClient("").List(ctx, metav1.ListOptions{})
        if err != nil {
            return nil, err
        }
        allPDBs = pdbList.Items
    } else {
        pdbList, err := pdbClient(namespace).List(ctx, metav1.ListOptions{})
        if err != nil {
            return nil, err
        }
        allPDBs = pdbList.Items
    }

    // Index PDBs by namespace
    pdbByNS := make(map[string][]policyv1.PodDisruptionBudget)
    for _, pdb := range allPDBs {
        pdbByNS[pdb.Namespace] = append(pdbByNS[pdb.Namespace], pdb)
    }

    for _, dep := range deps.Items {
        if dep.Namespace == "kube-system" {
            continue
        }
        if dep.Spec.Replicas != nil && *dep.Spec.Replicas < 2 {
            continue // PDB only makes sense with 2+ replicas
        }

        hasPDB := false
        for _, pdb := range pdbByNS[dep.Namespace] {
            selector, err := metav1.LabelSelectorAsSelector(pdb.Spec.Selector)
            if err != nil {
                continue
            }
            if selector.Matches(labels.Set(dep.Spec.Template.Labels)) {
                hasPDB = true
                break
            }
        }

        if !hasPDB {
            findings = append(findings, AuditFinding{
                Severity:  "MEDIUM",
                Category:  "availability",
                Resource:  "Deployment",
                Namespace: dep.Namespace,
                Name:      dep.Name,
                Message:   "No PodDisruptionBudget found",
                Fix:       "Create PDB with minAvailable or maxUnavailable",
            })
        }
    }
    return findings, nil
}

func (c *Client) auditProbes(ctx context.Context, namespace string) ([]AuditFinding, error) {
    var findings []AuditFinding

    var deps *appsv1.DeploymentList
    var err error
    if namespace == "" {
        deps, err = c.clientset.AppsV1().Deployments("").List(ctx, metav1.ListOptions{})
    } else {
        deps, err = c.clientset.AppsV1().Deployments(namespace).List(ctx, metav1.ListOptions{})
    }
    if err != nil {
        return nil, err
    }

    for _, dep := range deps.Items {
        if dep.Namespace == "kube-system" {
            continue
        }
        for _, container := range dep.Spec.Template.Spec.Containers {
            if container.ReadinessProbe == nil {
                findings = append(findings, AuditFinding{
                    Severity:  "HIGH",
                    Category:  "health",
                    Resource:  "Deployment",
                    Namespace: dep.Namespace,
                    Name:      dep.Name,
                    Message:   fmt.Sprintf("Container '%s' missing readinessProbe", container.Name),
                    Fix:       "Add readinessProbe — without it, traffic hits unready pods",
                })
            }
            if container.LivenessProbe == nil {
                findings = append(findings, AuditFinding{
                    Severity:  "MEDIUM",
                    Category:  "health",
                    Resource:  "Deployment",
                    Namespace: dep.Namespace,
                    Name:      dep.Name,
                    Message:   fmt.Sprintf("Container '%s' missing livenessProbe", container.Name),
                    Fix:       "Add livenessProbe — but be cautious, bad probes cause restart loops",
                })
            }
        }
    }
    return findings, nil
}

func (c *Client) auditOrphans(ctx context.Context, namespace string) ([]AuditFinding, error) {
    var findings []AuditFinding

    // List ConfigMaps
    var cms *corev1.ConfigMapList
    var err error
    if namespace == "" {
        cms, err = c.clientset.CoreV1().ConfigMaps("").List(ctx, metav1.ListOptions{})
    } else {
        cms, err = c.clientset.CoreV1().ConfigMaps(namespace).List(ctx, metav1.ListOptions{})
    }
    if err != nil {
        return nil, err
    }

    // List all pods to find referenced ConfigMaps
    pods, err := c.clientset.CoreV1().Pods("").List(ctx, metav1.ListOptions{})
    if err != nil {
        return nil, err
    }

    referencedCMs := make(map[string]bool)
    for _, pod := range pods.Items {
        for _, vol := range pod.Spec.Volumes {
            if vol.ConfigMap != nil {
                key := pod.Namespace + "/" + vol.ConfigMap.Name
                referencedCMs[key] = true
            }
        }
        for _, container := range pod.Spec.Containers {
            for _, env := range container.EnvFrom {
                if env.ConfigMapRef != nil {
                    key := pod.Namespace + "/" + env.ConfigMapRef.Name
                    referencedCMs[key] = true
                }
            }
            for _, env := range container.Env {
                if env.ValueFrom != nil && env.ValueFrom.ConfigMapKeyRef != nil {
                    key := pod.Namespace + "/" + env.ValueFrom.ConfigMapKeyRef.Name
                    referencedCMs[key] = true
                }
            }
        }
    }

    for _, cm := range cms.Items {
        // Skip system CMs
        if cm.Namespace == "kube-system" || cm.Namespace == "kube-public" {
            continue
        }
        // Skip Helm, ArgoCD, Istio managed
        if _, ok := cm.Labels["app.kubernetes.io/managed-by"]; ok {
            continue
        }

        key := cm.Namespace + "/" + cm.Name
        if !referencedCMs[key] {
            age := time.Since(cm.CreationTimestamp.Time)
            if age > 7*24*time.Hour { // Only flag if older than 7 days
                findings = append(findings, AuditFinding{
                    Severity:  "LOW",
                    Category:  "cleanup",
                    Resource:  "ConfigMap",
                    Namespace: cm.Namespace,
                    Name:      cm.Name,
                    Message:   fmt.Sprintf("Unreferenced ConfigMap (age: %s)", age.Round(time.Hour)),
                    Fix:       "Verify not needed, then delete to reduce etcd size",
                })
            }
        }
    }

    return findings, nil
}
```

---

## 9. INFORMERS AND WATCHES — REAL-TIME K8S DATA

```go
// ═══════════════════════════════════════════════════════════
// Informers are the CORRECT way to watch K8s resources.
// They maintain a local cache, reducing API server load.
// Every K8s controller uses informers. Non-optional knowledge.
// ═══════════════════════════════════════════════════════════

package watcher

import (
    "context"
    "fmt"
    "time"

    "github.com/rs/zerolog/log"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
)

// PodWatcher watches for pod events using SharedInformers
type PodWatcher struct {
    clientset *kubernetes.Clientset
    stopCh    chan struct{}
}

func NewPodWatcher(clientset *kubernetes.Clientset) *PodWatcher {
    return &PodWatcher{
        clientset: clientset,
        stopCh:    make(chan struct{}),
    }
}

func (w *PodWatcher) Start(ctx context.Context, namespace string) error {
    // SharedInformerFactory creates informers that share a cache
    factory := informers.NewSharedInformerFactoryWithOptions(
        w.clientset,
        30*time.Second, // Resync period — full re-list interval
        informers.WithNamespace(namespace),
    )

    // Get the Pod informer
    podInformer := factory.Core().V1().Pods().Informer()

    // Register event handlers
    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod, ok := obj.(*corev1.Pod)
            if !ok {
                return
            }
            log.Info().
                Str("pod", pod.Name).
                Str("namespace", pod.Namespace).
                Str("phase", string(pod.Status.Phase)).
                Msg("pod_added")
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            oldPod := oldObj.(*corev1.Pod)
            newPod := newObj.(*corev1.Pod)

            // Only log meaningful changes
            if oldPod.Status.Phase != newPod.Status.Phase {
                log.Info().
                    Str("pod", newPod.Name).
                    Str("old_phase", string(oldPod.Status.Phase)).
                    Str("new_phase", string(newPod.Status.Phase)).
                    Msg("pod_status_changed")
            }

            // Detect OOMKilled
            for _, cs := range newPod.Status.ContainerStatuses {
                if cs.State.Terminated != nil &&
                    cs.State.Terminated.Reason == "OOMKilled" {
                    log.Error().
                        Str("pod", newPod.Name).
                        Str("container", cs.Name).
                        Msg("container_oom_killed")
                    // Trigger alert, notification, etc.
                }
            }

            // Detect CrashLoopBackOff
            for _, cs := range newPod.Status.ContainerStatuses {
                if cs.State.Waiting != nil &&
                    cs.State.Waiting.Reason == "CrashLoopBackOff" &&
                    cs.RestartCount > 5 {
                    log.Error().
                        Str("pod", newPod.Name).
                        Str("container", cs.Name).
                        Int32("restarts", cs.RestartCount).
                        Msg("crash_loop_detected")
                }
            }
        },
        DeleteFunc: func(obj interface{}) {
            pod, ok := obj.(*corev1.Pod)
            if !ok {
                // Handle tombstone (deleted object we missed)
                tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
                if !ok {
                    return
                }
                pod, ok = tombstone.Obj.(*corev1.Pod)
                if !ok {
                    return
                }
            }
            log.Info().
                Str("pod", pod.Name).
                Str("namespace", pod.Namespace).
                Msg("pod_deleted")
        },
    })

    // Start the informer
    factory.Start(w.stopCh)

    // Wait for cache sync (CRITICAL — don't read from cache before sync)
    if !cache.WaitForCacheSync(ctx.Done(), podInformer.HasSynced) {
        return fmt.Errorf("failed to sync informer cache")
    }

    log.Info().Str("namespace", namespace).Msg("pod_watcher_started")

    // Use the LISTER for reads (cached, fast, no API call)
    lister := factory.Core().V1().Pods().Lister()
    pods, err := lister.Pods(namespace).List(labels.Everything())
    if err != nil {
        return fmt.Errorf("listing from cache: %w", err)
    }
    log.Info().Int("cached_pods", len(pods)).Msg("initial_cache_populated")

    // Block until context is cancelled
    <-ctx.Done()
    close(w.stopCh)
    return nil
}
```

**Informer failure modes:**
```go
// FAILURE 1: Reading cache before sync
// If you List() before WaitForCacheSync(), you get EMPTY results
// Not errors — just silently missing data
// ALWAYS call WaitForCacheSync() before reading

// FAILURE 2: Resync storm
// Resync period too short (e.g. 1s) = hammering API server
// Default 0 = no resync (only watch events). 30s is good for most cases.

// FAILURE 3: Event handler blocks
// Informer event handlers run SYNCHRONOUSLY in the same goroutine
// Slow handler = events queue up = stale data
// FIX: Use workqueue pattern — handler enqueues key, worker processes

// FAILURE 4: Stale cache used for writes
// Read from cache (lister) → make decision → write to API
// Between read and write, another controller changed the resource
// Result: 409 Conflict
// FIX: Always handle conflict errors with retry
```

---

## 10. BUILDING A K8S OPERATOR (KUBEBUILDER)

```go
// ═══════════════════════════════════════════════════════════
// Operators extend Kubernetes with custom behavior.
// NovaMart operator: automates service onboarding.
// 
// kubebuilder scaffold:
//   kubebuilder init --domain novamart.com --repo github.com/novamart/operator
//   kubebuilder create api --group platform --version v1 --kind NovaMartService
// ═══════════════════════════════════════════════════════════

// api/v1/novamartservice_types.go
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// NovaMartServiceSpec defines the desired state
type NovaMartServiceSpec struct {
    // Team that owns this service
    // +kubebuilder:validation:Required
    Team string `json:"team"`

    // Service tier determines resource defaults
    // +kubebuilder:validation:Enum=critical;standard;batch
    // +kubebuilder:default=standard
    Tier string `json:"tier,omitempty"`

    // Number of replicas
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=50
    // +kubebuilder:default=2
    Replicas int32 `json:"replicas,omitempty"`

    // Container image
    // +kubebuilder:validation:Required
    Image string `json:"image"`

    // Port the service listens on
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=65535
    // +kubebuilder:default=8080
    Port int32 `json:"port,omitempty"`

    // Enable monitoring (ServiceMonitor)
    // +kubebuilder:default=true
    Monitoring bool `json:"monitoring,omitempty"`

    // Enable network policies
    // +kubebuilder:default=true
    NetworkPolicy bool `json:"networkPolicy,omitempty"`

    // Resource profile
    Resources ResourceProfile `json:"resources,omitempty"`
}

type ResourceProfile struct {
    CPU    string `json:"cpu,omitempty"`
    Memory string `json:"memory,omitempty"`
}

// NovaMartServiceStatus defines the observed state
type NovaMartServiceStatus struct {
    // Current phase of the service
    // +kubebuilder:validation:Enum=Pending;Provisioning;Running;Error
    Phase string `json:"phase,omitempty"`

    // Human-readable message
    Message string `json:"message,omitempty"`

    // List of conditions
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // The deployment name
    DeploymentName string `json:"deploymentName,omitempty"`

    // Last reconciled generation
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Ready replicas
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Team",type=string,JSONPath=`.spec.team`
// +kubebuilder:printcolumn:name="Tier",type=string,JSONPath=`.spec.tier`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
type NovaMartService struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   NovaMartServiceSpec   `json:"spec,omitempty"`
    Status NovaMartServiceStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type NovaMartServiceList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []NovaMartService `json:"items"`
}


// ═══════════════════════════════════════════════════════════
// RECONCILER — The core logic
// ═══════════════════════════════════════════════════════════

// internal/controller/novamartservice_controller.go
package controller

import (
    "context"
    "fmt"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    networkingv1 "k8s.io/api/networking/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    "k8s.io/apimachinery/pkg/util/intstr"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"

    platformv1 "github.com/novamart/operator/api/v1"
)

const (
    finalizerName = "platform.novamart.com/cleanup"
)

type NovaMartServiceReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=platform.novamart.com,resources=novamartservices,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=platform.novamart.com,resources=novamartservices/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=platform.novamart.com,resources=novamartservices/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=networking.k8s.io,resources=networkpolicies,verbs=get;list;watch;create;update;patch;delete

func (r *NovaMartServiceReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,
) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // Step 1: Fetch the NovaMartService CR
    var svc platformv1.NovaMartService
    if err := r.Get(ctx, req.NamespacedName, &svc); err != nil {
        if errors.IsNotFound(err) {
            // CR deleted — nothing to do (finalizer already ran)
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, fmt.Errorf("fetching NovaMartService: %w", err)
    }

    // Step 2: Handle deletion (finalizer pattern)
    if !svc.DeletionTimestamp.IsZero() {
        return r.handleDeletion(ctx, &svc)
    }

    // Step 3: Add finalizer if missing
    if !controllerutil.ContainsFinalizer(&svc, finalizerName) {
        controllerutil.AddFinalizer(&svc, finalizerName)
        if err := r.Update(ctx, &svc); err != nil {
            return ctrl.Result{}, fmt.Errorf("adding finalizer: %w", err)
        }
    }

    // Step 4: Update status to Provisioning
    svc.Status.Phase = "Provisioning"
    svc.Status.ObservedGeneration = svc.Generation
    if err := r.Status().Update(ctx, &svc); err != nil {
        return ctrl.Result{}, fmt.Errorf("updating status: %w", err)
    }

    // Step 5: Reconcile child resources
    // Each function creates or updates the resource (idempotent)

    // 5a: Deployment
    if err := r.reconcileDeployment(ctx, &svc); err != nil {
        return r.setErrorStatus(ctx, &svc, "DeploymentFailed", err)
    }

    // 5b: Service
    if err := r.reconcileService(ctx, &svc); err != nil {
        return r.setErrorStatus(ctx, &svc, "ServiceFailed", err)
    }

    // 5c: NetworkPolicy (if enabled)
    if svc.Spec.NetworkPolicy {
        if err := r.reconcileNetworkPolicy(ctx, &svc); err != nil {
            return r.setErrorStatus(ctx, &svc, "NetworkPolicyFailed", err)
        }
    }

    // Step 6: Check deployment readiness
    var dep appsv1.Deployment
    depName := types.NamespacedName{
        Name:      svc.Name,
        Namespace: svc.Namespace,
    }
    if err := r.Get(ctx, depName, &dep); err != nil {
        return ctrl.Result{}, err
    }

    svc.Status.ReadyReplicas = dep.Status.ReadyReplicas
    svc.Status.DeploymentName = dep.Name

    if dep.Status.ReadyReplicas == *dep.Spec.Replicas {
        svc.Status.Phase = "Running"
        svc.Status.Message = "All replicas ready"
    } else {
        svc.Status.Phase = "Provisioning"
        svc.Status.Message = fmt.Sprintf("%d/%d replicas ready",
            dep.Status.ReadyReplicas, *dep.Spec.Replicas)
        // Requeue to check again
        return ctrl.Result{RequeueAfter: 10 * time.Second}, r.Status().Update(ctx, &svc)
    }

    if err := r.Status().Update(ctx, &svc); err != nil {
        return ctrl.Result{}, err
    }

    logger.Info("reconciliation complete",
        "service", svc.Name,
        "phase", svc.Status.Phase,
        "ready", svc.Status.ReadyReplicas)

    return ctrl.Result{}, nil
}

func (r *NovaMartServiceReconciler) reconcileDeployment(
    ctx context.Context,
    svc *platformv1.NovaMartService,
) error {
    // Determine resource defaults based on tier
    cpuReq, memReq, cpuLim, memLim := tierDefaults(svc.Spec.Tier)
    if svc.Spec.Resources.CPU != "" {
        cpuReq = svc.Spec.Resources.CPU
        cpuLim = svc.Spec.Resources.CPU
    }
    if svc.Spec.Resources.Memory != "" {
        memReq = svc.Spec.Resources.Memory
        memLim = svc.Spec.Resources.Memory
    }

    labels := map[string]string{
        "app":                          svc.Name,
        "app.kubernetes.io/name":       svc.Name,
        "app.kubernetes.io/managed-by": "novamart-operator",
        "team":                         svc.Spec.Team,
        "tier":                         svc.Spec.Tier,
    }

    dep := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      svc.Name,
            Namespace: svc.Namespace,
        },
    }

    // CreateOrUpdate — idempotent
    op, err := controllerutil.CreateOrUpdate(ctx, r.Client, dep, func() error {
        dep.Labels = labels
        dep.Spec.Replicas = &svc.Spec.Replicas
        dep.Spec.Selector = &metav1.LabelSelector{
            MatchLabels: map[string]string{"app": svc.Name},
        }
        dep.Spec.Template = corev1.PodTemplateSpec{
            ObjectMeta: metav1.ObjectMeta{Labels: labels},
            Spec: corev1.PodSpec{
                Containers: []corev1.Container{
                    {
                        Name:  svc.Name,
                        Image: svc.Spec.Image,
                        Ports: []corev1.ContainerPort{
                            {ContainerPort: svc.Spec.Port, Protocol: corev1.ProtocolTCP},
                        },
                        Resources: corev1.ResourceRequirements{
                            Requests: corev1.ResourceList{
                                corev1.ResourceCPU:    resource.MustParse(cpuReq),
                                corev1.ResourceMemory: resource.MustParse(memReq),
                            },
                            Limits: corev1.ResourceList{
                                corev1.ResourceCPU:    resource.MustParse(cpuLim),
                                corev1.ResourceMemory: resource.MustParse(memLim),
                            },
                        },
                        ReadinessProbe: &corev1.Probe{
                            ProbeHandler: corev1.ProbeHandler{
                                HTTPGet: &corev1.HTTPGetAction{
                                    Path: "/readyz",
                                    Port: intstr.FromInt32(svc.Spec.Port),
                                },
                            },
                            InitialDelaySeconds: 5,
                            PeriodSeconds:       10,
                        },
                        LivenessProbe: &corev1.Probe{
                            ProbeHandler: corev1.ProbeHandler{
                                HTTPGet: &corev1.HTTPGetAction{
                                    Path: "/healthz",
                                    Port: intstr.FromInt32(svc.Spec.Port),
                                },
                            },
                            InitialDelaySeconds: 15,
                            PeriodSeconds:       20,
                        },
                    },
                },
            },
        }

        // Set owner reference — garbage collection
        return controllerutil.SetControllerReference(svc, dep, r.Scheme)
    })

    if err != nil {
        return fmt.Errorf("reconciling deployment: %w", err)
    }

    log.FromContext(ctx).Info("deployment reconciled",
        "operation", op, "name", dep.Name)
    return nil
}

func (r *NovaMartServiceReconciler) reconcileService(
    ctx context.Context,
    svc *platformv1.NovaMartService,
) error {
    service := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      svc.Name,
            Namespace: svc.Namespace,
        },
    }

    op, err := controllerutil.CreateOrUpdate(ctx, r.Client, service, func() error {
        service.Spec.Selector = map[string]string{"app": svc.Name}
        service.Spec.Ports = []corev1.ServicePort{
            {
                Name:       "http",
                Port:       svc.Spec.Port,
                TargetPort: intstr.FromInt32(svc.Spec.Port),
                Protocol:   corev1.ProtocolTCP,
            },
        }
        return controllerutil.SetControllerReference(svc, service, r.Scheme)
    })

    if err != nil {
        return fmt.Errorf("reconciling service: %w", err)
    }

    log.FromContext(ctx).Info("service reconciled", "operation", op)
    return nil
}

func (r *NovaMartServiceReconciler) reconcileNetworkPolicy(
    ctx context.Context,
    svc *platformv1.NovaMartService,
) error {
    np := &networkingv1.NetworkPolicy{
        ObjectMeta: metav1.ObjectMeta{
            Name:      svc.Name + "-default",
            Namespace: svc.Namespace,
        },
    }

    op, err := controllerutil.CreateOrUpdate(ctx, r.Client, np, func() error {
        np.Spec.PodSelector = metav1.LabelSelector{
            MatchLabels: map[string]string{"app": svc.Name},
        }
        np.Spec.PolicyTypes = []networkingv1.PolicyType{
            networkingv1.PolicyTypeIngress,
        }
        np.Spec.Ingress = []networkingv1.NetworkPolicyIngressRule{
            {
                // Allow from same namespace
                From: []networkingv1.NetworkPolicyPeer{
                    {
                        NamespaceSelector: &metav1.LabelSelector{
                            MatchLabels: map[string]string{
                                "kubernetes.io/metadata.name": svc.Namespace,
                            },
                        },
                    },
                },
                Ports: []networkingv1.NetworkPolicyPort{
                    {
                        Port:     &intstr.IntOrString{IntVal: svc.Spec.Port},
                        Protocol: protocolPtr(corev1.ProtocolTCP),
                    },
                },
            },
        }
        return controllerutil.SetControllerReference(svc, np, r.Scheme)
    })

    if err != nil {
        return fmt.Errorf("reconciling network policy: %w", err)
    }

    log.FromContext(ctx).Info("network policy reconciled", "operation", op)
    return nil
}

func (r *NovaMartServiceReconciler) handleDeletion(
    ctx context.Context,
    svc *platformv1.NovaMartService,
) (ctrl.Result, error) {
    if controllerutil.ContainsFinalizer(svc, finalizerName) {
        // Cleanup logic here:
        // - Remove monitoring rules
        // - Archive logs
        // - Notify team
        log.FromContext(ctx).Info("cleaning up NovaMartService", "name", svc.Name)

        // Remove finalizer — allows K8s to complete deletion
        controllerutil.RemoveFinalizer(svc, finalizerName)
        if err := r.Update(ctx, svc); err != nil {
            return ctrl.Result{}, fmt.Errorf("removing finalizer: %w", err)
        }
    }
    return ctrl.Result{}, nil
}

func (r *NovaMartServiceReconciler) setErrorStatus(
    ctx context.Context,
    svc *platformv1.NovaMartService,
    reason string,
    err error,
) (ctrl.Result, error) {
    svc.Status.Phase = "Error"
    svc.Status.Message = err.Error()
    if statusErr := r.Status().Update(ctx, svc); statusErr != nil {
        log.FromContext(ctx).Error(statusErr, "failed to update error status")
    }
    return ctrl.Result{RequeueAfter: 30 * time.Second}, err
}

func tierDefaults(tier string) (cpuReq, memReq, cpuLim, memLim string) {
    switch tier {
    case "critical":
        return "500m", "512Mi", "2", "2Gi"
    case "standard":
        return "250m", "256Mi", "1", "1Gi"
    case "batch":
        return "100m", "128Mi", "500m", "512Mi"
    default:
        return "250m", "256Mi", "1", "1Gi"
    }
}

func protocolPtr(p corev1.Protocol) *corev1.Protocol { return &p }

// SetupWithManager registers the controller with the manager
func (r *NovaMartServiceReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&platformv1.NovaMartService{}).
        Owns(&appsv1.Deployment{}).   // Watch owned Deployments
        Owns(&corev1.Service{}).       // Watch owned Services
        Complete(r)
}
```

**Operator failure modes:**
```go
// FAILURE 1: Infinite reconciliation loop
// Reconcile() updates status → triggers watch → Reconcile() runs again → updates status → ...
// FIX: Only update status if it actually changed:
//   if svc.Status.Phase != "Running" { r.Status().Update(...) }

// FAILURE 2: Owner reference missing → orphaned resources
// If you don't set ControllerReference, deleting the CR leaves child resources behind
// ALWAYS use controllerutil.SetControllerReference()

// FAILURE 3: Finalizer deadlock
// Finalizer cleanup fails → CR stuck in Terminating forever
// FIX: Wrap cleanup in timeout, log and remove finalizer even on failure
// (Better to orphan a resource than have a stuck CR)

// FAILURE 4: No RBAC markers
// Operator deploys but can't create Deployments — RBAC denied
// FIX: +kubebuilder:rbac markers generate ClusterRole. Review generated RBAC.

// FAILURE 5: Reconcile returns error → exponential backoff
// Returning error causes requeue with increasing backoff (up to ~16 min)
// For transient errors: return ctrl.Result{RequeueAfter: 10*time.Second}, nil
// For permanent errors: set Error status, return ctrl.Result{}, nil (stop retrying)
```

---

## 11. TESTING IN GO

```go
// ═══════════════════════════════════════════════════════════
// Go tests are in the SAME package as the code, suffix _test.go
// go test -v -race -coverprofile=coverage.out ./...
// ═══════════════════════════════════════════════════════════

package retry_test

import (
    "context"
    "errors"
    "testing"
    "time"

    "github.com/novamart/novactl/pkg/retry"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// ── Basic test ────────────────────────────────────────────

func TestRetry_Success(t *testing.T) {
    attempts := 0
    err := retry.Do(context.Background(), retry.Config{
        MaxAttempts: 3,
        InitialBackoff: 10 * time.Millisecond,
    }, func() error {
        attempts++
        return nil
    })

    require.NoError(t, err)
    assert.Equal(t, 1, attempts) // Should succeed on first try
}

// ── Table-driven tests (the Go standard) ──────────────────

func TestRetry_TableDriven(t *testing.T) {
    tests := []struct {
        name            string
        failCount       int  // Number of times to fail before success
        maxAttempts     int
        expectError     bool
        expectAttempts  int
    }{
        {
            name:           "succeeds immediately",
            failCount:      0,
            maxAttempts:    3,
            expectError:    false,
            expectAttempts: 1,
        },
        {
            name:           "succeeds after 2 failures",
            failCount:      2,
            maxAttempts:    5,
            expectError:    false,
            expectAttempts: 3,
        },
        {
            name:           "exhausts all retries",
            failCount:      10,
            maxAttempts:    3,
            expectError:    true,
            expectAttempts: 3,
        },
        {
            name:           "single attempt",
            failCount:      1,
            maxAttempts:    1,
            expectError:    true,
            expectAttempts: 1,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            attempts := 0
            err := retry.Do(context.Background(), retry.Config{
                MaxAttempts:    tc.maxAttempts,
                InitialBackoff: 1 * time.Millisecond, // Fast for tests
            }, func() error {
                attempts++
                if attempts <= tc.failCount {
                    return errors.New("transient error")
                }
                return nil
            })

            if tc.expectError {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
            }
            assert.Equal(t, tc.expectAttempts, attempts)
        })
    }
}

// ── Testing with interfaces (dependency injection) ────────

// Mock implementation
type mockCloud struct {
    instances    []Instance
    listErr      error
    stopCalled   []string
    stopErr      error
}

func (m *mockCloud) ListInstances(region string) ([]Instance, error) {
    return m.instances, m.listErr
}

func (m *mockCloud) StopInstance(id string) error {
    m.stopCalled = append(m.stopCalled, id)
    return m.stopErr
}

func (m *mockCloud) TerminateInstance(id string) error {
    return nil
}

func TestStopIdleInstances(t *testing.T) {
    mock := &mockCloud{
        instances: []Instance{
            {ID: "i-111", State: "running", Tags: map[string]string{"env": "dev"}},
            {ID: "i-222", State: "running", Tags: map[string]string{"env": "prod"}},
            {ID: "i-333", State: "stopped", Tags: map[string]string{"env": "dev"}},
        },
    }

    // Function under test uses CloudProvider interface
    stopped, err := StopIdleInstances(context.Background(), mock, "us-east-1", "dev")

    require.NoError(t, err)
    assert.Equal(t, 1, stopped) // Only i-111 (dev + running)
    assert.Contains(t, mock.stopCalled, "i-111")
    assert.NotContains(t, mock.stopCalled, "i-222") // prod — skip
    assert.NotContains(t, mock.stopCalled, "i-333") // already stopped
}

func TestStopIdleInstances_APIError(t *testing.T) {
    mock := &mockCloud{
        listErr: errors.New("API rate limit"),
    }

    _, err := StopIdleInstances(context.Background(), mock, "us-east-1", "dev")
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "API rate limit")
}

// ── Context cancellation test ─────────────────────────────

func TestLongOperation_ContextCancelled(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
    defer cancel()

    err := LongOperation(ctx) // Should respect context
    assert.ErrorIs(t, err, context.DeadlineExceeded)
}

// ── Benchmark ─────────────────────────────────────────────

func BenchmarkParseARN(b *testing.B) {
    arn := "arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0"
    for i := 0; i < b.N; i++ {
        parse_arn(arn)
    }
}


// ═══════════════════════════════════════════════════════════
// TESTING K8S CONTROLLERS (envtest)
// ═══════════════════════════════════════════════════════════

// internal/controller/suite_test.go
package controller_test

import (
    "context"
    "path/filepath"
    "testing"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    "
