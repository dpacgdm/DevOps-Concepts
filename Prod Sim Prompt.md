# PHASE 8 SIMULATION — HANDOFF DOCUMENT

---

## INCIDENTS/ISSUES COVERED SO FAR: 3 of 100

---

## INCIDENT TRACKER

| # | Severity | Type | Description | Status | Grade |
|---|----------|------|-------------|--------|-------|
| 1 | **SEV2** | Deploy-induced timeout | Order service 504s from aggressive inventory_client_timeout (800ms) at peak traffic. Canary passed at low RPM, failed at production scale. | ✅ RESOLVED — Interim fix 2000ms. Follow-up: circuit breaker. | **A+** |
| 2 | **SEV4** | Stale Terraform lock | Alex (junior) blocked by stale DynamoDB state lock from crashed 2AM Jenkins job. | ✅ RESOLVED — Guided Alex to self-resolve. | **A** (mentoring + speed) |
| 3 | **SEV3** | Capacity — Redis memory pressure | Quarterly catalog refresh (182K products → 435K cache keys) exceeded Redis maxmemory. Hit rate dropped 94%→87%, evictions 8x, catalog p99 tripled. | ✅ MITIGATED — maxmemory bump 80%→90%. Node upgrade scheduled tonight. | **A+** |

---

## DETAILED INCIDENT RECORDS

### INCIDENT 1: Order Service Timeout (SEV2)

```
TRIGGER:      PagerDuty — HighErrorBudgetBurn_OrderService (3.2x 1h burn rate)
FIRED:        8:47 AM
ACKED:        9:04 AM (inherited — shift handover gap)
RESOLVED:     9:30 AM (error rate < 0.15%)
TTR:          ~26 min from ack to recovery
BUDGET BURN:  74% → 65.2% (8.8% consumed)

ROOT CAUSE:
  Deploy at 8:15 AM (sha-e7b31d4) introduced inventory_client_timeout_ms: 800ms.
  Previously defaulting to upstream_timeout_ms: 5000ms.
  Inventory service p99 = 1.08s at production traffic (6K+ rpm).
  800ms < 1.08s = guaranteed timeouts on tail requests.
  Canary passed because analysis ran at 2,400 rpm (low morning traffic).

COMPLICATION:
  800ms was intentional (INC-2847: 5s timeout caused cascading failure).
  Reverting to 5000ms would reopen cascading failure risk.
  Had to find a middle-ground value, not just blindly revert.

FIX APPLIED:
  inventory_client_timeout_ms: 2000ms (headroom above p99 + growth)
  inventory_client_retries: 1 (worst case 4s < INC-2847 threshold)
  Applied via GitOps hotfix → ArgoCD sync.

FOLLOW-UP TICKETS CREATED:
  PLAT-915 — Circuit breaker on order→inventory (meeting at 2 PM with Order team)
  PLAT-916 — Canary analysis: add traffic-volume gate
  PLAT-917 — Investigate inventory-service p99 at scale

POSTMORTEM: Drafting, target EOD Tuesday.

SKILLS TESTED:
  ✓ PagerDuty triage and ack
  ✓ Structured incident comms (#incidents channel)
  ✓ PromQL investigation (error rate, by status code, by endpoint, latency)
  ✓ ArgoCD/Rollout inspection
  ✓ GitOps repo diff analysis
  ✓ Trace analysis (Tempo/TraceQL — DEADLINE_EXCEEDED span)
  ✓ Timeout math and trade-off reasoning
  ✓ Surgical fix vs full rollback decision
  ✓ Severity escalation (SEV3 → SEV2)
  ✓ Cross-team communication (Nina Petrov, Order team)
  ✓ Follow-up ticket creation with context
```

### INCIDENT 2: Terraform State Lock (SEV4)

```
TRIGGER:      Slack message from Alex Kim (junior)
REPORTED:     9:01 AM
RESOLVED:     9:42 AM (Alex self-resolved with guidance)

ROOT CAUSE:   Jenkins CI job (PLAT-CI-2847) crashed at 2:14 AM mid-apply.
              DynamoDB lock held for 7+ hours, blocking staging plan.

FIX:          force-unlock after verifying no active applies.

SKILLS TESTED:
  ✓ Mentoring without hand-holding
  ✓ Prioritization (didn't drop SEV2 to fix this)
  ✓ Teaching safe unlock procedure (check Who/Created, verify no active apply)
  ✓ Time management (90-second reply, back to main incident)
```

