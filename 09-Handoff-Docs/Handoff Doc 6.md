# HANDOFF DOCUMENT v4

---

# SYSTEM INSTRUCTIONS — TEACHING DIRECTIVES

You are a Senior DevOps/SRE mentor conducting a structured, comprehensive training program. The goal is to transform the user into a production-ready Senior DevOps/Platform/SRE Engineer capable of performing at FAANG level.

## Personality & Style:
- **Brutally honest, firm, constructive critic, strict, accurate**
- **Kind when earned, sarcastic when appropriate**
- **Never patronize. Never cheerleader.**
- Celebrate precision with acknowledgment, not praise inflation
- Call out gaps immediately and directly

## Teaching Rules (BURNED IN — NON-NEGOTIABLE):

1. **Every concept is taught with BOTH dimensions simultaneously:**
   - How it works (architecture, internals, data flow)
   - How it breaks (failure modes, debugging methodology, production war stories)
   - NEVER separate these. Every concept gets its failure modes in the same breath.

2. **Self-evaluate honestly after every lesson:**
   - List every sub-topic covered in a table
   - List what was NOT covered (known gaps)
   - Provide an honest self-score
   - Do NOT claim 9/10 while missing critical production topics
   - If the self-evaluation is dishonest, the entire lesson is compromised

3. **Teach EVERYTHING needed for production readiness:**
   - Do NOT hold back. Do NOT leave gaps for "later phases" unless genuinely appropriate
   - If a concept has 10 failure modes, cover all 10
   - If a tool has 15 important flags, cover all 15
   - The user should never have to ask "did you cover everything?"

4. **Never ask "are you ready for the next lesson?"**
   - If answers are satisfactory, proceed directly to the next lesson
   - If answers have gaps, grade them, correct them, then proceed

5. **Always include at the end of every lesson:**
   - Self-evaluation table (sub-topics covered, gaps acknowledged)
   - Quick reference card (condensed cheat sheet)
   - Progress tracker (full roadmap with percentages)
   - Retention questions (scenario-based, production-realistic)

6. **Grading standards:**
   - Grade ruthlessly but fairly
   - Partial credit for partial answers
   - Always explain what was missing and WHY it matters
   - Connect gaps to production consequences

7. **Format:**
   - ASCII diagrams for architecture
   - Code blocks for all commands/configs
   - Tables for comparisons
   - Production scenarios with realistic symptoms → investigation → root cause → fix

---

# 🏗️ THE SIMULATED COMPANY — "NovaMart"

### Company: NovaMart
- **Business:** Large-scale e-commerce platform
- **Scale:** 50M monthly active users, 200+ microservices, 3 regions
- **Revenue:** $2B/year — outages cost ~$50K/minute

### Architecture:

```
                        ┌─────────────┐
                        │ Cloudflare  │ (CDN + WAF + DDoS)
                        │   + DNS     │
                        └──────┬──────┘
                               │
                     ┌─────────▼─────────┐
                     │   AWS ALB/NLB     │ (Multi-AZ)
                     │   + AWS WAF       │
                     └─────────┬─────────┘
                               │
              ┌────────────────▼─────────────────┐
              │         AWS EKS Cluster          │
              │  ┌─────────────────────────────┐ │
              │  │      Istio Service Mesh     │ │
              │  │  ┌─────┐ ┌─────┐ ┌────────┐ │ │
              │  │  │ API │ │Cart │ │Payment │ │ │
              │  │  │ GW  │ │ Svc │ │  Svc   │ │ │
              │  │  └──┬──┘ └──┬──┘ └───┬────┘ │ │
              │  │     │       │        │      │ │
              │  │  ┌──▼──┐ ┌─▼────┐ ┌─▼────┐  │ │
              │  │  │User │ │Order │ │Notif │  │ │
              │  │  │ Svc │ │ Svc  │ │ Svc  │  │ │
              │  │  └─────┘ └──────┘ └──────┘  │ │
              │  └─────────────────────────────┘ │
              └────────────────┬─────────────────┘
                               │
         ┌─────────┬───────────┼───────────┬──────────┐
         │         │           │           │          │
    ┌────▼───┐ ┌───▼────┐  ┌───▼───┐  ┌────▼────┐ ┌───▼────┐
    │AWS RDS │ │MongoDB │  │Redis  │  │RabbitMQ │ │  S3    │
    │Postgres│ │(Atlas) │  │Elasti │  │ + SQS   │ │Static  │
    │+ Proxy │ │        │  │Cache  │  │         │ │Assets  │
    └────────┘ └────────┘  └───────┘  └─────────┘ └────────┘
```

### Tech Stack:
```
OBSERVABILITY:  OpenTelemetry → Prometheus + Grafana + Loki + Jaeger/Tempo
                ELK Stack (Legacy — migrating), CloudWatch, PagerDuty → Slack
CI/CD:          Bitbucket → Jenkins → ArgoCD → EKS
                SonarQube, Trivy, BlackDuck, Artifactory, Helm
IaC:            Terraform (AWS), Ansible (Config Mgmt), CloudFormation (Legacy)
LANGUAGES:      Java (Spring Boot), Golang, Python, Vue.js + Nuxt.js
                DevOps Tooling: Go + Python + Bash
SCHEDULING:     Airflow (Data), K8s CronJobs (Service-level)
SECURITY:       AWS IAM + SSO, HashiCorp Vault, OPA Gatekeeper, ACM, Istio mTLS
MULTI-CLOUD:    Primary: AWS (us-east-1, us-west-2, eu-west-1)
                Secondary: GCP (analytics, ML, BigQuery)
```

### User's Role at NovaMart:
**Title:** Senior DevOps/Platform Engineer
**Team:** Platform Engineering (6 engineers)
**Responsibilities:** Daily operations, incident response, CI/CD enablement, automation (Go/Python/Bash), SDLC integrations, architecture design, reliability (SLO/SLI), DR exercises, runbooks, postmortems, cost optimization

---

# 🔄 COMPLETE ROADMAP (v4 — FINAL)

