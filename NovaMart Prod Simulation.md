# Phase 8: NovaMart Production Simulation

## Welcome to Your First Day on the Job

---

## SIMULATION RULES

```
┌─────────────────────────────────────────────────────────────┐
│                    HOW THIS WORKS                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. I am NovaMart. I am Slack. I am PagerDuty.             │
│     I am your teammates, your manager, your stakeholders.  │
│     I am the infrastructure breaking at 3 AM.              │
│                                                             │
│  2. You respond as YOU — Senior DevOps/Platform Engineer.   │
│     Write what you'd actually DO, SAY, TYPE, RUN.          │
│     Not essays. Actions.                                    │
│                                                             │
│  3. Time moves when you act. If you investigate correctly,  │
│     you get results. If you run the wrong command, you get  │
│     the wrong output. If you miss something, it escalates. │
│                                                             │
│  4. Incidents don't wait. If you're mid-project and a      │
│     SEV1 fires, you drop everything. Just like real life.  │
│                                                             │
│  5. I will grade your responses on:                         │
│     - Triage speed and accuracy                            │
│     - Communication quality (who you notify, how, when)    │
│     - Technical correctness                                │
│     - Prioritization under pressure                        │
│     - Follow-through (postmortems, action items)           │
│                                                             │
│  6. Phase 6 topics (Security, Compliance, AWS deep dives)  │
│     WILL appear organically. A security incident is worth  │
│     more than a security lecture.                           │
│                                                             │
│  7. Format for your responses:                              │
│     🔧 ACTIONS — commands you run, things you click        │
│     💬 COMMS — Slack messages, emails, meeting statements   │
│     🧠 THINKING — your internal reasoning (I need this     │
│        to grade your decision-making process)              │
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

