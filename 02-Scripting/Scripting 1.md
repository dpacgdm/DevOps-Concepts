# Phase 9, Lesson 1: Bash for DevOps

---

## Why This Lesson Exists

You've built NovaMart. You've operated it. You've written Terraform, Helm values, Jenkinsfiles, Kustomize overlays. But here's the truth: **every single day as a Senior DevOps engineer, you write shell scripts.** Entrypoint scripts. Wrapper scripts. Automation glue. Health checks. Incident response tools. Migration helpers.

Bad bash kills production. Unquoted variables delete directories. Missing error handling lets failures cascade silently. Scripts without logging become black boxes during incidents.

This lesson turns you from "I can write bash" to "I write production-grade bash that other engineers trust with their infrastructure."

---

## 1. THE NON-NEGOTIABLE HEADER

Every production script starts with this. No exceptions.

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

Let's dissect each piece — what it does, and **how it breaks when missing:**

### `set -e` (errexit)

**What it does:** Exit immediately if any command returns non-zero.

**Without it:**
```bash
#!/bin/bash
rm -rf /app/deploy/$ENVIRONMENT/old    # $ENVIRONMENT is empty → rm -rf /app/deploy//old
cp -r /tmp/build/* /app/deploy/        # This STILL RUNS even if rm failed
systemctl restart app                   # This STILL RUNS even if cp failed
# Congratulations, you deployed garbage and restarted the service
```

**With `set -e`:** Script stops at the first failure. You know exactly where it broke.

**The gotchas of `set -e`** (this is where people get burned):

```bash
set -e

# GOTCHA 1: Pipes — only last command's exit code matters by default
curl http://api.internal/health | grep "ok"
# If curl fails but grep succeeds on empty output → no error!
# FIX: set -o pipefail (covered below)

# GOTCHA 2: Commands in conditionals are EXEMPT from -e
if grep -q "error" /var/log/app.log; then
    echo "Found errors"
fi
# grep returning 1 (not found) does NOT exit the script. Correct behavior.

# GOTCHA 3: Commands in && or || chains
false && echo "this won't print"
echo "script continues here"  # This STILL RUNS because && "handled" the error
# The && makes bash consider the error "handled"

# GOTCHA 4: Command substitution
result=$(failing_command)    # set -e DOES catch this — script exits
echo "result: $(failing_command)"  # set -e DOES catch this too — script exits

# GOTCHA 5: Arithmetic
count=0
(( count++ ))  # Returns exit code 1 because result is 0 (falsy)!
               # With set -e, THIS KILLS YOUR SCRIPT
# FIX: (( count++ )) || true
# OR:  count=$(( count + 1 ))
```

### `set -u` (nounset)

**What it does:** Treat unset variables as errors.

**Without it — the infamous production disaster:**
```bash
#!/bin/bash
DEPLOY_DIR="/app/releases"
# Typo: DELOY_DIR instead of DEPLOY_DIR
rm -rf "${DELOY_DIR}/"*
# $DELOY_DIR is unset → expands to empty string
# rm -rf "/"*  ← YOU JUST DELETED EVERYTHING
```

**With `set -u`:** Script immediately errors: `bash: DELOY_DIR: unbound variable`

**Gotchas of `set -u`:**

```bash
set -u

# GOTCHA 1: Checking if variable is set
if [ -z "$MAYBE_SET" ]; then  # CRASHES — variable is unset
    echo "not set"
fi
# FIX: Use default value syntax
if [ -z "${MAYBE_SET:-}" ]; then  # ${var:-} = empty string if unset
    echo "not set"
fi

# GOTCHA 2: Positional parameters
echo "Arg 1: $1"  # CRASHES if no arguments passed
# FIX:
echo "Arg 1: ${1:-default_value}"

# GOTCHA 3: Empty arrays
set -u
arr=()
echo "${arr[@]}"  # In Bash < 4.4: CRASHES (unbound variable)
                   # In Bash >= 4.4: Fixed, works correctly
# FIX for older bash:
echo "${arr[@]+"${arr[@]}"}"  # Ugly but safe
```

### `set -o pipefail`

**What it does:** Pipeline returns the exit code of the **first** failing command, not just the last.

```bash
# WITHOUT pipefail:
curl http://failing-service/data | jq '.results'
echo $?  # Returns 0 (jq succeeded on empty input)
# You think everything is fine. It's not.

# WITH pipefail:
set -o pipefail
curl http://failing-service/data | jq '.results'
echo $?  # Returns curl's non-zero exit code
# Script exits (because set -e) — you KNOW something broke
```

### `IFS=$'\n\t'`

**What it does:** Changes the Internal Field Separator from `<space><tab><newline>` to just `<newline><tab>`.

**Why:** Prevents word splitting on spaces in filenames and data:

```bash
# DEFAULT IFS (space is separator):
files="my file.txt other file.txt"
for f in $files; do
    echo "Processing: $f"
done
# Output: Processing: my
#         Processing: file.txt
#         Processing: other
#         Processing: file.txt
# WRONG — split on spaces

# WITH IFS=$'\n\t':
# Now only splits on newlines and tabs — spaces are preserved
```

### The Complete Header Template

```bash
#!/usr/bin/env bash
#
# Script: deploy-wrapper.sh
# Purpose: Wraps ArgoCD deployment with pre/post validation
# Author: Platform Engineering
# Last Modified: 2024-01-15
# Usage: ./deploy-wrapper.sh --env production --service payment-service --version sha-abc123
#
set -euo pipefail
IFS=$'\n\t'

# Script directory (resolve symlinks)
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"

# Constants
readonly LOG_FILE="/var/log/novamart/${SCRIPT_NAME%.sh}.log"
readonly LOCK_FILE="/tmp/${SCRIPT_NAME%.sh}.lock"
readonly DRY_RUN="${DRY_RUN:-false}"
```

**Why `BASH_SOURCE[0]` instead of `$0`?**
- `$0` changes when script is sourced vs executed
- `BASH_SOURCE[0]` always points to the current script file
- Critical for scripts that `source` other scripts

**Why `readonly`?**
- Constants should never be reassigned
- If something tries to change them, bash errors immediately
- Self-documenting: reader knows these don't change

---

## 2. LOGGING FRAMEWORK

Production scripts without logging are undebuggable. Period.

```bash
# ─── Logging ───────────────────────────────────────────────
readonly LOG_LEVEL="${LOG_LEVEL:-INFO}"

_log_levels() {
    case "$1" in
        DEBUG) echo 0 ;;
        INFO)  echo 1 ;;
        WARN)  echo 2 ;;
        ERROR) echo 3 ;;
        FATAL) echo 4 ;;
        *)     echo 1 ;;
    esac
}

log() {
    local level="$1"
    shift
    local message="$*"
    local current_level
    current_level=$(_log_levels "$LOG_LEVEL")
    local msg_level
    msg_level=$(_log_levels "$level")

    if (( msg_level >= current_level )); then
        local timestamp
        timestamp="$(date -u '+%Y-%m-%dT%H:%M:%SZ')"  # Always UTC, ISO 8601
        local caller="${FUNCNAME[1]:-main}"
        local line="${BASH_LINENO[0]:-0}"

        # Color output for terminal, plain for log file
        local color_reset='\033[0m'
        local color
        case "$level" in
            DEBUG) color='\033[0;36m' ;;  # Cyan
            INFO)  color='\033[0;32m' ;;  # Green
            WARN)  color='\033[0;33m' ;;  # Yellow
            ERROR) color='\033[0;31m' ;;  # Red
            FATAL) color='\033[1;31m' ;;  # Bold Red
        esac

        # To stderr (visible in terminal) — with color
        printf "${color}[%s] %-5s [%s:%s] %s${color_reset}\n" \
            "$timestamp" "$level" "$caller" "$line" "$message" >&2

        # To log file (if writable) — no color
        if [[ -w "$(dirname "$LOG_FILE")" ]]; then
            printf "[%s] %-5s [%s:%s] %s\n" \
                "$timestamp" "$level" "$caller" "$line" "$message" >> "$LOG_FILE"
        fi
    fi
}

# Convenience functions
debug() { log "DEBUG" "$@"; }
info()  { log "INFO" "$@"; }
warn()  { log "WARN" "$@"; }
error() { log "ERROR" "$@"; }
fatal() { log "FATAL" "$@"; exit 1; }
```

**Output example:**
```
[2024-01-15T14:32:07Z] INFO  [deploy:45] Starting deployment of payment-service sha-abc123
[2024-01-15T14:32:08Z] WARN  [validate_pdb:112] PDB budget only allows 1 unavailable pod
[2024-01-15T14:32:15Z] ERROR [wait_healthy:203] Deployment not healthy after 120s
```

**Why this matters in production:**
- `FUNCNAME[1]` gives you which function logged it — instant debugging
- `BASH_LINENO[0]` gives line number — go straight to the problem
- UTC timestamps — no timezone confusion in multi-region
- stderr for terminal output — stdout stays clean for piping
- Log file for post-incident analysis

**How this breaks:**
```
# FAILURE 1: Log directory doesn't exist
mkdir -p "$(dirname "$LOG_FILE")"  # Add this to script init

# FAILURE 2: Log file grows forever
# Fix: logrotate config or max size check
# Or use systemd journal: logger -t "$SCRIPT_NAME" "message"

# FAILURE 3: Logging to NFS mount that's down
# Fix: The -w check handles this — falls back to stderr only
```

---

## 3. TRAP — CLEANUP AND SIGNAL HANDLING

`trap` is how bash scripts handle cleanup, signals, and unexpected exits. Without it, your scripts leak temp files, hold locks forever, and leave infrastructure in partial states.

```bash
# ─── Cleanup & Signal Handling ─────────────────────────────
cleanup() {
    local exit_code=$?
    
    # Remove lock file
    if [[ -f "$LOCK_FILE" ]]; then
        rm -f "$LOCK_FILE"
        debug "Removed lock file: $LOCK_FILE"
    fi
    
    # Remove temp directory
    if [[ -d "${TMPDIR:-}" ]]; then
        rm -rf "$TMPDIR"
        debug "Removed temp directory: $TMPDIR"
    fi
    
    # Report final status
    if (( exit_code == 0 )); then
        info "Script completed successfully"
    else
        error "Script failed with exit code: $exit_code"
        # Notify on failure
        if [[ "${SLACK_WEBHOOK:-}" ]]; then
            notify_slack "FAILED" "Script ${SCRIPT_NAME} failed with exit code ${exit_code}"
        fi
    fi
    
    exit "$exit_code"
}

on_error() {
    local exit_code=$?
    local line_no=$1
    error "Error on line ${line_no}: command exited with status ${exit_code}"
    # Stack trace
    local i=0
    while caller $i > /dev/null 2>&1; do
        local frame
        frame=$(caller $i)
        error "  Frame $i: $frame"
        (( i++ ))
    done
}

# Register traps
trap cleanup EXIT           # Always runs on exit (normal or error)
trap 'on_error $LINENO' ERR # Runs on error (before EXIT trap)
trap 'fatal "Received SIGINT (Ctrl+C)"' INT
trap 'fatal "Received SIGTERM"' TERM

# Create temp directory safely
TMPDIR="$(mktemp -d "/tmp/${SCRIPT_NAME%.sh}.XXXXXX")"
```

**Trap execution order:**
```
Error occurs on line 47
  → ERR trap fires: on_error 47 (logs error + stack trace)
  → EXIT trap fires: cleanup (removes temp files, removes lock, notifies Slack)
  → Script exits with original error code
```

**Critical trap gotchas:**

```bash
# GOTCHA 1: Traps in subshells DON'T inherit parent traps
(
    # This subshell has NO traps — cleanup won't run here
    some_dangerous_command
)

# GOTCHA 2: trap EXIT fires even on normal exit
# That's usually what you want — cleanup should always happen

# GOTCHA 3: ERR trap doesn't fire in functions called from conditionals
set -e
trap 'echo "error!"' ERR
failing_function() { return 1; }

if failing_function; then  # ERR trap does NOT fire here
    echo "ok"
fi
# This is correct behavior — conditionals suppress ERR

# GOTCHA 4: Nested traps — later trap REPLACES earlier one
trap 'echo "first"' EXIT
trap 'echo "second"' EXIT  # This REPLACES, doesn't chain
# Only "second" prints
# FIX: Chain manually:
trap 'first_cleanup; second_cleanup' EXIT

# GOTCHA 5: trap inside function vs global
# trap set inside a function is GLOBAL — it affects the whole script
```

---

## 4. ARGUMENT PARSING

Two patterns: `getopts` (built-in, POSIX) and manual parsing (for long options).

### Pattern 1: Manual Parsing (Production Standard)

```bash
# ─── Argument Parsing ──────────────────────────────────────
usage() {
    cat <<EOF
Usage: ${SCRIPT_NAME} [OPTIONS]

Wraps ArgoCD deployment with pre/post validation and safety checks.

Required:
    -e, --env ENV           Target environment (dev|staging|production)
    -s, --service SERVICE   Service name (e.g., payment-service)
    -v, --version VERSION   Image version (e.g., sha-abc123)

Optional:
    -d, --dry-run           Show what would be done without executing
    -t, --timeout SECONDS   Deployment timeout (default: 300)
    -f, --force             Skip PDB and approval checks
    --skip-canary           Deploy directly without canary (DANGEROUS)
    --no-slack              Suppress Slack notifications
    -h, --help              Show this help message

Environment Variables:
    LOG_LEVEL               Logging verbosity (DEBUG|INFO|WARN|ERROR) [default: INFO]
    SLACK_WEBHOOK           Slack webhook URL for notifications
    DRY_RUN                 Same as --dry-run flag

Examples:
    ${SCRIPT_NAME} --env production --service payment-service --version sha-abc123
    ${SCRIPT_NAME} -e staging -s cart-service -v sha-def456 --dry-run
    DRY_RUN=true ${SCRIPT_NAME} -e production -s order-service -v sha-789

EOF
    exit "${1:-0}"
}

# Defaults
ENVIRONMENT=""
SERVICE=""
VERSION=""
TIMEOUT=300
FORCE=false
SKIP_CANARY=false
NO_SLACK=false

parse_args() {
    if [[ $# -eq 0 ]]; then
        usage 1
    fi

    while [[ $# -gt 0 ]]; do
        case "$1" in
            -e|--env)
                ENVIRONMENT="${2:?ERROR: --env requires a value}"
                shift 2
                ;;
            -s|--service)
                SERVICE="${2:?ERROR: --service requires a value}"
                shift 2
                ;;
            -v|--version)
                VERSION="${2:?ERROR: --version requires a value}"
                shift 2
                ;;
            -t|--timeout)
                TIMEOUT="${2:?ERROR: --timeout requires a value}"
                shift 2
                ;;
            -d|--dry-run)
                DRY_RUN=true
                shift
                ;;
            -f|--force)
                FORCE=true
                shift
                ;;
            --skip-canary)
                SKIP_CANARY=true
                shift
                ;;
            --no-slack)
                NO_SLACK=true
                shift
                ;;
            -h|--help)
                usage 0
                ;;
            --)
                shift
                break
                ;;
            -*)
                error "Unknown option: $1"
                usage 1
                ;;
            *)
                error "Unexpected argument: $1"
                usage 1
                ;;
        esac
    done
}

validate_args() {
    local errors=0

    if [[ -z "$ENVIRONMENT" ]]; then
        error "Missing required: --env"
        (( errors++ ))
    elif [[ ! "$ENVIRONMENT" =~ ^(dev|staging|production)$ ]]; then
        error "Invalid environment: $ENVIRONMENT (must be dev|staging|production)"
        (( errors++ ))
    fi

    if [[ -z "$SERVICE" ]]; then
        error "Missing required: --service"
        (( errors++ ))
    fi

    if [[ -z "$VERSION" ]]; then
        error "Missing required: --version"
        (( errors++ ))
    elif [[ ! "$VERSION" =~ ^sha-[a-f0-9]{7,40}$ ]]; then
        error "Invalid version format: $VERSION (expected sha-<hash>)"
        (( errors++ ))
    fi

    if ! [[ "$TIMEOUT" =~ ^[0-9]+$ ]] || (( TIMEOUT < 30 || TIMEOUT > 1800 )); then
        error "Invalid timeout: $TIMEOUT (must be 30-1800 seconds)"
        (( errors++ ))
    fi

    if [[ "$ENVIRONMENT" == "production" && "$SKIP_CANARY" == true && "$FORCE" != true ]]; then
        error "Cannot skip canary in production without --force"
        (( errors++ ))
    fi

    if (( errors > 0 )); then
        fatal "$errors validation error(s). Aborting."
    fi
}
```