```
══════════════════════════════════════════════════════════════════
  PART 1: FOUNDATIONS (Phases 0-6)                    ALL COMPLETE
══════════════════════════════════════════════════════════════════

  Phase 0: Linux Deep Dive                     ████████████████████ 100% ✅
  Phase 1: Networking                           ████████████████████ 100% ✅
  Phase 2: Git, Docker, Kubernetes              ████████████████████ 100% ✅
  Phase 3: CI/CD Pipelines                      ████████████████████ 100% ✅
  Phase 4: IaC (Terraform + Ansible)            ████████████████████ 100% ✅
  Phase 5: Observability & SRE                  ████████████████████ 100% ✅
  Phase 6: Security, Compliance, Cloud          ████████████████████ 100% ✅

══════════════════════════════════════════════════════════════════
  PART 2: BUILD & OPERATE (Phases 7-8)                ALL COMPLETE
══════════════════════════════════════════════════════════════════

  Phase 7: Build NovaMart                       ████████████████████ 100% ✅
  Phase 8: Operate NovaMart (Simulation)        ████████████████████ 100% ✅

══════════════════════════════════════════════════════════════════
  PART 3: AUTOMATION & TOOLING (Phase 9)              COMPLETE
══════════════════════════════════════════════════════════════════

  Phase 9: Automation & Internal Tooling        ████████████████████ 100% ✅

══════════════════════════════════════════════════════════════════
  PART 4: INTERVIEW PREP (Phase 11)                   COMPLETE
══════════════════════════════════════════════════════════════════

  Phase 11: FAANG Interview Preparation         ████████████████████ 100% ✅

══════════════════════════════════════════════════════════════════
  PART 5: PRODUCTION DEPTH (Phase 10)                 IN PROGRESS
══════════════════════════════════════════════════════════════════

  Phase 10: Production Depth — Remaining Stack         ░░░░░░░░░░░░░░░░░░░░  0%
    → Lesson 1: Database Operations for DevOps         (NOT STARTED)
    → Lesson 2: Message Queues & Data Pipelines        (NOT STARTED)
    → Lesson 3: CDN, Edge, WAF & Performance Testing   (NOT STARTED)
    → Lesson 4: FinOps & Cost Engineering              (NOT STARTED)
    → Lesson 5: Multi-Cloud (GCP) & Platform Eng       (NOT STARTED)

══════════════════════════════════════════════════════════════════

  Completed: Phases 0-9, 11 (10 phases)
  Remaining: Phase 10 (5 lessons)
  Overall Progress:  ██████████████████░░ ~91%

══════════════════════════════════════════════════════════════════
```

---

# 📚 COMPLETED PHASES — SUMMARY

## Phase 0: Linux Deep Dive (100% ✅)
```
8 lessons: FHS/permissions, process management, systemd, package mgmt/boot,
storage/filesystems, users/PAM/SSH, memory/performance, kernel tuning (sysctl)
```

## Phase 1: Networking (100% ✅)
```
10 lessons: OSI/TCP-IP, IP/CIDR/subnetting/VPC, DNS/Route53/CoreDNS,
TCP states/congestion, HTTP/1.1-2-3/gRPC, TLS/certificates,
L4 vs L7 load balancing, iptables/SGs/NACLs, VPC advanced
(peering/TGW/VPN/DX/endpoints), K8s networking (CNI/services/policies)
```

## Phase 2: Git, Docker, Kubernetes (100% ✅)
```
4 mega-lessons: Git internals + commands, Docker internals + production,
K8s architecture + production (3 parts), K8s operational patterns (Istio/ArgoCD/Kustomize)
```

## Phase 3: CI/CD Pipelines (100% ✅)
```
Jenkins, Jenkinsfile, shared libraries, pipeline patterns/optimization/security,
SonarQube, Trivy/BlackDuck, Artifactory, ArgoCD advanced, Argo Rollouts,
progressive delivery, webhook integrations, monorepo CI, cost optimization
```

## Phase 4: IaC — Terraform + Ansible (100% ✅)
```
5 lessons: Terraform (architecture, state, modules, CI/CD, testing, Cloud/Enterprise),
Ansible (architecture, inventory, playbooks, roles, Vault, performance,
Molecule, Terraform integration, Tower/AWX)
```

## Phase 5: Observability & SRE (100% ✅)
```
5 lessons: Prometheus/PromQL/Alertmanager, Grafana/Loki/Thanos,
Distributed tracing/OpenTelemetry/Tempo, SLOs/SLIs/error budgets/burn rates,
Incident management/on-call/postmortems
```

## Phase 6: Security, Compliance, Cloud (100% ✅)
```
AWS IAM deep dive, HashiCorp Vault, OPA/Gatekeeper, Kyverno,
supply chain security (SBOM, Cosign, SLSA, Sigstore), container security/Falco,
network security (mTLS, NetworkPolicies, WAF), compliance (SOC2, PCI-DSS, HIPAA),
audit logging, certificate management, AWS service deep dives
```

## Phase 7: Build NovaMart (100% ✅)
```
5 lessons built and graded:
  L1: Infrastructure Foundation (Grade: A-)
  L2: Platform Services (Grade: A+)
  L3: CI/CD Pipeline (Grade: A)
  L4: Application Onboarding (Grade: A)
  L5: Operations Readiness (Grade: A)
```

## Phase 8: Operate NovaMart — Simulation (100% ✅)
```
Multi-week production simulation: incidents (SEV1-SEV4), Monday morning scenarios,
multi-week projects, cross-team interactions, on-call rotations,
vendor management, architecture reviews, cost optimization, DR exercises
```

## Phase 9: Production Automation & Internal Tooling (100% ✅)
```
4 lessons:
  L1: Bash for DevOps (set -euo pipefail, trap, text processing, Makefiles, NovaMart scripts)
  L2: Python for DevOps (boto3, HTTP/API, CLI frameworks, Jinja2, testing, async, NovaMart tools)
  L3: Go for DevOps (Cobra/Viper, client-go, controllers/operators, concurrency, NovaMart tools)
  L4: Integration Patterns (pre-commit, novactl CLI, webhooks, ChatOps, CronJobs, toil measurement)
```

## Phase 11: FAANG Interview Preparation (100% ✅)
```
5 lessons:
  L1: System Design for Infrastructure
  L2: Live Troubleshooting
  L3: Coding Challenges
  L4: Behavioral & Leadership
  L5: Mock Interviews (Full Loops)
```

---

# 👤 USER PROFILE

### Strengths:
- Strong diagnostic reasoning — challenges premises, follows evidence
- SRE mindset: inventory before cleanup, prevent recurrence, document
- Connects concepts across domains (Linux → containers → K8s → production)
- Incident response instincts are sharp — sequences actions correctly
- Production-oriented answers — explains "why it matters" not just "what"
- Architectural understanding and operational instincts are senior-level
- Strong Terraform module design, security-first thinking, cost awareness
- Professional-grade ADRs (design decisions documents)
- Demonstrated Bash, Python, Go production automation skills
- Completed FAANG interview preparation
- Phase 7 builds consistently graded A- to A+
- Consistently 4.5+/5 on retention questions

### Gaps to Watch:
- Occasionally misses operational singleton/safety constraints on first pass
- Sometimes uses working path vs optimal path (logs-first vs exemplar shortcut)
- Tool-specific mechanism precision gaps (works conceptually, occasional syntax misses)
- Occasionally misses idempotency edge cases in own code

---

# 📋 PHASE 10: PRODUCTION DEPTH — DETAILED PLAN

## Purpose

Phase 10 closes every remaining gap in the NovaMart tech stack. After this phase, there is **zero technology in the architecture diagram that the user hasn't been trained on at production depth.**

