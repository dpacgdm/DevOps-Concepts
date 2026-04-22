# HANDOFF DOCUMENT — FAANG INTERVIEW SIMULATION
## Phase 9: Continuation State

---

## CANDIDATE PROFILE

**Name:** Deepak Geddam
**Location:** Hyderabad, India
**Current Role:** DevOps Engineer, TCS (June 2023 – Present)
**Previous Role:** Linux System Administrator, SAP ECS (May 2022 – June 2023)
**Education:** B.Tech, V.R. Siddhartha Engineering College (2017-2021)
**Certifications:** CKA, AZ-400, AZ-104, GCP ACE
**Target Level:** Staff SRE (L6)

**Candidate Context:**
- Working on a single client engagement (NovaMart) — mid-size e-commerce SaaS, ~50 microservices on shared EKS cluster, ~200K DAU
- ~2 years DevOps experience + 1 year Linux admin at SAP (17K+ server fleet)
- Honest about scope limitations — single client, single region, small team (1.5 people)
- Not inflating experience to L6. Interviewing to pressure-test fundamentals and find gaps
- The candidate is the AI assistant playing the role of the interviewee. The human user is the interviewer.

---

## RESUME (FINAL VERSION)

Realistic, non-inflated TCS resume. Key points:
- Built CI/CD with GitHub Actions (weekly → 3-4 deploys/week)
- Managed AWS infra (EC2, EKS, S3, VPC, IAM, RDS)
- Wrote Terraform modules for common infra patterns
- Migrated ~50 apps from on-prem/Beanstalk to EKS
- Built reusable Helm charts (stateless API, worker, cron)
- Automated ops tasks (patching, cert expiry, cleanup)
- Troubleshooting experience (resource limits, noisy neighbors, connection pool exhaustion)
- Ansible playbooks for hardening, Packer for AMI builds
- On-call rotation member, SEV2/SEV3 incident handling
- SAP: 17K Linux servers, 99.9% uptime SLA, MTTR reduction ~20%

---

## INTERVIEW STRUCTURE

```
┌──────┬───────────────────────────────┬─────────────┬───────────────────────────────┐
│Round │ Type                          │ Status      │ Score                         │
├──────┼───────────────────────────────┼─────────────┼───────────────────────────────┤
│  0   │ Phone Screen (Breadth)        │ COMPLETE    │ 99.5% (199/200) — STRONG PASS │
│  1   │ System Design                 │ COMPLETE    │ 96/100 — STRONG HIRE          │
│  2   │ Troubleshooting               │ COMPLETE    │ 98/100 — STRONG HIRE          │
│  3   │ Coding (Go + Python)          │ COMPLETE    │ 97/100 — STRONG HIRE          │
│  4   │ Deep Dive (NovaMart)          │ IN PROGRESS │ —                             │
│  5   │ K8s & Infrastructure          │ NOT STARTED │ —                             │
│  6   │ Behavioral + Leadership       │ NOT STARTED │ —                             │
│FINAL │ Hire Committee Debrief        │ NOT STARTED │ —                             │
└──────┴───────────────────────────────┴─────────────┴───────────────────────────────┘
```

---

## COMPLETED ROUNDS — DETAILED SUMMARY

### ROUND 0: Phone Screen (99.5% — STRONG PASS)

**Format:** 20 rapid-fire questions across all domains
**Result:** 19 answers at L5-L6, 1 minor deduction

**Level Distribution:** L5 × 6, L6 × 14

**The one deduction (Q19):** Intermittent 503 causes — gave 3 valid causes but missed sidecar readiness races in a service mesh context. Minor gap given candidate runs Linkerd.

**Key strengths noted:**
- Leads with mechanism, not buzzwords
- Naturally says "it depends" when appropriate
- Prioritizes fixes correctly (root cause > mitigation > workaround)
- Volunteers failure modes without prompting
- Concise under time pressure

---

### ROUND 1: System Design (96/100 — STRONG HIRE, L6 with L7 flashes)

**Problem:** Design multi-region architecture for $2B e-commerce platform. 200+ microservices, 50M MAU, EKS in us-east-1 only. After 47-min regional outage, CEO promised board full multi-region in 6 months.

**Initial Design:**
- Two regions (us-east-1 primary + us-west-2), not three — timeline-driven
- Three-tier service classification:
  - Tier 1 (active-active): catalog, search, auth — read-heavy, eventually consistent
  - Tier 2 (active-passive hot standby): checkout, orders, payments — strong consistency needs
  - Tier 3 (cold): internal tools, analytics — recover last
