# PHASE 8 SIMULATION — HANDOFF DOCUMENT (v6)
# CHAOS ENGINE EDITION — REVISED TAXONOMY

---

## SIMULATION VERSION: v2 — CHAOS ENGINE ACTIVE

All 12 Chaos Engine rules in effect. Consequence branching is compounding. Revised incident taxonomy in effect for remaining 90 incidents.

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

## INCIDENTS/ISSUES COVERED: 10 of 100

---

## INCIDENT TRACKER

| # | Severity | Type | Category | Description | Status | Grade |
|---|----------|------|----------|-------------|--------|-------|
| 1 | **SEV2** | Deploy-induced timeout | Deploy/Release | Order service 504s from aggressive inventory_client_timeout (800ms) at peak traffic. | ✅ RESOLVED — Interim fix 2000ms. CB deployed. | **A+** |
| 2 | **SEV4** | Stale Terraform lock | IaC/Terraform | Alex blocked by stale DynamoDB state lock from crashed Jenkins job. | ✅ RESOLVED — Guided Alex to self-resolve. | **A** |
| 3 | **SEV3** | Capacity — Redis memory pressure | Database | Quarterly catalog refresh exceeded Redis maxmemory. Hit rate dropped 94%→87%. | ✅ RESOLVED — Node upgraded r6g.large→r6g.xlarge. | **A+ / A** |
| 4 | **SEV3** | Shadow infrastructure | Cross-team/Process | Unmanaged Kafka cluster + full data pipeline. Scope expanded: Kafka Connect, ETL CronJob, MongoDB Atlas, S3 PII exports. | ✅ MITIGATED — NetworkPolicy v3, S3 hardened, Atlas blocked. PCI review complete. GDPR notification filed. | **A- → A+** |
| 5 | **SEV2** | Zombie Jenkins agents | CI/CD | Jenkins agent pods stuck running 3-6+ hours, consuming entire Karpenter CI pool. | ✅ RESOLVED — Build discard config + heap monitoring. | **[Grade from session]** |
| 6 | **SEV2** | Deploy-induced code bug | Deploy/Release | Payment service BigDecimal.divide() without RoundingMode. GBP/SEK/DKK/NOK→EUR 100% failing. | ✅ RESOLVED — Rolled back, corrected fix deployed sha-c4d82e1. | **A+** |
| 7 | **SEV2** | Elasticsearch outage | Kubernetes/Stateful | ES data node evicted (ephemeral storage), StatefulSet blocked by ResourceQuota, scheduling constrained by PVC node affinity. | ✅ RESOLVED — Quota fixed, node freed, cluster GREEN. ILM deployed. | **Pending** |
| 8 | **SEV2** | RDS connection exhaustion | Database | analytics_reader with no connection limit, 256 connections from analytics dashboard + manual report pod. | ✅ RESOLVED — Dashboard scaled to 0, queries killed, CONNECTION LIMIT 20 set. | **Pending** |
| 9 | **SEV3** | TLS cert renewal failure | Certificate/Secrets | Wildcard cert for *.novamart.com failing renewal for 23 days. IAM key deactivated by security automation. | ✅ RESOLVED — New IAM key, cert renewed. IRSA migration planned. | **Pending** |
| 10 | **SEV1** | Data exfiltration — former employee | Security | 4-month unauthorized access campaign. 284,847 customers. Confirmed exfiltration to transfer.sh. | ✅ CONTAINED — Investigation phase. GDPR Art. 33 × 2 filed. Art. 34 sent. Board briefed. Law enforcement engaged. | **Pending** |

---

## DETAILED INCIDENT RECORDS

### INCIDENT 1: Order Service Timeout (SEV2) ✅

```
TRIGGER:      PagerDuty — HighErrorBudgetBurn_OrderService (3.2x 1h burn rate)
FIRED:        Mon Wk1 8:47 AM
ACKED:        Mon Wk1 9:04 AM
RESOLVED:     Mon Wk1 9:30 AM
TTR:          ~26 min from ack
BUDGET BURN:  74% → 65.2% (8.8% consumed)

ROOT CAUSE:
  Deploy sha-e7b31d4 set inventory_client_timeout_ms: 800ms.
  Inventory p99 = 1.08s at production scale. Canary passed at 
  2,400 rpm (low morning traffic), failed at 6K+ rpm.

FIX: inventory_client_timeout_ms: 2000ms, retries: 1

FOLLOW-UP STATUS (all complete):
  PLAT-915 — Circuit breaker: DEPLOYED TO PRODUCTION (Thu Wk2) ✅
    Config: failureRateThreshold: 25, slowCallDurationThreshold: 2000ms,
    slidingWindowSize: 100 (COUNT_BASED).
    Low-confidence fallback: available:true (product-approved).
    stockCache: PR approved (#354), deploying Fri Wk2.
    Three-tier fallback: live → cached stock → blind accept.
    Metrics: orders_with_cached_stock_total (tier 2)
             orders_without_stock_verification_total (tier 3)
    Production metrics after 16h at 100%:
      Error rate: 0.018% (improved from 0.03% baseline)
      p99: 395ms (improved from 438ms baseline)
      CB opens: 0. Degraded responses: 0.

  PLAT-916 — Canary analysis improvements: DONE ✅ (Lisa merged Fri Wk1)
  PLAT-917 — Investigate inventory p99: assigned Nina (not started)
  PLAT-923 — stockCache: PR #354 approved, staging validated, deploying Fri Wk2

  Lisa dual-SLI (platform-infra#353):
    Primary SLI: error rate + latency (existing)
    Secondary SLI: orders_degraded_mode_ratio (2% threshold, 7-day window)
    MERGED AND DEPLOYED ✅
```

### INCIDENT 2: Terraform State Lock (SEV4) ✅

```
TRIGGER:      Slack — Alex Kim
REPORTED:     Mon Wk1 9:01 AM
RESOLVED:     Mon Wk1 9:42 AM
ROOT CAUSE:   Jenkins CI job crashed at 2:14 AM, DynamoDB lock held.
FIX:          force-unlock after verifying no active applies.
```

### INCIDENT 3: Redis Cache Pressure (SEV3) ✅

```
TRIGGER:      Slack — Ryan Mitchell
REPORTED:     Mon Wk1 11:02 AM
RESOLVED:     Tue Wk1 2:15 AM (node upgrade r6g.large→r6g.xlarge)
CONFIRMED:    Tue Wk1 8:55 AM

ROOT CAUSE:   Quarterly catalog refresh: 182K products → 435K 
              cache keys exceeded Redis maxmemory.

POST-UPGRADE STATUS (as of Thu Wk2):
  Hit rate: 94.1%, Evictions: 0/min, Memory: 54%
  Cost: +$183/month

FOLLOW-UP STATUS:
  PLAT-918 — Scale Redis: DONE ✅
  PLAT-919 — Redis alerting: DONE ✅ (confirmed correctly scoped)
  PLAT-920 — Quarterly refresh capacity planning gate: To Do
```

### INCIDENT 4: Shadow Kafka / Data Pipeline (SEV3) ✅ Fully Mitigated — GDPR FILED

