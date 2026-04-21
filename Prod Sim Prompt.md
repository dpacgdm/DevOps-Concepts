# PHASE 8 SIMULATION — HANDOFF DOCUMENT (v4)
# CHAOS ENGINE EDITION

---

## SIMULATION VERSION: v2 — CHAOS ENGINE ACTIVE

The simulation has transitioned from v1 (linear, clean feedback, golden path) to v2 (consequence branching, degraded tooling, red herrings, compounding failures, delayed grading). All 12 rules are now in effect.

---

## CHAOS ENGINE RULES (Active)

```
RULE 1:  CONSEQUENCE BRANCHING — Bad decisions compound, good 
         decisions sometimes have unintended side effects
RULE 2:  SECOND-ORDER CONSEQUENCES — Every fix you've applied 
         interacts with other live system changes
RULE 3:  RED HERRINGS AND CORRELATED NOISE — Multiple signals 
         fire during incidents, most irrelevant
RULE 4:  YOU WILL BE WRONG — At least once per arc, your 
         hypothesis will be incorrect or incomplete
RULE 5:  DEGRADED TOOLING — Commands fail, dashboards lag, 
         logs are noisy, traces are sampled
RULE 6:  NON-LINEAR TIME PRESSURE — Minutes 0-5 cheap, 5-10 
         moderate, 10-15 expensive, 15-20 very expensive, 
         20+ critical. Penalizes SILENCE not thoroughness.
RULE 7:  UNPREDICTABLE TEAMMATE VARIABILITY — People have 
         their own priorities, context, and bad days
RULE 8:  AMBIGUOUS SEVERITY — Incidents don't arrive pre-labeled
RULE 9:  POST-INCIDENT EXECUTION REALITY — Follow-up tickets 
         don't magically get done
RULE 10: THE HAUNTED FOREST — At least once, paged for an 
         unfamiliar system with stale/missing docs
RULE 11: DELAYED AND ABSENT GRADING — Not always told if you 
         were right. Recovery weighted as heavily as avoidance.
RULE 12: GENUINE STUCK POINTS — Moments where the obvious 
         next step doesn't exist
```

---

## INCIDENTS/ISSUES COVERED SO FAR: 6 of 100