- DynamoDB Global Tables for Tier 1, RDS cross-region replica for Tier 2
- Global Accelerator + CloudFront origin failover for traffic routing
- ArgoCD ApplicationSets for GitOps across both clusters
- Karpenter in both regions, us-west-2 at 30% baseline
- Staggered CI/CD: deploy us-east-1 first, canary, then us-west-2
- Independent observability per region with Thanos global query layer
- Phased 6-month delivery plan (3 phases)
- Cost estimate: ~$113K/month ($1.35M/year) vs $50K/min downtime

**Follow-up 1 — Split-brain fencing:**
- Candidate initially proposed DynamoDB Global Tables distributed lock — WRONG (LWW conflict resolution)
- Admitted the flaw immediately, rebuilt from first principles
- Defense-in-depth fencing: RDS replica is physically read-only (natural fence), write-enabled flag with TTL/lease, security group revocation, Global Accelerator health checks
- Scored 92/100 — flaw existed but recovery was strong signal

**Follow-up 2 — Failover sequence minute-by-minute:**
- T+0:00-0:30: Detection (automated, three independent paths)
- T+0:30: Tier 1 already failed over (active-active, <1 min RTO)
- T+0:30-2:00: Human assessment + decision (with decision framework in runbook)
- T+2:00-5:00: Tier 2 failover execution (fence → RDS promote → verify → enable writes)
- Called out RDS promotion as bottleneck (1-3 min), recommended Aurora Global Database
- Traced full request path for a customer completing checkout post-failover
- Scored 96/100

**Follow-up 3 — Fail-back:**
- Identified three hard problems: broken replication chain, 6 hours of data in wrong region, false urgency
- "Don't merge databases — pick a winner and reconcile the delta"
- Payment processor as external source of truth for orphaned transactions
- Canary traffic shift for fail-back (10% → 25% → 50% → 75% → 100%)
- Told CEO: "Not today. Tomorrow during maintenance window. The site is fully operational — this is an optimization, not an emergency."
- "Fail-forward is an engineering problem, fail-back is an operations discipline problem"
- Scored 97/100 (L7)

**Follow-up 4 — CFO cost justification:**
- Reframed from insurance to revenue-generating asset
- West Coast latency improvement → conversion rate uplift (~$2M/year)
- Batch/analytics workloads on DR capacity with priority-class preemption
- Cross-region canary deployments for deployment safety
- Right-sized baseline from 30% to 15-20% with Karpenter warm pools
- Aurora Serverless v2 for replica cost reduction
- Revised cost: $86K/month ($1.03M/year), net cash-flow positive
- "Don't justify the insurance cost — eliminate the concept of insurance entirely"
- Scored 98/100 (L7)

**Follow-up 5 — DR drill program:**
- Progressive blast radius model (rocket analogy)
- Phase 0: Automated verification (config parity, synthetic transactions, replica health) — zero risk, daily
- Phase 1: Component-level drills (GA failover, RDS promotion dry run via replica-of-replica, fencing test)
- Phase 2: Integrated drills (1% canary → 10% → full failover during off-peak)
- Phase 3: Monthly automated partial, quarterly full, annual surprise (only after 6+ months confidence)
- Every drill has rollback faster than the drill itself
- Phase 0 as automated gate — don't drill if components aren't verified
- "If your DR drill is exciting, your DR program isn't mature"
- Scored 97/100 (L7)

**Gaps identified in Round 1:**
- Should challenge the premise before solving (was the outage really a regional failure or control plane issue?)
- Data residency/GDPR compliance for multi-region
- Secrets replication (Vault/Secrets Manager cross-region sync)

---

### ROUND 2: Troubleshooting (98/100 — STRONG HIRE, L6-L7)

**Scenario:** SEV1 at 2:47 AM — Payment service error rate >5%, p99 latency >8s, order service upstream timeouts, RabbitMQ queue at 47K messages. No deploy in 6 hours.

**Investigation path (4 steps to root cause, out of 8 allowed):**

1. **Impact assessment + hypothesis formation** — Read alerts before running commands. Deduced cascade direction from timestamps (latency → errors → timeouts → queue backup). Three ranked hypotheses: downstream dependency slow, resource constraints, RabbitMQ issue.

2. **Pod state + error codes** — Pods 5/5 healthy, no restarts. 504s dominant (gateway timeout = downstream slow). 503s secondary (circuit breaking). Eliminated pod health, confirmed hypothesis 1.

3. **Outbound latency by dependency** — Queried per-dependency p99. Postgres: 12ms → 9,800ms. All others flat. Isolated to database.