```
TRIGGER:      Jake Torres discovered unknown StatefulSets
REPORTED:     Mon Wk1 2:38 PM
MITIGATED:    Wed Wk1 12:15 PM (NetworkPolicy v3, S3 hardened, Atlas blocked)
PCI REVIEW:   Thu Wk1 2:00-2:38 PM — COMPLETED
GDPR FILED:   Fri Wk1 1:15 PM — SA-2024-0119-NM
SA RESPONSE:  Thu Wk2 3:47 AM CET — acknowledged, requesting 4 items
              Response due: Feb 5

FULL PIPELINE STATUS (as of Thu Wk2):
  Component 1: Kafka brokers (6) + ZooKeeper (3)
    ✅ ECR images (migrated Tue Wk2)
    ✅ Linkerd mesh injection (Wed Wk2)
    ✅ NetworkPolicy v3 restricting access
    ✅ PDB applied
    Running stable on approved infrastructure.

  Component 2: Kafka Connect (confluent cp-kafka-connect:7.5.3)
    ✅ ECR image (migrated Tue Wk2)
    ✅ Linkerd mesh injection (Wed Wk2)
    orders-sink-s3: RUNNING ✅
    payments-sink-mongo: FAILED (blocked by NetworkPolicy — expected)

  Component 3: ETL CronJob (etl-daily-aggregation)
    STATUS: RUNNING (exit code 1 expected — Atlas sink blocked)
    Alertmanager silence applied (7-day rolling)

  Component 4: Analytics Dashboard
    1 replica, HPA frozen at max 1
    analytics_reader PASSWORD ROTATED (Mon Wk2 night)
    Currently broken (expected — Metabase connection uses old password)
    Needs reconnection after security investigation stabilizes.

  Component 5: MongoDB Atlas (EXTERNAL) — 🔴 BLOCKED + PRESERVED
    42,847 documents, ~25-30K unique EU customers
    LEGAL HOLD: Do not purge until Rachel Torres clears
    STATUS: BLOCKED (NetworkPolicy egress deny since Wed Wk1)

  Component 6: Order Data Export CronJob (order-prod namespace)
    ✅ ONBOARDED to ArgoCD by Nina (Sat Wk1)
    ✅ Migrated to KMS-encrypted bucket path
    ✅ Running successfully

R-6a MILESTONE STATUS (committed Fri Jan 26):
  ✅ ECR migration (Tue Wk2)
  ✅ Mesh injection (Wed Wk2)
  ✅ Terraform imports — 3 S3 buckets (Wed Wk2, ahead of schedule)
  ⏳ Monitoring dashboards (Tom, Fri Wk2)
  ⏳ Runbook (Tom, Fri Wk2)
  ON TRACK — completing Friday.

CONTAINMENT STATUS (as of Thu Wk2):
  ✅ Atlas egress: BLOCKED + data preserved per Legal
  ✅ S3 exports: ENCRYPTED (KMS/CMK, logging, versioning)
  ✅ S3 buckets: ALL THREE IMPORTED TO TERRAFORM ✅
  ✅ Kafka network: RESTRICTED (NetworkPolicy v3)
  ✅ Kafka images: ECR (Docker Hub eliminated)
  ✅ Kafka mesh: Linkerd mTLS between all pods
  ✅ All S3 writers: KMS permissions verified
  ✅ analytics_reader: CONNECTION LIMIT 20, password rotated
  ✅ data-analytics namespace: ALL non-platform users read-only (RBAC)
  ✅ Nina's CronJob: onboarded to ArgoCD
  ⚠️ RDS egress: ipBlock CIDR-based (fragile on Multi-AZ failover)

KMS KEY:
  Alias: alias/novamart-data-exports
  Key policy (FINAL):
    platform-admin (full),
    order-data-export-role (encrypt/generate),
    data-analytics-role (decrypt),
    kafka-connect-s3-writer (encrypt/generate)

S3 BUCKETS (all three now in Terraform):
  novamart-data-exports:
    ✅ Public access blocked, KMS/CMK, logging, versioning
    ✅ Imported to Terraform (Wed Wk2)
    ❌ No lifecycle/retention policy

  novamart-analytics-datalake:
    ✅ Public access blocked, KMS/CMK, logging, versioning
    ✅ Imported to Terraform (Wed Wk2)
    ❌ No lifecycle policy

  novamart-db-backups:
    ✅ Public access blocked, KMS/CMK, logging, versioning
    ✅ Imported to Terraform (Wed Wk2)
    ⚠️ EVIDENCE HOLD — no lifecycle rules, no object deletion

SUPERVISORY AUTHORITY RESPONSE (SA-2024-0119-NM):
  Received: Thu Wk2 3:47 AM CET
  Requesting:
    1. Technical description of drift detection system
    2. Evidence it's tested and operational
    3. Complete list of ALL personal data processing involving 
       external or unmanaged infrastructure
    4. Timeline for all remediation items
  Response due: February 5
  STATUS: Rachel drafting. AWS audit table (Marcus) feeds item 3.
  NOTE: SA is reviewing both filings (Atlas + Jenkins) together.

GDPR STATUS:
  Art. 33 #1 (Atlas): FILED Fri Wk1. SA-2024-0119-NM. SA acknowledged.
  Art. 33 #2 (Jenkins/exfiltration): FILED Tue Wk2. SA-2024-0122-NM-002.
  Art. 34 (customer notification): SENT Thu Wk2 10:00 AM. 284,847 emails.
  UK ICO: FILED Thu Wk2. ICO-2024-NM-0122.

COMMITTED REMEDIATION TIMELINE (board-approved):
  ┌────────────────────────────────┬──────────────┬─────────┬──────────────┐
  │ Control                        │ Date         │ Owner   │ Status       │
  ├────────────────────────────────┼──────────────┼─────────┼──────────────┤
  │ Drift detection CronJob        │ Mon Jan 22   │ User    │ ✅ DONE      │
  │ Kafka pipeline SECURED (R-6a)  │ Fri Jan 26   │ User    │ ✅ On track  │
  │ Image admission (warn mode)    │ Fri Jan 26   │ Jake    │ ⏳ Prod Fri  │
  │ Image admission (enforce mode) │ Fri Feb 2    │ Jake    │ Staging ✅   │
  │ Kafka FULLY ONBOARDED (R-6b)   │ Fri Feb 9    │ Tom     │ In progress  │
  │ Self-service pipeline template │ Mid-Feb      │ User    │ Not started  │
  │ Default-deny NetworkPolicy     │ Mid-Feb      │ User    │ Not started  │
  │ IAM lifecycle automation       │ End Feb      │ Aisha   │ Not started  │
  │ SSH key mgmt through IdP       │ End Feb      │ Aisha   │ Not started  │
  │ Egress gateway                 │ End of Q1    │ User    │ Not started  │
  │ AWS resource tagging enforce   │ End of Q1    │ Marcus  │ Not started  │
  └────────────────────────────────┴──────────────┴─────────┴──────────────┘

TOM CHEN SPRINT STATUS (Wk2):
  ✅ Tue: ECR image migration (Kafka, ZK, Connect)
  ✅ Wed: Linkerd mesh injection
  ✅ Wed: Terraform imports (3 S3 buckets, ahead of schedule)
  ⏳ Thu: Monitoring dashboards (Grafana, Marcus reviewed labels)
  ⏳ Fri: Runbook + R-6a milestone close
  Tom quote: "This has been the most productive three days 
  I've had at NovaMart."
```

### INCIDENT 5: Zombie Jenkins Agents / CI Pool Exhaustion (SEV2) ✅

```
TRIGGER:      David Okafor reports builds failing since 7 AM
REPORTED:     Tue Wk1 7:45 AM
RESOLVED:     Tue Wk1
ROOT CAUSE:   JVM memory leak in Jenkins agent pods.
FIX:          Build discard config + heap monitoring.
STATUS:       Jenkins heap stable (5+ days). 6 agent pods (normal).
⚠️ UNVERIFIED: Build discard may be too aggressive for >30 builds.
```

### INCIDENT 6: Payment Service Currency Bug (SEV2) ✅

```
TRIGGER:      PagerDuty — PaymentServiceHighErrorRate
FIRED:        Wed Wk1 2:47 PM
ACKED:        Wed Wk1 2:48 PM
ROLLBACK:     Wed Wk1 2:53-2:58 PM
FIX DEPLOYED: Wed Wk1 4:28 PM (sha-c4d82e1)
TTR:          ~13 min ack-to-baseline (rollback)
BUDGET BURN:  91.3% → 87.1% (4.2% consumed)

BLAST RADIUS:
  847 failed payments, 89 failed refunds, 34 failed status checks
  ~500-700 unique customers affected

ROOT CAUSE:   BigDecimal.divide() without RoundingMode in 
              CurrencyService.convertAmount(). Non-terminating 
              decimal pairs throw ArithmeticException.

CORRECTED FIX: HALF_EVEN, scale 2, all three call sites.
  36 new tests covering all 14 EU currency pairs.

ROUNDING RECONCILIATION: COMPLETE ✅
  $47.23 discrepancy resolved. Martha/David one-time adjustment.
  Three rounding behaviors in one week's data documented.
  All future transactions: HALF_EVEN consistent.

VELOCITY FREEZE: Payment-service budget 86.0%. Freeze in effect.
```

### INCIDENT 7: Search / Elasticsearch Outage (SEV2) ✅

```
TRIGGER:      PagerDuty — SearchServiceHighErrorRate (>10% for 5 min)
FIRED:        Fri Wk1 5:47 AM
ACKED:        Fri Wk1 5:53 AM
CLUSTER GREEN: Fri Wk1 6:52 AM
TTR:           ~65 min ack-to-GREEN
PEAK ERROR:    19.8%

ROOT CAUSE CHAIN:
  1. No ILM policy → unbounded index growth → ephemeral storage exhaustion
  2. Node evicted es-data-3
  3. Karpenter preempted for its own workload
  4. StatefulSet blocked by ResourceQuota (64Gi limit)
  5. Quota fix → PVC AZ-locked to original node
  6. Freed capacity by evicting Kibana + 2 Jenkins agents

CRITICAL MOMENT:
  es-data-2 had 4/5 consecutive readiness probe failures.
  Scaled search-service 6→2 to reduce query pressure.
  es-data-2's 5th probe PASSED (by one cycle).

FOLLOW-UP STATUS (as of Thu Wk2):
  ✅ ILM policy DEPLOYED (Wed Wk2, Priya)
     Hot: 30 days, Warm: read-only + force merge, Delete: 180 days (pending legal)
     34 indices transitioned to warm
     Heap: es-data-2 81% → 53% ✅
     p99 latency: 1.1s → 680ms ✅
     Incident 7 root cause PERMANENTLY RESOLVED

  ✅ PDB replaced with correct labels (maxUnavailable: 1)
  ✅ ResourceQuota right-sized (80Gi requests, annotated)
  ✅ ILM monitoring dashboard (Priya building)

  ⏳ Search cluster interim ownership proposal: drafted, sent to Sarah
     Primary: Priya. Secondary: User.
  ⏳ ArgoCD onboarding for search namespace (Priya, by Feb 2)
  ⏳ gp2 → gp3 migration (Jake, next sprint)
  ⏳ ES pod PriorityClass (prevent Karpenter preemption)
  ⏳ Ephemeral storage limits on stateful workloads
  ⏳ Search SLO definition (Priya + Lisa, post-incident)
  ⏳ Runbook update (replace Carlos's stale Jun 2023 doc)
```

### INCIDENT 8: RDS Connection Exhaustion (SEV2) ✅

```
TRIGGER:      PagerDuty — RDSHighConnectionCount (>350)
FIRED:        Mon Wk2 11:07 AM
ACKED:        Mon Wk2 11:11 AM
RECOVERED:    Mon Wk2 11:22 AM
TTR:          ~11 min from ack
BUDGET BURN:  Order-service: 64.5% → 61.8% (2.7% consumed)

ROOT CAUSE:
  analytics_reader: NO CONNECTION LIMIT.
  Dashboard HPA scaled to 2 replicas, no connection pooling.
  Wei's manual report pod: 12 parallel workers.
  Combined: 256 analytics_reader connections.
  Feedback loop: more queries → HPA scales → more connections.

FIX: Dashboard scaled to 0, queries killed, CONNECTION LIMIT 20.

DISCOVERY CHAIN (led to Incident 10):
  47 NAT gateway connections → traced to Metabase EC2 →
  found legacy Jenkins EC2 → found admin/admin credentials →
  found unauthorized logins from peered VPC →
  SECURITY INCIDENT

POST-INCIDENT STATUS:
  ✅ analytics_reader CONNECTION LIMIT 20
  ✅ analytics_reader PASSWORD ROTATED (Mon Wk2 night — security incident)
  ✅ Dashboard restored at 1 replica, HPA frozen at max 1
  ✅ Wei's RBAC restricted to read-only in data-analytics
  ✅ Wei's report re-run completed (single-threaded)
```

