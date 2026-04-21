# PHASE 8 SIMULATION — HANDOFF DOCUMENT (v5)
# CHAOS ENGINE EDITION

---

## SIMULATION VERSION: v2 — CHAOS ENGINE ACTIVE

All 12 Chaos Engine rules in effect. Consequence branching is compounding. Multiple unresolved threads active simultaneously.

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

## INCIDENTS/ISSUES COVERED SO FAR: 9 of 100

*(Incident 9 is RESOLVED. Potential security incident emerging — legacy Jenkins unauthorized access from peered VPC. Not yet numbered.)*

---

## INCIDENT TRACKER

| # | Severity | Type | Description | Status | Grade |
|---|----------|------|-------------|--------|-------|
| 1 | **SEV2** | Deploy-induced timeout | Order service 504s from aggressive inventory_client_timeout (800ms) at peak traffic. | ✅ RESOLVED — Interim fix 2000ms. Follow-up: circuit breaker. | **A+** |
| 2 | **SEV4** | Stale Terraform lock | Alex blocked by stale DynamoDB state lock from crashed Jenkins job. | ✅ RESOLVED — Guided Alex to self-resolve. | **A** |
| 3 | **SEV3** | Capacity — Redis memory pressure | Quarterly catalog refresh exceeded Redis maxmemory. Hit rate dropped 94%→87%. | ✅ RESOLVED — Node upgraded r6g.large→r6g.xlarge overnight. | **A+ / A** |
| 4 | **SEV3** | Shadow infrastructure | Unmanaged Kafka cluster + full data pipeline. Scope expanded by audit: Kafka Connect, ETL CronJob, MongoDB Atlas, S3 PII exports. | ✅ MITIGATED — NetworkPolicy v3, S3 hardened, Atlas blocked. PCI review complete. GDPR notification filed. | **A- → A+** |
| 5 | **SEV2** | CI/CD — Zombie Jenkins agents | Jenkins agent pods stuck running 3-6+ hours, consuming entire Karpenter CI pool. | ✅ RESOLVED — Build discard config + heap monitoring. | **[Grade from session]** |
| 6 | **SEV2** | Deploy-induced code bug | Payment service BigDecimal.divide() without RoundingMode. GBP/SEK/DKK/NOK→EUR 100% failing. | ✅ RESOLVED — Rolled back, corrected fix deployed sha-c4d82e1. | **A+** |
| 7 | **SEV2** | Elasticsearch outage | ES data node evicted (ephemeral storage), StatefulSet blocked by ResourceQuota, scheduling constrained by PVC node affinity. | ✅ RESOLVED — Quota fixed, node freed, cluster GREEN. | **Pending** |
| 8 | **SEV2** | RDS connection exhaustion | analytics_reader with no connection limit, 256 connections from analytics dashboard + manual report pod with 12 parallel workers. | ✅ RESOLVED — Dashboard scaled to 0, queries killed, CONNECTION LIMIT 20 set. | **Pending** |
| 9 | **SEV3** | TLS cert renewal failure | Wildcard cert for *.novamart.com failing renewal for 23 days. IAM key deactivated by security automation. 7 days to expiry when caught. | ✅ RESOLVED — New IAM key, cert renewed. IRSA migration planned. | **Pending** |
| — | **TBD** | Potential security incident | Legacy Jenkins with admin/admin, repeated logins from peered VPC (172.31.4.89). Most recent: today 9:14 AM. | 🔴 ACTIVE — Password changed, investigation starting. | **—** |

---

## DETAILED INCIDENT RECORDS

### INCIDENT 1: Order Service Timeout (SEV2) ✅

```
TRIGGER:      PagerDuty — HighErrorBudgetBurn_OrderService (3.2x 1h burn rate)
FIRED:        Mon 8:47 AM
ACKED:        Mon 9:04 AM
RESOLVED:     Mon 9:30 AM
TTR:          ~26 min from ack
BUDGET BURN:  74% → 65.2% (8.8% consumed)

ROOT CAUSE:
  Deploy sha-e7b31d4 set inventory_client_timeout_ms: 800ms.
  Inventory p99 = 1.08s at production scale. Canary passed at 
  2,400 rpm (low morning traffic), failed at 6K+ rpm.

FIX: inventory_client_timeout_ms: 2000ms, retries: 1

FOLLOW-UP TICKETS:
  PLAT-915 — Circuit breaker (Derek PR) — MERGED, DEPLOYED TO PRODUCTION Mon
  PLAT-916 — Canary analysis improvements — DONE ✅ (Lisa merged Fri)
  PLAT-917 — Investigate inventory p99 (assigned Nina)

CIRCUIT BREAKER STATUS:
  PR merged Thu. Staging tested Fri. Production canary deploy Mon 2:43 PM.
  Config: failureRateThreshold: 25, slowCallDurationThreshold: 2000ms,
  slidingWindowSize: 100 (TIME_BASED 60s recommended, Derek used COUNT).
  Low-confidence fallback: available:true (product-approved business decision).
  stockCache: TODO (PLAT-923), follow-up PR from Derek.
  Metric: orders_accepted_without_stock_verification_total.
```

### INCIDENT 2: Terraform State Lock (SEV4) ✅

```
TRIGGER:      Slack — Alex Kim
REPORTED:     Mon 9:01 AM
RESOLVED:     Mon 9:42 AM
ROOT CAUSE:   Jenkins CI job crashed at 2:14 AM, DynamoDB lock held.
FIX:          force-unlock after verifying no active applies.
```

### INCIDENT 3: Redis Cache Pressure (SEV3) ✅

```
TRIGGER:      Slack — Ryan Mitchell
REPORTED:     Mon 11:02 AM
RESOLVED:     Tue 2:15 AM (node upgrade r6g.large→r6g.xlarge)
CONFIRMED:    Tue 8:55 AM

ROOT CAUSE:   Quarterly catalog refresh: 182K products → 435K 
              cache keys exceeded Redis maxmemory.

POST-UPGRADE (as of Mon Wk2):
  Hit rate: 94.1%, Evictions: 0/min, Memory: 54%
  Cost: +$183/month

FOLLOW-UP:
  PLAT-918 — Scale Redis ✅
  PLAT-919 — Redis alerting ✅ (confirmed correctly scoped Thu)
  PLAT-920 — Quarterly refresh capacity planning gate
```

### INCIDENT 4: Shadow Kafka / Data Pipeline (SEV3) ✅ Mitigated — GDPR FILED