4. **pg_stat_activity + RDS metrics** — Found `ALTER TABLE payment_transactions ADD COLUMN refund_metadata jsonb DEFAULT '{}'::jsonb` running for 22h47m, holding AccessExclusiveLock. 133 queries waiting. Confirmed with pg_locks. Explained delayed symptom onset (connection pool gradual saturation → inflection point when pool exhausted).

**Root cause:** Senior backend engineer (Sarah Chen) ran ALTER TABLE directly against production via DBeaver at 4 AM. Worked in staging (2M rows, 4 seconds). Production: 847M rows. Lock held for 23 hours.

**Fix:** `pg_terminate_backend(4821)` — communicated before killing, anticipated thundering herd, monitored recovery.

**Prevention (5 layers):**
1. Revoke direct production DDL access (P0, this week)
2. Schema migrations through CI/CD only with large-table gates (P1)
3. `statement_timeout` + `lock_timeout` on database (P0, today)
4. Alerting on long-running queries and lock waits (P0, today)
5. Production-scale data in staging (P2, this quarter)

**Safe migration technique provided:** ADD COLUMN without default (instant) → SET DEFAULT separately → batch backfill with sleep intervals

**Blameless post-mortem note:** "The failure is systemic — fix the system that allowed this, not the person who found the gap."

**Gaps identified in Round 2:**
- Should have mentioned VACUUM ANALYZE after killing long-running ALTER (dead tuple bloat)
- Should have reviewed connection pool configuration (connectionTimeout should fail fast, not block 18 min)

---

### ROUND 3: Coding (97/100 — STRONG HIRE, L6 with L7 on follow-ups)

**Problem 1 (Go): `slo-checker` CLI tool**
- Connects to Prometheus, checks SLO compliance from config file
- Full `main.go` with: Config types + validation, Prometheus HTTP client with proper error handling, SLO calculation (error budget math), dual output (human-readable stdout + JSON stderr), CronJob manifest with `concurrencyPolicy: Forbid`, context timeout (2 min), exit code semantics (0=healthy, 1=violation, 2=config error)
- Edge cases handled: zero requests, errors > total, empty results, HTTP errors, JSON parse errors
- Scored 95/100

**Follow-up 1 — Window mismatch bug:**
- Config `window` field vs hardcoded `[30d]` in queries = silent correctness bug
- Option A (ship today): validate that queries contain the configured window string
- Option B (iterate to): template variable `{{.Window}}` in queries, resolved at runtime, with regex guard catching hardcoded ranges
- Correct prioritization: ship A, build B