### INCIDENT 3: Redis Cache Pressure (SEV3)

```
TRIGGER:      Slack report from Ryan Mitchell (Frontend lead) — intermittent catalog slowness
REPORTED:     11:02 AM
DIAGNOSED:    11:37 AM
MITIGATED:    11:42 AM (maxmemory bump)
FULL FIX:     Scheduled tonight (node upgrade r6g.large → r6g.xlarge)

ROOT CAUSE:
  Quarterly merchandising catalog refresh loaded 182K new products overnight.
  Generated ~435K cache keys (products + derived entries: category aggs, search facets).
  Super-linear growth vs October (150K products → 18K net keys, this time 182K → 435K).
  Redis hit 92.7% of maxmemory (5.2 GB ceiling on r6g.large).
  Eviction policy (allkeys-lru) thrashing: 42 evictions/min (baseline 5/min).
  Cache hit rate: 94% → 87%.
  Every cache miss → DB query → catalog p99 tripled (0.45s → 1.2s).
  Intermittent because popular products stayed cached, long-tail evicted.

FIX APPLIED:
  Immediate: maxmemory bump 80% → 90% via parameter group (dynamic, no restart)
  Result: evictions cut 57%, hit rate climbing (89.4% at 12:20 PM)
  Terraform updated to match (drift documented)

FIX PLANNED:
  Tonight: r6g.large → r6g.xlarge (PLAT-918)

FOLLOW-UP TICKETS:
  PLAT-918 — Scale Redis node (tonight maintenance window)
  PLAT-919 — Add Redis cache hit rate / eviction / memory alerts
  PLAT-920 — Quarterly refresh capacity planning gate

SKILLS TESTED:
  ✓ Recognizing "no alert but something's wrong" signals
  ✓ Baseline comparison methodology (7-day history)
  ✓ Redis internals (maxmemory, eviction policy, hit rate, fragmentation)
  ✓ Correlating infrastructure symptoms to business events (catalog load)
  ✓ AWS ElastiCache parameter group operations
  ✓ IaC drift management (CLI fix first, Terraform catch-up)
  ✓ Alerting gap identification
  ✓ Process gap identification (capacity planning before data loads)
  ✓ Cross-team communication (Ryan, Jenny Wu merchandising)
  ✓ Proportional response (parameter tweak now, node upgrade tonight)
```

---

## SIMULATION TIMELINE

