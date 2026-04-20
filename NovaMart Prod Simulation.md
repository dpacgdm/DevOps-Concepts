# Phase 8: NovaMart Production Simulation

## Welcome to Your First Day on the Job

---

## SIMULATION RULES

```
┌─────────────────────────────────────────────────────────────┐
│                    HOW THIS WORKS                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. I am NovaMart. I am Slack. I am PagerDuty.              │
│     I am your teammates, your manager, your stakeholders.   │
│     I am the infrastructure breaking at 3 AM.               │
│                                                             │
│  2. You respond as YOU — Senior DevOps/Platform Engineer.   │
│     Write what you'd actually DO, SAY, TYPE, RUN.           │
│     Not essays. Actions.                                    │
│                                                             │
│  3. Time moves when you act. If you investigate correctly,  │
│     you get results. If you run the wrong command, you get  │
│     the wrong output. If you miss something, it escalates.  │
│                                                             │
│  4. Incidents don't wait. If you're mid-project and a       │
│     SEV1 fires, you drop everything. Just like real life.   │
│                                                             │
│  5. I will grade your responses on:                         │
│     - Triage speed and accuracy                             │
│     - Communication quality (who you notify, how, when)     │
│     - Technical correctness                                 │
│     - Prioritization under pressure                         │
│     - Follow-through (postmortems, action items)            │
│                                                             │
│  6. Phase 6 topics (Security, Compliance, AWS deep dives)   │
│     WILL appear organically. A security incident is worth   │
│     more than a security lecture.                           │
│                                                             │
│  7. Format for your responses:                              │
│     🔧 ACTIONS — commands you run, things you click        │
│     💬 COMMS — Slack messages, emails, meeting statements  │
│     🧠 THINKING — your internal reasoning (I need this     │
│        to grade your decision-making process)               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## YOUR TEAM

```
Platform Engineering Team:
  ├── Sarah Chen — Engineering Manager (your boss)
  ├── YOU — Senior DevOps/Platform Engineer
  ├── Marcus Webb — Senior DevOps Engineer (your peer, 2 yrs at NovaMart)
  ├── Priya Sharma — DevOps Engineer (mid-level, 8 months)
  ├── Jake Torres — DevOps Engineer (mid-level, 1 year)
  ├── Alex Kim — Junior DevOps Engineer (3 months, eager but green)
  └── Lisa Park — SRE (focuses on SLO/reliability, 1.5 yrs)

Adjacent Teams:
  ├── Payment Team — Lead: David Okafor
  ├── Order Team — Lead: Nina Petrov
  ├── Frontend Team — Lead: Ryan Mitchell
  ├── Security Team — Lead: Aisha Rahman
  ├── QA Team — Lead: Tom Bradley
  └── VP Engineering — James Morrison
```

## ON-CALL THIS WEEK

```
Primary:   YOU (started Monday 9 AM)
Secondary: Marcus Webb
```

---

# 🗓️ WEEK 1 — MONDAY MORNING

**Time: 9:04 AM EST**

You open your laptop. Coffee in hand. First day on primary on-call.

Here's what's waiting:

---

### 📱 PagerDuty — 1 Alert (Triggered 8:47 AM)

```
🔴 FIRING — SEV3
Alert: HighErrorBudgetBurn_OrderService
Summary: Order service 1h burn rate at 3.2x (threshold: 3x)
SLO: 99.9% availability | Error budget remaining: 68%
Dashboard: https://grafana.novamart.internal/d/slo-order-service
Runbook: https://runbooks.novamart.internal/RUNBOOK-APP-001
Triggered: 8:47 AM EST
Acknowledged: NOT YET
```

---

### 💬 Slack — #platform-engineering (3 unread)

**Marcus Webb** — 8:52 AM
> Hey, heads up — I saw the order service alert fire. I poked around before my on-call ended. Error rate ticked up around 8:30 AM. Might be related to that order-service deploy that went out at 8:15 AM via ArgoCD. Canary was green though. I'm handing off to you — let me know if you need context.

**Priya Sharma** — 8:58 AM
> Morning! I have the Linkerd upgrade PR ready for review when someone has time. No rush, just want it in this sprint. PR #347

**Alex Kim** — 9:01 AM
> Hi team, I'm getting a weird error trying to run `terraform plan` on the staging networking module. Getting "Error acquiring the state lock." Is someone running a plan?

---

### 💬 Slack — #incidents (1 unread)

**[PagerDuty Bot]** — 8:47 AM
> 🔴 **Alert Firing:** HighErrorBudgetBurn_OrderService
> Burn rate: 3.2x (1h window)
> On-call: [YOU]
> Status: **Not Acknowledged**

---

### 📧 Email — 1 unread

**From:** Sarah Chen (your manager)
**Subject:** Q1 Platform Priorities — Review Before Wednesday Standup
**Time:** 8:30 AM

> Hey — James (VP Eng) wants our Q1 priorities finalized by Wednesday standup. I've drafted the list below. Review and come with opinions:
>
> 1. EKS upgrade 1.28 → 1.29 (overdue)
> 2. Migrate remaining ELK → Loki
> 3. Implement cost optimization recommendations from last month's review
> 4. Self-service namespace provisioning for dev teams
>
> Also, Phase 2 of the PCI audit is next month. Aisha from Security will need our help documenting controls.
>
> — Sarah

---

### 📋 Jira — Assigned to You

```
PLAT-892 [In Progress] — EKS 1.28 → 1.29 Upgrade Planning
  Due: End of sprint (Friday)
  Description: Create upgrade runbook. Identify breaking changes.
  Status: You accepted this last week.

PLAT-901 [To Do] — Review Linkerd upgrade PR #347
  Due: Wednesday
  Assignee: You (reviewer)

PLAT-910 [To Do] — Q1 Cost Optimization: Implement GP3 migration for EBS volumes
  Due: Next sprint
```

---

**It's 9:04 AM. Your on-call shift started 4 minutes ago.**




**What do you do first? Walk me through your first 30 minutes.**

# Monday 9:04 AM — First 30 Minutes

---

## 🧠 THINKING — Triage (30 seconds mental scan)

```
PRIORITY STACK:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. PagerDuty alert — UNACKED 17 minutes. Order service = revenue.
   3.2x burn rate on 1h window. 68% budget left. Not catastrophic 
   YET, but a 3.2x burn sustained means we blow through remaining 
   budget in ~5 days. Deploy at 8:15 AM is obvious suspect.
   Marcus says canary was green — so either slow-bleed issue that 
   canary window didn't catch, or traffic pattern dependent.
   → ACK IMMEDIATELY, investigate.

2. Alex's state lock — quick unblock. 2-minute fix, but NOT before 
   the alert is handled. He's junior, he'll wait.
   → Reply after I have eyes on dashboards.

3. Priya's PR, Sarah's email, Jira tickets — all Wednesday or later.
   → Park entirely.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 9:04 AM — Ack Alert + Signal to Team

### 🔧 ACTION — Acknowledge PagerDuty alert

Click **Acknowledge** on `HighErrorBudgetBurn_OrderService`.

### 💬 COMMS — #incidents channel

```
🟡 ACK — HighErrorBudgetBurn_OrderService
Investigating now. Order service deploy at 8:15 AM is primary suspect.
Marcus noted canary passed — checking if this is a slow-bleed post-canary.
Will update in 15 min or sooner if severity changes.
— [me], primary on-call
```

---

## 9:05 AM — Eyes on Dashboards

### 🧠 THINKING

Before I touch anything, I need to answer three questions:
1. **What's actually failing?** (5xx? Timeouts? Upstream dependency?)
2. **When exactly did it start?** (Correlate with deploy timestamp)
3. **What changed?** (Deploy diff, config change, traffic pattern)

### 🔧 ACTION — Open Grafana SLO dashboard for order-service

```
https://grafana.novamart.internal/d/slo-order-service
```

Looking at:
- Error rate graph (last 2 hours, zoomed to 8:00–9:05 AM)
- Burn rate graph (1h and 6h windows)
- Request rate (did traffic spike or is it normal Monday morning ramp?)

### 🔧 ACTION — PromQL queries in Grafana Explore

**Query 1: Actual error ratio right now (5m window)**
```promql
sum(rate(http_requests_total{service="order-service", code=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="order-service"}[5m]))
```

**Query 2: Error rate broken down by status code — what's failing?**
```promql
sum by (code) (rate(http_requests_total{service="order-service", code=~"5.."}[5m]))
```

**Query 3: Error rate by endpoint — is it all endpoints or a specific one?**
```promql
sum by (path, code) (rate(http_requests_total{service="order-service", code=~"5.."}[5m]))
```

**Query 4: Request latency — are we seeing timeouts too?**
```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service="order-service"}[5m])))
```

---

**What do I see?**