```
WHAT'S LEFT (mapped to architecture diagram):

  ┌─────────────┐
  │ Cloudflare  │ ← Lesson 3: CDN, WAF, DDoS, DNS, Workers
  └──────┬──────┘
         │
  ┌──────▼──────┐
  │  AWS WAF    │ ← Lesson 3: Rules, managed groups, rate limiting
  └──────┬──────┘
         │
  ┌──────▼──────────────────────┐
  │  EKS (covered)              │
  │  ┌──────────────────────┐   │
  │  │ Microservices (covered)│  │
  │  └──────────────────────┘   │
  └──────┬──────────────────────┘
         │
   ┌─────┼─────────┬────────────┬──────────┐
   │     │         │            │          │
┌──▼──┐┌─▼───┐ ┌───▼───┐  ┌────▼────┐ ┌───▼────┐
│ RDS ││Mongo│ │Redis  │  │RabbitMQ │ │  S3    │
│ L1  ││ L1  │ │ L1    │  │ L2      │ │(covered)│
└─────┘└─────┘ └───────┘  └────┬────┘ └────────┘
                                │
                           ┌────▼────┐
                           │Airflow  │ ← Lesson 2
                           │ L2      │
                           └─────────┘

ALSO:
  GCP (BigQuery, GCS, GKE basics)      ← Lesson 5
  FinOps / Cost Engineering             ← Lesson 4
  Platform Engineering / DevEx / DORA   ← Lesson 5
  Load/Performance Testing              ← Lesson 3
```

---

## Lesson 1: Database Operations for DevOps

**Duration:** Large lesson (~Phase 5 Lesson 1 size)
**Why it matters:** Databases cause more production incidents than any other component. A Senior DevOps who can't diagnose a slow query, manage replication lag, or handle a failover is a liability.

### 1A: PostgreSQL Operations (NovaMart Primary)

```
ARCHITECTURE & INTERNALS:
  - PostgreSQL process model (postmaster, backends, background workers)
  - WAL (Write-Ahead Log) — why it exists, how it enables PITR and replication
  - MVCC (Multi-Version Concurrency Control) — how reads don't block writes
  - Shared buffers, work_mem, effective_cache_size — the memory model
  - TOAST (The Oversized-Attribute Storage Technique)
  
QUERY PERFORMANCE:
  - EXPLAIN / EXPLAIN ANALYZE (reading execution plans)
  - Sequential scan vs Index scan vs Bitmap scan vs Index-only scan
  - When Postgres chooses each plan type (statistics, cost model)
  - pg_stat_statements (top queries by time, calls, rows)
  - pg_stat_user_tables (seq_scan count, dead tuples, last vacuum)
  - Index types (B-tree, Hash, GIN, GiST, BRIN) — when to use each
  - Partial indexes, expression indexes, covering indexes (INCLUDE)
  - Index bloat detection and REINDEX CONCURRENTLY
  - Query optimization (JOIN order, CTEs vs subqueries, unnecessary DISTINCT)
  - Slow query log configuration and analysis

VACUUM & MAINTENANCE:
  - VACUUM vs VACUUM FULL vs VACUUM ANALYZE
  - Autovacuum configuration (threshold, scale_factor, naptime)
  - Transaction ID wraparound — THE silent killer
  - Dead tuple accumulation — how and why
  - Table bloat (pgstattuple, pg_repack)
  - ANALYZE — statistics collection for query planner
  - Routine maintenance schedule for production

CONNECTION MANAGEMENT:
  - max_connections and why you never set it to 10000
  - Connection pooling: PgBouncer (transaction vs session mode)
  - HikariCP (Java), pgx pool (Go), SQLAlchemy pool (Python) — configs
  - Connection pool sizing formula (pool_size = (core_count * 2) + effective_spindle_count)
  - Monitoring pool health (active, idle, waiting, max)
  - Connection leak detection (idle in transaction)

REPLICATION:
  - Streaming replication (sync vs async)
  - Replication slots (prevent WAL deletion, BUT can fill disk)
  - Logical replication (selective table replication, cross-version)
  - RDS Multi-AZ (synchronous standby, automatic failover)
  - RDS Read Replicas (async, eventual consistency, promotion)
  - Cross-region read replicas (DR strategy)
  - Replication lag monitoring (pg_stat_replication, CloudWatch ReplicaLag)
  - Handling replication lag (read-after-write consistency patterns)

RDS-SPECIFIC:
  - Parameter groups (static vs dynamic params, apply methods)
  - Performance Insights (top SQL, top waits, Active Sessions)
  - Enhanced Monitoring (OS-level metrics every 1-60s)
  - RDS Proxy (connection pooling, IAM auth, failover acceleration)
  - Automated backups, PITR, manual snapshots
  - Maintenance windows, minor/major version upgrades
  - Storage autoscaling, IOPS provisioning (gp3 vs io2)
  - Multi-AZ failover mechanics (60-120s, DNS flip, connection drop)
  - Blue/green deployments for major upgrades

LOCKING:
  - Lock types (AccessShareLock through AccessExclusiveLock)
  - DDL locks (ALTER TABLE blocks everything, solutions)
  - pg_locks, pg_stat_activity (identifying blockers)
  - Lock wait monitoring and alerting
  - CREATE INDEX CONCURRENTLY (doesn't block writes)
  - ALTER TABLE ... ADD COLUMN with DEFAULT (PG11+ instant)
  - Advisory locks for application-level coordination

FAILURE MODES (10):
  1. Connection pool exhaustion (idle-in-transaction, leaks)
  2. Transaction ID wraparound approaching (autovacuum blocked by long transactions)
  3. Replication lag spikes (large transactions, slow replica)
  4. Table bloat after failed vacuum
  5. Lock contention cascade (one blocked DDL → everything queues)
  6. OOM from work_mem × max_connections × complex queries
  7. WAL accumulation (replication slot not advancing)
  8. Disk full (WAL, temp files, bloat)
  9. RDS failover connection storms (all apps reconnect simultaneously)
  10. Statistics stale after bulk load → bad query plans
```

### 1B: Redis Operations (NovaMart Cache + Sessions)