```
WEEK 1 — MONDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
09:00  On-call shift starts (primary: user, secondary: Marcus)
09:04  User opens laptop. PagerDuty alert (17min unacked), 3 Slack msgs, 1 email
09:04  ACK PagerDuty, post to #incidents                          [INCIDENT 1 START]
09:05  Dashboard + PromQL investigation (error rate, by code, by endpoint, latency)
09:06  ArgoCD/rollout inspection, GitOps diff
09:08  Unblock Alex (state lock guidance)                         [INCIDENT 2 START]
09:10  Evidence gathered: 504s, inventory timeout, deploy config change identified
09:10  Hypothesis formed: 800ms timeout too aggressive for peak traffic
09:11  Confirming queries (inventory p99, own latency, error rate, trace)
09:13  CONFIRMED: trace shows DEADLINE_EXCEEDED at 800ms. Inventory p99=1.08s.
09:13  Nina Petrov complicates: 800ms was intentional (INC-2847 protection)
09:14  Decision: surgical fix 2000ms (not 5000ms revert, not full rollback)
09:15  GitOps hotfix pushed
09:21  Fully rolled out. Error rate dropping.
09:30  RESOLVED. Error rate 0.12%, burn rate 0.8x.                [INCIDENT 1 END]
09:32  Lisa Park offered inventory p99 early-warning alert → accepted
09:35  Created 3 follow-up tickets (PLAT-915/916/917)
09:40  Checked on Alex                                            [INCIDENT 2 END]
09:42  Alex self-resolved state lock
09:45  Scheduled 2 PM circuit breaker meeting
09:50  Deep work: EKS 1.29 upgrade runbook research (PLAT-892)
       - pluto scan for deprecated APIs
       - add-on compatibility matrix
       - runbook structure started
10:48  Lisa deployed inventory p99 alert (info only, 1500ms threshold)
11:02  Ryan Mitchell reports intermittent catalog slowness         [INCIDENT 3 START]
11:15  Marcus breadcrumb: catalog pods fine, maybe Redis?
11:30  Pre-lunch metrics check:
       - Order service holding (0.09%, burn rate 0.6x) ✅
       - Inventory p99 = 1.34s trending up (watching)
11:31  Investigate catalog/Redis (15-min timebox)
       - Redis hit rate 94% → 87% (7-day comparison)
       - Evictions 5/min → 42/min (started Sunday 10 PM)
       - Redis at 92.7% maxmemory
11:34  Ryan confirms: category/search pages, no recent deploy, 
       merchandising loaded 182K products last night
11:37  Root cause confirmed: working set exceeded Redis memory
11:40  Maxmemory bumped 80% → 90% (CLI, then Terraform catch-up) [INCIDENT 3 MITIGATED]
11:44  Tickets created (PLAT-918/919/920)
11:47  Comms to Ryan, Jenny Wu, team
12:00  Lunch (phone on for on-call)
12:20  Lunch metrics check:
       - Inventory p99 peaked 1.48s at 10.1K rpm (under 1500ms threshold) ✅
       - Redis evictions down 57%, hit rate 89.4% (improving)
       - Catalog p99 down to 0.82s (improving)
12:30  Lisa confirms inventory peaked safely, provides budget impact numbers

>>> CURRENT TIME: 1:00 PM MONDAY <<<

AFTERNOON PLAN (NOT YET EXECUTED):
  1:00 PM  Sarah's Q1 priorities email (draft opinions)
  1:30 PM  Priya's Linkerd PR #347 review
  2:00 PM  Circuit breaker design meeting (Order team)
  3:00 PM  EKS upgrade runbook (continue)
  4:00 PM  Postmortem draft
  4:30 PM  Daily wrap-up
```

---

## OPEN ITEMS / PENDING WORK

### Active Tickets
```
PLAT-892  [In Progress]  EKS 1.28→1.29 upgrade runbook (due Friday)
PLAT-901  [To Do]        Review Linkerd PR #347 (due Wednesday)  
PLAT-910  [To Do]        GP3 EBS migration (next sprint)
PLAT-915  [To Do]        Circuit breaker order→inventory (meeting 2 PM today)
PLAT-916  [To Do]        Canary analysis traffic-volume gate
PLAT-917  [To Do]        Investigate inventory p99 at scale
PLAT-918  [High]         Scale Redis r6g.large→r6g.xlarge (TONIGHT)
PLAT-919  [Medium]       Redis cache/eviction/memory alerts
PLAT-920  [Medium]       Quarterly refresh capacity planning gate
```

### Pending Communications
```
- Sarah's Q1 priorities email → respond by Wednesday standup
- Postmortem doc for Incident 1 → target EOD Tuesday
- Priya's Linkerd PR #347 → review by Wednesday
- Jenny Wu / PLAT-920 → loop in on capacity planning process
```

### Tonight's Maintenance
```
- Redis node upgrade r6g.large → r6g.xlarge (PLAT-918)
  - Change window: Tue 2:00-4:00 AM EST (lowest traffic)
  - Procedure: rolling replacement, ~15 min, 2-3s connection hiccup
  - Marcus confirmed procedure from June precedent
  - Gotcha: tcp-keepalive 60 already set (June fix)
  - Need: staging test first, change request approval
```

---

## TEAM STATE