*(Waiting for infrastructure to tell me what's happening before I take any remediation action.)*

---

## 9:06 AM — While dashboards load, check the deploy

### 🔧 ACTION — Check ArgoCD for order-service deploy details

```bash
kubectl get rollout order-service -n order-prod -o wide
```

```bash
argocd app get order-service-production --show-operation
```

### 🔧 ACTION — Get the current vs previous image

```bash
kubectl get rollout order-service -n order-prod -o jsonpath='{.status.currentStepIndex}' 
```

```bash
kubectl get rs -n order-prod -l app=order-service --sort-by='.metadata.creationTimestamp' -o wide
```

### 🧠 THINKING

If the canary completed and promoted to 100%, I need to see:
- Did it fully promote? Or is it still mid-rollout?
- What's the image diff? (`sha-<previous>` vs `sha-<current>`)
- The canary analysis might have used a short window (5min soak) — if the error is low-frequency (say 0.2% of requests fail), the canary window may not have had enough sample size to catch it.

### 🔧 ACTION — Check the deploy commit in GitOps repo

```bash
cd /tmp && git clone git@bitbucket.org:novamart/gitops-manifests.git --depth 5
cd gitops-manifests
git log --oneline -5 -- apps/order-service/production/
```

This tells me exactly what changed: image tag, config, env vars, resource limits, etc.

---

## 9:08 AM — Quick unblock for Alex (parallel, 90 seconds)

### 💬 COMMS — #platform-engineering (reply to Alex)

```
@Alex — State lock error usually means a previous plan/apply crashed 
without releasing the lock, or someone else is mid-apply.

First check who holds it:
  aws dynamodb get-item \
    --table-name novamart-terraform-locks \
    --key '{"LockID":{"S":"novamart-staging-networking/terraform.tfstate"}}'

If it shows a lock with a timestamp older than 30 min and no one is 
actively running, it's safe to force unlock:
  terraform force-unlock <LOCK-ID>

⚠️ BEFORE you force-unlock: confirm with the team that nobody is 
running an apply. Check the lock's "Who" and "Created" fields first.
If it's from a CI job, check if that Jenkins job is still running.

Don't force-unlock blindly — you could corrupt state if someone IS 
mid-apply. Ping me if the lock holder isn't obvious.
```

### 🧠 THINKING

Good teaching moment for Alex but not going to handhold more than this right now. He has clear steps. If he gets stuck, Jake or Priya can help. My focus stays on the order service alert.

---

## 9:09 AM — Back to investigation. Waiting for query results.

**What do the dashboards and queries show me?**

---

# SIMULATION RESPONSE — Infrastructure Talks Back

## 📊 Here's What the Infrastructure Shows You

### Grafana SLO Dashboard — Order Service (8:00–9:05 AM)

```
ERROR RATE (5m windows)
  ▲
4%│                                          ╭──── 3.8%
  │                                     ╭────╯
3%│                                ╭────╯
  │                           ╭────╯
2%│                      ╭────╯
  │                 ╭────╯
1%│            ╭────╯
  │  ─────────╯  
0%│────────────
  └──────────────────────────────────────────────▶ Time
  8:00  8:10  8:20  8:30  8:40  8:50  9:00  9:05
        ▲                    
     deploy 8:15        Ramp starts ~8:28

BURN RATE:
  1h window:  3.2x  (threshold: 3x) — FIRING
  6h window:  1.1x  (threshold: N/A for this alert tier)

REQUEST RATE (rpm):
  8:00-8:15:   ~2,400 rpm (overnight/early morning baseline)
  8:15-8:30:   ~3,100 rpm (normal morning ramp beginning)  
  8:30-9:00:   ~5,800 rpm (Monday morning peak ramp)
  9:00-9:05:   ~6,200 rpm (still climbing)

ERROR BUDGET: 68% remaining (was 74% at start of day)
```

### Query 1 Results — Current Error Ratio

```
sum(rate(http_requests_total{service="order-service", code=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="order-service"}[5m]))

→ 0.038 (3.8%)
```

### Query 2 Results — Error Breakdown by Status Code

```
sum by (code) (rate(http_requests_total{service="order-service", code=~"5.."}[5m]))

{code="504"} → 3.12 req/s
{code="500"} → 0.08 req/s
```

**Almost entirely 504s. Not 500s.**

### Query 3 Results — Error by Endpoint

```
sum by (path, code) (rate(http_requests_total{service="order-service", code=~"5.."}[5m]))

{path="/api/v1/orders", method="POST", code="504"} → 2.94 req/s
{path="/api/v1/orders/{id}", method="GET", code="504"} → 0.18 req/s
{path="/api/v1/orders", method="POST", code="500"} → 0.08 req/s
```

**POST /api/v1/orders is the problem. Order creation.**

### Query 4 Results — P99 Latency

```
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service="order-service"}[5m])))

→ 4.92s (was 1.8s at 8:00 AM)
```

**P99 has nearly tripled.**

---

### ArgoCD / Rollout Results

```bash
$ kubectl get rollout order-service -n order-prod -o wide

NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
order-service   12        12        12            12          47d

# Rollout is FULLY PROMOTED. Canary completed at 8:26 AM.
# All 12 replicas running new image.
```

```bash
$ kubectl get rs -n order-prod -l app=order-service \
    --sort-by='.metadata.creationTimestamp' -o wide

NAME                      DESIRED   CURRENT   READY   AGE    IMAGE
order-service-6d4f8b2c1   0         0         0       12d    order-service:sha-a4c92f1
order-service-8e7a3d5f9   12        12        12      51m    order-service:sha-e7b31d4
```

Previous image: `sha-a4c92f1`
Current image: `sha-e7b31d4`

### GitOps Repo Log

```bash
$ git log --oneline -5 -- apps/order-service/production/

e7b31d4 2024-01-15 08:14:22 [jenkins] Update order-service production image to sha-e7b31d4
a4c92f1 2024-01-03 14:22:11 [jenkins] Update order-service production image to sha-a4c92f1
```

```bash
$ git diff a4c92f1..e7b31d4 -- apps/order-service/production/

--- a/apps/order-service/production/kustomization.yaml
+++ b/apps/order-service/production/kustomization.yaml
@@ image tag change @@
-  newTag: sha-a4c92f1
+  newTag: sha-e7b31d4

--- a/apps/order-service/production/patches/config.yaml  
+++ b/apps/order-service/production/patches/config.yaml
@@ -12,7 +12,9 @@
     upstream_timeout_ms: 5000
+    inventory_client_timeout_ms: 800
+    inventory_client_retries: 1
     database_pool_size: 20
```

---

### 🔍 So Here's What You're Looking At

```
EVIDENCE SUMMARY:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. 3.8% error rate, climbing with traffic
2. Almost all errors are 504s (timeout), not 500s (app crash)
3. Concentrated on POST /api/v1/orders (order creation)
4. P99 latency tripled (1.8s → 4.9s)
5. Deploy at 8:15 fully promoted by 8:26
6. Deploy changed image AND added new config:
   - inventory_client_timeout_ms: 800  (NEW — was using upstream_timeout_ms: 5000?)
   - inventory_client_retries: 1       (NEW)
7. Error rate started climbing ~8:28 — AFTER promotion, AS traffic ramped
8. Canary passed because traffic was 2,400 rpm during canary window
   Now at 6,200 rpm and climbing
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

**It's 9:10 AM. Error rate is still climbing with traffic. Budget at 68% and dropping.**

**What's your call?**



# Monday 9:10 AM — Diagnosis + Action

---

## 🧠 THINKING — Root Cause Assessment (30 seconds)

```
CONNECTING THE DOTS:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
The picture is clear:

1. 504s = TIMEOUTS, not crashes. Something downstream is slow.
2. POST /api/v1/orders = order creation. This likely calls 
   inventory service to reserve stock.
3. Deploy added inventory_client_timeout_ms: 800 (NEW config).
   Previously this was probably falling through to upstream_timeout_ms: 5000.
   So they CUT the inventory call timeout from 5000ms → 800ms.
4. At low traffic (2,400 rpm during canary), inventory service 
   responds within 800ms. Canary passes.
5. At peak traffic (6,200 rpm and climbing), inventory service 
   p99 likely exceeds 800ms → order-service times out → 504.
6. inventory_client_retries: 1 means no retries — one slow call = fail.

HYPOTHESIS: The new 800ms inventory client timeout is too aggressive 
for peak traffic. Inventory service latency scales with load and 
exceeds 800ms at current volumes.

CONFIDENCE: HIGH. But I need to confirm inventory service latency 
before acting.

SEVERITY ASSESSMENT: 
- 3.8% of order creations failing = real revenue impact
- 6,200 rpm × 0.038 = ~235 failed orders/minute
- Monday morning traffic still climbing toward peak (~8-10K rpm)
- This will get WORSE, not better. Every minute matters.
- Not SEV1 (site isn't down), but this is SEV2 territory — 
  revenue-impacting, customer-facing, getting worse.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 🔧 ACTION 1 — Confirm the hypothesis (9:10 AM, 60 seconds)

**Query 5: Inventory service latency as seen from order-service (upstream)**
```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service="order-service", upstream="inventory-service"}[5m])))
```

**Query 6: Inventory service own latency (is it actually slow, or is it the network?)**
```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service="inventory-service"}[5m])))
```

**Query 7: Inventory service error rate (is IT healthy?)**
```promql
sum(rate(http_requests_total{service="inventory-service", code=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="inventory-service"}[5m]))
```

---

## 💬 COMMS 1 — #incidents (9:11 AM) — Severity Escalation

```
🟠 UPDATE — HighErrorBudgetBurn_OrderService — Escalating to SEV2

IMPACT: ~235 failed order creations/min (3.8% of POST /api/v1/orders). 
Revenue-impacting. Getting worse as Monday traffic ramps.

ROOT CAUSE (high confidence): 
Deploy at 8:15 (sha-e7b31d4) introduced inventory_client_timeout_ms: 800ms.
Previously defaulting to upstream_timeout_ms: 5000ms.
At peak traffic, inventory service p99 likely exceeds 800ms → order-service 
times out → 504.
Canary passed at 2,400 rpm; failing now at 6,200 rpm. Load-dependent.

CONFIRMING: Checking inventory service latency now.

PLAN: If confirmed, immediate config rollback of the timeout value 
(NOT full image rollback — the image change may be fine, it's the 
config that's the problem). ETA fix: ~5 min once confirmed.

@nina.petrov (Order team) — Heads up, your 8:15 deploy introduced 
a new inventory client timeout. I may need to revert that config 
change. Who authored it? We should loop them in.

— [me], primary on-call
```

---

## 🧠 THINKING — Remediation Strategy

```
TWO OPTIONS:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Option A: Full rollback (argo rollouts undo)
  - Reverts image AND config
  - Safe, guaranteed to restore previous behavior
  - But: loses whatever the new image shipped (bug fixes? features?)
  - Nuclear option

Option B: Surgical config fix (revert only the timeout)
  - Change inventory_client_timeout_ms back to 5000 (or remove it 
    to fall through to upstream_timeout_ms default)
  - Keep the new image
  - Faster, more precise
  - Risk: if there are OTHER issues in the new image, we miss them
  
DECISION: If my inventory latency queries confirm the hypothesis, 
Option B — surgical config revert. The error pattern (504 on 
specifically order creation, load-dependent, started at traffic ramp 
not at deploy time) perfectly matches the timeout config change. 
There's no evidence of other issues with the new image.

If queries come back inconclusive → Option A full rollback. Don't 
get clever when the building is on fire.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 🔧 ACTION 2 — Prep the rollback while waiting for queries (9:12 AM)

Not executing yet — preparing so I can fire immediately once confirmed.

```bash
cd /tmp/gitops-manifests
git checkout -b hotfix/order-service-revert-inventory-timeout

# Edit the config patch
cat apps/order-service/production/patches/config.yaml
```

Ready to remove the two new lines:
```yaml
# REMOVE these:
#    inventory_client_timeout_ms: 800
#    inventory_client_retries: 1
```

---

## 🔧 ACTION 3 — While waiting, quick trace check (9:12 AM)

Pull a sample failing trace to confirm the timeout visually:

```
Grafana → Explore → Tempo
TraceQL: {resource.service.name="order-service" && status=error && span.http.url="/api/v1/orders"} | duration > 750ms
```

This should show me a trace where order-service calls inventory-service and the span hits ~800ms then fails.

---

**Waiting for query results + trace to confirm before I execute the fix.**

**What does the infrastructure tell me?**

---

# SIMULATION RESPONSE — Confirmation + Complication

---

## 📊 Query Results

### Query 5 — Inventory Service Latency (as seen FROM order-service)

```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{
  service="order-service", upstream="inventory-service"}[5m])))

→ 1.14s
```

### Query 6 — Inventory Service Own Latency

```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{
  service="inventory-service"}[5m])))

→ 1.08s
```

**Inventory P99 is 1.08–1.14s. The 800ms timeout never had a chance at this traffic level.**

Small delta between "as seen from order-service" (1.14s) and "inventory's own measurement" (1.08s) — network/mesh overhead. Normal. Not the problem.

### Query 7 — Inventory Service Error Rate

```promql
sum(rate(http_requests_total{service="inventory-service", code=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="inventory-service"}[5m]))

→ 0.001 (0.1%)
```

**Inventory is healthy. It's just slow under load.** Not crashing, not erroring. Simply takes >800ms at P99 when traffic is above ~5K rpm.

---

### Tempo Trace Result

```
Trace ID: 7a3f2e1d9c8b4a5f

order-service POST /api/v1/orders ─────────────────────────────────── 812ms [ERROR]
  ├── validate-request ──── 3ms [OK]
  ├── auth-check ────────── 12ms [OK]
  ├── inventory-service /api/v1/inventory/reserve ──────── 801ms [TIMEOUT]
  │     ├── status: DEADLINE_EXCEEDED
  │     └── error.message: "context deadline exceeded (Client.Timeout=800ms)"
  └── [ABANDONED — never reached]
      ├── payment-service /api/v1/payments/authorize
      └── database INSERT orders
```

**Smoking gun.** `Client.Timeout=800ms` is right there in the span. Inventory took 801ms. Order-service killed it at 800ms. The order was never created.

---

## 💬 Slack — #incidents — Nina Petrov Responds (9:13 AM)

**Nina Petrov (Order Team Lead):**
> Thanks for the heads up. The timeout change was authored by @derek.huang. 
> 
> Context: we intentionally tightened it. Last month we had an incident (INC-2847) where inventory service went fully degraded for 40 minutes and order-service threads got saturated waiting on 5-second timeouts. Cascading failure — took down cart service too. Derek's fix was to fail fast with 800ms + no retries so order-service stays healthy even when inventory is slow.
> 
> The problem is 800ms was based on staging load testing which maxes at 3K rpm. Didn't account for production peak.
> 
> Reverting to 5000ms solves today but reintroduces the cascading failure risk from INC-2847. Can we find a middle ground?

---

## 📊 Additional Context — Traffic Projection

```
MONDAY TRAFFIC PATTERN (historical):
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  6:00 AM:   1,200 rpm
  8:00 AM:   2,400 rpm
  9:00 AM:   6,200 rpm  ← YOU ARE HERE
  10:00 AM:  8,500 rpm
  11:00 AM:  9,800 rpm  
  12:00 PM: 10,200 rpm  ← Monday peak
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  
  Current error rate at 6,200 rpm: 3.8%
  Projected error rate at 10,200 rpm: ???% (worse)
```

---

## Current Situation at 9:13 AM

```
┌─────────────────────────────────────────────────────────┐
│  CONFIRMED:                                             │
│  ✓ Hypothesis correct — 800ms timeout too aggressive    │
│  ✓ Inventory p99 = 1.08s at current load                │
│  ✓ Trace proves Client.Timeout=800ms is the killer      │
│  ✓ Traffic still climbing — gets worse every minute      │
│                                                         │
│  COMPLICATION:                                          │
│  ✗ 800ms was INTENTIONAL (prevent cascading failure)    │
│  ✗ Reverting to 5000ms reopens INC-2847 risk            │
│  ✗ Order team wants a middle ground, not a full revert  │
│                                                         │
│  CLOCK:                                                 │
│  • Error budget: ~66% and falling                       │
│  • ~235 failed orders/min, growing with traffic         │
│  • Peak traffic at noon = ~10,200 rpm                   │
│  • Every minute you deliberate costs money              │
└─────────────────────────────────────────────────────────┘
```

**It's 9:13 AM. What do you do?**



# Monday 9:13 AM — Remediation

---

## 🧠 THINKING — Decision (20 seconds)

```
REALITY CHECK:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Nina's concern is valid. 5000ms IS dangerous — INC-2847 proved it.
800ms IS too aggressive — today proves it.

But I am NOT going to design the perfect timeout strategy right now 
at 9:13 AM while orders are failing. That's an afternoon conversation.

RIGHT NOW I need to stop the bleeding with a value that:
  1. Stops the 504s immediately (must be > inventory p99)
  2. Doesn't reintroduce cascading failure risk (must be << 5000ms)
  3. Is explicitly temporary, with a follow-up ticket

THE MATH:
  - Inventory p99 at current load: 1.08s
  - Traffic doubles by noon → p99 will likely be higher
  - Need headroom above p99 but still fail-fast
  - 2000ms gives ~2x headroom over current p99 at peak
  - With retries: 1 retry means worst case 4s wall time, still 
    under the original 5s that caused INC-2847 thread saturation
  - A single retry also masks transient slow calls — legitimate 
    resilience improvement over the previous 5s-no-retry default

PLAN:
  inventory_client_timeout_ms: 2000
  inventory_client_retries: 1

This is the right middle ground for RIGHT NOW.
It stops the bleeding and doesn't reopen INC-2847.
Proper fix = circuit breaker + adaptive timeout. That's a ticket.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 💬 COMMS 1 — #incidents (9:14 AM)

```
🟠 UPDATE — OrderService SEV2 — Fix going in NOW

Root cause confirmed: inventory_client_timeout_ms=800ms too aggressive 
for production traffic. Inventory p99=1.08s at 6,200 rpm. Trace confirms 
DEADLINE_EXCEEDED at exactly 800ms.

Reverting to 5000ms is unsafe (INC-2847 cascading failure risk).

INTERIM FIX (deploying now):
  inventory_client_timeout_ms: 2000  (was 800)
  inventory_client_retries: 1        (keeping — adds transient resilience)

Rationale: 2000ms gives ~2x headroom over current p99. With 1 retry, 
worst case wall time = 4s, still under the 5s threshold that caused 
thread saturation in INC-2847.

This is a TEMPORARY bandaid. Proper fix needs circuit breaker + 
adaptive timeout. Will create follow-up ticket.

Deploying via GitOps now. ETA: ~3 min to rollout.
```

---

## 💬 COMMS 2 — #incidents reply to Nina (9:14 AM)

```
@nina.petrov Agree with your concern — not reverting to 5000ms. 
Pushing 2000ms as interim. Gives ~2x headroom over inventory p99 
at current traffic, and with 1 retry the worst case 4s wall time 
is still well under the 5s that caused INC-2847.

Proper fix: we should add a circuit breaker on the inventory call 
with adaptive timeout/retry. Let's get Derek and me in a room this 
afternoon once this is stabilized. I'll create the follow-up ticket.
```

---