**Why `${2:?ERROR message}` instead of just `$2`?**
- If `$2` is empty or missing, bash prints the error message and exits
- Concise one-liner that handles missing argument values

**Why validate in a separate function?**
- Collect ALL errors before aborting (UX: show all problems, not one at a time)
- Validation logic can be tested independently

---

## 5. LOCK FILES (flock)

Prevents concurrent execution — critical for deployment scripts, backup scripts, any script that shouldn't run twice simultaneously.

```bash
# ─── Lock Management ──────────────────────────────────────
acquire_lock() {
    local lock_fd=200  # File descriptor number (arbitrary, > 2)

    # Create lock file directory
    mkdir -p "$(dirname "$LOCK_FILE")"
    
    # Open lock file on file descriptor
    eval "exec ${lock_fd}>\"${LOCK_FILE}\""
    
    # Try to acquire exclusive lock (non-blocking)
    if ! flock -n "$lock_fd"; then
        local existing_pid
        existing_pid=$(cat "$LOCK_FILE" 2>/dev/null || echo "unknown")
        fatal "Another instance is running (PID: ${existing_pid}). Lock file: ${LOCK_FILE}"
    fi
    
    # Write our PID to lock file (for debugging)
    echo $$ > "$LOCK_FILE"
    debug "Lock acquired: $LOCK_FILE (PID: $$)"
}
```

**Why `flock` and not just checking if the file exists?**

```bash
# WRONG — Race condition:
if [[ -f "$LOCK_FILE" ]]; then
    echo "Already running"
    exit 1
fi
echo $$ > "$LOCK_FILE"
# Two scripts can BOTH pass the -f check before either writes the file
# This is a TOCTOU (Time of Check to Time of Use) race condition

# CORRECT — flock is atomic:
flock -n 200  # Kernel-level atomic lock — no race condition
```

**How locks break:**

```
FAILURE 1: Script killed with SIGKILL (-9)
  → trap doesn't run → lock file stays → next run thinks something is running
  FIX: cleanup trap handles normal exit. For SIGKILL, lock file contains PID.
       Check if that PID is still alive:

    if [[ -f "$LOCK_FILE" ]]; then
        old_pid=$(cat "$LOCK_FILE")
        if kill -0 "$old_pid" 2>/dev/null; then
            fatal "Process $old_pid still running"
        else
            warn "Stale lock file (PID $old_pid dead). Removing."
            rm -f "$LOCK_FILE"
        fi
    fi

FAILURE 2: Lock file on NFS
  → flock doesn't work reliably on NFS
  FIX: Use local filesystem (/tmp or /var/run) for lock files

FAILURE 3: Container restart
  → Lock file in /tmp is gone (tmpfs) — not a problem
  → Lock file in persistent volume — stale lock on restart
  FIX: Always check PID liveness, not just file existence
```

---

## 6. RETRY LOGIC WITH EXPONENTIAL BACKOFF

Every script that calls external services needs retry logic. APIs fail. DNS hiccups. Connections timeout. If your script can't retry, it can't be trusted.

```bash
# ─── Retry Logic ───────────────────────────────────────────
retry() {
    local max_attempts="${1:?Usage: retry <max_attempts> <delay> <command...>}"
    local delay="${2:?Usage: retry <max_attempts> <delay> <command...>}"
    shift 2
    local cmd=("$@")
    
    local attempt=1
    local exit_code=0
    
    while (( attempt <= max_attempts )); do
        if "${cmd[@]}"; then
            return 0
        fi
        exit_code=$?
        
        if (( attempt == max_attempts )); then
            error "Command failed after ${max_attempts} attempts: ${cmd[*]}"
            return "$exit_code"
        fi
        
        local backoff=$(( delay * (2 ** (attempt - 1)) ))
        # Add jitter (±25%) to prevent thundering herd
        local jitter=$(( (RANDOM % (backoff / 2 + 1)) - (backoff / 4) ))
        local sleep_time=$(( backoff + jitter ))
        (( sleep_time < 1 )) && sleep_time=1
        
        warn "Attempt ${attempt}/${max_attempts} failed (exit: ${exit_code}). Retrying in ${sleep_time}s..."
        sleep "$sleep_time"
        
        (( attempt++ ))
    done
}

# Usage examples:
retry 5 2 curl -sf --max-time 10 "http://argocd-server/api/v1/applications/${SERVICE}"
retry 3 5 kubectl rollout status "deployment/${SERVICE}" -n "$NAMESPACE" --timeout=60s
retry 4 3 aws ecr describe-images --repository-name "$SERVICE" --image-ids "imageTag=${VERSION}"
```

**Why jitter?**
- Without jitter: 100 scripts all retry at exactly the same time → thundering herd → service dies again
- With jitter: retries spread across time window → service has time to recover

**Backoff progression (base delay = 2s):**
```
Attempt 1: fail → wait ~2s  (2 * 2^0 = 2, ±jitter)
Attempt 2: fail → wait ~4s  (2 * 2^1 = 4, ±jitter)
Attempt 3: fail → wait ~8s  (2 * 2^2 = 8, ±jitter)
Attempt 4: fail → wait ~16s (2 * 2^3 = 16, ±jitter)
Attempt 5: fail → GIVE UP
```

---

## 7. DRY-RUN MODE

**Every destructive script needs a dry-run mode.** Non-negotiable. This is the difference between "oops I deleted production" and "let me verify first."

```bash
# ─── Dry-Run Execution ────────────────────────────────────
run() {
    if [[ "$DRY_RUN" == true ]]; then
        info "[DRY-RUN] Would execute: $*"
        return 0
    fi
    
    debug "Executing: $*"
    "$@"
}

# Usage:
run kubectl set image "deployment/${SERVICE}" "${SERVICE}=${IMAGE}:${VERSION}" -n "$NAMESPACE"
run argocd app sync "$SERVICE" --prune
run aws rds create-db-snapshot --db-instance-identifier "$DB_INSTANCE" --db-snapshot-identifier "$SNAPSHOT_ID"
```

**Pattern: Separate read-only commands from destructive ones:**
```bash
# Read-only — always execute (even in dry-run)
current_version=$(kubectl get deployment "$SERVICE" -n "$NAMESPACE" \
    -o jsonpath='{.spec.template.spec.containers[0].image}')
info "Current version: $current_version"

# Destructive — wrap in run()
run kubectl set image "deployment/${SERVICE}" "${SERVICE}=${REGISTRY}/${SERVICE}:${VERSION}" -n "$NAMESPACE"
```

---

## 8. TEXT PROCESSING — awk, sed, jq, yq

These are your daily-use power tools. Let's cover each with production-realistic examples.

### awk

```bash
# ─── awk: Field Extraction & Pattern Processing ───────────

# Basic: Extract specific fields
kubectl get pods -n production -o wide | awk 'NR>1 {print $1, $3, $7}'
# Skip header (NR>1), print pod name, status, node

# Filter + extract: Find pods not Running
kubectl get pods -n production | awk '$3 != "Running" && NR>1 {print $1, $3}'

# Sum a column: Total CPU requests
kubectl top pods -n production | awk 'NR>1 {sum += $2} END {print "Total CPU:", sum "m"}'

# Multi-field processing: Parse access logs
awk '{
    status = $9
    latency = $NF
    if (status >= 500) errors++
    if (latency > 1.0) slow++
    total++
}
END {
    printf "Total: %d, Errors: %d (%.1f%%), Slow: %d (%.1f%%)\n",
        total, errors, (errors/total)*100, slow, (slow/total)*100
}' /var/log/nginx/access.log

# Field separator: Parse /etc/passwd
awk -F: '$3 >= 1000 && $7 !~ /nologin/ {print $1, $6}' /etc/passwd
# Users with UID >= 1000 and valid shell: print username and home dir

# Associative arrays: Count HTTP status codes
awk '{count[$9]++} END {for (code in count) print code, count[code]}' \
    /var/log/nginx/access.log | sort -rn -k2

# BEGIN/END blocks: Add header and footer
kubectl get nodes -o custom-columns='NAME:.metadata.name,CPU:.status.capacity.cpu,MEM:.status.capacity.memory' | \
    awk 'BEGIN {print "=== NovaMart Cluster Capacity ==="} {print} END {print "=== End ==="}'
```

### sed

```bash
# ─── sed: Stream Editing ──────────────────────────────────

# In-place edit (THE production gotcha)
sed -i 's/old/new/g' file.txt           # Linux: works
sed -i '' 's/old/new/g' file.txt        # macOS: needs empty string for backup
sed -i.bak 's/old/new/g' file.txt       # Both: create backup — ALWAYS do this in production

# Update image tag in Kustomize
sed -i "s|newTag: sha-[a-f0-9]*|newTag: ${VERSION}|" \
    "kustomize/overlays/${ENVIRONMENT}/kustomization.yaml"

# Delete lines matching pattern
sed -i '/^#.*TODO/d' config.yaml         # Remove TODO comments

# Insert line after match
sed -i '/^\[server\]/a retry_limit = 5' config.ini  # Add after [server] section

# Multi-line replacement (use different delimiter for URLs)
sed -i "s|image: .*/${SERVICE}:.*|image: ${REGISTRY}/${SERVICE}:${VERSION}|" \
    deployment.yaml

# Extract between markers
sed -n '/BEGIN CERT/,/END CERT/p' fullchain.pem

# Production gotcha: sed regex vs grep regex
# sed uses BRE by default: \+, \?, \{n\} need escaping
# sed -E uses ERE: +, ?, {n} work without escaping
sed -E "s/replicas: [0-9]+/replicas: ${REPLICAS}/" deployment.yaml
```

**sed failure modes:**
```bash
# FAILURE 1: Variable contains / → breaks s/old/new/
VERSION="sha/abc"  # Contains /
sed "s/tag: .*/tag: ${VERSION}/"  # BREAKS — / in value conflicts with delimiter
# FIX: Use different delimiter: sed "s|tag: .*|tag: ${VERSION}|"

# FAILURE 2: In-place edit of symlink
sed -i 's/old/new/' /etc/some-symlink  # Replaces symlink with regular file!
# FIX: Resolve symlink first, or edit the target directly

# FAILURE 3: Empty file after sed error
sed -i '/pattern/d' important-file.conf  # If sed crashes mid-write → empty file
# FIX: sed -i.bak (keep backup), or write to temp then mv (atomic)
```

### jq — JSON Processing

```bash
# ─── jq: JSON Swiss Army Knife ────────────────────────────

# Basic field extraction
kubectl get pod "$POD" -o json | jq '.status.phase'

# Nested access
kubectl get pod "$POD" -o json | jq '.status.containerStatuses[0].restartCount'

# Filter and select
kubectl get pods -o json | jq -r '
    .items[] 
    | select(.status.phase != "Running") 
    | "\(.metadata.name)\t\(.status.phase)\t\(.status.conditions[-1].message // "unknown")"
'

# Transform: Build deployment summary
kubectl get deployments -n production -o json | jq -r '
    .items[] | [
        .metadata.name,
        (.spec.replicas | tostring),
        (.status.readyReplicas // 0 | tostring),
        .spec.template.spec.containers[0].image
    ] | @tsv
' | column -t

# Conditionals
aws ec2 describe-instances --output json | jq -r '
    .Reservations[].Instances[] 
    | select(.State.Name == "running")
    | select(any(.Tags[]?; .Key == "Environment" and .Value == "production"))
    | [.InstanceId, .InstanceType, (.Tags[] | select(.Key == "Name") | .Value)] 
    | @csv
'

# Modify JSON (useful for API calls)
cat base-config.json | jq \
    --arg env "$ENVIRONMENT" \
    --arg version "$VERSION" \
    '.environment = $env | .image.tag = $version | .replicas = 3'

# Reduce: Sum all request CPU across pods
kubectl get pods -o json | jq '
    [.items[].spec.containers[].resources.requests.cpu // "0"
     | rtrimstr("m") | tonumber] | add
'

# Group by: Count pods per node
kubectl get pods -o json | jq -r '
    [.items[] | {node: .spec.nodeName, name: .metadata.name}]
    | group_by(.node)
    | .[] | "\(.[0].node): \(length) pods"
'

# The --arg flag (parameterize jq safely)
SERVICE="payment-service"
kubectl get pods -o json | jq -r \
    --arg svc "$SERVICE" \
    '.items[] | select(.metadata.labels.app == $svc) | .metadata.name'
# NEVER interpolate bash variables directly into jq — injection risk

# --exit-status for use with set -e
if kubectl get pod "$POD" -o json | jq -e '.status.phase == "Running"' > /dev/null; then
    echo "Pod is running"
fi
# -e: exit 1 if result is false/null — works with set -e

# Raw output modes
jq -r '.name'         # Raw string (no quotes)
jq -c '.items[]'      # Compact (one JSON per line — good for while read loops)
jq -S '.'             # Sort keys (good for diff)
```

### yq — YAML Processing

```bash
# ─── yq: YAML Processing (mikefarah/yq v4) ───────────────

# Read values from Helm values.yaml
yq '.prometheus.server.retention' values.yaml

# Update Kustomize image tag
yq -i ".images[0].newTag = \"${VERSION}\"" kustomization.yaml

# Add/modify nested value
yq -i '.spec.template.spec.containers[0].resources.limits.memory = "512Mi"' deployment.yaml

# Merge YAML files (overlays)
yq eval-all 'select(fileIndex == 0) * select(fileIndex == 1)' base.yaml overlay.yaml

# Convert YAML to JSON (pipe to jq for complex processing)
yq -o json values.yaml | jq '.prometheus'

# Delete a field
yq -i 'del(.metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"])' resource.yaml

# Select from array
yq '.spec.containers[] | select(.name == "app")' pod.yaml

# Loop through items in script
yq -r '.services[].name' config.yaml | while IFS= read -r service; do
    echo "Processing: $service"
done

# Environment variable substitution
export REPLICAS=3
yq -i '.spec.replicas = env(REPLICAS)' deployment.yaml
```

**jq vs yq decision:**
- API responses → `jq` (APIs return JSON)
- K8s manifests, Helm values → `yq` (K8s is YAML-native)
- Complex transformations on YAML → convert to JSON with `yq -o json`, process with `jq`, convert back

---

## 9. CONTAINER ENTRYPOINT SCRIPTS

Every container you ship at NovaMart needs a proper entrypoint. This is where bash meets containers.

```bash
#!/usr/bin/env bash
# entrypoint.sh for payment-service
set -euo pipefail

# ─── Signal Handling ───────────────────────────────────────
# The #1 container bash mistake: not forwarding signals to the app process
shutdown() {
    echo "Received shutdown signal, forwarding to PID $APP_PID..."
    kill -TERM "$APP_PID" 2>/dev/null || true
    wait "$APP_PID" 2>/dev/null || true
    exit 0
}
trap shutdown SIGTERM SIGINT

# ─── Environment Validation ───────────────────────────────
required_vars=(
    "DB_HOST"
    "DB_PORT"
    "DB_NAME"
    "REDIS_URL"
    "OTEL_EXPORTER_OTLP_ENDPOINT"
)

missing=()
for var in "${required_vars[@]}"; do
    if [[ -z "${!var:-}" ]]; then
        missing+=("$var")
    fi
done

if (( ${#missing[@]} > 0 )); then
    echo "FATAL: Missing required environment variables: ${missing[*]}" >&2
    exit 1
fi

# ─── Config Generation from Environment ───────────────────
# Template substitution (envsubst)
envsubst < /app/config/config.yaml.tpl > /app/config/config.yaml
echo "Generated config.yaml from template"

# ─── Pre-flight Checks ────────────────────────────────────
# Wait for dependencies to be ready
echo "Waiting for database..."
timeout=30
elapsed=0
until pg_isready -h "$DB_HOST" -p "$DB_PORT" -q 2>/dev/null; do
    if (( elapsed >= timeout )); then
        echo "FATAL: Database not ready after ${timeout}s" >&2
        exit 1
    fi
    sleep 1
    (( elapsed++ ))
done
echo "Database is ready"

echo "Waiting for Redis..."
elapsed=0
until redis-cli -u "$REDIS_URL" ping 2>/dev/null | grep -q PONG; do
    if (( elapsed >= timeout )); then
        echo "FATAL: Redis not ready after ${timeout}s" >&2
        exit 1
    fi
    sleep 1
    (( elapsed++ ))
done
echo "Redis is ready"

# ─── Start Application ────────────────────────────────────
echo "Starting payment-service..."

# exec REPLACES the shell process with the app — PID 1 becomes the app
# This means the app receives signals directly from Kubernetes
# Use exec when there's nothing to do after the app starts

# BUT: if you need signal forwarding (background process pattern):
/app/payment-service "$@" &
APP_PID=$!
echo "Application started with PID: $APP_PID"

# Wait for the app to exit
wait "$APP_PID"
exit $?
```

