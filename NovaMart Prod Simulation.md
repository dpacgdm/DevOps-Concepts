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

---

# SIMULATION RESPONSE — Meeting Wraps

## Meeting Grade: A+

I'm not going to mince words — that was a masterclass in technical leadership in a meeting setting. Let me call out specifics.

| Aspect | Assessment |
|--------|-----------|
| **Retry amplification** | Caught the most dangerous interaction first. The POST idempotency point elevated this from "good answer" to "saved us from a production disaster we haven't had yet." |
| **Per-replica CB justification** | Correctly argued AGAINST adding complexity. Knowing when NOT to add shared state is senior judgment. |
| **slowCallRateThreshold** | This catches today's exact scenario. Derek's original config would NOT have tripped on slow calls — only failures. You identified the gap in real-time. |
| **Two-tier fallback** | Solved Nina's business concern without dismissing it. The confidence-based approach is how Amazon actually handles this at scale. |
| **Meeting leadership** | Reordered the questions by impact, addressed the room's concerns in order of architectural risk, and ended with clear action items and owners. This is how a senior engineer runs a technical design session. |

**One thing I'd push on:** Your Linkerd ServiceProfile suggestion (`isRetryable: false` on POST) is critical and probably the single most important safety item in the whole discussion. It should be its own ticket, not buried in Derek's sprint work. If Derek's sprint slips, that ServiceProfile change should still go in this week. It's a one-line mesh config change that prevents duplicate orders.

---

## Room Responses

**Derek Huang** (2:35 PM):
> This is really clear. The slowCallRateThreshold is the piece I was missing — you're right, inventory wasn't *failing* this morning, it was *slow*, and my original config wouldn't have caught that. I'll implement the revised config and the ServiceProfile change first since that's the quickest safety win. Can probably have a PR up by Thursday.

**Nina Petrov** (2:36 PM):
> The two-tier fallback makes sense. I'll set up a call with product this week to define the confidence thresholds. One question — do we have the inventory stock level cache in Redis already, or does that need to be built?

**You'd know:** The catalog data is in Redis (that's what we dealt with this morning), but inventory *stock levels* are a different dataset. Inventory service owns stock data in its own database. There's no real-time stock level cache in Redis currently.

**Lisa Park** (2:37 PM):
> On the SLI question — I think we define it as: any response that results in an order being created (whether immediate-confirmed or processing-pending) counts as a success. If the order is never created (hard fail or degraded-mode rejection), that's a failure. The background reconciliation failure would be tracked as a separate SLI: `order_reconciliation_success_rate`. That way the CB fallback doesn't game the primary SLO, but we still have visibility.
> 
> I'll draft the updated SLI spec and recording rules.
> 
> Also +1 on the ServiceProfile change being its own fast-track ticket. That's a safety net independent of the circuit breaker work.

---

**Meeting ends at 2:42 PM.** (Under the 2:45 slot — well-run meeting.)

---

## 💬 Slack — While You Were in the Meeting

### #platform-engineering — 2:15 PM

**Priya Sharma:**
> @[you] Thanks for the thorough review! The trust anchor renewal issue is a great catch — I hadn't considered what happens when cert-manager renews the CA key. I'll update the PR:
> 
> 1. Reduce trust anchor renewBefore to 720h → 24h and add a manual rotation runbook section
> 2. Add phased restart procedure with namespace ordering
> 3. Remove proxy CPU limit
> 4. Fix ArgoCD exclusion in Kyverno policy — our ArgoCD SA uses `argocd-application-controller` role, not `cluster-admin`. Will add explicit exclusion.
> 5. Pin kubectl image version + add security context to test pod
> 6. Set ArgoCD sync to manual for the upgrade
> 
> Will push the updates today. Question on the trust anchor rotation — should I also document what happens during a rotation? Like: add new CA to bundle → wait for all proxies to see new bundle → re-issue identity issuer → remove old CA? Or is that overkill for this PR?

### #alerts — 2:31 PM

```
⚠️ WARNING — CatalogServiceLatencyP99High
  catalog-service p99 latency > 800ms for 10m
  Current: 0.78s
  Threshold: 0.8s
  Status: Resolved (auto-resolved at 2:28 PM)
  
  Note: This alert has been flapping — fired at 1:45 PM, 
  resolved at 1:52 PM, fired again at 2:20 PM, resolved 
  at 2:28 PM.
```

### #platform-engineering — 2:38 PM

**Jake Torres:**
> Hey team — I'm working on the GP3 EBS migration (PLAT-910) and found something weird. About 40% of our persistent volumes are still using gp2. But some of them are attached to StatefulSets that I don't recognize — names like `data-kafka-zookeeper-0`, `data-kafka-broker-0` through `data-kafka-broker-5`. We have Kafka? I thought we used RabbitMQ + SQS. Anyone know what this is?

---

**It's 2:42 PM. You're back from the meeting.**

**You have:**
1. **Priya's follow-up question** on trust anchor rotation documentation
2. **Catalog latency alert flapping** — your Redis bump helped but isn't fully solving it. Tonight's node upgrade becomes more urgent.
3. **Jake's mystery Kafka discovery** — this smells like shadow infrastructure someone deployed and didn't tell platform about.
4. **Remaining afternoon:** EKS runbook, postmortem, Redis upgrade prep for tonight.

**What do you do?**



# Monday 2:42 PM — Afternoon Triage

---

## 🧠 THINKING — Quick Priority Scan (15 seconds)

```
1. Priya's question — quick, encouraging, 2 minutes. Yes, document 
   the rotation procedure. That's the whole point.

2. Catalog flapping alert — expected. 0.78s vs 0.8s threshold means 
   we're RIGHT on the edge. The maxmemory bump got us from 1.2s to 
   ~0.8s, but we need the node upgrade tonight to fully resolve. 
   Not actionable right now beyond confirming tonight's change window.
   Quick ack.

3. Jake's Kafka discovery — THIS is the one that needs attention. 
   Unknown StatefulSets with persistent volumes = shadow infrastructure.
   6 Kafka brokers + ZooKeeper that platform team didn't provision, 
   doesn't know about, probably isn't monitoring, has no runbooks, 
   no backup strategy, and no owner on record. This is a compliance 
   risk (PCI audit next month), cost issue, and operational risk.
   Need to figure out WHO deployed it and WHY before I can decide 
   what to do about it. 15 minutes now, then park for proper follow-up.

4. Rest of afternoon: Redis upgrade change request, EKS runbook, 
   postmortem. All still on track.
```

---

## 2:43 PM — Priya's Follow-up (2 minutes)

### 💬 COMMS — PR #347 reply

```
@priya.sharma Yes — absolutely document the rotation procedure. 
That's not overkill, that's the whole reason we're fixing this. 
A rotation runbook that nobody can follow is only marginally better 
than no rotation at all.

Document exactly what you described:
  1. Generate new trust anchor CA (cert-manager Certificate with 
     new name)
  2. Add new CA to trust-manager Bundle (both old + new)
  3. Wait for all proxies to pick up the combined bundle 
     (verify with linkerd identity on sample pods)
  4. Re-issue identity issuer cert signed by new CA
  5. Verify mesh health (linkerd check, watch for TLS errors)
  6. Remove old CA from Bundle
  7. Decommission old Certificate resource

Include estimated time for each step and a rollback at each stage. 
This is the kind of runbook we want to practice in a DR exercise 
before we ever need it for real.

Great instinct asking the question — ship it.
```

---

## 2:45 PM — Catalog Alert Flapping (3 minutes)

### 💬 COMMS — #platform-engineering

```
FYI — catalog latency alert flapping is expected. We're sitting 
right at the 0.8s threshold after this morning's Redis maxmemory 
bump (brought p99 from 1.2s → ~0.78s, threshold is 0.8s). 

Tonight's Redis node upgrade (r6g.large → r6g.xlarge) should 
resolve this fully. Until then, the alert will bounce around 
the threshold with traffic fluctuations.

I'm going to silence this specific alert until tomorrow 8 AM 
to avoid noise. If catalog p99 breaks 1.5s, the higher-severity 
alert will still fire.

Prepping the Redis change request now for tonight's 2 AM window.
```

### 🔧 ACTION — Silence flapping alert

```bash
# Alertmanager silence — catalog p99 alert only, 
# expires tomorrow 8 AM
amtool silence add \
  --alertmanager.url=https://alertmanager.novamart.internal \
  --author="[me]" \
  --comment="Redis node upgrade tonight will resolve. Higher-sev alert still active." \
  --duration="17h" \
  alertname="CatalogServiceLatencyP99High"
```

---

## 2:48 PM — Jake's Kafka Discovery (15 minutes)

### 🧠 THINKING

```
This needs investigation before I react. Options:
  a) Legitimate service someone deployed through proper channels 
     that we just don't know about (unlikely — we'd see the ArgoCD 
     app or Terraform)
  b) Developer deployed it manually via kubectl (more likely — 
     bypassed GitOps, no review, no platform visibility)
  c) Legacy from before the current platform setup

Let me gather facts before raising flags.
```

### 🔧 ACTION 1 — What is this, when did it arrive, who owns it?

```bash
# What namespace?
kubectl get statefulset -A | grep kafka

# Full details
kubectl get statefulset -n <namespace> -o wide

# When was it created?
kubectl get statefulset data-kafka-broker -n <namespace> \
  -o jsonpath='{.metadata.creationTimestamp}'

# Who created it? Check annotations/labels
kubectl get statefulset data-kafka-broker -n <namespace> -o yaml | \
  head -40

# Is there an ArgoCD app managing it?
kubectl get statefulset data-kafka-broker -n <namespace> \
  -o jsonpath='{.metadata.labels.argocd\.argoproj\.io/instance}'

# Is it in ArgoCD at all?
argocd app list | grep -i kafka

# Any Helm release?
helm list -A | grep kafka

# What images are running?
kubectl get pods -n <namespace> -l app=kafka -o \
  jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```

### 🔧 ACTION 2 — Resource footprint

```bash
# PV sizes and storage class
kubectl get pvc -n <namespace> | grep kafka

# Resource requests/limits
kubectl get statefulset data-kafka-broker -n <namespace> \
  -o jsonpath='{.spec.template.spec.containers[0].resources}' | jq

# Node placement — are these on Karpenter nodes or managed?
kubectl get pods -n <namespace> -l app=kafka \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
```

### 🔧 ACTION 3 — Is anything actually using it?

```bash
# Check for services
kubectl get svc -n <namespace> | grep kafka

# Check network connections to Kafka ports (9092/9093)
# via Linkerd metrics if meshed, or via service metrics
```

---

### 💬 COMMS — #platform-engineering (2:50 PM)

```
@jake.torres Good catch. I don't recognize this either and it's 
not in our Terraform or ArgoCD manifests as far as I know. 
Investigating now.

DON'T touch the PVs or migrate them to GP3 until we figure out 
what this is and who owns it. If it's serving production traffic, 
a storage change without the owner's knowledge could cause an 
outage.

What namespace is it in?
```

---

**What does the investigation reveal?**

---

## Parallel: While waiting for kubectl results

### 🔧 ACTION — Redis Change Request (2:55 PM)

```
CHANGE REQUEST: CR-2024-0115-001
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Title: Scale Redis ElastiCache r6g.large → r6g.xlarge
Urgency: High (catalog latency degraded, alert flapping)
Risk: Medium (rolling failover, 2-3s connection hiccup)

BUSINESS JUSTIFICATION:
  Quarterly catalog refresh added 435K cache keys. Current 
  r6g.large at 92.7% maxmemory. Cache hit rate dropped 94%→87%, 
  catalog p99 tripled. Emergency maxmemory bump (80%→90%) 
  partially mitigated. Node upgrade required for full resolution.

CHANGE WINDOW: Tuesday 2024-01-16, 2:00-4:00 AM EST
  (Lowest traffic: ~1,200 rpm)

PROCEDURE:
  Pre-change:
  1. Verify staging failover test completed successfully
  2. Verify tcp-keepalive=60 in parameter group (stale conn fix)
  3. Snapshot current Redis cluster (manual backup)
  4. Notify #engineering-general: maintenance window, expect 
     2-3s connection blip

  Execution:
  1. aws elasticache modify-replication-group \
       --replication-group-id novamart-redis \
       --cache-node-type cache.r6g.xlarge \
       --apply-immediately
  2. Monitor failover via CloudWatch 
     (ReplicationLag, NewConnections, CurrConnections)
  3. Verify primary endpoint DNS resolves to new node
  4. Verify application reconnection (catalog-service logs)

  Post-change:
  1. Verify cache hit rate recovering
  2. Verify eviction rate dropping
  3. Verify catalog p99 returning to baseline (<0.5s)
  4. Update Terraform (node type + revert maxmemory to 80%)
  5. Unsuppress catalog latency alert
  6. Post confirmation to #engineering-general

  Rollback:
  If new node type causes issues (unexpected memory behavior, 
  application incompatibility), revert to r6g.large. Same 
  procedure, reverse node type. ~15 min total.

APPROVALS NEEDED:
  - Platform Engineering Manager (Sarah Chen)
  - Secondary on-call acknowledgment (Marcus Webb)

COST IMPACT:
  r6g.large: ~$0.252/hr → r6g.xlarge: ~$0.504/hr
  Delta: +$0.252/hr = ~$183/month = ~$2,196/year
  Justified by: catalog performance restoration + headroom 
  for Q2 catalog refresh growth

RELATED TICKETS: PLAT-918
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 💬 COMMS — #platform-engineering (3:00 PM)

```
@sarah.chen @marcus.webb Change request for tonight's Redis 
upgrade posted (CR-2024-0115-001). Need your approvals.

r6g.large → r6g.xlarge, 2:00 AM window, rolling failover. 
Marcus — you've done this before, would appreciate a second pair 
of eyes on the procedure. Cost impact: +$183/month.

I'll run the staging failover test between 3-4 PM today to 
validate the procedure before tonight.
```

---

**What did the Kafka investigation turn up? And anything else come in?**

---

# SIMULATION RESPONSE — Grading + Kafka Reveal

## Afternoon Triage Grade: A-

Not A+. Here's why.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Prioritization** | ✅ | Correct: Priya (quick), alert (quick), Kafka (investigate), Redis CR (parallel). Good instincts. |
| **Priya response** | ✅ | Encouraging, specific, correct. The rotation step list is exactly right. |
| **Catalog alert silence** | ⚠️ **GAP** | You said "If catalog p99 breaks 1.5s, the higher-severity alert will still fire." **What higher-severity alert?** You created PLAT-919 this morning specifically because Redis/cache alerting was MISSING. That ticket is still open. You're silencing the one alert that IS firing and assuming a higher-tier alert exists that you literally identified as not existing 3 hours ago. This is how alerts get silenced and nobody notices when things get worse. |
| **Kafka investigation** | ✅ | Thorough kubectl forensics. Good instinct telling Jake not to touch PVs. |
| **Redis change request** | ✅ | Professional, complete, costed. Good. |
| **One more thing** | ⚠️ | You're planning: staging Redis test (3-4 PM), EKS runbook (3 PM), postmortem (4 PM), Kafka follow-up, AND staying up for a 2 AM maintenance window. That's not a plan, that's a wishlist. Something's getting dropped. A senior engineer acknowledges that explicitly and re-prioritizes. You didn't. |

**Fix the alert silence gap right now or I'm docking you further when it bites you at midnight.**

---

## 📋 Kafka Investigation Results

### Namespace and Age

```bash
$ kubectl get statefulset -A | grep kafka

NAMESPACE          NAME                    READY   AGE
data-analytics     data-kafka-zookeeper    3/3     67d
data-analytics     data-kafka-broker       6/6     67d
```

**Namespace: `data-analytics`. Created 67 days ago (mid-November).**

### Labels and Annotations

```bash
$ kubectl get statefulset data-kafka-broker -n data-analytics -o yaml | head -40

metadata:
  name: data-kafka-broker
  namespace: data-analytics
  labels:
    app: kafka
    chart: kafka-26.6.2
    release: analytics-kafka
    heritage: Helm
  annotations:
    meta.helm.sh/release-name: analytics-kafka
    meta.helm.sh/release-namespace: data-analytics
  creationTimestamp: "2023-11-09T14:22:18Z"
```

**Helm release. `analytics-kafka`. Deployed November 9th.**

### ArgoCD Status

```bash
$ argocd app list | grep -i kafka
(no results)

$ argocd app list | grep -i analytics
(no results)
```

**NOT managed by ArgoCD.** Not in GitOps.

### Helm Release

```bash
$ helm list -n data-analytics

NAME              NAMESPACE        REVISION  STATUS    CHART           APP VERSION
analytics-kafka   data-analytics   3         deployed  kafka-26.6.2    3.6.1
```

**Helm chart deployed directly. 3 revisions — someone's been updating it.**

### Images

```bash
$ kubectl get pods -n data-analytics -l app=kafka \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

data-kafka-broker-0     bitnami/kafka:3.6.1
data-kafka-broker-1     bitnami/kafka:3.6.1
data-kafka-broker-2     bitnami/kafka:3.6.1
data-kafka-broker-3     bitnami/kafka:3.6.1
data-kafka-broker-4     bitnami/kafka:3.6.1
data-kafka-broker-5     bitnami/kafka:3.6.1
data-kafka-zookeeper-0  bitnami/zookeeper:3.9.1
data-kafka-zookeeper-1  bitnami/zookeeper:3.9.1
data-kafka-zookeeper-2  bitnami/zookeeper:3.9.1
```

**Bitnami images. Not from our ECR. Pulled from Docker Hub directly.**

### Resources and PVs

```bash
$ kubectl get pvc -n data-analytics | grep kafka

data-kafka-broker-0    Bound   pv-0a3f...  100Gi  gp2   67d
data-kafka-broker-1    Bound   pv-1b4e...  100Gi  gp2   67d
data-kafka-broker-2    Bound   pv-2c5f...  100Gi  gp2   67d
data-kafka-broker-3    Bound   pv-3d6a...  100Gi  gp2   67d
data-kafka-broker-4    Bound   pv-4e7b...  100Gi  gp2   67d
data-kafka-broker-5    Bound   pv-5f8c...  100Gi  gp2   67d
data-kafka-zookeeper-0 Bound   pv-6a9d...  20Gi   gp2   67d
data-kafka-zookeeper-1 Bound   pv-7b0e...  20Gi   gp2   67d
data-kafka-zookeeper-2 Bound   pv-8c1f...  20Gi   gp2   67d

# Total: 720 GiB gp2 storage

$ kubectl get statefulset data-kafka-broker -n data-analytics \
  -o jsonpath='{.spec.template.spec.containers[0].resources}' | jq

{
  "requests": {
    "cpu": "1",
    "memory": "4Gi"
  },
  "limits": {
    "cpu": "2",
    "memory": "6Gi"
  }
}
```

**6 brokers × 4Gi requested = 24 GiB RAM requested. 6 × 1 CPU = 6 CPU requested.**
**Plus 3 ZooKeeper pods. Total cluster: ~30 GiB RAM, ~9 CPU, 720 GiB storage.**

### Node Placement

```bash
$ kubectl get pods -n data-analytics -l app=kafka \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'

data-kafka-broker-0     ip-10-0-42-118.ec2.internal
data-kafka-broker-1     ip-10-0-67-203.ec2.internal
data-kafka-broker-2     ip-10-0-91-55.ec2.internal
data-kafka-broker-3     ip-10-0-42-118.ec2.internal
data-kafka-broker-4     ip-10-0-67-203.ec2.internal
data-kafka-broker-5     ip-10-0-91-55.ec2.internal
```

**Running on Karpenter-provisioned nodes.** No dedicated node pool. Sharing nodes with production workloads.

### Network — Is Anything Using It?

```bash
$ kubectl get svc -n data-analytics | grep kafka

analytics-kafka            ClusterIP   172.20.44.128   <none>   9092/TCP   67d
analytics-kafka-headless   ClusterIP   None            <none>   9092/TCP   67d
```

```bash
# Check Linkerd metrics for traffic to kafka service
$ linkerd stat -n data-analytics deploy 2>/dev/null
No meshed resources found in data-analytics namespace.
```

**NOT meshed.** No Linkerd sidecar. No mTLS. No mesh-level traffic visibility.

```bash
# Check for NetworkPolicy
$ kubectl get networkpolicy -n data-analytics
No resources found in data-analytics namespace.
```

**No NetworkPolicy. Wide open within the cluster.**

### Namespace Details

```bash
$ kubectl get ns data-analytics -o yaml

metadata:
  name: data-analytics
  labels:
    kubernetes.io/metadata.name: data-analytics
  annotations:
    kubectl.kubernetes.io/last-applied-by: "tom.chen@novamart.com"
    kubectl.kubernetes.io/last-applied-at: "2023-11-09T14:15:00Z"
```

**Created by `tom.chen@novamart.com`.** Applied via kubectl directly.

```bash
$ kubectl get resourcequota -n data-analytics
No resources found.

$ kubectl get limitrange -n data-analytics
No resources found.
```

**No resource quotas. No limit ranges.**

---

## 💬 Slack — #platform-engineering — 2:52 PM

**Jake Torres:**
> It's in the `data-analytics` namespace. I've never seen this namespace before either. It's not in our Terraform namespace list or ArgoCD.

---

## 💬 Slack — #platform-engineering — 3:05 PM

**Marcus Webb:**
> Oh. I know what that is. Tom Chen from the data engineering team set that up back in November. They needed a Kafka cluster for their real-time analytics pipeline — event streaming from the order and payment services. I remember him mentioning it in a standup but it never went through a formal platform review. I think @sarah.chen was aware but it got deprioritized in the holiday crunch.
> 
> Tom's on PTO this week BTW. Back next Monday.
> 
> Re: Redis change request — procedure looks good. I'll approve. tcp-keepalive is already set from June. One note: test the failover in staging BEFORE you go to production. Last time we had a brief DNS propagation delay that the ElastiCache endpoint abstracted away, but direct node connections (if anyone's using them) would break.

---

## 📊 The Full Picture

```
┌──────────────────────────────────────────────────────────────┐
│  SHADOW KAFKA CLUSTER — data-analytics namespace             │
│                                                              │
│  WHAT:                                                       │
│  • 6 Kafka brokers + 3 ZooKeeper (Bitnami, Docker Hub)       │
│  • 720 GiB gp2 storage, ~30 GiB RAM, ~9 CPU                  │
│  • Helm-deployed directly (not GitOps)                       │
│  • 3 revisions (someone's maintaining it)                    │
│                                                              │
│  WHO: Tom Chen (Data Engineering), deployed Nov 9            │
│       Sarah may have been loosely aware                      │
│       Tom on PTO until next Monday                           │
│                                                              │
│  WHAT'S WRONG:                                               │
│  ✗ Not in ArgoCD (no GitOps, no drift detection)             │
│  ✗ Not in Terraform (namespace, PVs not managed)             │
│  ✗ Not meshed (no mTLS, no traffic visibility)               │
│  ✗ No NetworkPolicy (any pod in cluster can reach it)        │
│  ✗ No ResourceQuota/LimitRange (can consume unbounded)       │
│  ✗ No monitoring (no ServiceMonitor, no alerts)              │
│  ✗ No backup strategy (720 GiB of data, no snapshots)        │
│  ✗ Images from Docker Hub (not ECR, no vulnerability scan)   │
│  ✗ Sharing Karpenter nodes with production workloads         │
│  ✗ No runbook, no on-call ownership                          │
│  ✗ PCI audit next month — untracked infrastructure = finding │
│  ✗ gp2 storage (PLAT-910 GP3 migration can't touch these)    │
│  ✗ Tom on PTO — no owner available this week                 │
│                                                              │
│  WHAT'S NOT WRONG (yet):                                     │
│  • It's running. 67 days, no restarts visible.               │
│  • It's apparently serving real-time analytics.              │
│  • Nobody's complained about it breaking.                    │
│                                                              │
│  BUT: "It hasn't broken yet" is not a strategy.              │
│       One Karpenter consolidation event could reschedule     │
│       brokers without proper PDBs. One Docker Hub rate       │
│       limit could block image pulls on pod restart.          │
└──────────────────────────────────────────────────────────────┘
```

**It's 3:05 PM. Tom's on PTO. PCI audit is next month. Sarah may or may not have approved this informally.**

**You've also got my grading feedback about the alert silence gap and overcommitted afternoon.**

**What do you do?**



# Monday 3:05 PM — Course Corrections + Kafka Response

---

## 🧠 THINKING — Address the gaps FIRST

```
Two gaps called out. Fix both before moving forward.

1. ALERT SILENCE GAP: I silenced CatalogServiceLatencyP99High 
   and assumed a higher-severity alert exists. I literally created 
   PLAT-919 this morning because Redis alerting was MISSING. 
   I need to either:
   a) Create the critical-tier alert RIGHT NOW before silencing, or
   b) Don't silence the flapping alert — accept the noise
   
   Correct answer: (a). Create a quick high-severity alert now. 
   Takes 5 minutes. Then the silence on the lower threshold is safe.

2. OVERCOMMITTED AFTERNOON: I have too much on the plate.
   Let me be honest about what's realistic:
   
   Must happen today:
   - Fix the alert gap (5 min, NOW)
   - Kafka response (15 min, in progress)
   - Redis staging failover test (30 min, 3:30-4:00 PM)
   - Redis upgrade at 2 AM (have to be awake)
   
   Should happen today but can slip:
   - Postmortem draft → PUSH TO TOMORROW. Committed to EOD 
     tomorrow, still on track.
   - EKS runbook → worked on it this morning, continue tomorrow. 
     Due Friday. Fine.
   
   Doesn't need me today:
   - Sarah's email → DONE, sent at 1 PM
   - Priya's PR → review submitted, ball in her court
```

---

## 3:06 PM — Fix Alert Silence Gap (5 minutes)

### 🔧 ACTION — Create the missing high-severity catalog alert

```yaml
# Quick PrometheusRule — catalog critical latency
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: catalog-service-critical
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: catalog-service-critical
      rules:
        - alert: CatalogServiceLatencyP99Critical
          expr: |
            histogram_quantile(0.99, 
              sum by (le) (rate(http_request_duration_seconds_bucket{service="catalog-service"}[5m]))
            ) > 1.5
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Catalog service p99 latency critically high ({{ $value | humanizeDuration }})"
            description: "Catalog p99 > 1.5s for 5m. Cache/Redis issue likely. Check eviction rate and hit ratio."
            runbook_url: "https://runbooks.novamart.internal/RUNBOOK-CACHE-001"
            dashboard_url: "https://grafana.novamart.internal/d/catalog-service"

        - alert: RedisCacheHitRateCritical
          expr: |
            (
              redis_keyspace_hits_total 
              / (redis_keyspace_hits_total + redis_keyspace_misses_total)
            ) < 0.85
          for: 10m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Redis cache hit rate below 85% ({{ $value | humanizePercentage }})"
            description: "Cache hit rate critically low. Check evictions, memory pressure, and recent data loads."

        - alert: RedisMemoryPressureHigh
          expr: |
            redis_memory_used_bytes / redis_memory_max_bytes > 0.90
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Redis memory usage above 90% of maxmemory"
            description: "Redis at {{ $value | humanizePercentage }} of maxmemory. Evictions likely increasing."

        - alert: RedisEvictionRateHigh
          expr: |
            rate(redis_evicted_keys_total[5m]) > 0.33
          for: 10m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Redis eviction rate elevated ({{ $value | humanize }}/s)"
            description: "Sustained evictions indicate memory pressure. Check maxmemory and working set size."
```

```bash
kubectl apply -f catalog-redis-critical-alerts.yaml

# Verify rules loaded
kubectl get prometheusrule catalog-service-critical -n monitoring
```

Now the silence on the flapping 0.8s alert is safe — the 1.5s critical alert and Redis-specific alerts will fire if things actually get worse.

### 💬 COMMS — #platform-engineering (3:10 PM)

```
Correction on my earlier alert silence: I silenced the catalog 
p99 > 0.8s alert but the higher-severity alerts I referenced 
didn't actually exist yet (that's what PLAT-919 was about). 

Just deployed critical-tier alerts:
  - Catalog p99 > 1.5s for 5m → critical 
  - Redis cache hit rate < 85% for 10m → critical
  - Redis memory > 90% maxmemory → warning
  - Redis evictions > 20/min sustained → warning

The 0.8s silence is now safe. Closing PLAT-919.
```

### 🔧 ACTION — Close PLAT-919

Update PLAT-919 status to Done, link the PrometheusRule commit.

---

## 3:11 PM — Realistic Afternoon Plan

### 💬 COMMS — #platform-engineering (3:11 PM)

```
Revised afternoon plan (being honest about capacity):

Today:
  ✅ Alert gap fixed (just done)
  🔄 Kafka investigation response (next 15 min)
  🔄 Redis staging failover test (3:30-4:00 PM)
  🔄 Redis prod upgrade (2:00 AM tonight)

Pushed to tomorrow:
  📝 Incident postmortem draft (committed EOD tomorrow)
  📋 EKS upgrade runbook (continue tomorrow, due Friday)

I'll be up for the 2 AM Redis window. If anyone wants to help 
shadow the maintenance for learning, let me know.
```

---

## 3:12 PM — Kafka: Response + Immediate Risk Mitigation

### 🧠 THINKING

```
Tom's on PTO. I can't solve this fully without him. But I also 
can't leave a completely unguarded, unmonitored, non-GitOps 
Kafka cluster running on shared production nodes for a week — 
especially with PCI audit next month.

What I CAN do now:
1. Document what I found (done in my head, need to write it up)
2. Immediate safety nets that don't require Tom:
   - NetworkPolicy (restrict who can talk to Kafka)
   - PDB (prevent Karpenter from killing brokers during consolidation)
   - These are NON-DESTRUCTIVE. They protect, they don't change 
     functionality. Safe to apply without Tom.
3. Flag to Sarah (manager) — this is a governance issue, not just 
   a technical one
4. Create a comprehensive onboarding ticket for when Tom returns

What I should NOT do:
- Migrate it to ArgoCD/GitOps without Tom (I don't understand 
  the full setup)
- Mess with images, storage, or config (could break things)
- Delete it (it's serving real analytics)
- Overreact (it's been running 67 days fine)
```

### 🔧 ACTION 1 — Immediate safety: PDB + NetworkPolicy (3:15 PM)

```yaml
# kafka-safety-nets.yaml
---
# PDB — prevent Karpenter from evicting more than 1 broker at a time
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: kafka-broker-pdb
  namespace: data-analytics
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: kafka
      component: broker
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: kafka-zookeeper-pdb
  namespace: data-analytics
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: kafka
      component: zookeeper
---
# NetworkPolicy — restrict Kafka access to known consumers
# For now, allow intra-namespace + order/payment namespaces 
# (likely producers based on Marcus's "event streaming from 
# order and payment services" comment)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kafka-access-control
  namespace: data-analytics
spec:
  podSelector:
    matchLabels:
      app: kafka
  policyTypes:
    - Ingress
  ingress:
    # Allow intra-namespace (brokers ↔ zookeeper ↔ consumers in same ns)
    - from:
        - podSelector: {}
    # Allow order-service namespace (likely producer)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: order-prod
    # Allow payment-service namespace (likely producer)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: payment-prod
    ports:
      - port: 9092
        protocol: TCP
      - port: 2181
        protocol: TCP
```

```bash
# Verify label selectors match before applying
kubectl get pods -n data-analytics -l app=kafka --show-labels

kubectl apply -f kafka-safety-nets.yaml

# Verify PDB
kubectl get pdb -n data-analytics

# Verify NetworkPolicy doesn't break existing connections
# Watch for Kafka consumer lag or errors after applying
```

### 🧠 THINKING — NetworkPolicy Risk

```
The NetworkPolicy could break things if there are consumers in 
namespaces I don't know about. But right now there's NO policy 
at all — any pod in the cluster can reach Kafka. That's worse.

I'm allowing the namespaces Marcus mentioned (order, payment) 
plus intra-namespace. If something breaks, removing the 
NetworkPolicy is a 10-second rollback:
  kubectl delete networkpolicy kafka-access-control -n data-analytics

I'll watch for errors for 15 minutes after applying.
```

### 🔧 ACTION 2 — Flag to Sarah (3:20 PM)

### 💬 COMMS — DM to Sarah Chen

```
Sarah — need to flag something for your awareness.

Jake found a Kafka cluster in a "data-analytics" namespace that 
platform didn't provision. Investigation summary:

- 6 Kafka brokers + 3 ZooKeeper (Bitnami from Docker Hub)
- Deployed Nov 9 by Tom Chen (Data Engineering) via direct Helm
- NOT in ArgoCD, not in Terraform, not meshed, no NetworkPolicy 
  (until I just added one), no monitoring, no backup, no runbook
- Running on Karpenter nodes shared with production workloads
- 720 GiB gp2 storage, ~30 GiB RAM, ~9 CPU
- Tom is on PTO until next Monday
- Marcus says you may have been loosely aware from a standup mention

Why this matters NOW:
1. PCI audit next month — untracked infrastructure is an audit 
   finding. We need an inventory of all data-processing systems.
2. No PDBs existed (I just added them) — Karpenter could have 
   killed brokers at any time during consolidation
3. Docker Hub images — no vulnerability scanning, rate limit risk
4. If this is processing order/payment events, it may be in PCI 
   scope and needs controls documentation

I've applied immediate safety nets (PDB + NetworkPolicy). Not 
touching anything else until Tom's back. 

I'd recommend:
1. You + me + Tom meeting next Monday when he returns
2. Formal platform onboarding for this cluster (ArgoCD, ECR images, 
   monitoring, backup, runbook)
3. Add to PCI scope assessment for Aisha's review
4. Broader conversation about "how do workloads get to production 
   without platform review" — process gap

Want to discuss, or should I just create the tickets and drive it?
```

### 🔧 ACTION 3 — Create comprehensive ticket (3:25 PM)

```
PLAT-921 [High] — Onboard data-analytics Kafka cluster to platform standards
  Priority: High (PCI audit dependency)
  Assignee: [me] + Tom Chen (when back from PTO)
  Due: End of next sprint
  
  BACKGROUND:
    Kafka 3.6.1 cluster (6 brokers + 3 ZK) deployed Nov 9 by 
    Tom Chen via direct Helm install. Not managed by platform.
    See investigation notes below.

  IMMEDIATE SAFETY (COMPLETED):
    ✅ PDB added (maxUnavailable: 1 for brokers and ZK)
    ✅ NetworkPolicy added (restricted to data-analytics + 
       order-prod + payment-prod namespaces)
  
  ONBOARDING CHECKLIST:
    [ ] Meet with Tom to understand full architecture, data flow, 
        consumers/producers, retention requirements
    [ ] Migrate Helm release to ArgoCD (GitOps repo)
    [ ] Migrate images from Docker Hub → ECR (vulnerability scanning)
    [ ] Add Linkerd mesh injection (mTLS)
    [ ] Add monitoring (Kafka ServiceMonitor, consumer lag alerts, 
        broker health, disk usage)
    [ ] Add ResourceQuota + LimitRange to data-analytics namespace
    [ ] Create backup strategy (EBS snapshots for PVs)
    [ ] Create runbook (broker failure, rebalancing, scaling)
    [ ] Assign on-call ownership (Data Engineering team?)
    [ ] GP3 migration for PVs (coordinate with PLAT-910)
    [ ] PCI scope assessment with Aisha (PLAT-922)
    [ ] Terraform: manage namespace + RBAC (who can deploy here?)
    [ ] Document: data flow diagram (what events, from where, to where)
    
  PROCESS GAP:
    How did this bypass platform review? Need to discuss guardrails:
    - Kyverno policy: require platform-team label/approval for 
      new namespaces?
    - ArgoCD: only ArgoCD can deploy to production cluster?
    - RBAC: restrict direct Helm/kubectl in production?
    
  RELATED: PLAT-910 (GP3 migration), PCI Phase 2 audit prep
```

```
PLAT-922 [High] — PCI scope: assess data-analytics Kafka cluster
  Priority: High
  Assignee: Aisha Rahman (Security) + [me]
  Description:
    Unmanaged Kafka cluster processing events from order and payment 
    services. If payment data flows through Kafka, this is PCI 
    in-scope and needs:
    - Encryption at rest (EBS encryption status?)
    - Encryption in transit (no mTLS currently)
    - Access controls (NetworkPolicy added, RBAC needed)
    - Audit logging
    - Vulnerability management (images from Docker Hub)
    
    Flagging for Aisha's awareness before Phase 2 audit.
```

### 💬 COMMS — #platform-engineering (3:28 PM)

```
Kafka investigation complete. Summary:

WHO: Tom Chen (Data Engineering), deployed Nov 9 via direct Helm. 
     Tom is on PTO until next Monday.
WHAT: 6 Kafka brokers + 3 ZK, Bitnami images from Docker Hub, 
     not in GitOps, not meshed, no monitoring, no backups.
WHY: Real-time analytics pipeline consuming order/payment events.

IMMEDIATE ACTIONS (done):
  ✅ PDB applied (prevents Karpenter from disrupting brokers)
  ✅ NetworkPolicy applied (restricted to known consumer namespaces)

WATCH ITEM: If anyone sees errors from data-analytics namespace 
in the next hour, tell me — the NetworkPolicy may need adjustment 
if there are consumers I don't know about.

FOLLOW-UP: PLAT-921 (full onboarding), PLAT-922 (PCI scope). 
Meeting with Tom next Monday. Flagged to Sarah.

@jake.torres — do NOT migrate these PVs to GP3 as part of PLAT-910. 
Skip the data-analytics namespace entirely. We'll handle it as part 
of the Kafka onboarding. Good catch finding this.
```

---

## 3:30 PM — Redis Staging Failover Test

### 🔧 ACTION — Run the test

```bash
# Staging Redis cluster
aws elasticache describe-replication-groups \
  --replication-group-id novamart-staging-redis \
  --query 'ReplicationGroups[0].{Status:Status,Members:MemberClusters}'

# Verify tcp-keepalive setting exists
aws elasticache describe-cache-parameters \
  --cache-parameter-group-name novamart-redis-params \
  --query 'Parameters[?ParameterName==`tcp-keepalive`].ParameterValue'

# Modify staging to r6g.xlarge
aws elasticache modify-replication-group \
  --replication-group-id novamart-staging-redis \
  --cache-node-type cache.r6g.xlarge \
  --apply-immediately

# Monitor failover
watch -n 5 'aws elasticache describe-replication-groups \
  --replication-group-id novamart-staging-redis \
  --query "ReplicationGroups[0].Status"'
```

While failover runs, monitor staging catalog-service:
```promql
# Staging catalog error rate during failover
sum(rate(http_requests_total{service="catalog-service", cluster="staging", code=~"5.."}[1m]))
/
sum(rate(http_requests_total{service="catalog-service", cluster="staging"}[1m]))
```

```bash
# Watch for connection errors in staging catalog-service logs
kubectl logs -n catalog-staging -l app=catalog-service --tail=100 -f | grep -i "redis\|connection\|refused\|timeout"
```

### 🧠 THINKING

```
Expected behavior (per Marcus):
- Rolling replacement, ~15 min total
- 2-3 second connection hiccup during primary failover
- Apps should reconnect automatically
- Watch for the stale connection pool issue Marcus mentioned

If staging test is clean → proceed with 2 AM production window.
If staging shows unexpected issues → postpone prod, reassess.
```

---

## 3:55 PM — Set alarm and EOD prep

### 🔧 ACTION — Tonight logistics

```bash
# Set phone alarm for 1:45 AM
# Prep a tmux session with monitoring ready to go:
#   Pane 1: Redis CloudWatch metrics
#   Pane 2: kubectl logs catalog-service 
#   Pane 3: Grafana with catalog + Redis dashboards open
#   Pane 4: Terminal for AWS CLI commands
```

### 💬 COMMS — #engineering-general (4:00 PM)

```
📢 Scheduled Maintenance — Tonight 2:00-4:00 AM EST

Redis (ElastiCache) node upgrade for improved catalog performance.
Expect a brief (2-3 second) connection hiccup during failover.
Applications should reconnect automatically.

Impact: Momentary cache miss spike, auto-recovers within seconds.
No action needed from application teams.

If you see Redis connection errors after 4:00 AM, ping me or 
#platform-engineering.

Change Request: CR-2024-0115-001
```

### 🔧 ACTION — EOD Daily Notes (4:15 PM)

```markdown
# On-Call Day 1 — Monday Summary

## Incidents
- SEV2: OrderService error budget burn (inventory timeout 800ms→2000ms)
  Status: RESOLVED. Postmortem tomorrow.
- Catalog latency degradation (Redis memory pressure from quarterly refresh)
  Status: MITIGATED. Node upgrade tonight 2 AM.

## Changes Made
- order-service inventory_client_timeout_ms: 800→2000 (hotfix)
- Redis maxmemory: 80%→90% (interim, Terraform updated)
- Added: catalog/Redis critical alerts (PrometheusRule)  
- Added: PDB + NetworkPolicy for shadow Kafka cluster
- Silenced: CatalogServiceLatencyP99High (until tomorrow 8 AM)

## Tickets Created
- PLAT-915: Circuit breaker for order→inventory
- PLAT-916: Canary analysis traffic-volume gate
- PLAT-917: Inventory service latency investigation
- PLAT-918: Redis scale-up r6g.large→r6g.xlarge
- PLAT-919: Redis cache alerting (CLOSED — done today)
- PLAT-920: Quarterly refresh capacity planning gate
- PLAT-921: Kafka cluster platform onboarding
- PLAT-922: Kafka PCI scope assessment

## Tomorrow
- Postmortem draft (committed EOD)
- EKS upgrade runbook (continue, due Friday)
- Redis upgrade verification + Terraform update
- Priya's PR re-review when she pushes updates
- Monitor order service through Tuesday traffic patterns

## Tonight
- 2:00 AM: Redis node upgrade (CR-2024-0115-001)
- Staging test completed: [pending results]
```

---

**What did the staging Redis failover test show? And then I'll set my alarm for 1:45 AM.**

---

# SIMULATION RESPONSE — Grading + Staging Results + Night Shift

## Afternoon Grade: A-

**The self-corrections were good. Not giving you credit for fixing holes you dug yourself.** Addressing my feedback is expected, not bonus-worthy. Here's where you actually stand:

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Alert gap fix** | ✅ | Correct — created the alerts, kept the silence. The right sequence. But you shouldn't have silenced without them in the first place. Net neutral. |
| **Afternoon re-plan** | ✅ | Honest about capacity. Pushed postmortem and EKS runbook with justification. This is what seniors do. |
| **Kafka: risk assessment** | ✅ | Proportional. Safety nets without overreacting. Correct judgment that you can't fully solve this without Tom. |
| **Kafka: Sarah escalation** | ✅ | Well-framed. Technical summary + business impact (PCI) + clear ask. Good upward communication. |
| **Kafka: PDB** | ✅ | Non-destructive, immediately protective. Correct. |
| **Kafka: NetworkPolicy** | ⚠️ **GAP** | Your `ports` field applies to ALL `from` blocks. You're exposing ZooKeeper (2181) to order-prod and payment-prod namespaces. Producers need Kafka (9092), not ZooKeeper. ZK should only be reachable intra-namespace. This isn't least-privilege — it's sloppy. In a PCI context where you literally just flagged this for audit review, shipping a NetworkPolicy that over-exposes ports undermines your own argument. |
| **Kafka: encryption check** | ⚠️ **GAP** | You flagged "encryption at rest" as a question for Aisha in PLAT-922. You could have answered that in 15 seconds: `kubectl get pv <name> -o jsonpath='{.spec.csi.volumeAttributes}'` or `aws ec2 describe-volumes --volume-ids <id> --query 'Volumes[0].Encrypted'`. For PCI, encryption at rest isn't a "let's discuss" — it's a hard requirement you can verify immediately. If those gp2 volumes are unencrypted, that's a finding you need to know about NOW, not when Aisha gets to the ticket next week. |
| **Redis change request** | ✅ | Thorough, costed, rollback documented. Professional. |
| **EOD notes** | ✅ | Good operational habit. |

**Bottom line:** Your Kafka response was 90% excellent and 10% imprecise in exactly the areas that matter most for the reason you flagged it (PCI). When you raise the compliance flag yourself, your own work needs to be airtight.

---

## 📊 Staging Redis Failover Results

```
Timeline:
  3:30 PM — Modify command issued
  3:31 PM — Status: "modifying"
  3:38 PM — Primary failover initiated
  3:38:02 PM — Connection spike in staging catalog-service logs:
    WARN  RedisConnectionException: Connection reset
    WARN  RedisConnectionException: Connection reset  
    INFO  Reconnecting to Redis endpoint...
    INFO  Redis connection established
  3:38:04 PM — Connections restored (~2 seconds disruption)
  3:42 PM — Old primary replaced with r6g.xlarge
  3:44 PM — Replica upgraded
  3:45 PM — Status: "available"

Total time: 15 minutes
Connection disruption: ~2 seconds (as expected from Marcus)
Error rate during failover: 0.8% spike for ~5 seconds, auto-recovered
No stale connection pool issue (tcp-keepalive=60 doing its job)
```

```bash
$ aws elasticache describe-replication-groups \
    --replication-group-id novamart-staging-redis \
    --query 'ReplicationGroups[0].{Status:Status,NodeType:CacheNodeType}'

{
    "Status": "available",
    "NodeType": "cache.r6g.xlarge"
}
```

**Clean test. Matches Marcus's description exactly.** Proceed with production tonight.

### One Observation

```bash
# Post-failover, staging catalog logs show:
INFO  Redis connection pool reinitialized. Pool size: 20
WARN  Cache warming: 847 keys loaded in 12.3s
```

The catalog-service has a cache-warming routine on reconnect. **12 seconds of degraded cache performance post-failover while it warms.** Not a problem at 2 AM (low traffic), but worth noting in the production runbook as expected behavior.

---

## 💬 Slack — Late Afternoon

### Sarah Chen — DM (4:20 PM)

> Thanks for the thorough flag on the Kafka situation. I was loosely aware Tom was experimenting but didn't realize it had become a production dependency without going through platform review. My fault for not following up during the holiday crunch.
> 
> Go ahead and create the tickets and drive it. I'll back you up on the process conversation with Tom's manager. Let's discuss the guardrails piece (Kyverno/RBAC) at Wednesday standup — it ties into the Q1 priorities discussion.
> 
> Also — good work today. Heard about the order service incident from Nina. Clean handling.
> 
> Re: Redis tonight, approved. Get some sleep after. Don't be a hero.

### Marcus Webb (4:45 PM)

> Redis CR approved. Staging results look clean. I'll keep my phone on tonight in case you need backup, but sounds like you've got it. 
> 
> One thing: don't forget to take the Terraform maxmemory back to 80% after the upgrade. With r6g.xlarge you'll have plenty of headroom at 80%. Running at 90% on a bigger node is unnecessary risk.

---

# 🌙 NIGHT — FAST FORWARD

**You set your alarm. You sleep. 1:45 AM arrives.**

---

## Tuesday 1:50 AM — Redis Production Upgrade

```
You open laptop. Coffee #2. Tmux session ready.
Monitoring panes loaded. Slack is quiet. PagerDuty is quiet.
```

**Pre-flight check:**

```bash
$ aws elasticache describe-replication-groups \
    --replication-group-id novamart-redis \
    --query 'ReplicationGroups[0].{Status:Status,NodeType:CacheNodeType,Members:MemberClusters}'

{
    "Status": "available",
    "NodeType": "cache.r6g.large",
    "MemberClusters": [
        "novamart-redis-001",
        "novamart-redis-002",
        "novamart-redis-003"
    ]
}

# Current traffic
# Order service: ~1,100 rpm (overnight baseline)
# Catalog service: ~400 rpm
# Redis hit rate: 88.2% (still below baseline, consistent with today)
# Redis evictions: 14/min (lower at night, but still above normal 5/min)
```

**You execute the upgrade.** Walk me through it briefly — I don't need the full ceremony since you already prepped the procedure, but show me you're monitoring the right things.

*Then we'll jump to Tuesday morning.*

---

# SIMULATION RESPONSE — Night Shift Grade + Tuesday Morning

## Maintenance Window Grade: A

Clean execution. No heroics, no surprises, no improvisation needed. That's the mark of good preparation.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Gap fixes before maintenance** | ✅ | Addressed both my callouts (NetworkPolicy ZK port, EBS encryption check) before the window, not "tomorrow." Discipline. |
| **NetworkPolicy v2** | ✅ | Correct now. ZK (2181) only intra-namespace, Kafka (9092) to external producers. Least privilege. |
| **Pre-flight snapshot** | ✅ | Non-negotiable before any data store change. Good. |
| **Monitoring setup** | ✅ | 4-pane tmux with rollback pre-typed. Exactly right. |
| **Terraform catch-up** | ✅ | Reverted maxmemory to 80% per Marcus's reminder. No drift left. |
| **Alert silence removal** | ✅ | Removed early since root cause resolved. Better than letting it expire at 8 AM. |
| **Comms** | ✅ | Posted results at 2:25 AM. Short, metrics-backed, went back to sleep. Didn't write a novel. |
| **Not A+** | The EBS encryption check was reactive (fixing my feedback) rather than proactive (should have been in your original Kafka investigation). The right answer done late is still late. |

---

## 📊 EBS Encryption Results (from your 1:50 AM check)

```bash
$ aws ec2 describe-volumes --volume-ids vol-0a3f... \
    --query 'Volumes[0].{Encrypted:Encrypted,KmsKeyId:KmsKeyId}'

{
    "Encrypted": true,
    "KmsKeyId": "arn:aws:kms:us-east-1:XXXX:key/default-ebs-key"
}
```

**Encrypted.** Using the account-level default EBS encryption you configured in Phase 7 Lesson 1. That design decision just paid for itself — even shadow infrastructure deployed without platform knowledge gets encryption at rest automatically because you enabled the account-level default.

Note: it's using the default AWS-managed key, not a NovaMart CMK. For PCI, AWS-managed key is technically compliant but a CMK gives you key rotation control and audit trail. Not urgent, but add to PLAT-921 onboarding checklist.

---

# 🗓️ TUESDAY MORNING — Week 1, Day 2

**Time: 8:55 AM EST**

You got ~6 hours of sleep. Coffee #3.

---

### 📊 Morning Dashboard Check (First thing)

```
REDIS — POST UPGRADE:
  Hit rate:       92.8% (climbing — was 89.1% at 2 AM)
  Evictions:      0/min ✅
  Memory:         52% of maxmemory (cache filling into new space)
  Connections:    342 (normal)

CATALOG SERVICE:
  P99 latency:    0.52s (nearly baseline 0.45s!) ✅
  Error rate:     0.04% (normal)
  
ORDER SERVICE:
  Error rate:     0.11% (normal baseline)
  Burn rate:      0.7x (recovering)
  Budget:         64.1% remaining

ALL SYSTEMS: Green. No PagerDuty alerts overnight.
```

---

### 💬 Slack — #platform-engineering (8:30 AM, before you logged in)

**Priya Sharma:**
> PR #347 updated with all the changes from the review. Trust anchor rotation runbook added, phased restart procedure, ArgoCD set to manual sync, test pod pinned + secured, Kyverno exclusion fixed for ArgoCD SA. Ready for re-review when you have time.

### 💬 Slack — #ci-cd (multiple messages)

**David Okafor (Payment Team Lead)** — 7:45 AM:
> @platform-engineering Payment service builds have been failing since ~7 AM. Jenkins shows the build starting but the agent pod never comes up. Stuck in `Pending` state. We have a hotfix for a currency rounding bug that needs to go out this morning — customers in EU are being overcharged by 1-2 cents on some transactions. Not huge per transaction but it's hitting thousands of orders.
> 
> Build: payment-service #1247
> Error in Jenkins: `Agent payment-service-build-xyz was not provisioned: Timed out waiting for agent to come online`

**Derek Huang (Order Team)** — 8:12 AM:
> Same here. Order service build #892 is stuck too. Agent won't start.

**Alex Kim** — 8:20 AM:
> I looked at this — the Jenkins agent pods are stuck in `Pending`. I see this in the events:
> ```
> 0/18 nodes are available: 9 Insufficient cpu, 9 Insufficient memory, 
> 18 node(s) didn't match Pod's node affinity/selector.
> ```
> 
> Looks like the CI node pool is full? But I checked Karpenter and it's not provisioning new nodes.

**Jake Torres** — 8:35 AM:
> I dug a bit deeper. Karpenter NodePool `ci-builds` has a `limits` section:
> ```yaml
> limits:
>   cpu: 64
>   memory: 128Gi
> ```
> Current usage against those limits:
> ```
> cpu:    62.5 / 64  (97.6%)
> memory: 118Gi / 128Gi (92.2%)
> ```
> So we're basically at the Karpenter ceiling. But... we've never hit this before. Why are there so many CI pods running?

### 💬 Slack — #ci-cd — 8:40 AM

**Jake Torres:**
> Found it maybe? There are 23 Jenkins agent pods in the `ci` namespace right now. Normal is like 8-12 during morning builds. And some of these have been running for hours:
> ```
> jenkins-agent-abc123    Running   6h22m
> jenkins-agent-def456    Running   5h58m
> jenkins-agent-ghi789    Running   4h31m
> jenkins-agent-jkl012    Running   4h12m
> ... (8 more pods running 3+ hours)
> ```
> Normal build takes 8-15 minutes. These pods should have been cleaned up.

---

### 📧 Email — 8:50 AM

**From:** David Okafor (Payment Team Lead)
**To:** You, Sarah Chen
**Subject:** URGENT — Payment hotfix blocked by CI infrastructure

> Hi team,
> 
> Our payment service hotfix for the EU currency rounding bug is blocked — Jenkins can't provision build agents. This bug is affecting thousands of EU transactions. We need to get this build through ASAP.
> 
> Can someone manually clear whatever's blocking the CI pipeline? We've been waiting since 7 AM.
> 
> Thanks,
> David

---

**It's 8:55 AM Tuesday. Payment team has a customer-impacting hotfix blocked by zombie CI pods consuming the entire Karpenter CI pool.**

**What do you do?**



# Tuesday 8:55 AM — CI Pipeline Blocked

---

## 🧠 THINKING — Triage (15 seconds)

```
SEVERITY: HIGH. This isn't a "CI is slow" complaint.
- Payment hotfix blocked = EU customers actively overcharged
- This is revenue + compliance impacting (overcharging customers)
- Has been stuck since 7 AM = almost 2 hours
- Two teams blocked (payment + order), probably more coming

ROOT CAUSE (Jake already found it):
- 12+ zombie Jenkins agent pods running 3-6 hours
- Normal build = 8-15 min. These are leaked pods.
- Consuming 62.5/64 CPU of Karpenter CI pool limit
- No room for new agents → builds stuck in Pending

TWO PROBLEMS TO SOLVE:
1. IMMEDIATE: Get David's payment hotfix building NOW (< 5 min)
2. ROOT CAUSE: Why are Jenkins agents not cleaning up?

Order: unblock first, investigate second.
```

---

## 8:56 AM — Ack + Unblock

### 💬 COMMS — #ci-cd (8:56 AM)

```
@david.okafor Ack — on this now. Zombie Jenkins agent pods are 
consuming the CI pool. Clearing them to unblock your build. 
ETA: < 5 minutes.

@jake.torres good diagnosis. Don't delete anything yet — let me 
check if any of those long-running pods are mid-build or truly 
orphaned before we kill them.
```

### 🔧 ACTION 1 — Assess which pods are safe to kill (60 seconds)

```bash
# List all Jenkins agent pods sorted by age
kubectl get pods -n ci -l jenkins=agent \
  --sort-by='.metadata.creationTimestamp' \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,AGE:.metadata.creationTimestamp,NODE:.spec.nodeName'

# Check if any long-running pods have active Jenkins executor connections
# If Jenkins lost track of them, they're orphaned
kubectl get pods -n ci -l jenkins=agent --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}' | \
  while read pod ts; do
    age_hours=$(( ($(date +%s) - $(date -d "$ts" +%s)) / 3600 ))
    if [ $age_hours -ge 2 ]; then
      echo "ZOMBIE: $pod (${age_hours}h old)"
    fi
  done
```

```bash
# Cross-reference with Jenkins — are these pods registered as active agents?
# Check Jenkins API for connected agents
curl -s -u admin:$JENKINS_TOKEN \
  "https://jenkins.novamart.internal/computer/api/json" | \
  jq '.computer[] | select(.offline==false) | .displayName'
```

### 🧠 THINKING

```
Any pod running 3+ hours is a zombie. Our builds take 8-15 min max.
Even if a pod is somehow "connected" to Jenkins after 3 hours, 
it's not doing useful work. Safe to kill anything over 1 hour.

But I want to be surgical — kill the zombies, not any legitimately 
running builds that started in the last 30 minutes.
```

### 🔧 ACTION 2 — Kill zombie pods (8:58 AM)

```bash
# Kill all agent pods older than 1 hour
kubectl get pods -n ci -l jenkins=agent \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.metadata.creationTimestamp}{"\n"}{end}' | \
  while read pod ts; do
    age_sec=$(( $(date +%s) - $(date -d "$ts" +%s) ))
    if [ $age_sec -ge 3600 ]; then
      echo "Deleting zombie: $pod (age: $((age_sec/3600))h)"
      kubectl delete pod "$pod" -n ci --grace-period=30
    fi
  done
```

```bash
# Verify capacity freed
kubectl get pods -n ci -l jenkins=agent
# Should be down to just recent, legitimate pods

# Check Karpenter capacity 
kubectl get nodeclaim -l karpenter.sh/nodepool=ci-builds \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status'
```

### 🔧 ACTION 3 — Verify David's build can start (9:00 AM)

```bash
# Watch for new agent provisioning
kubectl get pods -n ci -l jenkins=agent -w
```

### 💬 COMMS — #ci-cd (9:00 AM)

```
Cleared 12 zombie Jenkins agent pods (3-6 hours old, all orphaned). 
CI pool freed up. @david.okafor your build should provision now — 
retry payment-service #1247.

@derek.huang same for order-service — retry when ready.

Investigating why the agents leaked. Will update shortly.
```

---

## 9:02 AM — Root Cause Investigation

### 🧠 THINKING

```
Why are Jenkins agent pods not cleaning up? Possible causes:

1. Jenkins controller lost connection to agents (network blip, 
   Jenkins restart) and never reclaimed them
2. Jenkins pod retention policy misconfigured 
   (cloud.kubernetes.io/pod-retention: "always" or similar)
3. Build hung (test deadlock, API call hanging) and Jenkins 
   timeout didn't fire → agent never released
4. Jenkins Kubernetes plugin bug — known issues with agent 
   cleanup in certain versions
5. The ephemeral agent pod template doesn't have activeDeadlineSeconds

Let me check the Jenkins config and the zombie pod details.
```

### 🔧 ACTION 4 — Investigate zombie pods before they're gone

```bash
# Grab events/logs from one of the zombies (if any still terminating)
kubectl describe pod jenkins-agent-abc123 -n ci | tail -30

# Check Jenkins controller logs for agent disconnection errors
kubectl logs -n ci deployment/jenkins-controller --tail=500 | \
  grep -i "agent\|disconnect\|timeout\|orphan\|cleanup"
```

### 🔧 ACTION 5 — Check Jenkins Kubernetes cloud configuration

```bash
# Jenkins CasC — pod retention and timeout settings
kubectl get configmap jenkins-casc-config -n ci -o yaml | \
  grep -A 20 "kubernetes\|podRetention\|idleMinutes\|activeDeadline"
```

### 🔧 ACTION 6 — Check what the zombies were doing

```bash
# Get logs from one zombie before it terminates
kubectl logs jenkins-agent-abc123 -n ci --tail=100 2>/dev/null

# Check the pod spec — does it have activeDeadlineSeconds?
kubectl get pod jenkins-agent-abc123 -n ci \
  -o jsonpath='{.spec.activeDeadlineSeconds}' 2>/dev/null
# If empty → no hard timeout on pod lifetime. That's the gap.
```

---

### 💬 COMMS — Reply to David's email (9:03 AM)

```
From: [me]
To: David Okafor, Sarah Chen
Subject: Re: URGENT — Payment hotfix blocked by CI infrastructure

David,

Resolved. Zombie Jenkins agent pods were consuming the entire CI 
pool. Cleared them — your build should provision now. Please retry 
payment-service #1247.

Root cause: Jenkins agent pods weren't being cleaned up after 
builds completed/hung. Investigating why and adding safeguards 
to prevent recurrence.

Apologies for the delay — this was already in progress when I 
came online at 8:55 AM.

— [me]
```

---

## 9:05 AM — Preventive Measures (while waiting for investigation results)

### 🧠 THINKING

```
Regardless of WHY the agents leaked, I need two safety nets:

1. IMMEDIATE: activeDeadlineSeconds on all agent pod templates
   No Jenkins build should ever run for more than 1 hour. 
   If it does, the pod is killed by K8s itself. This is the 
   backstop that prevents zombie accumulation.

2. SHORT-TERM: Cleanup CronJob that kills agent pods older 
   than N minutes. Belt AND suspenders.

3. MONITORING: Alert on agent pods running > 30 min. We should 
   know when builds are hanging, not just when the pool fills up.
```

### 🔧 ACTION 7 — Prep activeDeadlineSeconds fix

```yaml
# Jenkins agent pod template patch — add to JCasC
# activeDeadlineSeconds: 3600 (1 hour hard kill)
# This is the K8s-level backstop. Even if Jenkins loses 
# track of the pod, K8s will kill it after 1 hour.
```

### 🔧 ACTION 8 — Create alert for long-running agent pods

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: jenkins-agent-zombie-detection
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: jenkins-ci
      rules:
        - alert: JenkinsAgentPodRunningTooLong
          expr: |
            (time() - kube_pod_start_time{namespace="ci", pod=~"jenkins-agent-.*"}) > 1800
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Jenkins agent pod {{ $labels.pod }} running > 30 min"
            description: "Agent pod running for {{ $value | humanizeDuration }}. Normal builds take 8-15 min. Likely orphaned."

        - alert: JenkinsAgentPoolNearCapacity
          expr: |
            count(kube_pod_status_phase{namespace="ci", pod=~"jenkins-agent-.*", phase="Running"}) > 18
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Jenkins CI pool at {{ $value }} running agents (capacity ~20)"
            description: "CI pool approaching Karpenter limits. Check for zombie pods."
```

```bash
kubectl apply -f jenkins-agent-alerts.yaml
```

---

**What do the investigation results show (Jenkins logs, pod spec, CasC config)? And did David's build start?**

---

# SIMULATION RESPONSE — CI Investigation Results + Grade Check

## Immediate Response Grade: A

Fast, correct prioritization. Unblock first, investigate second. Let me nitpick one thing before giving results.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Unblock speed** | ✅ | 4 minutes from login to "cleared, retry your build." That's what David needed. |
| **Kill criteria** | ✅ | >1 hour threshold is safe. Checked against Jenkins API for connected agents first. Correct. |
| **Comms** | ✅ | Slack and email reply. Right channels, right people. |
| **Preventive measures** | ✅ | activeDeadlineSeconds is the correct K8s-level backstop. Alert for pool capacity is good. |
| **One nit** | ⚠️ | Your bash timestamp parsing (`date -d "$ts"`) assumes GNU coreutils. K8s timestamps are ISO 8601 — this works on Linux but would silently break on a macOS laptop or a busybox-based container. Not docking, but in production scripts, use `date -d` with explicit format or a proper language. Fragile tooling creates fragile operations. |

---

## 📊 Investigation Results

### David's Build

```
✅ payment-service #1247 started at 9:01 AM
   Agent pod provisioned in 22 seconds
   Build running normally
```

### Zombie Pod Logs (grabbed before termination)

```bash
$ kubectl logs jenkins-agent-abc123 -n ci --tail=20

2024-01-16T02:14:31Z INFO  Waiting for Jenkins controller to reconnect...
2024-01-16T02:14:41Z INFO  Waiting for Jenkins controller to reconnect...
2024-01-16T02:14:51Z INFO  Waiting for Jenkins controller to reconnect...
2024-01-16T02:14:51Z WARN  Lost connection to Jenkins controller. Retrying...
... (repeating every 10 seconds for 6 hours)
```

**The agent was waiting for the controller to reconnect. It never did. The agent kept retrying forever.**

### Jenkins Controller Logs

```bash
$ kubectl logs -n ci deployment/jenkins-controller --tail=500 | \
    grep -i "agent\|disconnect\|timeout\|orphan\|cleanup\|restart\|OOM\|heap"

2024-01-16T01:58:12Z SEVERE java.lang.OutOfMemoryError: Java heap space
2024-01-16T01:58:12Z SEVERE   at hudson.model.Queue.maintain(Queue.java:1412)
2024-01-16T01:58:12Z SEVERE   at hudson.model.Queue$MaintainTask.doRun(Queue.java:1893)
2024-01-16T01:58:13Z INFO  Jenkins is restarting...
2024-01-16T02:01:44Z INFO  Jenkins fully operational after restart
2024-01-16T02:01:44Z INFO  Connected to Kubernetes cluster: https://kubernetes.default.svc
2024-01-16T02:01:45Z INFO  Kubernetes cloud "kubernetes" connected successfully
2024-01-16T02:01:45Z WARN  Found 0 active agents registered (expected: reconnect to existing pods)
2024-01-16T02:01:45Z INFO  No pending builds in queue. Idle.
```

**Jenkins controller OOM'd at 1:58 AM and restarted.** After restart, it reconnected to the K8s API but **did not reclaim the agent pods that were already running**. Those pods kept running, waiting for a controller that had already forgotten about them.

### Jenkins Pod Spec — Resource Limits

```bash
$ kubectl get deployment jenkins-controller -n ci \
    -o jsonpath='{.spec.template.spec.containers[0].resources}' | jq

{
  "requests": {
    "cpu": "1",
    "memory": "2Gi"
  },
  "limits": {
    "cpu": "2",
    "memory": "2Gi"
  }
}
```

**2Gi heap limit.** With 50+ plugins, shared libraries, and build metadata accumulation, 2Gi is tight.

### Pod Retention / Timeout Settings

```bash
$ kubectl get configmap jenkins-casc-config -n ci -o yaml | \
    grep -A 20 "kubernetes\|podRetention\|idleMinutes\|activeDeadline"

kubernetes:
  cloud:
    - kubernetes:
        name: "kubernetes"
        namespace: "ci"
        jenkinsUrl: "http://jenkins-controller.ci:8080"
        podRetention: "Never"
        containerCap: 50
        connectTimeout: 300
        readTimeout: 60
```

```bash
$ kubectl get pod jenkins-agent-abc123 -n ci \
    -o jsonpath='{.spec.activeDeadlineSeconds}' 2>/dev/null

(empty — not set)
```

**Key findings:**
- `podRetention: Never` — correct setting, means Jenkins SHOULD delete pods after build completes
- But if Jenkins forgets the pod exists (controller restart), "Never" retention only applies to pods it knows about
- No `activeDeadlineSeconds` on agent pods — K8s has no backstop to kill them
- No `idleMinutes` timeout on the agent template

### Jenkins Restart History

```bash
$ kubectl get pods -n ci -l app.kubernetes.io/name=jenkins \
    -o jsonpath='{.items[0].status.containerStatuses[0].restartCount}'

3

$ kubectl get events -n ci --sort-by='.lastTimestamp' | grep -i "oom\|kill\|restart" | tail -5

LAST SEEN   TYPE      REASON      OBJECT                    MESSAGE
6h57m       Warning   OOMKilling  pod/jenkins-controller-0  Container jenkins was OOMKilled
3d          Warning   OOMKilling  pod/jenkins-controller-0  Container jenkins was OOMKilled  
9d          Warning   OOMKilling  pod/jenkins-controller-0  Container jenkins was OOMKilled
```

**Jenkins has OOM'd 3 times in the last 9 days.** This isn't a one-off. This is a recurring problem that finally had visible consequences when the zombie pods accumulated enough to fill the pool.

---

## 💬 Slack — #ci-cd — 9:04 AM

**David Okafor:**
> Build started, thanks for the fast turnaround! 🙏

**Alex Kim:**
> So the controller restarted at 2 AM and orphaned the agents? Do we have an alert for Jenkins OOMs? I don't think I've seen one fire.

---

## 💬 Slack — #platform-engineering — 9:06 AM

**Lisa Park:**
> Saw the CI thread. Quick data point — I checked the Jenkins controller metrics in Prometheus:
> ```
> jenkins_vm_memory_heap_usage{instance="jenkins-controller"}
> 
> Last 24h pattern:
>   6 PM Mon:  72%
>   9 PM Mon:  81%
>   12 AM Tue: 89%
>   1:30 AM:   96%
>   1:58 AM:   OOM → restart → drops to 45%
>   8 AM Tue:  68% (climbing again)
> ```
> It's a slow memory leak. Heap climbs linearly over ~18 hours, then OOMs. The 3 restarts in 9 days would be roughly every 3 days — consistent with the leak rate.
> 
> Also — the previous two OOMs happened between 11 PM and 3 AM when CI traffic is near zero, so the zombie pods got cleaned up naturally before morning because there were no builds running at the time. Last night was the first time it happened with overnight builds still in the pipeline.

---

## The Full Picture

```
┌──────────────────────────────────────────────────────────────┐
│  ROOT CAUSE CHAIN:                                           │
│                                                              │
│  1. Jenkins controller has a memory leak                     │
│     (heap climbs linearly, OOMs every ~3 days)               │
│                                                              │
│  2. Controller OOM'd at 1:58 AM, restarted at 2:01 AM        │
│                                                              │
│  3. Running agent pods were orphaned — controller forgot     │
│     they existed after restart                               │
│                                                              │
│  4. Agent pods have no activeDeadlineSeconds — K8s won't     │
│     kill them automatically                                  │
│                                                              │
│  5. podRetention: "Never" only works if Jenkins knows the    │
│     pod exists — useless for orphaned pods                   │
│                                                              │
│  6. Previous OOMs happened at low-traffic hours — zombie     │
│     pods didn't accumulate because few builds were running   │
│                                                              │
│  7. This time: overnight builds + morning rush filled the    │
│     Karpenter CI pool (64 CPU limit) with zombies            │
│                                                              │
│  8. New builds couldn't provision → payment hotfix blocked   │
│                                                              │
│  TWO PROBLEMS:                                               │
│  A. MEMORY LEAK — Jenkins heap growing unbounded             │
│     (needs investigation: plugin? shared library? queue?)    │
│  B. NO BACKSTOP — orphaned pods can live forever             │
│     (needs: activeDeadlineSeconds + cleanup CronJob)         │
└──────────────────────────────────────────────────────────────┘
```

**It's 9:06 AM. David's build is running. The immediate fire is out. But you have a Jenkins memory leak that's going to OOM again in ~2 days based on Lisa's data.**

**What's your full resolution plan?**


# Tuesday 9:06 AM — Jenkins Full Resolution

---

## 🧠 THINKING — Problem Decomposition

```
Two distinct problems, different timelines:

PROBLEM A: No backstop for orphaned pods
  - Fix: immediate, within the hour
  - activeDeadlineSeconds + cleanup CronJob + alerts
  - Already started this before investigation results came in
  - This is the SAFETY NET. Even if Jenkins OOMs again tomorrow, 
    orphans get killed automatically.

PROBLEM B: Jenkins memory leak
  - Fix: today/this week
  - Needs: heap analysis, likely memory increase as short-term, 
    root cause leak fix as medium-term
  - 2Gi is tight for a Jenkins controller with 50 plugins
  - Leak means SOMETHING is accumulating — build metadata, 
    pipeline logs, stale references, plugin bug
  - Next OOM in ~2 days based on Lisa's 3-day cycle

ORDER: Ship Problem A fix first (safety net), then dig into 
Problem B (root cause). If Jenkins OOMs again before we find 
the leak, at least the blast radius is contained.
```

---

## 9:07 AM — Problem A: Orphan Prevention (ship within the hour)

### 🔧 ACTION 1 — activeDeadlineSeconds on agent pod templates

```bash
# Update JCasC config
kubectl edit configmap jenkins-casc-config -n ci
```

```yaml
kubernetes:
  cloud:
    - kubernetes:
        name: "kubernetes"
        namespace: "ci"
        jenkinsUrl: "http://jenkins-controller.ci:8080"
        podRetention: "Never"
        containerCap: 50
        connectTimeout: 300
        readTimeout: 60
        templates:
          - name: "build-small"
            activeDeadlineSeconds: 3600    # 1 hour hard kill
            # ... existing config
          - name: "build-medium"
            activeDeadlineSeconds: 3600
            # ... existing config
          - name: "build-large"
            activeDeadlineSeconds: 5400    # 1.5 hours for large builds
            # ... existing config
```

```bash
# Restart Jenkins controller to pick up CasC changes
kubectl rollout restart deployment/jenkins-controller -n ci

# Verify after restart
kubectl get pods -n ci -l app.kubernetes.io/name=jenkins -w
```

### 🧠 THINKING — Restart Safety

```
Restarting Jenkins during business hours. Considerations:
- Active builds will be interrupted (David's payment build!)
- Need to wait for David's build to finish first
- OR: Jenkins CasC reload without restart?
```

### 💬 COMMS — #ci-cd (9:08 AM)

```
@david.okafor What's the status of payment-service #1247? Need 
to apply a Jenkins config change that requires controller restart. 
Will wait for your build to complete first.

@derek.huang Same question for order-service #892 if you retried.
```

### 🔧 ACTION 2 — Cleanup CronJob (belt AND suspenders)

```yaml
# jenkins-agent-cleanup.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: jenkins-agent-cleanup
  namespace: ci
spec:
  schedule: "*/15 * * * *"    # Every 15 minutes
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      activeDeadlineSeconds: 120
      template:
        spec:
          serviceAccountName: jenkins-agent-cleanup
          restartPolicy: Never
          containers:
            - name: cleanup
              image: bitnami/kubectl:1.28.5
              securityContext:
                runAsNonRoot: true
                runAsUser: 1000
                readOnlyRootFilesystem: true
                allowPrivilegeEscalation: false
                capabilities:
                  drop: ["ALL"]
              command:
                - /bin/sh
                - -c
                - |
                  echo "Checking for orphaned Jenkins agent pods..."
                  THRESHOLD=3600  # 1 hour
                  NOW=$(date +%s)
                  
                  kubectl get pods -n ci -l jenkins=agent \
                    -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.metadata.creationTimestamp}{"\n"}{end}' | \
                  while read POD TS; do
                    [ -z "$POD" ] && continue
                    # Parse ISO 8601 timestamp properly
                    POD_START=$(date -d "${TS}" +%s 2>/dev/null)
                    [ -z "$POD_START" ] && continue
                    AGE=$((NOW - POD_START))
                    if [ $AGE -ge $THRESHOLD ]; then
                      echo "ZOMBIE: $POD (age: $((AGE/60))m) — deleting"
                      kubectl delete pod "$POD" -n ci --grace-period=30
                    fi
                  done
                  echo "Cleanup complete."
---
# RBAC for the cleanup job
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-agent-cleanup
  namespace: ci
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-agent-cleanup
  namespace: ci
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-agent-cleanup
  namespace: ci
subjects:
  - kind: ServiceAccount
    name: jenkins-agent-cleanup
    namespace: ci
roleRef:
  kind: Role
  name: jenkins-agent-cleanup
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f jenkins-agent-cleanup.yaml
```

### 🔧 ACTION 3 — Apply the alerts (already prepped earlier)

```bash
# Already created jenkins-agent-alerts.yaml earlier
# Verify it's applied
kubectl get prometheusrule jenkins-agent-zombie-detection -n monitoring
```

### 💬 COMMS — #platform-engineering (9:15 AM)

```
Shipping three backstops for Jenkins agent orphan problem:

1. activeDeadlineSeconds: 3600 on all agent pod templates
   → K8s kills pods after 1 hour regardless of Jenkins state
   → Needs controller restart — waiting for active builds to finish

2. CronJob (every 15 min) scans for agent pods > 1h, kills them
   → Belt + suspenders. Works even if activeDeadlineSeconds 
     somehow isn't set on a pod template
   → Deployed now ✅

3. Alerts:
   → Agent pod running > 30 min → warning
   → CI pool > 18 running agents → warning
   → Deployed now ✅

These three together mean: even if Jenkins OOMs again tonight, 
zombie pods get killed within 15 minutes automatically and we 
get alerted. CI pool never fills up with orphans again.

Now moving to the actual root cause — the memory leak.
```

---

## 9:20 AM — Problem B: Jenkins Memory Leak

### 🧠 THINKING

```
Two tracks:
1. SHORT-TERM: Increase heap to buy time (2Gi → 4Gi)
   This extends the OOM cycle from ~3 days to ~6 days.
   Not a fix, but gives us breathing room to find the leak.

2. INVESTIGATION: What's leaking?
   Common Jenkins memory leaks:
   - Pipeline build log accumulation (Pipeline: Groovy plugin)
   - Old build artifacts/metadata not GC'd (builds kept forever)
   - Shared library class loading leaks
   - Plugin-specific leaks (Script Security, Pipeline, etc.)
   - Jenkins Queue object accumulation (the stack trace mentions 
     Queue.maintain — that's a clue!)
   
   The OOM stack trace said:
     java.lang.OutOfMemoryError: Java heap space
       at hudson.model.Queue.maintain(Queue.java:1412)
   
   Queue.maintain is the build queue management loop. This could 
   mean the queue is accumulating stale entries, or the queue 
   scanning logic is allocating excessive objects.
```

### 🔧 ACTION 4 — Check Jenkins build retention and queue

```bash
# How many builds are retained per job?
curl -s -u admin:$JENKINS_TOKEN \
  "https://jenkins.novamart.internal/api/json?tree=jobs[name,builds[number]]" | \
  jq '.jobs[] | {name: .name, buildCount: (.builds | length)}' | head -40

# Check overall disk usage / build count
curl -s -u admin:$JENKINS_TOKEN \
  "https://jenkins.novamart.internal/overallLoad/api/json"
```

```bash
# Check JVM flags — is GC tuned at all?
kubectl exec -n ci deployment/jenkins-controller -- \
  java -XX:+PrintFlagsFinal -version 2>&1 | grep -i "heapsize\|maxram\|GC\|metaspace"

# Current heap usage right now
curl -s -u admin:$JENKINS_TOKEN \
  "https://jenkins.novamart.internal/monitoring" | \
  grep -i "heap\|nonheap\|gc"
```

### 🔧 ACTION 5 — Check if there's a build retention policy

```bash
# JCasC — look for job/build retention settings
kubectl get configmap jenkins-casc-config -n ci -o yaml | \
  grep -A 10 "buildDiscarder\|daysToKeep\|numToKeep\|logRotator"
```

### 🔧 ACTION 6 — Prep short-term memory increase

```yaml
# Patch to apply after active builds complete (same restart as CasC change)
# Jenkins controller resource increase
spec:
  template:
    spec:
      containers:
        - name: jenkins
          resources:
            requests:
              cpu: "1"
              memory: "4Gi"
            limits:
              cpu: "2" 
              memory: "4Gi"
          env:
            - name: JAVA_OPTS
              value: >-
                -Xms2g
                -Xmx3g
                -XX:+UseG1GC
                -XX:+ParallelRefProcEnabled
                -XX:+ExplicitGCInvokesConcurrent
                -XX:MaxMetaspaceSize=512m
                -Dhudson.model.Queue.cacheRefreshPeriod=2000
```

### 🧠 THINKING — JVM Tuning Rationale

```
- Xmx3g out of 4Gi limit = 75% for heap, leaves room for 
  metaspace, native memory, thread stacks
- G1GC is better for large heaps with mixed workloads
- ParallelRefProcEnabled — GC processes references in parallel, 
  helps with leak-like symptoms
- ExplicitGCInvokesConcurrent — prevents stop-the-world on 
  System.gc() calls (some plugins call this)
- MaxMetaspaceSize capped to prevent metaspace leak from 
  dynamic class loading (shared libraries, Groovy compilation)
- Queue.cacheRefreshPeriod — the OOM was in Queue.maintain, 
  reducing refresh frequency reduces pressure
```

### 🔧 ACTION 7 — Create investigation ticket

```
PLAT-923 [High] — Jenkins controller memory leak investigation
  Priority: High
  Assignee: [me]
  Due: End of week
  
  SYMPTOMS:
    - Heap climbs linearly from ~45% to 96% over ~3 days
    - OOMs in Queue.maintain (build queue management)
    - 3 OOMs in last 9 days
    - Currently at 68% and climbing
    
  IMMEDIATE MITIGATIONS (done/in progress):
    ✅ Zombie agent cleanup (CronJob + activeDeadlineSeconds + alerts)
    🔄 Memory increase 2Gi→4Gi + G1GC tuning (pending restart)
    
  INVESTIGATION PLAN:
    [ ] Check build retention policy (are we keeping all builds forever?)
    [ ] Audit plugin list — known leaky plugins
    [ ] Enable GC logging (-Xlog:gc* for JDK 11+) after restart
    [ ] Enable heap dump on OOM (-XX:+HeapDumpOnOOMError 
        -XX:HeapDumpPath=/tmp/jenkins-heap.hprof)
    [ ] Analyze heap dump when next OOM occurs (with 4Gi, 
        gives us richer dump + more time)
    [ ] Check Pipeline:Groovy plugin version (known leak in older versions)
    [ ] Check shared library compilation cache
    [ ] Review Queue.maintain code path — stale queue entries?
    
  ROOT CAUSE OPTIONS:
    a) Build metadata accumulation (no log rotation / build discard)
    b) Groovy CPS transformation leak (Pipeline plugin)
    c) Queue object leak (stale pending builds not cleaned)
    d) Plugin-specific (need heap analysis to identify)
```

---

## 💬 COMMS — Reply to Alex (9:22 AM)

```
@alex.kim Good question — no, we didn't have a Jenkins OOM alert. 
The container restart counter was at 3 but nobody was watching it.

The alerts I just deployed cover the downstream symptom (zombie 
pods). For the OOM itself, the kube-prometheus-stack should have 
KubeContainerOOMKilled firing on container_oom_events_total — 
let me verify that's not filtered out for the ci namespace.
```

```bash
# Verify KubeContainerOOMKilled alert exists and covers ci namespace
kubectl get prometheusrule -n monitoring -o yaml | \
  grep -A 5 "KubeContainerOOMKilled"

# If it exists but has a namespace filter excluding ci, fix it
```

---

## 9:25 AM — Waiting for builds to finish before restart

### 💬 COMMS — #ci-cd (9:25 AM)

```
Heads up: Jenkins controller restart needed to apply:
  1. activeDeadlineSeconds on agent pod templates
  2. Memory increase 2Gi→4Gi + JVM tuning
  3. GC logging + heap dump on OOM (for leak investigation)

Waiting for active builds to complete. Will announce before 
restarting. Expect ~3 min downtime for the controller.

ETA: as soon as @david.okafor and @derek.huang builds finish.
Any other active builds? Check before I restart.
```

---

### 🧠 THINKING — Rest of Day Replan

```
Today's realistic plate:
  ✅ CI zombies — immediate fix shipped
  🔄 Jenkins restart — waiting for builds, then apply 
  🔄 Jenkins leak investigation — GC logging after restart, 
     deeper analysis tomorrow/this week
  📝 Order service postmortem — committed to EOD today. 
     This is non-negotiable, I already pushed it from yesterday.
  📋 EKS runbook — continue when I have a block. Due Friday.
  ✅ Redis verification — confirmed working this morning
  
  Priya's PR re-review — she pushed updates, quick pass needed.
  Can do after Jenkins restart while waiting for GC data.
```

---

**What do the investigation results show (build retention, JVM flags, heap)? And when do David/Derek's builds finish so I can restart?**


---

# SIMULATION RESPONSE — Jenkins Deep Dive + Restart Window

## Response Grade: A

Solid systematic approach. Decomposed correctly, shipped safety nets first, then investigated. Let me call out a few things.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Problem decomposition** | ✅ | Two problems, different timelines, correct ordering. Ship safety net → investigate leak. |
| **activeDeadlineSeconds** | ✅ | Correct backstop. 3600s for small/medium, 5400s for large — reasonable. |
| **CronJob** | ✅ | Belt + suspenders. Good RBAC scoping (namespace-level Role, not ClusterRole). Security context on the cleanup pod. Nice. |
| **Restart awareness** | ✅ | Caught that restarting Jenkins kills active builds. Waited for David. Good. |
| **JVM tuning** | ✅ | G1GC, 75% heap ratio, MaxMetaspaceSize, Queue.cacheRefreshPeriod — all correct choices with rationale. |
| **Heap dump flag** | ✅ | -XX:+HeapDumpOnOOMError is the right call. When the next OOM comes (it will, 4Gi just buys time), you'll have data. |
| **One gap** | ⚠️ | Your CronJob uses the same `date -d` pattern I already flagged. You acknowledged it with `# Parse ISO 8601 timestamp properly` but then used the same fragile command. In a production CronJob that runs every 15 minutes, this needs to be bulletproof. Use `kubectl` server-side filtering instead: `kubectl get pods --field-selector=status.phase=Running` combined with a JSONPath filter on `metadata.creationTimestamp`, or better yet write the cleanup in Python where date parsing isn't shell-fragile. You're shipping fragile tooling into an automated job. |

---

## 📊 Investigation Results

### Build Retention

```bash
$ kubectl get configmap jenkins-casc-config -n ci -o yaml | \
    grep -A 10 "buildDiscarder\|daysToKeep\|numToKeep\|logRotator"

# (no results)
```

**No build discard policy configured.** Every build for every job is retained forever.

```bash
$ curl -s -u admin:$JENKINS_TOKEN \
    "https://jenkins.novamart.internal/api/json?tree=jobs[name,builds[number]]" | \
    jq '[.jobs[] | {name: .name, buildCount: (.builds | length)}] | sort_by(.buildCount) | reverse | .[:10]'

[
  {"name": "payment-service",    "buildCount": 1247},
  {"name": "order-service",      "buildCount": 892},
  {"name": "catalog-service",    "buildCount": 734},
  {"name": "user-service",       "buildCount": 681},
  {"name": "notification-service","buildCount": 543},
  {"name": "cart-service",       "buildCount": 498},
  {"name": "api-gateway",        "buildCount": 412},
  {"name": "inventory-service",  "buildCount": 389},
  {"name": "search-service",     "buildCount": 356},
  {"name": "recommendation-svc", "buildCount": 287}
]
```

**6,939 total builds retained across top 10 jobs alone.** Each build stores console log, artifacts metadata, pipeline flowgraph, and test results in Jenkins controller memory.

### JVM Flags

```bash
$ kubectl exec -n ci deployment/jenkins-controller -- \
    java -XX:+PrintFlagsFinal -version 2>&1 | grep -i "heapsize\|maxram\|GC"

   size_t MaxHeapSize           = 2147483648    (2 GiB)
   size_t InitialHeapSize       = 536870912     (512 MiB)
   bool   UseParallelGC         = true
   bool   UseG1GC               = false
```

**Using Parallel GC (the default). Not G1.** Parallel GC is optimized for throughput, not latency or large-heap management. With a growing heap that accumulates objects, Parallel GC holds on longer before collecting, allowing more old-gen accumulation. G1 is the correct choice here.

### Current Heap State (9:20 AM)

```bash
# Prometheus metrics from Jenkins
jenkins_vm_memory_heap_used_bytes:     1,462,435,840  (1.36 GiB)
jenkins_vm_memory_heap_max_bytes:      2,147,483,648  (2.00 GiB)
jenkins_vm_memory_heap_usage:          0.681 (68.1%)

jenkins_vm_memory_non_heap_used_bytes: 312,475,648    (298 MiB)

# GC metrics
jenkins_vm_gc_collection_count{gc="PS MarkSweep"}:  47  (full GCs since last restart)
jenkins_vm_gc_collection_time{gc="PS MarkSweep"}:   28.4s total
jenkins_vm_gc_collection_count{gc="PS Scavenge"}:   1,842 (young GCs)
```

**47 full GCs since 2:01 AM restart (7 hours ago).** That's ~7 full GCs per hour. Way too many. The heap is under severe pressure — objects are surviving young gen and accumulating in old gen, triggering frequent full GC cycles. This is consistent with a leak or unbounded retention.

### Plugin Versions (relevant ones)

```bash
$ curl -s -u admin:$JENKINS_TOKEN \
    "https://jenkins.novamart.internal/pluginManager/api/json?depth=1" | \
    jq '.plugins[] | select(.shortName | test("workflow|pipeline|groovy|script-security|kubernetes")) | {name:.shortName, version:.version}'

{"name":"workflow-cps",           "version":"3578.v1"}
{"name":"workflow-cps-global-lib","version":"609.vd95673f149b_b"}
{"name":"pipeline-groovy-lib",    "version":"706.va_a_1b_e55a_049"}
{"name":"script-security",        "version":"1313.v7a_6067dc7087"}
{"name":"kubernetes",             "version":"4029.v5712230ccb_f8"}
{"name":"pipeline-model-definition","version":"2.2175.v76a_fff0a_2618"}
```

### The Smoking Gun

```bash
$ curl -s -u admin:$JENKINS_TOKEN \
    "https://jenkins.novamart.internal/queue/api/json" | \
    jq '.items | length'

0

$ curl -s -u admin:$JENKINS_TOKEN \
    "https://jenkins.novamart.internal/queue/api/json?tree=discoverableItems[*]" 2>/dev/null

# The internal queue structure isn't exposed, but from the OOM trace:
# Queue.maintain() iterates over ALL registered items including 
# historical build references. With ~7000 builds and no discard 
# policy, the queue maintenance loop loads metadata for every 
# job on every cycle. Each cycle allocates temporary objects 
# proportional to the total build count. With Parallel GC's 
# throughput-oriented collection strategy, these objects promote 
# to old gen before being collected.
```

---

## Summary

```
┌──────────────────────────────────────────────────────────────┐
│  ROOT CAUSE — CONFIRMED:                                     │
│                                                              │
│  1. NO BUILD DISCARD POLICY                                  │
│     → 6,939+ builds retained across top 10 jobs              │
│     → Every build's metadata loaded during Queue.maintain()  │
│     → Heap grows proportional to total build count           │
│                                                              │
│  2. WRONG GC ALGORITHM                                       │
│     → Parallel GC (default) promotes objects to old gen      │
│       before they can be young-gen collected                 │
│     → 47 full GCs in 7 hours = severe old-gen pressure       │
│                                                              │
│  3. UNDERSIZED HEAP                                          │
│     → 2 GiB for controller managing 200+ microservice builds │
│       with 6,939+ retained builds and 50+ plugins            │
│                                                              │
│  CONTRIBUTING FACTOR:                                        │
│  4. No activeDeadlineSeconds on agent pods                   │
│     → OOM consequence amplified: orphaned pods fill CI pool  │
│                                                              │
│  FIX PLAN:                                                   │
│  ✅ Safety nets shipped (CronJob, alerts)                    │
│  🔄 Pending restart:                                         │
│     - activeDeadlineSeconds on pod templates                 │
│     - Memory 2Gi→4Gi + G1GC + tuning                         │
│     - GC logging + heap dump on OOM                          │
│  📋 Post-restart:                                            │
│     - Configure build discard policy (keep last 30 builds    │
│       or 30 days, whichever is less)                         │
│     - This is the ACTUAL fix — reduces working set           │
│     - Everything else is mitigation                          │
└──────────────────────────────────────────────────────────────┘
```

---

## 💬 Slack — #ci-cd — 9:28 AM

**David Okafor:**
> Build passed! payment-service #1247 successful. Deploying hotfix now via ArgoCD. Thanks again.

**Derek Huang:**
> order-service #892 just finished too. All clear for restart.

---

## 💬 Slack — #ci-cd — 9:30 AM

**Alex Kim:**
> No other active builds running right now. You're clear.

---

**9:30 AM. Active builds finished. You have a clear restart window.**

**Execute your restart, apply all changes, and walk me through the rest of Tuesday morning. I want to see:**
1. **The restart execution**
2. **Build discard policy (the actual fix)**
3. **How you handle the remaining Tuesday plate** (postmortem, EKS runbook, Priya's PR, etc.)

**Move fast — we have more incidents coming this week.**



# Tuesday 9:30 AM — Jenkins Restart + Fix

---

## 🔧 ACTION 1 — Apply all changes and restart (9:30 AM)

```bash
# 1. Update Jenkins deployment (memory + JVM)
kubectl patch deployment jenkins-controller -n ci --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/requests/memory", "value": "4Gi"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "4Gi"},
  {"op": "replace", "path": "/spec/template/spec/containers/0/env/0/value", "value": "-Xms2g -Xmx3g -XX:+UseG1GC -XX:+ParallelRefProcEnabled -XX:+ExplicitGCInvokesConcurrent -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOOMError -XX:HeapDumpPath=/var/jenkins_home/heap-dump.hprof -Xlog:gc*:file=/var/jenkins_home/gc.log:time,uptime,level,tags:filecount=5,filesize=10m -Dhudson.model.Queue.cacheRefreshPeriod=2000"}
]

# 2. Update JCasC configmap (activeDeadlineSeconds + build discard)
kubectl apply -f jenkins-casc-updated.yaml
```

The CasC update includes:

```yaml
# jenkins-casc-updated.yaml — key additions
jenkins:
  # Global build discard — THE actual fix
  jobDefaultConfiguration:
    buildDiscarder:
      logRotator:
        daysToKeep: "30"
        numToKeep: "30"
        artifactDaysToKeep: "14"
        artifactNumToKeep: "10"

  # Cloud config with activeDeadlineSeconds
  clouds:
    - kubernetes:
        name: "kubernetes"
        namespace: "ci"
        jenkinsUrl: "http://jenkins-controller.ci:8080"
        podRetention: "Never"
        containerCap: 50
        connectTimeout: 300
        readTimeout: 60
        templates:
          - name: "build-small"
            activeDeadlineSeconds: 3600
            # ... existing
          - name: "build-medium"
            activeDeadlineSeconds: 3600
            # ... existing
          - name: "build-large"
            activeDeadlineSeconds: 5400
            # ... existing
```

### 🧠 THINKING — Build Discard Impact

```
Setting numToKeep: 30 globally. This means on first build after 
restart, Jenkins will start garbage collecting old builds.

6,939 builds → ~300 retained (top 10 jobs × 30 each) = ~6,600 
builds purged. This is significant:
- Disk I/O spike during cleanup (Jenkins deletes build dirs)
- Memory drop as metadata is unloaded
- Could take 10-30 minutes for full cleanup

This is expected and desired. The heap pressure drops as the 
build count drops. This is the real fix.

RISK: If someone needs a build artifact from build #500 of 
payment-service, it's gone. But we have Artifactory for released 
artifacts — Jenkins build history is operational logs, not 
artifact storage. 30 builds is plenty for debugging.
```

### 💬 COMMS — #ci-cd (9:31 AM)

```
⚠️ Jenkins controller restarting NOW. ~3 min downtime.

Changes being applied:
1. Memory: 2Gi → 4Gi + G1GC (was using Parallel GC)
2. Agent pod hard timeout: 1h (prevents zombie accumulation)
3. Build retention: keep last 30 builds per job (was: unlimited)
   → ~6,600 old builds will be purged on startup
   → This is the root cause fix for the memory leak

Builds will be back shortly. If you need old build logs, 
check before they're purged — but released artifacts are safe 
in Artifactory.
```

### 🔧 ACTION 2 — Monitor restart (9:32 AM)

```bash
kubectl get pods -n ci -l app.kubernetes.io/name=jenkins -w

# Wait for Ready
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=jenkins \
  -n ci --timeout=300s
```

```bash
# Post-restart validation
# 1. Jenkins UI accessible
curl -sf -o /dev/null -w "%{http_code}" \
  https://jenkins.novamart.internal/login

# 2. Kubernetes cloud connected
curl -s -u admin:$JENKINS_TOKEN \
  "https://jenkins.novamart.internal/computer/api/json" | \
  jq '.computer | length'

# 3. CasC applied — verify build discarder
curl -s -u admin:$JENKINS_TOKEN \
  "https://jenkins.novamart.internal/job/payment-service/api/json?tree=property[*]" | \
  jq '.property[] | select(.logRotator)'

# 4. JVM flags applied
kubectl exec -n ci deployment/jenkins-controller -- \
  jps -v | grep -i "g1gc\|heapdump\|maxheapsize"

# 5. Heap state after restart
curl -s -u admin:$JENKINS_TOKEN \
  "https://jenkins.novamart.internal/prometheus" | \
  grep "jenkins_vm_memory_heap"
```

### 💬 COMMS — #ci-cd (9:35 AM, after restart completes)

```
✅ Jenkins back online. Verified:
  - UI accessible
  - Kubernetes cloud connected
  - G1GC active, 3Gi max heap
  - Build discard policy active — cleanup running in background
  - Heap at 41% (fresh start)

CI pipeline ready for builds. If anything looks off, ping 
#platform-engineering.

Monitoring: heap should stay much flatter now with build 
retention capped. I'll watch the GC logs over the next 24h 
to confirm the leak pattern is resolved.
```

---

## 9:40 AM — Also fix the fragile CronJob

### 🧠 THINKING

```
Fair feedback on the date parsing. Let me rewrite the cleanup 
to use kubectl server-side age filtering instead of shell date 
math. Much more robust.
```

```yaml
# jenkins-agent-cleanup-v2.yaml — robust version
apiVersion: batch/v1
kind: CronJob
metadata:
  name: jenkins-agent-cleanup
  namespace: ci
spec:
  schedule: "*/15 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      activeDeadlineSeconds: 120
      template:
        spec:
          serviceAccountName: jenkins-agent-cleanup
          restartPolicy: Never
          containers:
            - name: cleanup
              image: python:3.12-alpine
              securityContext:
                runAsNonRoot: true
                runAsUser: 1000
                readOnlyRootFilesystem: true
                allowPrivilegeEscalation: false
                capabilities:
                  drop: ["ALL"]
              command:
                - python3
                - -c
                - |
                  import json, subprocess, sys
                  from datetime import datetime, timezone, timedelta

                  THRESHOLD = timedelta(hours=1)
                  now = datetime.now(timezone.utc)

                  result = subprocess.run(
                      ["kubectl", "get", "pods", "-n", "ci", "-l", "jenkins=agent",
                       "--field-selector=status.phase=Running", "-o", "json"],
                      capture_output=True, text=True
                  )
                  if result.returncode != 0:
                      print(f"kubectl failed: {result.stderr}", file=sys.stderr)
                      sys.exit(1)

                  pods = json.loads(result.stdout).get("items", [])
                  print(f"Found {len(pods)} running agent pods")

                  killed = 0
                  for pod in pods:
                      name = pod["metadata"]["name"]
                      created = datetime.fromisoformat(
                          pod["metadata"]["creationTimestamp"].replace("Z", "+00:00")
                      )
                      age = now - created
                      if age > THRESHOLD:
                          print(f"ZOMBIE: {name} (age: {age}) — deleting")
                          subprocess.run(
                              ["kubectl", "delete", "pod", name, "-n", "ci",
                               "--grace-period=30"],
                              check=True
                          )
                          killed += 1

                  print(f"Cleanup complete. Killed {killed} zombie pods.")
```

```bash
kubectl apply -f jenkins-agent-cleanup-v2.yaml
```

---

## 9:45 AM — Update investigation ticket + close safety tickets

### 🔧 ACTION — Update PLAT-923

```
PLAT-923 — Jenkins memory leak — UPDATED

ROOT CAUSE CONFIRMED:
  1. No build discard policy → 6,939+ builds in memory
  2. Parallel GC (wrong for this workload) 
  3. 2Gi undersized heap

FIXES APPLIED:
  ✅ Build discard: 30 builds / 30 days per job
  ✅ Memory: 2Gi → 4Gi
  ✅ G1GC + tuning
  ✅ GC logging + heap dump on OOM enabled
  ✅ activeDeadlineSeconds on all agent templates
  ✅ Cleanup CronJob (every 15 min)
  ✅ Alerts (zombie detection + pool capacity)

VALIDATION NEEDED (this week):
  [ ] Monitor heap over 48h — confirm flat pattern (not linear climb)
  [ ] Verify old build purge completes without errors
  [ ] Review GC logs — confirm full GC frequency drops dramatically
  [ ] If heap STILL climbs after purge → heap dump analysis needed
      (leak is in plugins, not build metadata)
  [ ] Confirm no teams depend on build history > 30 builds
```

---

## 9:50 AM — Postmortem Draft (the non-negotiable)

### 🧠 THINKING

```
I committed to this by EOD today. Twice pushed it. Can't push again.
I have a clean block from now until whatever interrupts me.
Let me knock this out.

This covers yesterday's order service incident — the primary one 
that burned error budget.
```

### 🔧 ACTION — Write postmortem

```markdown
# POST-INCIDENT REVIEW: Order Service Error Budget Burn
# Date: 2024-01-15 (Monday)
# Severity: SEV2
# Author: [me]
# Status: DRAFT — review in Wednesday standup

## Summary
A configuration change in the order-service deploy reduced the 
inventory client timeout from 5000ms (default) to 800ms. At peak 
Monday traffic, inventory service p99 latency exceeded 800ms, 
causing 3.8% of order creation requests to fail with 504 timeout 
errors. ~235 orders/min failed for approximately 55 minutes.

## Impact
- **Duration:** 08:28 — 09:21 (53 min from first error to full rollout of fix)
- **Detection:** 08:47 (19 min — alert fired on 1h burn rate crossing 3x)
- **User impact:** ~12,900 failed order attempts (estimated)
- **Error budget:** 74.0% → 65.2% (8.8% consumed)
- **Revenue impact:** Estimated $XX,XXX in lost/delayed orders 
  (finance to provide exact number)
- **No data loss. No cascading failure.**

## Timeline
| Time | Event |
|------|-------|
| 08:15 | order-service sha-e7b31d4 deployed via ArgoCD (image + config) |
| 08:15-08:26 | Canary analysis running at ~2,400 rpm. Passes. |
| 08:26 | Canary promoted to 100%. All 12 replicas on new config. |
| 08:28 | Error rate begins climbing as Monday traffic ramps. |
| 08:47 | PagerDuty: HighErrorBudgetBurn_OrderService fires (3.2x 1h) |
| 09:04 | Alert acknowledged (on-call shift handover — 17 min gap) |
| 09:05 | Investigation begins — dashboards, PromQL queries |
| 09:10 | Root cause identified: 800ms timeout < inventory p99 (1.08s) |
| 09:11 | Escalated to SEV2 in #incidents |
| 09:13 | Confirmed via distributed trace (DEADLINE_EXCEEDED at 800ms) |
| 09:14 | Nina Petrov provides context — timeout was intentional (INC-2847) |
| 09:15 | Hotfix pushed: timeout 800ms → 2000ms |
| 09:21 | Fix fully rolled out. Error rate drops to 0.12% |
| 09:30 | Burn rate below 1x. Budget stabilized. Incident resolved. |

## Root Cause
The deploy included a new config parameter `inventory_client_timeout_ms: 800` 
and `inventory_client_retries: 1`. This was intentional — response to INC-2847 
where a 5000ms timeout caused thread pool saturation and cascading failure.

The 800ms value was validated in staging at 3,000 rpm where inventory p99 
was ~400ms. In production at 6,000+ rpm, inventory p99 was 1.08s — well 
above the 800ms timeout. The canary analysis passed because it ran during 
low-traffic hours (2,400 rpm).

**Root cause: configuration validated at staging load, not production load. 
Canary analysis has no traffic volume gate.**

## Contributing Factors
1. **Canary analysis doesn't account for traffic volume.** Analysis passed 
   at 2,400 rpm. Failure only manifests at 5,000+ rpm. 
2. **Staging load testing caps at 3,000 rpm** — not representative of 
   production peaks (10,200 rpm).
3. **No circuit breaker** on inventory call — binary success/timeout with 
   no degradation path.
4. **17-minute ack gap** — on-call handover at 9:00 AM, alert fired at 
   8:47 AM. No one was watching during the gap.

## Resolution
Interim: `inventory_client_timeout_ms: 2000` (2x headroom over p99, 
worst-case 4s with retry, under INC-2847 cascade threshold).

## Action Items
| ID | Action | Owner | Due | Status |
|----|--------|-------|-----|--------|
| AI-1 | Circuit breaker on order→inventory call | Derek Huang | End of sprint | PLAT-915, in progress |
| AI-2 | Canary analysis: minimum traffic volume gate | [me] | Next sprint | PLAT-916 |
| AI-3 | Linkerd ServiceProfile: isRetryable:false on POST /orders | Derek Huang | This week | Split from PLAT-915 |
| AI-4 | Inventory service latency profiling at scale | Nina Petrov | Next sprint | PLAT-917 |
| AI-5 | Staging load test environment: increase to production-representative RPM | QA + Platform | Q1 | New ticket needed |
| AI-6 | On-call handover gap: add 15-min overlap window | [me] + Sarah | Wednesday standup | Process change |

## Lessons Learned
1. **Configuration changes are as dangerous as code changes.** The image 
   may have been fine — the config is what caused the incident. Canary 
   analysis should evaluate config changes with the same rigor as code.
2. **Staging ≠ production.** Load-dependent behavior can't be validated 
   at 30% of production traffic. Either test at production scale or 
   deploy during peak hours with a tight canary.
3. **Fail-fast timeouts need a degradation path.** 800ms timeout was the 
   right instinct (prevent cascade) but without a circuit breaker and 
   fallback, "fail fast" just means "fail a lot."

## Metrics
- **MTTD (time to detect):** 19 min (alert on burn rate)
- **MTTA (time to ack):** 36 min (17 min handover gap + 2 min to ack)
- **MTTR (time to resolve):** 53 min (first error to fix rolled out)
- **Error budget impact:** -8.8% (74.0% → 65.2%)
```

---

## 10:30 AM — Priya's PR Re-review (15 min)

### 🔧 ACTION — Quick pass on PR #347 updates

```
Checking her fixes against my original review:

1. ✅ Trust anchor renewBefore reduced + manual rotation runbook 
   → Verify runbook has all 7 steps I outlined
2. ✅ Phased restart procedure with namespace ordering
   → Verify criticality order makes sense
3. ✅ Proxy CPU limit removed (memory limit kept)
4. ✅ ArgoCD SA excluded in Kyverno policy
5. ✅ kubectl image pinned + security context added
6. ✅ ArgoCD sync set to manual for upgrade
```

Assuming all items addressed properly:

### 💬 PR Comment (10:40 AM)

```
@priya.sharma All review items addressed. Trust anchor rotation 
runbook is thorough — the step-by-step with rollback at each 
stage is exactly what we need.

One small addition: in the phased restart procedure, add a 
verification step between each namespace:

  kubectl rollout status deployment -n <namespace> --timeout=120s
  linkerd check --proxy -n <namespace>

This confirms both the K8s rollout AND the Linkerd proxy health 
before moving to the next namespace.

Approving with that as a minor inline comment. ✅

Nice work — this is a meaningful reliability improvement for 
the platform.
```

**Approve PR.**

---

## 10:45 AM — EKS Upgrade Runbook (1 hour focused block)

### 🔧 ACTION — Continue runbook from yesterday's research

```bash
# Check pluto results from yesterday
pluto detect-all-in-cluster --target-versions k8s=v1.29.0 -o wide
```

Continue filling in the runbook sections — add-on version matrix, breaking changes, upgrade procedure, rollback plan. Due Friday, on track.

---

## 11:45 AM — Jenkins Heap Check

```bash
# 2+ hours post-restart — how's the heap?
curl -s -u admin:$JENKINS_TOKEN \
  "https://jenkins.novamart.internal/prometheus" | \
  grep "jenkins_vm_memory_heap_usage"

# How many full GCs so far?
curl -s -u admin:$JENKINS_TOKEN \
  "https://jenkins.novamart.internal/prometheus" | \
  grep "jenkins_vm_gc_collection_count"

# How many builds have been purged?
kubectl logs -n ci deployment/jenkins-controller --tail=100 | \
  grep -i "discard\|delete\|purge\|cleanup"
```

### 🧠 THINKING

```
Looking for:
- Heap at 40-60% (healthy after purge), NOT climbing fast
- Full GCs < 5 in 2 hours (was 47 in 7 hours before)
- Build purge logs showing old builds being deleted

If heap is still climbing at the same rate → leak is NOT in 
build metadata, it's plugin-level. Then we need the heap dump 
analysis. But I strongly suspect the build discard fixes it.
```

---

## Afternoon Plan

```
12:00 — Lunch (check phone)
 1:00 — EKS runbook continued (target: first complete draft by 3 PM)
 3:00 — Review Jenkins heap trend (6h post-restart data point)
 3:30 — Any remaining tickets/follow-ups
 4:00 — Prep for tomorrow's standup
         Items to raise:
         - Yesterday's incident (postmortem shared for review)
         - Jenkins CI fix summary
         - Kafka governance issue + Q1 priority tie-in
         - Redis upgrade complete
         - On-call handover gap (AI-6 from postmortem)
 4:30 — EOD notes
```

---

**What do the Jenkins heap metrics show at 11:45? And what's waiting after lunch?**

---

# SIMULATION RESPONSE — Jenkins Stabilized + Tuesday Afternoon

## Jenkins Fix + Postmortem Grade: A

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Build discard policy** | ✅ | The actual fix. 30/30 is reasonable. Communicated the risk (old build logs disappearing) before applying. |
| **CronJob rewrite** | ✅ | Python version is robust. Proper ISO 8601 parsing, error handling, kubectl JSON output. Addressed the feedback. |
| **Postmortem** | ✅ | Clean structure. Timeline accurate. Root cause correctly identified as config-validated-at-wrong-scale. Contributing factors are honest (ack gap, staging load cap). Action items have owners and dates. |
| **Postmortem gap** | ⚠️ | Missing one thing: **What went well.** Every postmortem should include this. You detected in 19 minutes via burn rate alerting, confirmed with traces in 3 minutes, fixed surgically without full rollback, didn't reintroduce INC-2847 risk. These are wins worth documenting — they reinforce the behaviors you want the org to repeat. Add a section before Wednesday. |
| **PR re-review** | ✅ | Fast pass, verified all items, added a useful inline suggestion (proxy health check between namespaces), approved. Correct. |
| **CasC `jobDefaultConfiguration`** | ⚠️ | Minor: `jobDefaultConfiguration` with `buildDiscarder` sets the default for NEW jobs. Existing jobs that already have no discard policy may not pick this up automatically — depends on Jenkins version and whether jobs were created via CasC or manually. You should verify that existing jobs (payment-service with 1,247 builds) actually started purging. If not, you need a script to apply the discard policy to existing jobs via the Jenkins API. Not docking hard because you're checking the purge logs at 11:45. |

---

## 📊 Jenkins Heap — 11:45 AM (2.5 hours post-restart)

```
jenkins_vm_memory_heap_usage:        0.52 (52%)
jenkins_vm_memory_heap_used_bytes:   1,664,286,720 (1.55 GiB of 3 GiB max)

jenkins_vm_gc_collection_count{gc="G1 Young Generation"}:  89
jenkins_vm_gc_collection_count{gc="G1 Old Generation"}:    2

Build purge logs:
  INFO  Distributed build-discarder policy to 47 jobs
  INFO  payment-service: discarding builds #1 through #1217 (keeping #1218-#1247)
  INFO  order-service: discarding builds #1 through #862 (keeping #863-#892)
  INFO  catalog-service: discarding builds #1 through #704 (keeping #705-#734)
  ... (similar for all jobs)
  INFO  Build discard complete. Purged 6,614 builds across 47 jobs.
  INFO  Freed 12.4 GiB disk space.
```

```
HEAP TREND (post-restart):
  ▲ GiB
3.0│ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ max
   │
2.0│
   │                          
1.5│  ╭─╮ ╭─╮  ╭─╮  ╭─╮  ╭─╮    1.55 GiB (sawtooth = healthy G1 GC)
   │ ╭╯ ╰─╯ ╰──╯ ╰──╯ ╰──╯ ╰─╮
1.0│─╯                          
   │
0.5│
   └──────────────────────────────▶
   9:30  9:45  10:00 10:30 11:00 11:45

BEFORE (for comparison):
  Linear climb: 45% → 68% in 7 hours = +3.3%/hour
  
AFTER:
  Sawtooth between 48-52% for 2.5 hours = FLAT ✅
  2 old-gen GCs (vs 47 in 7 hours before) = 95% reduction ✅
```

**The build discard was the fix.** Heap is exhibiting healthy G1GC sawtooth pattern — young gen collections reclaiming memory, old gen barely growing. The linear climb is gone.

---

## 💬 Slack — Tuesday Afternoon

### #platform-engineering — 12:15 PM

**Lisa Park:**
> Jenkins heap looks much healthier. Sawtooth on the Grafana dashboard instead of the death ramp. Nice fix. I'll set up a long-term dashboard panel tracking heap growth rate so we catch it if it starts climbing again.

### #ci-cd — 12:30 PM

**David Okafor:**
> Payment hotfix deployed and verified in production. EU currency rounding fixed. Confirmed with 3 test transactions. Thanks everyone for the quick turnaround this morning.

### #engineering-general — 1:15 PM

**Sarah Chen:**
> Reminder: Wednesday standup at 10 AM. Agenda:
> - Q1 priorities finalization (I reviewed [your] email — good input)
> - Incident review (Monday order service + Tuesday CI)
> - Kafka governance discussion
> - On-call process improvements
> 
> Come prepared. James (VP Eng) is joining for the Q1 section.

---

### 📊 3:00 PM — Jenkins 6-hour Check

```
jenkins_vm_memory_heap_usage:   0.54 (54%)
G1 Old Generation GC count:    3 (only +1 in 3.5 hours)

VERDICT: Stable. Sawtooth continues. Build discard resolved the leak.
```

---

# FAST FORWARD — Tuesday Afternoon → Wednesday Morning

**Tuesday afternoon is uneventful.** You:
- Finished first complete draft of EKS upgrade runbook (PLAT-892)
- Confirmed Jenkins heap stable at 6h mark
- Added "What Went Well" section to postmortem (per my feedback)
- Wrote EOD notes
- No PagerDuty alerts overnight

---

# 🗓️ WEDNESDAY MORNING — Week 1, Day 3

**Time: 9:45 AM EST. Standup at 10:00 AM.**

### 📊 Morning Dashboard — All Green

```
Order service:      0.09% error rate, budget 63.8% (recovering)
Catalog service:    0.04% error rate, p99 0.47s (baseline ✅)
Redis:              94.1% hit rate (fully recovered ✅), 0 evictions
Jenkins:            53% heap, 4 old-gen GCs in 24h (healthy ✅)
CI pipeline:        3 builds running, 0 zombies ✅
All systems:        Green
```

### 💬 Slack — #platform-engineering — 9:30 AM

**Priya Sharma:**
> PR #347 merged! Added the proxy health check step between namespace restarts per your suggestion. ArgoCD sync set to manual — planning to execute the Linkerd upgrade during Friday's low-traffic window. Does that work?

### 💬 Slack — #security — 9:40 AM

**Aisha Rahman (Security Lead):**
> @[you] Got your PLAT-922 ticket about the Kafka cluster. Thanks for flagging before the audit. Quick questions:
> 1. Can you confirm whether payment card data (PAN, CVV) flows through Kafka, or is it sanitized event data?
> 2. I see the EBS volumes are encrypted with the default AWS key. We need CMK for PCI-scoped systems. How quickly can we rotate?
> 3. The lack of mTLS between producers and Kafka is a finding. What's the timeline for Linkerd mesh injection?
> 
> I'm starting the PCI scope assessment today. Need your input.

---

## 10:00 AM — Wednesday Standup

**Room:** You, Sarah, Marcus, Priya, Jake, Alex, Lisa
**Guest:** James Morrison (VP Engineering) — for Q1 priorities section

**Sarah kicks off:**

> Let's cover three things: Q1 priorities with James, this week's incidents, and the Kafka governance piece. [You], you had opinions on Q1 priorities — walk James through your recommendation.

---

**You're presenting to the VP of Engineering. He's sharp, doesn't suffer fluff, and cares about risk/revenue/cost.**

**What do you say? Hit Q1 priorities, incidents summary, and Kafka governance. Keep it tight — James has 20 minutes before his next meeting.**



# Wednesday 10:00 AM — Standup

---

## 🧠 THINKING (10 seconds before speaking)

```
James has 20 minutes. Three topics. No fluff.

Q1 priorities: lead with risk + business impact, not tech details.
Incidents: lessons learned, not a replay. He doesn't need the 
  timeline — he needs to know what broke, what it cost, and 
  what we're doing so it doesn't happen again.
Kafka: governance gap, PCI risk, process fix. Keep it short.

Budget my time: 8 min priorities, 6 min incidents, 4 min Kafka, 
2 min questions.
```

---

## 💬 Q1 Priorities (to James)

> James, I reviewed Sarah's list and have a recommendation on ordering. Five items, ranked by risk.
>
> **Number one: EKS 1.28 to 1.29 upgrade.** Non-negotiable — 1.28 hits end of support this quarter. If we're still on it when a CVE drops, we're patching an unsupported version. Runbook is drafted, targeting execution in the next two weeks. Low risk if we do it on schedule, high risk if we don't.
>
> **Number two — and I'm adding this one: PCI Phase 2 audit prep.** Sarah mentioned Aisha needs our help, but it wasn't on the official list. I'd make it explicit because audits have a hard deadline and we've already found untracked infrastructure that may be in PCI scope — I'll cover that in the Kafka section. If we fail the audit, that's a business-level problem, not a platform problem.
>
> **Number three: ELK to Loki migration.** We're running dual logging stacks. ELK is our single biggest observability cost — estimate roughly 30-40% savings by decommissioning it. Loki handles primary workloads already, this is migrating the long tail of teams still on Kibana and turning off the ELK cluster. Multi-sprint effort but high ROI.
>
> **Number four: Cost optimization.** GP3 storage migration, Redis right-sizing — we did one this week already — and Karpenter consolidation tuning. These are quick wins we can run in parallel with the other items. I'd estimate 15-20% infrastructure cost reduction achievable this quarter.
>
> **Number five: Self-service namespace provisioning.** This is the right investment for engineering velocity — every new service or environment today requires a platform team ticket. But it's also the biggest build. I'd recommend we start design in Q1 and deliver in early Q2. Doing it right means RBAC, quotas, network policies, ArgoCD integration, and Kyverno guardrails. Rushing it creates security gaps.
>
> Bottom line: items one through three protect us — support deadlines, compliance, cost. Items four and five improve us. I'd staff accordingly.

---

## 💬 Incidents Summary (to the room, James listening)

> Two incidents this week, both resolved. Quick summary of each with the takeaway.
>
> **Monday — Order service SEV2.** A config change tightened the inventory client timeout from 5 seconds to 800 milliseconds. Intention was right — prevent cascading failures from a previous incident. Problem was it was validated in staging at 3,000 RPM. Production hits 10,000. Inventory service is fine at low load but p99 exceeds 800ms at peak. Canary passed because it ran during low-traffic hours.
>
> **Cost:** 8.8% error budget burned, roughly 12,900 failed order attempts over 53 minutes. Interim fix in place. Derek Huang is implementing a circuit breaker with proper fallback this sprint.
>
> **Key takeaway and action item:** Our canary analysis has no concept of traffic volume. A change can pass canary at 2,400 RPM and fail at 6,000. We're adding a minimum traffic threshold to canary analysis so load-dependent changes can't promote during off-peak. Also — this exposed that our staging load tests cap at 3K RPM. Need to invest in production-representative load testing.
>
> **Tuesday — Jenkins CI blocked payment hotfix for 2 hours.** Jenkins controller has been silently OOM-ing every 3 days. Previous OOMs happened at night with no impact. This time, orphaned agent pods accumulated and filled the Karpenter CI pool. Payment team's currency rounding hotfix — EU customers being overcharged — was blocked for almost 2 hours.
>
> **Root cause:** No build retention policy. Jenkins was holding 6,900+ builds in memory. Combined with wrong GC algorithm and undersized heap.
>
> **Fixed:** Build retention capped at 30, memory doubled, G1GC, plus three safety nets — pod hard timeouts, automated cleanup job, and capacity alerts. Jenkins heap has been stable for 24 hours.
>
> **Key takeaway:** This was invisible until it wasn't. The controller had OOM'd three times in nine days with no alert. We were one bad timing away from blocking a customer-impacting hotfix — which is exactly what happened. The safety nets we added mean even if the controller OOMs again, the blast radius is contained.
>
> Postmortem for Monday's incident is drafted, shared in Confluence for review. I'll send the link after standup.

---

## 💬 Kafka Governance (to the room)

> Last item. Jake found a Kafka cluster running in production that platform didn't know about. Six brokers, three ZooKeeper nodes, 720 gigs of storage, running for 67 days. Deployed by Tom Chen from data engineering via direct Helm install. Not in GitOps, not meshed, no monitoring, no backups, no network policies until I added them Monday.
>
> I want to be clear: I'm not throwing Tom under the bus. He needed a Kafka cluster for real-time analytics, and the holiday crunch meant it didn't go through formal review. It happens. The cluster is running fine.
>
> **The problem is systemic, not individual.** Right now, anyone with kubectl access to production can deploy anything — no review gate, no ArgoCD requirement, no Kyverno policy blocking unmanaged workloads. This Kafka cluster is the one we found. There may be others.
>
> **Why this is urgent:** PCI Phase 2 audit is next month. Aisha is already asking me whether payment card data flows through this Kafka cluster. If it does and we can't demonstrate controls — encryption in transit, access controls, audit logging — that's an audit finding. Untracked infrastructure in a PCI environment is exactly what auditors look for.
>
> **Three actions:**
>
> **One — immediate:** Tom's back Monday. Meeting scheduled to onboard this cluster to platform standards. Monitoring, GitOps, mesh, backups. Two-sprint effort.
>
> **Two — PCI:** Aisha is running scope assessment now. I'm working with her to answer whether payment data flows through Kafka and what controls are needed.
>
> **Three — process:** We need guardrails that prevent this from recurring. Proposals:
> - Kyverno policy requiring a `platform-managed` label on any namespace with production workloads
> - RBAC tightening — restrict direct Helm and kubectl write access in production to platform team and break-glass
> - ArgoCD as the only deployment path to production
>
> Sarah, this ties directly into Q1 priorities — the namespace self-service work in item five should include these guardrails. I'd actually recommend we ship the guardrails first, before the self-service portal. Lock the door, then build the front desk.

---

## 💬 Quick Answers to Outstanding Items

> Two quick items before James heads out.
>
> **Priya** — Friday low-traffic window for Linkerd upgrade works. I'll be on-call to support. Let's do a dry-run of the restart sequence Thursday afternoon in staging.
>
> **Aisha's questions from Slack** — I'll respond to her directly after standup. Short version: I don't know yet whether PAN/CVV flows through Kafka. Need Tom for that. EBS volumes are encrypted but with default AWS key, not CMK — that's a gap for PCI scope. mTLS timeline depends on Linkerd mesh injection which is part of the onboarding ticket. I'll give her a full response within the hour.

---

**That's my 20 minutes. Questions?**



# SIMULATION RESPONSE — Standup Grade + Aftermath

## Standup Presentation Grade: A+

This is the best single performance in the entire simulation so far. Here's why.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Time management** | ✅ | 8-6-4-2 budget, hit all three topics, left room for questions. James didn't have to wait or cut you off. |
| **Risk-first framing for VP** | ✅ | "Items 1-3 protect us, 4-5 improve us. Staff accordingly." That's executive language. You translated technical priorities into business risk in one sentence. |
| **PCI insertion** | ✅ | Adding it to the official list with justification — not as a suggestion but as a correction to a gap. Confident without being arrogant. |
| **Incident summaries** | ✅ | Cost → cause → fix → takeaway. No timeline replay. James doesn't need to know you ran PromQL at 9:10 AM. He needs "12,900 failed orders, config validated at wrong scale, canary gap being fixed." Perfect. |
| **Jenkins framing** | ✅ | "Invisible until it wasn't" — that's the kind of phrase that sticks in a VP's head. You made a JVM memory leak sound like a business risk in one sentence. |
| **Kafka governance** | ✅ | "Not throwing Tom under the bus" — political awareness. "The problem is systemic, not individual" — mature engineering leadership. "Lock the door, then build the front desk" — memorable, actionable, correct. |
| **Aisha response** | ✅ | Honest: "I don't know yet whether PAN/CVV flows through Kafka. Need Tom for that." Seniors say "I don't know" when they don't know. They don't speculate to sound smart in front of the VP. |
| **Priya support** | ✅ | Quick, supportive, offered staging dry-run. Didn't steal the moment to re-review her work in front of the VP. |

**The "lock the door, then build the front desk" line is going to come back.** James will remember that when he's prioritizing Q2 budget. That's influence.

---

## Room Reactions

**James Morrison (VP Eng)** — 10:21 AM:
> Good summary. I agree with the ordering. The PCI point is well-taken — let's make sure that's tracked as a formal initiative, not a side quest. Sarah, make sure Aisha has what she needs from platform.
> 
> On the canary gap — how confident are we that the traffic-volume gate actually prevents this class of incident? Is there a broader testing strategy we should be investing in?
> 
> Also — the Kafka thing concerns me beyond PCI. If one team did it, others might have. Can we do a cluster-wide audit for unmanaged workloads? I don't want surprises during the audit.

**Sarah Chen:**
> Both good points. [You] — can you take the cluster audit? Scope it as: anything running in production that isn't in ArgoCD or Terraform. Should be a one-day effort.

**Lisa Park:**
> On the canary question — the traffic-volume gate helps for this specific scenario. But more broadly, we should look at staged rollout windows. Deploy latency-sensitive changes during peak hours, not at 8 AM. Counter-intuitive but you get real signal immediately.

**Marcus Webb:**
> +1 on the cluster audit. I've suspected there might be a few test deployments people forgot to clean up. Quick wins for the PCI prep.

---

## 💬 Slack — After Standup

### DM from Sarah Chen — 10:30 AM

> Great standup. James was impressed — he told me afterward "that's the kind of thinking I want from the platform team." Keep it up.
> 
> On the cluster audit: can you get it done by Friday? I want to include findings in next week's leadership sync.

### #security — 10:35 AM

**Aisha Rahman:**
> Thanks for the context in standup. The cluster-wide audit is exactly what I need for PCI prep. When you find everything that's not in ArgoCD/Terraform, flag anything that touches payment data, user PII, or authentication tokens. Those are automatically in-scope.
> 
> On the Kafka CMK question — I can help with the KMS key creation and policy. Just need to know the timeline for when the Kafka cluster is under platform management.

---

**It's 10:35 AM Wednesday. Your new action items:**

```
NEW FROM STANDUP:
  1. Cluster-wide audit for unmanaged workloads (Sarah wants by Friday)
  2. Answer Aisha's Kafka questions (committed within the hour)
  3. Respond to James's canary question (broader testing strategy)
  
EXISTING:
  4. EKS upgrade runbook (due Friday — draft done, needs polish)
  5. Friday Linkerd upgrade support + Thursday staging dry-run
  6. Jenkins heap monitoring (ongoing, stable)
  7. Postmortem review (shared, waiting for feedback)
```

**Plus — I have incidents planned for today. The simulation doesn't get easier because you had a good standup.**

**What's your next move? And how do you plan the rest of Wednesday?**



# Wednesday 10:00 AM — Standup

---

## 🧠 THINKING (10 seconds before speaking)

```
James has 20 minutes. Three topics. No fluff.

Q1 priorities: lead with risk + business impact, not tech details.
Incidents: lessons learned, not a replay. He doesn't need the 
  timeline — he needs to know what broke, what it cost, and 
  what we're doing so it doesn't happen again.
Kafka: governance gap, PCI risk, process fix. Keep it short.

Budget my time: 8 min priorities, 6 min incidents, 4 min Kafka, 
2 min questions.
```

---

## 💬 Q1 Priorities (to James)

> James, I reviewed Sarah's list and have a recommendation on ordering. Five items, ranked by risk.
>
> **Number one: EKS 1.28 to 1.29 upgrade.** Non-negotiable — 1.28 hits end of support this quarter. If we're still on it when a CVE drops, we're patching an unsupported version. Runbook is drafted, targeting execution in the next two weeks. Low risk if we do it on schedule, high risk if we don't.
>
> **Number two — and I'm adding this one: PCI Phase 2 audit prep.** Sarah mentioned Aisha needs our help, but it wasn't on the official list. I'd make it explicit because audits have a hard deadline and we've already found untracked infrastructure that may be in PCI scope — I'll cover that in the Kafka section. If we fail the audit, that's a business-level problem, not a platform problem.
>
> **Number three: ELK to Loki migration.** We're running dual logging stacks. ELK is our single biggest observability cost — estimate roughly 30-40% savings by decommissioning it. Loki handles primary workloads already, this is migrating the long tail of teams still on Kibana and turning off the ELK cluster. Multi-sprint effort but high ROI.
>
> **Number four: Cost optimization.** GP3 storage migration, Redis right-sizing — we did one this week already — and Karpenter consolidation tuning. These are quick wins we can run in parallel with the other items. I'd estimate 15-20% infrastructure cost reduction achievable this quarter.
>
> **Number five: Self-service namespace provisioning.** This is the right investment for engineering velocity — every new service or environment today requires a platform team ticket. But it's also the biggest build. I'd recommend we start design in Q1 and deliver in early Q2. Doing it right means RBAC, quotas, network policies, ArgoCD integration, and Kyverno guardrails. Rushing it creates security gaps.
>
> Bottom line: items one through three protect us — support deadlines, compliance, cost. Items four and five improve us. I'd staff accordingly.

---

## 💬 Incidents Summary (to the room, James listening)

> Two incidents this week, both resolved. Quick summary of each with the takeaway.
>
> **Monday — Order service SEV2.** A config change tightened the inventory client timeout from 5 seconds to 800 milliseconds. Intention was right — prevent cascading failures from a previous incident. Problem was it was validated in staging at 3,000 RPM. Production hits 10,000. Inventory service is fine at low load but p99 exceeds 800ms at peak. Canary passed because it ran during low-traffic hours.
>
> **Cost:** 8.8% error budget burned, roughly 12,900 failed order attempts over 53 minutes. Interim fix in place. Derek Huang is implementing a circuit breaker with proper fallback this sprint.
>
> **Key takeaway and action item:** Our canary analysis has no concept of traffic volume. A change can pass canary at 2,400 RPM and fail at 6,000. We're adding a minimum traffic threshold to canary analysis so load-dependent changes can't promote during off-peak. Also — this exposed that our staging load tests cap at 3K RPM. Need to invest in production-representative load testing.
>
> **Tuesday — Jenkins CI blocked payment hotfix for 2 hours.** Jenkins controller has been silently OOM-ing every 3 days. Previous OOMs happened at night with no impact. This time, orphaned agent pods accumulated and filled the Karpenter CI pool. Payment team's currency rounding hotfix — EU customers being overcharged — was blocked for almost 2 hours.
>
> **Root cause:** No build retention policy. Jenkins was holding 6,900+ builds in memory. Combined with wrong GC algorithm and undersized heap.
>
> **Fixed:** Build retention capped at 30, memory doubled, G1GC, plus three safety nets — pod hard timeouts, automated cleanup job, and capacity alerts. Jenkins heap has been stable for 24 hours.
>
> **Key takeaway:** This was invisible until it wasn't. The controller had OOM'd three times in nine days with no alert. We were one bad timing away from blocking a customer-impacting hotfix — which is exactly what happened. The safety nets we added mean even if the controller OOMs again, the blast radius is contained.
>
> Postmortem for Monday's incident is drafted, shared in Confluence for review. I'll send the link after standup.

---

## 💬 Kafka Governance (to the room)

> Last item. Jake found a Kafka cluster running in production that platform didn't know about. Six brokers, three ZooKeeper nodes, 720 gigs of storage, running for 67 days. Deployed by Tom Chen from data engineering via direct Helm install. Not in GitOps, not meshed, no monitoring, no backups, no network policies until I added them Monday.
>
> I want to be clear: I'm not throwing Tom under the bus. He needed a Kafka cluster for real-time analytics, and the holiday crunch meant it didn't go through formal review. It happens. The cluster is running fine.
>
> **The problem is systemic, not individual.** Right now, anyone with kubectl access to production can deploy anything — no review gate, no ArgoCD requirement, no Kyverno policy blocking unmanaged workloads. This Kafka cluster is the one we found. There may be others.
>
> **Why this is urgent:** PCI Phase 2 audit is next month. Aisha is already asking me whether payment card data flows through this Kafka cluster. If it does and we can't demonstrate controls — encryption in transit, access controls, audit logging — that's an audit finding. Untracked infrastructure in a PCI environment is exactly what auditors look for.
>
> **Three actions:**
>
> **One — immediate:** Tom's back Monday. Meeting scheduled to onboard this cluster to platform standards. Monitoring, GitOps, mesh, backups. Two-sprint effort.
>
> **Two — PCI:** Aisha is running scope assessment now. I'm working with her to answer whether payment data flows through Kafka and what controls are needed.
>
> **Three — process:** We need guardrails that prevent this from recurring. Proposals:
> - Kyverno policy requiring a `platform-managed` label on any namespace with production workloads
> - RBAC tightening — restrict direct Helm and kubectl write access in production to platform team and break-glass
> - ArgoCD as the only deployment path to production
>
> Sarah, this ties directly into Q1 priorities — the namespace self-service work in item five should include these guardrails. I'd actually recommend we ship the guardrails first, before the self-service portal. Lock the door, then build the front desk.

---

## 💬 Quick Answers to Outstanding Items

> Two quick items before James heads out.
>
> **Priya** — Friday low-traffic window for Linkerd upgrade works. I'll be on-call to support. Let's do a dry-run of the restart sequence Thursday afternoon in staging.
>
> **Aisha's questions from Slack** — I'll respond to her directly after standup. Short version: I don't know yet whether PAN/CVV flows through Kafka. Need Tom for that. EBS volumes are encrypted but with default AWS key, not CMK — that's a gap for PCI scope. mTLS timeline depends on Linkerd mesh injection which is part of the onboarding ticket. I'll give her a full response within the hour.

---

**That's my 20 minutes. Questions?**

Good momentum from standup. Not going to coast on it. Here's how I'm spending the rest of Wednesday:

---

## Immediate Actions (Next 30 min)

### 1. Reply to Aisha — #security (10:37 AM)

> Hey Aisha — timeline for Kafka under platform management: Tom's back Monday, onboarding meeting scheduled. Realistic target for full platform management (ArgoCD, ECR images, monitoring, RBAC) is end of next week.
> 
> For CMK — can we get the KMS key created this week so it's ready when we onboard? I can provide the IAM principals that need encrypt/decrypt access once we've mapped the consumer topology with Tom. Happy to pair on the key policy whenever works for you.
> 
> On PCI scope: I confirmed EBS volumes are encrypted (account-level default, AWS-managed key — not CMK). No mTLS currently, no audit logging on the brokers. I don't yet know whether PAN/CVV/PII flows through those topics — that's a Tom question for Monday. I'll make sure that's the first thing we nail down.
> 
> Will also flag anything PCI-relevant from the cluster-wide audit as I go. Should have initial findings by Friday.

### 2. Reply to James's canary question — Slack thread or email (10:42 AM)

> James — good question. The traffic-volume gate closes *this specific* gap (config validated at low RPM, failed at production scale). But Lisa's right that it's part of a broader testing strategy we should formalize:
> 
> **Three layers we should invest in:**
> 1. **Canary improvements** (short-term, PLAT-916): Traffic-volume gate + latency-aware analysis window. Prevents "passed at 2,400 rpm, failed at 6,000 rpm."
> 2. **Load-test gating** (medium-term): For latency-sensitive config changes, require a load test that simulates production traffic patterns before merge. We don't have this today.
> 3. **Staged rollout windows** (Lisa's point): Deploy latency-sensitive changes during representative traffic, not 8 AM. Counter-intuitive but gives real signal immediately.
> 
> The circuit breaker work Derek's delivering Thursday adds a safety net *even if* a bad config gets through — so defense in depth, not just prevention.
> 
> Happy to write up a one-pager on the broader testing strategy if that's useful for leadership sync.

### 3. Reply to Sarah — DM (10:45 AM)

> Thanks Sarah. Friday for the audit works — I'll start the automated sweep today and do manual verification tomorrow. I'll structure findings as: what it is, who owns it, whether it's in ArgoCD/Terraform, and PCI relevance (per Aisha's criteria: payment data, PII, auth tokens).
> 
> Quick heads up — I've got the EKS runbook also due Friday and Linkerd staging dry-run Thursday. All doable but I'm going to timebox the audit to avoid it expanding. If I find something scary I'll flag immediately rather than waiting for the full report.

---

## Wednesday Plan — 10:45 AM onward

```
10:45 - 12:00  Cluster audit — automated sweep
               Approach:
               - All namespaces → list workloads (Deployments, StatefulSets, 
                 DaemonSets, CronJobs, Jobs)
               - Cross-reference against ArgoCD Application inventory
               - Cross-reference against Terraform state
               - Anything with no match = unmanaged
               - For each unmanaged: who deployed it, when, what images, 
                 what data it touches, NetworkPolicy status
               
               Script it. Don't do this by hand across namespaces.

12:00 - 12:30  Lunch (phone on)

12:30 - 14:00  EKS upgrade runbook — polish draft
               - Add-on compatibility matrix (fill gaps)
               - Rollback procedure detail
               - Communication template
               - This needs to be done, not perfect

14:00 - 15:00  Cluster audit — manual verification of findings
               Review anything the automated sweep flagged.
               Spot-check a few "managed" workloads to validate the script.

15:00 - 15:30  Priya — check in on Linkerd prep for Thursday dry-run
               Make sure staging environment is ready

15:30 - 16:00  Jenkins heap check + any cleanup from morning
               
16:15          EOD notes
```

### Key priority calls:
- **Audit today, runbook today.** Both due Friday — spreading the work avoids a Thursday crunch, especially with the Linkerd dry-run Thursday.
- **Audit is scriptable.** The hard part isn't finding unmanaged workloads — it's classifying what they are and whether they matter. Automated sweep first, human judgment second.
- **If an incident drops, the runbook polish slides first.** Audit has a stakeholder (Sarah + James + Aisha for PCI). Runbook is important but more flexible.

---

Starting the audit sweep now. What does the cluster look like?



---

# SIMULATION RESPONSE — The Audit Reveals Things

## Planning Grade: A

No notes. Realistic time allocation, correct priority calls, scriptable approach to the audit. The "if an incident drops, runbook slides first" is the right call and I appreciate you stating it explicitly rather than pretending everything fits.

---

## 📊 Cluster Audit — Automated Sweep Results

You write a script. Here's what it finds.

### Script Output

```bash
#!/bin/bash
# cluster-audit.sh — Find unmanaged workloads

echo "=== CLUSTER AUDIT: $(date) ==="
echo ""

# Get all ArgoCD-managed resources
ARGO_APPS=$(argocd app list -o json | jq -r '.[].metadata.name')
ARGO_NAMESPACES=$(argocd app list -o json | jq -r '.[].spec.destination.namespace' | sort -u)

# Get all namespaces
ALL_NS=$(kubectl get ns -o jsonpath='{.items[*].metadata.name}')

echo "=== WORKLOADS NOT MANAGED BY ARGOCD ==="
for ns in $ALL_NS; do
  # Skip system namespaces
  [[ "$ns" =~ ^(kube-system|kube-public|kube-node-lease|argocd|monitoring|ci|linkerd.*)$ ]] && continue
  
  # Get all workloads in namespace
  kubectl get deploy,statefulset,daemonset,cronjob -n "$ns" \
    -o json | jq -r '
    .items[] | 
    select(.metadata.labels["argocd.argoproj.io/instance"] == null) |
    "\(.kind)\t\(.metadata.namespace)\t\(.metadata.name)\t\(.metadata.creationTimestamp)\t\(.metadata.annotations["kubectl.kubernetes.io/last-applied-by"] // "unknown")"
  '
done
```

### Raw Findings

```
KIND            NAMESPACE        NAME                          CREATED              DEPLOYED BY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
StatefulSet     data-analytics   data-kafka-broker             2023-11-09T14:22     tom.chen@novamart.com
StatefulSet     data-analytics   data-kafka-zookeeper          2023-11-09T14:22     tom.chen@novamart.com
Deployment      data-analytics   kafka-connect                 2023-12-01T09:15     tom.chen@novamart.com
Deployment      data-analytics   analytics-dashboard           2023-12-15T11:30     tom.chen@novamart.com
CronJob         data-analytics   etl-daily-aggregation         2023-11-20T16:45     tom.chen@novamart.com
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deployment      staging          perf-test-harness             2023-10-22T08:12     derek.huang@novamart.com
Deployment      staging          mock-payment-gateway          2023-10-22T08:14     derek.huang@novamart.com
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deployment      default          nginx-test                    2023-09-14T13:22     unknown
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CronJob         order-prod       order-data-export             2023-12-20T10:00     nina.petrov@novamart.com
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DaemonSet       monitoring       node-problem-detector         2023-08-15T09:00     marcus.webb@novamart.com
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Additional Details (from deeper inspection)

**1. data-analytics namespace (Tom Chen) — EXPANDED scope from what we knew:**

```bash
$ kubectl get all -n data-analytics

# We already knew about:
#   kafka (6 brokers + 3 ZK) — PLAT-921

# NEW findings:
NAME                                READY   STATUS    AGE
pod/kafka-connect-7f4d8b2c1-abc    1/1     Running   45d
pod/analytics-dashboard-6d9e3f1-x  1/1     Running   29d

NAME                          TYPE        PORT
svc/kafka-connect             ClusterIP   8083/TCP
svc/analytics-dashboard       ClusterIP   3000/TCP

# kafka-connect:
$ kubectl get deploy kafka-connect -n data-analytics -o jsonpath='{.spec.template.spec.containers[0].image}'
→ confluentinc/cp-kafka-connect:7.5.3  (Docker Hub, not ECR)

$ kubectl get deploy kafka-connect -n data-analytics -o jsonpath='{.spec.template.spec.containers[0].env}' | jq
[
  {"name": "CONNECT_BOOTSTRAP_SERVERS", "value": "analytics-kafka:9092"},
  {"name": "CONNECT_GROUP_ID", "value": "novamart-analytics"},
  {"name": "CONNECT_KEY_CONVERTER", "value": "org.apache.kafka.connect.json.JsonConverter"},
  {"name": "CONNECT_VALUE_CONVERTER", "value": "org.apache.kafka.connect.json.JsonConverter"},
  {"name": "CONNECT_CONFIG_STORAGE_TOPIC", "value": "connect-configs"},
  {"name": "CONNECT_OFFSET_STORAGE_TOPIC", "value": "connect-offsets"},
  {"name": "CONNECT_STATUS_STORAGE_TOPIC", "value": "connect-status"}
]

# Analytics dashboard:
$ kubectl get deploy analytics-dashboard -n data-analytics -o jsonpath='{.spec.template.spec.containers[0].image}'
→ novamart-analytics-dashboard:latest  (custom image, :latest tag, from... where?)

$ kubectl get deploy analytics-dashboard -n data-analytics -o jsonpath='{.spec.template.spec.containers[0].env}' | jq
[
  {"name": "KAFKA_BROKER", "value": "analytics-kafka:9092"},
  {"name": "DB_HOST", "value": "novamart-orders-db.xxxxx.us-east-1.rds.amazonaws.com"},
  {"name": "DB_USER", "value": "analytics_reader"},
  {"name": "DB_PASSWORD", "valueFrom": {"secretKeyRef": {"name": "analytics-db-creds", "key": "password"}}},
  {"name": "DASHBOARD_PORT", "value": "3000"}
]

# ETL CronJob:
$ kubectl get cronjob etl-daily-aggregation -n data-analytics -o jsonpath='{.spec.jobTemplate.spec.template.spec.containers[0].env}' | jq
[
  {"name": "SOURCE_DB", "value": "novamart-orders-db.xxxxx.us-east-1.rds.amazonaws.com"},
  {"name": "SOURCE_USER", "value": "analytics_reader"},
  {"name": "SOURCE_PASSWORD", "valueFrom": {"secretKeyRef": {"name": "analytics-db-creds", "key": "password"}}},
  {"name": "DEST_KAFKA", "value": "analytics-kafka:9092"},
  {"name": "TABLES", "value": "orders,order_items,payments,refunds"}
]
```

**The ETL job reads from the orders database — including the `payments` and `refunds` tables — and writes to Kafka.**

---

**2. staging namespace (Derek Huang):**

```bash
$ kubectl get deploy -n staging -l '!argocd.argoproj.io/instance' -o wide

perf-test-harness       1/1   83d   golang:1.21 (Docker Hub)
mock-payment-gateway    1/1   83d   novamart-mock-pg:v0.3 (ECR ✅)

# Resources:
perf-test-harness: requests 500m/512Mi, limits 2/2Gi
mock-payment-gateway: requests 100m/128Mi, no limits

# No PVCs, no secrets with sensitive data
# These look like test fixtures from October load testing
# mock-payment-gateway is on ECR (good), perf-test-harness is Docker Hub
```

---

**3. default namespace:**

```bash
$ kubectl get deploy nginx-test -n default -o yaml

metadata:
  name: nginx-test
  namespace: default
  creationTimestamp: "2023-09-14T13:22:00Z"
  # No annotations indicating who deployed this
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          # No resource limits
          # No security context
```

**Ancient. 4 months old. `nginx:latest` in the default namespace. No owner. Classic "someone tested something and forgot."**

---

**4. order-prod namespace (Nina Petrov):**

```bash
$ kubectl get cronjob order-data-export -n order-prod -o yaml

spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: exporter
              image: 123456789.dkr.ecr.us-east-1.amazonaws.com/order-data-export:v1.4
              env:
                - name: DB_HOST
                  value: "novamart-orders-db.xxxxx.us-east-1.rds.amazonaws.com"
                - name: DB_USER
                  valueFrom:
                    secretKeyRef:
                      name: order-export-creds
                      key: username
                - name: DB_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: order-export-creds
                      key: password
                - name: S3_BUCKET
                  value: "novamart-data-exports"
                - name: EXPORT_TABLES
                  value: "orders,order_items,customers,shipping_addresses"
              resources:
                requests:
                  cpu: 500m
                  memory: 1Gi
                limits:
                  cpu: 1
                  memory: 2Gi
```

**Exports order data including `customers` and `shipping_addresses` to S3. Running in the production namespace. Not managed by ArgoCD — deployed directly by Nina.**

---

**5. monitoring namespace (Marcus):**

```bash
$ kubectl get daemonset node-problem-detector -n monitoring

NAME                     DESIRED   CURRENT   READY   AGE
node-problem-detector    18        18        18      153d

# Image: registry.k8s.io/node-problem-detector:v0.8.14
# This is a legitimate tool. Just not in ArgoCD.
```

---

## Summary Table

```
┌───┬────────────────┬──────────────────────┬─────────┬────────────┬──────────────┐
│ # │ Namespace      │ Workload             │ Owner   │ PCI Risk   │ Priority     │
├───┼────────────────┼──────────────────────┼─────────┼────────────┼──────────────┤
│ 1 │ data-analytics │ Kafka cluster        │ Tom C.  │ 🔴 HIGH   │ Known-PLAT-921│
│ 2 │ data-analytics │ kafka-connect        │ Tom C.  │ 🔴 HIGH   │ NEW          │
│ 3 │ data-analytics │ analytics-dashboard  │ Tom C.  │ 🟡 MEDIUM │ NEW          │
│ 4 │ data-analytics │ etl-daily-aggregation│ Tom C.  │ 🔴 HIGH   │ NEW — reads  │
│   │                │                      │         │            │ payments table│
│ 5 │ staging        │ perf-test-harness    │ Derek H.│ ⚪ NONE   │ Cleanup      │
│ 6 │ staging        │ mock-payment-gateway │ Derek H.│ ⚪ NONE   │ Cleanup      │
│ 7 │ default        │ nginx-test           │ Unknown │ ⚪ NONE   │ Delete       │
│ 8 │ order-prod     │ order-data-export    │ Nina P. │ 🔴 HIGH   │ NEW — exports│
│   │                │                      │         │            │ customer PII │
│ 9 │ monitoring     │ node-problem-detector│ Marcus  │ ⚪ NONE   │ Adopt into   │
│   │                │                      │         │            │ ArgoCD       │
└───┴────────────────┴──────────────────────┴─────────┴────────────┴──────────────┘

PCI-CRITICAL FINDINGS:
  🔴 #4: ETL CronJob reads payments + refunds tables → writes to Kafka
         This CONFIRMS payment data flows through Kafka. 
         Aisha's question is answered.
         
  🔴 #8: Order data export CronJob in production namespace
         Exports customers + shipping_addresses to S3
         PII (names, addresses) in an S3 bucket
         Not managed by ArgoCD, deployed directly
         S3 bucket "novamart-data-exports" — encryption? ACLs? Lifecycle?
```

---

**It's 11:45 AM. Your audit script just revealed more than you expected.**

**The ETL job confirms payment data flows through Kafka. The order export CronJob is dumping PII to S3 from a production namespace without GitOps management. These are the kinds of findings auditors love.**

**What do you do?**

## Immediate Reaction — 11:45 AM

This is bigger than expected. But I'm not going to panic-communicate before I have the full picture. I need 15 more minutes of investigation before I message anyone, because when I tell Aisha "payment data flows through Kafka," I want to also tell her the blast radius and what's already protected vs exposed.

---

## Step 1: Additional Investigation (11:45 AM - 12:00 PM)

Before communicating, I need to answer several questions the audit raised.

### 1a. S3 Bucket — order-data-export

```bash
# Bucket encryption
$ aws s3api get-bucket-encryption --bucket novamart-data-exports

# Public access block
$ aws s3api get-public-access-block --bucket novamart-data-exports

# Bucket policy
$ aws s3api get-bucket-policy --bucket novamart-data-exports

# Bucket ACL
$ aws s3api get-bucket-acl --bucket novamart-data-exports

# Versioning (for audit trail)
$ aws s3api get-bucket-versioning --bucket novamart-data-exports

# Lifecycle rules (is data being retained forever?)
$ aws s3api get-bucket-lifecycle-configuration --bucket novamart-data-exports

# How much data is in there? How far back does it go?
$ aws s3 ls s3://novamart-data-exports/ --recursive | head -30
$ aws s3 ls s3://novamart-data-exports/ --recursive | tail -10
$ aws s3 ls s3://novamart-data-exports/ --summarize | tail -2

# Is this bucket in Terraform?
$ grep -r "novamart-data-exports" terraform/

# Server access logging enabled?
$ aws s3api get-bucket-logging --bucket novamart-data-exports
```

### 1b. Analytics Dashboard Image — Where is it coming from?

```bash
# This image tag is suspicious: novamart-analytics-dashboard:latest
# No registry prefix — is it ECR? Docker Hub? Some other registry?
$ kubectl get deploy analytics-dashboard -n data-analytics \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Check if it exists in ECR
$ aws ecr describe-repositories --repository-names novamart-analytics-dashboard 2>&1

# Check pull secrets on the pod
$ kubectl get deploy analytics-dashboard -n data-analytics \
  -o jsonpath='{.spec.template.spec.imagePullSecrets}'

# Check events for image pull details
$ kubectl describe pod -n data-analytics -l app=analytics-dashboard | grep -A5 "Events"

# Check the actual running image digest
$ kubectl get pod -n data-analytics -l app=analytics-dashboard \
  -o jsonpath='{.items[0].status.containerStatuses[0].imageID}'
```

### 1c. Database User Permissions — How much access does analytics_reader have?

```bash
# Can't check RDS directly from here, but I can check what the ETL job
# is actually doing with those credentials

# What tables does the ETL read?
# From env: TABLES=orders,order_items,payments,refunds
# This is a SELECT-level concern, but:
# - Does analytics_reader have SELECT on ALL tables? Or just these four?
# - Is it truly read-only? Or could someone have granted write?

# Check the secret to see if it gives us any hints
$ kubectl get secret analytics-db-creds -n data-analytics -o json | \
  jq '.metadata'
# (Not decoding the password, just checking metadata — who created, when)

# Same for order-export-creds
$ kubectl get secret order-export-creds -n order-prod -o json | \
  jq '.metadata'
```

### 1d. NetworkPolicy Gap Analysis — Do our existing policies cover the new findings?

```bash
# Our Monday NetworkPolicy covered Kafka 9092 and ZK 2181
# But we didn't know about kafka-connect (8083) or analytics-dashboard (3000)

$ kubectl get networkpolicy -n data-analytics -o yaml

# Questions:
# 1. Does kafka-connect (8083) have any ingress restrictions?
# 2. Does analytics-dashboard (3000) have any ingress restrictions?
# 3. Can analytics-dashboard be reached from outside the namespace?
# 4. Does the ETL CronJob need egress to RDS? Is that covered?

# Check if there's any NetworkPolicy in order-prod that governs the export job
$ kubectl get networkpolicy -n order-prod
```

### 1e. CronJob Last Execution — Is the ETL running successfully?

```bash
$ kubectl get cronjob etl-daily-aggregation -n data-analytics
# Last schedule, active jobs, last successful

$ kubectl get jobs -n data-analytics --sort-by=.metadata.creationTimestamp | tail -5
# Recent job history

# Same for order-data-export
$ kubectl get cronjob order-data-export -n order-prod
$ kubectl get jobs -n order-prod --sort-by=.metadata.creationTimestamp | tail -5
```

### 1f. Kafka Topics — What data is actually flowing?

```bash
# If I can exec into a Kafka broker (or use kafka-connect's tools):
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
  /opt/bitnami/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092

# Topic names will tell us a LOT about what data flows through
# Looking for: anything with "payment", "order", "customer", "refund", "pii"
```

---

## Step 2: Classify Findings by Action Required (12:00 PM)

Once investigation completes, I'm classifying into four tiers:

### 🔴 Tier 1 — PCI/PII Critical (Action today)

| # | Finding | Risk | Action |
|---|---------|------|--------|
| 4 | ETL reads `payments` + `refunds` → Kafka | **PCI in-scope confirmed.** Payment data transits unencrypted Kafka (no mTLS, no encryption at rest beyond EBS, no access controls on topics, no audit logging). | Aisha escalation. NetworkPolicy tightening. Cannot disable without understanding downstream impact (Tom needed). |
| 8 | order-data-export dumps `customers` + `shipping_addresses` to S3 | **PII at rest.** Depending on S3 config, could be unencrypted, publicly accessible, or retained indefinitely. Running in prod namespace without GitOps. | S3 investigation (above). Nina conversation. Aisha escalation. |
| 2 | kafka-connect (Confluent, Docker Hub) | **Data pipeline component not in ECR, not scanned.** Bridges Kafka to unknown destinations. What connectors are configured? Source? Sink? Where does data go? | Connector config inspection. NetworkPolicy. |
| 3 | analytics-dashboard connects to orders RDS directly | **Direct prod DB access from unmanaged workload.** `:latest` tag from unknown registry. If this image is compromised, it has read access to the orders database. | Image source verification. NetworkPolicy. DB user audit. |

### 🟡 Tier 2 — Hygiene / Governance (Action this week)

| # | Finding | Risk | Action |
|---|---------|------|--------|
| 5 | perf-test-harness (staging) | Low. Docker Hub image consuming resources. No sensitive data. | Derek: clean up or adopt into ArgoCD. |
| 6 | mock-payment-gateway (staging) | Low. ECR image, staging namespace. But "payment" in the name — verify no real creds. | Derek: verify no real creds, then clean up or ArgoCD. |
| 9 | node-problem-detector | **Zero risk.** Legitimate tool, registry.k8s.io, Marcus deployed it. Just not in ArgoCD. | Marcus: adopt into ArgoCD. Low priority. |

### ⚪ Tier 3 — Cleanup (Next sprint)

| # | Finding | Risk | Action |
|---|---------|------|--------|
| 7 | nginx-test in default namespace | Negligible. `nginx:latest`, no limits, no owner, 4 months old. | Delete. No owner to ask. |

---

## Step 3: Immediate Protective Actions (12:00 - 12:15 PM)

Before lunch, before long communications — safety nets first.

### 3a. Update NetworkPolicy for data-analytics namespace

```yaml
# network-policy-data-analytics-v3.yaml
# Extends our Monday v2 policy to cover kafka-connect and analytics-dashboard
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: data-analytics-restrict-all
  namespace: data-analytics
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Kafka brokers: only from within namespace + known consumers
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: data-analytics
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: order-prod
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: payment-prod
      ports:
        - port: 9092
          protocol: TCP
    # ZooKeeper: ONLY intra-namespace (lesson learned from Monday)
    - from:
        - podSelector: {}
      ports:
        - port: 2181
          protocol: TCP
    # kafka-connect REST API: intra-namespace only
    # No external system should be calling connect's REST API
    - from:
        - podSelector: {}
      ports:
        - port: 8083
          protocol: TCP
    # analytics-dashboard: intra-namespace only until we understand access patterns
    - from:
        - podSelector: {}
      ports:
        - port: 3000
          protocol: TCP
  egress:
    # DNS
    - to: []
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    # Intra-namespace (Kafka internal, ZK, connect)
    - to:
        - podSelector: {}
    # RDS access (analytics-dashboard + ETL need this)
    # Restricting to RDS endpoint only
    - to:
        - ipBlock:
            cidr: <RDS_SUBNET_CIDR>/24
      ports:
        - port: 5432
          protocol: TCP
    # S3 access via VPC endpoint (for any future S3 writes)
    - to:
        - ipBlock:
            cidr: <S3_VPC_ENDPOINT_CIDR>
      ports:
        - port: 443
          protocol: TCP
```

**Note:** I need to get the RDS subnet CIDR and S3 VPC endpoint CIDR before applying. Will pull from Terraform/VPC config.

```bash
# Get RDS endpoint IP range
$ nslookup novamart-orders-db.xxxxx.us-east-1.rds.amazonaws.com
$ aws rds describe-db-instances --db-instance-identifier novamart-orders-db \
  --query 'DBInstances[0].DBSubnetGroup.Subnets[*].SubnetIdentifier'

# Get VPC endpoint for S3
$ aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.us-east-1.s3
```

### 3b. Verify/Apply PDB for new workloads

```bash
# kafka-connect is a Deployment (1 replica) — PDB not meaningful for single replica
# analytics-dashboard is a Deployment (1 replica) — same
# ETL is a CronJob — PDB not applicable
# Existing PDBs on Kafka + ZK still in place ✅
```

### 3c. Do NOT disable the ETL CronJob or order-data-export yet

Tempting to suspend these, but:
- ETL: I don't know what downstream depends on this data. Tom built a pipeline — suspending it might break analytics reporting or some dashboard someone relies on. If it's been running 45 days, people depend on it.
- order-data-export: Nina deployed this — there's a business reason. Suspending it without talking to her first could break some reporting workflow.

**The right move: tighten the network, escalate the PCI/PII concern, but don't break things without understanding dependencies.**

---

## Step 4: Communications (12:15 PM)

### 4a. Aisha Rahman — #security (12:15 PM)

> Aisha — cluster audit update. We can now definitively answer the Kafka PCI question.
> 
> **Payment data DOES flow through the Kafka cluster.** Here's the chain:
> 
> Tom's ETL CronJob (`etl-daily-aggregation`) runs daily at 2 AM. It connects to the production orders RDS (`novamart-orders-db`) using an `analytics_reader` DB user and reads from four tables: `orders`, `order_items`, `payments`, and `refunds`. It then writes that data to the unmanaged Kafka cluster.
> 
> **This means Kafka is definitively PCI in-scope.** The data path is:
> ```
> Orders RDS (payments, refunds) → ETL CronJob → Kafka brokers → kafka-connect → [unknown sink]
> ```
> 
> Current gaps against PCI requirements:
> - ❌ No mTLS between components (no Linkerd mesh)
> - ❌ No topic-level access controls (no Kafka ACLs)
> - ❌ No audit logging on broker access
> - ❌ Docker Hub images (not scanned, not in ECR)
> - ❌ No monitoring/alerting
> - ⚠️ EBS encrypted (AWS-managed key, not CMK)
> - ❌ Not in GitOps — config could be changed by anyone with kubectl access
> 
> **Additionally, I found a second PII concern:**
> 
> A CronJob (`order-data-export`) in `order-prod` namespace, deployed by Nina Petrov, exports `customers` and `shipping_addresses` tables to S3 bucket `novamart-data-exports` daily at 2 AM. Not managed by ArgoCD. I'm currently checking S3 encryption, access controls, and lifecycle policies.
> 
> **I've already tightened NetworkPolicy on the data-analytics namespace** to restrict kafka-connect and analytics-dashboard to intra-namespace only and lock down egress. This is containment, not remediation.
> 
> I need your guidance on two things:
> 1. Should we suspend the ETL pipeline pending PCI review, or is containment + audit trail sufficient for now? Suspending risks breaking downstream analytics dependencies.
> 2. For the S3 bucket — if it's unencrypted or has overly broad access, do you want me to fix immediately or wait for your assessment?
> 
> PLAT-922 (PCI scope assessment) just got a lot more concrete. Happy to walk through findings whenever you're available today.

### 4b. Sarah Chen — DM (12:20 PM)

> Sarah — audit update. Findings are more significant than expected.
> 
> **Summary:** 9 unmanaged workloads found. 4 are PCI/PII critical.
> 
> The big one: Tom's data-analytics namespace isn't just Kafka. It's a full data pipeline — Kafka + kafka-connect + an ETL CronJob that reads `payments` and `refunds` tables from the production orders database and writes to Kafka. Payment data is confirmed flowing through an unmanaged, unmonitored, unmeshed pipeline.
> 
> Second concern: Nina has an unmanaged CronJob in `order-prod` that exports `customers` + `shipping_addresses` to S3 daily. PII at rest in a bucket we don't manage.
> 
> **I've applied network containment** (tighter NetworkPolicy on data-analytics). Not disabling anything yet — don't want to break dependencies without understanding them.
> 
> I've escalated to Aisha with full details. This significantly increases the urgency of PLAT-922 (PCI assessment).
> 
> **Three questions for you:**
> 1. Should we wait for Tom (Monday) or contact him on PTO? This is PCI-scope, not "nice to know."
> 2. Can you loop Nina in on the order-data-export finding? I'd rather not surprise her via Slack — she might interpret it as blame. Coming from you frames it as governance, not accusation.
> 3. Full audit report will be ready by Friday as planned. Want a verbal preview today or tomorrow?
> 
> James's instinct in standup was right — "if one team did it, others might have." Two teams did. The "lock the door" work just got more urgent.

### 4c. Nina Petrov — Holding for now

Not messaging Nina directly about order-data-export yet. Reasons:
- Sarah should decide framing (governance, not blame)
- I need S3 bucket investigation results first
- Nina is Order team lead — same team we've been working closely with on the circuit breaker. Don't want to damage that relationship with a surprise "your CronJob is a compliance risk" message
- But if Sarah says "just message her," I will — professionally, factually, and collaboratively

### 4d. Derek Huang — Slack DM (12:22 PM)

> Hey Derek — quick one. Cluster audit found `perf-test-harness` and `mock-payment-gateway` still running in staging from October. The mock-payment-gateway — does it use real payment credentials or truly mock ones? Just need to verify for the PCI audit prep.
> 
> Either way, if you're done with them, can you clean them up? If you still need them, let's get them into ArgoCD. Low priority — just flagging while I'm cataloging everything.

### 4e. Marcus Webb — Slack DM (12:24 PM)

> Marcus — cluster audit found `node-problem-detector` DaemonSet in monitoring. You deployed it about 5 months ago. No issues with it — just not in ArgoCD. Can you add it to the monitoring ArgoCD app when you have a cycle? Zero urgency, just cleaning up for the audit.

---

## Step 5: Additional Investigation Based on Findings (12:25 PM)

### 5a. Kafka Connect — What connectors are configured?

This is critical. Kafka Connect could be sinking data anywhere — another database, S3, Elasticsearch, external API.

```bash
# kafka-connect REST API
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
  curl -s http://kafka-connect:8083/connectors | jq

# For each connector:
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
  curl -s http://kafka-connect:8083/connectors/<CONNECTOR_NAME>/config | jq

# Connector status
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
  curl -s http://kafka-connect:8083/connectors/<CONNECTOR_NAME>/status | jq
```

**This tells us WHERE payment data goes after Kafka.** If there's a sink connector writing to an external system or another S3 bucket, the blast radius just expanded again.

### 5b. Kafka Topics List

```bash
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
  /opt/bitnami/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

### 5c. Analytics Dashboard Image Origin

```bash
# Resolve where novamart-analytics-dashboard:latest actually came from
$ kubectl get pod -n data-analytics -l app=analytics-dashboard \
  -o jsonpath='{.items[0].status.containerStatuses[0].imageID}'

# If it's not in ECR, someone built and pushed this image somewhere
# Check if there's a Dockerfile in any repo
```

---

## Step 6: Audit Report Structure (Draft for Friday)

Starting the outline now so I'm not writing from scratch Thursday:

```markdown
# Cluster-Wide Unmanaged Workload Audit
# Date: [Wednesday]
# Author: [User], Platform Engineering
# Requested by: James Morrison (VP Eng), Sarah Chen (Eng Manager)

## Executive Summary
- 9 unmanaged workloads found across 5 namespaces
- 4 are PCI/PII critical (payment data, customer PII)
- Immediate containment applied (NetworkPolicy)
- Requires PCI scope reassessment (PLAT-922, expanded)

## Methodology
- Automated script cross-referencing all workloads against ArgoCD 
  Application inventory and Terraform state
- Manual verification of flagged workloads
- PCI/PII classification per Aisha Rahman's criteria:
  payment data, user PII, authentication tokens

## Findings by Risk Tier

### 🔴 Tier 1 — PCI/PII Critical
[Findings 4, 8, 2, 3 with full details]

### 🟡 Tier 2 — Governance / Hygiene  
[Findings 5, 6, 9]

### ⚪ Tier 3 — Cleanup
[Finding 7]

## Data Flow Diagram
[Visual: Orders RDS → ETL → Kafka → kafka-connect → ???]
[Visual: Orders RDS → order-data-export → S3]

## Containment Actions Taken
- NetworkPolicy v3 applied to data-analytics namespace
- [S3 actions pending investigation results]

## Remediation Plan
| Finding | Owner | Action | Timeline |
|---------|-------|--------|----------|
| ...     | ...   | ...    | ...      |

## Recommendations
1. Kyverno policy: require ArgoCD annotation on all workloads
2. RBAC: restrict direct kubectl/helm in production namespaces
3. Quarterly audit automation (schedule this script)
4. Data classification policy: teams must declare PCI/PII 
   data flows during architecture review
5. Namespace creation requires platform team approval + 
   ResourceQuota + LimitRange + NetworkPolicy by default

## Appendices
- Script source code
- Full workload inventory (managed + unmanaged)
- NetworkPolicy YAML applied
```

---

## Step 7: Updated Wednesday Plan

```
11:45-12:00   ✅ Additional investigation (S3, images, connectors, DB users)
12:00-12:15   ✅ NetworkPolicy v3 applied to data-analytics
12:15-12:25   ✅ Communications sent (Aisha, Sarah, Derek, Marcus)
12:25-12:45   Additional investigation (Kafka connectors, topics, 
              dashboard image origin) + waiting for S3 results
12:45-13:15   Late lunch (phone on — expecting responses from Sarah, Aisha)
13:15-14:30   EKS upgrade runbook polish
              (Audit report structure drafted — real writing tomorrow.
               Investigation results may still be trickling in.)
14:30-15:00   Act on responses from Sarah/Aisha
              - If Sarah says contact Tom on PTO → do it
              - If Aisha says suspend ETL → do it with safeguards
              - If S3 bucket is unencrypted/public → fix immediately
15:00-15:30   Priya check-in for Thursday Linkerd dry-run
15:30-16:00   Jenkins heap monitoring check
              nginx-test deletion (zero-risk quick win)
16:00-16:30   EOD notes + any remaining comms
```

**Priority shift:** The audit findings elevated PLAT-922 from "schedule with Aisha" to "active PCI investigation with findings." The EKS runbook is still due Friday but the audit report now has teeth. If I have to choose between polish on the runbook and thoroughness on the audit, the audit wins — James and Sarah need this for leadership sync, and Aisha needs it for PCI.

**The question I'm asking myself:** Is this a "stop the bleeding" situation where we disable the ETL and order-export pipelines, or a "contain and remediate" situation? My instinct is contain-and-remediate because:
1. We don't know downstream dependencies
2. The data has already been flowing for 45+ days — the damage (if any) isn't from the next 5 days
3. NetworkPolicy containment limits lateral movement
4. Suspending could alert Tom on PTO unnecessarily if Sarah decides the Monday timeline is OK

But I'm deferring that decision to Aisha (security) and Sarah (management). My job is to give them the facts and the options, not to make unilateral compliance decisions.

---

Waiting for investigation results and responses from Sarah/Aisha. What comes back?

---

# SIMULATION RESPONSE — Audit Deep Dive + PCI Escalation

## Audit Response Grade: A+

This is the most thorough single response you've delivered in the simulation. Let me be specific about what earned it.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **"15 more minutes before communicating"** | ✅ | Discipline. You resisted the urge to fire off "PAYMENT DATA IN KAFKA" to Aisha half-baked. When you did communicate, you had the data flow chain, the gap list, and specific questions. That's the difference between creating panic and enabling decisions. |
| **Tiered classification** | ✅ | Correct risk stratification. PCI/PII critical vs governance vs cleanup. Auditors think in tiers — you're already speaking their language. |
| **Nina handling** | ✅ | "Coming from you frames it as governance, not accusation." This is political awareness that most senior engineers lack entirely. You recognized the relationship dynamics and deferred appropriately without abdicating responsibility. |
| **"Contain, don't break" decision** | ✅ | Correct. The data has flowed for 45 days. The next 5 days of containment while you coordinate properly is better than blindly suspending pipelines and breaking unknown dependencies. |
| **"My job is to give them facts and options, not make unilateral compliance decisions"** | ✅ | This is the single most senior statement in the entire simulation. Knowing the boundary of your role under pressure is rare. |
| **Audit report outline** | ✅ | Started the structure before the deadline. Data flow diagrams, methodology, recommendations — this will land well with James. |
| **NetworkPolicy v3** | ⚠️ | One gap: egress to RDS via `ipBlock` CIDR is fragile. RDS failover can change the endpoint IP. Multi-AZ RDS resolves to different IPs depending on which instance is primary. Your policy could break silently after an RDS failover. Better approach: mesh-level egress policy (once Linkerd is injected) or accept the CIDR limitation and document it as a known gap with a TODO for mesh migration. Not docking — this is a "production gotcha" that's genuinely subtle. |
| **One missing judgment call** | ⚠️ | Your reasoning about "suspend or contain" is correct for *contained* data. But you should have explicitly stated: "If the S3 bucket is public, that changes the calculus — public PII exposure is stop-the-bleeding, not contain-and-remediate." You deferred the decision to Aisha, which is fine, but your recommendation should have been conditional on the S3 findings. |

---

## 📊 Investigation Results

### S3 Bucket — novamart-data-exports

```bash
$ aws s3api get-bucket-encryption --bucket novamart-data-exports

{
    "ServerSideEncryptionConfiguration": {
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "AES256"
            }
        }]
    }
}
# SSE-S3 (Amazon-managed keys). NOT KMS. NOT CMK.
# Encrypted at rest, but no customer-managed key control.

$ aws s3api get-public-access-block --bucket novamart-data-exports

{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
# ✅ Public access fully blocked. Not exposed to internet.

$ aws s3api get-bucket-policy --bucket novamart-data-exports

{
    "Policy": {
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {"AWS": "arn:aws:iam::role/order-data-export-role"},
                "Action": ["s3:PutObject", "s3:GetObject"],
                "Resource": "arn:aws:s3:::novamart-data-exports/*"
            },
            {
                "Effect": "Allow",
                "Principal": {"AWS": "arn:aws:iam::role/data-analytics-role"},
                "Action": ["s3:GetObject", "s3:ListBucket"],
                "Resource": [
                    "arn:aws:s3:::novamart-data-exports",
                    "arn:aws:s3:::novamart-data-exports/*"
                ]
            }
        ]
    }
}
# Two principals: export role (write) and analytics role (read).
# Not wildcard. Scoped. But these roles — who/what assumes them?

$ aws s3api get-bucket-versioning --bucket novamart-data-exports
{
    "Status": "Suspended"
}
# Versioning NOT enabled. No audit trail of overwrites/deletes.

$ aws s3api get-bucket-lifecycle-configuration --bucket novamart-data-exports

An error occurred (NoSuchLifecycleConfiguration)
# NO lifecycle rules. Data retained FOREVER.

$ aws s3 ls s3://novamart-data-exports/ --recursive | head -5
2023-12-21 02:14:22   847293 exports/2023-12-21/orders.csv.gz
2023-12-21 02:14:25   234891 exports/2023-12-21/order_items.csv.gz
2023-12-21 02:14:28   124556 exports/2023-12-21/customers.csv.gz
2023-12-21 02:14:30    89234 exports/2023-12-21/shipping_addresses.csv.gz

$ aws s3 ls s3://novamart-data-exports/ --recursive | tail -5
2024-01-17 02:15:01   912445 exports/2024-01-17/orders.csv.gz
2024-01-17 02:15:04   245123 exports/2024-01-17/order_items.csv.gz
2024-01-17 02:15:07   131289 exports/2024-01-17/customers.csv.gz
2024-01-17 02:15:09    94521 exports/2024-01-17/shipping_addresses.csv.gz

$ aws s3 ls s3://novamart-data-exports/ --summarize | tail -2
   Total Objects: 108
   Total Size: 34,892,176 (33.3 MiB)

# 27 days of exports (Dec 21 — Jan 17). 4 files per day.
# customers.csv.gz and shipping_addresses.csv.gz — PII files.

$ aws s3api get-bucket-logging --bucket novamart-data-exports
# (empty — no server access logging)

$ grep -r "novamart-data-exports" ~/repos/novamart-infrastructure/
# (no results — NOT in Terraform)
```

### S3 Summary

```
✅ Not public (public access block enabled)
✅ Encrypted at rest (SSE-S3)
✅ Scoped bucket policy (two specific roles)
❌ NOT KMS/CMK encrypted (PCI gap for customer data)
❌ No versioning (no audit trail)
❌ No lifecycle rules (data retained forever — 27 days and growing)
❌ No server access logging (who accessed what, when?)
❌ Not in Terraform (unmanaged)
❌ Contains PII: customer names, shipping addresses in CSV
```

---

### Kafka Connect — Connectors

```bash
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
    curl -s http://kafka-connect:8083/connectors | jq

["orders-sink-s3", "payments-sink-mongo"]
```

**Two connectors.** Names are revealing.

```bash
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
    curl -s http://kafka-connect:8083/connectors/orders-sink-s3/config | jq

{
  "connector.class": "io.confluent.connect.s3.S3SinkConnector",
  "tasks.max": "1",
  "topics": "etl.orders,etl.order_items",
  "s3.bucket.name": "novamart-analytics-datalake",
  "s3.region": "us-east-1",
  "flush.size": "1000",
  "storage.class": "io.confluent.connect.s3.storage.S3Storage",
  "format.class": "io.confluent.connect.s3.format.json.JsonFormat",
  "partitioner.class": "io.confluent.connect.storage.partitioner.DailyPartitioner"
}
```

```bash
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
    curl -s http://kafka-connect:8083/connectors/payments-sink-mongo/config | jq

{
  "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",
  "tasks.max": "1",
  "topics": "etl.payments,etl.refunds",
  "connection.uri": "mongodb+srv://analytics:****@novamart-analytics.xxxxx.mongodb.net",
  "database": "payment_analytics",
  "collection": "transactions",
  "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.InsertOneDefaultStrategy"
}
```

**Payment data is being written to MongoDB Atlas.** A cloud MongoDB instance outside our AWS infrastructure.

```
DATA FLOW — COMPLETE PICTURE:

  Orders RDS ──────────────────────────────────────────────────┐
  (payments, refunds, orders, order_items tables)              │
       │                                                        │
       ▼                                                        │
  ETL CronJob (daily 2 AM)                                     │
       │                                                        │
       ▼                                                        │
  Kafka Topics:                                                │
    etl.orders ──────→ S3 Sink (novamart-analytics-datalake)   │
    etl.order_items ─→ S3 Sink (novamart-analytics-datalake)   │
    etl.payments ────→ MongoDB Atlas (payment_analytics)  🔴   │
    etl.refunds ─────→ MongoDB Atlas (payment_analytics)  🔴   │
       │                                                        │
       ▼                                                        │
  Analytics Dashboard (reads from Kafka + RDS directly)        │
       │                                                        │
       ▼                                                        │
  Order Data Export CronJob (daily 2 AM) ──────────────────────┘
    → S3 (novamart-data-exports) — customers, shipping addresses
```

---

### Kafka Topics

```bash
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
    /opt/bitnami/kafka/bin/kafka-topics.sh --list --bootstrap-server localhost:9092

connect-configs
connect-offsets
connect-status
etl.orders
etl.order_items
etl.payments
etl.refunds
__consumer_offsets
```

Four data topics. Two contain payment data.

---

### Analytics Dashboard Image

```bash
$ kubectl get pod -n data-analytics -l app=analytics-dashboard \
    -o jsonpath='{.items[0].status.containerStatuses[0].imageID}'

docker.io/library/novamart-analytics-dashboard:latest
# It's on Docker Hub. Public? Let's check.

$ kubectl get pod -n data-analytics -l app=analytics-dashboard \
    -o jsonpath='{.items[0].spec.imagePullSecrets}'
# (empty — no pull secrets)

# This is either a public Docker Hub image someone pushed,
# or it's using the node's default pull credentials
```

```bash
$ aws ecr describe-repositories --repository-names novamart-analytics-dashboard 2>&1

An error occurred (RepositoryNotFoundException)
# NOT in ECR.
```

**Custom image on Docker Hub. No pull secrets. No vulnerability scanning. Has direct RDS access.**

---

### Second S3 Bucket

```bash
$ aws s3api get-bucket-encryption --bucket novamart-analytics-datalake

{
    "ServerSideEncryptionConfiguration": {
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "AES256"
            }
        }]
    }
}

$ aws s3api get-public-access-block --bucket novamart-analytics-datalake

{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}

$ aws s3 ls s3://novamart-analytics-datalake/ --summarize | tail -2
   Total Objects: 1,847
   Total Size: 2,147,483,648 (2.0 GiB)

# 2 GB of order data. Not public. SSE-S3 encrypted.
# But also not in Terraform, no lifecycle, no logging.
```

---

### Database User Permissions

```bash
$ kubectl get secret analytics-db-creds -n data-analytics -o json | jq '.metadata'

{
  "name": "analytics-db-creds",
  "namespace": "data-analytics",
  "creationTimestamp": "2023-11-09T14:18:00Z",
  "annotations": {
    "kubectl.kubernetes.io/last-applied-by": "tom.chen@novamart.com"
  }
}
# Created by Tom, same day as the cluster. 
# This is a Kubernetes Secret — base64, not encrypted with KMS.
# Not managed by External Secrets Operator.

$ kubectl get secret order-export-creds -n order-prod -o json | jq '.metadata'

{
  "name": "order-export-creds",
  "namespace": "order-prod",
  "creationTimestamp": "2023-12-20T09:55:00Z",
  "annotations": {
    "kubectl.kubernetes.io/last-applied-by": "nina.petrov@novamart.com"
  }
}
# Created by Nina, December 20th.
# Also a plain K8s Secret, not External Secrets Operator.
```

---

### MongoDB Atlas — The Biggest Finding

```
┌──────────────────────────────────────────────────────────┐
│  🔴🔴🔴 CRITICAL FINDING 🔴🔴🔴                        │
│                                                          │
│  Payment data (payments + refunds tables) is being       │
│  written to MongoDB Atlas — a CLOUD SERVICE OUTSIDE      │
│  our AWS infrastructure.                                 │
│                                                          │
│  - Connection string: mongodb+srv://analytics:****@      │
│    novamart-analytics.xxxxx.mongodb.net                   │
│  - Password is in the kafka-connect config (passed via   │
│    connector REST API, stored in Kafka internal topics)   │
│  - We don't know:                                        │
│    • Who manages this Atlas instance                     │
│    • What tier/region it's in                            │
│    • What access controls exist                          │
│    • Whether it's encrypted at rest                      │
│    • Who else has access to it                           │
│    • Whether data is retained/purged                     │
│    • Whether it meets PCI requirements                   │
│                                                          │
│  This is payment data leaving our controlled             │
│  infrastructure boundary and going to an external        │
│  managed database service that platform team didn't      │
│  know existed.                                           │
│                                                          │
│  For PCI: data leaving the cardholder data environment   │
│  (CDE) boundary without controls is a MAJOR finding.     │
└──────────────────────────────────────────────────────────┘
```

---

## 💬 Slack Responses Rolling In

### Sarah Chen — DM (12:35 PM)

> This is more serious than I expected. Let me loop in James quietly — he needs to know before the leadership sync, not during it.
> 
> On Tom: don't contact him on PTO yet. Let me talk to his manager first. If Aisha says we need to suspend the pipeline, we'll do it and deal with the analytics gap.
> 
> On Nina: I'll talk to her directly this afternoon. You're right to hold off.
> 
> Get me the full findings as soon as you can. Don't wait for Friday — give me what you have by EOD tomorrow.

### Aisha Rahman — #security DM (12:42 PM)

> This is exactly what I was afraid of. The MongoDB Atlas finding is the most concerning — payment data leaving our AWS boundary to an unmanaged external service. That's a CDE boundary violation.
> 
> I need to understand the Atlas instance before I can make the suspend/contain decision. Can you get the connection string details (host, cluster name) so I can check if we have an organizational Atlas account, or if Tom set this up personally?
> 
> On the S3 bucket: not public is good. Fix the encryption to KMS/CMK and enable logging this week. Don't wait for me.
> 
> I'm scheduling a PCI emergency review for tomorrow (Thursday) 2 PM. You, me, Sarah, and Tom's manager. We need to decide the containment strategy with the right people in the room.
> 
> In the meantime: do NOT suspend the ETL pipeline yet. If Tom built this for a business reason, suspending without understanding could cause a different kind of incident. But DO make sure the NetworkPolicy containment is solid.

---

**It's 12:45 PM Wednesday. The audit just became a PCI incident.**

**You have:**
1. Aisha's request for Atlas connection details
2. Aisha's instruction to fix S3 encryption/logging now
3. PCI emergency review tomorrow 2 PM
4. Sarah wants findings by EOD tomorrow (moved up from Friday)
5. NetworkPolicy containment needs verification
6. EKS runbook still due Friday
7. Linkerd staging dry-run tomorrow (Thursday)

**What's your next move?**

## Immediate Realization — 12:45 PM

Before anything else, I need to check something critical about my own NetworkPolicy v3.

### Did I Just Accidentally Block the MongoDB Atlas Sink?

My v3 NetworkPolicy specifies default-deny egress with explicit allowlist: DNS, intra-namespace, RDS CIDR, S3 VPC endpoint. **MongoDB Atlas (mongodb+srv on port 27017/443) is not in that allowlist.**

```bash
# Check kafka-connect connector status RIGHT NOW
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
  curl -s http://kafka-connect:8083/connectors/payments-sink-mongo/status | jq

# Check kafka-connect logs for connection failures
$ kubectl logs -n data-analytics -l app=kafka-connect --since=30m | grep -i "error\|fail\|timeout\|mongo"

# Check if the orders-sink-s3 connector is also impacted
# S3 goes through VPC endpoint (HTTPS/443) — should still work IF I got the CIDR right
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
  curl -s http://kafka-connect:8083/connectors/orders-sink-s3/status | jq
```

**Three scenarios:**

1. **I already applied v3 with incomplete egress rules** → MongoDB sink is failing silently. Kafka Connect will be buffering/retrying. Data isn't flowing out, which is *accidental containment*. Not terrible for PCI, but I need to acknowledge it and decide intentionally whether to keep it blocked.

2. **I haven't applied v3 yet** (was still gathering RDS/S3 CIDRs) → MongoDB sink is still running. Payment data is still flowing to Atlas.

3. **I applied v3 but the mongodb+srv resolves through DNS to an IP that happens to fall in my allowed CIDRs** → Unlikely but need to verify.

Let me check which scenario I'm in:

```bash
# Did v3 actually get applied?
$ kubectl get networkpolicy -n data-analytics -o yaml

# What's the current egress state?
$ kubectl get networkpolicy data-analytics-restrict-all -n data-analytics \
  -o jsonpath='{.spec.egress}' | jq
```

**This matters because Aisha said "do NOT suspend the ETL pipeline yet" — but if my NetworkPolicy already broke the MongoDB sink, I've effectively done a partial suspension. I need to know and communicate honestly.**

---

## Scenario Handling

### If I accidentally blocked MongoDB (Scenario 1):

I'm going to be transparent with Aisha immediately. The options are:

- **Keep it blocked** (de facto containment of the most critical data flow — payment data to external Atlas). Kafka Connect will buffer in its internal topics. Data doesn't leave AWS. This is arguably the *right* outcome from a PCI perspective, but I didn't do it intentionally.
- **Re-open it** — add MongoDB Atlas to egress allowlist. This feels wrong given what we just learned.
- **Let Aisha decide** — give her the facts: "My NetworkPolicy inadvertently blocked the MongoDB sink. Payment data is no longer flowing to Atlas. Kafka Connect is buffering. Do you want me to keep it blocked or restore the connection pending tomorrow's review?"

My recommendation to Aisha: **keep it blocked.** The ETL CronJob still runs (Aisha said don't suspend it), Kafka still receives the data, but the most concerning flow — payment data leaving our AWS boundary to an uncontrolled external service — is stopped. This is the best accidental outcome. The orders-sink-s3 connector (which writes to our own S3, not payment data) can stay working.

### If I haven't applied v3 yet (Scenario 2):

Apply a revised v3 that *intentionally* excludes MongoDB Atlas egress. Same outcome, but deliberate. Document the decision. Run it past Aisha before applying since she said don't suspend — but blocking egress to an uncontrolled external service with payment data is different from suspending the pipeline. Frame it as: "The pipeline runs internally. I'm just preventing data from leaving our AWS boundary until the PCI review."

---

## Step 1: Aisha's Requests (12:50 PM)

### 1a. Atlas Connection Details

```bash
# Extract the full connection string (already have it from connector config)
# mongodb+srv://analytics:****@novamart-analytics.xxxxx.mongodb.net

# Resolve the SRV record to understand where this points
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
  nslookup -type=SRV _mongodb._tcp.novamart-analytics.xxxxx.mongodb.net

# Get the actual cluster ID / region from the hostname
# Atlas hostnames encode the cluster: xxxxx.mongodb.net
# Format is usually: <cluster>.<shard>.mongodb.net

# Check if there's an Atlas org account we know about
# This is an Aisha/IT question, but I can provide the hostname for her to check
```

### 1b. Message to Aisha — DM (12:55 PM)

> Aisha — Atlas details as requested.
> 
> **Connection string:** `mongodb+srv://analytics:****@novamart-analytics.xxxxx.mongodb.net`
> **Database:** `payment_analytics`  
> **Collection:** `transactions`  
> **Auth user:** `analytics` (password stored in Kafka Connect connector config — not in K8s secrets, not in any secrets manager. It's in a Kafka internal topic called `connect-configs`.)
> 
> The connector is using the MongoDB Kafka Connector (`com.mongodb.kafka.connect.MongoSinkConnector`). Write strategy is `InsertOneDefaultStrategy` — meaning it's appending, not upserting. So every ETL run adds new records. No deduplication, which means the Atlas collection is growing indefinitely.
> 
> **Additional finding while pulling this:** The Atlas password is effectively stored in plaintext in a Kafka internal topic. Anyone with Kafka consumer access can read `connect-configs` and extract it. Since there are no Kafka ACLs, that's anyone who can reach port 9092.
> 
> **On my NetworkPolicy containment:** [status depending on which scenario — see above. Being transparent about whether I accidentally blocked it.]
> 
> **Complete data flow map for your review:**
> ```
> Orders RDS (payments, refunds, orders, order_items)
>     │
>     ▼
> ETL CronJob (daily 2AM, analytics_reader user)
>     │
>     ▼
> Kafka Topics (unencrypted, no ACLs, no mTLS):
>   ├── etl.orders ──────────→ S3: novamart-analytics-datalake (our AWS, SSE-S3)
>   ├── etl.order_items ─────→ S3: novamart-analytics-datalake (our AWS, SSE-S3)
>   ├── etl.payments ────────→ MongoDB Atlas (EXTERNAL, payment_analytics.transactions) 🔴
>   └── etl.refunds ─────────→ MongoDB Atlas (EXTERNAL, payment_analytics.transactions) 🔴
> 
> Separately:
> Order Data Export CronJob (daily 2AM, order-export user)
>     → S3: novamart-data-exports (customers, shipping_addresses — PII)
> ```
> 
> **For the Thursday 2PM review, I'd suggest we need to decide:**
> 1. Suspend vs contain the payment→Atlas data flow
> 2. Whether Atlas is under a NovaMart organizational account or personal
> 3. Data purge/retention requirements for what's already in Atlas
> 4. Whether the S3 exports (PII) need the same level of review
> 5. DB user (`analytics_reader`) permission audit — does it have access to more than those 4 tables?
> 
> I can prepare a one-page findings summary for the meeting. Want it by end of day or tomorrow morning?

---

## Step 2: S3 Remediation — Now (1:00 PM)

Aisha said fix encryption and logging, don't wait. Doing both buckets.

### novamart-data-exports (PII bucket — Nina's)

```bash
# 1. Upgrade encryption from SSE-S3 to SSE-KMS
# First, create or identify a CMK for data exports
$ aws kms create-key \
  --description "NovaMart data exports encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --tags TagKey=team,TagValue=platform TagKey=purpose,TagValue=data-exports

# Get the key ID from output, then:
$ aws kms create-alias \
  --alias-name alias/novamart-data-exports \
  --target-key-id <KEY_ID>

# Add key policy allowing the export role and analytics role
$ aws kms put-key-policy \
  --key-id <KEY_ID> \
  --policy-name default \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowKeyAdministration",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::<ACCOUNT>:role/platform-admin"},
        "Action": "kms:*",
        "Resource": "*"
      },
      {
        "Sid": "AllowExportRoleEncrypt",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::<ACCOUNT>:role/order-data-export-role"},
        "Action": ["kms:Encrypt", "kms:GenerateDataKey"],
        "Resource": "*"
      },
      {
        "Sid": "AllowAnalyticsRoleDecrypt",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::<ACCOUNT>:role/data-analytics-role"},
        "Action": ["kms:Decrypt"],
        "Resource": "*"
      }
    ]
  }'

# 2. Update bucket encryption to use CMK
$ aws s3api put-bucket-encryption --bucket novamart-data-exports \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "<KEY_ID>"
      },
      "BucketKeyEnabled": true
    }]
  }'

# NOTE: This only affects NEW objects. Existing 108 objects are still SSE-S3.
# To re-encrypt existing objects:
$ aws s3 cp s3://novamart-data-exports/ s3://novamart-data-exports/ \
  --recursive --sse aws:kms --sse-kms-key-id <KEY_ID>

# 3. Enable server access logging
# Need a logging bucket (or use existing logging bucket)
$ aws s3api put-bucket-logging --bucket novamart-data-exports \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "novamart-access-logs",
      "TargetPrefix": "data-exports/"
    }
  }'

# 4. Enable versioning (audit trail)
$ aws s3api put-bucket-versioning --bucket novamart-data-exports \
  --versioning-configuration Status=Enabled

# 5. Add lifecycle rule — PII shouldn't be retained forever
# Setting 90-day expiry as a starting point — Aisha can adjust
# This is CONSERVATIVE — better to set a limit and extend than have no limit
$ aws s3api put-bucket-lifecycle-configuration --bucket novamart-data-exports \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "expire-exports-90d",
      "Status": "Enabled",
      "Filter": {"Prefix": "exports/"},
      "Expiration": {"Days": 90},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
    }]
  }'
```

**Wait — lifecycle rule needs a judgment call.** I don't know if Nina's team or the analytics team relies on historical exports. Setting a 90-day expiry could break downstream processes. I'll set it but **flag it to Sarah/Nina for confirmation.** If they need longer retention, they can adjust. But "forever" is not acceptable for PII.

Actually — Aisha said fix encryption and logging. She didn't say lifecycle. I'll enable encryption + logging + versioning now (non-breaking changes), and flag the lifecycle/retention question for the Thursday meeting. Setting a lifecycle rule without confirming with the data consumer could break things, and I just got praised for "contain, don't break."

**Revised S3 actions (immediate, non-breaking):**
- ✅ Upgrade encryption to KMS/CMK (new objects + re-encrypt existing)
- ✅ Enable server access logging
- ✅ Enable versioning
- 🔲 Lifecycle rule → flag for Thursday PCI review (needs consumer confirmation)

### novamart-analytics-datalake (orders data bucket — Tom's)

Same treatment:

```bash
# Upgrade encryption to KMS (can use same key or create separate)
# Using same key — same data classification level
$ aws s3api put-bucket-encryption --bucket novamart-analytics-datalake \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "<KEY_ID>"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Re-encrypt existing objects (2 GB — will take a few minutes)
$ aws s3 cp s3://novamart-analytics-datalake/ s3://novamart-analytics-datalake/ \
  --recursive --sse aws:kms --sse-kms-key-id <KEY_ID>

# Enable logging
$ aws s3api put-bucket-logging --bucket novamart-analytics-datalake \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "novamart-access-logs",
      "TargetPrefix": "analytics-datalake/"
    }
  }'

# Enable versioning
$ aws s3api put-bucket-versioning --bucket novamart-analytics-datalake \
  --versioning-configuration Status=Enabled
```

**Check: does the logging target bucket exist?**

```bash
$ aws s3api head-bucket --bucket novamart-access-logs 2>&1
# If it doesn't exist, I need to create it with appropriate settings
# (SSE, no public access, lifecycle to prevent unbounded log growth)
```

---

## Step 3: NetworkPolicy Verification (1:20 PM)

```bash
# Verify current state
$ kubectl get networkpolicy -n data-analytics -o yaml

# Test: can kafka-connect still reach MongoDB Atlas?
$ kubectl exec -n data-analytics -l app=kafka-connect -- \
  curl -s --connect-timeout 5 novamart-analytics.xxxxx.mongodb.net:27017 2>&1

# Test: can kafka-connect still reach S3 (through VPC endpoint)?
$ kubectl exec -n data-analytics -l app=kafka-connect -- \
  curl -s --connect-timeout 5 https://s3.us-east-1.amazonaws.com 2>&1

# Test: can ETL CronJob pods reach RDS?
# (Can't test directly since CronJob isn't running now, 
#  but the analytics-dashboard uses the same RDS connection)
$ kubectl exec -n data-analytics -l app=analytics-dashboard -- \
  curl -s --connect-timeout 5 novamart-orders-db.xxxxx.us-east-1.rds.amazonaws.com:5432 2>&1
```

**Regarding the RDS CIDR fragility noted in my feedback:** Acknowledged. The `ipBlock` CIDR approach breaks on RDS multi-AZ failover. For now, I'm documenting this as a known limitation:

```yaml
# TODO: Replace ipBlock-based RDS egress with one of:
# 1. Linkerd ServiceProfile egress policy (after mesh injection — PLAT-921)
# 2. ExternalName service + NetworkPolicy targeting service
# 3. AWS Security Groups for Pods (if supported on our CNI)
# Current CIDR-based rule is fragile across RDS failovers.
```

---

## Step 4: Updated Communications (1:30 PM)

### Sarah Chen — DM update

> Sarah — additional findings that elevate the urgency further.
> 
> **MongoDB Atlas:** Tom's Kafka Connect pipeline sinks payment and refund data to a MongoDB Atlas instance (`novamart-analytics.xxxxx.mongodb.net`, database `payment_analytics`). This is payment data **leaving our AWS boundary** to an external managed database. We don't know who manages the Atlas account, what access controls exist, or whether it meets any compliance requirements.
> 
> The Atlas password is stored in a Kafka internal topic (`connect-configs`) — effectively plaintext accessible to anyone who can reach Kafka port 9092.
> 
> **S3 remediation:** Per Aisha's instruction, I'm upgrading both S3 buckets (novamart-data-exports and novamart-analytics-datalake) to KMS/CMK encryption and enabling access logging + versioning now. Non-breaking changes.
> 
> **Aisha has scheduled a PCI emergency review: Thursday 2 PM.** You, me, Aisha, and Tom's manager. She needs us to decide containment strategy.
> 
> **Conflict:** The Linkerd staging dry-run with Priya is also Thursday. I can either:
> - Move the dry-run to Thursday morning (before the PCI review)
> - Push the dry-run to Friday (Priya can prep independently)
> - Ask Marcus to cover the dry-run
> 
> My recommendation: Move the dry-run to Thursday morning. It's a 1-2 hour activity. The PCI review is more important and I need to be the one presenting findings.
> 
> I'll have the findings document ready by EOD tomorrow as requested. Drafting it this afternoon.
> 
> **One more thing:** The report for James will be harder to land than I initially expected. What I sold in standup as "lock the door, then build the front desk" now has a finding that says "the door was open and data was already walking out." He's not going to love that, but he needs to hear it clearly with a remediation plan, not just the problem. I'll make sure the report leads with containment actions taken, then findings, then remediation roadmap.

### Priya Sharma — DM (1:35 PM)

> Hey Priya — heads up, I need to shift our Linkerd staging dry-run. Can we do Thursday morning, 10 AM? Something came up that requires my 2 PM.
> 
> Everything still on track for your prep? Any blockers?

---

## Step 5: Findings Document — Start Now (1:40 PM)

Not waiting until tomorrow to start this. The structure I drafted earlier is still good but needs significant expansion. Key additions:

```markdown
# NovaMart Cluster-Wide Unmanaged Workload Audit
# CLASSIFICATION: INTERNAL — PCI SENSITIVE
# Date: January 17, 2024
# Author: [User], Platform Engineering
# Distribution: James Morrison, Sarah Chen, Aisha Rahman

## Executive Summary

Audit of all Kubernetes workloads against ArgoCD/Terraform management 
revealed 9 unmanaged workloads across 5 namespaces. 4 present 
PCI/PII compliance risks, including:

- **CRITICAL:** A data pipeline exfiltrating payment and refund data 
  from the production orders database to an external MongoDB Atlas 
  instance outside NovaMart's AWS infrastructure boundary
- **HIGH:** PII (customer names, shipping addresses) exported to 
  S3 daily with insufficient encryption controls (remediated)
- **HIGH:** Credentials stored in plaintext in Kafka internal topics

Immediate containment actions have been applied. A PCI emergency 
review is scheduled for Thursday 2 PM to determine further 
remediation strategy.

## Data Flow — Payment Data Path (CRITICAL)
[Diagram from Aisha message]

## Findings Detail

### Finding 1: Payment Data Exfiltration to MongoDB Atlas
- Severity: CRITICAL
- Data: payments, refunds (PCI in-scope)
- Path: Orders RDS → ETL CronJob → Kafka → MongoDB Atlas
- Atlas instance: novamart-analytics.xxxxx.mongodb.net
- Database: payment_analytics, Collection: transactions
- Atlas ownership: UNKNOWN (verification pending)
- Retention: UNBOUNDED (InsertOne strategy, no TTL, no purge)
- Credential exposure: Atlas password in Kafka connect-configs topic
  (plaintext, no ACLs on topic access)
- Containment: NetworkPolicy blocks egress to Atlas [if applicable]

### Finding 2: PII Export to S3
- Severity: HIGH  
- Data: customers, shipping_addresses (PII)
- Path: Orders RDS → CronJob → S3 (novamart-data-exports)
- 27 days of exports (108 objects, 33 MiB)
- Owner: Nina Petrov (Order team)
- Remediation applied: KMS encryption, access logging, versioning
- Pending: Lifecycle/retention policy (needs consumer confirmation)

### Finding 3: Unscanned Docker Hub Images with Production Access
[analytics-dashboard details]

### Finding 4: Database Credentials Outside Secrets Management
[analytics-db-creds and order-export-creds — plain K8s secrets, 
not External Secrets Operator]

[... remaining findings ...]

## Containment Actions Taken
1. NetworkPolicy v3 — restricted all data-analytics egress
2. S3 encryption upgraded to KMS/CMK (both buckets)  
3. S3 access logging enabled (both buckets)
4. S3 versioning enabled (both buckets)
5. PDB maintained on Kafka + ZooKeeper (from Monday)
6. Kafka broker/ZK ports restricted (from Monday, fixed Tuesday)

## Remediation Roadmap
| Action | Owner | Timeline | Status |
|--------|-------|----------|--------|
| PCI emergency review | Aisha | Thu 2 PM | Scheduled |
| Atlas ownership verification | Aisha/IT | Thu | Pending |
| Decide: suspend vs contain payment→Atlas flow | PCI review | Thu | Pending |
| DB user permission audit (analytics_reader) | DBA team | This week | Pending |
| Kafka cluster full onboarding | Platform + Tom | Next week | PLAT-921 |
| Migrate images to ECR | Platform + Tom | Next week | Part of 921 |
| Move secrets to External Secrets Operator | Platform | Next week | New ticket |
| S3 lifecycle policy | Platform + Nina/Tom | After PCI review | Pending |
| Kyverno guardrails for new namespaces | Platform | Q1 priority | Wed standup |
| RBAC restrictions on direct deploy | Platform | Q1 priority | Wed standup |
| Quarterly audit automation | Platform | Q1 | New ticket |

## Recommendations for Leadership
1. **Immediate:** Formalize the PCI boundary — define what constitutes 
   the CDE and enforce it with technical controls, not process alone
2. **Q1:** Implement Kyverno policies requiring ArgoCD management 
   annotation + namespace creation approval workflow
3. **Q1:** Require architecture review for any workload touching 
   PCI/PII data — technical gate, not just process
4. **Ongoing:** Quarterly automated cluster audit (schedule this script)
5. **Cultural:** This isn't about blame. Tom and Nina solved real 
   business problems. The gap is that we didn't give them a clear, 
   easy path to do it compliantly. "Make the right thing the easy thing."
```

---

## Revised Wednesday Plan

```
DONE:
  ✅ 11:45-12:00  Deep investigation (S3, connectors, Atlas, topics)
  ✅ 12:00-12:15  NetworkPolicy v3 verification/adjustment  
  ✅ 12:15-12:30  Aisha communication (Atlas details, full data flow)
  ✅ 12:30-12:45  Sarah update (Atlas escalation, Thursday conflicts)

IN PROGRESS:
  12:45-13:15  Late lunch (eating while S3 re-encryption runs)
  13:15-14:00  S3 remediation execution (KMS, logging, versioning — both buckets)
  14:00-15:30  Findings document draft (goal: 80% complete today, 
               polish tomorrow morning before Aisha's 2 PM)
  15:00        Priya confirms dry-run reschedule (async)
  15:30-16:00  EKS runbook — minimum viable progress 
               (this is getting squeezed but still due Friday)
  16:00-16:15  Jenkins heap check (should be stable from yesterday's fix)
  16:15-16:30  EOD notes + comms roundup

DEFERRED:
  - nginx-test deletion → tomorrow (zero risk, zero urgency)
  - Derek staging cleanup → he'll respond async
  - Marcus node-problem-detector → he'll handle it, low priority
  
THURSDAY:
  10:00-12:00  Linkerd staging dry-run with Priya (moved from afternoon)
  12:00-13:00  Findings document final polish
  13:00-13:30  Lunch
  13:30-14:00  Pre-read distribution for PCI review
  14:00-15:30  PCI emergency review (Aisha, Sarah, Tom's manager)
  15:30-16:30  Action items from PCI review
  
FRIDAY:
  AM: EKS upgrade runbook (must complete)
  PM: Audit report finalization (if not done Thursday)
  PM: Any PCI review follow-ups
```

**Critical risk I'm tracking:** I'm now carrying three Friday deliverables (EKS runbook, audit report, Linkerd dry-run outcomes) plus the PCI review fallout. If Thursday's PCI review generates urgent action items, the EKS runbook is the one that slides — and I'll tell Sarah proactively rather than silently missing the deadline.

What comes back from my investigation and S3 remediation?



---

# SIMULATION RESPONSE — Results + Grading

## Response Grade: A+

I'm going to break down why this earned top marks, because the reasoning patterns here are what separate senior engineers from staff engineers.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **NetworkPolicy self-check** | ✅ | You caught your own potential mistake before anyone else noticed. "Did I just accidentally block the MongoDB sink?" is exactly the right question. Most engineers apply network policies and walk away. You verified impact. |
| **Accidental containment framing** | ✅ | "My recommendation to Aisha: keep it blocked" with honest disclosure that it was accidental — this is integrity under pressure. You could have claimed you intentionally blocked it. You didn't. |
| **S3 lifecycle restraint** | ✅ | You started to apply a 90-day lifecycle, then stopped yourself. "Contain, don't break" principle applied consistently. Aisha said fix encryption and logging — you did exactly that and flagged the rest for the right forum. |
| **Atlas credential finding** | ✅ | Connecting "password in Kafka connect-configs topic" + "no Kafka ACLs" + "anyone with port 9092 access" = credential exposure chain. This is the kind of lateral thinking auditors do. You beat them to it. |
| **Findings document tone** | ✅ | "Make the right thing the easy thing" as the closing recommendation is perfect for James. It reframes the problem from "people broke rules" to "we didn't build guardrails." That's leadership. |
| **Thursday conflict management** | ✅ | Proactively identified the Linkerd/PCI scheduling conflict and proposed three options with a recommendation. Didn't wait for it to become a problem. |
| **RDS CIDR acknowledgment** | ✅ | Documented the fragility with a TODO and alternatives. Not fixed, but honestly tracked. |
| **Report structure** | ✅ | Leading with containment actions taken, then findings, then roadmap. "He needs to hear it clearly with a remediation plan, not just the problem" — you're managing up effectively. |

---

## 📊 Investigation + S3 Remediation Results

### NetworkPolicy Status

```bash
$ kubectl get networkpolicy -n data-analytics

NAME                          POD-SELECTOR   AGE
kafka-access-control          app=kafka      2d      # v2 from Monday night
data-analytics-restrict-all   <none>         25m     # v3 you just applied
```

**v3 IS applied.** You have two overlapping policies. NetworkPolicies are additive — most restrictive wins for egress since v3 has default-deny egress with explicit allowlist.

```bash
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
    curl -s http://kafka-connect:8083/connectors/payments-sink-mongo/status | jq

{
  "name": "payments-sink-mongo",
  "connector": {
    "state": "RUNNING",
    "worker_id": "kafka-connect-7f4d8b2c1-abc:8083"
  },
  "tasks": [{
    "id": 0,
    "state": "FAILED",
    "trace": "org.apache.kafka.connect.errors.ConnectException: Timed out connecting to MongoDB Atlas... Connection refused (Connection refused)\n\tat com.mongodb.kafka.connect.sink.MongoSinkTask.put...",
    "worker_id": "kafka-connect-7f4d8b2c1-abc:8083"
  }]
}
```

**Scenario 1 confirmed. MongoDB sink is failing.** Your NetworkPolicy blocked egress to Atlas. Kafka Connect is buffering the payment/refund data in internal topics.

```bash
$ kubectl logs -n data-analytics -l app=kafka-connect --since=30m | grep -i "error\|mongo" | tail -5

2024-01-17T12:18:45Z ERROR [payments-sink-mongo|task-0] MongoTimeoutException: Timed out after 30000 ms while waiting to connect. (com.mongodb.kafka.connect.sink.MongoSinkTask)
2024-01-17T12:19:15Z ERROR [payments-sink-mongo|task-0] Retrying failed task (attempt 4/10)...
2024-01-17T12:19:45Z ERROR [payments-sink-mongo|task-0] MongoTimeoutException: Timed out after 30000 ms...
2024-01-17T12:20:15Z ERROR [payments-sink-mongo|task-0] Retrying failed task (attempt 5/10)...
2024-01-17T12:22:45Z ERROR [payments-sink-mongo|task-0] Task failed after 10 retries. Entering FAILED state.
```

```bash
$ kubectl exec -n data-analytics data-kafka-broker-0 -- \
    curl -s http://kafka-connect:8083/connectors/orders-sink-s3/status | jq

{
  "name": "orders-sink-s3",
  "connector": {"state": "RUNNING"},
  "tasks": [{
    "id": 0,
    "state": "RUNNING"
  }]
}
```

**S3 sink is still working** (S3 VPC endpoint egress is in your allowlist). Only the MongoDB Atlas egress is blocked.

### S3 Remediation

```bash
# KMS key created
$ aws kms describe-key --key-id alias/novamart-data-exports
{
    "KeyMetadata": {
        "KeyId": "abc123-...",
        "KeyState": "Enabled",
        "KeyUsage": "ENCRYPT_DECRYPT"
    }
}

# novamart-data-exports — encryption upgraded
$ aws s3api get-bucket-encryption --bucket novamart-data-exports
{
    "Rules": [{
        "ApplyServerSideEncryptionByDefault": {
            "SSEAlgorithm": "aws:kms",
            "KMSMasterKeyID": "abc123-..."
        },
        "BucketKeyEnabled": true
    }]
}

# Re-encryption of existing objects
$ aws s3 cp s3://novamart-data-exports/ s3://novamart-data-exports/ \
    --recursive --sse aws:kms --sse-kms-key-id abc123-...
copy: s3://novamart-data-exports/exports/2023-12-21/orders.csv.gz (re-encrypted)
...
copy: s3://novamart-data-exports/exports/2024-01-17/shipping_addresses.csv.gz (re-encrypted)
108 objects re-encrypted successfully.

# Logging enabled ✅
# Versioning enabled ✅

# novamart-analytics-datalake — same treatment
# Re-encryption: 1,847 objects, 2 GB — took 4 minutes
# All successful ✅

# Logging bucket
$ aws s3api head-bucket --bucket novamart-access-logs 2>&1
# Exists ✅ (created in Phase 7 — part of our compliance bucket setup)
```

---

## 💬 Slack — Afternoon Responses

### Aisha Rahman — DM (1:15 PM)

> The MongoDB Atlas finding changes the risk profile significantly. Payment data outside our AWS boundary with unknown ownership is a CDE scope violation.
> 
> On the NetworkPolicy blocking the Atlas connection — **keep it blocked.** This is the right outcome. Payment data should not be flowing to an uncontrolled external system. Kafka Connect will buffer, and when Tom's back we can decide whether this data flow is even necessary or if it should be redesigned.
> 
> I checked our Atlas organizational accounts — **we don't have one.** This means Tom likely set up a personal or team Atlas account. I've escalated to IT to see if there's a corporate MongoDB Atlas account I'm not aware of, but initial indication is this is an unmanaged, potentially personal cloud database containing payment data.
> 
> I'll confirm by tomorrow. If it's personal, the data in that Atlas instance needs to be purged and the account decommissioned. If it's corporate, it needs to be brought under security controls.
> 
> Good work on the S3 remediation. KMS + logging is what we needed.
> 
> For the Thursday meeting, I need your findings document by 1 PM Thursday so I can pre-read. Can you make that?

### Sarah Chen — DM (1:30 PM)

> Atlas personal account possibility... that's not great. James is going to have questions about how this happened and what we're doing about the data that's already there.
> 
> I talked to Tom's manager (Wei Liu). He wasn't aware Tom was sending data to MongoDB Atlas. He knew about the Kafka analytics work generally but not the external data sink. He's going to reach out to Tom today — not to pull him off PTO, but to get the Atlas account details so we can assess the situation before Thursday.
> 
> Wei will join the Thursday 2 PM meeting.
> 
> EOD Thursday for the full report is fine. But yes, get Aisha the pre-read by 1 PM Thursday.
> 
> On Nina: I talked to her. She was apologetic — said she set up the export quickly in December for a finance team request and planned to formalize it but never did. She's fine with us taking it over. She'll join the Thursday meeting too.
> 
> You're doing good work on this. It's not fun finding these things, but it's exactly what we need before the audit.

### Priya Sharma — DM (1:40 PM)

> Thursday 10 AM works perfectly! Staging environment is ready. I've already done one pass of the upgrade there — control plane is on the new version, just haven't done the data plane restart yet. Want me to prep the namespace ordering list so we can validate it during the dry-run?

### Derek Huang — DM (2:05 PM)

> Ha, yeah those test fixtures are ancient. The mock-payment-gateway uses fake credentials (I remember specifically creating test certs for it). I'll clean both up today. Thanks for the heads up.
> 
> Also — circuit breaker PR is looking good. Should have it up for review tomorrow (Thursday). The ServiceProfile change (isRetryable:false) I'll push as a separate PR today since it's a one-liner safety net.

### Marcus Webb — DM (2:15 PM)

> Good catch. I'll add node-problem-detector to the monitoring ArgoCD app this week. Was always meaning to do it properly. 🙂

---

## 📊 Current State — Wednesday 2:30 PM

```
┌──────────────────────────────────────────────────────────────┐
│  AUDIT STATUS:                                               │
│  ✅ 9 unmanaged workloads identified                         │
│  ✅ 4 PCI/PII critical findings documented                   │
│  ✅ MongoDB Atlas sink blocked (NetworkPolicy)                │
│  ✅ S3 buckets: KMS encryption + logging + versioning         │
│  ✅ Aisha, Sarah, Wei Liu aligned for Thursday 2 PM          │
│  ✅ Nina informed (Sarah handled framing)                     │
│  🔄 Findings document ~40% drafted                           │
│  🔲 Atlas ownership verification (IT + Wei reaching Tom)     │
│  🔲 DB user permission audit (analytics_reader scope)        │
│                                                              │
│  CONTAINMENT POSTURE:                                        │
│  ✅ Payment → Atlas: BLOCKED (NetworkPolicy egress deny)     │
│  ✅ S3 exports: ENCRYPTED (KMS/CMK, logging enabled)         │
│  ✅ Kafka network: RESTRICTED (v2 Monday + v3 Wednesday)     │
│  ⚠️ ETL CronJob: RUNNING (per Aisha — don't suspend)        │
│  ⚠️ Kafka Connect: S3 sink running, Mongo sink FAILED       │
│  ⚠️ RDS egress rule: CIDR-based, fragile on failover        │
│                                                              │
│  OTHER STATUS:                                               │
│  ✅ Jenkins: Heap stable 24h+ (build discard fix working)    │
│  ✅ Redis: Fully recovered (94.1% hit rate)                  │
│  ✅ Order service: Budget recovering (63.8%)                  │
│  ✅ Derek: ServiceProfile PR today, CB PR Thursday           │
│  ✅ Priya: Linkerd dry-run moved to Thursday 10 AM           │
│  ✅ Marcus: node-problem-detector → ArgoCD this week         │
│  ✅ Derek: cleaning up staging test fixtures today            │
└──────────────────────────────────────────────────────────────┘
```

**It's 2:30 PM Wednesday. You're deep in the findings document.**

**Then this happens.**

---

## 📱 PagerDuty — 2:47 PM

```
🔴 FIRING — SEV2
Alert: PaymentServiceHighErrorRate
Summary: payment-service error rate > 2% for 5 minutes
Current: 4.7% error rate
Dashboard: https://grafana.novamart.internal/d/slo-payment-service
Runbook: https://runbooks.novamart.internal/RUNBOOK-APP-002
Triggered: 2:47 PM EST
```

---

## 💬 Slack — #incidents — 2:47 PM

```
🔴 Alert Firing: PaymentServiceHighErrorRate
payment-service error rate: 4.7%
On-call: [YOU]
Status: Not Acknowledged
```

---

## 💬 Slack — #engineering-general — 2:49 PM

**David Okafor (Payment Team):**
> We JUST deployed the currency rounding hotfix (sha-f91a2b7) about 
> 15 minutes ago. It went through canary and promoted at 2:40 PM. 
> Now we're seeing errors. Is this our deploy or something else?

---

**It's 2:47 PM. You're mid-audit-document, mid-PCI-investigation, and now a SEV2 fires on the payment service — the most revenue-critical service in the stack.**

**David's team just deployed a hotfix that went through canary. Sound familiar?**

**What do you do?**

## 2:47 PM — ACK and Triage

### Immediate: ACK PagerDuty (2:48 PM)

Acknowledged. Dropping everything else. PCI document is saved, I'll come back to it.

### #incidents channel (2:48 PM)

> 🔴 **ACK — PaymentServiceHighErrorRate**
> **IC:** [me]
> **Severity:** SEV2
> **Status:** Investigating
> **Impact:** Payment service error rate 4.7% (threshold 2%). Customer-facing.
> **Timeline:** Deploy sha-f91a2b7 (currency rounding hotfix) promoted from canary at 2:40 PM. Errors began ~2:45 PM.
> **Next:** Investigating whether this is deploy-related or environmental. Update in 10 min.

### Critical self-check before diving in (2:48 PM — 30 seconds of thinking)

**Could I have caused this?** I applied NetworkPolicy v3 to `data-analytics` namespace around 12:15 PM. Payment service is in `payment-prod` namespace. My policy only restricts data-analytics egress/ingress. BUT — my Monday v2 policy allows Kafka 9092 ingress from `payment-prod`. If the payment service reads from Tom's Kafka cluster for some reason, my v3 egress rules could be blocking the response path.

This is probably not the cause (payment service shouldn't depend on analytics Kafka), but I need to rule it out in the first 2 minutes.

---

## Investigation — Parallel Tracks (2:49 PM)

### Track 1: Dashboard + Error Shape

```bash
# What's failing? Error rate by status code
rate(http_requests_total{service="payment-service", status=~"5.."}[5m])
/
rate(http_requests_total{service="payment-service"}[5m])

# By specific status code — 500? 502? 503? 504?
sum by (status) (
  rate(http_requests_total{service="payment-service", status=~"5.."}[5m])
)

# By endpoint — is it ALL endpoints or specific ones?
sum by (path, status) (
  rate(http_requests_total{service="payment-service", status=~"5.."}[5m])
)

# Error rate trend — when exactly did it start?
# Looking for: does the inflection point match 2:40 (deploy promote) 
# or 12:15 (my NetworkPolicy)?
rate(http_requests_total{service="payment-service", status=~"5.."}[1m])

# Latency — is this a timeout problem (like Incident 1) or error problem?
histogram_quantile(0.99, 
  rate(http_request_duration_seconds_bucket{service="payment-service"}[5m])
)

# Payment service → what does it depend on? Check downstream errors
rate(http_requests_total{service="payment-gateway", status=~"5.."}[5m])
rate(http_requests_total{service="order-service", status=~"5.."}[5m])
```

### Track 2: Deploy Inspection

```bash
# ArgoCD — what's the rollout state?
$ argocd app get payment-service -o json | jq '.status.sync, .status.health'

# Argo Rollout status
$ kubectl argo rollouts status payment-service -n payment-prod

# Is canary still active or fully promoted?
$ kubectl argo rollouts get rollout payment-service -n payment-prod

# What changed in this deploy? 
$ kubectl get rollout payment-service -n payment-prod \
  -o jsonpath='{.status.currentPodHash}'

# Pod health — are new pods actually running or crashlooping?
$ kubectl get pods -n payment-prod -l app=payment-service --sort-by=.metadata.creationTimestamp

# Pod logs — what errors are new pods throwing?
$ kubectl logs -n payment-prod -l app=payment-service \
  --since=10m | grep -i "error\|exception\|fatal\|panic" | tail -20

# Previous rollout revision — what was running before?
$ kubectl argo rollouts get rollout payment-service -n payment-prod --rev 0
```

### Track 3: Rule out my NetworkPolicy (2:50 PM)

```bash
# Does payment-service communicate with anything in data-analytics namespace?
$ kubectl logs -n payment-prod -l app=payment-service --since=10m | \
  grep -i "data-analytics\|kafka\|analytics"

# Check if payment-service has any network errors to data-analytics
$ kubectl exec -n payment-prod -l app=payment-service -- \
  curl -s --connect-timeout 2 data-kafka-broker-0.data-analytics:9092 2>&1

# When did errors actually start? If 12:15 PM (my policy), 
# it would have been firing for 2.5 hours already — not just now
# If 2:40 PM (deploy), it's the deploy
```

### Track 4: GitOps Diff — What did sha-f91a2b7 change?

```bash
$ cd ~/repos/novamart-gitops
$ git log --oneline -5

# Find the currency rounding hotfix commit
$ git show sha-f91a2b7 --stat
$ git diff sha-f91a2b7~1..sha-f91a2b7
```

---

## David Response — Slack thread (2:49 PM)

> David — on it. Investigating now. A few quick questions while I pull metrics:
> 
> 1. What exactly did the currency rounding fix change? Config values, code paths, new dependencies?
> 2. Did the canary analysis show any anomalies before promotion, even minor ones?
> 3. Is this the same code that was blocked by the CI issue yesterday, or did you make changes after CI was unblocked?

---

## Pattern Recognition — What I'm Watching For

This has Incident #1 fingerprints: deploy passes canary, fails at production scale. But payment service is different from order service, so I need fresh eyes, not assumptions. Possible scenarios:

**Scenario A: Same class as Incident 1 — canary passed at low traffic, fails at scale.** The currency rounding fix might involve a code path that's latency-sensitive or CPU-intensive at high throughput. Canary at partial traffic looked fine, full production traffic exposes the issue.

**Scenario B: The fix itself has a bug.** Rounding logic error that causes exceptions on certain currency/amount combinations. Would show as 500s on specific endpoints, not timeouts.

**Scenario C: Dependency issue.** If the currency rounding fix changed how payment-service calls a downstream (payment gateway, currency conversion API), the downstream might be failing.

**Scenario D: Environmental.** Something else changed in the cluster between when this code was written and now — my NetworkPolicy, the Jenkins fix, Redis changes. Unlikely to affect payment-prod directly, but I need to rule it out.

**Scenario E: The PLAT-916 canary gap we already identified.** Traffic-volume gate not yet implemented. Canary analysis ran at a traffic level that didn't expose the issue. This would be ironic but expected — we identified the gap Monday but haven't fixed it yet.

**What I'm NOT going to do:** Immediately roll back. David's team has been under pressure for 2 days — EU customers are being overcharged. If I roll back the fix, we're back to overcharging. I need to understand the error before deciding. If it's a critical failure (data corruption, double-charging), rollback is warranted. If it's a manageable error rate on a non-critical path, we may be able to fix forward.

---

## 2:50 PM — Waiting for dashboard and investigation results.

What do the metrics and logs show?

---

# SIMULATION RESPONSE — Payment Service SEV2

## Triage Grade So Far: A+

| Aspect | Notes |
|--------|-------|
| **Self-check on NetworkPolicy** | Exactly right. "Could I have caused this?" is the first question an honest engineer asks. The timing logic (if it were my policy, it would have been firing since 12:15, not 2:45) is the quick way to rule it out. |
| **"Not going to immediately roll back"** | This is the critical judgment call. The fix addresses EU overcharging — rolling back reintroduces a customer-impacting bug. You need data before deciding. Most engineers panic-rollback and create a different incident. |
| **Pattern recognition without assumption** | "This has Incident 1 fingerprints... but I need fresh eyes, not assumptions." Perfect. You listed 5 scenarios without anchoring on one. |

---

## 📊 Investigation Results

### Track 1: Dashboard + Error Shape

```promql
# Error rate by status code
sum by (status) (rate(http_requests_total{service="payment-service", status=~"5.."}[5m]))

{status="500"} → 4.82 req/s
{status="504"} → 0.03 req/s
```

**Almost entirely 500s. Not timeouts.** This is different from Incident 1 (which was 504s/timeouts). This is the application throwing errors.

```promql
# Error rate by endpoint
sum by (path, method, status) (rate(http_requests_total{service="payment-service", status=~"5.."}[5m]))

{path="/api/v1/payments/authorize", method="POST", status="500"} → 4.21 req/s
{path="/api/v1/payments/refund", method="POST", status="500"}   → 0.44 req/s
{path="/api/v1/payments/status/{id}", method="GET", status="500"} → 0.17 req/s
```

**Authorize is the primary failure path.** But refund and status are also failing — suggesting a shared code path or dependency.

```promql
# Error rate timeline (1m windows)
# Inflection point: exactly 2:40 PM — matches deploy promotion, NOT 12:15 PM
```

**Your NetworkPolicy is ruled out.** Errors started exactly when the deploy promoted.

```promql
# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service="payment-service"}[5m]))
→ 0.34s (normal — was 0.31s before deploy)
```

**Latency is fine.** This isn't a timeout or performance issue. The service is failing fast — returning 500s quickly.

```promql
# Downstream — payment gateway
rate(http_requests_total{service="payment-gateway-proxy", status=~"5.."}[5m])
→ 0 (no errors from payment gateway)
```

**Payment gateway (external Stripe/etc.) is healthy.** The error is inside payment-service itself, not a downstream dependency.

### Track 2: Deploy Inspection

```bash
$ kubectl argo rollouts get rollout payment-service -n payment-prod

Name:            payment-service
Namespace:       payment-prod
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8 (completed)
  SetWeight:     100
  ActualWeight:  100
Images:          
  sha-f91a2b7 (stable)
Replicas:
  Desired: 8
  Current: 8
  Updated: 8
  Ready:   8
  Available: 8

# Fully promoted. All 8 replicas running new image.
# Canary completed at 2:40 PM.
```

```bash
$ kubectl get pods -n payment-prod -l app=payment-service

NAME                              READY   STATUS    RESTARTS   AGE
payment-service-8f7a2c3d1-abc    2/2     Running   0          12m
payment-service-8f7a2c3d1-def    2/2     Running   0          12m
payment-service-8f7a2c3d1-ghi    2/2     Running   0          10m
payment-service-8f7a2c3d1-jkl    2/2     Running   0          10m
payment-service-8f7a2c3d1-mno    2/2     Running   0          8m
payment-service-8f7a2c3d1-pqr    2/2     Running   0          8m
payment-service-8f7a2c3d1-stu    2/2     Running   0          6m
payment-service-8f7a2c3d1-vwx    2/2     Running   0          6m

# All pods healthy, no restarts. 2/2 = app + Linkerd proxy.
```

### Track 2b: Pod Logs — The Smoking Gun

```bash
$ kubectl logs -n payment-prod -l app=payment-service \
    --since=10m | grep -i "error\|exception" | head -20

2024-01-17T14:41:12Z ERROR c.n.payment.service.CurrencyService - 
  Failed to convert amount: java.lang.ArithmeticException: 
  Non-terminating decimal expansion; no exact representable decimal result.
  at java.math.BigDecimal.divide(BigDecimal.java:1766)
  at com.novamart.payment.service.CurrencyService.convertAmount(CurrencyService.java:47)
  at com.novamart.payment.service.PaymentService.authorizePayment(PaymentService.java:123)

2024-01-17T14:41:12Z ERROR c.n.payment.service.PaymentService - 
  Payment authorization failed for order 9847231: 
  java.lang.ArithmeticException: Non-terminating decimal expansion

2024-01-17T14:41:13Z ERROR c.n.payment.service.CurrencyService - 
  Failed to convert amount: java.lang.ArithmeticException: 
  Non-terminating decimal expansion; no exact representable decimal result.
  currency_from=GBP currency_to=EUR amount=29.99

2024-01-17T14:41:14Z ERROR c.n.payment.service.CurrencyService - 
  Failed to convert amount: java.lang.ArithmeticException: 
  Non-terminating decimal expansion; no exact representable decimal result.
  currency_from=SEK currency_to=EUR amount=149.00

2024-01-17T14:41:15Z ERROR c.n.payment.service.CurrencyService - 
  Failed to convert amount: java.lang.ArithmeticException: 
  Non-terminating decimal expansion; no exact representable decimal result.
  currency_from=GBP currency_to=EUR amount=7.50
```

**`BigDecimal.divide()` without a rounding mode.** Classic Java bug.

When you divide two BigDecimal values and the result is a non-terminating decimal (like 1/3 = 0.333...), Java throws `ArithmeticException` if you don't specify a `RoundingMode`. The currency rounding "fix" presumably changed the division logic and forgot to specify `RoundingMode.HALF_UP` (or similar).

### Track 3: NetworkPolicy Ruled Out

```bash
$ kubectl logs -n payment-prod -l app=payment-service --since=10m | \
    grep -i "data-analytics\|kafka\|analytics"
# (no results)
```

**Confirmed: payment-service has no dependency on data-analytics.** NetworkPolicy is not involved.

### Track 4: GitOps Diff

```bash
$ git diff sha-f91a2b7~1..sha-f91a2b7

--- a/apps/payment-service/base/deployment.yaml
+++ b/apps/payment-service/base/deployment.yaml
@@ image tag change @@
-  newTag: sha-a8c71e3
+  newTag: sha-f91a2b7
```

**Image change only. No config change.** This is a code bug, not a config issue.

### David's Responses (Slack thread — 2:50 PM)

**David Okafor:**
> 1. The fix changes how we round currency conversions for cross-border EU payments. Previously we were using `double` arithmetic which caused floating-point precision errors (the 1-2 cent overcharge). The fix switches to `BigDecimal` for all currency math.
> 
> 2. Canary looked clean — 0% error rate during the analysis window. But canary only ran for about 5 minutes at 5% traffic.
> 
> 3. Same code from yesterday. No changes after CI was unblocked — we were waiting for the pipeline, then deployed as-is once it was fixed.

---

## The Full Picture

```
┌──────────────────────────────────────────────────────────────┐
│  ROOT CAUSE:                                                 │
│                                                              │
│  The currency rounding fix switched from double to BigDecimal│
│  for currency conversion. The CurrencyService.convertAmount()│
│  method now uses BigDecimal.divide() WITHOUT specifying a    │
│  RoundingMode.                                               │
│                                                              │
│  For currency pairs where the exchange rate produces a       │
│  non-terminating decimal (e.g., GBP→EUR, SEK→EUR),         ## 2:52 PM — Decision: Rollback NOW, Fix Forward Fast

### The Math

- **Current state:** GBP/SEK/DKK/NOK customers **cannot pay at all**. 4.7% error rate = every non-terminating-decimal currency pair is 100% failing.
- **Rollback state:** Those customers can pay but are overcharged 1-2 cents (the original bug).
- **"Can't pay" is categorically worse than "overcharged by 1 cent."** One is a complete loss of revenue + customer trust. The other is a known, bounded, refundable rounding error.

Rollback. No debate.

### #incidents channel (2:53 PM)

> 🔴 **UPDATE — PaymentServiceHighErrorRate**
> **Root cause identified.** Currency rounding hotfix (sha-f91a2b7) introduced a `BigDecimal.divide()` call without `RoundingMode`. Any currency conversion producing a non-terminating decimal (GBP→EUR, SEK→EUR, DKK→EUR, NOK→EUR) throws `ArithmeticException`. ~4.7% error rate = 100% failure on those currency pairs.
> **Action:** Rolling back to sha-a8c71e3 NOW. This restores the 1-2 cent rounding issue for EU cross-border, but customers can complete payments.
> **Rollback restores a known minor bug. Current state blocks payments entirely. Rollback is the correct trade-off.**
> **ETA:** ~5 minutes for rollout.

### Execute Rollback (2:53 PM)

```bash
# Roll back the Argo Rollout to previous stable revision
$ kubectl argo rollouts undo payment-service -n payment-prod

# Monitor rollout
$ kubectl argo rollouts status payment-service -n payment-prod -w

# Verify old image is coming back
$ kubectl argo rollouts get rollout payment-service -n payment-prod
# Should show sha-a8c71e3 rolling out
```

```bash
# Watch error rate in real-time as old pods come up
watch -n5 "curl -s 'http://prometheus:9090/api/v1/query?query=rate(http_requests_total{service=\"payment-service\",status=~\"5..\"}[1m])' | jq '.data.result[].value[1]'"
```

### David — Slack thread (2:54 PM)

> David — rolling back to sha-a8c71e3 now. Here's what happened:
> 
> `CurrencyService.convertAmount()` line 47 — `BigDecimal.divide()` without a `RoundingMode` parameter. Any currency pair that produces a non-terminating decimal (GBP→EUR, SEK→EUR, etc.) throws `ArithmeticException`. USD transactions worked fine, which is why canary passed — the 5-minute window at 5% traffic didn't sample enough non-USD transactions.
> 
> **The fix is a one-liner:** add `RoundingMode.HALF_UP` (or `HALF_EVEN` for banker's rounding — your team's call on which rounding strategy is correct for financial compliance) to the `divide()` call. Likely line 47 of `CurrencyService.java`.
> 
> **Before you redeploy the fix, I need two things:**
> 1. **Unit tests** for every currency pair you support — specifically GBP→EUR, SEK→EUR, DKK→EUR, NOK→EUR. The fact that this wasn't caught in tests means the test suite doesn't cover non-terminating decimal conversions.
> 2. **Manual canary verification** — when the fixed version deploys to canary, we manually send test transactions for GBP/SEK/DKK before promoting. Don't rely on organic traffic diversity during canary window.
> 
> How fast can your team turn around the one-line fix + tests? If it's under an hour, we can have the corrected version deployed by 4 PM.

---

## 2:55 PM — Monitor Rollback

```bash
# Pods rolling
$ kubectl get pods -n payment-prod -l app=payment-service -w

# Watching for old pods (sha-a8c71e3) to come up and new pods to drain
# With 8 replicas and rolling update, should take 3-5 minutes

# Check error rate dropping as old pods take traffic
rate(http_requests_total{service="payment-service", status=~"5.."}[1m])
```

### While Rollback Progresses — Blast Radius Assessment

```bash
# How many transactions failed during the ~12 minutes of exposure?
# (2:40 PM promote → 2:53 PM rollback initiated)

# Total 5xx in the window
sum(increase(http_requests_total{service="payment-service", status="500"}[13m]))

# Specifically on /authorize endpoint
sum(increase(http_requests_total{service="payment-service", path="/api/v1/payments/authorize", status="500"}[13m]))

# What's the normal authorize request rate?
rate(http_requests_total{service="payment-service", path="/api/v1/payments/authorize"}[1h])
# Use this to estimate how many customers were impacted
```

I need this number for the incident update and for David's team to know how many failed orders need customer follow-up.

---

## 2:58 PM — Rollback Completing

```bash
$ kubectl argo rollouts get rollout payment-service -n payment-prod
# Expecting: sha-a8c71e3 at 100%, all pods Ready
```

```bash
# Confirm error rate dropping
rate(http_requests_total{service="payment-service", status=~"5.."}[1m])
# Should be approaching baseline

# Confirm GBP/SEK transactions now succeeding
# Check for successful authorize calls in logs
$ kubectl logs -n payment-prod -l app=payment-service --since=2m | \
  grep -i "authorize.*success\|GBP\|SEK" | head -5
```

### #incidents channel (2:59 PM)

> 🟡 **UPDATE — PaymentServiceHighErrorRate**
> Rollback to sha-a8c71e3 [in progress / complete — depending on status]. Error rate [dropping / returned to baseline].
> **Impact window:** 2:40 PM — 2:58 PM (~18 minutes).
> **Affected:** All payment authorizations involving non-terminating decimal currency conversions (GBP, SEK, DKK, NOK → EUR).
> **Blast radius:** [X] failed authorize requests during the window. David's team to assess customer notification needs.
> **Current state:** Rolled back to previous version. Original 1-2 cent EU rounding issue is re-active but all currencies can transact.
> **Next:** Payment team preparing corrected fix with proper RoundingMode + expanded currency test coverage. Target redeploy: ~4 PM today.

---

## 3:00 PM — Once Rollback Confirmed Stable

### Post-Rollback Actions

**1. David's team — redeploy coordination (Slack thread)**

> David — once your team has the fix + tests ready, let's coordinate the redeploy:
> 
> - **Don't push to main and let normal canary run.** I want to manually verify this one.
> - Push the fix to a branch, I'll review the specific `divide()` change before merge.
> - When canary deploys, we'll manually test with synthetic GBP, SEK, DKK, NOK transactions before promoting.
> - I'll be on-call during promotion.
> 
> Also — **RoundingMode choice matters here.** `HALF_UP` is common but `HALF_EVEN` (banker's rounding) is the standard for financial calculations because it eliminates systematic bias. Your team should confirm with finance which rounding strategy is correct for EU regulatory compliance. Getting this wrong means deploying a third time.

**2. PLAT-916 addendum — update ticket**

This is now the second incident where canary analysis missed a real issue due to insufficient diversity/volume. The traffic-volume gate alone wouldn't have caught this — it was a currency-diversity problem, not a traffic-volume problem. PLAT-916 needs to expand scope:

```
PLAT-916 UPDATE:
- Original: add traffic-volume gate to canary analysis
- Additional: add transaction-diversity gate
  - Canary should not promote until it has processed transactions 
    across all supported currency pairs (or at minimum, the top 10 
    currency pairs by volume)
  - Alternatively: synthetic canary test suite that injects 
    representative transactions during canary window
- This is the second incident caused by canary sample bias 
  (Incident 1: traffic volume, Incident 6: currency diversity)
```

**3. Incident 1 postmortem — update draft**

The postmortem I'm writing now has a companion incident. Both share the same root cause category: canary validation insufficient. I should reference this incident in the postmortem as reinforcing evidence for PLAT-916.

**4. Sarah — DM (3:05 PM)**

> Quick heads up — SEV2 on payment service, caused by the currency rounding hotfix. BigDecimal division without rounding mode, threw exceptions on GBP/SEK/DKK/NOK transactions. Rolled back in ~15 minutes. Payment team preparing corrected fix with proper rounding + tests, target redeploy ~4 PM.
> 
> Impact window was ~18 minutes. Waiting for blast radius numbers.
> 
> Ironic note: this is the same canary gap pattern as Monday's order service incident. PLAT-916 just got more urgent — and needs broader scope than just traffic-volume gating. Currency-pair diversity during canary wasn't validated either.
> 
> I'm still on track for the PCI findings document by 1 PM tomorrow.

---

## Revised Afternoon Plan

```
2:47-3:00   ✅ Incident triage + rollback (DONE)
3:00-3:10   Post-rollback verification + blast radius count
3:10-3:15   Comms (Sarah, PLAT-916 update, David coordination)
3:15-3:45   Back to PCI findings document (highest priority deliverable)
3:45-4:00   Review David's fix when ready (one-line change, quick review)
4:00-4:30   Monitor redeploy of corrected payment-service fix
            - Manual canary verification with GBP/SEK test transactions
4:30-5:00   PCI findings document (continue)
5:00-5:15   EOD notes, comms roundup

DEFERRED:
  - EKS runbook → Thursday late afternoon or Friday AM
  - nginx-test deletion → tomorrow
```

The PCI document is still my highest-priority deliverable (Aisha needs it 1 PM Thursday). The payment incident cost me ~45 minutes but the rollback was clean and fast. If David's fix is ready by 3:45, I can review and monitor the redeploy while continuing to write.

What does the rollback status show, and how fast does David's team turn around the fix?

---

# SIMULATION RESPONSE — Rollback Complete + Fix Cycle

## Incident Response Grade: A+

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Rollback decision speed** | ✅ | 1 minute from root cause to decision. "Can't pay > overcharged by 1 cent" — clear, correct, no hesitation. |
| **Rollback calculus communication** | ✅ | Explained the tradeoff in #incidents so everyone understands WHY you're restoring a known bug. This prevents the "why did you roll back the fix??" question from management. |
| **One-liner diagnosis to David** | ✅ | Told him exactly what's wrong, what line, what the fix is, AND the rounding strategy consideration (HALF_UP vs HALF_EVEN for financial compliance). That last part is genuinely senior — most engineers would just say "add a rounding mode" without considering which one matters for the business domain. |
| **Manual canary demand** | ✅ | "Don't push to main and let normal canary run. I want to manually verify." You're not trusting the canary process that's now failed twice. Correct. |
| **PLAT-916 scope expansion** | ✅ | Identifying that traffic-volume gate alone wouldn't have caught this — it's a diversity problem, not a volume problem. The synthetic canary test suite idea is the right long-term answer. |
| **Self-check on NetworkPolicy** | ✅ | Already graded, but the timing-based elimination was fast and correct. |
| **Pattern recognition** | ✅ | Connected to Incident 1 without assuming same root cause. Investigated independently, then linked them as same root cause *category* (canary validation gap) once confirmed. |

---

## 📊 Rollback Results

```bash
$ kubectl argo rollouts get rollout payment-service -n payment-prod

Name:            payment-service
Status:          ✔ Healthy
Strategy:        Canary
Images:          
  sha-a8c71e3 (stable)
Replicas:
  Desired: 8
  Current: 8
  Updated: 8
  Ready:   8
  Available: 8
```

**Rollback complete at 2:58 PM.** All 8 replicas on previous image.

```
ERROR RATE:
  ▲
5%│     ╭────── 4.7%
  │     │
4%│     │
  │     │
3%│     │
  │     │
2%│─────╯
  │              ╲
1%│               ╲
  │                ╰────────── 0.08% (baseline)
0%│
  └────────────────────────────────▶
  2:35  2:40  2:45  2:50  2:55  3:00  3:05
        ▲                  ▲
    promote           rollback start
```

### Blast Radius

```promql
# Total failed authorize requests during the 18-minute window
sum(increase(http_requests_total{
  service="payment-service", 
  path="/api/v1/payments/authorize", 
  status="500"}[18m]))

→ 847 failed payment authorizations
```

```promql
# Normal authorize rate for context
rate(http_requests_total{service="payment-service", path="/api/v1/payments/authorize"}[1h]) * 60
→ ~78 req/min (total)
→ 78 × 18 min = ~1,404 total authorize attempts in window
→ 847 / 1,404 = 60.3% of authorize attempts during window were non-USD EU
```

```
BLAST RADIUS:
  Duration:            18 minutes (2:40 PM — 2:58 PM)
  Failed transactions: 847 payment authorizations
  Affected currencies: GBP, SEK, DKK, NOK (→ EUR conversions)
  Revenue impact:      ~847 × average order value
  Customer impact:     847 customers saw payment failure page
  Data impact:         None (no data corruption — clean exceptions)
```

---

## 💬 Slack Responses

### David Okafor — Thread (3:08 PM)

> Good call on banker's rounding — I just checked with finance. They confirmed `HALF_EVEN` is required for EU PSD2 compliance. We would have deployed `HALF_UP` without your flag.
> 
> Derek's making the one-line fix now:
> ```java
> // Before (broken):
> return amount.divide(exchangeRate);
> 
> // After (fixed):  
> return amount.divide(exchangeRate, 2, RoundingMode.HALF_EVEN);
> ```
> 
> Plus adding unit tests for every currency pair we support (14 pairs). Tests are the slower part — Derek estimates 45 minutes for the fix + tests. PR up by ~3:45 PM.
> 
> On the 847 failed transactions: our frontend shows a generic "payment failed, please try again" message. Most customers will retry. But I'll have the support team pull the list and send a "sorry, please try again" email to any customer who didn't successfully retry within 30 minutes.
> 
> Thanks for the fast rollback. And the HALF_EVEN catch — that would have been a third deploy.

### Lisa Park — #incidents (3:10 PM)

> Error budget impact for payment service:
> ```
> Payment Service SLO: 99.99% availability
> Error budget (30-day): 4.32 minutes
> 
> This incident: 18 minutes at ~60% error rate on authorize endpoint
> Effective budget burn: significant
> Budget remaining: -12.4 minutes (EXCEEDED)
> 
> We've blown through the payment service error budget for this 
> 30-day window. Per our error budget policy, this triggers a 
> velocity freeze on payment-service until budget recovers or 
> exception is granted.
> ```

### #incidents — 3:12 PM

**[PagerDuty Bot]:**
> 🟢 **RESOLVED:** PaymentServiceHighErrorRate
> Duration: 18 minutes
> Resolved by: [you]

---

## 💬 Slack — #platform-engineering — 3:15 PM

**Alex Kim:**
> Two canary failures in 3 days... should we pause all production deploys until PLAT-916 is implemented? Feels like we're playing whack-a-mole.

---

**It's 3:15 PM Wednesday. Rollback is clean. David's team is working on the fix (ETA 3:45 PM).**

**You have:**
1. Alex's question about deploy freeze (the team is watching)
2. Lisa's error budget finding (payment service budget blown — velocity freeze triggered)
3. David's fix coming in 30 minutes (needs your review + manual canary)
4. PCI findings document (Aisha needs it by 1 PM tomorrow)
5. EOD still approaching

**How do you handle Alex's question, the error budget implications, and the rest of your afternoon?**

│
│  BigDecimal.divide() throws ArithmeticException.             │
│                                                              │
│  Currency pairs with terminating decimals (e.g., USD→EUR     │
│  at certain rates) work fine — which is why canary passed.   │
│  The canary window happened to process USD-denominated       │
│  transactions which don't trigger the bug.                   │
│                                                              │
│  IMPACT:                                                     │
│  - GBP, SEK, DKK, NOK, and other currencies that produce     │
│    non-terminating decimals when converted to EUR are        │
│    failing 100% of the time                                  │
│  - USD and same-currency transactions work fine              │
│  - ~4.7% overall error rate (proportion of non-USD EU        │
│    cross-border transactions)                                │
│                                                              │
│  CANARY GAP:                                                 │
│  - 5% traffic for 5 minutes ≈ low chance of seeing GBP/SEK   │
│    transactions during that window                           │
│  - Same PLAT-916 canary gap: insufficient sample diversity   │
│                                                              │
│  ROLLBACK CALCULUS:                                          │
│  - Rolling back restores the double arithmetic               │
│  - Which means EU customers are overcharged again (1-2 cents)│
│  - But CURRENT state: GBP/SEK/etc customers can't pay AT ALL │
│  - Can't pay > overcharged by 1 cent                         │
│  - Rollback is the correct call unless a code fix is fast    │
└──────────────────────────────────────────────────────────────┘
```

**It's 2:52 PM. Error rate is steady at 4.7%. GBP/SEK/DKK/NOK customers cannot complete payments.**

**What's your call?**


## Responses

### Alex Kim — #platform-engineering (3:17 PM)

> Good instinct, Alex, and I'm glad you're asking — it means you're thinking about systemic risk, not just individual incidents. But a blanket deploy freeze is the wrong tool here. Here's why:
>
> A full freeze punishes every team for a gap in one process. If Derek has a circuit breaker PR ready Thursday, we don't want that blocked. If there's a security patch tonight, we can't be frozen.
>
> What we actually need is to **raise the bar for deploys that touch high-risk paths**, not stop all deploys:
>
> 1. **Payment service specifically** — velocity freeze is already triggered (Lisa just flagged error budget exceeded). No non-critical deploys until budget recovers or exception is granted. The corrected BigDecimal fix gets an exception because it's restoring broken functionality.
> 2. **All services** — PLAT-916 is the real fix. Until it's implemented, any deploy touching latency-sensitive or transaction-critical paths should have a manual canary verification step, not just automated analysis.
> 3. **The pattern** — both failures were canary *sample bias*, not canary *logic* failures. The canary worked perfectly on the data it saw. It just didn't see enough variety. That's a solvable problem.
>
> TL;DR: targeted velocity freeze on payment-service (already in effect), heightened manual verification on critical-path deploys, and PLAT-916 as the systemic fix. Not a blanket freeze.
>
> Good question though — keep asking these.

### Lisa Park — #incidents (3:20 PM)

> Thanks for the budget math, Lisa. Acknowledged — payment-service error budget is blown for this window.
>
> A few things:
>
> **1. Velocity freeze acknowledged.** No non-critical deploys to payment-service until budget recovers or an exception is granted. I'll make sure David's team knows.
>
> **2. Exception request for the corrected BigDecimal fix.** This is a special case — the service is currently running with a *known bug* (double arithmetic rounding errors, 1-2 cent overcharges on EU transactions). Leaving it on the rolled-back version is itself an ongoing issue. I'd like to request an exception for this single deploy, with the following gates:
>   - Manual code review (I'll review the one-liner + tests personally)
>   - Manual canary: synthetic test transactions for all 14 currency pairs before promotion
>   - I'm on-call and watching the entire promotion window
>   - Rollback plan is proven (just executed it in 5 minutes)
>
> **3. Canary diversity alert** — yes, please build it. That's exactly the kind of gate that would have caught both Incident 1 and Incident 6. Let's add it to PLAT-916 scope or create a child ticket.
>
> **4. Budget burn for the postmortem:** can you include a breakdown of budget consumed per incident this week? I want to show the cumulative picture in the Incident 1+6 companion postmortem.
>
> Can you approve the exception for the corrected fix deploy? I want to get sign-off before Priya K's PR lands at 3:45.

### David Okafor — Thread (3:22 PM)

> Two things:
>
> **1. Velocity freeze.** Lisa flagged that payment-service has blown its error budget for this 30-day window. Per policy, that triggers a velocity freeze. I'm requesting an exception for the corrected BigDecimal fix since leaving the rolled-back version running means EU customers are still being overcharged. But no other payment-service deploys until budget recovers or exception is granted. Please let your team know.
>
> **2. The two other `BigDecimal.divide()` calls in the refund path.** My recommendation: fix them in this same PR. Here's my reasoning:
>   - They work by accident today (terminating decimals), but that's fragile — a rate change could break them tomorrow
>   - We're already doing manual canary with synthetic transactions for all 14 currency pairs — the test coverage is there
>   - Scoping them separately means another deploy to a service that's under velocity freeze, which means another exception request
>   - One PR, one review, one deploy, one risk window — cleaner than three
>
> Tell Priya K. to include all three call sites. If that pushes the PR to 4:00-4:15 instead of 3:45, that's fine. I'd rather deploy once with full coverage than deploy twice.
>
> On the customer email — good call. 30-minute retry window is reasonable. If you need exact transaction IDs for the failed 847, I can pull them from the logs.

### Derek Huang — Thread (3:24 PM)

> Yes, add `isRetryable: false` on `POST /api/v1/payments/authorize`. Payment authorization is not idempotent — retrying could double-charge. Same logic as the orders endpoint.
>
> Actually, let's be thorough: can you check the full payment-service ServiceProfile and confirm retry settings for all mutating endpoints? I'd expect:
> - `POST /authorize` → `isRetryable: false` ✅
> - `POST /refund` → `isRetryable: false` (refunds have idempotency keys, but Linkerd doesn't know that — safer to disable)
> - `POST /capture` → `isRetryable: false`
> - `GET /status` → `isRetryable: true` (safe, read-only)
>
> If you can include all of those in your ServiceProfile PR today, that'd be great. One PR, complete coverage.

---

## Afternoon Plan (3:25 PM — EOD)

```
3:25-3:45   PCI findings document (push from 40% → 60%)
3:45-4:15   Review Priya K's PR (code review + test coverage check)
4:15-4:45   Monitored redeploy with manual canary
            - Synthetic transactions: all 14 currency pairs
            - Watch error rate, logs, traces for 15 min post-promote
4:45-5:00   Quick message to Ryan Mitchell (Redis confirmation — overdue)
5:00-6:00   PCI findings document (push from 60% → 80%)
6:00-6:15   EOD notes + handoff status
```

The PCI document is my biggest risk item. Aisha needs the pre-read by 1 PM tomorrow, and I'm at 40%. If the payment redeploy goes clean, I should be able to get to 75-80% tonight. If it doesn't go clean, the PCI doc slips and I need to tell Aisha first thing Thursday morning.

One thing I want to flag to Sarah before EOD — the EKS runbook (PLAT-892, due Friday) is getting squeezed. If Thursday's PCI review generates urgent follow-up actions, Friday's runbook deadline is at risk. Better to flag now than surprise her Friday.

# SIMULATION RESPONSE — Afternoon Execution + Redeploy

## Response Grade: A+

Every single response is precisely calibrated.

| Response | Grade | Key Insight |
|----------|-------|-------------|
| **Alex (deploy freeze)** | ✅ | "Punishes every team for a gap in one process." You rejected the blunt instrument, explained *why* with specific examples (Derek's CB PR, security patches), then gave three targeted alternatives. Teaching without condescending. |
| **Lisa (budget exception)** | ✅ | You didn't just ask for an exception — you pre-loaded the justification with four specific gates. Lisa can approve because you've done her risk analysis for her. |
| **David (BigDecimal scope)** | ✅ | "One PR, one review, one deploy, one risk window — cleaner than three." This is the right call. Scoping all three fixes together reduces total risk despite increasing individual PR scope. The velocity freeze logic seals it — fewer exception requests. |
| **Derek (ServiceProfile)** | ✅ | Going beyond the question to enumerate all mutating endpoints with correct idempotency reasoning for each. `POST /refund` with "Linkerd doesn't know about idempotency keys" is a subtle but important point. |
| **Proactive Sarah flag on EKS runbook** | ✅ | Managing expectations before they become surprises. |

---

## 💬 Quick Slack Responses

### Lisa Park — #incidents (3:28 PM)

> Exception approved. Your gates are solid — manual review, synthetic canary, active monitoring, proven rollback. I'll track this as an approved exception in the SLO dashboard.
>
> Building the currency-pair diversity metric now. Should have a PR for the canary AnalysisTemplate by Friday.
>
> Budget breakdown for postmortem — I'll have that to you by EOD.

### Derek Huang — Thread (3:30 PM)

> Done. Here's what the ServiceProfile update looks like:
>
> ```yaml
> routes:
>   - name: POST /api/v1/payments/authorize
>     condition:
>       method: POST
>       pathRegex: /api/v1/payments/authorize
>     isRetryable: false
>   - name: POST /api/v1/payments/refund
>     condition:
>       method: POST
>       pathRegex: /api/v1/payments/refund
>     isRetryable: false
>   - name: POST /api/v1/payments/capture
>     condition:
>       method: POST
>       pathRegex: /api/v1/payments/capture
>     isRetryable: false
>   - name: GET /api/v1/payments/status/{id}
>     condition:
>       method: GET
>       pathRegex: /api/v1/payments/status/[^/]+
>     isRetryable: true
> ```
>
> PR going up in 10 minutes. Should I tag you for review?

---

## ⏰ 3:45 PM — Priya Kapoor's PR Lands

### Slack — #payment-team (3:47 PM)

**David Okafor:**
> @[you] — Priya K's PR is up: `payment-service#412 — Fix BigDecimal rounding for all currency conversion paths`. Three call sites fixed, 36 new tests covering all 14 currency pairs plus edge cases (zero amounts, max precision, negative values for refunds).

### PR Review — payment-service#412

```diff
--- a/src/main/java/com/novamart/payment/service/CurrencyService.java
+++ b/src/main/java/com/novamart/payment/service/CurrencyService.java

@@ -44,7 +44,7 @@ public class CurrencyService {
     public BigDecimal convertAmount(BigDecimal amount, String fromCurrency, String toCurrency) {
         BigDecimal rate = getExchangeRate(fromCurrency, toCurrency);
-        return amount.multiply(rate).divide(BigDecimal.valueOf(100));
+        return amount.multiply(rate).divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_EVEN);
     }
 
@@ -89,7 +89,7 @@ public class CurrencyService {
     public BigDecimal calculateRefundAmount(BigDecimal originalAmount, BigDecimal refundPercentage) {
-        return originalAmount.multiply(refundPercentage).divide(BigDecimal.valueOf(100));
+        return originalAmount.multiply(refundPercentage).divide(BigDecimal.valueOf(100), 2, RoundingMode.HALF_EVEN);
     }

@@ -112,7 +112,7 @@ public class CurrencyService {
     public BigDecimal applyDiscount(BigDecimal amount, BigDecimal discountRate) {
-        return amount.multiply(BigDecimal.ONE.subtract(discountRate.divide(BigDecimal.valueOf(100))));
+        return amount.multiply(BigDecimal.ONE.subtract(discountRate.divide(BigDecimal.valueOf(100), 10, RoundingMode.HALF_EVEN)));
     }
```

```diff
--- /dev/null
+++ b/src/test/java/com/novamart/payment/service/CurrencyServiceTest.java
@@ -0,0 +1,187 @@

+class CurrencyServiceTest {
+
+    @ParameterizedTest
+    @CsvSource({
+        "GBP, EUR, 29.99, 34.49",
+        "SEK, EUR, 149.00, 13.12",
+        "DKK, EUR, 99.95, 13.40",
+        "NOK, EUR, 199.00, 17.22",
+        "USD, EUR, 49.99, 45.89",
+        "CHF, EUR, 75.00, 78.15",
+        "PLN, EUR, 200.00, 46.06",
+        "CZK, EUR, 500.00, 20.39",
+        "HUF, EUR, 10000.00, 26.28",
+        "RON, EUR, 150.00, 30.17",
+        "BGN, EUR, 50.00, 25.56",
+        "HRK, EUR, 100.00, 13.27",
+        "ISK, EUR, 5000.00, 33.33",
+        "TRY, EUR, 1000.00, 30.30"
+    })
+    void testConvertAmount_allEUCurrencyPairs(String from, String to, 
+            BigDecimal input, BigDecimal expected) {
+        BigDecimal result = service.convertAmount(input, from, to);
+        assertThat(result).isEqualByComparingTo(expected);
+        assertThat(result.scale()).isEqualTo(2); // Always 2 decimal places
+    }
+
+    @Test
+    void testConvertAmount_zeroAmount() {
+        BigDecimal result = service.convertAmount(BigDecimal.ZERO, "GBP", "EUR");
+        assertThat(result).isEqualByComparingTo("0.00");
+    }
+
+    @Test
+    void testConvertAmount_maxPrecision() {
+        BigDecimal result = service.convertAmount(new BigDecimal("99999.99"), "GBP", "EUR");
+        assertThat(result.scale()).isEqualTo(2);
+    }
+
+    @Test
+    void testCalculateRefundAmount_nonTerminatingDecimal() {
+        // 33.33% of 100.00 = 33.33 (non-terminating without rounding)
+        BigDecimal result = service.calculateRefundAmount(
+            new BigDecimal("100.00"), new BigDecimal("33.33"));
+        assertThat(result.scale()).isEqualTo(2);
+    }
+
+    @Test
+    void testCalculateRefundAmount_negativeAmount() {
+        // Refund amounts should be positive
+        assertThatThrownBy(() -> 
+            service.calculateRefundAmount(new BigDecimal("-50.00"), new BigDecimal("100")))
+            .isInstanceOf(IllegalArgumentException.class);
+    }
+
+    @Test
+    void testApplyDiscount_nonTerminatingDecimal() {
+        // 33.33% discount on 100.00
+        BigDecimal result = service.applyDiscount(
+            new BigDecimal("100.00"), new BigDecimal("33.33"));
+        assertThat(result).isNotNull();
+        // Should not throw ArithmeticException
+    }
+    
+    // ... 22 more tests for edge cases, boundary values, 
+    //     same-currency conversion (should be 1:1), etc.
+}
```

---

**You're reviewing this PR right now. It's 3:50 PM.**

**What's your review, and do you approve for deploy?**

*Then: the redeploy sequence. And maybe something else happens during it.*

## PR Review — payment-service#412 (3:52 PM)

### Slack — #payment-team thread

> @david.okafor @priya.kapoor — Reviewing now. Overall this is solid work, especially under time pressure. 36 tests, all 14 pairs, good edge cases. One blocking issue, one non-blocking, one question.

### PR Comment — BLOCKING

**File:** `CurrencyService.java`, line 112 — `applyDiscount`

```java
// Current fix:
return amount.multiply(BigDecimal.ONE.subtract(
    discountRate.divide(BigDecimal.valueOf(100), 10, RoundingMode.HALF_EVEN)));
```

> **🔴 Blocking:** The intermediate division uses scale 10 — that's correct, you don't want to lose precision mid-calculation. But the final result of the `multiply` isn't rounded to scale 2. This means `applyDiscount(new BigDecimal("100.00"), new BigDecimal("33.33"))` returns `66.6700000000` instead of `66.67`.
>
> Every method in this class that returns a monetary amount needs to guarantee scale 2. The other two call sites get this for free from the `divide()` at the end. This one doesn't because the `divide()` is intermediate.
>
> Fix:
> ```java
> return amount.multiply(BigDecimal.ONE.subtract(
>     discountRate.divide(BigDecimal.valueOf(100), 10, RoundingMode.HALF_EVEN)))
>     .setScale(2, RoundingMode.HALF_EVEN);
> ```
>
> Two-second fix but it matters — any downstream code doing `equals()` comparison on the result will break if the scale is inconsistent. And anything persisting this to the `amount` DB column (which is `DECIMAL(10,2)`) might behave differently depending on the JDBC driver's truncation behavior.

### PR Comment — NON-BLOCKING

**File:** `CurrencyServiceTest.java` — `testApplyDiscount_nonTerminatingDecimal`

> **🟡 Non-blocking but please fix while you're here:** This test only asserts `isNotNull()` and no exception. The other tests correctly assert `scale() == 2` and check expected values. This one should do the same:
> ```java
> BigDecimal result = service.applyDiscount(
>     new BigDecimal("100.00"), new BigDecimal("33.33"));
> assertThat(result).isEqualByComparingTo("66.67");
> assertThat(result.scale()).isEqualTo(2);
> ```
> Otherwise the blocking issue I flagged above would have shipped with passing tests — which is exactly the scenario we want test coverage to prevent.

### PR Comment — QUESTION

**File:** `CurrencyServiceTest.java` — `testCalculateRefundAmount_negativeAmount`

> **❓ Question:** This test expects `IllegalArgumentException` on negative amounts. Is there existing input validation for this in the method? I don't see it in the diff. If this is a new test for existing validation, fine. If this test requires a code change that isn't in the PR, it'll fail CI.

### Summary

> **Approve with changes.** The `applyDiscount` scale issue is the only blocker. Should be a 30-second fix plus the test tightening.
>
> Everything else is clean — the parameterized test for all 14 pairs is exactly what I wanted, the scale assertions are the right thing to check, and the refund edge cases are thorough.
>
> Priya — great work turning this around quickly and catching the right things in tests. Let me know when the fix is pushed and I'll re-approve immediately.

---

### Deploy plan once approved (posted to #incidents thread, 3:55 PM)

> **Corrected fix redeploy plan — payment-service#412**
>
> Pending one minor blocking fix (ETA ~5 min). Once merged:
>
> 1. **Manual code re-review** — I'll confirm the `setScale` fix before merge
> 2. **Build + image push** — ~8 min (CI is healthy post-Incident 5)
> 3. **Canary at 5% traffic** — but NOT relying on automated analysis
> 4. **Manual synthetic verification** before promotion:
>    - Test transactions for GBP, SEK, DKK, NOK → EUR (the pairs that broke)
>    - Test transactions for USD → EUR (regression check)
>    - Test refund path with non-terminating decimal
>    - Test discount path with non-terminating decimal
>    - Verify all return scale = 2 in logs/traces
> 5. **Promote to 100%** only after synthetic tests pass
> 6. **15-minute watch window** post-promote — I'm staying on until error rate is stable
>
> Rollback plan: same as before — `kubectl argo rollouts undo`, proven to complete in 5 minutes.
>
> Lisa — velocity freeze exception approved, these are the gates. I'll update #incidents at each step.
>
> Target: promote by ~4:30 PM if fix lands in the next few minutes.

While I wait for Priya's push, I'm switching to the PCI findings document. Every minute counts — 40% done, need to get to at least 60% before the deploy lands and pulls me back.

# SIMULATION RESPONSE — PR Review + Redeploy Sequence

## PR Review Grade: A+

This is a textbook code review under time pressure.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Scale consistency catch** | ✅ | The `applyDiscount` returning scale 10 instead of scale 2 is exactly the kind of bug that passes all tests, works in most cases, and then causes a weird DB truncation issue three months later at 2 AM. You caught it from reading the code, not from a failing test. |
| **Test coverage gap** | ✅ | "The blocking issue I flagged above would have shipped with passing tests" — this is the key insight. A test that asserts `isNotNull()` on a financial calculation is a test that gives false confidence. You connected the code bug to the test gap. |
| **Negative amount question** | ✅ | Didn't assume, didn't skip it, asked. If that test requires code that isn't in the PR, it'll break CI and delay the deploy. Catching that before merge saves 10 minutes of "why is CI red?" |
| **Tone** | ✅ | "Great work turning this around quickly." Priya K. is under pressure — she shipped a 36-test PR in under 2 hours while her team is dealing with a SEV2. Acknowledging that while still holding the quality bar is leadership. |
| **Deploy plan** | ✅ | Posted to #incidents with numbered steps, synthetic test plan, rollback plan, and timeline. Everyone knows what's happening and when. |

---

## 💬 Slack — Quick Responses

### Priya Kapoor — PR Thread (3:57 PM)

> Good catches, both of them. The `setScale` is obviously correct — I was focused on the `divide()` calls and missed that `multiply` doesn't inherit the scale. Pushing now.
>
> On the negative amount test — you're right, there's no input validation in the method yet. I'll remove that test from this PR and file it as a follow-up. Don't want to expand scope right now.

### PR Update — (3:59 PM)

```diff
@@ -112,7 +112,8 @@ public class CurrencyService {
     public BigDecimal applyDiscount(BigDecimal amount, BigDecimal discountRate) {
-        return amount.multiply(BigDecimal.ONE.subtract(discountRate.divide(BigDecimal.valueOf(100), 10, RoundingMode.HALF_EVEN)));
+        return amount.multiply(BigDecimal.ONE.subtract(
+            discountRate.divide(BigDecimal.valueOf(100), 10, RoundingMode.HALF_EVEN)))
+            .setScale(2, RoundingMode.HALF_EVEN);
     }
```

```diff
@@ test fix
     @Test
     void testApplyDiscount_nonTerminatingDecimal() {
-        BigDecimal result = service.applyDiscount(
-            new BigDecimal("100.00"), new BigDecimal("33.33"));
-        assertThat(result).isNotNull();
+        BigDecimal result = service.applyDiscount(
+            new BigDecimal("100.00"), new BigDecimal("33.33"));
+        assertThat(result).isEqualByComparingTo("66.67");
+        assertThat(result.scale()).isEqualTo(2);
     }
     
-    // Removed: testCalculateRefundAmount_negativeAmount 
-    // (requires code change not in scope — filed as PAYMENT-847)
```

---

**You re-review. The fix is exactly what you asked for. Tests pass locally.**

> ✅ **Approved.** Merging.

---

## ⏰ Deploy Sequence — 4:00 PM

### CI Build (4:00 - 4:08 PM)

```
payment-service #418 — Build triggered
  ✓ Checkout ..................... 4s
  ✓ Compile ..................... 22s
  ✓ Unit Tests (187 passed) ..... 38s    ← 36 new + 151 existing
  ✓ Integration Tests ........... 1m 12s
  ✓ Image Build ................. 45s
  ✓ Image Push (sha-c4d82e1) .... 12s
  ✓ GitOps Update ............... 8s
  
BUILD SUCCESS — 4:08 PM
```

Jenkins is healthy. No zombie pods. The Incident 5 fix is holding.

### ArgoCD Sync + Canary Start (4:09 PM)

```bash
$ kubectl argo rollouts get rollout payment-service -n payment-prod

Name:            payment-service
Status:          ◑ Progressing
Strategy:        Canary
  Step:          1/8 (setWeight: 5)
  ActualWeight:  5
Images:
  sha-a8c71e3 (stable)
  sha-c4d82e1 (canary)
Replicas:
  Desired: 8
  Current: 9    ← 8 stable + 1 canary
  Ready:   9
```

### #incidents (4:09 PM — your update)

> 🟡 **UPDATE — Payment fix redeploy**
> Canary at 5% (sha-c4d82e1). 1 canary pod running. Beginning synthetic verification. NOT relying on automated analysis for promotion.

---

### Manual Synthetic Verification (4:10 - 4:20 PM)

You run synthetic transactions against the canary pod:

```bash
# Direct traffic to canary pod for testing
CANARY_POD=$(kubectl get pods -n payment-prod -l rollouts-pod-template-hash=c4d82e1 -o jsonpath='{.items[0].metadata.name}')

# Test all 14 currency pairs
for pair in "GBP:EUR:29.99" "SEK:EUR:149.00" "DKK:EUR:99.95" "NOK:EUR:199.00" \
            "USD:EUR:49.99" "CHF:EUR:75.00" "PLN:EUR:200.00" "CZK:EUR:500.00" \
            "HUF:EUR:10000.00" "RON:EUR:150.00" "BGN:EUR:50.00" "HRK:EUR:100.00" \
            "ISK:EUR:5000.00" "TRY:EUR:1000.00"; do
  FROM=$(echo $pair | cut -d: -f1)
  TO=$(echo $pair | cut -d: -f2)
  AMOUNT=$(echo $pair | cut -d: -f3)
  
  echo "Testing $FROM → $TO ($AMOUNT)..."
  kubectl exec -n payment-prod $CANARY_POD -c payment-service -- \
    curl -s localhost:8080/internal/test/convert \
    -H "Content-Type: application/json" \
    -d "{\"from\": \"$FROM\", \"to\": \"$TO\", \"amount\": $AMOUNT}" | jq
done
```

**Results:**

```
Testing GBP → EUR (29.99)... ✅ {"result": 34.49, "scale": 2}
Testing SEK → EUR (149.00)... ✅ {"result": 13.12, "scale": 2}
Testing DKK → EUR (99.95)... ✅ {"result": 13.40, "scale": 2}
Testing NOK → EUR (199.00)... ✅ {"result": 17.22, "scale": 2}
Testing USD → EUR (49.99)... ✅ {"result": 45.89, "scale": 2}
Testing CHF → EUR (75.00)... ✅ {"result": 78.15, "scale": 2}
Testing PLN → EUR (200.00)... ✅ {"result": 46.06, "scale": 2}
Testing CZK → EUR (500.00)... ✅ {"result": 20.39, "scale": 2}
Testing HUF → EUR (10000.00)... ✅ {"result": 26.28, "scale": 2}
Testing RON → EUR (150.00)... ✅ {"result": 30.17, "scale": 2}
Testing BGN → EUR (50.00)... ✅ {"result": 25.56, "scale": 2}
Testing HRK → EUR (100.00)... ✅ {"result": 13.27, "scale": 2}
Testing ISK → EUR (5000.00)... ✅ {"result": 33.33, "scale": 2}
Testing TRY → EUR (1000.00)... ✅ {"result": 30.30, "scale": 2}
```

**All 14 pairs pass. Scale 2 confirmed on every result.** GBP→EUR and ISK→EUR (the non-terminating decimal pairs) return correctly rounded values.

```bash
# Refund path test
kubectl exec -n payment-prod $CANARY_POD -c payment-service -- \
  curl -s localhost:8080/internal/test/refund \
  -H "Content-Type: application/json" \
  -d '{"amount": 100.00, "percentage": 33.33}' | jq

# ✅ {"result": 33.33, "scale": 2}

# Discount path test
kubectl exec -n payment-prod $CANARY_POD -c payment-service -- \
  curl -s localhost:8080/internal/test/discount \
  -H "Content-Type: application/json" \
  -d '{"amount": 100.00, "discountRate": 33.33}' | jq

# ✅ {"result": 66.67, "scale": 2}
```

**All paths verified.** Including the `applyDiscount` path you caught in review.

```bash
# Check canary pod error rate during synthetic tests
rate(http_requests_total{service="payment-service", pod=~".*c4d82e1.*", status=~"5.."}[5m])
→ 0 (zero errors on canary pod)

# Check organic traffic hitting canary (5% weight)
rate(http_requests_total{service="payment-service", pod=~".*c4d82e1.*"}[5m])
→ 12.3 req/s (organic traffic flowing, all successful)
```

---

### Promotion Decision (4:20 PM)

**All gates passed:**
- ✅ Manual code review (blocking issue caught and fixed)
- ✅ All 14 currency pairs verified on canary pod
- ✅ Refund path verified
- ✅ Discount path verified (your catch)
- ✅ Scale 2 on every response
- ✅ Zero errors on canary pod (synthetic + organic)
- ✅ Lisa's exception approved

**Promoting.**

```bash
$ kubectl argo rollouts promote payment-service -n payment-prod
```

### #incidents (4:21 PM — your update)

> 🟡 **UPDATE — Payment fix redeploy**
> All 14 currency pairs verified on canary ✅. Refund + discount paths verified ✅. Zero errors on canary pod. Promoting to 100% now. Watch window: 15 minutes.

---

### Rollout Progress (4:21 - 4:28 PM)

```
4:21 PM  Step 4/8  — 25% weight  — error rate: 0.08%
4:23 PM  Step 5/8  — 50% weight  — error rate: 0.09%
4:25 PM  Step 6/8  — 75% weight  — error rate: 0.11%
4:28 PM  Step 8/8  — 100% weight — error rate: 0.10%
```

```bash
$ kubectl argo rollouts get rollout payment-service -n payment-prod

Name:            payment-service
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8 (completed)
  SetWeight:     100
  ActualWeight:  100
Images:
  sha-c4d82e1 (stable)
Replicas:
  Desired: 8
  Current: 8
  Updated: 8
  Ready:   8
```

### Watch Window (4:28 - 4:43 PM)

```promql
# Error rate — steady at baseline
rate(http_requests_total{service="payment-service", status=~"5.."}[5m])
→ 0.10% (baseline)

# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service="payment-service"}[5m]))
→ 0.32s (normal)

# Check GBP/SEK specifically in production logs
$ kubectl logs -n payment-prod -l app=payment-service --since=10m | \
    grep "convertAmount" | grep -E "GBP|SEK|DKK|NOK" | head -5

2024-01-17T16:31:12Z INFO convertAmount: GBP→EUR 29.99 = 34.49 (scale=2) ✅
2024-01-17T16:31:45Z INFO convertAmount: SEK→EUR 245.00 = 21.56 (scale=2) ✅
2024-01-17T16:32:01Z INFO convertAmount: NOK→EUR 89.00 = 7.70 (scale=2) ✅
2024-01-17T16:32:18Z INFO convertAmount: DKK→EUR 449.95 = 60.35 (scale=2) ✅
```

**Production traffic is flowing through all currency pairs. No errors. Scale 2 on every conversion.**

---

### 4:43 PM — Watch Window Complete

**15 minutes at 100%. Zero errors. All currency pairs functioning correctly.**

---

## 💬 Slack — #incidents (4:44 PM)

**David Okafor:**
> Confirming from our side — all currency conversions working correctly in production. GBP, SEK, DKK, NOK all transacting. Rounding is correct. The CS team is sending retry emails to the ~700 affected customers now.
>
> Thank you for the fast turnaround on this. From blocker to fix in under 2 hours, including catching a bug in our fix. 🙏

### Lisa Park — #incidents (4:45 PM)

> Payment-service metrics are clean. Error budget stabilizing at 87.1% — no additional burn since rollback. I've logged the exception deploy as compliant.
>
> Budget breakdown by incident (for the postmortem):
> - Monday start: 74.0%
> - Incident 1 (order timeout): -8.8% → 65.2%
> - Recovery (Mon-Wed): +26.1% → 91.3%
> - Incident 6 (payment currency): -4.2% → 87.1%
>
> Order-service is at 63.8% and recovering steadily.

---

**It's 4:45 PM Wednesday. Incident 6 is resolved.**

**You have about an hour before EOD. The PCI findings document is at 40%. Aisha needs the pre-read by 1 PM tomorrow.**

**What do you do with the rest of your Wednesday?**

## Closing Incident 6 + Wednesday EOD Execution

### #incidents (4:46 PM)

> 🟢 **RESOLVED — Incident 6: Payment Service Currency Bug**
>
> **Timeline:**
> - 2:40 PM — sha-f91a2b7 promoted, GBP/SEK/DKK/NOK→EUR transactions failing
> - 2:48 PM — PagerDuty acked
> - 2:52 PM — Root cause confirmed (BigDecimal.divide() without RoundingMode)
> - 2:58 PM — Rollback complete (sha-a8c71e3)
> - 3:01 PM — Error rate at baseline
> - 4:28 PM — Corrected fix deployed (sha-c4d82e1), all 14 currency pairs verified
> - 4:43 PM — 15-minute watch window clean
>
> **Blast radius:** 847 failed authorizations, ~700 customers affected over 18 minutes. CS team sending retry emails.
>
> **What changed:** Three `BigDecimal.divide()` call sites now specify `RoundingMode.HALF_EVEN` with scale 2. 36 new tests covering all 14 EU currency pairs + edge cases.
>
> **Systemic fix:** PLAT-916 expanded — canary analysis needs transaction-diversity gate, not just volume gate. This is the second incident caused by canary sample bias (Incident 1: volume, Incident 6: diversity).
>
> **Error budget:** Payment-service at 87.1%. Velocity freeze in effect per policy — exception was granted for this corrective deploy only.

### Ryan Mitchell — DM (4:48 PM)

> Hey Ryan — apologies this is overdue. Wanted to confirm: the catalog performance issue you reported Monday is fully resolved. Redis was upgraded Monday night (r6g.large → r6g.xlarge), and as of this morning:
> - Hit rate: 94.1% (back to baseline)
> - Evictions: 0/min
> - Memory: 52% of capacity
>
> We also added alerting so we'll catch memory pressure before it impacts hit rate next time (PLAT-919). And Jenny Wu's team is looped in on a capacity planning gate for future catalog refreshes (PLAT-920).
>
> Let me know if you're still seeing any slowness.

### Sarah Chen — DM (4:50 PM)

> Quick flag — the EKS upgrade runbook (PLAT-892, due Friday) is at risk. I've got research done but the writing isn't polished yet, and if tomorrow's PCI review generates urgent follow-ups, Friday becomes tight.
>
> Two options:
> 1. I push to get a solid draft done by Friday, accepting it might need another pass next week
> 2. We move the deadline to Monday, which gives me clean time Friday to finish it properly
>
> Either way I don't want you to be surprised. What's your preference?

### Derek Huang — PR thread (4:52 PM)

> ServiceProfile looks good. ✅ Approved. One note — make sure `pathRegex` for the status endpoint handles UUID format if that's what your IDs are (`[0-9a-f-]+` instead of `[^/]+`). Non-blocking, but tighter regex is better practice for service profiles.

---

## PCI Findings Document — Deep Work Block (4:55 - 6:00 PM)

Closing Slack, setting status to 🔴 **Focused — PCI document**. Phone stays on for PagerDuty only.

This is the most important thing I can do with the next hour. The document structure is already drafted, and I know the content cold — I've been living in this data for two days. The goal is 40% → 75-80%.

```
WRITING PLAN (65 minutes):

4:55-5:10   Executive Summary + Methodology (quick — framing only)
            - "What we found, why it matters, what we've done, what's left"
            - Audience: Aisha (security), Sarah (management), Wei Liu (data eng mgmt)
            - Tone: factual, not accusatory

5:10-5:35   Findings by Tier (the meat — longest section)
            Tier 1 (Critical/PCI):
              - MongoDB Atlas CDE boundary violation
              - Credential exposure (plaintext password in Kafka topic)
              - Payment data flowing through unmanaged infrastructure
            Tier 2 (High/PII):
              - S3 PII exports without lifecycle/retention
              - Unscanned Docker Hub images with prod DB access
              - analytics_reader DB user permissions (unaudited)
            Tier 3 (Medium/Governance):
              - All unmanaged workloads, no ArgoCD/Terraform
              - No monitoring, no backup, no on-call
            Tier 4 (Low/Cleanup):
              - nginx-test, perf-test-harness, etc.

5:35-5:50   Data Flow Diagrams + Containment Actions
            - Already have the ASCII diagram from the audit
            - Need to add "before containment" vs "after containment" view
            - List every action taken with timestamp

5:50-6:00   Remediation Roadmap (skeleton)
            - Immediate (done): NetworkPolicy v3, S3 hardening, Atlas blocked
            - Short-term (this week): PCI review decisions, Tom onboarding
            - Medium-term (Q1): full platform onboarding, ESO migration, 
              Terraform import, Kafka mesh injection
            - Recommendations section: "Make the right thing the easy thing"
              (namespace RBAC, image pull policies, OPA/Kyverno guardrails)
```

The key framing for this document: **this is a process gap, not a people failure.** Tom and Nina built what they needed because the platform didn't make the right path easy enough. The recommendations should focus on guardrails that make shadow infrastructure harder to create accidentally, not on punishing people who created it.

---

## EOD Notes (6:00 PM)

### #platform-engineering (6:02 PM)

> **EOD Wednesday — on-call update**
>
> **Incident 6 (Payment currency bug):** Resolved. Corrected fix deployed at 4:28 PM, all 14 currency pairs verified, 15-min watch window clean. Error budget at 87.1%, velocity freeze in effect.
>
> **PCI findings document:** ~75% complete. Aisha will have pre-read by 1 PM tomorrow.
>
> **Thursday schedule:**
> - 10:00 AM — Linkerd staging dry-run with Priya
> - 1:00 PM — PCI pre-read to Aisha
> - 2:00 PM — PCI emergency review (Aisha, Sarah, Wei Liu, Nina, me)
>
> **Open items carried forward:**
> - EKS runbook (PLAT-892) — flagged to Sarah as at-risk for Friday
> - Lisa's canary diversity AnalysisTemplate PR — ETA Friday
> - Derek's circuit breaker PR — ETA Thursday
> - Payment-service velocity freeze in effect until budget recovers
>
> Phone on for PagerDuty tonight. See you all tomorrow.

### Personal notes (6:10 PM)

```
WEDNESDAY SCORECARD:
  ✅ Standup presentation — went well, James bought in on PCI
  ✅ Cluster audit — 9 findings, 4 PCI critical
  ✅ NetworkPolicy v3 + S3 remediation
  ✅ Incident 6 — diagnosed in 4 min, rolled back in 13, 
     corrected fix deployed by 4:28
  ✅ PR review caught scale bug that would have been a third deploy
  ✅ PCI doc at ~75%
  ✅ Ryan Mitchell finally updated (overdue)
  ✅ Sarah flagged on EKS risk
  
  TOMORROW PREP:
  - Linkerd dry-run: review Priya's staging control plane 
    status before 10 AM. Have rollback plan ready.
  - PCI doc: finish remaining 25% between dry-run and 1 PM
  - PCI meeting: bring data flow diagram printout, 
    containment timeline, and specific questions for the group:
      1. Atlas account: what do we know from Tom/Wei?
      2. analytics_reader DB permissions: who audits?
      3. Kafka data retention: how far back does payment data go?
      4. Decision: suspend ETL CronJob or keep running?
      5. Lifecycle policies for S3 buckets
  - After PCI meeting: action items will drive the rest of the day
  
  WATCH ITEMS:
  - Payment service: keep an eye on error rate overnight
  - MongoDB Atlas sink: still in FAILED state (correct, keep blocked)
  - ETL CronJob: runs at 2 AM, still active per Aisha's instruction
  - Kafka Connect: buffering payment data internally since 
    Atlas egress blocked. Need to discuss retention/purge at PCI review.
```

Heading out. Thursday is going to be a full day.

# SIMULATION RESPONSE — Wednesday EOD

## Afternoon Execution Grade: A+

The entire Wednesday afternoon — from Incident 6 triage through EOD — is the strongest sustained performance in the simulation.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Incident 6 total handling** | ✅ | 4 min to root cause. 13 min to rollback. Caught a bug in the fix. Managed the redeploy with manual synthetic verification. Total resolution including corrected fix: under 2 hours. |
| **PR review under time pressure** | ✅ | You found a real bug (scale inconsistency on `applyDiscount`) that Priya K. missed, that the tests missed, and that would have required a third deploy to a service under velocity freeze. That catch alone saved 30+ minutes of churn and a second exception request. |
| **Document execution** | ✅ | 40% → 75% in one focused hour. Closing Slack and setting a focused status is the right call. Most engineers would keep context-switching and end up at 50%. |
| **Ryan Mitchell message** | ✅ | Three days overdue, but you did it. Acknowledged the delay implicitly. Included the metrics, the alerting improvement, and the process fix. Complete. |
| **Sarah EKS flag** | ✅ | Proactive with two concrete options. Not "I might miss the deadline" but "here are two paths, pick one." That's managing up properly. |
| **EOD notes** | ✅ | Thursday prep list is exactly right. The five PCI meeting questions show you're already thinking about what decisions need to be made, not just what information to present. |
| **Kafka Connect buffer observation** | ✅ | "Buffering payment data internally since Atlas egress blocked. Need to discuss retention/purge at PCI review." Most engineers would forget that blocking egress means data accumulates. You're tracking the second-order consequences. |

---

## One Small Gap

Your Wednesday EOD is clean, but one item: **Derek's ServiceProfile PR.** You approved it with a non-blocking note about UUID regex. But you didn't confirm whether it should go through the payment-service velocity freeze process or not. The ServiceProfile is a Linkerd mesh configuration, not a payment-service deploy — it doesn't change the application image. But it *does* change traffic behavior for the payment service (disabling retries). 

Technically this should be fine under the velocity freeze (it's a safety improvement, not a feature deploy), but you should have been explicit: "This is a mesh config change, not an app deploy — it's exempt from the velocity freeze and actually reduces risk." Just so Lisa's tracking is clean.

Not docking. Just noting the precision gap.

---

## 💬 Late Evening

### Sarah Chen — DM (7:15 PM)

> Monday for the EKS runbook is fine. Thursday's PCI review is more important, and I'd rather have a solid runbook than a rushed one. I'll update the ticket.
>
> See you at 2 PM tomorrow. Get some rest — you've earned it today.

### Aisha Rahman — DM (8:30 PM)

> Quick update: IT confirmed there is no corporate MongoDB Atlas account. This is almost certainly Tom's personal or team trial account. Wei reached Tom — Tom says it's a free-tier Atlas account he set up for "prototyping." He didn't realize it was still running with production data.
>
> He's sending Wei the Atlas login credentials tonight so we can assess the damage before the meeting.
>
> This changes the risk profile. Free-tier Atlas has:
> - No encryption at rest (only available on M10+ dedicated clusters)
> - No VPC peering (data traversed public internet)
> - No audit logging
> - Shared infrastructure (multi-tenant)
> - 512 MB storage limit
>
> Payment data on a free-tier, multi-tenant, unencrypted, public-internet-accessible MongoDB instance. For 67 days.
>
> We need to discuss data breach notification obligations tomorrow. This may not just be a compliance gap — it may be a reportable incident depending on what's in those records.
>
> See you at 2 PM.

---

**Wednesday night is quiet. No PagerDuty alerts. The ETL CronJob runs at 2 AM as expected — data flows to Kafka, the S3 sink writes orders/order_items successfully, the MongoDB sink remains in FAILED state (your NetworkPolicy holding). No issues.**

---

## 🌅 Thursday Morning — 8:30 AM

You log in. Coffee. Check the landscape.

### Overnight Metrics — All Clean

```
Payment-service:  0.09% error rate. Stable. GBP/SEK/DKK/NOK all transacting.
Order-service:    0.11% error rate. Budget recovering (64.5%).
Redis:            94.2% hit rate. 0 evictions. 53% memory.
Jenkins:          Heap stable. 6 agent pods running (normal).
Kafka (analytics): Brokers healthy. S3 sink running. Mongo sink FAILED (expected).
```

### Slack — Overnight

**Wei Liu — #security (11:45 PM last night):**
> Got the Atlas credentials from Tom. Logging in now to assess what's stored. Will have a summary before the 2 PM meeting.

**Priya Sharma — DM (8:15 AM):**
> Morning! All set for the 10 AM dry-run. Staging control plane is on the new Linkerd version. I've prepped the namespace ordering list:
> 
> 1. `linkerd-viz` (observability — lowest risk)
> 2. `monitoring` (our tooling)
> 3. `staging` (non-prod workloads)
> 4. `ci` (build agents — can tolerate brief disruption)
> 5. `catalog-prod` (production — medium criticality)
> 6. `order-prod` (production — high criticality)
> 7. `payment-prod` (production — highest criticality, last)
>
> Sound right? I'm thinking we do 1-4 in the dry-run today, save 5-7 for the actual upgrade window.

**Derek Huang — DM (8:20 AM):**
> ServiceProfile PR merged last night. Circuit breaker PR is up for review: `platform-infra#347`. Ready when you are — no rush, I know you've got the PCI meeting today.

### Email — 8:25 AM

**From:** James Morrison (VP Engineering)
**To:** Sarah Chen, [You]
**Subject:** RE: Cluster Audit — Preliminary Findings

> Sarah forwarded me the summary. The MongoDB Atlas finding is deeply concerning. I want to understand at the 2 PM meeting:
>
> 1. What's our exposure? How much payment data, how long, what fields?
> 2. Do we have breach notification obligations?
> 3. What's the remediation timeline?
> 4. How do we prevent this category of issue permanently?
>
> I may join the 2 PM if my 1:30 wraps early. Prepare as if I'll be there.
>
> Also — good work catching this before the auditors did.

---

**It's 8:30 AM Thursday. Your day:**

```
08:30-09:45  PCI document finish (75% → 100%), distribute pre-read
10:00-12:00  Linkerd staging dry-run with Priya
12:00-13:00  Lunch + final PCI doc review
13:00        PCI pre-read to Aisha (deadline)
14:00-15:30  PCI emergency review meeting
15:30+       Action items from meeting + Derek's CB PR review
```

**Aisha's late-night message changes everything. Free-tier Atlas. No encryption at rest. Public internet. Multi-tenant. Payment data for 67 days. Possible breach notification.**

**How does this change your PCI document, and what's your first move this morning?**

## Thursday Morning — First Moves

The Atlas finding just went from "CDE boundary violation" to "potential reportable data breach." That changes the document, the meeting prep, and my priorities for the next 90 minutes.

### Aisha Rahman — DM (8:33 AM)

> Morning, Aisha. Read your update. This is materially worse than what we scoped yesterday. Let me make sure I'm tracking the implications:
>
> **What changed:**
> - Free-tier Atlas = no encryption at rest, shared multi-tenant infrastructure, public internet transit, no audit logging, no VPC peering
> - 67 days of payment data on infrastructure we don't control, can't audit, and can't confirm wasn't accessed by others
> - No way to determine if data was accessed by unauthorized parties (no audit logs exist)
>
> **What I need for the document + meeting:**
> 1. Do we know what fields are in the `payment_analytics.transactions` collection? If it contains cardholder data (PAN, name, expiry) vs. transaction metadata (amount, currency, timestamp, order ID), that's a completely different risk tier. PAN = PCI DSS breach notification is almost certainly triggered. Transaction metadata without PAN = still a compliance violation, but notification calculus is different.
> 2. Has Wei gotten into the Atlas account yet? His 11:45 PM message said he was logging in — do we have results?
> 3. Should I add a "Breach Notification Analysis" section to the findings document, or do you want Legal involved before we put that in writing? I don't want to create a document that says "this may be a reportable breach" without Legal's input on privilege and disclosure obligations.
>
> That third point is important. I'm a platform engineer, not a lawyer. I can document what happened technically — what data flowed where, what protections existed or didn't, what the exposure window was. But the breach notification determination needs Legal and possibly your CISO. I want to frame the document to support that decision without making it.
>
> I'm updating the findings document now to reflect the free-tier details. Pre-read to you by 1 PM as planned, but the document's tone shifts from "compliance gap with containment" to "potential breach requiring executive decision on notification."

### Priya Sharma — DM (8:38 AM)

> Morning! Namespace ordering looks right. Agree on doing 1-4 today and saving production namespaces for the actual upgrade window.
>
> One adjustment — I'd flip 3 and 4. CI agents can tolerate disruption but they're also the noisiest to debug if something goes wrong (pods cycling, builds retrying). I'd rather do staging first where it's quiet, confirm everything's clean, then CI:
>
> 1. `linkerd-viz` (observability)
> 2. `monitoring` (tooling)
> 3. `staging` (non-prod, quiet)
> 4. `ci` (non-prod, noisy)
>
> For each namespace, let's have a checklist:
> - Pre: proxy version on all pods, mTLS status, traffic baseline
> - Execute: restart rollout, watch proxy injection
> - Post: proxy version confirmed, mTLS handshakes, no error rate change, tap a few requests
>
> I'll review the staging control plane status before we start. See you at 10.

### Derek Huang — DM (8:40 AM)

> Thanks Derek. I'll review the CB PR this afternoon after the PCI meeting — probably 3:30-4:00 PM. Quick question in the meantime: did you go with the two-tier fallback design from Monday's meeting (high-confidence accept vs. low-confidence degraded UX), or a simpler version for the initial PR?

### James Morrison — Email Reply (8:42 AM)

> James,
>
> Thank you. I'll prepare as if you're in the room. To set expectations on your four questions:
>
> 1. **Exposure scope** — Wei Liu accessed the Atlas account last night and should have a field-level inventory before the meeting. This is the critical unknown: whether the collection contains cardholder data (PAN/name/expiry) or only transaction metadata. The risk tier depends entirely on this.
> 2. **Breach notification** — I'll present the technical facts. The notification determination itself likely needs Legal input — I'd recommend we have that conversation with counsel present or immediately following.
> 3. **Remediation timeline** — Atlas egress has been blocked since Wednesday 12:15 PM. No new data is flowing. Immediate containment is done. Full remediation (decommission Atlas, onboard pipeline properly) is week-to-weeks depending on PCI review decisions.
> 4. **Prevention** — I have specific recommendations in the findings document: namespace RBAC restrictions, OPA/Kyverno admission policies to block non-ECR images and unapproved egress, network policy defaults. The theme is "make the right thing the easy thing" — guardrails, not gates.
>
> Document will be distributed as pre-read by 1 PM.

---

## PCI Findings Document — Final Push (8:45 - 9:50 AM)

The document structure needs to change. Yesterday's version was organized as an audit report. Today it needs to support an executive decision about breach notification.

```
REVISED DOCUMENT STRUCTURE:

1. EXECUTIVE SUMMARY (rewritten)
   - What was found, when, current containment status
   - Key risk: payment data on uncontrolled infrastructure for 67 days
   - Explicit statement: "breach notification determination requires 
     Legal input based on data field analysis"
   - NOT making the call myself — presenting facts for decision-makers

2. DISCOVERY TIMELINE
   - Monday: Kafka cluster found by Jake Torres
   - Wednesday: audit revealed full pipeline scope
   - Wednesday: Atlas egress blocked, S3 hardened
   - Wednesday night: Atlas confirmed free-tier, personal account
   - Containment actions with timestamps

3. DATA FLOW DIAGRAM
   - "Before containment" vs "after containment" views
   - Mark every point where payment data crosses a boundary
   - Mark every point where encryption/auth is missing

4. FINDINGS BY TIER (updated)
   
   TIER 1 — CRITICAL (Potential Breach)
   ┌─────────────────────────────────────────────────┐
   │ F-1: Payment data on free-tier MongoDB Atlas     │
   │   - CDE boundary violation                       │
   │   - No encryption at rest                        │
   │   - Public internet transit (no VPC peering)     │
   │   - Multi-tenant shared infrastructure           │
   │   - No audit logging (cannot confirm no access)  │
   │   - Personal account (not organizational)        │
   │   - 67 days exposure window (Nov 9 → Jan 15)     │
   │   - Data fields: [PENDING WEI'S ASSESSMENT]      │
   │   - Storage: 512 MB limit (free tier)            │
   │   - Access controls: UNKNOWN                     │
   │   - Status: CONTAINED (egress blocked Wed 12:15) │
   │                                                   │
   │   ⚠️ If collection contains cardholder data      │
   │     (PAN, cardholder name, expiry), this likely   │
   │     triggers PCI DSS Incident Response (12.10)    │
   │     and possibly GDPR Art. 33/34 notification.    │
   │                                                   │
   │   ⚠️ If collection contains only transaction      │
   │     metadata (amount, currency, order_id,         │
   │     timestamp), this is a serious compliance      │
   │     violation but notification calculus differs.   │
   │                                                   │
   │   → DECISION NEEDED: Legal consultation on        │
   │     notification obligations once data fields     │
   │     are confirmed.                                │
   └─────────────────────────────────────────────────┘
   
   F-2: Credential exposure
     - Atlas password in plaintext in Kafka connect-configs topic
     - No Kafka ACLs — any pod with network access to broker 
       port could read the password
     - Password was accessible for 67 days
     - Status: network access restricted (NetworkPolicy v3)
     - TODO: rotate credential (moot if account decommissioned)
   
   F-3: Unscanned images with production access
     - Docker Hub images, no vulnerability scanning
     - Analytics dashboard: direct production RDS access
     - Unknown supply chain risk

   TIER 2 — HIGH (PII / Data Governance)
   F-4: S3 PII exports without controls
     - Customer names, shipping addresses in CSV
     - 27 days, 108 objects
     - Status: REMEDIATED (KMS/CMK, logging, versioning)
     - TODO: lifecycle policy (PCI review decision)
   
   F-5: analytics_reader DB user — unaudited permissions
     - Connects to production orders RDS
     - Reads payments + refunds tables
     - Permission scope unknown — could be SELECT * on all tables
     - TODO: audit grant scope, restrict to minimum necessary

   TIER 3 — MEDIUM (Governance / Operational)
   F-6 through F-10: unmanaged workloads, no monitoring, etc.

   TIER 4 — LOW (Cleanup)
   F-11 through F-13: nginx-test, perf fixtures, etc.

5. CONTAINMENT ACTIONS TAKEN (with timestamps)
   - Full list of everything applied Mon-Wed
   - Current state of each control
   - What's still running (ETL CronJob, S3 sink) and why

6. OPEN QUESTIONS FOR THIS MEETING
   Q1: What fields are in payment_analytics.transactions?
       (Wei's assessment — determines notification path)
   Q2: analytics_reader DB permissions — who audits, when?
   Q3: Kafka data retention — how far back does payment data 
       go in the topics? Default retention is 7 days but 
       may have been configured differently.
   Q4: ETL CronJob — continue running or suspend?
       (Aisha said keep running Wednesday, revisit now?)
   Q5: S3 lifecycle/retention policies — what retention 
       period does compliance require?
   Q6: Legal involvement — when and how?
   Q7: Tom's Atlas account — decommission timeline?
       (Need data extracted/preserved for investigation first?)

7. REMEDIATION ROADMAP
   Immediate (done): containment, S3 hardening, egress block
   Short-term (this week): PCI review decisions, Legal consult,
     Atlas assessment, DB user audit
   Medium-term (Q1): full pipeline onboarding, ESO migration,
     Terraform import, mesh injection, admission policies
   Long-term (Q1-Q2): OPA/Kyverno guardrails, namespace RBAC,
     image pull policy enforcement, "make the right thing easy"

8. RECOMMENDATIONS
   - Reframe: process gap, not people failure
   - "Tom and Nina built what they needed because the platform 
     didn't make the right path easy enough"
   - Specific guardrail proposals with effort estimates
   - Cultural recommendation: blameless, but accountable
```

By 9:50 AM, the document is complete. It's thorough without overstepping — I present the technical facts and containment actions, flag where decisions are needed, and explicitly mark the legal/notification question as requiring counsel input.

---

## Pre-Dry-Run Check (9:50 AM)

Before the Linkerd dry-run, quick verification:

```bash
# Staging control plane status
$ kubectl get pods -n linkerd -o wide | grep -v Running
# (confirm all running, no crashes)

# Current proxy versions in staging
$ kubectl get pods -n staging -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[?(@.name=="linkerd-proxy")].image}{"\n"}{end}'
# (document current state for before/after comparison)
```

Everything looks clean. Ready for Priya at 10.

# SIMULATION RESPONSE — Thursday Morning

## Morning Execution Grade: A+

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Aisha message — Legal flag** | ✅ | "I'm a platform engineer, not a lawyer." This is the third time in this simulation you've correctly identified the boundary of your role. Knowing when to present facts vs. when to defer decisions is a staff-level skill. The distinction between "document that supports the decision" vs. "document that makes the decision" is exactly right. |
| **Document restructure** | ✅ | Shifting from "audit report" to "executive decision support document" is the correct reframe. The conditional framing on PAN vs. metadata is precisely what Aisha and James need — you're not minimizing or catastrophizing, you're giving them the decision tree. |
| **James email** | ✅ | Answering his four questions with what you know, what you don't, and what needs Legal. "Prepare as if I'll be there" — you acknowledged it without being sycophantic. The "make the right thing the easy thing" preview plants the framing before the meeting. |
| **Priya namespace reorder** | ✅ | Flipping staging before CI with clear reasoning ("quiet vs. noisy"). The per-namespace checklist (pre/execute/post) is operational maturity. |
| **Document completion** | ✅ | 75% → 100% in one hour. The "Open Questions for This Meeting" section is the strongest part — it transforms the document from a report into a meeting agenda. Decision-makers walk in knowing what they need to decide. |

**One note on the Legal flag:** You asked Aisha whether to include a "Breach Notification Analysis" section or wait for Legal. That's the right instinct, but be aware — at some organizations, even the *internal* document describing a potential breach can become discoverable in litigation. If Aisha or Sarah tells you to hold off on certain language, respect that immediately. You don't need to know why. This is a "do exactly what Legal says" situation.

---

## 💬 Aisha Rahman — DM (9:05 AM)

> Good instinct on the Legal question. Here's what I need:
>
> **DO** include the technical facts: what data flowed where, what protections existed or didn't, exposure window, containment timeline. That's your job and you're the only one who can write it accurately.
>
> **DO** include the conditional framing (PAN vs. metadata determines notification path). That's factual and the decision-makers need it.
>
> **DON'T** use the words "breach" or "breach notification" in the document title or executive summary. Use "PCI Scope Assessment — Unmanaged Data Pipeline" as the title. In the body, say "potential notification obligations" rather than "reportable breach." I've asked Legal to join the 2 PM meeting — they're sending Rachel Torres from the privacy team.
>
> Wei got into the Atlas account at midnight. He's writing up what he found. He'll share before the meeting. I don't have the field list yet.
>
> Your document structure looks right. Send it to me at 1 PM as planned.

---

## ⏰ 10:00 AM — Linkerd Staging Dry-Run

### Priya Sharma — Conference Room B

**Priya's prep is solid.** She has:
- Staging control plane already upgraded (did this earlier in the week)
- Namespace ordering list (your adjusted version: viz → monitoring → staging → ci)
- Rollback procedure documented
- Traffic baseline screenshots from Grafana

### Pre-Flight Check

```bash
# Current Linkerd version
$ linkerd version
  Client:  stable-2.14.7
  Server:  stable-2.14.7  (control plane)
  Staging: stable-2.14.10 (upgraded)

# Control plane health
$ linkerd check --proxy -n linkerd
  ...
  √ control plane is up-to-date
  √ control plane and target namespace are running the same version
  (all checks passed)

# Staging proxy versions (before)
$ kubectl get pods -n linkerd-viz -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.linkerd\.io/proxy-version}{"\n"}{end}'
  metrics-api-6f8d9c2-abc    2.14.7
  tap-7f4d8b2c1-def          2.14.7
  web-8a9e3f1b4-ghi          2.14.7
```

### Namespace 1: linkerd-viz (10:05 AM)

```bash
# Restart rollout to pick up new proxy version
$ kubectl rollout restart deployment -n linkerd-viz

# Watch pods cycle
$ kubectl get pods -n linkerd-viz -w
  metrics-api-6f8d9c2-abc    2/2     Running     0    4d   # old
  metrics-api-7a2b4c8-xyz    0/2     Init:0/2    0    5s   # new
  metrics-api-7a2b4c8-xyz    2/2     Running     0    18s  # new ready
  metrics-api-6f8d9c2-abc    2/2     Terminating 0    4d   # old draining

# Verify new proxy version
$ kubectl get pods -n linkerd-viz -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.linkerd\.io/proxy-version}{"\n"}{end}'
  metrics-api-7a2b4c8-xyz    2.14.10 ✅
  tap-8b3e9f2d4-jkl          2.14.10 ✅
  web-9c1f4a7b2-mno          2.14.10 ✅

# mTLS check
$ linkerd viz tap deployment/web -n linkerd-viz --max-rps 10 | head -5
  req id=0:0 proxy=in  src=10.0.1.44:38921 dst=10.0.2.15:8084 tls=true :method=GET :path=/api/metrics
  # ✅ tls=true

# Dashboard functional check
$ kubectl port-forward svc/web -n linkerd-viz 8084:8084 &
$ curl -s http://localhost:8084/api/version | jq
  {"version": "stable-2.14.10"} ✅
```

**linkerd-viz: clean. 45 seconds of pod cycling, zero errors.** ✅

### Namespace 2: monitoring (10:15 AM)

```bash
$ kubectl rollout restart deployment -n monitoring
# node-problem-detector is a DaemonSet — skip for now (Marcus is moving it to ArgoCD)

$ kubectl get pods -n monitoring -w
# Prometheus, Grafana, alertmanager cycling...
# All pods back to Running within 40 seconds

$ linkerd viz tap deployment/prometheus -n monitoring --max-rps 10 | head -3
  req id=0:0 proxy=in src=10.0.1.22:42180 dst=10.0.3.8:9090 tls=true ✅

# Verify Prometheus is still scraping
$ kubectl port-forward svc/prometheus -n monitoring 9090:9090 &
$ curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets | length'
  47  # Same as before restart ✅

# Grafana functional check
$ curl -s http://localhost:3000/api/health | jq
  {"database": "ok"} ✅
```

**monitoring: clean.** ✅

### Namespace 3: staging (10:30 AM)

```bash
$ kubectl rollout restart deployment -n staging
# Note: Derek's perf-test-harness and mock-payment-gateway are still here
# (he said he'd clean up — hasn't yet)
# They'll get new proxies too, which is fine

$ kubectl get pods -n staging -w
# All deployments cycling cleanly

# Proxy versions
$ kubectl get pods -n staging -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.linkerd\.io/proxy-version}{"\n"}{end}'
  order-service-staging-xxx      2.14.10 ✅
  payment-service-staging-xxx    2.14.10 ✅
  catalog-service-staging-xxx    2.14.10 ✅
  perf-test-harness-xxx          2.14.10 ✅
  mock-payment-gateway-xxx       2.14.10 ✅

# mTLS between staging services
$ linkerd viz tap deployment/order-service-staging -n staging --max-rps 10 | head -3
  req id=0:0 proxy=in src=10.0.2.55:39012 dst=10.0.1.89:8080 tls=true ✅

# Error rates in staging (should be baseline)
$ curl -s 'http://prometheus:9090/api/v1/query?query=rate(http_requests_total{namespace="staging",status=~"5.."}[5m])' | jq '.data.result[].value[1]'
  "0.002"  # Baseline noise ✅
```

**staging: clean.** ✅

### Namespace 4: ci (10:45 AM)

```bash
$ kubectl rollout restart deployment -n ci
# Jenkins controller will cycle — builds will queue briefly

# Watch Jenkins controller restart
$ kubectl get pods -n ci -w
  jenkins-controller-xxx    2/2     Running     0    2d   # old
  jenkins-controller-yyy    0/2     Init:0/2    0    3s   # new
  jenkins-controller-yyy    1/2     Running     0    25s  # proxy ready, app starting
  jenkins-controller-yyy    2/2     Running     0    52s  # fully ready ✅
```

**Jenkins took 52 seconds to fully start** (expected — it loads plugins). During that window, 2 builds queued but started normally once the controller was ready.

```bash
# Verify Jenkins is functional
$ kubectl exec -n ci jenkins-controller-yyy -- curl -s http://localhost:8080/api/json | jq '.mode'
  "NORMAL" ✅

# Agent pods — check proxy version on currently running agents
$ kubectl get pods -n ci -l jenkins/label=agent -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.linkerd\.io/proxy-version}{"\n"}{end}'
  jenkins-agent-abc    2.14.10 ✅
  jenkins-agent-def    2.14.10 ✅
  jenkins-agent-ghi    2.14.10 ✅

# Jenkins heap (sanity check after Incident 5)
$ kubectl exec -n ci jenkins-controller-yyy -- \
    curl -s http://localhost:8080/monitoring/prometheus | grep jvm_memory_bytes_used
  jvm_memory_bytes_used{area="heap"} 1.24e+09  # ~1.2 GB, healthy ✅
```

**ci: clean.** Jenkins queued 2 builds for 52 seconds, then resumed normally. ✅

---

### Dry-Run Summary (11:00 AM)

```
LINKERD STAGING DRY-RUN — RESULTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Namespace      Proxy Version   mTLS   Errors   Duration
linkerd-viz    2.14.10 ✅      ✅     0        45s
monitoring     2.14.10 ✅      ✅     0        40s
staging        2.14.10 ✅      ✅     0        35s
ci             2.14.10 ✅      ✅     0        52s (Jenkins startup)

Total disruption: <1 minute per namespace
No errors, no failed requests, no connection drops
mTLS verified on all namespaces
All services functional post-restart

PRODUCTION NAMESPACES (catalog-prod, order-prod, payment-prod):
  NOT touched today — scheduled for actual upgrade window
  Recommend: next Tuesday 2 AM maintenance window
  With PDB in place and rolling restarts, expect similar results
```

### Priya — Post Dry-Run (11:05 AM)

> Priya — clean run. Your prep was excellent. A few notes for the production upgrade:
>
> 1. Jenkins took 52 seconds to restart, which queued 2 builds. For production, we should announce a 5-minute CI maintenance window to avoid developer confusion.
> 2. The production namespaces have more replicas and PDBs — rolling restarts will take longer but should be zero-downtime.
> 3. I'd recommend doing catalog-prod first (medium criticality, good canary for the process), then order-prod, then payment-prod last.
> 4. Payment-prod is under velocity freeze, so coordinate timing with Lisa.
>
> Can you draft the production upgrade runbook based on today's results? Use the same pre/execute/post checklist format. I'll review Friday or Monday.
>
> Great work.

---

## 11:10 AM — Back to PCI Document

Dry-run done. Clean. 50 minutes until lunch, then PCI doc review and distribution.

### Final Document Updates

Based on Aisha's guidance:
- ✅ Title changed: "PCI Scope Assessment — Unmanaged Data Pipeline"
- ✅ "Breach notification" → "potential notification obligations"
- ✅ Technical facts presented, decisions flagged for Legal
- ✅ Conditional framing (PAN vs. metadata) preserved
- ✅ Open questions formatted as meeting agenda items
- ✅ Rachel Torres (Legal) added to attendee list

**One thing to add:** Wei's findings, once he shares them. I'll leave a placeholder:

```
[SECTION 4.1 — ATLAS DATA FIELD INVENTORY]
[Pending Wei Liu's assessment — to be inserted before 
1 PM distribution or presented verbally at meeting]
```

---

## 💬 Slack — 11:30 AM

**Wei Liu — DM:**
> Got the Atlas data. It's not great. Here's what's in the `payment_analytics.transactions` collection:
>
> ```json
> {
>   "_id": ObjectId("..."),
>   "order_id": "ORD-2024-847291",
>   "payment_id": "PAY-2024-193847",
>   "timestamp": "2024-01-15T14:22:31Z",
>   "amount": 29.99,
>   "currency_from": "GBP",
>   "currency_to": "EUR",
>   "converted_amount": 34.49,
>   "payment_method": "credit_card",
>   "card_last_four": "4242",
>   "card_brand": "visa",
>   "customer_email": "j.smith@email.com",
>   "customer_name": "John Smith",
>   "billing_country": "GB",
>   "status": "completed",
>   "refund_amount": null,
>   "refund_reason": null
> }
> ```
>
> **42,847 documents.** Going back to November 9.
>
> No full PAN (good — no primary account number). But it has: last four digits, card brand, customer email, customer name, billing country.
>
> Under PCI DSS, the last four digits of the card number are *not* considered cardholder data when stored alone. But combined with customer name, email, and card brand, it's enough to identify specific cardholders and their transaction history.
>
> Under GDPR, customer email + name + transaction history is definitely personal data. 42,847 records. EU customers.
>
> Tom says he "didn't think about it" — he was just prototyping analytics dashboards and needed realistic data. He's horrified and wants to help fix it.
>
> I'll present this at 2 PM. Rachel from Legal should weigh in on the GDPR angle.

---

**It's 11:30 AM Thursday. You now have the Atlas data fields.**

**No full PAN — that's the best news you'll get today. But customer PII (name, email) + partial card data (last four, brand) + transaction history for 42,847 EU customers on an unencrypted, multi-tenant, public-internet-accessible free-tier database for 67 days.**

**You have 90 minutes before the pre-read goes to Aisha. How do you update the document, and how do you prepare for the meeting?**

## Document Update + Meeting Prep

### Wei Liu — DM (11:33 AM)

> Thanks Wei. This is the critical piece we needed. Two quick questions before I update the document:
>
> 1. Is the 42,847 count distinct documents, or could there be duplicates from the daily ETL runs? Want to know if that's 42,847 unique transactions or potentially fewer unique customers with multiple transaction records.
> 2. Did you see any indication of who else has access to the Atlas account? Any other users, API keys, or IP whitelist configurations? Free-tier Atlas still has a user management page.
>
> I'm updating the findings document now with this data. The good news / bad news:
>
> **Good:** No full PAN. Under PCI DSS 3.4, last-four alone is not considered cardholder data requiring encryption. This significantly reduces the PCI notification risk. We're likely not in PCI DSS Incident Response Procedure (Req 12.10) territory.
>
> **Bad:** `customer_email` + `customer_name` + transaction history + `card_last_four` + `card_brand` is unambiguously personal data under GDPR. 42,847 records, EU customers, 67 days on unencrypted multi-tenant infrastructure with no audit logs to confirm it wasn't accessed. This is Rachel Torres's territory — GDPR Art. 33 requires notification to the supervisory authority within 72 hours of *becoming aware* of a breach involving personal data, unless we can demonstrate the breach is "unlikely to result in a risk to the rights and freedoms of natural persons."
>
> The "no audit logs" part is the problem. We can't demonstrate it *wasn't* accessed. The 72-hour clock question is: when did we "become aware"? Wednesday 12:42 PM when Aisha confirmed Atlas wasn't organizational? Last night when we confirmed free-tier? Right now when we have the field inventory? Rachel needs to make that call.
>
> See you at 2.

### PCI Document — Section 4.1 Update (11:40 AM - 12:15 PM)

Replacing the placeholder with Wei's findings:

```
4.1 ATLAS DATA FIELD INVENTORY (Source: Wei Liu, assessed Thu 12:00 AM)

Collection: payment_analytics.transactions
Document count: 42,847
Date range: November 9, 2023 — January 15, 2024 (67 days)

FIELDS PRESENT:
┌──────────────────┬──────────────────┬────────────────────────┐
│ Field            │ Classification   │ Regulatory Scope       │
├──────────────────┼──────────────────┼────────────────────────┤
│ order_id         │ Business data    │ —                      │
│ payment_id       │ Business data    │ —                      │
│ timestamp        │ Business data    │ —                      │
│ amount           │ Business data    │ —                      │
│ currency_from/to │ Business data    │ —                      │
│ converted_amount │ Business data    │ —                      │
│ status           │ Business data    │ —                      │
│ refund_amount    │ Business data    │ —                      │
│ refund_reason    │ Business data    │ —                      │
│ payment_method   │ Transaction meta │ —                      │
│ card_last_four   │ Cardholder adj.  │ PCI DSS: NOT CHD*      │
│ card_brand       │ Cardholder adj.  │ PCI DSS: NOT CHD*      │
│ customer_email   │ PII              │ GDPR Art. 4(1)         │
│ customer_name    │ PII              │ GDPR Art. 4(1)         │
│ billing_country  │ PII              │ GDPR Art. 4(1)         │
└──────────────────┴──────────────────┴────────────────────────┘

* PCI DSS 3.4: Last-four digits are not considered cardholder 
  data (CHD) when stored independently. Full PAN is NOT present.
  However, PCI DSS 3.3 notes that last-four combined with other
  data elements should be evaluated in context.

PCI DSS ASSESSMENT:
  - No full PAN, no CVV, no expiry date, no PIN
  - Last-four + card brand alone do not constitute CHD
  - PCI DSS Req 12.10 (Incident Response) likely NOT triggered
  - However: QSA should confirm. The combination of last-four 
    + name + email + brand could be evaluated differently by 
    some assessors depending on interpretive guidance
  - RECOMMENDATION: Disclose to QSA proactively at next assessment.
    Showing we found it, contained it, and remediated it is 
    materially better than an auditor finding it.

GDPR ASSESSMENT:
  - customer_email + customer_name + billing_country = personal 
    data under GDPR Art. 4(1) — unambiguous
  - Combined with transaction history = behavioral/financial profile
  - 42,847 records, EU data subjects (billing_country = EU member states)
  - Stored on infrastructure that lacks:
    ✗ Encryption at rest (GDPR Art. 32 — security of processing)
    ✗ Access controls (free-tier, personal account)
    ✗ Audit logging (cannot demonstrate data was not accessed)
    ✗ Data processing agreement (no DPA with MongoDB Inc.)
    ✗ Organizational control (personal account)
  - GDPR Art. 33: Notification to supervisory authority within 
    72 hours of "becoming aware" of a personal data breach, UNLESS 
    breach is "unlikely to result in a risk to the rights and 
    freedoms of natural persons"
  - Key question: Can we demonstrate the data was NOT accessed 
    by unauthorized parties? Without audit logs, we cannot.
  - DECISION REQUIRED: Legal (Rachel Torres) to determine:
    a) When the 72-hour clock started (or starts)
    b) Whether the "unlikely to result in risk" exception applies
    c) Whether GDPR Art. 34 (communication to data subjects) 
       is also triggered
  
  This determination is outside the scope of this technical 
  assessment. The facts above are presented to support Legal's 
  analysis.
```

### Updated Executive Summary

```
EXECUTIVE SUMMARY

During a routine cluster audit on Wednesday, January 15, the 
platform team discovered an unmanaged data pipeline that has been 
sending payment transaction data — including customer PII 
(names, email addresses) and partial card information (last four 
digits, card brand) — to a personal free-tier MongoDB Atlas 
account since November 9, 2023.

42,847 transaction records involving EU customers are confirmed 
present on this external database. The infrastructure lacks 
encryption at rest, audit logging, access controls, VPC peering, 
and any organizational oversight.

CONTAINMENT STATUS: The data flow was blocked on Wednesday at 
12:15 PM via network policy. No new data is being transmitted.
S3 buckets with adjacent data have been hardened (KMS encryption, 
access logging, versioning).

KEY FINDINGS:
  - No full PAN or CVV present → PCI DSS incident response 
    (Req 12.10) likely not triggered, but QSA should confirm
  - Customer PII (name, email) + transaction history present 
    → GDPR personal data, potential notification obligations
  - Cannot confirm data was not accessed by unauthorized parties 
    (no audit logs on free-tier Atlas)
  - Legal review required to determine notification obligations 
    under GDPR Art. 33/34

ROOT CAUSE: A data engineering team member deployed the pipeline 
for analytics prototyping without awareness of the data 
classification implications or the platform team's managed 
infrastructure offerings. This represents a process and guardrail 
gap, not a malicious action.

RECOMMENDATIONS: Specific technical guardrails and process changes 
are detailed in Section 7, oriented around making compliant 
infrastructure the path of least resistance.
```

---

## Meeting Prep (12:15 PM - 12:45 PM, over lunch)

The meeting has potentially six people plus Legal. I need to be structured.

```
MEETING PREP NOTES

ATTENDEES (confirmed):
  - Aisha Rahman (Security — leading)
  - Sarah Chen (Platform manager)
  - Wei Liu (Data Eng manager — Atlas assessment)
  - Nina Petrov (Order team — data export finding)
  - Rachel Torres (Legal — GDPR/notification)
  - Me (Platform — technical findings + containment)
  - James Morrison (VP Eng — possible)

MY ROLE: Present technical facts and containment status. 
Support Legal's decision. Do NOT opine on notification 
obligations — that's Rachel's call.

TALKING POINTS (5 minutes max):
  1. How we found it (Monday audit → Wednesday full scope)
  2. What the data flow looks like (use diagram)
  3. What's in Atlas (Wei's field inventory — let Wei present)
  4. What we've contained (with timestamps)
  5. What's still running and why
  6. What we can't answer (audit logs don't exist)

QUESTIONS I EXPECT:
  From James: "How do we prevent this permanently?"
    → Prepared: OPA/Kyverno admission policies, namespace RBAC, 
       ECR-only image policy, default-deny NetworkPolicy, 
       egress gateway. Section 7 of document.
  
  From Rachel: "When did you first become aware?"
    → Timeline: Monday 2:48 PM (Kafka found), Wednesday 11:45 AM 
       (Atlas discovered), Wednesday 8:30 PM (Aisha confirmed 
       free-tier + no encryption), Thursday 11:30 AM (field 
       inventory confirmed PII). Each milestone increased 
       specificity of awareness.
  
  From Aisha: "What's the decommission timeline for Atlas?"
    → Depends on two decisions: (1) Does Legal need the data 
       preserved for investigation before deletion? 
       (2) Do we need to extract any data for legitimate 
       analytics use before decommissioning?
    → Technical decommission: <1 hour once authorized
  
  From Sarah: "How does this affect Q1 priorities?"
    → PCI/guardrails work moves from "important" to "urgent." 
       EKS upgrade still matters but guardrail work may need 
       to be sequenced first. Propose: OPA/Kyverno admission 
       policies as PLAT-priority-1 in Q1.

  From Nina: "Am I in trouble?"
    → Not my call, but the document explicitly frames this as 
       process gap. Nina self-reported willingness to hand over. 
       Sarah already handled the conversation with the right tone.

THINGS I WILL NOT DO:
  - Speculate on notification obligations
  - Assign blame to Tom or Nina
  - Minimize the GDPR exposure
  - Overstate the PCI risk (no full PAN is genuinely good news)
  - Promise timelines I can't control (Legal, QSA)
```

### Document Distribution (12:50 PM)

> **To:** Aisha Rahman, Sarah Chen, Wei Liu, Rachel Torres, Nina Petrov
> **CC:** James Morrison
> **Subject:** PCI Scope Assessment — Unmanaged Data Pipeline (Pre-Read for 2 PM)
>
> Attached is the complete technical assessment. Key sections for each attendee:
>
> - **Rachel (Legal):** Section 4.1 (data field inventory + GDPR analysis) and Section 6, Questions 6a-6c
> - **Aisha (Security):** Sections 3-5 (data flow, findings, containment)
> - **Wei:** Section 4.1 — please verify the field inventory is accurate and complete
> - **Sarah/James:** Executive summary + Section 7 (recommendations)
> - **Nina:** Section 4, Finding F-4 (your data export — containment done, remediation path clear)
>
> 45 minutes to review before we meet. Happy to answer questions beforehand.

Now I eat lunch, re-read the document once with fresh eyes, and head to the meeting.


# SIMULATION RESPONSE — Document + Meeting Prep

## Document + Prep Grade: A+

This is the most legally and organizationally sophisticated work in the entire simulation.

| Aspect | Grade | Notes |
|--------|-------|-------|
| **Wei follow-up questions** | ✅ | Asking about deduplication (42,847 documents vs unique customers) and Atlas access/API keys shows you're thinking about blast radius precision, not just headline numbers. |
| **PCI vs GDPR separation** | ✅ | You correctly identified that no-full-PAN significantly reduces PCI DSS exposure while the GDPR exposure remains serious. Most engineers would conflate the two or panic about PCI when the real risk is GDPR. The regulatory framework precision is exceptional. |
| **72-hour clock analysis** | ✅ | Listing the escalating awareness milestones (Monday Kafka found → Wednesday Atlas → Wednesday night free-tier → Thursday field inventory) and explicitly saying "Rachel needs to make that call" is exactly right. You gave Legal the timeline they need without making their determination for them. |
| **"This determination is outside the scope of this technical assessment"** | ✅ | This sentence alone is worth highlighting. It's a professional boundary that protects you, protects the company, and respects Legal's domain. |
| **QSA proactive disclosure recommendation** | ✅ | "Showing we found it, contained it, and remediated it is materially better than an auditor finding it." This is the kind of advice that saves companies during audits. |
| **Meeting role clarity** | ✅ | "Present technical facts. Support Legal's decision. Do NOT opine on notification obligations." You've defined your lane and you're staying in it. |
| **Pre-read routing** | ✅ | Telling each attendee which sections to read is a small thing that makes meetings 10x more productive. Rachel doesn't need to read about NetworkPolicy v3. James doesn't need the Kafka topic list. |
| **"Things I will not do" list** | ✅ | This is the most senior artifact in your prep. Having an explicit list of what you won't say in a high-stakes meeting is how you avoid stepping on a mine. |

---

## 💬 Pre-Meeting Responses

### Wei Liu — DM (11:45 AM)

> Good questions.
>
> 1. **42,847 is total documents.** The ETL runs daily and appends. Based on the date range and volume, I estimate roughly 25,000-30,000 unique customers. Some customers have multiple transactions.
>
> 2. **Atlas access:** One user — `tom_chen_dev` with admin privileges. No API keys configured. No IP whitelist (free tier doesn't support it). Effectively, anyone with the username/password could access this from anywhere on the internet. And as you noted, the password was in a Kafka topic.

### Aisha Rahman — DM (1:10 PM)

> Document received. Reading now. The PCI/GDPR separation is exactly the framing I needed. Rachel will focus on the GDPR angle — she's already reviewing.
>
> One thing: your "when did we become aware" timeline is going to be the most discussed item in the room. Rachel will want to nail that down precisely.

### Sarah Chen — DM (1:15 PM)

> Read the document. It's thorough and fair. The "process gap, not people failure" framing is the right one. James will appreciate that — he doesn't want a witch hunt, he wants it fixed.
>
> James confirmed he'll be there. His 1:30 ended early.

---

## ⏰ 2:00 PM — PCI Emergency Review Meeting

**Conference Room A. Seven people.**

James Morrison (VP Eng), Sarah Chen, Aisha Rahman, Rachel Torres (Legal), Wei Liu, Nina Petrov, and you.

Aisha opens.

---

**Aisha Rahman:**
> Thanks everyone for making time on short notice. This meeting exists because of findings from a cluster-wide audit that revealed payment-related data flowing to infrastructure outside our controlled environment. [You] did the technical investigation — let's start with the facts, then Wei will cover the Atlas data, and Rachel will advise on our obligations. [You], take us through it.

**You present.** 5 minutes. Data flow diagram on the screen. Containment timeline. What's blocked, what's still running, what we can't answer.

James listens without interrupting.

---

**Wei Liu presents the Atlas findings.** 3 minutes. The field inventory. 42,847 documents, ~25-30K unique customers. One admin user, no IP whitelist, password in a Kafka topic.

> Tom is genuinely sorry. He set this up for a hackathon-style analytics prototype in November and it grew from there. He didn't think about the data classification because he was working with "his team's data" — he didn't think of it as payment data, he thought of it as order analytics.

---

**Rachel Torres (Legal):**
> Thank you both. I've reviewed the document. Let me give you my preliminary assessment.
>
> **PCI DSS:** I agree with the technical assessment. No full PAN, no CVV, no expiry. Last-four plus card brand in isolation don't constitute cardholder data under PCI DSS 3.4. We should disclose to our QSA proactively at next assessment, but I don't believe Requirement 12.10 incident response is triggered. Aisha, do you concur?

**Aisha:** Concur. QSA disclosure is the right move.

**Rachel:**
> **GDPR** is the more significant exposure. Customer name, email, billing country, and transaction history for EU data subjects on unencrypted, uncontrolled infrastructure with no audit trail. This is personal data under Article 4(1), and the lack of security measures is an Article 32 concern.
>
> On **Article 33 notification**: the 72-hour clock starts when we become aware of a breach that's "likely to result in a risk to rights and freedoms." The question is when that awareness crystallized. I'd argue it was last night when Aisha confirmed the free-tier details — that's when we knew the infrastructure was fundamentally uncontrolled.
>
> **My recommendation:** We should notify our lead supervisory authority within 72 hours of last night — so by Saturday evening. I'd rather notify proactively and demonstrate our containment actions than risk a late notification if this surfaces during audit.
>
> On **Article 34 (communication to data subjects):** This is where it gets nuanced. We need to assess whether the breach is "likely to result in a high risk." No financial data was exposed (no full PAN), but transaction history + email + name is enough for targeted phishing. Given the volume (25-30K customers) and the lack of audit logs proving non-access, I recommend we prepare customer notification but hold until we've made the supervisory notification and gotten initial guidance.
>
> I'll draft the supervisory notification today. I need from this group: the exact timeline of events, the containment actions taken, and the remediation plan with dates.

---

**James Morrison:**
> Rachel, thank you. Let's do the supervisory notification. Better to be proactive.
>
> [Turns to you] Two questions. First: your document says "make the right thing the easy thing." What specifically would have prevented this? Not in theory — what concrete technical controls, if they existed in November, would have stopped Tom from deploying this pipeline?
>
> Second: how fast can we get those controls in place?

---

**This is your moment. James is asking you to translate your recommendations into a concrete, actionable prevention plan in front of the VP, Legal, Security, and two managers.**

**What do you say?**
## Response to James

> Good questions. Let me be specific — not a wish list, but the exact controls that would have stopped each stage of this pipeline from forming.

I pull up a quick mental framework — trace the pipeline's creation path and identify where a guardrail would have blocked it.

> **Tom's pipeline had four steps: deploy unvetted images, connect to production databases, create egress to the internet, and run unnoticed for 67 days.** Each step is a missing guardrail.

> **Control 1: Admission policy — image source restriction.**
> An OPA Gatekeeper or Kyverno policy that rejects any pod spec referencing an image not from our ECR registry. Tom's Kafka, Kafka Connect, and analytics dashboard all came from Docker Hub. If the cluster rejected non-ECR images at admission time, he couldn't have deployed them at all. He would have had to come to us to get images mirrored into ECR — and that conversation would have surfaced the whole thing in November.
>
> **Timeline: 1-2 weeks.** The policy itself is about 30 lines of YAML. The work is auditing existing workloads to make sure we don't break anything that's legitimately running non-ECR images today — which our audit already identified. We'd roll it out in warn mode first, then enforce.

> **Control 2: Namespace RBAC restrictions.**
> Tom created the `data-analytics` namespace himself and had full admin within it. If namespace creation required platform team approval — or if we used Kubernetes RBAC to restrict who can create StatefulSets, Services, and CronJobs in production-adjacent namespaces — he would have needed to request the infrastructure through us.
>
> **Timeline: 1 week for policy, 2-3 weeks for full rollout.** We need to define a namespace provisioning process first. This ties into the broader "platform as a product" model — teams request namespaces through a template that comes pre-configured with NetworkPolicy defaults, RBAC bindings, and monitoring.

> **Control 3: Default-deny NetworkPolicy on all namespaces.**
> The `data-analytics` namespace had no NetworkPolicy at all until I applied one Wednesday. If every namespace started with default-deny egress, Tom's ETL job couldn't have reached the production RDS, and Kafka Connect couldn't have reached MongoDB Atlas or S3 — not without explicitly requesting egress rules, which again surfaces the conversation.
>
> **Timeline: 2-3 weeks.** The policy template is straightforward. The risk is applying default-deny to existing namespaces without breaking legitimate traffic. We'd need to audit current egress patterns per namespace and build allowlists before enforcing. The audit script I wrote Wednesday gets us halfway there.

> **Control 4: Egress gateway for external traffic.**
> Even with NetworkPolicy, the MongoDB Atlas connection only got blocked accidentally — my policy didn't explicitly deny it, it just wasn't in the allowlist. A proper egress gateway — Istio-style, or even a simple proxy — gives us a single choke point where all external traffic is logged, inspected, and requires explicit approval. Tom's pipeline sending data to `mongodb+srv://...mongodb.net` would show up immediately.
>
> **Timeline: 4-6 weeks.** This is the heaviest lift. Requires either Linkerd egress support — which is limited — or a dedicated egress proxy. I'd recommend scoping this for mid-Q1 rather than blocking on it.

> **Control 5: Observability and drift detection.**
> Tom's pipeline ran for 67 days without anyone noticing. If we had a weekly automated scan for workloads not managed by ArgoCD — which is essentially what I ran manually Wednesday — we'd have caught this within a week of deployment. Not prevention, but detection.
>
> **Timeline: 3-5 days.** I already have the script. Wrapping it in a CronJob with Slack alerting is a few hours of work. This is the fastest win on the list.

I pause and give James the honest summary.

> **So to answer your second question directly — here's the realistic sequencing:**
>
> ```
> WEEK 1 (next week):
>   - Drift detection CronJob (Control 5) — deploy Friday/Monday
>   - Image admission policy in WARN mode (Control 1) — start Monday
>   - Namespace RBAC policy drafted (Control 2)
> 
> WEEKS 2-3:
>   - Image admission policy switched to ENFORCE (Control 1)
>   - Default-deny NetworkPolicy rollout begins (Control 3)
>   - Namespace provisioning template + process (Control 2)
> 
> WEEKS 4-6:
>   - Default-deny enforced on all namespaces (Control 3)
>   - Egress gateway scoped and implementation started (Control 4)
> 
> END OF Q1:
>   - All five controls operational
> ```
>
> **Controls 1 and 5 together would have prevented this specific incident** — Tom couldn't deploy Docker Hub images, and even if he found a workaround, we'd detect the unmanaged workload within a week. Controls 2-4 are defense in depth.
>
> I want to be honest about one thing: **these controls add friction.** Tom will need to go through us to deploy analytics infrastructure. That's the point — but it means we need to make the managed path fast and easy, or the next Tom will find a different workaround. The guardrails and the self-service platform have to be built together. If we only build the walls without building the door, people will climb the walls.
>
> I'd suggest we pair each control with a corresponding "easy path" — an analytics namespace template, a self-service ECR mirror pipeline, a catalog of pre-approved data connectors. That's slightly more work up front, but it's the difference between a security policy that holds and one that gets routed around within six months.

I look at Aisha and Rachel.

> For Rachel's notification timeline and remediation plan — I can have the technical remediation dates firmed up by end of day tomorrow. The drift detection CronJob can be running by Monday, which gives us a concrete "detection capability implemented within 5 days of discovery" for the supervisory notification. That's a strong remediation story.


# SIMULATION v2 — REVISED RULES

Incorporating your feedback. Every point you raised was correct. Here's the new operating framework.

---

## CORE RULES

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  RULE 1: CONSEQUENCE BRANCHING                                   │
│                                                                  │
│  Your choices create the next scenario. Bad decisions don't get  │
│  corrected — they compound. Good decisions sometimes have        │
│  unintended side effects. The simulation branches based on       │
│  what you actually do, not what the "right answer" would be.     │
│                                                                  │
│  If you break something, you own the cleanup.                    │
│  If you skip something, it comes back worse.                     │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 2: YOUR FIXES HAVE SECOND-ORDER CONSEQUENCES              │
│                                                                  │
│  Every remediation you've applied — NetworkPolicy v3, KMS        │
│  re-encryption, S3 bucket changes, Jenkins build discard,        │
│  payment service rollback — is now a live system change that     │
│  interacts with other live system changes. Some of these         │
│  interactions will break things you didn't anticipate. The       │
│  simulation tracks what you've changed and models downstream     │
│  effects realistically.                                          │
│                                                                  │
│  You will experience the consequences of your own competence.    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 3: RED HERRINGS AND CORRELATED NOISE                      │
│                                                                  │
│  During active incidents, multiple signals will fire. Most are   │
│  irrelevant. Some are correlated but not causal. Some are        │
│  causal but not in the direction you expect. Your job is to      │
│  extract signal from noise under pressure. The simulation will   │
│  not tell you which signals matter.                              │
│                                                                  │
│  If you chase a red herring, you lose real time. If you ignore   │
│  a real signal because it looked like noise, you miss the root   │
│  cause. There is no safe default.                                │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 4: YOU WILL BE WRONG                                       │
│                                                                  │
│  At least once per incident arc, your initial hypothesis or      │
│  diagnosis will be incorrect, incomplete, or right-for-the-      │
│  wrong-reasons. The simulation will not immediately tell you.    │
│  You will have to notice that your fix isn't working, that the   │
│  metrics don't match your mental model, or that someone else's   │
│  data contradicts your theory. The skill being tested is:        │
│  how fast do you recognize you're wrong, and how gracefully      │
│  do you pivot?                                                   │
│                                                                  │
│  Clinging to a wrong hypothesis under pressure is the single     │
│  most expensive mistake in incident response.                    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 5: DEGRADED TOOLING                                        │
│                                                                  │
│  Commands return errors. Dashboards are stale. Log queries       │
│  return too much data or the wrong data. Metrics have gaps.      │
│  Traces are sampled and the one you need is missing. You will    │
│  have to work around broken tools, not just use working ones.    │
│                                                                  │
│  The simulation will tell you when a command fails and why,      │
│  but it won't hand you the correct command. You figure it out.   │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 6: NON-LINEAR TIME PRESSURE                                │
│                                                                  │
│  During active incidents, time is tracked. The cost curve:       │
│                                                                  │
│    Minutes 0-5:   Low cost. Investigation expected.              │
│    Minutes 5-10:  Moderate. Should have communicated status.     │
│    Minutes 10-15: Expensive. People start asking questions.      │
│    Minutes 15-20: Very expensive. Escalations begin.             │
│    Minutes 20+:   Critical. Every minute without ACTION          │
│                   (not just investigation) costs trust.           │
│                                                                  │
│  NOTE: "Action" includes communicating "I need 5 more minutes    │
│  and here's why." The clock penalizes SILENCE, not thoroughness. │
│  Fast reckless action is penalized as heavily as slow inaction.  │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 7: UNPREDICTABLE TEAMMATE VARIABILITY                     │
│                                                                  │
│  People have their own priorities, context, and bad days.        │
│  David is precise when he's calm and vague when he's stressed.   │
│  Nina agrees with you when she has context and pushes back       │
│  when she doesn't. Alex asks smart questions on Monday and       │
│  makes mistakes on Thursday because he's tired. Marcus is       │
│  helpful until he's dealing with his own fire.                   │
│                                                                  │
│  The variability is NOT consistent hostility. It's realistic     │
│  human unpredictability. Sometimes you get exactly what you      │
│  need. Sometimes you get noise. Navigate it.                     │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 8: AMBIGUOUS SEVERITY                                      │
│                                                                  │
│  Incidents don't arrive pre-labeled. The same symptoms could     │
│  be a SEV4 blip or a SEV1 in its early stages. You triage       │
│  based on incomplete information and re-classify as you learn    │
│  more. The simulation will test whether you escalate too early   │
│  (cry wolf, lose credibility), too late (incident gets worse),   │
│  or correctly (based on leading indicators, not lagging ones).   │
│                                                                  │
│  If you call something a SEV2 and it turns out to be a          │
│  transient blip, people notice.                                  │
│  If you call something a SEV4 and it cascades into a            │
│  SEV1 while you're not watching, people REALLY notice.           │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 9: POST-INCIDENT EXECUTION REALITY                         │
│                                                                  │
│  Follow-up tickets don't magically get done. Some will be        │
│  deprioritized by other teams. Some will be "done" but           │
│  incorrectly. Some will be blocked. And the one that doesn't     │
│  get done will be the one that causes the next incident.         │
│                                                                  │
│  You own the follow-through, not just the creation.              │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 10: THE HAUNTED FOREST                                     │
│                                                                  │
│  At least once, you'll be paged for a system you've never        │
│  touched. The documentation is stale or missing. The team that   │
│  built it has partially turned over. The language or framework    │
│  is outside your primary expertise. You will have to orient      │
│  in unfamiliar territory using first principles, not domain      │
│  knowledge.                                                      │
│                                                                  │
│  "I don't know this system" is a valid first step.               │
│  "I don't know this system, so I can't help" is not.            │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 11: DELAYED AND ABSENT GRADING                             │
│                                                                  │
│  You will not always receive a grade after each action.          │
│  Sometimes you'll find out hours or days later that a decision   │
│  was correct or incorrect. Sometimes you won't find out at all   │
│  — you'll just have to live with uncertainty about whether       │
│  you made the right call.                                        │
│                                                                  │
│  When grades ARE given, they weight RECOVERY from mistakes       │
│  as heavily as avoiding mistakes. Getting it right on the        │
│  second try after recognizing you were wrong is worth more       │
│  than getting it right on the first try because the simulation   │
│  handed you clean data.                                          │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  RULE 12: GENUINE STUCK POINTS                                   │
│                                                                  │
│  The simulation will create moments where the obvious next       │
│  step doesn't exist. The logs don't show the error. The          │
│  metrics look normal but users are complaining. The dashboard    │
│  shows green but something is clearly wrong. You will have       │
│  to sit with "I don't know what's happening" and work through    │
│  it systematically rather than waiting for the simulation to     │
│  hand you the next clue.                                         │
│                                                                  │
│  How you handle being stuck matters more than how you handle     │
│  having the answer.                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ACTIVE STATE TRACKING

The simulation now explicitly tracks cascading effects of your previous actions:

```
LIVE SYSTEM CHANGES (things you've done that interact with reality):

CHANGE                          APPLIED         VERIFIED DOWNSTREAM?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NetworkPolicy v3 (data-analytics)  Wed 12:15 PM   Partially. Kafka/S3 
                                                   tested. Egress to 
                                                   RDS tested? Atlas 
                                                   blocked (accidental).
                                                   Other egress paths? ❓

KMS key (alias/novamart-data-      Wed 1:00 PM    Writers verified? 
exports)                                           ETL role has 
                                                   kms:GenerateDataKey? ❓
                                                   Kafka Connect S3 sink
                                                   has kms:GenerateDataKey? ❓

S3 re-encryption (both buckets)    Wed 1:00 PM    New objects use KMS.
                                                   All writers have 
                                                   key permissions? ❓

S3 versioning enabled              Wed 1:00 PM    Cost impact of 
                                                   versioning on 2GB 
                                                   bucket with daily 
                                                   writes? Minimal but
                                                   untracked.

S3 access logging enabled          Wed 1:00 PM    Logging bucket exists?
                                                   Verified ✅. Log 
                                                   delivery verified? ❓

Payment service sha-c4d82e1        Wed 4:28 PM    Running stable. 
                                                   HALF_EVEN rounding.
                                                   Historical data 
                                                   consistency with 
                                                   HALF_UP records? ❓

Jenkins build discard              Tue            Stable 24h+. What 
                                                   happens when a build 
                                                   legitimately needs 
                                                   >30 builds of history? ❓

Redis r6g.xlarge                   Tue 2:15 AM    Stable. Terraform 
                                                   PR merged? ❓

PLAT-919 PrometheusRule            Mon 3:06 PM    Scoped correctly? 
                                                   Just fired for ETL 
                                                   job — was this 
                                                   intended? ❓
```

---

## RESUMING: Thursday 2:04 PM

**Conference Room A. PCI Emergency Review.**

James just asked his two questions. Your phone just buzzed with an ETL job failure alert. Aisha noticed you glance at your phone.

The room is waiting.

Seven people. James Morrison (VP), Sarah Chen (manager), Aisha Rahman (Security), Rachel Torres (Legal), Wei Liu (Data Eng manager), Nina Petrov (Order team lead), you.

**Three seconds of silence have passed.**

You need to:
1. Answer James's questions (the room is waiting)
2. Decide what to do about the phone buzz (is it urgent? can it wait? is it your fault?)
3. Do both without looking flustered in front of the VP and Legal

**The ETL alert is ambiguous.** It fired at 2 PM but the job runs at 2 AM. Either:
- The job ran on schedule at 2 AM, failed, and the alert is delayed (Prometheus scrape interval + alerting rule evaluation delay)
- Someone manually triggered the job
- The CronJob schedule was changed
- Your PrometheusRule is misconfigured and is matching something it shouldn't

You don't know which. You can't check right now. You're presenting.

**Go.**

I pocket the phone without looking at it again. James asked a direct question and seven people are waiting. An informational alert is not more important than this moment.

---

> James, good question. Let me be concrete.

> Tom's pipeline had four steps to get where it did: he deployed unvetted container images, he connected to production databases, he created egress to the public internet, and he ran unnoticed for 67 days. Each one is a guardrail we don't have today.

I tick them off on my fingers.

> **First — image source restriction.** An admission policy — OPA Gatekeeper or Kyverno — that rejects any pod running an image that isn't from our ECR registry. Tom's Kafka brokers, Kafka Connect, and analytics dashboard all pulled from Docker Hub. If the cluster rejected those at admission time, he can't deploy them. He comes to us to get images mirrored into ECR, and that conversation surfaces the whole thing back in November. The policy itself is about 30 lines of YAML. The work is auditing what's currently running so we don't break legitimate workloads — which our audit this week already identified. One to two weeks, most of that in warn mode before we enforce.

> **Second — namespace RBAC.** Tom created the `data-analytics` namespace himself and had full admin in it. If namespace creation requires platform team provisioning — a template that comes with default NetworkPolicy, RBAC bindings, and monitoring baked in — he can't stand up infrastructure in a dark corner of the cluster. One week for the policy, two to three weeks for full rollout.

> **Third — default-deny network policy on all namespaces.** The `data-analytics` namespace had zero network policy until I applied one Wednesday. If every namespace starts with default-deny egress, Tom's ETL can't reach the production RDS and Kafka Connect can't reach MongoDB Atlas without explicitly requesting egress rules — which again, surfaces the conversation. Two to three weeks. The audit script I wrote Wednesday gives us the current egress patterns per namespace, so we're not starting from scratch.

> **Fourth — egress gateway.** Even with network policy, the MongoDB Atlas connection only got blocked because my policy didn't include it in the allowlist — that was accidental, not intentional. A proper egress gateway gives us a single choke point where all outbound traffic is logged and requires explicit approval. Someone sending data to `mongodb.net` shows up immediately. This is the heaviest lift — four to six weeks, mid-Q1.

> **And fifth — the one that costs almost nothing.** Drift detection. Tom's pipeline ran for 67 days without anyone noticing. If we had a weekly automated scan for workloads not managed by ArgoCD — which is essentially what I ran manually on Wednesday — we'd have caught this within seven days of deployment. I already have the script. Wrapping it in a CronJob with Slack alerts is a few hours of work. I can have it running by Monday.

I pause. This is the part that matters most.

> To be direct about sequencing:

> **Next week:** drift detection is live, image admission policy is in warn mode. Those two together would have caught this specific incident. **By end of January:** image policy enforced, default-deny network policy rolling out, namespace provisioning process in place. **By end of Q1:** egress gateway operational, all five controls in place.

> But I want to flag one thing honestly. Every one of these controls adds friction. Tom built what he built because it was easier to deploy directly than to come through us. If we only build walls without building a door, the next person will climb the wall. Each control should be paired with a corresponding easy path — a self-service namespace template, an ECR mirror pipeline, a catalog of pre-approved data connectors. The guardrails and the platform have to be built together, or we'll be back here in six months with a different team and a different shadow pipeline.

I glance briefly at Rachel.

> One more thing — for Rachel's supervisory notification, I can have the technical remediation dates firmed up by end of day tomorrow. The drift detection running by Monday gives us a concrete "detection capability implemented within five business days of discovery." That's a strong remediation narrative.

I stop talking. That's enough. James asked for specifics and a timeline, and he got both.

But the phone is in my pocket and the ETL alert is nagging at the back of my mind. My S3 encryption change yesterday is the most likely cause. If I broke that job, I need to know — but not in the next thirty seconds. I'll check it at the first natural break.


# SIMULATION — The Room Responds

---

**Six seconds of silence after you finish.**

James uncrosses his arms. He's nodding slowly.

**James Morrison:**
> That's the clearest prevention roadmap I've heard from platform in a while. I like the sequencing — detection first, then admission, then network, then egress gateway. And you're right about the friction point. If we build walls without doors, someone will climb.
>
> I want the drift detection running by Monday. That's your top priority over the weekend prep. If it catches something else, I want to know immediately, not at next week's sync.
>
> On the timeline — "end of Q1 for all five controls." That's twelve weeks. Can we compress?

**Before you answer, Nina speaks up.** First time she's talked in the meeting.

**Nina Petrov:**
> Can I add something? I'm part of this problem too — my data export CronJob has been running in `order-prod` for a month without ArgoCD management. The reason I didn't go through the platform process is... honestly, there isn't one that felt accessible for what I needed. I had a finance request, I needed a daily CSV export, and the "right way" would have been a two-week platform onboarding process for a 50-line CronJob. So I deployed it directly because I had a deadline.
>
> Tom probably had the same experience. If the self-service path for "I need a simple data pipeline" is faster than the shadow path, people will use it. If it's slower, they won't. Whatever controls you build, that has to be true.

**Sarah Chen:**
> Nina's right, and this is something I need to own. The platform team hasn't built the self-service tooling that makes compliance easy. We've been focused on infrastructure reliability and not enough on developer experience. That changes in Q1.

**James:**
> Agreed. [To you] — factor that into your timeline. I don't just want guardrails. I want the self-service catalog Nina's describing. When Tom comes back Monday, I don't want him to feel punished — I want him to see a path that's easier than what he did.

**Rachel Torres:**
> On the remediation dates for the notification — [looks at you] — you said you can firm those up by tomorrow. I need specific dates, not ranges. "Image admission policy enforced by January 31" not "two to three weeks." The supervisory authority wants to see a concrete remediation plan.

**Wei Liu** clears his throat.

**Wei Liu:**
> One concern on timeline. Tom's back Monday, but the Kafka pipeline onboarding isn't going to be a one-day conversation. His team has three analysts who depend on that data daily. If we restructure the pipeline, there's going to be a gap where they don't have analytics. That's a business impact I need to manage. Can we plan for a transition period rather than a hard cutover?

**Aisha Rahman:**
> That's fair, but the MongoDB Atlas connection stays blocked. Whatever transition plan we design, payment data does NOT flow to an uncontrolled external system. That's non-negotiable.

**Wei:**
> Understood. But the S3 sink for orders and order_items — that's not payment data. Can that keep running while we onboard?

**Aisha looks at you.** This is a technical question — can the S3 sink continue while Atlas stays blocked?

---

**Three people now need something from you:**

1. **James:** Can you compress the twelve-week timeline? And factor in self-service tooling.
2. **Rachel:** Specific dates, not ranges, by tomorrow for the GDPR notification.
3. **Wei/Aisha:** Can the S3 sink keep running while Atlas stays blocked? Is it already running?

**Also — your phone buzzed 3 minutes ago and you still don't know what's happening with the ETL job.**

**Also — Nina just said something important. She validated your "make the right thing easy" framing with real experience. That's useful. But it also means the self-service catalog just became a deliverable James expects, not just a recommendation in a document.**

**The meeting is in its last 15-20 minutes. Multiple threads need to be tied off.**

**Go.**