## 🔧 ACTION 1 — Push the config fix (9:15 AM)

```bash
cd /tmp/gitops-manifests

# Already on hotfix/order-service-revert-inventory-timeout branch
vim apps/order-service/production/patches/config.yaml
```

Change:
```yaml
    upstream_timeout_ms: 5000
    inventory_client_timeout_ms: 2000    # was 800 — interim fix, see INC-XXXX
    inventory_client_retries: 1
    database_pool_size: 20
```

```bash
git add apps/order-service/production/patches/config.yaml
git commit -m "[HOTFIX] order-service: increase inventory timeout 800ms→2000ms (SEV2 active)

Root cause: 800ms timeout < inventory p99 (1.08s) at peak traffic.
Interim fix: 2000ms + 1 retry. Worst case 4s < INC-2847 threshold.
Follow-up: circuit breaker implementation needed.

Incident: SEV2 - HighErrorBudgetBurn_OrderService"

git push origin hotfix/order-service-revert-inventory-timeout
```

### 🔧 ACTION 2 — Merge and trigger ArgoCD

```bash
# Fast-track: create PR and merge immediately (hotfix during active incident)
# In Bitbucket: PR → hotfix/order-service-revert-inventory-timeout → main
# Self-approve per incident hotfix policy, merge.
```

### 🔧 ACTION 3 — Monitor ArgoCD sync

```bash
argocd app sync order-service-production --prune=false

argocd app wait order-service-production --health --timeout 180
```

### 🔧 ACTION 4 — Watch the rollout

```bash
kubectl get rollout order-service -n order-prod -w
```

---

## 🔧 ACTION 5 — Validate fix is working (9:18 AM, once rollout healthy)

**Error rate should start dropping:**
```promql
sum(rate(http_requests_total{service="order-service", code=~"5.."}[1m]))
/
sum(rate(http_requests_total{service="order-service"}[1m]))
```
*(Using 1m window for faster signal on the recovery)*

**504s specifically:**
```promql
sum(rate(http_requests_total{service="order-service", code="504"}[1m]))
```

**Inventory call latency still under new timeout:**
```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{
  service="order-service", upstream="inventory-service"}[1m])))
```

---

## 🧠 THINKING — What I'm watching for post-deploy

```
SUCCESS CRITERIA:
  ✓ 504 rate drops to near-zero within 2-3 minutes of rollout
  ✓ P99 latency comes down (won't be 1.8s — inventory is legitimately 
    slower at peak, but should be < 2s not 4.9s)
  ✓ Burn rate drops below 1x on the 1h window
  ✓ Error budget stabilizes

WHAT COULD GO WRONG:
  - 2000ms still too tight at noon peak (10,200 rpm). If inventory 
    p99 scales linearly we'd expect ~1.8s at 10K rpm. Tight but ok.
    I'll keep a close eye through the noon peak.
  - Rollout itself could have issues (unlikely — just config change)
  - There could be a SECOND issue in the new image hiding behind 
    the timeout noise. Watching for non-504 errors after fix.
```

---

**What do the post-fix metrics show? And does the rollout go clean?**

---

# SIMULATION RESPONSE — Fix Deployed

## Rollout Status

```bash
$ kubectl get rollout order-service -n order-prod -w

NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE
order-service   12        12        12            12          # Existing pods
order-service   12        13        1             12          # Canary starts
order-service   12        13        1             13          # Canary healthy
order-service   12        14        4             12          # Step 2: 25%
order-service   12        14        4             14          # Healthy
...
order-service   12        12        12            12          # Fully promoted 9:21 AM
```

Canary analysis passed immediately — error rate on canary pods was 0.02% vs 3.8% on old config. ArgoCD auto-promoted through all steps. Fully rolled out by **9:21 AM**.

---

## 📊 Post-Fix Metrics (9:21–9:30 AM)

### Error Rate (1m window)

```
  ▲
4%│ ────── 3.8%
  │        ╲
3%│         ╲
  │          ╲
2%│           ╲
  │            ╲
1%│             ╲
  │              ╲
0%│               ╰────────────── 0.12%
  └──────────────────────────────────────▶
  9:15  9:18  9:21  9:24  9:27  9:30
              ▲
         full rollout
```

### Key Metrics at 9:30 AM

```
Error rate:           0.12% (down from 3.8%) ✅
504 rate:             ~0.01 req/s (effectively zero) ✅
P99 latency:          2.1s (down from 4.9s, up from pre-deploy 1.8s)
Burn rate (1h):       0.8x (below 1x — budget recovering) ✅
Error budget:         65.2% remaining (stabilized)
Request rate:         7,100 rpm (still climbing toward peak)
Inventory p99:        1.12s (slightly up with traffic, well under 2s timeout)
```

### One Residual Observation

```
500 errors (NOT 504s):

sum(rate(http_requests_total{service="order-service", code="500"}[5m]))
→ 0.08 req/s

This was there BEFORE your fix. Same rate as earlier.
~0.08 req/s out of ~118 req/s = 0.07%. 
Pre-existing, not related to the timeout change.
Within SLO. Not actionable right now.
```

---

## 💬 Slack — #incidents

**Nina Petrov** — 9:22 AM
> Seeing order creation recover on our dashboards. Thanks for the fast fix. Derek and I are free at 2 PM if you want to discuss the circuit breaker approach.

**Lisa Park (SRE)** — 9:25 AM
> Burn rate back under 1x. Nice. I'll keep an eye on the budget through noon peak. Want me to set a temporary 1500ms latency alert on inventory to give early warning if it creeps toward the 2000ms timeout?

---

**It's 9:30 AM. The fire is out. Now what?**

You have:
1. An incident to **close out properly**
2. Lisa's offer on the table
3. The rest of your Monday morning (Alex's lock, Priya's PR, Sarah's email, Jira tickets)
4. 2 PM meeting to schedule with Order team

**Walk me through 9:30 AM → lunch.**



# Monday 9:30 AM → Lunch

---

## 🧠 THINKING — Post-incident + Day Planning (30 seconds)

```
PRIORITY STACK (re-triage):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Close out incident properly (comms, ticket, noon monitoring plan)
2. Accept Lisa's offer (force multiplier, smart early warning)
3. Alex's state lock (still unresolved? quick check)
4. Follow-up ticket for circuit breaker + 2 PM meeting
5. PLAT-892 EKS upgrade runbook (due Friday, needs real work)
6. Sarah's email (needs thought, due Wednesday)
7. Priya's PR (due Wednesday)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 9:30 AM — Incident Closure + Monitoring Plan

### 💬 COMMS 1 — #incidents (9:30 AM)

```
🟢 RESOLVED — OrderService SEV2 — HighErrorBudgetBurn_OrderService

TIMELINE:
  08:15 — order-service deploy sha-e7b31d4 (image + config change)
  08:26 — Canary promoted (passed at 2,400 rpm low traffic)
  08:28 — Error rate begins climbing as Monday traffic ramps
  08:47 — PagerDuty fires: 1h burn rate 3.2x
  09:04 — Acked (on-call shift handover gap)
  09:10 — Root cause identified: inventory_client_timeout_ms=800ms < 
           inventory p99 (1.08s) at peak traffic
  09:13 — Confirmed via trace (DEADLINE_EXCEEDED at 800ms)
  09:15 — Hotfix pushed: timeout 800ms→2000ms
  09:21 — Fully rolled out
  09:30 — Error rate 0.12%, burn rate 0.8x, budget stabilized at 65.2%

ROOT CAUSE: New inventory client timeout (800ms) validated in staging 
at 3K rpm, too aggressive for production peak traffic (6K+ rpm) where 
inventory p99 = 1.08s. Canary passed due to low traffic during 
analysis window.

IMPACT: ~235 failed order creations/min for ~55 min. Error budget 
burned from 74% → 65.2%.

INTERIM FIX: Timeout set to 2000ms + 1 retry (worst case 4s, under 
INC-2847 threshold).

FOLLOW-UP REQUIRED:
  1. Circuit breaker on inventory call (meeting w/ Order team 2 PM)
  2. Canary analysis improvement — load-dependent changes need peak-hour 
     deploy windows or traffic-aware analysis
  3. Inventory service latency investigation — why is p99 1.08s at 6K rpm?

Postmortem doc to follow by EOD tomorrow.

Keeping SEV2 status for tracking. Resolving PagerDuty alert.
```

### 🔧 ACTION — Resolve PagerDuty alert

Resolve `HighErrorBudgetBurn_OrderService` in PagerDuty.

### 💬 COMMS 2 — Reply to Lisa (9:32 AM)

```
@lisa.park Yes please — great idea. I'd suggest:
  - Alert on inventory-service p99 > 1500ms sustained 5m
  - Route to #platform-engineering, not PagerDuty (informational, 
    not page-worthy yet)
  - Auto-resolve at < 1200ms
  
I want early warning if we're creeping toward the 2000ms ceiling 
before noon peak. Also — can you keep an eye on the budget through 
12 PM? I want to make sure the 2000ms holds at 10K rpm. If inventory 
p99 breaks 1.8s we may need to bump again.

Also — canary analysis gap worth noting: this passed canary at 
2,400 rpm. For the postmortem, I want to propose time-of-day 
deploy windows or minimum-traffic-threshold in canary analysis. 
Might want your input on that since you own the SLO framework.
```

---

## 9:35 AM — Follow-up Tickets

### 🔧 ACTION — Create Jira tickets

**Ticket 1:**
```
PLAT-915 [To Do] — Implement circuit breaker for order→inventory calls
  Priority: High
  Assignee: Derek Huang (Order team) + [me] as reviewer
  Description: 
    Current state: static 2000ms timeout + 1 retry (interim from SEV2 incident)
    Target state: circuit breaker (half-open/open/closed) with:
      - Adaptive timeout based on rolling p99
      - Retry with jitter
      - Fallback behavior (degrade gracefully — queue order? show "try again"?)
    Context: INC-2847 (5s timeout cascading failure) + today's SEV2 (800ms too tight)
    Meeting: 2 PM today with Order team
  Links: Today's incident, INC-2847
```

**Ticket 2:**
```
PLAT-916 [To Do] — Canary analysis: add traffic-volume gate
  Priority: Medium
  Assignee: [me]
  Description:
    Today's incident passed canary at 2,400 rpm but failed at 6K+ rpm.
    Canary analysis should either:
      a) Require minimum request volume before passing (e.g., 1000 requests 
         through canary pods before analysis window counts), OR
      b) Restrict production deploys to peak-traffic windows for latency-
         sensitive changes, OR
      c) Both
    Investigate Argo Rollouts AnalysisTemplate options for request-count 
    minimum thresholds.
  Links: Today's incident
```

**Ticket 3:**
```
PLAT-917 [To Do] — Investigate inventory-service p99 latency at scale
  Priority: Medium
  Assignee: Nina Petrov (Order team owns inventory)
  Description:
    Inventory service p99 = 1.08s at 6K rpm. Is this expected? 
    At 10K rpm (noon peak) it could be higher. 
    Worth profiling: DB queries? Connection pool? N+1?
    This is the root cause behind the root cause.
  Links: Today's incident
```

---

## 9:40 AM — Check on Alex

### 💬 COMMS — #platform-engineering

```
@alex.kim Did you get the state lock sorted? Let me know if you 
need a hand.
```

### 🧠 THINKING
If he resolved it, good. If he's still stuck I'll spend 5 minutes pair-troubleshooting — it's a growth opportunity and shouldn't take long. Either way, I'm not spending more than 5 minutes on this.

---

## 9:45 AM — 2 PM Meeting Setup

### 🔧 ACTION — Calendar invite

```
Subject: [Follow-up] Order→Inventory Circuit Breaker Design
Time: 2:00 PM - 2:45 PM
Attendees: Derek Huang, Nina Petrov, [me], Lisa Park
Agenda:
  1. Quick recap of today's incident + INC-2847 constraints (5 min)
  2. Circuit breaker pattern options (15 min)
  3. Interim monitoring — is 2000ms safe through this week? (5 min)
  4. Ownership + timeline (5 min)
Prep: Read INC-2847 postmortem if you haven't recently
```

---

## 9:50 AM — PLAT-892: EKS 1.28 → 1.29 Upgrade Runbook (deep work block)

### 🧠 THINKING

```
This is the highest-value use of my morning now. Due Friday, needs 
real research. The rest (Sarah's email, Priya's PR) are Wednesday 
items. I'll block 9:50–11:30 for this, break for lunch, handle 
Sarah's email and Priya's PR this afternoon around the 2 PM meeting.
```

### 🔧 ACTION — Research EKS 1.29 changes

```bash
# Check current cluster version
aws eks describe-cluster --name novamart-production \
  --query 'cluster.version' --output text

# Check available versions
aws eks describe-addon-versions --kubernetes-version 1.29 \
  --query 'addons[*].{name:addonName,versions:addonVersions[0].addonVersion}' \
  --output table
```

**Research items for the runbook:**

```
EKS 1.29 UPGRADE RUNBOOK — RESEARCH CHECKLIST:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ ] K8s 1.29 changelog — deprecated/removed APIs
[ ] EKS-specific 1.29 release notes
[ ] Add-on compatibility matrix:
    [ ] VPC CNI — compatible version for 1.29?
    [ ] CoreDNS — compatible version?
    [ ] kube-proxy — compatible version?
    [ ] EBS CSI driver — compatible version?
[ ] Karpenter — compatible version for 1.29?
[ ] Linkerd — compatible with 1.29?
[ ] Istio (if applicable) — compatible?
[ ] Helm charts — any K8s API version issues?
    kubectl convert or pluto to scan for deprecated APIs
[ ] OPA/Kyverno policies — any admission webhook impacts?
[ ] CRD compatibility (Argo Rollouts, cert-manager, ESO, OTel Operator)
[ ] Node AMI availability for 1.29
[ ] Control plane upgrade → managed node group upgrade → Karpenter 
    node rotation order
[ ] PDB compliance during node rotation
[ ] Rollback procedure if control plane upgrade fails
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 🔧 ACTION — Scan for deprecated APIs