*(Incident #6 is RESOLVED. No new incident number assigned yet, but an ETL alert just fired during the PCI meeting.)*

---

## INCIDENT TRACKER

| # | Severity | Type | Description | Status | Grade |
|---|----------|------|-------------|--------|-------|
| 1 | **SEV2** | Deploy-induced timeout | Order service 504s from aggressive inventory_client_timeout (800ms) at peak traffic. Canary passed at low RPM, failed at production scale. | ✅ RESOLVED — Interim fix 2000ms. Follow-up: circuit breaker. | **A+** |
| 2 | **SEV4** | Stale Terraform lock | Alex (junior) blocked by stale DynamoDB state lock from crashed 2AM Jenkins job. | ✅ RESOLVED — Guided Alex to self-resolve. | **A** |
| 3 | **SEV3** | Capacity — Redis memory pressure | Quarterly catalog refresh (182K products → 435K cache keys) exceeded Redis maxmemory. Hit rate dropped 94%→87%, evictions 8x, catalog p99 tripled. | ✅ RESOLVED — Node upgraded r6g.large→r6g.xlarge overnight. | **A+** (investigation), **A** (maintenance) |
| 4 | **SEV3** | Shadow infrastructure | Unmanaged Kafka cluster + full data pipeline. Scope expanded by Wed audit: Kafka Connect, ETL CronJob reading payments/refunds from orders DB, MongoDB Atlas external sink, S3 PII exports. | ✅ MITIGATED — NetworkPolicy v3, S3 hardened, Atlas blocked. PCI review in progress. | **A-** (initial) → **A+** (audit + PCI escalation) |
| 5 | **SEV2** | CI/CD — Zombie Jenkins agents | Jenkins agent pods stuck running 3-6+ hours, consuming entire Karpenter CI pool. Root cause: JVM memory leak. | ✅ RESOLVED — Build discard config + heap monitoring. | **[Grade from session]** |
| 6 | **SEV2** | Deploy-induced code bug | Payment service BigDecimal.divide() without RoundingMode. GBP/SEK/DKK/NOK→EUR 100% failing. Canary passed on USD-only sample. | ✅ RESOLVED — Rolled back, corrected fix deployed sha-c4d82e1. | **A+** |

---

## DETAILED INCIDENT RECORDS

### INCIDENT 1: Order Service Timeout (SEV2) ✅

```
TRIGGER:      PagerDuty — HighErrorBudgetBurn_OrderService (3.2x 1h burn rate)
FIRED:        Mon 8:47 AM
ACKED:        Mon 9:04 AM (inherited — shift handover gap)
RESOLVED:     Mon 9:30 AM (error rate < 0.15%)
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

FOLLOW-UP TICKETS:
  PLAT-915 — Circuit breaker on order→inventory
             Derek PR up for review (platform-infra#347)
  PLAT-916 — Canary analysis: traffic-volume gate + transaction-diversity gate
             EXPANDED after Incident 6. Lisa building AnalysisTemplate PR (ETA Fri).
  PLAT-917 — Investigate inventory-service p99 at scale (assigned to Nina)

CIRCUIT BREAKER MEETING OUTCOMES (Mon 2 PM):
  - Resilience4j CB with revised config:
    failureRateThreshold: 25, slowCallRateThreshold: 50,
    slowCallDurationThreshold: 1500ms, slidingWindowSize: 50
  - Linkerd ServiceProfile: isRetryable:false on POST /api/v1/orders
  - Per-replica CB state is FINE (no shared state needed)
  - Two-tier fallback design (high-confidence → accept, low-confidence → degraded)
  - Needs inventory stock level cache (doesn't exist yet)
  - Lisa updating SLI definition for degraded-mode responses

POSTMORTEM: Draft shared, waiting for feedback. Updated to reference 
            Incident 6 as companion canary gap.
```

### INCIDENT 2: Terraform State Lock (SEV4) ✅

```
TRIGGER:      Slack message from Alex Kim (junior)
REPORTED:     Mon 9:01 AM
RESOLVED:     Mon 9:42 AM (Alex self-resolved with guidance)

ROOT CAUSE:   Jenkins CI job (PLAT-CI-2847) crashed at 2:14 AM mid-apply.
              DynamoDB lock held for 7+ hours, blocking staging plan.

FIX:          force-unlock after verifying no active applies.
```

### INCIDENT 3: Redis Cache Pressure (SEV3) ✅

```
TRIGGER:      Slack report from Ryan Mitchell (Frontend lead)
REPORTED:     Mon 11:02 AM
DIAGNOSED:    Mon 11:37 AM
MITIGATED:    Mon 11:42 AM (maxmemory bump 80%→90%)
RESOLVED:     Tue 2:15 AM (node upgrade r6g.large→r6g.xlarge)
CONFIRMED:    Tue 8:55 AM (metrics at baseline)

ROOT CAUSE:
  Quarterly merchandising catalog refresh loaded 182K new products.
  Generated ~435K cache keys. Redis hit 92.7% maxmemory.
  Eviction policy thrashing, hit rate 94%→87%, catalog p99 tripled.

FIX TIMELINE:
  Mon 11:42 AM — maxmemory bump 80%→90%
  Mon 3:30 PM — Staging failover test (clean)
  Tue 2:00 AM — Production upgrade r6g.large→r6g.xlarge
  Tue 2:22 AM — Terraform updated (node type + maxmemory reverted 80%)

POST-UPGRADE STATUS (as of Thu):
  Hit rate: 94.2%
  Evictions: 0/min
  Memory: 53% of maxmemory

COST IMPACT: +$183/month

FOLLOW-UP TICKETS:
  PLAT-918 — Scale Redis (COMPLETED ✅)
  PLAT-919 — Redis cache alerting (COMPLETED ✅ — PrometheusRules deployed)
             ⚠️ NOTE: PLAT-919 PrometheusRule may be scoped too broadly.
             Just fired for an ETL job failure in data-analytics namespace.
             Needs investigation.
  PLAT-920 — Quarterly refresh capacity planning gate
```

### INCIDENT 4: Shadow Kafka / Data Pipeline (SEV3) ✅ Mitigated — PCI REVIEW ACTIVE

```
TRIGGER:      Jake Torres discovered unknown StatefulSets during GP3 migration
REPORTED:     Mon 2:38 PM
INVESTIGATED: Mon 2:48 PM → Wed full audit
MITIGATED:    Mon 3:15 PM (PDB + NetworkPolicy v1 → v2 → v3)
              Wed 12:15 PM (NetworkPolicy v3, S3 hardened, Atlas blocked)
FULL FIX:     Pending: PCI review (Thu 2 PM — IN PROGRESS), Tom return (Mon)

FULL PIPELINE SCOPE (discovered via cluster audit Wed):
  
  COMPONENT 1: Kafka brokers (6) + ZooKeeper (3)
    - Bitnami images from Docker Hub
    - Deployed Nov 9 by tom.chen@novamart.com via direct Helm
    - NOT in ArgoCD, NOT in Terraform, NOT meshed
    - EBS encrypted (AWS-managed key, not CMK)
    - PDB applied (Mon)

  COMPONENT 2: Kafka Connect (confluent cp-kafka-connect:7.5.3, Docker Hub)
    - Two connectors:
      a) orders-sink-s3: etl.orders + etl.order_items → S3 (analytics-datalake)
         STATUS: RUNNING (not blocked by NetworkPolicy)
      b) payments-sink-mongo: etl.payments + etl.refunds → MongoDB Atlas
         STATUS: FAILED (blocked by NetworkPolicy v3 egress deny — accidental)
         Kafka Connect buffering payment data internally

  COMPONENT 3: ETL CronJob (etl-daily-aggregation, daily 2 AM)
    - Reads from production orders RDS (analytics_reader user)
    - Tables: orders, order_items, payments, refunds
    - Writes to Kafka topics: etl.orders, etl.order_items, etl.payments, etl.refunds
    - STATUS: RUNNING (per Aisha — don't suspend)
    - ⚠️ JUST FIRED A FAILURE ALERT (Thu 2:04 PM) — cause unknown

  COMPONENT 4: Analytics Dashboard
    - novamart-analytics-dashboard:latest (Docker Hub, NOT ECR)
    - No pull secrets, no vulnerability scanning
    - Direct connection to production orders RDS
    - Connects on port 3000

  COMPONENT 5: MongoDB Atlas (EXTERNAL) — 🔴 CRITICAL
    - Free-tier personal account (tom_chen_dev, admin privileges)
    - Connection: mongodb+srv://analytics:****@novamart-analytics.xxxxx.mongodb.net
    - Database: payment_analytics, Collection: transactions
    - 42,847 documents, ~25-30K unique EU customers, Nov 9 → Jan 15 (67 days)
    - Fields present:
      order_id, payment_id, timestamp, amount, currency_from/to,
      converted_amount, payment_method, card_last_four, card_brand,
      customer_email, customer_name, billing_country, status,
      refund_amount, refund_reason
    - NO full PAN, NO CVV, NO expiry
    - card_last_four + card_brand = NOT CHD under PCI DSS 3.4
    - customer_email + customer_name + billing_country + transaction 
      history = PERSONAL DATA under GDPR Art. 4(1)
    - Free-tier Atlas characteristics:
      ✗ No encryption at rest (only on M10+ dedicated clusters)
      ✗ No VPC peering (data traversed public internet)
      ✗ No audit logging
      ✗ Shared/multi-tenant infrastructure
      ✗ 512 MB storage limit
      ✗ No IP whitelist (free tier doesn't support it)
    - Atlas password stored in Kafka connect-configs topic (plaintext, no ACLs)
    - One user: tom_chen_dev (admin privileges)
    - IT confirmed: NOT a corporate Atlas account
    - Tom confirmed: personal free-tier for "prototyping"
    - STATUS: BLOCKED (NetworkPolicy egress deny since Wed 12:15 PM)

  COMPONENT 6: Order Data Export CronJob (order-prod namespace)
    - Deployed by Nina Petrov (Dec 20) for finance team request
    - Exports: customers, shipping_addresses → S3 (novamart-data-exports)
    - PII at rest (customer names, shipping addresses in CSV)
    - 27 days of exports, 108 objects, 33 MiB
    - Nina informed by Sarah, willing to hand over

KAFKA TOPICS:
  connect-configs, connect-offsets, connect-status (Kafka Connect internal)
  etl.orders, etl.order_items, etl.payments, etl.refunds (data topics)
  __consumer_offsets (Kafka internal)

S3 BUCKETS (both unmanaged, NOT in Terraform):
  novamart-data-exports:
    ✅ Public access blocked
    ✅ Encryption upgraded SSE-S3 → KMS/CMK (108 objects re-encrypted)
    ✅ Server access logging enabled
    ✅ Versioning enabled
    ✅ Bucket policy scoped (two IAM roles)
    ❌ No lifecycle/retention policy (deferred to PCI review)
    ❌ Not in Terraform
    
  novamart-analytics-datalake:
    ✅ Public access blocked
    ✅ Encryption upgraded SSE-S3 → KMS/CMK (1,847 objects, 2 GB re-encrypted)
    ✅ Server access logging enabled
    ✅ Versioning enabled
    ❌ No lifecycle policy
    ❌ Not in Terraform

KMS KEY:
  Alias: alias/novamart-data-exports
  Key policy: platform-admin (full), order-data-export-role (encrypt/generate),
              data-analytics-role (decrypt)
  ⚠️ UNVERIFIED: Does ETL CronJob IAM role have kms:GenerateDataKey?
  ⚠️ UNVERIFIED: Does Kafka Connect S3 sink IAM role have kms:GenerateDataKey?

CONTAINMENT STATUS:
  ✅ Payment → Atlas: BLOCKED (NetworkPolicy egress deny)
  ✅ S3 exports: ENCRYPTED (KMS/CMK, logging, versioning)
  ✅ Kafka network: RESTRICTED (NetworkPolicy v3)
  ✅ ZK 2181: intra-namespace only
  ✅ Kafka Connect 8083: intra-namespace only
  ✅ Analytics Dashboard 3000: intra-namespace only
  ⚠️ ETL CronJob: RUNNING (per Aisha) — but just failed (Thu 2:04 PM alert)
  ⚠️ Kafka Connect S3 sink: RUNNING (orders/order_items)
  ⚠️ RDS egress: ipBlock CIDR-based (fragile on Multi-AZ failover)

PCI ASSESSMENT (from findings document):
  - PCI DSS Req 12.10 incident response: LIKELY NOT TRIGGERED (no full PAN)
  - QSA proactive disclosure recommended at next assessment
  - GDPR Art. 33: Notification to supervisory authority within 72 hours
    of becoming aware. Rachel Torres (Legal) recommends notifying 
    proactively. 72-hour clock started Wed night (~8:30 PM) when 
    Aisha confirmed free-tier details.
    → DEADLINE: Saturday evening (~8:30 PM)
  - GDPR Art. 34: Communication to data subjects — Rachel recommends
    prepare notification but hold pending supervisory authority guidance
  - Rachel drafting supervisory notification Thursday

PCI EMERGENCY REVIEW MEETING — IN PROGRESS (Thu 2:00 PM):
  Attendees: James Morrison (VP), Sarah Chen, Aisha Rahman (Security),
             Rachel Torres (Legal), Wei Liu (Data Eng mgr), 
             Nina Petrov (Order lead), User (Platform)
  
  COMPLETED:
    - User presented technical findings (5 min)
    - Wei presented Atlas data field inventory (3 min)
    - Rachel gave preliminary Legal assessment (PCI/GDPR)
    - Rachel recommended supervisory notification by Saturday
    - James agreed: "Let's do the supervisory notification"
  
  ACTIVE:
    - James asked two questions:
      1. "What concrete technical controls would have prevented this?"
      2. "How fast can we get those controls in place?"
    - User's phone buzzed: ETL job failure alert (2:04 PM)
    - User has NOT YET responded to James's questions
    
  STATUS: User standing in front of room, phone just buzzed,
          three seconds of silence, needs to respond NOW.

FOLLOW-UP TICKETS:
  PLAT-921 — Full platform onboarding (Tom back Monday)
  PLAT-922 — PCI scope assessment (EXPANDED — Atlas, S3 PII, 
             credential exposure, CDE boundary violation, GDPR)

FINDINGS DOCUMENT:
  Title: "PCI Scope Assessment — Unmanaged Data Pipeline"
  Status: COMPLETE (100%), distributed as pre-read at 12:50 PM Thu
  Audience: James, Sarah, Aisha, Rachel, Wei, Nina
  Key sections: Executive Summary, Discovery Timeline, Data Flow Diagram,
                Findings by Tier, Containment Actions, Open Questions,
                Remediation Roadmap, Recommendations
  Tone: "Process gap, not people failure. Make the right thing the easy thing."

GAPS IN USER'S HANDLING:
  - Initial NetworkPolicy (Mon) exposed ZK port (fixed Tue 1:50 AM)
  - Didn't check EBS encryption immediately (fixed Tue 1:50 AM)
  - MongoDB Atlas block was accidental (honest about it)
  - NetworkPolicy v3 RDS egress uses ipBlock CIDR (fragile)
  - Should have stated S3 public check as condition for suspend/contain decision
  - KMS key permissions for ETL/Kafka Connect writers NOT verified
  - PLAT-919 PrometheusRule may be scoped too broadly
  - Derek ServiceProfile PR: didn't clarify velocity freeze exemption
```

### INCIDENT 5: Zombie Jenkins Agents / CI Pool Exhaustion (SEV2) ✅

```
TRIGGER:      Payment team (David Okafor) reports builds failing since 7 AM
REPORTED:     Tue 7:45 AM (David), 8:20 AM (Alex investigation)
RESOLVED:     Tuesday

ROOT CAUSE:   JVM memory leak in Jenkins agent pods.

FIX APPLIED:  Build discard configuration + heap monitoring.

STATUS (as of Thu):
  Jenkins heap stable. 6 agent pods running (normal).
  ⚠️ UNVERIFIED: What happens when a build legitimately needs 
  >30 builds of history? Build discard may be too aggressive.
```

### INCIDENT 6: Payment Service Currency Bug (SEV2) ✅

```
TRIGGER:      PagerDuty — PaymentServiceHighErrorRate (>2% for 5 min)
FIRED:        Wed 2:47 PM
ACKED:        Wed 2:48 PM
ROOT CAUSE:   Wed 2:52 PM (~4 min from ack)
ROLLBACK:     Wed 2:53 PM (initiated), 2:58 PM (complete)
BASELINE:     Wed 3:01 PM (error rate 0.12%)
FIX DEPLOYED: Wed 4:28 PM (sha-c4d82e1, all 14 currency pairs verified)
WATCH CLEAN:  Wed 4:43 PM (15-min window, zero errors)
TTR:          ~13 min ack-to-baseline (rollback), ~2 hours total (corrected fix)
BUDGET BURN:  91.3% → 87.1% (4.2% consumed)

BLAST RADIUS:
  847 failed payment authorizations
  89 failed refunds
  34 failed status checks
  Estimated 500-700 unique customers unable to complete purchases
  CS team sent retry emails to affected customers

ROOT CAUSE:
  Currency rounding hotfix (sha-f91a2b7) introduced BigDecimal.divide() 
  without RoundingMode in CurrencyService.convertAmount() (line 47).
  Non-terminating decimal currency pairs (GBP→EUR, SEK→EUR, DKK→EUR, 
  NOK→EUR) throw ArithmeticException. USD transactions unaffected.

CANARY GAP (SAME CLASS AS INCIDENT 1):
  - 5% traffic for 5 minutes, USD-dominated sample
  - No currency-pair diversity gate in canary analysis
  - Second incident from canary sample bias
    (Incident 1: volume bias, Incident 6: diversity bias)

CORRECTED FIX (sha-c4d82e1):
  - Three BigDecimal.divide() call sites: RoundingMode.HALF_EVEN, scale 2
    1. convertAmount() — the breaking change
    2. calculateRefundAmount() — pre-existing, worked by accident
    3. applyDiscount() — pre-existing, PR review caught scale inconsistency
       (returned scale 10 without .setScale(2) — user caught this in review)
  - 36 new tests covering all 14 EU currency pairs + edge cases
  - Finance confirmed HALF_EVEN (banker's rounding) for EU compliance

DEPLOYMENT VERIFICATION:
  - Manual code review (caught applyDiscount scale bug)
  - Manual synthetic canary: all 14 pairs, refund path, discount path
  - 15-minute watch window at 100% — zero errors
  - Production logs confirmed GBP/SEK/DKK/NOK transacting correctly

VELOCITY FREEZE: Payment-service error budget blown (87.1%). 
  Freeze in effect. Exception granted for corrective deploy only.

⚠️ UNVERIFIED DOWNSTREAM: HALF_EVEN vs historical HALF_UP rounding
  — potential reconciliation discrepancy on transactions calculated 
  with different rounding methods. Finance may notice.

FOLLOW-UP:
  PLAT-916 — EXPANDED (volume gate + diversity gate)
  Derek ServiceProfile PR merged (isRetryable:false on all mutating 
    payment endpoints)
  Lisa building canary AnalysisTemplate with currency-pair diversity metric
```

---

## SIMULATION TIMELINE

```
WEEK 1 — MONDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
09:00  On-call shift starts
09:04  ACK PagerDuty, post to #incidents                     [INC-1 START]
09:05  Dashboard + PromQL investigation
09:06  ArgoCD/rollout inspection, GitOps diff
09:08  Unblock Alex (state lock guidance)                    [INC-2 START]
09:10  Hypothesis: 800ms timeout too aggressive
09:13  CONFIRMED via trace (DEADLINE_EXCEEDED at 800ms)
09:14  Decision: surgical fix 2000ms
09:15  GitOps hotfix pushed
09:30  RESOLVED                                              [INC-1 END]
09:42  Alex self-resolved                                    [INC-2 END]
09:45  Scheduled CB meeting
09:50  EKS 1.29 upgrade runbook research
11:02  Ryan reports catalog slowness                         [INC-3 START]
11:37  Root cause: Redis memory from catalog refresh
11:40  Maxmemory bumped 80%→90%                              [INC-3 MITIGATED]
12:00  Lunch
13:00  Sarah Q1 priorities response
13:20  Priya Linkerd PR #347 review
14:00  Circuit breaker design meeting
14:48  Kafka investigation begins                            [INC-4 START]
15:06  Redis/catalog alerts deployed (PLAT-919)
15:15  PDB + NetworkPolicy applied for Kafka                 [INC-4 MITIGATED]
15:30  Redis staging failover test
16:15  EOD notes

MONDAY NIGHT / TUESDAY EARLY AM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
01:50  Fix Kafka NetworkPolicy v2 (ZK port bug). Check EBS encryption.
02:00  Redis upgrade                                         [INC-3 FINAL FIX]
02:22  Terraform updated
02:30  Back to sleep

TUESDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
07:45  David reports payment builds failing                  [INC-5 START]
~AM    Jenkins zombie pods investigated, resolved            [INC-5 RESOLVED]
       Build discard config + heap monitoring

WEDNESDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
10:00  Team standup (James VP present)                       [STANDUP — A+]
       PCI insertion, risk-first framing, Kafka governance
10:35  Aisha security response, audit coordination
10:45  Cluster audit begins (scripted sweep)
11:45  Audit results: 9 unmanaged workloads, 4 PCI critical
12:00  Deep investigation (Atlas, S3, connectors, topics)
12:15  NetworkPolicy v3 applied. Atlas egress blocked (accidental).
12:15  Communications: Aisha, Sarah, Derek, Marcus
12:35  Sarah: "More serious than expected." Findings EOD Thu.
12:42  Aisha: Atlas is CDE violation. PCI review Thu 2 PM.
12:55  Atlas details sent to Aisha
13:00  S3 remediation (KMS, logging, versioning, re-encryption)
13:30  Sarah: Wei joining Thu meeting. Nina informed.
14:00  PCI findings document ~40% drafted
14:47  PagerDuty: PaymentServiceHighErrorRate                [INC-6 START]
14:48  ACK. Investigation.
14:52  Root cause: BigDecimal.divide() without RoundingMode
14:53  ROLLBACK initiated
14:58  Rollback complete
15:01  Baseline restored
15:10  David: fix ready ~3:30 PM. Three pending responses.
15:17  Alex deploy freeze response (targeted, not blanket)
15:20  Lisa budget exception + canary diversity alert
15:22  David: scope all three BigDecimal fixes in one PR
15:24  Derek: isRetryable:false on all mutating payment endpoints
15:45  Review Priya K's PR (caught applyDiscount scale bug)
15:59  Fix pushed, re-approved
16:08  CI build complete (sha-c4d82e1)
16:09  Canary at 5%
16:20  Manual synthetic verification (14 pairs + refund + discount) ✅
16:21  Promoted to 100%
16:28  Fully rolled out
16:43  15-min watch window clean                             [INC-6 RESOLVED]
16:46  #incidents RESOLVED post
16:48  Ryan Mitchell Redis confirmation (overdue)
16:50  Sarah: EKS runbook at risk, proposed Mon extension
16:52  Derek ServiceProfile PR approved
16:55  PCI findings document deep work (40% → 75%)
18:00  EOD notes
18:10  Off

WEDNESDAY NIGHT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
20:30  Aisha DM: Atlas is free-tier. No encryption at rest. Public 
       internet. Multi-tenant. Potential GDPR notification obligations.
       Rachel Torres (Legal) joining Thu meeting.
23:45  Wei got Atlas credentials, logging in to assess

THURSDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
08:30  User logs in. Overnight metrics clean.
08:33  Aisha DM: updated document approach (no "breach" in title,
       "potential notification obligations" language, Legal in meeting)
08:38  Priya: namespace ordering for dry-run (adjusted: staging before CI)
08:40  Derek: CB PR up for review
08:42  James email: wants exposure, notification, remediation, prevention.
       "Prepare as if I'll be there."
08:45  PCI document final push (75% → 100%)
       Restructured: "audit report" → "executive decision support document"
       Title: "PCI Scope Assessment — Unmanaged Data Pipeline"
       Added conditional PCI vs GDPR framing
       Added "Open Questions for This Meeting" as agenda
09:50  Pre-dry-run verification
10:00  Linkerd staging dry-run with Priya                    [DRY-RUN]
       linkerd-viz: clean (45s, zero errors) ✅
       monitoring: clean (40s, zero errors) ✅
       staging: clean (35s, zero errors) ✅
       ci: clean (52s, Jenkins queued 2 builds) ✅
       Production namespaces NOT touched (scheduled for upgrade window)
11:05  Priya: draft production upgrade runbook from today's results
11:10  Back to PCI document final polish
11:30  Wei DM: Atlas data fields. No full PAN (good). But customer_email,
       customer_name, billing_country, card_last_four, card_brand,
       42,847 documents, ~25-30K unique EU customers
11:33  Wei follow-up: dedup question, Atlas access/API key question
11:40  Document Section 4.1 updated with field inventory
12:15  Document complete (100%)
12:45  Lunch + final doc review
12:50  Pre-read distributed to all attendees with section routing
13:10  Aisha: document structure approved
13:15  Sarah: James confirmed attending
14:00  PCI Emergency Review Meeting begins
       - User presents technical findings (5 min)
       - Wei presents Atlas data (3 min)
       - Rachel: PCI likely not triggered, GDPR notification recommended
       - James agrees on supervisory notification
14:04  James asks two questions (prevention + timeline)
14:04  User's phone buzzes: ETL job failure alert

>>> CURRENT TIME: Thursday 2:04 PM <<<
>>> PCI MEETING IN PROGRESS <<<
>>> JAMES WAITING FOR ANSWER <<<
>>> ETL ALERT ON PHONE <<<
>>> THREE SECONDS OF SILENCE <<<
```

---

## LIVE SYSTEM CHANGES — CONSEQUENCE TRACKING

```
⚠️ CHAOS ENGINE: These are your previous actions that may have 
downstream effects. Unverified items are potential failure points.

CHANGE                              APPLIED         DOWNSTREAM VERIFIED?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NetworkPolicy v3                    Wed 12:15 PM    PARTIALLY
  (data-analytics namespace)                        ✅ Kafka 9092 ingress tested
                                                    ✅ S3 sink working
                                                    ✅ Atlas blocked (accidental)
                                                    ❓ RDS egress (CIDR-based, fragile)
                                                    ❓ Other egress paths?
                                                    ❓ Analytics dashboard ext access?

KMS key creation                    Wed ~1:00 PM    PARTIALLY
  (alias/novamart-data-exports)                     ✅ Key created, policy scoped
                                                    ❓ ETL IAM role: kms:GenerateDataKey?
                                                    ❓ Kafka Connect S3 sink IAM role?
                                                    ❓ order-data-export IAM role?

S3 re-encryption                    Wed ~1:00 PM    PARTIALLY
  (both buckets, SSE-S3 → KMS)                     ✅ Existing objects re-encrypted
                                                    ❓ NEW objects by writers w/o KMS perms?
                                                    ❓ ETL → S3 path still works?

S3 versioning enabled               Wed ~1:00 PM   ❓ Cost impact on daily writes?
                                                    (minimal but untracked)

S3 access logging enabled           Wed ~1:00 PM   ❓ Log delivery verified?
                                                    ✅ Logging bucket exists

Payment service sha-c4d82e1         Wed 4:28 PM    ✅ Running stable
                                                    ❓ HALF_EVEN vs HALF_UP historical
                                                      reconciliation impact?

Jenkins build discard               Tue             ✅ Stable 24h+
                                                    ❓ Aggressive discard limit?

Redis r6g.xlarge                    Tue 2:15 AM     ✅ Stable
                                                    ❓ Terraform PR merged?

PLAT-919 PrometheusRule             Mon 3:06 PM     ❓ SCOPED CORRECTLY?
                                                    Just fired for ETL job failure
                                                    in data-analytics namespace.
                                                    Intended for Redis alerting only?

Derek ServiceProfile PR             Wed evening     ✅ Merged
                                                    ❓ Velocity freeze exemption 
                                                      not explicitly documented
```

---

## OPEN ITEMS / PENDING WORK

### Active Tickets

```
PLAT-892  [In Progress]  EKS 1.28→1.29 upgrade runbook (moved to Monday per Sarah)
                         Research done. Draft needs polish.
PLAT-901  [In Review]   Linkerd PR #347 — needs re-review after Priya's updates
PLAT-910  [To Do]       GP3 EBS migration (next sprint, skip data-analytics ns)
PLAT-915  [In Review]   Circuit breaker for order→inventory
                         Derek's PR up: platform-infra#347. Review after PCI meeting.
PLAT-916  [To Do]       Canary analysis improvements — EXPANDED SCOPE
                         Volume gate + diversity gate + synthetic canary tests
                         Lisa building AnalysisTemplate PR (ETA Fri)
PLAT-917  [To Do]       Investigate inventory p99 at scale (assigned to Nina)
PLAT-918  [Done ✅]     Redis scale-up
PLAT-919  [Done ✅]     Redis cache alerting — ⚠️ MAY BE MISCONFIGURED
                         Just fired for ETL job in wrong namespace
PLAT-920  [To Do]       Quarterly refresh capacity planning gate
PLAT-921  [High]        Kafka cluster platform onboarding (Tom back Mon)
                         Scope dramatically expanded by audit
PLAT-922  [Critical]    PCI scope assessment — EXPANDED
                         PCI review in progress. GDPR notification by Saturday.
[NEEDED]  [New]         Payment canary: currency-pair diversity gate
[NEEDED]  [New]         Audit all BigDecimal.divide() in payment codebase
[NEEDED]  [New]         Migrate secrets to External Secrets Operator
[NEEDED]  [New]         Add both S3 buckets to Terraform
[NEEDED]  [New]         S3 lifecycle/retention policies
[NEEDED]  [New]         DB user (analytics_reader) permission audit
[NEEDED]  [New]         Verify KMS permissions for all S3 writers
[NEEDED]  [New]         Verify PLAT-919 PrometheusRule scope
```

### Pending Deliverables

```
- PCI findings document → COMPLETE ✅, distributed 12:50 PM Thu
- EKS upgrade runbook → moved to Monday (Sarah approved)
- Linkerd production upgrade runbook → Priya drafting from dry-run results
- Derek CB PR review → after PCI meeting (platform-infra#347)
- Redis Terraform PR → needs team review/merge
- GDPR supervisory notification → Rachel drafting (deadline: Saturday ~8:30 PM)
- Tom Chen Kafka onboarding → Monday meeting
- Production Linkerd upgrade → next Tue 2 AM maintenance window (tentative)
```

---

## TEAM STATE

```
James Morrison (VP Eng):  IN THE ROOM. Asked two questions. Waiting 
                          for answer. Approved GDPR notification. 
                          "Good work catching this before auditors did."

Sarah Chen (manager):     IN THE ROOM. Approved EKS runbook extension 
                          to Monday. Wants prevention plan with timeline.

Aisha Rahman (Security):  IN THE ROOM. Leading PCI response. Noticed 
                          user glanced at phone. Concurred with QSA 
                          disclosure. Instructed: keep Atlas blocked.

Rachel Torres (Legal):    IN THE ROOM. Drafting GDPR Art. 33 
                          supervisory notification. Needs exact timeline,
                          containment actions, remediation plan with dates.

Wei Liu (Data Eng mgr):   IN THE ROOM. Presented Atlas data. Tom's 
                          manager. Tom is "horrified and wants to help."
                          Wasn't aware of Atlas externalization.

Nina Petrov (Order lead): IN THE ROOM. Trying to be invisible. 
                          Apologetic about data export. Willing to 
                          hand over. Joined meeting for completeness.

Marcus Webb (secondary):  Not in meeting. Adding node-problem-detector 
                          to ArgoCD. Available for consult.

Priya Sharma (mid):       Not in meeting. Dry-run complete. Drafting 
                          production Linkerd upgrade runbook. Next 
                          action: production upgrade next week.

Jake Torres (mid):        Not in meeting. Working GP3 migration. 
                          Warned to skip data-analytics.

Alex Kim (junior):        Not in meeting. Growing. Asked about deploy 
                          freeze (good instinct, wrong solution — 
                          user corrected constructively).

Lisa Park (SRE):          Not in meeting. Building canary diversity 
                          AnalysisTemplate. Budget tracking: payment 
                          87.1%, order 64.5%. Velocity freeze enforced.

David Okafor (Payment):   Not in meeting. Hotfix deployed successfully. 
                          CS team handling affected customers.

Derek Huang (Order):      Not in meeting. CB PR up for review. 
                          ServiceProfile merged. Staging cleanup 
                          (may not be done yet).

Ryan Mitchell (Frontend): Not in meeting. Redis confirmation received.

Jenny Wu (Merch):         Looped into PLAT-920. Not in meeting.

Aisha Rahman (Security):  (Also listed above — in meeting.)

Tom Chen (Data Eng):      On PTO until Monday. Knows about the issue 
                          (Wei reached him). "Horrified and wants to 
                          help fix it." Sent Wei Atlas credentials.

Priya Kapoor (Payment):   Not in meeting. BigDecimal fix deployed. 
                          Filed PAYMENT-847 for negative amount 
                          input validation.
```

---

## USER PERFORMANCE SUMMARY

### Grades (v1 simulation — pre-Chaos Engine)

```
Incident 1 (SEV2 — Order timeout):           A+
Incident 2 (SEV4 — State lock):              A
Incident 3 (SEV3 — Redis capacity):          A+ / A
Incident 4 (SEV3 — Shadow Kafka/Pipeline):   A- → A+
Incident 5 (SEV2 — Zombie CI pods):          [Grade from session]
Incident 6 (SEV2 — Payment currency bug):    A+

Standup Presentation:       A+
Cluster Audit:              A+
PCI Escalation:             A+
PR Reviews:                 A+
Circuit Breaker Meeting:    A+
Maintenance Window:         A
```

### v2 Grading Note

```
Under Chaos Engine rules (RULE 11), grades may be:
- Delayed (you find out later if you were right)
- Absent (you never find out — you live with uncertainty)
- Recovery-weighted (getting it right on second try after 
  recognizing you were wrong is valued)

The clean A+ streak from v1 is expected to get messier.
That's the point.
```

### Strengths (Demonstrated Through v1)

```
✓ Prioritization under competing demands
✓ Evidence-first investigation
✓ PromQL fluency
✓ Trace analysis
✓ Surgical fix reasoning
✓ Cross-team communication
✓ Structured incident comms
✓ IaC drift management
✓ Follow-up discipline
✓ Mentoring
✓ Time management
✓ PR review depth
✓ Technical meeting leadership
✓ Distributed systems reasoning
✓ Change management
✓ Upward communication
✓ PCI/GDPR compliance awareness
✓ kubectl forensics
✓ Operational discipline
✓ Self-correction
✓ Executive communication
✓ Political awareness
✓ Audit methodology
✓ Credential exposure chain analysis
✓ S3 security hardening
✓ Cross-language debugging (Java from platform perspective)
✓ Financial domain knowledge
✓ Rollback trade-off calculus
✓ Fast decisive action under pressure
✓ Systemic pattern recognition
✓ Influence without authority
```

### Gaps / Watch Items (Entering v2)

```
PATTERN: Correct instincts, occasionally imprecise execution under pressure.

SPECIFIC:
- NetworkPolicy precision (ZK port, Atlas accidental block)
- EBS encryption check deferred
- KMS key permissions for downstream writers NOT verified
- PLAT-919 PrometheusRule scope possibly too broad
- RDS egress CIDR fragility (documented, not fixed)
- S3 lifecycle: started applying then stopped (right call, but 
  instinct was to make destructive change without confirming)
- Derek ServiceProfile velocity freeze exemption not clarified
- HALF_EVEN vs HALF_UP historical reconciliation not flagged to finance
- Overcommitted schedule without acknowledging trade-offs (improved)
- Alert silencing without verifying higher-tier alert exists (improved)

V2 WILL TEST:
- Recovery from being wrong
- Signal extraction from noise
- Handling genuine "stuck" moments
- Navigation of unfamiliar systems
- Consequence management from own fixes
- Decision-making with incomplete/unknowable information
```

---

## INCIDENT CATEGORY COVERAGE (6/100)

```
CATEGORY                          COVERED  TARGET   INCIDENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deploy/Release failures            2        8       #1, #6
Database (RDS/Mongo/Redis)         1        10      #3
Kubernetes (pods/nodes/control)    0        12
Networking (DNS/LB/mesh/TLS)       0        10
CI/CD pipeline failures            1        8       #5
Security incidents                 0        10
Observability failures             0        6
Capacity/scaling                   1        8       #3
Cascading / multi-system           0        8
Cost/billing                       0        4
DR / region failover               0        4
Certificate / secrets              0        4
Cross-team / process               1        4       #4
Ambiguous / hard-to-diagnose       0        4
Simultaneous incidents             0        5
IaC/Terraform state issues         1        5       #2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                              6        100
```

---

## WHERE TO RESUME

```
SIMULATION TIME:  Thursday 2:04 PM EST, Week 1
SIMULATION MODE:  v2 — CHAOS ENGINE ACTIVE (all 12 rules in effect)
ON-CALL:          User (primary), Marcus (secondary)

ACTIVE SITUATION:
  PCI Emergency Review Meeting in progress.
  James Morrison (VP) just asked two questions:
    1. "What concrete technical controls would have prevented this?"
    2. "How fast can we get those controls in place?"
  
  User's phone buzzed: ETL job failure alert (2:04 PM)
    - Alert from PLAT-919 PrometheusRule (originally for Redis)
    - ETL CronJob runs at 2 AM, not 2 PM — timing is wrong
    - Could be: delayed alert, manual trigger, misscoped rule, 
      or consequence of user's own changes (KMS? NetworkPolicy?)
    - Aisha noticed user glance at phone
  
  User has not yet responded. Three seconds of silence.
  Seven people waiting.

IMMEDIATE DECISIONS NEEDED:
  1. Answer James's questions (room is waiting)
  2. Handle the phone alert (urgent? your fault? can it wait?)
  3. Navigate both without looking flustered

UNRESOLVED CONSEQUENCES TO TRACK:
  - KMS re-encryption may have broken ETL S3 writes
  - PLAT-919 may be misconfigured (too broad)
  - NetworkPolicy v3 egress may have blocked something unexpected
  - HALF_EVEN rounding change may cause reconciliation issues
  - Jenkins build discard may be too aggressive
  - RDS egress CIDR may break on failover

TODAY REMAINING (after meeting):
  - Handle whatever the ETL alert is
  - Derek CB PR review (platform-infra#347)
  - Rachel needs timeline + containment + remediation dates for 
    GDPR notification draft
  - Any action items from PCI meeting
  - GDPR notification deadline: Saturday ~8:30 PM

FRIDAY:
  - Lisa's canary AnalysisTemplate PR
  - Any PCI/GDPR follow-ups
  - Derek's CB PR (if not reviewed Thu)

MONDAY:
  - EKS upgrade runbook due (PLAT-892)
  - Tom Chen returns → Kafka onboarding meeting
  - Atlas decommissioning (pending Legal clearance)
  - Production Linkerd upgrade window (tentative Tue 2 AM)
```

---

**END OF HANDOFF DOCUMENT — v4 (CHAOS ENGINE EDITION)**

**Ready to continue simulation. Chaos Engine active. All 12 rules in effect.**

**The room is waiting. Your phone is buzzing. Three seconds.**

**Go.**

# HANDOFF DOCUMENT v4 — ADDENDUM (Things to Add)

*All content below is NEW since the v4 document was created. Add to the respective sections.*

---

## ADD TO: SIMULATION TIMELINE (after "14:04 User's phone buzzes")

```
THURSDAY (continued)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
14:04  Phone buzzes: ETL job failure alert. User pockets phone,
       does not break eye contact with James. Answers questions.
14:04  ANSWERS JAMES — Five concrete prevention controls:
       1. Image source restriction (OPA/Kyverno, reject non-ECR) — 1-2 weeks
       2. Namespace RBAC (platform-provisioned templates) — 1-3 weeks
       3. Default-deny NetworkPolicy on all namespaces — 2-3 weeks
       4. Egress gateway (single choke point for outbound) — 4-6 weeks
       5. Drift detection (weekly scan for non-ArgoCD workloads) — Monday
       Key line: "If we only build walls without building a door, 
       the next person will climb the wall."
14:10  Nina speaks up — validates "make the right thing easy" framing.
       "The right way would have been a two-week platform onboarding 
       process for a 50-line CronJob. So I deployed directly."
       Sarah: "The platform team hasn't built the self-service tooling."
14:15  James: "I don't just want guardrails. I want the self-service 
       catalog Nina's describing."
14:18  Rachel: needs specific dates (not ranges) by 10 AM Friday.
       "Evidence of completion" column for each control.
14:20  Wei: transition period needed for Tom's team (3 analysts depend 
       on data daily). Can S3 sink keep running?
14:22  User confirms S3 sink running, but hedges: "I should confirm 
       it's still healthy after encryption changes. Let me verify 
       within the hour."
14:24  Aisha confirms: containment posture stays. ETL CronJob runs, 
       Atlas stays blocked.
14:25  User asks Nina directly: "What would 'easy' look like?"
       Nina: "A template I can clone, fill in source/destination, 
       submit a PR. Deployed within a day or two."
14:28  Revised timeline presented to James with self-service factored in:
       - Drift detection: Monday Jan 22
       - Image admission warn mode: Friday Jan 26
       - Self-service template + default-deny: mid-February
       - Egress gateway: end of Q1
       "The wall and the door at the same time."
14:32  James: "Do that." Summarizes seven commitments.
       Aisha: Atlas stays blocked, data preserved until Legal clears purge.
       Rachel: "Don't purge anything until I say so."
       James: "Make Monday productive with Tom. Not an ambush."
       Sarah: Setting up Monday 10 AM meeting (Tom, Wei, user, Sarah).
14:38  Meeting ends. Rachel catches user at door: "Include 'evidence 
       of completion' column. Deployment date plus test or audit result."
14:39  User in hallway. Pulls out phone. ETL alert is 35 min old.

14:41  Slack to Wei: investigating ETL + S3 health, will message 
       within the hour
14:42  Investigation begins                                  [ETL INVESTIGATION]

       FINDING 1: Alert NOT from PLAT-919. 
         Pre-existing KubeJobFailed rule with `for: 12h` clause.
         ETL failed at 2 AM, alert fired at 2 PM. Twelve hours exactly.
         PLAT-919 is correctly scoped to Redis. ✅

       FINDING 2: ETL job succeeded at data processing.
         Read all 4 tables from RDS ✅
         Wrote all 4 topics to Kafka ✅
         S3 sink verified RUNNING ✅
         Atlas sink verified FAILED → exit code 1
         Job "failed" because built-in health check expects ALL sinks healthy.
         Atlas sink intentionally blocked = expected "failure."

       FINDING 3: KMS PERMISSION GAP (self-inflicted) 🔴
         S3 sink tried to write with explicit KMS → 403 AccessDenied
         Retried 3x, fell back to "write without encryption specification"
         Bucket default encryption (KMS) applied server-side
         Data IS encrypted, but by accident (bucket default), not design
         Root cause: kafka-connect-s3-writer IAM role NOT in KMS key policy
         User added: platform-admin, order-data-export-role, data-analytics-role
         User missed: kafka-connect-s3-writer
         Same class of error as what user caught in Priya's PR review

14:56  Investigation complete. Full picture understood.
15:00  KMS key policy fix:
       - Enumerated ALL S3 writers from access logs and service accounts
       - Added kafka-connect-s3-writer to KMS key policy
       - Restarted S3 connector, verified clean writes (no AccessDenied)
15:05  KMS permission gap fixed and verified                 ✅

15:08  Wei DM: full transparency about KMS gap.
       "It was working by accident, not by design. I fixed the key 
       policy just now. I should have caught this when I made the 
       encryption change yesterday."
       Wei: "Appreciate the honesty. Seriously."

15:12  Nightly ETL alert addressed:
       - Alertmanager silence applied (7-day expiry, clear comment)
       - Targets: KubeJobFailed for etl-daily-agg-* in data-analytics only
       - Comment references PLAT-921 and intentional Atlas block
       - Silence expires Mon Jan 25, forcing resolution

15:15  Self-accountability note written for PCI document + personal notes
15:18  Sarah DM: "Wei used the word 'refreshing.'"
       Sarah also: "James wants to present remediation timeline to the 
       board next week. Rachel's dates and your prevention controls are 
       going in front of the CEO."

15:25  Derek DM: 0.3% NR flags on payment authorize since ServiceProfile
       merge last night. "Could be noise, could be the retry change."

15:27  User investigates NR signal immediately (payment under velocity freeze)
       
       FINDING 4: NR rate step change at 11:00 PM Wed ← 13 min after 
         ServiceProfile took effect at 10:47 PM. NOT pre-existing noise.
       
       FINDING 5: NR flags on OUTBOUND to payment-gateway-proxy:443.
         Route matching correctly (not DEFAULT). Connection drops on 
         outbound, not inbound.
       
       FINDING 6: Application error rate bumped 0.09% → 0.41%.
         The 0.3% NR IS landing as 5xx to callers. Real user impact.

15:31  Derek reveals: added `timeout: 5s` to ALL routes after user's approval.
       "I thought a 5s default was reasonable and it was a one-line addition.
       I should have re-requested review."
       
       ROOT CAUSE: Per-route 5s timeout killing tail-latency requests.
         Payment gateway p99 = 3-4s, spikes to 6-7s.
         0.31% of requests exceed 5s → killed by proxy → NR → 5xx.
         Same pattern as the payment ServiceProfile timeout user 
         approved, but this time Derek added it post-approval.

15:36  User tells Derek to remove timeout from ALL routes. Keep isRetryable.
       "Any commit after an approval gets a re-review ping."
       Derek: "Fair. Won't happen again."

15:44  Derek pushes fix (removes timeout from all 4 routes)
15:46  ArgoCD syncs
15:54  NR rate back to zero. Fix confirmed.                  ✅

15:42  Lisa informed: 17-hour slow burn consumed ~1.1% error budget.
       Payment-service: 87.1% → 86.0%. Three hits this week.
       Lisa: "The timeout was an unapproved addition that caused a 
       secondary issue, detected and resolved within 17 hours."

15:55  Self-note: didn't re-check PR after approval. Approved based 
       on Slack description, not final merged diff. Process fix: 
       review the MERGED commit, not the PR description.

15:58  CB PR review begins (platform-infra#347)              [PR REVIEW]
16:18  Review complete. 3 blocking, 1 should-fix, 1 non-blocking:

       BLOCKING #1: Per-route timeout: 3s on inventory ServiceProfile.
         Same pattern just removed from payment. "We literally just 
         removed a per-route timeout 30 minutes ago." Remove it.

       BLOCKING #2: Low-confidence fallback returns available: true
         with no stock data. Overselling risk. Should return unknown 
         or false, let frontend handle degraded state.

       BLOCKING #3: stockCache doesn't exist yet. Was flagged in 
         Monday meeting as "doesn't exist." If null, high-confidence 
         path never fires, 100% of fallbacks = accept everything.

       SHOULD-FIX: slowCallDurationThreshold 1500ms too close to 
         inventory p99 (1.08s). slidingWindowSize 50 = 4 seconds 
         of traffic, too reactive. Recommend 2000ms + window of 100.

       NON-BLOCKING: PrometheusRule looks good. Suggested recording rule.

16:22  Derek responds: agrees on #1 (timeout habit acknowledged) and #3 
       (stockCache stubbed but not implemented — "writing the code path 
       I wanted to exist, not the code path that would exist at deploy").
       
       Pushback on #2: Nina told him product prefers accepting orders 
       over showing unavailable. Asks if they can keep available: true.

16:25  Nina DM: pushback on blocking #2 with business math.
       "Historical oversell rate ~0.4%. During CB event, ~360 orders 
       worst case, ~10 actual oversells, $200-300 in refunds. Less than 
       revenue lost from showing out-of-stock to 360 customers."
       "Product signed off on this."

>>> CURRENT TIME: Thursday 4:25 PM <<<
>>> Derek + Nina pushing back on CB PR blocking #2 <<<
>>> User needs to decide: hold block, unblock with conditions, or unblock <<<
>>> Rachel's dates table not started (due 10 AM Friday) <<<
>>> Drift detection CronJob not started (committed Monday to VP) <<<
```

---

## ADD TO: INCIDENT 4 — CONTAINMENT STATUS UPDATES

```
CONTAINMENT STATUS UPDATES (Thursday afternoon):

ETL ALERT INVESTIGATION (Thu 2:42-3:00 PM):
  ✅ PLAT-919 PrometheusRule confirmed correctly scoped (Redis only)
  ✅ Alert was from pre-existing KubeJobFailed rule (12h `for:` delay)
  ✅ ETL data processing SUCCEEDED (all 4 tables → Kafka → S3)
  ✅ ETL "failure" is exit code 1 from sink health check (Atlas = FAILED)
  ✅ This will recur nightly until pipeline restructured with Tom
  ✅ Alertmanager silence applied (7-day, expires Mon Jan 25)

KMS PERMISSION FIX (Thu 3:00-3:05 PM):
  ✅ kafka-connect-s3-writer added to KMS key policy
  ✅ S3 connector writes verified with explicit KMS (no more 403 fallback)
  ✅ All S3 writers enumerated from access logs before policy update
  ✅ Wei informed with full transparency

S3 SINK STATUS (confirmed Thu 3:08 PM):
  ✅ Orders/order_items flowing to S3 continuously
  ✅ Data encrypted with KMS (now via explicit client-side, not just bucket default)
  ✅ Wei's analysts confirmed data is current
```

## ADD TO: INCIDENT 4 — KMS KEY

```
KMS KEY (UPDATED):
  Alias: alias/novamart-data-exports
  Key policy (UPDATED Thu 3:05 PM): 
    platform-admin (full), 
    order-data-export-role (encrypt/generate),
    data-analytics-role (decrypt),
    kafka-connect-s3-writer (encrypt/generate) ← ADDED Thu, was missing
  
  ✅ All writers now have explicit KMS permissions
  ✅ Verified via connector restart + log inspection (no AccessDenied)
```

## ADD TO: INCIDENT 4 — GAPS IN HANDLING

```
ADDITIONAL GAPS (discovered Thu):
  - KMS key policy scoped to 3 IAM roles, missed kafka-connect-s3-writer.
    S3 sink continued via bucket default encryption fallback — working 
    by accident, not design. Fixed Thu 3:05 PM. Should have enumerated 
    all writers from access logs before scoping key policy.
    Self-identified through ETL investigation, not proactive verification.
  
  - Told seven people in PCI meeting "S3 data flow was uninterrupted" — 
    true by luck (bucket default encryption), not competence. Hedged 
    with "let me verify" rather than asserting certainty, which saved 
    credibility when the gap was discovered.
```

---

## ADD TO: LIVE SYSTEM CHANGES — CONSEQUENCE TRACKING

```
CHANGE                              APPLIED         DOWNSTREAM VERIFIED?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

KMS key policy update               Thu 3:05 PM     ✅ VERIFIED
  (added kafka-connect-s3-writer)                    ✅ S3 connector writes clean
                                                     ✅ No more AccessDenied warns
                                                     ✅ All writers enumerated

Alertmanager silence                 Thu 3:12 PM     ✅ Applied
  (KubeJobFailed for ETL)                            ✅ 7-day expiry (Jan 25)
                                                     ✅ Clear comment with context
                                                     ✅ Won't mask other job failures

ServiceProfile timeout removal       Thu 3:46 PM     ✅ VERIFIED
  (Derek pushed, removed timeout                     ✅ NR rate → 0% by 3:54 PM
   from all 4 payment routes)                        ✅ ArgoCD synced
                                                     ✅ Application error rate baseline

PREVIOUSLY UNVERIFIED — NOW RESOLVED:
  ✅ PLAT-919 scope: CONFIRMED correctly scoped (Redis only)
  ✅ KMS permissions for Kafka Connect S3 writer: FIXED
  ✅ S3 sink health post-encryption: CONFIRMED working
  ✅ ETL job data processing: CONFIRMED successful (exit code 1 
     is health check, not data failure)

STILL UNVERIFIED:
  ❓ HALF_EVEN vs HALF_UP historical reconciliation
  ❓ Jenkins build discard aggressive limit
  ❓ Redis Terraform PR merged?
  ❓ RDS egress CIDR fragility (documented, not fixed)
  ❓ order-data-export CronJob: does its IAM role have KMS perms?
     (writes to novamart-data-exports, runs at 2 AM — next run 
     will test this. Should verify proactively before then.)
  ❓ Analytics dashboard: any external access broken by NetworkPolicy v3?
```

---

## ADD TO: PCI MEETING OUTCOMES (new section or update existing)

```
PCI EMERGENCY REVIEW MEETING — COMPLETED (Thu 2:00-2:38 PM)

DECISIONS MADE:
  1. GDPR Art. 33 supervisory notification: APPROVED (Rachel drafting)
     Deadline: Saturday ~8:30 PM
  2. Atlas stays blocked: NON-NEGOTIABLE (Aisha)
  3. Atlas data: PRESERVED, no purge until Legal clears (Rachel)
  4. ETL CronJob: keeps running (Aisha confirmed)
  5. S3 sink for orders/order_items: keeps running (not payment data)
  6. Drift detection live by Monday: COMMITTED TO VP
  7. Image admission warn mode by Jan 26: COMMITTED
  8. Self-service template + default-deny by mid-Feb: COMMITTED
  9. Egress gateway by end of Q1: COMMITTED
  10. Tom meeting Monday 10 AM: Sarah scheduling (Tom, Wei, user, Sarah)
  11. GDPR Art. 34 (customer notification): prepare but hold pending 
      supervisory authority guidance

COMMITMENTS WITH VP VISIBILITY (board presentation next week):
  ┌────────────────────────────────┬──────────────┬─────────┐
  │ Control                        │ Date         │ Owner   │
  ├────────────────────────────────┼──────────────┼─────────┤
  │ Drift detection CronJob        │ Mon Jan 22   │ User    │
  │ Image admission (warn mode)    │ Fri Jan 26   │ User    │
  │ Self-service pipeline template │ Mid-Feb      │ User    │
  │ Default-deny NetworkPolicy     │ Mid-Feb      │ User    │
  │ Egress gateway                 │ End of Q1    │ User    │
  │ Rachel dates table (specific)  │ Fri 10 AM    │ User    │
  └────────────────────────────────┴──────────────┴─────────┘
  
  Note: Rachel needs SPECIFIC dates (not ranges) + "evidence of 
  completion" column (deployment date + test/audit result) for each.
  James presenting to board next week. CEO will see this.

KEY QUOTES:
  James: "I like 'the wall and the door at the same time.' Do that."
  Nina: "A template I can clone, fill in source and destination, 
        submit a PR. Deployed within a day or two."
  James: "Make Monday productive with Tom. Not an ambush."
  Wei: "Appreciate the honesty. Seriously." (re: KMS gap disclosure)
  Sarah: "James wants to present remediation timeline to the board."
```

---

## ADD TO: SERVICEPROFILE INCIDENT (new sub-incident or note)

```
SERVICEPROFILE TIMEOUT ISSUE (Thu ~10:47 PM Wed → 3:54 PM Thu)

NOT A NUMBERED INCIDENT — consequence of approved change.

TIMELINE:
  Wed 10:47 PM — Derek's ServiceProfile merged (isRetryable changes)
  Wed 10:47 PM — Derek also added timeout: 5s on ALL 4 routes (post-approval)
  Wed 11:00 PM — NR rate steps from ~0% to 0.3% on payment authorize
  Thu 3:25 PM  — Derek notices NR flags, reports to user
  Thu 3:27 PM  — User investigates immediately (payment under velocity freeze)
  Thu 3:33 PM  — Mechanism confirmed: 0.31% of requests > 5s, killed by timeout
  Thu 3:36 PM  — User tells Derek to remove all timeouts
  Thu 3:44 PM  — Derek pushes fix
  Thu 3:46 PM  — ArgoCD syncs
  Thu 3:54 PM  — NR rate back to zero

IMPACT:
  - 17 hours of 0.3% additional NR on payment authorize
  - Application error rate: 0.09% → 0.41%
  - Error budget: 87.1% → 86.0% (1.1% consumed)
  - No customer escalation (below alert threshold)

ROOT CAUSE:
  Derek added timeout: 5s to all routes after user approved the PR.
  Payment gateway tail latency 6-7s during slow periods.
  5s timeout killed legitimate requests that would have completed.

USER ACCOUNTABILITY:
  - Did not re-check PR after approval before merge
  - Approved based on Slack description, not final diff
  - Process fix established: review MERGED commit, not PR description
  - Told Derek: "Any commit after approval gets a re-review ping"
  - Derek acknowledged: "Won't happen again"

LESSON: Static per-route timeouts are a recurring anti-pattern.
  This is the second time in one day (payment authorize + inventory 
  in CB PR). The circuit breaker is the correct tool for downstream 
  latency protection, not static proxy timeouts.
```

---

## ADD TO: CB PR REVIEW (new section)

```
CB PR REVIEW — platform-infra#347 (Thu 3:58-4:18 PM)

STATUS: CHANGES REQUESTED — 3 blocking, 1 should-fix, 1 non-blocking

BLOCKING #1: Per-route timeout: 3s on inventory ServiceProfile
  Same anti-pattern just fixed on payment. Remove.
  Derek: AGREED, removing.

BLOCKING #2: Low-confidence fallback returns available: true
  User flagged as overselling risk.
  Derek: Product told him to prefer accepting orders.
  Nina: Business math — 10 oversells ($200-300) < revenue lost 
        from 360 "out of stock" displays. Product signed off.
  STATUS: PUSHBACK FROM DEREK + NINA. User deciding.

BLOCKING #3: stockCache doesn't exist (stubbed, no implementation)
  Without cache, high-confidence path never fires.
  100% of CB-open fallbacks = accept everything (available: true).
  Derek: AGREED, acknowledged gap.

SHOULD-FIX: CB sensitivity tuning
  slowCallDurationThreshold 1500ms too close to p99 (1.08s)
  slidingWindowSize 50 (COUNT_BASED) = 4 seconds, too reactive
  Recommend: 2000ms threshold, window of 100 or TIME_BASED 60s

NON-BLOCKING: PrometheusRule looks good. Suggested recording rule.

ACTIVE DECISION: User needs to respond to Derek + Nina on blocking #2.
  Business has explicitly chosen to accept overselling risk with math.
  User's blocking comment was technically correct but may be overriding
  a business decision that's already been made by the right people.
```

---

## ADD TO: OPEN ITEMS / PENDING WORK

### Updated Active Tickets

```
PLAT-915  [Changes Req] Circuit breaker PR — 3 blockers identified.
                         Derek + Nina pushing back on overselling fallback.
                         Timeout removed (agreed). stockCache gap (agreed).
                         Decision pending on available:true for low-confidence.

PLAT-919  [Done ✅]     Redis cache alerting — CONFIRMED correctly scoped.
                         ETL alert was from pre-existing KubeJobFailed rule,
                         NOT PLAT-919. False alarm on misconfiguration.
```

### New Pending Deliverables

```
- Rachel's remediation dates table → due 10 AM Friday
  Specific dates, "evidence of completion" column, board-presentation quality.
  NOT STARTED as of 4:25 PM Thursday.
  
- Drift detection CronJob → committed Monday Jan 22 to VP/Legal/Security
  Have audit script from Wednesday. Need: CronJob wrapper, Slack alerting,
  ArgoCD deployment, testing.
  NOT STARTED as of 4:25 PM Thursday.
  
- Self-service pipeline template → committed mid-February
  Nina's input: "clone, fill in, submit PR, deployed within a day"
  NEW DELIVERABLE from PCI meeting.

- Respond to Derek + Nina on CB PR blocking #2 → NOW
```

---

## ADD TO: TEAM STATE

```
UPDATES:

James Morrison (VP Eng):  Meeting concluded. Satisfied with prevention plan.
                          "The wall and the door at the same time."
                          Presenting remediation timeline to board next week.
                          CEO will see Rachel's dates + user's controls.

Sarah Chen (manager):     Set up Monday 10 AM meeting with Tom.
                          Relayed James's board presentation plan.
                          "Wei used the word 'refreshing'" (re: KMS honesty).

Rachel Torres (Legal):    Drafting GDPR Art. 33 notification.
                          Needs specific dates + evidence-of-completion by 10 AM Fri.
                          "Don't purge Atlas data until I say so."

Wei Liu (Data Eng mgr):   Relieved about S3 data continuity.
                          "Appreciate the honesty" about KMS gap.
                          Analysts confirmed data is current.
                          Flagged: Tom's sink verification pattern is 
                          good engineering, keep it in restructured pipeline.

Nina Petrov (Order lead): Spoke up in meeting (first time). Validated 
                          "easy path" framing. Gave concrete UX requirement 
                          for self-service template.
                          Now PUSHING BACK on CB PR blocking #2 with 
                          business math (product signed off on overselling).

Aisha Rahman (Security):  Confirmed containment posture unchanged.
                          Atlas blocked + preserved. ETL runs. S3 sink runs.

Derek Huang (Order):      Timeout pattern acknowledged ("I clearly have a habit").
                          CB PR has 3 blockers, agreed on 2, pushing back on 1.
                          Revealed stockCache is stubbed, not implemented.
                          Established: post-approval commits get re-review.

Lisa Park (SRE):          Budget: payment-service 86.0% (87.1% - 1.1% from 
                          ServiceProfile timeout burn). Order-service 64.5%.
                          Building canary AnalysisTemplate (ETA Fri).
```

---

## ADD TO: USER PERFORMANCE (v2 observations)

```
V2 CHAOS ENGINE OBSERVATIONS (Thursday afternoon):

CONSEQUENCES ENCOUNTERED:
  1. KMS permission gap — self-inflicted, discovered through investigation.
     Data safe by accident (bucket default), not design. Fixed + disclosed.
  2. ServiceProfile timeout — approved PR, Derek added post-approval.
     17-hour slow burn, 1.1% budget cost. Caught via Derek's signal.
  3. ETL "failure" — containment action (Atlas block) caused expected 
     but unhandled daily alert. Addressed with targeted silence.

RECOVERY QUALITY:
  ✅ KMS gap: investigated thoroughly (enumerated all writers before fix),
     disclosed honestly to Wei, documented in PCI findings. Strong recovery.
  ✅ ServiceProfile: investigated signal immediately (right priority),
     confirmed mechanism before acting, established process fix.
  ⚠️ Neither was caught proactively — both required external signals 
     (ETL alert, Derek's observation) to surface.

NEW STRENGTHS DEMONSTRATED:
  ✓ Honest self-accountability under pressure (KMS gap disclosure to Wei)
  ✓ Meeting presence (answered VP with specifics while phone was buzzing)
  ✓ Hedging correctly ("let me verify" instead of "it's fine")
  ✓ Translating technical controls to executive language (board-ready)
  ✓ Treating internal users as customers (Nina's "what would easy look like?")
  ✓ Pattern recognition across incidents (timeout anti-pattern, twice in one day)
  ✓ Post-approval process gap identification
  ✓ Not blaming Derek, establishing process instead

NEW GAPS:
  - Didn't enumerate S3 writers before scoping KMS key policy
  - Didn't re-check merged PR diff after approval
  - Two consequences (KMS, ServiceProfile) required external signals,
    not proactive verification
  - Rachel's dates table and drift detection CronJob both not started 
    with <18 hours until Friday 10 AM deadline

ACTIVE DECISION PENDING:
  Derek + Nina pushback on CB PR blocking #2. User must decide whether 
  to hold a technical block against an explicit business decision 
  backed by product, finance, and the order team lead.
```

---

## ADD TO: WHERE TO RESUME

```
SIMULATION TIME:  Thursday 4:25 PM EST, Week 1
SIMULATION MODE:  v2 — CHAOS ENGINE ACTIVE

ACTIVE SITUATION:
  Derek + Nina pushing back on CB PR blocking #2 (low-confidence 
  available:true). Business math provided. Product signed off.
  User needs to decide: hold, unblock with conditions, or unblock.

IMMEDIATE QUEUE:
  1. Respond to Derek + Nina on CB PR (NOW)
  2. Rachel's remediation dates table (due 10 AM Fri — NOT STARTED)
  3. Drift detection CronJob skeleton (committed Monday — NOT STARTED)
  4. EOD notes

RISK ITEMS:
  ⚠️ Rachel's table: board-presentation quality, specific dates, 
     evidence-of-completion column. Due in <18 hours. Not started.
  ⚠️ Drift detection: committed to VP in front of Legal + Security. 
     Have the script, need CronJob + Slack + ArgoCD + testing. 
     Must be done by Monday. Not started.
  ⚠️ order-data-export CronJob runs at 2 AM tonight — does its 
     IAM role have KMS permissions on novamart-data-exports? 
     Same gap pattern as kafka-connect-s3-writer. Should verify 
     BEFORE the 2 AM run, not after.

UNRESOLVED:
  ❓ CB PR blocking #2 decision (active)
  ❓ HALF_EVEN reconciliation (not flagged to finance)
  ❓ order-data-export KMS permissions (not verified)
  ❓ RDS egress CIDR fragility
  ❓ Jenkins build discard limit
  ❓ Redis Terraform PR merge status
```

---

**END OF ADDENDUM**

**Resume point: Thursday 4:25 PM. Derek and Nina are waiting. Rachel's table and drift detection are both un-started with deadlines approaching.**