### INCIDENT 9: TLS Certificate Renewal Failure (SEV3) ✅

```
TRIGGER:      Slack — CertificateExpiringWithin7Days
FIRED:        Mon Wk2 2:00 PM
CERT RENEWED: Mon Wk2 2:27 PM
NEW EXPIRY:   Apr 21, 2024

ROOT CAUSE:   Security automation deactivated cert-manager IAM key
              (58 days since last use, >30 day threshold).
              Renewal failing silently for 23 days.
              No alert for renewal failure — only expiry alert.

FIX: New IAM key, updated secret + ClusterIssuer, cert renewed.

FOLLOW-UP STATUS:
  ⏳ IRSA migration for cert-manager (this week — not started)
  ⏳ Alert for renewal failures (not just expiry)
  ✅ Aisha adding pre-deactivation notification to security automation
  ⏳ ACME account: update from Carlos email to current employee
```

### INCIDENT 10: Data Exfiltration — Former Employee (SEV1) ✅ CONTAINED

```
TRIGGER:      Marcus Webb investigation of legacy EC2 instances
DISCOVERED:   Mon Wk2 2:42 PM
IR DECLARED:  Mon Wk2 2:50 PM (Aisha)
CONTAINED:    Mon Wk2 6:58 PM (all credentials rotated, all paths severed)
INVESTIGATION: CrowdStrike — final report delivered Wed Wk2 5:45 AM
GDPR Art. 33:  FILED Tue Wk2 12:03 PM (SA-2024-0122-NM-002)
GDPR Art. 34:  SENT Thu Wk2 10:00 AM (284,847 emails)
UK ICO:        FILED Thu Wk2 (ICO-2024-NM-0122)
LAW ENFORCEMENT: Police report filed Tue Wk2. FBI CFAA referral by Fri Wk2.
BOARD:         Briefed Thu Wk2 3:00 PM. Remediation approved.

REFERENCE: SEC-2024-0122-001

ACTOR PROFILE (CrowdStrike assessment):
  Single actor. Former employee or contractor with pre-existing 
  system knowledge. "Technically competent but not sophisticated."
  Strong circumstantial evidence: Carlos Mendez (departed Sep 2023,
  went to competitor). Personal IAM key + personal SSH key + 
  personal infrastructure + system knowledge + targeted EU data.
  Not formally attributed pending law enforcement.

COMPLETE TIMELINE:
  Jun 2023      Carlos creates IAM user, SSH keys, cert-manager infra
  Aug 14 2023   Carlos creates: staging VPC, VPC peering, NAT/IGW,
                staging Metabase, carlos-dev-scratch S3 bucket,
                reporting-db-prod RDS, report-generator Lambda,
                etl-trigger Lambda, production Metabase (analytics)
  Sep 15 2023   Carlos departs. Okta disabled, email disabled.
                IAM user NOT deactivated. SSH keys NOT deleted.
                Infrastructure NOT decommissioned.
                Exit interview: mentions "staging stuff to clean up."
                Wei says "we'll handle it." No follow-up.
  Oct 8         Actor launches temp EC2 from staging Metabase (172.31.4.89)
                Using carlos-staging-key. First post-departure activity.
  Oct 10        Direct psql to orders-db-prod (customers table)
  Oct 12        pg_dump customers → local CSV
  Oct 15        CSV uploaded to carlos-dev-scratch (4.2 MB)
                Exfiltrated via NAT gateway (6.8 MB spike confirmed)
                Metabase export: EU customers (168,442 rows)
                Metabase export: full customer profiles (271,384 rows)
  Oct 18        Temporary EC2 terminated
  Nov 8         Metabase export: high-value customers (12,847 rows)
  Nov 11        Actor connects to reporting-db-prod
                Metabase export: support tickets (4,291 rows)
  Nov 12        customer_orders CSV exported + exfiltrated (31.2 MB spike)
  Jan 8 2024    Actor logs into Jenkins (2 sessions)
  Jan 14        Metabase export: stripe webhook payment methods (7,104 rows)
                Metabase export: product reviews (18,934 rows)
                Metabase query: stripe_webhook_events LIMIT 100
  Jan 15        ESCALATION: Full DB dumps downloaded via Jenkins + Metabase
                3 dumps (Sep, Oct, Nov) downloaded twice (Jenkins role + Metabase role)
                Exfiltrated via transfer.sh (1.16 GB NAT spike)
                curl -T uploads to transfer.sh (public file sharing)
  Jan 19        November dump re-downloaded + exfiltrated (524 MB spike)
                curl -T to transfer.sh
                Carlos personal IAM key last used (s3:GetObject)
  Jan 22 9:14a  Last Jenkins login. No API activity recorded.
  Jan 22 2:40p  Marcus changes Jenkins password (access cut)
  Jan 22 2:50p  Aisha declares security incident (SEC-2024-0122-001)
  Jan 22 3:01p  backup_admin password rotated + CONN LIMIT 0
  Jan 22 3:15p  James bridge call. Full scope briefed.
  Jan 22 3:22p  carlos-dev-key deleted from EC2
  Jan 22 3:24p  VPC peering DELETED (production network path severed)
  Jan 22 3:26p  Staging Metabase + Jenkins SGs locked to platform IPs
  Jan 22 5:41p  VPC flow logs enabled on legacy-staging VPC
  Jan 22 6:00p  CrowdStrike call. David Park begins imaging.
  Jan 22 6:47p  Production Metabase SG locked to platform IPs
  Jan 22 6:55p  reporting-db-prod: password rotated, staging VPC CIDR revoked
  Jan 22 6:58p  analytics_reader password rotated
  Jan 23 12:14a orders-db-prod MASTER password rotated (Aisha — never rotated before)
  Jan 23 12:22a jenkins-deployer IAM key DEACTIVATED (AdministratorAccess — unused by actor, LUCKY)
  Jan 23 12:24a jenkins-deployer: confirmed no usage from attacker IPs
  Jan 23 6:00a  CrowdStrike full report delivered (94 pages)
  Jan 23 9:17a  reporting-db-prod assessed: NO full card numbers, NOT PCI
  Jan 23 10:03a Carlos personal IAM key deactivated (Marcus found, Aisha deactivated)
  Jan 23 10:44a Secondary ENI on staging Metabase: public SSH revoked (missed in initial lockdown)
  Jan 23 10:51a Staging Metabase STOPPED
  Jan 23 10:51a Legacy Jenkins STOPPED
  Jan 23 10:54a Production Metabase STOPPED
  Jan 23 12:03p GDPR Art. 33 #2 FILED (SA-2024-0122-NM-002)
  Jan 24 4:00a  UK ICO FILED (ICO-2024-NM-0122)
  Jan 24 10:00a Customer notification sent (284,847 emails, staggered over 2h)
  Jan 24 3:00p  Board briefed. Remediation approved. 2 headcount approved.

EXFILTRATION — CONFIRMED:
  Method: curl -T to transfer.sh (public file sharing service)
  NAT gateway metrics correlate to every known access event:
    Oct 15 3PM:   6.8 MB  (matches customer_emails CSV — 4.2 MB + overhead)
    Nov 12 2PM:   31.2 MB (matches customer_orders CSV — 28.7 MB)
    Jan 15 3-4PM: 1.16 GB (matches 3x database dumps)
    Jan 19 10AM:  524 MB  (matches 1x database dump re-download)
  transfer.sh links expire 14 days. Jan 15 links expired.
  Jan 19 link may still be live (~expires Feb 2). Cannot take down.

DATA SCOPE:
  Confirmed accessed and exported:
    284,847 unique customers (union of Sep/Oct/Nov backup dumps)
      EU: 168,442 (59.1%)
      UK: 41,207 (14.5%)
      US: 58,934 (20.7%)
      Other: 16,264 (5.7%)
    Additional Metabase exports:
      EU customers filtered by country (168,442 rows)
      High-value customers >$500 orders (12,847 rows)
      Customer support tickets since Sep (4,291 rows)
      Stripe webhook payment methods (7,104 rows — billing details)
      Product reviews (18,934 rows)
      Full customer profiles (271,384 rows)

  PII categories: names, emails, phones, shipping addresses,
    billing addresses (from Stripe webhooks), order history,
    purchase amounts, customer support ticket contents,
    product reviews, customer segmentation/lifetime value,
    partial card info (last4, brand, expiry — NOT full PAN)

  NOT exposed:
    Full payment card numbers (PANs) — Stripe never sends in webhooks
    CVV/CVC codes
    Bank account information
    Customer passwords/authentication tokens
    Payment-service or inventory-service databases (read-only access
      via Metabase but no evidence of export beyond confirmed items)

REPORTING-DB-PROD ASSESSMENT:
  PostgreSQL, db.t3.medium, 100GB, UNENCRYPTED
  Created: Aug 14, 2023 by Carlos
  14 tables: customer_profiles, customer_orders_summary,
    customer_support_tickets, daily_revenue, inventory_snapshots,
    monthly_cohort_analysis, order_items_denormalized,
    payment_transactions, product_performance, product_reviews,
    refund_history, shipping_events, stripe_webhook_events,
    supplier_catalog
  stripe_webhook_events: 147,283 raw payloads (jsonb)
    Contain billing_details (name, email, phone, physical address)
    31,847 unique Stripe customers with billing details
    NO full card numbers (Stripe design — never sends PANs)
  PCI ASSESSMENT: NOT A PCI INCIDENT ✅
  Actor queried payment data (Q-227, Jan 14) and EXPORTED via
    Metabase CSV (7,104 rows) — initial "no export" assessment
    was WRONG (corrected within 2 hours, Rule 4)

CREDENTIALS COMPROMISED AND ROTATED (9 total):
  ✅ backup_admin (orders-db-prod) — rotated + CONN LIMIT 0
  ✅ analytics_reader (orders-db-prod) — rotated + CONN LIMIT 20
  ✅ reporting_admin (reporting-db-prod) — rotated via RDS
  ✅ orders-db-prod root/master — rotated (Aisha, midnight)
  ✅ etl_processor (orders-db-prod) — rotated + CONN LIMIT 0
  ✅ metabase_reader (inventory-db-prod) — rotated + CONN LIMIT 0
  ✅ jenkins-deployer IAM key — DEACTIVATED (AdministratorAccess)
  ✅ carlos.mendez IAM key — DEACTIVATED
  ✅ carlos-dev-key SSH key — DELETED from EC2

INFRASTRUCTURE LOCKED DOWN:
  ✅ Legacy Jenkins (i-0e8g3f): SG locked → STOPPED
  ✅ Staging Metabase (i-0c4d5e6f): SG locked, secondary ENI revoked → STOPPED
  ✅ Production Metabase (i-0d7f2e): SG locked → STOPPED
  ✅ VPC peering: DELETED (permanent)
  ✅ reporting-db-prod: staging VPC CIDR revoked from SG
  ✅ All three instances: EBS snapshots preserved for forensics
  ✅ carlos-dev-scratch S3: evidence hold, CrowdStrike has contents
  ✅ novamart-db-backups S3: hardened (KMS, logging), evidence hold

  Remaining SSH keys (SG-blocked, deletion pending CrowdStrike clearance):
    carlos-staging-key (used by Jenkins + staging Metabase)
    novamart-analytics-key (used by production Metabase)

CROWDSTRIKE FINDINGS SUMMARY:
  Report: 94 pages + 12-page exec summary + evidence appendix
  Status: FINAL (Wed Wk2)
  
  Key findings:
  - Single actor, strong Carlos attribution
  - 203.0.113.47: residential ISP IP, SSH source for all sessions
  - Access path: Home → Elastic IP → public subnet ENI → staging Metabase
    → VPC peering → production resources
  - bash_history: psql commands, pg_dump, aws s3 cp, curl -T transfer.sh
  - Metabase H2 database: 347 saved queries, 58 post-departure
    - 31 against orders-db-prod
    - 22 against reporting-db-prod
    - 5 against inventory-db-prod (same RDS as inventory service — NOT shadow)
  - 7 Metabase CSV exports (post-departure)
  - Deliberate targeting: EU customers (Q-201 filters by country code),
    high-value customers (Q-215 filters orders >$500),
    support tickets (Q-238), payment methods (Q-227)
  - "Consistent with data collection for potential sale or competitive use"
  - Actor left extensive forensic traces (bash_history not cleared,
    transfer.sh for exfil = low sophistication indicator)
  - jenkins-deployer IAM key (AdministratorAccess) found but NOT USED
    by actor. Lucky break — would have been full AWS account compromise.
  - Terminated instance (Oct 8-18): launched from staging Metabase,
    carlos-staging-key, first post-departure infra activity
  - Carlos personal IAM key used for Jan 15+19 S3 downloads
    (not just instance profiles)
  - David Park closing note: "The technical response by the NovaMart 
    team was among the fastest and most thorough we've seen."

CUSTOMER NOTIFICATION RESULTS (Thu Wk2):
  Emails sent: 284,847
  Emails delivered: 276,102 (97.1%)
  Bounced: ~8,745 (postal mail notification for those with addresses)
  Kroll enrollments (first day): 24,847 (8.7%)
  Support contacts (first day): 8,291 (2.9%)
  Press inquiries: 3 (handled with holding statement)
  Social media mentions: ~40 (neutral to sympathetic)
  Customer sentiment: measured, no viral backlash
  Credit monitoring vendor: Kroll, $2.80/customer, ~$797K total

BOARD MEETING OUTCOMES (Thu Wk2 3:00 PM):
  Remediation timeline: APPROVED
  2 new platform engineering headcount: APPROVED
  30-day follow-up board update: SCHEDULED
  CEO quote (via EA): "This is the most competent incident response 
    I've seen from a company this size."
  Ex-CTO director: "Whoever that engineer is, don't lose them."
  James: putting in headcount request for 2 more platform engineers.

FINANCIAL EXPOSURE:
  Credit monitoring (Kroll): ~$797K
  CrowdStrike forensics: $150-250K
  Legal fees: $200-400K
  Customer support surge: $50-100K
  Regulatory fines (est.): €500K-€2M
  Total (excl. fines): $1.2-1.55M

HR / ATTRIBUTION:
  Carlos Mendez departure records:
    Voluntary resignation, effective Sep 15, 2023
    Accepted position at competitor (name redacted)
    NDA: 2-year non-compete + perpetual confidentiality
    Exit interview (Wei Liu): Carlos mentioned "staging stuff 
      that could be cleaned up." Wei: "we'll handle it." No follow-up.
    IT offboarding: laptop returned, Okta disabled, email forwarded 30 days.
      NO mention of AWS access, SSH keys, infrastructure audit.
  Criminal: police report filed Tue Wk2. FBI CFAA referral by Fri Wk2.
  Civil: outside counsel evaluating NDA breach action.

USER GAPS IN HANDLING:
  - Assessed webhook data as "no evidence of export" based on bash 
    history only. Wrong — Metabase H2 export log showed CSV export.
    Corrected within 2 hours. (Rule 4)
  - Missed secondary ENI on staging Metabase during initial SG lockdown.
    Found by CrowdStrike report next morning. 16-hour window where actor 
    could have SSH'd in (flow logs show they didn't — lucky).
  - Seven-day gap between K8s audit (Jan 15) and AWS audit (Jan 22).
    Should have generalized threat model to AWS resources immediately 
    after finding K8s shadow infrastructure.
  - analytics_reader CONNECTION LIMIT was known gap since Wed Wk1, 
    caused Incident 8 on Mon Wk2. Two-second fix deferred to ticket.
```