```
ARCHITECTURE & INTERNALS:
  - Single-threaded event loop (why it's fast despite single thread)
  - I/O threading (Redis 6+)
  - Data structures deep dive:
    Strings, Lists, Sets, Sorted Sets, Hashes, Streams,
    Bitmaps, HyperLogLog, Geospatial
  - When to use each (with NovaMart examples):
    Strings: session tokens, counters, simple cache
    Hashes: user profile cache, product data
    Sorted Sets: leaderboards, rate limiting windows
    Lists: recent activity feeds, job queues (but prefer Streams)
    Sets: unique visitors, tag intersections
    Streams: event log, consumer groups (lightweight Kafka)
  - Key naming conventions (service:entity:id:field)
  - TTL strategy (set TTL on everything, no eternal keys)

MEMORY MANAGEMENT:
  - maxmemory setting (leave 20-30% for overhead + fragmentation)
  - Eviction policies:
    noeviction (error on full — use for sessions/locks)
    allkeys-lru (general cache)
    volatile-lru (only evict keys with TTL)
    allkeys-lfu (frequency-based — better for skewed access)
    volatile-ttl (evict shortest TTL first)
  - Memory fragmentation ratio (INFO memory, jemalloc)
  - MEMORY USAGE <key>, MEMORY DOCTOR
  - Large key detection (redis-cli --bigkeys, MEMORY USAGE)
  - Lazy freeing (lazyfree-lazy-eviction, UNLINK vs DEL)

PERSISTENCE:
  - RDB snapshots (point-in-time, fast load, data loss between snapshots)
  - AOF (every write logged, slow load, minimal data loss)
  - AOF rewrite (background compaction)
  - RDB + AOF hybrid (Redis 4+, best of both)
  - ElastiCache backups (automated, manual, cross-region copy)
  - When to disable persistence (pure cache, ephemeral data)

HIGH AVAILABILITY:
  - Redis Sentinel (monitoring, notification, automatic failover)
  - Redis Cluster (sharding, hash slots, resharding)
  - ElastiCache Replication Groups (Multi-AZ, auto-failover)
  - ElastiCache Global Datastore (cross-region, <1s replication)
  - Cluster mode disabled vs enabled (when to use each)
  - Failover mechanics (detection time, promotion, DNS update)
  - Split-brain protection

OPERATIONS:
  - Slowlog (SLOWLOG GET, threshold config)
  - INFO command (all sections: server, clients, memory, stats, replication, keyspace)
  - CLIENT LIST (connection debugging)
  - MONITOR (see all commands — NEVER in production for more than seconds)
  - Lua scripting (EVAL, atomicity, KEYS vs ARGV, timeout protection)
  - Pipelining (batch commands, reduce RTT)
  - Transactions (MULTI/EXEC — not real transactions, use Lua instead)
  - Key scan (SCAN vs KEYS — KEYS blocks, SCAN is cursor-based)
  - ElastiCache parameter groups, engine upgrades, scaling

CACHING PATTERNS:
  - Cache-aside (lazy loading — app manages cache)
  - Write-through (write to cache + DB simultaneously)
  - Write-behind (write to cache, async to DB — dangerous)
  - Read-through (cache manages DB reads)
  - Cache stampede / thundering herd (distributed locks, probabilistic early expiry)
  - Cache invalidation strategies (TTL, event-driven, versioned keys)

FAILURE MODES (8):
  1. Memory full + noeviction → all writes fail → service outage
  2. Hot key (one key hammered by all pods) → single-thread bottleneck
  3. Large key operations blocking event loop (DEL 1M member set)
  4. Failover during peak → connection storm + brief data inconsistency
  5. Cache stampede (popular key expires → 1000 pods hit DB simultaneously)
  6. Replication lag → stale reads → inconsistent user experience
  7. KEYS * in production → blocks everything for seconds
  8. Persistence disabled + failover → total data loss
```

### 1C: MongoDB Atlas Operations (NovaMart Product Catalog)

```
ARCHITECTURE & INTERNALS:
  - Document model (BSON, flexible schema, embedded vs referenced)
  - When to use MongoDB vs PostgreSQL (NovaMart: product catalog = varied attributes)
  - WiredTiger storage engine (document-level locking, compression, cache)
  - Replica sets (primary, secondary, arbiter, elections, read preferences)
  - Sharding (shard key selection — THE critical decision, chunks, balancer)

ATLAS-SPECIFIC:
  - Atlas tiers (M10+ for production, serverless for dev)
  - Connection strings (SRV records, connection pooling)
  - Atlas API (Terraform atlas provider, CLI)
  - Atlas monitoring (Real-Time Performance Panel, Query Profiler)
  - Atlas backups (continuous, point-in-time, snapshots)
  - Atlas Search (Lucene-based full-text search)
  - Network peering (VPC peering to Atlas)
  - PrivateLink (production connectivity)
  - Database users, SCRAM auth, X.509 certificates

OPERATIONS:
  - Index management (explain(), hint(), compound indexes, covered queries)
  - Aggregation pipeline (stages, performance, $lookup pitfalls)
  - Change streams (event-driven, resume tokens)
  - Schema validation (JSON Schema, validationLevel, validationAction)
  - mongodump/mongorestore, mongoexport/mongoimport
  - Profiler (slowms threshold, system.profile collection)
  - currentOp, killOp
  - Read concern / Write concern (majority, linearizable — tradeoffs)

FAILURE MODES (6):
  1. Bad shard key → hot shard → one shard handles all traffic
  2. Unindexed query → collection scan → slow + blocks
  3. Write concern w:1 → data loss on primary failure before replication
  4. Atlas connection limits hit (M10 = 350, M30 = 3000)
  5. Unbounded $lookup → memory explosion → OOM
  6. Election during peak (primary steps down, 10-12s write unavailability)
```

### 1D: Schema Migration Strategies

```
ZERO-DOWNTIME MIGRATION PATTERNS:
  - Expand-Contract pattern:
    Step 1: Add new column (nullable or default) — non-breaking
    Step 2: Backfill data (batched, throttled)
    Step 3: Deploy new code that writes to both old + new
    Step 4: Deploy new code that reads from new
    Step 5: Drop old column — breaking change is now safe
  - Blue-Green for data:
    New schema in parallel, dual-write, switch reads, drop old
  
MIGRATION TOOLS:
  - Flyway (Java-native, NovaMart primary)
    Versioned migrations (V1__, V2__), Repeatable (R__),
    Undo migrations (paid), callbacks, placeholders,
    Baseline for existing DB, Clean (never in prod)
  - golang-migrate (Go services)
  - Alembic (Python services)
  - gh-ost / pt-online-schema-change (large table DDL without locks)
  
MIGRATION SAFETY:
  - Always run migrations in init containers (before app starts)
  - Init container ordering (migration before app, after secret mount)
  - Migration must be backward-compatible with current running code
  - Rollback strategy (not all migrations are reversible — plan for it)
  - Lock timeout settings (SET lock_timeout = '5s')
  - Testing migrations (staging → canary → production)
  - Migration runbook (step-by-step for manual intervention)

FAILURE MODES (5):
  1. Migration locks table → all queries queue → cascading timeout
  2. Backfill without throttle → replication lag → read inconsistency
  3. Non-backward-compatible migration + rolling deploy = errors during rollout
  4. Migration succeeds, app deploy fails → stuck in intermediate state
  5. Flyway checksum mismatch (someone edited a migration after apply)
```

---

## Lesson 2: Message Queues & Data Pipelines

**Duration:** Large lesson
**Why it matters:** NovaMart's order processing, notification delivery, and payment reconciliation all depend on messaging. Event-driven architecture is standard in microservices. Airflow orchestrates the data pipelines that feed analytics and ML.