**The two patterns and when to use each:**

```bash
# PATTERN 1: exec (preferred when no cleanup needed)
exec /app/payment-service "$@"
# Shell is REPLACED by app. App IS PID 1.
# Signals go directly to app. Shell is gone.
# Use when: app handles signals itself (Go, Java)

# PATTERN 2: Background + wait (when you need cleanup)
/app/payment-service "$@" &
APP_PID=$!
wait "$APP_PID"
# Shell stays as PID 1. App is child process.
# Shell trap catches signals and forwards them.
# Use when: you need cleanup, logging, or signal forwarding

# WRONG — the silent killer:
/app/payment-service "$@"
# Shell runs app as FOREGROUND child.
# BUT: signals to PID 1 (shell) are NOT forwarded automatically.
# Kubernetes sends SIGTERM → shell receives it → app never knows.
# After terminationGracePeriodSeconds → SIGKILL → unclean shutdown.
```

**Why `"$@"` with quotes?**
```bash
# Without quotes:
exec /app/cmd $@
# If args contain spaces: --config "/path/my config/file" 
# becomes THREE args: --config "/path/my config/file" → --config, /path/my, config/file

# With quotes:
exec /app/cmd "$@"
# Args preserved exactly as passed
```

---

## 10. REAL NOVAMART SCRIPTS

### Script 1: EKS Node Drain Wrapper

```bash
#!/usr/bin/env bash
#
# Script: node-drain.sh
# Purpose: Safely drain EKS nodes with PDB validation, Slack notification, and rollback
# Usage: ./node-drain.sh --node ip-10-0-1-42.ec2.internal --reason "instance refresh"
#
set -euo pipefail
IFS=$'\n\t'

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
readonly LOG_FILE="/var/log/novamart/node-drain.log"
readonly LOCK_FILE="/tmp/node-drain.lock"
readonly DRY_RUN="${DRY_RUN:-false}"
readonly DRAIN_TIMEOUT="${DRAIN_TIMEOUT:-300}"

# Source shared functions (logging, retry, lock, slack)
# shellcheck source=lib/common.sh
source "${SCRIPT_DIR}/lib/common.sh"

# ─── Argument Parsing ──────────────────────────────────────
NODE=""
REASON=""
FORCE=false

parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -n|--node)   NODE="${2:?--node requires a value}"; shift 2 ;;
            -r|--reason) REASON="${2:?--reason requires a value}"; shift 2 ;;
            -f|--force)  FORCE=true; shift ;;
            -d|--dry-run) DRY_RUN=true; shift ;;
            -h|--help)   usage 0 ;;
            *) fatal "Unknown option: $1" ;;
        esac
    done

    [[ -z "$NODE" ]] && fatal "Missing required: --node"
    [[ -z "$REASON" ]] && fatal "Missing required: --reason"
}

# ─── Pre-Drain Validation ─────────────────────────────────
validate_node() {
    info "Validating node: $NODE"

    # Check node exists
    if ! kubectl get node "$NODE" &>/dev/null; then
        fatal "Node not found: $NODE"
    fi

    # Check node status
    local status
    status=$(kubectl get node "$NODE" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
    info "Node status: Ready=$status"

    # Get pods on this node (excluding DaemonSets)
    local pod_count
    pod_count=$(kubectl get pods --all-namespaces --field-selector "spec.nodeName=$NODE" \
        -o json | jq '[.items[] | select(
            .metadata.ownerReferences[0].kind != "DaemonSet"
        )] | length')
    info "Non-DaemonSet pods on node: $pod_count"

    if (( pod_count == 0 )); then
        info "Node has no workload pods — safe to drain"
        return 0
    fi

    # PDB validation — THE critical safety check
    info "Checking PDB constraints..."
    local blocked_pdbs=()

    while IFS= read -r line; do
        local pdb_name ns allowed current
        pdb_name=$(echo "$line" | awk '{print $1}')
        ns=$(echo "$line" | awk '{print $2}')
        allowed=$(echo "$line" | awk '{print $3}')
        current=$(echo "$line" | awk '{print $4}')

        # Check if draining this node would violate PDB
        local pods_on_node
        pods_on_node=$(kubectl get pods -n "$ns" --field-selector "spec.nodeName=$NODE" \
            -o json | jq --arg pdb "$pdb_name" '[.items[] | select(
                .metadata.labels | to_entries | any(.value)  # simplified match
            )] | length')

        if (( pods_on_node > 0 && allowed < 1 )); then
            blocked_pdbs+=("${ns}/${pdb_name} (allowed_disruptions=${allowed})")
        fi
    done < <(kubectl get pdb --all-namespaces -o custom-columns=\
'NAME:.metadata.name,NAMESPACE:.metadata.namespace,ALLOWED:.status.disruptionsAllowed,CURRENT:.status.currentHealthy' \
        --no-headers 2>/dev/null)

    if (( ${#blocked_pdbs[@]} > 0 )); then
        error "PDB constraints would be violated:"
        for pdb in "${blocked_pdbs[@]}"; do
            error "  - $pdb"
        done
        if [[ "$FORCE" != true ]]; then
            fatal "Drain blocked by PDBs. Use --force to override (DANGEROUS)."
        fi
        warn "FORCE mode: proceeding despite PDB constraints"
    else
        info "All PDB constraints satisfied"
    fi
}

# ─── Main Execution ────────────────────────────────────────
main() {
    parse_args "$@"
    acquire_lock

    info "═══ Node Drain: $NODE ═══"
    info "Reason: $REASON"
    info "Dry run: $DRY_RUN"

    validate_node

    # Notify team
    notify_slack "INFO" ":warning: Draining node \`${NODE}\`\nReason: ${REASON}\nOperator: $(whoami)"

    # Cordon first (prevent new pods)
    info "Cordoning node..."
    run kubectl cordon "$NODE"

    # Drain with safety flags
    info "Draining node (timeout: ${DRAIN_TIMEOUT}s)..."
    if run kubectl drain "$NODE" \
        --ignore-daemonsets \
        --delete-emptydir-data \
        --grace-period=60 \
        --timeout="${DRAIN_TIMEOUT}s" \
        --force="${FORCE}"; then

        info "Node drained successfully"
        notify_slack "SUCCESS" ":white_check_mark: Node \`${NODE}\` drained successfully"
    else
        error "Drain failed — uncordoning node to restore scheduling"
        run kubectl uncordon "$NODE"
        notify_slack "FAILED" ":x: Node \`${NODE}\` drain FAILED — node uncordoned"
        fatal "Drain failed. Node has been uncordoned."
    fi

    info "═══ Drain Complete ═══"
}

main "$@"
```

### Script 2: Certificate Expiry Checker

```bash
#!/usr/bin/env bash
#
# Script: cert-checker.sh
# Purpose: Scan all TLS certificates in cluster, alert on expiring certs
# Schedule: Runs as K8s CronJob daily at 06:00 UTC
#
set -euo pipefail
IFS=$'\n\t'

readonly WARN_DAYS=30
readonly CRIT_DAYS=14
readonly OUTPUT_FORMAT="${OUTPUT_FORMAT:-table}"  # table, json, csv

source "$(dirname "${BASH_SOURCE[0]}")/lib/common.sh"

check_k8s_secrets() {
    info "Scanning TLS secrets across all namespaces..."

    local findings=()
    local total=0
    local warnings=0
    local criticals=0
    local expired=0

    while IFS=$'\t' read -r namespace name; do
        (( total++ ))

        # Extract certificate
        local cert_data
        cert_data=$(kubectl get secret "$name" -n "$namespace" \
            -o jsonpath='{.data.tls\.crt}' 2>/dev/null | base64 -d 2>/dev/null) || continue

        # Parse expiry date
        local expiry_date expiry_epoch now_epoch days_left
        expiry_date=$(echo "$cert_data" | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2) || continue
        expiry_epoch=$(date -d "$expiry_date" +%s 2>/dev/null) || continue
        now_epoch=$(date +%s)
        days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

        # Parse subject and issuer
        local subject issuer
        subject=$(echo "$cert_data" | openssl x509 -noout -subject 2>/dev/null | sed 's/subject=//')
        issuer=$(echo "$cert_data" | openssl x509 -noout -issuer 2>/dev/null | sed 's/issuer=//')

        # Classify
        local severity="OK"
        if (( days_left < 0 )); then
            severity="EXPIRED"
            (( expired++ ))
        elif (( days_left <= CRIT_DAYS )); then
            severity="CRITICAL"
            (( criticals++ ))
        elif (( days_left <= WARN_DAYS )); then
            severity="WARNING"
            (( warnings++ ))
        fi

        if [[ "$severity" != "OK" ]]; then
            findings+=("$(printf '%s\t%s\t%s\t%d\t%s\t%s' \
                "$severity" "$namespace" "$name" "$days_left" "$subject" "$expiry_date")")
        fi

    done < <(kubectl get secrets --all-namespaces -o json | \
        jq -r '.items[] | select(.type == "kubernetes.io/tls") | 
        [.metadata.namespace, .metadata.name] | @tsv')

    # Output results
    info "Scanned $total TLS secrets"

    if (( ${#findings[@]} == 0 )); then
        info "All certificates healthy (> ${WARN_DAYS} days until expiry)"
        return 0
    fi

    # Sort by days remaining (most urgent first)
    local sorted
    sorted=$(printf '%s\n' "${findings[@]}" | sort -t$'\t' -k4 -n)

    echo ""
    printf "%-10s %-20s %-40s %-6s %-30s %s\n" \
        "SEVERITY" "NAMESPACE" "SECRET" "DAYS" "SUBJECT" "EXPIRES"
    printf '%.0s─' {1..140}; echo ""
    echo "$sorted" | while IFS=$'\t' read -r sev ns name days subj exp; do
        printf "%-10s %-20s %-40s %-6s %-30s %s\n" "$sev" "$ns" "$name" "$days" "$subj" "$exp"
    done

    # Alert
    if (( expired > 0 || criticals > 0 )); then
        local alert_msg=":rotating_light: *Certificate Alert*\n"
        alert_msg+="Expired: ${expired} | Critical (<${CRIT_DAYS}d): ${criticals} | Warning (<${WARN_DAYS}d): ${warnings}\n"
        alert_msg+="\`\`\`\n$(printf '%s\n' "${findings[@]}" | head -10)\n\`\`\`"
        notify_slack "CRITICAL" "$alert_msg"
        return 1
    elif (( warnings > 0 )); then
        notify_slack "WARN" ":warning: ${warnings} certificate(s) expiring within ${WARN_DAYS} days"
    fi
}

check_external_endpoints() {
    info "Checking external endpoint certificates..."

    local endpoints=(
        "api.novamart.com:443"
        "payments.novamart.com:443"
        "cdn.novamart.com:443"
        "grafana.novamart.internal:443"
        "argocd.novamart.internal:443"
    )

    for endpoint in "${endpoints[@]}"; do
        local host port
        host="${endpoint%%:*}"
        port="${endpoint##*:}"

        local cert_info
        if ! cert_info=$(echo | openssl s_client -servername "$host" -connect "$endpoint" 2>/dev/null | \
            openssl x509 -noout -enddate -subject 2>/dev/null); then
            error "Cannot connect to $endpoint"
            continue
        fi

        local expiry_date days_left
        expiry_date=$(echo "$cert_info" | grep notAfter | cut -d= -f2)
        local expiry_epoch now_epoch
        expiry_epoch=$(date -d "$expiry_date" +%s)
        now_epoch=$(date +%s)
        days_left=$(( (expiry_epoch - now_epoch) / 86400 ))

        if (( days_left <= CRIT_DAYS )); then
            error "CRITICAL: $endpoint expires in $days_left days ($expiry_date)"
        elif (( days_left <= WARN_DAYS )); then
            warn "WARNING: $endpoint expires in $days_left days ($expiry_date)"
        else
            info "OK: $endpoint expires in $days_left days"
        fi
    done
}

main() {
    info "═══ NovaMart Certificate Health Check ═══"
    check_k8s_secrets
    check_external_endpoints
    info "═══ Check Complete ═══"
}

main "$@"
```

### Script 3: Incident Response Log Collector

```bash
#!/usr/bin/env bash
#
# Script: incident-collect.sh
# Purpose: Collect diagnostic data during incidents for postmortem analysis
# Usage: ./incident-collect.sh --incident INC-2024-0142 --namespace production --service payment-service
#
set -euo pipefail
IFS=$'\n\t'

source "$(dirname "${BASH_SOURCE[0]}")/lib/common.sh"

INCIDENT_ID=""
NAMESPACE=""
SERVICE=""
SINCE="1h"
OUTPUT_DIR=""
S3_BUCKET="novamart-incident-artifacts"

parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -i|--incident)  INCIDENT_ID="${2:?}"; shift 2 ;;
            -n|--namespace) NAMESPACE="${2:?}"; shift 2 ;;
            -s|--service)   SERVICE="${2:?}"; shift 2 ;;
            --since)        SINCE="${2:?}"; shift 2 ;;
            -h|--help)      usage 0 ;;
            *)              fatal "Unknown: $1" ;;
        esac
    done
    [[ -z "$INCIDENT_ID" ]] && fatal "Missing: --incident"
    [[ -z "$NAMESPACE" ]] && fatal "Missing: --namespace"
}

collect_cluster_state() {
    info "Collecting cluster state..."
    local dir="${OUTPUT_DIR}/cluster"
    mkdir -p "$dir"

    kubectl get nodes -o wide > "${dir}/nodes.txt" 2>&1
    kubectl top nodes > "${dir}/node-resources.txt" 2>&1
    kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -200 > "${dir}/events.txt" 2>&1
    kubectl get componentstatuses > "${dir}/component-status.txt" 2>&1 || true
}

collect_namespace_state() {
    info "Collecting namespace state: $NAMESPACE"
    local dir="${OUTPUT_DIR}/namespace"
    mkdir -p "$dir"

    kubectl get all -n "$NAMESPACE" -o wide > "${dir}/all-resources.txt" 2>&1
    kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp' > "${dir}/events.txt" 2>&1
    kubectl top pods -n "$NAMESPACE" > "${dir}/pod-resources.txt" 2>&1
    kubectl get hpa -n "$NAMESPACE" -o wide > "${dir}/hpa.txt" 2>&1 || true
    kubectl get pdb -n "$NAMESPACE" -o wide > "${dir}/pdb.txt" 2>&1 || true
    kubectl get networkpolicies -n "$NAMESPACE" -o yaml > "${dir}/netpol.yaml" 2>&1 || true
}

collect_service_data() {
    [[ -z "$SERVICE" ]] && return 0
    info "Collecting service data: $SERVICE"
    local dir="${OUTPUT_DIR}/service"
    mkdir -p "$dir"

    # Deployment/rollout status
    kubectl describe deployment "$SERVICE" -n "$NAMESPACE" > "${dir}/describe-deployment.txt" 2>&1 || true
    kubectl rollout history "deployment/$SERVICE" -n "$NAMESPACE" > "${dir}/rollout-history.txt" 2>&1 || true

    # Current and recent pod logs
    local pods
    pods=$(kubectl get pods -n "$NAMESPACE" -l "app=$SERVICE" -o jsonpath='{.items[*].metadata.name}')
    for pod in $pods; do
        kubectl logs "$pod" -n "$NAMESPACE" --since="$SINCE" --all-containers \
            > "${dir}/logs-${pod}.txt" 2>&1 || true
        kubectl logs "$pod" -n "$NAMESPACE" --all-containers --previous \
            > "${dir}/logs-${pod}-previous.txt" 2>&1 || true
        kubectl describe pod "$pod" -n "$NAMESPACE" \
            > "${dir}/describe-${pod}.txt" 2>&1 || true
    done

    # Endpoints
    kubectl get endpoints "$SERVICE" -n "$NAMESPACE" -o yaml > "${dir}/endpoints.yaml" 2>&1 || true

    # ArgoCD application state
    if command -v argocd &>/dev/null; then
        argocd app get "$SERVICE" -o yaml > "${dir}/argocd-app.yaml" 2>&1 || true
    fi
}

collect_metrics_snapshot() {
    info "Collecting metrics snapshot..."
    local dir="${OUTPUT_DIR}/metrics"
    mkdir -p "$dir"

    local prom_url="http://prometheus.monitoring:9090"

    # Key metrics for the time window
    local queries=(
        "rate(http_requests_total{namespace=\"${NAMESPACE}\"}[5m])"
        "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{namespace=\"${NAMESPACE}\"}[5m]))"
        "rate(http_requests_total{namespace=\"${NAMESPACE}\",code=~\"5..\"}[5m])"
        "container_memory_working_set_bytes{namespace=\"${NAMESPACE}\"}"
        "rate(container_cpu_usage_seconds_total{namespace=\"${NAMESPACE}\"}[5m])"
    )

    for i in "${!queries[@]}"; do
        retry 3 2 curl -sf --max-time 30 \
            "${prom_url}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('''${queries[$i]}'''))")" \
            > "${dir}/query_${i}.json" 2>&1 || warn "Failed to collect metric query $i"
    done
}

package_and_upload() {
    info "Packaging collection..."
    local archive="${INCIDENT_ID}-$(date -u +%Y%m%d-%H%M%S).tar.gz"
    local archive_path="/tmp/${archive}"

    # Create manifest
    cat > "${OUTPUT_DIR}/MANIFEST.txt" <<EOF
Incident: ${INCIDENT_ID}
Collected: $(date -u '+%Y-%m-%dT%H:%M:%SZ')
Collector: $(whoami)@$(hostname)
Namespace: ${NAMESPACE}
Service: ${SERVICE:-all}
Since: ${SINCE}
K8s context: $(kubectl config current-context)
EOF

    tar -czf "$archive_path" -C "$(dirname "$OUTPUT_DIR")" "$(basename "$OUTPUT_DIR")"
    local size
    size=$(du -h "$archive_path" | cut -f1)
    info "Archive created: ${archive} (${size})"

    # Upload to S3
    if retry 3 2 aws s3 cp "$archive_path" \
        "s3://${S3_BUCKET}/incidents/${INCIDENT_ID}/${archive}"; then
        info "Uploaded to s3://${S3_BUCKET}/incidents/${INCIDENT_ID}/${archive}"
    else
        warn "S3 upload failed — archive preserved at ${archive_path}"
    fi
}

main() {
    parse_args "$@"

    OUTPUT_DIR=$(mktemp -d "/tmp/incident-${INCIDENT_ID}.XXXXXX")
    info "═══ Incident Collection: ${INCIDENT_ID} ═══"
    info "Output directory: ${OUTPUT_DIR}"

    collect_cluster_state
    collect_namespace_state
    collect_service_data
    collect_metrics_snapshot
    package_and_upload

    info "═══ Collection Complete ═══"
    notify_slack "INFO" ":mag: Incident \`${INCIDENT_ID}\` diagnostics collected and uploaded to S3"
}

main "$@"
```