### ServiceProfile Timeout Issue (consequence, not numbered)

```
TIMELINE:
  Wed Wk1 10:47 PM — Derek's ServiceProfile merged (isRetryable + unauthorized timeout)
  Thu Wk1 3:25 PM  — Derek notices NR flags
  Thu Wk1 3:46 PM  — ArgoCD syncs fix
  Thu Wk1 3:54 PM  — NR rate zero ✅

IMPACT: 17 hours, 0.3% NR, error rate 0.09%→0.41%
  Budget: payment-service 87.1% → 86.0% (1.1% consumed)
ROOT CAUSE: Derek added timeout: 5s post-approval.
PROCESS FIX: "Any commit after approval gets a re-review ping."
```

---

## SHADOW INFRASTRUCTURE TALLY (FINAL — as of Thu Wk2)

```
# | Type | Location | Owner | Status
1  | Kafka brokers (6) | K8s: data-analytics | Tom | ✅ ECR, meshed, contained
2  | ZooKeeper (3) | K8s: data-analytics | Tom | ✅ ECR, meshed, contained
3  | Kafka Connect | K8s: data-analytics | Tom | ✅ ECR, meshed, contained
4  | ETL CronJob | K8s: data-analytics | Tom | Running (expected exit 1)
5  | Analytics Dashboard | K8s: data-analytics | Tom | 1 replica, broken (pwd rotated)
6  | MongoDB Atlas | External (free-tier) | Tom | BLOCKED + Legal hold
7  | ES cluster | K8s: search | Priya (interim) | ✅ ILM deployed, stabilized
8  | Kibana | K8s: search | Priya (interim) | Running, Docker Hub image (Jake ECR list)
9  | Metabase EC2 (prod) | EC2 | Carlos (departed) | STOPPED, forensic hold
10 | Legacy Jenkins EC2 | EC2 | former-devops | STOPPED, forensic hold
11 | Reporting DB | RDS | Carlos (departed) | Password rotated, investigation complete
-- | S3: novamart-data-exports | AWS S3 | Tom/Nina | ✅ Hardened, IN TERRAFORM
-- | S3: novamart-analytics-datalake | AWS S3 | Tom | ✅ Hardened, IN TERRAFORM
-- | S3: novamart-db-backups | AWS S3 | legacy | ✅ Hardened, IN TERRAFORM, evidence hold
-- | S3: carlos-dev-scratch | AWS S3 | Carlos | Evidence hold
-- | Order data export CronJob | K8s: order-prod | Nina | ✅ ONBOARDED to ArgoCD
-- | Lambda: novamart-etl-trigger | AWS Lambda | Carlos | Documented, dormant
-- | Lambda: novamart-report-generator | AWS Lambda | Carlos | Documented, dormant since Nov 29

AWS AUDIT FINAL (Marcus, 52 resources):
  37 managed (Terraform/ArgoCD)
  8 known-unmanaged (all have owners + remediation dates)
  4 decommissioned this week
  3 evidence-hold (stopped instances)

DECOMMISSIONED THIS WEEK:
  ✅ load-test-runner EC2 (terminated)
  ✅ novamart-temp-uploads S3 (deleted — empty)
  ✅ grafana-cloud-metrics IAM key (deactivated)
  ✅ datadog-trial IAM key (deactivated)
  ✅ intern.analytics console access (disabled)
```

---

## SIMULATION TIMELINE