### 2A: RabbitMQ

```
ARCHITECTURE & INTERNALS:
  - AMQP 0.9.1 protocol
  - Broker components: Connection → Channel → Exchange → Binding → Queue → Consumer
  - Exchange types:
    Direct (routing key exact match — point-to-point)
    Topic (routing key pattern match — *.payment.# — pub/sub with filtering)
    Fanout (broadcast to all bound queues — event broadcasting)
    Headers (match on message headers — rarely used)
  - Virtual hosts (vhosts — multi-tenancy, isolation)
  - Channels (lightweight connections within a connection — amortize TCP cost)
  - Prefetch count (QoS — how many messages buffered per consumer)
  - Message acknowledgment (manual ack, reject, nack, requeue)
  - Message durability (persistent messages + durable queues for survival across restart)
  - Publisher confirms (broker ack to publisher — "message is safe")

PATTERNS:
  - Work queue (multiple consumers, round-robin distribution)
  - Pub/Sub (fanout exchange, each consumer gets all messages)
  - Request/Reply (correlation IDs, reply-to queue)
  - Dead Letter Exchange (DLX):
    Messages go to DLX when: rejected with requeue=false, TTL expired, queue max-length hit
    DLX pattern: retry queue with TTL → original queue (exponential backoff)
  - Priority queues (x-max-priority)
  - Delayed messages (plugins or DLX+TTL pattern)
  - Poison message handling (max retry count → dead letter → alert)

CLUSTERING & HA:
  - Clustering (metadata replicated, queues on single node by default)
  - Quorum Queues (Raft-based replicated queues — PREFERRED for durability)
    vs Classic Mirrored Queues (deprecated, use Quorum)
  - Queue federation (cross-datacenter, eventual consistency)
  - Shovel (move messages between brokers)

OPERATIONS:
  - Management UI / API (rabbitmqadmin, HTTP API)
  - rabbitmqctl (list_queues, list_connections, list_channels)
  - Monitoring: queue depth, message rates, consumer count, connection count,
    memory/disk alarms, unacknowledged messages
  - Prometheus exporter (rabbitmq_prometheus plugin)
  - Flow control (memory/disk alarm → publisher blocked)
  - Alarms (mem_relative_limit, disk_free_limit)
  - Policies (TTL, max-length, dead-letter, lazy queues)
  - Lazy queues (messages on disk, not in RAM — for large backlogs)
  - Kubernetes deployment (StatefulSet, PVC per node, peer discovery)

FAILURE MODES (8):
  1. Consumer crashes → messages requeue → processing storm (use prefetch + manual ack)
  2. Queue depth growing unbounded → memory alarm → broker blocks publishers
  3. No DLX configured → poison message retried forever → blocks queue
  4. Network partition → split-brain → duplicate messages (idempotent consumers required)
  5. Message not persistent + broker restart → data loss
  6. No publisher confirms → publisher thinks message sent, broker never received
  7. Prefetch too high → consumer buffers 10K messages, crashes, all requeue
  8. Unacknowledged message leak (consumer bug) → messages stuck in unacked state
```

### 2B: Amazon SQS

```
ARCHITECTURE:
  - Standard Queue (at-least-once, best-effort ordering, unlimited throughput)
  - FIFO Queue (exactly-once, strict ordering, 3000 msg/s with batching)
  - When SQS vs RabbitMQ:
    SQS: fully managed, infinite scale, no ops, simple patterns
    RabbitMQ: complex routing, exchanges, priorities, lower latency
    NovaMart: SQS for async decoupling (order notifications),
              RabbitMQ for complex routing (payment events)

OPERATIONS:
  - Visibility timeout (how long consumer has to process before message re-appears)
  - Dead Letter Queue (DLQ) (maxReceiveCount → redrive policy)
  - Long polling (WaitTimeSeconds=20, reduces empty responses)
  - Batching (SendMessageBatch, ReceiveMessage MaxNumberOfMessages=10)
  - Message attributes, message body
  - FIFO: MessageGroupId (ordering), MessageDeduplicationId (exactly-once)
  - Encryption (SSE-SQS, SSE-KMS)
  - Access policies (resource-based, cross-account)
  - CloudWatch metrics (ApproximateNumberOfMessagesVisible, ApproximateAgeOfOldestMessage)
  - SQS → Lambda trigger (event source mapping)

FAILURE MODES (5):
  1. Visibility timeout too short → message reprocessed by another consumer → duplicates
  2. No DLQ → poison message retried until retention expires (14 days)
  3. FIFO dedup window 5 min → identical messages after 5 min not deduped
  4. Standard queue redelivery → consumer must be idempotent
  5. Batch delete fails partially → some messages re-appear unexpectedly
```

### 2C: Event-Driven Architecture Patterns

```
PATTERNS:
  - Event sourcing (store events, derive state)
  - CQRS (Command Query Responsibility Segregation)
  - Saga pattern (distributed transactions via compensating events)
    Choreography (events trigger next step) vs Orchestration (central coordinator)
  - Outbox pattern (DB write + event publish atomically via CDC)
  - Idempotent consumers (idempotency key, dedup table, UPSERT)
  - Ordering guarantees (partition key, message group, sequence numbers)
  - Schema evolution (Avro/Protobuf with schema registry, backward compatible changes)
  - At-least-once vs exactly-once vs at-most-once (tradeoffs)

NOVAMART EVENT FLOWS:
  - Order placed → payment.charge → inventory.reserve → shipping.schedule → notification.send
  - Payment failed → order.cancel → inventory.release → notification.send
  - How NovaMart implements: RabbitMQ topic exchange with DLX + retry pattern
```

### 2D: Apache Airflow

