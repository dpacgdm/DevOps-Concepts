# PART 4: LEADERSHIP & SYSTEM DESIGN

# Phase 10, Lesson 1: Platform Architecture & System Design

---

## WHAT THIS PHASE COVERS

Parts 1-3 taught you **how to build and operate** the NovaMart platform. Part 4 teaches you **how to think about it** — how to design systems from scratch, make architectural decisions under uncertainty, communicate technical strategy to leadership, and navigate the organizational dynamics that determine whether good engineering actually ships.

```
╔══════════════════════════════════════════════════════════════════╗
║  Phase 10: The Senior/Staff Engineer Curriculum                  ║
║                                                                  ║
║  Lesson 1: Platform Architecture & System Design                 ║
║    → Design a complete platform from zero                        ║
║    → Make trade-off decisions and defend them                    ║
║    → Capacity planning and growth modeling                       ║
║                                                                  ║
║  Lesson 2: Organizational Patterns & Technical Strategy          ║
║    → Team topologies for platform engineering                    ║
║    → Writing RFCs and ADRs                                       ║
║    → Presenting technical strategy to executives                 ║
║    → Vendor evaluation frameworks                                ║
║                                                                  ║
║  Lesson 3: Interview Mastery & Career Growth                     ║
║    → System design interview frameworks                          ║
║    → Behavioral questions with STAR method                       ║
║    → Portfolio presentation                                      ║
║    → Negotiation                                                 ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## LESSON 1: PLATFORM ARCHITECTURE FROM SCRATCH

### The Core Question

You're hired as the first Platform Engineer at a company. There is no platform. There are 8 development teams, 40 microservices, everything is deployed manually to EC2 instances via SSH. You need to design the target state platform AND the migration path to get there.

This is the question that separates senior ICs from mid-level engineers. Mid-level says "let's set up Kubernetes." Senior says "let's understand the constraints first."

---

## 1. THE DESIGN PROCESS — CONSTRAINTS BEFORE SOLUTIONS

```
╔══════════════════════════════════════════════════════════════════╗
║  STEP 1: Understand the business context                         ║
║  STEP 2: Inventory what exists                                   ║
║  STEP 3: Define requirements (functional + non-functional)       ║
║  STEP 4: Identify constraints (budget, team, timeline, skills)   ║
║  STEP 5: Design the target architecture                          ║
║  STEP 6: Plan the migration (phases, risk mitigation)            ║
║  STEP 7: Define success metrics                                  ║
║  STEP 8: Write the RFC                                           ║
║                                                                  ║
║  Most engineers skip to Step 5. That's why most platforms fail.  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Step 1: Business Context Questions

Before drawing a single architecture diagram, you ask:

```
BUSINESS QUESTIONS:
├── What's the revenue model? (SaaS vs e-commerce vs marketplace)
│   → Determines: uptime requirements, traffic patterns, compliance
│
├── What's the growth trajectory? (10% vs 10x next year)
│   → Determines: whether to over-engineer now or keep it simple
│
├── What's the deploy frequency today? What should it be?
│   → Determines: CI/CD investment priority
│
├── What are the compliance requirements? (SOC2, PCI, HIPAA, GDPR)
│   → Determines: security architecture, data residency, audit logging
│
├── What's the incident frequency? What's the MTTR?
│   → Determines: observability investment priority
│
├── What's the monthly cloud spend? What's the budget for platform?
│   → Determines: what we can afford (managed services vs self-hosted)
│
├── What's the team's current skill level?
│   → Determines: complexity ceiling, training investment
│
└── What has been tried before and failed?
    → Determines: organizational constraints, political landmines
```

### Step 2: Platform Design Dimensions

Every platform decision lives on a spectrum. Your job is to pick the right point for your organization:

```
SIMPLICITY ◄──────────────────────────────────────────► CAPABILITY
  ECS Fargate                                          Kubernetes
  Heroku-like                                          Full CNCF stack
  "It just works"                                      "We can do anything"

MANAGED ◄────────────────────────────────────────────► SELF-HOSTED
  RDS, ElastiCache                                    PostgreSQL on EC2
  EKS managed node groups                             kOps self-managed
  Datadog SaaS                                        Prometheus + Thanos

CENTRALIZED ◄────────────────────────────────────────► FEDERATED
  Platform team owns everything                       Each team owns their infra
  Consistent but bottleneck                           Flexible but chaotic

GOLDEN PATH ◄───────────────────────────────────────► FULL FREEDOM
  "Use Go, gRPC, PostgreSQL"                          "Use whatever you want"
  Fast onboarding, constrained                        Slow onboarding, flexible
```

### NovaMart's Position (as designed throughout this curriculum)

```
Simplicity:   ████████░░  Kubernetes (complex but justified by scale)
Managed:      ███████░░░  EKS + RDS + ElastiCache (managed where possible)
Centralized:  ██████░░░░  Platform team provides, dev teams consume
Golden Path:  ████████░░  Strong defaults, opt-out with ADR justification
```

---

## 2. REFERENCE ARCHITECTURE — THE COMPLETE PICTURE

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        NOVAMART PLATFORM ARCHITECTURE                          │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌──── DEVELOPER EXPERIENCE ──────────────────────────────────────────────┐    │
│  │ novactl CLI     │ Backstage Portal │ GitOps PRs      │ Slack ChatOps   │    │
│  └────────┬────────┴─────────┬────────┴────────┬────────┴─────────┬───────┘    │
│           │                  │                 │                  │            │
│  ┌──── CI/CD ─────────────────────────────────────────────────────────────┐    │
│  │ Jenkins (shared library) → ECR → GitOps Repo → ArgoCD                  │    │
│  │ Pre-commit → Build → Test → Scan → Push → Update Manifests → Sync      │    │
│  └───────────────────────────────────┬────────────────────────────────────┘    │
│                                      │                                         │
│  ┌──── KUBERNETES (EKS) ─────────────┴──────────────────────────────────────┐  │
│  │                                                                          │  │
│  │ ┌─ Ingress ────┐  ┌─ Service Mesh ─────────────────────────────────────┐ │  │
│  │ │ ALB + NGINX  │  │                                                    │ │  │
│  │ │ Ingress      ├──┤  ┌───────────┐  ┌───────────┐  ┌───────────┐       │ │  │
│  │ │ Controller   │  │  │ order     │  │ payment   │  │ inventory │  ...  │ │  │
│  │ └──────────────┘  │  │ service   │  │ service   │  │ service   │       │ │  │
│  │                   │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘       │ │  │
│  │                   │        │              │              │             │ │  │
│  │                   └────────┼──────────────┼──────────────┼─────────────┘ │  │
│  │                            │              │              │               │  │
│  │ ┌─ Platform Services ──────┼──────────────┼──────────────┼───────────┐   │  │
│  │ │ External Secrets Op.     │ Karpenter    │ Cert-Manager │           │   │  │
│  │ │ Kyverno (policy)         │Metrics Server│ VPA (advisor)│           │   │  │
│  │ └──────────────────────────┴──────────────┴──────────────┴───────────┘   │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌──── DATA LAYER ────────────────────────────────────────────────────────┐    │
│  │ RDS (PostgreSQL)      │ ElastiCache (Redis)   │ MSK (Kafka)            │    │
│  │ Multi-AZ, encrypted, automated backups        │ Schema Registry        │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
│  ┌──── OBSERVABILITY ─────────────────────────────────────────────────────┐    │
│  │ Prometheus + Thanos   │ Loki (logs)           │ Tempo (traces)         │    │
│  │ Grafana (dashboards)  │ AlertManager          │ PagerDuty              │    │
│  │ SLO monitoring        │ DORA metrics          │ Cost attribution       │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
│  ┌──── SECURITY ──────────────────────────────────────────────────────────┐    │
│  │ IAM (IRSA)            │ NetworkPolicies       │ OPA/Kyverno policies   │    │
│  │ Secrets Manager       │ Falco (runtime)       │ Trivy (image scanning) │    │
│  │ WAF + Shield          │ GuardDuty             │ Audit logging          │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
│  ┌──── INFRASTRUCTURE ────────────────────────────────────────────────────┐    │
│  │ Terraform (IaC) │ 3 AWS Regions   │ Multi-AZ        │ DNS (Route53)    │    │
│  │ S3 (state)      │ DynamoDB (lock) │ VPC + TGW       │ CloudFront       │    │  
│  └─────────────────┴─────────────────┴─────────────────┴──────────────────┘    │    
│                                                                                │      
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. ARCHITECTURAL DECISION RECORDS — HOW TO DOCUMENT AND DEFEND TRADE-OFFS

Every significant architectural choice should be documented as an ADR. Here's the format and three real examples:

### ADR Template

```markdown
# ADR-XXX: [Decision Title]

## Status
Proposed | Accepted | Deprecated | Superseded by ADR-YYY

## Context
What is the issue or decision that needs to be made? What forces are at play?

## Decision
What is the change that we're proposing and/or doing?

## Consequences
### Positive
- What becomes easier or possible?
### Negative
- What becomes harder? What are we giving up?
### Risks
- What could go wrong? How do we mitigate?

## Alternatives Considered
### Alternative A: [Name]
- Pros: ...
- Cons: ...
- Why rejected: ...

### Alternative B: [Name]
- ...
```

### ADR-001: Kubernetes over ECS for Container Orchestration

```markdown
# ADR-001: Kubernetes (EKS) over ECS for Container Orchestration

## Status
Accepted (2024-01-15)

## Context
NovaMart runs 40 microservices across 8 teams. We're migrating from
manually-deployed EC2 instances to a container orchestration platform.
We need to choose between AWS ECS (Fargate or EC2 launch type) and
Amazon EKS (managed Kubernetes).

Current state:
- 40 microservices, growing to ~80 in 18 months
- 8 development teams with varying DevOps maturity
- Mix of Java, Go, Python services
- Requirements: multi-AZ HA, auto-scaling, rolling deploys, 
  service discovery, secrets management

Team composition:
- 3 platform engineers (1 has K8s experience, 2 have AWS experience)
- Development teams have basic Docker knowledge

## Decision
We will use Amazon EKS with managed node groups and Karpenter for 
autoscaling.

## Consequences

### Positive
- Rich ecosystem: Prometheus, ArgoCD, cert-manager, external-secrets,
  Kyverno — all Kubernetes-native. No reinventing the wheel.
- Portability: not locked into AWS-specific orchestration. Could move
  to GKE or on-prem if business requires.
- Industry standard: easier to hire. Candidates know K8s.
- GitOps native: ArgoCD + Kustomize is mature, well-documented.
- Network policies: fine-grained pod-to-pod security without custom VPC 
  configurations per service.
- HPA + Karpenter: sophisticated autoscaling at both pod and node level.

### Negative
- Complexity: K8s learning curve is steep. 2 of 3 platform engineers
  need training (budget: $15K for CKA/CKAD + hands-on workshops).
- Operational overhead: etcd, control plane upgrades, CRD management,
  RBAC policies. Mitigated by EKS managing the control plane.
- Cost: EKS control plane fee ($0.10/hr ≈ $73/month per cluster).
  Marginal vs. overall spend.

### Risks
- Team ramp-up takes longer than expected → mitigate with phased 
  migration (non-critical services first) and pair programming.
- K8s version upgrades cause downtime → mitigate with PDBs, upgrade
  runbooks, staging-first rollout, and 2-version support window.

## Alternatives Considered

### Alternative A: ECS Fargate
- Pros: Simpler operational model, no node management, faster
  team ramp-up, native AWS integration.
- Cons: Limited ecosystem (no ArgoCD, Prometheus requires custom 
  setup), vendor lock-in, no network policies (security gap),
  limited scheduling flexibility, weaker auto-scaling primitives.
- Why rejected: At 40+ services across 8 teams, we need the 
  ecosystem richness. The operational simplicity of Fargate doesn't
  outweigh the capability gap for our scale.

### Alternative B: ECS + EC2 Launch Type
- Pros: More control than Fargate, lower cost for steady-state.
- Cons: Same ecosystem limitations as Fargate, plus now we're
  managing EC2 capacity ourselves — worst of both worlds.
- Why rejected: If we're managing instances anyway, K8s gives us
  significantly more capability for similar operational cost.

### Alternative C: Nomad (HashiCorp)
- Pros: Simpler than K8s, excellent multi-workload support.
- Cons: Much smaller ecosystem, harder to hire, fewer managed
  offerings, no AWS-managed control plane equivalent.
- Why rejected: Ecosystem and hiring pool too small for our needs.
```

### ADR-002: GitOps (ArgoCD) over Push-Based Deployments

```markdown
# ADR-002: GitOps with ArgoCD over Push-Based Deployments

## Status
Accepted (2024-02-01)

## Context
We need a deployment model for 40+ microservices across 3 environments
(dev, staging, production). Current state: Jenkins runs `kubectl apply`
directly against clusters (push model).

Problems with current push model:
1. No single source of truth for "what's deployed"
2. Jenkins has cluster-admin credentials (security risk)
3. Drift: manual kubectl changes overwrite CI/CD state
4. No audit trail for who changed what and when
5. Rollback = re-run old Jenkins build (slow, error-prone)

## Decision
Adopt GitOps with ArgoCD. Git repository is the source of truth.
ArgoCD runs inside the cluster and pulls desired state.

### Key design decisions:
- Separate GitOps repo from application repos (app-of-apps pattern)
- Kustomize (not Helm) for manifest templating — simpler, more 
  transparent, better for GitOps diffs
- ArgoCD Application per service per environment
- Automated sync for dev/staging, manual sync approval for production
- Image Updater disabled — image tags updated by CI pipeline via 
  deploy-bot (audit trail in commit messages)

## Consequences

### Positive
- Git is the source of truth: `git log` shows deployment history
- Drift detection: ArgoCD alerts if cluster state ≠ Git state
- Rollback = `git revert` (instant, auditable)
- Reduced blast radius: ArgoCD has namespace-scoped RBAC, not 
  cluster-admin
- Self-healing: ArgoCD re-applies desired state if manually changed

### Negative
- Additional component to operate (ArgoCD HA deployment)
- Learning curve for teams unfamiliar with Kustomize overlays
- Slightly longer deploy feedback loop (commit → sync → rollout vs 
  direct kubectl apply)

### Risks
- ArgoCD becomes single point of failure for all deploys → mitigate 
  with HA deployment (3 replicas) and break-glass kubectl access
- Git repo becomes bottleneck with 40+ services committing → mitigate 
  with branch-per-environment structure and automated merge
```

---

## 4. CAPACITY PLANNING — THINKING ABOUT GROWTH

```
╔══════════════════════════════════════════════════════════════════╗
║  CAPACITY PLANNING IS NOT "ADD MORE SERVERS"                     ║
║  It's: "Given growth projections, when do we hit limits,         ║
║  and what do we need to change BEFORE we hit them?"              ║
╚══════════════════════════════════════════════════════════════════╝
```

### The Capacity Model

```
CURRENT STATE (measured):
├── Traffic: 5,000 RPS peak (Black Friday: 15,000 RPS)
├── Services: 40 microservices, 120 pods total
├── Nodes: 15 × m5.2xlarge (8 vCPU, 32 GB) = 120 vCPU, 480 GB RAM
├── Node utilization: 65% CPU, 55% memory (target: <70%)
├── Storage: 2 TB RDS, 500 GB Redis, 1 TB S3
├── Network: 500 Mbps avg, 2 Gbps peak (inter-AZ)
├── Monthly AWS spend: $45,000
└── Current headroom: ~35% CPU, ~45% memory

GROWTH PROJECTION (12 months):
├── Traffic: 2x (business projects 100% YoY growth)
├── Services: 40 → 70 (new features, decomposition)
├── Data: 2x (more products, more users, more events)
└── Teams: 8 → 12

CAPACITY CALCULATIONS:
├── Pods: 120 → ~210 (70 services × 3 avg replicas)
├── CPU required: 210 pods × 500m avg = 105 vCPU (request)
│   With 70% target utilization: 105 / 0.70 = 150 vCPU needed
│   Current: 120 vCPU → NEED 30 MORE vCPU (≈ 4 more nodes)
├── Memory: 210 pods × 512Mi avg = 107 GB (request)
│   With 70% target: 107 / 0.70 = 153 GB needed
│   Current: 480 GB → SUFFICIENT (32% utilization)
│   INSIGHT: We're CPU-constrained, not memory-constrained
│   DECISION: Switch to c5.2xlarge (compute-optimized) for new nodes
├── RDS: 2 TB → 4 TB, need to plan for read replicas at 3x traffic
├── Network: 2 Gbps → 4 Gbps peak (still within VPC limits)
└── Cost projection: $45K → ~$72K (60% increase for 2x traffic)
    EFFICIENCY GAIN: Cost grows 60% for 100% traffic growth
    (economies of scale from K8s bin-packing)

BOTTLENECK ANALYSIS:
├── FIRST BOTTLENECK (hits at 1.5x): RDS connections
│   40 services × 10 connections = 400 (current)
│   70 services × 10 connections = 700
│   RDS max_connections (db.r5.2xlarge) = 1,200 → OK but approaching
│   ACTION: Implement PgBouncer connection pooling before 60 services
│
├── SECOND BOTTLENECK (hits at 2x): Prometheus cardinality
│   120 pods × 500 metrics × 15s scrape = 4M active series (current)
│   210 pods × 500 metrics = 7M active series
│   Single Prometheus can handle ~10M → OK but need Thanos for HA
│   ACTION: Deploy Thanos by Q3, implement metric relabeling to 
│   control cardinality
│
├── THIRD BOTTLENECK (hits at 3x): NAT Gateway bandwidth
│   Current: 500 Mbps avg through NAT (external API calls, ECR pulls)
│   At 3x: 1.5 Gbps → NAT Gateway is 45 Gbps, but COST: $0.045/GB
│   ACTION: Implement VPC endpoints for ECR, S3, DynamoDB (saves 
│   $2K/month at current, $6K/month at 3x)
│
└── FOURTH BOTTLENECK (hits at 2x): CI/CD build throughput
    Current: 15 builds/hour peak, Jenkins 4 executors
    At 2x: 30 builds/hour → 2-min builds × 30 = need 6 executors
    ACTION: Scale Jenkins agents, implement build caching
```

### Capacity Planning Cadence

```
WEEKLY:   Check node utilization dashboards (Grafana)
MONTHLY:  Review growth metrics vs projections
QUARTERLY: Full capacity review + cost optimization
YEARLY:   Architecture review — does the current platform 
          design hold for next year's growth?
```

---

## 5. MIGRATION STRATEGY — HOW TO GET FROM HERE TO THERE

```
╔══════════════════════════════════════════════════════════════════╗
║  THE #1 RULE OF PLATFORM MIGRATIONS:                             ║
║                                                                  ║
║  Never do a big-bang migration. Never.                           ║
║                                                                  ║
║  The Strangler Fig Pattern:                                      ║
║  New traffic → new platform                                     ║
║  Old traffic gradually redirected                                ║
║  Old platform decommissioned when empty                         ║
╚══════════════════════════════════════════════════════════════════╝
```

### NovaMart Migration Plan (EC2 → EKS)

```
PHASE 0: FOUNDATION (Weeks 1-4)
──────────────────────────────
• Set up EKS cluster with Terraform (all the IaC from Phase 4)
• Deploy platform services: ArgoCD, Prometheus, cert-manager, ESO
• Set up CI/CD pipeline (Jenkins shared library)
• Set up GitOps repository structure
• Write service onboarding documentation
• SUCCESS CRITERIA: Platform team can deploy a "hello-world" service
  end-to-end through the golden path

PHASE 1: PILOT (Weeks 5-8)
──────────────────────────
• Migrate 1 non-critical, stateless service (e.g., notification-service)
• Run in PARALLEL with EC2 version (shadow traffic or canary)
• Platform team does the migration hands-on
• Document every issue, pain point, missing tool
• SUCCESS CRITERIA: Service runs on EKS with same reliability as EC2
  for 2 weeks. Dev team can deploy independently.

PHASE 2: EARLY ADOPTERS (Weeks 9-16)
────────────────────────────────────
• Migrate 3-5 more services (select teams that are enthusiastic)
• Build novactl service create from Phase 1 learnings
• Implement monitoring and alerting parity with EC2
• SUCCESS CRITERIA: 5 services on EKS, 2+ dev teams deploying
  without platform team involvement

PHASE 3: MAJORITY (Weeks 17-30)
───────────────────────────────
• Migrate remaining stateless services (20-25 services)
• Onboard remaining development teams
• Build self-service tooling (novactl, ChatOps)
• SUCCESS CRITERIA: >80% of services on EKS

PHASE 4: STRAGGLERS + STATEFUL (Weeks 31-40)
────────────────────────────────────────────
• Migrate stateful services (careful data migration)
• Migrate services with external dependencies (VPN, legacy APIs)
• Decommission EC2 instances as services move off
• SUCCESS CRITERIA: 100% on EKS, EC2 fleet decommissioned

PHASE 5: OPTIMIZATION (Weeks 41-52)
───────────────────────────────────
• Cost optimization (right-sizing, spot instances, Karpenter tuning)
• Advanced features (service mesh, progressive delivery)
• Platform maturity improvements (Level 4+ from Lesson 4)
```

### Risk Mitigation During Migration

```
RISK: Service breaks during migration
  MITIGATION: Run old and new in parallel. Route 1% traffic first.
  Blue-green cutover with instant rollback to EC2.

RISK: Dev teams resist change ("EC2 works fine")
  MITIGATION: Start with enthusiastic teams. Demonstrate faster 
  deploys, better monitoring, self-service. Let results sell.
  Don't mandate until you have success stories.

RISK: Platform team overwhelmed with migration + support
  MITIGATION: Batch migrations. Max 3 services migrating at once.
  Office hours for dev teams (not ad-hoc Slack messages).

RISK: Unknown dependencies between EC2 and K8s services
  MITIGATION: Service mesh or VPC peering between EC2 and K8s 
  during migration. Services can communicate across both platforms.

RISK: Data migration causes downtime
  MITIGATION: Stateful services migrate LAST. Use RDS blue-green 
  deployments. Maintain read replicas during cutover.
```

---

## 6. COST MODELING — TALKING TO LEADERSHIP

Leadership doesn't care about pods and nodes. They care about:
1. **How much does it cost?**
2. **What do we get for that cost?**
3. **What's the ROI?**

### The Business Case