```
TRIGGER:      Jake Torres discovered unknown StatefulSets
REPORTED:     Mon 2:38 PM
MITIGATED:    Wed 12:15 PM (NetworkPolicy v3, S3 hardened, Atlas blocked)
PCI REVIEW:   Thu 2:00-2:38 PM — COMPLETED
GDPR FILED:   Fri 1:15 PM — SA-2024-0119-NM

FULL PIPELINE SCOPE:
  Component 1: Kafka brokers (6) + ZooKeeper (3)
    - Bitnami images from Docker Hub (ECR migration in progress Mon Wk2)
    - Deployed Nov 9 by tom.chen@novamart.com
    - PDB applied, NetworkPolicy v3 restricting access

  Component 2: Kafka Connect (confluent cp-kafka-connect:7.5.3)
    - orders-sink-s3: RUNNING ✅ (orders + order_items → S3)
    - payments-sink-mongo: FAILED (blocked by NetworkPolicy v3)

  Component 3: ETL CronJob (etl-daily-aggregation, daily 2 AM)
    - Reads from production orders RDS (analytics_reader user)
    - Writes to Kafka topics → S3 and Atlas sinks
    - STATUS: RUNNING (exit code 1 expected — Atlas sink health check)
    - Alertmanager silence applied (7-day, expired Mon Jan 25)

  Component 4: Analytics Dashboard
    - novamart-analytics-dashboard:latest (Docker Hub)
    - Direct connection to production orders RDS
    - ⚠️ CAUSED INCIDENT 8 (Mon Wk2) — excessive connections
    - Currently: 1 replica, HPA frozen at max 1
    - analytics_reader CONNECTION LIMIT 20 applied

  Component 5: MongoDB Atlas (EXTERNAL) — 🔴 BLOCKED + PRESERVED
    - Free-tier personal account (tom_chen_dev)
    - 42,847 documents, ~25-30K unique EU customers
    - GDPR Art. 4(1) personal data (email, name, billing country)
    - No encryption at rest, no VPC peering, public internet
    - STATUS: BLOCKED (NetworkPolicy egress deny since Wed Wk1)
    - LEGAL HOLD: Do not purge until Rachel Torres clears

  Component 6: Order Data Export CronJob (order-prod namespace)
    - Originally deployed by Nina Petrov (Dec 20)
    - ✅ ONBOARDED to ArgoCD by Nina (Sat Jan 20)
    - ✅ Migrated to KMS-encrypted bucket path
    - ✅ Tested (2 AM run successful)

CONTAINMENT STATUS (as of Mon Wk2):
  ✅ Atlas egress: BLOCKED + data preserved per Legal
  ✅ S3 exports: ENCRYPTED (KMS/CMK, logging, versioning)
  ✅ Kafka network: RESTRICTED (NetworkPolicy v3)
  ✅ All S3 writers: KMS permissions verified and fixed
  ✅ analytics_reader: CONNECTION LIMIT 20
  ✅ data-analytics namespace: ALL non-platform users read-only (RBAC)
  ✅ Nina's CronJob: onboarded to ArgoCD
  ⚠️ RDS egress: ipBlock CIDR-based (fragile on Multi-AZ failover)
  ⚠️ ECR migration: in progress (Tom, Mon-Tue Wk2)

KMS KEY:
  Alias: alias/novamart-data-exports
  Key policy (FINAL):
    platform-admin (full),
    order-data-export-role (encrypt/generate),
    data-analytics-role (decrypt),
    kafka-connect-s3-writer (encrypt/generate) ← added Thu Wk1

S3 BUCKETS:
  novamart-data-exports:
    ✅ Public access blocked, KMS/CMK, logging, versioning
    ❌ No lifecycle/retention policy
    ❌ Not in Terraform

  novamart-analytics-datalake:
    ✅ Public access blocked, KMS/CMK, logging, versioning
    ❌ No lifecycle policy
    ❌ Not in Terraform

GDPR STATUS:
  - Art. 33 supervisory notification: FILED Fri 1:15 PM
    Reference: SA-2024-0119-NM
    Filed by: Rachel Torres (Legal)
    Includes: full remediation table, containment actions,
              search cluster (C-5), prevention timeline
  - Art. 34 (customer notification): prepared, holding pending
    supervisory authority guidance
  - 48-hour acknowledgment expected from supervisory authority
  - Supervisory authority may request on-site technical review
    within 30 days — all controls must be demonstrable

PCI MEETING OUTCOMES (Thu Wk1):
  James approved GDPR notification.
  Atlas stays blocked + preserved.
  ETL CronJob keeps running.
  Drift detection committed Monday.
  Tom meeting scheduled Monday 10 AM.
  Board presentation: James presenting remediation timeline.

COMMITTED REMEDIATION TIMELINE (board-visible):
  ┌────────────────────────────────┬──────────────┬─────────┬──────────────┐
  │ Control                        │ Date         │ Owner   │ Status       │
  ├────────────────────────────────┼──────────────┼─────────┼──────────────┤
  │ Drift detection CronJob        │ Mon Jan 22   │ User    │ ✅ DONE      │
  │ Kafka pipeline SECURED (R-6a)  │ Fri Jan 26   │ User    │ In progress  │
  │ Image admission (warn mode)    │ Fri Jan 26   │ User    │ Not started  │
  │ Image admission (enforce mode) │ Fri Feb 2    │ User    │ Not started  │
  │ Kafka FULLY ONBOARDED (R-6b)   │ Fri Feb 9    │ User    │ Not started  │
  │ Self-service pipeline template │ Mid-Feb      │ User    │ Not started  │
  │ Default-deny NetworkPolicy     │ Mid-Feb      │ User    │ Not started  │
  │ Egress gateway                 │ End of Q1    │ User    │ Not started  │
  └────────────────────────────────┴──────────────┴─────────┴──────────────┘

TOM CHEN ONBOARDING (Mon Wk2):
  Meeting completed Mon 10:00-10:55 AM. Productive. 
  Tom energized, has clear sprint plan:
    Tue: ECR image migration (Kafka, ZK, Connect)
    Wed: Linkerd mesh injection
    Thu-Fri: Monitoring dashboards, backup config, runbook
  Tom has existing Grafana dashboards (never deployed — no access).
  Atlas decommission: pending Legal clearance from Rachel.
  Tom's RBAC: read-only in data-analytics until Legal hold lifted.
  Wei's RBAC: read-only in data-analytics (restricted Mon after Incident 8).

GAPS IN HANDLING:
  - Initial NetworkPolicy (Mon Wk1) exposed ZK port
  - MongoDB Atlas block was accidental
  - NetworkPolicy v3 RDS egress uses ipBlock CIDR (fragile)
  - KMS key policy missed kafka-connect-s3-writer (fixed Thu, self-identified)
  - analytics_reader CONNECTION LIMIT not set until Incident 8 forced it
  - Wei's RBAC not restricted until after he created a pod in preserved namespace
  - HALF_EVEN rounding not proactively flagged to finance (caught Fri by Martha)
```

### INCIDENT 5: Zombie Jenkins Agents / CI Pool Exhaustion (SEV2) ✅

```
TRIGGER:      David Okafor reports builds failing since 7 AM
REPORTED:     Tue 7:45 AM
RESOLVED:     Tuesday
ROOT CAUSE:   JVM memory leak in Jenkins agent pods.
FIX:          Build discard config + heap monitoring.
STATUS:       Jenkins heap stable. 6 agent pods (normal).
⚠️ UNVERIFIED: Build discard may be too aggressive for >30 builds.
```

### INCIDENT 6: Payment Service Currency Bug (SEV2) ✅

```
TRIGGER:      PagerDuty — PaymentServiceHighErrorRate
FIRED:        Wed 2:47 PM
ACKED:        Wed 2:48 PM
ROLLBACK:     Wed 2:53-2:58 PM
FIX DEPLOYED: Wed 4:28 PM (sha-c4d82e1)
TTR:          ~13 min ack-to-baseline (rollback)
BUDGET BURN:  91.3% → 87.1% (4.2% consumed)

BLAST RADIUS:
  847 failed payments, 89 failed refunds, 34 failed status checks
  ~500-700 unique customers affected

ROOT CAUSE:   BigDecimal.divide() without RoundingMode in 
              CurrencyService.convertAmount(). Non-terminating 
              decimal pairs (GBP/SEK/DKK/NOK→EUR) throw 
              ArithmeticException.

CORRECTED FIX (sha-c4d82e1):
  Three BigDecimal.divide() sites fixed: HALF_EVEN, scale 2
  36 new tests covering all 14 EU currency pairs
  Finance confirmed HALF_EVEN for EU compliance

ROUNDING RECONCILIATION (discovered Fri Wk1):
  $47.23 discrepancy across 847 retried transactions.
  Three rounding behaviors in one week's data:
    1. Pre-Wednesday: double arithmetic (IEEE 754)
    2. Wednesday rollback window (2:58-4:28 PM): same double arithmetic
    3. Post-Wednesday 4:28 PM: BigDecimal HALF_EVEN
  Martha Reeves (Finance) handled reconciliation with David Okafor.
  One-time adjustment entry. All future transactions: HALF_EVEN consistent.
  ⚠️ USER GAP: Should have flagged this to finance Wednesday.
     Noted in personal log but didn't close the loop. Martha found 
     it in reconciliation Friday. Credibility hit was avoidable.

VELOCITY FREEZE: Payment-service budget 86.0%. Freeze in effect.
```

### INCIDENT 7: Search / Elasticsearch Outage (SEV2) ✅