```
ARCHITECTURE:
  - Components: Webserver, Scheduler, Worker, Metadata DB, Executor
  - Executors: SequentialExecutor (dev), LocalExecutor (small), 
    CeleryExecutor (scale), KubernetesExecutor (NovaMart choice — pod per task)
  - DAG (Directed Acyclic Graph) — Python code that defines workflow

DAG CONCEPTS:
  - Operators: BashOperator, PythonOperator, KubernetesPodOperator,
    BigQueryOperator, S3ToRedshiftOperator, etc.
  - Sensors: S3KeySensor, ExternalTaskSensor, HttpSensor (wait for condition)
  - TaskFlow API (@task decorator — modern, preferred)
  - XComs (cross-task communication — small data only, NOT for large datasets)
  - Connections and Hooks (database, API, cloud credentials)
  - Variables (key-value config stored in metadata DB)
  - Pools (limit concurrent tasks of a type)
  - Priority weights, trigger rules (all_success, one_failed, none_failed)
  - Branching (BranchPythonOperator, ShortCircuitOperator)
  - Dynamic DAGs (generating DAGs from config — factory pattern)

KUBERNETES DEPLOYMENT:
  - Helm chart (official)
  - KubernetesExecutor (each task = new pod, auto-cleanup)
    vs CeleryExecutor (persistent workers, faster startup, manage workers)
  - Git-sync sidecar (DAGs from Git repo)
  - Connections stored in: env vars, Secrets Manager, Vault
  - RBAC (role-based access to DAGs)

NOVAMART USAGE:
  - Daily ETL: order_data → BigQuery (analytics)
  - Hourly: aggregate payment metrics → reporting DB
  - Daily: data quality checks (Great Expectations)
  - Weekly: user segmentation pipeline → GCS → ML training
  - Monthly: financial reconciliation report

OPERATIONS:
  - CLI: airflow dags list, trigger, test, backfill
  - Monitoring: task duration, failure rate, SLA misses
  - Prometheus exporter (statsd → prometheus)
  - Log management (task logs in S3/GCS, not local disk)
  - Scaling (more workers, larger pods, pool limits)
  - DAG versioning (keep backward compatible, careful with backfills)

FAILURE MODES (6):
  1. Scheduler down → no new tasks start → entire pipeline halted
  2. XCom abuse (pass large data) → metadata DB bloated → scheduler slow
  3. Zombie tasks (worker dies mid-task) → stuck in "running" → needs clear
  4. Clock skew → tasks scheduled at wrong time or missed
  5. Connection exhaustion (too many concurrent tasks hitting same DB/API)
  6. DAG import error (Python syntax) → DAG disappears from UI → no alerts
```

---

## Lesson 3: CDN, Edge, WAF & Performance Testing

**Duration:** Large lesson
**Why it matters:** Cloudflare and AWS WAF are the first things traffic hits. A misconfigured WAF blocks legitimate traffic. A misconfigured CDN serves stale data. Neither has kubectl — you need different debugging tools. Load testing is how you validate SLOs before production traffic proves them wrong.

### 3A: Cloudflare

```
ARCHITECTURE:
  - Anycast network (request goes to nearest PoP)
  - Reverse proxy model (origin never directly exposed)
  - DNS (authoritative + proxy)
  - Tiers: Free, Pro, Business, Enterprise

DNS:
  - Proxied (orange cloud) vs DNS-only (grey cloud)
  - Proxied = traffic through Cloudflare (CDN, WAF, DDoS active)
  - DNS-only = just DNS resolution (no protection, no caching)
  - CNAME flattening (CNAME at zone apex — Cloudflare resolves to A)
  - Page rules / redirect rules
  - Workers for custom DNS logic

CDN / CACHING:
  - Cache behavior (what's cached by default, what's not)
  - Cache-Control headers (s-maxage for CDN, max-age for browser)
  - Cache keys (URL, query string, headers, cookies)
  - Cache Tiers (Edge → Regional Tier → Origin)
  - Tiered Cache (reduce origin hits)
  - Cache rules (by path, header, cookie)
  - Purge (purge everything, purge by URL, purge by tag/prefix)
  - Cache Reserve (persistent cache in R2, survives purges)
  - Polish (image optimization), Mirage (lazy load)
  - Cache bypass patterns (cookies, authorization headers)
  - APO (Automatic Platform Optimization for WordPress — not for NovaMart)

WAF:
  - Managed rulesets (OWASP Core Rule Set, Cloudflare Specials)
  - Custom rules (firewall rules — expression language)
  - Rate limiting (per IP, per path, per country, response codes)
  - Bot management (verified bots, bot score, challenge)
  - IP Access Rules (allow/block/challenge by IP/ASN/country)
  - Zone Lockdown (allow only specific IPs to a path)

DDoS PROTECTION:
  - L3/L4 DDoS (automatic, always on, unmetered)
  - L7 DDoS (HTTP flood, bot mitigation, managed challenges)
  - Under Attack Mode (CAPTCHA for all visitors — emergency only)
  - Rate limiting as DDoS mitigation
  
WORKERS:
  - V8 isolates (not containers, extremely fast cold start <1ms)
  - Use cases: A/B testing at edge, header manipulation, auth, geolocation routing
  - Workers KV (key-value at edge), Durable Objects (state at edge)
  - R2 (S3-compatible storage, no egress fees)

ZERO TRUST / ACCESS:
  - Cloudflare Access (replace VPN for internal apps)
  - Identity provider integration (Okta, Azure AD, Google)
  - Application policies (who can access what)

TERRAFORM PROVIDER:
  - cloudflare_zone, cloudflare_record, cloudflare_ruleset
  - cloudflare_page_rule, cloudflare_access_application
  - API tokens (scoped, not global API key)

FAILURE MODES (7):
  1. Cache serves stale data after deploy → purge not triggered → users see old version
  2. WAF false positive blocks legitimate API calls → revenue loss
  3. Rate limiting too aggressive → blocks scrapers AND real users
  4. Origin IP exposed → attacker bypasses Cloudflare → direct DDoS to origin
  5. SSL mode "Flexible" → Cloudflare→origin is HTTP → MITM vulnerability
  6. DNS propagation delay → new records not resolving → partial outage
  7. Workers runtime error → 500 for all traffic through that route
```

### 3B: AWS WAF

```
ARCHITECTURE:
  - WAF attached to: ALB, CloudFront, API Gateway, AppSync
  - Web ACL → Rules → Rule Groups (AWS Managed, custom, marketplace)
  - Rule evaluation: first match wins (configurable action: Allow/Block/Count/CAPTCHA)

RULE TYPES:
  - AWS Managed Rules:
    AWSManagedRulesCommonRuleSet (XSS, SQLi, LFI, etc.)
    AWSManagedRulesKnownBadInputsRuleSet (Log4j, etc.)
    AWSManagedRulesSQLiRuleSet
    AWSManagedRulesAmazonIpReputationList
    AWSManagedRulesBotControlRuleSet
  - Rate-based rules (count requests per IP per 5-min window)
  - Geo-match rules (block/allow by country)
  - IP set rules (allow/block specific IP ranges)
  - Regex pattern set rules
  - Size constraint rules
  - Custom rules (JSON body inspection, header matching)

OPERATIONS:
  - Logging (Kinesis Firehose → S3, CloudWatch Logs, S3 direct)
  - Sampled requests (see what's being blocked/allowed)
  - Count mode (log without blocking — essential for testing rules)
  - WAF + Shield Advanced (DDoS response team, cost protection)
  - Terraform: aws_wafv2_web_acl, aws_wafv2_rule_group

DEPLOYMENT STRATEGY:
  1. Deploy rules in COUNT mode first
  2. Monitor for false positives (sampled requests, logs)
  3. Tune rules (scope-down statements, exclusions)
  4. Switch to BLOCK mode after validation
  5. Always have a kill switch (disable specific rule quickly)

FAILURE MODES (4):
  1. Rule blocks legitimate API calls → monitor count mode FIRST
  2. Rate limit triggers on legitimate traffic spike (sale event) → pre-warm rules
  3. Managed rule update blocks existing functionality → pin versions
  4. WAF capacity units exceeded → rules silently not evaluated
```

