# Phase 9, Lesson 3: Go for DevOps

---

## WHY GO FOR DEVOPS — NOT OPINION, FACTS

```
╔══════════════════════════════════════════════════════════════════╗
║  The entire Kubernetes ecosystem is written in Go.               ║
║  kubectl, kubelet, kube-proxy, etcd, containerd, Prometheus,     ║
║  Terraform, Docker, Istio, ArgoCD, Helm, Vault, Consul,          ║
║  CoreDNS, Linkerd2, CRI-O, Buildah, Trivy, Falco...              ║
║                                                                  ║
║  If you can't write Go, you can't extend, debug, or contribute   ║
║  to ANY of these tools. You're a consumer, not an engineer.      ║
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
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/rest"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"

    platformv1 "github.com/novamart/operator/api/v1"
)

var (
    cfg       *rest.Config
    k8sClient client.Client
    testEnv   *envtest.Environment
    ctx       context.Context
    cancel    context.CancelFunc
)

func TestControllers(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))

    ctx, cancel = context.WithCancel(context.Background())

    // envtest starts a REAL etcd + API server (no kubelet, no scheduler)
    // CRD YAML loaded automatically from paths
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
    }

    var err error
    cfg, err = testEnv.Start()
    Expect(err).NotTo(HaveOccurred())
    Expect(cfg).NotTo(BeNil())

    // Register CRD types with scheme
    err = platformv1.AddToScheme(scheme.Scheme)
    Expect(err).NotTo(HaveOccurred())

    // Create client
    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
    Expect(err).NotTo(HaveOccurred())
    Expect(k8sClient).NotTo(BeNil())

    // Start controller manager
    mgr, err := ctrl.NewManager(cfg, ctrl.Options{
        Scheme: scheme.Scheme,
    })
    Expect(err).NotTo(HaveOccurred())

    err = (&controller.NovaMartServiceReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr)
    Expect(err).NotTo(HaveOccurred())

    go func() {
        defer GinkgoRecover()
        err = mgr.Start(ctx)
        Expect(err).NotTo(HaveOccurred())
    }()
})

var _ = AfterSuite(func() {
    cancel()
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})


// ═══════════════════════════════════════════════════════════
// ACTUAL CONTROLLER TESTS
// ═══════════════════════════════════════════════════════════

// internal/controller/novamartservice_controller_test.go
package controller_test

import (
    "time"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"

    platformv1 "github.com/novamart/operator/api/v1"
)

var _ = Describe("NovaMartService Controller", func() {

    const (
        timeout  = 30 * time.Second
        interval = 250 * time.Millisecond
    )

    Context("When creating a NovaMartService", func() {
        It("Should create Deployment and Service", func() {
            // Create namespace
            ns := &corev1.Namespace{
                ObjectMeta: metav1.ObjectMeta{Name: "test-payments"},
            }
            Expect(k8sClient.Create(ctx, ns)).To(Succeed())

            // Create NovaMartService CR
            svc := &platformv1.NovaMartService{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "payment-service",
                    Namespace: "test-payments",
                },
                Spec: platformv1.NovaMartServiceSpec{
                    Team:     "payments",
                    Tier:     "critical",
                    Replicas: 3,
                    Image:    "novamart/payment-service:v1.0.0",
                    Port:     8080,
                },
            }
            Expect(k8sClient.Create(ctx, svc)).To(Succeed())

            // Verify Deployment is created
            depKey := types.NamespacedName{
                Name:      "payment-service",
                Namespace: "test-payments",
            }
            Eventually(func() error {
                var dep appsv1.Deployment
                return k8sClient.Get(ctx, depKey, &dep)
            }, timeout, interval).Should(Succeed())

            // Verify Deployment spec
            var dep appsv1.Deployment
            Expect(k8sClient.Get(ctx, depKey, &dep)).To(Succeed())
            Expect(*dep.Spec.Replicas).To(Equal(int32(3)))
            Expect(dep.Spec.Template.Spec.Containers[0].Image).
                To(Equal("novamart/payment-service:v1.0.0"))
            // Critical tier should get 500m CPU request
            Expect(dep.Spec.Template.Spec.Containers[0].Resources.Requests.Cpu().String()).
                To(Equal("500m"))

            // Verify Service is created
            svcKey := types.NamespacedName{
                Name:      "payment-service",
                Namespace: "test-payments",
            }
            Eventually(func() error {
                var k8sSvc corev1.Service
                return k8sClient.Get(ctx, svcKey, &k8sSvc)
            }, timeout, interval).Should(Succeed())

            // Verify owner reference (for garbage collection)
            var k8sSvc corev1.Service
            Expect(k8sClient.Get(ctx, svcKey, &k8sSvc)).To(Succeed())
            Expect(k8sSvc.OwnerReferences).To(HaveLen(1))
            Expect(k8sSvc.OwnerReferences[0].Name).To(Equal("payment-service"))
        })

        It("Should update status to Running when replicas are ready", func() {
            svcKey := types.NamespacedName{
                Name:      "payment-service",
                Namespace: "test-payments",
            }

            // In envtest, no kubelet to mark pods Ready,
            // so we simulate by updating Deployment status
            depKey := types.NamespacedName{
                Name:      "payment-service",
                Namespace: "test-payments",
            }
            Eventually(func() error {
                var dep appsv1.Deployment
                if err := k8sClient.Get(ctx, depKey, &dep); err != nil {
                    return err
                }
                dep.Status.ReadyReplicas = 3
                dep.Status.Replicas = 3
                dep.Status.AvailableReplicas = 3
                return k8sClient.Status().Update(ctx, &dep)
            }, timeout, interval).Should(Succeed())

            // Check NovaMartService status
            Eventually(func() string {
                var svc platformv1.NovaMartService
                if err := k8sClient.Get(ctx, svcKey, &svc); err != nil {
                    return ""
                }
                return svc.Status.Phase
            }, timeout, interval).Should(Equal("Running"))
        })
    })

    Context("When deleting a NovaMartService", func() {
        It("Should clean up child resources via owner references", func() {
            svcKey := types.NamespacedName{
                Name:      "payment-service",
                Namespace: "test-payments",
            }

            var svc platformv1.NovaMartService
            Expect(k8sClient.Get(ctx, svcKey, &svc)).To(Succeed())
            Expect(k8sClient.Delete(ctx, &svc)).To(Succeed())

            // Deployment should be garbage collected
            Eventually(func() bool {
                var dep appsv1.Deployment
                err := k8sClient.Get(ctx, svcKey, &dep)
                return errors.IsNotFound(err)
            }, timeout, interval).Should(BeTrue())
        })
    })
})
```

---

## 12. AWS SDK FOR GO (aws-sdk-go-v2)

```go
// ═══════════════════════════════════════════════════════════
// aws-sdk-go-v2 is the current AWS SDK for Go
// v1 is in maintenance mode — always use v2 for new code
// ═══════════════════════════════════════════════════════════

package aws

import (
    "context"
    "fmt"
    "time"

    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/credentials/stscreds"
    "github.com/aws/aws-sdk-go-v2/service/ec2"
    ec2types "github.com/aws/aws-sdk-go-v2/service/ec2/types"
    "github.com/aws/aws-sdk-go-v2/service/sts"
    "github.com/rs/zerolog/log"
)

// ── Session/Config setup ──────────────────────────────────

func NewAWSConfig(ctx context.Context, region string, roleARN string) (aws.Config, error) {
    // Load default config (env vars, shared credentials, IMDS)
    cfg, err := config.LoadDefaultConfig(ctx,
        config.WithRegion(region),
        config.WithRetryMaxAttempts(5),
        config.WithRetryMode(aws.RetryModeAdaptive), // Adaptive retry with backoff
    )
    if err != nil {
        return aws.Config{}, fmt.Errorf("loading AWS config: %w", err)
    }

    // Cross-account assume role
    if roleARN != "" {
        stsClient := sts.NewFromConfig(cfg)
        cfg.Credentials = aws.NewCredentialsCache(
            stscreds.NewAssumeRoleProvider(stsClient, roleARN,
                func(o *stscreds.AssumeRoleOptions) {
                    o.RoleSessionName = "novactl"
                    o.Duration = 1 * time.Hour
                },
            ),
        )
        log.Info().Str("role", roleARN).Msg("assuming_cross_account_role")
    }

    return cfg, nil
}


// ── EC2 Operations ────────────────────────────────────────

type EC2Client struct {
    client *ec2.Client
    region string
}

func NewEC2Client(cfg aws.Config) *EC2Client {
    return &EC2Client{
        client: ec2.NewFromConfig(cfg),
        region: cfg.Region,
    }
}

type InstanceInfo struct {
    ID           string            `json:"id"`
    Type         string            `json:"type"`
    State        string            `json:"state"`
    AZ           string            `json:"az"`
    PrivateIP    string            `json:"private_ip"`
    PublicIP     string            `json:"public_ip,omitempty"`
    LaunchTime   time.Time         `json:"launch_time"`
    AgeDays      int               `json:"age_days"`
    Tags         map[string]string `json:"tags"`
    Name         string            `json:"name"`
    Platform     string            `json:"platform"`
    VpcID        string            `json:"vpc_id"`
    SubnetID     string            `json:"subnet_id"`
}

func (c *EC2Client) ListInstances(
    ctx context.Context,
    filters ...ec2types.Filter,
) ([]InstanceInfo, error) {
    var instances []InstanceInfo

    // Paginator handles pagination automatically — NEVER do manual NextToken
    paginator := ec2.NewDescribeInstancesPaginator(c.client,
        &ec2.DescribeInstancesInput{
            Filters: filters,
        },
    )

    for paginator.HasMorePages() {
        page, err := paginator.NextPage(ctx)
        if err != nil {
            return nil, fmt.Errorf("describing instances in %s: %w", c.region, err)
        }

        for _, reservation := range page.Reservations {
            for _, inst := range reservation.Instances {
                info := InstanceInfo{
                    ID:         aws.ToString(inst.InstanceId),
                    Type:       string(inst.InstanceType),
                    State:      string(inst.State.Name),
                    AZ:         aws.ToString(inst.Placement.AvailabilityZone),
                    PrivateIP:  aws.ToString(inst.PrivateIpAddress),
                    PublicIP:   aws.ToString(inst.PublicIpAddress),
                    LaunchTime: aws.ToTime(inst.LaunchTime),
                    AgeDays:    int(time.Since(aws.ToTime(inst.LaunchTime)).Hours() / 24),
                    Tags:       tagsToMap(inst.Tags),
                    Platform:   string(inst.PlatformDetails),
                    VpcID:      aws.ToString(inst.VpcId),
                    SubnetID:   aws.ToString(inst.SubnetId),
                }
                info.Name = info.Tags["Name"]
                instances = append(instances, info)
            }
        }
    }

    log.Info().
        Str("region", c.region).
        Int("count", len(instances)).
        Msg("instances_listed")

    return instances, nil
}

func (c *EC2Client) StopInstances(
    ctx context.Context,
    instanceIDs []string,
    dryRun bool,
) error {
    if len(instanceIDs) == 0 {
        return nil
    }

    log.Info().
        Strs("instances", instanceIDs).
        Bool("dry_run", dryRun).
        Msg("stopping_instances")

    _, err := c.client.StopInstances(ctx, &ec2.StopInstancesInput{
        InstanceIds: instanceIDs,
        DryRun:      aws.Bool(dryRun),
    })
    if err != nil {
        return fmt.Errorf("stopping instances: %w", err)
    }

    if !dryRun {
        // Wait for instances to stop
        waiter := ec2.NewInstanceStoppedWaiter(c.client)
        err = waiter.Wait(ctx, &ec2.DescribeInstancesInput{
            InstanceIds: instanceIDs,
        }, 10*time.Minute)
        if err != nil {
            return fmt.Errorf("waiting for instances to stop: %w", err)
        }
    }

    return nil
}

func (c *EC2Client) FindUntaggedResources(ctx context.Context) ([]InstanceInfo, error) {
    // Find instances missing required tags
    requiredTags := []string{"team", "environment", "service"}

    allInstances, err := c.ListInstances(ctx,
        ec2types.Filter{
            Name:   aws.String("instance-state-name"),
            Values: []string{"running"},
        },
    )
    if err != nil {
        return nil, err
    }

    var untagged []InstanceInfo
    for _, inst := range allInstances {
        for _, tag := range requiredTags {
            if _, ok := inst.Tags[tag]; !ok {
                untagged = append(untagged, inst)
                break
            }
        }
    }

    return untagged, nil
}

// ── Multi-region operations ───────────────────────────────

func ListInstancesAllRegions(
    ctx context.Context,
    regions []string,
    roleARN string,
) ([]InstanceInfo, error) {
    type regionResult struct {
        region    string
        instances []InstanceInfo
        err       error
    }

    resultCh := make(chan regionResult, len(regions))

    for _, region := range regions {
        go func(r string) {
            cfg, err := NewAWSConfig(ctx, r, roleARN)
            if err != nil {
                resultCh <- regionResult{region: r, err: err}
                return
            }

            client := NewEC2Client(cfg)
            instances, err := client.ListInstances(ctx,
                ec2types.Filter{
                    Name:   aws.String("instance-state-name"),
                    Values: []string{"running"},
                },
            )
            resultCh <- regionResult{region: r, instances: instances, err: err}
        }(region)
    }

    var allInstances []InstanceInfo
    var errs []error

    for range regions {
        result := <-resultCh
        if result.err != nil {
            log.Error().Err(result.err).Str("region", result.region).Msg("region_scan_failed")
            errs = append(errs, result.err)
            continue
        }
        allInstances = append(allInstances, result.instances...)
    }

    if len(errs) > 0 && len(allInstances) == 0 {
        return nil, fmt.Errorf("all regions failed: %v", errs[0])
    }

    return allInstances, nil
}

func tagsToMap(tags []ec2types.Tag) map[string]string {
    m := make(map[string]string, len(tags))
    for _, t := range tags {
        m[aws.ToString(t.Key)] = aws.ToString(t.Value)
    }
    return m
}
```