```
TRIGGER:      PagerDuty — SearchServiceHighErrorRate (>10% for 5 min)
FIRED:        Fri 5:47 AM
ACKED:        Fri 5:53 AM (waking up)
ROOT CAUSE:   Fri 6:06 AM (13 min from ack)
ES-DATA-3 BACK: Fri 6:22 AM
CLUSTER YELLOW: Fri 6:28 AM (error rate 3.8%)
CLUSTER GREEN:  Fri 6:52 AM (error rate 0.2%)
TTR:           ~65 min ack-to-GREEN
PEAK ERROR:    19.8%

ROOT CAUSE CHAIN:
  1. No ILM policy (7-month-old TODO from Carlos Mendez)
     → unbounded index growth → ephemeral storage exhaustion
  2. Node ip-10-0-42-44 evicted es-data-3 (ephemeral-storage pressure)
  3. Karpenter preempted es-data-3 for its own workload
  4. StatefulSet tried to recreate → ResourceQuota blocked (64Gi limit)
  5. Quota fix → pod created but PENDING (no node capacity in AZ)
  6. PVC node affinity locked to ip-10-0-42-44 (gp2 EBS AZ-bound)
  7. Freed capacity by evicting Kibana + 2 Jenkins agents from node
  8. es-data-3 scheduled → 127 local shards recovered, 14 corrupt 
     (peer recovery from replicas)

CRITICAL MOMENT:
  es-data-2 had 4/5 consecutive readiness probe failures.
  Scaled search-service from 6→2 replicas to reduce query pressure.
  es-data-2's 5th probe PASSED (by one cycle). 
  If it had failed → cascade to 40-60% errors.

STUCK POINT (RULE 12):
  Quota was obstacle 1 (cleared). 
  AZ-bound PVC scheduling was obstacle 2 (cleared by freeing node).
  Corrupt segments were obstacle 3 (handled by peer recovery).

COLLATERAL:
  - Alex's Jenkins build killed (40 min requeue)
  - catalog-service 2/3 replicas (unrelated eviction, recovered)
  - PDB applied with wrong labels on first attempt (wasted ~60 sec)

TRIAGE (RULE 3 — RED HERRINGS):
  Signal 1: Search errors — PRIMARY (incident)
  Signal 2: Kafka consumer lag — PARKED (recovery from Thu connector restart)
  Signal 3: Redis hit rate 91.2% — PARKED (low-traffic noise)
  Both correctly identified as unrelated within 60 seconds.

SYSTEM CONTEXT:
  Search cluster deployed by Carlos Mendez (departed Sep 2023).
  No ArgoCD, no ILM, gp2 storage, stale runbook (Jun 2023).
  Helm chart (elasticsearch-7.17.x). Nobody formally assigned 
  ownership after Carlos left. Wei's team "kept an eye on it."
  RULE 10: Haunted Forest — stale docs, departed owner.

CLEANUP COMPLETED (Fri 7:15 AM):
  ✅ PDB replaced with correct labels (maxUnavailable: 1)
  ✅ ResourceQuota right-sized (80Gi requests, annotated with reason)
  ✅ Kibana rescheduled and running
  ✅ catalog-service 3/3 recovered
  ✅ search-service 6/6 running

FOLLOW-UP NEEDED:
  - ILM policy (root cause of storage exhaustion)
  - Search cluster ownership (platform team interim, effective Mon)
  - ArgoCD onboarding for search namespace
  - gp2 → gp3 migration
  - ES pod PriorityClass (prevent Karpenter preemption)
  - Ephemeral storage limits on stateful workloads
  - Pod topology spread for catalog-service
  - EKS upgrade: exclude search namespace until onboarded (Priya informed)
```

### INCIDENT 8: RDS Connection Exhaustion (SEV2) ✅

```
TRIGGER:      PagerDuty — RDSHighConnectionCount (>350)
FIRED:        Mon Wk2 11:07 AM
ACKED:        Mon Wk2 11:11 AM
ROOT CAUSE:   Mon Wk2 11:15 AM (4 min from ack)
MITIGATED:    Mon Wk2 11:17 AM (dashboard scaled to 0, queries killed)
RECOVERED:    Mon Wk2 11:22 AM (connections 183, latency baseline)
TTR:          ~11 min from ack
BUDGET BURN:  Order-service: 64.5% → 61.8% (2.7% consumed in 25 min)
              Payment-service: held at 86.0% (latency below error threshold)

ROOT CAUSE:
  analytics_reader user: NO CONNECTION LIMIT.
  All service users had limits (order: 100, payment: 50, inventory: 30).
  analytics_reader was the only one without.
  
  Monday morning analyst reports via analytics dashboard:
    - 2 dashboard replicas (HPA scaled from 1 to 2 on CPU)
    - No connection pooling — each query opens new connection
    - Heavy analytical queries: multi-table JOINs, full table scans
    - All waiting on IO (DataFileRead — sequential scans)
  
  Tipping point: Wei's manual report pod (created 10:52 AM):
    - weekly_summary.py spawns 12 parallel workers for export
    - 12 simultaneous full-table-scan queries hit at 11:00 AM
    - Combined: 256 analytics_reader connections (184 active, 72 idle)
    - Total: 459/500 connections, climbing ~4/min toward limit

  Feedback loop: more queries → higher CPU → HPA scales up → 
  more replicas → more connections → more IO → slower queries → 
  connections held longer → count climbs faster

CONNECTION SPIKE:
  10:50 AM  187 (normal)
  11:00 AM  243 (+51 in 5 min — 12 parallel workers start)
  11:07 AM  418 (alert fires)
  11:13 AM  447
  11:17 AM  Fix applied
  11:22 AM  183 (recovered)

FIX APPLIED (parallel execution):
  Terminal 1: kubectl scale deployment analytics-dashboard --replicas=0
  Terminal 2: pg_terminate_backend on all analytics_reader active sessions
  Terminal 3: ALTER ROLE analytics_reader CONNECTION LIMIT 20

ADDITIONAL DISCOVERY:
  47 connections from 10.0.43.22 (NAT Gateway) → traced to 
  Metabase EC2 instance (analytics-metabase, t3.medium) also 
  using analytics_reader. Shadow infrastructure item #9.
  
  Also found: novamart-jenkins-legacy (t3.large) in same subnet.
  Shadow infrastructure item #10. admin/admin credentials.
  → LED TO POTENTIAL SECURITY INCIDENT (see below).

POST-INCIDENT ACTIONS:
  ✅ analytics_reader CONNECTION LIMIT 20
  ✅ Dashboard restored at 1 replica, HPA frozen at max 1
  ✅ Wei's RBAC restricted to read-only in data-analytics
  ✅ Wei's partial report salvaged + export re-run safely (single-threaded)
  ⚠️ Wei's manual pod created in Legal-hold namespace (documented, not escalated)

WEI CONTEXT:
  Wei ran kubectl run to generate a VP report. Dashboard was slow.
  Script spawns 12 parallel workers (Tom wrote it for speed).
  Wei didn't realize the parallel mode existed.
  Same pattern: "reasonable action without full context."
  Wei acknowledged: "I did exactly what I was worried Tom would do."

USER GAP:
  analytics_reader CONNECTION LIMIT was on TODO list since Wed Wk1.
  Committed to Rachel's table as R-7 (Wed Jan 24). Incident happened 
  Mon Jan 22 — two days before the committed date. ALTER ROLE takes 
  2 seconds. Should have done it immediately when identified, not 
  deferred. Priority order ≠ exposure order for database-level fixes.
```

### INCIDENT 9: TLS Certificate Renewal Failure (SEV3) ✅