### Shared Library: lib/common.sh

```bash
#!/usr/bin/env bash
#
# Shared library for NovaMart operational scripts
# Source this: source "${SCRIPT_DIR}/lib/common.sh"
#

# Prevent double-sourcing
[[ -n "${_COMMON_SH_LOADED:-}" ]] && return 0
readonly _COMMON_SH_LOADED=1

# ─── Logging (same as Section 2 above) ────────────────────
# [logging functions here — log, info, warn, error, fatal, debug]

# ─── Lock Management ──────────────────────────────────────
# [acquire_lock function from Section 5]

# ─── Retry Logic ──────────────────────────────────────────
# [retry function from Section 6]

# ─── Dry-Run Wrapper ─────────────────────────────────────
run() {
    if [[ "${DRY_RUN:-false}" == true ]]; then
        info "[DRY-RUN] Would execute: $*"
        return 0
    fi
    debug "Executing: $*"
    "$@"
}

# ─── Slack Notification ───────────────────────────────────
notify_slack() {
    local severity="$1"
    local message="$2"
    local webhook="${SLACK_WEBHOOK:-}"

    [[ -z "$webhook" ]] && { debug "No SLACK_WEBHOOK set, skipping notification"; return 0; }
    [[ "${NO_SLACK:-false}" == true ]] && { debug "Slack suppressed by --no-slack"; return 0; }

    local color
    case "$severity" in
        SUCCESS|OK|INFO) color="#36a64f" ;;
        WARN|WARNING)    color="#ff9900" ;;
        CRITICAL|FAILED|ERROR) color="#ff0000" ;;
        *) color="#cccccc" ;;
    esac

    local payload
    payload=$(jq -n \
        --arg color "$color" \
        --arg text "$message" \
        --arg footer "$(hostname) | $(date -u '+%H:%M UTC')" \
        '{attachments: [{color: $color, text: $text, footer: $footer}]}')

    # Fire and forget — don't let Slack failure kill the script
    curl -sf --max-time 5 -X POST -H 'Content-type: application/json' \
        -d "$payload" "$webhook" > /dev/null 2>&1 || \
        warn "Slack notification failed (non-fatal)"
}

# ─── Prerequisite Check ──────────────────────────────────
require_commands() {
    local missing=()
    for cmd in "$@"; do
        if ! command -v "$cmd" &>/dev/null; then
            missing+=("$cmd")
        fi
    done
    if (( ${#missing[@]} > 0 )); then
        fatal "Missing required commands: ${missing[*]}"
    fi
}

# ─── Confirmation Prompt ─────────────────────────────────
confirm() {
    local message="${1:-Are you sure?}"
    if [[ "${FORCE:-false}" == true ]]; then
        warn "FORCE mode: auto-confirming: $message"
        return 0
    fi
    printf "%s [y/N]: " "$message" >&2
    local answer
    read -r answer
    [[ "$answer" =~ ^[Yy]([Ee][Ss])?$ ]] || fatal "Aborted by user"
}

# ─── AWS Helpers ──────────────────────────────────────────
aws_account_id() {
    aws sts get-caller-identity --query 'Account' --output text
}

aws_region() {
    aws configure get region || echo "${AWS_DEFAULT_REGION:-us-east-1}"
}

ecr_login() {
    local region
    region=$(aws_region)
    local account
    account=$(aws_account_id)
    aws ecr get-login-password --region "$region" | \
        docker login --username AWS --password-stdin "${account}.dkr.ecr.${region}.amazonaws.com"
}

# ─── K8s Helpers ──────────────────────────────────────────
wait_for_rollout() {
    local resource="$1"
    local namespace="$2"
    local timeout="${3:-300}"

    info "Waiting for rollout: ${resource} in ${namespace} (timeout: ${timeout}s)"
    if ! kubectl rollout status "$resource" -n "$namespace" --timeout="${timeout}s"; then
        error "Rollout failed or timed out"
        kubectl get pods -n "$namespace" -l "app=${resource#*/}" -o wide >&2
        return 1
    fi
    info "Rollout complete: ${resource}"
}

get_pod_status_summary() {
    local namespace="$1"
    local label="${2:-}"

    local selector=""
    [[ -n "$label" ]] && selector="-l $label"

    kubectl get pods -n "$namespace" $selector -o json | jq -r '
        .items | group_by(.status.phase) | 
        map({phase: .[0].status.phase, count: length}) |
        .[] | "\(.phase): \(.count)"
    '
}

is_pod_ready() {
    local pod="$1"
    local namespace="$2"
    kubectl get pod "$pod" -n "$namespace" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null | grep -q "True"
}
```

---

## 11. MAKEFILES FOR DEVOPS WORKFLOWS

Every NovaMart repo has a Makefile. It's the universal entry point — any engineer runs `make help` and knows what's available.

```makefile
# ─── NovaMart Service Makefile ─────────────────────────────
.DEFAULT_GOAL := help
SHELL := /bin/bash
.SHELLFLAGS := -euo pipefail -c

# ─── Variables ─────────────────────────────────────────────
SERVICE_NAME    := payment-service
REGISTRY        := 123456789012.dkr.ecr.us-east-1.amazonaws.com
VERSION         ?= $(shell git rev-parse --short HEAD)
IMAGE           := $(REGISTRY)/$(SERVICE_NAME):sha-$(VERSION)
ENVIRONMENT     ?= dev
KUBECONFIG      ?= ~/.kube/config

# Go variables
GO              := go
GOFLAGS         := -trimpath
LDFLAGS         := -s -w -X main.version=$(VERSION) -X main.buildTime=$(shell date -u +%Y-%m-%dT%H:%M:%SZ)

# ─── Help (auto-generated from comments) ──────────────────
.PHONY: help
help: ## Show this help message
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

# ─── Development ──────────────────────────────────────────
.PHONY: build test lint fmt vet
build: ## Build the binary
	$(GO) build $(GOFLAGS) -ldflags "$(LDFLAGS)" -o bin/$(SERVICE_NAME) ./cmd/$(SERVICE_NAME)

test: ## Run unit tests
	$(GO) test -race -coverprofile=coverage.out -count=1 ./...
	$(GO) tool cover -func=coverage.out | tail -1

lint: ## Run linters
	golangci-lint run --timeout 5m ./...
	hadolint Dockerfile
	shellcheck scripts/*.sh

fmt: ## Format code
	$(GO) fmt ./...
	goimports -w .

vet: ## Run go vet
	$(GO) vet ./...

# ─── Docker ───────────────────────────────────────────────
.PHONY: docker-build docker-push docker-scan
docker-build: ## Build Docker image
	docker build \
		--build-arg VERSION=$(VERSION) \
		--tag $(IMAGE) \
		--tag $(REGISTRY)/$(SERVICE_NAME):latest \
		.

docker-push: docker-build ## Push Docker image to ECR
	aws ecr get-login-password | docker login --username AWS --password-stdin $(REGISTRY)
	docker push $(IMAGE)

docker-scan: docker-build ## Scan Docker image for vulnerabilities
	trivy image --severity HIGH,CRITICAL --exit-code 1 $(IMAGE)

# ─── Kubernetes ───────────────────────────────────────────
.PHONY: deploy rollback status logs
deploy: ## Deploy to environment (ENVIRONMENT=dev|staging|production)
	@echo "Deploying $(SERVICE_NAME):sha-$(VERSION) to $(ENVIRONMENT)"
	cd kustomize/overlays/$(ENVIRONMENT) && \
		kustomize edit set image $(SERVICE_NAME)=$(IMAGE)
	kubectl apply -k kustomize/overlays/$(ENVIRONMENT)
	kubectl rollout status deployment/$(SERVICE_NAME) -n $(ENVIRONMENT) --timeout=300s

rollback: ## Rollback to previous revision
	kubectl rollout undo deployment/$(SERVICE_NAME) -n $(ENVIRONMENT)
	kubectl rollout status deployment/$(SERVICE_NAME) -n $(ENVIRONMENT) --timeout=120s

status: ## Show deployment status
	@echo "=== Deployment ==="
	kubectl get deployment $(SERVICE_NAME) -n $(ENVIRONMENT) -o wide
	@echo ""
	@echo "=== Pods ==="
	kubectl get pods -n $(ENVIRONMENT) -l app=$(SERVICE_NAME) -o wide
	@echo ""
	@echo "=== HPA ==="
	kubectl get hpa $(SERVICE_NAME) -n $(ENVIRONMENT) 2>/dev/null || echo "No HPA"

logs: ## Tail logs (ENVIRONMENT=dev)
	kubectl logs -n $(ENVIRONMENT) -l app=$(SERVICE_NAME) --all-containers -f --tail=100

# ─── Infrastructure ───────────────────────────────────────
.PHONY: tf-plan tf-apply tf-fmt
tf-plan: ## Terraform plan
	cd terraform/environments/$(ENVIRONMENT) && \
		terraform init -backend=true && \
		terraform plan -out=tfplan

tf-apply: ## Terraform apply (requires tf-plan first)
	cd terraform/environments/$(ENVIRONMENT) && \
		terraform apply tfplan

tf-fmt: ## Format Terraform files
	terraform fmt -recursive terraform/

# ─── CI Checks (run before pushing) ──────────────────────
.PHONY: ci pre-commit
ci: lint vet test docker-scan ## Run all CI checks locally

pre-commit: ## Install pre-commit hooks
	pre-commit install
	pre-commit run --all-files

# ─── Cleanup ─────────────────────────────────────────────
.PHONY: clean
clean: ## Clean build artifacts
	rm -rf bin/ coverage.out dist/
	docker rmi $(IMAGE) 2>/dev/null || true
```

**Why Makefiles work for DevOps:**
- Universal: every OS has `make`
- Self-documenting (the `help` target parses `##` comments)
- Dependency chains (`docker-push` depends on `docker-build`)
- Variable overrides from CLI: `make deploy ENVIRONMENT=production`
- `.PHONY` prevents conflicts with files named `build`, `test`, etc.

**Makefile gotchas:**

```makefile
# GOTCHA 1: Recipes use TABS not SPACES
build:
    go build ./...  # THIS FAILS — looks like spaces but MUST be tab
	go build ./...  # This works — actual tab character

# GOTCHA 2: Each line runs in a separate shell
deploy:
	cd terraform/
	terraform plan    # WRONG — this runs in ORIGINAL directory
	                  # because cd was in a different shell

# FIX: Use && on one line, or .ONESHELL
deploy:
	cd terraform/ && terraform plan

# Or:
.ONESHELL:
deploy:
	cd terraform/
	terraform plan    # Now works — same shell

# GOTCHA 3: Make variables vs shell variables
VERSION := $(shell git rev-parse --short HEAD)  # Make variable (expanded once)
deploy:
	echo "Deploying $$VERSION"     # $$ escapes to single $ for shell
	echo "Deploying $(VERSION)"    # This is the Make variable
```

**Alternatives to Make:**

```yaml
# ─── Taskfile.yml (Go-based, YAML syntax) ─────────────────
version: '3'

vars:
  SERVICE_NAME: payment-service
  VERSION:
    sh: git rev-parse --short HEAD

tasks:
  build:
    desc: Build the binary
    cmds:
      - go build -trimpath -o bin/{{.SERVICE_NAME}} ./cmd/{{.SERVICE_NAME}}

  test:
    desc: Run tests
    cmds:
      - go test -race -coverprofile=coverage.out ./...

  deploy:
    desc: Deploy to environment
    vars:
      ENV: '{{.ENV | default "dev"}}'
    cmds:
      - echo "Deploying to {{.ENV}}"
      - kubectl apply -k kustomize/overlays/{{.ENV}}

  ci:
    desc: Run all CI checks
    deps: [lint, test]
    cmds:
      - task: docker-scan
```

---

## 12. ANTI-PATTERNS — THE BASH HALL OF SHAME

These are real patterns I've seen kill production. Burn them into memory.

```bash
# ═══ ANTI-PATTERN 1: No error handling ═══════════════════
# BAD:
cd /app/deploy
rm -rf old/
cp -r new/* .
systemctl restart app
# If cd fails (directory doesn't exist), rm -rf runs in CURRENT DIRECTORY

# GOOD:
set -euo pipefail
cd /app/deploy || fatal "Deploy directory not found"

# ═══ ANTI-PATTERN 2: Parsing ls output ══════════════════
# BAD:
for f in $(ls /var/log/*.log); do
    process "$f"
done
# Breaks on spaces in filenames, depends on ls output format

# GOOD:
for f in /var/log/*.log; do
    [[ -f "$f" ]] || continue  # Handle no matches (nullglob alternative)
    process "$f"
done

# ═══ ANTI-PATTERN 3: Unquoted variables ═════════════════
# BAD:
filename="my important file.txt"
rm $filename
# Executes: rm my important file.txt (3 separate arguments!)

# GOOD:
rm "$filename"
# ALWAYS. QUOTE. VARIABLES. (except in [[ ]] where it's technically safe)

# ═══ ANTI-PATTERN 4: eval usage ═════════════════════════
# BAD:
cmd="ls -la /tmp"
eval $cmd
# eval executes ANYTHING — command injection risk
# If cmd comes from user input: cmd="ls; rm -rf /"

# GOOD: Use arrays
cmd=(ls -la /tmp)
"${cmd[@]}"

# ═══ ANTI-PATTERN 5: Bash beyond 200 lines ══════════════
# If your bash script exceeds ~200 lines of logic:
#   - Complex argument parsing → Use Python (Click) or Go (Cobra)
#   - API calls → Use Python (requests + retry)
#   - JSON processing → Use Python or Go
#   - Concurrent operations → Use Go
#   - State management → Use Python or Go
# Bash is glue. Not application logic.

# ═══ ANTI-PATTERN 6: Hardcoded credentials ══════════════
# BAD:
DB_PASSWORD="super_secret_123"
mysql -u admin -p"$DB_PASSWORD" ...

# GOOD:
DB_PASSWORD=$(aws secretsmanager get-secret-value \
    --secret-id "novamart/production/db" \
    --query 'SecretString' --output text | jq -r '.password')

# ═══ ANTI-PATTERN 7: No logging, no dry-run ═════════════
# BAD:
kubectl delete pods -l app=payment-service -n production
# No record of what happened. No way to verify before executing.

# GOOD:
info "Deleting pods matching: app=payment-service in production"
pods=$(kubectl get pods -n production -l app=payment-service -o name)
info "Pods to delete: $(echo "$pods" | wc -l)"
info "Pod list: $pods"
run kubectl delete $pods -n production

# ═══ ANTI-PATTERN 8: Not checking command existence ═════
# BAD:
jq '.data' file.json  # What if jq isn't installed?

# GOOD:
require_commands jq kubectl aws curl
# Fails fast with clear message if anything is missing

# ═══ ANTI-PATTERN 9: Heredoc indentation ════════════════
# BAD (spaces):
    cat <<EOF
    This will have leading spaces in output
    EOF
# Also BAD: <<EOF with tab-indented terminator → won't match

# GOOD (<<- strips leading TABS):
	cat <<-EOF
		This output has no leading whitespace
		Because <<- strips leading tabs
	EOF
# WARNING: <<- only strips TABS. Your editor must use real tabs, not spaces.
# Safest: don't indent heredocs, or use <<EOF without -.

# ═══ ANTI-PATTERN 10: Not handling signals in wrappers ══
# BAD (container entrypoint):
#!/bin/bash
./setup-config.sh
./my-app
# SIGTERM goes to bash (PID 1), never reaches my-app
# K8s waits terminationGracePeriodSeconds, then SIGKILL → unclean shutdown

# GOOD:
#!/bin/bash
./setup-config.sh
exec ./my-app  # exec REPLACES bash — my-app becomes PID 1, receives signals directly
```