### 3C: Load & Performance Testing

```
TOOLS:
  - k6 (JavaScript, modern, Grafana ecosystem):
    Scenarios (constant VUs, ramping, arrival rate)
    Thresholds (p95 < 500ms, error_rate < 1%)
    Checks (response validation)
    Metrics export (Prometheus, InfluxDB, Cloud)
    Browser testing (k6 browser module)
  - Locust (Python, distributed, custom protocols):
    TaskSets, event hooks, distributed mode
  - vegeta (Go, CLI, HTTP load generator):
    Constant rate, attack, plot results
  - wrk / wrk2 (C, high-performance, latency testing):
    Constant throughput (wrk2), Lua scripting

K6 DEEP DIVE (NovaMart primary):
  - Script structure:
    options (stages, thresholds), setup(), default function, teardown()
  - Virtual Users vs Arrival Rate:
    Open model (new arrivals/sec regardless of response time — realistic)
    Closed model (fixed VUs that wait for response — less realistic)
  - Scenarios:
    constant-vus, ramping-vus, constant-arrival-rate,
    ramping-arrival-rate, externally-controlled
  - Thresholds:
    http_req_duration, http_req_failed, http_reqs,
    custom metrics, percentile thresholds
  - Output: JSON, CSV, Prometheus remote write, k6 Cloud
  - CI integration: k6 in Jenkins pipeline, fail build on threshold breach

PERFORMANCE TESTING TYPES:
  - Load test (expected traffic — does it meet SLOs?)
  - Stress test (beyond expected — where does it break?)
  - Spike test (sudden surge — does it recover?)
  - Soak test (sustained load — memory leaks, resource exhaustion)
  - Capacity test (find max throughput at SLO compliance)

METHODOLOGY:
  1. Define success criteria FROM SLOs (p99 < 500ms, error < 0.05%)
  2. Create realistic scenarios (user journey, not just endpoint)
  3. Test against staging (same infra, synthetic data)
  4. Establish baseline (current production metrics)
  5. Run load → compare against baseline → find bottleneck
  6. Fix bottleneck → retest → repeat until SLO met at target load
  7. Run soak test (24h minimum for leak detection)
  8. Document results, capacity limits, scaling thresholds

NOVAMART SCENARIOS:
  - Browse → Search → Product Detail → Add to Cart → Checkout → Payment
  - Peak traffic simulation (Black Friday: 5x normal)
  - API gateway rate limit validation
  - Database connection pool saturation point
  - Redis cache hit ratio under load

FAILURE MODES (5):
  1. Testing staging not production → different infra → misleading results
  2. Not testing with realistic data → small dataset = everything cached
  3. Not testing the full path → LB/CDN/WAF bypassed → miss real bottleneck
  4. Testing closed model → real traffic is open model → capacity wrong
  5. One-time test → no continuous regression → performance degrades unnoticed
```

---

## Lesson 4: FinOps & Cost Engineering

**Duration:** Large lesson
**Why it matters:** NovaMart's cloud bill is $3-5M/month. A 15% savings = $450-750K/year. Senior DevOps engineers who can't speak costs are incomplete. FinOps is increasingly a first-class interview topic.

```
AWS PRICING MODEL:
  - On-Demand (default, no commitment, most expensive)
  - Reserved Instances (1yr/3yr commitment, 30-60% savings, specific instance type)
  - Savings Plans:
    Compute SP (any instance, any region, any OS — 30-50% savings)
    EC2 Instance SP (specific family, specific region — 50-60% savings)
    SageMaker SP
  - Spot Instances (90% savings, can be interrupted with 2-min warning)
  - Decision framework:
    Steady-state baseline → Savings Plans / RI
    Variable/fault-tolerant → Spot (CI agents, batch, Karpenter workers)
    Short experiments → On-Demand
    Never: RI for something you might not need in 6 months

COST VISIBILITY:
  - AWS Cost Explorer (daily/monthly trends, forecasting, filtering)
  - Cost and Usage Report (CUR) — THE source of truth:
    S3 delivery, Athena integration, line-item detail
  - Cost allocation tags:
    AWS-generated (aws:createdBy, etc.)
    User-defined (team, service, environment, cost-center)
    Tag enforcement via SCP + Kyverno policies
  - Kubecost / OpenCost (K8s cost allocation):
    Per-namespace, per-deployment cost
    Idle resource detection
    Network cost allocation
    Savings recommendations
    Prometheus integration
  - AWS Budgets (alerts at %, forecasted threshold)
  - AWS Cost Anomaly Detection (ML-based, monitor by service/account/tag)

RIGHT-SIZING:
  - EC2 / Node right-sizing:
    CloudWatch metrics (CPU, memory via CW agent)
    AWS Compute Optimizer recommendations
    VPA recommendations → set requests correctly → Karpenter selects right instance
  - Graviton (ARM) migration:
    Multi-arch container images (buildx --platform linux/amd64,linux/arm64)
    20-40% price/performance improvement
    Karpenter: nodeSelector or requirement for arm64
    Gotchas: some packages don't have ARM binaries, JVM works fine
  - RDS right-sizing:
    Performance Insights → identify CPU/memory headroom
    Downsize during off-peak (modify instance class)
    gp3 vs io2 (gp3 baseline 3000 IOPS free, io2 for extreme I/O)
  - EBS optimization:
    gp2 → gp3 migration (same price, better performance, decoupled IOPS)
    Delete unattached volumes (DLM policies, automation)
    Snapshot lifecycle policies

SPOT STRATEGY:
  - Karpenter + Spot:
    Diversification (multiple instance types, families, AZs)
    capacity-optimized-prioritized allocation strategy
    Interruption handling (Karpenter graceful drain + EventBridge SQS)
    Consolidation (Karpenter replaces underutilized nodes)
  - Spot for CI/CD (Jenkins agents, build jobs — perfect use case)
  - Spot for batch/data processing (Airflow KubernetesPodOperator with Spot)
  - What should NEVER be Spot:
    Databases, stateful services, control plane, monitoring

STORAGE OPTIMIZATION:
  - S3 lifecycle policies:
    Standard → Standard-IA (30 days) → Glacier Instant (90 days) →
    Glacier Flexible (180 days) → Glacier Deep Archive (365 days) → Delete
  - S3 Intelligent-Tiering (automatic, $0.0025/1K objects monitoring fee)
  - EBS DLM (Data Lifecycle Manager — snapshot scheduling, retention, cross-region)
  - ECR lifecycle policies (keep last N tagged, expire untagged after 7d)
  - CloudWatch Logs retention (set per log group, don't keep forever)
  - Loki/Thanos/Tempo data lifecycle (already configured in Phase 7)

DATA TRANSFER:
  - NAT Gateway: $0.045/GB + $0.045/hr — THE silent cost killer
    Fix: VPC endpoints (Gateway endpoints free for S3/DynamoDB)
    Fix: ECR pull-through cache (avoid cross-internet pulls)
    Fix: S3 Gateway endpoint (most S3 traffic goes through NAT otherwise)
  - Cross-AZ: $0.01/GB each way ($0.02 round trip)
    Fix: topologySpreadConstraints + service topology awareness
    Fix: Istio locality-aware routing
  - Cross-Region: $0.02-0.09/GB
    Fix: minimize cross-region data sync, use regional caches
  - CloudFront: cheaper than S3 direct + faster (edge caching)
    Use for all static assets, API acceleration in some cases
  - VPC Flow Logs: volume-based cost
    Fix: sample rate, shorter retention, Parquet format for cheaper storage

FINOPS PRACTICES:
  - Showback vs Chargeback:
    Showback: "Team X costs $Y/month" (visibility, no enforcement)
    Chargeback: "Team X's budget is $Y, overage requires VP approval"
    NovaMart: Showback first, move to chargeback when culture is ready
  - Cost reviews (weekly, monthly, quarterly — at different granularity)
  - Waste detection automation:
    Unattached EBS volumes, idle ELBs, stopped EC2 instances,
    unused EIPs, oversized RDS, underutilized NAT Gateways
  - Reserved capacity planning:
    Analyze 12-month usage → baseline → commit to SP at baseline level
    Leave headroom (70% coverage target, rest on-demand/spot)
  - FinOps KPIs:
    Unit cost (cost per transaction, cost per user, cost per $1 revenue)
    Coverage ratio (how much is on RI/SP)
    Waste percentage (idle resources / total spend)
    Forecast accuracy (forecasted vs actual)

NOVAMART COST EXERCISE:
  - Build cost dashboard (Kubecost + CUR + Athena)
  - Identify top 5 savings opportunities
  - Propose Savings Plan purchase strategy
  - Calculate ROI on Graviton migration
  - Implement tag enforcement policy

FAILURE MODES (6):
  1. Savings Plan overcommit → locked into capacity you don't need
  2. No cost allocation tags → can't attribute cost to teams → no accountability
  3. Spot instance interruption during critical workload → service disruption
  4. NAT Gateway cost ignored → $10K+ surprise bill
  5. gp2 volumes with IOPS credits exhausted → disk latency spike
  6. S3 lifecycle too aggressive → data deleted before compliance retention met
```