```
TRIGGER:      Slack — CertificateExpiringWithin7Days
FIRED:        Mon Wk2 2:00 PM
EXPIRY:       Jan 29, 2024 (7 days from discovery)
CERT RENEWED: Mon Wk2 2:27 PM
NEW EXPIRY:   Apr 21, 2024

CERTIFICATE:
  Name: novamart-tls-wildcard
  Domains: *.novamart.com, novamart.com
  Issuer: Let's Encrypt (letsencrypt-prod ClusterIssuer)
  Challenge: DNS-01 via Route53
  Secret: novamart-tls-wildcard (production namespace)
  ACME account: registered to carlos.mendez@novamart.com

ROOT CAUSE CHAIN:
  Jun 15, 2023  — Carlos creates IAM user + static access key
  Sep 2023      — Carlos departs. No cert infra handover.
  Oct 31        — Last successful renewal
  Nov 15        — Security automation policy updated: last-used 
                   threshold reduced 90→30 days
  Dec 28        — security-automation deactivates key (58 days 
                   since last use, >30 day threshold)
  Dec 30        — cert-manager renewal attempt: rate limited (429)
                   (prior failed authorizations from key failure)
  Jan 6         — retry: DNS-01 challenge timeout (key inactive)
  Jan 10        — retry 5/5 exhausted. "Manual intervention required."
  Jan 22        — 7-day expiry alert fires (first human notification)

  RULE 2: Security automation did the right thing (deactivate 
  old key). Policy change in November set a tighter threshold. 
  Nobody audited existing keys against the new threshold.

ALERT GAP:
  CertificateExpiringSoon alert: fires at 7 days before expiry.
  No alert for renewal FAILURE. Renewal has been failing for 23 
  days with zero human notification.

FIX:
  1. Created new IAM access key for cert-manager-dns01 user
  2. Updated K8s secret (route53-credentials)
  3. Updated ClusterIssuer (new accessKeyID)
  4. Verified Route53 permissions via policy simulation
  5. Cleared failed CertificateRequest/Order objects
  6. Triggered renewal via annotation
  7. Cert issued in 2 minutes (DNS propagation + validation)
  8. Verified live cert via openssl s_client

BACKUP PLANS (documented but not needed):
  Plan B: Switch to HTTP-01 (can't do wildcards)
  Plan C: New ACME account (clean rate limit history)
  Plan D: Manual certbot generation + secret upload

FOLLOW-UP:
  - IRSA migration for cert-manager (eliminate static keys) — this week
  - Alert for renewal failures (not just expiry) — this week
  - Document cert infrastructure ownership
  - Aisha adding pre-deactivation notification to security automation
    (7-day warning before key deactivation, sent to owner + manager)
  - Update ACME account registration to current employee email

AISHA CONTEXT ON SECURITY AUTOMATION:
  Policy checks: key age >180 days AND last used >threshold.
  Threshold was 90 days, changed to 30 days Nov 15.
  cert-manager key: 58 days since last use at time of deactivation.
  Would have survived under old policy. Caught by new threshold.
  Nobody audited existing keys when threshold changed.
  Aisha adding pre-deactivation warnings.
```

### POTENTIAL SECURITY INCIDENT: Legacy Jenkins Unauthorized Access (ACTIVE)

```
TRIGGER:      Marcus Webb investigation of legacy EC2 instances
DISCOVERED:   Mon Wk2 2:42 PM
STATUS:       🔴 ACTIVE — UNDER INVESTIGATION

LEGACY JENKINS INSTANCE:
  Name: novamart-jenkins-legacy
  Instance: i-0e8g3f..., t3.large, Amazon Linux 2
  IP: 10.0.43.201 (private subnet, routes through NAT)
  Launched: March 2, 2022 (~2 years old)
  Launched by: former-devops@novamart.com (account disabled)
  Jenkins version: 2.361 (current: 2.426, 65 versions behind)
  Credentials: admin/admin (DEFAULT — changed by Marcus at ~2:40 PM)
  Security group: SSH + 8080 from 10.0.0.0/8 (entire VPC)
  
  14 configured jobs, all disabled except:
    - monthly-db-backup: dumps production orders database → S3
      Last successful: November 1, 2023
      Last attempted: December 1, 2023 — FAILED
      Failing silently for 2 months (no alerts)

UNAUTHORIZED ACCESS EVIDENCE:
  Jenkins audit log — last 30 days:

  Date       User    Source IP       Notes
  Dec 3      admin   10.0.43.201    Localhost (cron? maintenance?)
  Dec 5      admin   10.0.41.33     EKS node (internal)
  Dec 12     admin   10.0.42.88     EKS node (internal)
  Jan 8      admin   10.0.41.33     EKS node (internal, 2 logins)
  Jan 15     admin   172.31.4.89    ⚠️ PEERED VPC (2 logins)
  Jan 19     admin   172.31.4.89    ⚠️ PEERED VPC
  Jan 22     admin   172.31.4.89    ⚠️ PEERED VPC (TODAY, 9:14 AM)

  172.31.4.89 is NOT in our VPC CIDR (10.0.0.0/16).
  Traced to VPC peering connection to vpc-0f8a3c... tagged 'legacy-staging'.
  Peering created August 2023.
  
  4 logins from peered VPC in last 7 days. Most recent: 4 hours ago.

RISK ASSESSMENT:
  Through Jenkins admin access, attacker could:
  - Execute arbitrary code on the instance
  - Access monthly-db-backup job credentials (production RDS dump creds)
  - Access anything the instance's IAM role can reach
  - Access production RDS, internal services via network
  - View build history/configuration of all 14 jobs
  
  UNKNOWN:
  - Identity of 172.31.4.89 (person? compromised instance? automation?)
  - What actions were taken after login (Jenkins activity log detail?)
  - Whether db-backup credentials were accessed
  - Whether legacy-staging VPC is legitimate infrastructure
  - Whether the peered VPC route tables allow production access
  - Whether the 10.0.4x.xx (EKS node) logins were from pods or humans

PASSWORD CHANGED: ~2:40 PM Mon by Marcus (attacker locked out)
SECURITY GROUP: Being restricted by Marcus (platform team IPs only)

STATUS: 
  Marcus found this at 2:42 PM. 
  User has been informed.
  User has NOT YET responded to Marcus.
  Aisha has NOT YET been informed.
  No formal incident declared yet.

>>> THIS IS THE ACTIVE SITUATION <<<
```

---

### SERVICEPROFILE TIMEOUT ISSUE (not numbered — consequence of approved change)

```
TIMELINE:
  Wed Wk1 10:47 PM — Derek's ServiceProfile merged (isRetryable changes)
  Wed Wk1 10:47 PM — Derek ALSO added timeout: 5s on all 4 routes (post-approval)
  Thu Wk1 3:25 PM  — Derek notices NR flags, reports to user
  Thu Wk1 3:36 PM  — User: remove all timeouts
  Thu Wk1 3:46 PM  — ArgoCD syncs fix
  Thu Wk1 3:54 PM  — NR rate back to zero ✅

IMPACT: 17 hours, 0.3% NR, error rate 0.09%→0.41%
  Budget: payment-service 87.1% → 86.0% (1.1% consumed)

ROOT CAUSE: Derek added timeout: 5s post-approval. Payment 
  gateway tail latency 6-7s during slow periods.

USER GAP: Approved based on Slack description, not final merged diff.
PROCESS FIX: Review merged commit, not PR description.
  "Any commit after approval gets a re-review ping."
  Derek acknowledged.

PATTERN: Static per-route timeouts are a recurring anti-pattern.
  Appeared twice in one day (payment ServiceProfile, then 
  inventory in CB PR review). CB is the correct tool.
```

---

### ADDITIONAL EC2 DISCOVERIES (Mon Wk2)

```
ANALYTICS-METABASE (i-0d7f2e...)
  Launched: August 14, 2023
  Launched by: carlos.mendez@novamart.com (CloudTrail)
  t3.medium, Ubuntu 22.04
  Metabase 0.47.2 (current: 0.48.x)
  Connected to:
    1. orders-db-prod (analytics_reader) — confirmed source of 
       47 NAT gateway connections during Incident 8
    2. reporting-db-prod — UNKNOWN RDS INSTANCE, NOT IN TERRAFORM
  Security group: inbound 3000 from 10.0.0.0/8, 443 from 0.0.0.0/0
    ALB: metabase.internal.novamart.com (internal, not internet-facing)
    0.0.0.0/0 rule is stale — bad hygiene but not currently exploitable
  Users: 7 accounts (Carlos admin, Wei, Tom, 4 unknown — likely product/finance)
  Auth: local Metabase passwords, no MFA, no SSO
  Shadow infrastructure item #9

REPORTING-DB-PROD (RDS, unknown):
  Connected to by Metabase. Not in Terraform. Not in any inventory.
  Investigation pending.
  Shadow infrastructure item #11

NOVAMART-JENKINS-LEGACY (i-0e8g3f...):
  See "Potential Security Incident" section above.
  Shadow infrastructure item #10
```

---