---

## 13. OUTPUT FORMATTING

```go
// ═══════════════════════════════════════════════════════════
// All NovaMart CLIs support --output table|json|yaml|csv
// stdout = data (pipeable), stderr = human messages
// ═══════════════════════════════════════════════════════════

package output

import (
    "encoding/csv"
    "encoding/json"
    "fmt"
    "io"
    "os"

    "github.com/olekukonez/tablewriter"
    "gopkg.in/yaml.v3"
)

// Render outputs data in the requested format
func Render(format string, data interface{}) error {
    switch format {
    case "json":
        return renderJSON(os.Stdout, data)
    case "yaml":
        return renderYAML(os.Stdout, data)
    case "csv":
        return renderCSV(os.Stdout, data)
    case "table":
        return renderTable(os.Stdout, data)
    default:
        return fmt.Errorf("unsupported format: %s", format)
    }
}

func renderJSON(w io.Writer, data interface{}) error {
    encoder := json.NewEncoder(w)
    encoder.SetIndent("", "  ")
    return encoder.Encode(data)
}

func renderYAML(w io.Writer, data interface{}) error {
    encoder := yaml.NewEncoder(w)
    encoder.SetIndent(2)
    return encoder.Encode(data)
}

// Table rendering — type-specific implementations
func renderTable(w io.Writer, data interface{}) error {
    switch v := data.(type) {
    case []PodInfo:
        return renderPodTable(w, v)
    case []NodeInfo:
        return renderNodeTable(w, v)
    case []AuditFinding:
        return renderAuditTable(w, v)
    case []InstanceInfo:
        return renderInstanceTable(w, v)
    default:
        // Fallback to JSON for unknown types
        return renderJSON(w, data)
    }
}

func renderPodTable(w io.Writer, pods []PodInfo) error {
    table := tablewriter.NewWriter(w)
    table.SetHeader([]string{"NAMESPACE", "NAME", "READY", "STATUS", "RESTARTS", "AGE", "NODE"})
    table.SetBorder(false)
    table.SetColumnSeparator("")
    table.SetHeaderAlignment(tablewriter.ALIGN_LEFT)
    table.SetAlignment(tablewriter.ALIGN_LEFT)

    for _, pod := range pods {
        table.Append([]string{
            pod.Namespace,
            pod.Name,
            pod.Ready,
            pod.Status,
            fmt.Sprintf("%d", pod.Restarts),
            pod.Age,
            pod.Node,
        })
    }

    table.Render()
    return nil
}

func renderAuditTable(w io.Writer, findings []AuditFinding) error {
    table := tablewriter.NewWriter(w)
    table.SetHeader([]string{"SEVERITY", "CATEGORY", "RESOURCE", "NAMESPACE", "NAME", "MESSAGE"})
    table.SetBorder(false)
    table.SetAutoWrapText(false)

    severityColors := map[string]int{
        "CRITICAL": tablewriter.FgRedColor,
        "HIGH":     tablewriter.FgHiRedColor,
        "MEDIUM":   tablewriter.FgYellowColor,
        "LOW":      tablewriter.FgBlueColor,
    }

    for _, f := range findings {
        row := []string{
            f.Severity,
            f.Category,
            f.Resource,
            f.Namespace,
            f.Name,
            f.Message,
        }

        colors := make([]tablewriter.Colors, len(row))
        if color, ok := severityColors[f.Severity]; ok {
            colors[0] = tablewriter.Colors{tablewriter.Bold, color}
        }
        table.Rich(row, colors)
    }

    table.Render()

    // Summary
    critCount := 0
    highCount := 0
    for _, f := range findings {
        switch f.Severity {
        case "CRITICAL":
            critCount++
        case "HIGH":
            highCount++
        }
    }
    fmt.Fprintf(w, "\nTotal: %d findings (%d critical, %d high)\n",
        len(findings), critCount, highCount)

    return nil
}

func renderInstanceTable(w io.Writer, instances []InstanceInfo) error {
    table := tablewriter.NewWriter(w)
    table.SetHeader([]string{"ID", "NAME", "TYPE", "STATE", "AZ", "PRIVATE IP", "AGE"})
    table.SetBorder(false)

    for _, inst := range instances {
        table.Append([]string{
            inst.ID,
            inst.Name,
            inst.Type,
            inst.State,
            inst.AZ,
            inst.PrivateIP,
            fmt.Sprintf("%dd", inst.AgeDays),
        })
    }

    table.Render()
    return nil
}
```

---

## 14. RETRY PACKAGE (REUSABLE)

```go
// pkg/retry/retry.go — production retry with backoff + jitter
package retry

import (
    "context"
    "fmt"
    "math"
    "math/rand"
    "time"

    "github.com/rs/zerolog/log"
)

type Config struct {
    MaxAttempts    int
    InitialBackoff time.Duration
    MaxBackoff     time.Duration
    Multiplier     float64
    Jitter         bool
    RetryableFunc  func(error) bool // Optional: decide if error is retryable
}

func DefaultConfig() Config {
    return Config{
        MaxAttempts:    3,
        InitialBackoff: 1 * time.Second,
        MaxBackoff:     30 * time.Second,
        Multiplier:     2.0,
        Jitter:         true,
    }
}

func Do(ctx context.Context, cfg Config, fn func() error) error {
    if cfg.MaxAttempts <= 0 {
        cfg.MaxAttempts = 3
    }
    if cfg.InitialBackoff == 0 {
        cfg.InitialBackoff = 1 * time.Second
    }
    if cfg.MaxBackoff == 0 {
        cfg.MaxBackoff = 30 * time.Second
    }
    if cfg.Multiplier == 0 {
        cfg.Multiplier = 2.0
    }

    var lastErr error
    for attempt := 1; attempt <= cfg.MaxAttempts; attempt++ {
        lastErr = fn()
        if lastErr == nil {
            return nil
        }

        // Check if retryable
        if cfg.RetryableFunc != nil && !cfg.RetryableFunc(lastErr) {
            return fmt.Errorf("non-retryable error: %w", lastErr)
        }

        // Don't sleep after last attempt
        if attempt == cfg.MaxAttempts {
            break
        }

        // Calculate backoff
        backoff := time.Duration(float64(cfg.InitialBackoff) *
            math.Pow(cfg.Multiplier, float64(attempt-1)))
        if backoff > cfg.MaxBackoff {
            backoff = cfg.MaxBackoff
        }

        // Add jitter (±25%)
        if cfg.Jitter {
            jitter := time.Duration(rand.Int63n(int64(backoff) / 2))
            backoff = backoff/2 + jitter
        }

        log.Warn().
            Err(lastErr).
            Int("attempt", attempt).
            Int("max_attempts", cfg.MaxAttempts).
            Dur("backoff", backoff).
            Msg("retrying")

        select {
        case <-ctx.Done():
            return fmt.Errorf("context cancelled during retry: %w", ctx.Err())
        case <-time.After(backoff):
        }
    }

    return fmt.Errorf("exhausted %d attempts: %w", cfg.MaxAttempts, lastErr)
}
```

---

## 15. GORELEASER — PROFESSIONAL DISTRIBUTION

```yaml
# .goreleaser.yml — build and release for all platforms
project_name: novactl

before:
  hooks:
    - go mod tidy
    - go generate ./...
    - golangci-lint run

builds:
  - id: novactl
    main: ./cmd/novactl/
    binary: novactl
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w
      - -X github.com/novamart/novactl/internal/version.Version={{.Version}}
      - -X github.com/novamart/novactl/internal/version.Commit={{.ShortCommit}}
      - -X github.com/novamart/novactl/internal/version.Date={{.Date}}

archives:
  - id: default
    format: tar.gz
    format_overrides:
      - goos: windows
        format: zip
    name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"

checksum:
  name_template: "checksums.txt"
  algorithm: sha256

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
      - "^chore:"

dockers:
  - image_templates:
      - "ghcr.io/novamart/novactl:{{ .Version }}-amd64"
    use: buildx
    build_flag_templates:
      - "--platform=linux/amd64"
    dockerfile: Dockerfile.goreleaser
  - image_templates:
      - "ghcr.io/novamart/novactl:{{ .Version }}-arm64"
    use: buildx
    build_flag_templates:
      - "--platform=linux/arm64"
    goarch: arm64
    dockerfile: Dockerfile.goreleaser

docker_manifests:
  - name_template: "ghcr.io/novamart/novactl:{{ .Version }}"
    image_templates:
      - "ghcr.io/novamart/novactl:{{ .Version }}-amd64"
      - "ghcr.io/novamart/novactl:{{ .Version }}-arm64"
  - name_template: "ghcr.io/novamart/novactl:latest"
    image_templates:
      - "ghcr.io/novamart/novactl:{{ .Version }}-amd64"
      - "ghcr.io/novamart/novactl:{{ .Version }}-arm64"

brews:
  - name: novactl
    repository:
      owner: novamart
      name: homebrew-tap
    homepage: "https://github.com/novamart/novactl"
    description: "NovaMart Platform CLI"
    install: |
      bin.install "novactl"
      # Shell completions
      bash_completion.install "completions/novactl.bash" => "novactl"
      zsh_completion.install "completions/novactl.zsh" => "_novactl"
    test: |
      system "#{bin}/novactl", "version"
```

---

## 16. ADMISSION WEBHOOKS