```
WEEK 1 — MONDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
09:00  On-call shift starts
09:04  ACK PagerDuty                                   [INC-1 START]
09:08  Unblock Alex (state lock)                        [INC-2 START]
09:13  Root cause confirmed: 800ms timeout
09:15  GitOps hotfix pushed (2000ms)
09:30  RESOLVED                                         [INC-1 END]
09:42  Alex self-resolved                               [INC-2 END]
11:02  Ryan reports catalog slowness                    [INC-3 START]
11:40  Redis maxmemory bumped                           [INC-3 MITIGATED]
14:00  Circuit breaker design meeting
14:48  Kafka investigation                              [INC-4 START]
15:15  PDB + NetworkPolicy for Kafka                    [INC-4 MITIGATED]

MONDAY NIGHT / TUESDAY EARLY AM
01:50  NetworkPolicy v2 fix (ZK port)
02:00  Redis upgrade                                    [INC-3 RESOLVED]

TUESDAY
07:45  David reports builds failing                     [INC-5 START]
~AM    Jenkins zombie pods resolved                     [INC-5 RESOLVED]

WEDNESDAY
10:00  Standup — PCI insertion
10:45  Cluster audit (scripted sweep)
12:15  NetworkPolicy v3, Atlas blocked, S3 remediation  [INC-4 MITIGATED v3]
14:47  PagerDuty: PaymentServiceHighErrorRate           [INC-6 START]
14:53  Rollback initiated
16:08  sha-c4d82e1 deployed                             [INC-6 RESOLVED]
22:47  Derek ServiceProfile merged (+unauthorized timeout)

THURSDAY
10:00  Linkerd staging dry-run with Priya
12:15  PCI document complete
14:00  PCI Emergency Review Meeting
14:56  ETL alert investigation (KMS gap found + fixed)
15:25  ServiceProfile timeout investigation + fix
15:58  CB PR review (3 blocking, 1 should-fix, 1 non-blocking)

FRIDAY
05:47  SearchServiceHighErrorRate                       [INC-7 START]
06:52  Cluster GREEN                                    [INC-7 END]
08:05  Drift detection script development
09:28  Rachel remediation table
09:33  Finance $47.23 reconciliation
10:02  Derek merges CB PR
10:30  Deploy drift detection to staging
13:15  Rachel files GDPR Art. 33 — SA-2024-0119-NM
14:00  Lisa's canary AnalysisTemplate PR approved       [PLAT-916 DONE]

WEEKEND (QUIET)
  Nina: added order-data-export to ArgoCD (Sat)
  Marcus: node-problem-detector to ArgoCD (Sun)
  Sun: Wei DM re Tom → RBAC restricted, Tom called

WEEK 2 — MONDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
08:00  Drift detection first production run (8 CRITICAL) [DELIVERABLE ✅]
10:00  Tom Chen onboarding meeting                      [MEETING — productive]
11:07  PagerDuty: RDSHighConnectionCount                [INC-8 START]
11:22  Database recovered                               [INC-8 RESOLVED]
12:15  Marcus: Metabase, legacy Jenkins, unknown RDS found
14:00  CertificateExpiringWithin7Days                   [INC-9 START]
14:27  Certificate renewed                              [INC-9 RESOLVED]
14:42  Marcus: unauthorized Jenkins logins from peered VPC [INC-10 START]
14:50  Aisha: security incident declared (SEC-2024-0122-001)
15:15  James bridge call
15:24  VPC peering DELETED
15:30  Rachel + Aisha + User: investigation coordination
17:00  Rachel working session (GDPR notification planning)
17:28  Aisha: CrowdStrike CloudTrail — S3 GetObject from staging Metabase
17:30  Marcus: carlos-dev-scratch S3 bucket, Oct/Nov CSV exports
17:33  Exfiltration assessment: "possible" → "probable"
17:41  VPC flow logs enabled
17:58  NAT gateway metrics: CONFIRMED exfiltration (4 data spikes match)
18:00  CrowdStrike call (David Park)                    [FORENSICS START]
18:22  VPC peering route: full production CIDR (10.0.0.0/16) reachable
18:28  Production Metabase uses Carlos SSH key — added to forensic scope
18:44  Containment: production Metabase SG, reporting-db-prod, analytics_reader
19:00  All known credentials rotated, all network paths severed
19:12  Handed night watch to Aisha + CrowdStrike

TUESDAY (Wk2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
06:58  SearchServiceHighLatency alert (p99 >2s)
06:59  Delegated to Priya (secondary on-call)
07:00  CrowdStrike overnight findings review:
         bash_history: psql, pg_dump, curl -T transfer.sh
         Jenkins credentials store: RDS master pwd, IAM admin key
         jenkins-deployer: AdministratorAccess (DEACTIVATED by Aisha 12:22 AM)
         orders-db-prod master: ROTATED (Aisha 12:14 AM)
         H2 Metabase DB: 347 queries, 58 post-departure
07:15  Priya: search GREEN, heap 79%, chronic ILM issue (not incident)
08:30  James sync: breach scope briefed, all questions answered
09:04  reporting-db-prod card data check (PRIORITY)
09:17  Results: NO full card numbers. NOT PCI. ✅
09:35  Rachel + James updated. Noon filing clean.
10:01  Marcus: Carlos personal IAM user still ACTIVE, key last used Jan 19
10:03  Aisha deactivates Carlos IAM key
10:01  Marcus + Tom: ECR rolling restart begins
10:14  Derek: CB at 25%, clean
10:38  CrowdStrike report full review (94 pages)
10:44  MISSED: secondary ENI with public SSH — REVOKED
10:51  All three forensic instances STOPPED
11:01  CORRECTION: webhook data WAS exported via Metabase (not just viewed)
11:15  Marcus: ECR restart complete, Lambdas found, intern account disabled
11:40  Derek: CB at 50%
12:03  Rachel: Art. 33 #2 FILED (SA-2024-0122-NM-002)
14:00  Derek: CB at 100% ✅ (clean deploy, metrics improved)
15:00  Marcus: AWS audit 90% complete
15:30  Tom: S3 bucket Terraform imports
15:40  Derek: stockCache design decision (Option A — write-through)
15:42  Lisa: budget update, dual-SLI proposal
16:30  Board materials draft sent to James/Sarah
17:50  EOD

WEDNESDAY (Wk2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
06:00  Marcus: AWS audit table complete (52 resources)
06:30  Rachel: Art. 34 draft, UK ICO draft, Kroll vendor selected ($797K)
06:45  Tom: mesh injection ready
07:00  Jake: Kyverno staging clean (3 violations, all expected)
07:30  Board materials: 3 revisions (Controls Assessment, fine exposure, Day 1 vs Day 2)
08:35  Rachel Art. 34 review (added phishing specificity)
09:10  Marcus AWS audit review + Rachel handoff
10:00  Priya: ILM policy deployed                       [INC-7 ROOT CAUSE FIXED]
         34 indices → warm, heap 81%→53%, p99 1.1s→680ms
10:05  Tom: mesh injection deployed (all Kafka pods meshed)
10:38  Tom: all components running with Linkerd mTLS ✅
11:12  Sarah: James approved board materials. "Day 1 vs Day 2 is the money slide."
         James wants User to present Controls Assessment + Day 1/Day 2 directly.
11:20  Aisha: HR file — Carlos exit interview, Wei context
11:25  Derek: stockCache PR #354 (approved, staging Fri)
12:00  Lunch
12:40  Derek stockCache PR review (2 non-blocking comments)
13:05  Priya: EKS runbook approved (PLAT-892 DONE)
14:00  Customer support briefing (Rachel-led, User 10 min technical Q&A)
15:00  Search cluster ownership proposal sent to Sarah
15:30  Tom: all 3 S3 buckets imported to Terraform ✅
16:00  Wei: offboarding infrastructure walkthrough proposal
16:30  Jake: image admission production approved for Thu AM
17:00  EOD

THURSDAY (Wk2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
04:00  UK ICO filed (ICO-2024-NM-0122)
06:15  Jake: image admission production audit mode deployed
06:30  Tom: dashboards finalized, runbook 80%
08:00  Office. Customer notification preparation.
10:00  CUSTOMER NOTIFICATION SENT (284,847 emails)       [ART. 34]
10:12  Support queue active. Volume moderate. FAQ handling bulk.
10:30  Wei: message about exit interview, offboarding proposal
15:00  BOARD PRESENTATION                                [BOARD MEETING]
         James 15 min (breach/impact) → User 10 min (controls/remediation) → Q&A
         Board approves remediation. 2 headcount approved.
         CEO: sustainability question (answered honestly)
15:32  Meeting ends.
17:00  EOD

FRIDAY (Wk2) — PLANNED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Tom: R-6a milestone close (dashboards + runbook)
  Derek: stockCache production deploy
  Jake: image admission validation report
  R-6a DEADLINE: Kafka secured ✅ (on track)
  Image admission warn mode DEADLINE ✅ (on track)
  FBI CFAA referral (Rachel/counsel)

>>> SIMULATION PAUSED: Thursday Wk2 5:30 PM <<<
>>> RESUME: Friday Wk2 morning <<<
```

---

## LIVE SYSTEM CHANGES — CONSEQUENCE TRACKING