## SHADOW INFRASTRUCTURE TALLY

```
# | Type | Location | Owner | Status
1  | Kafka brokers (6) | K8s: data-analytics | Tom Chen | Containment, ECR migration Mon
2  | ZooKeeper (3) | K8s: data-analytics | Tom Chen | Containment, ECR migration Mon  
3  | Kafka Connect | K8s: data-analytics | Tom Chen | Containment
4  | ETL CronJob | K8s: data-analytics | Tom Chen | Running (expected exit 1)
5  | Analytics Dashboard | K8s: data-analytics | Tom Chen | 1 replica, HPA frozen
6  | MongoDB Atlas | External (free-tier) | Tom Chen | BLOCKED + Legal hold
7  | ES cluster (master+data) | K8s: search | Carlos (departed) | Platform interim ownership
8  | Kibana | K8s: search | Carlos (departed) | Running
9  | Metabase EC2 | EC2: analytics-metabase | Carlos (departed) | Under investigation
10 | Legacy Jenkins EC2 | EC2: novamart-jenkins-legacy | former-devops (departed) | 🔴 SECURITY INCIDENT
11 | Reporting DB | RDS: reporting-db-prod | Unknown | Under investigation
-- | S3: novamart-data-exports | AWS S3 | Tom/Nina | Hardened, not in Terraform
-- | S3: novamart-analytics-datalake | AWS S3 | Tom | Hardened, not in Terraform
-- | Order data export CronJob | K8s: order-prod | Nina | ✅ ONBOARDED to ArgoCD

Total: 11 active shadow items + 2 unmanaged S3 buckets
Onboarded: 1 (Nina's CronJob)
Three departed owners. One security incident.
```

---

## SIMULATION TIMELINE

```
WEEK 1 — MONDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
09:00  On-call shift starts
09:04  ACK PagerDuty                                        [INC-1 START]
09:05  Dashboard + PromQL investigation
09:06  ArgoCD/rollout, GitOps diff
09:08  Unblock Alex (state lock)                             [INC-2 START]
09:10  Hypothesis: 800ms timeout too aggressive
09:13  CONFIRMED via trace
09:14  Decision: 2000ms
09:15  GitOps hotfix pushed
09:30  RESOLVED                                              [INC-1 END]
09:42  Alex self-resolved                                    [INC-2 END]
09:45  CB meeting scheduled
09:50  EKS 1.29 upgrade research
11:02  Ryan reports catalog slowness                         [INC-3 START]
11:37  Root cause: Redis memory
11:40  Maxmemory bumped                                      [INC-3 MITIGATED]
12:00  Lunch
13:00  Sarah Q1 priorities
13:20  Priya Linkerd PR review
14:00  Circuit breaker design meeting
14:48  Kafka investigation                                   [INC-4 START]
15:06  Redis/catalog alerts deployed (PLAT-919)
15:15  PDB + NetworkPolicy for Kafka                         [INC-4 MITIGATED]
15:30  Redis staging failover test
16:15  EOD

MONDAY NIGHT / TUESDAY EARLY AM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
01:50  NetworkPolicy v2 fix (ZK port). EBS encryption check.
02:00  Redis upgrade                                         [INC-3 RESOLVED]
02:22  Terraform updated

TUESDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
07:45  David reports builds failing                          [INC-5 START]
~AM    Jenkins zombie pods resolved                          [INC-5 RESOLVED]

WEDNESDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
10:00  Standup (James present) — PCI insertion               [STANDUP — A+]
10:35  Aisha security response, audit coordination
10:45  Cluster audit (scripted sweep)
11:45  Audit: 9 unmanaged workloads, 4 PCI critical
12:15  NetworkPolicy v3. Atlas blocked. S3 remediation.
12:42  Aisha: PCI review Thu 2 PM
13:00  S3 KMS encryption + hardening
14:00  PCI findings document drafting
14:47  PagerDuty: PaymentServiceHighErrorRate                [INC-6 START]
14:48  ACK. Root cause 4 min.
14:53  Rollback initiated
14:58  Rollback complete
15:01  Baseline restored
15:22  David: scope all BigDecimal fixes
15:45  PR review (caught applyDiscount scale bug)
16:08  sha-c4d82e1 deployed
16:21  Promoted to 100%
16:43  Watch clean                                           [INC-6 RESOLVED]
16:50  Sarah: EKS runbook → Monday
16:55  PCI document 40%→75%
18:00  EOD

WEDNESDAY NIGHT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
20:30  Aisha: Atlas free-tier details. GDPR implications.
22:47  Derek ServiceProfile merged (+unauthorized timeout)
23:45  Wei: Atlas credentials, logging in

THURSDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
08:30  Login. Overnight clean.
08:33  Aisha: document language adjustments
08:38  Priya: namespace ordering for dry-run
08:40  Derek: CB PR up
08:42  James email: wants exposure/notification/remediation/prevention
08:45  PCI document final push (75%→100%)
09:50  Pre-dry-run verification
10:00  Linkerd staging dry-run with Priya                    [DRY-RUN]
       Four namespaces clean ✅
11:05  Priya: draft production runbook
11:30  Wei: Atlas data fields (42,847 docs, PII confirmed)
12:15  PCI document complete (100%)
12:50  Pre-read distributed
14:00  PCI Emergency Review Meeting                          [PCI MEETING]
14:04  James asks prevention + timeline questions
14:04  Phone buzzes: ETL alert
14:04  Answers James: 5 prevention controls, "wall and door"
14:10  Nina validates "make the right thing easy"
14:18  Rachel: specific dates + evidence by 10 AM Fri
14:28  Revised timeline with self-service
14:32  James: "Do that."
14:38  Meeting ends
14:39  Pull out phone — ETL alert 35 min old
14:41  ETL investigation begins
       - Alert from KubeJobFailed (12h delay), NOT PLAT-919
       - ETL data processing succeeded
       - "Failure" = Atlas sink health check (intentionally blocked)
       - KMS PERMISSION GAP: kafka-connect-s3-writer missing from key policy
       - S3 writes working by accident (bucket default encryption)
14:56  Investigation complete
15:00  KMS key policy fix: added kafka-connect-s3-writer
15:05  Verified ✅
15:08  Wei: full transparency about KMS gap
15:12  Alertmanager silence for ETL (7-day)
15:18  Sarah: "Wei used the word 'refreshing.'"
       Board presentation: remediation timeline going to CEO.
15:25  Derek: 0.3% NR flags on payment authorize
15:27  Investigation — NR from ServiceProfile timeout
15:36  User: remove timeouts. Derek pushes fix.
15:46  ArgoCD syncs
15:54  NR rate zero ✅
15:58  CB PR review (platform-infra#347)                     [PR REVIEW]
16:18  Review: 3 blocking, 1 should-fix, 1 non-blocking
16:22  Derek agrees on timeout + stockCache
16:25  Nina pushback on overselling fallback (business math)
16:28  User: unblocks #2 with conditions (metric + code comment)
       Converts to should-fix. Blockers: timeout + stockCache.
16:33  Tells Derek: merge with TODO, stockCache follow-up Monday
16:35  Rachel's remediation dates table drafted
17:15  Drift detection CronJob skeleton written
17:47  EOD posted
17:50  Personal notes

FRIDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
05:47  SearchServiceHighErrorRate fires                      [INC-7 START]
05:53  Acked (waking up)
05:57  Three signals triaged (search=real, Kafka=parked, Redis=parked)
05:58  #incidents posted
06:00  Investigation: ES cluster YELLOW → RED, 1 data node missing
06:06  Root cause: eviction → quota → scheduling chain
06:07  ResourceQuota increased (64Gi→80Gi requests)
06:08  PDB applied (wrong labels — FIXED on second attempt)
06:15  Stuck: PVC AZ-locked, no node capacity                [RULE 12]
06:17  Parallel execution: scale search to 2 + evict pods from node
06:19  es-data-3 scheduling. es-data-2 passes readiness (by 1 cycle).
06:22  es-data-3 in cluster, shards recovering
06:28  Cluster YELLOW, error rate 3.8%
06:29  Scale search-service back to 4
06:31  Comms: Alex, Sarah, Ryan, Lisa
06:35  Monitor recovery loop
06:45  Scale search to 6
06:52  Cluster GREEN                                         [INC-7 END]
06:53  #incidents RESOLVED
07:15  Cleanup: PDB, quota, Kibana, catalog
08:00  Cluster stable 1h at GREEN
08:05  Drift detection script development
09:20  Drift detection tested against cluster (9 findings, all correct)
09:28  Rachel's comments on remediation table
09:33  Finance: $47.23 reconciliation discrepancy             [HALF_EVEN]
09:35  Finance response: full explanation + apology
09:40  Rachel's table revision (4 issues addressed)
09:58  Email to Rachel: R-3 enforcement approach, R-6 split,
       Note 4 reframing, search cluster C-5 added
10:02  Derek: merge CB PR with TODO
10:05  Drift detection finishing touches
10:30  Deploy drift detection to staging
10:45  Status check — all clear
10:50  Rachel: approved with one change (C-5 needs interim owner date)
10:53  Fixed: platform team interim effective Mon Jan 22
10:58  Rachel: "Filed at 1. Thank you."
11:02  Finance follow-up (Martha: one-time adjustment, future clean)
11:04  David: three-rounding-behavior question
11:30  EKS runbook polish
12:00  Lunch
13:15  Rachel files GDPR Art. 33 — SA-2024-0119-NM           [GDPR FILED]
14:00  Lisa's canary AnalysisTemplate PR review + approved    [PLAT-916 DONE]
14:30  Derek merges CB PR
15:00  Derek staging load test (clean)
16:30  EOD / weekly summary posted
       7 incidents. All resolved. Budgets tight.

FRIDAY NIGHT / SATURDAY / SUNDAY (QUIET)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
No pages Fri night / Sat / Sun.
Nina: added order-data-export to ArgoCD (Sat, self-initiated).
Marcus: added node-problem-detector to ArgoCD (Sun 11:42 PM).

Sun 15:47  Wei DM: Tom wants to access data-analytics namespace
           User responds: call Tom (Wei), restrict RBAC (user)
           Tom's RBAC reduced to read-only
           Aisha informed
           Tom called by Wei — understood, waiting for meeting
           Wei: "He asked if he's in trouble. I told him no."

WEEK 2 — MONDAY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
07:45  Login. Overnight clean.
07:20  Priya: EKS runbook ready, search namespace question
07:31  Tom: pre-meeting message (apology, read all Slack, Atlas context)
08:00  DRIFT DETECTION FIRST PRODUCTION RUN                  [COMMITTED DELIVERABLE]
       8 CRITICAL, 0 HIGH (node-problem-detector resolved)
       Nina's CronJob correctly identified as MANAGED ✅
       Sarah: "This is exactly what James wanted to see."
08:15  Tom response: not an ambush, framing, context
08:18  Priya: exclude search from EKS upgrade
08:20  Nina: ArgoCD onboarding acknowledged, asked for README
09:45  Meeting prep (1-page agenda)
10:00  Tom Chen Onboarding Meeting                           [TOM MEETING]
       Sarah, Wei, Tom, User. Productive.
       Tom: pipeline well-engineered, no self-service path existed.
       Sprint plan: ECR Tue, mesh Wed, monitoring Thu-Fri.
       Atlas: decommission when Legal clears.
10:55  Meeting ends. Sarah: "That went well."
11:07  PagerDuty: RDSHighConnectionCount                     [INC-8 START]
11:08  OrderServiceLatencyP99High
11:09  PaymentServiceLatencyP99High
11:10  InventoryServiceLatencyP99High
11:11  Acked. Investigation.
11:15  Root cause: analytics_reader 256 connections, no limit
11:17  Dashboard scaled to 0, queries killed, HPA frozen
11:18  CONNECTION LIMIT 20 set
11:22  Database recovered                                    [INC-8 RESOLVED]
11:24  Wei: admits manual report pod caused spike
11:25  Marcus: mystery pod is Wei's. Also found Metabase EC2.
11:27  Sarah: needs answers for James 11:30 meeting
11:29  Wei handled: honest, not punitive, RBAC restricted
11:30  Sarah briefed (one-liner for James)
11:31  #incidents posted
11:33  RBAC lockdown: all non-platform read-only in data-analytics
11:36  Wei's partial report salvaged
11:40  Dashboard restored at 1 replica, HPA frozen at max 1
12:15  Marcus findings: Metabase, legacy Jenkins, unknown RDS
       Jenkins: admin/admin, 7 users, monthly-db-backup (failing 2mo)
13:15  Wei export re-run (single-threaded, 14 min, database stable)
14:00  CertificateExpiringWithin7Days alert                  [INC-9 START]
       *.novamart.com expires Jan 29. Renewal failing 23 days.
14:08  Root cause: IAM key deactivated by security-automation Dec 28
14:10  New IAM key created
14:15  Cert-manager secret + ClusterIssuer updated
14:25  Renewal triggered
14:27  Certificate issued ✅                                  [INC-9 RESOLVED]
14:30  #platform-engineering update
14:35  Aisha: security-automation policy context
14:40  Marcus: Jenkins lockdown. Password changed.
14:42  Marcus: UNAUTHORIZED LOGINS from peered VPC           [SECURITY INCIDENT?]
       172.31.4.89 — legacy-staging VPC, 4 logins in 7 days
       Most recent: today 9:14 AM
14:43  Derek: CB canary at 5%, looking clean
14:44  Tom: ECR images ready, wants to do rolling restart

>>> CURRENT TIME: Monday 2:44 PM <<<
>>> POTENTIAL SECURITY INCIDENT ACTIVE <<<
>>> Marcus waiting for guidance <<<
>>> Derek's CB canary running <<<
>>> Tom wants to restart StatefulSets in Legal-hold namespace <<<
```