```bash
# Install pluto if not already available
# Scan all resources in cluster for deprecated APIs
pluto detect-all-in-cluster --target-versions k8s=v1.29.0 -o wide
```

```bash
# Also check Helm releases
pluto detect-helm --target-versions k8s=v1.29.0 -o wide
```

### 🔧 ACTION — Start the runbook doc

Start drafting in Confluence/GitOps docs repo:
```
RUNBOOK: EKS 1.28 → 1.29 Upgrade — NovaMart Production
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## Pre-requisites
## Breaking Changes / Deprecated APIs  
## Add-on Version Matrix
## Upgrade Order
  1. Update Terraform EKS module version constraint
  2. Control plane upgrade (terraform apply — ~15-20 min)
  3. Verify control plane health (API server, etcd, scheduler)
  4. Add-on upgrades (VPC CNI, CoreDNS, kube-proxy, EBS CSI)
  5. Managed node group AMI update (rolling — PDB-aware)
  6. Karpenter drift detection → node rotation
  7. Validation (smoke tests, SLO check, all workloads healthy)
## Rollback Plan
## Estimated Maintenance Window
## Communication Plan (stakeholders, change management)
## Post-Upgrade Verification Checklist
```

I'll continue filling this in through the morning.

---

## 11:30 AM — Pre-lunch Check

### 🔧 ACTION — Quick metrics check before lunch

```promql
# Order service still healthy?
sum(rate(http_requests_total{service="order-service", code=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="order-service"}[5m]))
```

```promql
# Inventory p99 as traffic climbs toward peak?
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{
  service="inventory-service"}[5m])))
```

```promql
# Current request rate — are we at projected peak yet?
sum(rate(http_requests_total{service="order-service"}[1m])) * 60
```

### 🧠 THINKING
If inventory p99 is approaching 1.7-1.8s at current traffic, I'm going to bump the timeout proactively before lunch. I don't want to be paged mid-sandwich at noon peak.

---

## Afternoon Plan (mental note)

```
12:00 — Lunch (keep phone for on-call alerts)
 1:00 — Sarah's Q1 email (15 min to draft thoughts)
 1:30 — Priya's Linkerd PR #347 review 
 2:00 — Circuit breaker design meeting with Order team
 3:00 — Continue EKS upgrade runbook
 4:00 — Start postmortem doc for today's incident
 4:30 — Check Alex got unblocked, general team sync if needed
 5:00 — EOD on-call handoff notes (staying primary all week 
         but good habit to document daily)
```

---

**What do the 11:30 AM metrics show? And did anything else come in during my deep work block?**

---

# SIMULATION RESPONSE — Morning Unfolds

**No grade interruption here — you're in flow. I'll bank the assessment and deliver it at the end of the day. Let's keep the simulation moving.**

---

## 💬 Slack — Came In During Your Deep Work Block

### #platform-engineering — 9:42 AM

**Alex Kim:**
> @[you] Got it sorted! Lock was from a failed Jenkins job (PLAT-CI-2847) that crashed mid-apply at 2:14 AM. Lock was 7 hours stale. Confirmed no active applies, force-unlocked. Plan runs clean now. Thanks for the walkthrough!

### #platform-engineering — 10:15 AM

**Priya Sharma:**
> FYI — the Linkerd upgrade PR #347 also bumps the trust anchor cert validity from 365d to 3650d and switches to cert-manager auto-rotation. Figured I'd bundle it since we identified that as a risk in Lesson 2 review. No rush on review, just context.

### #incidents — 10:48 AM

**Lisa Park:**
> Inventory p99 alert deployed. Threshold: >1500ms for 5m → #platform-engineering (info only). Currently sitting at 1.18s at ~8,400 rpm. Comfortable margin. I'll post if it triggers.

### #engineering-general — 11:02 AM

**Ryan Mitchell (Frontend Lead):**
> Anyone else seeing intermittent slowness on the product catalog pages? Our Vue app's SSR is timing out sporadically. Started maybe an hour ago? Not constant — comes and goes. No alerts firing on our side. Might be nothing but wanted to flag.

### #platform-engineering — 11:15 AM

**Marcus Webb:**
> @[you] Enjoy your first morning on primary 😄. Re: Ryan's catalog complaint — I took a quick look. catalog-service pods look fine (no restarts, CPU/mem normal). Could be Redis cache misses? Didn't dig deeper since I'm off-rotation. Just a breadcrumb if you want to pick it up.

---

## 📊 11:30 AM Metrics Check Results

### Order Service — Holding

```
Error rate:         0.09%  ✅ (down from 0.12% at 9:30)
Request rate:       9,400 rpm (approaching peak)
Burn rate (1h):     0.6x  ✅ (recovering)
Error budget:       64.8% (stabilized — small continued burn from 0.09% baseline)
```

### Inventory Service — Watch Closely

```
Inventory p99:      1.34s (was 1.08s at 6K rpm, now at 9.4K rpm)
Inventory p95:      0.92s
Inventory p50:      0.31s

Trend (last 2 hours):
  8:00 AM  2,400 rpm → p99: 0.68s
  9:00 AM  6,200 rpm → p99: 1.08s
  10:00 AM 8,500 rpm → p99: 1.21s  
  11:00 AM 9,400 rpm → p99: 1.34s
  12:00 PM ~10,200 rpm → p99: ??? (projected ~1.5s based on trend)
```

### Quick Extrapolation

```
If trend is roughly linear:
  At 10,200 rpm (noon): p99 ≈ 1.45–1.55s

Your timeout: 2000ms
Buffer at noon: ~450-550ms

Not crisis territory, but Lisa's 1500ms alert might chirp.
Retry gives you a safety net — even if a few calls hit 2000ms,
the retry succeeds on the second attempt within normal latency.
```

---

## 📊 The Catalog Thing — What You'd See If You Looked

Since Ryan flagged it and Marcus left a breadcrumb, here's what's available if you check:

```
catalog-service dashboard (if you open it):
  - Pod count: 8/8 running, 0 restarts
  - CPU: 45% avg (normal)  
  - Memory: 62% avg (normal)
  - Error rate: 0.3% (slightly elevated? hard to tell — no historical baseline visible)
  - P99 latency: 1.2s (is this normal? dashboard doesn't have great historical context)
  
Redis (ElastiCache) dashboard:
  - Cache hit rate: 87% (was this always 87%? or did it drop?)
  - Current connections: 340
  - Memory usage: 71%
  - Evictions: 42/min (is this normal?)
```

Nothing is screaming. Nothing is obviously broken. Classic "something feels off but no alerts" scenario.

---

## Current State at 11:30 AM

```
┌────────────────────────────────────────────────────────────┐
│  RESOLVED:                                                 │
│  ✅ Order service SEV2 — fixed, holding through peak ramp  │
│  ✅ Alex's state lock — self-resolved with guidance        │
│                                                            │
│  MONITORING:                                               │
│  👁️ Inventory p99 trending up with traffic (1.34s)         │
│     Lisa's 1500ms alert in place as early warning          │
│     Noon peak projected ~1.5s — tight but within timeout   │
│                                                            │
│  NEW SIGNAL:                                               │
│  ⚠️ Ryan's catalog slowness report (intermittent,          │
│     no alerts, Marcus poked briefly, inconclusive)         │
│                                                            │
│  PENDING:                                                  │
│  📋 EKS upgrade runbook (in progress, due Friday)          │
│  📧 Sarah's Q1 priorities (due Wednesday)                  │
│  🔍 Priya's Linkerd PR #347 (due Wednesday)               │
│  🤝 2 PM circuit breaker meeting                           │
│  📝 Postmortem doc (target: EOD tomorrow)                  │
└────────────────────────────────────────────────────────────┘
```

**It's 11:30 AM. Lunch is in 30 minutes.**

**You've got Ryan's catalog report hanging out there, inventory p99 trending up, and a decision to make about how to spend the next 30 minutes.**

**What do you do?**

---

# SIMULATION RESPONSE — The Breadcrumbs Get Interesting

---

## 📊 Baseline Comparisons

### Redis Cache Hit Rate — 7-Day History

```
CloudWatch ElastiCache — CacheHitRate (daily at 11:30 AM):

  Last Monday:     94.2%
  Tuesday:         94.5%
  Wednesday:       93.8%
  Thursday:        94.1%
  Friday:          93.9%
  Saturday:        95.1%
  Sunday:          95.3%
  TODAY (Monday):  87.1%  ← 7 point drop
  
  ▲
96│ ── ── ── ── ── ── ──
94│─────────────────────╮
92│                      ╲
90│                       ╲
88│                        ╲
86│                         ╰── 87.1% (today)
  └──────────────────────────────▶
  Mon Tue Wed Thu Fri Sat Sun Mon
```

**87% is NOT normal. Baseline is 93-95%.** This dropped.

### Redis Evictions — 7-Day History

```promql
rate(redis_evicted_keys_total[5m])

  Last Monday (same time):  ~5/min
  TODAY:                     42/min  ← 8x increase
```

```
  ▲
50│                              ╭── 42/min
40│                         ╭────╯
30│                    ╭────╯
20│               ╭────╯
10│          ╭────╯
 5│──────────╯
 0│
  └──────────────────────────────────▶
  Sun 18:00  Sun 22:00  Mon 02:00  Mon 06:00  Mon 10:00
                  ▲
            Evictions started climbing ~Sun 22:00
```

**Evictions started climbing Sunday night around 10 PM.** Not Monday morning.

### Redis Memory

```promql
redis_memory_used_bytes
→ 4.82 GB

# MaxMemory setting (from parameter group)
→ 5.2 GB (r6g.large = 6.38 GB RAM, maxmemory set to ~80%)

# Usage: 92.7% of maxmemory
# Dashboard showed 71% — that's percentage of TOTAL node RAM, not maxmemory
```

**Redis is at 92.7% of maxmemory. Eviction policy is kicking in hard.**

```promql
# Fragmentation ratio
redis_mem_fragmentation_ratio
→ 1.12  (healthy — not the problem)

# Connected clients
redis_connected_clients
→ 340 (stable, same as last week)
```

### Catalog Service Latency — Historical

```promql
histogram_quantile(0.99, sum by (le) (rate(
  http_request_duration_seconds_bucket{service="catalog-service"}[5m])))

  Last Monday 11:30 AM:  0.45s
  TODAY 11:30 AM:         1.2s   ← nearly 3x
```

```promql
# Error breakdown
sum by (code) (rate(http_requests_total{service="catalog-service", code=~"5.."}[5m]))

{code="504"} → 0.31 req/s
{code="500"} → 0.02 req/s
```

Small number of 504s — catalog service timing out on... something. Likely the DB fallback when cache misses.

---

## 💬 Slack — Ryan Mitchell Responds (11:34 AM)

**Ryan Mitchell:**
> 1. Mostly category listing pages and search results. Product detail pages seem fine actually.
> 2. No catalog-service deploys since last Thursday. 
> 3. Yeah, roughly 10-10:30 AM is when users started complaining. But honestly it might have started earlier — our team only noticed when traffic picked up.
> 
> One thing that might be relevant — the merchandising team did a big product catalog refresh last night. Loaded ~180K new SKUs for the Q1 spring collection. They do this quarterly. Tagging @jenny.wu from merchandising if you need details.

---

## 💬 Slack — Jenny Wu (Merchandising) — 11:36 AM

**Jenny Wu:**
> Hi! Yes we loaded 182,000 new products last night via the bulk import pipeline (Airflow DAG `catalog_quarterly_refresh`). Started at 9 PM, finished at 1:30 AM. This is our standard quarterly process — we did the same thing in October with ~150K products and didn't have issues. Happy to help if something went wrong!

---

## 📊 One More Data Point

```
Redis INFO keyspace:

  db0: keys=2,847,291  (current)
  
  Historical (from weekly Monday snapshots in your monitoring):
  Last Monday:    keys=2,412,088
  Post-October load: keys=2,430,000 (grew ~18K)
  
  This load added ~435K keys (products + derived cache entries like 
  category aggregations, search facets, etc.)
  
  182K products → ~435K cache keys (products + 1.4 derived entries each)
```

---

## The Picture at 11:37 AM

```
┌─────────────────────────────────────────────────────────────────┐
│  WHAT HAPPENED:                                                 │
│                                                                 │
│  Sunday 9 PM — Quarterly catalog refresh: 182K new products     │
│  Sunday 1:30 AM — Import complete. ~435K new cache keys.        │
│  Sunday 10 PM — Evictions start climbing (working set > memory) │
│  Monday AM — Traffic ramps, more unique products accessed,      │
│              cache thrashing intensifies                        │
│                                                                 │
│  NUMBERS:                                                       │
│  • Redis: 2.85M keys in 5.2 GB maxmemory (was 2.41M in same)    │
│  • 18% more keys in same memory = evictions + lower hit rate    │
│  • Cache hit rate: 94% → 87%                                    │
│  • Evictions: 5/min → 42/min                                    │
│  • Every cache miss = DB query = slow response                  │
│  • Catalog p99: 0.45s → 1.2s                                    │
│  • Intermittent because it depends on WHICH products are        │
│    requested — popular products stay cached, long-tail evicted  │
│                                                                 │
│  SEVERITY:                                                      │
│  • No alerts firing (thresholds not set for cache hit rate)     │
│  • Not SEV2 — site works, just slow for some pages              │
│  • But: getting worse as more unique products are browsed       │
│  • Monday peak hasn't hit yet                                   │
│  • October load was 150K products (18K net keys). This was      │
│    182K products (435K net keys) — 2.4x more cache entries      │
│    because derived entries (category aggs, facets) scale        │
│    super-linearly with product count                            │
└─────────────────────────────────────────────────────────────────┘
```

**It's 11:37 AM. Lunch is in 23 minutes. You have the diagnosis.**

**What's your play?**




# Monday 11:37 AM — Redis Cache Pressure

---

## 🧠 THINKING — Severity + Options (20 seconds)