```
┌─────────────────────────────────────────────────────────────────┐
│              PLATFORM INVESTMENT BUSINESS CASE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  INVESTMENT (Year 1):                                            │
│  ├── Platform engineering team: 3 engineers × $180K = $540K     │
│  ├── Training (CKA/CKAD, AWS): $15K                            │
│  ├── Tooling licenses: $24K (Datadog evaluation, misc)          │
│  ├── Additional AWS spend during parallel run: $15K             │
│  ├── TOTAL: ~$594K                                              │
│                                                                  │
│  RETURNS (Year 1):                                               │
│  ├── Developer productivity: 40 devs × 5h/month saved           │
│  │   = 2,400 dev-hours/year × $90/hr = $216K                   │
│  ├── Reduced incidents: 50% fewer (better monitoring, auto-heal) │
│  │   = 12 fewer incidents × $8K avg cost = $96K                 │
│  ├── Infrastructure savings: 30% cost reduction from bin-packing │
│  │   = $45K/month × 12 × 30% = $162K                          │
│  ├── Faster time-to-market: 2x deploy frequency                 │
│  │   = Hard to quantify, but critical for business              │
│  ├── TOTAL QUANTIFIABLE: ~$474K                                 │
│                                                                  │
│  YEAR 1 ROI: $474K / $594K = 80% (payback in ~15 months)       │
│  YEAR 2 ROI: $474K / $200K (team only) = 237%                  │
│                                                                  │
│  NON-QUANTIFIABLE BENEFITS:                                      │
│  ├── Compliance readiness (SOC2 audit trail from GitOps)        │
│  ├── Hiring advantage ("modern platform" attracts talent)       │
│  ├── Reduced bus factor (documented, automated processes)       │
│  └── Foundation for future capabilities (ML, edge, multi-region)│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## LESSON 1 SUMMARY

```
╔══════════════════════════════════════════════════════════════════╗
║  1. CONSTRAINTS BEFORE SOLUTIONS: Understand business context,   ║
║     team skills, budget, and compliance before choosing tools.   ║
║                                                                  ║
║  2. DOCUMENT DECISIONS: ADRs are how you defend trade-offs       ║
║     today and explain them to the engineer who joins in 2 years. ║
║                                                                  ║
║  3. CAPACITY PLAN: Model bottlenecks before they hit. The first  ║
║     bottleneck is almost never what you expect.                  ║
║                                                                  ║
║  4. MIGRATE INCREMENTALLY: Strangler fig. Parallel running.      ║
║     Never big-bang. Let success stories sell the platform.       ║
║                                                                  ║
║  5. SPEAK BUSINESS: ROI, cost savings, risk reduction.           ║
║     Executives don't buy Kubernetes. They buy outcomes.          ║
╚══════════════════════════════════════════════════════════════════╝
```

---

# Phase 10, Lesson 1 — Retention Questions

---

**Q1 (System Design — Complete Platform):**

A Series B startup (150 engineers, 30 microservices, $2M/month AWS spend) is migrating from a monolith to microservices. They currently deploy via CodeDeploy to EC2 Auto Scaling Groups. They've asked you to design their target platform.

Constraints:
- Must support 10,000 RPS today, projected 50,000 RPS in 12 months
- PCI-DSS compliance required (they process payments)
- 4-person platform team (you + 3 engineers)
- $300K annual budget for platform tooling (excluding compute)
- Teams use Java, Go, and Python
- Current MTTR is 45 minutes, target is <15 minutes
- Current deploy frequency: 2/week, target: multiple per day

Deliver:
1. The target architecture (draw it in ASCII)
2. Three critical ADRs (the three hardest decisions, in full ADR format)
3. A 6-month migration plan with phases, success criteria, and risk mitigation
4. The capacity plan: what are the first 3 bottlenecks they'll hit at 5x scale?
5. The business case: ROI calculation for the first year

---

**Q2 (Architecture Trade-offs):**

For each pair, state which you'd choose for NovaMart and defend your position with specific technical reasoning. No fence-sitting — pick one and explain the trade-off you're accepting:

1. **Kustomize vs Helm** for K8s manifest management
2. **Prometheus + Thanos vs Datadog** for observability
3. **ArgoCD vs Flux** for GitOps
4. **Karpenter vs Cluster Autoscaler** for node scaling
5. **Istio vs Cilium Service Mesh** for service mesh (or no mesh at all)

---

**Q3 (Capacity Planning — Scenario):**

NovaMart's Black Friday is in 6 weeks. Last year's Black Friday did 3x normal traffic. This year, marketing projects 5x normal traffic based on a viral TikTok campaign.

Current baseline:
- 8,000 RPS normal, 24,000 RPS last Black Friday
- 20 nodes (m5.2xlarge), 65% CPU utilization at baseline
- RDS: db.r5.2xlarge, 380 connections avg (max 1,200)
- ElastiCache: cache.r5.xlarge, 3 GB used of 26 GB
- NAT Gateway: 800 Mbps avg

For 5x traffic (40,000 RPS):
1. Calculate exact node count needed (show your work)
2. Identify every bottleneck that will break before 40K RPS
3. For each bottleneck, specify the exact AWS resource change needed
4. Create the pre-Black Friday runbook (what to scale when, in what order)
5. What's the cost delta for the Black Friday scaling? (estimate)

---

**Q4 (Migration Design):**

A NovaMart service (`legacy-order-processor`) can't be containerized easily:
- It's a 500K-line Java monolith running on bare EC2 (m5.4xlarge)
- It reads from a shared Oracle database (not RDS — self-managed on EC2)
- It uses local filesystem for temp files (EBS volume, 500GB)
- It communicates with 12 other services via direct IP addresses (no DNS)
- It has no health check endpoint
- It has no tests
- It processes 2,000 orders/minute and cannot have more than 30 seconds of downtime

Design the migration strategy. You cannot do a big-bang rewrite. Show:
1. The strangler fig decomposition plan (what gets extracted first and why)
2. How you handle the shared Oracle database during migration
3. How you implement the parallel-run with traffic routing
4. The rollback plan if migration goes wrong
5. The timeline with milestones (realistic, not optimistic)


# Phase 10, Lesson 1 — Retention Answers

---

# Q1 — System Design: Complete Platform for Series B Startup

## 1. Target Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                TARGET PLATFORM ARCHITECTURE (Series B — PCI-DSS)                 │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──── DEVELOPER EXPERIENCE ──────────────────────────────────────────────────┐  │
│  │ Platform CLI (plat)  │ GitHub PRs (GitOps)      │ Slack ChatOps            │  │
│  │ Golden path templates│ Self-service deploys     │ /plat deploy, /oncall    │  │
│  └──────────┬───────────┴────────────┬─────────────┴───────────┬──────────────┘  │
│             │                        │                         │                 │
│  ┌──── CI/CD ─────────────────────────────────────────────────────────────────┐  │
│  │ GitHub Actions (build+test+scan)                                           │  │
│  │   ┌───────────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐ ┌──────────┐      │  │
│  │   │ Lint+Test │→│ Trivy    │→│ ECR Push   │→│ GitOps   │→│ ArgoCD   │      │  │
│  │   │           │ │ +Grype   │ │ (immutable │ │ Commit   │ │ Sync     │      │  │
│  │   │           │ │ scan     │ │ tags)      │ │ (bot)    │ │          │      │  │
│  │   └───────────┘ └──────────┘ └────────────┘ └──────────┘ └──────────┘      │  │
│  │ PCI: Every image scanned. No CRITICAL/HIGH CVEs reach production.          │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌──── EDGE / INGRESS ────────────────────────────────────────────────────────┐  │
│  │ CloudFront (CDN + WAF)  →  ALB (TLS termination)  →  NGINX Ingress         │  │
│  │ AWS Shield Advanced        ACM certificates          Rate limiting         │  │
│  │ PCI: WAF rules for OWASP Top 10, bot detection, geo-blocking               │  │
│  └───────────────────────────────────┬────────────────────────────────────────┘  │
│                                      │                                           │
│  ┌──── KUBERNETES (EKS) ─────────────┴────────────────────────────────────────┐  │
│  │                                                                            │  │
│  │ ┌── PCI CDE Namespace ───────────────────────────────────────────────────┐ │  │
│  │ │ payment-service │ card-tokenizer │ fraud-engine                        │ │  │
│  │ │ NetworkPolicy: DENY ALL except explicit allow                          │ │  │
│  │ │ Encryption: in-transit (mTLS) + at-rest                                │ │  │
│  │ │ Audit: every API call logged to immutable S3                           │ │  │
│  │ │ Node isolation: dedicated node group (taint: pci=true:NoSchedule)      │ │  │
│  │ └────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                            │  │
│  │ ┌── General Namespaces ──────────────────────────────────────────────────┐ │  │
│  │ │ order-svc │ inventory-svc │ catalog-svc │ user-svc │ ... (26)          │ │  │
│  │ │ NetworkPolicy: default-deny with explicit ingress/egress               │ │  │
│  │ │ Per-namespace ResourceQuota + LimitRange                               │ │  │
│  │ └────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                            │  │
│  │ ┌── Platform Namespace ──────────────────────────────────────────────────┐ │  │
│  │ │ ArgoCD │ Prometheus │ Loki │ Tempo │ cert-manager │ ESO                │ │  │
│  │ │ Kyverno │ Karpenter │ metrics-server │ OTEL collector                  │ │  │
│  │ └────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                            │  │
│  │ Node Strategy:                                                             │  │
│  │  ├─ PCI nodes: m5.2xlarge, dedicated, tainted, labeled pci=true            │  │
│  │  ├─ General nodes: m5.xlarge / c5.xlarge, Karpenter-managed                │  │
│  │  └─ Spot nodes: for dev/staging, batch jobs (70% cost savings)             │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌──── DATA LAYER ────────────────────────────────────────────────────────────┐  │
│  │ RDS PostgreSQL (Multi-AZ)        │ ElastiCache Redis (cluster mode)        │  │
│  │  ├─ PCI DB: encrypted, audit     │ MSK (Kafka) for event streaming         │  │
│  │  │  logging, separate subnet     │ S3: immutable audit logs, backups       │  │
│  │  └─ General DBs: per-service     │ DynamoDB: session store, feature flags  │  │
│  │ PCI: TDE enabled, connections via TLS only, IAM auth                       │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌──── OBSERVABILITY ─────────────────────────────────────────────────────────┐  │
│  │ Prometheus + Thanos (metrics)   │ Loki (logs)         │ Tempo (traces)     │  │
│  │ Grafana (dashboards + alerts)   │ PagerDuty (on-call) │                    │  │
│  │ OTEL Collector (unified ingest) │ SLO dashboards (per service)             │  │
│  │ PCI: 12-month log retention, tamper-proof S3 archival                      │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌──── SECURITY (PCI-DSS) ────────────────────────────────────────────────────┐  │
│  │ LAYER          │ CONTROL                       │ TOOL                      │  │
│  │ ────────────────────────────────────────────────────────────────────────── │  │
│  │ Identity       │ IRSA (no static creds)        │ IAM + OIDC                │  │
│  │ Network        │ Segment CDE from non-CDE      │ NetworkPolicy + SG        │  │
│  │ Secrets        │ No secrets in code/env        │ AWS Secrets Manager+ESO   │  │
│  │ Image security │ Scan all images, sign them    │ Trivy + cosign            │  │
│  │ Runtime        │ Detect anomalous behavior     │ Falco                     │  │
│  │ Policy         │ Enforce standards             │ Kyverno                   │  │
│  │ Audit          │ Log all access to CDE         │ CloudTrail + k8s audit    │  │
│  │ Encryption     │ At-rest + in-transit          │ KMS + cert-manager        │  │
│  │ WAF            │ OWASP Top 10 protection       │ AWS WAF on ALB            │  │
│  │ Pen testing    │ Quarterly external pen test   │ Third-party vendor        │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌──── INFRASTRUCTURE (Terraform) ────────────────────────────────────────────┐  │
│  │ VPC: 3 AZ, public/private/data subnets, separate PCI subnet                │  │
│  │ Accounts: prod, staging, dev, security (AWS Organizations)                 │  │
│  │ State: S3 + DynamoDB locking, per-layer state files                        │  │
│  │ Modules: VPC, EKS, RDS, ElastiCache, IAM — all versioned                   │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## 2. Three Critical ADRs

### ADR-001: EKS with PCI CDE Namespace Isolation over Separate PCI Cluster

```markdown
# ADR-001: Single EKS Cluster with Namespace Isolation for PCI CDE

## Status
Accepted

## Context
We process payments and must comply with PCI-DSS. The Cardholder Data 
Environment (CDE) — services that touch card data — must be isolated 
from non-CDE workloads. We have two options:

A) Two separate EKS clusters: one for PCI, one for general workloads
B) Single EKS cluster with namespace-level isolation for the CDE

Our platform team is 4 engineers. Our PCI scope includes 3 services:
payment-service, card-tokenizer, fraud-engine.

## Decision
Single EKS cluster with namespace-level PCI isolation, using:
- Dedicated PCI node group (tainted, labeled, in isolated subnet)
- Strict NetworkPolicy (default-deny, explicit allow-list)
- Kyverno policies enforcing PCI constraints on the CDE namespace
- Separate IAM roles (IRSA) for PCI workloads
- Kubernetes audit logging → immutable S3 bucket
- mTLS between PCI services (via cert-manager issued certs)

## Consequences

### Positive
- Single cluster to operate: 4-person team can manage one cluster well,
  not two poorly. Operational complexity is the #1 threat to a small team.
- Shared platform services (Prometheus, ArgoCD, cert-manager) — no 
  duplication of effort.
- Cost efficiency: shared control plane fee ($73/month once, not twice).
- Karpenter manages both PCI and general nodes — simpler autoscaling.
- Service-to-service communication between CDE and non-CDE is simpler 
  (in-cluster with NetworkPolicy controls vs cross-cluster via ALB).

### Negative
- PCI audit scope includes the entire cluster control plane (etcd, 
  API server). This is acceptable because EKS manages the control plane
  and AWS provides PCI-DSS compliance attestation for EKS.
- A K8s vulnerability could theoretically allow cross-namespace escape.
  Mitigated by: node isolation (PCI pods only on PCI nodes), 
  NetworkPolicy, Pod Security Standards (restricted), Falco runtime 
  detection.
- PCI auditor may push back on shared cluster. We have documentation
  showing: node isolation, network segmentation, RBAC boundaries, 
  audit logging. AWS's shared responsibility model covers control plane.

### Risks
- PCI auditor rejects namespace isolation → FALLBACK: split to 2 clusters.
  The Terraform modules are parameterized; spinning up a second cluster
  is a 2-day effort, not a redesign. NetworkPolicies become security 
  groups. ArgoCD targets two clusters instead of one.
- CDE namespace misconfiguration → MITIGATE: Kyverno mutation policies
  auto-inject required labels, network policies, and pod security context.
  OPA/Gatekeeper audit mode catches drift.

## Alternatives Considered

### Alternative A: Two Separate Clusters
- Pros: Clearest PCI isolation boundary. Simplest auditor conversation.
- Cons: Double operational overhead for 4-person team. Duplicate platform
  services. Cross-cluster service communication complexity. $73/month 
  additional control plane + ~$500/month additional node baseline.
- Why rejected: Operational burden on a 4-person team outweighs the
  marginal security benefit. We can achieve equivalent isolation via 
  namespace + node affinity + network policy, which are PCI-compliant
  per AWS's CDE-on-EKS reference architecture.

### Alternative B: PCI Workloads on ECS Fargate (Separate from EKS)
- Pros: Fargate provides stronger workload isolation (microVM per task).
- Cons: Two orchestration platforms to maintain. Different deployment
  pipelines. Different monitoring stack. Team must be expert in both.
- Why rejected: Operational split is worse than shared cluster risk.
```

### ADR-002: GitHub Actions over Jenkins for CI/CD

```markdown
# ADR-002: GitHub Actions over Jenkins for CI/CD

## Status
Accepted

## Context
We need a CI/CD platform for 30 microservices across 150 engineers.
Current state: CodeDeploy pipelines, no standardization. 
Requirements: fast builds, security scanning in pipeline, 
reproducible, low operational overhead.

Options: Jenkins, GitHub Actions, GitLab CI, CircleCI.

Our code is already on GitHub (Enterprise).

Team: 4 platform engineers. We cannot afford to operate Jenkins 
(controller HA, agent scaling, plugin management, security patching).

Budget: $300K/year for platform tooling total.

## Decision
GitHub Actions with:
- Reusable workflows in a shared `.github` repository (equivalent of 
  Jenkins shared library)
- Self-hosted runners on EKS (Actions Runner Controller) for builds 
  requiring AWS VPC access and fast ECR push
- GitHub-hosted runners for basic lint/test (no infra to manage)
- OIDC federation for AWS access (no static IAM credentials in CI)

## Consequences

### Positive
- Zero operational overhead for the CI platform itself (GitHub manages it)
- Native GitHub integration: PR checks, status badges, CODEOWNERS, 
  branch protection — all first-class
- Self-hosted runners on EKS: reuses existing infrastructure, auto-scales 
  with ARC, spot instances for cost savings
- OIDC federation: no IAM access keys stored in CI (PCI requirement)
- Reusable workflows: standardized pipeline across 30 services with
  per-team customization
- Cost: GitHub Enterprise already paid for. Actions minutes included.
  Self-hosted runners only pay for EC2 compute.

### Negative
- Vendor coupling: deeper lock-in to GitHub ecosystem. Acceptable because
  we're already all-in on GitHub.
- Reusable workflows have limitations vs Jenkins shared library
  (less flexible Groovy scripting). Acceptable for our use cases.
- Self-hosted runner security: compromised runner has VPC access.
  Mitigate: ephemeral runners (new pod per job), no persistent state,
  IRSA with minimal permissions.

### Risks
- GitHub outage blocks all CI/CD → MITIGATE: critical hotfix path via 
  local `docker build` + `aws ecr push` + manual GitOps commit. 
  Documented in break-glass runbook.

## Alternatives Considered

### Alternative A: Jenkins
- Pros: Maximum flexibility, mature plugin ecosystem, we own it.
- Cons: Operational burden is enormous for a 4-person team: HA controller,
  agent scaling, plugin updates, security patches, JVM tuning, 
  credential management. Jenkins is a full-time job for 1 engineer.
  That's 25% of our team ON CI MAINTENANCE.
- Why rejected: We cannot afford to dedicate 25% of the team to CI infra.

### Alternative B: GitLab CI
- Pros: Excellent CI/CD, built-in container registry, security scanning.
- Cons: Would require migrating from GitHub to GitLab (or running both).
  Migration cost for 150 engineers + 30 repos is prohibitive.
- Why rejected: Migration cost > benefit.
```

### ADR-003: Prometheus+Thanos+Loki over Datadog for Observability

```markdown
# ADR-003: Prometheus + Thanos + Loki over Datadog for Observability

## Status
Accepted

## Context
We need full observability (metrics, logs, traces) across 30 services.
Must support PCI audit requirements (12-month log retention, tamper-proof).

Annual budget for all platform tooling: $300K.
Datadog quote for 150 engineers, 30 services, 12-month retention: ~$180K/year.
That's 60% of our entire platform budget on ONE tool.

## Decision
Self-hosted CNCF stack: Prometheus + Thanos (metrics), Loki (logs), 
Tempo (traces), Grafana (dashboards), all running on EKS.

## Consequences

### Positive
- Cost: ~$20K/year (compute for observability workloads + S3 storage).
  Saves $160K/year vs Datadog — that's another engineer's salary.
- Full control over data: PCI audit logs never leave our AWS account.
  No third-party data processing agreement needed.
- No per-host, per-container, per-metric pricing surprises as we scale
  from 30 to 80 services.
- Deep Kubernetes integration: Prometheus is the native K8s monitoring 
  standard. ServiceMonitor CRD, kube-state-metrics, etc.

### Negative
- Operational overhead: We own the stack. Upgrades, scaling, debugging
  are our responsibility. Estimated: 10-15% of one engineer's time.
- No built-in APM (Datadog APM is excellent). We use OTEL + Tempo, 
  which has a less polished UI but adequate functionality.
- Learning curve: Grafana dashboards must be built. Datadog has 
  pre-built integrations for everything.
- On-call for observability stack: if Prometheus is down, we're flying 
  blind. Mitigate with Thanos HA + PagerDuty alert on Prometheus health.

### Risks
- Thanos complexity spirals → MITIGATE: Start with Prometheus only.
  Add Thanos when single Prometheus can't handle cardinality (~10M series).
  We're at ~2M today, so we have headroom.
- Team lacks Prometheus expertise → MITIGATE: $5K training budget for 
  PromQL workshop + Grafana certification. Two engineers attend.

## Alternatives Considered

### Alternative: Datadog
- Pros: Best-in-class UX, zero operational overhead, built-in APM/RUM/
  synthetics, excellent PCI compliance documentation.
- Cons: $180K/year at current scale. At 5x scale (80 services, 500 hosts):
  projected $400K+/year. Vendor lock-in on query language (DQL vs PromQL).
  Custom metrics pricing can be surprising.
- Why rejected: 60% of total platform budget for observability alone is
  not sustainable. At 5x scale, it exceeds total budget. The $160K 
  saved buys us an additional platform engineer.

### Compromise considered: Datadog for APM only, Prometheus for infra metrics
- Rejected: Two systems = two query languages, two alert configs, 
  context-switching during incidents. Operational complexity > benefit.
```

## 3. Six-Month Migration Plan

```
═══════════════════════════════════════════════════════════════════
PHASE 0: FOUNDATION (Weeks 1-4)
═══════════════════════════════════════════════════════════════════
  TASKS:
  ├── Terraform: VPC, EKS cluster, node groups (PCI + general)
  ├── Platform services: ArgoCD, Prometheus, cert-manager, ESO, Kyverno
  ├── CI/CD: GitHub Actions reusable workflows + ARC runners on EKS
  ├── GitOps repo structure: app-of-apps, Kustomize overlays
  ├── PCI controls: NetworkPolicies, Kyverno policies, audit logging
  ├── Security: IRSA roles, Trivy scanning in CI, Falco deployment
  ├── Platform CLI: basic `plat service create` scaffolding
  └── Documentation: onboarding guide, architecture overview, runbooks

  TEAM ALLOCATION:
  ├── Engineer 1 (you): Terraform + EKS + security
  ├── Engineer 2: CI/CD + GitHub Actions + image scanning
  ├── Engineer 3: Observability stack (Prometheus, Loki, Grafana)
  └── Engineer 4: ArgoCD + GitOps + platform CLI

  SUCCESS CRITERIA:
  ├── ✅ EKS cluster passes CIS Kubernetes benchmark (kube-bench)
  ├── ✅ CI pipeline builds, scans, and deploys a sample app end-to-end
  ├── ✅ Prometheus scrapes all platform services
  ├── ✅ Kyverno blocks non-compliant pods in PCI namespace
  └── ✅ PCI QSA reviews architecture and gives preliminary approval

  RISKS + MITIGATION:
  ├── EKS cluster provisioning issues → test in dev account first
  ├── PCI auditor concerns about K8s → prepare CDE isolation doc early
  └── Timeline pressure → Phase 0 is non-negotiable, push Phase 1 if needed

═══════════════════════════════════════════════════════════════════
PHASE 1: PILOT SERVICE (Weeks 5-8)
═══════════════════════════════════════════════════════════════════
  TASKS:
  ├── Select pilot: notification-service (stateless, non-critical, 
  │   low traffic, team is enthusiastic)
  ├── Containerize: Dockerfile, health endpoint, graceful shutdown
  ├── Write K8s manifests: Deployment, Service, HPA, PDB, NetworkPolicy
  ├── Deploy to staging EKS via GitOps
  ├── Run parallel: EC2 version handles traffic, K8s version receives 
  │   shadow traffic (mirrored via ALB weighted routing)
  ├── Validate: metrics match, latency equivalent, no errors
  ├── Cutover: shift 10% → 50% → 100% traffic via weighted target groups
  ├── Decommission: EC2 ASG for notification-service
  └── Retrospective: document pain points, update onboarding guide

  SUCCESS CRITERIA:
  ├── ✅ Service runs on EKS for 2 weeks with zero incidents
  ├── ✅ Latency p99 within 5% of EC2 baseline
  ├── ✅ Team deployed 3+ times to staging independently
  ├── ✅ Rollback tested: GitOps revert restores previous version <5 min
  └── ✅ Pain points documented and top 3 addressed

═══════════════════════════════════════════════════════════════════
PHASE 2: EARLY ADOPTERS — Non-PCI Services (Weeks 9-16)
═══════════════════════════════════════════════════════════════════
  TASKS:
  ├── Migrate 8-10 stateless, non-PCI services
  │   (catalog-svc, search-svc, recommendation-svc, email-svc, etc.)
  ├── Onboard 3-4 development teams (hands-on pairing sessions)
  ├── Build `plat service create` golden path (automated scaffolding)
  ├── Build standard Grafana dashboards (RED method per service)
  ├── Implement SLO monitoring (error rate, latency per service)
  ├── Set up PagerDuty integration + on-call rotations
  └── Build ChatOps basics (/plat deploy status, /plat oncall)

  SUCCESS CRITERIA:
  ├── ✅ 10+ services running on EKS
  ├── ✅ 4+ dev teams deploying independently (no platform team involved)
  ├── ✅ MTTR reduced to <25 minutes (from 45) for migrated services
  ├── ✅ Deploy frequency: daily for migrated services (from 2/week)
  └── ✅ Zero PCI-scoped services migrated yet (intentional)