---

## LIVE SYSTEM CHANGES — CONSEQUENCE TRACKING

```
CHANGE                              APPLIED         VERIFIED?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

NetworkPolicy v3                    Wed Wk1         PARTIALLY
  (data-analytics namespace)                        ✅ Kafka ingress
                                                    ✅ S3 sink
                                                    ✅ Atlas blocked
                                                    ❓ RDS egress CIDR (fragile)
                                                    ❓ Analytics dashboard ext

KMS key policy                      Wed+Thu Wk1     ✅ FULLY VERIFIED
  (all 4 writer roles added)                        ✅ All S3 writes clean

S3 encryption + hardening           Wed Wk1         ✅ VERIFIED
  (both buckets, KMS/CMK)

Payment sha-c4d82e1                 Wed Wk1         ✅ Running stable
                                                    ✅ HALF_EVEN reconciliation 
                                                       handled with finance

Jenkins build discard               Tue Wk1         ✅ Stable 5+ days
                                                    ❓ Aggressive discard limit

Redis r6g.xlarge                    Tue Wk1         ✅ Stable (94.1%, 54% mem)
                                                    ❓ Terraform PR merged?

PLAT-919 PrometheusRule             Mon Wk1         ✅ Correctly scoped (confirmed)

Derek ServiceProfile                Thu Wk1         ✅ Timeout removed, NR zero
  (isRetryable only, no timeout)

ResourceQuota (search ns)           Fri Wk1         ✅ 80Gi, annotated
  (64Gi → 80Gi requests)

PDB (search ns)                     Fri Wk1         ✅ maxUnavailable: 1
  (replaced emergency PDB)

analytics_reader CONN LIMIT 20      Mon Wk2         ✅ Applied, tested
                                                    ✅ Dashboard at 1 replica stable

RBAC: Tom read-only (data-analytics) Sun Wk2        ✅ Applied, verified
RBAC: Wei read-only (data-analytics) Mon Wk2        ✅ Applied

Dashboard HPA frozen (max 1)        Mon Wk2         ✅ Applied

Cert-manager IAM key rotation       Mon Wk2         ✅ New key active
                                                    ✅ Cert renewed (Apr 21)
                                                    ❓ Old key still exists (inactive)

Jenkins admin password changed      Mon Wk2 ~2:40   ✅ Marcus applied
                                                    ❓ SG restriction in progress

Drift detection CronJob             Mon Wk2 8:00    ✅ First production run clean
  (monitoring namespace)                             ✅ 8 findings, all correct

CB deployed to staging              Fri Wk1         ✅ Load test clean
CB canary in production             Mon Wk2 2:43    ⏳ In progress (5%, 10 min in)

STILL UNVERIFIED:
  ❓ Jenkins build discard aggressive limit
  ❓ Redis Terraform PR merged?
  ❓ RDS egress CIDR fragility
  ❓ Analytics dashboard external access (NetworkPolicy v3)
  ❓ Metabase EC2 full assessment (Marcus investigating)
  ❓ reporting-db-prod (unknown RDS instance)
  ❓ Legacy Jenkins: what did the 172.31.4.89 logins do?
  ❓ Legacy-staging VPC: what is it, who owns it?
  ❓ Tom's ECR rolling restart: safe in Legal-hold namespace?
```