```go
// ═══════════════════════════════════════════════════════════
// Admission webhooks intercept K8s API requests BEFORE persistence.
// Validating: reject bad requests. Mutating: modify requests.
// NovaMart: enforce labels, resource limits, image registry.
// ═══════════════════════════════════════════════════════════

package webhook

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "strings"

    "github.com/rs/zerolog/log"
    admissionv1 "k8s.io/api/admission/v1"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer"
)

var (
    scheme = runtime.NewScheme()
    codecs = serializer.NewCodecFactory(scheme)
)

func init() {
    _ = admissionv1.AddToScheme(scheme)
    _ = appsv1.AddToScheme(scheme)
    _ = corev1.AddToScheme(scheme)
}

const (
    allowedRegistry = "123456789012.dkr.ecr.us-east-1.amazonaws.com"
)

// ValidateDeployment validates Deployment create/update requests
func ValidateDeployment(w http.ResponseWriter, r *http.Request) {
    review, err := parseAdmissionReview(r)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    var dep appsv1.Deployment
    if err := json.Unmarshal(review.Request.Object.Raw, &dep); err != nil {
        sendResponse(w, review, false, fmt.Sprintf("failed to parse deployment: %v", err))
        return
    }

    logger := log.With().
        Str("name", dep.Name).
        Str("namespace", dep.Namespace).
        Str("operation", string(review.Request.Operation)).
        Logger()

    var violations []string

    // Rule 1: Required labels
    requiredLabels := []string{"app", "team", "version"}
    for _, label := range requiredLabels {
        if _, ok := dep.Labels[label]; !ok {
            violations = append(violations,
                fmt.Sprintf("missing required label: %s", label))
        }
    }

    // Rule 2: All containers must have resource limits
    for _, container := range dep.Spec.Template.Spec.Containers {
        if container.Resources.Limits == nil {
            violations = append(violations,
                fmt.Sprintf("container '%s' missing resource limits", container.Name))
        } else {
            if container.Resources.Limits.Cpu().IsZero() {
                violations = append(violations,
                    fmt.Sprintf("container '%s' missing CPU limit", container.Name))
            }
            if container.Resources.Limits.Memory().IsZero() {
                violations = append(violations,
                    fmt.Sprintf("container '%s' missing memory limit", container.Name))
            }
        }
    }

    // Rule 3: Images must come from approved registry
    for _, container := range dep.Spec.Template.Spec.Containers {
        if !strings.HasPrefix(container.Image, allowedRegistry) {
            violations = append(violations,
                fmt.Sprintf("container '%s' uses unapproved registry: %s (must use %s)",
                    container.Name, container.Image, allowedRegistry))
        }
    }

    // Rule 4: No :latest tag
    for _, container := range dep.Spec.Template.Spec.Containers {
        if strings.HasSuffix(container.Image, ":latest") || !strings.Contains(container.Image, ":") {
            violations = append(violations,
                fmt.Sprintf("container '%s': :latest or untagged images not allowed", container.Name))
        }
    }

    // Rule 5: Readiness probe required
    for _, container := range dep.Spec.Template.Spec.Containers {
        if container.ReadinessProbe == nil {
            violations = append(violations,
                fmt.Sprintf("container '%s' missing readinessProbe", container.Name))
        }
    }

    if len(violations) > 0 {
        msg := "Deployment violates NovaMart policies:\n- " + strings.Join(violations, "\n- ")
        logger.Warn().Strs("violations", violations).Msg("deployment_rejected")
        sendResponse(w, review, false, msg)
        return
    }

    logger.Info().Msg("deployment_approved")
    sendResponse(w, review, true, "")
}

// MutateDeployment injects defaults into Deployments
func MutateDeployment(w http.ResponseWriter, r *http.Request) {
    review, err := parseAdmissionReview(r)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    var dep appsv1.Deployment
    if err := json.Unmarshal(review.Request.Object.Raw, &dep); err != nil {
        sendResponse(w, review, false, fmt.Sprintf("failed to parse: %v", err))
        return
    }

    var patches []map[string]interface{}

    // Inject standard labels if missing
    if dep.Labels == nil {
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/metadata/labels",
            "value": map[string]string{},
        })
    }
    if _, ok := dep.Labels["managed-by"]; !ok {
        patches = append(patches, map[string]interface{}{
            "op":    "add",
            "path":  "/metadata/labels/managed-by",
            "value": "novamart-platform",
        })
    }

    // Inject pod anti-affinity for HA (spread across AZs)
    if dep.Spec.Replicas != nil && *dep.Spec.Replicas >= 2 {
        if dep.Spec.Template.Spec.Affinity == nil {
            affinity := map[string]interface{}{
                "podAntiAffinity": map[string]interface{}{
                    "preferredDuringSchedulingIgnoredDuringExecution": []map[string]interface{}{
                        {
                            "weight": 100,
                            "podAffinityTerm": map[string]interface{}{
                                "topologyKey": "topology.kubernetes.io/zone",
                                "labelSelector": map[string]interface{}{
                                    "matchLabels": dep.Spec.Selector.MatchLabels,
                                },
                            },
                        },
                    },
                },
            }
            patches = append(patches, map[string]interface{}{
                "op":    "add",
                "path":  "/spec/template/spec/affinity",
                "value": affinity,
            })
        }
    }

    patchBytes, _ := json.Marshal(patches)
    patchType := admissionv1.PatchTypeJSONPatch

    response := &admissionv1.AdmissionResponse{
        UID:       review.Request.UID,
        Allowed:   true,
        PatchType: &patchType,
        Patch:     patchBytes,
    }

    sendAdmissionResponse(w, review, response)
}

func parseAdmissionReview(r *http.Request) (*admissionv1.AdmissionReview, error) {
    var review admissionv1.AdmissionReview
    if err := json.NewDecoder(r.Body).Decode(&review); err != nil {
        return nil, fmt.Errorf("decoding admission review: %w", err)
    }
    return &review, nil
}

func sendResponse(w http.ResponseWriter, review *admissionv1.AdmissionReview, allowed bool, message string) {
    response := &admissionv1.AdmissionResponse{
        UID:     review.Request.UID,
        Allowed: allowed,
    }
    if !allowed {
        response.Result = &metav1.Status{
            Message: message,
            Code:    403,
        }
    }
    sendAdmissionResponse(w, review, response)
}

func sendAdmissionResponse(w http.ResponseWriter, review *admissionv1.AdmissionReview, response *admissionv1.AdmissionResponse) {
    review.Response = response
    review.Response.UID = review.Request.UID

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(review)
}
```

---

## 17. GO FAILURE MODES — COMPLETE LIST

```go
// ═══════════════════════════════════════════════════════════
// Every Go failure mode a Senior DevOps engineer hits
// ═══════════════════════════════════════════════════════════

// FAILURE 1: Goroutine leak (covered above)
// Missing context cancellation, unbuffered channel nobody reads
// Detection: runtime.NumGoroutine() grows, pprof goroutine profile
// FIX: Always use context, always close channels from sender side

// FAILURE 2: Nil pointer dereference
// var dep *appsv1.Deployment  // nil
// dep.Name  // PANIC
// Also: interface nil trap:
var err error = (*MyError)(nil)  // err != nil is TRUE even though value is nil
// Interface is (type=*MyError, value=nil) — not nil interface
// FIX: Return plain nil, not typed nil: return nil, not return (*MyError)(nil)

// FAILURE 3: Slice gotcha — shared backing array
original := []string{"a", "b", "c", "d"}
slice := original[:2]   // ["a", "b"] — shares backing array with original
slice = append(slice, "X")  // Overwrites original[2]! original is now ["a", "b", "X", "d"]
// FIX: Use full slice expression: slice := original[:2:2] (sets capacity)
// Or: copy to new slice

// FAILURE 4: Map concurrent access panic
// Maps are NOT goroutine-safe. Concurrent read+write = panic (not just wrong data)
// FIX: sync.RWMutex or sync.Map

// FAILURE 5: defer in loop
func processFiles(files []string) {
    for _, f := range files {
        fd, _ := os.Open(f)
        defer fd.Close()  // ALL defers run at FUNCTION exit, not loop iteration
        // 1000 files = 1000 open file descriptors until function returns
    }
}
// FIX: Use closure:
func processFiles(files []string) {
    for _, f := range files {
        func() {
            fd, _ := os.Open(f)
            defer fd.Close()  // Closes at closure exit (each iteration)
        }()
    }
}

// FAILURE 6: HTTP response body not closed
resp, err := http.Get(url)
if err != nil { return err }
// Missing: defer resp.Body.Close()
// Result: connection leak, eventually connection pool exhaustion
// FIX: ALWAYS close body, even if you don't read it
defer resp.Body.Close()
io.Copy(io.Discard, resp.Body)  // Drain to allow connection reuse

// FAILURE 7: http.Client without timeout (covered above)
// Default client blocks forever. Always set Timeout.

// FAILURE 8: JSON unmarshalling into wrong type
// Go silently ignores unknown fields (won't error)
// Go silently uses zero values for missing fields
// FIX: Use json.Decoder with DisallowUnknownFields() when strict parsing needed
decoder := json.NewDecoder(reader)
decoder.DisallowUnknownFields()

// FAILURE 9: String iteration yields runes, not bytes
s := "héllo"
for i, c := range s {
    // i = byte offset (not index), c = rune (not byte)
    // i: 0, 1, 3, 4, 5 (index 2 is second byte of é)
}
// If you need byte-by-byte: for i := 0; i < len(s); i++ { s[i] }
// If you need runes: for _, r := range s { }

// FAILURE 10: time.After leak in select
for {
    select {
    case msg := <-ch:
        process(msg)
    case <-time.After(5 * time.Second):  // Creates NEW timer every iteration
        // Leaked timers pile up until GC catches them
    }
}
// FIX: Use time.NewTimer and Reset()
timer := time.NewTimer(5 * time.Second)
defer timer.Stop()
for {
    select {
    case msg := <-ch:
        process(msg)
        if !timer.Stop() {
            <-timer.C
        }
        timer.Reset(5 * time.Second)
    case <-timer.C:
        handleTimeout()
        timer.Reset(5 * time.Second)
    }
}

// FAILURE 11: init() functions hidden dependencies
// init() runs at import time — can't control order, can't test, can't inject
// FIX: Use explicit initialization functions called from main()

// FAILURE 12: Error wrapping breaks errors.Is()
// BAD:
return fmt.Errorf("failed: %v", err)  // %v = err is NOT wrapped, Is/As won't work
// GOOD:
return fmt.Errorf("failed: %w", err)  // %w = err IS wrapped, Is/As traverses chain
```


## QUICK REFERENCE CARD

```
╔═══════════════════════════════════════════════════════════════╗
║              GO FOR DEVOPS — CHEAT SHEET                      ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║ PROJECT:                                                      ║
║   cmd/ → internal/ → pkg/                                     ║
║   go mod init, go mod tidy                                    ║
║   CGO_ENABLED=0 go build -ldflags="-s -w" ./cmd/novactl/     ║
║   GOOS=linux GOARCH=arm64 go build  (cross-compile)          ║
║                                                               ║
║ CLI (Cobra + Viper):                                          ║
║   rootCmd.PersistentFlags() — global flags                    ║
║   viper.BindPFlag() — unify flags/env/config                  ║
║   viper.SetEnvPrefix("NOVACTL")                               ║
║   cobra.ExactArgs(1) — argument validation                    ║
║                                                               ║
║ ERRORS:                                                       ║
║   return fmt.Errorf("doing X: %w", err)  — ALWAYS wrap       ║
║   errors.Is(err, ErrNotFound)  — sentinel check               ║
║   errors.As(err, &awsErr)  — typed check                      ║
║   NEVER: err.Error() == "string"                              ║
║                                                               ║
║ CONCURRENCY:                                                  ║
║   errgroup.SetLimit(N) — bounded parallelism (90% use case)  ║
║   context.WithTimeout() — ALWAYS for external calls           ║
║   defer cancel() — ALWAYS                                     ║
║   go test -race ./... — ALWAYS in CI                          ║
║                                                               ║
║ HTTP CLIENT:                                                  ║
║   ALWAYS set Timeout on http.Client (default = forever)       ║
║   Reuse client (don't create per-request)                     ║
║   defer resp.Body.Close() + io.Copy(io.Discard, resp.Body)   ║
║                                                               ║
║ K8S (client-go):                                              ║
║   rest.InClusterConfig() || clientcmd.BuildConfigFromFlags()  ║
║   config.QPS = 50, config.Burst = 100                         ║
║   Informers for watch, Listers for cached reads               ║
║   WaitForCacheSync() BEFORE reading from cache                ║
║   controllerutil.CreateOrUpdate() for idempotent reconcile    ║
║   controllerutil.SetControllerReference() for GC              ║
║                                                               ║
║ AWS (v2):                                                     ║
║   config.LoadDefaultConfig(ctx, WithRegion, WithRetryMode)    ║
║   Paginator for List operations (NEVER manual NextToken)      ║
║   Waiter for state transitions                                ║
║   aws.ToString() / aws.ToTime() for pointer derefs            ║
║                                                               ║
║ TESTING:                                                      ║
║   Table-driven tests (standard Go pattern)                    ║
║   Interface mocking (no framework needed)                     ║
║   go test -race -coverprofile=coverage.out ./...              ║
║   envtest for controller tests (real etcd + API server)       ║
║                                                               ║
║ GOTCHAS:                                                      ║
║   ✗ default http.Client      → set Timeout                    ║
║   ✗ defer in loop            → wrap in closure                ║
║   ✗ shared slice backing     → full slice expr [:n:n]         ║
║   ✗ concurrent map access    → sync.RWMutex or sync.Map       ║
║   ✗ time.After in loop       → time.NewTimer + Reset          ║
║   ✗ resp.Body not closed     → defer Close + Drain            ║
║   ✗ fmt.Errorf("%v", err)    → use %w for wrapping            ║
║   ✗ typed nil interface trap → return plain nil                ║
║   ✗ init() hidden deps       → explicit init functions        ║
║                                                               ║
║ DISTRIBUTION:                                                 ║
║   GoReleaser → multi-arch binaries + Docker + Homebrew        ║
║   Distroless container (gcr.io/distroless/static-debian12)    ║
║   ldflags -s -w (strip debug, ~30% smaller binary)            ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```


---

## RETENTION QUESTIONS

**Q1 (Architecture — Design a Go CLI Tool):**

Design `novactl cost report` — a CLI command that:
1. Queries AWS Cost Explorer across 3 regions for the last 30 days
2. Queries Kubecost API for per-namespace K8s cost attribution
3. Correlates AWS costs with K8s namespaces using tags (team, service)
4. Outputs: cost-by-team table, cost-by-service table, untagged spend, daily trend, anomalies (>20% above 7-day avg)
5. Sends Slack summary with top cost drivers

Show: file layout (which packages), Cobra command definition, the concurrency model for multi-region + Kubecost queries, error handling strategy (partial failures OK), the data model (structs), output format routing, and how you'd test the cost correlation logic without hitting AWS.