═══════════════════════════════════════════════════════════════════
PHASE 3: PCI SERVICES + MAJORITY (Weeks 17-22)
═══════════════════════════════════════════════════════════════════
  TASKS:
  ├── Migrate PCI services: payment-service, card-tokenizer, fraud-engine
  │   (WITH PCI QSA oversight and sign-off at each step)
  ├── PCI-specific validation:
  │   ├── Pen test the CDE namespace
  │   ├── Verify NetworkPolicy isolation (no cross-namespace traffic)
  │   ├── Verify audit log completeness (every API call logged)
  │   ├── Verify encryption: in-transit (mTLS) + at-rest (KMS)
  │   └── Verify no CDE pod can reach internet directly (egress policy)
  ├── Migrate remaining non-PCI services (10-12 more)
  ├── Deploy Thanos for Prometheus HA (if cardinality requires it)
  └── Advanced features: progressive delivery (Argo Rollouts) for PCI

  SUCCESS CRITERIA:
  ├── ✅ PCI services pass QSA audit on EKS
  ├── ✅ 25+ services on EKS (>80%)
  ├── ✅ MTTR <15 minutes for all migrated services
  ├── ✅ Deploy frequency: multiple per day for most teams
  └── ✅ PCI audit evidence automatically generated (not manual)

═══════════════════════════════════════════════════════════════════
PHASE 4: COMPLETION + OPTIMIZATION (Weeks 23-26)
═══════════════════════════════════════════════════════════════════
  TASKS:
  ├── Migrate remaining stragglers (stateful, complex dependencies)
  ├── Decommission all EC2 ASGs and CodeDeploy pipelines
  ├── Cost optimization: Karpenter consolidation, spot instances,
  │   right-sizing based on 3 months of production data
  ├── Implement cost attribution (per-team, per-service)
  ├── Complete self-service tooling (full `plat` CLI)
  ├── Write operational runbooks for all Tier-1 alert scenarios
  └── Platform maturity assessment → plan Phase 2 roadmap

  SUCCESS CRITERIA:
  ├── ✅ 100% services on EKS, EC2 fleet decommissioned
  ├── ✅ MTTR <15 minutes
  ├── ✅ Deploy frequency: multiple per day
  ├── ✅ PCI-DSS certification renewed on new architecture
  ├── ✅ $2M/month AWS spend reduced by 15-25% ($300-500K/year savings)
  └── ✅ Platform team toil <40% (from ~80%)

═══════════════════════════════════════════════════════════════════
RISK MITIGATION (ALL PHASES)
═══════════════════════════════════════════════════════════════════
  RISK: Service breaks during migration
    → Parallel run with weighted traffic routing for EVERY service
    → 1% → 10% → 50% → 100% cutover with 24h soak at each step
    → Instant rollback: shift ALB weight back to EC2 target group

  RISK: PCI auditor rejects K8s
    → Engage QSA in Phase 0, not Phase 3. Early feedback.
    → Prepare AWS CDE-on-EKS reference architecture documentation
    → Fallback: separate PCI cluster (2-day effort with existing Terraform)

  RISK: Platform team burnout (4 people, 26-week sprint)
    → Phase 2-3: dev teams do their own migrations with platform support
    → Platform team builds tooling, not migrates services
    → "Office hours" model: 2 hours/day for support, rest for engineering

  RISK: Organizational resistance ("CodeDeploy works fine")
    → Phase 1 success story: demo faster deploys, better monitoring
    → Executive sponsor ensures mandate after Phase 1 success
    → Train developer advocates in each team (1 person = K8s champion)
```

## 4. Capacity Plan: First 3 Bottlenecks at 5x Scale

**Current:** 10,000 RPS → **Target:** 50,000 RPS (12 months)

```
BOTTLENECK #1: RDS Connection Exhaustion (hits at ~2.5x)
════════════════════════════════════════════════════════
  Current: 30 services × 20 connections avg = 600 connections
  At 5x: 80 services × 20 connections = 1,600 connections
  RDS db.r5.2xlarge max_connections = 1,200 → EXCEEDED AT ~2.5x

  WHY THIS IS FIRST: Connection exhaustion causes cascading failures.
  Services get "too many connections" errors → retry storms → 
  connection count spikes further → database CPU spikes → everything dies.
  This is the most dangerous bottleneck because it's non-linear.

  FIX:
  ├── Phase 1 (immediate): Deploy PgBouncer as connection pooler
  │   30 services × 20 app connections → PgBouncer → 200 DB connections
  │   Each service thinks it has 20 connections, DB sees 200 total
  ├── Phase 2 (at 3x): Read replicas for read-heavy services
  │   Catalog, search, recommendations → read replica
  │   Reduces primary connections by ~40%
  └── Phase 3 (at 5x): Upgrade to db.r5.4xlarge (max_connections: 2,400)
      + per-service database where justified (orders, payments separate)

  TIMELINE: PgBouncer must be deployed BEFORE Phase 2 migrations.

BOTTLENECK #2: Prometheus Cardinality (hits at ~3x)
════════════════════════════════════════════════════
  Current: 30 services × 150 pods × 500 metrics = ~2.25M active series
  At 5x: 80 services × 400 pods × 500 metrics = ~16M active series
  Single Prometheus max: ~10M series (with 64GB RAM)
  
  At 3x (~10M series): Prometheus OOM, query latency >30s, alerting 
  delays → you're blind during incidents.

  FIX:
  ├── Phase 1 (at 2x): Metric relabeling — drop high-cardinality labels
  │   (pod_name, container_id on non-critical metrics)
  │   Reduces cardinality ~40% → extends runway to ~3.5x
  ├── Phase 2 (at 3x): Deploy Thanos
  │   ├── Thanos Sidecar on each Prometheus
  │   ├── Thanos Query for unified view
  │   ├── Thanos Store Gateway for long-term queries (S3 backend)
  │   └── Thanos Compactor for downsampling
  └── Phase 3 (at 5x): Shard Prometheus by namespace groups
      ├── Prometheus-pci: PCI namespaces only
      ├── Prometheus-core: order, payment, inventory
      └── Prometheus-rest: everything else
      Thanos Query federates across all shards transparently.

  TIMELINE: Thanos deployment in Phase 3 of migration (weeks 17-22).

BOTTLENECK #3: NAT Gateway Cost and Throughput (hits at ~3x)
═══════════════════════════════════════════════════════════════
  Current: 800 Mbps avg through NAT
  At 5x: ~4 Gbps avg → NAT Gateway handles 45 Gbps (throughput OK)
  BUT: Cost: $0.045/GB processed
  Current: 800 Mbps × 86400 sec × 30 days ÷ 8 = ~260 TB/month
  At 5x: ~1,300 TB/month × $0.045 = $58,500/MONTH just for NAT
  
  NAT Gateway data processing becomes the single largest line item.

  FIX:
  ├── Phase 1 (immediate): VPC Endpoints for AWS services
  │   ├── ECR (dkr + api): eliminates image pull traffic through NAT
  │   ├── S3 (gateway endpoint): free, eliminates S3 traffic
  │   ├── DynamoDB (gateway endpoint): free
  │   ├── STS, CloudWatch Logs, Secrets Manager: interface endpoints
  │   └── Estimated savings: 60-70% of NAT traffic eliminated
  ├── Phase 2: Move external API calls to dedicated egress VPC
  │   with smaller NAT Gateway, monitored separately
  └── Phase 3: Evaluate AWS PrivateLink for remaining external services

  TIMELINE: VPC Endpoints in Phase 0 (week 1-4). This is a day-1 
  infrastructure decision, not an afterthought.
```

## 5. Business Case: Year 1 ROI

```
┌──────────────────────────────────────────────────────────────────────┐
│                    YEAR 1 BUSINESS CASE                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  INVESTMENT                                                          │
│  ────────────────────────────────────────────────────────────────────│
│  Platform team (4 engineers × $190K fully loaded)      $760,000      │
│  Training (CKA, PromQL, security certifications)       $20,000       │
│  GitHub Enterprise (already paid)                           $0       │
│  Additional AWS during parallel run (6 months)         $60,000       │
│  Observability stack compute (Prometheus/Loki/Grafana)  $20,000      │
│  PCI QSA audit engagement (re-certification)           $40,000       │
│  Miscellaneous tooling                                  $10,000      │
│  ────────────────────────────────────────────────────────────────────│
│  TOTAL INVESTMENT                                      $910,000      │
│                                                                      │
│  RETURNS (Quantifiable)                                              │
│  ────────────────────────────────────────────────────────────────────│
│                                                                      │
│  1. DEVELOPER PRODUCTIVITY                                           │
│     150 devs × 3 hours/month saved (self-service deploys,            │
│     no waiting for ops team, faster debugging with observability)    │
│     = 5,400 dev-hours/year × $85/hr loaded cost                      │
│                                                        $459,000      │
│                                                                      │
│  2. INFRASTRUCTURE COST REDUCTION                                    │
│     Current: $2M/month = $24M/year                                   │
│     K8s bin-packing + Karpenter + spot instances: 20% savings        │
│     (conservative — industry average is 25-40%)                      │
│     VPC Endpoints: $58K/month NAT savings at 5x scale                │
│     Year 1 (partial, last 6 months at 15% savings):                  │
│     $2M × 6 months × 15%                              $1,800,000     │
│                                                                      │
│  3. INCIDENT REDUCTION                                               │
│     Current: ~3 incidents/month × $12K avg cost                      │
│     (lost revenue + eng time + customer impact)                      │
│     50% reduction with better monitoring + auto-healing:             │
│     18 fewer incidents × $12K                           $216,000     │
│                                                                      │
│  4. OBSERVABILITY SAVINGS (vs Datadog alternative)                   │
│     Datadog quote: $180K/year                                        │
│     Self-hosted: $20K/year                              $160,000     │
│                                                                      │
│  5. COMPLIANCE AUTOMATION                                            │
│     Current: 2 engineers × 2 weeks for quarterly PCI audit prep      │
│     Automated: 1 engineer × 2 days                                   │
│     4 quarters × (80 hrs - 16 hrs) × $85/hr             $21,760      │
│                                                                      │
│  ────────────────────────────────────────────────────────────────────│
│  TOTAL QUANTIFIABLE RETURNS                           $2,656,760     │
│                                                                      │
│  ────────────────────────────────────────────────────────────────────│
│  YEAR 1 NET ROI: ($2,656,760 - $910,000) / $910,000 = 192%           │
│  PAYBACK PERIOD: ~4.1 months                                         │
│                                                                      │
│  NON-QUANTIFIABLE RETURNS                                            │
│  ├── PCI compliance posture improvement (continuous vs quarterly)    │
│  ├── Hiring advantage (modern platform attracts senior talent)       │
│  ├── Reduced bus factor (documented, automated processes)            │
│  ├── Foundation for multi-region (DR) when business requires         │
│  ├── Faster time-to-market (multiple deploys/day vs 2/week)          │
│  └── Deploy frequency improvement feeds directly into revenue:       │
│      faster feature delivery → competitive advantage                 │
│                                                                      │
│  YEAR 2 PROJECTION (team only + compute):                            │
│  Investment: $760K (team) + $20K (observability) = $780K             │
│  Returns: $3.2M+ (full-year savings + scaled traffic)                │
│  ROI: 310%+                                                          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

# Q2 — Architecture Trade-offs

### 1. Kustomize vs Helm → **Kustomize**

**Choice:** Kustomize for all NovaMart K8s manifests.

**Reasoning:**

Kustomize operates on **plain YAML with overlays**. Helm operates on **Go templates that generate YAML**. The distinction matters enormously for GitOps.

With Kustomize, what's in Git is what's applied. You `diff` the PR and see exactly what changes in the cluster. There's no template rendering step, no hidden `values.yaml` → `template` → `output` indirection. When ArgoCD shows drift, you compare Git YAML to cluster YAML directly.

With Helm, you review `values.yaml` changes and mentally compute what the template engine will produce. During an incident at 3 AM, mental template rendering is the last thing you want. A one-line `values.yaml` change can produce 50 lines of manifest changes across 4 files, and the reviewer has no way to verify without running `helm template` locally.

Additionally, Helm charts from the community are black boxes. `helm upgrade` can silently change resource definitions between chart versions in ways that aren't visible in the values diff. Kustomize patches are explicit — you see every field you're overriding.

**What I'm accepting:** Helm has a richer ecosystem of community charts (nginx-ingress, cert-manager, Prometheus). For **third-party platform components**, I'd use Helm via ArgoCD's Helm support. For **our own services** (the 30+ microservices we author), Kustomize. This hybrid avoids rewriting community charts while keeping our service manifests transparent.

**The trade-off:** Kustomize's strategic merge patches can be confusing for JSON arrays (the infamous `patchesJson6902` vs `patchesStrategicMerge`). Teams need a 30-minute onboarding session. Worth it for the GitOps transparency.

---

### 2. Prometheus + Thanos vs Datadog → **Prometheus + Thanos**

**Choice:** Self-hosted Prometheus + Thanos + Loki + Tempo.

**Reasoning:**

This was covered exhaustively in ADR-003 above. The core argument: at $2M/month AWS spend with 30 services growing to 80, Datadog pricing becomes the single largest platform tooling cost. The quote of $180K/year at current scale projects to $400K+ at 5x. That's **more than the salary of two senior engineers** who could be building platform capabilities instead.

Prometheus is the Kubernetes-native monitoring standard. Every CNCF project, every Helm chart, every operator exposes Prometheus metrics. ServiceMonitor CRDs make scrape configuration declarative. PromQL is the industry-standard query language — every candidate we interview knows it.