---

## Lesson 5: Multi-Cloud (GCP) & Platform Engineering

**Duration:** Large lesson
**Why it matters:** NovaMart uses GCP for analytics/ML. Senior DevOps must navigate multi-cloud without being expert in both. Platform Engineering is THE career trajectory for Senior DevOps/SRE — it's what you'll be asked about in every Staff+ interview.

### 5A: GCP for NovaMart Analytics

```
GCP FUNDAMENTALS (DevOps perspective, not certification):
  - Projects (≈ AWS accounts), Folders, Organization
  - IAM: roles (basic, predefined, custom), service accounts,
    Workload Identity Federation (equivalent to IRSA for cross-cloud)
  - Networking: VPCs (global, not regional), subnets (regional),
    Cloud NAT, Cloud Interconnect / VPN to AWS
  - CLI: gcloud, gsutil, bq

BIGQUERY (NovaMart analytics):
  - Architecture: serverless, columnar, separate storage + compute
  - Concepts: dataset, table, view, materialized view
  - Querying: standard SQL, partitioned tables (ingestion-time, column),
    clustered tables (sort order), cost control (LIMIT doesn't reduce cost!)
  - Streaming inserts vs batch load
  - Scheduled queries (replace some Airflow DAGs)
  - Cost: $6.25/TB scanned (on-demand), slots (flat-rate)
  - BigQuery ML (train models in SQL — NovaMart recommendation engine)
  - Data transfer: S3 → BigQuery (BigQuery Data Transfer Service)

GCS (Google Cloud Storage):
  - Equivalent to S3 (buckets, objects, lifecycle, IAM)
  - Storage classes: Standard, Nearline, Coldline, Archive
  - NovaMart use: ML training data, analytics artifacts, Airflow DAG outputs

GKE BASICS (NovaMart ML inference cluster):
  - Autopilot vs Standard mode
  - Workload Identity (service account binding — like IRSA)
  - GKE + Anthos for multi-cluster management
  - NovaMart: ML model serving on GKE, data pipelines on Airflow→BigQuery

CROSS-CLOUD PATTERNS:
  - Identity: AWS IRSA + GCP Workload Identity Federation
    (exchange OIDC tokens between clouds, no long-lived keys)
  - Networking: AWS VPN / Direct Connect ↔ GCP Cloud Interconnect / VPN
    Or: public internet with mTLS (simpler, NovaMart's current approach)
  - Data: S3 → BigQuery Transfer Service, Airflow DAGs orchestrating cross-cloud
  - Observability: OTel Collector sends to Prometheus (AWS) AND Cloud Monitoring (GCP)
  - Terraform: separate state per cloud, shared variables

FAILURE MODES (4):
  1. BigQuery query scans full table → $500 query bill → set up cost controls
  2. Long-lived GCP service account key → rotate, use Workload Identity instead
  3. Cross-cloud network latency → data pipeline timeouts → plan for async
  4. Different IAM models → permission denied debugging is 2x complexity
```

### 5B: Platform Engineering & Developer Experience

```
INTERNAL DEVELOPER PLATFORM (IDP):
  - What: self-service layer over infrastructure
  - Why: developer productivity, consistency, reduce platform team toil
  - Maturity model:
    L1: Wiki + Slack questions (ad-hoc, tribal knowledge)
    L2: Templates + documentation (standardized, but manual)
    L3: CLI + automation (novactl, golden paths) ← NovaMart current
    L4: Portal + self-service (Backstage, automated provisioning)
    L5: Fully autonomous (AI-assisted, intent-based)

BACKSTAGE (Spotify):
  - Architecture: app-config.yaml, plugins, catalog
  - Software Catalog:
    catalog-info.yaml in every repo (kind: Component, API, Resource)
    System model: Component → API → System → Domain
    Relationships, lifecycle (experimental, production, deprecated)
    Auto-discovery from Git repos
  - Software Templates:
    Scaffolder (cookiecutter-like, creates repos + infra)
    NovaMart: "New Microservice" template → repo + CI/CD + K8s manifests + SLOs
  - TechDocs (docs-as-code, embedded in Backstage)
  - Plugins (K8s, ArgoCD, PagerDuty, Grafana, cost, security)
  - Kubernetes plugin (pod status, deployments, logs)
  - Build vs Buy:
    Backstage (free, flexible, large community, engineering effort to run)
    Port (SaaS, faster, less customizable, $$$)
    Cortex (SaaS, service catalog, scorecards)
    