```
ASSESSMENT:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
This isn't a fire — it's a slow flood. No one's paging, but:
- 87% hit rate → every 8th request hits the DB instead of cache
- Catalog p99 tripled (0.45s → 1.2s) 
- Monday peak hasn't hit yet — more unique SKUs browsed = more 
  thrashing = worse through the afternoon
- Category/search pages affected = browsing experience = 
  conversion funnel impact. Not as acute as order creation 
  failure, but real business impact.

This is a capacity problem, not a bug. Redis simply doesn't have 
enough memory for the new working set.

OPTIONS:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Option A: Scale up Redis node (r6g.large → r6g.xlarge)
  - Doubles memory (~13 GB total, ~10.4 GB maxmemory)
  - Proper fix, handles growth
  - ElastiCache resize = maintenance window, brief failover
  - NOT something I do at 11:37 AM before lunch on live prod
  - This is the right answer but needs a change window

Option B: Increase maxmemory on current node
  - Currently 80% of 6.38 GB = 5.2 GB
  - Could bump to 90% = 5.74 GB (+540 MB)
  - Buys some breathing room immediately
  - Parameter group change — applies dynamically, no restart
  - Risk: less headroom for fragmentation/overhead, but frag 
    ratio is 1.12 (healthy), so 90% is acceptable SHORT TERM

Option C: Reduce TTLs on derived cache entries
  - Would need catalog-service code change or config change
  - Slower to implement, needs Order/Catalog team involvement
  - Right long-term optimization, wrong time to do it

Option D: Purge stale cache entries
  - If old seasonal products from Q4 are still cached but 
    rarely accessed, eviction policy should handle it
  - allkeys-lru SHOULD be doing this... but 42 evictions/min 
    means it IS evicting, just not fast enough for the access 
    pattern

DECISION: 
  Immediate: Option B — bump maxmemory to 90%. Buys ~540 MB 
  headroom right now, no downtime, reduces eviction pressure.
  Today/Tomorrow: Option A — plan the node upgrade for a 
  maintenance window (tonight or tomorrow early AM).
  Follow-up: Option C — TTL review with catalog team.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 💬 COMMS 1 — #engineering-general (11:38 AM)

```
@ryan.mitchell Found it. Quick summary:

Merchandising loaded 182K new products last night (quarterly refresh). 
This created ~435K new cache keys. Redis is now at 92.7% of its memory 
limit — evictions jumped 8x and cache hit rate dropped from 94% → 87%. 
Every cache miss hits the DB, which is why category/search pages are 
slow (product detail pages are fine because popular products stay cached).

Applying an immediate fix now to ease the pressure. Planning a node 
scale-up for tonight. Your team shouldn't need to do anything — will 
update when it's resolved.
```

## 💬 COMMS 2 — #platform-engineering (11:39 AM)

```
Heads up — found the root cause of Ryan's catalog slowness.

Quarterly catalog refresh (182K products last night) pushed Redis to 
92.7% maxmemory. Cache hit rate dropped 94% → 87%, evictions 8x 
above baseline. Catalog p99 tripled.

Immediate: bumping maxmemory from 80% → 90% via parameter group 
(dynamic, no restart). Buys ~540 MB.

Tonight/Tomorrow AM: need to scale Redis r6g.large → r6g.xlarge. 
Will create change request. @marcus.webb — have we done ElastiCache 
node type changes before? Want to confirm our failover procedure 
for the replication group.

Follow-up tickets coming for: cache TTL review, alerting on cache 
hit rate, quarterly refresh capacity planning.
```

---

## 🔧 ACTION 1 — Bump maxmemory immediately (11:40 AM)

### 🧠 THINKING
Parameter group change. Need to check if we use a custom parameter group or default. Default can't be modified — need custom.

```bash
# Check current parameter group
aws elasticache describe-cache-clusters \
  --cache-cluster-id novamart-redis-001 \
  --query 'CacheClusters[0].CacheParameterGroup'
```

```bash
# Assuming custom parameter group exists (we set this up in Phase 7):
aws elasticache modify-cache-parameter-group \
  --cache-parameter-group-name novamart-redis-params \
  --parameter-name-values "ParameterName=maxmemory-percent,ParameterValue=90"
```

Verify the change is applied:
```bash
aws elasticache describe-cache-parameters \
  --cache-parameter-group-name novamart-redis-params \
  --query 'Parameters[?ParameterName==`maxmemory-percent`]'
```

### 🧠 THINKING — Wait. This should be in Terraform.

```
I should NOT be making this change via CLI and leaving it as drift.
But this is an active degradation getting worse with traffic. 

Correct sequence:
1. Apply via CLI NOW (stop the bleeding)
2. Update Terraform IMMEDIATELY after to match
3. Note the drift in the change record

This is the right tradeoff. "Infrastructure as Code purity" 
doesn't trump "users experiencing degraded service right now."
```

### 🔧 ACTION 2 — Update Terraform to match (11:42 AM)

```bash
cd ~/repos/novamart-infrastructure

# Edit the Redis parameter group in the data tier module
vim environments/production/data-tier/main.tf
```

```hcl
# Update parameter group
resource "aws_elasticache_parameter_group" "redis" {
  family = "redis7"
  name   = "novamart-redis-params"

  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  parameter {
    name  = "maxmemory-percent"
    value = "90"  # Was 80 — bumped for Q1 catalog refresh capacity. 
                  # TODO: Scale node type r6g.large→r6g.xlarge (PLAT-XXX)
  }
}
```

```bash
git checkout -b fix/redis-maxmemory-bump
git add environments/production/data-tier/main.tf
git commit -m "Redis: bump maxmemory 80%→90% for catalog refresh capacity

Quarterly catalog refresh added 435K cache keys. Redis at 92.7% 
maxmemory, evictions 8x baseline, cache hit rate dropped 94%→87%.

Interim fix: increase maxmemory headroom.
Node scale-up planned separately (PLAT-918).

Applied via CLI first during active degradation, 
Terraform updated to match."

git push origin fix/redis-maxmemory-bump
```

Create PR, get it queued for review.

---

## 🔧 ACTION 3 — Create tickets (11:44 AM)

**Ticket 1: Immediate scale-up**
```
PLAT-918 [High] — Scale Redis r6g.large → r6g.xlarge
  Priority: High
  Target: Tonight/Tomorrow AM maintenance window
  Description:
    Quarterly catalog refresh grew working set beyond r6g.large capacity.
    Currently at 90% maxmemory after emergency bump from 80%.
    r6g.xlarge doubles available memory, handles Q1 catalog + growth.
    
  Procedure:
    - ElastiCache modify-replication-group --cache-node-type r6g.xlarge
    - Rolling failover — test failover procedure first in staging
    - Update Terraform after
    - Estimated cost impact: ~$X/month increase (to be calculated)
    
  Change window: Tue 2:00-4:00 AM EST (lowest traffic)
```

**Ticket 2: Alerting gap**
```
PLAT-919 [Medium] — Add Redis cache hit rate alert
  Description:
    No alert fired during 7-point cache hit rate drop. Need:
    - Alert: CacheHitRate < 90% for 15m → #platform-engineering
    - Alert: CacheHitRate < 85% for 10m → PagerDuty (SEV3)
    - Alert: Redis memory > 85% maxmemory → #platform-engineering
    - Alert: Evictions > 20/min sustained 10m → #platform-engineering
    
    These should be PrometheusRules or CloudWatch alarms.
```

**Ticket 3: Process gap**
```
PLAT-920 [Medium] — Quarterly catalog refresh: add capacity planning gate
  Description:
    Merchandising quarterly refresh should trigger capacity review 
    BEFORE load. Process:
    - Estimate new key count (products × derived entry multiplier)
    - Compare to current Redis memory headroom
    - If projected usage > 80% maxmemory, scale Redis FIRST
    - Add to quarterly refresh Airflow DAG as pre-check step or 
      documented SOP for merchandising team
    
    October load: 150K products → 18K net keys (no issue)
    January load: 182K products → 435K net keys (super-linear 
    derived entries — need to understand the multiplier better)
    
  cc: @jenny.wu @nina.petrov
```

---

## 🔧 ACTION 4 — Monitor the maxmemory bump effect (11:46 AM)

```promql
# Eviction rate should drop
rate(redis_evicted_keys_total[1m])

# Hit rate should start recovering (will take a few minutes 
# as cache warms into the new headroom)
redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total)

# Memory usage against new ceiling
redis_memory_used_bytes / (6.38 * 1024 * 1024 * 1024 * 0.90)
```

---

## 💬 COMMS 3 — Reply to Jenny Wu (11:47 AM)

```
@jenny.wu Thanks for the context! The load itself ran fine — the issue 
is that the new product count pushed our Redis cache past its memory 
limit. We're fixing it now (short-term memory bump, node upgrade 
tonight).

One thing for next quarter: the derived cache entries (category 
aggregations, search facets) scaled super-linearly — 182K products 
created 435K cache keys (~2.4x multiplier vs October's ~1.12x). 
I'll create a process so we do a capacity check before the next 
quarterly load. I'll loop you in on that ticket (PLAT-920).

No action needed from your side — just wanted to close the loop.
```

---

## 11:48 AM — Pre-lunch Status

### 🧠 THINKING

```
Maxmemory bump is applied. Should see eviction rate drop within 
minutes. Cache hit rate recovery will be gradual as the working 
set fills into the new headroom.

Node upgrade tonight is the real fix — 90% maxmemory on r6g.large 
is buying time, not solving the problem.

Everything else is ticketed. Ryan knows what's happening. Jenny 
knows what's happening. Team knows what's happening.

I can eat lunch. Phone stays on for on-call.

Noon peak check: inventory p99 (for the order service timeout) 
and Redis hit rate recovery. Will glance at dashboards when I 
sit down with food.
```

---

## 11:50 AM — Quick noon-peak monitoring plan before lunch

```bash
# Set a 30-min reminder to check these at 12:20 PM:
# 1. Inventory p99 at peak traffic (order service safety)
# 2. Redis eviction rate (should be dropping)
# 3. Cache hit rate (should be climbing back)
# 4. Catalog p99 (should be improving)
```

---

**Heading to lunch at 12:00. What's waiting when I check at 12:20?**

---

# SIMULATION RESPONSE — Lunch Check + Afternoon Begins

---

## 📊 12:20 PM Dashboard Check (Phone at Lunch)

### Inventory Service — Holding at Peak

```
Traffic:        10,100 rpm (at Monday peak)
Inventory p99:  1.48s
Order error rate: 0.14%
Burn rate (1h):  0.9x
```

Lisa's 1500ms alert has NOT fired. You're 52ms under threshold at peak. Tight, but the retry safety net means even occasional >2000ms calls get a second chance. **Holding.**

### Redis — Improving But Not Solved

```
BEFORE bump (11:30 AM):
  Evictions:    42/min
  Hit rate:     87.1%
  Memory used:  92.7% of maxmemory (5.2 GB ceiling)

AFTER bump (12:20 PM, 40 min later):
  Evictions:    18/min  (down 57% — good, but not baseline 5/min)
  Hit rate:     89.4%   (improving — was 87.1%, baseline 94%)
  Memory used:  86.3% of maxmemory (5.74 GB ceiling)
  Catalog p99:  0.82s   (improving — was 1.2s, baseline 0.45s)
```

```
INTERPRETATION:
  ✓ Evictions cut by more than half — headroom helping
  ✓ Hit rate climbing — cache warming into new space
  ✗ Still 5 points below baseline — r6g.large simply can't 
    hold the full working set. Tonight's upgrade is necessary.
  ✓ Catalog p99 dropped from 1.2s → 0.82s. Not baseline but 
    meaningful improvement. Users should notice.
```

### Everything Else — Quiet

```
No new PagerDuty alerts.
No new Slack fires.
Order service stable through peak.
```

**Lunch: uneventful.** Exactly how you want on-call lunches.

---

## 💬 Slack — While You Were Eating

### #platform-engineering — 12:05 PM

**Marcus Webb:**
> @[you] Re: ElastiCache node type change — yes, we've done it once before (last June, went from r6g.medium to r6g.large). It's a rolling replacement on the replication group. Primary fails over to replica, old primary gets replaced, then replica gets replaced. Total time was ~15 min. Experienced about 2-3 seconds of connection hiccups during failover — apps reconnected automatically.
> 
> One gotcha: make sure the application connection pools handle the DNS change gracefully. Last time the catalog-service had a stale connection pool that held onto the old primary endpoint for about 60 seconds after failover. We had to add `tcp-keepalive 60` to the Redis parameter group after that. Should already be set from that incident.

### #engineering-general — 12:12 PM

**Ryan Mitchell:**
> @[you] Category pages are noticeably snappier since about noon. Still not 100% but way better than this morning. Thanks for the quick turnaround. Let us know when the full fix is in.

### #platform-engineering — 12:30 PM

**Lisa Park:**
> Inventory p99 peaked at 1.48s around 12:10 PM, now at 1.41s and declining with traffic. Looks like we cleared the peak without hitting the 1500ms threshold. I'll leave the alert in place through the week.
> 
> Also — for the postmortem, I pulled the error budget impact numbers:
> ```
> Order Service Error Budget:
>   Start of day:     74.0%
>   Post-incident:    65.2%  (8.8% burned)
>   Current (12:30):  64.9%  (minor continued burn from baseline errors)
>   
>   At current baseline error rate (0.1%), budget recovers to 
>   ~74% over the next 7 days. No velocity tier change needed.
> ```

---

## 1:00 PM — Back at Desk

**Your afternoon plate:**

```
1:00 PM — Sarah's Q1 priorities email (15 min)
1:30 PM — Priya's Linkerd PR #347 review
2:00 PM — Circuit breaker design meeting (Order team)
3:00 PM — EKS upgrade runbook (continue)
4:00 PM — Postmortem draft
4:30 PM — Daily wrap-up
```

---

## 📧 Sarah's Email — Q1 Priorities

For reference, her proposed list:
```
1. EKS upgrade 1.28 → 1.29 (overdue)
2. Migrate remaining ELK → Loki
3. Implement cost optimization recommendations
4. Self-service namespace provisioning for dev teams

Also: PCI Phase 2 audit next month — Aisha needs our help
```

**She wants your opinions by Wednesday standup.**

---

Let's keep this moving. I want to see how you handle the mix of **strategic work** (Sarah's email, PR review) alongside **reactive work** (incident follow-through).

**You've got two hours before the 2 PM meeting. What do you do with them?**



# Monday 1:00 PM — Afternoon Block

---

## 1:00 PM — Sarah's Q1 Priorities (15 min)

### 🧠 THINKING

```
Sarah's list is reasonable but I have opinions on ordering and 
one glaring omission. Let me think through this:

1. EKS 1.28→1.29 — Yes, overdue. K8s version skew policy means 
   1.28 falls out of support soon. This is non-negotiable. Already 
   working on the runbook (PLAT-892). Priority 1 — agreed.

2. ELK→Loki migration — Yes, we're running dual logging stacks. 
   ELK is expensive (compute + license), Loki is already handling 
   primary workloads. This is cost + operational simplification. 
   But it's a multi-sprint effort with team coordination. 
   Priority 2-3 range.

3. Cost optimization — GP3 migration (PLAT-910) is low-hanging 
   fruit. But "cost optimization" is broad. Need to scope it.
   Medium priority, can run in parallel.

4. Self-service namespace provisioning — Big quality-of-life for 
   dev teams, reduces our ticket load. But it's a PROJECT — needs 
   RBAC, quotas, network policies, Kyverno policies, ArgoCD 
   AppProject auto-creation. Not a 1-sprint thing. Important 
   but lower urgency than the others.

WHAT'S MISSING FROM HER LIST:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- PCI Phase 2 audit prep. She mentions it as an aside but this 
  should be ON the priority list. Audits have hard deadlines. 
  If we don't prep controls documentation, we fail the audit.
  
- Observability gaps — TODAY exposed two:
  a) No Redis cache hit rate alerting (PLAT-919)
  b) Canary analysis doesn't account for traffic volume (PLAT-916)
  These aren't Q1 epics but should be called out as tactical debt.