**What I'm accepting:**
- We own the operational burden (~15% of one engineer's time)
- Grafana dashboards must be built (no pre-built integrations)
- No built-in APM with transaction tracing waterfall charts (Tempo is less polished than Datadog APM)
- Thanos adds complexity (sidecar, query, store, compactor components)

**When I'd choose Datadog instead:** If we were a 10-person startup with no platform team and $500K ARR, Datadog's zero-ops model wins. The cost is justified when the alternative is "no monitoring because nobody has time to set it up." At 150 engineers and $24M/year infrastructure spend, we can afford 15% of an engineer to save $160K+/year.

---

### 3. ArgoCD vs Flux → **ArgoCD**

**Choice:** ArgoCD.

**Reasoning:**

ArgoCD and Flux are both mature GitOps controllers. They solve the same problem with different philosophies:

- **ArgoCD:** Application-centric. Has a **web UI** for visualizing sync status, resource tree, diff view. Supports multi-cluster from a single control plane. Application CRD is the unit of management.
- **Flux:** Git-centric. No built-in UI. Composed of multiple controllers (source-controller, kustomize-controller, helm-controller). More modular, more "Kubernetes-native."

I choose ArgoCD for three reasons:

**First, the UI.** During an incident, being able to open the ArgoCD dashboard and see "order-service is OutOfSync, here's the diff, here are the failing pods" in 10 seconds is invaluable. Flux requires `kubectl` + CLI for the same information. With 150 engineers, most of whom are NOT platform engineers, the visual dashboard lowers the barrier to understanding deployment state. This directly contributes to our MTTR target of <15 minutes.

**Second, multi-cluster management.** ArgoCD manages applications across clusters from a single instance. When we add a staging cluster or a DR cluster, ArgoCD's existing dashboard shows all environments. Flux requires per-cluster bootstrap.

**Third, the app-of-apps pattern.** ArgoCD's ApplicationSet controller dynamically generates Application CRDs from templates — perfect for our model where a new service gets a new Application per environment. With 80 services × 3 environments = 240 Applications, manual creation is untenable. ApplicationSet with a Git directory generator automates this.

**What I'm accepting:** ArgoCD is a heavier deployment than Flux (HA mode: 3 replicas of API server, repo server, application controller). It has a larger attack surface (web UI, API, Dex OIDC). ArgoCD's RBAC model (AppProject + RBAC policies) has a learning curve. Flux is simpler to operate, lighter in resource usage, and arguably more "Kubernetes-native" in design philosophy.

---

### 4. Karpenter vs Cluster Autoscaler → **Karpenter**

**Choice:** Karpenter.

**Reasoning:**

Cluster Autoscaler (CAS) operates on **node groups** — it scales ASGs up and down based on pending pods. Karpenter operates on **individual nodes** — it provisions the optimal instance type for the pending workload directly, bypassing ASGs entirely.

The difference is transformative:

**CAS:** You pre-define node groups (e.g., m5.2xlarge general, c5.2xlarge compute, r5.2xlarge memory). When pods are pending, CAS scales up the node group that matches. If you need a mix of instance types, you create more node groups. At NovaMart with PCI nodes, spot nodes, compute-optimized nodes, and general nodes, we'd have 8-12 ASGs to manage.

**Karpenter:** You define a `NodePool` with constraints (instance families: m5, c5, r5; capacity type: on-demand for PCI, spot for batch; architecture: amd64). Karpenter selects the cheapest instance type that fits the pending pods' resource requests. It bin-packs intelligently. It consolidates underutilized nodes automatically.

Concrete advantage: A pod requesting 6 vCPU and 8GB RAM. CAS scales up an m5.2xlarge (8 vCPU, 32GB) — 75% CPU utilized, 25% memory utilized. Karpenter might provision a c5.2xlarge (8 vCPU, 16GB) at a lower price point, or even a c5.xlarge (4 vCPU) if it can bin-pack with other pending pods on a larger instance.

**The cost impact is significant:** Karpenter typically achieves 15-25% better cost efficiency than CAS through optimal instance selection and automatic consolidation. On $2M/month spend, that's $300K-500K/year.

**What I'm accepting:** Karpenter is AWS-specific (it provisions EC2 instances directly, not through ASGs). If we ever need to run on GKE or on-prem, Karpenter doesn't transfer. CAS is cloud-agnostic. I'm accepting AWS lock-in on the autoscaler layer — acceptable because we're already locked into EKS, VPC, IAM, and every other AWS service. The autoscaler is not the thing that locks us in.

Karpenter is also younger than CAS (GA since late 2023). Fewer edge cases documented in the wild. Mitigated by: we're on EKS (AWS actively develops Karpenter), and we have a 4-person platform team that can debug issues.

---

### 5. Istio vs Cilium Service Mesh vs No Mesh → **No mesh today; Cilium when needed**

**Choice:** No service mesh now. Plan for Cilium service mesh when we need mTLS or advanced traffic management.

**Reasoning:**

This is the most contrarian answer, and it's deliberate.

**Why not Istio now:** Istio is an enormously complex system. It injects an Envoy sidecar into every pod (doubling pod count for resource accounting), adds ~10ms p99 latency per hop, requires its own control plane (istiod), and introduces failure modes that are extremely difficult to debug. The mTLS benefit is real, but can be achieved at the CNI layer without sidecars.

For a 4-person platform team managing 30 services growing to 80, Istio's operational burden is prohibitive. I've seen teams spend 30-50% of their platform engineering time operating Istio. That's 1-2 engineers ON MESH MAINTENANCE. We don't have that headroom.

**Why not Cilium mesh now:** Cilium is a better choice when we need mesh capabilities (eBPF-based, no sidecar, lower latency overhead, native NetworkPolicy replacement). But we don't need a mesh yet. Our services communicate via HTTP with service names resolved by kube-dns. NetworkPolicy provides sufficient segmentation. TLS between services can be handled by application-level TLS (cert-manager issuing certs).

**When we add Cilium mesh:** When we need:
- **mTLS everywhere** (PCI requirement tightens, or we need zero-trust networking)
- **Advanced traffic management** (canary deploys, traffic mirroring, circuit breaking)
- **L7 observability** (request-level metrics without application instrumentation)

At that point, Cilium as the CNI + service mesh is the right choice because:
1. No sidecar injection (eBPF operates at the kernel level)
2. We're likely already running Cilium as our CNI (replacing aws-vpc-cni)
3. Single data plane for networking + security + observability
4. Lower latency overhead than Envoy-based meshes

**What I'm accepting today:** No mTLS between non-PCI services. Service-to-service communication is encrypted via VPC internal traffic (not internet-routable, but not encrypted by default). For PCI services, we use application-level TLS with cert-manager certificates. This is a pragmatic compromise — adding mTLS to 3 PCI services is straightforward; adding it to 80 services via Istio is a massive undertaking with marginal security benefit for internal traffic.

---

# Q3 — Capacity Planning: Black Friday 5x

## Current Baseline vs 5x Target

```
                        BASELINE        5x TARGET        HEADROOM
RPS:                    8,000           40,000            -
Nodes:                  20 × m5.2xlarge  ???              -
CPU utilization:        65%              target <75%      -
RDS connections:        380 avg          ???              max 1,200
ElastiCache:            3 GB / 26 GB     ???              -
NAT Gateway:            800 Mbps         ???              45 Gbps limit
```

## 1. Node Count Calculation (Show Work)

```
CURRENT STATE:
  20 nodes × m5.2xlarge (8 vCPU, 32 GB each)
  Total cluster: 160 vCPU, 640 GB RAM
  At 65% CPU utilization: 104 vCPU USED of 160 available
  This serves 8,000 RPS

STEP 1: CPU required at 5x
  Assumption: CPU scales roughly linearly with RPS for HTTP services.
  (Validated by: HTTP request processing is CPU-bound — parsing, 
  serialization, business logic. Memory doesn't scale linearly with RPS.)

  Current CPU used: 104 vCPU at 8,000 RPS
  CPU per RPS: 104 / 8,000 = 0.013 vCPU per RPS

  At 40,000 RPS: 40,000 × 0.013 = 520 vCPU NEEDED

STEP 2: Account for target utilization
  Target max utilization during Black Friday: 75%
  (Not 70% — we accept slightly higher during a planned event)

  Total vCPU required: 520 / 0.75 = 693 vCPU

STEP 3: Account for system overhead
  Kubernetes system pods (kube-proxy, CNI, node-exporter, etc.): 
  ~0.5 vCPU per node reserved (kubelet reservation)
  DaemonSets (Falco, Datadog-agent, fluent-bit): ~0.5 vCPU per node

  Effective vCPU per m5.2xlarge: 8 - 1.0 = 7.0 usable vCPU

STEP 4: Calculate node count
  Nodes needed: 693 / 7.0 = 99 nodes

STEP 5: Account for AZ distribution + headroom
  3 AZs: must handle 1 AZ failure (lose ~1/3 of nodes)
  During AZ failure: 66 nodes serve 40K RPS
  Need: 99 nodes × 1.5 (50% AZ headroom) = 149 nodes
  
  BUT: AZ failure during Black Friday is a scenario we plan for 
  separately (multi-region failover). For capacity planning within 
  normal 3-AZ operation:

  Nodes needed: 99 nodes (round up to 100 for clean math)
  Additional AZ safety margin (10%): 110 nodes

ANSWER: 110 nodes (m5.2xlarge) for 5x Black Friday traffic
  Currently: 20 nodes
  Need to add: 90 nodes
  Scale factor: 5.5x on nodes for 5x on traffic 
  (slightly super-linear due to utilization headroom)

MEMORY CHECK:
  110 nodes × 32 GB = 3,520 GB total
  Memory at 5x: current 55% × 5x scaling = 275% → NOT LINEAR
  Memory per pod stays constant (heap sizes don't change with RPS).
  More pods needed (HPA scaling) but same memory per pod.
  
  Estimated pod count at 5x: 150 pods × 3 (avg HPA scale) = 450 pods
  450 pods × 512 MB avg = 230 GB / 3,520 GB = 6.5% utilization
  
  MEMORY IS NOT THE BOTTLENECK. We're massively over-provisioned on 
  memory at 5x. Consider switching some nodes to c5.2xlarge:
    c5.2xlarge: 8 vCPU, 16 GB RAM, $0.34/hr (vs m5.2xlarge $0.384/hr)
    11% cheaper per node × 110 nodes = significant savings.
  
  REVISED RECOMMENDATION: 
    40 × m5.2xlarge (existing services needing memory)
    70 × c5.2xlarge (compute-optimized for HTTP workloads)
```

## 2. Every Bottleneck That Breaks Before 40K RPS

```
BOTTLENECK #1: RDS CONNECTION EXHAUSTION ⚠️ BREAKS AT ~2.5x (20K RPS)
══════════════════════════════════════════════════════════════════════
  Current: 380 connections avg, peaks to 500 during traffic spikes
  At 5x: HPA scales pods → more DB connections per service
  Each pod holds ~10 connections (connection pool)
  Current pods: ~150, at 5x: ~450 pods
  450 pods × 10 connections = 4,500 connections requested
  RDS db.r5.2xlarge max_connections: 1,200 → HARD WALL at ~2.5x

  CONSEQUENCE: Connection refused errors → 500 errors → cascading 
  failure across all services that use the database.

BOTTLENECK #2: ALB TARGET GROUP SCALING ⚠️ SLOW AT 3-4x
══════════════════════════════════════════════════════════════════════
  ALB scales automatically BUT has a warm-up period.
  Sudden traffic spike (5x in <5 minutes as sale starts) → 
  ALB returns 503 while scaling. AWS calls this "pre-warming."

  CONSEQUENCE: First 2-5 minutes of Black Friday see 503 errors.

BOTTLENECK #3: HPA SCALING LAG ⚠️ CRITICAL DURING INITIAL SPIKE
══════════════════════════════════════════════════════════════════════
  HPA default scaling window: 15-second metric check, 3-minute 
  stabilization window. Pod startup time (Java services): 30-45 seconds.
  
  Timeline: Traffic spikes → HPA detects at +15s → decisions at +30s → 
  pods scheduled at +45s → pods ready at +90s → FIRST NEW POD SERVING
  at +90 seconds minimum.
  
  During that 90 seconds: existing pods handle 5x load → CPU saturation 
  → response times spike → timeouts → customer impact.

  CONSEQUENCE: 90-second degradation window at traffic spike onset.

BOTTLENECK #4: NAT GATEWAY DATA PROCESSING COST ⚠️ FINANCIAL RISK
══════════════════════════════════════════════════════════════════════
  At 5x: ~4 Gbps through NAT
  Black Friday is 3-5 days of elevated traffic
  4 Gbps × 86400 sec × 5 days / 8 = ~216 TB
  216 TB × $0.045/GB = $9,720 in NAT data processing IN 5 DAYS
  (vs normal week: ~$2,000)

  CONSEQUENCE: Surprise $8K bill if VPC endpoints aren't configured.

BOTTLENECK #5: KAFKA (MSK) PARTITION THROUGHPUT ⚠️ BREAKS AT ~3x
══════════════════════════════════════════════════════════════════════
  Order events flow through Kafka. Current: 2,000 orders/min = 33 msg/s
  At 5x: 10,000 orders/min = 167 msg/s
  If order-events topic has 6 partitions, each partition handles ~28 msg/s
  
  Single partition throughput limit: ~1 MB/s (default)
  If order messages avg 2 KB: 167 msg/s × 2 KB = 334 KB/s total (OK)
  
  BUT: Consumer lag. If order-processor can't keep up at 5x, consumer 
  group falls behind → backpressure → order processing delays.

  CONSEQUENCE: Order confirmations delayed by minutes during peak.

BOTTLENECK #6: PROMETHEUS SCRAPE LOAD ⚠️ DEGRADES AT 4x
══════════════════════════════════════════════════════════════════════
  Current: 150 pods × 500 metrics = 75K series at 15s interval
  At 5x: 450 pods × 500 metrics = 225K series
  Prometheus RAM required: ~225K × 2 KB = ~450 MB (OK)
  BUT: scrape duration increases. 450 targets × 15s interval = 
  30 targets/second. Each scrape takes ~50ms = 1.5 seconds of scrape 
  time per second. Approaching limits.

  CONSEQUENCE: Metrics delayed → alerting delayed → blind during incident.
```

## 3. Exact AWS Resource Changes

```
RESOURCE CHANGE #1: RDS — PgBouncer + Upgrade
═══════════════════════════════════════════════
  BEFORE: db.r5.2xlarge, direct connections from pods
  AFTER:
  ├── Deploy PgBouncer on EKS (3 replicas, HA)
  │   └── Mode: transaction pooling (releases connection after each txn)
  │   └── Config: max_client_conn=5000, default_pool_size=20
  │   └── Result: 450 pods → PgBouncer → 200 DB connections
  ├── Upgrade RDS: db.r5.2xlarge → db.r5.4xlarge
  │   └── vCPU: 8 → 16, RAM: 64 → 128 GB
  │   └── max_connections: 2,400 (with PgBouncer, we'll use ~200)
  │   └── Method: Multi-AZ failover upgrade (30 seconds downtime)
  └── Add read replica: db.r5.2xlarge
      └── Route catalog, search, recommendations to read replica
      └── Reduces primary load by ~40%

  COST DELTA: +$2,500/month (r5.4xlarge vs r5.2xlarge + read replica)

RESOURCE CHANGE #2: ALB — Pre-warm
═══════════════════════════════════
  BEFORE: ALB auto-scales reactively
  AFTER:
  ├── Contact AWS Support 2 weeks before Black Friday
  │   └── Request ALB pre-warming for expected 40K RPS
  │   └── Specify date/time window and expected peak
  └── Alternative: Deploy AWS Global Accelerator in front of ALB
      └── Anycast IP, always warm, no pre-warming needed
      └── Cost: $0.025/hr + $0.01/GB = ~$300/month

  COST DELTA: $0 (AWS pre-warming is free via support ticket)

RESOURCE CHANGE #3: Pre-scale pods before traffic spike
═══════════════════════════════════════════════════════════
  BEFORE: HPA scales reactively (90-second lag)
  AFTER:
  ├── CronJob: 30 minutes before sale start time
  │   └── kubectl scale deployment/order-service --replicas=15
  │   └── kubectl scale deployment/payment-service --replicas=15
  │   └── kubectl scale deployment/inventory-service --replicas=15
  │   └── (repeat for all critical services)
  ├── HPA minReplicas temporarily increased:
  │   └── kubectl patch hpa order-service -p '{"spec":{"minReplicas":15}}'
  │   └── HPA can still scale UP beyond 15 if needed
  └── Post-sale CronJob: revert to normal minReplicas

  COST DELTA: Included in node cost (pods run on pre-provisioned nodes)

RESOURCE CHANGE #4: Nodes — Pre-provision via Karpenter
═══════════════════════════════════════════════════════════
  BEFORE: 20 × m5.2xlarge
  AFTER:
  ├── Pre-sale (T-2 hours): Scale Karpenter NodePool limits UP
  │   └── maxInstances: 20 → 120
  ├── Pre-sale (T-1 hour): Deploy "placeholder" pods that reserve 
  │   capacity (low-priority pods that get evicted when real pods need space)
  │   └── 90 × placeholder pods requesting 7 vCPU each
  │   └── Karpenter provisions 90 new nodes
  ├── On sale start: HPA scales real pods → placeholder pods evicted 
  │   → real pods scheduled on pre-provisioned nodes → INSTANT
  └── Post-sale (T+6 hours): Remove placeholder pods
      └── Karpenter consolidation removes empty nodes over 30 minutes

  COST DELTA:
  ├── 70 × c5.2xlarge × $0.34/hr × 12 hours = $285
  ├── 40 × m5.2xlarge × $0.384/hr × 12 hours = $184  (already running: 20)
  ├── Net additional: 90 nodes × 12 hours × ~$0.36 avg = $389
  └── Total Black Friday compute delta: ~$400 (for 12-hour peak)

RESOURCE CHANGE #5: VPC Endpoints
═══════════════════════════════════
  BEFORE: All AWS API traffic through NAT Gateway
  AFTER:
  ├── com.amazonaws.{region}.ecr.api (interface endpoint)
  ├── com.amazonaws.{region}.ecr.dkr (interface endpoint)
  ├── com.amazonaws.{region}.s3 (gateway endpoint — FREE)
  ├── com.amazonaws.{region}.dynamodb (gateway endpoint — FREE)
  ├── com.amazonaws.{region}.sts (interface endpoint)
  ├── com.amazonaws.{region}.logs (interface endpoint)
  └── com.amazonaws.{region}.secretsmanager (interface endpoint)

  COST DELTA: Interface endpoints: 5 × $0.01/hr × 3 AZ = $1,095/month
  NAT savings: ~60% reduction = ~$4,000/month saved
  NET SAVINGS: ~$2,900/month (permanent, not just Black Friday)

RESOURCE CHANGE #6: Kafka (MSK) partition increase
═══════════════════════════════════════════════════════
  BEFORE: order-events topic, 6 partitions
  AFTER: order-events topic, 24 partitions
  ├── Allows 24 concurrent consumers (parallelism)
  ├── Each partition handles ~7 msg/s (comfortable margin)
  └── MSK broker count: 3 → 6 (if throughput requires)

COST DELTA: Additional brokers: ~$500/month (kafka.m5.large × 3 extra)
  Can revert post-Black Friday if sustained traffic doesn't justify it.
```

## 4. Pre-Black Friday Runbook

```
╔══════════════════════════════════════════════════════════════════════╗
║            PRE-BLACK FRIDAY SCALING RUNBOOK                          ║
║            Target: 40,000 RPS (5x baseline)                         ║
║            Prepared by: Platform Team                                ║
║            Last updated: [date]                                      ║
║            Slack channel: #black-friday-war-room                     ║
╚══════════════════════════════════════════════════════════════════════╝

═══════════════════════════════════════════════════════════════════════
T-6 WEEKS: INFRASTRUCTURE PREPARATION
═══════════════════════════════════════════════════════════════════════

□ OWNER: Platform Engineer 1
  Deploy VPC Endpoints (ECR, S3, DynamoDB, STS, Logs, Secrets Manager)
  ├── Terraform apply in staging, validate ECR pulls work via endpoint
  ├── Terraform apply in production
  ├── Verify: NAT Gateway CloudWatch metrics show reduced throughput
  └── ROLLBACK: Remove endpoint, traffic falls back to NAT automatically

□ OWNER: Platform Engineer 2
  Deploy PgBouncer
  ├── Deploy in staging, run load test at 3x traffic
  ├── Validate: connection count to RDS drops by 80%
  ├── Deploy in production during low-traffic window (Tuesday 3 AM)
  ├── Monitor for 48 hours: error rates, connection pool exhaustion
  └── ROLLBACK: kubectl scale pgbouncer --replicas=0, services fall 
      back to direct DB connections (update Service DNS)

□ OWNER: Platform Engineer 3
  Increase Kafka partitions (order-events: 6 → 24)
  ├── Apply in staging, run producer/consumer load test
  ├── Apply in production (partition increase is non-disruptive)
  ├── Verify consumer group rebalancing completes cleanly
  └── ROLLBACK: Cannot reduce partitions. This is permanent.
      Ensure consumer group handles 24 partitions before applying.

□ OWNER: Platform Engineer 4
  Contact AWS Support for ALB pre-warming
  ├── Open support case with: expected RPS, date/time window, region
  ├── Confirm pre-warming will be active by T-2 days
  └── FALLBACK: Deploy Global Accelerator if support case is delayed

═══════════════════════════════════════════════════════════════════════
T-2 WEEKS: LOAD TESTING
═══════════════════════════════════════════════════════════════════════

□ OWNER: Platform Engineer 1 + Development Teams
  Run full load test against staging at 5x traffic (40K RPS)
  ├── Tool: k6 or Locust, traffic profile matching real user patterns
  ├── Duration: 2 hours sustained
  ├── MEASURE:
  │   ├── p99 latency per service (target: <500ms)
  │   ├── Error rate (target: <0.1%)
  │   ├── RDS connection count (target: <300 via PgBouncer)
  │   ├── Node CPU utilization (target: <75%)
  │   ├── Kafka consumer lag (target: <1000 messages)
  │   ├── HPA scaling behavior (target: scales within 60s)
  │   └── Pod startup time (target: <45s for Java, <10s for Go)
  ├── DOCUMENT: Every metric that failed target
  ├── FIX: Address each failure before T-1 week
  └── RE-TEST: After fixes, run 2-hour load test again

□ OWNER: Platform Engineer 2
  Chaos test during load test
  ├── Kill random pods during 5x load (verify PDB protection)
  ├── Drain one node during 5x load (verify Karpenter reprovision)
  ├── Simulate AZ failure (cordon all nodes in one AZ)
  └── DOCUMENT: Recovery time for each scenario

═══════════════════════════════════════════════════════════════════════
T-1 WEEK: FINAL PREPARATIONS
═══════════════════════════════════════════════════════════════════════

□ OWNER: Platform Engineer 3
  RDS upgrade: db.r5.2xlarge → db.r5.4xlarge
  ├── Schedule Multi-AZ failover upgrade for Tuesday 3 AM
  ├── Expected downtime: 30-60 seconds (automatic failover)
  ├── Notify all development teams 48 hours in advance
  ├── Post-upgrade: verify replication lag, connection count, query perf
  └── ROLLBACK: Downgrade via Multi-AZ failover (another 30-60s window)

□ OWNER: Platform Engineer 4
  Deploy read replica for catalog/search/recommendation services
  ├── Create db.r5.2xlarge read replica in same region
  ├── Update service configs to route reads to replica endpoint
  ├── Verify replica lag <100ms under load
  └── ROLLBACK: Revert service configs to primary endpoint

□ ALL: Deploy freeze for non-critical services
  ├── ArgoCD: enable manual sync approval for ALL production apps
  ├── Slack announcement: "No production deploys Wed-Mon without 
  │   platform team approval"
  └── Exception: critical hotfixes approved by on-call + team lead

═══════════════════════════════════════════════════════════════════════
T-2 HOURS: PRE-SCALE (Day of Black Friday)
═══════════════════════════════════════════════════════════════════════

□ OWNER: Platform Engineer 1 (PRIMARY)
  Step 1: Increase Karpenter NodePool limits
    kubectl patch nodepool general -p '{"spec":{"limits":{"cpu":"900"}}}'
    kubectl patch nodepool pci -p '{"spec":{"limits":{"cpu":"200"}}}'
    VERIFY: kubectl get nodepool -o yaml | grep limits

  Step 2: Deploy placeholder pods to pre-provision nodes
    kubectl apply -f black-friday/placeholder-deployment.yaml
    # 90 pods requesting 7 vCPU each, priority class: low
    # Karpenter provisions ~90 new nodes (10-15 minutes)
    VERIFY: kubectl get nodes | wc -l  → expect ~110 nodes
    VERIFY: kubectl get nodes -l karpenter.sh/nodepool=general | wc -l

  Step 3: Pre-scale HPA minimums for critical services
    ./scripts/black-friday-scale.sh up
    # Sets minReplicas for:
    #   order-service: 3 → 15
    #   payment-service: 3 → 12
    #   inventory-service: 3 → 15
    #   catalog-service: 2 → 10
    #   search-service: 2 → 10
    #   cart-service: 3 → 12
    VERIFY: kubectl get hpa -A | grep -E "(order|payment|inventory|catalog|search|cart)"
    WAIT: All pods Running + Ready (5-10 minutes for Java services)

  Step 4: Verify all systems nominal
    ./scripts/black-friday-verify.sh
    # Checks:
    #   ✅ All pods Ready
    #   ✅ All nodes Ready  
    #   ✅ RDS connections <500 (via PgBouncer)
    #   ✅ ElastiCache hit rate >95%
    #   ✅ Kafka consumer lag <100
    #   ✅ Prometheus scraping all targets
    #   ✅ AlertManager routing working (send test alert)
    #   ✅ All ArgoCD apps in Synced state

□ OWNER: Platform Engineer 2 (BACKUP)
  Step 5: Open war room
    /novactl incident create "Black Friday Scale-Up — Planned"
    ├── Slack channel: #black-friday-war-room
    ├── Invite: platform team + on-call from each dev team
    ├── Pin: Grafana dashboard links, runbook link, escalation contacts
    └── Post: "Pre-scale complete. All systems nominal. 
              Standing by for sale start at [TIME]."

═══════════════════════════════════════════════════════════════════════
T+0: SALE STARTS — ACTIVE MONITORING
═══════════════════════════════════════════════════════════════════════

□ OWNER: All platform engineers on-call (shifts if >8 hours)
  Monitor dashboards continuously:
  ├── PRIMARY: Grafana "Black Friday Overview" dashboard
  │   ├── Total RPS (target: track against projected 40K)
  │   ├── Error rate by service (alert: >0.5%)
  │   ├── p99 latency by service (alert: >1s)
  │   ├── Node CPU utilization (alert: >80%)
  │   ├── RDS connections + CPU (alert: connections >800 or CPU >70%)
  │   ├── Kafka consumer lag (alert: >5000 messages)
  │   └── Pod restart count (alert: >0)
  ├── SECONDARY: AWS Console → EC2 → ALB monitoring
  │   └── Watch for 503s, target group health
  └── TERTIARY: PagerDuty → silence non-critical alerts
      └── Only page on: service error rate >1%, pod crash loops, 
          node failures, database issues

  ESCALATION LADDER:
  ├── L1: On-call platform engineer (responds in 5 min)
  ├── L2: Platform team lead (responds in 10 min)  
  ├── L3: VP Engineering (responds in 15 min)
  └── EXECUTIVE: CTO (for customer-facing decisions)

═══════════════════════════════════════════════════════════════════════
T+6 TO T+12 HOURS: TRAFFIC DECLINING
═══════════════════════════════════════════════════════════════════════

□ OWNER: Platform Engineer 1
  DO NOT scale down immediately.
  ├── Wait until traffic sustained below 2x for 2 hours
  ├── Then: ./scripts/black-friday-scale.sh down
  │   └── Reverts HPA minReplicas to normal values
  ├── Delete placeholder pods:
  │   kubectl delete -f black-friday/placeholder-deployment.yaml
  ├── Karpenter consolidation will remove empty nodes over 30-60 min
  └── Verify: node count decreasing, no pod disruptions

═══════════════════════════════════════════════════════════════════════
T+48 HOURS: POST-BLACK FRIDAY CLEANUP
═══════════════════════════════════════════════════════════════════════

□ OWNER: Platform Engineer 3
  ├── Evaluate: do we keep the RDS upgrade? (check sustained traffic)
  │   └── If traffic returns to baseline: downgrade to save cost
  │   └── If traffic elevated (holiday season): keep upgrade
  ├── Review: read replica usage — keep or decommission?
  ├── Review: Karpenter NodePool limits — revert to normal
  ├── Re-enable: ArgoCD auto-sync for production
  └── Remove: deploy freeze

□ OWNER: Platform Engineer 4
  Post-mortem (even if nothing went wrong)
  ├── Document: what actually happened vs what we planned
  ├── Metrics: peak RPS achieved, peak latency, any errors
  ├── Cost: actual Black Friday compute cost vs estimate
  ├── What worked well (keep for next time)
  ├── What didn't work (fix before next event)
  └── Publish: to #engineering and share with leadership
```

## 5. Cost Delta for Black Friday Scaling

```
┌────────────────────────────────────────────────────────────────────────┐
│                       BLACK FRIDAY COST ESTIMATE                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  TEMPORARY COSTS (Black Friday window: ~24 hours)                      │
│  ────────────────────────────────────────────────────────────────────  │
│  Additional nodes: 90 × c5.2xlarge × $0.34/hr × 24hr          $734.40  │
│  Read replica: db.r5.2xlarge × $0.48/hr × 24hr                 $11.52  │
│  Additional NAT traffic (even with VPC endpoints):            $150.00  │
│  Kafka additional brokers (already provisioned):                $0.00  │
│  ────────────────────────────────────────────────────────────────────  │
│  SUBTOTAL TEMPORARY:                                          $895.92  │
│                                                                        │
│  SEMI-PERMANENT COSTS (keeping for holiday season: 6 weeks)            │
│  ────────────────────────────────────────────────────────────────────  │
│  RDS upgrade (r5.2xlarge → r5.4xlarge) difference:                     │
│    ($0.96 - $0.48)/hr × 24hr × 42 days                        $967.68  │
│  Read replica (keep for holiday season):                               │
│    $0.48/hr × 24hr × 42 days                                  $483.84  │
│  VPC Endpoints (keep permanently — saves money):                       │
│    5 interface endpoints × $0.01/hr × 3AZ × 24hr × 42d        $151.20  │
│  ────────────────────────────────────────────────────────────────────  │
│  SUBTOTAL SEMI-PERMANENT:                                   $1,602.72  │
│                                                                        │
│  COST SAVINGS (from VPC Endpoints — permanent)                         │
│  ────────────────────────────────────────────────────────────────────  │
│  NAT Gateway reduction: ~60% of $4,000/month               -$2,400/mo  │
│  VPC Endpoint cost:                                        +$1,095/mo  │
│  NET MONTHLY SAVINGS:                                      -$1,305/mo  │
│                                                                        │
│  ════════════════════════════════════════════════════════════════════  │
│  TOTAL BLACK FRIDAY DIRECT COST DELTA:                        ~$2,500  │
│  TOTAL HOLIDAY SEASON (6 weeks) DELTA:                        ~$4,500  │
│  ANNUAL SAVINGS FROM VPC ENDPOINTS:                          -$15,660  │
│  ════════════════════════════════════════════════════════════════════  │
│                                                                        │
│  NET OUTCOME: Black Friday scaling PAYS FOR ITSELF because the         │
│  VPC endpoint optimization (done as part of preparation) saves more    │
│  annually than the temporary scaling costs.                            │
│                                                                        │
│  COMPARISON: Last year's Black Friday on EC2:                          │
│  Manually provisioned 30 extra EC2 instances 2 days early              │
│  (needed them "just in case") × $0.384/hr × 48hr = $552.96             │
│  BUT: they were m5.2xlarge (most idle). K8s bin-packing + Karpenter    │
│  auto-selects optimal instance types. Equivalent scaling on EC2        │
│  would have cost ~$1,200 (larger instances to handle pod scheduling    │
│  overhead that doesn't exist on EC2).                                  │
│                                                                        │
│  K8s scaling is CHEAPER and FASTER than EC2 ASG scaling.               │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Q4 — Migration Design: `legacy-order-processor`

## Constraints Summary

```
THE BEAST:
├── 500K lines of Java monolith
├── Bare EC2 (m5.4xlarge — 16 vCPU, 64 GB RAM)
├── Shared Oracle database (self-managed on EC2, NOT RDS)
├── Local EBS filesystem for temp files (500 GB)
├── 12 services communicate via hardcoded IP addresses
├── No health check endpoint
├── No tests
├── Processes 2,000 orders/minute
├── Max 30 seconds downtime allowed
└── "Cannot do a big-bang rewrite"
```

## 1. Strangler Fig Decomposition Plan

**Principle:** Extract capabilities from the edges inward. Start with features that have clear boundaries, low coupling to the Oracle DB, and high value if isolated.

```
EXTRACTION ORDER (outside-in):
══════════════════════════════════════════════════════════════

EXTRACT #1: Order Notifications (Weeks 1-6)
──────────────────────────────────────────
  WHY FIRST:
  ├── Clearest boundary: "order placed → send email/SMS" is a 
  │   well-defined event handler with minimal DB coupling
  ├── Lowest risk: notification failure doesn't lose orders
  ├── Easiest to verify: "did the customer get the email?"
  ├── Read-only from Oracle: queries order data, doesn't write
  └── Establishes the pattern for all future extractions

  HOW:
  ├── Create notification-service (new microservice on EKS)
  ├── Monolith publishes "OrderPlaced" event to Kafka
  │   (add Kafka producer to monolith — small, safe change)
  ├── notification-service consumes events and sends notifications
  ├── Initially: both monolith AND new service send notifications
  │   (dual-write, verify new service matches)
  ├── After 2 weeks: disable notification in monolith
  └── Oracle dependency: NONE (notification data comes via Kafka event)

  VALIDATION:
  ├── Compare: emails sent by monolith vs new service (should match)
  ├── Metric: notification delivery rate (must not decrease)
  └── Rollback: re-enable monolith notifications (feature flag)

EXTRACT #2: Order Status / Tracking API (Weeks 7-14)
──────────────────────────────────────────────────────
  WHY SECOND:
  ├── Read-only from Oracle (SELECT queries only)
  ├── High-traffic endpoint (customers checking order status)
  ├── Extracting this reduces 30% of monolith's HTTP traffic
  ├── Can be fronted by a cache (Redis) for further DB offloading
  └── Builds team confidence for harder extractions

  HOW:
  ├── Create order-status-service on EKS
  ├── Replicate relevant Oracle tables to PostgreSQL (RDS)
  │   using CDC (Change Data Capture) via Debezium
  │   (Oracle → Kafka → PostgreSQL, real-time sync)
  ├── order-status-service reads from PostgreSQL, NOT Oracle
  ├── API gateway routes /api/orders/{id}/status to new service
  ├── Monolith's /status endpoint remains active (fallback)
  └── Compare responses between old and new for 2 weeks

  ORACLE DEPENDENCY: Eliminated via CDC replication. New service 
  never touches Oracle directly.

EXTRACT #3: Order Validation / Pricing (Weeks 15-24)
──────────────────────────────────────────────────────
  WHY THIRD:
  ├── Business-critical but well-bounded logic
  ├── "Given cart + customer, calculate price + validate inventory"
  ├── Currently buried in monolith, causes bugs on every change
  ├── Extracting this enables faster pricing experiments
  └── Writes to Oracle (creates order records) — hardest extraction so far

  HOW:
  ├── Create order-validation-service on EKS
  ├── Uses PostgreSQL for order creation (new orders go to PG)
  ├── CDC syncs new orders BACK to Oracle 
  │   (PostgreSQL → Kafka → Oracle sink connector)
  │   so monolith can still process them
  ├── Dual-write period: both monolith and new service create orders
  │   Reconciliation job compares nightly
  └── After validation period: monolith delegates to new service

EXTRACT #4-N: Remaining capabilities (Weeks 25-52)
──────────────────────────────────────────────────────
  ├── Inventory reservation (read-write, complex locking)
  ├── Payment orchestration (PCI scope, highest risk)
  ├── Fulfillment workflow (stateful, long-running)
  ├── Reporting / analytics (read-only, bulk queries)
  └── Admin operations (low traffic, can migrate last)

  ORDER PRINCIPLE: Each extraction reduces the monolith's 
  responsibilities. Eventually it becomes a thin routing shell
  that delegates to microservices. The shell can then be 
  decommissioned when all routes are migrated.

WHAT STAYS IN THE MONOLITH LONGEST:
  └── The Oracle write path. This is the last thing to migrate 
      because it's the most coupled and the most dangerous.
      We migrate reads first (via CDC), then writes (via new services
      that write to PostgreSQL with CDC back-sync to Oracle).
```

## 2. Handling the Shared Oracle Database

```
THE ORACLE PROBLEM:
═══════════════════
Oracle is the gravitational center. 12 services connect to it.
You CANNOT migrate the database in one step. You must decouple 
services from Oracle incrementally while keeping Oracle as the 
source of truth during the transition.

STRATEGY: CHANGE DATA CAPTURE (CDC) + DUAL DATABASE
════════════════════════════════════════════════════

                    ┌──────────────┐
                    │    Oracle    │ (source of truth during migration)
                    │  (EC2, self  │
                    │   managed)   │
                    └──────┬───────┘
                           │
                    CDC (Debezium)
                    Oracle LogMiner
                           │
                           ▼
                    ┌──────────────┐
                    │    Kafka     │ (change events)
                    │   (MSK)     │
                    └──────┬───────┘
                           │
               ┌───────────┼───────────┐
               ▼           ▼           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │PostgreSQL│ │PostgreSQL│ │PostgreSQL│
        │ orders   │ │ catalog  │ │ users    │
        │ (RDS)    │ │ (RDS)    │ │ (RDS)    │
        └──────────┘ └──────────┘ └──────────┘
              ▲             ▲            ▲
              │             │            │
        order-service  catalog-svc  user-service
        (new, on EKS)  (new)        (new)

PHASE 1: READ PATH MIGRATION
  ├── Deploy Debezium on EKS, connecting to Oracle via LogMiner
  ├── CDC replicates Oracle tables to per-service PostgreSQL databases
  ├── New microservices read from PostgreSQL (near-real-time replica)
  ├── Oracle remains the write target — monolith still writes to Oracle
  ├── CDC lag: typically <1 second (monitor with Kafka consumer lag)
  └── VALIDATION: Compare query results between Oracle and PostgreSQL
      Run nightly reconciliation job (row counts, checksums)

PHASE 2: WRITE PATH MIGRATION (per service)
  ├── New service writes to PostgreSQL directly
  ├── Reverse CDC: PostgreSQL → Kafka → Oracle sink connector
  │   (keeps Oracle in sync for services still reading from it)
  ├── Monolith reads from Oracle (sees changes from new service)
  ├── DUAL-WRITE PERIOD: both paths active, reconciliation catches gaps
  └── When all readers of a table are off Oracle:
      stop reverse CDC, table is fully migrated to PostgreSQL

PHASE 3: ORACLE DECOMMISSION
  ├── When ALL services read/write PostgreSQL exclusively
  ├── Keep Oracle read-only for 30 days (safety net)
  ├── Run final reconciliation: PostgreSQL == Oracle (row-level)
  ├── Decommission Oracle EC2 instances
  └── Cancel Oracle license ($$$ savings)

CRITICAL RISKS:
  ├── CDC lag during peak traffic → Oracle changes not yet in PG
  │   MITIGATE: Monitor Kafka consumer lag. Alert if >5 seconds.
  │   MITIGATE: For critical reads, add fallback to Oracle if PG 
  │   data is stale (check replication timestamp).
  ├── Schema changes in Oracle break CDC
  │   MITIGATE: Schema changes go through review process.
  │   Debezium handles most DDL changes, but column renames break it.
  ├── Oracle-specific SQL (PL/SQL, Oracle functions) not portable
  │   MITIGATE: Identify Oracle-specific SQL in monolith early.
  │   Rewrite in application code or PostgreSQL equivalents.
  └── Oracle license cost continues during entire migration
      ACCEPT: This is the cost of safe migration. Budget for 12+ months 
      of dual-database operation.
```

## 3. Parallel Run with Traffic Routing

```
TRAFFIC ROUTING STRATEGY:
═══════════════════════════

The monolith listens on a fixed IP. 12 services connect via hardcoded IPs.
We can't change this overnight. Here's the migration pattern:

STEP 1: Put a reverse proxy in front of the monolith
──────────────────────────────────────────────────────

  BEFORE:
    Client → ALB → Monolith (EC2, IP: 10.0.1.50)
    Service-A → 10.0.1.50:8080/api/orders

  AFTER:
    Client → ALB → NGINX (EKS) → Monolith (EC2)
                         │
                         ├── /api/orders/status → order-status-service (EKS)
                         ├── /api/notifications → notification-service (EKS)
                         └── /api/* → Monolith (EC2) [default fallback]

  HOW: Deploy NGINX Ingress with path-based routing rules.
  The ALB target group changes from EC2 instance to EKS NGINX pods.
  Monolith becomes an "upstream" in NGINX config.

STEP 2: Fix hardcoded IPs with DNS
──────────────────────────────────────
  12 services use hardcoded IPs to reach the monolith.
  
  APPROACH:
  ├── Create Route53 private hosted zone record:
  │   legacy-orders.internal.novamart.com → 10.0.1.50 (monolith EC2)
  ├── Update each service (one at a time) to use DNS name instead of IP
  │   (This is a config change, not a code change — usually an env var)
  ├── Once all 12 services use DNS:
  │   Swap DNS to point to NGINX Ingress on EKS
  │   legacy-orders.internal.novamart.com → NGINX Ingress IP
  └── Now ALL traffic flows through NGINX → routes to monolith or new services

  TIMELINE: 2-3 weeks (one service per day, verify after each)
  ROLLBACK: Swap DNS back to monolith IP (instant, <30 second TTL)

STEP 3: Gradual traffic shifting per endpoint
──────────────────────────────────────────────
  For each extracted service:

  Phase A: Shadow mode (1 week)
    NGINX mirrors 100% of traffic to both monolith and new service.
    New service responses are DISCARDED — only monolith responses returned.
    Compare: log both responses, nightly diff report.

  Phase B: Canary (1 week)
    NGINX routes 1% of traffic to new service, 99% to monolith.
    Monitor error rates, latency, business metrics.
    Increase: 1% → 5% → 10% → 25% → 50% → 100%

  Phase C: Full cutover
    NGINX routes 100% to new service.
    Monolith endpoint remains active but receives no traffic.
    Keep for 2 weeks as fallback.

  Phase D: Decomission monolith endpoint
    Remove endpoint code from monolith (reduces monolith surface area).

  NGINX configuration (example):
  ```nginx
  upstream monolith {
      server legacy-orders-ec2.internal:8080;
  }
  upstream order_status_new {
      server order-status-service.orders.svc.cluster.local:8080;
  }

  # Canary: 10% to new service
  split_clients "${request_id}" $order_status_backend {
      10%  order_status_new;
      *    monolith;
  }

  location /api/orders/status {
      proxy_pass http://$order_status_backend;
  }
  ```
```

## 4. Rollback Plan

```
ROLLBACK STRATEGY — LAYERED
═══════════════════════════════

LAYER 1: TRAFFIC ROLLBACK (instant, <30 seconds)
─────────────────────────────────────────────────
  TRIGGER: New service error rate >1% or latency >2x baseline
  ACTION:
  ├── NGINX: set canary weight to 0% (all traffic → monolith)
  │   kubectl edit configmap nginx-canary-config
  │   OR: kubectl annotate ingress order-status 
  │       nginx.ingress.kubernetes.io/canary-weight="0"
  ├── DNS fallback: If NGINX is the problem, swap Route53 record
  │   back to monolith EC2 IP directly (30-second TTL)
  └── VERIFY: error rate drops to baseline within 60 seconds

  RECOVERY TIME: <30 seconds (DNS or NGINX weight change)
  RISK: None — monolith never stopped running

LAYER 2: DATA ROLLBACK (minutes)
─────────────────────────────────
  TRIGGER: New service wrote incorrect data to PostgreSQL
  ACTION:
  ├── Stop new service from accepting writes (scale to 0 replicas)
  ├── Reverse CDC is still running → Oracle still has correct data
  ├── Re-route reads back to Oracle (monolith handles everything)
  ├── Assess: what incorrect data was written?
  ├── Fix: manual correction in PostgreSQL from Oracle source
  └── Resume: re-enable new service after fix verified

  RECOVERY TIME: 5-15 minutes (depending on data volume)
  RISK: Orders processed by new service during bad period need review

LAYER 3: FULL MIGRATION ROLLBACK (hours)
──────────────────────────────────────────
  TRIGGER: Fundamental architecture issue (CDC can't keep up, 
  Oracle performance degraded by LogMiner, etc.)
  ACTION:
  ├── Revert all traffic to monolith (Layer 1 for each service)
  ├── Stop Debezium CDC connectors
  ├── Disable all new microservices (scale to 0)
  ├── Monolith operates exactly as it did before migration
  ├── Assess: what went wrong, can it be fixed?
  └── Re-plan: adjust migration approach based on learnings

  RECOVERY TIME: 1-2 hours (full revert)
  RISK: New data written to PostgreSQL needs reconciliation back to Oracle

ROLLBACK TESTING SCHEDULE:
  ├── Week 1 of each extraction: Test Layer 1 rollback in staging
  ├── Week 2: Test Layer 1 rollback in production (during low traffic)
  ├── Monthly: Test Layer 2 rollback in staging with synthetic data
  └── Never tested in production: Layer 3 (but documented and reviewed)

THE 30-SECOND DOWNTIME CONSTRAINT:
  All rollback paths maintain <30s downtime because:
  ├── The monolith NEVER stops running during migration
  ├── Traffic routing is the ONLY thing that changes
  ├── DNS TTL is set to 30 seconds
  ├── NGINX canary weight changes are instant (0 downtime)
  └── The monolith is the always-available fallback
```

## 5. Realistic Timeline with Milestones

```
══════════════════════════════════════════════════════════════════
MONTH 1-2: FOUNDATION + FIRST EXTRACTION
══════════════════════════════════════════════════════════════════
  Week 1-2:  Infrastructure setup
    ├── Deploy NGINX reverse proxy in front of monolith
    ├── Set up Debezium CDC pipeline (Oracle → Kafka → PostgreSQL)
    ├── Deploy Kafka (MSK) cluster
    └── MILESTONE: CDC replicating 3 core Oracle tables to PostgreSQL

  Week 3-4:  DNS migration (hardcoded IPs → DNS names)
    ├── Create Route53 private zone
    ├── Update 12 services to use DNS (1-2 per day)
    └── MILESTONE: All 12 services using DNS, zero using hardcoded IPs

  Week 5-8:  Extract #1 — Notification Service
    ├── Week 5-6: Build notification-service, deploy to EKS
    ├── Week 7: Shadow mode (dual-send, compare)
    ├── Week 8: Full cutover, disable monolith notifications
    └── MILESTONE: Notifications running on EKS, monolith code removed

══════════════════════════════════════════════════════════════════
MONTH 3-4: SECOND EXTRACTION + CONFIDENCE BUILDING
══════════════════════════════════════════════════════════════════
  Week 9-11:  Build order-status-service
    ├── CDC replication for order tables validated
    ├── Service reads from PostgreSQL
    └── MILESTONE: Service passing integration tests against replicated data

  Week 12-14: Traffic migration for order-status
    ├── Shadow mode (1 week)
    ├── Canary: 1% → 10% → 50% → 100% (2 weeks)
    └── MILESTONE: 100% order-status traffic on EKS, 30% reduction in 
        monolith HTTP traffic

  Week 15-16: Add health check to monolith
    ├── This seems backwards — but NOW we have monitoring to justify it
    ├── Add /health endpoint to monolith (returns 200 if DB connected)
    ├── Add liveness probe via NGINX (proxy_pass to monolith /health)
    └── MILESTONE: Monolith health visible in Grafana dashboards

══════════════════════════════════════════════════════════════════
MONTH 5-8: WRITE PATH MIGRATION (HARDEST PHASE)
══════════════════════════════════════════════════════════════════
  Week 17-20: Extract #3 — Order Validation/Pricing
    ├── First service that WRITES to the database
    ├── New service writes to PostgreSQL
    ├── Reverse CDC: PostgreSQL → Kafka → Oracle (keeps monolith in sync)
    ├── Dual-write reconciliation (nightly comparison)
    └── MILESTONE: Order creation working on EKS with bidirectional sync

  Week 21-24: Extract #4 — Inventory Reservation
    ├── Stateful, requires distributed locking (Redis)
    ├── Most complex extraction (inventory + orders tightly coupled)
    └── MILESTONE: Inventory service on EKS, Oracle read dependency eliminated
        for inventory data

  Week 25-30: Extract #5-7 — Remaining services
    ├── Fulfillment workflow
    ├── Reporting (read-only, easiest of this batch)
    ├── Admin operations
    └── MILESTONE: Monolith handles <20% of original responsibilities

══════════════════════════════════════════════════════════════════
MONTH 9-12: ORACLE DECOMMISSION + MONOLITH RETIREMENT
══════════════════════════════════════════════════════════════════
  Week 31-36: Payment orchestration extraction (PCI scope)
    ├── Highest risk extraction — saved for last intentionally
    ├── Requires PCI QSA review of new payment service
    ├── Must maintain PCI compliance throughout migration
    └── MILESTONE: Payment processing on EKS, PCI audit passed

  Week 37-42: Oracle read elimination
    ├── All services now read/write PostgreSQL
    ├── Oracle receives reverse CDC writes only (for any stragglers)
    ├── Stop reverse CDC → Oracle becomes read-only archive
    ├── 30-day observation period: Oracle read-only, verify nothing breaks
    └── MILESTONE: Oracle decommissioned, license terminated

  Week 43-48: Monolith retirement
    ├── Monolith is now an empty shell (all routes delegate to services)
    ├── Remove NGINX proxy routing to monolith
    ├── Decommission monolith EC2 instance
    ├── Decommission EBS volume (archive to S3 first)
    └── MILESTONE: Monolith fully retired. Zero EC2 for order processing.

  Week 49-52: Optimization + celebration
    ├── Right-size all PostgreSQL databases based on 3 months production data
    ├── Implement connection pooling where needed
    ├── Cost optimization (spot instances for non-PCI, Karpenter tuning)
    ├── Document the migration as a case study
    └── MILESTONE: Total order processing on microservices, 
        12-month migration complete

══════════════════════════════════════════════════════════════════
RISK BUFFER: 20% (add 2 months to total timeline)
══════════════════════════════════════════════════════════════════
  Realistic total: 12-14 months (not 12 optimistic months)
  
  WHAT WILL GO WRONG (and it will):
  ├── CDC lag during peak traffic (1-2 weeks to tune)
  ├── Oracle-specific SQL that doesn't translate to PostgreSQL (weeks)
  ├── Undocumented monolith behaviors discovered during shadow testing
  ├── Team resistance from monolith developers ("why are we doing this")
  ├── Unplanned Oracle maintenance disrupts CDC pipeline
  └── PCI audit findings require additional work on payment extraction

  WHAT WON'T GO WRONG (because of our approach):
  ├── No customer-facing downtime (monolith always available as fallback)
  ├── No data loss (CDC + reconciliation catches everything)
  ├── No big-bang risk (each extraction is independently rollback-able)
  └── No "stop the world" migration (business continues operating normally)
```

```
TIMELINE VISUALIZATION:

Month:  1    2    3    4    5    6    7    8    9   10   11   12
        ├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
Infra:  ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
DNS:    ░░████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Notif:  ░░░░████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Status: ░░░░░░░░██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Orders: ░░░░░░░░░░░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Invent: ░░░░░░░░░░░░░░░░░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░
Fulfil: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████░░░░░░░░░░░░░░░░░░░
Paymnt: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████░░░░░░░░░░░
Oracle: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████░░░░░
Retire: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████░

Legend: █ = active work, ░ = not yet started / complete
```

```
# Phase 10, Lesson 2: Organizational Patterns & Technical Strategy


## WHAT THIS LESSON COVERS

Lesson 1 taught you how to design platforms. This lesson teaches you how to **get them built** — how to navigate organizational dynamics, communicate with executives, write documents that drive decisions, evaluate vendors, and structure teams that can execute.

```
╔══════════════════════════════════════════════════════════════════╗
║  The hardest problems in platform engineering are not technical. ║
║  They're organizational.                                         ║
║                                                                  ║
║  Topics:                                                         ║
║   1. Team Topologies for platform engineering                    ║
║   2. Writing RFCs that get approved                              ║
║   3. Writing ADRs that survive the test of time                  ║
║   4. Presenting technical strategy to executives                 ║
║   5. Vendor evaluation framework                                 ║
║   6. Technical debt prioritization                               ║
║   7. Building a platform adoption culture                        ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 1. TEAM TOPOLOGIES FOR PLATFORM ENGINEERING

```
╔══════════════════════════════════════════════════════════════════╗
║  TEAM TOPOLOGIES (Matthew Skelton & Manuel Pais)                 ║
║                                                                  ║
║  Four fundamental team types:                                    ║
║                                                                  ║
║  1. STREAM-ALIGNED TEAM                                          ║
║     Aligned to a single flow of business value                   ║
║     "We own the order experience end-to-end"                     ║
║     Example: Orders team, Payments team, Search team             ║
║                                                                  ║
║  2. PLATFORM TEAM                                                ║
║     Provides self-service capabilities to stream-aligned teams   ║
║     "We make it easy for you to deploy, monitor, and operate"    ║
║     Example: Platform Engineering team (that's you)              ║
║                                                                  ║
║  3. ENABLING TEAM                                                ║
║     Helps other teams adopt new capabilities                     ║
║     "We'll pair with you for 2 weeks to adopt observability"     ║
║     Example: SRE embedded with a product team temporarily        ║
║                                                                  ║
║  4. COMPLICATED SUBSYSTEM TEAM                                   ║
║     Owns a complex component that requires specialist knowledge  ║
║     "We own the ML recommendation engine"                        ║
║     Example: Data/ML team, Video encoding team                   ║
║                                                                  ║
║  Three interaction modes:                                        ║
║  ├── COLLABORATION: Working closely together (temporary)         ║
║  ├── X-AS-A-SERVICE: Consuming via API/interface (steady state)  ║
║  └── FACILITATING: Helping, coaching, unblocking (temporary)     ║
╚══════════════════════════════════════════════════════════════════╝
```

### NovaMart Team Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                    NOVAMART ENGINEERING ORG                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STREAM-ALIGNED TEAMS (own business domains):                    │
│  ├── Orders Team (6 engineers)                                   │
│  │   └── order-service, order-processor, order-status-api        │
│  ├── Payments Team (5 engineers)                                 │
│  │   └── payment-service, card-tokenizer, fraud-engine           │
│  ├── Inventory Team (5 engineers)                                │
│  │   └── inventory-service, warehouse-api, stock-reconciler      │
│  ├── Search & Catalog Team (6 engineers)                         │
│  │   └── search-service, catalog-api, recommendation-engine      │
│  ├── User & Auth Team (4 engineers)                              │
│  │   └── user-service, auth-service, profile-api                 │
│  ├── Notifications Team (3 engineers)                            │
│  │   └── notification-service, email-renderer, sms-gateway       │
│  ├── Fulfillment Team (5 engineers)                              │
│  │   └── fulfillment-service, shipping-tracker, returns-api      │
│  └── Marketplace Team (6 engineers)                              │
│      └── seller-portal, listing-service, reviews-api             │
│                                                                  │
│  PLATFORM TEAM (provides self-service infrastructure):           │
│  ├── Platform Engineering (4 engineers) ← YOU                    │
│  │   └── EKS, CI/CD, GitOps, novactl, monitoring infra           │
│  │   └── INTERACTION: X-as-a-Service to all stream teams         │
│  │   └── INTERFACE: novactl CLI, GitOps PRs, Grafana dashboards  │
│  │   └── DO NOT: deploy application code, debug business logic   │
│  │                                                               │
│  └── Data Platform (3 engineers)                                 │
│      └── Kafka, data pipelines, analytics infrastructure         │
│      └── INTERACTION: X-as-a-Service for event streaming         │
│                                                                  │
│  ENABLING TEAM (temporary engagement):                           │
│  └── SRE Enablement (2 engineers, embedded rotation)             │
│      └── Currently embedded with Payments team (PCI migration)   │
│      └── Next: embedded with Marketplace team (scaling issues)   │
│      └── INTERACTION: Facilitating (teach, don't do)             │
│      └── Goal: team is self-sufficient after 4-6 week engagement │
│                                                                  │
│  COMPLICATED SUBSYSTEM:                                          │
│  └── ML/Recommendations (4 engineers)                            │
│      └── recommendation-engine, ML pipeline, model serving       │
│      └── INTERACTION: X-as-a-Service (API) to Search team        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### The Platform Team's Interaction Contract

```
╔══════════════════════════════════════════════════════════════════╗
║  THE PLATFORM TEAM IS NOT AN OPS TEAM.                           ║
║                                                                  ║
║  We provide:                    We do NOT:                       ║
║  ├── Self-service tooling       ├── Deploy your application      ║
║  ├── Golden path templates      ├── Debug your business logic    ║
║  ├── Monitoring infrastructure  ├── Write your Dockerfiles       ║
║  ├── CI/CD pipelines            ├── On-call for your service     ║
║  ├── Documentation + training   ├── Manage your database schema  ║
║  └── Incident support tooling   └── Be the bottleneck for        ║
║                                     every operational task       ║
║                                                                  ║
║  THE CONTRACT:                                                   ║
║  "We make the platform so good that you don't need us."          ║
║  If teams constantly need platform help, the platform is failing.║
║                                                                  ║
║  MEASURE OF SUCCESS:                                             ║
║  ├── % of deploys that need platform team involvement → target 0%║
║  ├── Time to onboard new service → target <1 day                 ║
║  ├── Self-service adoption rate → target >90%                    ║
║  └── Platform NPS from dev teams → target >8/10                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Cognitive Load and Team Boundaries

```
COGNITIVE LOAD TYPES (Team Topologies):

INTRINSIC:    Knowledge needed to do the core job
              "How does order processing work?"
              This is UNAVOIDABLE and VALUABLE.

EXTRANEOUS:   Knowledge needed because the tools are bad
              "How do I configure the Kubernetes Ingress?"
              "Why is my deploy stuck? What's a PDB?"
              This is WASTEFUL. The platform team's job is to 
              ELIMINATE extraneous cognitive load.

GERMANE:      Knowledge gained through learning new approaches
              "How do I use the SLO dashboard to set better alerts?"
              This is INVESTMENT. The enabling team facilitates this.

PLATFORM TEAM'S MISSION:
Minimize extraneous cognitive load for stream-aligned teams.

BEFORE PLATFORM:
  Order team engineer needs to:
  ├── Write Kubernetes YAML (50+ lines per service)
  ├── Understand Kustomize overlays
  ├── Debug HPA scaling behavior
  ├── Configure Prometheus ServiceMonitor
  ├── Set up Grafana dashboards
  ├── Manage secrets rotation
  └── Debug networking issues (NetworkPolicy, DNS)
  
  Total extraneous cognitive load: HIGH
  Time available for order business logic: LOW

AFTER PLATFORM:
  Order team engineer needs to:
  ├── Run: novactl service create order-service --team=orders
  ├── Run: novactl service deploy order-service --version v1.2.3
  ├── Look at: standard Grafana dashboard (auto-generated)
  └── Respond to: clear, actionable alerts with runbook links
  
  Total extraneous cognitive load: MINIMAL
  Time available for order business logic: HIGH
```

---

## 2. WRITING RFCs THAT GET APPROVED

```
╔══════════════════════════════════════════════════════════════════╗
║  RFC = Request For Comments                                      ║
║                                                                  ║
║  An RFC is a PROPOSAL for a significant technical change.        ║
║  It's how you get buy-in BEFORE you write code.                  ║
║                                                                  ║
║  WHY RFCs MATTER:                                                ║
║  ├── Forces you to think before you build                        ║
║  ├── Gets feedback from people who'll be affected                ║
║  ├── Creates a decision record (why we did this)                 ║
║  ├── Prevents "I didn't know about this" surprises               ║
║  └── Distributes technical context across the org                ║
║                                                                  ║
║  WHEN TO WRITE AN RFC:                                           ║
║  ├── New system or major component                               ║
║  ├── Significant architectural change                            ║
║  ├── Cross-team dependency change                                ║
║  ├── Technology adoption/replacement                             ║
║  └── Anything that takes >2 engineer-weeks to build              ║
║                                                                  ║
║  WHEN NOT TO WRITE AN RFC:                                       ║
║  ├── Bug fixes                                                   ║
║  ├── Small improvements within existing architecture             ║
║  ├── Routine maintenance                                         ║
║  └── Anything one person can build in <1 week                    ║
╚══════════════════════════════════════════════════════════════════╝
```

### RFC Template

```markdown
# RFC-XXXX: [Title]

## Metadata
- **Author:** [Name]
- **Status:** Draft | In Review | Accepted | Rejected | Superseded
- **Created:** [Date]
- **Last Updated:** [Date]
- **Reviewers:** [Names — include at least one person from each affected team]
- **Decision Deadline:** [Date — RFCs without deadlines never get decided]

## TL;DR
[2-3 sentences. What are we doing and why? If the reader reads nothing 
else, this should give them the gist.]

## Motivation
[Why are we doing this? What problem does it solve? What pain are we 
currently experiencing? Include data: metrics, incident counts, 
developer survey results, time measurements.]

## Proposal
[What specifically are we building/changing? Be concrete. Include 
diagrams, API contracts, data models. This section should be detailed 
enough that an engineer can start implementing from it.]

### Architecture
[System diagram, component interactions, data flow]

### API / Interface
[How will users interact with this? CLI commands, API endpoints, 
UI mockups]

### Data Model
[Schema, state machines, event definitions]

### Migration
[How do we get from here to there? Phased rollout plan.]

## Trade-offs and Alternatives

### Alternative A: [Name]
[Description, pros, cons, why rejected]

### Alternative B: [Name]
[Description, pros, cons, why rejected]

### What we're giving up
[Every decision has a cost. Be explicit about what we're NOT getting 
with this approach.]

## Risks
[What could go wrong? How do we mitigate each risk?]

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| ... | Low/Med/High | Low/Med/High | ... |

## Dependencies
[What do we need from other teams? What do they need from us?]

## Success Metrics
[How do we know this worked? Specific, measurable.]

| Metric | Current | Target | Measurement Method |
|--------|---------|--------|--------------------|

## Timeline
[Phased delivery plan with milestones]

## Open Questions
[Things you haven't decided yet. These are what the review should focus on.]

1. [Question 1]
2. [Question 2]
```

### RFC Anti-Patterns

```
ANTI-PATTERN 1: "The Novel"
  20-page RFC with every implementation detail.
  Nobody reads it. Nobody reviews it. It gets approved by exhaustion.
  FIX: RFC should be 3-5 pages. Save implementation details for design docs.

ANTI-PATTERN 2: "The Fait Accompli"  
  RFC written AFTER the code is already merged.
  "We built this, now please approve it."
  FIX: RFC before code. If you've already built it, be honest:
  "We prototyped this. Here's what we learned. Should we keep it?"

ANTI-PATTERN 3: "No Alternatives"
  RFC proposes one option with no alternatives considered.
  Readers think: "Did they even evaluate other approaches?"
  FIX: Always include at least 2 alternatives with honest pros/cons.

ANTI-PATTERN 4: "No Deadline"
  RFC sits in "In Review" for 6 weeks. Nobody decides.
  FIX: Set a decision deadline. "If no blocking concerns by [date],
  this RFC is accepted by default." Lazy consensus.

ANTI-PATTERN 5: "The Approval Bottleneck"
  RFC requires approval from 8 people. 2 are on vacation.
  FIX: Require 2-3 explicit approvals + lazy consensus from others.
  "Approved unless you object within 5 business days."
```

---

## 3. PRESENTING TECHNICAL STRATEGY TO EXECUTIVES

```
╔══════════════════════════════════════════════════════════════════╗
║  RULE 1: Executives don't care about technology.                 ║
║          They care about: revenue, cost, risk, speed, customers. ║
║                                                                  ║
║  RULE 2: If your presentation has the word "Kubernetes" on       ║
║          slide 1, you've already lost.                           ║
║                                                                  ║
║  RULE 3: Start with the business outcome. End with the ask.      ║
║          Technology is the middle.                               ║
║                                                                  ║
║  RULE 4: One number per slide. One message per slide.            ║
║          If you need more, add a slide.                          ║
║                                                                  ║
║  RULE 5: Have an ask. "I need $X budget" or "I need Y headcount" ║
║          or "I need executive sponsorship for Z."                ║
║          If you don't ask, you don't get.                        ║
╚══════════════════════════════════════════════════════════════════╝
```

### Executive Presentation Structure (6 slides)

```
SLIDE 1: THE PROBLEM (business language)
═════════════════════════════════════════
  "We deploy twice a week. Our competitors deploy 10 times a day.
   We lose $12K per incident. We have 3 incidents per month.
   It takes 45 minutes to recover. Industry benchmark is 15 minutes."

  ONE NUMBER: $432K/year lost to incidents
  (3 incidents × $12K × 12 months)

SLIDE 2: THE IMPACT ON BUSINESS
═════════════════════════════════
  "Slow deploys mean slow features. Last quarter, we shipped 40% 
   fewer features than planned because deploys were manual and risky.
   
   Marketing can't launch flash sales without 2-day lead time for 
   engineering to pre-scale servers manually."

  ONE NUMBER: 40% feature delivery gap

SLIDE 3: THE SOLUTION (outcome-focused, not technology-focused)
═══════════════════════════════════════════════════════════════════
  "We propose building an internal developer platform that:
   - Lets any team deploy in <10 minutes (currently 4 hours)
   - Auto-scales for flash sales (no manual intervention)
   - Reduces incidents by 50% through automated guardrails
   - Cuts infrastructure costs by 20% through better utilization"
  
  [Small architecture diagram — ONE box per layer, not detailed]
  
  ONE MESSAGE: Self-service platform = faster + cheaper + safer

SLIDE 4: THE INVESTMENT
═════════════════════════
  "Year 1 investment: $910K (4 engineers + tooling + training)
   Year 1 returns: $2.7M (productivity + cost savings + fewer incidents)
   Payback period: 4 months
   Year 2 ROI: 310%"

  [Simple table: Investment vs. Returns]
  
  ONE NUMBER: 192% Year 1 ROI

SLIDE 5: THE PLAN
═══════════════════
  "6-month rollout in 4 phases:
   Phase 1 (Month 1): Foundation — infrastructure, no customer impact
   Phase 2 (Month 2-3): Pilot — one team, prove the model
   Phase 3 (Month 3-5): Scale — all teams onboarded
   Phase 4 (Month 6): Optimize — cost savings realized"

  [Gantt chart — 4 bars, one per phase]
  
  ONE MESSAGE: Incremental delivery, not big-bang risk

SLIDE 6: THE ASK
══════════════════
  "I need:
   1. Budget approval for $300K annual platform tooling
   2. Headcount approval for 1 additional platform engineer
   3. Executive sponsorship to mandate team adoption after Phase 2
   
   Without #3, adoption will be voluntary and slow. With it, we hit 
   ROI targets by Month 8 instead of Month 18."

  ONE MESSAGE: Three things I need. Clear, numbered, actionable.
```

### What Executives Actually Ask (and how to answer)

```
Q: "Why can't we just use AWS managed services and skip the platform team?"
A: "AWS provides building blocks, not solutions. ECS/EKS gives us containers,
   but doesn't give us deployment pipelines, monitoring dashboards, cost 
   attribution, or self-service tooling. Without a platform team, every 
   development team reinvents these. 8 teams × 4 weeks setup = 32 
   engineer-weeks of duplicated effort. The platform team does it once."

Q: "What's the risk if we don't do this?"
A: "Our current incident rate of 3/month will increase proportionally 
   with service count. We're going from 30 to 80 services. Without 
   platform investment, we'll have 8+ incidents/month by year end. 
   At $12K per incident, that's $96K/month in impact — plus the 
   engineering time that isn't building features."

Q: "Can we do this with 2 engineers instead of 4?"
A: "With 2 engineers, we can build the platform but can't onboard teams.
   The platform exists but nobody uses it. We need 2 engineers building 
   + 2 engineers onboarding/supporting. After 6 months, the support load 
   drops as teams become self-sufficient, and all 4 engineers can focus 
   on platform improvements."

Q: "What if we hire a vendor to do this?"
A: "Vendors can accelerate Phase 0-1 (foundation + pilot). Cost: ~$200K 
   for a 3-month engagement. But the ongoing operation, customization, 
   and team onboarding must be in-house. A vendor builds it; we run it.
   I'd recommend a vendor engagement for Phase 0 only if we can't hire 
   the 4th engineer within 6 weeks."
```

---

## 4. VENDOR EVALUATION FRAMEWORK

```
╔══════════════════════════════════════════════════════════════════╗
║  WHEN TO BUY vs BUILD                                            ║
║                                                                  ║
║  BUY when:                    BUILD when:                        ║
║  ├── Problem is well-defined  ├── Problem is unique to you       ║
║  ├── Many vendors exist       ├── Competitive advantage          ║
║  ├── Not a differentiator     ├── Integration overhead > build   ║
║  ├── Team lacks expertise     ├── Vendor lock-in is unacceptable ║
║  └── Time-to-value matters    └── Long-term cost favors build    ║
║                                                                  ║
║  NOVAMART EXAMPLES:                                              ║
║  BUY: Datadog (if budget allowed), PagerDuty, Slack              ║
║  BUILD: novactl (unique to our platform), deployment pipeline     ║
║  HYBRID: Prometheus (open-source) + Grafana Cloud (managed)      ║
╚══════════════════════════════════════════════════════════════════╝
```

### Evaluation Scorecard

```
CATEGORY                          WEIGHT   VENDOR A   VENDOR B   BUILD
═══════════════════════════════════════════════════════════════════════
FUNCTIONAL FIT
  Core feature coverage             15%     9/10       7/10       10/10
  API/integration quality           10%     8/10       6/10       10/10
  Customizability                   10%     5/10       7/10       10/10

OPERATIONAL
  Reliability (SLA)                 10%     9/10       8/10       7/10
  Performance at our scale          10%     8/10       6/10       9/10
  Operational overhead              10%     9/10       7/10       4/10

STRATEGIC
  Vendor lock-in risk               10%     3/10       6/10       10/10
  Data sovereignty/compliance       5%      7/10       5/10       10/10
  Roadmap alignment                 5%      6/10       8/10       10/10

FINANCIAL
  Year 1 TCO                        10%     6/10       7/10       5/10
  Year 3 TCO (at 5x scale)          5%      3/10       5/10       8/10

WEIGHTED SCORE                      100%    6.8        6.5        8.1
═══════════════════════════════════════════════════════════════════════

DECISION: Build (if team has capacity) or Vendor A (if time-to-value critical)
```

---

## 5. TECHNICAL DEBT PRIORITIZATION

```
╔══════════════════════════════════════════════════════════════════╗
║  TECHNICAL DEBT IS NOT BAD CODE.                                 ║
║  Technical debt is the delta between the current system and      ║
║  the system you'd build today with full knowledge.               ║
║                                                                  ║
║  Like financial debt, some tech debt is GOOD (ships faster)      ║
║  and some is BAD (slows everything down).                        ║
║                                                                  ║
║  The question is never "should we pay it off?"                   ║
║  It's "WHICH debt should we pay off FIRST?"                      ║
╚══════════════════════════════════════════════════════════════════╝
```

### Tech Debt Quadrant (Martin Fowler)

```
                    DELIBERATE                INADVERTENT
                ┌─────────────────────┬─────────────────────┐
                │                     │                     │
   PRUDENT      │ "We know this is    │ "Now we know how    │
                │  a shortcut, we'll  │  we should have     │
                │  fix it in sprint 3"│  built it"          │
                │                     │                     │
                │ GOOD DEBT           │ LEARNING DEBT       │
                │ Schedule payoff     │ Refactor when       │
                │                     │ working in the area │
                ├─────────────────────┼─────────────────────┤
                │                     │                     │
   RECKLESS     │ "We don't have time │ "What's a           │
                │  for design"        │  connection pool?"   │
                │                     │                     │
                │ DANGEROUS DEBT      │ IGNORANCE DEBT      │
                │ Fix immediately     │ Train the team      │
                │ if in critical path │                     │
                │                     │                     │
                └─────────────────────┴─────────────────────┘
```

### Prioritization Formula

```
DEBT PRIORITY = (Blast Radius × Frequency of Pain × Severity) / Effort

Where:
  Blast Radius:     How many teams/services affected (1-10)
  Frequency:        How often this causes problems (1-10)
  Severity:         How bad is it when it happens (1-10)
  Effort:           Engineer-weeks to fix (1-20)
```

### NovaMart Tech Debt Registry Example

```
┌────┬──────────────────────────────┬────┬────┬─────┬──────┬───────┬──────────┐
│ ID │ Debt Item                    │ BR │ FQ │ SEV │ EFF  │ SCORE │ ACTION   │
├────┼──────────────────────────────┼────┼────┼─────┼──────┼───────┼──────────┤
│ D1 │ No connection pooling (all   │ 10 │  8 │  9  │  3   │ 240.0 │ FIX NOW  │
│    │ services direct to RDS)      │    │    │     │      │       │          │
├────┼──────────────────────────────┼────┼────┼─────┼──────┼───────┼──────────┤
│ D2 │ Hardcoded secrets in 5       │  5 │  2 │ 10  │  2   │  50.0 │ FIX NOW  │
│    │ legacy services              │    │    │     │      │       │(security)│
├────┼──────────────────────────────┼────┼────┼─────┼──────┼───────┼──────────┤
│ D3 │ No structured logging in     │  8 │  6 │  5  │  4   │  60.0 │ NEXT Q   │
│    │ 12 services                  │    │    │     │      │       │          │
├────┼──────────────────────────────┼────┼────┼─────┼──────┼───────┼──────────┤
│ D4 │ Monolith still handles 20%  │  3 │  3 │  6  │ 12   │   4.5 │ LATER    │
│    │ of order processing          │    │    │     │      │       │          │
├────┼──────────────────────────────┼────┼────┼─────┼──────┼───────┼──────────┤
│ D5 │ Test coverage <30% on        │  4 │  4 │  4  │  8   │   8.0 │ ONGOING  │
│    │ payment-service              │    │    │     │      │       │20%/sprint│
└────┴──────────────────────────────┴────┴────┴─────┴──────┴───────┴──────────┘
```

---

## 6. BUILDING PLATFORM ADOPTION CULTURE

```
╔══════════════════════════════════════════════════════════════════╗
║  You can build the perfect platform. If nobody uses it, it's     ║
║  worthless. Adoption is a PRODUCT problem, not a tech problem.   ║
║                                                                  ║
║  Your platform is a PRODUCT. Your developers are CUSTOMERS.      ║
║  Treat them accordingly.                                         ║
╚══════════════════════════════════════════════════════════════════╝

ADOPTION STRATEGY:

1. START WITH PAIN
   Don't start with "we built a platform."
   Start with "we know deploys take 4 hours and you hate it."
   Solve their #1 pain point first. Then they'll listen.

2. GOLDEN PATH, NOT MANDATE
   "Here's the easy way to do it" > "You MUST do it this way."
   Make the right thing the easy thing.
   If teams go off-path, ask "what's missing from the golden path?"
   — their answer reveals your platform gaps.

3. INTERNAL MARKETING
   ├── Demo day: Monthly 15-min demo of new platform features
   ├── Success stories: "Orders team now deploys 10x/day"
   ├── Metrics dashboard: DORA metrics visible to leadership
   ├── Newsletter: Monthly platform update (what shipped, what's coming)
   └── Swag: Stickers for "I deployed with novactl" (seriously, it works)

4. MEASURE ADOPTION
   ├── % of services on the platform (target: 100%)
   ├── % of deploys via golden path (target: >95%)
   ├── Developer NPS survey quarterly (target: >8/10)
   ├── Time to first deploy for new service (target: <1 day)
   └── Support tickets per month (trending DOWN = platform improving)

5. FEEDBACK LOOPS
   ├── Weekly office hours (drop-in, no agenda)
   ├── Platform channel in Slack (#platform-questions)
   ├── Quarterly platform retrospective (what's working, what's not)
   └── Platform advisory board (1 rep from each stream team)
```

---

## LESSON 2 SUMMARY

```
╔══════════════════════════════════════════════════════════════════╗
║  1. TEAM TOPOLOGIES: Platform team provides self-service.        ║
║     Minimize extraneous cognitive load for stream teams.         ║
║     "If teams constantly need you, the platform is failing."     ║
║                                                                  ║
║  2. RFCs: Write them BEFORE building. Include alternatives.      ║
║     Set a decision deadline. Lazy consensus for approval.        ║
║                                                                  ║
║  3. EXECUTIVE COMMUNICATION: Business outcomes first.            ║
║     One number per slide. Always have a specific ask.            ║
║     "Kubernetes" is not a business outcome.                      ║
║                                                                  ║
║  4. VENDOR EVALUATION: Weighted scorecard. TCO at current AND    ║
║     5x scale. Lock-in risk matters more at scale.                ║
║                                                                  ║
║  5. TECH DEBT: Prioritize by (blast radius × frequency ×         ║
║     severity) / effort. Fix dangerous debt immediately.          ║
║     Schedule prudent debt payoff. Train away ignorance debt.     ║
║                                                                  ║
║  6. ADOPTION: Your platform is a product. Devs are customers.    ║
║     Start with their pain. Make the right thing the easy thing.  ║
║     Measure adoption. Market internally.                         ║
╚══════════════════════════════════════════════════════════════════╝
```

---

# Phase 10, Lesson 2 — Retention Questions

---

**Q1 (RFC — Write It):**

Write a complete RFC for introducing progressive delivery (canary deployments with automated rollback) to NovaMart's deployment pipeline. The current state: all deploys are rolling updates via ArgoCD. The problem: two incidents last quarter were caused by deploys that passed CI but broke in production (one was a configuration error, one was a performance regression that only appeared under real traffic).

Include every section from the template: motivation (with data), proposal (with architecture), alternatives (at least 2), risks, success metrics, timeline, and open questions. This should be ready to send to 8 team leads for review.

---

**Q2 (Executive Presentation):**

NovaMart's CTO has asked you to present a proposal for building an Internal Developer Platform (IDP) — moving from Level 3 to Level 4 on the platform maturity model. You have 15 minutes and 6 slides.

Write the content for all 6 slides (not the slides themselves, but the talking points and the ONE key number/message per slide). Include the expected CTO questions and your prepared answers.

---

**Q3 (Vendor Evaluation):**

NovaMart is evaluating three options for secrets management:
- **Option A:** AWS Secrets Manager + External Secrets Operator (current)
- **Option B:** HashiCorp Vault (self-hosted on EKS)
- **Option C:** HashiCorp Vault (HCP Cloud — managed)

Build the complete evaluation scorecard with weighted scoring across all categories (functional, operational, strategic, financial). Show the Year 1 AND Year 3 cost analysis (at current 30 services AND projected 80 services). Make a recommendation with clear reasoning.

---

**Q4 (Team Dynamics — Scenario):**

The Payments team (5 engineers) has been refusing to adopt the platform for 3 months. They say:
- "We have our own deployment scripts that work fine"
- "Kubernetes is too complex for what we need"
- "We don't trust the platform team to not break our PCI-compliant setup"
- "We've been burned before by centralized tooling that got abandoned"

You need them on the platform — they're the last team, and their resistance is slowing down platform improvements (you can't deprecate the old deployment path while they're on it).

Design your strategy. What specific actions do you take over the next 8 weeks? How do you address each of their four objections? What's your escalation path if persuasion fails? What's the one thing you should NOT do?

# Phase 10, Lesson 2 — Retention Answers

---

# Q1 — RFC: Progressive Delivery at NovaMart

```markdown
# RFC-0042: Progressive Delivery with Automated Canary Analysis

## Metadata
- **Author:** [Your Name], Platform Engineering
- **Status:** In Review
- **Created:** 2024-11-15
- **Last Updated:** 2024-11-15
- **Reviewers:** 
  - Orders Team Lead
  - Payments Team Lead
  - Search Team Lead
  - Inventory Team Lead
  - SRE Lead
  - Platform Engineering (all)
- **Decision Deadline:** 2024-12-02 (lazy consensus: approved if no 
  blocking concerns by this date)

## TL;DR

Replace all-at-once rolling updates with automated canary deployments 
for production services. New versions will serve 5% of traffic first, 
be automatically validated against error rate, latency, and business 
metrics for 10 minutes, then progressively promoted to 100%. If metrics 
degrade, the canary is automatically rolled back with zero human 
intervention. This eliminates the class of incidents where deploys pass 
CI but fail under real production traffic.

## Motivation

### The Problem

In Q3 2024, we had **two production incidents directly caused by 
deployments** that passed all CI checks:

**Incident INC-2024-087 (Sept 12): Configuration regression**
- payment-service deployed with a misconfigured connection timeout
- All unit/integration tests passed (used mock connections)
- Rolling update replaced 100% of pods within 3 minutes
- Error rate spiked to 15% across all payment processing
- MTTR: 22 minutes (manual rollback + ArgoCD revert)
- Business impact: ~$18,000 in failed transactions
- Root cause: configuration error only visible under real DB load

**Incident INC-2024-103 (Oct 28): Performance regression**
- order-service deployed with an N+1 query introduced in a refactor
- Load tests in CI use synthetic data (500 records)
- Production database has 12M records → query took 4.2s instead of 50ms
- p99 latency spiked from 200ms to 5,000ms
- MTTR: 35 minutes (took time to identify the deploy as root cause)
- Business impact: ~$31,000 in abandoned carts during peak hours
- Root cause: performance regression only visible at production data scale

**Combined Q3 impact: $49,000 in direct revenue loss + 57 minutes MTTR**

### Why Rolling Updates Are Insufficient

Rolling updates have a binary outcome: the new version either replaces 
the old version completely, or doesn't deploy at all (if pods fail 
readiness probes). There is no middle ground.

```
Rolling Update:
  Old ████████████ → Old ████████ New ████ → New ████████████
                     ALL traffic shifts within 2-5 minutes
                     No validation against production metrics
                     If it passes readiness probe → it's live for everyone
```

The readiness probe checks "can this pod serve HTTP?" — not "is this 
pod performing well under real production traffic patterns?"

### What We Need

A deployment strategy that:
1. Exposes new code to a **small subset** of real traffic first
2. **Automatically validates** the new version against production metrics
3. **Automatically rolls back** if metrics degrade
4. Progressively increases traffic to the new version only after validation
5. Requires **zero human intervention** for both promotion and rollback

## Proposal

### Architecture

Deploy **Argo Rollouts** alongside our existing ArgoCD installation. 
Replace `Deployment` resources with `Rollout` resources for production 
services. Use NGINX Ingress canary annotations for traffic splitting 
and Prometheus metrics for automated analysis.

```
                    ┌─────────────────────────────┐
                    │       NGINX Ingress          │
                    │  (traffic splitting layer)   │
                    └──────────┬──────────────────┘
                               │
                    ┌──────────┴──────────────┐
                    │                         │
              95% traffic                5% traffic
                    │                         │
                    ▼                         ▼
          ┌─────────────────┐      ┌─────────────────┐
          │  Stable Service │      │  Canary Service  │
          │  (current ver)  │      │  (new version)   │
          │  order-svc:v1.1 │      │  order-svc:v1.2  │
          │  Replicas: 10   │      │  Replicas: 1     │
          └─────────────────┘      └─────────────────┘
                    │                         │
                    │    ┌────────────┐       │
                    └───►│ Prometheus │◄──────┘
                         │  metrics   │
                         └─────┬──────┘
                               │
                    ┌──────────┴──────────┐
                    │  Argo Rollouts      │
                    │  AnalysisRun        │
                    │                     │
                    │  Every 60s, query:  │
                    │  ├─ Error rate      │
                    │  ├─ p99 latency     │
                    │  ├─ Success rate    │
                    │  └─ Business metric │
                    │                     │
                    │  IF degraded:       │
                    │    → Auto rollback  │
                    │  IF healthy:        │
                    │    → Promote to     │
                    │      next step      │
                    └─────────────────────┘
```

### Canary Progression Strategy

```
Step 1:  5% traffic → canary (1 replica)
         Analysis: 10 minutes, check every 60 seconds
         Gate: error_rate < baseline + 0.5%, p99 < baseline × 1.2

Step 2:  20% traffic → canary (3 replicas)
         Analysis: 10 minutes
         Gate: same thresholds

Step 3:  50% traffic → canary (5 replicas)
         Analysis: 10 minutes
         Gate: same thresholds

Step 4:  100% traffic → canary becomes stable
         Old version scaled to 0
         Total canary duration: ~40 minutes

FAILURE AT ANY STEP:
  → Canary traffic set to 0% immediately (<5 seconds)
  → Canary pods terminated
  → Stable version continues serving 100% traffic
  → Slack notification sent with analysis report
  → DORA change_failure_rate incremented
  → NO HUMAN INTERVENTION REQUIRED
```

### Rollout Resource Definition

```yaml
# base/rollout.yaml (replaces deployment.yaml)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
  labels:
    app: order-service
    team: orders
spec:
  replicas: 10
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    canary:
      canaryService: order-service-canary
      stableService: order-service-stable
      trafficRouting:
        nginx:
          stableIngress: order-service
          annotationPrefix: nginx.ingress.kubernetes.io
      steps:
        # Step 1: 5% canary, validate for 10 minutes
        - setWeight: 5
        - analysis:
            templates:
              - templateName: canary-analysis
            args:
              - name: service-name
                value: order-service
              - name: namespace
                value: production-orders
        
        # Step 2: 20% canary
        - setWeight: 20
        - analysis:
            templates:
              - templateName: canary-analysis
            args:
              - name: service-name
                value: order-service
              - name: namespace
                value: production-orders
        
        # Step 3: 50% canary
        - setWeight: 50
        - analysis:
            templates:
              - templateName: canary-analysis
            args:
              - name: service-name
                value: order-service
              - name: namespace
                value: production-orders
        
        # Step 4: full promotion
        - setWeight: 100
      
      # Automatic rollback on failure
      abortScaleDownDelaySeconds: 30
      
      # Anti-affinity: canary and stable on different nodes
      antiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          weight: 100
```

### Analysis Template (Prometheus-based)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: canary-analysis
spec:
  args:
    - name: service-name
    - name: namespace
  metrics:
    # Metric 1: Error rate comparison (canary vs stable)
    - name: error-rate
      interval: 60s
      count: 10          # 10 checks × 60s = 10 minutes
      failureLimit: 2     # Allow 2 failures before aborting
      successCondition: "result[0] < 0.05"  # <5% error rate
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{
              app="{{args.service-name}}",
              namespace="{{args.namespace}}",
              rollouts_pod_template_hash="{{templates.podTemplateHash}}",
              code=~"5.."
            }[2m]))
            /
            sum(rate(http_requests_total{
              app="{{args.service-name}}",
              namespace="{{args.namespace}}",
              rollouts_pod_template_hash="{{templates.podTemplateHash}}"
            }[2m]))

    # Metric 2: p99 latency comparison
    - name: latency-p99
      interval: 60s
      count: 10
      failureLimit: 2
      successCondition: "result[0] < 0.500"  # <500ms p99
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{
                app="{{args.service-name}}",
                namespace="{{args.namespace}}",
                rollouts_pod_template_hash="{{templates.podTemplateHash}}"
              }[2m])) by (le)
            )

    # Metric 3: Saturation — CPU not spiking
    - name: cpu-usage
      interval: 60s
      count: 10
      failureLimit: 3
      successCondition: "result[0] < 0.80"  # <80% CPU
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            avg(rate(container_cpu_usage_seconds_total{
              pod=~"{{args.service-name}}-.*",
              namespace="{{args.namespace}}",
              container="{{args.service-name}}"
            }[2m]))
            /
            avg(kube_pod_container_resource_requests{
              pod=~"{{args.service-name}}-.*",
              namespace="{{args.namespace}}",
              resource="cpu"
            })
```

### Integration with Existing Pipeline

```
CURRENT FLOW (unchanged):
  Code push → GitHub Actions (build, test, scan) → ECR push → 
  GitOps commit (image tag update) → ArgoCD sync

WHAT CHANGES:
  ArgoCD syncs a Rollout resource instead of a Deployment resource.
  ArgoCD detects the Rollout controller and delegates progression 
  to Argo Rollouts. No changes to CI pipeline, GitOps repo structure,
  or developer workflow.

DEVELOPER EXPERIENCE:
  BEFORE: `novactl service deploy order-service --version v1.2.3`
          → ArgoCD syncs → rolling update → done in 3 minutes
  
  AFTER:  `novactl service deploy order-service --version v1.2.3`
          → ArgoCD syncs → canary starts → analysis runs → 
          → promoted in 40 minutes (or rolled back automatically)

  THE DEPLOY COMMAND DOESN'T CHANGE.
  The deploy just takes longer (40 min vs 3 min) and is safer.

  New CLI output:
  $ novactl service deploy order-service --version v1.2.3
  ✅ Pre-checks passed
  ✅ GitOps commit: abc123
  🔄 Canary started: 5% traffic
     ├── Analysis: error-rate ✅ (0.2%)
     ├── Analysis: latency-p99 ✅ (145ms)
     └── Analysis: cpu-usage ✅ (42%)
  🔄 Promoting: 20% traffic
     ├── Analysis: error-rate ✅ (0.3%)
     ...
  ✅ Deploy complete: 100% traffic on v1.2.3 (41m 23s)
```

### Emergency Bypass

```
For critical hotfixes that cannot wait 40 minutes:

  novactl service deploy order-service --version v1.2.4-hotfix --skip-canary

This sets the Rollout to "skip all steps" and immediately promotes.
Requires: --skip-canary flag is gated behind a confirmation prompt:

  ⚠️  WARNING: Skipping canary analysis for production deploy.
  ⚠️  This bypasses automated safety checks.
  ⚠️  Type "I ACCEPT THE RISK" to proceed:

Audit: Skip events are logged and Slack-notified to #platform-alerts.
Policy: Skip requires post-deploy review within 24 hours (automated 
Jira ticket created).
```

### Migration Plan (Rollout to Rollout)

```
We cannot convert all 30 services from Deployment to Rollout at once.
Phased rollout of the progressive delivery capability itself:

PHASE 1 (Week 1-2): Platform setup
  ├── Deploy Argo Rollouts controller on EKS
  ├── Deploy AnalysisTemplate resources
  ├── Update novactl to display canary progress
  └── Update Grafana dashboard with canary vs stable comparison

PHASE 2 (Week 3-4): Pilot — 2 non-critical services
  ├── notification-service (low risk, validates the flow)
  ├── catalog-service (read-heavy, validates latency analysis)
  ├── Both teams pair with platform team during first deploy
  └── SUCCESS: 3+ successful canary deploys per service

PHASE 3 (Week 5-8): Core services
  ├── order-service, payment-service, inventory-service
  ├── PCI services get additional analysis metrics (PCI audit events)
  └── SUCCESS: All Tier-1 services using canary

PHASE 4 (Week 9-12): All remaining services
  ├── Remaining 25 services converted
  ├── Golden path template updated (novactl service create generates 
  │   Rollout instead of Deployment)
  └── SUCCESS: 100% of production services use progressive delivery

ROLLBACK PATH: 
  If Argo Rollouts has issues, converting a Rollout back to a 
  Deployment is a one-line Kustomize patch (change apiVersion + kind).
  ArgoCD handles the resource type change seamlessly.
```

## Trade-offs and Alternatives

### Alternative A: Blue-Green Deployments

**Description:** Maintain two full environments (blue and green). Deploy 
to inactive environment, validate, then switch traffic all at once.

**Pros:**
- Simple mental model — traffic is either on blue or green
- Instant rollback (switch back to previous environment)
- Full environment validation before any traffic shift

**Cons:**
- 2x resource cost during deployment (full replica set for both versions)
- For 30 services with avg 10 replicas = 300 extra pods during every deploy
- At NovaMart scale: ~$2,000/month additional compute just for deployment overhead
- No gradual traffic shift — validation is binary (works or doesn't), 
  doesn't catch issues that only appear at partial load
- Would NOT have caught INC-2024-103 (performance regression) because 
  blue-green validates with synthetic traffic, not real user traffic

**Why rejected:** Higher cost, doesn't solve the core problem (validating 
under real traffic), and canary provides a superset of blue-green's 
benefits with gradual rollout.

### Alternative B: Feature Flags (LaunchDarkly / Flagsmith)

**Description:** Deploy code with features behind flags. Enable features 
gradually via flag management platform.

**Pros:**
- Decouples deploy from release (deploy anytime, enable when ready)
- Granular control (enable per user, per region, per percentage)
- No infrastructure changes needed
- Can target specific user segments for validation

**Cons:**
- Doesn't solve infrastructure/performance regressions 
  (the code is deployed and running regardless of flag state)
- Would NOT have caught INC-2024-087 (config regression) because 
  the misconfigured timeout affected all code paths, not a specific feature
- Adds code complexity (if/else branches everywhere)
- Requires a feature flag service ($15K-50K/year for LaunchDarkly)
- Technical debt: old flags accumulate if not cleaned up
- PCI compliance concern: feature flag service becomes part of CDE 
  if it controls payment processing behavior

**Why rejected:** Feature flags solve a different problem (feature release 
management, not deployment safety). They complement canary deployments 
but don't replace them. We may adopt feature flags later for gradual 
feature releases, but infrastructure-level canary is the priority.

### Alternative C: Manual Canary (No Automation)

**Description:** Deploy canary pod manually, watch dashboards, promote 
or rollback manually.

**Pros:**
- No new tooling required (kubectl + Grafana)
- Human judgment for complex scenarios
- Zero learning curve

**Cons:**
- Requires an engineer watching dashboards for 40 minutes per deploy
- At target deploy frequency (10+/day across 30 services): impossible
- Human error: "it looks fine" → promote → it wasn't fine
- 3 AM deploys: nobody is watching
- Doesn't scale

**Why rejected:** Our target is multiple deploys per day. Manual canary 
doesn't scale beyond 2-3 deploys/day, and introduces human error in 
the one place we're trying to eliminate it.

### What We're Giving Up

1. **Deploy speed:** Deploys take 40 minutes instead of 3 minutes. For 
   urgent hotfixes, we provide the `--skip-canary` escape hatch.
2. **Simplicity:** Rollout resources are more complex than Deployments. 
   Teams need to understand the canary progression model (mitigated by: 
   novactl abstracts the complexity, teams don't edit Rollout YAML directly).
3. **Resource overhead during canary:** Canary replicas consume extra 
   resources during the 40-minute analysis window. Impact: ~10% additional 
   compute during deploy windows. Acceptable.

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Argo Rollouts controller failure blocks all deploys | Low | Critical | HA deployment (3 replicas). Break-glass: convert Rollout→Deployment via Kustomize patch. |
| False positive: canary rolls back a healthy deploy | Medium | Medium | Tune thresholds during Phase 2. Start with loose thresholds, tighten based on data. `failureLimit: 2` allows transient spikes. |
| False negative: canary promotes an unhealthy deploy | Low | High | Multiple metric types (error rate + latency + CPU). Business metric (order completion rate) catches application-level issues. |
| Low-traffic services don't generate enough metrics | Medium | Medium | Minimum traffic threshold in analysis. For services <100 RPS, extend analysis window to 20 minutes. Fallback: manual approval step. |
| Teams resist slower deploys | Medium | Low | Demo the value with Q3 incident data. Skip-canary for emergencies. Deploy speed is the trade-off for deploy safety — make this explicit. |
| Analysis template PromQL queries are wrong | Medium | High | Test every query against production data before enabling. Unit tests for PromQL queries using Prometheus recording rules. |

## Dependencies

| Dependency | Owner | Status | Needed By |
|-----------|-------|--------|-----------|
| Argo Rollouts Helm chart deployed | Platform Team | Not started | Week 2 |
| NGINX Ingress canary annotations enabled | Platform Team | Available | Week 1 |
| Prometheus metrics available per-revision | Platform Team | Available (rollouts_pod_template_hash label) | Week 1 |
| Service teams update to Rollout resources | Each team (with platform support) | Not started | Phased, weeks 3-12 |
| novactl updated with canary status display | Platform Team | Not started | Week 3 |
| Grafana dashboard: canary vs stable comparison | Platform Team | Not started | Week 2 |

## Success Metrics

| Metric | Current | Target | Measurement | Timeline |
|--------|---------|--------|-------------|----------|
| Deploy-caused incidents per quarter | 2 | 0 | PagerDuty + postmortem tag | Q1 after full rollout |
| Change failure rate (DORA) | 8% | <2% | Argo Rollouts abort rate | Q1 after full rollout |
| Automated rollback success rate | N/A | >95% | Argo Rollouts metrics | Ongoing |
| Mean time to detect bad deploy | 22-35 min | <10 min (automated) | Canary analysis duration | Immediate |
| Mean time to rollback bad deploy | 22-35 min | <1 min (automated) | Argo Rollouts metrics | Immediate |
| False positive rate | N/A | <5% | Manual review of rollbacks | Monthly review |
| Developer satisfaction with deploy process | 6/10 (survey) | >8/10 | Quarterly survey | 6 months |

## Timeline

```
Week 1-2:   Platform setup (Argo Rollouts, AnalysisTemplates, dashboards)
Week 3-4:   Pilot (notification-service, catalog-service)
Week 5-6:   Core Tier-1 services (order, payment, inventory)
Week 7-8:   Remaining Tier-1 + Tier-2 services
Week 9-12:  All remaining services + golden path template update
Week 13+:   Continuous threshold tuning based on false positive/negative data
```

## Open Questions

1. **Should we make canary mandatory or opt-in?**
   Proposal: Mandatory for Tier-1 (critical) services, opt-in for Tier-2/3 
   during Phase 2-3, mandatory for all by Phase 4. Open to feedback.

2. **What business metric should we include in the analysis?**
   Options: order completion rate (orders team), payment success rate 
   (payments team), search result click-through (search team). Each team 
   should define their own business metric. Platform provides the template; 
   teams fill in the PromQL query.

3. **How do we handle database migrations during canary?**
   Database schema changes must be backward-compatible (expand-and-contract 
   pattern). Canary runs new code against old schema → must work. This is 
   already our policy but needs enforcement. Should we add a pre-check for 
   this?

4. **Should canary analysis run during off-peak hours?**
   Low traffic = unreliable metrics. Options: (a) extend analysis window 
   at night, (b) pause canary overnight and resume in morning, (c) deploy 
   only during business hours. Feedback welcome.

5. **What's the right threshold for auto-rollback?**
   Starting proposal: error_rate < baseline + 0.5%, p99 < baseline × 1.2.
   These are intentionally loose. We'll tighten based on false positive 
   data after 1 month.
```

---

# Q2 — Executive Presentation: Internal Developer Platform

## Slide 1: THE PROBLEM

**Talking points:**

"Our developers spend 30% of their time on infrastructure tasks instead of building features. That's the equivalent of losing 45 engineers' worth of productivity out of our 150 engineering headcount. When a developer wants to launch a new service, it takes 3 weeks — 2 days of coding and 19 days waiting for infrastructure, CI/CD setup, monitoring configuration, and security reviews."

"Meanwhile, our competitors are launching new features weekly. We're shipping monthly."

**ONE NUMBER:** 30% of engineering time lost to infrastructure friction
(150 engineers × 30% = 45 full-time-equivalent engineers doing ops instead of product)

---

## Slide 2: WHAT SELF-SERVICE LOOKS LIKE

**Talking points:**

"Imagine a developer wants to launch a new microservice. Today: 3 weeks of tickets, meetings, and manual configuration. With an Internal Developer Platform, they run one command: `novactl service create my-service --team=orders` — and in 10 minutes they have a fully configured service with CI/CD, monitoring, dashboards, alerts, security policies, and documentation. All following our standards automatically."

"This isn't a fantasy. This is what Spotify, Netflix, and Shopify have built. It's what our engineers expect when they interview here."

**ONE MESSAGE:** From 3 weeks to 10 minutes for new service creation

---

## Slide 3: THE IMPACT ON BUSINESS METRICS

**Talking points:**

"This directly impacts four business outcomes:

First, **speed to market**: deploy frequency increases from 2/week to 10+/day. Each feature reaches customers days sooner — that's revenue we're leaving on the table.

Second, **reliability**: our current MTTR is 45 minutes. Automated guardrails and self-healing bring it to under 15 minutes. That's $36K less revenue lost per incident.

Third, **engineering retention**: we lost 3 senior engineers last quarter. In exit interviews, two cited 'outdated infrastructure' as a factor. At $150K recruiting cost per senior hire, that's $450K in replacement costs.

Fourth, **compliance**: PCI audit prep currently takes 2 engineers × 2 weeks per quarter. Automated compliance reduces that to 2 days — freeing an entire engineer-month per quarter."

**ONE NUMBER:** $2.7M annual return across productivity, reliability, retention, and compliance

---

## Slide 4: THE INVESTMENT

**Talking points:**

"We need three things:

One additional platform engineer — bringing our team to 5. Total team cost: $950K/year fully loaded.

$150K for tooling and managed services — Backstage license, additional compute for platform services, training.

The total investment: $1.1M in Year 1.

Against $2.7M in quantified returns, that's a 145% ROI with a 5-month payback period. Year 2 ROI exceeds 300% because the investment stays flat while the returns compound with team growth."

**ONE NUMBER:** 145% ROI, 5-month payback

*[Simple table on slide: Investment $1.1M | Returns $2.7M | ROI 145%]*

---

## Slide 5: THE PLAN

**Talking points:**

"Six-month delivery in three phases. No big-bang risk.

Phase 1 — Months 1-2: Build the developer portal and golden path templates. Internal launch to 2 volunteer teams. Measure: can they create and deploy a service in under 1 hour?

Phase 2 — Months 3-4: Onboard all 8 teams. Self-service for 80% of operational tasks. Measure: platform team involvement in deploys drops to near-zero.

Phase 3 — Months 5-6: Advanced capabilities — automated compliance checks, cost attribution dashboards, progressive delivery. Measure: DORA metrics at 'Elite' level.

Each phase has a clear success gate. If Phase 1 doesn't prove the value, we stop and reassess. Your total exposure before Phase 2 is $200K."

**ONE MESSAGE:** Incremental delivery with an exit ramp after Phase 1 — $200K maximum exposure before we commit fully

---

## Slide 6: THE ASK

**Talking points:**

"I need three things from you:

One — headcount approval for 1 additional platform engineer. We have a strong candidate in the pipeline who can start in 4 weeks.

Two — $150K annual budget for platform tooling, approved through procurement.

Three — and this is the most important one — your sponsorship for platform adoption. After Phase 1 proves the value, I need your support to mandate that all new services use the golden path. Without this, adoption will be voluntary and slow. With it, we hit ROI targets by Month 8 instead of Month 18.

History shows that successful platform teams have executive sponsorship. Without it, they become 'optional tooling that nobody uses.'"

**ONE MESSAGE:** Three things: 1 headcount, $150K budget, your sponsorship for adoption

---

## Expected CTO Questions and Prepared Answers

```
Q: "Why can't development teams just build their own tooling?"

A: "They can, and some do. The Payments team has their own deploy scripts.
   The Search team built a custom monitoring setup. The problem is: 8 teams 
   × custom tooling = 8 different ways to deploy, monitor, and debug. When 
   there's an incident, the on-call engineer needs to understand each team's 
   custom setup. That's why our MTTR is 45 minutes — it's not the fix that 
   takes long, it's figuring out how THIS team's infrastructure works.

   A platform team builds it once, correctly, for everyone. 8 teams stop 
   reinventing the wheel and spend that time on features instead."


Q: "What if we just buy a PaaS like Heroku or use AWS App Runner?"

A: "For a 10-service startup, that's the right call. At our scale — 30 
   services, PCI compliance, 150 engineers — PaaS platforms hit limitations 
   quickly. No VPC control for PCI segmentation. No custom autoscaling 
   policies. No integration with our existing observability stack. And the 
   cost: AWS App Runner at our scale would be 3-4x what we pay for EKS.

   The IDP isn't replacing our infrastructure — it's making our existing 
   infrastructure self-service."


Q: "How do we know this won't be abandoned like the last platform initiative?"

A: "Fair concern. The 2022 'Platform v1' failed for three reasons I've 
   analyzed: no dedicated team (it was a side project), no executive sponsor,
   and no user research (they built what they thought devs needed, not what 
   devs actually needed).

   We're addressing all three: dedicated 5-person team, your sponsorship 
   (that's my ask), and we're starting with developer pain surveys — we 
   already know the top 3 pain points. Phase 1 proves value before we 
   commit to Phase 2."


Q: "Can we do this with 3 platform engineers instead of 5?"

A: "With 3, we can maintain the current platform but can't build the IDP.
   We'd be in maintenance mode — fixing issues and handling requests from 
   8 teams. Zero capacity for new capabilities.

   With 4 (current), we can build the IDP but onboarding will be slow — 
   we'd dedicate 2 to building and 2 to support. Timeline extends to 
   9-12 months.

   With 5, we hit the 6-month timeline. The 5th engineer pays for 
   themselves in 3 months through developer productivity gains."


Q: "What's the risk if we wait 6 months to start?"

A: "Three compounding risks:

   First, we're growing from 30 to 50 services in the next 6 months. 
   Each new service without a golden path adds to the inconsistency 
   problem. Retrofitting is 3x harder than building correctly from the start.

   Second, we're about to hire 20 engineers in Q1. Without an IDP, each 
   new hire spends their first 2 weeks figuring out our deployment process 
   instead of shipping code. That's 40 engineer-weeks wasted on onboarding.

   Third, our PCI auditor flagged 'inconsistent security configurations 
   across services' in last quarter's review. An IDP with policy 
   enforcement addresses this before the next audit in March."


Q: "What does success look like in 6 months? How do I know it worked?"

A: "Four metrics, measured monthly, reported to you:

   1. Time to deploy: from 4 hours to under 30 minutes
   2. Time to create new service: from 3 weeks to under 1 day
   3. MTTR: from 45 minutes to under 15 minutes  
   4. Developer NPS for platform: from 4/10 (current survey) to 8/10

   I'll send you a monthly dashboard with these four numbers. If any 
   metric isn't trending in the right direction by Month 3, we'll 
   course-correct together."
```

---

# Q3 — Vendor Evaluation: Secrets Management

## Option Descriptions

| | Option A: AWS SM + ESO | Option B: Vault Self-Hosted | Option C: Vault HCP Cloud |
|---|---|---|---|
| **What** | AWS Secrets Manager storing secrets, External Secrets Operator syncing to K8s | HashiCorp Vault deployed on EKS, managed by platform team | HashiCorp Cloud Platform Vault — fully managed by HashiCorp |
| **Operational model** | AWS manages SM, we manage ESO | We manage everything: storage, HA, unsealing, upgrades | HashiCorp manages Vault, we configure and use |
| **Secret types** | Key-value, DB credentials, API keys | Key-value, DB creds, PKI/TLS, dynamic secrets, transit encryption | Same as B, fully managed |

## Evaluation Scorecard

```
CATEGORY                          WEIGHT   OPTION A    OPTION B     OPTION C
                                           AWS SM+ESO  Vault Self   Vault HCP
═══════════════════════════════════════════════════════════════════════════════

FUNCTIONAL FIT
──────────────
Core secret storage + retrieval    5%      9/10        10/10        10/10
Dynamic secrets (DB creds on       8%      3/10        10/10        10/10
  demand, auto-rotating)                   (SM rotation  (native     (native
                                           is limited)   feature)    feature)
PKI / TLS certificate issuance     5%      2/10        10/10        10/10
                                           (use cert-    (PKI engine  (PKI engine
                                           manager       built-in)    built-in)
                                           separately)
Transit encryption (encrypt        3%      1/10        10/10        10/10
  data without exposing keys)              (not avail)  (native)     (native)
K8s integration quality            5%      9/10        8/10         8/10
                                           (ESO is      (CSI driver  (CSI driver
                                           excellent)    + agent)     + agent)
Multi-region / DR                  4%      9/10        5/10         8/10
                                           (native      (manual      (managed
                                           replication)  replication) replication)

FUNCTIONAL SUBTOTAL               30%      5.3         8.8          9.1

OPERATIONAL
──────────────
Operational overhead               12%     9/10        3/10         8/10
                                           (AWS manages (HA cluster,  (HCP manages
                                           SM, ESO is   unseal,       everything)
                                           lightweight) upgrades,
                                                        backup)
HA / reliability                   8%      9/10        6/10         9/10
                                           (AWS SLA     (depends on   (HCP SLA
                                           99.99%)      our ops)      99.95%)
Upgrade / patching                 5%      9/10        4/10         9/10
                                           (AWS manages (manual,      (HCP manages
                                           SM upgrades) disruptive)   upgrades)
Disaster recovery                  5%      9/10        5/10         8/10
                                           (native      (manual       (managed
                                           cross-region) snapshots)   snapshots)

OPERATIONAL SUBTOTAL              30%      9.0         4.2          8.5

STRATEGIC
──────────────
Vendor lock-in risk                5%      4/10        9/10         7/10
                                           (AWS SM API  (open-source  (Vault API
                                           is unique)   core, own     but HCP-
                                                        data)         managed)
PCI-DSS compliance                 5%      8/10        9/10         9/10
                                           (SM has PCI  (Vault has    (Vault +
                                           attestation) audit log,    HCP has SOC2
                                                        namespaces)   + PCI)
Ecosystem / community              3%      7/10        10/10        10/10
                                           (AWS docs)   (massive      (same as B)
                                                        community)
Team expertise required            2%      8/10        4/10         7/10
                                           (our team    (Vault admin  (less ops,
                                           knows AWS)   is a skill)   still config)

STRATEGIC SUBTOTAL                15%      6.3         8.1          8.1

FINANCIAL (detailed below)
──────────────
Year 1 TCO (30 services)          12%     8/10        4/10         6/10
Year 3 TCO (80 services)           8%     6/10        5/10         5/10
Cost predictability                5%     7/10        8/10         6/10
                                          (per-secret  (fixed infra  (per-secret
                                           pricing)     cost)         pricing)

FINANCIAL SUBTOTAL                25%      7.0         5.3          5.6

═══════════════════════════════════════════════════════════════════════════════
WEIGHTED TOTAL                    100%     6.8         6.1          7.7
═══════════════════════════════════════════════════════════════════════════════
```

## Cost Analysis

### Year 1: 30 Services, ~200 Secrets

```
OPTION A: AWS Secrets Manager + External Secrets Operator
─────────────────────────────────────────────────────────
Secrets Manager:
  200 secrets × $0.40/secret/month              = $960/year
  API calls: 30 services × 6 pods avg × 
    4 syncs/hour × 24h × 365d = ~1.6M calls
    1.6M × $0.05/10,000                        = $8/year
ESO (runs on EKS):
  3 pods × 128Mi RAM = trivial compute          = ~$50/year
Operational cost:
  ~2% of 1 platform engineer's time             = ~$3,600/year
  (ESO maintenance, SM config)
──────────────────────────────────────────────────────────
YEAR 1 TOTAL:                                    ~$4,618/year

OPTION B: HashiCorp Vault Self-Hosted on EKS
─────────────────────────────────────────────
Compute:
  3 Vault pods (HA) × 2 vCPU, 4GB RAM each
  On m5.large equivalent: ~$300/month           = $3,600/year
  Consul backend (3 pods): ~$150/month          = $1,800/year
Storage:
  EBS volumes for Consul: 3 × 50GB × $0.10/GB  = $180/year
Operational cost:
  ~20% of 1 platform engineer's time            = ~$36,000/year
  (HA management, unsealing, upgrades, backup,
   troubleshooting, policy management)
Training:
  Vault certification for 2 engineers           = $3,000/year
──────────────────────────────────────────────────────────
YEAR 1 TOTAL:                                    ~$44,580/year

OPTION C: HashiCorp Vault HCP Cloud
────────────────────────────────────
HCP Vault (Standard tier):
  $1.58/hr for small cluster                    = $13,841/year
  (includes HA, auto-unseal, managed upgrades)
Operational cost:
  ~5% of 1 platform engineer's time             = ~$9,000/year
  (policy configuration, onboarding, monitoring)
Training:
  Vault basics for 2 engineers                  = $1,500/year
──────────────────────────────────────────────────────────
YEAR 1 TOTAL:                                    ~$24,341/year
```

### Year 3: 80 Services, ~600 Secrets, 5x Traffic

```
OPTION A: AWS SM + ESO
──────────────────────
Secrets Manager:
  600 secrets × $0.40                           = $2,880/year
  API calls (5x): ~8M × $0.05/10K              = $40/year
ESO + operational:                              = ~$4,000/year
──────────────────────────────────────────────────────────
YEAR 3 TOTAL:                                    ~$6,920/year

OPTION B: Vault Self-Hosted
───────────────────────────
Compute (same HA cluster handles 80 svcs):      = $5,400/year
  (may need larger instances at high throughput)
Operational (increases with complexity):         = ~$45,000/year
  (more policies, more namespaces, more audit)
──────────────────────────────────────────────────────────
YEAR 3 TOTAL:                                    ~$50,400/year

OPTION C: Vault HCP Cloud
─────────────────────────
HCP Vault (Plus tier for 80 services):
  $3.16/hr (higher tier for throughput)         = $27,682/year
Operational:                                     = ~$12,000/year
──────────────────────────────────────────────────────────
YEAR 3 TOTAL:                                    ~$39,682/year
```

### 3-Year TCO Comparison

```
┌──────────────────┬───────────┬───────────┬───────────┐
│                  │ Option A  │ Option B  │ Option C  │
│                  │ AWS SM    │ Vault Self│ Vault HCP │
├──────────────────┼───────────┼───────────┼───────────┤
│ Year 1           │  $4,618   │ $44,580   │ $24,341   │
│ Year 2           │  $5,800   │ $47,000   │ $32,000   │
│ Year 3           │  $6,920   │ $50,400   │ $39,682   │
├──────────────────┼───────────┼───────────┼───────────┤
│ 3-YEAR TOTAL     │ $17,338   │$141,980   │ $96,023   │
├──────────────────┼───────────┼───────────┼───────────┤
│ Dynamic secrets  │    ❌    │    ✅     │    ✅     │
│ PKI engine       │    ❌    │    ✅     │    ✅     │
│ Transit encrypt  │    ❌    │    ✅     │    ✅     │
│ Ops overhead     │   Low     │  Very High│  Medium   │
│ Team expertise   │  Exists   │ Must Build│ Must Build│
└──────────────────┴───────────┴───────────┴───────────┘
```

## Recommendation: **Stay with Option A (AWS SM + ESO) — with a re-evaluation trigger**

### Reasoning

**Option A wins on the two metrics that matter most for a 4-person platform team: operational overhead and cost.**

At $4,618/year vs $24,341 (HCP) or $44,580 (self-hosted), Option A is 5-10x cheaper. The operational overhead is minimal — ESO is a lightweight operator, and AWS manages Secrets Manager entirely.

**But Option A has functional gaps.** No dynamic secrets, no PKI engine, no transit encryption. These are real capabilities that matter for security maturity.

**The key insight: we don't need those capabilities today.** Our current secret management needs are: store secrets, sync to K8s, rotate periodically. AWS SM + ESO does this perfectly. Adding Vault to get dynamic secrets we don't use yet is paying $20K+/year for future capabilities.

**Re-evaluation triggers (when to move to Vault HCP):**
1. PCI auditor requires dynamic database credentials (not currently required)
2. We need PKI for mTLS across all services (currently only PCI namespace)
3. We need transit encryption for sensitive data fields
4. Secret count exceeds 1,000 AND we need advanced policy (namespaces, ACLs)

**If ANY trigger is met:** Re-evaluate Vault HCP (Option C). Self-hosted (Option B) is permanently off the table — the $36K/year in engineer time for a 4-person team is 9% of our entire engineering capacity on ONE tool's operations.

**Action items:**
1. Continue with AWS SM + ESO (no change)
2. Add re-evaluation triggers to the platform roadmap
3. Schedule review in 6 months or when a trigger is met
4. If adopting Vault: HCP Cloud (Option C) only — never self-hosted

---

# Q4 — Team Dynamics: Payments Team Adoption

## Understanding the Real Problem

Before designing a strategy, I need to decode what the Payments team is actually saying:

```
WHAT THEY SAY                    │ WHAT THEY MEAN
─────────────────────────────────┼──────────────────────────────────
"Our deploy scripts work fine"   │ "We've invested time in these.
                                 │  You're asking us to throw away
                                 │  our work for your thing."
                                 │
"K8s is too complex"             │ "We're afraid of a technology we
                                 │  don't understand. Our current
                                 │  setup is within our comfort zone."
                                 │
"We don't trust you won't break  │ "We've been burned before. PCI is
 our PCI setup"                  │  our responsibility. If YOUR platform
                                 │  causes a PCI violation, WE get
                                 │  blamed."
                                 │
"Centralized tooling gets        │ "The 2022 Platform v1 was abandoned.
 abandoned"                      │  Why should we bet on Platform v2?
                                 │  We need to see commitment."
```

**Root cause:** This is a **trust deficit**, not a technical disagreement. Every objection maps to a fear: loss of control, loss of expertise, loss of compliance, loss of stability.

**The one thing I should NOT do:** Escalate to their VP or mandate adoption through executive authority. This would:
- Confirm their fear that the platform team doesn't respect their expertise
- Create compliance theater (they adopt on paper, keep old scripts running secretly)
- Poison the relationship permanently
- Set a precedent that adoption is forced, not earned

Mandates should be the LAST resort, not the first.

## 8-Week Strategy

### Week 1-2: Listen and Validate

```
ACTIONS:
├── Schedule 1:1 with Payments team lead (30 min, coffee, not a meeting room)
│   └── NOT to convince them. To LISTEN.
│   └── "I want to understand your deployment process and what works well.
│       I'm not here to sell you anything."
│
├── Ask to shadow their deployment process
│   └── Sit with them during a deploy. Take notes. Ask questions.
│   └── Genuinely understand WHY their scripts work well for them.
│   └── Identify: what do their scripts do that our platform doesn't?
│
├── Ask about their PCI concerns specifically
│   └── "What would need to be true for you to trust the platform 
│       with PCI workloads?"
│   └── This surfaces SPECIFIC requirements, not vague fear.
│
└── Review the 2022 "Platform v1" failure from their perspective
    └── What specifically burned them?
    └── What commitment would they need to see from us?

GOAL: Understand their world. Build rapport. Identify specific, 
      addressable concerns. DO NOT PROPOSE SOLUTIONS YET.
```

### Week 3-4: Address Concerns with Evidence, Not Arguments

```
ADDRESSING OBJECTION 1: "Our deploy scripts work fine"
──────────────────────────────────────────────────────
  Don't argue. AGREE that their scripts work.
  Then show what they're MISSING (without criticizing):

  "Your deploy scripts are solid for deployment. But here's what I 
   noticed during the shadow session:
   - No automated rollback. When the Oct 15 deploy had issues, Bob 
     manually SSH'd to roll back. That took 12 minutes.
   - No canary analysis. The deploy goes to 100% immediately.
   - No correlation between deploy events and monitoring. When something 
     breaks, you check CloudWatch separately.
   
   What if we could keep everything you like about your current process 
   AND add automatic rollback, canary analysis, and deploy-correlated 
   monitoring? Without changing your workflow?"

  KEY: Frame it as ADDING capabilities, not REPLACING their work.

ADDRESSING OBJECTION 2: "K8s is too complex"
─────────────────────────────────────────────
  They're right. Kubernetes IS complex. Don't deny it.

  "You're right — Kubernetes is complex. That's exactly why the 
   platform team exists: so you don't have to learn it. 
   
   Here's what your deploy command looks like on the platform:
   
   $ novactl service deploy payment-service --version v2.3.1
   
   That's it. You don't write YAML. You don't manage pods. You don't
   configure networking. The platform handles all of that.
   
   Would it help if I showed you a 30-minute demo of what the developer 
   experience actually looks like? Not the internals — just what YOUR 
   team would interact with."

  KEY: Offer a demo, not a training session. Show the SIMPLE surface,
       not the complex internals.

ADDRESSING OBJECTION 3: "We don't trust you with PCI"
──────────────────────────────────────────────────────
  This is the most legitimate concern. Take it seriously.

  "This is a fair concern, and I want to address it directly. 
   Here's my proposal:

   1. Your team OWNS the PCI Kyverno policies. You write them, you 
      review them, you approve changes. We provide the mechanism; 
      you provide the policy.

   2. We run the PCI migration together — pair programming, not 
      hand-off. Your engineers and ours, side by side.

   3. We schedule a PCI QSA review of the K8s PCI setup BEFORE you 
      migrate. If the QSA has concerns, we fix them first.

   4. You have a kill switch. If at any point you need to revert 
      to EC2, we maintain that path for 90 days after migration.

   I'm not asking you to trust us blindly. I'm asking you to validate 
   our approach with your own expertise and your own QSA."

  KEY: Give them CONTROL over PCI decisions. Don't ask them to trust.
       Give them verification mechanisms.

ADDRESSING OBJECTION 4: "Centralized tooling gets abandoned"
────────────────────────────────────────────────────────────
  Acknowledge the history. Show what's different.

  "Platform v1 in 2022 failed. I've read the postmortem. It was a 
   side project with no dedicated team, no executive sponsor, and 
   no user research. Here's what's different now:

   - Dedicated team: 4 full-time platform engineers. This is our job.
   - Executive sponsor: [CTO name] has approved a 3-year platform 
     investment with committed headcount.
   - Track record: 7 teams are already on the platform. Ask them. 
     The Orders team deployed 47 times last month. The Search team 
     created a new service in 2 hours.

   I can't guarantee the future. But I can show you 7 teams that bet 
   on us and are getting results. Would you be willing to talk to 
   [Orders team lead] about their experience?"

  KEY: Social proof > promises. Let other teams sell for you.
```

### Week 5-6: Joint Proof of Concept

```
ACTIONS:
├── Propose a LIMITED proof of concept:
│   "Let's migrate ONE non-PCI service from your team to the platform.
│    Something small — maybe your internal admin dashboard. 
│    If it works well, we talk about more. If it doesn't, you've lost 
│    nothing and I'll stop asking."
│
├── Their team drives the POC, we support:
│   └── THEY containerize the service (we help if asked)
│   └── THEY review the generated K8s manifests (we explain each line)
│   └── THEY trigger the first deploy (we observe)
│   └── THEY verify monitoring (we show them the dashboard)
│
├── Success criteria they define:
│   └── Ask THEM: "What would make this POC a success for you?"
│   └── Their criteria, not ours. This gives them ownership.
│
└── Weekly check-in during POC:
    └── "What's working? What's not? What's missing?"
    └── Fix issues SAME DAY. Responsiveness builds trust.

GOAL: Low-risk, low-commitment experience that demonstrates value 
      through hands-on experience, not slide decks.
```

### Week 7-8: Expand or Escalate

```
SCENARIO A: POC succeeds, team is cautiously positive
─────────────────────────────────────────────────────
  ├── Celebrate the win publicly (#engineering Slack):
  │   "Payments team's admin-dashboard is now on the platform! 🎉"
  ├── Ask: "Ready to migrate the next service?"
  ├── Propose: PCI migration plan (with QSA review milestone)
  ├── Offer: Dedicated platform engineer embedded with their team for 
  │   4 weeks during PCI migration (enabling team pattern)
  └── TIMELINE: PCI services migrated within 8 more weeks

SCENARIO B: POC succeeds technically but team still resistant
─────────────────────────────────────────────────────────────
  ├── Identify the holdout: Is it the team lead? One vocal engineer?
  ├── Address individual concerns 1:1 (not in group setting)
  ├── Ask the team lead directly: "What would it take?"
  │   └── If the answer is reasonable: do it
  │   └── If the answer is "nothing": it's a people problem, not tech
  ├── Bring in social proof: arrange lunch between Payments lead and 
  │   Orders lead (who was initially skeptical, now a champion)
  └── If still blocked: escalate to shared manager (see below)

SCENARIO C: POC fails (something broke, bad experience)
───────────────────────────────────────────────────────
  ├── Own the failure publicly: "We tried, it didn't work, here's why"
  ├── Fix the root cause (this is valuable platform feedback)
  ├── Ask for another shot in 4 weeks after fixing the issues
  └── If they refuse: respect the decision for now, circle back in 
      a quarter with the fix and new success stories
```

### Escalation Path (if persuasion fails after 8 weeks)

```
ESCALATION IS THE LAST RESORT. Use it only if:
├── POC succeeded but team refuses to continue
├── Team refuses to even do a POC
├── The resistance is blocking platform-wide improvements

ESCALATION STEPS:
1. Conversation with shared VP/Director (NOT a complaint):
   "The Payments team has valid concerns about PCI compliance on 
    the platform. I've addressed them with [specific actions]. The 
    POC succeeded with [metrics]. They're still hesitant. I'd like 
    your help facilitating a conversation about the path forward."

2. Frame as a BUSINESS problem, not an interpersonal conflict:
   "We can't deprecate the EC2 deployment path while one team is 
    on it. That means we maintain two systems: $X/month in infra 
    cost + Y hours/month in platform team time. The business case 
    for platform migration includes decommissioning EC2."

3. Propose a TIMELINE, not a mandate:
   "Can we agree that the Payments team will complete migration by 
    [date]? We'll provide dedicated support. They own the PCI 
    validation. If the QSA raises concerns, we pause."

4. If executive mandates adoption:
   └── Ensure it comes with SUPPORT, not just a deadline
   └── Dedicated platform engineer embedded with Payments for 6 weeks
   └── PCI QSA engaged and paid for by platform budget
   └── Payments team lead included in platform advisory board
       (give them a voice in platform decisions going forward)
```

### What I Should NOT Do (Expanded)

```
╔══════════════════════════════════════════════════════════════════╗
║  NEVER:                                                          ║
║                                                                  ║
║  1. Mandate adoption through executive fiat as the FIRST move.   ║
║     This creates resentment, not adoption. They'll comply        ║
║     minimally and sabotage subtly.                               ║
║                                                                  ║
║  2. Dismiss their concerns as "resistance to change."            ║
║     They process PAYMENTS. PCI compliance is THEIR responsibility║
║     If they break PCI, THEY face the audit findings. Their       ║
║     caution is RATIONAL, not obstructionist.                     ║
║                                                                  ║
║  3. Talk negatively about their deploy scripts to others.        ║
║     "The Payments team is still using shell scripts" — this      ║
║     makes them defensive and makes you look political.           ║
║                                                                  ║
║  4. Bypass the team lead and convince individual engineers.      ║
║     This undermines the team lead's authority and creates        ║
║     internal team conflict. Always work WITH the lead.           ║
║                                                                  ║
║  5. Make promises you can't keep.                                ║
║     "The platform will never break your PCI setup" — you can't   ║
║     guarantee this. Instead: "Here's how we verify together."    ║
║                                                                  ║
║  6. Set arbitrary deadlines without their input.                 ║
║     "You must be on the platform by March 1" — without their     ║
║     agreement, this is a mandate, not a timeline.                ║
╚══════════════════════════════════════════════════════════════════╝
```

### The Underlying Principle

```
╔══════════════════════════════════════════════════════════════════╗
║                                                                  ║
║  Platform adoption is a TRUST problem.                           ║
║                                                                  ║
║  Trust is built by:                                              ║
║  1. Listening before proposing                                   ║
║  2. Giving control, not taking it                                ║
║  3. Delivering small wins, then big ones                         ║
║  4. Being responsive when things break                           ║
║  5. Letting results speak louder than slide decks                ║
║  6. Respecting their expertise (PCI is serious, not paranoia)    ║
║                                                                  ║
║  If you build trust, adoption follows naturally.                 ║
║  If you force adoption, trust is destroyed permanently.          ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