**Problem 2 (Python): `drift-detector`**
- Compares Terraform state (S3) against live AWS, reports to Slack
- ABC checker pattern (SecurityGroupChecker, RDSInstanceChecker, S3BucketChecker)
- Security-aware severity classification (MISSING, MODIFIED, CRITICAL)
- Error isolation per resource (one failure doesn't abort the run)
- Slack formatting with actionable remediation steps
- Lambda handler works both deployed and locally
- No Slack spam on clean runs ("no news is good news")
- `dry_run` flag for testing
- Dependency injection for testability
- Scored 96/100

**Follow-up 2 — Secrets in state + staleness:**
- Problem 1 (secrets): Strip sensitive attributes before checkers see them, tight IAM scope, encrypted state
- Problem 2 (staleness): `terraform plan -refresh-only -json` pipeline as correct end-state, solves both problems simultaneously
- Architecture: EventBridge → CodeBuild (tf plan) → S3 (results) → Lambda (parse + Slack)
- Current tool positioned as Phase 1 (catches high-severity drift today), plan-based approach as Phase 2
- "Don't let perfect block useful" — with explicit acknowledgment of what's imperfect

**Follow-up 3 — Normalization false positives/negatives:**
Three specific examples:
1. IPv6/prefix lists not captured → false negative (miss real drift)
2. Protocol representation mismatch (tcp vs "6") → false positive
3. Self-referencing SG rules (`self=true` vs GroupId) → false negative
- Unknown-field sentinel pattern: log warning when encountering unrecognized fields so tool doesn't silently become incomplete when AWS adds new features

**Gaps identified in Round 3:**
- Go tool should use interface for Prometheus client (testability)
- `url.Parse` too permissive — need scheme + host validation
- No retry logic on AWS API calls (`botocore.config.Config(retries=...)`)

---

### ROUND 4: Deep Dive — IN PROGRESS

**Interviewer:** Morgan, Engineering Manager, 12 years experience
**Format:** Candidate presents, interviewer interrogates. 1 initial + up to 6 follow-ups.
**Exchanges used:** Initial presentation + 2 follow-ups completed, up to 4 follow-ups remaining

**Initial Presentation — NovaMart Architecture:**

Candidate walked through three major pieces:

**1. Migration to EKS + GitOps:**
- Inherited: 50 microservices split across EC2, Beanstalk, and hand-deployed K8s. No IaC, no consistent CI/CD, monitoring was CloudWatch dashboards nobody checked.
- Consolidated everything onto EKS (reduced operational surface area for 1.5-person team)
- Three reusable Helm chart templates (stateless API, worker, cron)
- ArgoCD for GitOps (chose over Flux for UI — app teams new to K8s)
- New EKS cluster via Terraform, migrated from hand-built eksctl cluster
- Terraform monorepo with environment-specific tfvars, remote state S3 + DynamoDB locking

**2. CI/CD Pipeline:**
- GitHub Actions: PR → lint/test → build → ECR → dev (auto) → staging (auto + integration tests) → prod (manual approval gate)
- Rollback via ArgoCD or git revert
- Image tags = git SHAs, never :latest
- Manual gate for prod — deliberate decision given team maturity

**3. Observability Stack:**
- Prometheus + Grafana + Loki on-cluster
- Defined SLOs BEFORE building dashboards (checkout 99.9%, search 99.5%, auth 99.95%)
- Alert on SLO burn rate, not symptoms — reduced alert noise from 40-50/week to 5-8/week
- Structured JSON logging standard, low-cardinality Loki labels
- Fought off requests to add user_id as Loki label

**What candidate would do differently:**
1. Should have added Linkerd (service mesh) earlier — two cascading failure incidents could have been prevented
2. Helm chart approach has scaling limits — should evolve to internal developer platform abstraction
3. Underinvested in runbooks — should write one per alert from day one

**Follow-up 1 — EKS migration de-risking (COMPLETED):**
- New cluster built 3 weeks before migration, all services running alongside old cluster
- Databases stayed in place (no data migration — only traffic shift)
- Weighted traffic shift over 4 hours (10% → 25% → 50% → 90% → 100%)
- Rollback = DNS weight change back to old cluster (<1 min)
- Old cluster kept alive 3 full business days after migration
- One issue found at 25%: hardcoded hostname referencing old cluster DNS — fixed in 10 min
- Client convinced via one-page risk matrix comparing in-place import vs clean migration
- Interviewer reaction: "databases don't move" was the most important de-risk decision

**Follow-up 2 — Manual approval gate effectiveness (COMPLETED):**
- First 3 months: gate caught 4 genuine issues (staging test failure, missing config, wrong memory limits, dependency ordering)
- Month 4-5: rubber-stamping — measured via approval timestamps (median approval time dropped from 15-25 min to under 2 min)
- Fixed by adding automated preconditions: staging health check, 15-min soak requirement, integration test verification — human approval became final confirmation, not primary gate
- Maturity trajectory: manual gate → automated preconditions + manual → full canary automation (future)
- "A control without measurement is just a ritual"
- Three metrics to track gate effectiveness: time-to-approve, rejection rate, post-deploy incident correlation

**Follow-up 2 was the last completed exchange. The interviewer has NOT yet responded to Follow-up 2. Round 4 resumes with the interviewer's reaction to Follow-up 2 and their next question (Follow-up 3).**

---

## REMAINING INTERVIEW

### Round 4 Continuation:
- Up to 4 more follow-ups remain
- Likely areas of interrogation based on the presentation:
  - "Why Linkerd instead of Istio?"
  - "What's your blast radius if Karpenter bugs out?" (Not applicable — NovaMart doesn't use Karpenter, used standard managed node groups)
  - "How do you know your SLOs are correct?"
  - "What's the single biggest risk in your architecture right now?"
  - "If you had to onboard 50 teams tomorrow, what breaks first?"
  - "Walk me through a deploy from git push to production traffic"
  - "Your CTO says move to GCP. What's your migration plan?"
  - Challenging "why not simpler" alternatives for any decision

### Round 5: K8s & Infrastructure Deep Dive
- Interviewer: Principal Engineer
- 5-6 questions with 2-3 depth levels each
- K8s internals, networking, storage, scheduling, security, AWS infrastructure, Linux fundamentals
- "Go deeper" pressure — relentless depth probing
- Will test "I don't know" honesty vs bullshitting

### Round 6: Behavioral + Leadership
- Interviewer: Director of Engineering
- Minimum 6 questions with 2-3 follow-ups each
- STAR+ format required
- Leadership principles: Ownership, Bias for Action, Disagree and Commit, Earn Trust, Dive Deep, Think Big, Deliver Results, Hire and Develop
- Will probe for: a failure YOU caused, a conflict you LOST, being WRONG publicly, saying NO to leadership, influencing without authority, inheriting a mess
- Behavioral answers must be specific with quantified results

### Hire Committee Debrief (After Round 6):
- Consolidated scorecard across all rounds
- Overall verdict with justification
- Level assessment, strengths, concerns
- "Would I fight for this candidate?"
- Top 5 things to sharpen
- Company-specific calibration (Google/Amazon/Meta/Netflix)

---

## GRADING SYSTEM

### Per-Round Grading Weights:
```
Technical Accuracy:      25%
Depth of Understanding:  20%
Structure / Clarity:     15%
Production Awareness:    15%
Tradeoff Articulation:   10%
Communication:           10%
Pressure Handling:        5%
```

### Verdict Scale:
```
Strong Hire:  ≥ 85
Hire:         70-84
Lean Hire:    60-69
Lean No Hire: 50-59
No Hire:      < 50

TARGET: Strong Hire (≥85) in ALL rounds
```

### Level Calibration:
```
L4 (Mid):      Correct textbook answer, no failure modes, single-solution
L5 (Senior):   Correct + tradeoffs + failure modes + 2-3 alternatives
L6 (Staff):    Above + business impact + second-order effects + migration paths + cross-team
L7 (Principal): Above + industry perspective + challenges premise + 3-year evolution + teaches while answering
```

After every answer: tag with level, show what next level looks like if below L6.

---

## CANDIDATE PERFORMANCE PATTERNS (Across Rounds 0-3)

### Consistent Strengths:
- Hypothesis-first investigation (never shotguns)
- Reads existing information before generating new data
- Communicates at the right level for the audience (exec vs engineer)
- Prevention thinking is automatic, not prompted
- Connects technical root cause to organizational fixes
- Admits mistakes cleanly and rebuilds from first principles
- Separates "ship today" from "correct long-term" and addresses both
- Security awareness is automatic
- Error handling is comprehensive
- Gets BETTER under pressure during follow-ups

### Consistent Level:
- L6 with L7 flashes
- L7 moments come during follow-ups when pushed past initial answer
- Technical execution: L6-L7
- Communication: L7
- Organizational awareness: L7
- Operational precision: L6 (minor gaps in post-incident hardening details)

### Known Gaps (Accumulated):
1. Challenge the premise before solving (Round 1)
2. Data residency/GDPR compliance for multi-region (Round 1)
3. Secrets replication across regions (Round 1)
4. Sidecar readiness races in service mesh context (Round 0)
5. Post-kill VACUUM ANALYZE as standard DB incident follow-up (Round 2)
6. Connection pool timeout review after pool saturation incidents (Round 2)
7. Interface-based design for testability in Go (Round 3)
8. Input validation beyond parsing (Round 3)
9. AWS SDK retry configuration for batch tools (Round 3)

### One Redo Available:
- Candidate has not used their single redo across all 7 rounds
- Available strategically for any round

---

## RULES OF ENGAGEMENT (For Continuing Interviewer)

1. No hints mid-answer
2. "I don't know" respected, bullshitting penalized
3. Push back even on correct answers ("Are you sure?")
4. Change constraints mid-question
5. Exchange limits are hard (per round structure)
6. Follow-ups are not optional — dodging = deduction
7. Behavioral answers must be specific with quantified results
8. Tell candidate when round starts and ends — clean boundaries

---

## ANSWER FRAMEWORKS (Available to Candidate)

1. **System Design:** Requirements → Math → High-Level → Deep Dive → Failure Modes → Tradeoffs
2. **Troubleshooting:** Impact Assessment → Hypothesis → Systematic Investigation → Confirm Root Cause → Fix + Verify → Prevention
3. **Deep Dive (STAR+):** Situation → Task → Action → Result → Retrospective
4. **Coding:** Clarify → State Approach → Code Happy Path → Error Handling → Test/Verify → Production Hardening

---

## RESUME TO CONTINUE

**Start Round 4 Follow-up 3.** The interviewer (Morgan, Engineering Manager) has just heard the candidate's answer about the manual approval gate evolving from rubber-stamp to automated preconditions. The interviewer should react to Follow-up 2, then ask Follow-up 3 continuing the deep dive on NovaMart architecture decisions. Up to 4 follow-up exchanges remain in Round 4.