---

## OPEN ITEMS / PENDING WORK

### Active Tickets

```
PLAT-892  [In Progress]  EKS 1.28→1.29 upgrade runbook — mostly done, due today
PLAT-901  [In Review]   Linkerd PR #347 — needs re-review (Priya updates)
PLAT-910  [To Do]       GP3 EBS migration (next sprint, skip data-analytics + search)
PLAT-915  [Done ✅]     Circuit breaker — merged, staging tested, prod canary active
PLAT-916  [Done ✅]     Canary analysis — Lisa's AnalysisTemplate merged Fri
PLAT-917  [To Do]       Investigate inventory p99 (Nina)
PLAT-918  [Done ✅]     Redis scale-up
PLAT-919  [Done ✅]     Redis alerting (confirmed correctly scoped)
PLAT-920  [To Do]       Quarterly refresh capacity planning gate
PLAT-921  [In Progress] Kafka platform onboarding — Tom sprint started Mon
                         ECR images built. Rolling restart pending approval.
PLAT-922  [Done ✅]     PCI scope assessment — GDPR filed SA-2024-0119-NM
PLAT-923  [To Do]       stockCache implementation (Derek, follow-up PR Mon)
[NEEDED]  Cert-manager IRSA migration (this week)
[NEEDED]  Cert renewal failure alert (not just expiry)
[NEEDED]  AWS-level infrastructure audit (James wants by Wed)
[NEEDED]  Search cluster: ILM, PriorityClass, ArgoCD, gp2→gp3
[NEEDED]  Metabase assessment + containment
[NEEDED]  Legacy Jenkins investigation + decommission
[NEEDED]  reporting-db-prod investigation
[NEEDED]  VPC peering assessment (legacy-staging)
[NEEDED]  Read replica for analytics workloads
[NEEDED]  Self-service data pipeline template (mid-Feb committed)
[NEEDED]  Default-deny NetworkPolicy all namespaces (mid-Feb committed)
[NEEDED]  Egress gateway (end of Q1 committed)
[NEEDED]  S3 lifecycle/retention policies (both buckets)
[NEEDED]  S3 buckets → Terraform
[NEEDED]  Migrate secrets to External Secrets Operator
[NEEDED]  Security automation: pre-deactivation notifications (Aisha)
[NEEDED]  ACME account: update from Carlos email to current employee
[NEEDED]  weekly_summary.py: connection pooling / max-workers cap
```

### Pending Deliverables

```
- AWS-level audit scope → James wants by Wednesday
- Cert-manager IRSA migration → this week
- Cert renewal failure alert → this week
- Tom ECR migration → Tue (images ready, restart pending)
- Tom mesh injection → Wed
- Tom monitoring/backup/runbook → Thu-Fri
- R-6a (Kafka secured) → committed Fri Jan 26
- R-6b (Kafka fully onboarded) → committed Fri Feb 9
- EKS upgrade runbook → due today (mostly done)
- Production Linkerd upgrade → Priya drafting runbook, next Tue 2 AM
- Derek stockCache PR → this week
- Supervisory authority acknowledgment → expected within 48h of filing
  (filed Fri 1:15 PM → expected by Sun 1:15 PM, may arrive today)
```

---

## ERROR BUDGETS

```
SERVICE          BUDGET    TREND    NOTES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Order-service    61.8%     ↓↓       Three incidents (1, 7 indirect, 8)
                                    8 days remaining in window
                                    Baseline ~0.5%/day = ~57.8% at window end
                                    THIN. One more incident → freeze territory.
                                    CB deploying today (risk reduction).

Payment-service  86.0%     ↓        Three hits (6, ServiceProfile, held on 8)
                                    Velocity freeze in effect.
                                    Exception: corrective deploys only.

Search-service   No SLO    —        Define post-incident (next week)

Inventory-svc    Not tracked separately (part of order-service SLI)
```

---

## TEAM STATE

```
James Morrison (VP Eng):  Satisfied with prevention plan. Presenting 
                          remediation timeline to board. Wants AWS audit 
                          by Wednesday. Asked about order-service budget 
                          (Sarah briefing him now at 11:30 meeting).
                          Quote: "I want to know about every piece of 
                          infrastructure that isn't in a manifest somewhere."

Sarah Chen (manager):     Briefed for James 11:30 meeting. Set up Mon 
                          10 AM Tom meeting (successful). Forwarded drift 
                          scan to James. "Wei used the word 'refreshing.'"
                          Currently in meeting with James (11:30).

Aisha Rahman (Security):  Leading PCI/GDPR response. Updated shadow infra 
                          tally. Confirmed containment posture. Adding 
                          pre-deactivation notification to security automation.
                          Has NOT YET been informed of Jenkins unauthorized 
                          access finding. Needs to know IMMEDIATELY.

Rachel Torres (Legal):    GDPR Art. 33 filed (SA-2024-0119-NM, Fri 1:15 PM).
                          Awaiting supervisory authority acknowledgment.
                          "Don't purge Atlas until I say so."
                          Remediation table approved. Board-ready.

Wei Liu (Data Eng mgr):   RBAC restricted to read-only in data-analytics.
                          Admitted fault on manual report pod. "I did exactly 
                          what I was worried Tom would do." VP report deadline 
                          Tue 10 AM — export re-run completed (single-threaded).
                          Relieved about S3 data continuity.

Tom Chen (Data Eng):      Back from PTO. Onboarding meeting completed.
                          Energized. Sprint plan: ECR Tue, mesh Wed, 
                          monitoring Thu-Fri. RBAC read-only in data-analytics.
                          ECR images built and pushed. Wants to do rolling 
                          restart of StatefulSets NOW — waiting for approval.
                          "Horrified" about Atlas but constructive.

Nina Petrov (Order lead): Onboarded order-data-export to ArgoCD (Sat, 
                          self-initiated). Migrated to KMS bucket path.
                          Pushed back on CB PR #2 with business math — 
                          resolved. Gave self-service template UX requirement.
                          Added README per user's request.

Marcus Webb (secondary):  Investigating EC2 instances. Found Metabase, 
                          legacy Jenkins, unknown RDS. Changed Jenkins 
                          admin password. Found unauthorized logins from 
                          peered VPC. WAITING FOR GUIDANCE on next steps.
                          Asking: "Do I need to call Aisha?"

Priya Sharma (mid):       Linkerd staging dry-run complete. Drafting 
                          production upgrade runbook. Recommended excluding 
                          search namespace from EKS upgrade. Next: production 
                          Linkerd upgrade (tentative Tue 2 AM next week).
                          EKS runbook ready for user review.

Jake Torres (mid):        GP3 migration. Skipping data-analytics + search.

Alex Kim (junior):        Jenkins build killed during Incident 7 (apologized).
                          Growing. Good instincts on deploy freeze.

Lisa Park (SRE):          Canary AnalysisTemplate merged (PLAT-916 ✅).
                          Budget tracking: payment 86.0%, order 61.8%.
                          Warning: order-service one incident from freeze.
                          CB canary monitoring.

David Okafor (Payment):   Reconciliation adjustment with Martha (Finance).
                          Three-rounding-behavior question answered.
                          sha-c4d82e1 stable.

Derek Huang (Order):      CB PR merged. Staging tested. Production canary 
                          at 5% (started 2:43 PM). Asking about production 
                          deploy timing given budget. stockCache follow-up 
                          PR planned for this week.
                          Timeout anti-pattern acknowledged ("I clearly 
                          have a habit"). Post-approval process established.

Ryan Mitchell (Frontend): Confirmed services recovered after Incident 8.

Jenny Wu (Merch):         Looped into PLAT-920. Not active.

Martha Reeves (Finance):  $47.23 reconciliation handled with David. 
                          One-time adjustment. Future batches clean (HALF_EVEN).

Priya Kapoor (Payment):   BigDecimal fix stable. PAYMENT-847 filed 
                          (negative amount validation).
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
Incident 9 (SEV3 — TLS cert renewal):         Pending

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
```

