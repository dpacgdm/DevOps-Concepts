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

---

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

```
Let me deliver this at A+ grade. Every trap addressed, every edge case handled, zero TODOs.

---

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

| State     | Budget Remaining | Velocity Impact                          | Deploy Rules                               | Escalation            |
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
            "thresholds