```
CHANGE                              APPLIED         STATUS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

NetworkPolicy v3                    Wed Wk1         ✅ Verified
  (data-analytics namespace)                        ⚠️ RDS egress CIDR (fragile)

KMS key policy (all 4 roles)        Wed+Thu Wk1     ✅ Fully verified

S3 encryption + hardening           Wed Wk1         ✅ Verified
  (all buckets, KMS/CMK)

S3 Terraform imports                Wed Wk2         ✅ All 3 buckets imported
  (data-exports, datalake, db-backups)               No drift on terraform plan

Payment sha-c4d82e1                 Wed Wk1         ✅ Stable
  (HALF_EVEN reconciliation complete)

Jenkins build discard               Tue Wk1         ✅ Stable 8+ days
                                                    ⚠️ Aggressive discard limit unverified

Redis r6g.xlarge                    Tue Wk1         ✅ Stable (94.1%, 54% mem)

PLAT-919 PrometheusRule             Mon Wk1         ✅ Correctly scoped

Derek ServiceProfile (timeout fix)  Thu Wk1         ✅ NR zero

ResourceQuota (search ns, 80Gi)     Fri Wk1         ✅ Annotated

PDB (search ns)                     Fri Wk1         ✅ Correct labels

ILM policy (search ns)              Wed Wk2         ✅ DEPLOYED
  hot: 30d, warm: read-only+merge                   Heap 81%→53%
  delete: 180d (pending legal)                       p99 1.1s→680ms
                                                    34 indices in warm tier

analytics_reader CONN LIMIT 20      Mon Wk2         ✅ Applied
analytics_reader PASSWORD ROTATED   Mon Wk2         ✅ Applied

RBAC: Tom read-only (data-analytics) Sun Wk2        ✅ Applied
RBAC: Wei read-only (data-analytics) Mon Wk2        ✅ Applied
RBAC: All non-platform read-only     Mon Wk2        ✅ Applied

Dashboard HPA frozen (max 1)        Mon Wk2         ✅ Applied

Cert-manager IAM key rotation       Mon Wk2         ✅ Cert renewed (Apr 21)

Drift detection CronJob             Mon Wk2         ✅ Running daily, 8 findings → resolving

Circuit breaker (PLAT-915)          Thu Wk2         ✅ 100% production, clean metrics
                                                    Error rate improved (0.018%)
                                                    p99 improved (395ms)

Dual-SLI (Lisa #353)                Wed Wk2         ✅ Merged, recording rules active

stockCache (Derek #354)             Wed Wk2         ✅ PR approved, staging Thu
                                                    Production deploy Fri Wk2

Image admission (Kyverno)           Thu Wk2         ✅ Production audit mode
  Staging: 3 violations (all expected)               No unexpected violations
  Production: 3 violations (Kibana, Grafana, nginx:latest)
  Enforce mode: Feb 2

Linkerd mesh (data-analytics)       Wed Wk2         ✅ All pods meshed, mTLS verified

ECR migration (Kafka/ZK/Connect)    Tue Wk2         ✅ Docker Hub eliminated

SECURITY INCIDENT CONTAINMENT:
  All credentials rotated            Mon-Tue Wk2    ✅ 9 credentials
  All network paths severed          Mon Wk2        ✅ VPC peering deleted
  All SGs locked                     Mon-Tue Wk2    ✅ Including secondary ENI
  All forensic instances stopped     Tue Wk2        ✅ EBS snapshots preserved
  VPC flow logs enabled              Mon Wk2        ✅ Prospective only
  Carlos IAM user deactivated        Tue Wk2        ✅ Key deactivated
  jenkins-deployer IAM deactivated   Tue Wk2        ✅ (Aisha, midnight)
  intern.analytics disabled          Tue Wk2        ✅ Console access removed
  grafana-cloud-metrics deactivated  Wed Wk2        ✅
  datadog-trial deactivated          Wed Wk2        ✅

STILL UNVERIFIED:
  ❓ Jenkins build discard aggressive limit
  ❓ RDS egress CIDR fragility (NetworkPolicy v3)
  ❓ carlos-staging-key SSH key (SG-blocked, pending CrowdStrike clearance to delete)
  ❓ novamart-analytics-key SSH key (SG-blocked, pending CrowdStrike clearance)
```

---

## ERROR BUDGETS (as of Thu Wk2 EOD)

```
SERVICE          BUDGET    TREND    NOTES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Order-service    61.2%     ↑        CB p99 improvement earning ~0.04%/day
                                    Projected window end: ~59.6%
                                    THIN. One more incident → freeze.
                                    CB deployed (risk reduction).
                                    stockCache deploying Fri (further reduction).

Payment-service  86.0%     →        Velocity freeze in effect.
                                    Exception: corrective deploys only.
                                    sha-c4d82e1 stable.
                                    HALF_EVEN reconciliation complete.

Search-service   No SLO    —        ILM deployed. Heap stable. p99 680ms.
                                    SLO definition: Priya + Lisa, next week.

Inventory-svc    Not tracked separately (part of order-service SLI)
```

---

## OPEN ITEMS / PENDING WORK

### Active Tickets

```
PLAT-892  [Done ✅]     EKS 1.28→1.29 upgrade runbook — approved Wed Wk2
PLAT-901  [In Review]   Linkerd PR #347 — needs re-review (Priya updates)
PLAT-910  [To Do]       GP3 EBS migration (next sprint, skip data-analytics + search)
PLAT-915  [Done ✅]     Circuit breaker — 100% production
PLAT-916  [Done ✅]     Canary analysis — Lisa's AnalysisTemplate merged
PLAT-917  [To Do]       Investigate inventory p99 (Nina)
PLAT-918  [Done ✅]     Redis scale-up
PLAT-919  [Done ✅]     Redis alerting
PLAT-920  [To Do]       Quarterly refresh capacity planning gate
PLAT-921  [Done ✅]     Kafka platform onboarding R-6a (closing Fri)
PLAT-922  [Done ✅]     PCI scope assessment + GDPR filed
PLAT-923  [In Progress] stockCache (Derek PR #354, production Fri)
[NEEDED]  Cert-manager IRSA migration (this week — not started)
[NEEDED]  Cert renewal failure alert (not just expiry)
[NEEDED]  Search cluster: ArgoCD onboarding (Priya, by Feb 2)
[NEEDED]  Search cluster: gp2 → gp3 migration (Jake, next sprint)
[NEEDED]  Search cluster: PriorityClass
[NEEDED]  Search cluster: SLO definition (Priya + Lisa)
[NEEDED]  Search cluster: runbook update
[NEEDED]  Kibana ECR migration (Jake, pre-Feb 2)
[NEEDED]  Grafana ECR migration (Jake, pre-Feb 2)
[NEEDED]  Self-service data pipeline template (mid-Feb committed)
[NEEDED]  Default-deny NetworkPolicy all namespaces (mid-Feb committed)
[NEEDED]  IAM lifecycle automation — Okta cross-ref (Aisha, end Feb)
[NEEDED]  SSH key mgmt through IdP (Aisha, end Feb)
[NEEDED]  Egress gateway (end of Q1 committed)
[NEEDED]  AWS resource tagging enforcement (Marcus, end Q1)
[NEEDED]  S3 lifecycle/retention policies (all buckets)
[NEEDED]  Migrate secrets to External Secrets Operator
[NEEDED]  ACME account: update from Carlos email
[NEEDED]  weekly_summary.py: connection pooling / max-workers cap
[NEEDED]  Read replica for analytics workloads
[NEEDED]  Offboarding infrastructure walkthrough (Wei proposal — review)
[NEEDED]  SA-2024-0119-NM supervisory response (due Feb 5)
[NEEDED]  FBI CFAA referral (Rachel/counsel, Fri Wk2)
[NEEDED]  Carlos SSH keys: delete after CrowdStrike clearance
[NEEDED]  Lambda functions: assess and decommission or onboard
[NEEDED]  reporting-db-prod: Terraform import or decommission (pending Legal)
[NEEDED]  Analytics dashboard: reconnect after password rotation (or decommission)
```

---

## TEAM STATE (as of Thu Wk2 EOD)

```
James Morrison (VP Eng):  Board presentation done. Remediation approved.
                          2 headcount approved. 30-day follow-up scheduled.
                          "Don't oversell the remediation." Trusts the team.

Sarah Chen (manager):     Board materials formatted. Credit monitoring vendor
                          selected (Kroll $797K). Press statement ready.
                          Forwarded drift scan to James week ago — now board-visible.
                          Coordinating marketing + support for notification response.

Aisha Rahman (Security):  IR lead for SEC-2024-0122-001. CrowdStrike coordination.
                          Overnight: rotated RDS master, deactivated jenkins-deployer
                          IAM key (AdministratorAccess). Adding pre-deactivation
                          notification to security automation. Planning IAM lifecycle
                          automation (Okta cross-ref). HR coordination for Carlos
                          attribution. "Going in the post-incident remediation plan."

Rachel Torres (Legal):    GDPR Art. 33 × 2 filed. Art. 34 sent (284,847 emails).
                          UK ICO filed. Police report filed. FBI referral by Fri.
                          Supervisory authority response due Feb 5.
                          Credit monitoring: Kroll, $2.80/customer.
                          Customer notification response: measured, no backlash.
                          "Atlas stays blocked until I say so."

Wei Liu (Data Eng mgr):   RBAC restricted. VP report delivered. Admitted fault on
                          manual pod + exit interview follow-up gap. Proposed
                          offboarding infrastructure walkthrough — "mandatory
                          infrastructure walkthrough with departing engineers."
                          Cooperative, accountable, constructive.

Tom Chen (Data Eng):      RBAC read-only. Sprint executing ahead of schedule.
                          ✅ ECR Tue, ✅ mesh Wed, ✅ Terraform Wed (early),
                          dashboards Thu, runbook Fri.
                          "First time I've hit a deadline in months."
                          "Most productive three days at NovaMart."
                          Marcus + Tom pair: "Like pair programming for infra."

Nina Petrov (Order lead): Onboarded order-data-export to ArgoCD (self-initiated).
                          README added. CB PR pushback resolved (business math).
                          Self-service template UX requirement given.

Marcus Webb (secondary):  AWS audit COMPLETE (52 resources, all reconciled).
                          Found: legacy Jenkins, staging Metabase, reporting-db-prod,
                          Lambdas, intern account, IAM users, S3 buckets.
                          Evidence collection. Credential rotations.
                          Shadow tally: 11 → 3 remaining (all with plans).
                          "Before I eat: carlos-dev-scratch S3 bucket."

Priya Sharma (mid→senior): ILM deployed (Incident 7 root cause fixed).
                          Search cluster interim owner (proposal sent).
                          EKS runbook approved. Linkerd dry-run complete.
                          Triaged search latency alert independently.
                          ILM monitoring dashboard building.
                          Significant growth during this period.

Jake Torres (mid):        Kyverno deployed: staging clean, production audit mode.
                          GP3 migration planning resumed.
                          ECR migration punch list: Kibana, Grafana (pre-Feb 2).

Alex Kim (junior):        Jenkins build killed during Inc-7 (apologized).
                          nginx:latest test pod flagged by image admission.
                          Growing. Good instincts.

Lisa Park (SRE):          Canary AnalysisTemplate merged. Dual-SLI merged.
                          Budget tracking: order 61.2%, payment 86.0%.
                          CB metrics confirmed improvement.

David Okafor (Payment):   Reconciliation complete with Martha.
                          sha-c4d82e1 stable. PAYMENT-847 filed.

Derek Huang (Order):      CB deployed to 100% — cleanest deploy of quarter.
                          stockCache PR approved, staging Thu, production Fri.
                          Three-tier fallback: live → cached → blind accept.
                          Timeout anti-pattern acknowledged. Post-approval process.

Ryan Mitchell (Frontend): Confirmed services recovered after incidents.

Martha Reeves (Finance):  $47.23 reconciliation handled. One-time adjustment.
                          Metabase user (discovered during investigation).

Priya Kapoor (Payment):   BigDecimal fix stable. PAYMENT-847 filed.

EXTERNAL:
  David Park (CrowdStrike): Final report delivered. "Best-prepared client handoff 
    in two years." "Fastest and most thorough response we've seen."
  Kroll: Credit monitoring enrollment live. 24,847 signups day 1.
  Outside counsel: Police report filed. FBI CFAA referral by Fri.
  Supervisory authority: Both filings acknowledged. Reviewing together. 
    Response to SA-2024-0119-NM due Feb 5.
```