---

## 13. SHELLCHECK — YOUR BEST FRIEND

ShellCheck is a static analysis tool for bash. It catches the mistakes above automatically.

```bash
# Install
apt-get install shellcheck   # Debian/Ubuntu
brew install shellcheck       # macOS

# Run
shellcheck script.sh
shellcheck -x script.sh       # -x follows sourced files
shellcheck -s bash script.sh  # Specify shell dialect

# Top violations you'll see:
# SC2086: Double quote to prevent globbing and word splitting
#   BAD:  echo $var
#   GOOD: echo "$var"

# SC2046: Quote this to prevent word splitting
#   BAD:  docker rm $(docker ps -aq)
#   GOOD: docker rm "$(docker ps -aq)"
#   OR:   readarray -t ids < <(docker ps -aq); docker rm "${ids[@]}"

# SC2034: Variable appears unused
#   Usually means typo in variable name, or variable used in sourced file
#   Fix: # shellcheck disable=SC2034  (with comment explaining why)

# SC2155: Declare and assign separately to avoid masking return values
#   BAD:  local result=$(some_command)   # local always returns 0!
#   GOOD: local result
#         result=$(some_command)          # Now set -e can catch failure

# SC2164: Use cd ... || exit in case cd fails
#   BAD:  cd /some/dir
#   GOOD: cd /some/dir || exit 1

# SC2128: Expanding an array without an index only gives the first element
#   BAD:  echo "$array"
#   GOOD: echo "${array[@]}"

# In CI (Jenkinsfile):
# stage('Lint Scripts') {
#     sh 'find scripts/ -name "*.sh" -exec shellcheck -x {} +'
# }

# In pre-commit:
# - repo: https://github.com/koalaman/shellcheck-precommit
#   rev: v0.9.0
#   hooks:
#     - id: shellcheck
#       args: ['-x']
```

**SC2155 deserves special attention** — it's subtle and dangerous:

```bash
set -e

# BAD:
local result=$(failing_command)
echo "Got: $result"
# 'local' returns 0, masking the failure. Script continues with empty result.
# set -e DOES NOT CATCH THIS because local succeeded.

# GOOD:
local result
result=$(failing_command)   # If this fails, set -e catches it
echo "Got: $result"
```

---

## 14. PROCESS MANAGEMENT IN SCRIPTS

### Parallel Execution

```bash
# ─── Pattern 1: Background + Wait ─────────────────────────
collect_from_clusters() {
    local clusters=("us-east-1" "us-west-2" "eu-west-1")
    local pids=()

    for cluster in "${clusters[@]}"; do
        (
            info "Collecting from $cluster..."
            kubectl --context "$cluster" get pods -A -o json > "/tmp/pods-${cluster}.json"
        ) &
        pids+=($!)
    done

    # Wait for all and collect exit codes
    local failed=0
    for pid in "${pids[@]}"; do
        if ! wait "$pid"; then
            (( failed++ ))
        fi
    done

    if (( failed > 0 )); then
        error "$failed cluster collection(s) failed"
        return 1
    fi
}

# ─── Pattern 2: xargs for bounded parallelism ─────────────
# Process 5 namespaces at a time
kubectl get namespaces -o jsonpath='{.items[*].metadata.name}' | \
    tr ' ' '\n' | \
    xargs -P 5 -I{} bash -c '
        echo "Processing namespace: {}"
        kubectl get pods -n {} --no-headers | wc -l
    '

# ─── Pattern 3: GNU Parallel (if available) ───────────────
# Drain 3 nodes at a time with progress bar
printf '%s\n' "${nodes_to_drain[@]}" | \
    parallel -j 3 --bar --halt soon,fail=1 \
    './scripts/node-drain.sh --node {} --reason "rolling update"'
```

### Signal Forwarding in Wrapper Scripts

```bash
#!/usr/bin/env bash
# Wrapper that runs app + sidecar and forwards signals to both
set -euo pipefail

# Start sidecar
/app/otel-collector --config /etc/otel/config.yaml &
SIDECAR_PID=$!

# Start main app
/app/payment-service &
APP_PID=$!

# Forward signals to both
shutdown() {
    info "Shutting down..."
    kill -TERM "$APP_PID" 2>/dev/null || true
    # Give app time to drain, then stop sidecar
    wait "$APP_PID" 2>/dev/null || true
    kill -TERM "$SIDECAR_PID" 2>/dev/null || true
    wait "$SIDECAR_PID" 2>/dev/null || true
}
trap shutdown SIGTERM SIGINT

# Wait for either to exit
wait -n  # Wait for FIRST child to exit (bash 4.3+)
EXIT_CODE=$?

# If one died, kill the other
shutdown
exit "$EXIT_CODE"
```

**`wait -n` is powerful but version-dependent:**
- Bash 4.3+: `wait -n` waits for any one child
- Older bash: must poll or wait for specific PIDs

---

## QUICK REFERENCE CARD

```
╔══════════════════════════════════════════════════════════════╗
║                 BASH FOR DEVOPS — CHEAT SHEET                ║
╠══════════════════════════════════════════════════════════════╣
║ HEADER (every script):                                       ║
║   #!/usr/bin/env bash                                        ║
║   set -euo pipefail                                          ║
║   IFS=$'\n\t'                                                ║
║                                                              ║
║ VARIABLES:                                                   ║
║   "${var}"         Always quote                              ║
║   "${var:-default}" Default if unset                         ║
║   "${var:?error}"  Error if unset                            ║
║   "${var%%pattern}" Remove longest suffix                    ║
║   "${var##pattern}" Remove longest prefix                    ║
║   readonly VAR=x   Immutable constant                        ║
║                                                              ║
║ TRAP:                                                        ║
║   trap cleanup EXIT          Always run on exit              ║
║   trap 'onerr $LINENO' ERR  Run on error                     ║
║   trap 'die "SIGINT"' INT   Ctrl+C handler                   ║
║                                                              ║
║ CONDITIONALS:                                                ║
║   [[ -f file ]]    File exists                               ║
║   [[ -d dir ]]     Directory exists                          ║
║   [[ -z "$var" ]]  String is empty                           ║
║   [[ -n "$var" ]]  String is not empty                       ║
║   [[ "$a" == "$b" ]] String equality                         ║
║   [[ "$a" =~ ^regex$ ]] Regex match                          ║
║   (( num > 5 ))    Arithmetic comparison                     ║
║                                                              ║
║ TEXT TOOLS:                                                  ║
║   awk '{print $1}'           Extract field 1                 ║
║   awk -F: '{print $1}'      Custom delimiter                 ║
║   sed 's|old|new|g'         Substitute (| delimiter)         ║
║   sed -i.bak 's|a|b|' f    In-place with backup              ║
║   jq -r '.key'              JSON extract (raw)               ║
║   jq --arg k "$v" '...'    Parameterized jq                  ║
║   jq -e '.check'            Exit 1 if false/null             ║
║   yq '.key' file.yaml       YAML extract                     ║
║   yq -i '.k = "v"' f.yaml  YAML in-place edit                ║
║                                                              ║
║ PATTERNS:                                                    ║
║   retry N DELAY cmd...      Exponential backoff              ║
║   run cmd...                Dry-run wrapper                  ║
║   flock -n FD               Atomic lock (no race)            ║
║   exec "$@"                 Replace shell (containers)       ║
║   cmd & PID=$!; wait $PID   Background + track               ║
║                                                              ║
║ ANTI-PATTERNS:                                               ║
║   ✗ for f in $(ls *.txt)   → for f in *.txt                 ║
║   ✗ eval "$cmd"            → "${cmd[@]}"                    ║
║   ✗ local x=$(cmd)         → local x; x=$(cmd)              ║
║   ✗ echo $var              → echo "$var"                    ║
║   ✗ cd dir; rm -rf .       → cd dir || exit 1               ║
║   ✗ >200 lines bash        → Rewrite in Python/Go           ║
║                                                              ║
║ SHELLCHECK:                                                  ║
║   shellcheck -x script.sh   Lint with source following       ║
║   # shellcheck disable=SC2086  Inline suppression            ║
╚══════════════════════════════════════════════════════════════╝
```


---

## RETENTION QUESTIONS

**Q1 (Scenario — Incident Script):**

It's 2 AM. PagerDuty fires: `payment-service` in `production` namespace is returning 5xx errors. You need to write a quick bash script (on the spot, from memory) that:
1. Gets all pods for `payment-service` in production
2. For each pod: checks if it's Ready, captures its restart count and last restart reason
3. Finds the pod with the most restarts
4. Collects the last 500 lines of logs from that pod (current + previous container)
5. Outputs a summary to stdout AND saves full details to a file

Write the script. Production quality — proper error handling, quoting, and the non-negotiable header. You have 10 minutes.

---