### v2 Chaos Engine Observations

```
CONSEQUENCES ENCOUNTERED (cumulative):
  1. KMS permission gap (self-inflicted Wed, found Thu) — fixed, disclosed
  2. ServiceProfile timeout (approved PR, Derek added post-approval) — fixed
  3. ETL "failure" alert (containment caused expected daily alert) — silenced
  4. HALF_EVEN reconciliation (known Wed, hit finance Fri) — $47.23, handled
  5. analytics_reader no connection limit (known Wed, hit Mon) — Incident 8
  6. Wei RBAC gap (restricted Tom Sun, Wei caused incident Mon) — fixed
  7. Security automation broke cert renewal (RULE 2) — fixed
  8. Legacy Jenkins unauthorized access (discovered through audit) — ACTIVE

PATTERN: 
  User identifies risks correctly in investigation, documents them 
  as TODOs, but fixes them in priority order rather than exposure 
  order. Database-level and credential-level fixes are low-effort 
  but high-exposure — should be applied immediately when identified, 
  not deferred to ticket timelines.

RECOVERY QUALITY:
  Consistently strong. Self-identifies gaps, discloses honestly, 
  fixes thoroughly. Wei KMS disclosure, finance HALF_EVEN apology, 
  PDB label correction, RBAC expansion after Wei incident — all 
  demonstrate fast self-correction without defensiveness.

RULE 4 INSTANCES (wrong/incomplete hypothesis):
  - PDB labels (wrong on first attempt, corrected in 60 sec)
  - HALF_EVEN impact scope (assumed clean cutover, actually three 
    rounding behaviors — David caught the rollback window gap)
  - RBAC scope (restricted Tom, missed Wei — pattern, not individual)
  - Cert renewal: initial assumption was PLAT-919 misconfiguration 
    for ETL alert — actually correctly scoped (quick correction)

STRENGTHS DEMONSTRATED (v2 additions):
  ✓ Parallel execution under extreme time pressure (Incident 7)
  ✓ Counterintuitive tactical decisions (scale DOWN to save UP)
  ✓ Unfamiliar system navigation (ES cluster — RULE 10)
  ✓ Genuine stuck point resolution (AZ-bound PVC — RULE 12)
  ✓ Signal/noise separation under pressure (3 alerts → 1 incident, twice)
  ✓ Honest self-accountability without self-flagellation
  ✓ Executive communication under pressure (answered James while phone buzzing)
  ✓ Protecting people from themselves (Tom RBAC, Wei RBAC)
  ✓ Building trust through transparency (Wei "refreshing", Tom "not an ambush")
  ✓ Board-quality deliverables under deadline (Rachel's table)
  ✓ Committed deliverable met exactly on time (drift detection Mon 8 AM)
  ✓ Converting blockers through business context (Nina CB pushback)
  ✓ Process fixes over blame (Derek post-approval commits)
  ✓ VP-level framing ("wall and door at the same time")

GAPS (v2 additions):
  - analytics_reader CONNECTION LIMIT deferred (caused Incident 8)
  - HALF_EVEN not proactively flagged to finance (credibility hit)
  - RBAC restriction incomplete (Tom but not Wei)
  - PDB wrong labels on first attempt (Incident 7)
  - Cert renewal failure invisible for 23 days (alert gap, not user's 
    direct fault, but cert infrastructure was unowned)
  - AWS-level shadow infra (EC2, RDS) not visible to K8s drift scan
  - Legacy Jenkins admin/admin existed for 2 years undetected
```

---

## INCIDENT CATEGORY COVERAGE (9/100)

```
CATEGORY                          COVERED  TARGET   INCIDENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deploy/Release failures            2        8       #1, #6
Database (RDS/Mongo/Redis)         2        10      #3, #8
Kubernetes (pods/nodes/control)    1        12      #7
Networking (DNS/LB/mesh/TLS)       0        10
CI/CD pipeline failures            1        8       #5
Security incidents                 1*       10      Jenkins (emerging)
Observability failures             0        6
Capacity/scaling                   2        8       #3, #7
Cascading / multi-system           1        8       #8 (RDS→3 services)
Cost/billing                       0        4
DR / region failover               0        4
Certificate / secrets              1        4       #9
Cross-team / process               1        4       #4
Ambiguous / hard-to-diagnose       0        4
Simultaneous incidents             0        5
IaC/Terraform state issues         1        5       #2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL                              9*       100     (*security emerging)
```

---

## WHERE TO RESUME

```
SIMULATION TIME:  Monday 2:44 PM EST, Week 2
SIMULATION MODE:  v2 — CHAOS ENGINE ACTIVE (all 12 rules)
ON-CALL:          User (primary), Marcus (secondary — on Jenkins investigation)

ACTIVE SITUATIONS:

  1. POTENTIAL SECURITY INCIDENT (HIGHEST PRIORITY)
     Legacy Jenkins: admin/admin, 172.31.4.89 logins from peered VPC
     Password changed (attacker locked out). Instance untouched (evidence).
     Marcus waiting for guidance. Aisha NOT YET INFORMED.
     Unknowns: who is 172.31.4.89, what did they do, what did they access
     
  2. Derek CB canary (LOW — self-running)
     Production canary at 5%, ~10 minutes in, looking clean.
     Derek monitoring. Scaling to 25% at 15-min mark.
     
  3. Tom ECR restart (MEDIUM — needs answer)
     ECR images ready. Wants to restart StatefulSets in Legal-hold namespace.
     Needs nuanced approval: image swap is operational improvement within 
     containment scope, but creates pod churn in preserved namespace.
     Tom has read-only RBAC — needs platform team to execute restart.

IMMEDIATE ACTIONS NEEDED:
  1. Inform Aisha about Jenkins unauthorized access — NOW
  2. Direct Marcus on evidence preservation
  3. Investigate 172.31.4.89 and legacy-staging VPC
  4. Determine severity: security incident or benign?
  5. Answer Tom on ECR restart
  6. Monitor Derek's CB canary (passive)

UNRESOLVED CONSEQUENCES:
  ❓ Jenkins IAM role — what can that instance do in AWS?
  ❓ monthly-db-backup credentials — were they accessed?
  ❓ 172.31.4.89 identity and intent
  ❓ legacy-staging VPC — legitimate or forgotten?
  ❓ VPC peering route tables — production access scope?
  ❓ reporting-db-prod — unknown RDS, who created it, what's in it?
  ❓ Metabase — 7 users with no MFA, connected to unknown RDS
  ❓ Jenkins build discard aggressive limit
  ❓ Redis Terraform PR merged?
  ❓ RDS egress CIDR fragility

TODAY REMAINING:
  - Jenkins security investigation (ACTIVE)
  - Derek CB canary → production decision
  - Tom ECR restart approval
  - EKS runbook finalize (due today)
  - AWS audit scoping (James wants Wednesday)
  - IRSA migration planning (this week)
  - Cert renewal failure alert creation

THIS WEEK:
  - R-6a deliverables (Kafka secured by Fri Jan 26)
  - Image admission warn mode (by Fri Jan 26)
  - AWS-level infrastructure audit
  - Cert-manager IRSA migration
  - Search cluster: ILM, PriorityClass
  - Derek stockCache PR
  - Supervisory authority response (expected any day)
```

---

**END OF HANDOFF DOCUMENT — v5 (CHAOS ENGINE EDITION)**

**Ready to continue simulation. Chaos Engine active. All 12 rules in effect.**

**Marcus is waiting. Aisha doesn't know yet. 172.31.4.89 logged in four hours ago.**

**Go.**