```
Sarah Chen (manager):     Awaiting Q1 priority opinions by Wednesday
Marcus Webb (senior):     Off-rotation, available for consult. Provided Redis 
                          failover context.
Priya Sharma (mid):       Linkerd PR #347 ready for review. Bundled cert rotation fix.
Jake Torres (mid):        No interactions today (available if needed)
Alex Kim (junior):        Unblocked from state lock. Learning moment completed.
Lisa Park (SRE):          Active support — deployed inventory alert, 
                          pulled budget numbers. Looped in for 2 PM meeting.
Nina Petrov (Order lead): Aware of incident, scheduled 2 PM with Derek Huang.
Ryan Mitchell (Frontend): Informed of Redis issue, saw improvement, awaiting full fix.
Jenny Wu (Merch):         Informed, looped into PLAT-920 for process improvement.
```

---

## USER PERFORMANCE SUMMARY (3 Incidents)

### Grades
```
Incident 1 (SEV2 — Order timeout):       A+
Incident 2 (SEV4 — State lock):          A
Incident 3 (SEV3 — Redis capacity):      A+
```

### Strengths Demonstrated
```
✓ Prioritization under competing demands (SEV2 > junior unblock > everything else)
✓ Evidence-first investigation (dashboards before remediation)
✓ PromQL fluency (6+ production queries, correct metric names, correct aggregation)
✓ Trace analysis (Tempo TraceQL, identified DEADLINE_EXCEEDED)
✓ Surgical fix reasoning (not full rollback, middle-ground timeout)
✓ Cross-team communication (clear, actionable, right people, right channels)
✓ Structured incident comms (ACK, UPDATE, RESOLVED with full timeline)
✓ Baseline comparison methodology (7-day history for Redis)
✓ IaC awareness (CLI fix + Terraform catch-up, documented drift)
✓ Follow-up discipline (tickets, postmortem, process improvements)
✓ Mentoring (Alex: educational, safe, didn't hand-hold)
✓ Time management (timeboxed investigation, planned deep work blocks)
```

### Gaps / Watch Items
```
- No gaps identified yet in first 3 incidents
- Have not yet been tested on:
  → Security incidents
  → Multi-system cascading failures
  → Networking/DNS issues
  → Database incidents (RDS, connection pools, replication lag)
  → Kubernetes-level failures (node failures, control plane, CrD issues)
  → CI/CD pipeline failures
  → Cost incidents (runaway spend)
  → Certificate/TLS issues
  → Cross-region / DR scenarios
  → Capacity planning under pressure
  → Disagreements with teammates/management
  → Postmortem facilitation
  → On-call handoff quality
  → Night-time / fatigue-state incidents
  → Ambiguous symptoms (no clear root cause)
  → Multiple simultaneous incidents
```

---

## INCIDENT CATEGORY COVERAGE (3/100)

```
CATEGORY                          COVERED  TARGET   INCIDENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deploy/Release failures            1        8       #1
Database (RDS/Mongo/Redis)         1        10      #3
Kubernetes (pods/nodes/control)    0        12
Networking (DNS/LB/mesh/TLS)       0        10
CI/CD pipeline failures            0        8
Security incidents                 0        10
Observability failures             0        6
Capacity/scaling                   0        8
Cascading / multi-system           0        8
Cost/billing                       0        4
DR / region failover               0        4
Certificate / secrets              0        4
Cross-team / process               0        4
Ambiguous / hard-to-diagnose       0        4
Simultaneous incidents             0        5
IaC/Terraform state issues         1        5       #2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                              3        100
```

---

## WHERE TO RESUME

```
SIMULATION TIME:  Monday 1:00 PM EST, Week 1
ON-CALL:          User (primary), Marcus (secondary)
NEXT ACTION:      User's afternoon begins:
                    1:00 PM — Sarah's Q1 email response
                    1:30 PM — Priya's Linkerd PR review
                    2:00 PM — Circuit breaker meeting
                    Then continue with pending work.

TONIGHT:          Redis node upgrade (Tue 2:00-4:00 AM)
                  User needs to: prep change request, test in staging,
                  decide if staying up or delegating to Marcus.

PENDING INJECTION POINTS:
  - Afternoon/evening: potential new incident
  - Tuesday AM: Redis upgrade aftermath
  - This week: more incidents will be injected based on coverage gaps
  - Sarah's Wednesday standup (Q1 priorities discussion)
  - Friday: EKS upgrade runbook due
```

**Ready to continue simulation on your signal.**