---

**Q2 (Debugging — Find the Bugs):**

Your teammate's Go code runs a nightly cleanup of old ECR images. It worked for 6 months, then started silently skipping images. No errors in logs. Find every bug:

```go
func cleanupECRImages(ctx context.Context, repos []string, maxAge int) {
    client := ecr.NewFromConfig(getAWSConfig())
    
    for _, repo := range repos {
        images, _ := client.DescribeImages(ctx, &ecr.DescribeImagesInput{
            RepositoryName: &repo,
        })
        
        for _, img := range images.ImageDetails {
            age := time.Now().Sub(*img.ImagePushedAt)
            if age.Hours() / 24 > float64(maxAge) {
                if img.ImageTags != nil && len(img.ImageTags) > 0 {
                    log.Printf("Deleting %s:%s (age: %d days)", 
                        repo, img.ImageTags[0], int(age.Hours()/24))
                }
                
                client.BatchDeleteImage(ctx, &ecr.BatchDeleteImageInput{
                    RepositoryName: &repo,
                    ImageIds: []types.ImageIdentifier{
                        {ImageDigest: img.ImageDigest},
                    },
                })
            }
        }
    }
}
```

---

**Q3 (Production Code — Write It):**

Write a complete, tested Go function that implements a K8s deployment validator pre-check. Before a deployment rolls out, it should:
1. Check the target image exists in ECR (don't deploy if image is missing)
2. Check namespace resource quota has room for the new pods
3. Check no PDB would block the rollout (all pods disrupted)
4. Check the previous rollout is complete (no in-progress rollouts)
5. Return a structured result with pass/fail per check and overall go/no-go

Include: struct definitions, the function, and a table-driven test with mocked K8s and ECR clients.

---

**Q4 (Concurrency — Trace the Execution):**

```go
func main() {
    ch := make(chan int, 2)
    done := make(chan bool)

    go func() {
        for v := range ch {
            fmt.Println("received:", v)
            time.Sleep(100 * time.Millisecond)
        }
        done <- true
    }()

    for i := 0; i < 5; i++ {
        ch <- i
        fmt.Println("sent:", i)
    }
    close(ch)
    <-done
}
```

Trace the exact execution order. Which sends block? When? Why is the buffer size 2 relevant? What happens if you change it to `make(chan int)` (unbuffered)? What if the consumer goroutine panics on `i == 3` — what happens to the main goroutine?



# Q1 — Architecture: `novactl cost report`

## File Layout

```
novactl/
├── cmd/
│   ├── root.go                       # Cobra root command
│   └── cost/
│       ├── cost.go                   # `novactl cost` subcommand group
│       └── report.go                 # `novactl cost report` command definition
├── internal/
│   ├── aws/
│   │   ├── costexplorer/
│   │   │   ├── client.go            # Interface + real implementation
│   │   │   ├── client_test.go
│   │   │   └── types.go             # AWS-specific response mapping
│   │   └── session.go               # Shared AWS config factory
│   ├── kubecost/
│   │   ├── client.go                # Interface + HTTP client for Kubecost API
│   │   ├── client_test.go
│   │   └── types.go                 # Kubecost response structs
│   ├── cost/
│   │   ├── correlator.go            # Core: joins AWS costs ↔ K8s namespaces via tags
│   │   ├── correlator_test.go       # Table-driven tests with fakes
│   │   ├── anomaly.go               # Anomaly detection (7-day avg comparison)
│   │   ├── anomaly_test.go
│   │   ├── models.go                # Domain structs: CostReport, TeamCost, etc.
│   │   └── aggregator.go            # Aggregation by team, service, day
│   ├── output/
│   │   ├── table.go                 # Rich terminal table (tablewriter)
│   │   ├── json.go                  # JSON output
│   │   ├── csv.go                   # CSV output
│   │   └── formatter.go             # Interface + factory
│   ├── notify/
│   │   ├── slack.go                 # Slack webhook sender
│   │   └── slack_test.go
│   └── concurrency/
│       └── fanout.go                # Generic fan-out/fan-in helper
├── pkg/
│   └── testutil/
│       ├── fakes.go                 # Fake CostExplorer + Kubecost clients
│       └── fixtures.go              # Test data builders
├── go.mod
├── go.sum
└── main.go
```

## Data Model (`internal/cost/models.go`)

```go
package cost

import "time"

// ── Domain Models ───────────────────────────────────────────────────────────

// CostRecord is the normalized unit from any source (AWS or Kubecost).
type CostRecord struct {
    Date        time.Time         `json:"date"`
    Service     string            `json:"service"`      // e.g., "order-service"
    Team        string            `json:"team"`         // e.g., "payments-team"
    Namespace   string            `json:"namespace"`    // K8s namespace
    AWSService  string            `json:"aws_service"`  // e.g., "AmazonEC2", "AmazonRDS"
    Region      string            `json:"region"`
    Amount      float64           `json:"amount"`
    Currency    string            `json:"currency"`
    Tags        map[string]string `json:"tags"`
    Source      CostSource        `json:"source"`       // AWS or Kubecost
}

type CostSource string

const (
    SourceAWS      CostSource = "aws_cost_explorer"
    SourceKubecost CostSource = "kubecost"
)

// CorrelatedCost represents a single cost line item after AWS↔K8s join.
type CorrelatedCost struct {
    CostRecord
    K8sCPUCost    float64 `json:"k8s_cpu_cost"`
    K8sMemoryCost float64 `json:"k8s_memory_cost"`
    K8sGPUCost    float64 `json:"k8s_gpu_cost"`
    Correlated    bool    `json:"correlated"` // false = untagged/unmatched
}

// TeamCostSummary aggregates costs for a single team.
type TeamCostSummary struct {
    Team          string  `json:"team"`
    TotalCost     float64 `json:"total_cost"`
    AWSCost       float64 `json:"aws_cost"`
    K8sCost       float64 `json:"k8s_cost"`
    ServiceCount  int     `json:"service_count"`
    TopService    string  `json:"top_service"`
    TopServiceAmt float64 `json:"top_service_amount"`
}

// ServiceCostSummary aggregates costs for a single service.
type ServiceCostSummary struct {
    Service      string  `json:"service"`
    Team         string  `json:"team"`
    Namespace    string  `json:"namespace"`
    TotalCost    float64 `json:"total_cost"`
    DailyAvg     float64 `json:"daily_avg"`
    SevenDayAvg  float64 `json:"seven_day_avg"`
    IsAnomaly    bool    `json:"is_anomaly"`
    AnomalyPct   float64 `json:"anomaly_pct"` // % above 7-day avg
}

// DailyTrend represents one day's aggregate cost.
type DailyTrend struct {
    Date      time.Time `json:"date"`
    TotalCost float64   `json:"total_cost"`
    AWSCost   float64   `json:"aws_cost"`
    K8sCost   float64   `json:"k8s_cost"`
}

// Anomaly flags a service whose daily cost deviates from its 7-day average.
type Anomaly struct {
    Service     string    `json:"service"`
    Team        string    `json:"team"`
    Date        time.Time `json:"date"`
    DailyCost   float64   `json:"daily_cost"`
    SevenDayAvg float64   `json:"seven_day_avg"`
    DeviationPct float64  `json:"deviation_pct"`
}

// CostReport is the top-level output of the entire pipeline.
type CostReport struct {
    GeneratedAt    time.Time            `json:"generated_at"`
    PeriodStart    time.Time            `json:"period_start"`
    PeriodEnd      time.Time            `json:"period_end"`
    Regions        []string             `json:"regions"`
    TotalCost      float64              `json:"total_cost"`
    UntaggedCost   float64              `json:"untagged_cost"`
    UntaggedPct    float64              `json:"untagged_pct"`
    ByTeam         []TeamCostSummary    `json:"by_team"`
    ByService      []ServiceCostSummary `json:"by_service"`
    DailyTrend     []DailyTrend         `json:"daily_trend"`
    Anomalies      []Anomaly            `json:"anomalies"`
    Errors         []string             `json:"errors,omitempty"`
}
```

## Client Interfaces (for testability)

```go
// internal/aws/costexplorer/client.go
package costexplorer

import "context"

type CostExplorerClient interface {
    GetCostByTags(ctx context.Context, params GetCostInput) ([]CostRecord, error)
}

type GetCostInput struct {
    Region    string
    StartDate time.Time
    EndDate   time.Time
    TagKeys   []string // ["team", "service"]
}
```

```go
// internal/kubecost/client.go
package kubecost

import "context"

type KubecostClient interface {
    GetNamespaceCosts(ctx context.Context, window string) ([]NamespaceCost, error)
}

type NamespaceCost struct {
    Namespace   string            `json:"namespace"`
    CPUCost     float64           `json:"cpuCost"`
    MemoryCost  float64           `json:"memoryCost"`
    GPUCost     float64           `json:"gpuCost"`
    TotalCost   float64           `json:"totalCost"`
    Labels      map[string]string `json:"labels"`
}
```

## Concurrency Model (`internal/concurrency/fanout.go`)

```go
package concurrency

import (
    "context"
    "fmt"
    "sync"
)

// Result wraps a typed result with its source label and potential error.
type Result[T any] struct {
    Source string
    Value  T
    Err    error
}

// FanOut executes functions concurrently, collects all results.
// Partial failures are captured in Result.Err — never short-circuits.
func FanOut[T any](ctx context.Context, tasks map[string]func(context.Context) (T, error)) []Result[T] {
    results := make([]Result[T], 0, len(tasks))
    var mu sync.Mutex
    var wg sync.WaitGroup

    for name, fn := range tasks {
        wg.Add(1)
        go func(name string, fn func(context.Context) (T, error)) {
            defer wg.Done()
            defer func() {
                if r := recover(); r != nil {
                    mu.Lock()
                    results = append(results, Result[T]{
                        Source: name,
                        Err:    fmt.Errorf("panic in %s: %v", name, r),
                    })
                    mu.Unlock()
                }
            }()

            val, err := fn(ctx)
            mu.Lock()
            results = append(results, Result[T]{Source: name, Value: val, Err: err})
            mu.Unlock()
        }(name, fn)
    }

    wg.Wait()
    return results
}
```

**Usage in the report pipeline:**

```go
// internal/cost/correlator.go — orchestration

func BuildReport(
    ctx context.Context,
    ceClient costexplorer.CostExplorerClient,
    kcClient kubecost.KubecostClient,
    regions []string,
    period time.Duration,
) (*CostReport, error) {
    now := time.Now().UTC()
    start := now.Add(-period)

    // ── Fan-out: 3 AWS regions + 1 Kubecost query = 4 concurrent tasks ────
    tasks := make(map[string]func(context.Context) (interface{}, error))

    for _, region := range regions {
        r := region // capture
        tasks["aws-"+r] = func(ctx context.Context) (interface{}, error) {
            return ceClient.GetCostByTags(ctx, costexplorer.GetCostInput{
                Region:    r,
                StartDate: start,
                EndDate:   now,
                TagKeys:   []string{"team", "service"},
            })
        }
    }

    tasks["kubecost"] = func(ctx context.Context) (interface{}, error) {
        return kcClient.GetNamespaceCosts(ctx, "30d")
    }

    results := concurrency.FanOut[interface{}](ctx, tasks)

    // ── Fan-in: collect results, capture errors ───────────────────────────
    var awsRecords []CostRecord
    var kcRecords []kubecost.NamespaceCost
    var errors []string

    for _, r := range results {
        if r.Err != nil {
            errors = append(errors, fmt.Sprintf("%s: %v", r.Source, r.Err))
            continue
        }
        switch v := r.Value.(type) {
        case []CostRecord:
            awsRecords = append(awsRecords, v...)
        case []kubecost.NamespaceCost:
            kcRecords = append(kcRecords, v...)
        }
    }

    // ── Correlate + analyze ───────────────────────────────────────────────
    correlated := correlateCosts(awsRecords, kcRecords)
    report := aggregate(correlated, start, now)
    report.Anomalies = detectAnomalies(report.ByService, 0.20)
    report.Errors = errors
    report.Regions = regions

    return report, nil
}
```

## Cobra Command (`cmd/cost/report.go`)

```go
package cost

import (
    "context"
    "fmt"
    "os"
    "time"

    "github.com/spf13/cobra"
    "github.com/novamart/novactl/internal/aws/costexplorer"
    costpkg "github.com/novamart/novactl/internal/cost"
    "github.com/novamart/novactl/internal/kubecost"
    "github.com/novamart/novactl/internal/notify"
    "github.com/novamart/novactl/internal/output"
)

var (
    flagRegions        []string
    flagPeriodDays     int
    flagOutputFormat   string
    flagSlack          bool
    flagAnomalyThresh  float64
    flagKubecostURL    string
)

func NewReportCmd() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "report",
        Short: "Generate cost attribution report across AWS and K8s",
        Long: `Queries AWS Cost Explorer and Kubecost, correlates costs by team/service
tags, detects anomalies, and outputs a comprehensive cost report.

Partial failures are tolerated — if one region is unreachable, the report
includes data from the others with a warning.`,
        Example: `  novactl cost report
  novactl cost report --regions us-east-1,eu-west-1 --period 14 -o json
  novactl cost report --no-slack --format csv > costs.csv`,
        RunE: runReport,
    }

    cmd.Flags().StringSliceVarP(&flagRegions, "regions", "r",
        []string{"us-east-1", "us-west-2", "eu-west-1"}, "AWS regions to query")
    cmd.Flags().IntVarP(&flagPeriodDays, "period", "p", 30, "Lookback period in days")
    cmd.Flags().StringVarP(&flagOutputFormat, "format", "o", "table",
        "Output format: table, json, csv")
    cmd.Flags().BoolVar(&flagSlack, "slack", true, "Send Slack summary")
    cmd.Flags().Float64Var(&flagAnomalyThresh, "anomaly-threshold", 0.20,
        "Anomaly threshold (fraction above 7-day avg)")
    cmd.Flags().StringVar(&flagKubecostURL, "kubecost-url",
        "http://kubecost-cost-analyzer.kubecost:9090", "Kubecost API base URL")

    return cmd
}

func runReport(cmd *cobra.Command, args []string) error {
    ctx, cancel := context.WithTimeout(cmd.Context(), 5*time.Minute)
    defer cancel()

    // Build clients
    ceClient, err := costexplorer.New(ctx)
    if err != nil {
        return fmt.Errorf("initializing cost explorer client: %w", err)
    }
    kcClient := kubecost.New(flagKubecostURL)

    // Build report
    period := time.Duration(flagPeriodDays) * 24 * time.Hour
    report, err := costpkg.BuildReport(ctx, ceClient, kcClient, flagRegions, period)
    if err != nil {
        return fmt.Errorf("building cost report: %w", err)
    }

    // Warn about partial failures
    for _, e := range report.Errors {
        fmt.Fprintf(os.Stderr, "WARNING: %s\n", e)
    }

    // Format output
    formatter, err := output.NewFormatter(flagOutputFormat)
    if err != nil {
        return err
    }
    if err := formatter.Render(os.Stdout, report); err != nil {
        return fmt.Errorf("rendering output: %w", err)
    }

    // Slack notification
    if flagSlack {
        webhookURL := os.Getenv("SLACK_WEBHOOK_URL")
        if webhookURL == "" {
            fmt.Fprintln(os.Stderr, "WARNING: SLACK_WEBHOOK_URL not set, skipping notification")
        } else if err := notify.SendCostSummary(ctx, webhookURL, report); err != nil {
            fmt.Fprintf(os.Stderr, "WARNING: Slack notification failed: %v\n", err)
        }
    }

    // Exit code: non-zero if anomalies found (for CI gating)
    if len(report.Anomalies) > 0 {
        return fmt.Errorf("found %d cost anomalies", len(report.Anomalies))
    }

    return nil
}
```

## Correlation Logic + Test (`internal/cost/correlator.go`)

```go
package cost

// correlateCosts joins AWS cost records with Kubecost namespace data via tags.
// Strategy:
//   1. Index Kubecost records by namespace label "service" -> NamespaceCost
//   2. For each AWS record, look up by "service" tag -> enrich with K8s costs
//   3. Unmatched AWS records are flagged as uncorrelated (untagged)
//   4. Unmatched Kubecost records become AWS-unattributed K8s costs
func correlateCosts(
    awsRecords []CostRecord,
    kcRecords []kubecost.NamespaceCost,
) []CorrelatedCost {
    // Build lookup: service-name → kubecost data
    kcByService := make(map[string]kubecost.NamespaceCost, len(kcRecords))
    kcUsed := make(map[string]bool)
    for _, kc := range kcRecords {
        svcLabel := kc.Labels["service"]
        if svcLabel == "" {
            svcLabel = kc.Namespace // fallback: use namespace as service name
        }
        kcByService[svcLabel] = kc
    }

    result := make([]CorrelatedCost, 0, len(awsRecords)+len(kcRecords))

    for _, aws := range awsRecords {
        cc := CorrelatedCost{CostRecord: aws}

        svc := aws.Tags["service"]
        if svc == "" {
            svc = aws.Service
        }

        if kc, ok := kcByService[svc]; ok {
            cc.K8sCPUCost = kc.CPUCost
            cc.K8sMemoryCost = kc.MemoryCost
            cc.K8sGPUCost = kc.GPUCost
            cc.Correlated = true
            kcUsed[svc] = true
        }

        result = append(result, cc)
    }

    // Add K8s-only costs (no matching AWS record)
    for svc, kc := range kcByService {
        if !kcUsed[svc] {
            result = append(result, CorrelatedCost{
                CostRecord: CostRecord{
                    Service:   svc,
                    Namespace: kc.Namespace,
                    Team:      kc.Labels["team"],
                    Amount:    kc.TotalCost,
                    Source:    SourceKubecost,
                },
                K8sCPUCost:    kc.CPUCost,
                K8sMemoryCost: kc.MemoryCost,
                Correlated:    false,
            })
        }
    }

    return result
}
```

## Test (Table-Driven, No AWS)

```go
package cost

import (
    "testing"
    "time"

    "github.com/novamart/novactl/internal/kubecost"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestCorrelateCosts(t *testing.T) {
    tests := []struct {
        name             string
        awsRecords       []CostRecord
        kcRecords        []kubecost.NamespaceCost
        wantTotal        int
        wantCorrelated   int
        wantUncorrelated int
        wantUntaggedAmt  float64
    }{
        {
            name: "perfect match — all services correlated",
            awsRecords: []CostRecord{
                {Service: "order-service", Team: "orders-team", Amount: 100.0,
                    Tags: map[string]string{"service": "order-service", "team": "orders-team"}},
                {Service: "payment-service", Team: "payments-team", Amount: 200.0,
                    Tags: map[string]string{"service": "payment-service", "team": "payments-team"}},
            },
            kcRecords: []kubecost.NamespaceCost{
                {Namespace: "orders", CPUCost: 30, MemoryCost: 10, TotalCost: 40,
                    Labels: map[string]string{"service": "order-service"}},
                {Namespace: "payments", CPUCost: 60, MemoryCost: 20, TotalCost: 80,
                    Labels: map[string]string{"service": "payment-service"}},
            },
            wantTotal: 2, wantCorrelated: 2, wantUncorrelated: 0,
        },
        {
            name: "untagged AWS spend — no service tag",
            awsRecords: []CostRecord{
                {Service: "", Amount: 50.0, Tags: map[string]string{}},
            },
            kcRecords:       []kubecost.NamespaceCost{},
            wantTotal:       1,
            wantCorrelated:  0,
            wantUncorrelated: 1,
        },
        {
            name: "K8s-only cost — no matching AWS record",
            awsRecords: []CostRecord{},
            kcRecords: []kubecost.NamespaceCost{
                {Namespace: "ml-training", CPUCost: 500, MemoryCost: 200,
                    TotalCost: 700, Labels: map[string]string{"service": "ml-pipeline"}},
            },
            wantTotal: 1, wantCorrelated: 0, wantUncorrelated: 1,
        },
        {
            name: "mixed — some correlated, some orphaned on both sides",
            awsRecords: []CostRecord{
                {Service: "order-service", Amount: 100,
                    Tags: map[string]string{"service": "order-service", "team": "orders"}},
                {Service: "legacy-batch", Amount: 75,
                    Tags: map[string]string{"service": "legacy-batch"}}, // no K8s match
            },
            kcRecords: []kubecost.NamespaceCost{
                {Namespace: "orders", CPUCost: 30, MemoryCost: 10, TotalCost: 40,
                    Labels: map[string]string{"service": "order-service"}},
                {Namespace: "experiments", CPUCost: 5, MemoryCost: 2, TotalCost: 7,
                    Labels: map[string]string{"service": "experiment-svc"}}, // no AWS match
            },
            wantTotal: 3, wantCorrelated: 1, wantUncorrelated: 2,
        },
        {
            name: "empty inputs",
            awsRecords: []CostRecord{},
            kcRecords:  []kubecost.NamespaceCost{},
            wantTotal:  0,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := correlateCosts(tt.awsRecords, tt.kcRecords)

            assert.Len(t, result, tt.wantTotal)

            correlated := 0
            uncorrelated := 0
            for _, r := range result {
                if r.Correlated {
                    correlated++
                } else {
                    uncorrelated++
                }
            }
            assert.Equal(t, tt.wantCorrelated, correlated, "correlated count mismatch")
            assert.Equal(t, tt.wantUncorrelated, uncorrelated, "uncorrelated count mismatch")
        })
    }
}

func TestDetectAnomalies(t *testing.T) {
    tests := []struct {
        name       string
        services   []ServiceCostSummary
        threshold  float64
        wantCount  int
    }{
        {
            name: "25% spike detected at 20% threshold",
            services: []ServiceCostSummary{
                {Service: "order-svc", DailyAvg: 125.0, SevenDayAvg: 100.0},
            },
            threshold: 0.20,
            wantCount: 1,
        },
        {
            name: "15% spike below 20% threshold — no anomaly",
            services: []ServiceCostSummary{
                {Service: "order-svc", DailyAvg: 115.0, SevenDayAvg: 100.0},
            },
            threshold: 0.20,
            wantCount: 0,
        },
        {
            name: "zero seven-day avg — no division by zero",
            services: []ServiceCostSummary{
                {Service: "new-svc", DailyAvg: 50.0, SevenDayAvg: 0.0},
            },
            threshold: 0.20,
            wantCount: 0, // can't compute deviation; skip
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            anomalies := detectAnomalies(tt.services, tt.threshold)
            assert.Len(t, anomalies, tt.wantCount)
        })
    }
}
```

## Error Handling Strategy Summary

| Scenario | Behavior |
|---|---|
| One AWS region unreachable | Captured in `report.Errors`; other regions' data included. stderr warning. |
| Kubecost API down | Captured in `report.Errors`; report shows AWS-only data. K8s columns empty. |
| All sources fail | `BuildReport` returns report with `Errors` populated, zero cost data. CLI prints warnings. |
| Slack fails | stderr warning, non-fatal. Report still output to stdout. |
| Anomalies found | CLI returns exit code 1 (for CI gating), report fully rendered. |

---

# Q2 — Debugging: ECR Cleanup Silent Failures

## Bug 1: **Swallowed error on `DescribeImages`** — THE SILENT SKIP ROOT CAUSE

```go
images, _ := client.DescribeImages(ctx, &ecr.DescribeImagesInput{
    RepositoryName: &repo,
})
```

**Problem:** The error is discarded with `_`. If `DescribeImages` fails (permissions, throttling, repo doesn't exist), `images` is `nil`. Accessing `images.ImageDetails` on a nil `images` should panic — **but it doesn't always** because if the API returns a non-nil response with an empty `ImageDetails` slice, the for loop simply iterates nothing.

More critically: **AWS API throttling** returns retryable errors. After 6 months with more images in the repos, the API calls are more likely to be throttled. The error is silently discarded, and the entire repo is skipped.

**Production consequence:** Repos that trigger API errors are silently skipped. No images are cleaned from those repos. Over time, ECR storage costs balloon and the "silent skipping" goes undetected.

**Fix:**
```go
images, err := client.DescribeImages(ctx, &ecr.DescribeImagesInput{
    RepositoryName: &repo,
})
if err != nil {
    log.Printf("ERROR: DescribeImages for %s: %v", repo, err)
    continue
}
```

---

## Bug 2: **Missing pagination — only first page of images returned**

```go
images, _ := client.DescribeImages(ctx, &ecr.DescribeImagesInput{
    RepositoryName: &repo,
})
```

**Problem:** `DescribeImages` returns a maximum of **1,000 images per page** (default 100). The response includes a `NextToken` for pagination. This code reads only the first page and ignores the rest.

After 6 months with CI/CD pushing images constantly, many repos likely have >100 images. Only the first page is checked; all subsequent images are invisible.

**Production consequence:** This is the primary cause of "silently skipping images." With 500 images in a repo, only the first 100 are candidates for cleanup. The other 400 accumulate forever.

**Fix:**
```go
paginator := ecr.NewDescribeImagesPaginator(client, &ecr.DescribeImagesInput{
    RepositoryName: &repo,
})
for paginator.HasMorePages() {
    page, err := paginator.NextPage(ctx)
    if err != nil {
        log.Printf("ERROR: paginating %s: %v", repo, err)
        break
    }
    for _, img := range page.ImageDetails {
        // ... process image
    }
}
```

---

## Bug 3: **Swallowed error on `BatchDeleteImage`**

```go
client.BatchDeleteImage(ctx, &ecr.BatchDeleteImageInput{
    RepositoryName: &repo,
    ImageIds: []types.ImageIdentifier{
        {ImageDigest: img.ImageDigest},
    },
})
```

**Problem:** The return value and error are both discarded. `BatchDeleteImage` can fail due to:
- Permissions (IAM `ecr:BatchDeleteImage` missing)
- Image is referenced by a manifest list
- Throttling
- The response contains a `Failures` slice with per-image failure reasons

You never know if the delete actually succeeded. The log says "Deleting…" but the image may still be there.

**Production consequence:** The nightly job reports deletions that didn't happen. ECR continues accumulating images. Storage costs grow. The team thinks cleanup is working.

**Fix:**
```go
resp, err := client.BatchDeleteImage(ctx, &ecr.BatchDeleteImageInput{...})
if err != nil {
    log.Printf("ERROR: BatchDeleteImage for %s: %v", repo, err)
    continue
}
for _, f := range resp.Failures {
    log.Printf("WARN: failed to delete %s in %s: %s (%s)",
        *f.ImageId.ImageDigest, repo, *f.FailureReason, f.FailureCode)
}
```

---

## Bug 4: **Untagged images are silently skipped — only logged when tags exist, but always deleted**

```go
if img.ImageTags != nil && len(img.ImageTags) > 0 {
    log.Printf("Deleting %s:%s (age: %d days)", 
        repo, img.ImageTags[0], int(age.Hours()/24))
}

client.BatchDeleteImage(ctx, &ecr.BatchDeleteImageInput{...})
```

**Problem:** The `if` block only controls the **log message**, not the delete operation. The `BatchDeleteImage` call happens unconditionally. This means:

1. **Untagged images are deleted without any log.** You have no audit trail.
2. **The intent is ambiguous.** Did the author mean to skip untagged images? The structure suggests the delete was supposed to be inside the `if` block.

**Production consequence:** Untagged images (which may include intermediate build layers or critical base images) are silently deleted. When debugging image pull failures later, there's no log of what was removed.

**Fix (if intent is to delete all old images, but log all):**
```go
tagStr := "<untagged>"
if len(img.ImageTags) > 0 {
    tagStr = img.ImageTags[0]
}
log.Printf("Deleting %s:%s (digest: %s, age: %d days)", 
    repo, tagStr, *img.ImageDigest, int(age.Hours()/24))

// Then delete...
```

---

## Bug 5: **`time.Now().Sub(...)` uses local time — `ImagePushedAt` is UTC**

```go
age := time.Now().Sub(*img.ImagePushedAt)
```

**Problem:** `time.Now()` returns local time. `img.ImagePushedAt` is UTC (from AWS API). While Go's `time.Sub` correctly handles timezone arithmetic (it computes monotonic duration), the **semantic age may be misleading** in log output and in edge cases around the threshold boundary.

More importantly: if this runs in a container without a timezone set, `time.Now()` defaults to UTC — no bug. But if it runs on a developer laptop or EC2 instance with a non-UTC timezone, the **displayed** age in logs may be wrong, and images at the boundary could be deleted prematurely or skipped.

**Fix:**
```go
age := time.Since(*img.ImagePushedAt) // idiomatic; equivalent but clearer
// Or explicitly: time.Now().UTC().Sub(...)
```

---

## Bug 6: **Single-image `BatchDeleteImage` calls — massively inefficient and throttle-prone**

```go
client.BatchDeleteImage(ctx, &ecr.BatchDeleteImageInput{
    RepositoryName: &repo,
    ImageIds: []types.ImageIdentifier{
        {ImageDigest: img.ImageDigest},
    },
})
```

**Problem:** `BatchDeleteImage` accepts **up to 100 image IDs per call**. This code sends one image per API call. For a repo with 500 old images, that's 500 API calls instead of 5. This dramatically increases the chance of hitting AWS API throttle limits, which (per Bug 1) are silently swallowed.

**Production consequence:** After adding 200 new instances and more CI/CD activity, the repos have more images to clean. The 1-at-a-time deletion pattern triggers throttling. Throttled calls return errors, which are silently discarded (Bug 1), and those images are never cleaned.

**Fix:**
```go
const batchSize = 100
var toDelete []types.ImageIdentifier

for _, img := range oldImages {
    toDelete = append(toDelete, types.ImageIdentifier{ImageDigest: img.ImageDigest})
    if len(toDelete) >= batchSize {
        deleteBatch(ctx, client, repo, toDelete)
        toDelete = toDelete[:0]
    }
}
if len(toDelete) > 0 {
    deleteBatch(ctx, client, repo, toDelete)
}
```

---

## Bug 7: **`ImagePushedAt` may be nil — potential nil pointer dereference**

```go
age := time.Now().Sub(*img.ImagePushedAt)
```

**Problem:** `img.ImagePushedAt` is a `*time.Time`. For images that are still being uploaded or in a weird state, this can be `nil`. Dereferencing a nil `*time.Time` causes a **panic** that crashes the entire function — every remaining repo is skipped.

**Production consequence:** One malformed image metadata entry kills the cleanup for all remaining repos.

**Fix:**
```go
if img.ImagePushedAt == nil {
    log.Printf("WARN: %s has nil ImagePushedAt, skipping", *img.ImageDigest)
    continue
}
```

---

## Bug 8: **No context timeout / cancellation propagation**

```go
func cleanupECRImages(ctx context.Context, repos []string, maxAge int) {
```

**Problem:** The function accepts a `ctx` but never checks if it's cancelled. If the calling code sets a deadline (e.g., Lambda 15-minute timeout), and this function is processing hundreds of repos, it will continue making API calls after the context is cancelled — receiving errors it then discards (Bug 1).

**Fix:** Check context in the loop:
```go
for _, repo := range repos {
    if ctx.Err() != nil {
        log.Printf("Context cancelled, stopping cleanup")
        return
    }
    // ...
}
```

---

## Bug 9: **`&repo` captures the loop variable — all calls use the same pointer**

```go
for _, repo := range repos {
    images, _ := client.DescribeImages(ctx, &ecr.DescribeImagesInput{
        RepositoryName: &repo,
    })
```

**Problem:** In Go versions prior to 1.22, `&repo` takes the address of the loop variable, which is **reused across iterations**. Since `DescribeImages` is synchronous here, this happens to work — by the time the next iteration overwrites `repo`, the previous call has completed. **However**, `BatchDeleteImage` also uses `&repo`, and if the code is ever made concurrent (e.g., with goroutines per repo), all calls would reference the same `repo` pointer — the last repo in the slice.

Even in the synchronous version, this is a latent bug that will bite during the inevitable refactor.

**Post Go 1.22:** The loop variable is now per-iteration, so this is safe. But if running on Go <1.22 (which is likely for "running 6 months"), this is a real risk.

**Fix:**
```go
for _, repo := range repos {
    repo := repo // capture for pre-1.22 safety
    // ...
}
```

---

## Corrected Code

```go
func cleanupECRImages(ctx context.Context, repos []string, maxAge int) error {
    cfg := getAWSConfig()
    client := ecr.NewFromConfig(cfg)
    const batchSize = 100

    var totalDeleted, totalFailed int

    for _, repo := range repos {
        repo := repo // capture loop variable

        if ctx.Err() != nil {
            return fmt.Errorf("context cancelled: %w", ctx.Err())
        }

        log.Printf("Scanning repo: %s", repo)

        paginator := ecr.NewDescribeImagesPaginator(client, &ecr.DescribeImagesInput{
            RepositoryName: &repo,
        })

        var toDelete []types.ImageIdentifier

        for paginator.HasMorePages() {
            page, err := paginator.NextPage(ctx)
            if err != nil {
                log.Printf("ERROR: DescribeImages page for %s: %v", repo, err)
                break
            }

            for _, img := range page.ImageDetails {
                if img.ImagePushedAt == nil {
                    log.Printf("WARN: %s/%s has nil ImagePushedAt, skipping",
                        repo, safeDigest(img.ImageDigest))
                    continue
                }

                age := time.Since(*img.ImagePushedAt)
                ageDays := int(age.Hours() / 24)

                if ageDays > maxAge {
                    tagStr := "<untagged>"
                    if len(img.ImageTags) > 0 {
                        tagStr = img.ImageTags[0]
                    }
                    log.Printf("Queuing deletion: %s:%s (digest: %s, age: %dd)",
                        repo, tagStr, safeDigest(img.ImageDigest), ageDays)

                    toDelete = append(toDelete, types.ImageIdentifier{
                        ImageDigest: img.ImageDigest,
                    })
                }
            }
        }

        // Batch delete
        for i := 0; i < len(toDelete); i += batchSize {
            end := i + batchSize
            if end > len(toDelete) {
                end = len(toDelete)
            }
            batch := toDelete[i:end]

            resp, err := client.BatchDeleteImage(ctx, &ecr.BatchDeleteImageInput{
                RepositoryName: &repo,
                ImageIds:       batch,
            })
            if err != nil {
                log.Printf("ERROR: BatchDeleteImage for %s: %v", repo, err)
                totalFailed += len(batch)
                continue
            }
            totalDeleted += len(resp.ImageIds)
            for _, f := range resp.Failures {
                log.Printf("WARN: delete failure in %s: %s (code: %s)",
                    repo, *f.FailureReason, f.FailureCode)
                totalFailed++
            }
        }

        log.Printf("Repo %s: queued %d for deletion", repo, len(toDelete))
    }

    log.Printf("Cleanup complete: %d deleted, %d failed", totalDeleted, totalFailed)
    return nil
}

func safeDigest(d *string) string {
    if d == nil {
        return "<nil>"
    }
    if len(*d) > 19 {
        return (*d)[:19] + "…"
    }
    return *d
}
```

---

# Q3 — Production Code: K8s Deployment Pre-Check Validator

## Full Implementation

```go
// internal/validator/precheck.go
package validator

import (
    "context"
    "fmt"
    "log/slog"
    "strings"
    "time"
)

// ── Check Result Model ─────────────────────────────────────────────────────

type CheckStatus string

const (
    StatusPass CheckStatus = "PASS"
    StatusFail CheckStatus = "FAIL"
    StatusWarn CheckStatus = "WARN"
    StatusSkip CheckStatus = "SKIP"
)

type CheckResult struct {
    Name    string      `json:"name"`
    Status  CheckStatus `json:"status"`
    Message string      `json:"message"`
    Details string      `json:"details,omitempty"`
}

type PreCheckResult struct {
    Deployment string        `json:"deployment"`
    Namespace  string        `json:"namespace"`
    Image      string        `json:"image"`
    Timestamp  time.Time     `json:"timestamp"`
    Checks     []CheckResult `json:"checks"`
    GoNoGo     bool          `json:"go_no_go"`
    Summary    string        `json:"summary"`
}

func (r *PreCheckResult) addCheck(c CheckResult) {
    r.Checks = append(r.Checks, c)
}

func (r *PreCheckResult) computeGoNoGo() {
    r.GoNoGo = true
    var failures []string
    for _, c := range r.Checks {
        if c.Status == StatusFail {
            r.GoNoGo = false
            failures = append(failures, c.Name)
        }
    }
    if r.GoNoGo {
        r.Summary = "All pre-checks passed — safe to deploy"
    } else {
        r.Summary = fmt.Sprintf("BLOCKED: %d check(s) failed: %s",
            len(failures), strings.Join(failures, ", "))
    }
}

// ── Input ───────────────────────────────────────────────────────────────────

type DeploymentSpec struct {
    Name      string
    Namespace string
    Image     string // "registry.novamart.com/order-service:v1.2.3"
    Replicas  int32
}

// ── Interfaces (for testing) ────────────────────────────────────────────────

type ECRClient interface {
    ImageExists(ctx context.Context, repository, tag string) (bool, error)
}

type K8sClient interface {
    // Quota check: returns available CPU (millicores) and memory (bytes) in namespace
    GetAvailableQuota(ctx context.Context, namespace string) (cpuMillis int64, memBytes int64, err error)

    // PDB check: returns true if any PDB in the namespace would block a full rollout
    PDBWouldBlock(ctx context.Context, namespace string, deploymentName string, replicas int32) (blocked bool, reason string, err error)

    // Rollout check: returns true if deployment currently has an in-progress rollout
    RolloutInProgress(ctx context.Context, namespace string, deploymentName string) (bool, error)

    // Resource requirements for the deployment's pod spec
    GetPodResourceRequests(ctx context.Context, namespace, deployment string) (cpuMillis int64, memBytes int64, err error)
}

// ── Validator ───────────────────────────────────────────────────────────────

type PreCheckValidator struct {
    ecr    ECRClient
    k8s    K8sClient
    logger *slog.Logger
}

func NewPreCheckValidator(ecr ECRClient, k8s K8sClient, logger *slog.Logger) *PreCheckValidator {
    if logger == nil {
        logger = slog.Default()
    }
    return &PreCheckValidator{ecr: ecr, k8s: k8s, logger: logger}
}

func (v *PreCheckValidator) Validate(ctx context.Context, spec DeploymentSpec) (*PreCheckResult, error) {
    if spec.Name == "" || spec.Namespace == "" || spec.Image == "" {
        return nil, fmt.Errorf("deployment spec requires name, namespace, and image")
    }

    result := &PreCheckResult{
        Deployment: spec.Name,
        Namespace:  spec.Namespace,
        Image:      spec.Image,
        Timestamp:  time.Now().UTC(),
    }

    v.logger.Info("starting pre-check validation",
        "deployment", spec.Name,
        "namespace", spec.Namespace,
        "image", spec.Image,
    )

    // Check 1: Image exists in ECR
    v.checkImageExists(ctx, spec, result)

    // Check 2: Namespace resource quota
    v.checkResourceQuota(ctx, spec, result)

    // Check 3: PDB won't block rollout
    v.checkPDB(ctx, spec, result)

    // Check 4: No in-progress rollout
    v.checkRolloutStatus(ctx, spec, result)

    result.computeGoNoGo()

    v.logger.Info("pre-check complete",
        "deployment", spec.Name,
        "go_no_go", result.GoNoGo,
        "summary", result.Summary,
    )

    return result, nil
}

// ── Individual Checks ──────────────────────────────────────────────────────

func (v *PreCheckValidator) checkImageExists(ctx context.Context, spec DeploymentSpec, result *PreCheckResult) {
    repo, tag, err := parseImage(spec.Image)
    if err != nil {
        result.addCheck(CheckResult{
            Name:    "image_exists",
            Status:  StatusFail,
            Message: fmt.Sprintf("Invalid image reference: %v", err),
        })
        return
    }

    exists, err := v.ecr.ImageExists(ctx, repo, tag)
    if err != nil {
        result.addCheck(CheckResult{
            Name:    "image_exists",
            Status:  StatusFail,
            Message: fmt.Sprintf("ECR check failed: %v", err),
            Details: "Could not verify image existence — failing safe",
        })
        return
    }

    if !exists {
        result.addCheck(CheckResult{
            Name:    "image_exists",
            Status:  StatusFail,
            Message: fmt.Sprintf("Image %s:%s not found in ECR", repo, tag),
            Details: "Deployment would fail with ImagePullBackOff",
        })
        return
    }

    result.addCheck(CheckResult{
        Name:    "image_exists",
        Status:  StatusPass,
        Message: fmt.Sprintf("Image %s:%s exists in ECR", repo, tag),
    })
}

func (v *PreCheckValidator) checkResourceQuota(ctx context.Context, spec DeploymentSpec, result *PreCheckResult) {
    availCPU, availMem, err := v.k8s.GetAvailableQuota(ctx, spec.Namespace)
    if err != nil {
        result.addCheck(CheckResult{
            Name:    "resource_quota",
            Status:  StatusWarn,
            Message: fmt.Sprintf("Could not check quota: %v", err),
            Details: "Namespace may not have ResourceQuota configured",
        })
        return
    }

    reqCPU, reqMem, err := v.k8s.GetPodResourceRequests(ctx, spec.Namespace, spec.Name)
    if err != nil {
        result.addCheck(CheckResult{
            Name:    "resource_quota",
            Status:  StatusWarn,
            Message: fmt.Sprintf("Could not get pod resource requests: %v", err),
        })
        return
    }

    // During a rolling update, there's a surge — at least 1 extra pod
    totalCPU := reqCPU * int64(spec.Replicas+1)
    totalMem := reqMem * int64(spec.Replicas+1)

    if totalCPU > availCPU || totalMem > availMem {
        result.addCheck(CheckResult{
            Name:   "resource_quota",
            Status: StatusFail,
            Message: fmt.Sprintf("Insufficient quota: need %dm CPU/%dMi mem, have %dm/%dMi",
                totalCPU, totalMem/(1024*1024), availCPU, availMem/(1024*1024)),
            Details: "Rolling update surge would exceed namespace ResourceQuota",
        })
        return
    }

    result.addCheck(CheckResult{
        Name:    "resource_quota",
        Status:  StatusPass,
        Message: "Namespace has sufficient quota for rolling update",
    })
}

func (v *PreCheckValidator) checkPDB(ctx context.Context, spec DeploymentSpec, result *PreCheckResult) {
    blocked, reason, err := v.k8s.PDBWouldBlock(ctx, spec.Namespace, spec.Name, spec.Replicas)
    if err != nil {
        result.addCheck(CheckResult{
            Name:    "pdb_check",
            Status:  StatusWarn,
            Message: fmt.Sprintf("Could not evaluate PDBs: %v", err),
        })
        return
    }

    if blocked {
        result.addCheck(CheckResult{
            Name:    "pdb_check",
            Status:  StatusFail,
            Message: "PodDisruptionBudget would block rollout",
            Details: reason,
        })
        return
    }

    result.addCheck(CheckResult{
        Name:    "pdb_check",
        Status:  StatusPass,
        Message: "No PDB conflicts detected",
    })
}

func (v *PreCheckValidator) checkRolloutStatus(ctx context.Context, spec DeploymentSpec, result *PreCheckResult) {
    inProgress, err := v.k8s.RolloutInProgress(ctx, spec.Namespace, spec.Name)
    if err != nil {
        result.addCheck(CheckResult{
            Name:    "rollout_status",
            Status:  StatusWarn,
            Message: fmt.Sprintf("Could not check rollout status: %v", err),
        })
        return
    }

    if inProgress {
        result.addCheck(CheckResult{
            Name:    "rollout_status",
            Status:  StatusFail,
            Message: "A rollout is already in progress",
            Details: "Deploying now could cause unexpected behavior or double surge",
        })
        return
    }

    result.addCheck(CheckResult{
        Name:    "rollout_status",
        Status:  StatusPass,
        Message: "No in-progress rollout detected",
    })
}

// ── Helpers ─────────────────────────────────────────────────────────────────

func parseImage(image string) (repo, tag string, err error) {
    // Handle: registry.novamart.com/order-service:v1.2.3
    //         order-service:v1.2.3
    //         registry.novamart.com/order-service (implicit :latest)
    //         registry.novamart.com/order-service@sha256:abc123

    if strings.Contains(image, "@") {
        // Digest reference: split on @
        parts := strings.SplitN(image, "@", 2)
        return parts[0], parts[1], nil
    }

    lastColon := strings.LastIndex(image, ":")
    // Check if the colon is part of a port number (before the first /)
    firstSlash := strings.Index(image, "/")

    if lastColon == -1 || (firstSlash != -1 && lastColon < firstSlash) {
        // No tag specified
        return image, "latest", nil
    }

    repo = image[:lastColon]
    tag = image[lastColon+1:]

    if repo == "" || tag == "" {
        return "", "", fmt.Errorf("invalid image reference: %q", image)
    }

    return repo, tag, nil
}
```

## Table-Driven Tests

```go
// internal/validator/precheck_test.go
package validator

import (
    "context"
    "fmt"
    "log/slog"
    "os"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// ── Mocks ───────────────────────────────────────────────────────────────────

type mockECR struct {
    images map[string]bool  // "repo:tag" -> exists
    err    error
}

func (m *mockECR) ImageExists(_ context.Context, repo, tag string) (bool, error) {
    if m.err != nil {
        return false, m.err
    }
    return m.images[repo+":"+tag], nil
}

type mockK8s struct {
    availCPU        int64
    availMem        int64
    quotaErr        error
    podCPU          int64
    podMem          int64
    podResourceErr  error
    pdbBlocked      bool
    pdbReason       string
    pdbErr          error
    rolloutActive   bool
    rolloutErr      error
}

func (m *mockK8s) GetAvailableQuota(_ context.Context, _ string) (int64, int64, error) {
    return m.availCPU, m.availMem, m.quotaErr
}

func (m *mockK8s) GetPodResourceRequests(_ context.Context, _, _ string) (int64, int64, error) {
    return m.podCPU, m.podMem, m.podResourceErr
}

func (m *mockK8s) PDBWouldBlock(_ context.Context, _, _ string, _ int32) (bool, string, error) {
    return m.pdbBlocked, m.pdbReason, m.pdbErr
}

func (m *mockK8s) RolloutInProgress(_ context.Context, _, _ string) (bool, error) {
    return m.rolloutActive, m.rolloutErr
}

// ── Tests ───────────────────────────────────────────────────────────────────

func TestPreCheckValidator(t *testing.T) {
    logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelWarn}))

    baseSpec := DeploymentSpec{
        Name:      "order-service",
        Namespace: "production",
        Image:     "registry.novamart.com/order-service:v1.2.3",
        Replicas:  3,
    }

    tests := []struct {
        name           string
        spec           DeploymentSpec
        ecr            *mockECR
        k8s            *mockK8s
        wantGoNoGo     bool
        wantCheckCount int
        wantStatuses   map[string]CheckStatus // check name -> expected status
    }{
        {
            name: "all checks pass — green light",
            spec: baseSpec,
            ecr: &mockECR{
                images: map[string]bool{
                    "registry.novamart.com/order-service:v1.2.3": true,
                },
            },
            k8s: &mockK8s{
                availCPU: 8000, availMem: 16 * 1024 * 1024 * 1024,
                podCPU: 500, podMem: 512 * 1024 * 1024,
            },
            wantGoNoGo:     true,
            wantCheckCount: 4,
            wantStatuses: map[string]CheckStatus{
                "image_exists":   StatusPass,
                "resource_quota": StatusPass,
                "pdb_check":      StatusPass,
                "rollout_status": StatusPass,
            },
        },
        {
            name: "image missing in ECR — blocked",
            spec: baseSpec,
            ecr: &mockECR{
                images: map[string]bool{}, // empty — nothing exists
            },
            k8s: &mockK8s{
                availCPU: 8000, availMem: 16 * 1024 * 1024 * 1024,
                podCPU: 500, podMem: 512 * 1024 * 1024,
            },
            wantGoNoGo: false,
            wantStatuses: map[string]CheckStatus{
                "image_exists": StatusFail,
            },
        },
        {
            name: "ECR unreachable — fail safe",
            spec: baseSpec,
            ecr: &mockECR{
                err: fmt.Errorf("connection timeout"),
            },
            k8s: &mockK8s{
                availCPU: 8000, availMem: 16 * 1024 * 1024 * 1024,
                podCPU: 500, podMem: 512 * 1024 * 1024,
            },
            wantGoNoGo: false,
            wantStatuses: map[string]CheckStatus{
                "image_exists": StatusFail, // fail-safe: can't verify = don't deploy
            },
        },
        {
            name: "insufficient quota — blocked",
            spec: baseSpec,
            ecr: &mockECR{
                images: map[string]bool{
                    "registry.novamart.com/order-service:v1.2.3": true,
                },
            },
            k8s: &mockK8s{
                availCPU: 1000, availMem: 1024 * 1024 * 1024, // only 1 CPU, 1GB
                podCPU: 500, podMem: 512 * 1024 * 1024, // 500m per pod × 4 (3 replicas + 1 surge) = 2000m > 1000m
            },
            wantGoNoGo: false,
            wantStatuses: map[string]CheckStatus{
                "image_exists":   StatusPass,
                "resource_quota": StatusFail,
            },
        },
        {
            name: "PDB would block rollout",
            spec: baseSpec,
            ecr: &mockECR{
                images: map[string]bool{
                    "registry.novamart.com/order-service:v1.2.3": true,
                },
            },
            k8s: &mockK8s{
                availCPU: 8000, availMem: 16 * 1024 * 1024 * 1024,
                podCPU: 500, podMem: 512 * 1024 * 1024,
                pdbBlocked: true,
                pdbReason:  "PDB order-service-pdb allows 0 disruptions",
            },
            wantGoNoGo: false,
            wantStatuses: map[string]CheckStatus{
                "image_exists":   StatusPass,
                "resource_quota": StatusPass,
                "pdb_check":      StatusFail,
            },
        },
        {
            name: "rollout already in progress",
            spec: baseSpec,
            ecr: &mockECR{
                images: map[string]bool{
                    "registry.novamart.com/order-service:v1.2.3": true,
                },
            },
            k8s: &mockK8s{
                availCPU: 8000, availMem: 16 * 1024 * 1024 * 1024,
                podCPU: 500, podMem: 512 * 1024 * 1024,
                rolloutActive: true,
            },
            wantGoNoGo: false,
            wantStatuses: map[string]CheckStatus{
                "image_exists":   StatusPass,
                "resource_quota": StatusPass,
                "pdb_check":      StatusPass,
                "rollout_status": StatusFail,
            },
        },
        {
            name: "multiple failures compound correctly",
            spec: baseSpec,
            ecr: &mockECR{images: map[string]bool{}}, // missing image
            k8s: &mockK8s{
                availCPU: 100, availMem: 100, // insufficient
                podCPU: 500, podMem: 512 * 1024 * 1024,
                pdbBlocked:    true,
                pdbReason:     "minAvailable=3",
                rolloutActive: true,
            },
            wantGoNoGo:     false,
            wantCheckCount: 4,
            wantStatuses: map[string]CheckStatus{
                "image_exists":   StatusFail,
                "resource_quota": StatusFail,
                "pdb_check":      StatusFail,
                "rollout_status": StatusFail,
            },
        },
        {
            name: "quota API unavailable — warn, not fail",
            spec: baseSpec,
            ecr: &mockECR{
                images: map[string]bool{
                    "registry.novamart.com/order-service:v1.2.3": true,
                },
            },
            k8s: &mockK8s{
                quotaErr: fmt.Errorf("namespace has no ResourceQuota"),
                podCPU: 500, podMem: 512 * 1024 * 1024,
            },
            wantGoNoGo: true, // warn doesn't block
            wantStatuses: map[string]CheckStatus{
                "image_exists":   StatusPass,
                "resource_quota": StatusWarn,
                "pdb_check":      StatusPass,
                "rollout_status": StatusPass,
            },
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            v := NewPreCheckValidator(tt.ecr, tt.k8s, logger)
            result, err := v.Validate(context.Background(), tt.spec)
            require.NoError(t, err)

            assert.Equal(t, tt.wantGoNoGo, result.GoNoGo, "GoNoGo mismatch")

            if tt.wantCheckCount > 0 {
                assert.Len(t, result.Checks, tt.wantCheckCount)
            }

            // Verify individual check statuses
            checkMap := make(map[string]CheckResult)
            for _, c := range result.Checks {
                checkMap[c.Name] = c
            }

            for checkName, wantStatus := range tt.wantStatuses {
                c, ok := checkMap[checkName]
                require.True(t, ok, "check %q not found in results", checkName)
                assert.Equal(t, wantStatus, c.Status,
                    "check %q: want %s, got %s: %s",
                    checkName, wantStatus, c.Status, c.Message)
            }
        })
    }
}

// ── parseImage Tests ────────────────────────────────────────────────────────

func TestParseImage(t *testing.T) {
    tests := []struct {
        input   string
        repo    string
        tag     string
        wantErr bool
    }{
        {
            input: "registry.novamart.com/order-service:v1.2.3",
            repo:  "registry.novamart.com/order-service",
            tag:   "v1.2.3",
        },
        {
            input: "order-service:latest",
            repo:  "order-service",
            tag:   "latest",
        },
        {
            input: "registry.novamart.com/order-service",
            repo:  "registry.novamart.com/order-service",
            tag:   "latest", // implicit
        },
        {
            input: "registry.novamart.com:5000/order-service:v2.0.0",
            repo:  "registry.novamart.com:5000/order-service",
            tag:   "v2.0.0",
        },
        {
            input: "registry.novamart.com/order-service@sha256:abc123def456",
            repo:  "registry.novamart.com/order-service",
            tag:   "sha256:abc123def456",
        },
        {
            input:   ":bad",
            wantErr: true,
        },
        {
            input:   "repo:",
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.input, func(t *testing.T) {
            repo, tag, err := parseImage(tt.input)
            if tt.wantErr {
                assert.Error(t, err)
                return
            }
            require.NoError(t, err)
            assert.Equal(t, tt.repo, repo)
            assert.Equal(t, tt.tag, tag)
        })
    }
}

// ── Validation Input Tests ──────────────────────────────────────────────────

func TestValidate_InvalidInput(t *testing.T) {
    logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelWarn}))
    v := NewPreCheckValidator(&mockECR{}, &mockK8s{}, logger)

    tests := []struct {
        name string
        spec DeploymentSpec
    }{
        {
            name: "empty name",
            spec: DeploymentSpec{Namespace: "prod", Image: "img:v1"},
        },
        {
            name: "empty namespace",
            spec: DeploymentSpec{Name: "svc", Image: "img:v1"},
        },
        {
            name: "empty image",
            spec: DeploymentSpec{Name: "svc", Namespace: "prod"},
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := v.Validate(context.Background(), tt.spec)
            assert.Error(t, err)
        })
    }
}
```

---

# Q4 — Concurrency Trace: Buffered Channel Execution

## The Setup

```go
ch := make(chan int, 2)    // buffered: capacity 2
done := make(chan bool)     // unbuffered: synchronization signal
```

Two channels:
- `ch` can hold 2 values before a send blocks
- `done` is a synchronization gate — send blocks until receiver is ready

## Detailed Execution Trace

I'll track the channel buffer state at each step:

```
Legend:
  ch buffer: [slot0, slot1] (capacity 2)
  M = main goroutine
  G = consumer goroutine

─── STEP 1: G spawns, blocks on range ch (buffer empty) ───────────────────
    M: spawns G
    G: enters `for v := range ch` → blocks (ch is empty)
    ch: []

─── STEP 2: M sends 0 — buffer has room, does NOT block ──────────────────
    M: ch <- 0          → buffer absorbs it, NO BLOCK
    M: prints "sent: 0"
    ch: [0]
    G: may or may not wake yet (scheduler-dependent)

─── STEP 3: M sends 1 — buffer has room, does NOT block ──────────────────
    M: ch <- 1          → buffer absorbs it, NO BLOCK
    M: prints "sent: 1"
    ch: [0, 1]          ← BUFFER FULL

─── STEP 4: M sends 2 — buffer FULL, M BLOCKS ────────────────────────────
    M: ch <- 2          → BLOCKS (buffer is [0, 1], no room)
    
    G: NOW MUST run to drain the buffer
    G: receives 0 from ch
    G: prints "received: 0"
    G: sleeps 100ms
    ch: [1]             ← slot freed, M can now send

    M: unblocks, 2 enters buffer
    M: prints "sent: 2"
    ch: [1, 2]          ← BUFFER FULL AGAIN

─── STEP 5: M sends 3 — buffer FULL, M BLOCKS AGAIN ─────────────────────
    M: ch <- 3          → BLOCKS

    G: receives 1 from ch
    G: prints "received: 1"
    G: sleeps 100ms
    ch: [2]             ← slot freed

    M: unblocks, 3 enters buffer
    M: prints "sent: 3"
    ch: [2, 3]

─── STEP 6: M sends 4 — buffer FULL, M BLOCKS AGAIN ─────────────────────
    M: ch <- 4          → BLOCKS

    G: receives 2 from ch
    G: prints "received: 2"
    G: sleeps 100ms
    ch: [3]

    M: unblocks, 4 enters buffer
    M: prints "sent: 4"
    ch: [3, 4]

─── STEP 7: M exits loop, closes ch ──────────────────────────────────────
    M: close(ch)
    M: <-done           → BLOCKS waiting for G to finish

    G: receives 3, prints "received: 3", sleeps 100ms
    G: receives 4, prints "received: 4", sleeps 100ms
    G: range ch detects close, exits loop
    G: done <- true

    M: receives true from done, unblocks
    M: exits
```

## Expected Output

```
sent: 0
sent: 1
received: 0
sent: 2
received: 1
sent: 3
received: 2
sent: 4
received: 3
received: 4
```

**Note:** The exact interleaving of "sent" and "received" lines varies slightly by scheduler timing, but the **blocking points are deterministic:**

| Send | Blocks? | Why |
|------|---------|-----|
| `ch <- 0` | No | Buffer empty (0/2) |
| `ch <- 1` | No | Buffer has room (1/2) |
| `ch <- 2` | **YES** | Buffer full (2/2). Blocks until G receives one value. |
| `ch <- 3` | **YES** | Buffer full again after previous send filled it. |
| `ch <- 4` | **YES** | Same pattern. |

## Why Buffer Size 2 Is Relevant

The buffer creates a **2-element head start** for the producer. The first 2 sends are non-blocking, which means the main goroutine can race ahead of the consumer. After that, the system becomes **lock-step**: the producer can only send when the consumer drains one slot.

**Throughput implication:** With buffer size 2, the producer can "pre-load" work while the consumer processes. This hides latency when production is bursty. With 5 items and 100ms per consumption, the total wall-clock time is approximately:

```
5 items × 100ms = ~500ms
minus the 2-item head start ≈ ~300ms effective blocking time for producer
Total ≈ ~500ms (dominated by consumer)
```

---

## What Changes with `make(chan int)` (Unbuffered)?

```
Every single send blocks until the consumer is ready to receive.

─── ch <- 0: M blocks → G receives 0 → M unblocks, prints "sent: 0"
─── ch <- 1: M blocks → G finishes sleep(100ms), receives 1 → M unblocks
─── ch <- 2: same pattern
─── ch <- 3: same pattern
─── ch <- 4: same pattern
```

**Key differences:**

| Aspect | Buffered (2) | Unbuffered |
|--------|-------------|------------|
| First 2 sends | Non-blocking | Each blocks |
| Output interleaving | "sent 0, sent 1" appear before any "received" | "received: 0" appears **before** "sent: 0" (or interleaved 1:1) |
| Wall-clock time | ~500ms | ~500ms (same — consumer-dominated) |
| Coupling | Loosely coupled (2-item buffer) | Fully synchronous rendezvous |

**The output with unbuffered channel:**
```
received: 0
sent: 0
received: 1
sent: 1
received: 2
sent: 2
received: 3
sent: 3
received: 4
sent: 4
```

Each "received" appears before or simultaneously with its corresponding "sent" — because the send only completes **when the receive happens**. They are a synchronized handshake.

---

## What If the Consumer Panics on `i == 3`?

```go
go func() {
    for v := range ch {
        if v == 3 {
            panic("boom")  // hypothetical
        }
        fmt.Println("received:", v)
        time.Sleep(100 * time.Millisecond)
    }
    done <- true
}()
```

**Execution trace:**

```
G receives 0, 1, 2 normally.
G receives 3 → panic("boom")

The panic is UNRECOVERED (no defer/recover in the goroutine).
```

**What happens to the main goroutine:**

### Phase 1: Immediate effect of the panic

In Go, an **unrecovered panic in any goroutine crashes the entire program**. The runtime prints:

```
goroutine 6 [running]:
main.main.func1()
    panic: boom
    ...
exit status 2
```

**The main goroutine does NOT deadlock.** It doesn't get a chance to. The panic tears down the entire process.

### But what if the panic WERE recovered?

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("consumer panicked: %v", r)
        }
        // NOTE: done <- true is NEVER reached because
        // the for loop exited via panic, not via close(ch)
    }()
    for v := range ch {
        if v == 3 { panic("boom") }
        // ...
    }
    done <- true  // unreachable after panic
}()
```

Now the situation is different and **much worse**:

```
1. G receives 0, 1, 2 normally
2. G receives 3 → panic → recover catches it
3. G's deferred function runs, logs the error
4. G exits (the goroutine is done)
5. done <- true was NEVER sent

