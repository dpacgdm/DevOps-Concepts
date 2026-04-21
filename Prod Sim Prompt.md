# PHASE 8 SIMULATION — HANDOFF DOCUMENT (Updated v3)

---

## INCIDENTS/ISSUES COVERED SO FAR: 6 of 100

*(Incident #6 — Payment Service Currency Bug — is ACTIVE, rolled back, fix pending redeploy)*

---

## INCIDENT TRACKER

| # | Severity | Type | Description | Status | Grade |
|---|----------|------|-------------|--------|-------|
| 1 | **SEV2** | Deploy-induced timeout | Order service 504s from aggressive inventory_client_timeout (800ms) at peak traffic. Canary passed at low RPM, failed at production scale. | ✅ RESOLVED — Interim fix 2000ms. Follow-up: circuit breaker. | **A+** |
| 2 | **SEV4** | Stale Terraform lock | Alex (junior) blocked by stale DynamoDB state lock from crashed 2AM Jenkins job. | ✅ RESOLVED — Guided Alex to self-resolve. | **A** |
| 3 | **SEV3** | Capacity — Redis memory pressure | Quarterly catalog refresh (182K products → 435K cache keys) exceeded Redis maxmemory. Hit rate dropped 94%→87%, evictions 8x, catalog p99 tripled. | ✅ RESOLVED — Node upgraded r6g.large→r6g.xlarge overnight. | **A+** (investigation), **A** (maintenance execution) |
| 4 | **SEV3** | Shadow infrastructure | Unmanaged Kafka cluster (6 brokers + 3 ZK) discovered in data-analytics namespace. Deployed by Tom Chen (Data Eng) via direct Helm. Scope expanded by Wednesday audit: full pipeline including Kafka Connect, ETL CronJob reading payments/refunds, MongoDB Atlas external sink. | ✅ MITIGATED — PDB + NetworkPolicy v3 applied. MongoDB Atlas egress blocked. S3 buckets remediated. PCI emergency review Thursday 2 PM. Full onboarding when Tom returns Monday. | **A-** (initial: missed ZK port + EBS check) → **A+** (audit + PCI escalation) |
| 5 | **SEV2** | CI/CD — Zombie Jenkins agents | Jenkins agent pods stuck running 3-6+ hours, consuming entire Karpenter CI pool (62.5/64 CPU). Payment team hotfix blocked. Root cause: JVM memory leak. | ✅ RESOLVED — Build discard config + heap monitoring. Stable 24h+. | **[Grade from earlier]** |
| 6 | **SEV2** | Deploy-induced code bug | Payment service currency rounding hotfix (sha-f91a2b7) introduced `BigDecimal.divide()` without `RoundingMode`. GBP/SEK/DKK/NOK→EUR transactions 100% failing. Canary passed on USD-only traffic sample. | 🟡 ROLLED BACK — sha-a8c71e3 restored. Corrected fix pending redeploy. | **A+** (so far) |

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
  PLAT-915 — Circuit breaker on order→inventory (Derek PR expected Thursday)
  PLAT-916 — Canary analysis: traffic-volume gate + transaction-diversity gate
             (EXPANDED after Incident 6 — second canary sample bias failure)
  PLAT-917 — Investigate inventory-service p99 at scale

CIRCUIT BREAKER MEETING OUTCOMES (Mon 2 PM):
  - Resilience4j CB with revised config
  - Linkerd ServiceProfile: isRetryable:false on POST /api/v1/orders
  - Per-replica CB state is FINE (no shared state needed)
  - Two-tier fallback design (high-confidence → accept with "Processing",
    low-confidence → degraded UX)
  - Needs inventory stock level cache (doesn't exist yet)
  - Derek PR by Thursday
  - Lisa updating SLI definition for degraded-mode responses

POSTMORTEM: Draft shared, waiting for feedback. 
            Updated to reference Incident 6 as companion canary gap.
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
  Tue 2:22 AM — Terraform updated

POST-UPGRADE STATUS (as of Wed):
  Hit rate: 94.1% (fully recovered)
  Evictions: 0/min
  Memory: 52% of maxmemory

COST IMPACT: +$183/month

FOLLOW-UP TICKETS:
  PLAT-918 — Scale Redis (COMPLETED ✅)
  PLAT-919 — Redis cache alerting (COMPLETED ✅)
  PLAT-920 — Quarterly refresh capacity planning gate
```

### INCIDENT 4: Shadow Kafka / Data Pipeline (SEV3) ✅ Mitigated — SCOPE EXPANDED BY AUDIT

```
TRIGGER:      Jake Torres discovered unknown StatefulSets during GP3 migration
REPORTED:     Mon 2:38 PM
INVESTIGATED: Mon 2:48-3:25 PM
MITIGATED:    Mon 3:15 PM (PDB + NetworkPolicy v1 → v2)
AUDIT EXPANSION: Wed 11:45 AM — cluster audit revealed full data pipeline scope
RE-MITIGATED: Wed 12:15 PM (NetworkPolicy v3 + S3 remediation)
FULL FIX:     Pending Tom Chen return from PTO (next Monday) + PCI review (Thu 2 PM)

ORIGINAL DISCOVERY (Monday):
  - 6 Kafka brokers + 3 ZooKeeper (Bitnami images from Docker Hub)
  - Namespace: data-analytics
  - Deployed Nov 9 by tom.chen@novamart.com via direct Helm
  - NOT in ArgoCD, NOT in Terraform, NOT meshed
  - No monitoring, no backup, no runbook, no on-call owner

AUDIT EXPANSION (Wednesday — CRITICAL):
  Cluster-wide audit revealed the Kafka cluster is part of a larger 
  unmanaged data pipeline:

  COMPONENT 1: Kafka Connect (confluent cp-kafka-connect:7.5.3, Docker Hub)
    - Two connectors running:
      a) orders-sink-s3: etl.orders + etl.order_items → S3 (novamart-analytics-datalake)
      b) payments-sink-mongo: etl.payments + etl.refunds → MongoDB Atlas (EXTERNAL)
         🔴 CRITICAL: Payment data leaving AWS boundary

  COMPONENT 2: ETL CronJob (etl-daily-aggregation, daily 2 AM)
    - Connects to production orders RDS (analytics_reader user)
    - Reads: orders, order_items, payments, refunds tables
    - Writes to Kafka topics: etl.orders, etl.order_items, etl.payments, etl.refunds
    - 🔴 CONFIRMS: Payment data flows through Kafka

  COMPONENT 3: Analytics Dashboard (novamart-analytics-dashboard:latest)
    - Custom image on Docker Hub (NOT ECR, no vulnerability scanning)
    - No image pull secrets
    - Direct connection to production orders RDS
    - 🔴 Unscanned image with production DB access

  COMPONENT 4: MongoDB Atlas (EXTERNAL)
    - Connection: mongodb+srv://analytics:****@novamart-analytics.xxxxx.mongodb.net
    - Database: payment_analytics, Collection: transactions
    - Password stored in Kafka connect-configs topic (plaintext, no ACLs)
    - 🔴 NOT a NovaMart organizational account (IT confirmed)
    - 🔴 Likely personal/team Atlas account containing payment data
    - 🔴 CDE boundary violation — payment data outside controlled infrastructure
    - Unknown: access controls, encryption, retention, who has access
    - Wei Liu (Tom's manager) reaching out to Tom for Atlas account details

  COMPONENT 5: Order Data Export CronJob (order-prod namespace)
    - Deployed by Nina Petrov (Dec 20) for finance team request
    - Exports: customers, shipping_addresses tables → S3 (novamart-data-exports)
    - 🟡 PII at rest (customer names, shipping addresses in CSV)
    - Nina informed by Sarah, willing to hand over to platform team

COMPLETE DATA FLOW MAP:
  Orders RDS (payments, refunds, orders, order_items, customers, shipping_addresses)
      │                                                    │
      ▼                                                    ▼
  ETL CronJob (daily 2AM)                    Order Data Export CronJob (daily 2AM)
      │                                                    │
      ▼                                                    ▼
  Kafka Topics (no ACLs, no mTLS):           S3: novamart-data-exports
    etl.orders ────→ S3: analytics-datalake    (customers.csv.gz, 
    etl.order_items → S3: analytics-datalake    shipping_addresses.csv.gz)
    etl.payments ──→ MongoDB Atlas 🔴          27 days of exports, 108 objects
    etl.refunds ───→ MongoDB Atlas 🔴          PII files
      │
      ▼
  Analytics Dashboard (reads Kafka + RDS directly)

PCI/COMPLIANCE GAP SUMMARY:
  ❌ Payment data flowing to external MongoDB Atlas (CDE boundary violation)
  ❌ Atlas likely personal account (not organizational)
  ❌ Atlas password in plaintext in Kafka topic (no ACLs)
  ❌ No mTLS between components (no Linkerd mesh)
  ❌ No Kafka topic-level access controls
  ❌ No audit logging on broker access
  ❌ Docker Hub images (not scanned, not in ECR)
  ❌ K8s Secrets not in External Secrets Operator
  ❌ S3 buckets not in Terraform
  ❌ No monitoring/alerting on pipeline
  ❌ PII in S3 with no lifecycle/retention policy
  ⚠️ EBS encrypted (AWS-managed key, not CMK)
  ⚠️ RDS egress NetworkPolicy uses ipBlock CIDR (fragile on failover)

CONTAINMENT ACTIONS APPLIED:
  ✅ PDB on Kafka brokers + ZooKeeper (Monday)
  ✅ NetworkPolicy v1 → v2 (Monday, ZK port fix Tuesday 1:50 AM)
  ✅ NetworkPolicy v3 (Wednesday 12:15 PM):
     - Default-deny egress for data-analytics namespace
     - Kafka 9092: intra-namespace + order-prod + payment-prod
     - ZK 2181: intra-namespace only
     - kafka-connect 8083: intra-namespace only
     - analytics-dashboard 3000: intra-namespace only
     - Egress: DNS, intra-namespace, RDS CIDR, S3 VPC endpoint
     - MongoDB Atlas: BLOCKED (egress deny — accidental but correct)
  ✅ MongoDB Atlas sink: FAILED state (blocked by NetworkPolicy v3)
     - Kafka Connect buffering payment/refund data internally
     - S3 sink still running (orders/order_items — not payment data)
     - Aisha confirmed: KEEP BLOCKED
  ✅ S3 novamart-data-exports remediated:
     - Encryption upgraded SSE-S3 → KMS/CMK (108 objects re-encrypted)
     - Server access logging enabled
     - Versioning enabled
     - Public access block confirmed (was already enabled)
     - Bucket policy scoped to two specific IAM roles
     - NOT in Terraform (to be added)
     - Lifecycle/retention: deferred to Thursday PCI review
  ✅ S3 novamart-analytics-datalake remediated:
     - Encryption upgraded SSE-S3 → KMS/CMK (1,847 objects, 2 GB re-encrypted)
     - Server access logging enabled
     - Versioning enabled
     - NOT in Terraform (to be added)
  ✅ KMS key created: alias/novamart-data-exports (key policy scoped)
  ✅ Escalated to Aisha Rahman (Security) with full data flow + gap list
  ✅ Escalated to Sarah Chen (governance, management framing)
  ✅ Sarah talked to Nina (governance framing, not blame)
  ✅ Wei Liu (Tom's manager) reaching out to Tom for Atlas details
  ✅ PCI emergency review scheduled: Thursday 2 PM
     - Attendees: User, Aisha, Sarah, Wei Liu, Nina
  ⚠️ ETL CronJob: STILL RUNNING (per Aisha — don't suspend yet)
  ⚠️ DB user (analytics_reader) permissions: NOT YET AUDITED

FULL CLUSTER AUDIT — OTHER FINDINGS:
  ┌───┬────────────────┬──────────────────────┬─────────┬────────────┬──────────────┐
  │ # │ Namespace       │ Workload             │ Owner   │ PCI Risk   │ Status       │
  ├───┼────────────────┼──────────────────────┼─────────┼────────────┼──────────────┤
  │ 5 │ staging        │ perf-test-harness    │ Derek H.│ ⚪ NONE   │ Derek cleaning│
  │   │                │                      │         │            │ up today      │
  │ 6 │ staging        │ mock-payment-gateway │ Derek H.│ ⚪ NONE   │ Fake creds    │
  │   │                │                      │         │            │ confirmed,    │
  │   │                │                      │         │            │ cleanup today │
  │ 7 │ default        │ nginx-test           │ Unknown │ ⚪ NONE   │ Delete (todo) │
  │ 8 │ monitoring     │ node-problem-detector│ Marcus  │ ⚪ NONE   │ Marcus adding │
  │   │                │                      │         │            │ to ArgoCD     │
  └───┴────────────────┴──────────────────────┴─────────┴────────────┴──────────────┘

AUDIT REPORT:
  - Structure drafted (Executive Summary, Methodology, Findings by 
    Tier, Data Flow Diagrams, Containment Actions, Remediation Roadmap,
    Recommendations)
  - ~40% complete as of Wed 2:45 PM (interrupted by Incident 6)
  - Deadline moved up: Aisha needs pre-read by Thu 1 PM
  - Sarah wants full report EOD Thursday (moved from Friday)
  - Key recommendation: "Make the right thing the easy thing" — 
    reframe from blame to guardrails

FOLLOW-UP TICKETS:
  PLAT-921 — Full platform onboarding (ArgoCD, ECR, monitoring,
             backup, runbook, RBAC). Meeting with Tom next Monday.
  PLAT-922 — PCI scope assessment (EXPANDED — now includes Atlas,
             S3 PII, credential exposure, CDE boundary violation).
             PCI emergency review Thu 2 PM.

KNOWN GAPS IN HANDLING:
  - Initial NetworkPolicy (Mon) exposed ZK port (fixed Tue 1:50 AM)
  - Didn't check EBS encryption immediately (fixed Tue 1:50 AM)
  - NetworkPolicy v3 egress to RDS uses ipBlock CIDR — fragile on 
    RDS multi-AZ failover. Documented as TODO for mesh migration.
  - MongoDB Atlas block was accidental (NetworkPolicy didn't include 
    Atlas in egress allowlist). Disclosed honestly to Aisha. 
    Correct outcome, unintentional path.
  - Should have stated explicitly: "If S3 bucket were public, 
    that changes the calculus from contain-and-remediate to 
    stop-the-bleeding." Recommendation should have been conditional 
    on S3 investigation results.
```

### INCIDENT 5: Zombie Jenkins Agents / CI Pool Exhaustion (SEV2) ✅

```
TRIGGER:      Payment team (David Okafor) reports builds failing since 7 AM
              Alex/Jake investigated: 23 Jenkins agents, 12 running 3-6+ hours
              Karpenter CI pool at 97.6% CPU limit (62.5/64)
REPORTED:     Tue 7:45 AM (David), 8:20 AM (Alex investigation)
RESOLVED:     Tuesday (exact time from session)

ROOT CAUSE:   JVM memory leak in Jenkins agent pods.

FIX APPLIED:  Build discard configuration + heap monitoring.

STATUS (as of Wed):
  Jenkins heap stable 24h+. Build discard fix working.
  CI pool operating normally.

SKILLS TESTED:
  → Jenkins on K8s agent lifecycle
  → Karpenter NodePool limits and resource management
  → Zombie pod identification and cleanup
  → Priority decision (unblock hotfix vs investigate root cause)
  → Cross-team communication under pressure
```

### INCIDENT 6: Payment Service Currency Bug (SEV2) 🟡 ROLLED BACK — FIX PENDING

```
TRIGGER:      PagerDuty — PaymentServiceHighErrorRate (>2% for 5 min)
FIRED:        Wed 2:47 PM
ACKED:        Wed 2:48 PM
ROOT CAUSE:   Wed 2:52 PM (~4 min from ack)
ROLLBACK:     Wed 2:53 PM (initiated), 2:58 PM (complete)
BASELINE:     Wed 3:01 PM (error rate 0.12%)
TTR:          ~13 min from ack to baseline
IMPACT WINDOW: 2:40 PM — 2:58 PM (~18 minutes)
BUDGET BURN:  91.3% → 87.1% (4.2% consumed)

BLAST RADIUS:
  847 failed payment authorizations
  89 failed refunds
  34 failed status checks
  Estimated 500-700 unique customers unable to complete purchases
  All non-terminating-decimal EU currency pairs affected (GBP, SEK, DKK, NOK → EUR)
  USD and same-currency transactions unaffected

ROOT CAUSE:
  Payment team deployed currency rounding hotfix (sha-f91a2b7) at ~2:30 PM.
  Fix switched from double arithmetic to BigDecimal for EU currency math.
  CurrencyService.convertAmount() (line 47) uses BigDecimal.divide() 
  WITHOUT specifying RoundingMode.
  
  For currency pairs where exchange rate produces non-terminating 
  decimal (e.g., GBP→EUR), BigDecimal.divide() throws 
  java.lang.ArithmeticException: "Non-terminating decimal expansion; 
  no exact representable decimal result."
  
  Currency pairs with terminating decimals (e.g., USD→EUR at certain 
  rates) work fine — which is why canary passed.

CANARY GAP (SAME CLASS AS INCIDENT 1):
  - Canary ran 5 minutes at 5% traffic
  - Traffic sample was USD-dominated (time of day + low volume)
  - No currency-pair diversity gate in canary analysis
  - Canary passed with 0% errors because it didn't sample 
    GBP/SEK/DKK/NOK transactions
  - This is the SECOND incident caused by canary sample bias:
    - Incident 1: traffic volume bias (low RPM → missed latency)
    - Incident 6: traffic diversity bias (USD only → missed currency bug)

INVESTIGATION APPROACH:
  1. Self-check: "Could my NetworkPolicy have caused this?" 
     → Ruled out by timing (errors started 2:40, policy applied 12:15)
  2. Error shape: 500s (not 504s), fast failures (not timeouts)
     → Application code bug, not infrastructure/timeout issue
  3. Endpoint analysis: authorize primary, refund + status also failing
     → Shared code path (CurrencyService)
  4. Downstream check: payment gateway proxy 0% errors
     → Error inside payment-service, not downstream dependency
  5. Logs: ArithmeticException stack trace → BigDecimal.divide() 
     without RoundingMode → immediate root cause

ROLLBACK DECISION:
  "Can't pay" is categorically worse than "overcharged by 1 cent."
  - Current state: GBP/SEK/DKK/NOK customers cannot pay AT ALL
  - Rollback state: Those customers overcharged 1-2 cents (original bug)
  - Original bug is bounded, known, refundable
  - Rollback is correct. No debate.

FIX APPLIED:
  Rollback to sha-a8c71e3 via kubectl argo rollouts undo.
  All 8 replicas confirmed running old image by 2:58 PM.

CORRECTED FIX STATUS:
  - Priya Kapoor (payment eng) has one-liner ready: 
    BigDecimal.divide() with RoundingMode.HALF_EVEN (banker's rounding)
  - Finance confirmed HALF_EVEN is correct for EU regulatory compliance
  - Adding tests for all 12 EU currency pairs
  - Fix estimated ready for review: 3:30 PM Wednesday
  - ALSO FOUND: Two other BigDecimal.divide() calls in refund path 
    without RoundingMode (pre-existing, work by accident due to 
    terminating decimals). Decision pending: fix now or scope separately.

REDEPLOY PLAN:
  - Manual review of one-line fix before merge
  - Canary with manual verification: synthetic GBP/SEK/DKK/NOK 
    test transactions before promotion
  - User on-call during promotion
  - Target: ~4 PM Wednesday (if fix passes review)

PENDING RESPONSES (as of 3:10 PM):
  1. David Okafor: Should Priya K. fix the two other BigDecimal.divide() 
     calls in refund path (pre-existing, not in hotfix)? 
  2. Lisa Park: Canary currency-pair diversity alert offer. 
     Budget burned 4.2% (91.3% → 87.1%).
  3. Derek Huang: Add isRetryable:false to POST /api/v1/payments/authorize 
     in ServiceProfile?

FOLLOW-UP TICKETS:
  PLAT-916 — EXPANDED: Now includes transaction-diversity gate in 
             addition to traffic-volume gate. Two incidents from 
             same canary gap class.
  [NEW TICKET NEEDED] — Payment service canary: require currency-pair 
             diversity before promotion
  [NEW TICKET NEEDED] — Audit all BigDecimal.divide() calls in 
             payment codebase for missing RoundingMode

SKILLS TESTED:
  ✓ PagerDuty triage and immediate ack
  ✓ Self-check ("could I have caused this?" — NetworkPolicy timing analysis)
  ✓ Pattern recognition without assumption (listed 5 scenarios, didn't anchor)
  ✓ PromQL error shape analysis (500 vs 504, by endpoint, by status code)
  ✓ Downstream dependency elimination (payment gateway healthy)
  ✓ Java BigDecimal bug identification from stack trace
  ✓ Rollback trade-off reasoning (can't pay vs overcharged)
  ✓ Fast decisive rollback (4 min root cause, immediate rollback decision)
  ✓ RoundingMode domain knowledge (HALF_EVEN for financial, not HALF_UP)
  ✓ Connecting incidents to systemic pattern (Incident 1 + 6 = canary gap class)
  ✓ PLAT-916 scope expansion (volume + diversity)
  ✓ Redeploy gating (manual review + synthetic test transactions)
  ✓ Blast radius quantification (847 failed auths, 500-700 customers)
  ✓ Structured incident comms (ACK → ROOT CAUSE → ROLLBACK → UPDATE)
  ✓ Cross-team code-level support (platform eng diagnosing Java code bug)
```

---

## SIMULATION TIMELINE

```
WEEK 1 — MONDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
09:00  On-call shift starts (primary: user, secondary: Marcus)
09:04  Laptop open. PagerDuty (17min unacked), 3 Slack msgs, 1 email
09:04  ACK PagerDuty, post to #incidents                     [INC-1 START]
09:05  Dashboard + PromQL investigation
09:06  ArgoCD/rollout inspection, GitOps diff
09:08  Unblock Alex (state lock guidance)                    [INC-2 START]
09:10  Hypothesis formed: 800ms timeout too aggressive
09:13  CONFIRMED: trace shows DEADLINE_EXCEEDED at 800ms
09:13  Nina complicates: 800ms was intentional (INC-2847)
09:14  Decision: surgical fix 2000ms (middle-ground)
09:15  GitOps hotfix pushed
09:21  Fully rolled out
09:30  RESOLVED. Error rate 0.12%, burn rate 0.8x           [INC-1 END]
09:32  Lisa offered inventory p99 alert → accepted
09:35  Created 3 follow-up tickets (PLAT-915/916/917)
09:40  Checked on Alex                                       [INC-2 END]
09:42  Alex self-resolved state lock
09:45  Scheduled 2 PM circuit breaker meeting
09:50  Deep work: EKS 1.29 upgrade runbook research
11:02  Ryan Mitchell reports intermittent catalog slowness   [INC-3 START]
11:31  Investigate catalog/Redis (15-min timebox)
11:37  Root cause: quarterly refresh exceeded Redis memory
11:40  Maxmemory bumped 80%→90%                              [INC-3 MITIGATED]
11:44  Tickets created (PLAT-918/919/920)
12:00  Lunch (phone on)
13:00  Sarah's Q1 priorities email response
13:20  Priya's Linkerd PR #347 review
14:00  Circuit breaker design meeting
14:43  Priya's trust anchor rotation question
14:45  Catalog alert flapping — silenced
14:48  Kafka investigation begins                            [INC-4 START]
15:06  Alert gap fixed — critical Redis/catalog alerts deployed
15:15  PDB + NetworkPolicy applied for Kafka                 [INC-4 MITIGATED]
15:20  Sarah DM — flagged governance + PCI implications
15:25  PLAT-921 + PLAT-922 created
15:30  Redis staging failover test
16:00  Maintenance notification posted
16:15  EOD daily notes

MONDAY NIGHT / TUESDAY EARLY AM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
01:50  Fix Kafka NetworkPolicy (ZK port bug). Check EBS encryption.
01:58  Pre-flight: manual Redis snapshot
02:00  Execute Redis upgrade                                 [INC-3 FINAL FIX]
02:15  Upgrade complete.
02:22  Terraform updated.
02:30  Back to sleep.

TUESDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
07:45  David Okafor reports payment builds failing           [INC-5 START]
08:20  Alex: Jenkins agents stuck Pending, Karpenter full
08:55  User logs in, investigates
       (Jenkins zombie pods — JVM memory leak)
       Resolved: Build discard config + heap monitoring      [INC-5 RESOLVED]
       Payment team unblocked.

WEDNESDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
10:00  Team standup (James Morrison VP Eng present)          [STANDUP — A+]
       - Q1 priorities: risk-first framing ("1-3 protect, 4-5 improve")
       - PCI insertion to official priority list
       - Incident summaries (cost → cause → fix → takeaway)
       - Jenkins: "Invisible until it wasn't"
       - Kafka: "Lock the door, then build the front desk"
       - "Not throwing Tom under the bus" — political awareness
       - Honest "I don't know" on PCI data flow (Aisha question)
       
       OUTCOMES:
       - James: "PCI is a formal initiative, not a side quest"
       - James: requested cluster-wide audit for unmanaged workloads
       - Sarah: assigned audit to user, due Friday (later moved to Thu EOD)
       - Lisa: suggested staged rollout windows for latency-sensitive deploys
       - Sarah (post-standup DM): "James said 'that's the kind of thinking 
         I want from the platform team'"

10:35  Aisha response: needs PCI-relevant findings flagged from audit
       Offered help with KMS key creation for Kafka
10:37  Replied to Aisha (Kafka CMK timeline, PCI scope details)
10:42  Replied to James (broader testing strategy — three layers)
10:45  Replied to Sarah (Friday audit, EKS runbook, timeboxing plan)
10:45  Cluster audit — automated sweep begins
11:45  Audit results: 9 unmanaged workloads, 4 PCI/PII critical  [AUDIT]
       Major discoveries: Kafka Connect, ETL pipeline, MongoDB Atlas,
       S3 PII exports, analytics dashboard with prod DB access
12:00  Additional investigation (Atlas, S3 bucket, connectors, topics)
12:15  NetworkPolicy v3 applied (data-analytics namespace)
       MongoDB Atlas egress blocked (accidental but correct)
12:15  Communications: Aisha (full data flow + gap list), 
       Sarah (Atlas escalation), Derek (staging cleanup), 
       Marcus (node-problem-detector)
12:35  Sarah responds: "More serious than expected." Don't contact Tom on PTO.
       She'll talk to Nina. Findings by EOD Thursday (moved up from Friday).
12:42  Aisha responds: Atlas is CDE boundary violation. Checking Atlas ownership.
       PCI emergency review scheduled: Thursday 2 PM.
       "Keep MongoDB blocked." Fix S3 encryption now.
12:55  Sent Atlas connection details to Aisha
       (password exposure in connect-configs topic flagged)
13:00  S3 remediation begins
       - KMS key created
       - Both buckets: encryption upgraded, logging enabled, versioning enabled
       - 108 + 1,847 objects re-encrypted
       - Lifecycle deferred to PCI review (contain, don't break)
13:30  Sarah: Wei Liu joining Thursday meeting. Nina apologetic, 
       willing to hand over. Tom's manager wasn't aware of Atlas.
13:40  Priya confirms Thursday 10 AM dry-run works
14:00  PCI findings document ~40% drafted
14:05  Derek: ServiceProfile PR today, CB PR Thursday
14:15  Marcus: will add NPD to ArgoCD this week
14:30  Continue findings document
14:47  PagerDuty: PaymentServiceHighErrorRate                [INC-6 START]
14:48  ACK. Post to #incidents. Drop everything.
14:49  Investigation: error shape (500s not 504s), endpoints, 
       timeline (matches deploy at 2:40, not NetworkPolicy at 12:15),
       downstream (payment gateway healthy), logs (ArithmeticException)
14:52  Root cause confirmed: BigDecimal.divide() without RoundingMode
14:53  ROLLBACK initiated (sha-f91a2b7 → sha-a8c71e3)
14:58  Rollback complete. All 8 replicas on old image.
15:01  Error rate back to baseline (0.12%)
15:05  Blast radius: 847 failed auths, 89 failed refunds, 34 failed status
15:10  David: Priya Kapoor has fix ready ~3:30 PM. HALF_EVEN confirmed.
       Found two more BigDecimal.divide() calls without RoundingMode.
15:10  Lisa: Budget burned 4.2%. Offers canary currency diversity alert.
15:10  Derek: Offers isRetryable:false on POST /api/v1/payments/authorize.

>>> CURRENT TIME: Wednesday 3:10 PM <<<
>>> INCIDENT 6 ROLLED BACK — FIX PENDING REDEPLOY <<<
>>> THREE PENDING RESPONSES: David, Lisa, Derek <<<
```

---

## OPEN ITEMS / PENDING WORK

### Active Tickets

```
PLAT-892  [In Progress]  EKS 1.28→1.29 upgrade runbook (due Friday)
                         Research done. Draft needs polish.
                         Getting squeezed by audit + incidents.
PLAT-901  [In Review]   Linkerd PR #347 — Priya pushed updates, needs re-review
PLAT-910  [To Do]       GP3 EBS migration (next sprint, skip data-analytics ns)
PLAT-915  [To Do]       Circuit breaker for order→inventory
                         Derek: ServiceProfile PR today, CB PR Thursday
PLAT-916  [To Do]       Canary analysis improvements — EXPANDED SCOPE
                         Original: traffic-volume gate
                         Added: transaction-diversity gate (currency pairs)
                         Now covers two incident classes (INC-1 + INC-6)
PLAT-917  [To Do]       Investigate inventory p99 at scale (assigned to Nina)
PLAT-918  [Done ✅]     Redis scale-up (completed Tue 2:15 AM)
PLAT-919  [Done ✅]     Redis cache alerting (PrometheusRules deployed Mon 3:06 PM)
PLAT-920  [To Do]       Quarterly refresh capacity planning gate
PLAT-921  [High]        Kafka cluster platform onboarding (Tom back next Mon)
                         Scope dramatically expanded by audit findings
PLAT-922  [Critical]    PCI scope assessment — EXPANDED
                         Now includes: Atlas, S3 PII, credential exposure,
                         CDE boundary violation
                         PCI emergency review: Thu 2 PM
[NEEDED]  [New]         Payment canary: currency-pair diversity gate
[NEEDED]  [New]         Audit all BigDecimal.divide() calls in payment codebase
[NEEDED]  [New]         Migrate secrets to External Secrets Operator
[NEEDED]  [New]         Add both S3 buckets to Terraform
[NEEDED]  [New]         S3 lifecycle/retention policies (after PCI review)
```

### Pending Deliverables

```
- PCI findings document → Aisha pre-read by Thu 1 PM, Sarah full report EOD Thu
  STATUS: ~40% drafted, interrupted by Incident 6
- EKS upgrade runbook → due Friday (PLAT-892)
  STATUS: Research done, needs polish. Getting squeezed.
- Linkerd staging dry-run → moved to Thursday 10 AM (was Thu afternoon)
- Payment service corrected fix → review + monitored redeploy ~3:30-4:30 PM today
- Incident 1 postmortem → updated with Incident 6 reference
- Redis Terraform PR → needs team review/merge
- Thursday PCI emergency review → present findings, decide containment strategy
```

### Pending Responses (IMMEDIATE)

```
1. David Okafor → Should Priya K. fix two other BigDecimal.divide() 
   calls in refund path? (pre-existing, work by accident)
2. Lisa Park → Canary currency-pair diversity alert + budget update
3. Derek Huang → isRetryable:false on POST /api/v1/payments/authorize
```

### Pending Communications

```
- Ryan Mitchell → post Redis upgrade confirmation (owed since Tue morning)
- Wednesday standup follow-ups still active
- Thursday PCI review prep
```

---

## TEAM STATE

```
Sarah Chen (manager):     Aware of full audit scope + Atlas finding. 
                          Talked to Nina. Backed PCI escalation. 
                          Wants findings EOD Thursday. James impressed 
                          at standup.
James Morrison (VP Eng):  Requested cluster audit. Aware of Kafka 
                          governance gap. Expects findings in leadership 
                          sync. "PCI is formal initiative, not side quest."
Marcus Webb (secondary):  Adding node-problem-detector to ArgoCD. 
                          Available for consult.
Priya Sharma (mid):       Linkerd PR updated. Staging dry-run Thu 10 AM.
                          Staging control plane already upgraded.
Jake Torres (mid):        Found original Kafka cluster. Working GP3 
                          migration. Warned to skip data-analytics.
Alex Kim (junior):        State lock self-resolved Mon. Jenkins 
                          investigation Tue. Growing.
Lisa Park (SRE):          Active support all week. Budget tracking.
                          Offering canary diversity alert. SLI update 
                          for degraded-mode responses.
David Okafor (Payment):   Hotfix rolled back. Priya Kapoor preparing 
                          corrected fix ~3:30 PM. Needs blast radius 
                          for CS team. Waiting on BigDecimal scope Q.
Derek Huang (Order):      ServiceProfile PR (isRetryable) today. 
                          CB PR Thursday. Cleaning up staging fixtures. 
                          Asking about payment authorize retryability.
Nina Petrov (Order lead): Informed of order-data-export finding by 
                          Sarah. Apologetic, willing to hand over. 
                          Joining Thu PCI review.
Ryan Mitchell (Frontend): Catalog issue resolved. Awaiting confirmation.
Jenny Wu (Merch):         Informed of Redis root cause. Looped into PLAT-920.
Aisha Rahman (Security):  Leading PCI response. Confirmed Atlas not 
                          org account. Scheduled Thu 2 PM emergency review.
                          Instructed: keep Atlas blocked, fix S3 now.
Wei Liu (Tom's manager):  Wasn't aware of Atlas. Reaching out to Tom 
                          for account details. Joining Thu PCI review.
Tom Chen (Data Eng):      On PTO until next Monday. Kafka/pipeline owner.
                          Wei contacting for Atlas details only.
Priya Kapoor (Payment):   Preparing corrected BigDecimal fix + tests.
                          HALF_EVEN confirmed with finance. ETA 3:30 PM.
```

---

## USER PERFORMANCE SUMMARY (6 Incidents, 5 Resolved, 1 Active)

### Grades

```
Incident 1 (SEV2 — Order timeout):           A+
Incident 2 (SEV4 — State lock):              A
Incident 3 (SEV3 — Redis capacity):          A+ (investigation), A (maintenance)
Incident 4 (SEV3 — Shadow Kafka/Pipeline):   A- (initial) → A+ (audit + PCI)
Incident 5 (SEV2 — Zombie CI pods):          [Grade from session]
Incident 6 (SEV2 — Payment currency bug):    A+ (so far — pending redeploy)

Monday triage:              A
Monday afternoon:           A-
PR Review (#347):           A+
Circuit Breaker Meeting:    A+
Maintenance Window:         A
Standup Presentation:       A+ ("best single performance in simulation")
Cluster Audit:              A+ ("most thorough single response")
PCI Escalation:             A+ ("knowing the boundary of your role 
                                 under pressure is rare")
```

### Strengths Demonstrated

```
ALL PREVIOUS STRENGTHS PLUS:

✓ Executive communication (VP-level risk framing, business language)
✓ Political awareness ("not throwing Tom under the bus", Nina framing)
✓ Audit methodology (scripted, cross-referenced, tiered classification)
✓ PCI/compliance reasoning (CDE boundary, data flow mapping)
✓ Credential exposure chain analysis (password → Kafka topic → no ACLs → exposure)
✓ Honest disclosure (accidental NetworkPolicy block — didn't claim intentional)
✓ S3 security hardening (KMS/CMK, logging, versioning)
✓ KMS key policy design (scoped principals, least privilege)
✓ Cross-language debugging (Java BigDecimal from platform eng perspective)
✓ Financial domain knowledge (banker's rounding, EU regulatory implications)
✓ Rollback trade-off calculus under time pressure
✓ Fast incident resolution (13 min ack-to-baseline on Incident 6)
✓ Systemic pattern recognition (Incident 1 + 6 = canary gap class)
✓ Deliverable management under competing demands
✓ "Contain, don't break" principle applied consistently
✓ Self-correction pattern: catches own potential mistakes (NetworkPolicy self-check)
✓ Influence without authority ("lock the door, then build the front desk")
```

### Gaps / Watch Items

```
ALL PREVIOUS GAPS PLUS:

- NetworkPolicy v3 egress to RDS uses ipBlock CIDR — fragile on Multi-AZ 
  failover. Documented as TODO. Not yet fixed.
  → Mitigation: mesh-level egress policy after Linkerd injection
  
- MongoDB Atlas block was accidental, not intentional. Correct outcome,
  but indicates the NetworkPolicy wasn't fully thought through for ALL 
  egress paths before applying. Should have enumerated all known egress 
  destinations (S3, RDS, Atlas, DNS) before writing the policy.
  → Self-disclosed honestly. Integrity > ego.

- S3 remediation: started applying 90-day lifecycle rule, then stopped.
  Correct to stop (contain, don't break), but the initial instinct to 
  apply a destructive change without confirming with consumers is a 
  recurring theme. Think before acting on changes that delete data.

- "If S3 bucket were public, that changes the calculus" — should have 
  stated this explicitly before deferring suspend/contain decision to 
  Aisha. Recommendation should be conditional on investigation results.

- EKS runbook getting squeezed. Friday deadline at risk if Thursday 
  PCI review generates urgent follow-ups. Has proactively identified 
  this but hasn't yet told Sarah.

PATTERN: Gaps remain at "precision under time pressure" level. 
         Consistently self-corrects. Integrity pattern strong.
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

## NON-INCIDENT WORK COMPLETED

```
STRATEGIC:
  ✅ Sarah Q1 priorities email — reordered with PCI addition
  ✅ Circuit breaker design meeting — led technical session
  ✅ Kafka governance escalation — framed as process gap
  ✅ Standup presentation to VP — risk-first framing, PCI insertion
  ✅ Cluster-wide audit — scripted, 9 findings, 4 PCI critical
  ✅ PCI escalation — full data flow mapping, gap analysis, containment
  ✅ MongoDB Atlas discovery + containment
  ✅ Broader testing strategy proposal (three layers) for James

PR REVIEWS:
  ✅ Linkerd PR #347 — thorough review, 2 blocking + 5 non-blocking
  🔲 Priya's updated PR — needs re-review

OPERATIONAL:
  ✅ EKS 1.29 upgrade runbook — research done, needs polish
  ✅ Redis alerts deployed (PLAT-919)
  ✅ Redis maintenance executed (PLAT-918)
  ✅ Kafka safety nets: PDB + NetworkPolicy v1 → v2 → v3
  ✅ S3 security remediation (KMS, logging, versioning — both buckets)
  ✅ KMS key created with scoped policy
  ✅ Change request authored and executed (Redis CR)
  ✅ Staging failover test completed
  ✅ Cluster audit script written and executed
  ✅ Audit report ~40% drafted

MENTORING:
  ✅ Alex Kim — guided through Terraform state lock
  ✅ Jake Torres — warned about data-analytics PVs, praised for catch
  ✅ Priya Sharma — PR review with offer to pair, dry-run scheduled
```

---

## WHERE TO RESUME

```
SIMULATION TIME:  Wednesday 3:10 PM EST, Week 1
ON-CALL:          User (primary), Marcus (secondary)

ACTIVE SITUATION: Incident #6 rolled back, fix pending redeploy.
                  Three responses pending (David, Lisa, Derek).
                  Payment service corrected fix ETA 3:30 PM.

IMMEDIATE ACTIONS:
  1. Respond to David (BigDecimal scope question)
  2. Respond to Lisa (canary diversity alert)
  3. Respond to Derek (isRetryable on payment authorize)
  4. Review Priya Kapoor's fix when ready (~3:30 PM)
  5. Monitor corrected fix redeploy (~4:00 PM)

TODAY REMAINING:
  - Payment fix redeploy + manual canary verification
  - PCI findings document (continue, ~40% → target 80%)
  - Ryan Mitchell Redis confirmation
  - EOD notes

THURSDAY:
  10:00-12:00  Linkerd staging dry-run with Priya Sharma
  12:00-13:00  PCI findings document finalize
  13:00        Pre-read distributed to Aisha
  14:00-15:30  PCI emergency review (Aisha, Sarah, Wei Liu, Nina, User)
  Remaining:   Action items from PCI review + EKS runbook

FRIDAY:
  - EKS upgrade runbook due (PLAT-892) — AT RISK if Thu generates urgent work
  - Audit report finalization if needed
  - PCI review follow-ups
  - Derek's circuit breaker PR review

NEXT MONDAY:
  - Tom Chen returns → Kafka onboarding meeting
  - Atlas account decommissioning (pending PCI review decisions)

PENDING INJECTION POINTS:
  - Security, networking, observability, cascading failures, DR, 
    simultaneous incidents, certificate/secrets, cost/billing, 
    ambiguous root cause all untested
  - Night-time / fatigue-state incident testing pending
  - 94 more incidents to cover
```

---

**END OF HANDOFF DOCUMENT — v3**

**Ready to continue simulation. Three pending responses + payment fix redeploy active. Incidents will continue to be injected.**