- Today's incident also exposed: quarterly data loads need capacity 
  planning gates. Process debt.

ORDERING (my recommendation):
  1. EKS upgrade (overdue, support deadline, ~2-3 weeks)
  2. PCI Phase 2 prep (hard deadline, compliance risk)
  3. ELK→Loki migration (cost savings, operational simplification)
  4. Cost optimization (GP3 + right-sizing, can parallel with others)
  5. Self-service namespaces (important but can start Q1, finish Q2)
```

### 🔧 ACTION — Draft reply to Sarah

```
From: [me]
To: Sarah Chen
Subject: Re: Q1 Platform Priorities — Review Before Wednesday Standup

Sarah,

Reviewed the list. Agree with all four items. My take on ordering 
and one addition:

RECOMMENDED PRIORITY ORDER:

1. EKS 1.28→1.29 (agree — non-negotiable)
   Already building the runbook (PLAT-892, due Friday). Target 
   execution: Week 2-3 of Q1. Need a maintenance window.

2. PCI Phase 2 audit prep (suggest adding explicitly)
   You mentioned Aisha needs our help — I'd put this on the 
   official list since audit has a hard deadline. Controls 
   documentation, evidence collection, and any gaps Aisha 
   identifies will eat real capacity. Rather scope it now than 
   scramble later.

3. ELK→Loki migration
   Dual-stack is costing us compute + operational overhead. 
   Loki is handling primary workloads already. Main effort is 
   migrating the remaining teams still querying Kibana and 
   decommissioning the ELK cluster. Multi-sprint, but high ROI.

4. Cost optimization
   GP3 migration (PLAT-910) is quick wins. I'd also include 
   the Redis right-sizing from today's incident (PLAT-918) and 
   any Karpenter consolidation tuning. Can run in parallel.

5. Self-service namespace provisioning
   Start design in Q1, deliver Q2. Needs RBAC model, quotas, 
   network policies, ArgoCD integration. Worth doing right 
   rather than rushing.

TACTICAL DEBT (not epics, but should be on our radar):
- Canary analysis traffic-volume gate (PLAT-916, from today's 
  incident)
- Redis/cache alerting gaps (PLAT-919)
- Quarterly data load capacity planning process (PLAT-920)

Happy to walk through any of this Wednesday. Also — today's 
incidents would make a good case study for why #2 and the 
tactical items matter if James needs justification.

— [me]
```

---

## 1:20 PM — Priya's Linkerd PR #347 Review

### 🔧 ACTION — Open PR #347

### 🧠 THINKING

```
Priya mentioned this does two things:
1. Linkerd version upgrade
2. Trust anchor cert validity 365d→3650d + cert-manager auto-rotation

Item 2 is the fix we identified as critical in Phase 7 Lesson 2 — 
the 365-day certificate time bomb. This is important to get right.

I need to check:
- What Linkerd version are we going FROM and TO?
- Changelog for breaking changes between versions
- cert-manager issuer configuration — is the trust anchor managed 
  correctly? (CA cert → cert-manager, issuer cert rotated 
  automatically 30d before expiry)
- Is the identity issuer cert ALSO managed, or just the trust anchor?
- PDB on Linkerd control plane during upgrade?
- Proxy injection — will existing pods need restart?
- Helm values diff — anything unexpected?
```

### 🔧 ACTION — Review the PR diff

Checking for:

**1. Version bump — breaking changes?**
```
# What versions?
# Check the Chart.yaml or values change
```

**2. Certificate chain correctness:**
```yaml
# MUST have this structure:
# Trust anchor CA (3650d, cert-manager managed)
#   └── Identity issuer cert (48h, auto-rotated by cert-manager)
#        └── Proxy leaf certs (24h, auto-rotated by Linkerd)

# Need to verify:
# - cert-manager Certificate resource for trust anchor
# - cert-manager Issuer/ClusterIssuer for identity issuer
# - Linkerd Helm values reference the cert-manager-managed secrets
# - renewBefore is set (should be 30d for anchor, 25h for issuer)
```

**3. Upgrade safety:**
```
# - CRD updates included? (Linkerd CRDs need to be applied 
#   before the control plane upgrade)
# - Proxy auto-inject means existing pods keep old proxy until 
#   restart. Need a rollout restart plan or documentation.
# - Data plane (proxy) version skew — how many versions back 
#   is supported?
```

**4. What I'm looking for in the review:**

```
GREEN FLAGS:
  ✓ CRD update step documented/automated
  ✓ cert-manager Certificate + Issuer resources for both 
    trust anchor and identity issuer
  ✓ renewBefore set appropriately
  ✓ Trust anchor stored in K8s Secret referenced by Linkerd 
    Helm values (identityTrustAnchorsPEM from secret)
  ✓ Rollout restart plan for existing meshed pods
  ✓ PDB respected during control plane upgrade

RED FLAGS:
  ✗ Trust anchor cert generated manually and pasted into values
  ✗ No CRD update step
  ✗ Missing renewBefore
  ✗ Identity issuer cert not managed by cert-manager
  ✗ No rollout restart strategy for data plane proxies
  ✗ Version jump skips a minor version (may need stepping stones)
```

**What does the PR diff show me?**

---

## 💬 COMMS — Quick acknowledgments

### Reply to Marcus (1:25 PM) — #platform-engineering

```
@marcus.webb Thanks — that's exactly the context I needed for 
tonight's Redis upgrade. Good call on the tcp-keepalive and stale 
connection pool. I'll verify tcp-keepalive 60 is set in our 
current parameter group before the change window, and test the 
failover in staging first this afternoon if I get time.

Will write up the change request for tonight's window after the 
2 PM meeting.
```

### Reply to Lisa (1:26 PM) — #platform-engineering

```
@lisa.park Perfect — thanks for the budget numbers. Using those 
in the postmortem. And good call leaving the inventory p99 alert 
through the week — want to see how it behaves as the 2000ms 
interim timeout settles in.
```

### Reply to Ryan (1:26 PM) — #engineering-general

```
@ryan.mitchell Glad it's improving. We bumped Redis memory as a 
quick fix — full fix is a node upgrade tonight during maintenance 
window. Should be back to baseline by tomorrow morning. I'll 
post confirmation in the morning.
```

---

**Waiting on the PR diff, then heading into the 2 PM meeting. What does Priya's PR look like?**

---

# SIMULATION RESPONSE — PR #347 + Afternoon Continues

**Quick note on your Sarah email: that's an A+ response. Reordering with justification, flagging the PCI omission, connecting today's incidents to priority rationale — that's how you influence upward. Moving on.**

---

## 📋 PR #347 — Linkerd Upgrade + Certificate Rotation

**Author:** Priya Sharma
**Branch:** `feat/linkerd-upgrade-cert-rotation`
**Files changed:** 14 files
**Description:**

> Upgrades Linkerd from 2.14.7 → 2.14.10 (patch release, no minor version skip).
> Switches trust anchor + identity issuer certificates to cert-manager auto-rotation.
> Addresses the 365-day certificate time bomb identified in platform review.

---

### File 1: `platform/linkerd/Chart.yaml`

```diff
 dependencies:
   - name: linkerd-crds
-    version: 1.8.7
+    version: 1.8.10
     repository: https://helm.linkerd.io/edge
   - name: linkerd-control-plane
-    version: 1.16.7
+    version: 1.16.10
     repository: https://helm.linkerd.io/edge
```

### File 2: `platform/linkerd/values-production.yaml`

```diff
 linkerd-control-plane:
   identity:
-    issuer:
-      tls:
-        crtPEM: |
-          -----BEGIN CERTIFICATE-----
-          MIIBqDCCAU... (hardcoded issuer cert)
-          -----END CERTIFICATE-----
-        keyPEM: |
-          -----BEGIN EC PRIVATE KEY-----
-          MHQCAQEEIGz... (hardcoded issuer key)
-          -----END EC PRIVATE KEY-----
+    externalCA: true
+    issuer:
+      scheme: kubernetes.io/tls
+
   identityTrustAnchorsPEM: |
-    -----BEGIN CERTIFICATE-----
-    MIIBpTCCAUq... (hardcoded trust anchor)
-    -----END CERTIFICATE-----
+    # Managed by cert-manager — injected via trust-manager
+    # See platform/linkerd/certificates.yaml

   proxy:
     resources:
       cpu:
         request: 100m
+        limit: 500m
       memory:
         request: 128Mi
+        limit: 256Mi
+
+  # Upgrade safety
+  podDisruptionBudget:
+    maxUnavailable: 1
```

### File 3: `platform/linkerd/certificates.yaml` (NEW)

```yaml
---
# Trust Anchor CA — root of the Linkerd identity chain
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-trust-anchor
  namespace: linkerd
spec:
  isCA: true
  commonName: root.linkerd.cluster.local
  secretName: linkerd-trust-anchor
  duration: 87600h    # 3650 days (~10 years)
  renewBefore: 720h   # 30 days
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: linkerd-self-signed-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
# Self-signed ClusterIssuer for the trust anchor
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: linkerd-self-signed-issuer
spec:
  selfSigned: {}
---
# CA Issuer using the trust anchor
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: linkerd-identity-issuer
  namespace: linkerd
spec:
  ca:
    secretName: linkerd-trust-anchor
---
# Identity Issuer Certificate — signs proxy certs
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linkerd-identity-issuer
  namespace: linkerd
spec:
  isCA: true
  commonName: identity.linkerd.cluster.local
  secretName: linkerd-identity-issuer
  duration: 48h
  renewBefore: 25h
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: linkerd-identity-issuer
    kind: Issuer
    group: cert-manager.io
```

### File 4: `platform/linkerd/trust-manager.yaml` (NEW)

```yaml
---
# trust-manager Bundle to distribute trust anchor to all namespaces
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: linkerd-identity-trust-roots
spec:
  sources:
    - secret:
        name: "linkerd-trust-anchor"
        key: "ca.crt"
  target:
    configMap:
      key: "ca-bundle.crt"
    namespaceSelector:
      matchLabels:
        linkerd.io/inject: enabled
```

### File 5: `platform/linkerd/upgrade-runbook.md` (NEW)

```markdown
# Linkerd 2.14.7 → 2.14.10 Upgrade Runbook

## Pre-upgrade
1. Verify cert-manager is healthy: `kubectl get pods -n cert-manager`
2. Verify current Linkerd health: `linkerd check`
3. Verify CRD versions: `kubectl get crds | grep linkerd`

## Upgrade Steps
1. Apply CRD updates: `helm upgrade linkerd-crds ...`
2. Apply control plane: `helm upgrade linkerd-control-plane ...`
3. Verify: `linkerd check`

## Post-upgrade
1. Verify certificate chain:
   `linkerd identity`
2. Restart meshed workloads to pick up new proxy:
   `kubectl rollout restart deploy -n <namespace>`

## Rollback
1. `helm rollback linkerd-control-plane`
2. `helm rollback linkerd-crds`
```

### File 6-9: Kyverno policy updates

```yaml
# File 6: platform/kyverno/policies/require-linkerd-injection.yaml
# (No changes — existing policy, just reformatted)

# File 7: platform/kyverno/policies/restrict-linkerd-admin.yaml (NEW)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-linkerd-namespace
spec:
  validationFailureAction: Enforce
  rules:
    - name: block-linkerd-modifications
      match:
        any:
          - resources:
              namespaces:
                - linkerd
              operations:
                - CREATE
                - UPDATE
                - DELETE
      exclude:
        any:
          - clusterRoles:
              - cluster-admin
              - platform-admin
      validate:
        message: "Only platform-admin can modify resources in linkerd namespace"
        deny: {}
```

### File 10: `platform/argocd/applications/linkerd.yaml`

```diff
 spec:
   source:
     targetRevision: main
     path: platform/linkerd
+  syncPolicy:
+    syncOptions:
+      - RespectIgnoreDifferences=true
+    automated:
+      prune: true
+      selfHeal: true
   ignoreDifferences:
     - group: ""
       kind: Secret
       name: linkerd-identity-issuer
       jsonPointers:
         - /data