Meanwhile, M is still trying to send:
  - If M is blocked on ch <- 4: ch has no receiver anymore.
    M blocks forever on the send.
    
  - Eventually M would need to reach <-done, but it's stuck on ch <- 4.

RESULT: DEADLOCK

fatal error: all goroutines are asleep - deadlock!
goroutine 1 [chan send]:
    ch <- 4
```

The Go runtime detects that **all goroutines are blocked** and terminates with a deadlock error.

### Summary of Panic Scenarios

| Scenario | Outcome |
|----------|---------|
| Panic, no recover | **Entire program crashes immediately** — exit status 2 |
| Panic + recover, no cleanup | **Deadlock** — main blocks on `ch <- 4` forever, then `<-done` is unreachable. Runtime detects deadlock. |
| Panic + recover + `close(done)` in defer | Main's `<-done` unblocks, but `ch <- 4` may still deadlock if main hasn't sent it yet. Need to drain or close `ch`. |

**The correct defensive pattern:**

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("consumer panicked: %v", r)
        }
        // ALWAYS signal done, even on panic
        done <- true
    }()
    for v := range ch {
        // ...
    }
}()
```

But even this doesn't fully fix it — the main goroutine may still be blocked on `ch <- 4` with no receiver. The real fix requires either:
1. A **buffered `ch`** large enough to absorb remaining sends, OR
2. The main goroutine using a **`select` with a context/timeout** on sends, OR
3. A **separate drainer goroutine** that empties `ch` on error

This is why production Go code almost always uses `context.Context` for cancellation propagation rather than relying on channel close semantics alone.