---

## USER PERFORMANCE SUMMARY

### Grades

```
Incident 1 (SEV2 — Order timeout):           A+
Incident 2 (SEV4 — State lock):              A
Incident 3 (SEV3 — Redis capacity):          A+ / A
Incident 4 (SEV3 — Shadow Kafka/Pipeline):   A- → A+
Incident 5 (SEV2 — Zombie CI pods):          [Grade from session]
Incident 6 (SEV2 — Payment currency bug):    A+
Incident 7 (SEV2 — ES/Search outage):        Pending
Incident 8 (SEV2 — RDS connections):          Pending
Incident 9 (SEV3 — TLS cert renewal):        Pending
Incident 10 (SEV1 — Data exfiltration):       Pending

Standup Presentation:       A+
Cluster Audit:              A+
PCI Escalation:             A+
PCI Meeting:                A+
PR Reviews:                 A+
Circuit Breaker Meeting:    A+
Linkerd Dry-Run:            A+
Tom Onboarding Meeting:     A+
Drift Detection Delivery:   A+
Maintenance Window:         A
Board Presentation:         A+
CrowdStrike Coordination:   A+
Rachel Working Session:     A+
Customer Support Briefing:  A+
```

### v2 Chaos Engine Observations

```
CONSEQUENCES ENCOUNTERED (cumulative, 11 days):
  1. KMS permission gap (self-inflicted Wed Wk1, found Thu) — fixed, disclosed
  2. ServiceProfile timeout (approved PR, Derek added post-approval) — fixed
  3. ETL "failure" alert (containment caused expected daily alert) — silenced
  4. HALF_EVEN reconciliation (known Wed, hit finance Fri) — $47.23, handled
  5. analytics_reader no connection limit (known Wed Wk1, hit Mon Wk2) — Incident 8
  6. Wei RBAC gap (restricted Tom Sun, Wei caused incident Mon) — fixed
  7. Security automation broke cert renewal (RULE 2) — fixed
  8. Legacy Jenkins unauthorized access (discovered through audit) — Incident 10
  9. Webhook export assessment WRONG (bash history only, missed H2 export log) — corrected
  10. Secondary ENI missed during SG lockdown (found by CrowdStrike report) — fixed

RULE 4 INSTANCES (wrong/incomplete hypothesis):
  - PDB labels (wrong on first attempt, corrected in 60 sec)
  - HALF_EVEN impact scope (three rounding behaviors, not clean cutover)
  - RBAC scope (restricted Tom, missed Wei — pattern, not individual)
  - Webhook export: "no evidence of export" → Metabase H2 showed CSV export
  - Secondary ENI: locked primary SG, missed second network interface

STRENGTHS DEMONSTRATED:
  ✓ Parallel execution under extreme time pressure
  ✓ Counterintuitive tactical decisions (scale DOWN to save UP)
  ✓ Unfamiliar system navigation (ES cluster — RULE 10)
  ✓ Genuine stuck point resolution (AZ-bound PVC — RULE 12)
  ✓ Signal/noise separation under pressure
  ✓ Honest self-accountability without self-flagellation
  ✓ Executive communication under pressure
  ✓ Board presentation (10 min, direct, honest, CEO-praised)
  ✓ Protecting people from themselves (Tom RBAC, Wei RBAC)
  ✓ Building trust through transparency (Wei "refreshing", Tom "not an ambush")
  ✓ Board-quality deliverables under deadline
  ✓ Committed deliverables met on time (drift detection, R-6a on track)
  ✓ Converting blockers through business context
  ✓ Process fixes over blame (Derek, Wei, Tom — all constructive)
  ✓ VP-level framing ("wall and door at the same time")
  ✓ CrowdStrike brief quality ("best-prepared in two years")
  ✓ Forensic evidence preservation under pressure
  ✓ Self-correction speed (webhook assessment corrected <2 hours)
  ✓ Delegation to growing engineers (Priya search, Jake admission, Marcus audit)
  ✓ Security incident management (44-min containment, same-day forensics)

GAPS:
  - analytics_reader CONNECTION LIMIT deferred (caused Incident 8)
  - HALF_EVEN not proactively flagged to finance (credibility hit)
  - RBAC restriction incomplete (Tom but not Wei)
  - 7-day gap between K8s audit and AWS audit scope expansion
  - Webhook "no export" assessment wrong (corrected <2h)
  - Secondary ENI missed in initial SG lockdown (16h window)
  - PDB wrong labels on first attempt (Incident 7)
  - Cert renewal failure invisible for 23 days (alert gap)
  - AWS-level shadow infra not visible to K8s drift scan

OVERALL PATTERN:
  Exceptionally strong on: diagnosis speed, containment execution, 
  executive communication, team development, honest self-assessment.
  
  Growth area: immediate application of low-effort/high-exposure fixes 
  (database-level changes, credential rotations) rather than deferring 
  to ticket timelines. Also: enumerate ALL access paths during containment,
  not just obvious ones. And: qualify assessments during active 
  investigations — "based on current evidence" vs definitive statements.
```

---

## INCIDENT CATEGORY COVERAGE (10/100)

### REVISED TAXONOMY (v2)

```
CATEGORY                              DONE  TARGET  INCIDENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deploy/Release failures                2      8     #1, #6
Database (connections/replication/     2      8     #3, #8
  corruption/migration)
Kubernetes (scheduling/resources/      1      8     #7
  eviction/autoscaling)
Networking — DNS                       0      5     
Networking — Load Balancer/Ingress     0      4     
Networking — Service Mesh              0      3     
Networking — General (VPN/peering/     0      3     
  NACLs/MTU)
CI/CD pipeline failures                1      5     #5
Security incidents                     1      4     #10
Observability stack failures           0      4     
Capacity/scaling                       1      6     #3 (shared)
Cascading / multi-system               1      5     #8 (RDS→3 svcs)
Cloud provider degradation             0      4     
Kernel/OS/Node-level                   0      5     
Control plane / etcd                   0      3     
Data pipeline / streaming              0      4     
Third-party / dependency               0      4     
Storage (EBS/PV/S3)                    0      3     
Certificate / secrets                  1      3     #9
Cost/billing anomalies                 0      3     
DR / region failover                   0      3     
Scheduled jobs / CronJobs              0      2     
Auth / RBAC / SSO                      0      3     
Simultaneous incidents                 0      5     
IaC / Terraform / GitOps drift         1      3     #2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                                 10    100

COVERAGE RATE: 10% complete, 9/25 categories touched.
Major categories with zero coverage: DNS, LB/Ingress, Mesh,
  Observability, Cloud provider, Kernel/OS, Control plane,
  Data pipeline, Third-party, Storage, Cost, DR, CronJobs,
  Auth/SSO, Simultaneous incidents.
```

---

## WHERE TO RESUME

```
SIMULATION TIME:  Friday Wk2 morning (Day 12 of 100)
SIMULATION MODE:  v2 — CHAOS ENGINE ACTIVE (all 12 rules)
ON-CALL:          User (primary), Priya (secondary)

ACTIVE SITUATIONS:
  1. SEC-2024-0122-001: Investigation phase (CrowdStrike final)
     - Customer notification sent, support queue active
     - FBI CFAA referral pending (Rachel/counsel, today)
     - SSH keys pending CrowdStrike clearance to delete
     - SA-2024-0119-NM response due Feb 5

  2. R-6a milestone closing today
     - Tom: dashboards + runbook (final items)
     - All other R-6a components DONE

  3. Derek stockCache production deploy (today)
  
  4. Jake image admission validation (today)

TODAY (FRIDAY Wk2):
  - R-6a milestone close
  - stockCache production deploy
  - Image admission validation report
  - FBI CFAA referral
  - Normal operations

NEXT WEEK PRIORITIES:
  - Cert-manager IRSA migration
  - Cert renewal failure alert
  - Search: ArgoCD onboarding (Priya)
  - Search: SLO definition (Priya + Lisa)
  - Kibana + Grafana ECR migration (Jake, pre-Feb 2)
  - Image admission enforce mode prep (Jake, Feb 2)
  - SA-2024-0119-NM supervisory response prep (Rachel, due Feb 5)
  - Wei offboarding proposal review
  - Lambda assessment (onboard or decommission)
  - R-6b planning (Kafka fully onboarded, Feb 9)
  - Headcount planning (2 new positions)
  - Default-deny NetworkPolicy scoping (mid-Feb committed)
  - Self-service pipeline template design (mid-Feb committed)
```

---

## 90 REMAINING INCIDENTS — REVISED TAXONOMY

### Distribution Plan

