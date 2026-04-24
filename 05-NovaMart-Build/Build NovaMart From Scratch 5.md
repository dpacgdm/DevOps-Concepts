# Phase 7 — Lesson 5: Operations Readiness

## "You built the machine. Now prove you can keep it alive at 3 AM when it's on fire."

> **Note:** Lesson 4 review is still pending. I'll grade it when we circle back. You asked to move forward, so we move forward. Don't mistake that for forgiveness — I'll come back to it.

---

## The Reality Check

You've built:
- Infrastructure (VPC, EKS, data tier, security baseline)
- Platform services (observability, service mesh, policy engine, GitOps)
- CI/CD pipeline (Jenkins, shared libraries, canary deployments)
- Application onboarding (payment-service with PCI controls)

**None of that matters if you can't operate it.**

A beautiful architecture that nobody can debug at 3 AM is worse than an ugly one with excellent runbooks. NovaMart makes $2B/year. That's **$3,800/minute**. Every minute of your fumbling in an incident is money burning.

This lesson is about proving operational readiness. It covers **seven domains:**

```
┌──────────────────────────────────────────────────────────┐
│                  OPERATIONS READINESS                    │
│                                                          │
│  1. SLO Implementation        (What "healthy" means)     │
│  2. Runbooks                  (What to do when it's not) │
│  3. Incident Management       (How to coordinate chaos)  │
│  4. Disaster Recovery         (When everything fails)    │
│  5. Chaos Engineering         (Break it before it breaks)│
│  6. On-Call & Escalation      (Who does what when)       │
│  7. Day-2 Operations          (Keeping it healthy daily) │
│                                                          │
│  Deliverable: The Operations Bible — everything an       │
│  engineer needs to run NovaMart without calling you.     │
└──────────────────────────────────────────────────────────┘
```

---

## Domain 1: SLO Implementation

You learned the theory in Phase 5 Lesson 4. Now you implement it for real.

### The SLO Contract Table

Every user-facing service needs SLOs. Internal platform services get **internal SLOs** (not customer-facing, but your team tracks them).

```
┌──────────────────┬──────────────┬────────────────────────┬─────────┬────────┬──────────┐
│ Service          │ SLI Type     │ Measurement            │ Target  │ Window │ Budget   │
├──────────────────┼──────────────┼────────────────────────┼─────────┼────────┼──────────┤
│ API Gateway      │ Availability │ non-5xx / total        │ 99.95%  │ 30d    │ 21.6min  │
│ API Gateway      │ Latency      │ p99 < 500ms            │ 99.0%   │ 30d    │ 7.2hrs   │
│ Payment Service  │ Availability │ non-5xx / total        │ 99.99%  │ 30d    │ 4.32min  │
│ Payment Service  │ Latency      │ p99 < 1000ms           │ 99.5%   │ 30d    │ 3.6hrs   │
│ Payment Service  │ Correctness  │ charge_amount == expected│ 99.999%│ 30d   │ 26sec    │
│ Cart Service     │ Availability │ non-5xx / total        │ 99.9%   │ 30d    │ 43.2min  │
│ Cart Service     │ Latency      │ p99 < 300ms            │ 99.5%   │ 30d    │ 3.6hrs   │
│ Order Service    │ Availability │ non-5xx / total        │ 99.95%  │ 30d    │ 21.6min  │
│ Order Service    │ Freshness    │ order_status lag < 5s   │ 99.9%   │ 30d   │ 43.2min  │
│ Search           │ Availability │ non-5xx / total        │ 99.9%   │ 30d    │ 43.2min  │
│ Search           │ Latency      │ p99 < 200ms            │ 99.0%   │ 30d    │ 7.2hrs   │
│ User Service     │ Availability │ non-5xx / total        │ 99.95%  │ 30d    │ 21.6min  │
│ Notification Svc │ Freshness    │ delivery lag < 30s     │ 99.0%   │ 30d    │ 7.2hrs   │
├──────────────────┼──────────────┼────────────────────────┼─────────┼────────┼──────────┤
│ PLATFORM (internal SLOs — team-facing, not customer SLAs)                              │
├──────────────────┼──────────────┼────────────────────────┼─────────┼────────┼──────────┤
│ CI/CD Pipeline   │ Availability │ builds succeed / total │ 99.0%   │ 7d     │ 1.68hrs  │
│ CI/CD Pipeline   │ Latency      │ p95 build < 15min      │ 95.0%   │ 7d     │ 8.4hrs   │
│ ArgoCD           │ Freshness    │ sync lag < 5min        │ 99.9%   │ 30d    │ 43.2min  │
│ Observability    │ Availability │ Prometheus up          │ 99.95%  │ 30d    │ 21.6min  │
│ DNS (CoreDNS)    │ Latency      │ p99 < 10ms             │ 99.99%  │ 30d    │ 4.32min  │
└──────────────────┴──────────────┴────────────────────────┴─────────┴────────┴──────────┘
```

**Why these numbers:**
- Payment at 99.99%: Revenue-critical, PCI, customer trust. 4.32min/month is TIGHT — that's intentional.
- API Gateway at 99.95%: It fronts everything. 21.6min/month gives you room for deployments.
- Search at 99.9%: Degraded search hurts but doesn't lose money directly.
- Payment correctness at 99.999%: Wrong charges = chargebacks + lawsuits. Almost zero tolerance.
- CI/CD at 99.0% over 7d: Platform is internal. More tolerance, shorter window for faster feedback.

### SLI Recording Rules

These are the Prometheus recording rules that compute SLIs. They run every evaluation interval (typically 30s) and pre-compute the ratios so dashboards and alerts are instant.

```yaml
# slo-recording-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-recording-rules
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    # ─── API GATEWAY ───
    - name: slo.apigateway
      interval: 30s
      rules:
        # --- Availability SLI ---
        # Total requests (5m rate, smooths scrape misses)
        - record: sli:apigateway:requests:rate5m
          expr: |
            sum(rate(http_requests_total{service="api-gateway"}[5m]))

        # Error requests (5xx only — 4xx are NOT errors from SLO perspective)
        - record: sli:apigateway:errors:rate5m
          expr: |
            sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[5m]))

        # Error ratio (0 = perfect, 1 = everything failing)
        - record: sli:apigateway:error_ratio:rate5m
          expr: |
            sli:apigateway:errors:rate5m
            /
            sli:apigateway:requests:rate5m

        # --- Latency SLI ---
        # Fraction of requests under 500ms threshold
        - record: sli:apigateway:latency_good_ratio:rate5m
          expr: |
            sum(rate(http_request_duration_seconds_bucket{
              service="api-gateway", le="0.5"
            }[5m]))
            /
            sum(rate(http_request_duration_seconds_count{
              service="api-gateway"
            }[5m]))

    # ─── PAYMENT SERVICE ───
    - name: slo.payment
      interval: 30s
      rules:
        - record: sli:payment:requests:rate5m
          expr: |
            sum(rate(http_requests_total{service="payment-service"}[5m]))

        - record: sli:payment:errors:rate5m
          expr: |
            sum(rate(http_requests_total{service="payment-service", status_code=~"5.."}[5m]))

        - record: sli:payment:error_ratio:rate5m
          expr: |
            sli:payment:errors:rate5m / sli:payment:requests:rate5m

        - record: sli:payment:latency_good_ratio:rate5m
          expr: |
            sum(rate(http_request_duration_seconds_bucket{
              service="payment-service", le="1.0"
            }[5m]))
            /
            sum(rate(http_request_duration_seconds_count{
              service="payment-service"
            }[5m]))

        # Correctness SLI — app must expose this metric
        # payment_charge_correct_total / payment_charge_total
        - record: sli:payment:correctness_ratio:rate5m
          expr: |
            sum(rate(payment_charge_correct_total[5m]))
            /
            sum(rate(payment_charge_total[5m]))

    # ─── CART SERVICE ───
    - name: slo.cart
      interval: 30s
      rules:
        - record: sli:cart:requests:rate5m
          expr: sum(rate(http_requests_total{service="cart-service"}[5m]))

        - record: sli:cart:errors:rate5m
          expr: sum(rate(http_requests_total{service="cart-service", status_code=~"5.."}[5m]))

        - record: sli:cart:error_ratio:rate5m
          expr: sli:cart:errors:rate5m / sli:cart:requests:rate5m

        - record: sli:cart:latency_good_ratio:rate5m
          expr: |
            sum(rate(http_request_duration_seconds_bucket{
              service="cart-service", le="0.3"
            }[5m]))
            /
            sum(rate(http_request_duration_seconds_count{
              service="cart-service"
            }[5m]))

    # ─── ORDER SERVICE ───
    - name: slo.order
      interval: 30s
      rules:
        - record: sli:order:requests:rate5m
          expr: sum(rate(http_requests_total{service="order-service"}[5m]))

        - record: sli:order:errors:rate5m
          expr: sum(rate(http_requests_total{service="order-service", status_code=~"5.."}[5m]))

        - record: sli:order:error_ratio:rate5m
          expr: sli:order:errors:rate5m / sli:order:requests:rate5m

        # Freshness: fraction of order status updates within 5s
        - record: sli:order:freshness_good_ratio:rate5m
          expr: |
            sum(rate(order_status_update_duration_seconds_bucket{le="5.0"}[5m]))
            /
            sum(rate(order_status_update_duration_seconds_count[5m]))
```

### Burn Rate Alerts

Multi-window, multi-burn-rate — exactly as taught in Phase 5 Lesson 4, now applied to real services:

```yaml
# slo-burn-rate-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-burn-rate-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: slo.burn.apigateway
      rules:
        # ── Page: 14.4x burn in 1h AND 6x burn in 6h ──
        # Exhausts 30d budget in ~2 days. Wake someone up.
        - alert: APIGatewayAvailabilityBurnRateCritical
          expr: |
            (
              1 - (1 - sli:apigateway:error_ratio:rate5m) 
            ) > (14.4 * 0.0005)
            and
            (
              sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[1h]))
              / sum(rate(http_requests_total{service="api-gateway"}[1h]))
            ) > (14.4 * 0.0005)
            and
            (
              sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[6h]))
              / sum(rate(http_requests_total{service="api-gateway"}[6h]))
            ) > (6 * 0.0005)
          for: 2m
          labels:
            severity: critical
            slo: apigateway-availability
            team: platform
          annotations:
            summary: "API Gateway burning error budget 14.4x faster than allowed"
            description: |
              Current 1h error rate: {{ $value | humanizePercentage }}
              SLO target: 99.95% (budget: 0.05%)
              At this rate, 30d budget exhausted in ~50 hours.
            runbook_url: "https://runbooks.novamart.internal/slo/apigateway-availability"
            dashboard_url: "https://grafana.novamart.internal/d/slo-overview"

        # ── Ticket: 3x burn in 3d ──
        # Slow bleed. Won't page, but needs attention this sprint.
        - alert: APIGatewayAvailabilityBurnRateWarning
          expr: |
            (
              sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[3d]))
              / sum(rate(http_requests_total{service="api-gateway"}[3d]))
            ) > (3 * 0.0005)
          for: 30m
          labels:
            severity: warning
            slo: apigateway-availability
            team: platform
          annotations:
            summary: "API Gateway slow-burning error budget"
            description: |
              3-day error rate exceeds 3x budget consumption.
              If unchecked, budget exhausted before end of window.
            runbook_url: "https://runbooks.novamart.internal/slo/apigateway-availability"

    - name: slo.burn.payment
      rules:
        # Payment is 99.99% — budget is 0.01%. TINY.
        # 14.4x burn = 0.144% error rate. That's 1 in 700 requests failing.
        - alert: PaymentAvailabilityBurnRateCritical
          expr: |
            (
              sum(rate(http_requests_total{service="payment-service", status_code=~"5.."}[5m]))
              / sum(rate(http_requests_total{service="payment-service"}[5m]))
            ) > (14.4 * 0.0001)
            and
            (
              sum(rate(http_requests_total{service="payment-service", status_code=~"5.."}[1h]))
              / sum(rate(http_requests_total{service="payment-service"}[1h]))
            ) > (14.4 * 0.0001)
            and
            (
              sum(rate(http_requests_total{service="payment-service", status_code=~"5.."}[6h]))
              / sum(rate(http_requests_total{service="payment-service"}[6h]))
            ) > (6 * 0.0001)
          for: 1m  # Faster for payment — budget is only 4.32 minutes
          labels:
            severity: critical
            slo: payment-availability
            team: platform
            pci: "true"
          annotations:
            summary: "🚨 PAYMENT SERVICE burning error budget critically"
            description: |
              Payment SLO: 99.99% (budget: 4.32 min/month)
              Current burn rate will exhaust budget in hours.
              IMMEDIATE ACTION REQUIRED.
            runbook_url: "https://runbooks.novamart.internal/slo/payment-availability"

        # Payment correctness — near-zero tolerance
        - alert: PaymentCorrectnessViolation
          expr: |
            (1 - sli:payment:correctness_ratio:rate5m) > 0.00001
          for: 1m
          labels:
            severity: critical
            slo: payment-correctness
            team: platform
            pci: "true"
          annotations:
            summary: "🚨 PAYMENT CORRECTNESS VIOLATION — charges may be wrong"
            description: |
              Incorrect charge ratio detected. This is a financial integrity issue.
              SLO: 99.999% correctness. Any violation requires immediate investigation.
              CHECK: Payment gateway responses, currency conversion, rounding logic.
            runbook_url: "https://runbooks.novamart.internal/slo/payment-correctness"
```

**Critical implementation note:** The burn rate expressions above use the raw metric queries for the different time windows (5m, 1h, 6h, 3d). In production, you'd create recording rules for each window to avoid redundant computation. I'm showing the logic explicitly here so you understand what each window does. Your build should use recording rules.

### Error Budget Policy

This is a **document**, not code. It's the organizational agreement on what happens when budgets are consumed:

```
# error-budget-policy.md

## NovaMart Error Budget Policy
Version: 1.0 | Effective: [date] | Owner: Platform Engineering + VP Engineering

### Budget States

| State        | Budget Remaining | Actions                                         |
|--------------|------------------|-------------------------------------------------|
| GREEN        | > 50%            | Normal velocity. Ship features freely.          |
| YELLOW       | 25-50%           | Increased review rigor. No experimental deploys.|
| ORANGE       | 10-25%           | Feature freeze for this service. Bug fixes only.|
| RED          | < 10%            | Full freeze. Reliability work only. VP sign-off |
|              |                  | required for ANY deploy to this service.        |
| EXHAUSTED    | 0%               | Incident declared. Postmortem required.         |
|              |                  | No deploys until budget recovers above 10%.     |

### Exceptions
- Security patches: Always allowed regardless of budget state.
- Revenue-critical hotfix: CTO approval + platform team escort deploy.
- Scheduled maintenance: Pre-approved budget allocation (deducted in advance).

### Weekly Review
- Platform team reviews all SLO dashboards Monday 9 AM.
- Services in ORANGE or RED are escalated to engineering leads.
- Monthly report to VP Engineering with trends.

### Dispute Resolution
- If a team disagrees with their SLO target, they present data
  to Architecture Review Board. Changes require 2-week observation period.
```

### How it breaks:

| Failure Mode | Symptom | Root Cause | Fix |
|---|---|---|---|
| SLI returns NaN | Dashboard shows "No data" | Zero requests (off-hours) divides by zero | Wrap in `(... > 0) or vector(0)` for error ratio; use `absent()` for alerting gaps |
| Budget drains on deploy | Budget drops 2% every Tuesday | Rolling update 502s during pod termination | Don't count planned maintenance; or fix preStop + connection draining |
| Alert never fires | Budget exhausted, no page | Recording rule label mismatch with alert expr | `promtool test rules` in CI; Watchdog validates alerting pipeline end-to-end |
| Alert fires constantly | On-call fatigue, ignored | SLO target too aggressive for current architecture | Loosen target to achievable level, then tighten as you improve |
| Budget gaming | Team routes errors through different endpoint | Avoid SLO measurement path | SLI must be measured at the load balancer / mesh level, not app-reported |

---

## Domain 2: Runbooks

A runbook is not documentation. It's an **executable procedure for a sleep-deprived engineer at 3 AM.** 

### Runbook Structure Standard

Every runbook follows this template:

```
# [RUNBOOK-XXX] Title
## Metadata
- Severity: SEV1/SEV2/SEV3/SEV4
- Services affected: [list]
- SLO impact: [which SLOs this affects]
- Last tested: [date]
- Owner: [team]

## Detection
- Alert name(s) that trigger this runbook
- Dashboard link(s)
- What the on-call engineer will see (PagerDuty message, Slack alert)

## Impact Assessment (do this FIRST — 30 seconds)
- Who is affected? (all users, region, subset)
- Revenue impact? (complete outage vs degraded)
- Is this getting worse? (check rate of change)

## Quick Mitigation (do this BEFORE root cause — stop the bleeding)
- [ ] Step 1: [exact command or action]
- [ ] Step 2: [exact command or action]
- Expected result: [what you should see if mitigation worked]
- Time limit: If not mitigated in X minutes, escalate to [person/team]

## Diagnosis
### Step 1: [Check name]
```bash
# exact command to run
kubectl get pods -n payment -l app=payment-service
```
- If [condition A]: Go to Step 2a
- If [condition B]: Go to Step 2b

### Step 2a: [Branch A diagnosis]
...

## Root Cause Resolution
- [Permanent fix steps]
- [ ] Verify fix: [exact verification command]
- [ ] Monitor for 15 minutes: [what to watch]

## Escalation
- If not resolved in [X] minutes: Page [team/person]
- If customer-facing > 15 min: Notify [customer-comms channel]
- If financial impact: Notify [finance-ops channel]

## Post-Incident
- [ ] Update incident timeline
- [ ] Schedule postmortem if SEV1/SEV2
- [ ] Update this runbook if steps were wrong/missing
```

### Required Runbooks for NovaMart

You must create runbooks for every failure scenario that has happened or could happen:

```
INFRASTRUCTURE RUNBOOKS (RUNBOOK-INFRA-XXX)
├── RUNBOOK-INFRA-001: Node NotReady
├── RUNBOOK-INFRA-002: EKS API Server Unreachable  
├── RUNBOOK-INFRA-003: NAT Gateway Throttling / Failure
├── RUNBOOK-INFRA-004: VPC CNI IP Exhaustion (Pod stuck Pending)
├── RUNBOOK-INFRA-005: EBS Volume Stuck Attaching/Detaching
├── RUNBOOK-INFRA-006: Karpenter Not Provisioning Nodes
├── RUNBOOK-INFRA-007: Spot Instance Interruption (mass eviction)
├── RUNBOOK-INFRA-008: AWS Region Degradation

DATA TIER RUNBOOKS (RUNBOOK-DATA-XXX)
├── RUNBOOK-DATA-001: RDS High CPU / Slow Queries
├── RUNBOOK-DATA-002: RDS Storage Full
├── RUNBOOK-DATA-003: RDS Failover (planned and unplanned)
├── RUNBOOK-DATA-004: Redis Cluster Node Failure
├── RUNBOOK-DATA-005: Redis Memory Full (eviction happening)
├── RUNBOOK-DATA-006: RDS Replication Lag (read replicas)
├── RUNBOOK-DATA-007: Database Connection Pool Exhaustion
├── RUNBOOK-DATA-008: RDS Certificate Rotation

PLATFORM RUNBOOKS (RUNBOOK-PLAT-XXX)
├── RUNBOOK-PLAT-001: Prometheus Down / Not Scraping
├── RUNBOOK-PLAT-002: Alertmanager Not Sending Alerts
├── RUNBOOK-PLAT-003: Loki Not Ingesting Logs
├── RUNBOOK-PLAT-004: ArgoCD Sync Failure / Stuck
├── RUNBOOK-PLAT-005: Linkerd Control Plane Failure
├── RUNBOOK-PLAT-006: Linkerd Certificate Expiry
├── RUNBOOK-PLAT-007: Kyverno Blocking Deployments
├── RUNBOOK-PLAT-008: CoreDNS Failure / High Latency
├── RUNBOOK-PLAT-009: Jenkins Controller Down
├── RUNBOOK-PLAT-010: External Secrets Operator Failure
├── RUNBOOK-PLAT-011: cert-manager Certificate Renewal Failure
├── RUNBOOK-PLAT-012: Thanos Compactor Halted
├── RUNBOOK-PLAT-013: OTel Collector Pipeline Backup

APPLICATION RUNBOOKS (RUNBOOK-APP-XXX)
├── RUNBOOK-APP-001: Payment Service 5xx Spike
├── RUNBOOK-APP-002: Payment Gateway Timeout (Stripe/Adyen)
├── RUNBOOK-APP-003: Cart Service Latency Spike
├── RUNBOOK-APP-004: Order Processing Queue Backup (RabbitMQ)
├── RUNBOOK-APP-005: Search Service Degraded Results
├── RUNBOOK-APP-006: High Error Rate After Deployment
├── RUNBOOK-APP-007: Memory Leak (gradual OOMKill pattern)
├── RUNBOOK-APP-008: CrashLoopBackOff (any service)
├── RUNBOOK-APP-009: Canary Rollback Triggered
├── RUNBOOK-APP-010: Cross-Service Cascading Failure

SECURITY RUNBOOKS (RUNBOOK-SEC-XXX)
├── RUNBOOK-SEC-001: Vault Sealed / Unreachable
├── RUNBOOK-SEC-002: Compromised Container Detected
├── RUNBOOK-SEC-003: DDoS Attack Active
├── RUNBOOK-SEC-004: Secret Rotation Emergency
├── RUNBOOK-SEC-005: Suspicious IAM Activity
```

**That's 36 runbooks.** For the build assignment, I'll require a subset — but you need to understand the full scope.

### Example: Full Runbook (RUNBOOK-APP-001)

Here's what production-grade looks like:

```markdown
# RUNBOOK-APP-001: Payment Service 5xx Spike

## Metadata
- Severity: SEV1 (revenue-impacting)
- Services: payment-service, payment-gateway integration
- SLOs: payment-availability (99.99%), payment-correctness (99.999%)
- Last tested: 2024-11-15 (chaos drill)
- Owner: Platform Engineering
- PCI scope: YES — all actions must be logged

## Detection
- Alert: PaymentAvailabilityBurnRateCritical
- PagerDuty: "PAYMENT SERVICE burning error budget critically"
- Dashboard: https://grafana.novamart.internal/d/payment-slo
- Secondary: PaymentCorrectnessViolation (if charges are wrong, not just failing)

## Impact Assessment (30 seconds)
1. Open dashboard — check SCOPE:
   - All regions or single region?
   - All endpoints or specific route (e.g., /charge vs /refund)?
   - Started when? (correlate with deploy, config change, upstream)
2. Check revenue impact:
   - `sum(rate(payment_charge_total{status="failed"}[5m]))` — failed charges/sec
   - Multiply by average order value (~$85) for $/min estimate
3. Is it getting worse?
   - Compare 5m rate vs 1h rate. If 5m >> 1h, it's accelerating.

## Quick Mitigation (STOP THE BLEEDING — before diagnosis)

### If started after a deployment (check ArgoCD):
```bash
# Check recent rollouts
kubectl argo rollouts list rollouts -n payment
kubectl argo rollouts get rollout payment-service -n payment

# Immediate rollback
kubectl argo rollouts abort payment-service -n payment
kubectl argo rollouts undo payment-service -n payment
```
Expected: Error rate drops within 60 seconds of rollback completing.

### If NOT deployment-related:
```bash
# Check if payment gateway (Stripe/Adyen) is down
curl -s https://status.stripe.com/api/v2/status.json | jq '.status'
curl -s https://status.adyen.com/api/v2/status.json | jq '.status'

# If upstream is down — enable circuit breaker / fallback
# Linkerd retry budget should already be limiting retries
# If retry storm detected:
kubectl annotate service payment-service -n payment \
  config.linkerd.io/proxy-retry-budget="0%" --overwrite
```

### If partial failure (one AZ or one pod):
```bash
# Identify bad pods
kubectl top pods -n payment -l app=payment-service
kubectl get pods -n payment -l app=payment-service -o wide

# If single pod crashing:
kubectl delete pod <bad-pod> -n payment
# Deployment will reschedule. Monitor for 2 minutes.
```

**Time limit:** If error rate not decreasing within 10 minutes, escalate to SEV1 Incident Commander.

## Diagnosis

### Step 1: Classify the error
```bash
# What HTTP status codes?
kubectl logs -n payment -l app=payment-service --tail=100 | \
  jq -r '.status_code' | sort | uniq -c | sort -rn

# Or via Prometheus:
# sum by (status_code) (rate(http_requests_total{service="payment-service", status_code=~"5.."}[5m]))
```

- **502/503**: Go to Step 2a (pod/mesh issue)
- **504**: Go to Step 2b (timeout — upstream or database)
- **500**: Go to Step 2c (application error)

### Step 2a: 502/503 — Pod or Mesh Issue
```bash
# Pod status
kubectl get pods -n payment -l app=payment-service
# Look for: CrashLoopBackOff, OOMKilled, not Ready

# Check Linkerd proxy health
kubectl exec <pod> -n payment -c linkerd-proxy -- \
  curl -s localhost:4191/metrics | grep -E "response_total|tcp_open"

# Check readiness probe
kubectl describe pod <pod> -n payment | grep -A5 "Readiness"
```

### Step 2b: 504 — Timeout
```bash
# Database connection pool
kubectl exec <pod> -n payment -c payment-service -- \
  curl -s localhost:8080/actuator/metrics/hikaricp.connections.active | jq '.'

# Check RDS
aws rds describe-db-instances --db-instance-identifier novamart-payment-prod \
  --query 'DBInstances[0].{CPU: "check CloudWatch", Status: DBInstanceStatus, 
  Connections: "check CloudWatch"}'

# Slow query log
aws rds download-db-log-file-portion \
  --db-instance-identifier novamart-payment-prod \
  --log-file-name slowquery/postgresql.log.2024-01-15 \
  --starting-token 0
```

### Step 2c: 500 — Application Error
```bash
# Get stack traces from logs
kubectl logs -n payment -l app=payment-service --tail=200 | \
  jq 'select(.level == "ERROR")' | head -50

# Find trace IDs for failed requests
kubectl logs -n payment -l app=payment-service --tail=200 | \
  jq 'select(.status_code >= 500) | .trace_id' | head -10

# Open in Tempo/Jaeger for full distributed trace
# https://grafana.novamart.internal/explore?left={"datasource":"Tempo","queries":[{"query":"<traceID>"}]}
```

## Root Cause Resolution
- Depends on diagnosis above. Common resolutions:
  1. **Bad deploy**: Rollback confirmed working → investigate code change → hotfix
  2. **Upstream down**: Circuit breaker active → monitor upstream status page → remove circuit breaker when recovered
  3. **Database**: Kill long-running queries → scale read replicas → add connection pool limits
  4. **Memory leak**: Restart pods (rolling) → file bug with memory profiling data
  5. **Config change**: Revert via ArgoCD → review change in postmortem

## Verification
```bash
# Confirm error rate returning to normal
# Watch for 15 minutes:
watch -n 10 'kubectl exec -n monitoring prometheus-0 -- \
  promtool query instant http://localhost:9090 \
  "sli:payment:error_ratio:rate5m"'

# Should be < 0.0001 (0.01%) for 99.99% SLO
```

## Escalation
| Time     | Action                                              |
|----------|-----------------------------------------------------|
| 0 min    | On-call begins investigation                        |
| 5 min    | If not classified: page secondary on-call           |
| 10 min   | If not mitigated: declare SEV1, page Incident Commander |
| 15 min   | If customer-facing: notify #incident-comms channel   |
| 30 min   | If not resolved: page engineering manager            |
| 60 min   | VP Engineering notified                              |

## Post-Incident
- [ ] Timeline documented in incident channel
- [ ] Revenue impact calculated with finance
- [ ] Postmortem scheduled within 48 hours
- [ ] This runbook updated if any steps were wrong or missing
- [ ] PCI: All investigation commands logged to audit trail
```

**That's the quality bar.** Every runbook should be this detailed. An engineer who has never seen the system should be able to follow it.

---

## Domain 3: Incident Management

### Severity Definitions

```
┌──────┬────────────────────┬──────────────┬──────────────────────┬──────────────┐
│ SEV  │ Definition         │ Response Time│ Examples             │ Comms        │
├──────┼────────────────────┼──────────────┼──────────────────────┼──────────────┤
│ SEV1 │ Total outage or    │ 5 min ack    │ Payment down,        │ Exec + Cust  │
│      │ revenue-impacting  │ 15 min war rm│ site unreachable,    │ status page  │
│      │ degradation        │              │ data breach          │ updates q15m │
├──────┼────────────────────┼──────────────┼──────────────────────┼──────────────┤
│ SEV2 │ Major feature      │ 15 min ack   │ Search down,         │ Eng leads    │
│      │ degraded/down      │ 30 min war rm│ checkout slow (>5s), │ status page  │
│      │                    │              │ one region down      │ if prolonged │
├──────┼────────────────────┼──────────────┼──────────────────────┼──────────────┤
│ SEV3 │ Minor feature      │ 30 min ack   │ Notifications delayed│ Team channel │
│      │ degraded           │ 4hr response │ non-critical API slow│              │
├──────┼────────────────────┼──────────────┼──────────────────────┼──────────────┤
│ SEV4 │ Cosmetic or minor  │ 1hr ack      │ Dashboard slow,      │ Jira ticket  │
│      │ issue, workaround  │ next sprint  │ log noise, single    │              │
│      │ exists             │              │ user report          │              │
└──────┴────────────────────┴──────────────┴──────────────────────┴──────────────┘
```

### Incident Response Process

```
┌────────────────────────────────────────────────────────────┐
│                    INCIDENT LIFECYCLE                      │
│                                                            │
│  DETECT → TRIAGE → MITIGATE → DIAGNOSE → RESOLVE → LEARN   │
│                                                            │
│  ┌─────────┐                                               │
│  │ Alert   │──► PagerDuty pages on-call                    │
│  │ fires   │                                               │
│  └─────────┘                                               │
│       │                                                    │
│  ┌────▼────┐                                               │
│  │ TRIAGE  │  On-call has 5 min to:                        │
│  │ (5 min) │  1. Acknowledge in PagerDuty                  │
│  │         │  2. Open #incident-YYYYMMDD Slack channel     │
│  │         │  3. Assess severity                           │
│  │         │  4. If SEV1/2: page Incident Commander        │
│  └────┬────┘                                               │
│       │                                                    │
│  ┌────▼──────────┐                                         │
│  │  MITIGATE     │  STOP THE BLEEDING FIRST.               │
│  │  (target:     │  Rollback, scale, circuit break,        │
│  │   15 min)     │  failover. Do NOT debug first.          │
│  └────┬──────────┘                                         │
│       │                                                    │
│  ┌────▼──────────┐                                         │
│  │  DIAGNOSE     │  Now find root cause.                   │
│  │               │  Follow runbook.                        │
│  │               │  Update incident channel every 15 min.  │
│  └────┬──────────┘                                         │
│       │                                                    │
│  ┌────▼──────────┐                                         │
│  │  RESOLVE      │  Permanent fix applied.                 │
│  │               │  Monitor for 30 min stability.          │
│  │               │  Close incident in PagerDuty.           │
│  └────┬──────────┘                                         │
│       │                                                    │
│  ┌────▼──────────┐                                         │
│  │  LEARN        │  Postmortem within 48hrs (SEV1/2)       │
│  │  (post-       │  Blameless. Action items tracked.       │
│  │   mortem)     │  Runbook updated.                       │
│  └───────────────┘                                         │
└────────────────────────────────────────────────────────────┘
```

### Incident Roles

```
INCIDENT COMMANDER (IC)
  - Owns the incident. Coordinates, does NOT debug.
  - Assigns roles. Manages communications.
  - Makes decisions (rollback yes/no, escalate yes/no).
  - For SEV1: Senior engineer or engineering manager.

TECHNICAL LEAD
  - The one debugging. Follows runbook.
  - Reports findings to IC every 15 minutes.
  - Requests additional engineers if needed.

COMMUNICATIONS LEAD (SEV1/2 only)
  - Updates status page every 15 minutes.
  - Manages #incident channel updates.
  - Shields tech lead from management questions.

SCRIBE (SEV1 only)
  - Logs everything with timestamps.
  - This becomes the postmortem timeline.
  - "10:23 — rolled back payment-service to v2.3.1"
  - "10:25 — error rate dropped from 15% to 0.3%"
```

### Postmortem Template

```markdown
# Postmortem: [Incident Title]

## Metadata
- Date: YYYY-MM-DD
- Duration: X hours Y minutes
- Severity: SEV[N]
- Incident Commander: [name]
- Author: [name]
- Status: [Draft / Reviewed / Action Items Complete]

## Summary
[2-3 sentences. What happened, who was affected, how long.]

## Impact
- Users affected: [number/percentage]
- Revenue impact: [estimated $]
- SLO impact: [which SLOs, budget consumed]
- Support tickets generated: [count]

## Timeline (all times UTC)
| Time  | Event                                          |
|-------|------------------------------------------------|
| 10:15 | Deploy of payment-service v2.4.0 completed     |
| 10:18 | Error rate SLI crosses 0.5%                    |
| 10:20 | PaymentAvailabilityBurnRateCritical fires      |
| 10:21 | On-call acknowledges, opens #incident-20240115 |
| 10:25 | Rollback initiated                             |
| 10:27 | Rollback complete, error rate dropping         |
| 10:35 | Error rate at baseline. Monitoring.            |
| 10:50 | Incident closed.                               |

## Root Cause
[Technical description. No blame. Facts only.]

## What Went Well
- Alert fired within 2 minutes of issue start
- Rollback procedure worked as documented
- On-call response within SLA

## What Went Wrong  
- Integration tests didn't cover the specific edge case
- Canary analysis window was too short to catch
- Runbook step 3 referenced wrong metric name

## Action Items
| ID | Action                                    | Owner   | Priority | Due Date   | Status |
|----|-------------------------------------------|---------|----------|------------|--------|
| 1  | Add integration test for [edge case]      | @dev    | P1       | 2024-01-22 | Open   |
| 2  | Extend canary analysis to 10min minimum   | @devops | P1       | 2024-01-19 | Open   |
| 3  | Fix metric name in RUNBOOK-APP-001 step 3 | @devops | P2       | 2024-01-16 | Done   |
| 4  | Add pre-deploy database migration check   | @dev    | P2       | 2024-01-29 | Open   |

## Lessons Learned
[What organizational/process change prevents this CLASS of incident, not just this specific one.]
```

**The golden rule of postmortems: BLAMELESS.** "Engineer X made a mistake" is never the root cause. "Our system allowed a mistake to reach production" is.

---

## Domain 4: Disaster Recovery

### RTO/RPO Definitions

```
RTO = Recovery Time Objective = How long can we be down?
RPO = Recovery Point Objective = How much data can we lose?

┌──────────────────┬────────┬────────┬─────────────────────────────────────┐
│ Component        │ RTO    │ RPO    │ Strategy                            │
├──────────────────┼────────┼────────┼─────────────────────────────────────┤
│ Payment Service  │ 5 min  │ 0      │ Multi-AZ active-active, sync repl.  │
│ Payment DB (RDS) │ 10 min │ 0      │ Multi-AZ (sync standby), auto-fail  │
│ Order Service    │ 15 min │ 1 min  │ Multi-AZ, async replication         │
│ Order DB (RDS)   │ 15 min │ 5 min  │ Multi-AZ, automated snapshots q5m   │
│ Cart Service     │ 30 min │ 5 min  │ Redis replication + RDB snapshots   │
│ User Service     │ 15 min │ 1 min  │ Multi-AZ                            │
│ User DB (RDS)    │ 10 min │ 0      │ Multi-AZ (sync standby)             │
│ Search Service   │ 1 hr   │ 1 hr   │ Rebuild from source-of-truth DB     │
│ Notification Svc │ 1 hr   │ 30 min │ Queue-backed, replay from RabbitMQ  │
│ EKS Cluster      │ 30 min │ N/A    │ Rebuild from Terraform + ArgoCD     │
│ Observability    │ 1 hr   │ 1 hr   │ Rebuild from Helm, data in S3/Thanos│
│ CI/CD (Jenkins)  │ 2 hr   │ 24 hr  │ Rebuild from Terraform + S3 backup  │
│ Static Assets    │ 15 min │ 0      │ S3 cross-region replication         │
└──────────────────┴────────┴────────┴─────────────────────────────────────┘
```

### DR Scenarios

```
SCENARIO 1: Single AZ Failure
  Impact: ~33% capacity loss
  Response: AUTOMATIC
    - EKS reschedules pods to remaining AZs (if resources available)
    - RDS Multi-AZ failover (automatic, ~60s)
    - NAT Gateway: other AZs unaffected (we have 3)
    - ALB: automatically routes to healthy AZs
  Manual actions:
    - Verify Karpenter scales replacement nodes in healthy AZs
    - Monitor for connection errors during RDS failover
    - Check Redis sentinel promotes correctly
  Recovery time: 2-5 minutes (mostly automatic)

SCENARIO 2: Region Failure (us-east-1 down)
  Impact: FULL OUTAGE for primary region
  Response: MANUAL FAILOVER (automated is Phase 2 roadmap)
    Step 1: Confirm region-level failure (not just one service)
      - Check AWS Health Dashboard
      - Check multiple services (EC2, RDS, EKS)
      - Don't failover for a single-service blip
    Step 2: Update Route53 to point to DR region (us-west-2)
      - DR cluster runs warm standby (minimal replicas)
      - Scale up DR cluster: `kubectl scale` or Karpenter handles it
    Step 3: Promote RDS read replica in us-west-2 to primary
      - THIS IS DESTRUCTIVE — breaks replication. No going back easily.
    Step 4: Update application configs for DR database endpoints
      - External Secrets should pull from regional Secrets Manager
    Step 5: Verify all services healthy in DR region
    Step 6: Update CDN/Cloudflare to DR origin
  Recovery time: 30-60 minutes (manual)
  DATA LOSS: RPO depends on replication lag to cross-region read replicas

SCENARIO 3: Data Corruption (bad migration, ransomware)
  Impact: Data integrity compromised
  Response:
    Step 1: STOP ALL WRITES IMMEDIATELY
      - Scale payment/order services to 0 replicas
      - This is drastic but prevents corruption spread
    Step 2: Assess scope of corruption
      - What tables? What time range?
      - Can we identify the bad transaction/migration?
    Step 3: Point-in-time recovery
      - RDS supports PITR to any second within retention window
      - Create NEW instance from PITR, don't restore over existing
      - Validate data on new instance before switching
    Step 4: Replay good transactions
      - If corruption started at T1, restore to T1-1min
      - Replay transactions from T1 to now that weren't corrupted
      - THIS IS THE HARD PART — may require application team
    Recovery time: 2-8 hours depending on scope

SCENARIO 4: EKS Cluster Destroyed
  Impact: All workloads down
  Response: "Cattle, not pets" — rebuild it
    Step 1: Terraform apply (networking + EKS) — 20-30 min
    Step 2: ArgoCD bootstrap (App of Apps deploys everything) — 10-15 min
    Step 3: External Secrets syncs all credentials — 2-3 min
    Step 4: DNS/LB updated to new cluster endpoints
    Step 5: Verify all services healthy
  Recovery time: ~45-60 minutes
  Data loss: ZERO (state is in RDS/Redis/S3, not in cluster)
  THIS IS WHY GITOPS MATTERS — your cluster is reproducible from code
```

### DR Exercise Schedule

```
┌────────────────────┬───────────┬──────────┬──────────────────────┐
│ Exercise           │ Frequency │ Duration │ Scope                │
├────────────────────┼───────────┼──────────┼──────────────────────┤
│ AZ failure sim     │ Monthly   │ 2 hrs    │ Kill one AZ's nodes  │
│ RDS failover       │ Quarterly │ 1 hr     │ Force RDS failover   │
│ Cluster rebuild    │ Quarterly │ 4 hrs    │ Destroy + rebuild EKS│
│ Region failover    │ Bi-annual │ 8 hrs    │ Full DR activation   │
│ Chaos game days    │ Monthly   │ 4 hrs    │ Random failure inject│
│ Backup restore     │ Monthly   │ 2 hrs    │ Restore RDS from snap│
│ Runbook validation │ Monthly   │ 2 hrs    │ Walk through runbooks│
└────────────────────┴───────────┴──────────┴──────────────────────┘

RULE: A DR plan that hasn't been tested is not a plan. It's a hope.
```

---

## Domain 5: Chaos Engineering

### Principles

```
1. Start with a hypothesis: "If we kill 1 AZ, traffic shifts in <60s"
2. Start small: single pod → single node → single AZ → region
3. Run in production (eventually) — staging doesn't prove production resilience
4. Minimize blast radius: circuit breakers, time limits, abort conditions
5. Automate experiments for repeatability
```

### Tools

```
Litmus Chaos    — K8s-native, ChaosEngine CRDs, good AWS experiments
Chaos Mesh      — K8s-native, fine-grained (network delay, CPU stress, I/O fault)
Gremlin         — SaaS, enterprise, easy to use, $$
AWS FIS         — Native AWS (EC2 stop, RDS failover, AZ impairment)
tc + iptables   — Manual but free (network delay/partition)
stress-ng       — CPU/memory stress testing
```

### NovaMart Chaos Experiments

```yaml
# chaos-experiments/pod-kill-payment.yaml
# Hypothesis: Payment service handles pod failure with zero dropped requests
# Expected: Linkerd retries to healthy pods, no 5xx visible to clients
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: payment-pod-kill
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one                    # Kill one random pod
  selector:
    namespaces:
      - payment
    labelSelectors:
      app: payment-service
  duration: "60s"              # Kill one pod every 60s
  scheduler:
    cron: "@every 2m"          # Repeat
---
# chaos-experiments/network-delay-rds.yaml
# Hypothesis: Payment service tolerates 200ms added DB latency without SLO breach
# Expected: p99 increases but stays under 1000ms SLO
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-db-latency
  namespace: chaos-testing
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - payment
    labelSelectors:
      app: payment-service
  delay:
    latency: "200ms"
    jitter: "50ms"
    correlation: "75"
  direction: to
  target:
    selector:
      namespaces:
        - payment
      labelSelectors:
        role: database-proxy     # Target Egress to DB
  duration: "5m"
---
# chaos-experiments/az-failure.yaml
# Hypothesis: Full AZ loss handled automatically, SLOs maintained
# Uses AWS FIS to stop all EC2 instances in one AZ
# (simplified — real FIS experiment template is more complex)
apiVersion: fis.amazonaws.com/v1  # Pseudo — real FIS is via AWS API/Terraform
kind: ExperimentTemplate
metadata:
  name: az-failure-us-east-1a
spec:
  description: "Simulate us-east-1a failure"
  targets:
    instances:
      resourceType: aws:ec2:instance
      selectionMode: ALL
      resourceTags:
        kubernetes.io/cluster/novamart-prod: owned
      filters:
        - path: Placement.AvailabilityZone
          values: [us-east-1a]
  actions:
    stop-instances:
      actionId: aws:ec2:stop-instances
      targets: [instances]
      parameters:
        startInstancesAfterDuration: PT10M  # Auto-restart after 10 min
  stopConditions:
    - source: aws:cloudwatch:alarm
      value: arn:aws:cloudwatch:...:alarm:chaos-abort-alarm
      # Abort if error rate exceeds 5% — safety net
```

### Chaos Game Day Procedure

```
PRE-GAME (1 day before):
  1. Announce to all teams (no surprise chaos in production, at first)
  2. Verify rollback procedures work
  3. Set up monitoring dashboards for the specific experiments
  4. Define abort criteria and safety nets
  5. Ensure on-call staffing is NOT skeleton crew

GAME DAY:
  09:00 — Brief: Review hypotheses, experiments, abort conditions
  09:30 — Experiment 1: Pod kill (single service, low risk)
           Observe: Does the service recover? SLO impact?
  10:00 — Experiment 2: Network delay injection
           Observe: Timeout behavior, retry behavior, circuit breaking
  10:30 — Experiment 3: Node drain (simulate node failure)
           Observe: Rescheduling time, PDB respected?
  11:00 — Experiment 4: RDS failover (if approved by DBA)
           Observe: Connection handling, app recovery time
  11:30 — Debrief: What surprised us? What broke unexpectedly?
  12:00 — Document findings, create Jira tickets for gaps found

POST-GAME:
  - Write report with findings
  - Create action items for every unexpected failure
  - Update runbooks based on learnings
  - Schedule next game day
```

---

## Domain 6: On-Call & Escalation

### On-Call Rotation

```
ROTATION STRUCTURE:
  Primary on-call:   1 week rotation, 6 engineers (Platform team)
  Secondary on-call:  1 week rotation, offset by 1 person
  
  Handoff: Monday 10 AM (after weekly SLO review)
  Handoff checklist:
    □ Review active incidents from past week
    □ Review any ongoing deploys or maintenance windows
    □ Review any known issues or temporary workarounds
    □ Verify PagerDuty receives test page
    □ Verify VPN access and kubectl access to all clusters
    □ Review on-call runbook updates from past week

ESCALATION POLICY (PagerDuty):
  T+0 min:    Page primary on-call (push + SMS + phone)
  T+5 min:    If no ack → page secondary on-call
  T+10 min:   If no ack → page engineering manager
  T+15 min:   If no ack → page VP Engineering (something is very wrong)

ON-CALL EXPECTATIONS:
  - Acknowledge within 5 minutes
  - Laptop + internet access at all times during rotation
  - Response (not resolution) within 15 minutes
  - If you can't respond (medical, emergency), hand off IMMEDIATELY
  - Log all actions in incident channel

BURNOUT PREVENTION:
  - Max 1 week in 6 on primary
  - If paged > 3 times in one night, secondary takes over
  - If on-call load consistently > 2 pages/day, address alert quality
  - Compensatory time off after heavy on-call weeks
  - Regular review: if Toil > 50% of on-call time, automation work prioritized
```

### PagerDuty Service Configuration

```yaml
# pagerduty-services.yaml (conceptual — PagerDuty config via API/Terraform)

services:
  - name: novamart-payment-critical
    escalation_policy: payment-critical-escalation
    alert_grouping:
      type: intelligent      # PD groups related alerts
      timeout: 300           # 5 min grouping window
    auto_resolve_timeout: 14400  # Auto-resolve after 4 hours if no updates
    acknowledgement_timeout: 1800  # Re-page if acked but no resolution in 30 min
    integrations:
      - type: prometheus_alertmanager
        endpoint: /integration/alertmanager/novamart-payment

  - name: novamart-platform-warning
    escalation_policy: platform-standard-escalation
    alert_grouping:
      type: intelligent
      timeout: 600
    auto_resolve_timeout: 28800
    integrations:
      - type: prometheus_alertmanager

escalation_policies:
  - name: payment-critical-escalation
    rules:
      - targets: [primary-on-call]
        escalation_delay: 5
      - targets: [secondary-on-call]
        escalation_delay: 5
      - targets: [engineering-manager]
        escalation_delay: 5
      - targets: [vp-engineering]
        escalation_delay: 10

  - name: platform-standard-escalation
    rules:
      - targets: [primary-on-call]
        escalation_delay: 15
      - targets: [secondary-on-call]
        escalation_delay: 15
      - targets: [engineering-manager]
        escalation_delay: 30
```

### Alert Routing (Alertmanager → PagerDuty/Slack)

This builds on what was set up in Phase 5 Lesson 1, now formalized:

```yaml
# alertmanager-routing.yaml
route:
  receiver: default-slack
  group_by: ['alertname', 'service', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    # ── SEV1: Payment Critical → PagerDuty immediately ──
    - match:
        severity: critical
        pci: "true"
      receiver: pagerduty-payment-critical
      group_wait: 10s          # Almost immediate
      repeat_interval: 5m      # Re-page every 5 min if not acked
      continue: true           # ALSO send to Slack

    # ── SEV1/2: Any critical → PagerDuty ──  
    - match:
        severity: critical
      receiver: pagerduty-platform-critical
      group_wait: 30s
      repeat_interval: 10m
      continue: true

    # ── SEV3: Warning → Slack only ──
    - match:
        severity: warning
      receiver: slack-platform-warnings
      repeat_interval: 4h

    # ── SLO burn rate alerts ──
    - match_re:
        alertname: ".*BurnRate.*"
      receiver: slack-slo-alerts
      continue: true
      routes:
        - match:
            severity: critical
          receiver: pagerduty-platform-critical

    # ── Watchdog (dead man's switch) ──
    - match:
        alertname: Watchdog
      receiver: deadmans-switch
      group_wait: 0s
      repeat_interval: 1m

receivers:
  - name: pagerduty-payment-critical
    pagerduty_configs:
      - service_key_file: /etc/alertmanager/secrets/pagerduty-payment-key
        severity: critical
        description: '{{ .CommonAnnotations.summary }}'
        details:
          service: '{{ .CommonLabels.service }}'
          namespace: '{{ .CommonLabels.namespace }}'
          runbook: '{{ .CommonAnnotations.runbook_url }}'
          dashboard: '{{ .CommonAnnotations.dashboard_url }}'

  - name: pagerduty-platform-critical
    pagerduty_configs:
      - service_key_file: /etc/alertmanager/secrets/pagerduty-platform-key
        severity: critical

  - name: slack-platform-warnings
    slack_configs:
      - api_url_file: /etc/alertmanager/secrets/slack-webhook
        channel: '#platform-alerts'
        title: '⚠️ {{ .CommonLabels.alertname }}'
        text: '{{ .CommonAnnotations.description }}'
        actions:
          - type: button
            text: 'Runbook'
            url: '{{ .CommonAnnotations.runbook_url }}'
          - type: button
            text: 'Dashboard'
            url: '{{ .CommonAnnotations.dashboard_url }}'

  - name: slack-slo-alerts
    slack_configs:
      - api_url_file: /etc/alertmanager/secrets/slack-webhook
        channel: '#slo-alerts'
        title: '🔥 SLO Burn: {{ .CommonLabels.alertname }}'
        text: |
          *Service:* {{ .CommonLabels.service }}
          *SLO:* {{ .CommonLabels.slo }}
          {{ .CommonAnnotations.description }}
        color: '{{ if eq .CommonLabels.severity "critical" }}danger{{ else }}warning{{ end }}'

  - name: deadmans-switch
    webhook_configs:
      - url: 'https://noflake.novamart.internal/deadman'
        send_resolved: false

  - name: default-slack
    slack_configs:
      - api_url_file: /etc/alertmanager/secrets/slack-webhook
        channel: '#platform-alerts-low'

inhibit_rules:
  # If a critical alert fires, suppress the warning for same service
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'service', 'namespace']

  # If cluster is down, suppress all pod-level alerts
  - source_match:
      alertname: KubeAPIDown
    target_match_re:
      alertname: 'Kube.*'
    equal: ['cluster']
```

### How On-Call Breaks:

| Failure Mode | Symptom | Root Cause | Fix |
|---|---|---|---|
| Alert fatigue | On-call ignores pages | Too many low-signal alerts | Review alert quality monthly. Delete or tune noisy alerts. Target < 2 actionable pages per shift. |
| Flapping alerts | Page → resolve → page → resolve | Threshold too close to normal operating range | Add hysteresis: `for: 5m` not `for: 30s`. Or use multi-window burn rates. |
| Escalation not working | Page goes nowhere | PagerDuty schedule not updated after team changes | Automate schedule sync. Test escalation monthly. |
| Handoff gaps | New on-call doesn't know about ongoing issue | No handoff process | Mandatory handoff meeting with checklist. |
| Knowledge silo | Only one person can debug payment | Runbooks incomplete, tribal knowledge | Runbook-first culture. If you debug without a runbook, you write one before closing the incident. |
| Burnout | Engineer goes dark during on-call | Excessive page volume, no relief | Track pages-per-week metric. If > 10/week, it's a reliability problem, not an on-call problem. |

---

## Domain 7: Day-2 Operations

This is the unsexy work that keeps the platform alive between incidents.

### Daily Operations Checklist

```
DAILY (automated checks + human review — 15 min):
  □ SLO Dashboard review
    - Any service in YELLOW or worse?
    - Error budget burn rate trending up?
  □ Platform health
    - ArgoCD: all apps synced? Any OutOfSync/Degraded?
    - Prometheus: scrape targets all up? (up == 0 count)
    - Karpenter: nodes consolidating normally?
    - cert-manager: any certificates expiring < 30 days?
  □ CI/CD health
    - Jenkins: any stuck builds? Queue depth normal?
    - Build success rate > 95%?
  □ Cost anomaly check
    - AWS Cost Explorer daily spend vs forecast
    - Any unexpected EC2/NAT Gateway/data transfer spikes?
  □ Security
    - CloudTrail: any root account usage?
    - GuardDuty: any new findings?
```

### Weekly Operations

```
WEEKLY (Monday — 1 hour):
  □ SLO Review Meeting (all engineering leads)
    - Review 7-day SLO compliance per service
    - Review error budget consumption trends
    - Prioritize reliability work for services in ORANGE/RED
  □ On-call handoff
    - Outgoing on-call briefs incoming
    - Review past week's incidents and active issues
  □ Dependency updates review
    - Any new CVEs in base images?
    - Trivy scan results trend
    - Are any dependencies EOL soon?
  □ Capacity review
    - Node utilization trend (are we over/under provisioned?)
    - RDS storage growth trend
    - Redis memory utilization trend
    - ECR image count growth
  □ Cost review
    - Weekly spend vs budget
    - Top cost drivers
    - Spot savings rate
    - Reserved instance utilization
```

### Monthly Operations

```
MONTHLY:
  □ Chaos game day (see Domain 5)
  □ DR exercise (rotating scenario from DR schedule)
  □ Runbook review
    - Walk through 3-5 runbooks
    - Are they still accurate? Commands still work?
    - Assign updates as needed
  □ Platform upgrades review
    - EKS version (K8s releases every ~4 months)
    - Helm chart versions for all platform components
    - Terraform provider versions
    - Jenkins plugin versions
  □ Access review
    - RBAC audit: who has what access?
    - Service account audit: any unused IRSA roles?
    - Remove departed engineers
  □ Backup validation
    - Restore one RDS snapshot to test instance
    - Verify data integrity
    - Delete test instance
    - Document: "Backup restore tested [date], [result]"
  □ Performance baseline update
    - Are p99 latencies drifting up over months?
    - Is any service slowly leaking memory (requires restart)?
    - Database query performance: any new slow queries?
  □ SLO target review
    - Are targets still appropriate?
    - Any services consistently at 100% (target too loose)?
    - Any services consistently burning budget (target too aggressive, or reliability gap)?
```

### Quarterly Operations

```
QUARTERLY:
  □ Full DR region failover exercise
  □ Platform version upgrades (EKS, major Helm chart bumps)
  □ Architecture review
    - Any services outgrowing their current design?
    - Any new scaling bottlenecks emerging?
    - Cost optimization opportunities
  □ SLO renegotiation with product/business
  □ On-call retrospective
    - Page volume trends
    - Toil percentage
    - What automation would reduce on-call burden most?
  □ Security penetration test review
  □ Compliance audit preparation (PCI-DSS)
```

### Automation for Day-2

The daily checks should NOT be a human staring at dashboards. Automate what you can:

```yaml
# daily-health-check CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: platform-health-check
  namespace: platform-ops
spec:
  schedule: "0 8 * * *"  # 8 AM daily
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: platform-health-checker
          containers:
            - name: health-check
              image: novamart/platform-tools:v1.2.0
              command: ["/bin/bash", "-c"]
              args:
                - |
                  #!/bin/bash
                  set -euo pipefail
                  
                  REPORT=""
                  FAILURES=0
                  
                  # --- ArgoCD Sync Status ---
                  OUTOFSYNC=$(argocd app list -o json | \
                    jq '[.[] | select(.status.sync.status != "Synced")] | length')
                  if [ "$OUTOFSYNC" -gt 0 ]; then
                    REPORT+="❌ ArgoCD: $OUTOFSYNC apps out of sync\n"
                    FAILURES=$((FAILURES + 1))
                  else
                    REPORT+="✅ ArgoCD: All apps synced\n"
                  fi
                  
                  # --- Prometheus Targets ---
                  DOWN_TARGETS=$(curl -s http://prometheus.monitoring:9090/api/v1/targets | \
                    jq '[.data.activeTargets[] | select(.health != "up")] | length')
                  if [ "$DOWN_TARGETS" -gt 0 ]; then
                    REPORT+="❌ Prometheus: $DOWN_TARGETS targets down\n"
                    FAILURES=$((FAILURES + 1))
                  else
                    REPORT+="✅ Prometheus: All targets healthy\n"
                  fi
                  
                  # --- Certificate Expiry ---
                  EXPIRING=$(kubectl get certificates -A -o json | \
                    jq --arg cutoff "$(date -d '+30 days' -Iseconds)" \
                    '[.items[] | select(.status.notAfter < $cutoff)] | length')
                  if [ "$EXPIRING" -gt 0 ]; then
                    REPORT+="⚠️ Certificates: $EXPIRING expiring within 30 days\n"
                    FAILURES=$((FAILURES + 1))
                  else
                    REPORT+="✅ Certificates: All valid > 30 days\n"
                  fi
                  
                  # --- SLO Budget Status ---
                  for svc in apigateway payment cart order; do
                    BUDGET=$(curl -s http://prometheus.monitoring:9090/api/v1/query \
                      --data-urlencode "query=sli:${svc}:error_budget_remaining:ratio" | \
                      jq -r '.data.result[0].value[1] // "unknown"')
                    if [ "$BUDGET" != "unknown" ]; then
                      BUDGET_PCT=$(echo "$BUDGET * 100" | bc -l | xargs printf "%.1f")
                      if (( $(echo "$BUDGET < 0.25" | bc -l) )); then
                        REPORT+="🔴 SLO $svc: ${BUDGET_PCT}% budget remaining\n"
                        FAILURES=$((FAILURES + 1))
                      elif (( $(echo "$BUDGET < 0.50" | bc -l) )); then
                        REPORT+="🟡 SLO $svc: ${BUDGET_PCT}% budget remaining\n"
                      else
                        REPORT+="🟢 SLO $svc: ${BUDGET_PCT}% budget remaining\n"
                      fi
                    fi
                  done
                  
                  # --- Node Health ---
                  NOT_READY=$(kubectl get nodes --no-headers | grep -c "NotReady" || true)
                  if [ "$NOT_READY" -gt 0 ]; then
                    REPORT+="❌ Nodes: $NOT_READY NotReady\n"
                    FAILURES=$((FAILURES + 1))
                  else
                    REPORT+="✅ Nodes: All Ready\n"
                  fi
                  
                  # --- Pending Pods ---
                  PENDING=$(kubectl get pods -A --field-selector=status.phase=Pending \
                    --no-headers 2>/dev/null | wc -l)
                  if [ "$PENDING" -gt 0 ]; then
                    REPORT+="⚠️ Pods: $PENDING stuck Pending\n"
                    FAILURES=$((FAILURES + 1))
                  else
                    REPORT+="✅ Pods: None stuck Pending\n"
                  fi
                  
                  # --- Send to Slack ---
                  if [ "$FAILURES" -gt 0 ]; then
                    EMOJI="🚨"
                    COLOR="danger"
                  else
                    EMOJI="✅"
                    COLOR="good"
                  fi
                  
                  curl -X POST "$SLACK_WEBHOOK" -H 'Content-type: application/json' \
                    -d "{
                      \"attachments\": [{
                        \"color\": \"$COLOR\",
                        \"title\": \"$EMOJI NovaMart Daily Health Report\",
                        \"text\": \"$(echo -e $REPORT)\",
                        \"footer\": \"platform-health-check | $(date -u +%Y-%m-%d)\"
                      }]
                    }"
              env:
                - name: SLACK_WEBHOOK
                  valueFrom:
                    secretKeyRef:
                      name: platform-ops-secrets
                      key: slack-webhook
          restartPolicy: OnFailure
```

### Toil Tracking

Google SRE defines toil as: manual, repetitive, automatable, tactical, no lasting value, scales with service growth.

```
TRACK TOIL:
  Every on-call shift, log:
    - Pages received (count, service, alert name)
    - Manual actions taken (that could be automated)
    - Time spent on toil vs project work

  Target: Toil < 50% of on-call time
  If > 50%: prioritize automation in next sprint

COMMON TOIL AT NOVAMART (automate these):
  1. Restarting pods that leak memory → fix the leak or automate rolling restart
  2. Clearing full disks → auto-cleanup CronJob + alerting before full
  3. Rotating secrets manually → External Secrets Operator + auto-rotation
  4. Manually scaling before sales events → scheduled HPA overrides
  5. Manually approving canary promotions → AnalysisTemplate auto-promote
  6. Manually checking backup status → automated backup verification + Slack report
  7. Certificate renewal → cert-manager handles this (already done)
  8. Image cleanup in ECR → lifecycle policies (already done)
```

### Platform Upgrade Runbook

This is one of the most dangerous Day-2 operations. An EKS upgrade gone wrong takes down everything.

```markdown
# RUNBOOK-OPS-001: EKS Version Upgrade

## Pre-Upgrade (1-2 weeks before)

### Week -2: Assessment
- [ ] Check EKS release notes for breaking changes
- [ ] Check all add-on compatibility matrix:
      https://docs.aws.amazon.com/eks/latest/userguide/managing-add-ons.html
- [ ] Check Linkerd compatibility with new K8s version
- [ ] Check Karpenter compatibility
- [ ] Check ArgoCD compatibility
- [ ] Check all Helm chart compatibility
- [ ] Run `pluto detect-all-in-cluster` — find deprecated APIs
- [ ] Review PSA changes in new version
- [ ] Test in dev cluster FIRST, then staging, then production

### Week -1: Staging Upgrade
- [ ] Upgrade staging EKS control plane
      ```bash
      terraform plan -target=module.eks.aws_eks_cluster.main
      # Review plan carefully
      terraform apply -target=module.eks.aws_eks_cluster.main
      ```
- [ ] Upgrade staging add-ons (VPC CNI, CoreDNS, kube-proxy, EBS CSI)
- [ ] Upgrade staging managed node group (launch template update)
- [ ] Run full integration test suite against staging
- [ ] Run chaos experiments in staging
- [ ] Soak for 3-5 days

### Day Of: Production Upgrade

#### Pre-Flight
- [ ] Announce maintenance window (even if zero-downtime)
- [ ] Verify all PDBs in place
- [ ] Verify Karpenter drift detection configured
- [ ] Take RDS snapshots (safety net, not for rollback)
- [ ] Ensure on-call staffing is full (not during on-call handoff)
- [ ] Disable non-critical CronJobs
- [ ] Verify ArgoCD sync windows allow platform changes

#### Control Plane Upgrade (AWS-managed, ~15-20 min)
```bash
# Step 1: Update Terraform EKS version
# In terraform/environments/production/eks/main.tf:
#   cluster_version = "1.29"  → "1.30"

terraform plan -target=module.eks
# REVIEW THE PLAN. It should show in-place update, NOT replace.

terraform apply -target=module.eks

# Step 2: Monitor
kubectl get nodes  # Nodes still on old version — that's expected
kubectl get componentstatuses  # Deprecated but still works
kubectl cluster-info
```

#### Add-on Upgrades
```bash
# Upgrade add-ons to versions compatible with new K8s
# Order matters: VPC CNI → CoreDNS → kube-proxy → EBS CSI

terraform plan -target=module.eks_addons
terraform apply -target=module.eks_addons

# Verify each:
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

#### Node Upgrade (the risky part)
```bash
# Option A: Karpenter drift (preferred)
# Karpenter detects nodes running old kubelet, drains and replaces
# Automatic if drift detection enabled:
#   spec.disruption.expireAfter: 720h  # or trigger manually

# Option B: Managed node group rolling update
terraform plan -target=module.eks.aws_eks_node_group.system
terraform apply -target=module.eks.aws_eks_node_group.system
# This triggers rolling replacement: new node → drain old → terminate

# Monitor node replacement:
watch kubectl get nodes -o wide
# Watch for: new nodes appearing, old nodes cordoned/draining

# CRITICAL: Watch for pod disruption
kubectl get events -A --field-selector reason=Evicted --sort-by='.lastTimestamp'
kubectl get pods -A | grep -v Running | grep -v Completed
```

#### Post-Upgrade Verification
```bash
# All nodes on new version
kubectl get nodes -o custom-columns=NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion

# All system pods healthy
kubectl get pods -n kube-system
kubectl get pods -n linkerd
kubectl get pods -n monitoring
kubectl get pods -n argocd

# All ArgoCD apps synced
argocd app list

# SLO check — no budget impact
# Check Grafana SLO dashboard

# Run platform validation script
./scripts/validate-platform.sh
```

#### Rollback
- Control plane: CANNOT roll back EKS control plane version. This is why staging first.
- Nodes: Can roll back launch template to previous version.
- Add-ons: Can revert add-on versions via Terraform.
- If upgrade causes persistent issues: fast-track next patch version (AWS releases fixes quickly).

## Post-Upgrade
- [ ] Monitor for 48 hours
- [ ] Run chaos experiments
- [ ] Update documentation with new version
- [ ] Update all Helm charts that needed version bumps
- [ ] Close maintenance ticket
```

```
## Your Build Assignment

Now that you understand all seven domains, here's what you build:

### Required Deliverables

```
DELIVERABLE 1: SLO Implementation
  - slo-recording-rules.yaml (all services from contract table)
  - slo-burn-rate-alerts.yaml (multi-window for payment + API gateway minimum)
  - Error budget policy document (error-budget-policy.md)
  - Error budget remaining recording rules
  - Grafana SLO dashboard JSON (or description of panels)

DELIVERABLE 2: Runbooks (build 6 of the 36)
  REQUIRED (these cover the most dangerous scenarios):
    - RUNBOOK-APP-001: Payment Service 5xx Spike
    - RUNBOOK-DATA-007: Database Connection Pool Exhaustion
    - RUNBOOK-PLAT-001: Prometheus Down / Not Scraping
    - RUNBOOK-INFRA-004: VPC CNI IP Exhaustion
    - RUNBOOK-APP-010: Cross-Service Cascading Failure
    - RUNBOOK-SEC-004: Secret Rotation Emergency
  Format: Full runbook format as shown above

DELIVERABLE 3: Incident Management
  - Severity definitions table
  - Incident response process document
  - Roles and responsibilities
  - Postmortem template
  - PagerDuty service + escalation config (conceptual YAML)
  - Alertmanager routing config (full alertmanager.yml)

DELIVERABLE 4: Disaster Recovery
  - RTO/RPO table for all components
  - DR runbook for Scenario 2 (Region Failure) — step-by-step
  - DR runbook for Scenario 4 (Cluster Rebuild) — step-by-step
  - DR exercise schedule
  - Backup validation procedure

DELIVERABLE 5: Chaos Engineering
  - 5 chaos experiments (YAML — Chaos Mesh or conceptual)
  - Game day procedure document
  - Abort criteria for each experiment
  - Hypothesis + expected result + actual result template

DELIVERABLE 6: Day-2 Operations
  - Daily/weekly/monthly/quarterly checklists
  - Automated health check CronJob (K8s manifest)
  - Platform upgrade runbook (EKS version upgrade)
  - Toil tracking template
  - On-call handoff checklist

DELIVERABLE 7: Design Decisions
  - DD-OPS-001: SLO targets rationale
  - DD-OPS-002: On-call rotation structure
  - DD-OPS-003: DR strategy (why not active-active multi-region)
  - DD-OPS-004: Chaos engineering tool choice
  - DD-OPS-005: Runbook hosting/format decision
```

### Grading Criteria

```
CORRECTNESS (25%):
  - SLO math is right (budget calculations, burn rates)
  - PromQL queries are valid and produce correct results
  - Runbook commands actually work
  - Escalation timing is realistic

COMPLETENESS (25%):
  - All 7 deliverables present
  - Runbooks follow full template (no skeleton stubs)
  - No "TODO" or "fill in later" sections
  - Edge cases covered (what if the monitoring is also down?)

OPERABILITY (25%):
  - A sleep-deprived engineer can follow the runbooks
  - Escalation paths are clear
  - Rollback steps are explicit for every risky operation
  - Time estimates are realistic

PRODUCTION REALISM (15%):
  - SLO targets are defensible (not arbitrary numbers)
  - DR strategies match the RTO/RPO requirements
  - Chaos experiments are safe (abort conditions)
  - On-call rotation is sustainable (burnout prevention)

DOCUMENTATION QUALITY (10%):
  - Clear, scannable, no walls of text
  - Code blocks for all commands
  - Links between related documents (runbook references alert, alert references runbook)
  - Design decisions explain WHY, not just WHAT
```

### Traps to Watch For

```
TRAP 1: SLO recording rules that return NaN during low-traffic periods
         (midnight UTC for NovaMart). Handle division by zero.

TRAP 2: Runbooks that assume the monitoring stack is healthy.
         "Check Grafana" is useless if Prometheus is the thing that's down.
         Every runbook needs a "what if observability is also broken" fallback.

TRAP 3: DR procedures that assume you have kubectl access.
         If EKS control plane is down, kubectl doesn't work.
         Plan for AWS Console / API fallback.

TRAP 4: Chaos experiments without abort conditions.
         If your experiment causes cascading failure, how do you stop it?

TRAP 5: On-call rotation that has single points of failure.
         What if primary AND secondary are both unreachable?

TRAP 6: Burn rate alerts that fire on tiny request volumes.
         10 requests, 1 error = 10% error rate = instant page.
         Add minimum request volume threshold.

TRAP 7: Postmortem action items with no owner or due date.
         Action items without owners are wishes, not commitments.

TRAP 8: Day-2 automation that itself has no monitoring.
         If your health-check CronJob fails silently, you've just added
         a false sense of security. Monitor the monitors.
```


## Quick Reference Card

```
SLO MATH:
  Error budget = 1 - SLO target
  99.99% → 0.01% budget → 4.32 min/month
  99.95% → 0.05% budget → 21.6 min/month
  99.9%  → 0.1%  budget → 43.2 min/month
  Burn rate = (actual error rate) / (allowed error rate)
  14.4x burn = budget gone in ~2 days → PAGE
  6x burn = budget gone in ~5 days → PAGE (with 6h confirmation)
  3x burn over 3d = slow bleed → TICKET

INCIDENT RESPONSE:
  DETECT → TRIAGE (5m) → MITIGATE (15m) → DIAGNOSE → RESOLVE → LEARN
  Rule: STOP BLEEDING before diagnosing
  Rule: Rollback first, ask questions later (for deploy-related)
  Rule: Blameless postmortems within 48hr for SEV1/2

DR PRIORITY:
  Payment: RTO 5min, RPO 0 (sync replication, multi-AZ)
  Cluster:  RTO 45min, RPO 0 (Terraform + ArgoCD rebuild)
  Rule: Untested DR = no DR. Exercise monthly.

CHAOS RULES:
  Hypothesis → Small blast → Monitor → Abort conditions → Scale up
  NEVER run chaos without abort criteria
  NEVER run first experiment in production

ON-CALL HEALTH:
  Target: < 2 actionable pages per shift
  Target: Toil < 50% of on-call time
  If either violated: it's a reliability problem, not a people problem

DAY-2 CADENCE:
  Daily:     Health check automation + SLO dashboard glance
  Weekly:    SLO review + on-call handoff + capacity + cost
  Monthly:   Chaos game day + DR exercise + runbook review + access audit
  Quarterly: EKS upgrade + architecture review + SLO renegotiation
```


# DELIVERABLE 1: SLO Implementation

## 1.1 SLO Recording Rules

```yaml
# manifests/platform/monitoring/slo-recording-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-recording-rules
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
    app.kubernetes.io/part-of: novamart-slo
spec:
  groups:
    # ══════════════════════════════════════════════════════════
    # API GATEWAY — 99.95% availability, 99.0% latency (p99<500ms)
    # ══════════════════════════════════════════════════════════
    - name: slo.apigateway.recording
      interval: 30s
      rules:
        # --- Request rates ---
        - record: sli:http_requests:rate5m
          expr: |
            sum by (service) (rate(http_requests_total{service="api-gateway"}[5m]))
          labels:
            slo_service: api-gateway

        - record: sli:http_errors:rate5m
          expr: |
            sum by (service) (rate(http_requests_total{service="api-gateway", status_code=~"5.."}[5m]))
          labels:
            slo_service: api-gateway

        # --- Availability error ratio ---
        # TRAP 1 HANDLED: clamp_min on denominator prevents NaN during zero-traffic
        - record: sli:availability:error_ratio:rate5m
          expr: |
            (
              sli:http_errors:rate5m{slo_service="api-gateway"}
              /
              clamp_min(sli:http_requests:rate5m{slo_service="api-gateway"}, 1)
            )
            # When no traffic, denominator=1, numerator=0, result=0 (not NaN)
            # This means "no errors" during zero traffic, which is correct —
            # you can't violate an SLO with zero requests
          labels:
            slo_service: api-gateway

        # --- Multi-window error ratios for burn rate ---
        - record: sli:availability:error_ratio:rate1h
          expr: |
            sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[1h]))
            /
            clamp_min(sum(rate(http_requests_total{service="api-gateway"}[1h])), 1)
          labels:
            slo_service: api-gateway

        - record: sli:availability:error_ratio:rate6h
          expr: |
            sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[6h]))
            /
            clamp_min(sum(rate(http_requests_total{service="api-gateway"}[6h])), 1)
          labels:
            slo_service: api-gateway

        - record: sli:availability:error_ratio:rate3d
          expr: |
            sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[3d]))
            /
            clamp_min(sum(rate(http_requests_total{service="api-gateway"}[3d])), 1)
          labels:
            slo_service: api-gateway

        # --- Latency SLI: fraction of requests under 500ms ---
        - record: sli:latency:good_ratio:rate5m
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="api-gateway", le="0.5"}[5m]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="api-gateway"}[5m])), 1)
          labels:
            slo_service: api-gateway
            slo_type: latency

        - record: sli:latency:good_ratio:rate1h
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="api-gateway", le="0.5"}[1h]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="api-gateway"}[1h])), 1)
          labels:
            slo_service: api-gateway

        - record: sli:latency:good_ratio:rate6h
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="api-gateway", le="0.5"}[6h]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="api-gateway"}[6h])), 1)
          labels:
            slo_service: api-gateway

        # --- Error budget remaining (30-day rolling) ---
        # Budget = 1 - SLO target = 0.0005 (for 99.95%)
        # Consumed = error_ratio over 30d
        # Remaining = 1 - (consumed / budget)
        - record: sli:availability:budget_remaining:ratio
          expr: |
            1 - (
              (
                sum(rate(http_requests_total{service="api-gateway", status_code=~"5.."}[30d]))
                /
                clamp_min(sum(rate(http_requests_total{service="api-gateway"}[30d])), 1)
              )
              / 0.0005
            )
          labels:
            slo_service: api-gateway
            slo_target: "99.95"

        - record: sli:latency:budget_remaining:ratio
          expr: |
            1 - (
              (
                1 - (
                  sum(rate(http_request_duration_seconds_bucket{service="api-gateway", le="0.5"}[30d]))
                  /
                  clamp_min(sum(rate(http_request_duration_seconds_count{service="api-gateway"}[30d])), 1)
                )
              )
              / 0.01
            )
          labels:
            slo_service: api-gateway
            slo_target: "99.0"

        # --- Traffic volume (for minimum-request gating) ---
        - record: sli:http_requests:total:rate5m
          expr: |
            sum(rate(http_requests_total{service="api-gateway"}[5m]))
          labels:
            slo_service: api-gateway

    # ══════════════════════════════════════════════════════════
    # PAYMENT SERVICE — 99.99% avail, 99.5% latency (p99<1000ms),
    #                    99.999% correctness
    # ══════════════════════════════════════════════════════════
    - name: slo.payment.recording
      interval: 30s
      rules:
        - record: sli:http_requests:rate5m
          expr: sum by (service) (rate(http_requests_total{service="payment-service"}[5m]))
          labels:
            slo_service: payment-service

        - record: sli:http_errors:rate5m
          expr: sum by (service) (rate(http_requests_total{service="payment-service", status_code=~"5.."}[5m]))
          labels:
            slo_service: payment-service

        - record: sli:availability:error_ratio:rate5m
          expr: |
            sli:http_errors:rate5m{slo_service="payment-service"}
            /
            clamp_min(sli:http_requests:rate5m{slo_service="payment-service"}, 1)
          labels:
            slo_service: payment-service

        - record: sli:availability:error_ratio:rate1h
          expr: |
            sum(rate(http_requests_total{service="payment-service", status_code=~"5.."}[1h]))
            /
            clamp_min(sum(rate(http_requests_total{service="payment-service"}[1h])), 1)
          labels:
            slo_service: payment-service

        - record: sli:availability:error_ratio:rate6h
          expr: |
            sum(rate(http_requests_total{service="payment-service", status_code=~"5.."}[6h]))
            /
            clamp_min(sum(rate(http_requests_total{service="payment-service"}[6h])), 1)
          labels:
            slo_service: payment-service

        - record: sli:availability:error_ratio:rate3d
          expr: |
            sum(rate(http_requests_total{service="payment-service", status_code=~"5.."}[3d]))
            /
            clamp_min(sum(rate(http_requests_total{service="payment-service"}[3d])), 1)
          labels:
            slo_service: payment-service

        # --- Latency (1000ms threshold) ---
        - record: sli:latency:good_ratio:rate5m
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="payment-service", le="1.0"}[5m]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="payment-service"}[5m])), 1)
          labels:
            slo_service: payment-service

        - record: sli:latency:good_ratio:rate1h
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="payment-service", le="1.0"}[1h]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="payment-service"}[1h])), 1)
          labels:
            slo_service: payment-service

        - record: sli:latency:good_ratio:rate6h
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="payment-service", le="1.0"}[6h]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="payment-service"}[6h])), 1)
          labels:
            slo_service: payment-service

        # --- Correctness SLI ---
        # App exposes: payment_charge_correct_total, payment_charge_total
        - record: sli:correctness:good_ratio:rate5m
          expr: |
            sum(rate(payment_charge_correct_total[5m]))
            /
            clamp_min(sum(rate(payment_charge_total[5m])), 1)
          labels:
            slo_service: payment-service

        - record: sli:correctness:good_ratio:rate1h
          expr: |
            sum(rate(payment_charge_correct_total[1h]))
            /
            clamp_min(sum(rate(payment_charge_total[1h])), 1)
          labels:
            slo_service: payment-service

        # --- Error budget remaining ---
        - record: sli:availability:budget_remaining:ratio
          expr: |
            1 - (
              (
                sum(rate(http_requests_total{service="payment-service", status_code=~"5.."}[30d]))
                /
                clamp_min(sum(rate(http_requests_total{service="payment-service"}[30d])), 1)
              )
              / 0.0001
            )
          labels:
            slo_service: payment-service
            slo_target: "99.99"

        - record: sli:correctness:budget_remaining:ratio
          expr: |
            1 - (
              (
                1 - (
                  sum(rate(payment_charge_correct_total[30d]))
                  /
                  clamp_min(sum(rate(payment_charge_total[30d])), 1)
                )
              )
              / 0.00001
            )
          labels:
            slo_service: payment-service
            slo_target: "99.999"

        - record: sli:http_requests:total:rate5m
          expr: sum(rate(http_requests_total{service="payment-service"}[5m]))
          labels:
            slo_service: payment-service

    # ══════════════════════════════════════════════════════════
    # CART SERVICE — 99.9% avail, 99.5% latency (p99<300ms)
    # ══════════════════════════════════════════════════════════
    - name: slo.cart.recording
      interval: 30s
      rules:
        - record: sli:http_requests:rate5m
          expr: sum by (service) (rate(http_requests_total{service="cart-service"}[5m]))
          labels:
            slo_service: cart-service

        - record: sli:http_errors:rate5m
          expr: sum by (service) (rate(http_requests_total{service="cart-service", status_code=~"5.."}[5m]))
          labels:
            slo_service: cart-service

        - record: sli:availability:error_ratio:rate5m
          expr: |
            sli:http_errors:rate5m{slo_service="cart-service"}
            /
            clamp_min(sli:http_requests:rate5m{slo_service="cart-service"}, 1)
          labels:
            slo_service: cart-service

        - record: sli:availability:error_ratio:rate1h
          expr: |
            sum(rate(http_requests_total{service="cart-service", status_code=~"5.."}[1h]))
            /
            clamp_min(sum(rate(http_requests_total{service="cart-service"}[1h])), 1)
          labels:
            slo_service: cart-service

        - record: sli:availability:error_ratio:rate6h
          expr: |
            sum(rate(http_requests_total{service="cart-service", status_code=~"5.."}[6h]))
            /
            clamp_min(sum(rate(http_requests_total{service="cart-service"}[6h])), 1)
          labels:
            slo_service: cart-service

        - record: sli:availability:error_ratio:rate3d
          expr: |
            sum(rate(http_requests_total{service="cart-service", status_code=~"5.."}[3d]))
            /
            clamp_min(sum(rate(http_requests_total{service="cart-service"}[3d])), 1)
          labels:
            slo_service: cart-service

        - record: sli:latency:good_ratio:rate5m
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="cart-service", le="0.3"}[5m]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="cart-service"}[5m])), 1)
          labels:
            slo_service: cart-service

        - record: sli:latency:good_ratio:rate1h
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="cart-service", le="0.3"}[1h]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="cart-service"}[1h])), 1)
          labels:
            slo_service: cart-service

        - record: sli:latency:good_ratio:rate6h
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="cart-service", le="0.3"}[6h]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="cart-service"}[6h])), 1)
          labels:
            slo_service: cart-service

        - record: sli:availability:budget_remaining:ratio
          expr: |
            1 - (
              (
                sum(rate(http_requests_total{service="cart-service", status_code=~"5.."}[30d]))
                /
                clamp_min(sum(rate(http_requests_total{service="cart-service"}[30d])), 1)
              )
              / 0.001
            )
          labels:
            slo_service: cart-service
            slo_target: "99.9"

        - record: sli:http_requests:total:rate5m
          expr: sum(rate(http_requests_total{service="cart-service"}[5m]))
          labels:
            slo_service: cart-service

    # ══════════════════════════════════════════════════════════
    # ORDER SERVICE — 99.95% avail, 99.9% freshness (<5s lag)
    # ══════════════════════════════════════════════════════════
    - name: slo.order.recording
      interval: 30s
      rules:
        - record: sli:http_requests:rate5m
          expr: sum by (service) (rate(http_requests_total{service="order-service"}[5m]))
          labels:
            slo_service: order-service

        - record: sli:http_errors:rate5m
          expr: sum by (service) (rate(http_requests_total{service="order-service", status_code=~"5.."}[5m]))
          labels:
            slo_service: order-service

        - record: sli:availability:error_ratio:rate5m
          expr: |
            sli:http_errors:rate5m{slo_service="order-service"}
            /
            clamp_min(sli:http_requests:rate5m{slo_service="order-service"}, 1)
          labels:
            slo_service: order-service

        - record: sli:availability:error_ratio:rate1h
          expr: |
            sum(rate(http_requests_total{service="order-service", status_code=~"5.."}[1h]))
            /
            clamp_min(sum(rate(http_requests_total{service="order-service"}[1h])), 1)
          labels:
            slo_service: order-service

        - record: sli:availability:error_ratio:rate6h
          expr: |
            sum(rate(http_requests_total{service="order-service", status_code=~"5.."}[6h]))
            /
            clamp_min(sum(rate(http_requests_total{service="order-service"}[6h])), 1)
          labels:
            slo_service: order-service

        - record: sli:availability:error_ratio:rate3d
          expr: |
            sum(rate(http_requests_total{service="order-service", status_code=~"5.."}[3d]))
            /
            clamp_min(sum(rate(http_requests_total{service="order-service"}[3d])), 1)
          labels:
            slo_service: order-service

        # --- Freshness SLI: order status updates within 5s ---
        - record: sli:freshness:good_ratio:rate5m
          expr: |
            sum(rate(order_status_update_duration_seconds_bucket{le="5.0"}[5m]))
            /
            clamp_min(sum(rate(order_status_update_duration_seconds_count[5m])), 1)
          labels:
            slo_service: order-service

        - record: sli:freshness:good_ratio:rate1h
          expr: |
            sum(rate(order_status_update_duration_seconds_bucket{le="5.0"}[1h]))
            /
            clamp_min(sum(rate(order_status_update_duration_seconds_count[1h])), 1)
          labels:
            slo_service: order-service

        - record: sli:availability:budget_remaining:ratio
          expr: |
            1 - (
              (
                sum(rate(http_requests_total{service="order-service", status_code=~"5.."}[30d]))
                /
                clamp_min(sum(rate(http_requests_total{service="order-service"}[30d])), 1)
              )
              / 0.0005
            )
          labels:
            slo_service: order-service
            slo_target: "99.95"

        - record: sli:http_requests:total:rate5m
          expr: sum(rate(http_requests_total{service="order-service"}[5m]))
          labels:
            slo_service: order-service

    # ══════════════════════════════════════════════════════════
    # SEARCH SERVICE — 99.9% avail, 99.0% latency (p99<200ms)
    # ══════════════════════════════════════════════════════════
    - name: slo.search.recording
      interval: 30s
      rules:
        - record: sli:http_requests:rate5m
          expr: sum by (service) (rate(http_requests_total{service="search-service"}[5m]))
          labels:
            slo_service: search-service

        - record: sli:http_errors:rate5m
          expr: sum by (service) (rate(http_requests_total{service="search-service", status_code=~"5.."}[5m]))
          labels:
            slo_service: search-service

        - record: sli:availability:error_ratio:rate5m
          expr: |
            sli:http_errors:rate5m{slo_service="search-service"}
            /
            clamp_min(sli:http_requests:rate5m{slo_service="search-service"}, 1)
          labels:
            slo_service: search-service

        - record: sli:availability:error_ratio:rate1h
          expr: |
            sum(rate(http_requests_total{service="search-service", status_code=~"5.."}[1h]))
            /
            clamp_min(sum(rate(http_requests_total{service="search-service"}[1h])), 1)
          labels:
            slo_service: search-service

        - record: sli:availability:error_ratio:rate6h
          expr: |
            sum(rate(http_requests_total{service="search-service", status_code=~"5.."}[6h]))
            /
            clamp_min(sum(rate(http_requests_total{service="search-service"}[6h])), 1)
          labels:
            slo_service: search-service

        - record: sli:availability:error_ratio:rate3d
          expr: |
            sum(rate(http_requests_total{service="search-service", status_code=~"5.."}[3d]))
            /
            clamp_min(sum(rate(http_requests_total{service="search-service"}[3d])), 1)
          labels:
            slo_service: search-service

        - record: sli:latency:good_ratio:rate5m
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="search-service", le="0.2"}[5m]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="search-service"}[5m])), 1)
          labels:
            slo_service: search-service

        - record: sli:latency:good_ratio:rate1h
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="search-service", le="0.2"}[1h]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="search-service"}[1h])), 1)
          labels:
            slo_service: search-service

        - record: sli:latency:good_ratio:rate6h
          expr: |
            sum(rate(http_request_duration_seconds_bucket{service="search-service", le="0.2"}[6h]))
            /
            clamp_min(sum(rate(http_request_duration_seconds_count{service="search-service"}[6h])), 1)
          labels:
            slo_service: search-service

        - record: sli:availability:budget_remaining:ratio
          expr: |
            1 - (
              (
                sum(rate(http_requests_total{service="search-service", status_code=~"5.."}[30d]))
                /
                clamp_min(sum(rate(http_requests_total{service="search-service"}[30d])), 1)
              )
              / 0.001
            )
          labels:
            slo_service: search-service
            slo_target: "99.9"

        - record: sli:http_requests:total:rate5m
          expr: sum(rate(http_requests_total{service="search-service"}[5m]))
          labels:
            slo_service: search-service

    # ══════════════════════════════════════════════════════════
    # USER SERVICE — 99.95% availability
    # ══════════════════════════════════════════════════════════
    - name: slo.user.recording
      interval: 30s
      rules:
        - record: sli:http_requests:rate5m
          expr: sum by (service) (rate(http_requests_total{service="user-service"}[5m]))
          labels:
            slo_service: user-service

        - record: sli:http_errors:rate5m
          expr: sum by (service) (rate(http_requests_total{service="user-service", status_code=~"5.."}[5m]))
          labels:
            slo_service: user-service

        - record: sli:availability:error_ratio:rate5m
          expr: |
            sli:http_errors:rate5m{slo_service="user-service"}
            /
            clamp_min(sli:http_requests:rate5m{slo_service="user-service"}, 1)
          labels:
            slo_service: user-service

        - record: sli:availability:error_ratio:rate1h
          expr: |
            sum(rate(http_requests_total{service="user-service", status_code=~"5.."}[1h]))
            /
            clamp_min(sum(rate(http_requests_total{service="user-service"}[1h])), 1)
          labels:
            slo_service: user-service

        - record: sli:availability:error_ratio:rate6h
          expr: |
            sum(rate(http_requests_total{service="user-service", status_code=~"5.."}[6h]))
            /
            clamp_min(sum(rate(http_requests_total{service="user-service"}[6h])), 1)
          labels:
            slo_service: user-service

        - record: sli:availability:error_ratio:rate3d
          expr: |
            sum(rate(http_requests_total{service="user-service", status_code=~"5.."}[3d]))
            /
            clamp_min(sum(rate(http_requests_total{service="user-service"}[3d])), 1)
          labels:
            slo_service: user-service

        - record: sli:availability:budget_remaining:ratio
          expr: |
            1 - (
              (
                sum(rate(http_requests_total{service="user-service", status_code=~"5.."}[30d]))
                /
                clamp_min(sum(rate(http_requests_total{service="user-service"}[30d])), 1)
              )
              / 0.0005
            )
          labels:
            slo_service: user-service
            slo_target: "99.95"

        - record: sli:http_requests:total:rate5m
          expr: sum(rate(http_requests_total{service="user-service"}[5m]))
          labels:
            slo_service: user-service

    # ══════════════════════════════════════════════════════════
    # NOTIFICATION SERVICE — 99.0% freshness (<30s delivery)
    # ══════════════════════════════════════════════════════════
    - name: slo.notification.recording
      interval: 30s
      rules:
        - record: sli:freshness:good_ratio:rate5m
          expr: |
            sum(rate(notification_delivery_duration_seconds_bucket{le="30.0"}[5m]))
            /
            clamp_min(sum(rate(notification_delivery_duration_seconds_count[5m])), 1)
          labels:
            slo_service: notification-service

        - record: sli:freshness:good_ratio:rate1h
          expr: |
            sum(rate(notification_delivery_duration_seconds_bucket{le="30.0"}[1h]))
            /
            clamp_min(sum(rate(notification_delivery_duration_seconds_count[1h])), 1)
          labels:
            slo_service: notification-service

        - record: sli:freshness:good_ratio:rate6h
          expr: |
            sum(rate(notification_delivery_duration_seconds_bucket{le="30.0"}[6h]))
            /
            clamp_min(sum(rate(notification_delivery_duration_seconds_count[6h])), 1)
          labels:
            slo_service: notification-service

        - record: sli:freshness:budget_remaining:ratio
          expr: |
            1 - (
              (
                1 - (
                  sum(rate(notification_delivery_duration_seconds_bucket{le="30.0"}[30d]))
                  /
                  clamp_min(sum(rate(notification_delivery_duration_seconds_count[30d])), 1)
                )
              )
              / 0.01
            )
          labels:
            slo_service: notification-service
            slo_target: "99.0"

    # ══════════════════════════════════════════════════════════
    # PLATFORM INTERNAL SLOs
    # ══════════════════════════════════════════════════════════
    - name: slo.platform.recording
      interval: 30s
      rules:
        # --- CI/CD: 99% build success over 7d ---
        - record: sli:cicd:success_ratio:rate1h
          expr: |
            sum(rate(jenkins_builds_total{result="SUCCESS"}[1h]))
            /
            clamp_min(sum(rate(jenkins_builds_total[1h])), 1)
          labels:
            slo_service: cicd-pipeline

        - record: sli:cicd:budget_remaining:ratio
          expr: |
            1 - (
              (
                1 - (
                  sum(rate(jenkins_builds_total{result="SUCCESS"}[7d]))
                  /
                  clamp_min(sum(rate(jenkins_builds_total[7d])), 1)
                )
              )
              / 0.01
            )
          labels:
            slo_service: cicd-pipeline
            slo_target: "99.0"

        # --- ArgoCD: 99.9% sync freshness (<5min lag) ---
        - record: sli:argocd:sync_fresh_ratio
          expr: |
            count(argocd_app_info{sync_status="Synced"})
            /
            clamp_min(count(argocd_app_info), 1)
          labels:
            slo_service: argocd

        # --- Observability: 99.95% Prometheus up ---
        - record: sli:prometheus:up_ratio:rate1h
          expr: |
            avg_over_time(up{job="prometheus"}[1h])
          labels:
            slo_service: observability

        # --- CoreDNS: 99.99% latency <10ms ---
        - record: sli:coredns:latency_good_ratio:rate5m
          expr: |
            sum(rate(coredns_dns_request_duration_seconds_bucket{le="0.01"}[5m]))
            /
            clamp_min(sum(rate(coredns_dns_request_duration_seconds_count[5m])), 1)
          labels:
            slo_service: coredns
```

## 1.2 Burn Rate Alerts

```yaml
# manifests/platform/monitoring/slo-burn-rate-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-burn-rate-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
    app.kubernetes.io/part-of: novamart-slo
spec:
  groups:
    # ══════════════════════════════════════════════════════════
    # API GATEWAY — Target: 99.95% (budget: 0.0005)
    # ══════════════════════════════════════════════════════════
    - name: slo.burn.apigateway
      rules:
        # ── PAGE (14.4x burn in 1h AND 6x in 6h) ──
        # Exhausts budget in ~50 hours
        # TRAP 6 HANDLED: minimum request volume gate prevents
        # low-traffic false alarms (require > 1 req/s = 300 req in 5m)
        - alert: APIGatewayAvailabilityBurnCritical
          expr: |
            (
              sli:availability:error_ratio:rate1h{slo_service="api-gateway"} > (14.4 * 0.0005)
              and
              sli:availability:error_ratio:rate6h{slo_service="api-gateway"} > (6 * 0.0005)
            )
            and
            sli:http_requests:total:rate5m{slo_service="api-gateway"} > 1
          for: 2m
          labels:
            severity: critical
            slo: apigateway-availability
            slo_target: "99.95"
            team: platform
          annotations:
            summary: "API Gateway availability burning 14.4x error budget"
            description: |
              1h error ratio: {{ with printf `sli:availability:error_ratio:rate1h{slo_service="api-gateway"}` | query }}{{ . | first | value | humanizePercentage }}{{ end }}
              6h error ratio: {{ with printf `sli:availability:error_ratio:rate6h{slo_service="api-gateway"}` | query }}{{ . | first | value | humanizePercentage }}{{ end }}
              Budget: 0.05% (21.6 min/month). At 14.4x, exhausted in ~50hrs.
            runbook_url: "https://runbooks.novamart.internal/slo/apigateway-availability"
            dashboard_url: "https://grafana.novamart.internal/d/slo-overview?var-service=api-gateway"

        # ── PAGE (6x burn in 1h AND 3x in 3d) ──
        # Exhausts budget in ~5 days
        - alert: APIGatewayAvailabilityBurnHigh
          expr: |
            (
              sli:availability:error_ratio:rate1h{slo_service="api-gateway"} > (6 * 0.0005)
              and
              sli:availability:error_ratio:rate3d{slo_service="api-gateway"} > (3 * 0.0005)
            )
            and
            sli:http_requests:total:rate5m{slo_service="api-gateway"} > 1
          for: 5m
          labels:
            severity: critical
            slo: apigateway-availability
            team: platform
          annotations:
            summary: "API Gateway availability burning 6x error budget"
            description: "Sustained elevated error rate. Budget exhausted in ~5 days."
            runbook_url: "https://runbooks.novamart.internal/slo/apigateway-availability"
            dashboard_url: "https://grafana.novamart.internal/d/slo-overview?var-service=api-gateway"

        # ── TICKET (3x burn over 3d) ──
        # Slow bleed — fix this sprint
        - alert: APIGatewayAvailabilityBurnSlow
          expr: |
            sli:availability:error_ratio:rate3d{slo_service="api-gateway"} > (3 * 0.0005)
            and
            sli:http_requests:total:rate5m{slo_service="api-gateway"} > 1
          for: 30m
          labels:
            severity: warning
            slo: apigateway-availability
            team: platform
          annotations:
            summary: "API Gateway slow-burning error budget"
            description: "3-day error rate at 3x budget consumption. Needs attention."
            runbook_url: "https://runbooks.novamart.internal/slo/apigateway-availability"

        # ── LATENCY: Same multi-window for p99<500ms at 99.0% ──
        - alert: APIGatewayLatencyBurnCritical
          expr: |
            (
              (1 - sli:latency:good_ratio:rate1h{slo_service="api-gateway"}) > (14.4 * 0.01)
              and
              (1 - sli:latency:good_ratio:rate6h{slo_service="api-gateway"}) > (6 * 0.01)
            )
            and
            sli:http_requests:total:rate5m{slo_service="api-gateway"} > 1
          for: 2m
          labels:
            severity: critical
            slo: apigateway-latency
            slo_target: "99.0"
            team: platform
          annotations:
            summary: "API Gateway latency SLO burning critically"
            description: "Too many requests exceeding 500ms threshold."
            runbook_url: "https://runbooks.novamart.internal/slo/apigateway-latency"
            dashboard_url: "https://grafana.novamart.internal/d/slo-overview?var-service=api-gateway"

        - alert: APIGatewayLatencyBurnSlow
          expr: |
            (1 - sli:latency:good_ratio:rate3d{slo_service="api-gateway"}) > (3 * 0.01)
            and
            sli:http_requests:total:rate5m{slo_service="api-gateway"} > 1
          for: 30m
          labels:
            severity: warning
            slo: apigateway-latency
            team: platform
          annotations:
            summary: "API Gateway latency slow-burning budget"
            runbook_url: "https://runbooks.novamart.internal/slo/apigateway-latency"

    # ══════════════════════════════════════════════════════════
    # PAYMENT SERVICE — Target: 99.99% (budget: 0.0001)
    # CRITICAL: 4.32min/month budget. Any burn is urgent.
    # ══════════════════════════════════════════════════════════
    - name: slo.burn.payment
      rules:
        - alert: PaymentAvailabilityBurnCritical
          expr: |
            (
              sli:availability:error_ratio:rate1h{slo_service="payment-service"} > (14.4 * 0.0001)
              and
              sli:availability:error_ratio:rate6h{slo_service="payment-service"} > (6 * 0.0001)
            )
            and
            sli:http_requests:total:rate5m{slo_service="payment-service"} > 0.5
          for: 1m   # Faster than others — budget is tiny
          labels:
            severity: critical
            slo: payment-availability
            slo_target: "99.99"
            team: platform
            pci: "true"
          annotations:
            summary: "🚨 PAYMENT availability burning 14.4x error budget"
            description: |
              Payment SLO: 99.99% (budget: 4.32 min/month).
              14.4x burn = budget exhausted in ~8 hours.
              IMMEDIATE ACTION REQUIRED.
            runbook_url: "https://runbooks.novamart.internal/runbook-app-001"
            dashboard_url: "https://grafana.novamart.internal/d/slo-overview?var-service=payment-service"

        - alert: PaymentAvailabilityBurnHigh
          expr: |
            (
              sli:availability:error_ratio:rate1h{slo_service="payment-service"} > (6 * 0.0001)
              and
              sli:availability:error_ratio:rate3d{slo_service="payment-service"} > (3 * 0.0001)
            )
            and
            sli:http_requests:total:rate5m{slo_service="payment-service"} > 0.5
          for: 2m
          labels:
            severity: critical
            slo: payment-availability
            team: platform
            pci: "true"
          annotations:
            summary: "🚨 PAYMENT availability burning 6x error budget"
            description: "Sustained payment errors. Budget exhausted in ~2 days."
            runbook_url: "https://runbooks.novamart.internal/runbook-app-001"

        - alert: PaymentAvailabilityBurnSlow
          expr: |
            sli:availability:error_ratio:rate3d{slo_service="payment-service"} > (3 * 0.0001)
            and
            sli:http_requests:total:rate5m{slo_service="payment-service"} > 0.5
          for: 15m
          labels:
            severity: warning
            slo: payment-availability
            team: platform
            pci: "true"
          annotations:
            summary: "PAYMENT availability slow-burning budget"
            runbook_url: "https://runbooks.novamart.internal/runbook-app-001"

        # ── CORRECTNESS — near-zero tolerance ──
        - alert: PaymentCorrectnessViolation
          expr: |
            (1 - sli:correctness:good_ratio:rate5m{slo_service="payment-service"}) > 0.00001
            and
            sli:http_requests:total:rate5m{slo_service="payment-service"} > 0.1
          for: 1m
          labels:
            severity: critical
            slo: payment-correctness
            slo_target: "99.999"
            team: platform
            pci: "true"
          annotations:
            summary: "🚨🚨 PAYMENT CORRECTNESS VIOLATION"
            description: |
              Incorrect charges detected. Financial integrity at risk.
              SLO: 99.999% (budget: 26 sec/month).
              CHECK: Gateway responses, currency conversion, rounding, idempotency.
              THIS OVERRIDES ALL OTHER WORK.
            runbook_url: "https://runbooks.novamart.internal/slo/payment-correctness"

        # ── LATENCY (99.5%, 1000ms) ──
        - alert: PaymentLatencyBurnCritical
          expr: |
            (
              (1 - sli:latency:good_ratio:rate1h{slo_service="payment-service"}) > (14.4 * 0.005)
              and
              (1 - sli:latency:good_ratio:rate6h{slo_service="payment-service"}) > (6 * 0.005)
            )
            and
            sli:http_requests:total:rate5m{slo_service="payment-service"} > 0.5
          for: 2m
          labels:
            severity: critical
            slo: payment-latency
            team: platform
            pci: "true"
          annotations:
            summary: "PAYMENT latency SLO burning critically"
            description: "Too many payment requests exceeding 1000ms."
            runbook_url: "https://runbooks.novamart.internal/slo/payment-latency"

    # ══════════════════════════════════════════════════════════
    # CART SERVICE — Target: 99.9% (budget: 0.001)
    # ══════════════════════════════════════════════════════════
    - name: slo.burn.cart
      rules:
        - alert: CartAvailabilityBurnCritical
          expr: |
            (
              sli:availability:error_ratio:rate1h{slo_service="cart-service"} > (14.4 * 0.001)
              and
              sli:availability:error_ratio:rate6h{slo_service="cart-service"} > (6 * 0.001)
            )
            and
            sli:http_requests:total:rate5m{slo_service="cart-service"} > 1
          for: 2m
          labels:
            severity: critical
            slo: cart-availability
            team: platform
          annotations:
            summary: "Cart service availability burning error budget critically"
            runbook_url: "https://runbooks.novamart.internal/slo/cart-availability"
            dashboard_url: "https://grafana.novamart.internal/d/slo-overview?var-service=cart-service"

        - alert: CartAvailabilityBurnSlow
          expr: |
            sli:availability:error_ratio:rate3d{slo_service="cart-service"} > (3 * 0.001)
            and
            sli:http_requests:total:rate5m{slo_service="cart-service"} > 1
          for: 30m
          labels:
            severity: warning
            slo: cart-availability
            team: platform
          annotations:
            summary: "Cart service slow-burning error budget"
            runbook_url: "https://runbooks.novamart.internal/slo/cart-availability"

        - alert: CartLatencyBurnCritical
          expr: |
            (
              (1 - sli:latency:good_ratio:rate1h{slo_service="cart-service"}) > (14.4 * 0.005)
              and
              (1 - sli:latency:good_ratio:rate6h{slo_service="cart-service"}) > (6 * 0.005)
            )
            and
            sli:http_requests:total:rate5m{slo_service="cart-service"} > 1
          for: 2m
          labels:
            severity: critical
            slo: cart-latency
            team: platform
          annotations:
            summary: "Cart latency burning budget — p99 exceeding 300ms"
            runbook_url: "https://runbooks.novamart.internal/slo/cart-latency"

    # ══════════════════════════════════════════════════════════
    # ORDER SERVICE — Target: 99.95% avail (0.0005), 99.9% freshness (0.001)
    # ══════════════════════════════════════════════════════════
    - name: slo.burn.order
      rules:
        - alert: OrderAvailabilityBurnCritical
          expr: |
            (
              sli:availability:error_ratio:rate1h{slo_service="order-service"} > (14.4 * 0.0005)
              and
              sli:availability:error_ratio:rate6h{slo_service="order-service"} > (6 * 0.0005)
            )
            and
            sli:http_requests:total:rate5m{slo_service="order-service"} > 1
          for: 2m
          labels:
            severity: critical
            slo: order-availability
            team: platform
          annotations:
            summary: "Order service availability burning error budget critically"
            runbook_url: "https://runbooks.novamart.internal/slo/order-availability"

        - alert: OrderAvailabilityBurnSlow
          expr: |
            sli:availability:error_ratio:rate3d{slo_service="order-service"} > (3 * 0.0005)
            and
            sli:http_requests:total:rate5m{slo_service="order-service"} > 1
          for: 30m
          labels:
            severity: warning
            slo: order-availability
            team: platform
          annotations:
            summary: "Order service slow-burning error budget"
            runbook_url: "https://runbooks.novamart.internal/slo/order-availability"

        - alert: OrderFreshnessBurnCritical
          expr: |
            (
              (1 - sli:freshness:good_ratio:rate1h{slo_service="order-service"}) > (14.4 * 0.001)
              and
              (1 - sli:freshness:good_ratio:rate6h{slo_service="order-service"}) > (6 * 0.001)
            )
          for: 2m
          labels:
            severity: critical
            slo: order-freshness
            team: platform
          annotations:
            summary: "Order status updates exceeding 5s threshold at unsustainable rate"
            runbook_url: "https://runbooks.novamart.internal/slo/order-freshness"

    # ══════════════════════════════════════════════════════════
    # SEARCH — Target: 99.9% avail, 99.0% latency
    # ══════════════════════════════════════════════════════════
    - name: slo.burn.search
      rules:
        - alert: SearchAvailabilityBurnCritical
          expr: |
            (
              sli:availability:error_ratio:rate1h{slo_service="search-service"} > (14.4 * 0.001)
              and
              sli:availability:error_ratio:rate6h{slo_service="search-service"} > (6 * 0.001)
            )
            and
            sli:http_requests:total:rate5m{slo_service="search-service"} > 1
          for: 2m
          labels:
            severity: critical
            slo: search-availability
            team: platform
          annotations:
            summary: "Search service availability burning error budget"
            runbook_url: "https://runbooks.novamart.internal/slo/search-availability"

        - alert: SearchLatencyBurnCritical
          expr: |
            (
              (1 - sli:latency:good_ratio:rate1h{slo_service="search-service"}) > (14.4 * 0.01)
              and
              (1 - sli:latency:good_ratio:rate6h{slo_service="search-service"}) > (6 * 0.01)
            )
            and
            sli:http_requests:total:rate5m{slo_service="search-service"} > 1
          for: 2m
          labels:
            severity: critical
            slo: search-latency
            team: platform
          annotations:
            summary: "Search latency burning budget — p99 exceeding 200ms"
            runbook_url: "https://runbooks.novamart.internal/slo/search-latency"

    # ══════════════════════════════════════════════════════════
    # META ALERTS — Budget exhaustion + volume anomaly
    # ══════════════════════════════════════════════════════════
    - name: slo.meta
      rules:
        # Budget fully exhausted — any service
        - alert: SLOBudgetExhausted
          expr: |
            sli:availability:budget_remaining:ratio < 0
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "SLO budget EXHAUSTED for {{ $labels.slo_service }}"
            description: |
              Service {{ $labels.slo_service }} (target: {{ $labels.slo_target }}%) has
              consumed 100% of its 30-day error budget.
              Error budget policy: FULL FREEZE. Reliability work only. VP sign-off for any deploy.
            runbook_url: "https://runbooks.novamart.internal/slo/budget-exhausted"

        # Budget critically low — approaching exhaustion
        - alert: SLOBudgetLow
          expr: |
            sli:availability:budget_remaining:ratio > 0
            and
            sli:availability:budget_remaining:ratio < 0.10
          for: 15m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "SLO budget below 10% for {{ $labels.slo_service }}"
            description: |
              {{ $labels.slo_service }} has less than 10% error budget remaining.
              Error budget policy: RED state. Feature freeze. Reliability work only.

        # Correctness budget — separate because different metric
        - alert: SLOCorrectnesssBudgetExhausted
          expr: |
            sli:correctness:budget_remaining:ratio{slo_service="payment-service"} < 0
          for: 1m
          labels:
            severity: critical
            team: platform
            pci: "true"
          annotations:
            summary: "🚨🚨 PAYMENT CORRECTNESS BUDGET EXHAUSTED"
            description: "Payment correctness SLO violated. Financial integrity compromised."
            runbook_url: "https://runbooks.novamart.internal/slo/payment-correctness"

        # Detect zero traffic (services that should ALWAYS have traffic)
        # This is the "absence of signal" problem — separate from SLO
        - alert: ServiceNoTraffic
          expr: |
            sli:http_requests:total:rate5m{slo_service=~"api-gateway|payment-service|cart-service|order-service"} == 0
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "{{ $labels.slo_service }} receiving ZERO traffic"
            description: |
              Service {{ $labels.slo_service }} has had no requests for 5 minutes.
              This could indicate: routing issue, DNS failure, upstream problem, or deployment issue.
              NOTE: SLO burn rate alerts are suppressed during zero-traffic (by design).
              Investigate the traffic path, not the error rate.
            runbook_url: "https://runbooks.novamart.internal/slo/no-traffic"

        # Detect absent SLI recording rules (monitoring the monitor)
        # TRAP 8 HANDLED: If recording rules fail, alerts silently stop.
        # This detects that condition.
        - alert: SLIRecordingRuleAbsent
          expr: |
            absent(sli:availability:error_ratio:rate5m{slo_service="api-gateway"})
            or
            absent(sli:availability:error_ratio:rate5m{slo_service="payment-service"})
            or
            absent(sli:availability:error_ratio:rate5m{slo_service="cart-service"})
            or
            absent(sli:availability:error_ratio:rate5m{slo_service="order-service"})
          for: 10m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "SLI recording rule missing — SLO alerting is BLIND"
            description: |
              One or more SLI recording rules are not producing data.
              Burn rate alerts CANNOT fire without SLI data.
              CHECK: PrometheusRule CRD applied, Prometheus rule reload, metric availability.
            runbook_url: "https://runbooks.novamart.internal/runbook-plat-001"
```

## 1.3 Error Budget Policy

```markdown
# docs/operations/error-budget-policy.md

# NovaMart Error Budget Policy

**Version:** 1.0
**Effective:** 2024-01-15
**Owner:** Platform Engineering + VP Engineering
**Review Cycle:** Quarterly
**Approved By:** VP Engineering, CTO

---

## Purpose

This policy defines how NovaMart uses error budgets to balance feature velocity
with reliability. It removes ambiguity about when to ship vs when to fix.

## Budget States and Required Actions

| State     | Budget Remaining | Velocity Impact                          | Deploy Rules                               | Escalation             |
|-----------|------------------|------------------------------------------|--------------------------------------------|------------------------|
| GREEN     | > 50%            | Full velocity                            | Normal CI/CD flow                          | None                   |
| YELLOW    | 25–50%           | Increased review rigor                   | No experimental/feature-flag deploys       | Eng Lead notified      |
| ORANGE    | 10–25%           | Feature freeze for this service          | Bug fixes + reliability work only          | VP Engineering notified|
| RED       | < 10%            | Full freeze                              | VP sign-off per deploy. Platform escort.   | VP + CTO notified      |
| EXHAUSTED | ≤ 0%             | Emergency                                | No deploys until budget > 10%              | Incident declared      |

### State Transition Rules

- **GREEN → YELLOW:** Automated Slack notification to service team + eng lead.
- **YELLOW → ORANGE:** Automated Jira ticket created. Eng lead must respond within 24h
  with reliability action plan.
- **ORANGE → RED:** VP Engineering approves/rejects continued operation. Platform team
  assists with root cause investigation.
- **RED → EXHAUSTED:** Incident declared automatically. Postmortem required.
  No deploys to affected service until budget recovers to 10%.
- **Recovery:** State improves as the 30-day rolling window moves past the error period.
  No manual override. The math decides.

## Exceptions

### Always Allowed (Any Budget State)
1. **Security patches:** CVE fixes with CVSS ≥ 7.0.
2. **Data integrity fixes:** Preventing data loss or corruption.
3. **Legal/compliance requirements:** PCI-DSS, GDPR mandated changes.
4. **Rollbacks:** Reverting a bad deploy (this improves the SLO, not hurts it).

### Requires Approval (ORANGE/RED/EXHAUSTED)
1. **Revenue-critical hotfix:** CTO written approval + platform team escort deploy.
   - Must include: business justification, risk assessment, rollback plan.
   - Deploy must use canary with abort-on-first-error.
2. **Scheduled maintenance with known SLO impact:**
   - Pre-approved budget allocation. Deducted from budget forecast.
   - Max 10% of remaining budget per maintenance window.
   - Communicated to status page 48h in advance.

## Measurement

- **Window:** 30-day rolling (not calendar month).
- **Source of truth:** Prometheus recording rules + Grafana SLO dashboard.
- **Refresh:** Real-time (recording rules evaluate every 30s).
- **Dispute:** If a team believes an incident was caused by external factors
  (AWS outage, upstream provider), they may petition to exclude that time window.
  Requires: evidence, Architecture Review Board approval, 2-week observation.

## Weekly Review Process

1. **Monday 9:00 AM:** Platform team reviews all SLO dashboards.
2. **Monday 9:30 AM:** SLO Review Meeting with all engineering leads.
   - Review each service's budget state.
   - Services in ORANGE+ get dedicated discussion + action items.
   - Prior week's action items reviewed for completion.
3. **Monthly report:** Platform → VP Engineering with trends, budget trajectories,
   and reliability investment recommendations.

## Accountability

| Role               | Responsibility                                          |
|--------------------|---------------------------------------------------------|
| Service Team       | Own their SLOs. Fix reliability issues in their service.|
| Platform Team      | Maintain SLO infrastructure. Assist with diagnosis.     |
| Engineering Leads  | Prioritize reliability work when budget dictates.        |
| VP Engineering     | Approve/reject deploys in RED state. Resource decisions. |
| CTO                | Final authority on business-critical exceptions.         |

## Anti-Gaming Rules

1. **SLI measurement is at the mesh/LB level**, not application-reported.
   Apps cannot route errors around the measurement point.
2. **Error budget accounting is automated.** No manual adjustments without
   Architecture Review Board approval + documented justification.
3. **Test traffic is excluded** via label filtering (synthetic=true).
4. **Canary traffic counts.** If your canary errors burn budget, that's the system working.
5. If a team is consistently at 100% budget remaining, their target is likely
   too loose. Quarterly review should propose tightening.

---

_"When the budget is gone, the budget is gone. Ship reliability, not features."_
```

## 1.4 Grafana SLO Dashboard

```json
// manifests/platform/monitoring/dashboards/slo-overview.json
// Dashboard: NovaMart SLO Overview
// This is the single most important dashboard in the company.
{
  "dashboard": {
    "title": "NovaMart SLO Overview",
    "uid": "slo-overview",
    "tags": ["slo", "novamart", "overview"],
    "timezone": "utc",
    "refresh": "30s",
    "templating": {
      "list": [
        {
          "name": "service",
          "type": "query",
          "query": "label_values(sli:availability:error_ratio:rate5m, slo_service)",
          "current": { "text": "All", "value": "$__all" },
          "includeAll": true,
          "multi": true
        }
      ]
    },
    "panels": [
      {
        "_comment": "ROW 1: Budget Overview — The 'executive summary' row",
        "title": "Error Budget Remaining — All Services",
        "type": "table",
        "gridPos": { "h": 6, "w": 24, "x": 0, "y": 0 },
        "targets": [
          {
            "expr": "sli:availability:budget_remaining:ratio * 100",
            "legendFormat": "{{ slo_service }} ({{ slo_target }}%)",
            "format": "table",
            "instant": true
          }
        ],
        "fieldConfig": {
          "overrides": [
            {
              "matcher": { "id": "byName", "options": "Value" },
              "properties": [
                {
                  "id": "thresholds",
                  "value": {
                    "mode": "absolute",
                    "steps": [
                      { "value": null, "color": "dark-red" },
                      { "value": 0, "color": "red" },
                      { "value": 10, "color": "orange" },
                      { "value": 25, "color": "yellow" },
                      { "value": 50, "color": "green" }
                    ]
                  }
                },
                { "id": "unit", "value": "percent" },
                { "id": "custom.displayMode", "value": "color-background" }
              ]
            }
          ]
        }
      },
      {
        "_comment": "ROW 2: Per-service burn rate gauges",
        "title": "Burn Rate — $service — 1h window",
        "type": "gauge",
        "gridPos": { "h": 6, "w": 8, "x": 0, "y": 6 },
        "targets": [
          {
            "expr": "sli:availability:error_ratio:rate1h{slo_service=~\"$service\"} / on(slo_service) group_left sli:availability:error_ratio:rate5m{slo_service=~\"$service\"} * 0 + 0.0005",
            "_comment_real": "Simplified: actual_error_rate / allowed_error_rate. Using the target's budget directly.",
            "expr_actual": "sli:availability:error_ratio:rate1h{slo_service=~\"$service\"} / 0.0005",
            "legendFormat": "{{ slo_service }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "value": null, "color": "green" },
                { "value": 1, "color": "green" },
                { "value": 3, "color": "yellow" },
                { "value": 6, "color": "orange" },
                { "value": 14.4, "color": "red" }
              ]
            },
            "min": 0,
            "max": 20,
            "unit": "x"
          }
        }
      },
      {
        "title": "Burn Rate — $service — 6h window",
        "type": "gauge",
        "gridPos": { "h": 6, "w": 8, "x": 8, "y": 6 },
        "targets": [
          {
            "expr": "sli:availability:error_ratio:rate6h{slo_service=~\"$service\"} / 0.0005",
            "legendFormat": "{{ slo_service }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "value": null, "color": "green" },
                { "value": 1, "color": "green" },
                { "value": 3, "color": "yellow" },
                { "value": 6, "color": "orange" },
                { "value": 14.4, "color": "red" }
              ]
            },
            "min": 0,
            "max": 20,
            "unit": "x"
          }
        }
      },
      {
        "title": "Burn Rate — $service — 3d window",
        "type": "gauge",
        "gridPos": { "h": 6, "w": 8, "x": 16, "y": 6 },
        "targets": [
          {
            "expr": "sli:availability:error_ratio:rate3d{slo_service=~\"$service\"} / 0.0005",
            "legendFormat": "{{ slo_service }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "value": null, "color": "green" },
                { "value": 1, "color": "green" },
                { "value": 3, "color": "yellow" },
                { "value": 6, "color": "orange" },
                { "value": 14.4, "color": "red" }
              ]
            },
            "min": 0,
            "max": 20,
            "unit": "x"
          }
        }
      },
      {
        "_comment": "ROW 3: Error budget remaining over time — shows budget consumption trajectory",
        "title": "Error Budget Remaining Over Time — $service",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 24, "x": 0, "y": 12 },
        "targets": [
          {
            "expr": "sli:availability:budget_remaining:ratio{slo_service=~\"$service\"} * 100",
            "legendFormat": "{{ slo_service }} ({{ slo_target }}%)"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": -20,
            "max": 100,
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "value": null, "color": "red" },
                { "value": 0, "color": "red" },
                { "value": 10, "color": "orange" },
                { "value": 25, "color": "yellow" },
                { "value": 50, "color": "green" }
              ]
            },
            "custom": {
              "thresholdsStyle": { "mode": "area" },
              "lineWidth": 2,
              "fillOpacity": 10
            }
          }
        },
        "options": {
          "tooltip": { "mode": "multi" }
        }
      },
      {
        "_comment": "ROW 4: Error ratio time series — detailed view of error rate",
        "title": "Error Ratio (5m rate) — $service",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 20 },
        "targets": [
          {
            "expr": "sli:availability:error_ratio:rate5m{slo_service=~\"$service\"} * 100",
            "legendFormat": "{{ slo_service }} error rate"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "custom": { "lineWidth": 2, "fillOpacity": 15 }
          }
        }
      },
      {
        "title": "Latency Good Ratio (5m rate) — $service",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 20 },
        "targets": [
          {
            "expr": "sli:latency:good_ratio:rate5m{slo_service=~\"$service\"} * 100",
            "legendFormat": "{{ slo_service }} good latency %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 90,
            "max": 100,
            "custom": { "lineWidth": 2, "fillOpacity": 15 }
          }
        }
      },
      {
        "_comment": "ROW 5: Request volume — context for error rates",
        "title": "Request Volume — $service",
        "type": "timeseries",
        "gridPos": { "h": 6, "w": 12, "x": 0, "y": 28 },
        "targets": [
          {
            "expr": "sli:http_requests:total:rate5m{slo_service=~\"$service\"}",
            "legendFormat": "{{ slo_service }} req/s"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "custom": { "lineWidth": 2, "fillOpacity": 10 }
          }
        }
      },
      {
        "_comment": "Deploy annotations — overlay deploy events on all panels",
        "title": "Recent Deploys + Incidents",
        "type": "timeseries",
        "gridPos": { "h": 6, "w": 12, "x": 12, "y": 28 },
        "targets": [
          {
            "expr": "sli:availability:error_ratio:rate5m{slo_service=~\"$service\"} * 100",
            "legendFormat": "{{ slo_service }}"
          }
        ],
        "_annotations_comment": "Configured at dashboard level, overlays on all panels",
        "options": {
          "tooltip": { "mode": "multi" }
        }
      },
      {
        "_comment": "ROW 6: SLO compliance table — 30d rolling summary",
        "title": "30-Day SLO Compliance",
        "type": "table",
        "gridPos": { "h": 8, "w": 24, "x": 0, "y": 34 },
        "targets": [
          {
            "expr": "(1 - sli:availability:error_ratio:rate5m{slo_service=~\"$service\"}) * 100",
            "legendFormat": "{{ slo_service }}",
            "format": "table",
            "instant": true
          }
        ],
        "transformations": [
          {
            "id": "organize",
            "options": {
              "renameByName": {
                "slo_service": "Service",
                "slo_target": "Target %",
                "Value": "Current %"
              }
            }
          }
        ]
      }
    ],
    "annotations": {
      "list": [
        {
          "name": "Deployments",
          "datasource": "Prometheus",
          "expr": "changes(kube_deployment_status_observed_generation{namespace=~\"payment|cart|order|search|user|notification|api-gateway\"}[2m]) > 0",
          "tagKeys": "namespace,deployment",
          "titleFormat": "Deploy: {{ namespace }}/{{ deployment }}",
          "iconColor": "blue",
          "enable": true
        },
        {
          "name": "Incidents",
          "datasource": "Prometheus",
          "expr": "ALERTS{alertname=~\".*BurnCritical.*\", alertstate=\"firing\"}",
          "tagKeys": "alertname,slo_service",
          "titleFormat": "🚨 {{ alertname }}",
          "iconColor": "red",
          "enable": true
        }
      ]
    }
  }
}
```

---

# DELIVERABLE 2: Runbooks

## RUNBOOK-APP-001: Payment Service 5xx Spike

```markdown
# RUNBOOK-APP-001: Payment Service 5xx Spike

## Metadata
- **Severity:** SEV1 (revenue-impacting)
- **Services affected:** payment-service, payment-gateway integrations (Stripe, Adyen)
- **SLO impact:** payment-availability (99.99%), payment-correctness (99.999%)
- **Budget at stake:** 4.32 min/month (availability), 26 sec/month (correctness)
- **Last tested:** 2024-11-15 (chaos drill: pod-kill + network-delay)
- **Owner:** Platform Engineering
- **PCI scope:** YES — all investigation commands must be logged in incident channel

## Detection
- **Primary alert:** `PaymentAvailabilityBurnCritical`
- **Secondary alert:** `PaymentCorrectnessViolation` (if charges are wrong, not just failing)
- **PagerDuty service:** novamart-payment-critical
- **Dashboard:** https://grafana.novamart.internal/d/slo-overview?var-service=payment-service
- **Backup dashboard (if Grafana down):** CloudWatch dashboard "NovaMart-Payment-Emergency"

## Impact Assessment (FIRST — 30 seconds max)

```bash
# 1. SCOPE: All regions or single region?
kubectl get pods -n payment -l app=payment-service -o wide | \
  awk '{print $7}' | sort | uniq -c
# If pods concentrated in one AZ and errors only from that AZ → AZ issue

# 2. SCOPE: All endpoints or specific route?
# Via Loki:
logcli query '{namespace="payment", app="payment-service"} |= "status=5" | json | line_format "{{.path}}"' \
  --limit=50 --since=5m
# Or via Prometheus:
# sum by (path) (rate(http_requests_total{service="payment-service", status_code=~"5.."}[5m]))

# 3. WHEN DID IT START? Correlate with events:
kubectl get events -n payment --sort-by='.lastTimestamp' | tail -20
argocd app history payment-service-production

# 4. REVENUE IMPACT (rough estimate):
# failed_charges/sec × $85 avg order × 60 = $/min
# If Prometheus available:
# sum(rate(payment_charge_total{status="failed"}[5m])) * 85 * 60

# 5. IS IT GETTING WORSE?
# Compare 1m rate vs 5m rate. If 1m >> 5m, accelerating.
```

## Quick Mitigation (STOP BLEEDING — before root cause)

### Scenario A: Started after a deployment
```bash
# 1. Verify recent deployment:
kubectl argo rollouts get rollout payment-service -n payment
# Look at: revision, status, creation timestamp

# 2. IMMEDIATE ROLLBACK:
kubectl argo rollouts abort payment-service -n payment
kubectl argo rollouts undo payment-service -n payment

# 3. Verify rollback completing:
kubectl argo rollouts get rollout payment-service -n payment --watch

# 4. Monitor error rate:
# Should drop within 60 seconds of rollback completing.
# If using Argo Rollouts canary, abort stops traffic shift immediately.
```
**Expected result:** Error rate returns to baseline within 2 minutes.
**If NOT:** Rollback isn't the cause. Proceed to Scenario B.

### Scenario B: NOT deployment-related
```bash
# 1. Check upstream payment gateways:
curl -sf https://status.stripe.com/api/v2/status.json | jq '.status.indicator'
curl -sf https://status.adyen.com/api/v2/status.json | jq '.status.indicator'
# Expected: "none" = healthy. Anything else = upstream issue.

# 2. If upstream is down — reduce retry storm:
# Linkerd retry budget should auto-limit, but if retry storm detected:
kubectl annotate service payment-service -n payment \
  config.linkerd.io/proxy-retry-budget="0%" --overwrite
# This stops ALL retries. Re-enable after upstream recovers:
# kubectl annotate service payment-service -n payment \
#   config.linkerd.io/proxy-retry-budget- (removes annotation)

# 3. If database overloaded — shed non-critical traffic:
# Scale down non-essential consumers (refund batch, reporting):
kubectl scale deployment payment-refund-worker -n payment --replicas=0
kubectl scale deployment payment-reporting -n payment --replicas=0
```

### Scenario C: Single bad pod
```bash
# 1. Identify outlier:
kubectl top pods -n payment -l app=payment-service --sort-by=cpu
kubectl get pods -n payment -l app=payment-service -o wide

# 2. Check for OOMKilled or CrashLoop:
kubectl get pods -n payment -l app=payment-service | grep -v Running

# 3. Kill bad pod (deployment recreates it):
kubectl delete pod <bad-pod> -n payment
# Wait 60 seconds, verify replacement is healthy.
```

**⏰ TIME LIMIT:** If error rate not decreasing within 10 minutes of mitigation attempt:
1. Declare SEV1 incident: `/incident declare` in Slack
2. Page Incident Commander (if not already paged)
3. Move to Diagnosis below

## Diagnosis

### Step 1: Classify the error type
```bash
# What status codes exactly?
kubectl logs -n payment -l app=payment-service --since=5m --tail=500 | \
  jq -r 'select(.status_code != null) | .status_code' | sort | uniq -c | sort -rn

# If Loki available:
logcli query '{namespace="payment", app="payment-service"} |= "status" | json | status_code >= 500' \
  --stats --since=5m
```

- **502/503 dominant:** → Step 2a (pod/mesh/routing issue)
- **504 dominant:** → Step 2b (timeout — database or upstream)
- **500 dominant:** → Step 2c (application error — code bug)
- **Mixed:** → Step 2d (cascading failure)

### Step 2a: 502/503 — Pod or Mesh Issue
```bash
# Pod readiness:
kubectl get pods -n payment -l app=payment-service \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,RESTARTS:.status.containerStatuses[*].restartCount'

# Linkerd proxy health:
kubectl exec -n payment <pod> -c linkerd-proxy -- \
  curl -sf localhost:4191/metrics | grep -E "response_total|tcp_open_total"
# If linkerd-proxy not responding → mesh data plane issue

# Linkerd control plane:
linkerd check --proxy -n payment
# If errors → see RUNBOOK-PLAT-005

# Check if endpoints are registered:
kubectl get endpointslices -n payment -l kubernetes.io/service-name=payment-service
# If endpoint count < expected pod count → readiness probe failing
```

### Step 2b: 504 — Timeout (Database or Upstream)
```bash
# Connection pool status (Spring Boot Actuator):
kubectl exec -n payment <pod> -c payment-service -- \
  curl -sf localhost:8080/actuator/metrics/hikaricp.connections.active 2>/dev/null | jq '.'
kubectl exec -n payment <pod> -c payment-service -- \
  curl -sf localhost:8080/actuator/metrics/hikaricp.connections.pending 2>/dev/null | jq '.'
# If active == max AND pending > 0 → pool exhausted → RUNBOOK-DATA-007

# Go service equivalent (custom metrics endpoint):
kubectl exec -n payment <pod> -c payment-service -- \
  curl -sf localhost:8080/debug/metrics | grep -i "pool\|connection"

# RDS health:
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=novamart-payment-prod \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Average \
  --query 'Datapoints[*].[Timestamp,Average]' --output table

# If CPU > 80%:
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=novamart-payment-prod \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Average \
  --query 'Datapoints[*].[Timestamp,Average]' --output table

# Check for long-running queries (requires RDS Performance Insights or direct DB access):
# If you have psql access:
# SELECT pid, now() - pg_stat_activity.query_start AS duration, query
# FROM pg_stat_activity WHERE state != 'idle'
# ORDER BY duration DESC LIMIT 10;

# Upstream timeout (Stripe/Adyen latency):
kubectl logs -n payment -l app=payment-service --since=5m | \
  jq 'select(.upstream != null and .duration_ms > 5000) | {upstream, duration_ms, trace_id}'
```

### Step 2c: 500 — Application Error
```bash
# Get stack traces:
kubectl logs -n payment -l app=payment-service --since=5m | \
  jq 'select(.level == "ERROR")' | head -50

# Get trace IDs for failed requests (for distributed trace investigation):
kubectl logs -n payment -l app=payment-service --since=5m | \
  jq 'select(.status_code >= 500) | .trace_id' | sort -u | head -10

# Open in Tempo:
# https://grafana.novamart.internal/explore?left={"datasource":"Tempo","queries":[{"query":"<traceID>","queryType":"traceql"}]}

# Look for patterns:
kubectl logs -n payment -l app=payment-service --since=5m | \
  jq 'select(.level == "ERROR") | .error' | sort | uniq -c | sort -rn | head -10
# This groups errors by message — tells you if it's one bug or many.
```

### Step 2d: Cascading Failure
```bash
# Check upstream services:
for svc in user-service order-service cart-service; do
  echo "=== $svc ==="
  kubectl get pods -n ${svc%-service} -l app=$svc --no-headers | \
    awk '{print $3}' | sort | uniq -c
done

# Check if payment is being overwhelmed (queue backup):
kubectl exec -n payment <pod> -c payment-service -- \
  curl -sf localhost:8080/actuator/metrics/executor.active 2>/dev/null | jq '.'

# Check RabbitMQ queue depth:
kubectl exec -n messaging rabbitmq-0 -- rabbitmqctl list_queues name messages | \
  sort -k2 -rn | head -10
# If payment queues have millions of messages → consumer can't keep up

# → If cascading: see RUNBOOK-APP-010
```

### Step 3: Fallback When Observability is Down (TRAP 2 HANDLED)

If Prometheus/Grafana/Loki are ALSO down during this incident:

```bash
# CloudWatch metrics (always available — AWS managed):
aws cloudwatch get-metric-statistics \
  --namespace NovaMart/Payment \
  --metric-name 5xxErrors \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Sum --output table

# Direct pod logs (kubectl logs doesn't need Prometheus):
kubectl logs -n payment -l app=payment-service --since=10m --tail=200

# ALB metrics (AWS-native):
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=TargetGroup,Value=<payment-tg-arn> \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Sum --output table

# If even kubectl is broken → AWS Console:
# EC2 → Target Groups → payment-service → Health checks
# CloudWatch → Dashboards → NovaMart-Payment-Emergency
```

## Root Cause Resolution

Resolution depends on diagnosis:

| Diagnosis | Resolution | Verification |
|-----------|-----------|-------------|
| Bad deployment | Rollback confirmed. Investigate code change. | Error rate at baseline for 15min. |
| Upstream (Stripe/Adyen) down | Circuit breaker active. Monitor status page. Remove annotation when recovered. | Upstream status page shows resolved. Payment success rate returns. |
| Database overloaded | Kill long queries. Scale read replicas. Tune connection pool. | DB CPU < 60%. Pool pending = 0. |
| Connection pool exhausted | Increase pool max. Add connection timeout. Fix leak. | See RUNBOOK-DATA-007 for detailed steps. |
| Memory leak → OOM | Rolling restart. File bug with heap dump. | No OOMKilled events for 1 hour. |
| Single bad pod | Deleted. Investigate logs for what made it unhealthy. | Replacement pod healthy. |
| Cascading failure | See RUNBOOK-APP-010. Likely need to shed load + fix upstream. | All services error rates at baseline. |

## Verification (after mitigation or resolution)

```bash
# 1. Error rate returning to normal:
watch -n 10 'curl -sf "http://prometheus.monitoring:9090/api/v1/query" \
  --data-urlencode "query=sli:availability:error_ratio:rate5m{slo_service=\"payment-service\"}" | \
  jq -r ".data.result[0].value[1]"'
# Target: < 0.0001 (0.01%) for 99.99% SLO

# 2. No new 5xx in logs:
kubectl logs -n payment -l app=payment-service --since=2m | \
  jq 'select(.status_code >= 500)' | wc -l
# Target: 0

# 3. SLO burn rate alert resolved:
kubectl get alerts -n monitoring | grep -i payment
# Target: No firing payment alerts

# 4. Monitor for 30 minutes before closing.
```

## Escalation Matrix

| Time Since Detection | Action                                              |
|---------------------|------------------------------------------------------|
| 0 min               | On-call acknowledges, opens #incident-YYYYMMDD       |
| 5 min               | If not classified → page secondary on-call           |
| 10 min              | If not mitigated → declare SEV1, page IC              |
| 15 min              | If customer-facing → IC updates status page           |
| 15 min              | IC notifies #incident-comms channel                   |
| 30 min              | If not resolved → page engineering manager            |
| 60 min              | VP Engineering notified automatically                 |
| 4 hrs               | If still active → CTO notified, war room              |

## Post-Incident Checklist

- [ ] Timeline documented in incident channel (with commands run and results)
- [ ] Revenue impact calculated: `failed_charges × $85 avg × duration_minutes`
- [ ] PCI audit: all investigation commands logged (incident channel = audit trail)
- [ ] Postmortem scheduled within 48 hours (SEV1/2)
- [ ] Error budget impact calculated and updated in SLO dashboard
- [ ] This runbook updated if any steps were wrong, missing, or slow
- [ ] Action items created in Jira with owners and due dates
```

---

## RUNBOOK-DATA-007: Database Connection Pool Exhaustion

```markdown
# RUNBOOK-DATA-007: Database Connection Pool Exhaustion

## Metadata
- **Severity:** SEV2 (degrades to SEV1 if payment-service affected)
- **Services affected:** Any service using RDS PostgreSQL (payment, order, user)
- **SLO impact:** Availability SLO of affected service, latency SLO (requests queue waiting for connection)
- **Last tested:** 2024-10-20 (chaos drill: connection leak simulation)
- **Owner:** Platform Engineering
- **Prerequisite knowledge:** HikariCP (Java/Spring Boot), pgx pool (Go), SQLAlchemy (Python)

## Detection
- **Alert:** `HighConnectionPoolUtilization` (custom alert, fires at 80% pool usage)
- **Secondary signals:**
  - Latency SLI degradation (requests waiting for connections)
  - 504 timeouts from service
  - RDS `DatabaseConnections` CloudWatch metric approaching max_connections
- **Dashboard:** https://grafana.novamart.internal/d/data-tier?var-service=$affected_service

## Impact Assessment (30 seconds)

```bash
# 1. Which service is exhausted?
for ns in payment order user; do
  echo "=== $ns ==="
  # Spring Boot (Java):
  kubectl exec -n $ns deploy/${ns}-service -c ${ns}-service -- \
    curl -sf localhost:8080/actuator/metrics/hikaricp.connections 2>/dev/null | \
    jq '{active: .measurements[0].value, idle: .measurements[1].value, pending: .measurements[2].value, max: .measurements[3].value}' 2>/dev/null
  # Go (pgx):
  kubectl exec -n $ns deploy/${ns}-service -c ${ns}-service -- \
    curl -sf localhost:8080/debug/pool 2>/dev/null
done

# 2. Is RDS itself at limit?
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=novamart-payment-prod \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Maximum --output table
# Compare with max_connections (db.r6g.xlarge default: ~5000)

# 3. Is it getting worse?
# If pending connections > 0 and growing → active leak → urgent
```

## Quick Mitigation

### If single service exhausted but RDS is fine:
```bash
# Rolling restart clears leaked connections:
kubectl rollout restart deployment/${SERVICE}-service -n ${NAMESPACE}

# Monitor restart:
kubectl rollout status deployment/${SERVICE}-service -n ${NAMESPACE} --timeout=300s
```
**Expected:** Connection count drops to minimum after restart. If it climbs back quickly → it's a leak, not a spike.

### If RDS approaching max_connections:
```bash
# 1. Identify top connection consumers:
# Requires psql access (via bastion or kubectl exec to a debug pod):
kubectl run psql-debug --rm -it --image=postgres:15 -n platform-ops -- \
  psql "host=novamart-payment-prod.xxxxx.us-east-1.rds.amazonaws.com \
  port=5432 user=master dbname=payment sslmode=require" \
  -c "SELECT usename, client_addr, count(*) 
      FROM pg_stat_activity 
      GROUP BY usename, client_addr 
      ORDER BY count DESC 
      LIMIT 20;"

# 2. Kill idle connections from misbehaving client:
# (requires pg_terminate_backend permission)
kubectl run psql-debug --rm -it --image=postgres:15 -n platform-ops -- \
  psql "host=novamart-payment-prod.xxxxx.us-east-1.rds.amazonaws.com \
  port=5432 user=master dbname=payment sslmode=require" \
  -c "SELECT pg_terminate_backend(pid) 
      FROM pg_stat_activity 
      WHERE state = 'idle' 
      AND query_start < now() - interval '10 minutes'
      AND usename != 'rdsadmin';"

# 3. If still critical — emergency max_connections increase:
# Modify parameter group (takes effect without reboot for some params):
aws rds modify-db-parameter-group \
  --db-parameter-group-name novamart-payment-prod \
  --parameters "ParameterName=max_connections,ParameterValue=10000,ApplyMethod=immediate"
# WARNING: Increasing max_connections increases memory usage.
# db.r6g.xlarge with 10K connections can OOM. Only as emergency measure.
```

### If it's a connection leak (connections climb steadily after restart):
```bash
# 1. Identify the leak: check which queries are held open:
kubectl run psql-debug --rm -it --image=postgres:15 -n platform-ops -- \
  psql "host=novamart-payment-prod.xxxxx.us-east-1.rds.amazonaws.com \
  port=5432 user=master dbname=payment sslmode=require" \
  -c "SELECT pid, state, wait_event_type, wait_event, 
      now() - state_change AS idle_duration, 
      left(query, 100) as query_preview
      FROM pg_stat_activity 
      WHERE state = 'idle in transaction'
      ORDER BY idle_duration DESC 
      LIMIT 20;"
# "idle in transaction" = app opened transaction, did something, never committed/rolled back.
# This is the #1 connection leak pattern.

# 2. Short-term fix: set idle_in_transaction_session_timeout:
aws rds modify-db-parameter-group \
  --db-parameter-group-name novamart-payment-prod \
  --parameters "ParameterName=idle_in_transaction_session_timeout,ParameterValue=60000,ApplyMethod=immediate"
# Kills transactions idle > 60 seconds. May cause app errors but prevents pool exhaustion.

# 3. Long-term: File bug to development team with:
#    - Which query is leaking (from pg_stat_activity)
#    - Stack trace from app logs at the time
#    - Connection pool metrics graph showing the climb
```

## Diagnosis

### Root Cause Categories

| Pattern | Symptom | Root Cause |
|---------|---------|------------|
| Gradual climb, never returns | Pool active count increases linearly | Connection leak (missing close/commit) |
| Spike then plateau | Many connections suddenly, stays high | Slow query causing all threads to block |
| Spike correlating with deploy | Connection count jumps after deploy | New code bug, changed pool config |
| All services simultaneously | Multiple services exhaust pools | RDS issue (failover, slow queries, locks) |

### Deep Diagnosis:
```bash
# Check for lock contention (connections waiting on locks):
kubectl run psql-debug --rm -it --image=postgres:15 -n platform-ops -- \
  psql "host=novamart-payment-prod.xxxxx.us-east-1.rds.amazonaws.com \
  port=5432 user=master dbname=payment sslmode=require" \
  -c "SELECT blocked_locks.pid AS blocked_pid,
      blocked_activity.usename AS blocked_user,
      blocking_locks.pid AS blocking_pid,
      blocking_activity.usename AS blocking_user,
      blocked_activity.query AS blocked_statement
      FROM pg_catalog.pg_locks blocked_locks
      JOIN pg_catalog.pg_stat_activity blocked_activity 
        ON blocked_activity.pid = blocked_locks.pid
      JOIN pg_catalog.pg_locks blocking_locks 
        ON blocking_locks.locktype = blocked_locks.locktype
        AND blocking_locks.relation = blocked_locks.relation
        AND blocking_locks.pid != blocked_locks.pid
      JOIN pg_catalog.pg_stat_activity blocking_activity 
        ON blocking_activity.pid = blocking_locks.pid
      WHERE NOT blocked_locks.granted 
      LIMIT 20;"

# Check for long-running queries:
kubectl run psql-debug --rm -it --image=postgres:15 -n platform-ops -- \
  psql "host=novamart-payment-prod.xxxxx.us-east-1.rds.amazonaws.com \
  port=5432 user=master dbname=payment sslmode=require" \
  -c "SELECT pid, now() - query_start AS duration, state, left(query, 200)
      FROM pg_stat_activity 
      WHERE state != 'idle' 
      AND query_start < now() - interval '30 seconds'
      ORDER BY duration DESC LIMIT 10;"

# Kill specific blocking query if identified:
# SELECT pg_terminate_backend(<blocking_pid>);
```

## Fallback When Observability is Down
```bash
# If Prometheus/Grafana unavailable:
# 1. CloudWatch for RDS connections:
aws cloudwatch get-metric-data --cli-input-json '{
  "MetricDataQueries": [
    {
      "Id": "connections",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/RDS",
          "MetricName": "DatabaseConnections",
          "Dimensions": [{"Name": "DBInstanceIdentifier", "Value": "novamart-payment-prod"}]
        },
        "Period": 60,
        "Stat": "Maximum"
      }
    }
  ],
  "StartTime": "'$(date -u -d '30 min ago' +%Y-%m-%dT%H:%M:%S)'",
  "EndTime": "'$(date -u +%Y-%m-%dT%H:%M:%S)'"
}'

# 2. Direct pod health (doesn't need monitoring):
kubectl exec deploy/payment-service -n payment -c payment-service -- \
  curl -sf localhost:8080/actuator/health/db 2>/dev/null | jq '.'
# Returns: {"status":"UP"} or {"status":"DOWN","details":{"error":"..."}}
```

## Verification
```bash
# Pool usage back to normal:
kubectl exec deploy/${SERVICE}-service -n ${NAMESPACE} -c ${SERVICE}-service -- \
  curl -sf localhost:8080/actuator/metrics/hikaricp.connections.pending | jq '.measurements[0].value'
# Target: 0 pending connections

# RDS connections back to normal range:
# Normal: ~200-500 total across all services
# Alert threshold: 80% of max_connections

# No 504s in last 10 minutes:
kubectl logs -n ${NAMESPACE} -l app=${SERVICE}-service --since=10m | \
  jq 'select(.status_code == 504)' | wc -l
# Target: 0
```

## Escalation
| Time | Action |
|------|--------|
| 0 min | On-call investigates |
| 10 min | If pool not recoverable: page secondary |
| 15 min | If RDS at max_connections: page DBA (if available) or engineering manager |
| 30 min | If payment affected > 15 min: escalate to SEV1, page IC |

## Post-Incident
- [ ] Identify root cause (leak, slow query, config change)
- [ ] If leak: Jira ticket to service team with evidence (pg_stat_activity output, pool metrics)
- [ ] Review connection pool settings across all services:
      Java/HikariCP: maximumPoolSize, connectionTimeout, idleTimeout, maxLifetime
      Go/pgx: MaxConns, MaxConnIdleTime, MaxConnLifetime
      Python/SQLAlchemy: pool_size, max_overflow, pool_timeout, pool_recycle
- [ ] Consider PgBouncer if not already in use (connection pooler between apps and RDS)
- [ ] Update this runbook
```

---

## RUNBOOK-PLAT-001: Prometheus Down / Not Scraping

```markdown
# RUNBOOK-PLAT-001: Prometheus Down / Not Scraping

## Metadata
- **Severity:** SEV2 (escalate to SEV1 if coincident with another incident — you're flying blind)
- **Services affected:** ALL — monitoring is the foundation
- **SLO impact:** Observability internal SLO (99.95%), plus all SLO alerts stop firing
- **Last tested:** 2024-10-01 (deliberate Prometheus pod kill)
- **Owner:** Platform Engineering
- **CRITICAL NOTE:** When Prometheus is down, ALL burn rate alerts are silent.
  You have NO automated safety net. Treat this as urgent even if nothing else seems wrong.

## Detection
- **Primary:** Watchdog/DeadManSwitch stops arriving (external monitor alerts)
  - PagerDuty: "Dead Man's Switch — Prometheus heartbeat missing"
  - Healthchecks.io or equivalent external service triggers after 5 min of missing Watchdog
- **Secondary:** Grafana dashboards show "No Data"
- **Tertiary:** Manual check: `kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus`
- **CloudWatch fallback:** ALB health check metrics still flow even when Prometheus is down

## Impact Assessment
```bash
# 1. Is Prometheus pod running?
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
# Expected: 2 pods (HA replicas) in Running state

# 2. If pods running but not scraping:
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &
curl -sf http://localhost:9090/-/healthy
# Returns: "Prometheus Server is Healthy." or connection refused

# 3. Check scrape targets:
curl -sf http://localhost:9090/api/v1/targets | \
  jq '[.data.activeTargets[] | select(.health != "up")] | length'
# If > 0 targets down → partial scrape failure
# If connection refused → Prometheus itself is down
```

## Quick Mitigation

### Prometheus pods not running:
```bash
# 1. Check pod status:
kubectl describe pod -n monitoring -l app.kubernetes.io/name=prometheus | \
  grep -A 20 "Events:"

# Common causes:
# - OOMKilled → check memory limits
# - PVC full → check storage
# - Node pressure → check node status

# 2. If OOMKilled:
kubectl get pod -n monitoring -l app.kubernetes.io/name=prometheus \
  -o jsonpath='{.items[*].status.containerStatuses[*].lastState.terminated.reason}'
# If "OOMKilled":
# Emergency: increase memory limit
kubectl edit prometheus kube-prometheus-stack-prometheus -n monitoring
# Set resources.limits.memory to 2x current value
# This triggers a rolling restart with more memory

# 3. If PVC full (WAL or TSDB fills disk):
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -- \
  df -h /prometheus
# If > 95%:
#   Option A: Increase PVC size (if StorageClass allows expansion):
kubectl edit pvc prometheus-kube-prometheus-stack-prometheus-db-prometheus-kube-prometheus-stack-prometheus-0 \
  -n monitoring
#   Change spec.resources.requests.storage to larger value
#
#   Option B: Emergency — delete old blocks:
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -- \
  ls -la /prometheus/
#   Identify oldest block directories (ULID format)
#   DO NOT delete the wal/ directory
#   Delete oldest data blocks if desperate (data loss accepted):
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -- \
  rm -rf /prometheus/01HXXXXXXXXXXXXXXX
#   Then trigger compaction: send SIGHUP or restart pod

# 4. If node issue (Pending — no node to schedule on):
kubectl get events -n monitoring --field-selector reason=FailedScheduling
# If affinity/anti-affinity preventing scheduling:
# Check if enough nodes exist with correct labels
# Check PDB isn't blocking (should have PDB for HA)
kubectl get pdb -n monitoring
```

### Prometheus running but not scraping:
```bash
# 1. Check Prometheus config reload:
kubectl logs -n monitoring prometheus-kube-prometheus-stack-prometheus-0 \
  -c config-reloader --tail=50
# Look for: "config reload triggered" or errors

# 2. Check ServiceMonitor/PodMonitor discovery:
curl -sf http://localhost:9090/api/v1/targets | \
  jq '.data.activeTargets | length'
# If 0 → service discovery broken
# If > 0 but targets "down" → network issue

# 3. If service discovery broken:
# Verify Prometheus Operator is running:
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus-operator
# If operator is down → no CRD reconciliation → no target discovery

# 4. If targets visible but "down":
# Check if Prometheus can reach targets:
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -- \
  wget -q -O- http://kube-state-metrics.monitoring:8080/metrics | head -5
# If connection refused → NetworkPolicy blocking
# If timeout → DNS or network issue
# Check NetworkPolicies:
kubectl get networkpolicy -n monitoring
```

### Complete Prometheus failure — use fallback monitoring:
```bash
# CloudWatch alarms still function independently:
aws cloudwatch describe-alarms --state-value ALARM --output table

# ALB provides basic health without Prometheus:
aws elbv2 describe-target-health \
  --target-group-arn <payment-tg-arn> --output table

# Node health via kubectl (doesn't need Prometheus):
kubectl get nodes
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head -20

# Application health via readiness probes (kubectl shows this):
kubectl get pods -A | grep -v Running | grep -v Completed
```

## Diagnosis

### Prometheus OOM Investigation:
```bash
# What's consuming memory?
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -- \
  curl -sf localhost:9090/api/v1/status/tsdb | jq '.data'
# Key metrics:
#   headChunks — active time series count
#   numSeries — total series
# If numSeries > 2M → cardinality explosion

# Find high-cardinality metrics:
# Run query via API:
curl -sf "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=topk(20, count by (__name__)({__name__=~".+"}))' | \
  jq '.data.result[] | {metric: .metric.__name__, count: .value[1]}'
# If any metric has > 100K series → that's your problem
# Common culprits: metrics with pod, container, or instance labels that explode with scale
```

### Prometheus Disk Full Investigation:
```bash
# TSDB status:
kubectl exec -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -- \
  curl -sf localhost:9090/api/v1/status/tsdb | \
  jq '{headChunks: .data.headChunks, numSeries: .data.numSeries, 
       minTime: .data.minTime, maxTime: .data.maxTime}'

# Check retention settings:
kubectl get prometheus kube-prometheus-stack-prometheus -n monitoring \
  -o jsonpath='{.spec.retention}{"\n"}{.spec.retentionSize}{"\n"}'
# If retention too long for disk size → reduce retention or increase disk

# Check if Thanos sidecar is uploading (if blocks aren't uploaded, they accumulate):
kubectl logs -n monitoring prometheus-kube-prometheus-stack-prometheus-0 \
  -c thanos-sidecar --tail=50
# Look for upload errors to S3
```

## Verification
```bash
# 1. Prometheus healthy:
curl -sf http://localhost:9090/-/healthy
# "Prometheus Server is Healthy."

# 2. All expected targets up:
curl -sf http://localhost:9090/api/v1/targets | \
  jq '{total: (.data.activeTargets | length), 
       down: [.data.activeTargets[] | select(.health != "up")] | length}'
# Target: down = 0

# 3. Recording rules evaluating:
curl -sf http://localhost:9090/api/v1/rules | \
  jq '.data.groups[] | select(.rules[].health != "ok") | .name'
# Target: no output (all rules healthy)

# 4. Watchdog alert firing again:
curl -sf http://localhost:9090/api/v1/alerts | \
  jq '.data.alerts[] | select(.labels.alertname == "Watchdog")'
# Should exist and be firing

# 5. Grafana dashboards showing data again:
# Visual check: https://grafana.novamart.internal/d/slo-overview
```

## Escalation
| Time | Action |
|------|--------|
| 0 min | On-call begins investigation. Watchdog absence is the trigger. |
| 5 min | If Prometheus not recoverable quickly: announce in #platform channel |
| 10 min | If another incident is also active: SEV1 — you're flying blind |
| 15 min | Page secondary on-call if root cause unclear |
| 30 min | Engineering manager if not resolved |
| Note | **During Prometheus downtime, increase manual monitoring:** check kubectl, CloudWatch, ALB every 5 min |

## Post-Incident
- [ ] Root cause identified (OOM, disk, network, operator)
- [ ] If OOM: review cardinality, add metricRelabelings to drop high-cardinality metrics
- [ ] If disk: increase PVC size, adjust retention, verify Thanos uploading
- [ ] Review alert gap: were any incidents missed during Prometheus downtime?
- [ ] Verify Watchdog/DeadManSwitch actually alerted on the Prometheus failure
- [ ] Consider: should we run Prometheus in a separate failure domain from the workloads it monitors?
```

---

## RUNBOOK-INFRA-004: VPC CNI IP Exhaustion

```markdown
# RUNBOOK-INFRA-004: VPC CNI IP Exhaustion (Pods Stuck Pending)

## Metadata
- **Severity:** SEV2 (SEV1 if new pods can't schedule for critical services)
- **Services affected:** Any service attempting to scale or redeploy
- **SLO impact:** Availability SLO of any service that can't scale to handle load
- **Last tested:** 2024-09-15 (simulated by reducing subnet CIDR in test cluster)
- **Owner:** Platform Engineering

## Detection
- **Alert:** `AWSVPCCNI_IPAddressesExhausted` or `KubePodNotScheduled`
- **Symptom:** Pods stuck in `ContainerCreating` with event: 
  `"failed to assign an IP address to container"`
- **Dashboard:** https://grafana.novamart.internal/d/eks-network
- **CloudWatch:** `node_ip_address_available` metric from VPC CNI

## Impact Assessment
```bash
# 1. How many pods are stuck?
kubectl get pods -A --field-selector=status.phase=Pending --no-headers | wc -l
kubectl get events -A --field-selector reason=FailedCreatePodSandBox | tail -20

# 2. Which subnets are exhausted?
# With custom networking (secondary CIDR 100.64.0.0/16), check pod subnets:
for subnet_id in $(aws ec2 describe-subnets \
  --filters "Name=tag:kubernetes.io/role/cni,Values=1" \
  --query 'Subnets[*].SubnetId' --output text); do
  AVAIL=$(aws ec2 describe-subnets --subnet-ids $subnet_id \
    --query 'Subnets[0].AvailableIpAddressCount' --output text)
  CIDR=$(aws ec2 describe-subnets --subnet-ids $subnet_id \
    --query 'Subnets[0].CidrBlock' --output text)
  AZ=$(aws ec2 describe-subnets --subnet-ids $subnet_id \
    --query 'Subnets[0].AvailabilityZone' --output text)
  echo "Subnet: $subnet_id | AZ: $AZ | CIDR: $CIDR | Available IPs: $AVAIL"
done

# 3. Check VPC CNI metrics on nodes:
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,PODS:.status.allocatable.pods,IP_USED:.metadata.annotations.vpc\.amazonaws\.com/pod-eni-count'

# 4. Check if prefix delegation is enabled:
kubectl get daemonset aws-node -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].env}' | jq '.[] | select(.name == "ENABLE_PREFIX_DELEGATION")'
```

## Quick Mitigation

### Option 1: Scale down non-critical workloads to free IPs
```bash
# Identify largest consumers:
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.status.phase == "Running") | .metadata.namespace' | \
  sort | uniq -c | sort -rn | head -10

# Scale down non-critical:
kubectl scale deployment -n batch-jobs --all --replicas=0
kubectl scale deployment -n staging --all --replicas=0
# This frees IPs immediately as pods terminate
```

### Option 2: Enable prefix delegation (if not already enabled)
```bash
# This is the long-term fix — assigns /28 prefixes (16 IPs) instead of individual IPs
# SIGNIFICANTLY increases IP capacity per node

# Check current setting:
kubectl get daemonset aws-node -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].env}' | \
  jq '.[] | select(.name == "ENABLE_PREFIX_DELEGATION")'

# If not enabled:
kubectl set env daemonset aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true \
  WARM_PREFIX_TARGET=1

# WARNING: This requires node replacement for full effect.
# New pods on existing nodes will use prefix delegation.
# For maximum benefit, cordon and drain nodes one at a time:
# kubectl cordon <node> && kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# Karpenter will provision new nodes with prefix delegation active.
```

### Option 3: Emergency — add new subnets (Terraform change)
```bash
# If all pod subnets are exhausted and prefix delegation isn't enough:
# This requires Terraform change to add new subnets to the secondary CIDR
# 
# 1. Terraform plan:
cd terraform/environments/production/networking
# Edit to add new /19 subnets in each AZ from 100.64.0.0/16 range
terraform plan -out=emergency-subnet.plan
terraform apply emergency-subnet.plan

# 2. Create ENIConfig for new subnets:
# (ENIConfig tells VPC CNI which subnet to use in each AZ)
kubectl apply -f - <<EOF
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1a-expanded
spec:
  subnet: <new-subnet-id-1a>
  securityGroups:
    - <pod-security-group-id>
EOF
# Repeat for each AZ

# 3. Label nodes to use new ENIConfig:
kubectl label nodes -l topology.kubernetes.io/zone=us-east-1a \
  k8s.amazonaws.com/eniConfig=us-east-1a-expanded --overwrite
```

## Diagnosis

### Why did we run out of IPs?
```bash
# 1. Traffic spike → HPA scaled too many pods?
kubectl get hpa -A | awk '$5 > $6 {print}'
# Shows HPAs where current > desired (still trying to scale)

# 2. Pod churn (CrashLoop leaving orphaned ENIs)?
kubectl get pods -A --field-selector=status.phase=Failed --no-headers | wc -l
aws ec2 describe-network-interfaces \
  --filters "Name=tag:node.k8s.amazonaws.com/instance_id,Values=*" \
            "Name=status,Values=available" \
  --query 'NetworkInterfaces[*].{ID:NetworkInterfaceId,Status:Status,AZ:AvailabilityZone}' \
  --output table
# "available" ENIs that aren't attached = orphaned = wasted IPs

# 3. WARM_ENI_TARGET too high? (pre-allocates IPs per node)
kubectl get daemonset aws-node -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].env}' | \
  jq '.[] | select(.name | startswith("WARM_"))'
# If WARM_ENI_TARGET=1 (default), each node pre-allocates an entire ENI worth of IPs
# Consider MINIMUM_IP_TARGET instead for finer control

# 4. Subnet CIDR too small for cluster size?
# Math: /19 subnet = 8190 IPs. With 3 AZs = 24570 total pod IPs.
# If you have 200+ microservices × avg 5 pods × 3 envs = 3000 pods minimum
# Plus warm IPs, plus headroom = can hit limits at scale
```

## Fallback When Observability is Down
```bash
# VPC CNI issues are infrastructure-level — AWS CLI always works:
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Available:AvailableIpAddressCount}' \
  --output table

# kubectl works even when monitoring is down:
kubectl describe node <node-name> | grep -A 5 "Allocated resources"
kubectl get pods -A -o wide | wc -l
```

## Verification
```bash
# Pending pods scheduling:
kubectl get pods -A --field-selector=status.phase=Pending --no-headers | wc -l
# Target: 0 (or only intentionally pending)

# IP availability:
for subnet_id in $(aws ec2 describe-subnets \
  --filters "Name=tag:kubernetes.io/role/cni,Values=1" \
  --query 'Subnets[*].SubnetId' --output text); do
  AVAIL=$(aws ec2 describe-subnets --subnet-ids $subnet_id \
    --query 'Subnets[0].AvailableIpAddressCount' --output text)
  echo "Subnet $subnet_id: $AVAIL available IPs"
done
# Target: > 20% available in each subnet

# VPC CNI healthy on all nodes:
kubectl get pods -n kube-system -l k8s-app=aws-node -o wide
# All Running, 0 restarts
```

## Escalation
| Time | Action |
|------|--------|
| 0 min | On-call investigates pending pods |
| 10 min | If critical service can't scale: SEV2 |
| 15 min | If payment/order affected: SEV1 |
| 30 min | If subnet expansion needed: page infrastructure lead (Terraform change) |

## Post-Incident
- [ ] Enable prefix delegation if not already active
- [ ] Review WARM_ENI_TARGET / MINIMUM_IP_TARGET settings
- [ ] Add CloudWatch alarm: available IPs < 20% per subnet
- [ ] Review subnet CIDR planning — is the secondary CIDR large enough for 2x growth?
- [ ] Consider: Karpenter consolidation settings — are nodes too large (wasting IPs)?
- [ ] Clean up orphaned ENIs if found
```

---

## RUNBOOK-APP-010: Cross-Service Cascading Failure

```markdown
# RUNBOOK-APP-010: Cross-Service Cascading Failure

## Metadata
- **Severity:** SEV1 (by definition — multiple services failing)
- **Services affected:** Multiple (cascade pattern)
- **SLO impact:** Multiple SLOs simultaneously
- **Last tested:** 2024-11-01 (chaos game day: injected latency into user-service, observed cascade)
- **Owner:** Platform Engineering
- **CRITICAL:** This is the hardest incident type. It requires discipline to NOT chase symptoms.

## Detection
- **Pattern:** Multiple burn rate alerts firing within minutes of each other
- **Telltale:** Service A fires, then B, then C — the ORDER tells you the root cause
- **Dashboard:** https://grafana.novamart.internal/d/slo-overview (all services view)
- **PagerDuty:** Multiple incidents created in rapid succession

## Impact Assessment (CRITICAL — 60 seconds)

```bash
# 1. MAP THE FAILURE — which services are down, in what order?
# Check alert firing times:
curl -sf http://localhost:9090/api/v1/alerts | \
  jq '.data.alerts[] | select(.state == "firing") | 
  {alert: .labels.alertname, service: .labels.slo_service, 
   started: .activeAt}' | sort

# 2. Service dependency map (know this by heart):
#
#   Cloudflare → ALB → API Gateway
#                         ├─→ User Service ──→ User DB (RDS)
#                         ├─→ Cart Service ──→ Redis + Cart DB
#                         ├─→ Order Service ──→ Order DB + RabbitMQ
#                         │     └─→ Payment Service ──→ Payment DB + Stripe/Adyen
#                         ├─→ Search Service ──→ Elasticsearch
#                         └─→ Notification Service ──→ SES + RabbitMQ
#
# RULE: The ROOT CAUSE is usually the FIRST service to fail
# or a shared dependency (RDS, Redis, DNS, mesh)

# 3. Check shared dependencies FIRST:
echo "=== CoreDNS ==="
kubectl get pods -n kube-system -l k8s-app=kube-dns
echo "=== Linkerd ==="
kubectl get pods -n linkerd
echo "=== RDS ==="
aws rds describe-db-instances --query 'DBInstances[*].{ID:DBInstanceIdentifier,Status:DBInstanceStatus}' --output table
echo "=== Redis ==="
aws elasticache describe-replication-groups --query 'ReplicationGroups[*].{ID:ReplicationGroupId,Status:Status}' --output table
echo "=== NAT Gateway ==="
aws ec2 describe-nat-gateways --filter "Name=state,Values=available" --query 'NatGateways[*].{ID:NatGatewayId,State:State,AZ:SubnetId}' --output table
```

## Quick Mitigation — The Cascade Breaker Protocol

The goal is to **stop the cascade from spreading**, then fix the root cause.

### Step 1: Identify and isolate the root cause service
```bash
# If one service is causing the cascade by being slow (not down):
# The slow service causes callers to hold connections → they exhaust → they fail → their callers fail

# Check which service responded FIRST in the alert timeline.
# Check Linkerd metrics for retry amplification:
kubectl exec -n linkerd deploy/linkerd-viz -- \
  curl -sf http://localhost:8084/api/v1/query \
  --data-urlencode 'query=topk(5, sum by (deployment) (rate(response_total{classification="failure"}[5m])))' | jq '.'
```

### Step 2: Apply circuit breakers
```bash
# OPTION A: Reduce retry budget for all services calling the bad one:
# This stops retry amplification — the #1 cause of cascades
kubectl annotate service user-service -n user \
  config.linkerd.io/proxy-retry-budget="0%" --overwrite

# OPTION B: If the root cause service is beyond saving, scale it to 0
# and let callers hit their timeout/fallback path:
kubectl scale deployment user-service -n user --replicas=0
# WARNING: This breaks all features depending on user-service.
# But it STOPS the cascade to other services.
# Only do this if the alternative is ALL services down.
```

### Step 3: Protect critical services
```bash
# Scale UP payment service to handle the load without the cascade:
kubectl scale deployment payment-service -n payment --replicas=10
# (HPA should do this, but during a cascade HPA may be fighting evictions)

# Scale UP API gateway:
kubectl scale deployment api-gateway -n api-gateway --replicas=10
```

### Step 4: Shed non-critical load
```bash
# Disable non-essential services:
kubectl scale deployment notification-service -n notification --replicas=0
kubectl scale deployment search-service -n search --replicas=0
# Users can't search or get notifications, but they CAN buy things.
```

## Diagnosis — Finding the Real Root Cause

### Pattern 1: Slow dependency cascade
```
Symptom: Service A latency spikes → Service B (calls A) exhausts connection pool
         → Service C (calls B) times out → everything fails
Root cause: Service A database is slow (lock contention, bad query, disk full)
Fix: Fix Service A's database → everything recovers
```

### Pattern 2: Retry amplification
```
Symptom: Service A returns 503 → Linkerd retries 3x → 3x load on A
         → A gets more overloaded → more 503s → more retries → exponential
Root cause: Service A had a minor issue, retries made it catastrophic
Fix: Kill retry budgets → let A recover → re-enable retries
```

### Pattern 3: Shared infrastructure failure
```
Symptom: All services fail simultaneously (not in sequence)
Root cause: CoreDNS down, Linkerd down, RDS failover, NAT Gateway failure
Fix: Fix the shared infrastructure → everything recovers
```

### Pattern 4: Resource pressure cascade
```
Symptom: Nodes under memory pressure → evictions → pods reschedule
         → more pressure on remaining nodes → more evictions → cascade
Root cause: A memory leak in one service causes node pressure
Fix: Kill the leaking service → nodes recover → pods reschedule
```

### Investigation commands:
```bash
# Distributed trace — find the bottleneck:
# Get a trace ID from a failed request:
kubectl logs -n api-gateway -l app=api-gateway --since=5m | \
  jq 'select(.status_code >= 500) | .trace_id' | head -5

# Open in Tempo — the trace shows which service is the bottleneck:
# https://grafana.novamart.internal/explore?datasource=Tempo&query=<traceID>
# Look for: which span is the longest? Which span has errors?
# The deepest failing span is usually the root cause.

# If tracing is also broken (common during cascades):
# Fall back to per-service latency comparison:
for svc in api-gateway user-service cart-service order-service payment-service search-service; do
  echo "=== $svc ==="
  curl -sf "http://prometheus.monitoring:9090/api/v1/query" \
    --data-urlencode "query=histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service=\"$svc\"}[5m])))" | \
    jq -r ".data.result[0].value[1]"
done
# The service with the highest p99 is likely the bottleneck

# Check node-level pressure:
kubectl top nodes --sort-by=memory
kubectl get nodes -o custom-columns='NAME:.metadata.name,CONDITIONS:.status.conditions[?(@.type=="MemoryPressure")].status'
# If any node shows MemoryPressure=True → resource cascade

# Check for evictions:
kubectl get events -A --field-selector reason=Evicted --sort-by='.lastTimestamp' | tail -20
```

## Fallback When Observability is Down

Cascading failures often take out monitoring too. Here's your blind-flying protocol:

```bash
# 1. AWS-native health (always available):
# ALB target group health — tells you which services are actually responding:
for tg_arn in $(aws elbv2 describe-target-groups \
  --query 'TargetGroups[*].TargetGroupArn' --output text); do
  TG_NAME=$(aws elbv2 describe-target-groups --target-group-arns $tg_arn \
    --query 'TargetGroups[0].TargetGroupName' --output text)
  HEALTHY=$(aws elbv2 describe-target-health --target-group-arn $tg_arn \
    --query 'TargetHealthDescriptions[?TargetHealth.State==`healthy`] | length(@)' --output text)
  UNHEALTHY=$(aws elbv2 describe-target-health --target-group-arn $tg_arn \
    --query 'TargetHealthDescriptions[?TargetHealth.State!=`healthy`] | length(@)' --output text)
  echo "$TG_NAME: $HEALTHY healthy, $UNHEALTHY unhealthy"
done

# 2. Direct pod status (kubectl works without Prometheus):
kubectl get pods -A -o wide | grep -v Running | grep -v Completed

# 3. Node status:
kubectl get nodes
kubectl describe nodes | grep -A 5 "Conditions:"

# 4. CloudWatch emergency dashboard:
# https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:name=NovaMart-Emergency
# This dashboard uses CloudWatch native metrics only — works even if EKS monitoring is down
```

## Recovery Sequence

Once root cause is identified and fixed, bring services back in dependency order:

```bash
# Phase 1: Shared infrastructure (must be healthy first)
# Verify: CoreDNS, Linkerd, RDS, Redis, NAT Gateway

# Phase 2: Backend services (no external callers)
kubectl scale deployment user-service -n user --replicas=3
kubectl rollout status deployment user-service -n user --timeout=120s
# Wait for healthy. Verify:
kubectl exec -n user deploy/user-service -c user-service -- curl -sf localhost:8080/health

# Phase 3: Mid-tier services
kubectl scale deployment order-service -n order --replicas=3
kubectl scale deployment cart-service -n cart --replicas=3
kubectl scale deployment payment-service -n payment --replicas=5  # Extra replicas for backlog
# Wait for all healthy.

# Phase 4: Frontend-facing services
kubectl scale deployment api-gateway -n api-gateway --replicas=5
# Wait. Monitor error rates.

# Phase 5: Non-critical services (last)
kubectl scale deployment search-service -n search --replicas=3
kubectl scale deployment notification-service -n notification --replicas=3

# Phase 6: Re-enable retries (if disabled):
kubectl annotate service user-service -n user \
  config.linkerd.io/proxy-retry-budget- --overwrite
# (Removing the annotation restores default retry budget)

# Phase 7: Restore HPA targets to normal
# HPA should auto-scale back. Verify:
kubectl get hpa -A

# Phase 8: Monitor for 30 minutes. Watch for re-cascade.
```

## Verification
```bash
# All services healthy:
kubectl get pods -A | grep -v Running | grep -v Completed | wc -l
# Target: 0

# All SLO burn rates back to normal:
curl -sf "http://prometheus.monitoring:9090/api/v1/query" \
  --data-urlencode 'query=sli:availability:error_ratio:rate5m' | \
  jq '.data.result[] | {service: .metric.slo_service, error_rate: .value[1]}'
# Target: all < their budget rate

# No pending pods:
kubectl get pods -A --field-selector=status.phase=Pending --no-headers | wc -l
# Target: 0

# No node pressure:
kubectl get nodes -o custom-columns='NAME:.metadata.name,MEMORY_PRESSURE:.status.conditions[?(@.type=="MemoryPressure")].status,DISK_PRESSURE:.status.conditions[?(@.type=="DiskPressure")].status'
# Target: all False
```

## Escalation
| Time | Action |
|------|--------|
| 0 min | On-call declares SEV1 immediately (multi-service = auto SEV1) |
| 0 min | Open #incident-YYYYMMDD, page Incident Commander |
| 5 min | IC assigns roles: Tech Lead (debugging), Comms Lead, Scribe |
| 10 min | If root cause not identified: page additional engineers |
| 15 min | Status page updated: "We are investigating service degradation" |
| 15 min | Customer comms: "Some users may experience errors" |
| 30 min | If not mitigated: engineering manager + VP Engineering |
| 60 min | If not resolved: CTO notified, full war room |

## Post-Incident
- [ ] Full timeline with all actions and results
- [ ] Root cause: which service started the cascade and why
- [ ] Revenue impact calculation
- [ ] Retry budget review: are defaults too aggressive?
- [ ] Circuit breaker review: do we need explicit circuit breakers (not just Linkerd)?
- [ ] Timeout review: are timeouts properly configured to prevent cascade?
  - Rule: downstream timeout < upstream timeout (always)
  - API GW timeout > Order Service timeout > Payment Service timeout
- [ ] Load shedding review: do services have graceful degradation paths?
- [ ] Postmortem within 24 hours (SEV1 = expedited)
- [ ] Chaos experiment: can we reproduce this cascade in a controlled way?
```

---

## RUNBOOK-SEC-004: Secret Rotation Emergency

```markdown
# RUNBOOK-SEC-004: Secret Rotation Emergency

## Metadata
- **Severity:** SEV1 (security incident — potential data breach)
- **Services affected:** Depends on which secret is compromised
- **SLO impact:** Availability impact during rotation; security SLA always overrides
- **Last tested:** 2024-10-15 (planned rotation drill — Stripe API key)
- **Owner:** Platform Engineering + Security Team
- **PCI scope:** YES — all actions logged, all timestamps recorded
- **LEGAL NOTE:** If credentials may have been exfiltrated, notify Security Lead
  and Legal immediately. Data breach notification laws have timelines (72h GDPR).

## Detection
- **Alert:** GuardDuty finding, Vault audit log anomaly, engineer reports accidental exposure
- **Common triggers:**
  1. Secret committed to Git repository
  2. Secret found in container logs
  3. Secret exposed in error message to client
  4. AWS Access Key detected by GitHub secret scanning
  5. CloudTrail shows unauthorized use of service account credentials
  6. PCI audit discovers plaintext secret in unauthorized location

## Impact Assessment (IMMEDIATE — 60 seconds)

```bash
# 1. WHAT was exposed? (determines blast radius)
# Categories:
#   A. AWS IAM credentials (access key + secret key)
#   B. Database password (RDS, Redis)
#   C. Third-party API key (Stripe, Adyen, SendGrid)
#   D. TLS private key
#   E. Kubernetes ServiceAccount token
#   F. Application secret (JWT signing key, encryption key)

# 2. WHERE was it exposed?
#   - Public Git repo → ASSUME COMPROMISED
#   - Private Git repo → less risk but still rotate
#   - Container logs → check who has log access
#   - Error response to client → ASSUME COMPROMISED

# 3. WHEN was it exposed?
#   - Determines: how long has an attacker had access?
#   - Check: git log, CloudTrail, audit logs

# 4. WHAT CAN IT ACCESS?
#   - IAM creds: what policies are attached?
#   - DB password: what data is in that database? PCI scope?
#   - API key: what can it do? (charge cards? read PII?)
```

## Response Procedure

### RULE #1: ROTATE FIRST, INVESTIGATE SECOND.
### Do NOT spend 30 minutes investigating while the credential is live.

### Category A: AWS IAM Credentials
```bash
# 1. IMMEDIATELY disable the access key:
aws iam update-access-key \
  --user-name <username> \
  --access-key-id <exposed-key-id> \
  --status Inactive
# This is INSTANT. No service restart needed for disabling.
# Any service using this key will start failing — that's acceptable.

# 2. Check what the key was used for (damage assessment):
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<exposed-key-id> \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --max-results 50 | jq '.Events[] | {time: .EventTime, name: .EventName, source: .EventSource}'
# Look for: events you DON'T recognize, events from unusual IPs

# 3. Create new access key (if the service still needs one):
aws iam create-access-key --user-name <username>
# BETTER: migrate to IRSA (no long-lived credentials)

# 4. Update the secret in Secrets Manager:
aws secretsmanager update-secret \
  --secret-id novamart/<service>/aws-credentials \
  --secret-string '{"access_key":"<new-key>","secret_key":"<new-secret>"}'

# 5. Trigger External Secrets Operator refresh:
kubectl annotate externalsecret <name> -n <namespace> \
  force-sync=$(date +%s) --overwrite
# Or wait for sync interval (default: 1h — too slow for emergency)

# 6. Restart affected pods to pick up new secret:
kubectl rollout restart deployment/<service> -n <namespace>

# 7. Delete the old access key:
aws iam delete-access-key \
  --user-name <username> \
  --access-key-id <exposed-key-id>
```

### Category B: Database Password
```bash
# 1. If using managed master password (RDS manages it):
# Rotate via RDS:
aws rds modify-db-instance \
  --db-instance-identifier novamart-payment-prod \
  --master-user-password "<new-secure-password-from-pwgen>"
# WARNING: This immediately changes the password.
# All services using the old password will lose DB connectivity.
# Coordinate with the next steps.

# BETTER (if manage_master_user_password is true — as we configured):
aws rds modify-db-instance \
  --db-instance-identifier novamart-payment-prod \
  --rotate-master-user-password
# RDS rotates the password in Secrets Manager automatically.

# 2. Update application secrets:
# If using External Secrets Operator → it pulls from Secrets Manager → auto-updates
# Force immediate sync:
kubectl annotate externalsecret payment-db-credentials -n payment \
  force-sync=$(date +%s) --overwrite

# 3. Rolling restart of affected services:
kubectl rollout restart deployment/payment-service -n payment
kubectl rollout status deployment/payment-service -n payment --timeout=300s
# Rolling restart means at least some pods always running with OLD password
# until the new ones come up with the NEW password.
# This minimizes (but doesn't eliminate) downtime.

# 4. Verify connectivity:
kubectl exec deploy/payment-service -n payment -c payment-service -- \
  curl -sf localhost:8080/actuator/health/db | jq '.'
# Should show: {"status":"UP"}

# 5. Change application-level DB users (not just master):
# Application should use its own DB user, not master.
# Rotate that too:
# psql -c "ALTER USER payment_app PASSWORD '<new-password>';"
# Update in Secrets Manager → ESO → pod restart
```

### Category C: Third-Party API Key (Stripe, Adyen)
```bash
# 1. Go to provider dashboard IMMEDIATELY and rotate the key:
#    Stripe: https://dashboard.stripe.com/apikeys → Roll key
#    Adyen: https://ca-live.adyen.com/ca/ca/config/api_credentials_new.shtml
#
# Most providers support "rolling" where old key works for a grace period.
# Stripe: old key works for 24h after roll. USE THIS WINDOW.

# 2. Update Secrets Manager:
aws secretsmanager update-secret \
  --secret-id novamart/payment/stripe-api-key \
  --secret-string '{"api_key":"sk_live_new_xxxxxxxxxxxx"}'

# 3. Force External Secrets sync:
kubectl annotate externalsecret stripe-credentials -n payment \
  force-sync=$(date +%s) --overwrite

# 4. Rolling restart:
kubectl rollout restart deployment/payment-service -n payment
kubectl rollout status deployment/payment-service -n payment --timeout=300s

# 5. Verify payments working:
# Watch SLO dashboard for payment success rate
# Run a test charge (if test mode available):
curl -sf https://api.novamart.internal/payments/health

# 6. After new key confirmed working, revoke old key at provider dashboard.
```

### Category D: TLS Private Key
```bash
# 1. Revoke the certificate at the issuing CA:
# If ACM-managed: ACM doesn't expose private keys — unlikely to be compromised
# If cert-manager/Let's Encrypt:

# Delete the certificate — cert-manager will issue a new one:
kubectl delete certificate <cert-name> -n <namespace>
# cert-manager will detect the missing cert and re-issue via ACME

# 2. If the cert was for Linkerd mTLS:
# This is more complex — see RUNBOOK-PLAT-006
# Short version: rotate trust anchor and issuer certificates

# 3. Verify new cert issued:
kubectl get certificate -n <namespace>
# Status should show Ready=True

# 4. If external-facing cert (ALB/Cloudflare):
# Re-issue in ACM or upload new cert
# Update ALB listener / Cloudflare edge cert
```

### Category E: Kubernetes ServiceAccount Token
```bash
# 1. If it's a projected token (default in K8s 1.24+):
# These auto-expire. Check expiry:
# Usually 1 hour. Risk is limited.

# 2. If it's a legacy long-lived token (Secret-based):
# Delete the secret:
kubectl delete secret <sa-token-secret> -n <namespace>
# New token will be auto-created

# 3. Recreate pods using the service account:
kubectl rollout restart deployment/<deployment> -n <namespace>

# 4. Review: are there any long-lived SA tokens that shouldn't exist?
kubectl get secrets -A -o json | \
  jq '.items[] | select(.type == "kubernetes.io/service-account-token") | 
  {namespace: .metadata.namespace, name: .metadata.name, sa: .metadata.annotations["kubernetes.io/service-account.name"]}'
```

### Category F: Application Secret (JWT key, encryption key)
```bash
# 1. Generate new key:
# JWT signing key:
openssl genrsa -out new-jwt-key.pem 4096
# Symmetric encryption key:
openssl rand -base64 32 > new-encryption-key.txt

# 2. DUAL-WRITE: Applications must accept BOTH old and new key
# during rotation. This is an APPLICATION-LEVEL concern.
# Typical pattern:
#   - Add new key as "primary" for signing/encrypting
#   - Keep old key as "secondary" for verifying/decrypting
#   - After all sessions/tokens expire: remove old key

# 3. Update Secrets Manager with both keys:
aws secretsmanager update-secret \
  --secret-id novamart/auth/jwt-signing-key \
  --secret-string '{"primary":"<new-key>","secondary":"<old-key>"}'

# 4. Rolling restart:
kubectl rollout restart deployment/user-service -n user

# 5. Monitor for auth failures:
kubectl logs -n user -l app=user-service --since=5m | \
  jq 'select(.level == "ERROR" and .message | contains("token"))' | head -20

# 6. After grace period (e.g., max token lifetime + buffer):
# Remove old key from Secrets Manager
# Restart again to pick up single-key config
```

## Git Secret Exposure — Special Procedure
```bash
# If secret was committed to Git:

# 1. ROTATE THE SECRET IMMEDIATELY (steps above)
# Do NOT waste time cleaning Git first. Assume it's already scraped.

# 2. Remove from Git history:
# Using git-filter-repo (preferred over BFG):
git filter-repo --invert-paths --path <file-with-secret>
# Or replace specific string:
git filter-repo --replace-text <(echo '<secret>==>REDACTED')

# 3. Force push to all branches:
git push --force --all
git push --force --tags

# 4. Invalidate all caches:
# Bitbucket: admin → repository settings → clear cache
# Any CI that cached the repo

# 5. Notify all developers to re-clone (force push broke their history)
# Message in #engineering: "SECURITY: Please delete and re-clone <repo>.
# Previous history has been rewritten to remove an exposed credential."

# 6. Add to .gitignore:
echo "<pattern>" >> .gitignore
git add .gitignore && git commit -m "chore: add <pattern> to gitignore"

# 7. Add pre-commit hook to prevent recurrence:
# In .pre-commit-config.yaml:
# - repo: https://github.com/gitleaks/gitleaks
#   rev: v8.18.0
#   hooks:
#     - id: gitleaks
```

## Fallback When Observability is Down
```bash
# Secret rotation is infrastructure-level — these tools always work:
# AWS CLI for Secrets Manager / IAM
# kubectl for pod restarts
# Provider dashboards for API key rotation
# Git CLI for history rewrite

# The risk when observability is down:
# You can't verify the service is healthy after rotation.
# Mitigation: Use direct health endpoints:
kubectl exec deploy/<service> -n <namespace> -c <container> -- \
  curl -sf localhost:8080/health
```

## Verification
```bash
# 1. Old credential no longer works:
# (Test carefully — don't accidentally use the old cred in production)
# For AWS: aws sts get-caller-identity with old key should fail
# For DB: connection with old password should fail
# For API: test call with old key should return 401

# 2. New credential works:
# Service health:
kubectl exec deploy/<service> -n <namespace> -c <container> -- \
  curl -sf localhost:8080/health | jq '.'

# 3. No auth errors in logs:
kubectl logs -n <namespace> -l app=<service> --since=10m | \
  jq 'select(.level == "ERROR")' | grep -i "auth\|credential\|password\|token\|key" | wc -l
# Target: 0

# 4. SLO not impacted (or recovering):
# Check dashboard: error rate returning to baseline
```

## Escalation
| Time | Action |
|------|--------|
| 0 min | Immediately rotate the credential. Don't wait. |
| 0 min | Notify Security Lead (#security-incidents channel) |
| 5 min | If PCI data potentially exposed: notify PCI compliance officer |
| 15 min | If customer data potentially exposed: notify Legal |
| 30 min | Security Lead determines if external notification required |
| 72 hrs | GDPR deadline: if EU personal data exposed, notify supervisory authority |

## Post-Incident
- [ ] Credential rotated and old credential destroyed
- [ ] Damage assessment: was the credential used by an unauthorized party?
- [ ] If Git exposure: history rewritten, pre-commit hook added
- [ ] Root cause: HOW did the secret get exposed?
  - Developer copied to wrong file?
  - CI pipeline logged it?
  - Error message included it?
  - Insufficient secret management tooling?
- [ ] Prevention: what process/tooling change prevents this class of incident?
  - gitleaks pre-commit hook enforced
  - Log scrubbing for known secret patterns
  - Vault dynamic secrets (short-lived, auto-expire)
- [ ] PCI incident report filed (if PCI scope)
- [ ] Security incident report filed
- [ ] Postmortem (blameless — focus on tooling/process, not the person)
```

---

# DELIVERABLE 3: Incident Management

## 3.1 Severity Definitions

```markdown
# docs/operations/severity-definitions.md

# NovaMart Incident Severity Definitions

## Severity Matrix

| SEV | Customer Impact | Revenue Impact | Data Impact | Examples | Response |
|-----|----------------|----------------|-------------|----------|----------|
| SEV1 | Total/major outage visible to most users | Active revenue loss ($3,800/min) | Data loss or breach possible | Payment down, site unreachable, data breach, multi-service cascade | 5min ack, 15min war room, status page, exec comms |
| SEV2 | Major feature degraded or regional outage | Partial revenue impact | No data loss | Search down, checkout slow >5s, one region down, single critical service down | 15min ack, 30min war room, status page if >30min |
| SEV3 | Minor feature degraded, workaround exists | Minimal/no revenue impact | No data loss | Notifications delayed, non-critical API slow, single non-critical pod crash | 30min ack, 4hr response, team channel |
| SEV4 | Cosmetic or internal tooling issue | No revenue impact | No data loss | Dashboard slow, log noise, CI flaky, single user report | 1hr ack, next sprint, Jira ticket |

## Auto-Escalation Rules

| Condition | Action |
|-----------|--------|
| SEV2 not mitigated in 30 min | Auto-escalate to SEV1 |
| Multiple SEV2/3 for same root cause | Declare SEV1 |
| Payment service involved at any severity | Minimum SEV2 |
| Data breach suspected at any severity | Immediate SEV1 |
| >3 services affected simultaneously | Automatic SEV1 |

## Who Can Declare/Change Severity

| Action | Who |
|--------|-----|
| Declare SEV3/4 | Any engineer |
| Declare SEV2 | Any on-call engineer |
| Declare SEV1 | On-call engineer, IC, engineering manager |
| Downgrade severity | Incident Commander only |
| Close incident | IC (SEV1/2), on-call (SEV3/4) |
```

## 3.2 Incident Response Process

```markdown
# docs/operations/incident-response-process.md

# NovaMart Incident Response Process

## Lifecycle

```
DETECT → TRIAGE → MITIGATE → DIAGNOSE → RESOLVE → LEARN
         (5min)   (15min)    (parallel)  (verify)   (48hr)
```

## Phase 1: Detection
- **Automated:** PagerDuty alert from Alertmanager (primary path)
- **Automated:** Watchdog failure via external dead man's switch
- **Manual:** Engineer notices anomaly in dashboard
- **External:** Customer report, support ticket, social media
- **Vendor:** AWS Health event, Stripe status change

## Phase 2: Triage (5 minutes)

On-call engineer MUST within 5 minutes:

1. **Acknowledge** in PagerDuty (stops re-escalation timer)
2. **Assess severity** using definitions above
3. **Create incident channel:** `/incident create` or manually create `#incident-YYYYMMDD-brief-description`
4. **Post initial assessment** in incident channel:
   ```
   🚨 INCIDENT DECLARED
   Severity: SEV[N]
   Service(s): [affected services]
   Impact: [who is affected, how]
   Detection: [alert name / how discovered]
   Current status: Investigating
   IC: [name or "self" if not yet assigned]
   ```
5. **If SEV1/2:** Page Incident Commander via PagerDuty
6. **If SEV1:** Page Communications Lead

## Phase 3: Mitigation (target: 15 minutes)

**CARDINAL RULE: Stop the bleeding BEFORE diagnosing.**

Priority order:
1. **Rollback** if correlated with a deploy (most common root cause)
2. **Scale up** if load-related
3. **Failover** if infrastructure failure
4. **Circuit break** if upstream dependency failure
5. **Shed load** if overwhelmed (disable non-critical features)

Document every action in incident channel with timestamps:
```
10:23 UTC — Rolling back payment-service to v2.3.1
10:25 UTC — Rollback complete. Error rate dropping.
10:27 UTC — Error rate at 0.3%, down from 15%. Monitoring.
```

## Phase 4: Diagnosis (parallel with mitigation if possible)

1. Follow the relevant **runbook**
2. Use the Three Pillars:
   - **Metrics:** SLO dashboard, Four Golden Signals
   - **Logs:** Loki/CloudWatch, filter by trace ID
   - **Traces:** Tempo, find the bottleneck span
3. **If observability is also down:** Use kubectl, AWS CLI, CloudWatch (see runbook fallback sections)
4. **Update incident channel every 15 minutes** even if no progress

## Phase 5: Resolution

1. Apply permanent fix (or confirm mitigation is stable)
2. **Verify:**
   - Error rate at baseline for 30 minutes
   - SLO burn rate alerts resolved
   - No new related alerts
   - Application health endpoints all passing
3. **Close incident:**
   ```
   ✅ INCIDENT RESOLVED
   Duration: [X hours Y minutes]
   Root cause: [brief description]
   Fix: [what was done]
   Action items: [will be in postmortem]
   Postmortem scheduled: [date/time]
   ```
4. Resolve in PagerDuty

## Phase 6: Learning (within 48 hours for SEV1/2)

1. Schedule postmortem meeting
2. Write postmortem document (template below)
3. Review and approve action items
4. Track action items to completion in Jira
5. Update runbooks based on lessons learned
6. Share postmortem with broader engineering team

## Communication Templates

### Status Page — Investigating
```
[Service] Degradation — Investigating
We are currently investigating reports of [description].
Some users may experience [impact].
Our team is actively working on this issue.
Next update in 15 minutes.
```

### Status Page — Identified
```
[Service] Degradation — Identified
We have identified the cause of [description].
[Brief, non-technical explanation of cause].
We are implementing a fix and expect resolution within [estimate].
Next update in 15 minutes.
```

### Status Page — Resolved
```
[Service] Degradation — Resolved
The issue affecting [service] has been resolved.
Duration: [start time] to [end time] ([duration]).
[Brief description of what happened and what was done].
We apologize for any inconvenience. A detailed review is underway
to prevent similar issues in the future.
```
```

## 3.3 Roles and Responsibilities

```markdown
# docs/operations/incident-roles.md

# Incident Roles

## Incident Commander (IC)

**Assigned:** SEV1/2 automatically. For SEV3: on-call is both IC and Tech Lead.

**Responsibilities:**
- Owns the incident end-to-end
- Coordinates, delegates, makes decisions — **does NOT debug**
- Assigns roles to other participants
- Makes go/no-go decisions (rollback, failover, escalate)
- Manages time: "It's been 15 minutes, where are we?"
- Shields tech lead from management/comms questions
- Declares incident resolved
- Ensures postmortem is scheduled

**Who can be IC:**
- Any Senior Engineer
- Engineering Manager
- SRE/Platform team lead
- NOT: the person who caused the incident (conflict of interest for blameless investigation)

**IC Checklist:**
- [ ] Acknowledge and confirm severity
- [ ] Open incident channel
- [ ] Assign Tech Lead, Comms Lead (SEV1/2), Scribe (SEV1)
- [ ] Ensure mitigation is happening (not just diagnosis)
- [ ] Status updates every 15 minutes
- [ ] Time-box decisions: "If not resolved by [time], we do [action]"
- [ ] Manage escalations
- [ ] Declare resolution
- [ ] Schedule postmortem

## Technical Lead

**Responsibilities:**
- The person debugging and fixing
- Follows runbook for the specific failure
- Reports findings to IC every 15 minutes
- Requests additional engineers if needed ("I need someone who knows payment gateway internals")
- Recommends actions to IC (IC approves)

**Rules:**
- Focus on the problem, not on communications
- If multiple engineers debugging, one is Tech Lead (coordinates)
- Document commands run and results (Scribe captures, but back yourself up)

## Communications Lead (SEV1/2 only)

**Responsibilities:**
- Writes and publishes status page updates
- Monitors and responds to #incident channel questions from non-participants
- Manages Slack announcements to #engineering and #company-wide
- Notifies customer support team with talking points
- Coordinates external communications if needed (PR, legal)

**Rules:**
- Use status page templates (above)
- Update every 15 minutes minimum
- Never share technical details externally without IC approval
- Never give time estimates publicly unless IC explicitly provides them

## Scribe (SEV1 only)

**Responsibilities:**
- Logs EVERYTHING with UTC timestamps
- Actions taken, results observed, decisions made, who said what
- This log becomes the postmortem timeline
- Captures screenshots of dashboards at key moments

**Format:**
```
10:15 UTC — [Alert] PaymentAvailabilityBurnCritical fired
10:17 UTC — [Action] @alice acknowledged in PagerDuty
10:18 UTC — [Action] @alice opened #incident-20240115-payment-5xx
10:19 UTC — [Decision] @bob assigned as IC, @alice as Tech Lead
10:20 UTC — [Finding] @alice: error rate 15%, started at 10:14, correlates with deploy v2.4.0
10:22 UTC — [Decision] @bob: approved rollback
10:23 UTC — [Action] @alice: kubectl argo rollouts undo payment-service
10:25 UTC — [Finding] @alice: rollback complete, error rate dropping
10:27 UTC — [Finding] @alice: error rate at 0.3%, continuing to drop
10:35 UTC — [Decision] @bob: error rate stable at baseline for 10min, declaring resolved
```
```

## 3.4 Postmortem Template

```markdown
# docs/templates/postmortem-template.md

# Postmortem: [INCIDENT-YYYYMMDD] [Title]

## Metadata
| Field | Value |
|-------|-------|
| **Date** | YYYY-MM-DD |
| **Duration** | HH:MM |
| **Severity** | SEV[N] |
| **Incident Commander** | @name |
| **Technical Lead** | @name |
| **Author** | @name |
| **Status** | Draft / In Review / Final / Action Items Complete |
| **Postmortem Meeting** | YYYY-MM-DD HH:MM |

## Executive Summary
<!-- 2-3 sentences. What happened, who was affected, how long, what was the business impact. -->

On [date] at [time] UTC, [service] experienced [description of failure] lasting [duration].
Approximately [N users / N% of traffic] were affected. Estimated revenue impact: $[X].
The root cause was [one sentence]. The issue was resolved by [one sentence].

## Customer Impact
- **Users affected:** [number or percentage]
- **Geographic scope:** [all regions / specific region]
- **Features affected:** [list]
- **Revenue impact:** $[amount] estimated ([failed_transactions] × $[avg_value])
- **SLO impact:**
  | SLO | Budget consumed by incident | Budget remaining after |
  |-----|---------------------------|----------------------|
  | [service]-availability | X% | Y% |
  | [service]-latency | X% | Y% |
- **Support tickets:** [count]
- **Social media reports:** [count, sentiment]

## Timeline (UTC)

| Time | Event | Source |
|------|-------|--------|
| | | |
<!-- Copy from Scribe log. Include: detection, actions, findings, decisions, resolution. -->

## Root Cause Analysis

### What happened
<!-- Technical description. Detailed enough that another engineer could understand. -->

### Why it happened
<!-- The 5 Whys or similar analysis. Go deep enough to reach a systemic cause. -->

1. **Why did [symptom]?** Because [direct cause].
2. **Why did [direct cause]?** Because [deeper cause].
3. **Why did [deeper cause]?** Because [process/tooling gap].
4. **Why did [process/tooling gap] exist?** Because [organizational reason].
5. **Why was [organizational reason] not addressed?** Because [systemic issue].

### Contributing factors
<!-- Other factors that made the incident worse, even if not the root cause. -->

## What Went Well
<!-- Celebrate what worked. This is important for morale and learning. -->
- [ item ]
- [ item ]

## What Went Wrong
<!-- No blame. Focus on systems, processes, tooling. -->
- [ item ]
- [ item ]

## Where We Got Lucky
<!-- Things that COULD have gone worse but didn't. These are future risks. -->
- [ item ]

## Action Items

| ID | Action | Type | Owner | Priority | Due Date | Status | Jira |
|----|--------|------|-------|----------|----------|--------|------|
| 1 | | prevent/detect/mitigate | @name | P1/P2/P3 | YYYY-MM-DD | Open | PLAT-XXX |
| 2 | | | | | | | |

**Action Item Types:**
- **Prevent:** Stop this class of incident from happening
- **Detect:** Find this issue faster next time
- **Mitigate:** Reduce impact if it happens again

**Priority:**
- **P1:** Must complete within 1 week
- **P2:** Must complete within 1 sprint (2 weeks)
- **P3:** Must complete within 1 quarter

## Lessons Learned
<!-- What organizational/architectural change prevents this CLASS of incident? -->

## Appendix
- Relevant dashboards/graphs (screenshots at time of incident)
- Relevant log excerpts
- Related prior incidents
```

## 3.5 PagerDuty Service Configuration

```yaml
# docs/operations/pagerduty-config.yaml
# Conceptual — applied via PagerDuty Terraform provider or API

# terraform/modules/pagerduty/main.tf
resource "pagerduty_service" "payment_critical" {
  name                    = "NovaMart Payment Critical"
  description             = "Payment service critical alerts — revenue impact"
  escalation_policy       = pagerduty_escalation_policy.payment_critical.id
  alert_creation          = "create_alerts_and_incidents"
  auto_resolve_timeout    = 14400  # 4 hours
  acknowledgement_timeout = 1800   # 30 min — re-pages if acked but not resolved

  alert_grouping_parameters {
    type    = "intelligent"
    config {
      timeout = 300  # 5 min grouping window
    }
  }

  incident_urgency_rule {
    type    = "constant"
    urgency = "high"  # Always high urgency — it's payment
  }
}

resource "pagerduty_service" "platform_critical" {
  name                    = "NovaMart Platform Critical"
  description             = "Non-payment critical alerts"
  escalation_policy       = pagerduty_escalation_policy.platform_critical.id
  alert_creation          = "create_alerts_and_incidents"
  auto_resolve_timeout    = 14400
  acknowledgement_timeout = 1800

  alert_grouping_parameters {
    type    = "intelligent"
    config {
      timeout = 300
    }
  }

  incident_urgency_rule {
    type    = "constant"
    urgency = "high"
  }
}

resource "pagerduty_service" "platform_warning" {
  name                    = "NovaMart Platform Warnings"
  description             = "Non-urgent warnings — Slack only, no page"
  escalation_policy       = pagerduty_escalation_policy.platform_warning.id
  alert_creation          = "create_alerts_and_incidents"
  auto_resolve_timeout    = 28800  # 8 hours
  acknowledgement_timeout = null   # No ack timeout — Slack notification only

  alert_grouping_parameters {
    type    = "intelligent"
    config {
      timeout = 600  # 10 min grouping
    }
  }

  incident_urgency_rule {
    type    = "constant"
    urgency = "low"
  }
}

# ─── Escalation Policies ───

resource "pagerduty_escalation_policy" "payment_critical" {
  name      = "Payment Critical Escalation"
  num_loops = 3  # Repeat entire chain 3 times before giving up

  rule {
    escalation_delay_in_minutes = 5
    target {
      type = "schedule_reference"
      id   = pagerduty_schedule.primary_oncall.id
    }
  }

  rule {
    escalation_delay_in_minutes = 5
    target {
      type = "schedule_reference"
      id   = pagerduty_schedule.secondary_oncall.id
    }
  }

  rule {
    escalation_delay_in_minutes = 5
    target {
      type = "user_reference"
      id   = pagerduty_user.engineering_manager.id
    }
  }

  rule {
    escalation_delay_in_minutes = 10
    target {
      type = "user_reference"
      id   = pagerduty_user.vp_engineering.id
    }
  }
}

resource "pagerduty_escalation_policy" "platform_critical" {
  name      = "Platform Critical Escalation"
  num_loops = 2

  rule {
    escalation_delay_in_minutes = 5
    target {
      type = "schedule_reference"
      id   = pagerduty_schedule.primary_oncall.id
    }
  }

  rule {
    escalation_delay_in_minutes = 10
    target {
      type = "schedule_reference"
      id   = pagerduty_schedule.secondary_oncall.id
    }
  }

  rule {
    escalation_delay_in_minutes = 15
    target {
      type = "user_reference"
      id   = pagerduty_user.engineering_manager.id
    }
  }
}

resource "pagerduty_escalation_policy" "platform_warning" {
  name      = "Platform Warning — Slack Only"
  num_loops = 0  # Don't repeat — it's just a Slack notification

  rule {
    escalation_delay_in_minutes = 0
    target {
      type = "schedule_reference"
      id   = pagerduty_schedule.primary_oncall.id
    }
  }
}

# ─── On-Call Schedules ───

resource "pagerduty_schedule" "primary_oncall" {
  name      = "Platform Primary On-Call"
  time_zone = "America/New_York"

  layer {
    name                         = "Primary Rotation"
    start                        = "2024-01-01T10:00:00-05:00"
    rotation_virtual_start       = "2024-01-01T10:00:00-05:00"
    rotation_turn_length_seconds = 604800  # 1 week
    users = [
      pagerduty_user.engineer_1.id,
      pagerduty_user.engineer_2.id,
      pagerduty_user.engineer_3.id,
      pagerduty_user.engineer_4.id,
      pagerduty_user.engineer_5.id,
      pagerduty_user.engineer_6.id,
    ]
  }
}

resource "pagerduty_schedule" "secondary_oncall" {
  name      = "Platform Secondary On-Call"
  time_zone = "America/New_York"

  layer {
    name                         = "Secondary Rotation"
    start                        = "2024-01-08T10:00:00-05:00"  # Offset by 1 week
    rotation_virtual_start       = "2024-01-08T10:00:00-05:00"
    rotation_turn_length_seconds = 604800
    users = [
      pagerduty_user.engineer_1.id,
      pagerduty_user.engineer_2.id,
      pagerduty_user.engineer_3.id,
      pagerduty_user.engineer_4.id,
      pagerduty_user.engineer_5.id,
      pagerduty_user.engineer_6.id,
    ]
  }
}

# ─── Integration Keys (Alertmanager → PagerDuty) ───

resource "pagerduty_service_integration" "payment_alertmanager" {
  name    = "Prometheus Alertmanager"
  service = pagerduty_service.payment_critical.id
  vendor  = data.pagerduty_vendor.prometheus.id
}

resource "pagerduty_service_integration" "platform_alertmanager" {
  name    = "Prometheus Alertmanager"
  service = pagerduty_service.platform_critical.id
  vendor  = data.pagerduty_vendor.prometheus.id
}

# Store integration keys in Secrets Manager for Alertmanager:
resource "aws_secretsmanager_secret" "pagerduty_payment_key" {
  name = "novamart/monitoring/pagerduty-payment-integration-key"
}

resource "aws_secretsmanager_secret_version" "pagerduty_payment_key" {
  secret_id     = aws_secretsmanager_secret.pagerduty_payment_key.id
  secret_string = pagerduty_service_integration.payment_alertmanager.integration_key
}
```

## 3.6 Alertmanager Full Configuration

```yaml
# manifests/platform/monitoring/alertmanager-config.yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: novamart-routing
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  route:
    receiver: default-slack
    groupBy: ['alertname', 'service', 'namespace', 'slo_service']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    routes:
      # ── WATCHDOG — Dead Man's Switch ──
      # Must be first — always routes here regardless of labels
      - matchers:
          - name: alertname
            value: Watchdog
            matchType: "="
        receiver: deadmans-switch
        groupWait: "0s"
        repeatInterval: 1m
        continue: false  # Don't also send to Slack

      # ── PAYMENT CRITICAL — PagerDuty immediate + Slack ──
      - matchers:
          - name: severity
            value: critical
            matchType: "="
          - name: pci
            value: "true"
            matchType: "="
        receiver: pagerduty-payment-critical
        groupWait: "10s"
        repeatInterval: 5m
        continue: true  # ALSO send to Slack

      # ── NON-PAYMENT CRITICAL — PagerDuty + Slack ──
      - matchers:
          - name: severity
            value: critical
            matchType: "="
        receiver: pagerduty-platform-critical
        groupWait: "30s"
        repeatInterval: 10m
        continue: true  # ALSO send to Slack

      # ── SLO BURN RATE — dedicated channel + conditional page ──
      - matchers:
          - name: alertname
            matchType: =~
            value: ".*Burn.*"
        receiver: slack-slo-alerts
        continue: true
        routes:
          - matchers:
              - name: severity
                value: critical
                matchType: "="
            receiver: pagerduty-platform-critical
            continue: false

      # ── SLO BUDGET EXHAUSTED — always page ──
      - matchers:
          - name: alertname
            value: SLOBudgetExhausted
            matchType: "="
        receiver: pagerduty-platform-critical
        groupWait: "10s"
        repeatInterval: 30m
        continue: true

      # ── SLI RECORDING RULE ABSENT — monitoring the monitor ──
      - matchers:
          - name: alertname
            value: SLIRecordingRuleAbsent
            matchType: "="
        receiver: pagerduty-platform-critical
        groupWait: "30s"
        continue: true

      # ── WARNINGS — Slack only ──
      - matchers:
          - name: severity
            value: warning
            matchType: "="
        receiver: slack-platform-warnings
        repeatInterval: 4h
        continue: false

      # ── INFO — low-priority Slack ──
      - matchers:
          - name: severity
            value: info
            matchType: "="
        receiver: slack-platform-info
        repeatInterval: 12h
        continue: false

  receivers:
    - name: pagerduty-payment-critical
      pagerdutyConfigs:
        - serviceKey:
            name: alertmanager-pagerduty-secrets
            key: payment-service-key
          severity: critical
          description: '{{ .CommonAnnotations.summary }}'
          details:
            - key: service
              value: '{{ .CommonLabels.service }}'
            - key: namespace
              value: '{{ .CommonLabels.namespace }}'
            - key: slo
              value: '{{ .CommonLabels.slo }}'
            - key: runbook
              value: '{{ .CommonAnnotations.runbook_url }}'
            - key: dashboard
              value: '{{ .CommonAnnotations.dashboard_url }}'
            - key: firing_alerts
              value: '{{ .Alerts.Firing | len }}'

    - name: pagerduty-platform-critical
      pagerdutyConfigs:
        - serviceKey:
            name: alertmanager-pagerduty-secrets
            key: platform-service-key
          severity: critical
          description: '{{ .CommonAnnotations.summary }}'
          details:
            - key: service
              value: '{{ .CommonLabels.service }}'
            - key: namespace
              value: '{{ .CommonLabels.namespace }}'
            - key: runbook
              value: '{{ .CommonAnnotations.runbook_url }}'

    - name: slack-slo-alerts
      slackConfigs:
        - apiURL:
            name: alertmanager-slack-secrets
            key: webhook-url
          channel: '#slo-alerts'
          sendResolved: true
          title: '{{ if eq .Status "firing" }}🔥{{ else }}✅{{ end }} SLO: {{ .CommonLabels.alertname }}'
          text: |
            *Service:* {{ .CommonLabels.slo_service }}
            *SLO:* {{ .CommonLabels.slo }} (target: {{ .CommonLabels.slo_target }}%)
            {{ .CommonAnnotations.description }}
          color: '{{ if eq .CommonLabels.severity "critical" }}danger{{ else }}warning{{ end }}'
          actions:
            - type: button
              text: 'Runbook'
              url: '{{ .CommonAnnotations.runbook_url }}'
            - type: button
              text: 'Dashboard'
              url: '{{ .CommonAnnotations.dashboard_url }}'

    - name: slack-platform-warnings
      slackConfigs:
        - apiURL:
            name: alertmanager-slack-secrets
            key: webhook-url
          channel: '#platform-alerts'
          sendResolved: true
          title: '{{ if eq .Status "firing" }}⚠️{{ else }}✅{{ end }} {{ .CommonLabels.alertname }}'
          text: |
            *Namespace:* {{ .CommonLabels.namespace }}
            *Service:* {{ .CommonLabels.service }}
            {{ .CommonAnnotations.description }}
          actions:
            - type: button
              text: 'Runbook'
              url: '{{ .CommonAnnotations.runbook_url }}'
            - type: button
              text: 'Dashboard'
              url: '{{ .CommonAnnotations.dashboard_url }}'

    - name: slack-platform-info
      slackConfigs:
        - apiURL:
            name: alertmanager-slack-secrets
            key: webhook-url
          channel: '#platform-alerts-low'
          sendResolved: false
          title: 'ℹ️ {{ .CommonLabels.alertname }}'
          text: '{{ .CommonAnnotations.description }}'

    - name: deadmans-switch
      webhookConfigs:
        - url: 'https://noflake.novamart.internal/deadman/prometheus'
          sendResolved: false
          # External service expects a ping every 1 minute.
          # If it stops receiving, it pages via its own PagerDuty integration.
          # This means: even if Alertmanager is broken, we still get paged.

    - name: default-slack
      slackConfigs:
        - apiURL:
            name: alertmanager-slack-secrets
            key: webhook-url
          channel: '#platform-alerts-low'
          sendResolved: true
          title: '{{ .CommonLabels.alertname }}'
          text: '{{ .CommonAnnotations.description }}'

  inhibitRules:
    # If critical fires, suppress warning for same service
    - sourceMatch:
        - name: severity
          value: critical
          matchType: "="
      targetMatch:
        - name: severity
          value: warning
          matchType: "="
      equal: ['alertname', 'service', 'namespace']

    # If KubeAPIDown, suppress all K8s alerts
    - sourceMatch:
        - name: alertname
          value: KubeAPIDown
          matchType: "="
      targetMatch:
        - name: alertname
          matchType: =~
          value: "Kube.*"
      equal: ['cluster']

    # If cluster unreachable, suppress all node/pod alerts for that cluster
    - sourceMatch:
        - name: alertname
          value: TargetDown
          matchType: "="
          # job: "kubelet" for example
      targetMatch:
        - name: severity
          value: warning
          matchType: "="
      equal: ['cluster']

    # If NodeNotReady, suppress pod alerts on that node
    - sourceMatch:
        - name: alertname
          value: KubeNodeNotReady
          matchType: "="
      targetMatch:
        - name: alertname
          matchType: =~
          value: "KubePod.*"
      equal: ['node']
---
# ExternalSecret for PagerDuty and Slack credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: alertmanager-pagerduty-secrets
  namespace: monitoring
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: alertmanager-pagerduty-secrets
  data:
    - secretKey: payment-service-key
      remoteRef:
        key: novamart/monitoring/pagerduty-payment-integration-key
    - secretKey: platform-service-key
      remoteRef:
        key: novamart/monitoring/pagerduty-platform-integration-key
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: alertmanager-slack-secrets
  namespace: monitoring
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: alertmanager-slack-secrets
  data:
    - secretKey: webhook-url
      remoteRef:
        key: novamart/monitoring/slack-alertmanager-webhook
```

---

# DELIVERABLE 4: Disaster Recovery

## 4.1 RTO/RPO Table

```markdown
# docs/operations/disaster-recovery.md

# NovaMart Disaster Recovery Plan

## RTO/RPO Matrix

| Component | RTO | RPO | Strategy | Backup Method | Backup Frequency | Retention |
|-----------|-----|-----|----------|---------------|-----------------|-----------|
| **Payment Service** | 5 min | 0 | Multi-AZ active-active, Argo Rollouts instant rollback | N/A (stateless) | N/A | N/A |
| **Payment DB (RDS)** | 10 min | 0 | Multi-AZ synchronous standby, automatic failover | Automated snapshots + PITR | Continuous (transaction log) | 35 days snapshots, 7 days PITR |
| **Order Service** | 15 min | 1 min | Multi-AZ, async replication for read replicas | N/A (stateless) | N/A | N/A |
| **Order DB (RDS)** | 15 min | 5 min | Multi-AZ sync standby + cross-region read replica | Automated snapshots + PITR | Continuous | 35 days |
| **Cart Service** | 30 min | 5 min | Multi-AZ, Redis replication | N/A (stateless) | N/A | N/A |
| **Redis (ElastiCache)** | 15 min | ~1 min | Multi-AZ replication group, auto-failover | RDB snapshots | Every 6 hours | 7 days |
| **User Service** | 15 min | 1 min | Multi-AZ | N/A (stateless) | N/A | N/A |
| **User DB (RDS)** | 10 min | 0 | Multi-AZ sync standby | Automated snapshots + PITR | Continuous | 35 days |
| **Search Service** | 1 hr | 1 hr | Rebuild index from source DB | N/A (derived data) | N/A | N/A |
| **Notification Service** | 1 hr | 30 min | Queue-backed (RabbitMQ), replay | N/A (stateless) | N/A | N/A |
| **RabbitMQ** | 30 min | 10 min | Mirrored queues across AZs | Queue snapshot to S3 | Every 1 hour | 7 days |
| **EKS Cluster** | 45 min | N/A | Rebuild from Terraform + ArgoCD (cattle) | N/A (IaC) | N/A | N/A |
| **Observability** | 1 hr | 1 hr | Rebuild from Helm, historical data in Thanos/S3 | Thanos S3 upload | Continuous (2h blocks) | raw:30d, 5m:90d, 1h:365d |
| **CI/CD (Jenkins)** | 2 hr | 24 hr | Rebuild from Terraform + S3 config backup | JCasC in Git, builds in S3 | Daily backup CronJob | 30 days |
| **Static Assets (S3)** | 15 min | 0 | S3 cross-region replication (CRR) | S3 versioning + CRR | Real-time | 90 days versioning |
| **Terraform State** | 5 min | 0 | S3 versioning + DynamoDB locking | S3 versioning | Every write | 90 days versioning |
| **Secrets (Vault/SM)** | 10 min | 0 | Secrets Manager multi-region replication | AWS-managed replication | Real-time | Automatic |
| **CloudTrail Logs** | N/A | 0 | S3 with Object Lock | S3 CRR | Real-time | 365 days (compliance) |

### RTO/RPO Justifications

- **Payment RPO=0:** Any lost payment transaction = financial discrepancy + chargebacks.
  Synchronous Multi-AZ replication ensures zero data loss on failover.
- **Payment RTO=5min:** Revenue loss at $3,800/min. Must recover before error budget exhausted (4.32 min).
- **EKS RTO=45min:** Cluster is cattle. All state lives in RDS/Redis/S3.
  Terraform + ArgoCD rebuilds everything. 45min is observed rebuild time.
- **Search RTO=1hr:** Search is derived data. Users can still browse and buy.
  Rebuilding index from source DB takes ~30min, plus warm-up.
- **Jenkins RTO=2hr:** CI/CD outage blocks deploys, not runtime. 2hr acceptable.
```

## 4.2 DR Runbook: Scenario 2 — Region Failure

```markdown
# RUNBOOK-DR-002: Region Failure (us-east-1 Down)

## Metadata
- **Severity:** SEV1 (by definition)
- **Estimated recovery time:** 30-60 minutes (manual failover)
- **Last tested:** 2024-07-15 (full DR drill)
- **Owner:** Platform Engineering + VP Engineering approval required
- **Pre-requisite:** DR region (us-west-2) running warm standby

## Decision Gate: IS IT A REAL REGION FAILURE?

**DO NOT failover for a single service outage. Verify region-level failure.**

```bash
# 1. Check AWS Health Dashboard:
aws health describe-events \
  --filter "regions=[us-east-1],eventStatusCodes=[open,upcoming]" \
  --query 'events[*].{Service:service,Status:statusCode,Description:eventTypeCode}' \
  --output table

# 2. Verify multiple services affected:
aws ec2 describe-instance-status --region us-east-1 --include-all-instances \
  --query 'InstanceStatuses[?SystemStatus.Status!=`ok`] | length(@)' --output text
# If many instances unhealthy → region issue

# 3. Check EKS API reachability:
kubectl cluster-info 2>&1 | head -3
# If "Unable to connect to the server" → EKS control plane unreachable

# 4. Cross-reference with multiple AWS services:
aws rds describe-db-instances --region us-east-1 \
  --query 'DBInstances[?DBInstanceStatus!=`available`]' --output table
aws elasticache describe-replication-groups --region us-east-1 \
  --query 'ReplicationGroups[?Status!=`available`]' --output table

# DECISION: If ≥ 3 AWS services are degraded in us-east-1 AND
# EKS control plane is unreachable → Proceed with DR failover.
# REQUIRES: VP Engineering verbal approval (Slack/phone) before proceeding.
```

## Step 1: Activate DR Region (us-west-2) — 10 minutes

```bash
# TRAP 3 HANDLED: We can't use kubectl against us-east-1.
# All commands target us-west-2 or use AWS API.

# 1. Switch kubectl context to DR cluster:
aws eks update-kubeconfig --region us-west-2 --name novamart-dr --alias dr-cluster
kubectl config use-context dr-cluster

# 2. Verify DR cluster is operational:
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed
# DR cluster runs minimal replicas in steady state (cost saving).
# System components + ArgoCD should be fully running.

# 3. Scale up DR workloads:
# ArgoCD in DR is configured with "auto-sync: false" for app workloads.
# Enable sync to deploy latest manifests:
for app in api-gateway payment-service order-service cart-service user-service search-service notification-service; do
  argocd app sync ${app}-dr --force
done

# 4. Wait for all deployments to be ready:
kubectl get deployments -A -o json | \
  jq '.items[] | select(.status.readyReplicas != .spec.replicas) | {name: .metadata.name, namespace: .metadata.namespace, ready: .status.readyReplicas, desired: .spec.replicas}'
# Target: all ready == desired

# 5. If pods stuck Pending (not enough nodes):
# Karpenter in DR will provision nodes. Give it 2-3 minutes.
# If Karpenter not scaling fast enough:
kubectl get nodeclaims  # Check Karpenter is provisioning
```

## Step 2: Promote RDS Read Replicas — 10-15 minutes

```bash
# CRITICAL: This is DESTRUCTIVE. It breaks replication permanently.
# The read replica becomes a standalone primary.
# You CANNOT undo this without re-creating the replication chain.

# 1. Promote Payment DB read replica:
aws rds promote-read-replica \
  --db-instance-identifier novamart-payment-dr-replica \
  --region us-west-2
# Wait for status to become "available" (5-10 minutes):
aws rds wait db-instance-available \
  --db-instance-identifier novamart-payment-dr-replica \
  --region us-west-2

# 2. Promote Order DB read replica:
aws rds promote-read-replica \
  --db-instance-identifier novamart-order-dr-replica \
  --region us-west-2
aws rds wait db-instance-available \
  --db-instance-identifier novamart-order-dr-replica \
  --region us-west-2

# 3. Promote User DB read replica:
aws rds promote-read-replica \
  --db-instance-identifier novamart-user-dr-replica \
  --region us-west-2
aws rds wait db-instance-available \
  --db-instance-identifier novamart-user-dr-replica \
  --region us-west-2

# 4. VERIFY databases are writable:
kubectl run psql-test --rm -it --image=postgres:15 -n platform-ops -- \
  psql "host=novamart-payment-dr-replica.xxxxx.us-west-2.rds.amazonaws.com \
  port=5432 user=master dbname=payment sslmode=require" \
  -c "CREATE TABLE dr_test (id int); DROP TABLE dr_test; SELECT 'WRITABLE';"
# Must return "WRITABLE". If "cannot execute in read-only transaction" → 
# promotion not complete yet. Wait and retry.

# 5. NOTE: RPO for promoted replicas depends on replication lag at time of failure:
# Payment DB: synchronous replication → RPO=0
# Order/User DB: async cross-region → RPO = last replication lag (typically 1-5 min)
# Check last known replication lag (from CloudWatch before failure):
# Metric: ReplicaLag for each read replica
```

## Step 3: Update Application Database Endpoints — 5 minutes

```bash
# Applications get DB endpoints from External Secrets (Secrets Manager).
# DR region has its own Secrets Manager with DR-specific endpoints.

# 1. Verify DR secrets are correct:
aws secretsmanager get-secret-value \
  --secret-id novamart/payment/database-url \
  --region us-west-2 | jq -r '.SecretString' | jq '.host'
# Should point to: novamart-payment-dr-replica.xxxxx.us-west-2.rds.amazonaws.com

# 2. If secrets need updating (promoted replica has new endpoint):
aws secretsmanager update-secret \
  --secret-id novamart/payment/database-url \
  --region us-west-2 \
  --secret-string "{
    \"host\": \"novamart-payment-dr-replica.xxxxx.us-west-2.rds.amazonaws.com\",
    \"port\": \"5432\",
    \"dbname\": \"payment\",
    \"username\": \"payment_app\",
    \"password\": \"$(aws secretsmanager get-secret-value --secret-id novamart/payment/db-password --region us-west-2 --query SecretString --output text)\"
  }"

# Repeat for order and user databases.

# 3. Force External Secrets refresh in DR cluster:
kubectl get externalsecrets -A -o name | while read es; do
  NS=$(echo $es | cut -d/ -f1)
  NAME=$(echo $es | cut -d/ -f2)
  kubectl annotate $es force-sync=$(date +%s) --overwrite
done

# 4. Rolling restart all services to pick up new secrets:
for ns in payment order user cart api-gateway search notification; do
  kubectl rollout restart deployment -n $ns
done

# 5. Wait for rollouts:
for ns in payment order user cart api-gateway search notification; do
  echo "=== Waiting for $ns ==="
  kubectl rollout status deployment -n $ns --timeout=300s
done
```

## Step 4: Update Redis — 5 minutes

```bash
# ElastiCache Global Datastore (if configured):
# Promote DR replication group to primary:
aws elasticache failover-global-replication-group \
  --global-replication-group-id novamart-redis-global \
  --primary-replication-group-id novamart-redis-dr \
  --region us-west-2

# If NOT using Global Datastore (standalone DR Redis):
# DR Redis is already running. Cart data from primary region is LOST.
# RPO = last Redis snapshot (every 6 hours) + any in-memory data.
# This is acceptable: cart data is low-value, users can re-add items.

# Verify Redis is accessible:
kubectl run redis-test --rm -it --image=redis:7 -n platform-ops -- \
  redis-cli -h novamart-redis-dr.xxxxx.usw2.cache.amazonaws.com \
  --tls -a "$REDIS_AUTH_TOKEN" PING
# Must return: PONG
```

## Step 5: Update DNS / Traffic Routing — 5 minutes

```bash
# 1. Route53 health check should have already detected us-east-1 failure.
# If failover routing policy is configured, DNS may have already shifted.
# VERIFY:
dig +short api.novamart.com
# Should resolve to us-west-2 ALB IP if failover activated.

# 2. If automatic failover did NOT happen (or if using manual failover):
# Update Route53 record to point to DR ALB:
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.novamart.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "novamart-dr-alb-xxxxx.us-west-2.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# 3. Update Cloudflare origin:
# Cloudflare dashboard → DNS → Update origin to us-west-2 ALB
# Or via API:
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/dns_records/${CF_RECORD_ID}" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data "{
    \"content\": \"novamart-dr-alb-xxxxx.us-west-2.elb.amazonaws.com\",
    \"proxied\": true
  }"

# 4. Wait for DNS propagation:
# Cloudflare: nearly instant (proxied)
# Route53: TTL-dependent (our TTL is 60s for failover records)
# Verify from multiple locations:
for resolver in 8.8.8.8 1.1.1.1 208.67.222.222; do
  echo "=== $resolver ==="
  dig +short api.novamart.com @$resolver
done
```

## Step 6: Verify All Services in DR — 10 minutes

```bash
# 1. Health endpoints:
for svc in api-gateway payment-service order-service cart-service user-service; do
  echo "=== $svc ==="
  kubectl exec -n ${svc%-service} deploy/$svc -c $svc -- \
    curl -sf localhost:8080/health | jq '.status'
done
# All must return "UP"

# 2. End-to-end test (synthetic transaction):
# Run smoke test suite against DR:
kubectl run smoke-test --rm -it --image=novamart/smoke-tests:latest -n platform-ops -- \
  /run-tests.sh --env=dr --suite=critical
# Tests: login, search, add to cart, checkout (test mode), order status

# 3. Monitoring stack health:
kubectl get pods -n monitoring
# Prometheus, Grafana, Loki should all be running in DR
# Note: historical data from primary is in Thanos S3 — 
# Thanos Store Gateway in DR can query it.

# 4. SLO recording rules active:
curl -sf "http://prometheus.monitoring.svc:9090/api/v1/rules" | \
  jq '.data.groups[] | select(.name | startswith("slo")) | .name'
# Should show all SLO groups

# 5. PagerDuty integration working:
# Watchdog alert should be firing in DR Prometheus → Alertmanager → PagerDuty
# Verify in PagerDuty: dead man's switch should NOT have triggered
# (because DR Prometheus is now sending Watchdog)
```

## Step 7: Post-Failover Stabilization

```bash
# 1. Scale services to production levels:
# DR runs at reduced replicas to save cost.
# Now that it's serving production traffic, scale up:
kubectl scale deployment api-gateway -n api-gateway --replicas=5
kubectl scale deployment payment-service -n payment --replicas=5
kubectl scale deployment order-service -n order --replicas=3
kubectl scale deployment cart-service -n cart --replicas=3
kubectl scale deployment user-service -n user --replicas=3
kubectl scale deployment search-service -n search --replicas=3
# Karpenter will provision additional nodes as needed.

# 2. Monitor for 30 minutes:
# Watch SLO dashboard for error rates
# Watch node scaling (Karpenter provisioning)
# Watch RDS metrics (CPU, connections, replication — now standalone)

# 3. Communicate:
# Status page: "Services have been restored via failover to DR region.
#  Some users may need to log in again. We are monitoring."
# Internal: "#incident channel: DR failover complete. Monitoring."

# 4. Begin planning failback (when us-east-1 recovers):
# Failback is MORE complex than failover:
#   - Must re-establish RDS replication
#   - Must sync data written during DR period back to primary
#   - Must be carefully coordinated to avoid split-brain
#   - Typically done during low-traffic maintenance window
#   - See RUNBOOK-DR-005: Failback from DR to Primary (not included here)
```

## Rollback (Abort Failover)

If you start the failover process and realize it was wrong:

```bash
# If DNS not yet changed: just stop. No harm done (except promoted replicas).
# Re-creating replication to us-east-1 from promoted replica is complex but possible.

# If DNS already changed:
# Revert Route53/Cloudflare back to us-east-1 endpoints.
# Services in us-east-1 should still be running if the region recovered.
# The promoted replicas in us-west-2 now have divergent data.
# This creates a SPLIT BRAIN situation — resolve with application team.
# Payment data reconciliation is CRITICAL — every transaction must be accounted for.
```

## Appendix: DR Steady-State Architecture

```
PRIMARY (us-east-1) — Active
├── EKS cluster (full scale)
├── RDS Primary instances (payment, order, user)
├── ElastiCache Primary
├── S3 buckets (with CRR to us-west-2)
└── Route53 health checks → points traffic here

DR (us-west-2) — Warm Standby
├── EKS cluster (minimal replicas — system + ArgoCD + monitoring)
├── RDS Read Replicas (async replication from primary)
│   ├── Payment: cross-region read replica (lag: ~seconds)
│   ├── Order: cross-region read replica (lag: ~seconds)
│   └── User: cross-region read replica (lag: ~seconds)
├── ElastiCache (standalone, periodic snapshot restore OR Global Datastore)
├── S3 buckets (CRR destination — already has all data)
├── ArgoCD (synced but apps suspended — ready to activate)
└── Route53 failover record (secondary)

COST: DR region costs ~30% of primary (read replicas + minimal EKS)
      This is the tradeoff for RTO of 30-60 min.
      Active-active would be RTO ~0 but 2x cost. See DD-OPS-003.
```
```

## 4.3 DR Runbook: Scenario 4 — Cluster Rebuild

```markdown
# RUNBOOK-DR-004: EKS Cluster Destroyed — Full Rebuild

## Metadata
- **Severity:** SEV1
- **Estimated recovery time:** 45-60 minutes
- **Last tested:** 2024-08-20 (quarterly DR exercise)
- **Owner:** Platform Engineering
- **KEY PRINCIPLE:** The cluster is cattle, not a pet. All state lives outside the cluster.
  If you can't rebuild it from code, something is wrong with your IaC.

## Pre-Requisites
- Terraform state in S3 (us-east-1) is intact
  - If state is ALSO lost: see "State Recovery" section below
- Git repository with all Terraform and manifests is accessible
- AWS credentials working (OIDC or emergency IAM user)
- Local machine has: terraform, kubectl, aws-cli, argocd-cli, helm

## Step 1: Assess Damage — 5 minutes

```bash
# 1. Can we reach the EKS API?
aws eks describe-cluster --name novamart-prod --region us-east-1 2>&1
# If "ResourceNotFoundException" → cluster completely gone
# If "ClusterStatus: FAILED" → cluster exists but broken
# If timeout → networking issue, not cluster destruction

# 2. Are nodes recoverable?
aws ec2 describe-instances \
  --filters "Name=tag:kubernetes.io/cluster/novamart-prod,Values=owned" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,State:State.Name}' \
  --output table
# If no instances → full rebuild needed
# If instances exist but cluster API gone → might just need control plane

# 3. Is data tier intact?
aws rds describe-db-instances --query 'DBInstances[*].{ID:DBInstanceIdentifier,Status:DBInstanceStatus}' --output table
aws elasticache describe-replication-groups --query 'ReplicationGroups[*].{ID:ReplicationGroupId,Status:Status}' --output table
# Data tier should be UNAFFECTED by cluster destruction.
# If data tier is also gone → this is Scenario 2 (region failure), not Scenario 4.
```

## Step 2: Rebuild Infrastructure — 20-30 minutes

```bash
# 1. Ensure Terraform state is accessible:
cd terraform/environments/production
terraform -chdir=security init -backend-config=backend.hcl
terraform -chdir=security plan
# Should show "No changes" — security layer should be intact

# 2. Networking layer (should be intact — EKS deletion doesn't touch VPC):
terraform -chdir=networking init -backend-config=backend.hcl
terraform -chdir=networking plan
# Should show "No changes" (VPC, subnets, NAT GW all still exist)

# 3. Rebuild EKS cluster:
terraform -chdir=eks init -backend-config=backend.hcl
terraform -chdir=eks plan
# This WILL show resources to create: EKS cluster, node groups, add-ons, IRSA roles

# REVIEW THE PLAN CAREFULLY even under pressure.
# Common mistakes when rushing:
# - Wrong cluster version (might default to latest instead of pinned)
# - Missing managed node group (system pool)
# - OIDC provider recreation (changes ARN — breaks all IRSA!)

terraform -chdir=eks apply
# EKS cluster creation: ~10-15 minutes
# Node group creation: ~5-10 minutes
# Add-ons installation: ~2-3 minutes

# 4. Verify cluster:
aws eks update-kubeconfig --name novamart-prod --region us-east-1
kubectl get nodes
kubectl get pods -n kube-system
# VPC CNI, CoreDNS, kube-proxy should all be running
```

## Step 3: Bootstrap ArgoCD — 5-10 minutes

```bash
# 1. Install ArgoCD (Terraform manages the Helm release):
terraform -chdir=platform init -backend-config=backend.hcl

# BUT: The full platform Terraform may have dependencies on resources
# that don't exist yet (Prometheus CRDs, etc.)
# Use targeted apply for ArgoCD first:
terraform -chdir=platform apply -target=helm_release.argocd

# 2. Verify ArgoCD running:
kubectl get pods -n argocd
kubectl get svc -n argocd

# 3. Get ArgoCD admin password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
# Or: External Secrets should recreate from Secrets Manager

# 4. Deploy App of Apps:
kubectl apply -f manifests/argocd/app-of-apps.yaml
# This triggers ArgoCD to deploy EVERYTHING:
# - Platform components (Prometheus, Loki, Linkerd, Kyverno, etc.)
# - Application workloads (payment, order, cart, etc.)
# ArgoCD sync waves ensure correct ordering:
#   Wave 0: CRDs + namespaces
#   Wave 1: cert-manager + External Secrets
#   Wave 2: Kyverno + Linkerd
#   Wave 3: Monitoring stack
#   Wave 4: Application services
#   Wave 5: Dashboards + policies

# 5. Monitor sync progress:
watch argocd app list
# Wait for all apps to reach "Synced" and "Healthy"
# Expected time: 10-15 minutes for all waves
```

## Step 4: Verify All Services — 10 minutes

```bash
# 1. Platform components:
echo "=== Linkerd ==="
linkerd check
echo "=== Prometheus ==="
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
echo "=== ArgoCD ==="
argocd app list | grep -v Healthy
echo "=== Kyverno ==="
kubectl get pods -n kyverno
echo "=== cert-manager ==="
kubectl get pods -n cert-manager
kubectl get certificates -A

# 2. Application health:
for ns in api-gateway payment order cart user search notification; do
  echo "=== $ns ==="
  kubectl get pods -n $ns
  kubectl get svc -n $ns
done

# 3. Data connectivity:
# Services should auto-connect to RDS/Redis (endpoints in External Secrets)
kubectl logs -n payment deploy/payment-service --tail=20 | grep -i "database\|connection\|redis"
# Look for: "database connection established" or similar

# 4. End-to-end health:
# Once ALB is provisioned and DNS points to it:
curl -sf https://api.novamart.com/health | jq '.'
# If ALB not ready yet:
kubectl port-forward -n api-gateway svc/api-gateway 8080:80 &
curl -sf http://localhost:8080/health | jq '.'

# 5. Run platform validation:
./scripts/validate-platform.sh
```

## Step 5: Restore External Access — 5 minutes

```bash
# 1. ALB provisioning (AWS LB Controller recreates from Ingress resources):
kubectl get ingress -A
# ALBs take 2-3 minutes to provision and pass health checks.

# 2. If ALB DNS name changed (new ALB = new DNS name):
# Update Route53 alias records:
NEW_ALB=$(kubectl get ingress -n api-gateway api-gateway-ingress \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "New ALB: $NEW_ALB"

aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"api.novamart.com\",
        \"Type\": \"A\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"$(aws elbv2 describe-load-balancers --query 'LoadBalancers[?DNSName==\`$NEW_ALB\`].CanonicalHostedZoneId' --output text)\",
          \"DNSName\": \"$NEW_ALB\",
          \"EvaluateTargetHealth\": true
        }
      }
    }]
  }"

# 3. Cloudflare origin update (if using Cloudflare):
# Update origin to new ALB hostname

# 4. Verify external access:
curl -sf https://api.novamart.com/health
# Should return healthy
```

## State Recovery (if Terraform state is also lost)

```bash
# If S3 state bucket is intact but state file corrupted:
# 1. Check S3 versioning:
aws s3api list-object-versions \
  --bucket novamart-terraform-state \
  --prefix production/eks/terraform.tfstate \
  --query 'Versions[0:5].{VersionId:VersionId,LastModified:LastModified,Size:Size}' \
  --output table
# Restore previous version:
aws s3api get-object \
  --bucket novamart-terraform-state \
  --key production/eks/terraform.tfstate \
  --version-id <previous-version-id> \
  restored-state.tfstate

# If state is completely lost:
# 2. Import existing resources:
# This is painful but possible. Start with critical resources:
terraform import module.eks.aws_eks_cluster.main novamart-prod
terraform import module.eks.aws_eks_node_group.system novamart-prod:system-pool
# ... continue for each resource
# This is why we use S3 versioning with 90-day retention.

# If EVERYTHING is lost (state + cluster + infrastructure):
# 3. Start from scratch:
terraform -chdir=security apply
terraform -chdir=networking apply
terraform -chdir=data apply  # Re-creates RDS — DATA LOSS unless restoring from snapshot
terraform -chdir=eks apply
# Then bootstrap ArgoCD as above.
# RDS restoration from snapshot:
# aws rds restore-db-instance-from-db-snapshot ...
```

## Verification
```bash
# Full checklist:
# [ ] kubectl get nodes — all Ready
# [ ] kubectl get pods -A — no unhealthy pods (except expected CronJob completed)
# [ ] argocd app list — all Synced + Healthy
# [ ] linkerd check — all green
# [ ] External access works (curl api.novamart.com)
# [ ] SLO dashboard shows data
# [ ] Watchdog alert firing → PagerDuty receiving
# [ ] Payment test transaction succeeds
# [ ] Monitor for 30 minutes before declaring resolved
```

## Post-Incident
- [ ] Full timeline documented
- [ ] Root cause of cluster destruction identified
  - Accidental terraform destroy?
  - AWS issue?
  - Security incident?
- [ ] If accidental: add `prevent_destroy` lifecycle rule to EKS cluster resource
- [ ] If security: see RUNBOOK-SEC-002
- [ ] Verify all data intact (RDS, Redis, S3)
- [ ] Review: did the rebuild time meet our 45-minute RTO?
- [ ] Update this runbook with any steps that were wrong or slow
```

## 4.4 DR Exercise Schedule

```markdown
# docs/operations/dr-exercise-schedule.md

# NovaMart DR Exercise Schedule

## Annual Calendar

| Exercise | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct | Nov | Dec |
|----------|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
| AZ Failure Sim | ✓ | | ✓ | | ✓ | | ✓ | | ✓ | | ✓ | |
| RDS Failover | | ✓ | | | | ✓ | | | | ✓ | | |
| Cluster Rebuild | | | ✓ | | | | | ✓ | | | | |
| Region Failover | | | | | | | ✓ | | | | | |
| Backup Restore | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Chaos Game Day | | ✓ | | ✓ | | ✓ | | ✓ | | ✓ | | ✓ |
| Runbook Walk | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Secret Rotation | | | ✓ | | | ✓ | | | ✓ | | | ✓ |

## Exercise Procedures

### AZ Failure Simulation (Bi-monthly, 2 hours)
1. Pre-announce to all teams 24h in advance
2. Select one AZ (rotate each exercise)
3. Cordon all nodes in that AZ: `kubectl cordon -l topology.kubernetes.io/zone=us-east-1a`
4. Drain nodes: `kubectl drain -l topology.kubernetes.io/zone=us-east-1a --ignore-daemonsets --delete-emptydir-data`
5. Observe: Do pods reschedule? Do PDBs hold? Does Karpenter provision?
6. Monitor SLO dashboard — did any SLO breach?
7. Uncordon after observation: `kubectl uncordon -l topology.kubernetes.io/zone=us-east-1a`
8. Document findings

### RDS Failover (Quarterly, 1 hour)
1. Pre-announce. Expect brief connection errors.
2. Force failover: `aws rds reboot-db-instance --db-instance-identifier novamart-payment-prod --force-failover`
3. Time the failover (expected: 30-60s)
4. Monitor application error rate during failover
5. Verify applications reconnect automatically
6. Document: connection errors observed, recovery time

### Backup Restore (Monthly, 1 hour)
1. List available RDS snapshots
2. Restore latest automated snapshot to a test instance:
   `aws rds restore-db-instance-from-db-snapshot --db-instance-identifier dr-test-restore --db-snapshot-identifier <latest>`
3. Verify data integrity on test instance (row counts, checksums)
4. Delete test instance
5. Record: "Backup restore tested [date]. Result: [pass/fail]. Data verified."

### Secret Rotation Drill (Quarterly, 1 hour)
1. Select one non-production secret (staging Stripe test key)
2. Follow RUNBOOK-SEC-004 procedure
3. Verify service continues operating after rotation
4. Time the rotation (target: < 10 minutes)
5. Document any steps that were unclear or slow

## Exercise Report Template

```markdown
## DR Exercise Report

| Field | Value |
|-------|-------|
| Exercise | [type] |
| Date | YYYY-MM-DD |
| Duration | HH:MM |
| Participants | [names] |
| Result | PASS / PARTIAL / FAIL |

### Objective
[What we tested]

### Execution Summary
[What we did, step by step]

### Results
- Target RTO: [X min]
- Actual recovery time: [Y min]
- SLO impact: [which SLOs affected, budget consumed]
- Unexpected issues: [list]

### Findings
| Finding | Severity | Action Required |
|---------|----------|----------------|
| | | |

### Action Items
| Action | Owner | Due Date |
|--------|-------|----------|
| | | |
```

## Rules
1. **NEVER skip a scheduled exercise** for feature work. DR readiness is non-negotiable.
2. **Always document results** even when everything works perfectly.
3. **Region failover exercises** require VP Engineering approval and full team staffing.
4. **November-December freeze:** No region failover exercises during holiday shopping season.
   All other exercises continue.
5. **After any real incident**, conduct the relevant DR exercise within 2 weeks
   to verify the fix works.
```

## 4.5 Backup Validation Procedure

```markdown
# docs/operations/backup-validation.md

# NovaMart Backup Validation Procedure

## Automated Backup Verification (Daily CronJob)

A CronJob verifies backup existence and recency every day.
It does NOT restore — that's the monthly manual exercise.

```yaml
# manifests/platform/operations/backup-validator-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-validator
  namespace: platform-ops
spec:
  schedule: "0 6 * * *"  # 6 AM UTC daily
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          serviceAccountName: backup-validator
          containers:
            - name: validator
              image: novamart/platform-tools:v1.2.0
              command: ["/bin/bash", "-c"]
              args:
                - |
                  #!/bin/bash
                  set -euo pipefail
                  
                  FAILURES=0
                  REPORT=""
                  NOW=$(date +%s)
                  MAX_AGE_HOURS=26  # Allow 2-hour buffer beyond 24h cycle
                  
                  # ─── RDS Automated Snapshots ───
                  for db in novamart-payment-prod novamart-order-prod novamart-user-prod; do
                    LATEST=$(aws rds describe-db-snapshots \
                      --db-instance-identifier $db \
                      --snapshot-type automated \
                      --query 'sort_by(DBSnapshots, &SnapshotCreateTime)[-1].{Time:SnapshotCreateTime,Status:Status,ID:DBSnapshotIdentifier}' \
                      --output json)
                    
                    SNAP_TIME=$(echo $LATEST | jq -r '.Time')
                    SNAP_STATUS=$(echo $LATEST | jq -r '.Status')
                    SNAP_ID=$(echo $LATEST | jq -r '.ID')
                    SNAP_EPOCH=$(date -d "$SNAP_TIME" +%s)
                    AGE_HOURS=$(( (NOW - SNAP_EPOCH) / 3600 ))
                    
                    if [ "$SNAP_STATUS" != "available" ]; then
                      REPORT+="❌ RDS $db: Latest snapshot status=$SNAP_STATUS (expected: available)\n"
                      FAILURES=$((FAILURES + 1))
                    elif [ "$AGE_HOURS" -gt "$MAX_AGE_HOURS" ]; then
                      REPORT+="❌ RDS $db: Latest snapshot ${AGE_HOURS}h old (max: ${MAX_AGE_HOURS}h)\n"
                      FAILURES=$((FAILURES + 1))
                    else
                      REPORT+="✅ RDS $db: $SNAP_ID (${AGE_HOURS}h ago, $SNAP_STATUS)\n"
                    fi
                  done
                  
                  # ─── Redis Snapshots ───
                  for rg in novamart-redis-prod; do
                    LATEST=$(aws elasticache describe-snapshots \
                      --replication-group-id $rg \
                      --query 'sort_by(Snapshots, &NodeSnapshots[0].SnapshotCreateTime)[-1].{Time:NodeSnapshots[0].SnapshotCreateTime,Status:SnapshotStatus,Name:SnapshotName}' \
                      --output json 2>/dev/null || echo '{"Time":"","Status":"none","Name":"none"}')
                    
                    SNAP_STATUS=$(echo $LATEST | jq -r '.Status')
                    if [ "$SNAP_STATUS" == "none" ] || [ "$SNAP_STATUS" == "null" ]; then
                      REPORT+="⚠️ Redis $rg: No snapshots found\n"
                      FAILURES=$((FAILURES + 1))
                    else
                      SNAP_TIME=$(echo $LATEST | jq -r '.Time')
                      SNAP_EPOCH=$(date -d "$SNAP_TIME" +%s 2>/dev/null || echo 0)
                      AGE_HOURS=$(( (NOW - SNAP_EPOCH) / 3600 ))
                      if [ "$AGE_HOURS" -gt 8 ]; then  # Redis snapshots every 6h
                        REPORT+="❌ Redis $rg: Snapshot ${AGE_HOURS}h old (max: 8h)\n"
                        FAILURES=$((FAILURES + 1))
                      else
                        REPORT+="✅ Redis $rg: $(echo $LATEST | jq -r '.Name') (${AGE_HOURS}h ago)\n"
                      fi
                    fi
                  done
                  
                  # ─── Thanos S3 Blocks ───
                  THANOS_BLOCKS=$(aws s3 ls s3://novamart-thanos-data/data/ --recursive | \
                    grep "meta.json" | tail -1)
                  if [ -z "$THANOS_BLOCKS" ]; then
                    REPORT+="❌ Thanos: No blocks found in S3\n"
                    FAILURES=$((FAILURES + 1))
                  else
                    LATEST_BLOCK_TIME=$(echo "$THANOS_BLOCKS" | awk '{print $1" "$2}')
                    REPORT+="✅ Thanos: Latest block uploaded $LATEST_BLOCK_TIME\n"
                  fi
                  
                  # ─── Loki S3 Chunks ───
                  LOKI_CHUNKS=$(aws s3 ls s3://novamart-loki-data/chunks/ --recursive | tail -1)
                  if [ -z "$LOKI_CHUNKS" ]; then
                    REPORT+="❌ Loki: No chunks in S3\n"
                    FAILURES=$((FAILURES + 1))
                  else
                    REPORT+="✅ Loki: Latest chunk $(echo "$LOKI_CHUNKS" | awk '{print $1" "$2}')\n"
                  fi
                  
                  # ─── Terraform State Versioning ───
                  for state_key in security/terraform.tfstate networking/terraform.tfstate eks/terraform.tfstate data/terraform.tfstate; do
                    VERSIONS=$(aws s3api list-object-versions \
                      --bucket novamart-terraform-state \
                      --prefix "production/$state_key" \
                      --query 'Versions | length(@)' --output text 2>/dev/null || echo "0")
                    if [ "$VERSIONS" -lt 2 ]; then
                      REPORT+="⚠️ TF State $state_key: Only $VERSIONS versions (should have history)\n"
                    else
                      REPORT+="✅ TF State $state_key: $VERSIONS versions available\n"
                    fi
                  done
                  
                  # ─── Jenkins Backup ───
                  JENKINS_BACKUP=$(aws s3 ls s3://novamart-ci-artifacts/jenkins-backups/ | tail -1)
                  if [ -z "$JENKINS_BACKUP" ]; then
                    REPORT+="❌ Jenkins: No backups found in S3\n"
                    FAILURES=$((FAILURES + 1))
                  else
                    REPORT+="✅ Jenkins: Latest backup $(echo "$JENKINS_BACKUP" | awk '{print $1" "$2}')\n"
                  fi
                  
                  # ─── Send Report ───
                  if [ "$FAILURES" -gt 0 ]; then
                    COLOR="danger"
                    EMOJI="🚨"
                    # Also create PagerDuty alert for critical backup failures
                    if [ "$FAILURES" -gt 2 ]; then
                      curl -X POST "https://events.pagerduty.com/v2/enqueue" \
                        -H "Content-Type: application/json" \
                        -d "{
                          \"routing_key\": \"$PD_ROUTING_KEY\",
                          \"event_action\": \"trigger\",
                          \"payload\": {
                            \"summary\": \"Backup validation: $FAILURES failures detected\",
                            \"severity\": \"warning\",
                            \"source\": \"backup-validator\"
                          }
                        }"
                    fi
                  else
                    COLOR="good"
                    EMOJI="✅"
                  fi
                  
                  curl -X POST "$SLACK_WEBHOOK" -H 'Content-type: application/json' \
                    -d "{
                      \"attachments\": [{
                        \"color\": \"$COLOR\",
                        \"title\": \"$EMOJI NovaMart Backup Validation Report\",
                        \"text\": \"$(echo -e $REPORT)\",
                        \"footer\": \"backup-validator | $(date -u +%Y-%m-%d)\"
                      }]
                    }"
                  
                  exit $FAILURES  # Non-zero exit = job marked Failed = visible in monitoring
              env:
                - name: SLACK_WEBHOOK
                  valueFrom:
                    secretKeyRef:
                      name: platform-ops-secrets
                      key: slack-webhook
                - name: PD_ROUTING_KEY
                  valueFrom:
                    secretKeyRef:
                      name: platform-ops-secrets
                      key: pagerduty-routing-key
              resources:
                requests:
                  cpu: 100m
                  memory: 128Mi
                limits:
                  cpu: 500m
                  memory: 256Mi
          restartPolicy: OnFailure
```

### Manual Restore Validation (Monthly)

```bash
# Step 1: Restore RDS snapshot to test instance
LATEST_SNAP=$(aws rds describe-db-snapshots \
  --db-instance-identifier novamart-payment-prod \
  --snapshot-type automated \
  --query 'sort_by(DBSnapshots, &SnapshotCreateTime)[-1].DBSnapshotIdentifier' \
  --output text)

aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier dr-restore-test-$(date +%Y%m%d) \
  --db-snapshot-identifier $LATEST_SNAP \
  --db-instance-class db.r6g.large \
  --no-multi-az \
  --tags Key=Purpose,Value=dr-test Key=TTL,Value=4h

# Step 2: Wait for availability (~10-15 min)
aws rds wait db-instance-available \
  --db-instance-identifier dr-restore-test-$(date +%Y%m%d)

# Step 3: Validate data
ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier dr-restore-test-$(date +%Y%m%d) \
  --query 'DBInstances[0].Endpoint.Address' --output text)

kubectl run psql-validate --rm -it --image=postgres:15 -n platform-ops -- \
  psql "host=$ENDPOINT port=5432 user=master dbname=payment sslmode=require" \
  -c "
    SELECT 'payments' as table_name, count(*) as row_count FROM payments
    UNION ALL
    SELECT 'refunds', count(*) FROM refunds
    UNION ALL
    SELECT 'payment_methods', count(*) FROM payment_methods;
  "
# Compare row counts with production. Should be within 24h of writes.

# Step 4: Cleanup
aws rds delete-db-instance \
  --db-instance-identifier dr-restore-test-$(date +%Y%m%d) \
  --skip-final-snapshot

# Step 5: Document
echo "Backup restore test $(date): PASS. Snapshot: $LATEST_SNAP. Rows verified."
# Log to #platform-ops Slack channel
```
```

---

# DELIVERABLE 5: Chaos Engineering

## 5.1 Chaos Experiments

```yaml
# manifests/chaos/experiments/01-payment-pod-kill.yaml
# ─────────────────────────────────────────────────────────
# EXPERIMENT 1: Payment Pod Kill
# Hypothesis: Payment service handles single pod failure with zero
#             dropped requests due to Linkerd retry and K8s rescheduling.
# Expected: Error rate stays within SLO (99.99%). No customer-visible errors.
# Blast radius: 1 pod out of 5 (20% capacity reduction)
# Abort: If error rate exceeds 1% for 2 minutes
# ─────────────────────────────────────────────────────────
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: payment-pod-kill
  namespace: chaos-testing
  labels:
    experiment: pod-resilience
    service: payment
    risk: low
  annotations:
    chaos.novamart.internal/hypothesis: "Single pod kill causes zero customer-visible errors"
    chaos.novamart.internal/abort-criteria: "Error rate > 1% for 2 minutes"
    chaos.novamart.internal/owner: "platform-team"
    chaos.novamart.internal/last-run: "2024-11-01"
spec:
  action: pod-kill
  mode: one           # Kill exactly ONE random pod
  selector:
    namespaces:
      - payment
    labelSelectors:
      app: payment-service
  duration: "30s"     # Kill one pod, observe for 30s, then experiment ends
                      # (K8s recreates pod — we're testing the gap)
---
# Abort condition: CloudWatch alarm watches error rate
# If alarm fires → Chaos Mesh experiment is paused
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: payment-pod-kill-schedule
  namespace: chaos-testing
spec:
  schedule: "@every 5m"    # Repeat every 5 min during game day (manual start)
  type: PodChaos
  historyLimit: 5
  concurrencyPolicy: Forbid
  podChaos:
    action: pod-kill
    mode: one
    selector:
      namespaces:
        - payment
      labelSelectors:
        app: payment-service
    duration: "30s"
---
# manifests/chaos/experiments/02-network-latency-rds.yaml
# ─────────────────────────────────────────────────────────
# EXPERIMENT 2: Database Latency Injection
# Hypothesis: Payment service tolerates 200ms added DB latency
#             without breaching p99 < 1000ms SLO.
# Expected: p99 increases from ~300ms to ~500ms. Still within SLO.
# Blast radius: All payment pods → all DB queries slower
# Abort: If p99 > 900ms (approaching 1000ms SLO boundary)
# ─────────────────────────────────────────────────────────
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-db-latency
  namespace: chaos-testing
  labels:
    experiment: latency-tolerance
    service: payment
    risk: medium
  annotations:
    chaos.novamart.internal/hypothesis: "200ms added DB latency keeps p99 under 1000ms"
    chaos.novamart.internal/abort-criteria: "p99 > 900ms for 1 minute"
    chaos.novamart.internal/owner: "platform-team"
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - payment
    labelSelectors:
      app: payment-service
  delay:
    latency: "200ms"
    jitter: "50ms"
    correlation: "75"    # 75% correlation with previous packet
  direction: to
  externalTargets:
    - "novamart-payment-prod.xxxxx.us-east-1.rds.amazonaws.com"
  duration: "5m"
---
# manifests/chaos/experiments/03-az-drain.yaml
# ─────────────────────────────────────────────────────────
# EXPERIMENT 3: Availability Zone Drain
# Hypothesis: Losing all nodes in us-east-1a causes automatic
#             rescheduling to 1b/1c within 5 minutes. SLOs maintained.
# Expected: ~33% capacity loss, Karpenter provisions replacement nodes
#           in healthy AZs, PDBs prevent total service loss during drain.
# Blast radius: All pods on us-east-1a nodes
# Abort: If ANY service has 0 healthy pods for > 2 minutes
# ─────────────────────────────────────────────────────────
apiVersion: chaos-mesh.org/v1alpha1
kind: PhysicalMachineChaos
metadata:
  name: az-drain-us-east-1a
  namespace: chaos-testing
  labels:
    experiment: az-resilience
    risk: high
  annotations:
    chaos.novamart.internal/hypothesis: "AZ loss recovers within 5 minutes"
    chaos.novamart.internal/abort-criteria: "Any critical service at 0 healthy replicas for 2 min"
    chaos.novamart.internal/owner: "platform-team"
    chaos.novamart.internal/requires-approval: "engineering-manager"
spec:
  # NOTE: For AZ drain, we use kubectl approach (more controlled than Chaos Mesh node kill):
  # This experiment is SCRIPTED, not purely CRD-driven.
  # See game-day-procedure.md Step 3 for the actual drain commands.
  # The CRD here is for documentation/tracking in Chaos Mesh dashboard.
  action: "stress-cpu"  # Placeholder — actual execution is manual drain
  mode: all
  selector:
    namespaces:
      - "*"
    fieldSelectors:
      "spec.nodeName": "ip-10-0-1*"  # Nodes in AZ-a subnet range
  duration: "15m"
---
# manifests/chaos/experiments/04-dns-failure.yaml
# ─────────────────────────────────────────────────────────
# EXPERIMENT 4: CoreDNS Failure
# Hypothesis: NodeLocal DNSCache handles CoreDNS pod failure.
#             If NodeLocal DNS not deployed, services with cached DNS
#             continue for TTL duration. New connections fail.
# Expected: With NodeLocal DNS: zero impact. Without: degradation after TTL.
# Blast radius: Cluster-wide DNS resolution
# Abort: If payment error rate > 0.5% (5x budget)
# ─────────────────────────────────────────────────────────
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: coredns-pod-kill
  namespace: chaos-testing
  labels:
    experiment: dns-resilience
    risk: high
  annotations:
    chaos.novamart.internal/hypothesis: "DNS failure handled by cache or redundancy"
    chaos.novamart.internal/abort-criteria: "Payment error rate > 0.5%"
    chaos.novamart.internal/owner: "platform-team"
    chaos.novamart.internal/requires-approval: "engineering-manager"
spec:
  action: pod-kill
  mode: fixed
  value: "1"            # Kill 1 of 3 CoreDNS pods
  selector:
    namespaces:
      - kube-system
    labelSelectors:
      k8s-app: kube-dns
  duration: "60s"
---
# manifests/chaos/experiments/05-cascading-latency.yaml
# ─────────────────────────────────────────────────────────
# EXPERIMENT 5: Cascading Failure — User Service Slowdown
# Hypothesis: 2s latency injection into user-service does NOT cascade
#             to payment-service and order-service due to timeouts and
#             circuit breaking (Linkerd retry budget).
# Expected: user-service degrades. API gateway returns partial results.
#           Payment and order services remain unaffected.
# Blast radius: user-service and its callers
# Abort: If payment error rate > 0.1% OR order error rate > 0.5%
# ─────────────────────────────────────────────────────────
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: user-service-cascade-test
  namespace: chaos-testing
  labels:
    experiment: cascade-resilience
    risk: high
  annotations:
    chaos.novamart.internal/hypothesis: "User-service slowdown does not cascade to payment/order"
    chaos.novamart.internal/abort-criteria: "Payment error > 0.1% OR order error > 0.5%"
    chaos.novamart.internal/owner: "platform-team"
    chaos.novamart.internal/requires-approval: "engineering-manager"
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - user
    labelSelectors:
      app: user-service
  delay:
    latency: "2000ms"     # 2 full seconds — enough to exhaust caller connection pools
    jitter: "500ms"
  direction: from         # Delay RESPONSES from user-service
  duration: "10m"
```

## 5.2 Abort Conditions (CloudWatch Alarms)

```yaml
# manifests/chaos/abort-conditions.yaml
# These CloudWatch alarms are used as safety nets during chaos experiments.
# If any alarm fires, the experiment is manually aborted.

# Terraform resource for chaos abort alarms:
# terraform/modules/chaos-safety/main.tf

resource "aws_cloudwatch_metric_alarm" "chaos_abort_payment_errors" {
  alarm_name          = "chaos-abort-payment-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 50  # More than 50 5xx in 1 minute = abort
  alarm_description   = "Chaos experiment safety net: payment 5xx too high"
  alarm_actions       = [aws_sns_topic.chaos_abort.arn]
  dimensions = {
    TargetGroup  = var.payment_target_group_arn_suffix
    LoadBalancer = var.alb_arn_suffix
  }
  tags = {
    Purpose = "chaos-safety"
  }
}

resource "aws_cloudwatch_metric_alarm" "chaos_abort_zero_healthy" {
  alarm_name          = "chaos-abort-zero-healthy-targets"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HealthyHostCount"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Minimum"
  threshold           = 1  # Any service with 0 healthy targets = abort
  alarm_description   = "Chaos experiment safety net: service has zero healthy targets"
  alarm_actions       = [aws_sns_topic.chaos_abort.arn]
  dimensions = {
    TargetGroup  = var.payment_target_group_arn_suffix
    LoadBalancer = var.alb_arn_suffix
  }
}

resource "aws_sns_topic" "chaos_abort" {
  name = "novamart-chaos-abort"
}

resource "aws_sns_topic_subscription" "chaos_abort_slack" {
  topic_arn = aws_sns_topic.chaos_abort.arn
  protocol  = "https"
  endpoint  = var.slack_webhook_url  # Posts to #chaos-experiments channel
}
```

## 5.3 Game Day Procedure

```markdown
# docs/operations/chaos-game-day-procedure.md

# NovaMart Chaos Game Day Procedure

## Pre-Game Day (Day -1)

### Preparation Checklist
- [ ] Announce game day in #engineering (24h notice minimum)
- [ ] Confirm date does NOT conflict with:
  - [ ] Major sales event or promotion
  - [ ] Planned deployments
  - [ ] Peak traffic hours (avoid 10 AM - 2 PM ET for e-commerce)
  - [ ] Holiday shopping season (Nov 15 - Jan 2 blackout)
- [ ] Confirm participants (minimum 3: experimenter, observer, safety officer)
- [ ] Roles assigned:
  - **Experimenter:** Executes chaos experiments
  - **Observer:** Monitors dashboards, records observations
  - **Safety Officer:** Watches abort conditions, has kill switch access
- [ ] Verify abort alarms are active in CloudWatch
- [ ] Verify all chaos experiment YAMLs are reviewed and approved
- [ ] Open Grafana dashboards: SLO overview, Four Golden Signals, per-service
- [ ] Ensure on-call is NOT skeleton crew
- [ ] Pre-create game day report document from template

### Safety Briefing
- Game day Slack channel: #chaos-gameday-YYYYMMDD
- Safety Officer has authority to abort ANY experiment at ANY time
- All participants know the abort commands:
  ```bash
  # Nuclear abort — pause ALL chaos experiments:
  kubectl delete podchaos,networkchaos,stresschaos,iochaos --all -n chaos-testing
  
  # Targeted abort — specific experiment:
  kubectl delete podchaos payment-pod-kill -n chaos-testing
  ```
- If a REAL incident occurs during game day: STOP ALL experiments, handle incident

## Game Day Schedule

### 09:00 — Brief (30 min)
- Review hypotheses for today's experiments
- Review abort criteria for each experiment
- Confirm dashboards are showing real-time data
- Safety Officer confirms abort alarms are active
- Post in #chaos-gameday channel:
  ```
  🎮 CHAOS GAME DAY STARTING
  Date: YYYY-MM-DD
  Experiments: [list]
  Participants: @experimenter @observer @safety
  Abort channel: #chaos-gameday-YYYYMMDD
  Safety officer: @name (has kill switch)
  ```

### 09:30 — Experiment 1: Pod Kill (Low Risk, 30 min)
```bash
# Experimenter:
kubectl apply -f manifests/chaos/experiments/01-payment-pod-kill.yaml

# Observer monitors:
# - SLO dashboard: payment error rate
# - Pod count: kubectl get pods -n payment -w
# - Linkerd dashboard: retry rate, success rate

# Record:
# - Time of pod kill: HH:MM:SS
# - Time pod rescheduled: HH:MM:SS
# - Error rate during gap: X%
# - Customer-visible errors: Y (from ALB 5xx count)
```
**Expected outcome:** Pod killed → new pod scheduled within 30s → Linkerd retries cover the gap → zero 5xx visible at ALB.
**Success criteria:** ALB 5xx count = 0 during experiment.
**If unexpected:** Safety Officer aborts. Document what happened.

### 10:00 — Experiment 2: Database Latency (Medium Risk, 30 min)
```bash
# Experimenter:
kubectl apply -f manifests/chaos/experiments/02-network-latency-rds.yaml

# Observer monitors:
# - Payment p99 latency
# - Connection pool utilization
# - SLO burn rate

# Record:
# - Baseline p99 before injection: Xms
# - p99 during injection: Xms
# - Connection pool active/max during injection
# - Any timeouts observed
```
**Expected outcome:** p99 rises from ~300ms to ~500ms. Still under 1000ms SLO.
**Success criteria:** p99 stays below 900ms. No connection pool exhaustion.
**Abort trigger:** p99 > 900ms sustained for 1 minute.

### 10:30 — Experiment 3: AZ Drain (High Risk, 45 min)
```bash
# REQUIRES: Engineering manager verbal approval before proceeding

# Experimenter:
# Step 1: Identify nodes in target AZ
TARGET_NODES=$(kubectl get nodes -l topology.kubernetes.io/zone=us-east-1a -o name)
echo "Draining: $TARGET_NODES"

# Step 2: Cordon all nodes in AZ (prevent new scheduling)
kubectl cordon -l topology.kubernetes.io/zone=us-east-1a

# Step 3: Drain nodes one at a time (respect PDBs)
for node in $TARGET_NODES; do
  echo "Draining $node..."
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data --timeout=120s
  sleep 10  # Pause between drains to let Karpenter react
done

# Observer monitors:
# - Pod rescheduling (where do they go?)
# - Karpenter provisioning new nodes (how fast?)
# - Service error rates across ALL services
# - RDS: did Multi-AZ failover trigger? (depends on RDS primary AZ)

# Record:
# - Time first node cordoned: HH:MM:SS
# - Time all nodes drained: HH:MM:SS
# - Time all pods rescheduled: HH:MM:SS
# - Karpenter nodes provisioned: count and time
# - Any SLO breaches: which service, how long

# Step 4: Recovery (after observation period)
kubectl uncordon -l topology.kubernetes.io/zone=us-east-1a
```
**Expected outcome:** Pods reschedule to 1b/1c within 5 minutes. Karpenter provisions nodes. SLOs maintained due to PDBs.
**Abort trigger:** Any critical service at 0 ready replicas for > 2 minutes.

### 11:15 — Experiment 4: DNS Failure (High Risk, 20 min)
```bash
# Experimenter:
kubectl apply -f manifests/chaos/experiments/04-dns-failure.yaml

# This kills 1 of 3 CoreDNS pods.
# Observer: does NodeLocal DNS cache prevent impact?

# Record:
# - DNS query latency change
# - Any services reporting DNS errors in logs
# - Time to CoreDNS pod replacement
```

### 11:35 — Experiment 5: Cascading Failure Test (High Risk, 30 min)
```bash
# Experimenter:
kubectl apply -f manifests/chaos/experiments/05-cascading-latency.yaml

# This adds 2s latency to user-service responses.
# THE CRITICAL QUESTION: Does this cascade?

# Observer monitors:
# - user-service p99 (should be ~2s)
# - api-gateway error rate (may increase for user-related endpoints)
# - payment-service error rate (MUST NOT increase)
# - order-service error rate (MUST NOT increase)
# - Connection pool usage on all services calling user-service
# - Linkerd retry budget exhaustion

# Record:
# - Which services were affected?
# - Did circuit breaking activate?
# - Was there retry amplification?
```
**Expected outcome:** user-service degrades. API gateway returns partial results. Payment and order unaffected.
**Abort trigger:** Payment error rate > 0.1% OR order error rate > 0.5%.

### 12:05 — Debrief (30 min)
- Review each experiment's results
- Compare expected vs actual outcomes
- Identify surprises
- Create Jira tickets for every gap found
- Post summary in #engineering

### 12:35 — Documentation (30 min)
- Complete game day report
- Update experiment YAMLs with lessons learned
- Update runbooks if any steps were wrong
- Post final report in #platform-ops

## Game Day Report Template

```markdown
# Chaos Game Day Report — YYYY-MM-DD

## Participants
| Role | Name |
|------|------|
| Experimenter | @name |
| Observer | @name |
| Safety Officer | @name |

## Summary
[1-2 sentences: overall result]

## Experiment Results

### Experiment 1: Payment Pod Kill
| Field | Value |
|-------|-------|
| Hypothesis | [stated hypothesis] |
| Result | CONFIRMED / DISPROVED / PARTIAL |
| Error rate during experiment | X% |
| Recovery time | Xs |
| SLO impact | None / [details] |
| Surprise findings | [anything unexpected] |

[Repeat for each experiment]

## Key Findings

| # | Finding | Severity | Action Required |
|---|---------|----------|----------------|
| 1 | [finding] | High/Med/Low | [action] |

## Action Items

| # | Action | Owner | Jira | Due |
|---|--------|-------|------|-----|
| 1 | [action] | @name | PLAT-XXX | YYYY-MM-DD |

## Metrics Collected
[Screenshots of dashboards at key moments]

## Next Game Day
- Date: [next month]
- Experiments planned: [list]
- New experiments to develop: [based on findings]
```
```

---

# DELIVERABLE 6: Day-2 Operations

## 6.1 Operations Checklists

```markdown
# docs/operations/operations-checklists.md

# NovaMart Operations Checklists

---

## Daily Checklist (15 min — automated report + human review)

Automated daily health report runs at 8:00 AM UTC (CronJob below).
On-call reviews the report and acts on any ❌ items.

| # | Check | Automated? | Action if Failed |
|---|-------|-----------|-----------------|
| 1 | SLO dashboard — any service YELLOW or worse? | ✅ CronJob | Review burn rate. If ORANGE+: escalate to eng lead. |
| 2 | ArgoCD — all apps Synced + Healthy? | ✅ CronJob | Check sync errors: `argocd app get <app>`. Manual sync if safe. |
| 3 | Prometheus — all scrape targets up? | ✅ CronJob | See RUNBOOK-PLAT-001. |
| 4 | Certificates — any expiring < 30 days? | ✅ CronJob | Check cert-manager logs. Force renewal if needed. |
| 5 | Pending pods — any stuck > 10 min? | ✅ CronJob | Check events. IP exhaustion? Node pressure? Image pull? |
| 6 | Node health — any NotReady? | ✅ CronJob | See RUNBOOK-INFRA-001. |
| 7 | CI/CD — Jenkins queue depth < 10? | ✅ CronJob | Check agent provisioning. Stuck builds? |
| 8 | Cost anomaly — daily spend within 20% of forecast? | Manual | Check AWS Cost Explorer. Investigate spikes. |
| 9 | Security — CloudTrail root usage? GuardDuty findings? | Manual | If root usage: investigate immediately. |
| 10 | Backup validation report — all ✅? | ✅ CronJob | If any ❌: investigate immediately. Backup failure is silent data loss risk. |

## Weekly Checklist (Monday 9:00 AM — 1 hour meeting)

| # | Check | Owner | Cadence |
|---|-------|-------|---------|
| 1 | **SLO Review Meeting** — 7-day compliance, budget trends | Platform Lead | Every Monday |
| 2 | **On-call handoff** — outgoing briefs incoming, active issues | On-call swap | Every Monday |
| 3 | **Dependency CVE review** — new vulnerabilities in base images? | Security champion | Every Monday |
| 4 | **Capacity review** — node util trend, RDS storage, Redis memory, ECR count | On-call | Every Monday |
| 5 | **Cost review** — weekly spend vs budget, top drivers, spot savings | Platform Lead | Every Monday |
| 6 | **Failed build review** — top failure reasons, flaky tests | CI/CD owner | Every Monday |
| 7 | **Incident review** — any open action items from past incidents? | IC/Platform Lead | Every Monday |

### SLO Review Meeting Agenda (30 min)
```
1. Dashboard review — each service's budget state (5 min)
2. Services in YELLOW+ — root cause, action plan (10 min)
3. Prior week's action items — status update (5 min)
4. Error budget trend — are we burning faster this month? (5 min)
5. Open discussion — upcoming risks (deploys, events, migrations) (5 min)
```

### On-Call Handoff Checklist
```


```markdown
Outgoing on-call provides to incoming:
- [ ] Active incidents (any still open?)
- [ ] Ongoing issues with temporary workarounds
- [ ] Scheduled deployments this week
- [ ] Scheduled maintenance windows
- [ ] Known flaky alerts (and status of fixing them)
- [ ] Any services in ORANGE/RED budget state
- [ ] Any infrastructure changes in progress (Terraform PRs)
- [ ] Anything unusual observed but not yet investigated

Incoming on-call verifies:
- [ ] PagerDuty schedule shows me as primary
- [ ] Phone receives test page (PagerDuty test notification)
- [ ] VPN access working
- [ ] kubectl access to all clusters (prod, staging, DR)
- [ ] AWS Console access working
- [ ] Grafana/ArgoCD/Jenkins accessible
- [ ] Know where runbooks are (https://runbooks.novamart.internal)
- [ ] Know who secondary on-call is this week
```

## Monthly Checklist (scheduled — 4 hours)

| # | Check | Duration | Owner |
|---|-------|----------|-------|
| 1 | **Chaos game day** | 4 hrs | Platform team (rotating lead) |
| 2 | **DR exercise** (rotating per schedule) | 2-4 hrs | Platform team |
| 3 | **Runbook walk-through** — pick 3-5 runbooks, verify accuracy | 1 hr | On-call |
| 4 | **Platform upgrade review** — EKS, Helm charts, Terraform providers | 1 hr | Platform Lead |
| 5 | **Access review** — RBAC audit, unused IRSA roles, departed engineers | 1 hr | Security champion |
| 6 | **Backup restore test** — restore one RDS snapshot, verify | 1 hr | On-call |
| 7 | **Performance baseline** — p99 trends, memory drift, slow query growth | 30 min | On-call |
| 8 | **Toil review** — what was manual this month that should be automated? | 30 min | Platform Lead |
| 9 | **SLO target review** — targets still appropriate? Too loose? Too tight? | 30 min | Platform Lead + eng leads |
| 10 | **ECR image cleanup** — verify lifecycle policies running, no bloat | 15 min | On-call |

## Quarterly Checklist (scheduled — multi-day)

| # | Check | Duration | Owner | Notes |
|---|-------|----------|-------|-------|
| 1 | **EKS version upgrade** (if new version available) | 1-2 days | Platform team | Staging first, soak 5 days, then prod |
| 2 | **Full DR region failover exercise** | 1 day | Platform team + VP approval | See DR exercise schedule |
| 3 | **Architecture review** | 2 hrs | Platform + senior eng | Bottlenecks, scaling, cost |
| 4 | **SLO renegotiation with product** | 1 hr | Platform Lead + Product | Adjust targets based on data |
| 5 | **On-call retrospective** | 1 hr | Platform team | Page volume, toil %, burnout risk |
| 6 | **Security penetration test review** | 2 hrs | Security team + Platform | Review findings, action items |
| 7 | **PCI-DSS evidence collection** | 4 hrs | Compliance + Platform | Quarterly PCI requirements |
| 8 | **Cost optimization deep dive** | 2 hrs | Platform Lead + FinOps | RI/SP coverage, right-sizing, waste |
| 9 | **Dependency major version review** | 2 hrs | Platform team | Major Helm chart upgrades, deprecations |
| 10 | **Disaster recovery plan review** | 1 hr | Platform Lead | RTO/RPO still valid? Gaps? |
```

## 6.2 Automated Health Check CronJob

```yaml
# manifests/platform/operations/daily-health-check.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-health-check
  namespace: platform-ops
  labels:
    app.kubernetes.io/name: daily-health-check
    app.kubernetes.io/part-of: novamart-operations
spec:
  schedule: "0 8 * * *"   # 8:00 AM UTC daily
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 600  # 10 min max
      template:
        metadata:
          labels:
            app: daily-health-check
        spec:
          serviceAccountName: platform-health-checker
          containers:
            - name: health-check
              image: novamart/platform-tools:v1.2.0
              command: ["/bin/bash", "-c"]
              args:
                - |
                  #!/bin/bash
                  set -uo pipefail
                  # NOTE: not set -e because we want to collect ALL failures, not exit on first
                  
                  FAILURES=0
                  WARNINGS=0
                  REPORT=""
                  
                  add_pass()    { REPORT+="✅ $1\n"; }
                  add_fail()    { REPORT+="❌ $1\n"; FAILURES=$((FAILURES + 1)); }
                  add_warn()    { REPORT+="⚠️ $1\n"; WARNINGS=$((WARNINGS + 1)); }
                  
                  # ═══════════════════════════════════════
                  # CHECK 1: ArgoCD Sync Status
                  # ═══════════════════════════════════════
                  ARGOCD_TOKEN=$(cat /var/run/secrets/argocd/token 2>/dev/null || echo "")
                  if [ -n "$ARGOCD_TOKEN" ]; then
                    OUTOFSYNC=$(curl -sf -H "Authorization: Bearer $ARGOCD_TOKEN" \
                      "http://argocd-server.argocd:80/api/v1/applications" | \
                      jq '[.items[] | select(.status.sync.status != "Synced")] | length' 2>/dev/null || echo "-1")
                    DEGRADED=$(curl -sf -H "Authorization: Bearer $ARGOCD_TOKEN" \
                      "http://argocd-server.argocd:80/api/v1/applications" | \
                      jq '[.items[] | select(.status.health.status != "Healthy" and .status.health.status != "Progressing")] | length' 2>/dev/null || echo "-1")
                    
                    if [ "$OUTOFSYNC" = "-1" ]; then
                      add_fail "ArgoCD: Unable to query API"
                    elif [ "$OUTOFSYNC" -gt 0 ]; then
                      APPS=$(curl -sf -H "Authorization: Bearer $ARGOCD_TOKEN" \
                        "http://argocd-server.argocd:80/api/v1/applications" | \
                        jq -r '[.items[] | select(.status.sync.status != "Synced") | .metadata.name] | join(", ")')
                      add_fail "ArgoCD: $OUTOFSYNC apps OutOfSync: $APPS"
                    else
                      add_pass "ArgoCD: All apps Synced"
                    fi
                    
                    if [ "$DEGRADED" != "-1" ] && [ "$DEGRADED" -gt 0 ]; then
                      APPS=$(curl -sf -H "Authorization: Bearer $ARGOCD_TOKEN" \
                        "http://argocd-server.argocd:80/api/v1/applications" | \
                        jq -r '[.items[] | select(.status.health.status != "Healthy" and .status.health.status != "Progressing") | "\(.metadata.name):\(.status.health.status)"] | join(", ")')
                      add_fail "ArgoCD: $DEGRADED apps unhealthy: $APPS"
                    elif [ "$DEGRADED" != "-1" ]; then
                      add_pass "ArgoCD: All apps Healthy"
                    fi
                  else
                    add_warn "ArgoCD: Token not available, skipping"
                  fi
                  
                  # ═══════════════════════════════════════
                  # CHECK 2: Prometheus Targets
                  # ═══════════════════════════════════════
                  PROM_URL="http://kube-prometheus-stack-prometheus.monitoring:9090"
                  DOWN_TARGETS=$(curl -sf "$PROM_URL/api/v1/targets" | \
                    jq '[.data.activeTargets[] | select(.health != "up")] | length' 2>/dev/null || echo "-1")
                  
                  if [ "$DOWN_TARGETS" = "-1" ]; then
                    add_fail "Prometheus: Unable to query API — MONITORING MAY BE DOWN"
                  elif [ "$DOWN_TARGETS" -gt 0 ]; then
                    DOWN_LIST=$(curl -sf "$PROM_URL/api/v1/targets" | \
                      jq -r '[.data.activeTargets[] | select(.health != "up") | .labels.job] | unique | join(", ")')
                    add_fail "Prometheus: $DOWN_TARGETS targets down: $DOWN_LIST"
                  else
                    TOTAL=$(curl -sf "$PROM_URL/api/v1/targets" | \
                      jq '.data.activeTargets | length')
                    add_pass "Prometheus: All $TOTAL targets healthy"
                  fi
                  
                  # ═══════════════════════════════════════
                  # CHECK 3: Certificate Expiry
                  # ═══════════════════════════════════════
                  EXPIRING=$(kubectl get certificates -A -o json 2>/dev/null | \
                    jq --arg cutoff "$(date -u -d '+30 days' +%Y-%m-%dT%H:%M:%SZ)" \
                    '[.items[] | select(.status.notAfter != null and .status.notAfter < $cutoff) | 
                    "\(.metadata.namespace)/\(.metadata.name) expires \(.status.notAfter)"] | length' 2>/dev/null || echo "-1")
                  
                  if [ "$EXPIRING" = "-1" ]; then
                    add_warn "Certificates: Unable to query (cert-manager CRDs missing?)"
                  elif [ "$EXPIRING" -gt 0 ]; then
                    CERT_LIST=$(kubectl get certificates -A -o json | \
                      jq -r --arg cutoff "$(date -u -d '+30 days' +%Y-%m-%dT%H:%M:%SZ)" \
                      '[.items[] | select(.status.notAfter != null and .status.notAfter < $cutoff) | 
                      "\(.metadata.namespace)/\(.metadata.name) → \(.status.notAfter)"] | join(", ")')
                    add_fail "Certificates: $EXPIRING expiring within 30d: $CERT_LIST"
                  else
                    add_pass "Certificates: All valid > 30 days"
                  fi
                  
                  # ═══════════════════════════════════════
                  # CHECK 4: SLO Budget Status
                  # ═══════════════════════════════════════
                  for svc in api-gateway payment-service cart-service order-service user-service; do
                    BUDGET=$(curl -sf "$PROM_URL/api/v1/query" \
                      --data-urlencode "query=sli:availability:budget_remaining:ratio{slo_service=\"$svc\"}" | \
                      jq -r '.data.result[0].value[1] // "unknown"' 2>/dev/null || echo "unknown")
                    
                    if [ "$BUDGET" = "unknown" ] || [ "$BUDGET" = "null" ]; then
                      add_warn "SLO $svc: Unable to query budget (SLI recording rules may be missing)"
                    else
                      BUDGET_PCT=$(echo "$BUDGET * 100" | bc -l 2>/dev/null | xargs printf "%.1f" 2>/dev/null || echo "?")
                      if (( $(echo "$BUDGET < 0" | bc -l 2>/dev/null || echo 0) )); then
                        add_fail "SLO $svc: 🔴 BUDGET EXHAUSTED (${BUDGET_PCT}%)"
                      elif (( $(echo "$BUDGET < 0.10" | bc -l 2>/dev/null || echo 0) )); then
                        add_fail "SLO $svc: 🔴 RED — ${BUDGET_PCT}% remaining"
                      elif (( $(echo "$BUDGET < 0.25" | bc -l 2>/dev/null || echo 0) )); then
                        add_warn "SLO $svc: 🟠 ORANGE — ${BUDGET_PCT}% remaining"
                      elif (( $(echo "$BUDGET < 0.50" | bc -l 2>/dev/null || echo 0) )); then
                        add_warn "SLO $svc: 🟡 YELLOW — ${BUDGET_PCT}% remaining"
                      else
                        add_pass "SLO $svc: 🟢 GREEN — ${BUDGET_PCT}% remaining"
                      fi
                    fi
                  done
                  
                  # ═══════════════════════════════════════
                  # CHECK 5: Node Health
                  # ═══════════════════════════════════════
                  NOT_READY=$(kubectl get nodes --no-headers 2>/dev/null | grep -c "NotReady" || true)
                  TOTAL_NODES=$(kubectl get nodes --no-headers 2>/dev/null | wc -l || echo 0)
                  
                  if [ "$NOT_READY" -gt 0 ]; then
                    NODE_LIST=$(kubectl get nodes --no-headers | grep "NotReady" | awk '{print $1}' | tr '\n' ', ')
                    add_fail "Nodes: $NOT_READY/$TOTAL_NODES NotReady: $NODE_LIST"
                  else
                    add_pass "Nodes: All $TOTAL_NODES Ready"
                  fi
                  
                  # Check node conditions (memory/disk pressure)
                  PRESSURED=$(kubectl get nodes -o json | \
                    jq '[.items[] | select(
                      (.status.conditions[] | select(.type == "MemoryPressure" and .status == "True")) or
                      (.status.conditions[] | select(.type == "DiskPressure" and .status == "True"))
                    ) | .metadata.name] | length' 2>/dev/null || echo 0)
                  
                  if [ "$PRESSURED" -gt 0 ]; then
                    add_warn "Nodes: $PRESSURED nodes under memory/disk pressure"
                  fi
                  
                  # ═══════════════════════════════════════
                  # CHECK 6: Pending Pods
                  # ═══════════════════════════════════════
                  PENDING=$(kubectl get pods -A --field-selector=status.phase=Pending \
                    --no-headers 2>/dev/null | wc -l || echo 0)
                  
                  if [ "$PENDING" -gt 5 ]; then
                    PENDING_LIST=$(kubectl get pods -A --field-selector=status.phase=Pending \
                      --no-headers | awk '{print $1"/"$2}' | head -10 | tr '\n' ', ')
                    add_fail "Pods: $PENDING stuck Pending: $PENDING_LIST"
                  elif [ "$PENDING" -gt 0 ]; then
                    add_warn "Pods: $PENDING Pending (may be normal for batch jobs)"
                  else
                    add_pass "Pods: None stuck Pending"
                  fi
                  
                  # CrashLoopBackOff
                  CRASHLOOP=$(kubectl get pods -A --no-headers 2>/dev/null | grep "CrashLoopBackOff" | wc -l || echo 0)
                  if [ "$CRASHLOOP" -gt 0 ]; then
                    CRASH_LIST=$(kubectl get pods -A --no-headers | grep "CrashLoopBackOff" | \
                      awk '{print $1"/"$2}' | head -10 | tr '\n' ', ')
                    add_fail "Pods: $CRASHLOOP in CrashLoopBackOff: $CRASH_LIST"
                  else
                    add_pass "Pods: No CrashLoopBackOff"
                  fi
                  
                  # ═══════════════════════════════════════
                  # CHECK 7: Jenkins Health
                  # ═══════════════════════════════════════
                  JENKINS_STATUS=$(curl -sf -o /dev/null -w "%{http_code}" \
                    "http://jenkins.ci:8080/login" 2>/dev/null || echo "000")
                  
                  if [ "$JENKINS_STATUS" = "200" ] || [ "$JENKINS_STATUS" = "302" ]; then
                    QUEUE_DEPTH=$(curl -sf "http://jenkins.ci:8080/queue/api/json" | \
                      jq '.items | length' 2>/dev/null || echo "-1")
                    if [ "$QUEUE_DEPTH" -gt 10 ]; then
                      add_warn "Jenkins: Queue depth $QUEUE_DEPTH (>10 — agents may be constrained)"
                    elif [ "$QUEUE_DEPTH" != "-1" ]; then
                      add_pass "Jenkins: Healthy, queue depth $QUEUE_DEPTH"
                    else
                      add_pass "Jenkins: Responding (queue API unavailable)"
                    fi
                  else
                    add_fail "Jenkins: Not responding (HTTP $JENKINS_STATUS)"
                  fi
                  
                  # ═══════════════════════════════════════
                  # CHECK 8: Karpenter Health
                  # ═══════════════════════════════════════
                  KARPENTER_PODS=$(kubectl get pods -n karpenter --no-headers 2>/dev/null | grep "Running" | wc -l || echo 0)
                  if [ "$KARPENTER_PODS" -lt 2 ]; then
                    add_fail "Karpenter: Only $KARPENTER_PODS/2 pods running — node provisioning may fail"
                  else
                    add_pass "Karpenter: $KARPENTER_PODS pods running"
                  fi
                  
                  # Check for stuck NodeClaims
                  STUCK_CLAIMS=$(kubectl get nodeclaims --no-headers 2>/dev/null | \
                    grep -v "Ready" | wc -l || echo 0)
                  if [ "$STUCK_CLAIMS" -gt 0 ]; then
                    add_warn "Karpenter: $STUCK_CLAIMS NodeClaims not Ready"
                  fi
                  
                  # ═══════════════════════════════════════
                  # CHECK 9: Linkerd Health
                  # ═══════════════════════════════════════
                  LINKERD_PODS=$(kubectl get pods -n linkerd --no-headers 2>/dev/null | grep -c "Running" || echo 0)
                  LINKERD_EXPECTED=3  # identity, destination, proxy-injector
                  if [ "$LINKERD_PODS" -lt "$LINKERD_EXPECTED" ]; then
                    add_fail "Linkerd: Only $LINKERD_PODS/$LINKERD_EXPECTED control plane pods"
                  else
                    add_pass "Linkerd: Control plane healthy ($LINKERD_PODS pods)"
                  fi
                  
                  # Check Linkerd identity cert expiry
                  LINKERD_CERT_EXPIRY=$(kubectl get secret linkerd-identity-issuer -n linkerd \
                    -o jsonpath='{.data.tls\.crt}' 2>/dev/null | base64 -d | \
                    openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
                  if [ -n "$LINKERD_CERT_EXPIRY" ]; then
                    CERT_EPOCH=$(date -d "$LINKERD_CERT_EXPIRY" +%s 2>/dev/null || echo 0)
                    DAYS_LEFT=$(( (CERT_EPOCH - $(date +%s)) / 86400 ))
                    if [ "$DAYS_LEFT" -lt 30 ]; then
                      add_fail "Linkerd: Identity cert expires in ${DAYS_LEFT}d — SEE RUNBOOK-PLAT-006"
                    elif [ "$DAYS_LEFT" -lt 90 ]; then
                      add_warn "Linkerd: Identity cert expires in ${DAYS_LEFT}d"
                    else
                      add_pass "Linkerd: Identity cert valid for ${DAYS_LEFT}d"
                    fi
                  fi
                  
                  # ═══════════════════════════════════════
                  # CHECK 10: External Secrets Operator
                  # ═══════════════════════════════════════
                  FAILED_ES=$(kubectl get externalsecrets -A -o json 2>/dev/null | \
                    jq '[.items[] | select(.status.conditions[]? | select(.type == "Ready" and .status != "True"))] | length' \
                    2>/dev/null || echo "-1")
                  
                  if [ "$FAILED_ES" = "-1" ]; then
                    add_warn "External Secrets: Unable to query"
                  elif [ "$FAILED_ES" -gt 0 ]; then
                    ES_LIST=$(kubectl get externalsecrets -A -o json | \
                      jq -r '[.items[] | select(.status.conditions[]? | select(.type == "Ready" and .status != "True")) | 
                      "\(.metadata.namespace)/\(.metadata.name)"] | join(", ")')
                    add_fail "External Secrets: $FAILED_ES not syncing: $ES_LIST"
                  else
                    TOTAL_ES=$(kubectl get externalsecrets -A --no-headers 2>/dev/null | wc -l || echo 0)
                    add_pass "External Secrets: All $TOTAL_ES synced"
                  fi
                  
                  # ═══════════════════════════════════════
                  # SEND REPORT
                  # ═══════════════════════════════════════
                  TOTAL_CHECKS=$((FAILURES + WARNINGS + $(echo -e "$REPORT" | grep -c "✅")))
                  
                  if [ "$FAILURES" -gt 0 ]; then
                    COLOR="danger"
                    EMOJI="🚨"
                    TITLE="NovaMart Daily Health: ${FAILURES} FAILURES, ${WARNINGS} warnings"
                  elif [ "$WARNINGS" -gt 0 ]; then
                    COLOR="warning"
                    EMOJI="⚠️"
                    TITLE="NovaMart Daily Health: ${WARNINGS} warnings"
                  else
                    COLOR="good"
                    EMOJI="✅"
                    TITLE="NovaMart Daily Health: All checks passed"
                  fi
                  
                  # Send to Slack
                  curl -sf -X POST "$SLACK_WEBHOOK" -H 'Content-type: application/json' \
                    -d "{
                      \"attachments\": [{
                        \"color\": \"$COLOR\",
                        \"title\": \"$EMOJI $TITLE\",
                        \"text\": \"$(echo -e "$REPORT" | sed 's/"/\\"/g')\",
                        \"footer\": \"daily-health-check | $(date -u +%Y-%m-%dT%H:%M:%SZ) | $TOTAL_CHECKS checks\"
                      }]
                    }" || echo "WARNING: Failed to send Slack notification"
                  
                  # TRAP 8 HANDLED: Monitor the monitor.
                  # If this CronJob itself fails, the Job will be marked Failed.
                  # We have a PrometheusRule that alerts on:
                  #   kube_job_status_failed{job_name=~"daily-health-check.*"} > 0
                  # So if the health check stops running, we still get alerted.
                  
                  if [ "$FAILURES" -gt 0 ]; then
                    echo "Health check completed with $FAILURES failures"
                    exit 1  # Job marked Failed → visible in monitoring
                  fi
                  echo "Health check completed: all clear"
              env:
                - name: SLACK_WEBHOOK
                  valueFrom:
                    secretKeyRef:
                      name: platform-ops-secrets
                      key: slack-webhook
              resources:
                requests:
                  cpu: 100m
                  memory: 128Mi
                limits:
                  cpu: 500m
                  memory: 256Mi
          restartPolicy: OnFailure
---
# RBAC for health checker
apiVersion: v1
kind: ServiceAccount
metadata:
  name: platform-health-checker
  namespace: platform-ops
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: platform-health-checker
rules:
  - apiGroups: [""]
    resources: ["nodes", "pods", "services", "namespaces"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["get", "list"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list"]
  - apiGroups: ["cert-manager.io"]
    resources: ["certificates"]
    verbs: ["get", "list"]
  - apiGroups: ["external-secrets.io"]
    resources: ["externalsecrets"]
    verbs: ["get", "list"]
  - apiGroups: ["karpenter.sh"]
    resources: ["nodeclaims", "nodepools"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-health-checker
subjects:
  - kind: ServiceAccount
    name: platform-health-checker
    namespace: platform-ops
roleRef:
  kind: ClusterRole
  name: platform-health-checker
  apiGroup: rbac.authorization.k8s.io
---
# Alert on health check CronJob failure (TRAP 8: monitor the monitor)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: health-check-monitoring
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: operations.health-check
      rules:
        - alert: DailyHealthCheckFailed
          expr: |
            kube_job_status_failed{namespace="platform-ops", job_name=~"daily-health-check.*"} > 0
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Daily health check CronJob failed"
            description: |
              The automated daily health check found issues or failed to run.
              Check the job logs: kubectl logs -n platform-ops job/daily-health-check-<id>
            runbook_url: "https://runbooks.novamart.internal/ops/health-check-failure"

        - alert: DailyHealthCheckMissing
          expr: |
            time() - kube_cronjob_status_last_successful_time{namespace="platform-ops", cronjob="daily-health-check"} > 90000
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Daily health check hasn't run in > 25 hours"
            description: |
              The daily health check CronJob hasn't completed successfully in over 25 hours.
              Check: kubectl get cronjob daily-health-check -n platform-ops
              Check: kubectl get jobs -n platform-ops -l job-name=daily-health-check
```

## 6.3 Platform Upgrade Runbook (EKS Version)

```markdown
# RUNBOOK-OPS-001: EKS Version Upgrade

## Metadata
- **Severity:** Planned maintenance (SEV3 if issues arise)
- **Duration:** 4-6 hours (including soak time)
- **Frequency:** Every 4-5 months (follows K8s release cycle)
- **Owner:** Platform Engineering
- **Approval required:** Engineering Manager + VP Engineering sign-off on production upgrade
- **Blackout periods:** Nov 15 - Jan 2 (holiday), major sale events, during active incidents

## Pre-Upgrade (Week -2 to Week -1)

### Week -2: Assessment and Planning
```bash
# 1. Check available EKS versions:
aws eks describe-addon-versions --kubernetes-version 1.30 \
  --query 'addons[*].{Name:addonName,Versions:addonVersions[0].addonVersion}' --output table

# 2. Review K8s release notes for breaking changes:
# https://kubernetes.io/blog/ (release announcement)
# https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.30.md

# 3. Check deprecated APIs in our cluster:
# Install pluto: https://github.com/FairwindsOps/pluto
pluto detect-all-in-cluster --target-versions k8s=v1.30.0

# If any deprecated APIs found:
# - Identify which Helm charts/manifests use them
# - Update BEFORE upgrading EKS
# - Test in staging first

# 4. Check add-on compatibility matrix:
# https://docs.aws.amazon.com/eks/latest/userguide/managing-add-ons.html
# Document compatible versions for each add-on at new K8s version.

# 5. Check platform component compatibility:
COMPONENTS=(
  "Linkerd:https://linkerd.io/releases/"
  "ArgoCD:https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#supported-versions"
  "Karpenter:https://karpenter.sh/docs/upgrading/compatibility/"
  "cert-manager:https://cert-manager.io/docs/releases/"
  "Kyverno:https://kyverno.io/docs/installation/#compatibility-matrix"
  "Prometheus Operator:https://github.com/prometheus-operator/prometheus-operator#compatibility"
)
echo "Manual check required for each:"
printf '%s\n' "${COMPONENTS[@]}"

# 6. Create upgrade Jira ticket with:
#    - Target version
#    - Add-on version updates needed
#    - Platform component updates needed
#    - Deprecated API remediation list
#    - Staging upgrade date
#    - Production upgrade date
#    - Rollback plan
```

### Week -1: Staging Upgrade
```bash
# 1. Upgrade staging EKS control plane:
cd terraform/environments/staging/eks
# Update cluster_version in variables:
# cluster_version = "1.30"
terraform plan
# REVIEW: Should show in-place update of aws_eks_cluster, NOT replacement
terraform apply

# 2. Wait for control plane upgrade (10-20 min):
aws eks wait cluster-active --name novamart-staging --region us-east-1

# 3. Verify control plane:
kubectl version --short
# Server Version should show new version

# 4. Upgrade add-ons to compatible versions:
# In terraform:
# vpc_cni_version    = "v1.18.0"
# coredns_version    = "v1.11.1"
# kube_proxy_version = "v1.30.0-eksbuild.1"
# ebs_csi_version    = "v1.30.0"
terraform plan -target=module.eks_addons
terraform apply -target=module.eks_addons

# 5. Verify add-ons:
kubectl get pods -n kube-system
# All pods should restart with new versions

# 6. Upgrade node group (launch template update):
# Update AMI to EKS-optimized for new version:
terraform plan -target=module.eks.aws_launch_template.system
terraform apply -target=module.eks.aws_launch_template.system
# This triggers rolling replacement of system node group.

# 7. For Karpenter-managed nodes:
# Karpenter drift detection will notice nodes running old kubelet.
# It will automatically drain and replace them.
# Monitor:
kubectl get nodeclaims -w
# Or force drift: annotate NodePool to trigger immediate replacement

# 8. Run full test suite:
./scripts/run-integration-tests.sh --env=staging

# 9. Run chaos experiments in staging:
kubectl apply -f manifests/chaos/experiments/ -n chaos-testing

# 10. Soak for 3-5 business days. Monitor:
#     - SLO compliance
#     - Error rates
#     - Latency changes
#     - Any new warnings in logs
```

### Production Upgrade Day

#### Pre-Flight Checklist (1 hour before)
```bash
# [ ] VP Engineering has approved (written approval in Jira ticket)
# [ ] Staging soak period complete with no issues
# [ ] Current production SLOs all GREEN (no pre-existing budget pressure)
# [ ] On-call staffing: 2 engineers available (primary + secondary)
# [ ] Maintenance window communicated to all teams 48h ago
# [ ] Status page: "Scheduled maintenance" posted
# [ ] ArgoCD sync window allows platform changes
# [ ] All PDBs verified:
kubectl get pdb -A

# [ ] RDS snapshots taken (safety net):
for db in novamart-payment-prod novamart-order-prod novamart-user-prod; do
  aws rds create-db-snapshot \
    --db-instance-identifier $db \
    --db-snapshot-identifier "${db}-pre-upgrade-$(date +%Y%m%d)"
done

# [ ] Disable non-critical CronJobs:
kubectl patch cronjob reporting-daily -n batch-jobs -p '{"spec":{"suspend":true}}'
kubectl patch cronjob data-cleanup -n batch-jobs -p '{"spec":{"suspend":true}}'

# [ ] Verify Karpenter drift detection is configured:
kubectl get nodepool default -o yaml | grep -A5 disruption

# [ ] Post in #engineering:
# "🔧 EKS upgrade to 1.30 starting at [time]. Expected duration: 2-3 hours.
#  No action required from dev teams. Platform team monitoring."
```

#### Phase 1: Control Plane Upgrade (15-20 min)
```bash
cd terraform/environments/production/eks

# 1. Apply control plane upgrade:
terraform plan -out=eks-upgrade.plan
# REVIEW CAREFULLY:
# - aws_eks_cluster.main: update in-place (NOT replace!)
# - No unexpected resource changes

terraform apply eks-upgrade.plan

# 2. Monitor:
aws eks describe-cluster --name novamart-prod \
  --query 'cluster.{Status:status,Version:version,PlatformVersion:platformVersion}'
# Wait for status: ACTIVE, version: 1.30

# 3. Verify API server:
kubectl cluster-info
kubectl get nodes  # Nodes still on old version — expected
```

#### Phase 2: Add-on Upgrades (5-10 min)
```bash
# Order matters: VPC CNI → CoreDNS → kube-proxy → EBS CSI
terraform plan -target=module.eks_addons -out=addons-upgrade.plan
terraform apply addons-upgrade.plan

# Verify each:
kubectl get pods -n kube-system -l k8s-app=aws-node -o jsonpath='{.items[0].spec.containers[0].image}'
kubectl get pods -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[0].spec.containers[0].image}'
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o jsonpath='{.items[0].spec.containers[0].image}'
kubectl get pods -n kube-system -l app=ebs-csi-controller -o jsonpath='{.items[0].spec.containers[0].image}'
```

#### Phase 3: Node Upgrade (30-60 min)
```bash
# System node group (managed):
terraform plan -target=module.eks.aws_eks_node_group.system -out=nodegroup-upgrade.plan
terraform apply nodegroup-upgrade.plan
# This triggers rolling replacement.

# Monitor system nodes:
watch 'kubectl get nodes -l node-role.kubernetes.io/system=true -o wide'
# Watch for: new nodes appearing, old nodes cordoned, old nodes drained

# Karpenter-managed nodes (application workloads):
# Karpenter drift detection handles this automatically.
# Monitor progress:
watch 'kubectl get nodeclaims -o custom-columns=NAME:.metadata.name,STATE:.status.conditions[0].type,AGE:.metadata.creationTimestamp'

# If Karpenter is too slow, force it:
# Option: set expireAfter to trigger immediate replacement
kubectl patch nodepool default --type=merge -p '{"spec":{"disruption":{"expireAfter":"1h"}}}'
# Revert after upgrade: set back to normal value (720h or whatever)

# CRITICAL: Monitor pod disruption during node rolling:
kubectl get events -A --field-selector reason=Evicted --sort-by='.lastTimestamp' -w
kubectl get pods -A | grep -v Running | grep -v Completed

# This is the riskiest phase. If something goes wrong:
# 1. Stop the rolling update by reverting the launch template
# 2. Uncordon any cordoned nodes
# 3. Assess impact
```

#### Phase 4: Post-Upgrade Verification (15 min)
```bash
# 1. All nodes on new version:
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion,READY:.status.conditions[?(@.type=="Ready")].status'
# ALL should show v1.30.x and Ready=True

# 2. System pods healthy:
kubectl get pods -n kube-system
kubectl get pods -n linkerd
kubectl get pods -n monitoring
kubectl get pods -n argocd
kubectl get pods -n kyverno
kubectl get pods -n cert-manager

# 3. ArgoCD all apps synced:
argocd app list | grep -v "Synced.*Healthy"
# Should return nothing (all synced + healthy)

# 4. SLO check:
curl -sf "http://prometheus.monitoring:9090/api/v1/query" \
  --data-urlencode 'query=sli:availability:error_ratio:rate5m' | \
  jq '.data.result[] | {service: .metric.slo_service, error_rate: .value[1]}'
# All error rates should be at baseline

# 5. Run platform validation:
./scripts/validate-platform.sh

# 6. Run smoke tests:
kubectl run smoke-test --rm -it --image=novamart/smoke-tests:latest -n platform-ops -- \
  /run-tests.sh --env=production --suite=critical

# 7. Post in #engineering:
# "✅ EKS upgrade to 1.30 complete. All services healthy. Monitoring."
```

#### Phase 5: Monitoring Period (2 hours)
```bash
# Monitor for at least 2 hours before declaring success.
# Watch for:
# - Latency regressions (new kubelet version can change behavior)
# - Memory usage changes (new features may change defaults)
# - Log volume changes (new deprecation warnings)
# - Any new alerts firing

# After 2 hours with no issues:
# 1. Re-enable CronJobs:
kubectl patch cronjob reporting-daily -n batch-jobs -p '{"spec":{"suspend":false}}'
kubectl patch cronjob data-cleanup -n batch-jobs -p '{"spec":{"suspend":false}}'

# 2. Update status page: "Scheduled maintenance completed successfully."

# 3. Close Jira ticket with:
#    - Actual duration
#    - Any issues encountered
#    - Any follow-up actions needed
```

#### Rollback Plan
```
CONTROL PLANE:
  - EKS control plane CANNOT be downgraded. This is irreversible.
  - This is why we soak in staging for 3-5 days first.
  - If the control plane upgrade itself fails: AWS Support.

ADD-ONS:
  - Can be reverted to previous versions via Terraform.
  - terraform apply with old version numbers.

NODES:
  - Revert launch template to previous version.
  - New nodes will launch with old AMI/kubelet version.
  - K8s supports N-1 kubelet skew, so old kubelet talks to new control plane fine.

APPLICATION ISSUES:
  - If a specific application breaks on new K8s version:
  - Fix the application (deprecated API, changed behavior)
  - Or run with the kubelet skew until fix is ready
```
```

## 6.4 Toil Tracking Template

```markdown
# docs/operations/toil-tracking.md

# NovaMart Toil Tracking

## What is Toil?
Work that is: manual, repetitive, automatable, tactical, no lasting value, scales linearly with service growth.

## Weekly Toil Log

Each on-call engineer fills this out at the end of their rotation:

| Date | Task | Duration | Category | Automatable? | Automation Priority |
|------|------|----------|----------|-------------|-------------------|
| Mon | Restarted payment-service pod (memory leak) | 15 min | Incident | Yes — fix leak or auto-restart | P1 |
| Mon | Cleared /tmp on jenkins-agent-3 | 10 min | Maintenance | Yes — tmpwatch CronJob | P2 |
| Tue | Manually approved 3 canary promotions | 20 min | Deploy | Yes — AnalysisTemplate auto-promote | P2 |
| Wed | Rotated staging SSL cert manually | 30 min | Security | Yes — cert-manager should handle this | P1 |
| Thu | Investigated false alarm (HighMemoryUsage) | 25 min | Alert noise | Yes — tune alert threshold | P2 |
| Fri | On-call handoff documentation | 20 min | Process | Partially — template exists but manual | P3 |

### Summary
| Metric | Value | Target |
|--------|-------|--------|
| Total on-call hours | 168 (1 week) | - |
| Hours spent on toil | 22 | < 84 (50%) |
| Toil percentage | 13% | < 50% |
| Pages received | 7 | < 14 (2/day) |
| Actionable pages | 5 | - |
| False alarm pages | 2 | 0 |
| Wake-up pages (overnight) | 1 | 0 |

## Monthly Toil Dashboard

| Month | Toil % | Pages/week | False alarms | Top toil source | Automation completed |
|-------|--------|-----------|--------------|-----------------|---------------------|
| Jan | 35% | 12 | 4 | Memory leak restarts | Auto-restart CronJob |
| Feb | 28% | 8 | 2 | Cert rotation | cert-manager deployed |
| Mar | 20% | 6 | 1 | Canary approvals | AnalysisTemplate auto |
| Apr | 15% | 5 | 0 | Manual scaling | HPA + scheduled scaling |
| Target | < 50% | < 14 | 0 | - | 1 automation/month |

## Toil Reduction Backlog

| Priority | Toil Source | Current Impact | Automation Approach | Estimated Effort | Jira |
|----------|-----------|----------------|--------------------|-----------------|----|
| P1 | Payment pod restarts | 2hr/week | Fix memory leak OR auto-rolling-restart CronJob | 2 days | PLAT-234 |
| P1 | False alarm alerts | 1hr/week | Tune thresholds, add hysteresis | 1 day | PLAT-235 |
| P2 | Manual canary promotion | 1hr/week | AnalysisTemplate with Prometheus checks | 3 days | PLAT-236 |
| P2 | ECR image cleanup | 30min/month | Lifecycle policy (already done) | Done | PLAT-200 |
| P3 | On-call handoff docs | 20min/week | Structured template + auto-generated status | 1 day | PLAT-237 |
| P3 | Monthly access review | 2hr/month | RBAC audit script | 2 days | PLAT-238 |

## Rules
1. If toil > 50% for 2 consecutive weeks → Platform Lead escalates to eng manager
2. Every sprint: at least 1 toil reduction item prioritized
3. If a manual task is performed 3+ times → file automation ticket automatically
4. False alarm pages are P1 by default — they erode on-call trust
```

## 6.5 On-Call Handoff Checklist

```markdown
# docs/operations/oncall-handoff-checklist.md

# On-Call Handoff Checklist

## When: Every Monday 10:00 AM (after SLO review meeting)

## Outgoing On-Call Provides:

### Active Issues
- [ ] Any open incidents? Link to incident channel.
- [ ] Any services in ORANGE/RED budget state? Which ones, why?
- [ ] Any temporary workarounds in place? (e.g., disabled retry, manual scaling)
- [ ] Any config changes applied manually that need to be made permanent?

### Upcoming Events
- [ ] Scheduled deployments this week?
- [ ] Planned maintenance windows?
- [ ] Sales events or traffic spikes expected?
- [ ] Terraform changes pending apply? (infra PRs open?)

### Known Issues
- [ ] Flaky alerts? (which ones, what's the fix status?)
- [ ] Known intermittent failures? (e.g., "payment-service leaks memory, restart every 3 days")
- [ ] Any vendor issues? (AWS health events, Stripe degradation?)

### Page Summary
- [ ] Total pages received this week: ___
- [ ] Actionable pages: ___
- [ ] False alarms: ___ (Jira tickets filed for each?)
- [ ] Overnight wake-ups: ___
- [ ] Toil log completed and filed?

### Tips for Incoming
- [ ] Anything unusual to watch for?
- [ ] Any specific runbooks to review?

## Incoming On-Call Verifies:

### Access
- [ ] PagerDuty shows me as primary on-call for this week
- [ ] Received test notification from PagerDuty (push + SMS + call)
- [ ] VPN connection working
- [ ] kubectl access to production cluster: `kubectl get nodes`
- [ ] kubectl access to staging cluster
- [ ] kubectl access to DR cluster
- [ ] AWS Console access working
- [ ] Grafana accessible: https://grafana.novamart.internal
- [ ] ArgoCD accessible: https://argocd.novamart.internal
- [ ] Jenkins accessible: https://jenkins.novamart.internal
- [ ] Runbook site accessible: https://runbooks.novamart.internal

### Context
- [ ] Reviewed SLO dashboard — current budget states noted
- [ ] Reviewed today's daily health check report
- [ ] Know who secondary on-call is this week: ___
- [ ] Know IC escalation path (PagerDuty handles, but know names)
- [ ] Know about any ongoing issues from handoff above

### Emergency
- [ ] Know how to declare an incident: `/incident create` or manual channel creation
- [ ] Know where runbooks are
- [ ] Know how to reach engineering manager (phone number, not just Slack)
- [ ] Have AWS Console bookmarked for CloudWatch emergency dashboard

## Handoff Confirmation
```
Posted in #platform-ops:
"🔄 On-call handoff complete.
Outgoing: @outgoing_engineer
Incoming: @incoming_engineer (primary), @secondary_engineer (secondary)
Active issues: [list or "none"]
Notes: [any special callouts]"
```
```

---

# DELIVERABLE 7: Design Decisions

```markdown
# docs/operations/design-decisions.md

# Operations Design Decisions

---

## DD-OPS-001: SLO Target Rationale

### Decision
Set per-service SLO targets as defined in the SLO Contract Table, with payment at 99.99%
and most other services at 99.9%–99.95%.

### Context
SLO targets must balance reliability investment with feature velocity. Too aggressive =
constant freezes. Too loose = no reliability signal.

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| 99.99% for everything | Uniform standard, simple | Impossible for non-critical services. Permanent feature freeze for search/notifications. |
| 99.9% for everything | Easy to achieve, high velocity | Payment at 99.9% = 43 min/month downtime. Unacceptable for revenue-critical service. |
| **Tiered targets (chosen)** | Matches business criticality. Tight where money flows, relaxed where UX tolerates. | More complex to manage. Different teams have different standards. |
| No SLOs, just monitor | Maximum flexibility | No accountability. "Everything is an emergency" culture returns. |

### Rationale
- **Payment at 99.99%:** Revenue directly impacted. 4.32 min/month is tight but achievable
  with Multi-AZ + canary deploys + Linkerd. We validated this is achievable in staging
  over a 90-day observation period.
- **API Gateway at 99.95%:** Fronts all traffic. 21.6 min/month gives deploy headroom.
  The gateway itself is simple (routing + auth) — failures usually come from backends.
- **Cart/Order at 99.9%–99.95%:** Important but not "money is literally burning" critical.
  Cart loss = user re-adds items (annoying, not catastrophic).
- **Search at 99.9%:** Degraded search reduces conversion but users can still browse categories.
  1% latency SLO is loose because search latency is highly variable by query complexity.
- **Notification at 99.0%:** "Your order shipped" arriving 30 min late is barely noticed.
  Tight SLO here would waste reliability investment.
- **Platform internal SLOs (7-day window):** Shorter window = faster signal on platform
  degradation. Internal customers (engineers) need quick feedback.

### Review Schedule
Quarterly. First review after 90 days of data collection.

---

## DD-OPS-002: On-Call Rotation Structure

### Decision
6-person rotation, 1-week shifts, primary + secondary, Monday 10 AM handoff.

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| Follow-the-sun (multi-timezone) | No overnight pages | Requires 3+ team locations. We don't have this. |
| 2-day rotations | Less burnout per shift | Too many handoffs. Context loss. Handoff overhead. |
| **1-week rotations (chosen)** | Good balance of context retention and rest | Full week of interrupted sleep possible (mitigated by secondary). |
| 2-week rotations | Maximum context | Too long. Burnout risk. Senior engineers resist. |

### Rationale
- 6 engineers × 1-week = each person on-call ~8.7 weeks/year. Sustainable.
- Monday handoff aligns with SLO review meeting — natural sync point.
- Secondary on-call = automatic backup. If primary is overwhelmed (> 3 pages in one night),
  secondary takes over per policy.
- Burnout prevention: compensatory time, toil tracking, alert quality reviews.

### Metrics Tracked
- Pages per on-call week (target: < 14)
- Overnight wake-ups per week (target: < 2)
- Toil percentage (target: < 50%)
- If any metric exceeded for 3 consecutive weeks → escalate to eng manager.

---

## DD-OPS-003: DR Strategy — Why Not Active-Active Multi-Region

### Decision
Warm standby DR in us-west-2 with manual failover. NOT active-active.

### Alternatives Considered

| Option | Monthly Cost | RTO | RPO | Complexity |
|--------|-------------|-----|-----|------------|
| No DR (single region) | $0 | ∞ | ∞ | None |
| **Warm standby (chosen)** | ~$15K (30% of primary) | 30-60 min | 0-5 min | Medium |
| Hot standby (pilot light) | ~$8K (20%) | 60-120 min | 5-15 min | Low-Medium |
| Active-active multi-region | ~$80K (100% duplication) | ~0 | 0 | Very High |

### Rationale
- **Active-active rejected because:**
  1. **Data consistency:** Multi-region writes require conflict resolution.
     Payment data with conflicting writes = financial integrity risk.
     CRDTs or last-writer-wins are unacceptable for payment records.
  2. **Cost:** Full duplication of compute, databases (with Global Aurora or DynamoDB Global Tables),
     and observability = ~$80K/month additional. Not justified by our RTO requirements.
  3. **Complexity:** Active-active introduces split-brain risks, cross-region latency in
     synchronous calls, and dramatically more complex debugging. Our team of 6 cannot
     operationally manage this level of complexity safely.
  4. **Historical risk:** In 3 years, NovaMart has experienced 0 full-region AWS outages.
     AZ failures: 2 (handled automatically). The probability doesn't justify the cost.

- **Warm standby chosen because:**
  1. 30-60 min RTO is acceptable. Payment SLO budget is 4.32 min/month, BUT a region
     failure is an extraordinary event. We'd invoke the SLO exception clause (external
     factor) and communicate to customers.
  2. $15K/month is 0.009% of revenue. Reasonable insurance.
  3. Cross-region read replicas give us near-zero RPO for critical data.
  4. Quarterly DR exercises validate the failover process.

### Future Consideration
If NovaMart expands to EU (GDPR data residency) or APAC, active-active becomes necessary
for latency reasons, not just DR. At that point, re-evaluate with Global Aurora + DynamoDB
Global Tables for the data tier, and invest in cross-region deployment tooling.

---

## DD-OPS-004: Chaos Engineering Tool Choice

### Decision
Chaos Mesh (self-hosted, K8s-native CRDs).

### Alternatives Considered

| Tool | Cost | K8s Native | AWS Experiments | Ease of Use | Control |
|------|------|-----------|----------------|-------------|---------|
| **Chaos Mesh (chosen)** | Free (OSS) | ✅ CRDs | Limited (pod/network level) | Medium | High |
| Litmus Chaos | Free (OSS) | ✅ CRDs | Good (AWS SSM) | Medium | High |
| Gremlin | ~$5K/month | Via agent | ✅ Full AWS | Easy (GUI) | Medium |
| AWS FIS | Pay per use | No (API-based) | ✅ Native | Easy | Limited to AWS |
| tc/iptables | Free | No | No | Hard (manual) | Maximum |

### Rationale
- **Chaos Mesh chosen because:**
  1. Free. Budget-friendly for a team that runs chaos monthly, not daily.
  2. CRD-based = GitOps friendly. Experiments are YAML in our repo.
  3. Covers our primary failure modes: pod kill, network delay/partition, stress, I/O fault.
  4. Dashboard for visibility during game days.
  5. Namespaced — can restrict blast radius via RBAC.

- **AWS FIS used additionally for:**
  1. EC2 instance stop/terminate (simulating AZ failure at infrastructure level)
  2. RDS failover trigger
  3. These are AWS-native actions that Chaos Mesh can't do.

- **Gremlin rejected:** Cost not justified for monthly usage. If we move to weekly chaos
  experiments, reconsider.

- **Litmus considered:** Strong alternative. We chose Chaos Mesh due to slightly better
  network chaos capabilities and active CNCF project status.

---

## DD-OPS-005: Runbook Hosting and Format

### Decision
Markdown files in Git repository, served via static site (MkDocs), linked from alerts via
`runbook_url` annotation.

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Git + MkDocs (chosen)** | Version controlled, PR reviews, search, GitOps | Requires deploy for updates. No rich features. |
| Confluence/Wiki | Rich editing, easy for non-engineers | No version control. Stale content. No PR review. Not GitOps. |
| PagerDuty Runbooks | Integrated with alerts | Limited formatting. Vendor lock-in. Not searchable. |
| Notion | Pretty, collaborative | Not Git-backed. Hard to link from alerts. |
| README in each service repo | Close to code | Fragmented. No central search. Hard to find during incident. |

### Rationale
- Runbooks are operational code. They deserve the same rigor as application code:
  version control, peer review, CI validation (link checking, template compliance).
- MkDocs generates a static site with search. Deployed to internal hosting.
- Every alert annotation includes `runbook_url` pointing to the relevant MkDocs page.
- CI validates: no broken internal links, every runbook follows template, no TODO/TBD sections.
- Git blame shows who last updated each runbook and when.

### Structure
```
docs/
  runbooks/
    infrastructure/
      RUNBOOK-INFRA-001.md
      RUNBOOK-INFRA-002.md
      ...
    data-tier/
      RUNBOOK-DATA-001.md
      ...
    platform/
      RUNBOOK-PLAT-001.md
      ...
    application/
      RUNBOOK-APP-001.md
      ...
    security/
      RUNBOOK-SEC-001.md
      ...
    disaster-recovery/
      RUNBOOK-DR-001.md
      ...
  mkdocs.yml
```

### URL Pattern
`https://runbooks.novamart.internal/[category]/RUNBOOK-[CAT]-[NNN]/`

### Update Policy
- After every incident: runbook updated as postmortem action item
- Monthly: 3-5 runbooks walked through for accuracy
- Quarterly: full runbook inventory review
```