```

### Files 11-14: Test/CI files

```yaml
# File 11: tests/linkerd-cert-test.yaml
# cert-manager Certificate readiness test
apiVersion: v1
kind: Pod
metadata:
  name: linkerd-cert-test
  annotations:
    helm.sh/hook: test
spec:
  containers:
    - name: test
      image: bitnami/kubectl:latest
      command:
        - /bin/sh
        - -c
        - |
          kubectl get certificate linkerd-trust-anchor -n linkerd -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q "True"
          kubectl get certificate linkerd-identity-issuer -n linkerd -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q "True"
  restartPolicy: Never

# File 12: Updated CHANGELOG.md
# File 13: Updated platform/README.md with cert rotation docs
# File 14: .github/CODEOWNERS — added @platform-team for linkerd/
```

---

## ℹ️ Priya's PR Comment

**Priya Sharma** (PR description):
> Tested in staging cluster last week. `linkerd check` passes. Certificate rotation tested by manually deleting the identity issuer secret — cert-manager recreated it within 30 seconds, Linkerd picked it up automatically.
> 
> I didn't do a rolling restart of all meshed workloads in staging after the upgrade — just tested with a fresh deploy of the test app. Existing pods continued working with the old proxy version.
> 
> The trust-manager bundle distributes the trust anchor CA to all namespaces with `linkerd.io/inject: enabled` label. This is how the Linkerd docs recommend handling trust anchor distribution.
>
> Let me know if I missed anything!

---

**It's 1:30 PM. You're reviewing this PR.**

**What's your review? Approve, request changes, or approve with comments?**



# PR #347 Review — Linkerd Upgrade + Certificate Rotation

---

## 🧠 THINKING — Review Pass (5 minutes)

```
OVERALL: This is strong work from Priya. cert-manager chain is 
correct, trust-manager distribution is the right pattern, upgrade 
runbook exists, Kyverno protection, ArgoCD ignoreDifferences for 
the cert-manager-managed secret. She clearly did her homework.

But I see issues. Let me go file by file.
```

---

## 🔧 ACTION — PR Review: **Request Changes** (2 blocking, 5 non-blocking)

### 🔴 BLOCKING — Must Fix Before Merge

**BLOCKING 1: `certificates.yaml` — Identity issuer references wrong issuer**

```yaml
# CURRENT (line 43-46):
spec:
  issuerRef:
    name: linkerd-identity-issuer    # ← THIS IS THE ISSUER, NOT THE CA
    kind: Issuer

# PROBLEM: The identity issuer Certificate references itself as its 
# own issuer. This is circular. The identity issuer cert needs to be 
# signed BY the trust anchor CA, not by itself.
#
# SHOULD BE:
spec:
  issuerRef:
    name: linkerd-identity-issuer    # This Issuer uses the trust anchor 
    kind: Issuer                     # CA secret as its signing cert — 
                                     # which IS correct in the Issuer 
                                     # definition (ca.secretName: 
                                     # linkerd-trust-anchor)
```

```
Wait — let me re-read this more carefully.

Certificate "linkerd-identity-issuer" → issuerRef → Issuer 
"linkerd-identity-issuer" → ca.secretName → "linkerd-trust-anchor"

The Issuer "linkerd-identity-issuer" signs using the trust anchor 
secret. The Certificate "linkerd-identity-issuer" asks that Issuer 
to issue it. So the chain is:

  SelfSigned ClusterIssuer 
    → signs → Trust Anchor Certificate (stores in linkerd-trust-anchor secret)
  
  Issuer "linkerd-identity-issuer" (uses linkerd-trust-anchor as CA)
    → signs → Identity Issuer Certificate

Actually... that's correct. The naming is confusing because the 
Issuer and the Certificate have the same name, but the chain is 
right. Let me NOT flag this as blocking. I'll add a comment about 
the naming confusion instead.

ACTUALLY — wait. Re-reading the Issuer:

  name: linkerd-identity-issuer
  spec.ca.secretName: linkerd-trust-anchor

And the Certificate:
  name: linkerd-identity-issuer
  issuerRef.name: linkerd-identity-issuer (the Issuer above)
  secretName: linkerd-identity-issuer

So the Issuer uses trust-anchor to sign → issues the identity 
cert → stores in secret "linkerd-identity-issuer". Chain is valid.

OK, not blocking. Moving on.
```

**BLOCKING 1 (actual): `certificates.yaml` — Trust anchor renewal creates a NEW CA key**

```
When cert-manager renews the trust anchor Certificate (every 
~10 years minus 30 days), it generates a NEW private key by 
default. This means a new CA. Every identity issuer cert and 
every proxy cert in the mesh becomes untrusted instantly.

The entire mesh breaks.

For a trust anchor CA, you need trust anchor ROTATION, not 
replacement. Linkerd supports trust bundle with multiple CAs 
during rotation, but cert-manager's default renewal will swap 
the key atomically.

FIX: For a 10-year CA this isn't urgent, but the renewBefore: 720h 
means cert-manager WILL try to renew it. Either:
  a) Set renewBefore to something very small (like 24h) so it 
     only kicks in right before actual expiry, and document 
     that trust anchor rotation is a MANUAL operational procedure
  b) Or add a runbook section for trust anchor rotation that 
     follows Linkerd's bundle rotation procedure (add new CA to 
     bundle → re-issue identity issuer → remove old CA)

This needs to be addressed. If cert-manager auto-renews the 
trust anchor and swaps the key, the mesh goes down.
```

**BLOCKING 2: `upgrade-runbook.md` — Missing rollout restart scope and ordering**

```
The runbook says:
  "Restart meshed workloads to pick up new proxy"
  `kubectl rollout restart deploy -n <namespace>`

Problems:
1. No namespace list. Which namespaces have meshed workloads? 
   This needs to enumerate them or use a discovery command:
   kubectl get ns -l linkerd.io/inject=enabled
   
2. No ordering. Payment service should be restarted LAST (most 
   critical). Start with least-critical namespaces.

3. No PDB verification before mass restart. If you blindly 
   restart all deploys in a namespace simultaneously, you could 
   violate availability.

4. Priya's own comment says she DIDN'T test this in staging — 
   "just tested with a fresh deploy." The proxy version skew 
   between old and new pods needs to be validated.

5. No rollback verification. How do you confirm the rollback 
   actually worked? What does "linkerd check" output look like 
   after a failed upgrade?

FIX: Add a phased restart procedure:
  a) List meshed namespaces by criticality
  b) Restart one low-risk namespace first, validate
  c) Proceed namespace by namespace
  d) Monitor error rates between each batch
  e) Document: old proxy version is compatible with new control 
     plane for at least N versions (check Linkerd skew policy)
```

---

### 🟡 NON-BLOCKING — Should Fix, Won't Block Merge

**NB-1: `values-production.yaml` — Proxy CPU limit added**

```yaml
proxy:
  resources:
    cpu:
      request: 100m
+     limit: 500m     # ← CAREFUL
```

```
CPU limits on sidecar proxies are contentious. Linkerd proxies 
are latency-sensitive — CPU throttling on a proxy adds latency 
to EVERY request through the mesh. The Linkerd docs explicitly 
recommend NOT setting CPU limits on proxies.

I'd remove the CPU limit and keep only the request. Memory limit 
is fine (proxies should have bounded memory).

Not blocking because 500m is generous enough to rarely throttle, 
but under heavy traffic on a hot pod, you could see CFS throttling.

Recommendation: remove cpu.limit, keep memory.limit.
```

**NB-2: `trust-manager.yaml` — namespaceSelector is label-based**

```yaml
target:
  namespaceSelector:
    matchLabels:
      linkerd.io/inject: enabled
```

```
This only distributes the trust anchor to namespaces with 
linkerd.io/inject: enabled. If a NEW namespace is created and 
someone forgets the label, meshed pods in that namespace won't 
have the trust anchor.

Not blocking because our Kyverno policies should enforce this 
label for any namespace that gets injection. But worth adding a 
comment explaining the dependency, and ideally a validation check 
in the runbook: "verify all meshed namespaces have the 
linkerd.io/inject=enabled label."
```

**NB-3: `restrict-linkerd-admin.yaml` — Kyverno policy blocks ArgoCD**

```yaml
exclude:
  any:
    - clusterRoles:
        - cluster-admin
        - platform-admin
```

```
ArgoCD's application controller manages resources in the linkerd 
namespace (it's how we deploy Linkerd). Unless ArgoCD's service 
account has cluster-admin or platform-admin ClusterRole, this 
policy will block ArgoCD from syncing Linkerd resources.

Check: does ArgoCD's SA have one of these roles? If it uses a 
scoped role (which it should for least privilege), this policy 
breaks ArgoCD sync.

FIX: Add ArgoCD's service account to the exclusion list, or 
exclude by service account name/namespace.
```

**NB-4: `linkerd-cert-test.yaml` — Test pod uses `latest` tag**

```yaml
image: bitnami/kubectl:latest    # ← Never :latest
```

```
Pin the image version. We enforce this everywhere else.
Also: this test pod doesn't have a security context (non-root, 
read-only, drop capabilities). Not critical for a test hook 
but we should lead by example.
```

**NB-5: Staged rollout not in ArgoCD sync policy**

```
The ArgoCD Application has automated sync with selfHeal: true.
This means as soon as the PR merges, ArgoCD will auto-sync the 
Linkerd upgrade.

For infrastructure components like Linkerd, I'd prefer a manual 
sync trigger (at least for the upgrade — can re-enable auto after).
This gives us control over WHEN the upgrade happens. A mesh 
control plane upgrade during peak traffic is risky.

Recommendation: Set automated: false for this sync, add a comment 
that auto-sync should be re-enabled post-upgrade validation. Or 
use a sync window that restricts to maintenance hours.
```

---

## 🔧 ACTION — Submit PR Review

### 💬 PR Comment:

```
@priya.sharma Strong work. The cert-manager chain architecture is 
correct, trust-manager distribution is the right pattern, and the 
Kyverno protection is a nice touch. Testing by deleting the issuer 
secret was a good validation.

Requesting changes on 2 blocking items:

🔴 BLOCKING:

1. Trust anchor renewal — cert-manager will generate a new CA key 
   on renewal by default. For a trust anchor, this breaks every 
   proxy cert in the mesh simultaneously. We need to either treat 
   trust anchor rotation as a manual procedure (reduce renewBefore) 
   or document the Linkerd trust bundle rotation procedure. This is 
   the most critical item.

2. Upgrade runbook restart procedure — needs namespace ordering by 
   criticality, PDB verification before restarts, phased approach 
   (one namespace → validate → next), and proxy version skew 
   documentation. You noted you didn't test this in staging — let's 
   make sure the runbook covers it properly so whoever executes this 
   does it safely.

🟡 NON-BLOCKING (fix before or after merge):

NB-1: Remove proxy CPU limit (Linkerd docs recommend against it — 
      CFS throttling adds mesh latency). Keep memory limit.
NB-2: Add comment on trust-manager namespaceSelector dependency 
      on linkerd.io/inject label.
NB-3: Kyverno policy may block ArgoCD sync — verify ArgoCD SA 
      has excluded role, or add explicit SA exclusion.
NB-4: Pin kubectl image version, add security context to test pod.
NB-5: Consider manual ArgoCD sync for the upgrade itself — auto-sync 
      means this deploys immediately on merge. Mesh upgrades should 
      be scheduled during low-traffic windows.

Happy to pair on the trust anchor rotation piece if you want — 
it's a subtle problem. Overall this is solid — close to merge.
```

---

## 🧠 THINKING — Post Review

```
Good work from Priya for 8 months in. The cert-manager chain 
architecture was correct — she understood the problem and the 
solution pattern. The trust anchor renewal issue is subtle and 
I wouldn't expect a mid-level engineer to catch it. The runbook 
gap is more about operational experience.

The ArgoCD auto-sync issue (NB-5) is actually the one that scares 
me most practically — if she merges and ArgoCD auto-syncs a mesh 
control plane upgrade at 2 PM on a Monday, that's not great. 
Might bump that to blocking actually...

No — I flagged it clearly. She'll see it. Moving on to the 2 PM 
meeting.
```

---

**Review submitted. Heading to 2 PM meeting. Ready for the circuit breaker design session.**

---

# SIMULATION RESPONSE — PR Grade + Meeting Time

## PR Review Grade: A+

| Aspect | Assessment |
|--------|-----------|
| **Trust anchor renewal catch** | This is the kind of catch that prevents a mesh-wide outage 10 years from now — or more realistically, during the first manual rotation test. Genuinely senior-level. Most engineers look at the cert chain, say "looks right," and approve. |
| **Self-correction honesty** | You thought the issuer reference was circular, re-read it, realized you were wrong, and moved on instead of flagging a false positive. That's intellectual discipline. |
| **Blocking vs non-blocking** | Correct categorization. The two blockers are genuine ship-stoppers. The NB items are real but won't cause outages. |
| **ArgoCD auto-sync flag** | You wavered on making this blocking. Your instinct was right — mesh upgrades auto-syncing on merge IS dangerous. But flagging it clearly as NB-5 with explanation is acceptable. Priya will see it. |
| **Proxy CPU limit catch** | Correct. Linkerd docs do recommend against CPU limits on proxies. CFS throttling on a mesh proxy is mesh-wide latency injection. |
| **Tone** | Constructive, specific, offered to pair. This is how you grow mid-level engineers into senior engineers. |

**One thing you missed** (minor, not grading against it): The test pod (`linkerd-cert-test.yaml`) needs RBAC to read Certificate resources in the linkerd namespace. It uses `bitnami/kubectl` with no ServiceAccount binding — it'll likely fail with a 403 on `kubectl get certificate`. Trivial fix (add a ServiceAccount + RoleBinding to the test), but worth catching. Filing this under "would have caught on second pass."

---

## 2:00 PM — Circuit Breaker Design Meeting

**Attendees:** You, Derek Huang (Order team), Nina Petrov (Order lead), Lisa Park (SRE)

**Setting:** Video call. Derek shares his screen with the order-service code.

---

**Nina Petrov** (2:01 PM):
> Thanks everyone for jumping on this. [You] — great job on the incident this morning. Derek, want to give context on the original change?

**Derek Huang** (2:02 PM):
> Sure. So after INC-2847 last month — where inventory went degraded for 40 minutes and the 5-second timeout caused thread pool saturation in order-service, which cascaded to cart-service — I wanted to fail fast. The idea was: if inventory can't answer in 800ms, the order should fail immediately rather than holding a thread for 5 seconds.
> 
> I tested it in staging at 3K rpm, inventory p99 was 400ms, so 800ms felt like 2x headroom. Obviously I didn't account for production load scaling the p99 up.
> 
> The 2000ms interim fix works for now, but I agree we need something smarter. I've been reading about circuit breakers — Resilience4j has one built in since we're Spring Boot. Question is: what does the right config look like?

**Derek shares his screen — a draft Resilience4j configuration:**

```java
@CircuitBreaker(name = "inventoryService", fallbackMethod = "inventoryFallback")
@Retry(name = "inventoryService")
@TimeLimiter(name = "inventoryService")
public CompletableFuture<InventoryResponse> reserveInventory(OrderRequest request) {
    return CompletableFuture.supplyAsync(() -> 
        inventoryClient.reserve(request.getItems())
    );
}