```
CATEGORY                           REMAINING  PLANNED INCIDENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Deploy/Release failures (6 remaining)
  11. Canary analysis passes but baseline shift masks regression
  12. Helm value override collision during multi-team deploy
  13. ArgoCD sync loop (CRD ordering dependency)
  14. Blue-green cutover fails: new version can't read old session format
  15. Feature flag gradual rollout causes split-brain state
  16. Init container image pull fails silently, main container starts with missing config

Database (6 remaining)
  17. RDS Multi-AZ failover during business hours (planned maintenance window surprise)
  18. PostgreSQL bloat / autovacuum stall on high-write table
  19. Replication lag on read replica causes stale reads in search indexing
  20. Connection pool leak: connections acquired but never returned (slow growth over days)
  21. Schema migration partially applied (liquibase/flyway lock + partial DDL)
  22. Redis cluster slot migration stuck during scaling event

Kubernetes (7 remaining)
  23. Karpenter consolidation evicts pods during peak traffic
  24. Pod topology spread constraint prevents scheduling after AZ loss
  25. HPA flapping: scale-up/scale-down oscillation from metric lag
  26. Resource quota prevents legitimate scaling during traffic spike
  27. CrashLoopBackOff from config map change (bad YAML value)
  28. Node pressure eviction cascade (memory, not storage this time)
  29. PodDisruptionBudget blocks node drain during upgrade

Networking — DNS (5)
  30. CoreDNS pod OOM: intermittent NXDOMAIN across cluster
  31. DNS TTL caching causes traffic to old endpoints after migration
  32. ndots:5 default causing excessive DNS queries + latency
  33. Route53 health check flapping → traffic shifts between regions
  34. External DNS record not updating after service IP change

Networking — Load Balancer / Ingress (4)
  35. ALB target group health check timeout too aggressive: healthy pods marked unhealthy
  36. Ingress controller (nginx) OOM under connection surge
  37. gRPC load balancing failure: all traffic hitting single pod (HTTP/2 connection reuse)
  38. WebSocket connections not draining during rolling deploy → client disconnects

Networking — Service Mesh (3)
  39. Linkerd proxy memory leak after upgrade (slow growth, eventually OOMKilled)
  40. mTLS certificate rotation failure: expired identity certs break inter-service auth
  41. Retry storm: mesh retries amplify partial outage into full outage

Networking — General (3)
  42. MTU mismatch causing packet fragmentation between VPCs
  43. Security group rule limit hit: new rules silently fail
  44. NAT gateway throttling: burst of external API calls exceeds connection limit

CI/CD pipeline failures (4 remaining)
  45. GitHub Actions runner OOM on large monorepo build
  46. Container registry (ECR) pull rate throttled during mass deploy
  47. ArgoCD application-of-applications sync: child app stuck in "OutOfSync" despite matching
  48. Build cache poisoning: cached layer contains old dependency version

Security incidents (3 remaining)
  49. Cryptominer pod deployed via compromised container image in public registry
  50. AWS access key leaked in public GitHub commit (automated scanner alert)
  51. Service account token mounted in pod used for lateral movement (pentest finding)

Observability stack failures (4)
  52. Prometheus TSDB corruption: OOM during compaction → metrics gap
  53. Metric cardinality explosion: unbounded label (request_id) fills TSDB
  54. Alertmanager cluster split-brain: some alerts fire, some don't
  55. Log pipeline backpressure: FluentBit buffer overflow → log loss during incident

Capacity / scaling (5 remaining)
  56. Karpenter provisioner launches wrong instance type (ARM vs x86 mismatch)
  57. Vertical Pod Autoscaler recommendation causes OOMKill (aggressive downscale)
  58. Burst traffic from marketing campaign: 10x normal in 5 minutes
  59. Cluster autoscaler max-node limit hit: pods pending for 20+ minutes
  60. Ephemeral storage exhaustion on logging sidecar (container log rotation failure)

Cascading / multi-system (4 remaining)
  61. Payment service timeout → order service retry storm → inventory service overload
  62. Cache stampede after Redis failover: all services hit DB simultaneously
  63. Logging pipeline failure → disk pressure on nodes → pod evictions → more logging
  64. Certificate expiry cascade: root CA rotation missed intermediate certs

Cloud provider degradation (4)
  65. AWS EBS io2 latency spike: no status page update, only your metrics show it
  66. S3 ListBucket throttled during backup job: rate limit errors cascade
  67. IAM eventual consistency: role policy attached but assume-role fails for 3 minutes
  68. EC2 insufficient capacity error in primary AZ during autoscaling event

Kernel / OS / Node-level (5)
  69. Conntrack table overflow: new connections dropped, existing connections fine
  70. File descriptor exhaustion: process at ulimit, new connections refused
  71. TIME_WAIT socket accumulation: high-churn service runs out of ephemeral ports
  72. NTP clock skew: 30-second drift causes JWT validation failures and TLS handshake errors
  73. OOM killer targets wrong process: kills database agent instead of runaway job

Control plane / etcd (3)
  74. API server latency spike: webhook timeout on admission controller
  75. etcd database size alarm: compaction not running, 8GB limit approaching
  76. Kubelet certificate rotation failure: node goes NotReady after 12 months

Data pipeline / streaming (4)
  77. Kafka partition rebalancing storm after broker restart (consumer group thrashing)
  78. Consumer lag growing unboundedly: poison message blocks partition processing
  79. Schema registry outage: all Avro-encoded producers fail simultaneously
  80. Dead letter queue overflow: unprocessed messages lost after DLQ retention expires

Third-party / dependency (4)
  81. Stripe API degradation: payment processing latency 10x, timeouts at 30s
  82. Docker Hub rate limit: ImagePullBackOff during incident-driven scaling
  83. CDN cache purge failure: customers seeing stale product prices for hours
  84. SSO/OAuth provider outage: no employee can log into internal tools

Storage (3)
  85. EBS gp3 burst credit exhaustion: IOPS drops from 3000 to baseline 100
  86. PersistentVolume resize stuck: filesystem expansion requires pod restart
  87. S3 request rate throttling: prefix-based hot spot during batch export

Certificate / secrets (2 remaining)
  88. Vault token renewal failure: all services lose secret access simultaneously
  89. Kubernetes secret not updated after rotation: pods using stale credentials

Cost / billing anomalies (3)
  90. NAT gateway data processing charges spike 400%: debugging which service
  91. Spot instance interruption: 2-minute warning, stateful workload affected
  92. Orphaned EBS volumes accumulating: $4K/month in unused storage discovered

DR / region failover (3)
  93. Database failover promotion: replica promoted but application config still points to old primary
  94. Cross-region replication lag: failover loses 30 seconds of transactions
  95. Backup restore test failure: backup exists but restore process has never been tested

Scheduled jobs / CronJobs (2)
  96. CronJob overlap: previous run still executing when next scheduled run starts
  97. Timezone bug: CronJob in UTC, business logic assumes local time, runs at wrong hour

Auth / RBAC / SSO (3)
  98. RBAC change removes critical permission: deploy pipeline breaks across all teams
  99. Service account key expiry: automated process silently fails for 6 hours
  100. OIDC issuer URL change after provider migration: all token validation fails

Simultaneous incidents (woven into above — 5 will fire concurrently):
  Incidents 30+61 fire together (DNS + cascading — correlated root cause)
  Incidents 52+55 fire together (Prometheus down + log pipeline — meta-incident)
  Incidents 58+66 fire together (traffic spike + S3 throttle — capacity pressure)
  Incidents 69+71 fire together (conntrack + TIME_WAIT — node-level networking)
  Incidents 81+41 fire together (Stripe degradation + mesh retry storm — amplification)
```

---

## NOTES ON THE 90 PLANNED INCIDENTS

```
DESIGN PRINCIPLES:
  1. Each incident teaches a DISTINCT diagnostic skill
  2. No two incidents have the same root cause pattern
  3. Difficulty ramps: Wk3-4 are moderate, Wk5-8 introduce 
     OS-level and control plane, Wk9-12 are advanced/cascading
  4. Simultaneous incidents appear ~5 times, increasing frequency
  5. Chaos Engine rules remain active (consequences compound,
     tooling degrades, red herrings appear)
  6. Human/process dimension continues (new team members from 
     headcount, handoffs, on-call rotation, sprint ceremonies)
  7. The NovaMart infrastructure continues evolving (Linkerd 
     upgrade happens, EKS upgrade happens, egress gateway 
     deploys) — incidents interact with these changes
  8. Security investigation follow-up threads continue in 
     background (supervisory authority responses, Carlos 
     attribution, civil action)

APPROXIMATE TIMELINE:
  Week 2 Fri (Day 12):  Incidents 11-12 (close out sprint)
  Week 3 (Days 13-17):  Incidents 13-20 (DNS + DB focus)
  Week 4 (Days 18-22):  Incidents 21-28 (K8s + pipeline)
  Week 5 (Days 23-27):  Incidents 29-36 (networking deep dive)
  Week 6 (Days 28-32):  Incidents 37-44 (mesh + LB + OS-level)
  Week 7 (Days 33-37):  Incidents 45-52 (CI/CD + observability)
  Week 8 (Days 38-42):  Incidents 53-60 (capacity + cloud provider)
  Week 9 (Days 43-47):  Incidents 61-68 (cascading + advanced)
  Week 10 (Days 48-52): Incidents 69-76 (kernel + control plane)
  Week 11 (Days 53-57): Incidents 77-84 (streaming + third-party)
  Week 12 (Days 58-62): Incidents 85-92 (storage + cost + DR)
  Week 13 (Days 63-67): Incidents 93-100 (auth + final gauntlet)

  ~7-8 incidents per week = ~1.5 per day average
  Some days have 0 incidents (planned work). Some have 3.
  Matches realistic FAANG on-call cadence for a platform team.
```

---

**END OF HANDOFF DOCUMENT — v6 (REVISED TAXONOMY)**

**Simulation paused at Thursday Wk2 5:30 PM (Day 11).**

**Resume: Friday Wk2 morning (Day 12).**

**90 incidents planned with revised taxonomy covering 25 categories.**

**Ready to continue when you are.**