**Q2 (Debugging — What's Wrong?):**

Your teammate wrote this deployment script. It's been "working fine" for months but last night it caused a partial outage. Find **every bug** and explain the production consequence of each:

```bash
#!/bin/bash

ENVIRONMENT=$1
SERVICE=$2
VERSION=$3

cd deploy/$ENVIRONMENT
rm -rf old/*
cp -r current/* old/
cp -r /tmp/build/$VERSION/* current/

kubectl set image deployment/$SERVICE $SERVICE=registry.novamart.com/$SERVICE:$VERSION
kubectl rollout status deployment/$SERVICE --timeout=60s

if [ $? -eq 0 ]; then
    echo "Deploy successful"
    curl -X POST $SLACK_WEBHOOK -d '{"text": "Deployed '$SERVICE' '$VERSION' to '$ENVIRONMENT'"}'
else
    echo "Deploy failed, rolling back"
    kubectl rollout undo deployment/$SERVICE
fi
```

---

**Q3 (Design — Entrypoint):**

You're writing the container entrypoint for a Java Spring Boot microservice (`order-service`) that:
- Needs environment variables: `DB_HOST`, `DB_PORT`, `REDIS_URL`, `KAFKA_BROKERS`
- Has a config template at `/app/config/application.yaml.tpl` that uses `envsubst`
- Must wait for the database to be reachable before starting
- Runs as non-root user (UID 1000)
- Must handle SIGTERM gracefully (Spring Boot has its own shutdown hooks)
- Must log when the container starts and when it receives shutdown signals

Which pattern do you use (exec vs background+wait) and why? Write the complete entrypoint script.

---

**Q4 (Production — jq + awk pipeline):**

Write a one-liner (or short pipeline) for each:

1. Find all pods in the cluster that have been OOMKilled in the last container restart, output: `NAMESPACE/POD_NAME  CONTAINER  RESTART_COUNT`

2. From an Nginx access log, find the top 10 slowest endpoints (by average response time), excluding health checks. Log format: `$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_time`

3. Given a Helm values file, extract all image references (repository + tag) across all services, output as `SERVICE: REPOSITORY:TAG`

# Q1 — Incident Response Script

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

###############################################################################
# incident-triage.sh — Payment Service 5xx Incident Triage
# Usage: ./incident-triage.sh
# Captures pod health, restart counts, and logs for the worst-offending pod
# in the payment-service deployment (production namespace).
###############################################################################

readonly NAMESPACE="production"
readonly SERVICE="payment-service"
readonly LABEL_SELECTOR="app=${SERVICE}"
readonly LOG_LINES=500
readonly OUTFILE="/tmp/incident-${SERVICE}-$(date +%Y%m%d-%H%M%S).txt"

log()   { printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$1" | tee -a "${OUTFILE}"; }
error() { log "ERROR: $1" >&2; }
die()   { error "$1"; exit 1; }

# ── Pre-flight ───────────────────────────────────────────────────────────────
command -v kubectl >/dev/null 2>&1 || die "kubectl not found in PATH"
command -v jq      >/dev/null 2>&1 || die "jq not found in PATH"

kubectl auth can-i get pods -n "${NAMESPACE}" >/dev/null 2>&1 \
    || die "No RBAC permission to read pods in ${NAMESPACE}"

: > "${OUTFILE}"   # truncate / create output file

log "=== Incident Triage: ${SERVICE} in ${NAMESPACE} ==="
log "Output file: ${OUTFILE}"

# ── Step 1: Get all pods ─────────────────────────────────────────────────────
log "--- Fetching pods with selector: ${LABEL_SELECTOR} ---"

pod_json=$(kubectl get pods \
    -n "${NAMESPACE}" \
    -l "${LABEL_SELECTOR}" \
    -o json) || die "Failed to fetch pods"

pod_count=$(printf '%s' "${pod_json}" | jq '.items | length')

if [[ "${pod_count}" -eq 0 ]]; then
    die "No pods found for selector ${LABEL_SELECTOR} in ${NAMESPACE}"
fi

log "Found ${pod_count} pod(s)"

# ── Step 2 & 3: Per-pod health summary, track worst offender ────────────────
log ""
log "--- Pod Health Summary ---"
printf '%-50s %-8s %-10s %s\n' "POD" "READY" "RESTARTS" "LAST RESTART REASON" | tee -a "${OUTFILE}"
printf '%0.s-' {1..110} | tee -a "${OUTFILE}"
echo "" | tee -a "${OUTFILE}"

max_restarts=-1
worst_pod=""
worst_container=""

while IFS= read -r pod_entry; do
    pod_name=$(printf '%s' "${pod_entry}"   | jq -r '.pod')
    ready=$(printf '%s' "${pod_entry}"      | jq -r '.ready')
    restarts=$(printf '%s' "${pod_entry}"   | jq -r '.restarts')
    reason=$(printf '%s' "${pod_entry}"     | jq -r '.reason')
    container=$(printf '%s' "${pod_entry}"  | jq -r '.container')

    printf '%-50s %-8s %-10s %s\n' "${pod_name}" "${ready}" "${restarts}" "${reason}" \
        | tee -a "${OUTFILE}"

    if [[ "${restarts}" -gt "${max_restarts}" ]]; then
        max_restarts="${restarts}"
        worst_pod="${pod_name}"
        worst_container="${container}"
    fi
done < <(printf '%s' "${pod_json}" | jq -c '
    [ .items[] | 
      .metadata.name as $pod |
      .status.containerStatuses[]? |
      {
        pod:       $pod,
        container: .name,
        ready:     .ready,
        restarts:  .restartCount,
        reason:    (
            .lastState.terminated.reason //
            .state.waiting.reason //
            "None"
        )
      }
    ] | .[]
')

log ""

if [[ -z "${worst_pod}" ]]; then
    die "Could not determine worst pod (no container statuses found)"
fi

log "=== Worst offender: ${worst_pod} (container: ${worst_container}, restarts: ${max_restarts}) ==="

# ── Step 4: Collect logs (current + previous) ───────────────────────────────
log ""
log "--- Current container logs (last ${LOG_LINES} lines) ---"
{
    echo "====== CURRENT LOGS: ${worst_pod} / ${worst_container} ======"
    kubectl logs \
        -n "${NAMESPACE}" \
        "${worst_pod}" \
        -c "${worst_container}" \
        --tail="${LOG_LINES}" 2>&1 || echo "(failed to retrieve current logs)"
} | tee -a "${OUTFILE}"

log ""
log "--- Previous container logs (last ${LOG_LINES} lines) ---"
{
    echo "====== PREVIOUS LOGS: ${worst_pod} / ${worst_container} ======"
    kubectl logs \
        -n "${NAMESPACE}" \
        "${worst_pod}" \
        -c "${worst_container}" \
        --previous \
        --tail="${LOG_LINES}" 2>&1 || echo "(no previous container logs available)"
} | tee -a "${OUTFILE}"

# ── Step 5: Additional context ───────────────────────────────────────────────
log ""
log "--- Pod describe (events) ---"
{
    echo "====== DESCRIBE: ${worst_pod} ======"
    kubectl describe pod \
        -n "${NAMESPACE}" \
        "${worst_pod}" 2>&1 || echo "(failed to describe pod)"
} >> "${OUTFILE}"   # describe is verbose; file only

log ""
log "============================================================"
log "  SUMMARY"
log "============================================================"
log "  Service:        ${SERVICE}"
log "  Namespace:      ${NAMESPACE}"
log "  Total pods:     ${pod_count}"
log "  Worst pod:      ${worst_pod}"
log "  Container:      ${worst_container}"
log "  Restart count:  ${max_restarts}"
log "  Full details:   ${OUTFILE}"
log "============================================================"

exit 0
```

---

# Q2 — Bug Audit

Here is **every bug**, its root cause, and its production consequence:

---

### Bug 1: Missing `set -euo pipefail` (and no arg validation)

```bash
ENVIRONMENT=$1
SERVICE=$2
VERSION=$3
```

**Problem:** If any argument is empty or omitted, every subsequent variable expansion is an empty string. `$ENVIRONMENT` being empty means `cd deploy/` goes to the wrong directory. `$SERVICE` being empty means `kubectl set image deployment/ ...` acts on a bare resource name. `$VERSION` empty means you deploy with tag `:` (empty tag resolves to `:latest` or errors).

**Production consequence:** Last night someone likely called the script with args in the wrong order or missing one. Without `set -u`, the script **silently continued** with empty variables. This is almost certainly the root cause of the outage.

**Fix:**
```bash
set -euo pipefail
[[ $# -eq 3 ]] || { echo "Usage: $0 <env> <service> <version>"; exit 1; }
```

---

### Bug 2: `cd deploy/$ENVIRONMENT` — unquoted and unchecked

```bash
cd deploy/$ENVIRONMENT
```

**Problem (a):** No error check. If the directory doesn't exist, `cd` fails but the script **continues in the current directory**. The next line `rm -rf old/*` now deletes `old/*` relative to wherever the script was invoked — potentially a completely unrelated directory.

**Problem (b):** Unquoted. If `$ENVIRONMENT` contains spaces or glob characters, word splitting or pathname expansion occurs.

**Production consequence:** If the directory was missing (e.g., a typo: `prod` vs `production`), `rm -rf old/*` and `cp` commands operated against the wrong path, corrupting files outside the deploy tree.

**Fix:**
```bash
cd "deploy/${ENVIRONMENT}" || die "Cannot cd to deploy/${ENVIRONMENT}"
```

---

### Bug 3: `rm -rf old/*` before confirming `cp` succeeds — no atomic backup

```bash
rm -rf old/*
cp -r current/* old/
```

**Problem:** The old backup is **destroyed before** the new backup is written. If `cp` fails (disk full, permission error), you now have **no backup and no rollback path** on the filesystem level.

**Production consequence:** In a failure scenario, the rollback artifacts are gone. You can't manually revert the filesystem deployment.

**Fix:**
```bash
rm -rf old.bak/
cp -r old/ old.bak/          # keep safety net
rm -rf old/*
cp -r current/* old/ || { echo "Backup failed"; exit 1; }
```

---

### Bug 4: Glob `*` with empty directories

```bash
rm -rf old/*
cp -r current/* old/
```

**Problem:** If `current/` is empty, the glob `current/*` expands to the literal string `current/*`, and `cp` fails with "No such file or directory." With `set -e` (if added), this kills the script. Without it, the script continues and deploys with an empty `current/` dir.

**Fix:** Use `cp -r current/. old/` (dot syntax copies directory contents safely) or check for emptiness first.

---

### Bug 5: `$?` is **not** checking `kubectl rollout status` — it checks `kubectl set image`... actually no, let me re-read.

```bash
kubectl set image deployment/$SERVICE $SERVICE=registry.novamart.com/$SERVICE:$VERSION
kubectl rollout status deployment/$SERVICE --timeout=60s

if [ $? -eq 0 ]; then
```

**Problem:** `$?` captures the exit code of the **last command** — `kubectl rollout status`, which is correct in isolation. **But**: `kubectl set image` on the line above is **completely unchecked**. If `set image` fails (invalid image, wrong deployment name, RBAC error), it silently fails, then `rollout status` may report success on the **old rollout** (nothing changed), and the script prints "Deploy successful" even though the image was never updated.

**Production consequence:** False-positive success. The team thinks v2.3.1 is deployed but the pods still run v2.3.0. Traffic serves stale code. The Slack message lies.

**Fix:**
```bash
kubectl set image "deployment/${SERVICE}" \
    "${SERVICE}=registry.novamart.com/${SERVICE}:${VERSION}" \
    || die "Failed to set image"

if ! kubectl rollout status "deployment/${SERVICE}" --timeout=60s; then
    # rollback
fi
```

---

### Bug 6: 60-second rollout timeout is dangerously short

```bash
kubectl rollout status deployment/$SERVICE --timeout=60s
```

**Problem:** Large deployments, slow image pulls, or readiness probe startup delays can easily exceed 60 seconds. The script treats timeout as **failure** and triggers a rollback — even if the deployment was healthy and just slow.

**Production consequence:** A slow but healthy deploy gets rolled back mid-rollout, causing unnecessary churn and potentially a partial outage (exactly what happened).

**Fix:** Use a reasonable timeout (300s+), or make it configurable.

---

### Bug 7: Unquoted variables throughout — injection and word-splitting

```bash
cd deploy/$ENVIRONMENT          # unquoted
deployment/$SERVICE             # unquoted
$SLACK_WEBHOOK                  # unquoted
```

**Production consequence:** If any value contains spaces, glob characters, or is empty, bash performs word splitting and pathname expansion. A malicious or accidental version string like `v1.0 ; rm -rf /` would be catastrophic in certain contexts.

**Fix:** Double-quote every expansion: `"${ENVIRONMENT}"`, `"${SERVICE}"`, etc.

---

### Bug 8: `$SLACK_WEBHOOK` — unquoted, unvalidated, and the JSON is broken

```bash
curl -X POST $SLACK_WEBHOOK -d '{"text": "Deployed '$SERVICE' '$VERSION' to '$ENVIRONMENT'"}'
```

**Problem (a):** `$SLACK_WEBHOOK` is unquoted. If unset, `curl -X POST -d ...` sends a POST to `-d` (interpreted as URL), causing bizarre errors.

**Problem (b):** The JSON construction uses single-quote-break-out: `'...'$VAR'...'`. The variables are **unquoted in the shell** and are **not JSON-escaped**. If `$SERVICE` contains a double quote, backslash, or newline, the JSON payload is malformed and the Slack notification silently fails.

**Problem (c):** `curl` failure is not checked. You never know if the team was actually notified.

**Production consequence:** The team may not receive deployment notifications. In outage scenarios, nobody knows what version is deployed or when it happened.

**Fix:**
```bash
if [[ -n "${SLACK_WEBHOOK:-}" ]]; then
    payload=$(jq -n \
        --arg svc "${SERVICE}" \
        --arg ver "${VERSION}" \
        --arg env "${ENVIRONMENT}" \
        '{"text": "Deployed \($svc) \($ver) to \($env)"}')
    curl -sf -X POST "${SLACK_WEBHOOK}" \
        -H 'Content-Type: application/json' \
        -d "${payload}" || echo "WARNING: Slack notification failed" >&2
fi
```

---

### Bug 9: No `Content-Type` header on the curl request

The `curl` call is missing `-H 'Content-Type: application/json'`. Slack's webhook API may reject or misinterpret the payload.

---

### Bug 10: The rollback path has no verification or notification

```bash
echo "Deploy failed, rolling back"
kubectl rollout undo deployment/$SERVICE
```

**Problem:** The `rollout undo` itself is not checked for success. If the undo fails, the script exits 0 (success). No Slack notification is sent on failure — the team sleeps through the outage.

**Production consequence:** A failed deploy + failed rollback = the service is in an unknown state, and nobody is notified.

---

### Bug 11: No shebang best-practice (`#!/usr/bin/env bash` vs `#!/bin/bash`)

Minor, but `#!/bin/bash` assumes bash is at `/bin/bash`. On some systems (NixOS, FreeBSD, some containers), it isn't.

---

### Bug 12: Script exits 0 on failure

After the rollback branch, the script falls through with an implicit `exit 0`. Callers (CI/CD pipelines) see success even when the deploy failed and was rolled back.

**Fix:** `exit 1` in the else branch.

---

### Corrected Script

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly ENVIRONMENT="${1:?Usage: $0 <environment> <service> <version>}"
readonly SERVICE="${2:?Usage: $0 <environment> <service> <version>}"
readonly VERSION="${3:?Usage: $0 <environment> <service> <version>}"

die() { echo "FATAL: $1" >&2; exit 1; }

cd "deploy/${ENVIRONMENT}" || die "Cannot cd to deploy/${ENVIRONMENT}"

cp -a old/ "old.bak.$(date +%s)/"
rm -rf old/*
cp -r current/. old/     || die "Backup of current -> old failed"
cp -r "/tmp/build/${VERSION}/." current/ || die "Copy of build ${VERSION} failed"

kubectl set image "deployment/${SERVICE}" \
    "${SERVICE}=registry.novamart.com/${SERVICE}:${VERSION}" \
    || die "kubectl set image failed"

if kubectl rollout status "deployment/${SERVICE}" --timeout=300s; then
    echo "Deploy successful"
    notify_text="✅ Deployed ${SERVICE} ${VERSION} to ${ENVIRONMENT}"
else
    echo "Deploy failed, rolling back" >&2
    kubectl rollout undo "deployment/${SERVICE}" \
        || die "CRITICAL: Rollback also failed!"
    kubectl rollout status "deployment/${SERVICE}" --timeout=300s \
        || die "CRITICAL: Rollback did not stabilize!"
    notify_text="🚨 Deploy FAILED & rolled back: ${SERVICE} ${VERSION} in ${ENVIRONMENT}"
    deploy_failed=1
fi

if [[ -n "${SLACK_WEBHOOK:-}" ]]; then
    payload=$(jq -n --arg t "${notify_text}" '{"text":$t}')
    curl -sf -X POST "${SLACK_WEBHOOK}" \
        -H 'Content-Type: application/json' \
        -d "${payload}" || echo "WARNING: Slack notification failed" >&2
fi

[[ "${deploy_failed:-0}" -eq 1 ]] && exit 1
exit 0
```

---

# Q3 — Container Entrypoint Design

## Pattern Choice: `exec` — and here's why

**`exec` replaces the shell with the Java process** so that:

1. **PID 1 is the JVM.** `SIGTERM` from Kubernetes goes directly to Spring Boot's shutdown hooks. No signal-forwarding gymnastics needed.
2. **No zombie processes.** The shell is gone; the JVM is the init process.
3. **`docker stop` / `kubectl delete` works correctly.** The container terminates when the JVM terminates, with the correct exit code.
4. **Spring Boot's graceful shutdown** (`server.shutdown=graceful` in Spring Boot 2.3+) handles connection draining natively — we just need to deliver the signal.

The background+wait pattern is necessary only when you need the shell to **remain alive after starting the app** (e.g., to run a sidecar, multiplex signals to multiple children, or do post-startup work). None of those apply here.

## Complete Entrypoint

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

###############################################################################
# entrypoint.sh — order-service container entrypoint
# Pattern: exec (JVM becomes PID 1, receives signals directly)
# Expects: DB_HOST, DB_PORT, REDIS_URL, KAFKA_BROKERS
# Config:  /app/config/application.yaml.tpl -> application.yaml via envsubst
###############################################################################

readonly APP_DIR="/app"
readonly CONFIG_TPL="${APP_DIR}/config/application.yaml.tpl"
readonly CONFIG_OUT="${APP_DIR}/config/application.yaml"
readonly JAR="${APP_DIR}/order-service.jar"
readonly DB_WAIT_TIMEOUT="${DB_WAIT_TIMEOUT:-30}"
readonly DB_WAIT_INTERVAL="${DB_WAIT_INTERVAL:-2}"

# ── Logging ──────────────────────────────────────────────────────────────────
log()  { printf '[entrypoint %s] %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$1"; }
die()  { log "FATAL: $1" >&2; exit 1; }

# ── Signal Handling (pre-exec: log shutdown signal; post-exec: JVM handles) ─
shutdown_handler() {
    log "Received SIGTERM — passing to JVM (PID 1 after exec)"
    # This handler only runs if exec hasn't happened yet (during init).
    # After exec, the JVM is PID 1 and receives SIGTERM directly.
    exit 143  # 128 + 15
}
trap shutdown_handler SIGTERM SIGINT

log "Starting order-service container (PID $$, UID $(id -u))"

# ── Validate UID ─────────────────────────────────────────────────────────────
if [[ "$(id -u)" -eq 0 ]]; then
    die "Container must not run as root. Expected UID 1000, got 0."
fi
log "Running as UID $(id -u), GID $(id -g)"

# ── Validate Required Environment Variables ──────────────────────────────────
required_vars=(DB_HOST DB_PORT REDIS_URL KAFKA_BROKERS)
missing=()
for var in "${required_vars[@]}"; do
    if [[ -z "${!var:-}" ]]; then
        missing+=("${var}")
    fi
done
if [[ ${#missing[@]} -gt 0 ]]; then
    die "Missing required environment variables: ${missing[*]}"
fi

log "Environment validated: DB_HOST=${DB_HOST}, DB_PORT=${DB_PORT}"

# ── Render Config Template ───────────────────────────────────────────────────
if [[ ! -f "${CONFIG_TPL}" ]]; then
    die "Config template not found: ${CONFIG_TPL}"
fi

# Only export the variables we want substituted (defense against envsubst
# replacing unrelated $VAR references in the YAML).
envsubst '${DB_HOST} ${DB_PORT} ${REDIS_URL} ${KAFKA_BROKERS}' \
    < "${CONFIG_TPL}" \
    > "${CONFIG_OUT}" \
    || die "Failed to render config template"

log "Configuration rendered: ${CONFIG_OUT}"

# ── Wait for Database ────────────────────────────────────────────────────────
log "Waiting for database at ${DB_HOST}:${DB_PORT} (timeout: ${DB_WAIT_TIMEOUT}s)"

elapsed=0
until nc -z -w 2 "${DB_HOST}" "${DB_PORT}" 2>/dev/null; do
    if [[ "${elapsed}" -ge "${DB_WAIT_TIMEOUT}" ]]; then
        die "Database ${DB_HOST}:${DB_PORT} not reachable after ${DB_WAIT_TIMEOUT}s"
    fi
    log "Database not ready — retrying in ${DB_WAIT_INTERVAL}s (${elapsed}/${DB_WAIT_TIMEOUT}s)"
    sleep "${DB_WAIT_INTERVAL}"
    elapsed=$((elapsed + DB_WAIT_INTERVAL))
done

log "Database is reachable"

# ── Build JVM Args ───────────────────────────────────────────────────────────
JVM_OPTS=(
    -server
    -XX:+UseContainerSupport
    -XX:MaxRAMPercentage=75.0
    -XX:+ExitOnOutOfMemoryError
    -Djava.security.egd=file:/dev/./urandom
    -Dspring.config.additional-location="file:${CONFIG_OUT}"
)

# Allow operators to inject additional JVM flags via JAVA_OPTS env var
if [[ -n "${JAVA_OPTS:-}" ]]; then
    read -ra extra_opts <<< "${JAVA_OPTS}"
    JVM_OPTS+=("${extra_opts[@]}")
fi

log "Launching JVM: java ${JVM_OPTS[*]} -jar ${JAR}"
log "=== entrypoint init complete — exec to JVM ==="

# ── Exec: Replace shell with JVM (JVM becomes PID 1) ────────────────────────
# From this point, SIGTERM goes directly to the JVM.
# Spring Boot's graceful shutdown hooks handle connection draining.
exec java "${JVM_OPTS[@]}" -jar "${JAR}" "$@"
```

### Key design decisions annotated:

| Decision | Rationale |
|---|---|
| `exec java` as final line | JVM becomes PID 1; receives SIGTERM directly; shell is gone |
| `trap` before `exec` | Catches signals during init phase (DB wait could take 30s) |
| `envsubst` with explicit var list | Prevents accidental substitution of `${}` in YAML that aren't ours |
| `nc -z` for DB wait | Lightweight TCP check; no client libraries needed; available in slim images |
| `-XX:+UseContainerSupport` | JVM respects cgroup memory/CPU limits |
| `-XX:+ExitOnOutOfMemoryError` | OOM → JVM exits → Kubernetes restarts → no limbo state |
| UID 0 check | Defense-in-depth: even if Dockerfile/securityContext is misconfigured, fail loud |
| `"$@"` on exec line | Allows CMD overrides from docker-compose / k8s args |

---

# Q4 — Production jq + awk Pipelines

### 1. Find all OOMKilled pods

```bash
kubectl get pods \
    --all-namespaces \
    -o json \
  | jq -r '
    .items[] |
    .metadata.namespace as $ns |
    .metadata.name as $pod |
    (.status.containerStatuses // [])[] |
    select(.lastState.terminated.reason == "OOMKilled") |
    "\($ns)/\($pod)\t\(.name)\t\(.restartCount)"
  ' | column -t -s $'\t'
```

Output:
```
payments/payment-api-7f8b9-xk2lz    payment-api    14
orders/order-worker-5c4d8-rm9np      order-worker   7
```

**Why it works:** `.lastState.terminated` captures the **last** container restart. We filter on `.reason == "OOMKilled"` exactly as the question specifies. The `// []` guard prevents errors on pods with no `containerStatuses` (pending pods).

---

### 2. Top 10 slowest endpoints by average response time (excluding health checks)

```bash
awk '
    $7 !~ /^\/(health|ready|alive|livez|readyz|healthz)/ && NF >= 10 {
        # $7 = URI path (from "$request" = "$method $uri $proto")
        # $NF = $request_time (last field)
        split($6, method, "\"");   # $6 starts with "METHOD
        split($7, uri_parts, "?"); # strip query string
        endpoint = method[2] " " uri_parts[1];
        sum[endpoint] += $NF;
        count[endpoint]++;
    }
    END {
        for (ep in sum)
            printf "%.4f\t%d\t%s\n", sum[ep]/count[ep], count[ep], ep
    }
' /var/log/nginx/access.log \
  | sort -t$'\t' -k1 -rn \
  | head -10 \
  | awk -F'\t' 'BEGIN {printf "%-12s %-8s %s\n","AVG_TIME","HITS","ENDPOINT"}
                      {printf "%-12s %-8s %s\n", $1"s", $2, $3}'
```

Output:
```
AVG_TIME     HITS     ENDPOINT
2.3417s      342      POST /api/v1/orders/bulk
1.8921s      1208     GET /api/v1/reports/generate
0.9843s      89       POST /api/v1/payments/process
...
```

**Parsing logic:** In the standard Nginx combined log format with `$request_time` appended, the request field is quoted (`"GET /path HTTP/1.1"`). `$6` captures `"GET`, `$7` captures `/path`, and `$NF` captures the last field — `request_time`.

---

### 3. Extract all image references from a Helm values file

Assuming the standard Helm values convention where services define `image.repository` and `image.tag`:

```bash
# Standard structure:
# serviceA:
#   image:
#     repository: registry.novamart.com/service-a
#     tag: "v1.2.3"

yq -r '
    .. | objects |
    select(has("image")) |
    to_entries[] |
    select(.value | type == "object" and has("repository")) |
    .key as $svc |
    .value |
    "\($svc): \(.repository):\(.tag // "latest")"
' values.yaml

# If yq is unavailable, pure jq approach (convert YAML first):
python3 -c 'import sys,yaml,json; json.dump(yaml.safe_load(sys.stdin),sys.stdout)' \
    < values.yaml \
  | jq -r '
    paths(objects | has("repository","tag")) as $p |
    "\($p[-1]): \(getpath($p).repository):\(getpath($p).tag // "latest")"
  '
```

For a more **robust** approach that handles all common patterns (including inline `image: repo:tag` strings):

```bash
yq -r '
    # Pattern 1: .service.image.repository + .service.image.tag
    (to_entries[] |
     select(.value.image?.repository?) |
     "\(.key): \(.value.image.repository):\(.value.image.tag // "latest")"),

    # Pattern 2: .service.image = "repo:tag" (string shorthand)
    (to_entries[] |
     select(.value.image? | type == "string") |
     "\(.key): \(.value.image)")
' values.yaml | sort -u
```

Output:
```
api-gateway: registry.novamart.com/api-gateway:v2.1.0
order-service: registry.novamart.com/order-service:v3.4.2
payment-service: registry.novamart.com/payment-service:v1.8.7
redis: docker.io/library/redis:7-alpine
```

# Honest Answer: No. That's Maybe 65% of What You Need.

I caught some gaps in my self-evaluation but was too generous with a 9/10. Let me be real about what's missing.

---

## What I Actually Missed

### Category 1: Bash Language Features (Daily Use)

**Process Substitution `<()` — Used constantly, never taught:**
```bash
# Compare two sorted lists without temp files
diff <(kubectl get pods -n staging -o name | sort) \
     <(kubectl get pods -n production -o name | sort)

# Feed command output as a "file" to tools that need file arguments
while IFS= read -r line; do
    process "$line"
done < <(kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}')

# Why < <() and not pipe?
# Pipe creates a SUBSHELL for the while loop.
# Variables set inside the loop are LOST when the subshell exits.

# BROKEN:
count=0
kubectl get pods -o name | while IFS= read -r pod; do
    (( count++ ))
done
echo "$count"  # ALWAYS 0 — count was modified in subshell

# FIXED with process substitution:
count=0
while IFS= read -r pod; do
    (( count++ ))
done < <(kubectl get pods -o name)
echo "$count"  # Correct count — no subshell
```

This is **not an edge case.** Every script that reads command output line-by-line and needs to track state runs into the pipe-subshell trap. I should have taught this.

---

**String Manipulation (Parameter Expansion) — mentioned in cheat sheet, never taught:**
```bash
filepath="/var/log/nginx/access.log.2024-01-15.gz"

# ── Extraction ──────────────────────────────────────────
${filepath##*/}          # access.log.2024-01-15.gz  (basename — remove longest prefix match)
${filepath%/*}           # /var/log/nginx            (dirname — remove shortest suffix match)
${filepath%%.*}          # /var/log/nginx/access     (remove longest suffix match)
${filepath#*.}           # log.2024-01-15.gz         (remove shortest prefix match)

# ── Substitution ────────────────────────────────────────
version="v1.2.3-beta"
${version/v/}            # 1.2.3-beta                (replace first match)
${version//-/.}          # v1.2.3.beta               (replace ALL matches — double /)
${version%-*}            # v1.2.3                    (strip pre-release suffix)

# ── Length and Slicing ──────────────────────────────────
${#version}              # 12                        (string length)
${version:1:5}           # 1.2.3                     (substring: offset:length)

# ── Case Conversion (Bash 4+) ──────────────────────────
env="Production"
${env,,}                 # production                (lowercase all)
${env^^}                 # PRODUCTION                (uppercase all)
${env,}                  # production                (lowercase first)

# ── Default Values (covered briefly, but not comprehensively) ──
${var:-default}          # Use default if unset OR empty
${var-default}           # Use default if unset ONLY (empty string is valid)
${var:=default}          # Set AND use default if unset/empty
${var:+alternate}        # Use alternate if var IS set (inverse logic)
${var:?error msg}        # Error if unset/empty

# ── Production Example: Parse image reference ───────────
image="registry.novamart.com/payment-service:sha-abc123"
registry="${image%%/*}"                    # registry.novamart.com
name_tag="${image#*/}"                     # payment-service:sha-abc123
name="${name_tag%%:*}"                     # payment-service
tag="${name_tag#*:}"                       # sha-abc123

# ── Why this matters vs calling external tools ──────────
# BAD (spawns 4 subprocesses):
registry=$(echo "$image" | cut -d/ -f1)
tag=$(echo "$image" | cut -d: -f2)
# In a loop over 1000 images, that's 4000 fork+exec calls.

# GOOD (pure bash, zero subprocesses):
registry="${image%%/*}"
tag="${image##*:}"
# In a loop: milliseconds vs seconds.
```

---

**Regex in Bash — `BASH_REMATCH` capture groups:**
```bash
# Extract version components
version="v3.14.2-rc1"
if [[ "$version" =~ ^v([0-9]+)\.([0-9]+)\.([0-9]+)(-(.+))?$ ]]; then
    major="${BASH_REMATCH[1]}"   # 3
    minor="${BASH_REMATCH[2]}"   # 14
    patch="${BASH_REMATCH[3]}"   # 2
    prerelease="${BASH_REMATCH[5]}"  # rc1 (group 5 because group 4 includes the -)
    echo "Parsed: ${major}.${minor}.${patch} pre=${prerelease}"
else
    echo "Invalid version format"
fi

# Extract timestamp from log line
log_line='[2024-01-15T14:32:07Z] ERROR [payment:42] Connection refused'
if [[ "$log_line" =~ ^\[([^\]]+)\]\ ([A-Z]+)\ \[([^:]+):([0-9]+)\]\ (.+)$ ]]; then
    timestamp="${BASH_REMATCH[1]}"   # 2024-01-15T14:32:07Z
    level="${BASH_REMATCH[2]}"       # ERROR
    source="${BASH_REMATCH[3]}"      # payment
    line_no="${BASH_REMATCH[4]}"     # 42
    message="${BASH_REMATCH[5]}"     # Connection refused
fi

# GOTCHA: Do NOT quote the regex
[[ "$var" =~ "^pattern$" ]]   # WRONG — treats as literal string
[[ "$var" =~ ^pattern$ ]]     # RIGHT — treats as regex

# GOTCHA: Store complex regex in a variable to avoid quoting issues
pattern='^([0-9]{1,3}\.){3}[0-9]{1,3}$'
[[ "$ip" =~ $pattern ]] && echo "Valid IP format"
```

---

**Associative Arrays — Lookup tables, dedup, counting:**
```bash
# Declare
declare -A service_ports=(
    [payment-service]=8080
    [order-service]=8081
    [user-service]=8082
    [notification-service]=8083
)

# Access
echo "${service_ports[payment-service]}"  # 8080

# Iterate
for svc in "${!service_ports[@]}"; do     # keys
    echo "$svc -> ${service_ports[$svc]}"  # key -> value
done

# Check existence
if [[ -v service_ports[payment-service] ]]; then
    echo "Port defined"
fi

# ── Production: Count pods per status ───────────────────
declare -A status_count
while IFS= read -r status; do
    (( status_count[$status]++ )) || true  # || true for set -e safety (first increment = 0)
done < <(kubectl get pods -n production -o jsonpath='{range .items[*]}{.status.phase}{"\n"}{end}')

for status in "${!status_count[@]}"; do
    echo "$status: ${status_count[$status]}"
done

# ── Production: Deduplication ───────────────────────────
declare -A seen
while IFS= read -r image; do
    if [[ -v seen[$image] ]]; then
        continue
    fi
    seen[$image]=1
    echo "Scanning: $image"
    trivy image --severity HIGH,CRITICAL "$image"
done < <(kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}')
```

---

**`mapfile`/`readarray` — Proper line-to-array:**
```bash
# BAD: Word splitting breaks on spaces, glob expansion is dangerous
files=( $(find /var/log -name "*.log") )

# GOOD:
mapfile -t files < <(find /var/log -name "*.log")
# -t strips trailing newlines

# GOOD with null delimiter (handles ALL filenames including newlines):
mapfile -t -d '' files < <(find /var/log -name "*.log" -print0)

# Read specific number of lines
mapfile -t -n 10 recent_logs < <(tail -100 /var/log/app.log | grep ERROR)

# This replaces the ugly pattern:
IFS=$'\n' read -r -d '' -a files < <(find ... && printf '\0')
```

---

**Here strings `<<<`:**
```bash
# Feed a string directly as stdin (no echo | pipe)
read -r first rest <<< "hello world"
# first=hello, rest=world

# Parse a variable
IFS=: read -r user _ uid _ _ home shell <<< "$(getent passwd root)"

# Check if string contains substring
if grep -q "error" <<< "$log_line"; then
    alert
fi

# Feed to jq
result=$(jq -r '.status' <<< "$json_response")

# vs pipe (spawns subshell):
result=$(echo "$json_response" | jq -r '.status')
```

---

**Subshells vs Command Groups — Scope matters:**
```bash
# () = subshell — separate process, separate variable scope
(
    cd /tmp
    VAR="changed"
    # cd and VAR changes DON'T affect parent
)
echo "$VAR"  # unchanged
pwd           # unchanged

# {} = command group — same process, same scope
{
    cd /tmp
    VAR="changed"
    # cd and VAR changes DO affect current shell
}
echo "$VAR"  # "changed"
pwd           # /tmp

# When subshells matter:
# 1. Pipe creates subshell (the pipe-subshell trap above)
# 2. Command substitution $() creates subshell
# 3. Background & creates subshell
# 4. Explicit () creates subshell

# Production gotcha: Isolating risky operations
(
    set +e  # disable errexit in subshell only
    risky_command_that_might_fail
    EXIT_CODE=$?
    echo "$EXIT_CODE"  # captured
)
# Parent shell still has set -e active
```

---

**`printf` vs `echo` — Portability and formatting:**
```bash
# echo is non-portable:
echo -e "hello\tworld"   # Works in bash, not in all sh
echo -n "no newline"     # Works in bash, not in all sh

# printf is POSIX, predictable:
printf 'hello\tworld\n'
printf 'no newline'
printf '%s\t%s\n' "key" "value"

# printf for formatted output (numbers, padding):
printf '%-30s %6.2f%%\n' "payment-service" 99.95
printf '%-30s %6.2f%%\n' "order-service" 98.12
# payment-service                99.95%
# order-service                  98.12%

# printf for safer variable output:
name='%s exploited\n'
echo "$name"     # Prints: %s exploited\n (safe in echo, but...)
printf "$name"   # DANGER: %s is a format specifier!
printf '%s\n' "$name"  # SAFE: always use format string + argument

# printf -v: Assign to variable without subshell
printf -v timestamp '%(%Y-%m-%dT%H:%M:%S)T' -1  # -1 = current time (Bash 4.2+)
# No $(date ...) subprocess needed!
```

---

### Category 2: Debugging Bash Scripts

**This was completely missing from my lesson.**

```bash
# ── set -x: Trace execution ────────────────────────────
set -x  # Enable: prints every command before execution
set +x  # Disable

# Selective tracing (don't dump entire script):
some_safe_function

set -x
problematic_function
set +x

the_rest_of_the_script

# ── PS4: Customize trace prefix ────────────────────────
# Default PS4 is '+ ' (boring)
export PS4='+${BASH_SOURCE}:${LINENO}: ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
# Now traces show: +script.sh:47: deploy(): kubectl apply -f deployment.yaml

# ── Trace to a file (don't pollute stderr) ─────────────
exec 4>/tmp/debug-trace.log
BASH_XTRACEFD=4
set -x
# Trace goes to /tmp/debug-trace.log, stderr stays clean for real errors

# ── DEBUG trap: Run command before every line ───────────
trap 'echo "DEBUG: line $LINENO, command: $BASH_COMMAND"' DEBUG
# Useful for understanding execution flow

# ── Conditional debugging ──────────────────────────────
if [[ "${DEBUG:-false}" == true ]]; then
    set -x
    export PS4='+${BASH_SOURCE}:${LINENO}: '
fi
# Usage: DEBUG=true ./deploy.sh --env production

# ── Common debugging patterns ──────────────────────────
# 1. Print variable state at key points
debug_vars() {
    local func="${FUNCNAME[1]}"
    echo "=== DEBUG: $func ===" >&2
    for var in "$@"; do
        echo "  $var = '${!var:-<unset>}'" >&2
    done
}
# Usage: debug_vars ENVIRONMENT SERVICE VERSION

# 2. Dry-run + trace combo
if [[ "$DRY_RUN" == true ]]; then
    set -x  # Show exactly what WOULD run
fi
```

---

### Category 3: File Descriptors and Advanced Redirection

```bash
# ── Basics review + what I missed ──────────────────────
# 0 = stdin, 1 = stdout, 2 = stderr

# Redirect stderr to file, stdout to terminal:
command 2>/var/log/errors.log

# Redirect both stdout and stderr to file:
command &>/var/log/all.log      # Bash shorthand
command >/var/log/all.log 2>&1  # POSIX (ORDER MATTERS: redirect stdout first, then dup stderr to stdout)

# WRONG ORDER:
command 2>&1 >/var/log/output.log
# stderr goes to terminal (was duped to stdout BEFORE stdout was redirected)

# ── Custom file descriptors ────────────────────────────
# Open FD 3 for writing to a log file
exec 3>/var/log/audit.log

# Write to it
echo "Audit: deployment started" >&3
echo "Audit: image set to $IMAGE" >&3

# Close it
exec 3>&-

# ── Read and write simultaneously ──────────────────────
# Open FD 4 for reading from a config file
exec 4<config.txt
while IFS= read -r line <&4; do
    process "$line"
done
exec 4<&-

# ── Capture stderr separately ─────────────────────────
# Run command, capture stdout in var, stderr in file
output=$(command 2>/tmp/stderr.log)
stderr=$(cat /tmp/stderr.log)

# Without temp file (Bash 4+ trick with FD swapping):
{ output=$(command 2>&1 1>&3 3>&-); } 3>&1
# Dark magic. In practice, just use the temp file approach.

# ── Practical: Tee to multiple destinations ────────────
# Log stdout AND stderr to file while still showing on terminal
deploy_service 2>&1 | tee -a /var/log/deploy.log

# Separate streams:
exec 1> >(tee -a /var/log/stdout.log)
exec 2> >(tee -a /var/log/stderr.log)
# Now ALL stdout and stderr in script go to both terminal and log files
```

---

### Category 4: Glob Patterns and Options

```bash
# ── shopt options that matter ──────────────────────────

# nullglob: Unmatched globs expand to nothing (not literal)
shopt -s nullglob
files=( /var/log/*.log.gz )
echo "${#files[@]}"  # 0 if no matches (instead of literal "*.log.gz")

# globstar: ** matches recursively
shopt -s globstar
for f in /app/**/*.yaml; do   # All YAML files at any depth
    yamllint "$f"
done

# extglob: Extended patterns
shopt -s extglob
case "$ENVIRONMENT" in
    @(dev|staging|qa))     echo "Non-production" ;;  # Match any of
    !(production))         echo "Not production" ;;   # Negate
    +([0-9]))              echo "All digits" ;;       # One or more
esac

# failglob: Error on unmatched glob (instead of passing literal)
shopt -s failglob
ls /nonexistent/*.txt  # Error: no match, instead of "ls /nonexistent/*.txt"

# nocasematch: Case-insensitive matching
shopt -s nocasematch
if [[ "$input" == "yes" ]]; then  # Matches YES, Yes, yes, etc.
    proceed
fi
```

---

### Category 5: Functions — Proper Patterns

```bash
# ── Return values (the bash trap) ──────────────────────
# return only returns 0-255 (exit code). NOT arbitrary data.

# BAD: Can't return strings with "return"
get_version() {
    return "v1.2.3"  # WRONG — return takes an integer
}

# PATTERN 1: Echo the result (most common)
get_version() {
    local version
    version=$(kubectl get deployment "$1" -o jsonpath='{.spec.template.spec.containers[0].image}')
    echo "${version##*:}"  # stdout IS the return value
}
result=$(get_version payment-service)  # Captures stdout

# PATTERN 2: Nameref (Bash 4.3+, avoids subshell)
get_version() {
    local -n result_ref="$1"  # nameref: alias for caller's variable
    result_ref=$(kubectl get deployment "$2" -o jsonpath='{.spec.template.spec.containers[0].image}')
    result_ref="${result_ref##*:}"
}
get_version my_var payment-service
echo "$my_var"  # No subshell! Variable set directly.

# ── Passing arrays to functions ────────────────────────
process_services() {
    local -a services=("$@")  # Capture all args as array
    for svc in "${services[@]}"; do
        echo "Processing: $svc"
    done
}
my_services=("payment" "order" "notification")
process_services "${my_services[@]}"

# ── Local scope is critical ────────────────────────────
bad_function() {
    count=42       # GLOBAL — pollutes caller's namespace
}

good_function() {
    local count=42  # Local — dies when function returns
}

# GOTCHA: local doesn't work in subshells
my_func() {
    local result
    result=$(some_command)  # Subshell! If some_command fails,
                             # 'local' masks the exit code
}
# FIX (SC2155):
my_func() {
    local result
    result=$(some_command)  # Now failure is caught by set -e
}
```

---

### Category 6: Performance

```bash
# ── The #1 bash performance killer: Subshells in loops ──
# BAD (spawns a process for EVERY pod):
while IFS= read -r pod; do
    status=$(echo "$pod" | awk '{print $3}')  # Fork + exec for EVERY line
    restarts=$(echo "$pod" | awk '{print $4}')
done < <(kubectl get pods --no-headers)
# 200 pods = 400 subprocess spawns

# GOOD (pure bash, zero subprocesses):
while IFS=' ' read -r name ready status restarts age; do
    # Variables already split — no awk needed
    echo "$name is $status with $restarts restarts"
done < <(kubectl get pods --no-headers)

# GOOD (single awk call processes everything):
kubectl get pods --no-headers | awk '$3 != "Running" {print $1, $3, $4}'

# ── Measure performance ────────────────────────────────
time for i in {1..1000}; do
    dirname=$(echo "/a/b/c" | rev | cut -d/ -f2- | rev)   # ~2.5 seconds
done

time for i in {1..1000}; do
    dirname="${path%/*}"                                     # ~0.01 seconds
done
# 250x faster with parameter expansion

# ── Avoid repeated command execution ───────────────────
# BAD: kubectl called twice
if kubectl get pod "$pod" -o jsonpath='{.status.phase}' | grep -q Running; then
    kubectl get pod "$pod" -o json | jq '.status'
fi

# GOOD: Call once, reuse
pod_json=$(kubectl get pod "$pod" -o json)
phase=$(jq -r '.status.phase' <<< "$pod_json")
if [[ "$phase" == "Running" ]]; then
    jq '.status' <<< "$pod_json"
fi
```

---

### Category 7: Date Manipulation

```bash
# ── Epoch conversions ──────────────────────────────────
# Current epoch
date +%s                           # 1705334527

# Date to epoch
date -d "2024-01-15T14:30:00Z" +%s  # Linux (GNU date)
date -j -f "%Y-%m-%dT%H:%M:%SZ" "2024-01-15T14:30:00Z" +%s  # macOS (BSD date)

# Epoch to date
date -d @1705334527 -u +%Y-%m-%dT%H:%M:%SZ   # Linux
date -r 1705334527 -u +%Y-%m-%dT%H:%M:%SZ     # macOS

# ── Relative dates ─────────────────────────────────────
date -d "3 days ago" +%Y-%m-%d       # 2024-01-12
date -d "last monday" +%Y-%m-%d
date -d "+2 hours" +%Y-%m-%dT%H:%M:%SZ

# ── Time arithmetic ───────────────────────────────────
start=$(date +%s)
# ... do work ...
end=$(date +%s)
elapsed=$(( end - start ))
printf "Completed in %dm%ds\n" $((elapsed / 60)) $((elapsed % 60))

# ── Cross-platform gotcha ──────────────────────────────
# GNU date (Linux): date -d "string"
# BSD date (macOS): date -j -f "format" "string"
# In scripts that must run on both:
if date -d "1 day ago" +%s &>/dev/null 2>&1; then
    DATE_CMD="date -d"
else
    DATE_CMD="date -j -v-1d"  # BSD
fi

# OR: use python for portability
epoch_from_iso() {
    python3 -c "from datetime import datetime; print(int(datetime.fromisoformat('$1'.replace('Z','+00:00')).timestamp()))"
}

# ── printf builtin for timestamps (Bash 4.2+, no subprocess) ──
printf '%(%Y-%m-%dT%H:%M:%S%z)T\n' -1   # Current time, no fork
printf '%(%s)T\n' -1                       # Current epoch, no fork
```

---

### Category 8: Promised Scripts Not Delivered

I promised these in the lesson plan and didn't deliver:

1. **Database backup verification** — listed in roadmap
2. **Disk cleanup script** — listed in roadmap  
3. **Deployment rollback automation** — partially covered but not standalone

I'll deliver one now to close the biggest gap:

```bash
#!/usr/bin/env bash
#
# Script: disk-cleanup.sh
# Purpose: Safe disk cleanup on EKS nodes with thresholds and dry-run
# Schedule: K8s CronJob daily or triggered by disk pressure alert
#
set -euo pipefail
IFS=$'\n\t'

source "$(dirname "${BASH_SOURCE[0]}")/lib/common.sh"

readonly WARN_THRESHOLD=75
readonly CRIT_THRESHOLD=85
readonly DRY_RUN="${DRY_RUN:-true}"  # Default to dry-run for safety!

# Cleanup targets ordered by safety (least risky first)
cleanup_docker_build_cache() {
    local freed
    info "Pruning Docker build cache..."
    if [[ "$DRY_RUN" == true ]]; then
        freed=$(docker builder du 2>/dev/null | tail -1 | awk '{print $NF}')
        info "[DRY-RUN] Would free ~${freed} of build cache"
    else
        docker builder prune -f --filter 'until=72h' 2>/dev/null || true
    fi
}

cleanup_unused_images() {
    info "Pruning unused container images..."
    local image_count
    image_count=$(docker images -q --filter 'dangling=true' 2>/dev/null | wc -l)
    info "Dangling images: $image_count"
    
    if [[ "$DRY_RUN" == true ]]; then
        docker images --filter 'dangling=true' --format '{{.Repository}}:{{.Tag}} {{.Size}}' 2>/dev/null
        info "[DRY-RUN] Would remove $image_count dangling images"
    else
        docker image prune -f 2>/dev/null || true
        # Also remove images not used in 7 days
        docker image prune -a -f --filter 'until=168h' 2>/dev/null || true
    fi
}

cleanup_old_logs() {
    info "Cleaning old container logs..."
    local log_dir="/var/log/containers"
    
    if [[ ! -d "$log_dir" ]]; then
        info "No container log directory found at ${log_dir}"
        return 0
    fi
    
    # Find logs older than 7 days and larger than 100MB
    local large_old_logs
    large_old_logs=$(find "$log_dir" -name '*.log' -mtime +7 -size +100M 2>/dev/null || true)
    
    if [[ -z "$large_old_logs" ]]; then
        info "No old large logs found"
        return 0
    fi
    
    while IFS= read -r logfile; do
        local size
        size=$(du -h "$logfile" | cut -f1)
        if [[ "$DRY_RUN" == true ]]; then
            info "[DRY-RUN] Would truncate: $logfile ($size)"
        else
            info "Truncating: $logfile ($size)"
            : > "$logfile"  # Truncate (don't delete — container still has fd open)
        fi
    done <<< "$large_old_logs"
}

cleanup_tmp() {
    info "Cleaning /tmp..."
    local old_tmp_count
    old_tmp_count=$(find /tmp -maxdepth 1 -mtime +3 -not -name 'tmp' 2>/dev/null | wc -l)
    
    if [[ "$DRY_RUN" == true ]]; then
        info "[DRY-RUN] Would remove ${old_tmp_count} items from /tmp older than 3 days"
    else
        find /tmp -maxdepth 1 -mtime +3 -not -name 'tmp' -exec rm -rf {} + 2>/dev/null || true
    fi
}

cleanup_journal() {
    info "Cleaning systemd journal..."
    if command -v journalctl &>/dev/null; then
        if [[ "$DRY_RUN" == true ]]; then
            local journal_size
            journal_size=$(journalctl --disk-usage 2>/dev/null | grep -oP '[\d.]+[GM]' || echo "unknown")
            info "[DRY-RUN] Journal using ${journal_size}. Would vacuum to 500M"
        else
            journalctl --vacuum-size=500M 2>/dev/null || true
        fi
    fi
}

main() {
    info "═══ Disk Cleanup (dry_run=${DRY_RUN}) ═══"
    
    # Check current usage
    local usage
    usage=$(df / --output=pcent | tail -1 | tr -d ' %')
    info "Current disk usage: ${usage}%"
    
    if (( usage < WARN_THRESHOLD )); then
        info "Disk usage below warning threshold (${WARN_THRESHOLD}%). No cleanup needed."
        exit 0
    fi
    
    if (( usage >= CRIT_THRESHOLD )); then
        warn "Disk usage CRITICAL (${usage}% >= ${CRIT_THRESHOLD}%)"
        # Override dry-run in critical situation
        if [[ "$DRY_RUN" == true ]]; then
            warn "Auto-disabling dry-run due to critical disk pressure"
            DRY_RUN=false
        fi
    fi
    
    # Execute cleanup in order of safety
    cleanup_docker_build_cache
    cleanup_unused_images
    cleanup_old_logs
    cleanup_journal
    cleanup_tmp
    
    # Post-cleanup check
    local new_usage
    new_usage=$(df / --output=pcent | tail -1 | tr -d ' %')
    local freed=$(( usage - new_usage ))
    info "Post-cleanup usage: ${new_usage}% (freed ${freed}%)"
    
    if (( new_usage >= CRIT_THRESHOLD )); then
        fatal "Disk still critical after cleanup (${new_usage}%). Manual intervention required."
    fi
    
    notify_slack "INFO" ":broom: Disk cleanup on $(hostname): ${usage}% → ${new_usage}% (freed ${freed}%)"
    info "═══ Cleanup Complete ═══"
}

main "$@"
```

---

## Revised Self-Evaluation

| Sub-topic | Original Lesson | This Addendum | Combined |
|-----------|:-:|:-:|:-:|
| set -euo pipefail + gotchas | ✅ | — | ✅ |
| Logging framework | ✅ | — | ✅ |
| trap (EXIT, ERR, signals) | ✅ | — | ✅ |
| Argument parsing | ✅ | — | ✅ |
| Lock files (flock) | ✅ | — | ✅ |
| Retry + backoff | ✅ | — | ✅ |
| Dry-run mode | ✅ | — | ✅ |
| awk / sed / jq / yq | ✅ | — | ✅ |
| Container entrypoints | ✅ | — | ✅ |
| Real NovaMart scripts | ✅ (3) | ✅ (+1) | ✅ (4) |
| Shared library | ✅ | — | ✅ |
| Makefiles / Taskfile | ✅ | — | ✅ |
| Anti-patterns | ✅ | — | ✅ |
| ShellCheck | ✅ | — | ✅ |
| Parallel execution | ✅ | — | ✅ |
| **Process substitution `<()`** | ❌ | ✅ | ✅ |
| **String manipulation / parameter expansion** | ❌ | ✅ | ✅ |
| **BASH_REMATCH / regex captures** | ❌ | ✅ | ✅ |
| **Associative arrays** | ❌ | ✅ | ✅ |
| **mapfile / readarray** | ❌ | ✅ | ✅ |
| **Here strings `<<<`** | ❌ | ✅ | ✅ |
| **Subshells `()` vs groups `{}`** | ❌ | ✅ | ✅ |
| **printf vs echo** | ❌ | ✅ | ✅ |
| **Bash debugging (set -x, PS4, BASH_XTRACEFD)** | ❌ | ✅ | ✅ |
| **File descriptors & advanced redirection** | ❌ | ✅ | ✅ |
| **Glob options (nullglob, globstar, extglob)** | ❌ | ✅ | ✅ |
| **Function patterns (nameref, arrays, scope)** | ❌ | ✅ | ✅ |
| **Performance (subshells in loops, parameter expansion)** | ❌ | ✅ | ✅ |
| **Date manipulation (epoch, relative, cross-platform)** | ❌ | ✅ | ✅ |
| **Disk cleanup script** | ❌ | ✅ | ✅ |

### Remaining Gaps (genuinely minor):

| Gap | Why I'm leaving it |
|-----|---------------------|
| `getopts` built-in | Manual parsing is the production standard for long options. getopts only does short opts. |
| `select` for interactive menus | DevOps scripts should be non-interactive. If you need menus, use Python. |
| Named pipes (FIFOs) | Niche. Go/Python better for IPC. |
| Bash completion scripts | Nice to have. Will cover when we build the `novactl` CLI in Go (Phase 9 Lesson 3). |
| coproc | I'll die on this hill: if you need coproc, rewrite in Go. |

**Revised honest score: 9.5/10**

The original lesson was **7/10** for completeness. With this addendum it's 9.5/10. The 0.5 gap is the genuinely minor items above that I'm deliberately excluding because they cross into "use a real language" territory.

Good catch calling me out. The original lesson taught you how to write production scripts but left holes in the language fundamentals you'd need to *read and debug* other people's bash — which is half the job.

---