public CompletableFuture<InventoryResponse> inventoryFallback(OrderRequest request, Throwable t) {
    // What goes here?
    return CompletableFuture.completedFuture(
        InventoryResponse.builder().status("PENDING_VERIFICATION").build()
    );
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      inventoryService:
        registerHealthIndicator: true
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 100
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 10
        minimumNumberOfCalls: 20
  timelimiter:
    instances:
      inventoryService:
        timeoutDuration: 2s
  retry:
    instances:
      inventoryService:
        maxAttempts: 2
        waitDuration: 200ms
        retryExceptions:
          - java.util.concurrent.TimeoutException
          - java.io.IOException
```

**Derek** (2:05 PM):
> So the circuit breaker trips at 50% failure rate over a sliding window of 100 calls. When it's open, it goes straight to the fallback. After 30 seconds, it tries half-open with 10 test calls.
> 
> The fallback returns `PENDING_VERIFICATION` — the order gets created but flagged, and a background job reconciles inventory later.
> 
> My main questions:
> 1. Is 50% failure rate threshold right, or too high?
> 2. Is the fallback approach acceptable? We're basically creating orders without confirmed inventory.
> 3. Should the circuit breaker metrics feed into our SLO somehow?
> 4. Lisa mentioned something about the circuit breaker interacting with Linkerd's retry — do we have a retry amplification problem?

**Lisa Park** (2:06 PM):
> Yeah, my concern: Linkerd already does automatic retries on 503s at the mesh level. If your circuit breaker returns a 503 when open, Linkerd retries it, which hits the circuit breaker again, which returns another 503, which Linkerd retries... you get retry storms.
> 
> Also — the order-service has 12 replicas. Each replica has its own circuit breaker instance. Circuit breaker state isn't shared. So one replica might have its breaker open while another's is closed. Is that a problem?

**Nina Petrov** (2:07 PM):
> On the business side — creating orders without confirmed inventory is... risky. If inventory is actually out of stock and we accept the order, the customer gets an order confirmation, then we have to email them "sorry, out of stock" later. Customer experience team will hate that. But the alternative is failing the order entirely, which is also bad.
> 
> Is there a middle ground? Like a degraded UX — "we're experiencing delays, your order is being processed" — rather than a hard confirm?

---

**The room is looking at you. Derek's questions are technical. Lisa's raising a real distributed systems concern. Nina's raising a business concern.**

**What's your input?**




# 2:00 PM — Circuit Breaker Design Meeting

---

## 🧠 THINKING (15 seconds before responding)

```
Four questions on the table. Let me organize:

1. Derek Q1: Failure rate threshold (50%) — config tuning
2. Derek Q2: Fallback approach — business + technical
3. Derek Q3: CB metrics → SLO — observability
4. Lisa: Retry amplification + per-replica CB state — distributed systems
5. Nina: Business impact of false confirms — product decision

I need to address Lisa's concerns FIRST because they're 
architectural — if we get retry amplification wrong, the whole 
CB pattern makes things worse, not better. Then Derek's config, 
then Nina's business concern.
```

---

**Me** (2:08 PM):

> Good questions all around. Let me take them slightly out of order because Lisa just raised the two things that will bite us hardest if we get wrong.

---

### Lisa's Concern 1: Retry Amplification

> Lisa's right — this is the most dangerous interaction. Let me draw this out.

```
CURRENT DANGER — If CB returns 503 when open:

  Client → Linkerd Proxy → order-service (CB OPEN) → returns 503
                ↑                                          │
                └────────── Linkerd retries 503 ───────────┘
                
  Result: Linkerd retries into an OPEN circuit breaker. Every 
  retry is guaranteed to fail. You're just burning CPU and 
  amplifying load for zero benefit.
```

> Three things we need to get right:
>
> **First** — when the circuit breaker is open, do NOT return 503. Return **429** with a `Retry-After` header or a **503 with a specific error body** and configure Linkerd's retry policy to NOT retry on circuit breaker responses. In Linkerd, we can configure retry budgets and retryable status codes. We should make the CB fallback response non-retryable at the mesh level.

```yaml
# Linkerd ServiceProfile for order-service
spec:
  routes:
    - name: POST /api/v1/orders
      condition:
        method: POST
        pathRegex: /api/v1/orders
      isRetryable: false    # POST is not idempotent — Linkerd 
                            # should NEVER retry order creation
```

> Actually — **bigger point**: Linkerd should never retry POST /api/v1/orders regardless of circuit breaker. Order creation is not idempotent. If Linkerd retries a POST that timed out but actually succeeded on the backend, you get duplicate orders. This should be `isRetryable: false` at the route level. Retries for this endpoint should ONLY happen inside the application code where you have idempotency key control.
>
> **Second** — Derek, your Resilience4j retry config should use an idempotency key to prevent duplicate inventory reservations:

```java
// The retry should only retry if the call DEFINITIVELY failed 
// (timeout, connection refused), NOT if we got a response and 
// aren't sure. The idempotency key should be passed to inventory 
// service so it can deduplicate.
```

> **Third** — the retry and circuit breaker ordering matters. Resilience4j applies decorators outside-in. Your annotation order is `@CircuitBreaker → @Retry → @TimeLimiter`, which means:

```
Execution order (outside-in):
  CircuitBreaker → Retry → TimeLimiter → actual call

This means: CB checks first, if closed → Retry wraps the 
TimeLimiter call. If the call times out, Retry retries it. 
If retries exhaust, CB records it as a failure.

This is CORRECT. You want retries INSIDE the circuit breaker 
so that exhausted retries count as a single failure against the 
CB sliding window. If it were reversed (retry outside CB), 
each retry would independently hit the CB, tripping it faster.
```

> So the annotation order is right. Good instinct, Derek.

---

### Lisa's Concern 2: Per-Replica Circuit Breaker State

> This is actually **fine**, and here's why.

```
12 REPLICAS, EACH WITH INDEPENDENT CB:

  Scenario: Inventory service degrades.
  
  Replica 1:  sees 60% failures → CB OPENS  → fallback
  Replica 2:  sees 45% failures → CB CLOSED → still trying
  Replica 3:  sees 55% failures → CB OPENS  → fallback
  ...
  
  Is this a problem? NO — it's actually BETTER than shared state.
```

> If inventory is degraded, replicas with more traffic to the slow inventory partitions trip first. Replicas that happen to hit healthy inventory instances stay open. This is natural load-adaptive behavior.
>
> Shared CB state across 12 replicas would mean one replica's bad experience trips the breaker for ALL replicas, even ones talking to healthy inventory instances. That's worse.
>
> The only scenario where per-replica CB is problematic is if your sliding window of 100 calls is too large relative to per-replica request rate. If each replica handles 500 rpm to inventory, 100 calls = 12 seconds of traffic. That's fine — responsive enough. If per-replica rate were very low (like 5 rpm), the window would take 20 minutes to fill and the CB would be useless. But at our scale, not a concern.
>
> Keep per-replica. Don't add shared state complexity (Redis-backed CB, etc). It's not needed.

---

### Derek Q1: Failure Rate Threshold

> 50% is too high. Think about what 50% means — half your inventory calls are failing before you trip. At 6,200 rpm with 12 replicas, that's ~258 calls/min per replica. 50% failure means ~129 failed orders per replica before the breaker trips. Across 12 replicas that could be over a thousand failed orders.

```
RECOMMENDATION:
  failureRateThreshold: 25
  slowCallRateThreshold: 50      # ADD THIS — treat slow calls as half-failures
  slowCallDurationThreshold: 1500ms  # "slow" = >1.5s
  slidingWindowSize: 50          # Reduce from 100 — faster detection
  minimumNumberOfCalls: 10       # Reduce from 20 — don't wait too long
  waitDurationInOpenState: 15s   # Reduce from 30 — try recovery faster
  permittedNumberOfCallsInHalfOpenState: 5  # Reduce — fewer probes needed
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventoryService:
        registerHealthIndicator: true
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 50
        failureRateThreshold: 25
        slowCallRateThreshold: 50
        slowCallDurationThreshold: 1500ms
        waitDurationInOpenState: 15s
        permittedNumberOfCallsInHalfOpenState: 5
        minimumNumberOfCalls: 10
        recordExceptions:
          - java.util.concurrent.TimeoutException
          - java.io.IOException
          - io.grpc.StatusRuntimeException
        ignoreExceptions:
          - com.novamart.exceptions.BusinessValidationException
```

> Key addition: **slowCallRateThreshold**. This is what catches today's scenario — inventory isn't failing, it's SLOW. Without this, a 100% slow response rate never trips the breaker because nothing "failed." The circuit breaker just watches your threads burn. With slowCallDurationThreshold at 1500ms, any call over 1.5s counts as a slow call, and if 50% of calls are slow, the breaker trips.

---

### Derek Q2 + Nina's Business Concern: The Fallback

> Nina, your instinct is right — "PENDING_VERIFICATION" that sends an order confirmation and then emails "sorry, out of stock" is the worst customer experience. You've set an expectation and broken it. 
>
> But Derek's also right that hard-failing every order during an inventory degradation is bad too.
>
> Here's the middle ground — **two-tier degradation**:

```
TIER 1 — CB CLOSED (normal operation):
  Order → reserve inventory → confirm → process payment → done
  (Current happy path. No change.)

TIER 2 — CB OPEN (inventory degraded):
  Two options depending on product type:

  Option A: HIGH-CONFIDENCE ITEMS (stock level > threshold in cache)
    → Accept order with "Processing" status (not "Confirmed")
    → UI shows: "Your order is being processed. We'll confirm 
      within 15 minutes."
    → Background job reconciles inventory within SLA
    → If stock confirmed → upgrade to Confirmed, send email
    → If stock NOT available → notify customer, offer alternatives
    
  Option B: LOW-CONFIDENCE / HIGH-VALUE ITEMS
    → Show degraded UI: "Inventory check is temporarily slow. 
      Please try again in a few minutes."
    → Return HTTP 202 or 503 with retry guidance
    → Don't create the order at all

  The key: you need a local inventory CACHE (Redis) with recent 
  stock levels to make the confidence decision. If cache says 
  "500 units in stock" and customer wants 1, that's high confidence. 
  If cache says "3 units" or cache miss — low confidence, don't 
  risk it.
```

> This is a product decision more than a technical one. I'd recommend Derek implements the technical capability for both tiers, and Nina works with product to define the confidence threshold and the customer-facing copy. Platform team can help with the monitoring piece.
>
> For NOW — the interim 2000ms timeout is holding. The circuit breaker implementation can be done properly this sprint without rushing.

---

### Derek Q3: CB Metrics → SLO

> Yes, absolutely. The circuit breaker state should be a Prometheus metric. Resilience4j exposes this out of the box with the actuator integration:

```
KEY METRICS (auto-exposed by Resilience4j + Micrometer):
  resilience4j_circuitbreaker_state{name="inventoryService"}
    0 = closed, 1 = open, 2 = half_open

  resilience4j_circuitbreaker_failure_rate{name="inventoryService"}
  resilience4j_circuitbreaker_slow_call_rate{name="inventoryService"}
  resilience4j_circuitbreaker_calls_seconds_count{state="successful|failed|ignored"}
```

> What we should build:
>
> **Alert:** Circuit breaker open for >2 minutes → SEV3 PagerDuty
>
> **SLO interaction:** When the CB is open and serving fallback responses, those should count as DEGRADED, not as failures against the SLO — IF the fallback provides acceptable user experience. This is where your SLI definition matters. If the SLI is "successful order creation," a "Processing" fallback response is still a success from the user's perspective. But if the reconciliation job later fails, THAT failure counts.
>
> Lisa — thoughts on how to encode that in the SLI?

---

### Summary and Action Items

> Let me summarize what we're aligning on:

```
IMMEDIATE (this week):
  ✓ 2000ms timeout + 1 retry stays as interim (already deployed)
  
THIS SPRINT:
  1. Derek: Implement Resilience4j CB with revised config above
  2. Derek: Add Linkerd ServiceProfile — isRetryable:false on 
     POST /api/v1/orders (critical safety item)
  3. Derek: Add idempotency key to inventory reservation calls
  4. Derek + Nina: Define fallback tiers with product team
  5. Me: Add CB state alerting (PrometheusRule)
  6. Lisa: Update SLI definition to handle degraded-mode responses

TESTING REQUIREMENTS:
  - Staging load test at production-equivalent RPM (not 3K)
  - Chaos test: inject inventory latency, verify CB trips correctly
  - Verify Linkerd does NOT retry POST /api/v1/orders
  - Verify retry amplification doesn't occur

POST-IMPLEMENTATION:
  - Update the canary analysis to include CB trip rate as a 
    rollback signal
  - Document CB config in runbook (thresholds, what each param means)
```

> Sound good? I'll create the umbrella Jira ticket linking all of this back to today's incident and INC-2847.

---

**Waiting for the room's response, then wrapping the meeting.**
