# Phase 7, Lesson 2: Platform Services

---

## Context

```
Lesson 1 DELIVERED:
  ✅ VPC + Networking (3-AZ, dual CIDR, VPC endpoints)
  ✅ EKS Cluster (managed node groups, Karpenter, IRSA, add-ons)
  ✅ Data Tier (RDS ×3, ElastiCache Redis)
  ✅ Security Baseline (KMS, CloudTrail, S3, Flow Logs)

The infrastructure foundation is built.
Now we need the PLATFORM LAYER — the services that make
applications production-ready WITHOUT each app team
reinventing observability, security, and networking.

Without this layer, every microservice team will:
  - Build their own logging (inconsistently)
  - Skip distributed tracing ("we'll add it later")
  - Hardcode secrets ("it's just dev")
  - Have no mutual TLS between services
  - Have no SLO dashboards
  - Get paged at 3am with no context

The platform layer prevents all of this.
```

---

## Your Mission

```
You are still the Senior Platform Engineer at NovaMart.
The infrastructure is up. The EKS cluster has nodes.
Now build the platform that 200+ microservice teams will depend on.

REQUIREMENTS:

  1. OBSERVABILITY STACK
     a. METRICS — Prometheus + Grafana
        - Prometheus server (or Victoria Metrics for scale)
        - Grafana with persistent dashboards
        - kube-state-metrics, node-exporter
        - Pre-built dashboards: cluster health, node health,
          pod health, namespace resource usage
        - Retention: 15 days local, long-term to S3 (Thanos or
          Cortex sidecar)
        - Federation-ready for multi-cluster future

     b. LOGGING — Fluent Bit + CloudWatch + S3
        - Fluent Bit DaemonSet (NOT Fluentd — resource efficiency)
        - Structured JSON log parsing
        - Route to CloudWatch Logs (hot, 30-day retention)
        - Route to S3 (cold, 1-year retention, Parquet for Athena)
        - Namespace-based log routing (payments logs → separate
          log group for PCI isolation)
        - Log enrichment: add pod labels, node name, namespace,
          trace ID to every log line

     c. TRACING — OpenTelemetry Collector + AWS X-Ray (or Tempo)
        - OpenTelemetry Collector as a DaemonSet + Gateway
        - Collect traces, metrics, and logs (unified pipeline)
        - Export traces to AWS X-Ray (or Grafana Tempo)
        - Sampling strategy: 100% for errors, 10% for success
        - Trace context propagation headers (W3C TraceContext)

     d. ALERTING
        - Alertmanager with PagerDuty/Slack routing
        - Alert severity levels: critical, warning, info
        - Inhibition rules (don't page for pod alerts during
          node-down events)
        - Silencing capability for maintenance windows

  2. SECURITY LAYER
     a. SECRETS MANAGEMENT
        - External Secrets Operator (ESO) connected to AWS
          Secrets Manager
        - SecretStore + ClusterSecretStore configurations
        - Automatic secret rotation awareness
        - IRSA-based auth (already created in Lesson 1)

     b. NETWORK POLICIES
        - Default-deny in all application namespaces
        - Allow rules for known communication patterns:
          payment-svc → payments RDS
          payment-svc → Redis
          frontend → payment-svc (via service mesh)
        - DNS egress allowed for all pods
        - Monitoring namespace exemptions

     c. POD SECURITY
        - Pod Security Standards enforcement (Restricted baseline)
        - OPA Gatekeeper or Kyverno for custom policies:
          * No privileged containers
          * No latest tag
          * Required resource limits
          * Required labels (team, service, version)
          * No hostPath mounts
          * Required security context

     d. IMAGE SECURITY
        - ECR repositories with image scanning enabled
        - Admission controller to block unscanned images
        - Image pull policy: Always for production

  3. SERVICE MESH (Istio or Linkerd — justify your choice)
     - Mutual TLS between all services (zero-trust networking)
     - Traffic management (canary deployments, traffic splitting)
     - Retry policies and circuit breakers
     - Service-level observability (request rate, error rate,
       latency — RED metrics)
     - Ingress gateway configuration
     - Sidecar injection for application namespaces

  4. INGRESS + CERTIFICATE MANAGEMENT
     - AWS Load Balancer Controller (IRSA from Lesson 1)
     - Ingress class configuration
     - cert-manager with Let's Encrypt (or ACM)
     - TLS termination strategy
     - WAF integration on ALB

  5. GITOPS FOUNDATION (ArgoCD)
     - ArgoCD installation with HA
     - IRSA for ArgoCD to access ECR
     - Application of Applications pattern
     - Project-level RBAC (platform team vs app teams)
     - Sync policies (auto-sync for staging, manual for production)

CONSTRAINTS:
  - All installations via Helm charts managed in Terraform
    (or ArgoCD — justify which manages what)
  - Every component must have resource requests AND limits
  - Every component must have pod anti-affinity (HA)
  - Every component must tolerate the system node taint
    OR run on Karpenter-provisioned nodes
  - Secrets must NEVER appear in Helm values or Terraform state
  - Everything must be namespace-isolated

DELIVERABLES:
  1. Platform architecture diagram (what talks to what)
  2. Namespace strategy document
  3. Helm releases (Terraform code for each platform component)
  4. Observability pipeline diagram (data flow)
  5. Network policy manifests
  6. Pod security policies (Kyverno or OPA)
  7. Service mesh configuration
  8. ArgoCD bootstrap configuration
  9. Design decisions for every non-obvious choice
  10. Resource budget (CPU/memory reserved for platform vs apps)
```

---

## What I'm Looking For

```
I will evaluate on:

  INTEGRATION — Do components work together? (traces in Grafana,
                alerts from Prometheus, logs correlated with traces)
  TENANT ISOLATION — Can app teams interfere with each other?
  RESOURCE DISCIPLINE — How much cluster capacity does the
                        platform consume? Is it justified?
  SECURITY DEPTH — Defense in depth, not just perimeter
  OPERATIONAL MATURITY — Can you debug a production issue at 3am
                         using what you've built?
  DAY-2 READINESS — Upgrades, scaling, secret rotation, certificate
                     renewal — are these automated or manual?

I will NOT accept:
  ❌ "Install Prometheus with default values"
  ❌ Helm releases without explicit resource limits
  ❌ "We'll add network policies later"
  ❌ Service mesh without explaining the sidecar resource overhead
  ❌ Monitoring that doesn't monitor itself
  ❌ Security policies with no enforcement (audit-only forever)
  ❌ ArgoCD with admin password in plain text
```


# NovaMart Platform Services — Complete Build

---

## 1. Platform Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                              NOVAMART PLATFORM SERVICES LAYER                                 │
│                                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                              INGRESS TIER                                                │  │
│  │                                                                                          │  │
│  │   Internet → CloudFront → WAF → ALB (AWS LB Controller) → Linkerd Ingress Gateway       │  │
│  │                                      ↓                                                   │  │
│  │                           cert-manager (Let's Encrypt + ACM)                             │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                          │                                                    │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                           SERVICE MESH (Linkerd)                                         │  │
│  │                                                                                          │  │
│  │   ┌──────────┐   mTLS    ┌──────────┐   mTLS    ┌──────────┐                            │  │
│  │   │frontend  │ ────────→ │payment   │ ────────→ │ order    │                            │  │
│  │   │  + proxy │           │  + proxy │           │  + proxy │                            │  │
│  │   └──────────┘           └──────────┘           └──────────┘                            │  │
│  │         │                      │                      │                                  │  │
│  │         │  Every sidecar emits:                       │                                  │  │
│  │         │  • RED metrics (rate, errors, duration)     │                                  │  │
│  │         │  • TCP metrics (connections, bytes)         │                                  │  │
│  │         │  • mTLS identity (SPIFFE)                   │                                  │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                          │                                                    │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                        OBSERVABILITY PIPELINE                                            │  │
│  │                                                                                          │  │
│  │  ┌──────────────────────────────────────────────────────────────┐                        │  │
│  │  │  METRICS PIPELINE                                            │                        │  │
│  │  │                                                              │                        │  │
│  │  │  kube-state-metrics ──┐                                     │                        │  │
│  │  │  node-exporter ───────┼→ Prometheus ──→ Thanos Sidecar ──→ S3│                       │  │
│  │  │  Linkerd metrics ─────┤       │                              │                        │  │
│  │  │  App /metrics ────────┘       ↓                              │                        │  │
│  │  │                        Alertmanager ──→ PagerDuty / Slack    │                        │  │
│  │  │                               │                              │                        │  │
│  │  │                          Grafana (dashboards)                │                        │  │
│  │  └──────────────────────────────────────────────────────────────┘                        │  │
│  │                                                                                          │  │
│  │  ┌──────────────────────────────────────────────────────────────┐                        │  │
│  │  │  LOGGING PIPELINE                                            │                        │  │
│  │  │                                                              │                        │  │
│  │  │  Pod stdout/stderr                                           │                        │  │
│  │  │       ↓                                                      │                        │  │
│  │  │  Fluent Bit (DaemonSet)                                      │                        │  │
│  │  │       │  ← Enriches: namespace, pod, labels, trace_id       │                        │  │
│  │  │       │  ← Parses: JSON, multiline exceptions               │                        │  │
│  │  │       ├──→ CloudWatch Logs (hot, 30-day, per-namespace)      │                        │  │
│  │  │       ├──→ S3 (cold, 1-year, Parquet, Athena-queryable)     │                        │  │
│  │  │       └──→ payments-* → separate CW Log Group (PCI scope)   │                        │  │
│  │  └──────────────────────────────────────────────────────────────┘                        │  │
│  │                                                                                          │  │
│  │  ┌──────────────────────────────────────────────────────────────┐                        │  │
│  │  │  TRACING PIPELINE                                            │                        │  │
│  │  │                                                              │                        │  │
│  │  │  App (OTel SDK) ──→ OTel Collector (DaemonSet/Agent)         │                        │  │
│  │  │                          ↓                                   │                        │  │
│  │  │                     OTel Collector (Deployment/Gateway)       │                        │  │
│  │  │                          │                                   │                        │  │
│  │  │                          ├──→ Grafana Tempo (trace storage)  │                        │  │
│  │  │                          └──→ AWS X-Ray (AWS-native view)    │                        │  │
│  │  │                                                              │                        │  │
│  │  │  Sampling: 100% errors, 10% success, 1% health checks       │                        │  │
│  │  └──────────────────────────────────────────────────────────────┘                        │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                           SECURITY LAYER                                                 │  │
│  │                                                                                          │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │  │
│  │  │  External     │  │   Kyverno    │  │   Network    │  │    ECR       │                 │  │
│  │  │  Secrets      │  │  (Policy     │  │   Policies   │  │  (Image      │                 │  │
│  │  │  Operator     │  │   Engine)    │  │  (Default    │  │   Scanning)  │                 │  │
│  │  │              │  │              │  │   Deny)      │  │              │                 │  │
│  │  │  AWS Secrets  │  │  No latest   │  │  Namespace   │  │  Block       │                 │  │
│  │  │  Manager ←──→│  │  No priv     │  │  isolation   │  │  unscanned   │                 │  │
│  │  │  (IRSA)      │  │  Req limits  │  │  + explicit  │  │  images      │                 │  │
│  │  │              │  │  Req labels  │  │  allow rules │  │              │                 │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘                 │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                           GITOPS (ArgoCD)                                                │  │
│  │                                                                                          │  │
│  │  ArgoCD Server (HA) ──→ Git Repo ──→ Kubernetes Apply                                   │  │
│  │       │                                                                                  │  │
│  │       ├─ Project: platform (auto-sync)                                                   │  │
│  │       ├─ Project: applications (manual-sync in prod)                                     │  │
│  │       └─ App-of-Apps pattern (declarative app management)                                │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Design Decisions Document

```markdown
# Platform Services — Design Decisions

## DD-PS-001: Linkerd Over Istio for Service Mesh

**Decision:** Use Linkerd 2.x as the service mesh, not Istio.

**Rationale:**
- SIDECAR RESOURCE OVERHEAD: Linkerd's proxy (linkerd2-proxy, Rust)
  uses ~10MB memory and <1% CPU per sidecar. Istio's Envoy sidecar
  uses ~50-100MB memory and 1-5% CPU. With 200+ microservices at
  2,000-5,000 pods, this difference is:
    Linkerd: 5,000 × 10MB  = 50GB sidecar memory
    Istio:   5,000 × 70MB  = 350GB sidecar memory
  At r6g.large pricing, that's ~$400/mo vs ~$2,800/mo in sidecar overhead.

- LATENCY: Linkerd's proxy adds ~1ms p99 latency per hop.
  Istio/Envoy adds ~3-5ms p99. For payment flows traversing
  5-6 services, that's 5ms vs 25ms added latency. For an e-commerce
  platform where checkout conversion drops ~7% per second of latency,
  this matters.

- OPERATIONAL COMPLEXITY: Linkerd has ~10 CRDs. Istio has ~50+ CRDs.
  Linkerd's control plane is 3 components. Istio's is 5+.
  With a small platform team, operational simplicity wins.

- mTLS: Both provide automatic mTLS. Linkerd enables it by default
  with no configuration. Istio requires PeerAuthentication policies.

- OBSERVABILITY: Linkerd auto-generates RED metrics (requests, errors,
  duration) for every meshed service — no application instrumentation
  needed. Istio does the same via Envoy, but with heavier resource cost.

**What we lose vs Istio:**
- Advanced traffic policies (fault injection, request mirroring).
  Mitigation: Linkerd covers canary, traffic split, retries, timeouts.
  We don't need fault injection at this maturity level.
- Envoy ecosystem (Lua filters, WASM extensions).
  Mitigation: Not needed for NovaMart's use case.
- Multi-cluster mesh (Istio is more mature here).
  Mitigation: We're single-cluster for now. Linkerd multi-cluster
  exists if we need it later.

**Cost impact:** ~$2,400/mo savings in sidecar overhead at scale.

---

## DD-PS-002: Kyverno Over OPA/Gatekeeper for Policy Engine

**Decision:** Use Kyverno for Kubernetes policy enforcement, not
OPA/Gatekeeper.

**Rationale:**
- POLICY LANGUAGE: Kyverno policies are native Kubernetes YAML.
  OPA/Gatekeeper requires Rego, a purpose-built query language.
  For application teams who need to understand why their deployment
  was rejected, YAML policies are immediately readable. Rego requires
  training.

- MUTATION + VALIDATION: Kyverno can both validate (reject bad config)
  AND mutate (inject good defaults). Gatekeeper can only validate.
  Example: Kyverno can auto-inject required labels if missing, rather
  than just rejecting the deployment.

- GENERATION: Kyverno can generate resources (e.g., auto-create
  NetworkPolicy when a new namespace is created). Gatekeeper cannot.

- AUDIT + ENFORCE: Kyverno supports audit mode → enforce mode
  per-policy, making rollout safe. Gatekeeper also supports this,
  so no difference here.

**What we lose vs Gatekeeper:**
- Rego's expressiveness for complex cross-resource queries.
  Mitigation: Kyverno handles 95% of use cases. For the remaining
  5%, we can add Gatekeeper as a supplement.
- Larger community adoption in enterprise environments.
  Mitigation: Kyverno is a CNCF Incubating project with strong
  adoption growth.

**Cost impact:** Zero. Both are open source.

---

## DD-PS-003: Fluent Bit Over Fluentd for Log Collection

**Decision:** Fluent Bit as the log collector DaemonSet, not Fluentd.

**Rationale:**
- RESOURCE USAGE: Fluent Bit uses ~15MB memory per node.
  Fluentd uses ~100-200MB per node. Across 50+ nodes at peak,
  that's 750MB vs 10GB of memory consumed by log collection.

- PERFORMANCE: Fluent Bit processes ~30K logs/sec per node.
  Fluentd processes ~10-15K logs/sec. For 200+ microservices
  generating verbose JSON logs, Fluent Bit has more headroom.

- MAINTAINED BY SAME PROJECT: Both are CNCF. Fluent Bit is
  the younger, leaner project designed specifically for containers.
  Fluentd is the older, plugin-heavy project designed for VMs.

- PLUGIN ECOSYSTEM: Fluent Bit has native outputs for CloudWatch,
  S3, and Kinesis. Fluentd has more plugins overall, but we only
  need AWS outputs.

**Cost impact:** ~$200/mo savings in DaemonSet memory at peak.

---

## DD-PS-004: Grafana Tempo Over Jaeger for Trace Storage

**Decision:** Use Grafana Tempo for trace storage, with X-Ray
as a secondary export for AWS-native correlation.

**Rationale:**
- NATIVE GRAFANA INTEGRATION: Tempo integrates with Grafana's
  Explore view, enabling trace-to-log and trace-to-metric
  correlation in a single pane. Jaeger requires a separate UI.

- S3 BACKEND: Tempo stores traces in S3 (object storage), which
  is cheap and scales infinitely. Jaeger requires Elasticsearch
  or Cassandra, which are expensive to operate.

- COST: Tempo on S3 costs ~$0.023/GB/month for trace storage.
  Jaeger on Elasticsearch costs ~$0.10-0.30/GB/month.
  At 50GB/day of traces, that's $35/mo vs $150-450/mo.

- OpenTelemetry native: Tempo accepts OTLP directly.
  Jaeger requires format conversion for newer OTel signals.

**Why also export to X-Ray:**
- X-Ray integrates with AWS console, Lambda, API Gateway.
- If we add Lambda functions later, X-Ray traces them natively.
- X-Ray's service map provides an automatic topology view.
- X-Ray is free-tier-eligible for the first 100K traces/month.

**Cost impact:** ~$100-400/mo savings over Jaeger/Elasticsearch.

---

## DD-PS-005: Terraform Bootstraps Platform, ArgoCD Manages Applications

**Decision:** Terraform installs core platform services via Helm.
ArgoCD manages application deployments after bootstrap.

**Rationale — The Bootstrap Problem:**
  ArgoCD cannot install itself. Something must install ArgoCD first.
  That something is Terraform.

  But once ArgoCD is running, we DON'T want Terraform managing
  every application Helm release because:
  1. Terraform apply takes minutes; ArgoCD syncs take seconds.
  2. Terraform requires CI/CD pipeline access; ArgoCD watches Git.
  3. Terraform state tracks every Kubernetes resource — state bloat.
  4. ArgoCD provides drift detection, auto-heal, and rollback.

**Boundary definition:**
  TERRAFORM MANAGES (infrastructure-coupled):
    - Prometheus stack (kube-prometheus-stack)
    - Fluent Bit
    - OpenTelemetry Collector
    - Linkerd control plane
    - cert-manager
    - AWS Load Balancer Controller
    - External Secrets Operator
    - Kyverno
    - ArgoCD itself
    - Grafana Tempo

  ARGOCD MANAGES (application-coupled):
    - All microservice deployments (payment-svc, order-svc, etc.)
    - Application-specific ConfigMaps/Secrets
    - Application Ingress resources
    - Karpenter NodePool/EC2NodeClass (may change frequently)
    - Linkerd service profiles (per-service retry/timeout config)
    - Grafana dashboards (as ConfigMaps)

**Why this split:**
  Platform services change rarely (monthly). They're infrastructure.
  Applications change frequently (daily). They're deployments.
  Different change cadences → different management tools.

---

## DD-PS-006: Namespace Strategy — Isolation by Concern

**Decision:** Dedicated namespaces per concern, not per team.

| Namespace | Contents | Rationale |
|-----------|----------|-----------|
| kube-system | CoreDNS, kube-proxy, VPC CNI, EBS CSI | AWS-managed add-ons |
| linkerd | Linkerd control plane | Mesh isolation |
| monitoring | Prometheus, Grafana, Alertmanager, Thanos | Observability stack |
| logging | Fluent Bit, OTel Collector | Log pipeline |
| tracing | Tempo, OTel Gateway | Trace pipeline |
| cert-manager | cert-manager controller | Certificate lifecycle |
| external-secrets | ESO controller | Secrets sync |
| kyverno | Kyverno controller + policies | Policy engine |
| argocd | ArgoCD server, repo server, app controller | GitOps |
| ingress | AWS LB Controller, Linkerd ingress gateway | Traffic entry |
| novamart-payments | payment-svc | PCI-scoped workloads |
| novamart-orders | order-svc, inventory-svc | Order domain |
| novamart-users | user-svc, auth-svc | User domain |
| novamart-frontend | web frontend, BFF | Frontend domain |
| novamart-shared | shared libraries, message queues | Cross-domain |

**PCI isolation:** The novamart-payments namespace has:
  - Separate NetworkPolicies (stricter egress)
  - Separate log routing (dedicated CloudWatch Log Group)
  - Separate IRSA roles (only payment-svc accesses payments DB)
  - Kyverno policies enforcing additional security context

---

## DD-PS-007: Prometheus Retention Strategy — Hot + Cold

**Decision:** 15-day local retention in Prometheus, long-term
storage in S3 via Thanos Sidecar.

**Rationale:**
- Local Prometheus with 15 days of data at 200+ services generates
  ~50-100GB of TSDB data. This fits on a 200GB PV (gp3).
- Queries older than 15 days go through Thanos Store Gateway → S3.
- S3 storage: ~$2.30/month per 100GB. Annual trace data ~$30/mo.
- Thanos Compactor runs nightly to downsample old data:
  Raw (0-15 days) → 5m resolution (15d-90d) → 1h resolution (90d+)

**Why not VictoriaMetrics:**
- VictoriaMetrics is excellent but less ecosystem integration.
- kube-prometheus-stack (Prometheus Operator) is the de facto
  standard with the richest library of pre-built alerts and dashboards.
- If we hit Prometheus scaling limits (>1M active series), we
  can migrate to VictoriaMetrics as a drop-in backend.

---

## DD-PS-008: Alert Routing Strategy — Severity × Team

**Decision:** Three severity tiers with distinct routing:

| Severity | Routing | Response SLA | Example |
|----------|---------|-------------|---------|
| critical | PagerDuty (wake people) | 15 min | Node NotReady, RDS down, payment errors >5% |
| warning | Slack #alerts-warning | 1 hour | CPU >80%, disk >75%, pod restart loop |
| info | Slack #alerts-info | Next business day | Certificate expiring in 30d, addon update available |

**Inhibition rules:**
- IF node-down THEN suppress all pod-alerts on that node
- IF cluster-wide network issue THEN suppress individual service alerts
- IF maintenance-mode label set THEN suppress all non-critical alerts

**Grouping:**
- Group by namespace + alertname (avoid alert storms)
- group_wait: 30s, group_interval: 5m, repeat_interval: 4h

---

## DD-PS-009: Resource Budget — Platform vs Application

**Decision:** Platform services consume a defined resource budget.
Anything above this budget requires platform team approval.

| Component | CPU Request | Memory Request | Replicas | Total CPU | Total Memory |
|-----------|-------------|----------------|----------|-----------|-------------|
| Prometheus | 1000m | 4Gi | 2 | 2000m | 8Gi |
| Thanos Sidecar | 200m | 512Mi | 2 | 400m | 1Gi |
| Thanos Store GW | 500m | 1Gi | 2 | 1000m | 2Gi |
| Thanos Compactor | 500m | 2Gi | 1 | 500m | 2Gi |
| Alertmanager | 100m | 256Mi | 3 | 300m | 768Mi |
| Grafana | 500m | 512Mi | 2 | 1000m | 1Gi |
| Tempo | 500m | 1Gi | 3 | 1500m | 3Gi |
| kube-state-metrics | 100m | 256Mi | 2 | 200m | 512Mi |
| node-exporter | 100m | 128Mi | per-node | ~5000m | ~6.4Gi |
| Fluent Bit | 200m | 128Mi | per-node | ~10000m | ~6.4Gi |
| OTel Agent | 200m | 256Mi | per-node | ~10000m | ~12.8Gi |
| OTel Gateway | 500m | 1Gi | 3 | 1500m | 3Gi |
| Linkerd CP | 500m | 512Mi | 3 | 1500m | 1.5Gi |
| Linkerd Proxy | 100m | 70Mi | per-pod | ~500000m | ~350Gi |
| cert-manager | 100m | 128Mi | 2 | 200m | 256Mi |
| ESO | 100m | 128Mi | 2 | 200m | 256Mi |
| Kyverno | 500m | 512Mi | 3 | 1500m | 1.5Gi |
| ArgoCD | 500m | 512Mi | 3(×3) | 4500m | 4.5Gi |
| AWS LB Controller | 100m | 128Mi | 2 | 200m | 256Mi |
| **TOTAL (fixed)** | | | | **~18 vCPU** | **~32Gi** |
| **TOTAL (per-node)** | | | | **~500m/node** | **~512Mi/node** |
| **TOTAL (per-pod, mesh)** | | | | **~100m/pod** | **~70Mi/pod** |

At 50 nodes: Fixed (18 vCPU + 32Gi) + Per-node (25 vCPU + 25Gi) + 
Mesh at 2000 pods (200 vCPU + 140Gi) = ~243 vCPU + ~197Gi

This is ~12% of a 2000-vCPU cluster. Acceptable for full
observability + security + mesh. The mesh proxy overhead (Linkerd)
is the largest line item — justified by the mTLS and observability
it provides without application code changes.
```

---

## 3. Directory Structure

```
novamart-infrastructure/
├── modules/                          # (from Lesson 1)
│   ├── vpc/
│   ├── eks/
│   ├── data-tier/
│   └── security-baseline/
│
├── platform/                         # NEW — Platform Services
│   ├── modules/
│   │   ├── observability/
│   │   │   ├── prometheus/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   ├── outputs.tf
│   │   │   │   ├── values.yaml        # Helm values
│   │   │   │   ├── alerts/
│   │   │   │   │   ├── cluster-alerts.yaml
│   │   │   │   │   ├── node-alerts.yaml
│   │   │   │   │   ├── pod-alerts.yaml
│   │   │   │   │   └── platform-alerts.yaml
│   │   │   │   └── dashboards/
│   │   │   │       ├── cluster-overview.json
│   │   │   │       ├── node-health.json
│   │   │   │       ├── namespace-resources.json
│   │   │   │       └── linkerd-service.json
│   │   │   ├── fluent-bit/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   ├── outputs.tf
│   │   │   │   └── values.yaml
│   │   │   ├── otel-collector/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   ├── outputs.tf
│   │   │   │   └── values.yaml
│   │   │   └── tempo/
│   │   │       ├── main.tf
│   │   │       ├── variables.tf
│   │   │       ├── outputs.tf
│   │   │       └── values.yaml
│   │   │
│   │   ├── security/
│   │   │   ├── external-secrets/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   ├── outputs.tf
│   │   │   │   └── cluster-secret-store.yaml
│   │   │   ├── kyverno/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   ├── outputs.tf
│   │   │   │   ├── values.yaml
│   │   │   │   └── policies/
│   │   │   │       ├── disallow-privileged.yaml
│   │   │   │       ├── disallow-latest-tag.yaml
│   │   │   │       ├── require-resource-limits.yaml
│   │   │   │       ├── require-labels.yaml
│   │   │   │       ├── require-security-context.yaml
│   │   │   │       ├── disallow-host-path.yaml
│   │   │   │       ├── restrict-image-registries.yaml
│   │   │   │       └── require-run-as-non-root.yaml
│   │   │   ├── network-policies/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   ├── default-deny.yaml
│   │   │   │   ├── allow-dns.yaml
│   │   │   │   ├── allow-monitoring.yaml
│   │   │   │   ├── allow-mesh-control-plane.yaml
│   │   │   │   └── app-policies/
│   │   │   │       ├── payments-namespace.yaml
│   │   │   │       ├── orders-namespace.yaml
│   │   │   │       └── users-namespace.yaml
│   │   │   └── ecr/
│   │   │       ├── main.tf
│   │   │       ├── variables.tf
│   │   │       └── outputs.tf
│   │   │
│   │   ├── service-mesh/
│   │   │   ├── linkerd/
│   │   │   │   ├── main.tf
│   │   │   │   ├── variables.tf
│   │   │   │   ├── outputs.tf
│   │   │   │   ├── values-control-plane.yaml
│   │   │   │   └── values-viz.yaml
│   │   │   └── trust-anchor/
│   │   │       ├── main.tf              # Generates TLS certs
│   │   │       └── variables.tf
│   │   │
│   │   ├── ingress/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── cert-manager-values.yaml
│   │   │   └── aws-lb-controller-values.yaml
│   │   │
│   │   └── gitops/
│   │       ├── argocd/
│   │       │   ├── main.tf
│   │       │   ├── variables.tf
│   │       │   ├── outputs.tf
│   │       │   ├── values.yaml
│   │       │   └── projects/
│   │       │       ├── platform.yaml
│   │       │       └── applications.yaml
│   │       └── app-of-apps/
│   │           ├── main.tf
│   │           └── root-application.yaml
│   │
│   └── environments/
│       └── production/
│           ├── main.tf                  # Composes all platform modules
│           ├── variables.tf
│           ├── outputs.tf
│           ├── backend.tf
│           ├── data.tf                  # Remote state from Lesson 1
│           ├── terraform.tfvars
│           └── versions.tf
│
├── kubernetes/                          # Raw manifests (managed by ArgoCD post-bootstrap)
│   ├── namespaces/
│   │   ├── novamart-payments.yaml
│   │   ├── novamart-orders.yaml
│   │   ├── novamart-users.yaml
│   │   ├── novamart-frontend.yaml
│   │   └── novamart-shared.yaml
│   └── network-policies/
│       └── (applied via platform module)
│
└── docs/
    ├── platform-architecture.md
    ├── namespace-strategy.md
    ├── observability-pipeline.md
    ├── runbook-platform-operations.md
    └── resource-budget.md
```

---

## 4. Terraform Code — Platform Environment Composition

### `platform/environments/production/versions.tf`

```hcl
terraform {
  required_version = ">= 1.7.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.27"
    }
    kubectl = {
      source  = "alekc/kubectl"
      version = "~> 2.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}
```

### `platform/environments/production/backend.tf`

```hcl
terraform {
  backend "s3" {
    key     = "novamart/production/platform/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true
  }
}
```

### `platform/environments/production/data.tf`

```hcl
# ─── Remote State from Lesson 1 Infrastructure ──────────────────

data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "novamart/production/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

data "terraform_remote_state" "eks" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "novamart/production/eks/terraform.tfstate"
    region = "us-east-1"
  }
}

data "terraform_remote_state" "security" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "novamart/production/security/terraform.tfstate"
    region = "us-east-1"
  }
}

data "terraform_remote_state" "data" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "novamart/production/data/terraform.tfstate"
    region = "us-east-1"
  }
}

# ─── Provider Configuration ─────────────────────────────────────

provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = "production"
      Team        = "platform"
      ManagedBy   = "terraform"
      Project     = "novamart"
      CostCenter  = "PLATFORM-001"
    }
  }
}

provider "helm" {
  kubernetes {
    host                   = data.terraform_remote_state.eks.outputs.cluster_endpoint
    cluster_ca_certificate = base64decode(data.terraform_remote_state.eks.outputs.cluster_certificate_authority)

    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args = [
        "eks", "get-token",
        "--cluster-name", data.terraform_remote_state.eks.outputs.cluster_name,
        "--region", "us-east-1"
      ]
    }
  }
}

provider "kubernetes" {
  host                   = data.terraform_remote_state.eks.outputs.cluster_endpoint
  cluster_ca_certificate = base64decode(data.terraform_remote_state.eks.outputs.cluster_certificate_authority)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks", "get-token",
      "--cluster-name", data.terraform_remote_state.eks.outputs.cluster_name,
      "--region", "us-east-1"
    ]
  }
}

provider "kubectl" {
  host                   = data.terraform_remote_state.eks.outputs.cluster_endpoint
  cluster_ca_certificate = base64decode(data.terraform_remote_state.eks.outputs.cluster_certificate_authority)
  load_config_file       = false

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks", "get-token",
      "--cluster-name", data.terraform_remote_state.eks.outputs.cluster_name,
      "--region", "us-east-1"
    ]
  }
}
```

### `platform/environments/production/variables.tf`

```hcl
variable "state_bucket" {
  description = "S3 bucket for Terraform remote state"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
  default     = "novamart-production"
}

variable "domain_name" {
  description = "Primary domain for NovaMart"
  type        = string
  default     = "novamart.com"
}

variable "pagerduty_integration_key" {
  description = "PagerDuty Events API v2 integration key for critical alerts"
  type        = string
  sensitive   = true
}

variable "slack_webhook_url_warning" {
  description = "Slack incoming webhook for warning alerts"
  type        = string
  sensitive   = true
}

variable "slack_webhook_url_info" {
  description = "Slack incoming webhook for info alerts"
  type        = string
  sensitive   = true
}

variable "acme_email" {
  description = "Email for Let's Encrypt ACME registration"
  type        = string
  default     = "platform-team@novamart.com"
}

variable "argocd_github_ssh_key" {
  description = "SSH private key for ArgoCD to access GitHub repos"
  type        = string
  sensitive   = true
}

variable "argocd_repo_url" {
  description = "Git repository URL for ArgoCD application manifests"
  type        = string
  default     = "git@github.com:novamart/kubernetes-manifests.git"
}
```

### `platform/environments/production/main.tf`

```hcl
# ═══════════════════════════════════════════════════════════════════
# NOVAMART PLATFORM SERVICES — PRODUCTION
#
# Apply order (dependencies):
#   1. Namespaces
#   2. Linkerd (mesh must be ready before apps)
#   3. Observability (monitoring, logging, tracing)
#   4. Security (ESO, Kyverno, Network Policies)
#   5. Ingress (ALB Controller, cert-manager)
#   6. ArgoCD (GitOps — last, manages everything after)
# ═══════════════════════════════════════════════════════════════════

locals {
  cluster_name            = data.terraform_remote_state.eks.outputs.cluster_name
  cluster_endpoint        = data.terraform_remote_state.eks.outputs.cluster_endpoint
  oidc_provider_arn       = data.terraform_remote_state.eks.outputs.oidc_provider_arn
  oidc_provider_url       = data.terraform_remote_state.eks.outputs.oidc_provider_url
  node_security_group_id  = data.terraform_remote_state.eks.outputs.node_security_group_id
  external_secrets_role_arn = data.terraform_remote_state.eks.outputs.external_secrets_role_arn
  aws_lb_controller_role_arn = data.terraform_remote_state.eks.outputs.aws_lb_controller_role_arn
  kms_general_key_arn     = data.terraform_remote_state.security.outputs.kms_key_arns["general"]
  flow_logs_bucket_arn    = data.terraform_remote_state.security.outputs.s3_bucket_arns["flow_logs"]
}

# ─── Step 1: Namespaces ──────────────────────────────────────────

module "namespaces" {
  source = "../../modules/namespaces"

  environment = var.environment
  namespaces = {
    # Platform namespaces
    linkerd            = { labels = { "linkerd.io/is-control-plane" = "true" } }
    monitoring         = { labels = { "novamart.com/tier" = "platform" } }
    logging            = { labels = { "novamart.com/tier" = "platform" } }
    tracing            = { labels = { "novamart.com/tier" = "platform" } }
    cert-manager       = { labels = { "novamart.com/tier" = "platform" } }
    external-secrets   = { labels = { "novamart.com/tier" = "platform" } }
    kyverno            = { labels = { "novamart.com/tier" = "platform" } }
    argocd             = { labels = { "novamart.com/tier" = "platform" } }
    ingress            = { labels = { "novamart.com/tier" = "platform" } }

    # Application namespaces — mesh-injected
    novamart-payments  = {
      labels = {
        "novamart.com/tier"        = "application"
        "novamart.com/domain"      = "payments"
        "novamart.com/pci-scope"   = "true"
        "linkerd.io/inject"        = "enabled"
      }
    }
    novamart-orders    = {
      labels = {
        "novamart.com/tier"   = "application"
        "novamart.com/domain" = "orders"
        "linkerd.io/inject"   = "enabled"
      }
    }
    novamart-users     = {
      labels = {
        "novamart.com/tier"   = "application"
        "novamart.com/domain" = "users"
        "linkerd.io/inject"   = "enabled"
      }
    }
    novamart-frontend  = {
      labels = {
        "novamart.com/tier"   = "application"
        "novamart.com/domain" = "frontend"
        "linkerd.io/inject"   = "enabled"
      }
    }
    novamart-shared    = {
      labels = {
        "novamart.com/tier"   = "application"
        "novamart.com/domain" = "shared"
        "linkerd.io/inject"   = "enabled"
      }
    }
  }
}

# ─── Step 2: Service Mesh (Linkerd) ──────────────────────────────

module "linkerd" {
  source = "../../modules/service-mesh/linkerd"

  environment  = var.environment
  cluster_name = local.cluster_name

  depends_on = [module.namespaces]
}

# ─── Step 3a: Observability — Prometheus Stack ───────────────────

module "prometheus" {
  source = "../../modules/observability/prometheus"

  environment            = var.environment
  cluster_name           = local.cluster_name
  oidc_provider_arn      = local.oidc_provider_arn
  oidc_provider_url      = local.oidc_provider_url
  kms_key_arn            = local.kms_general_key_arn
  thanos_s3_bucket       = module.observability_storage.thanos_bucket_id

  pagerduty_integration_key = var.pagerduty_integration_key
  slack_webhook_url_warning = var.slack_webhook_url_warning
  slack_webhook_url_info    = var.slack_webhook_url_info

  depends_on = [module.namespaces, module.linkerd]
}

# ─── Step 3b: Observability — Fluent Bit ─────────────────────────

module "fluent_bit" {
  source = "../../modules/observability/fluent-bit"

  environment       = var.environment
  cluster_name      = local.cluster_name
  oidc_provider_arn = local.oidc_provider_arn
  oidc_provider_url = local.oidc_provider_url
  aws_region        = "us-east-1"
  logs_s3_bucket    = module.observability_storage.logs_bucket_id

  depends_on = [module.namespaces]
}

# ─── Step 3c: Observability — Tempo ──────────────────────────────

module "tempo" {
  source = "../../modules/observability/tempo"

  environment       = var.environment
  cluster_name      = local.cluster_name
  oidc_provider_arn = local.oidc_provider_arn
  oidc_provider_url = local.oidc_provider_url
  traces_s3_bucket  = module.observability_storage.traces_bucket_id

  depends_on = [module.namespaces]
}

# ─── Step 3d: Observability — OTel Collector ─────────────────────

module "otel_collector" {
  source = "../../modules/observability/otel-collector"

  environment       = var.environment
  cluster_name      = local.cluster_name
  oidc_provider_arn = local.oidc_provider_arn
  oidc_provider_url = local.oidc_provider_url
  tempo_endpoint    = module.tempo.distributor_endpoint
  xray_region       = "us-east-1"

  depends_on = [module.namespaces, module.tempo]
}

# ─── Step 3e: Observability — S3 Storage Buckets ─────────────────

module "observability_storage" {
  source = "../../modules/observability/storage"

  environment = var.environment
  kms_key_arn = local.kms_general_key_arn
}

# ─── Step 4a: Security — External Secrets Operator ───────────────

module "external_secrets" {
  source = "../../modules/security/external-secrets"

  environment            = var.environment
  irsa_role_arn          = local.external_secrets_role_arn
  aws_region             = "us-east-1"

  depends_on = [module.namespaces]
}

# ─── Step 4b: Security — Kyverno ─────────────────────────────────

module "kyverno" {
  source = "../../modules/security/kyverno"

  environment    = var.environment
  ecr_registry   = "${data.aws_caller_identity.current.account_id}.dkr.ecr.us-east-1.amazonaws.com"

  depends_on = [module.namespaces]
}

# ─── Step 4c: Security — Network Policies ────────────────────────

module "network_policies" {
  source = "../../modules/security/network-policies"

  environment = var.environment

  depends_on = [module.namespaces, module.kyverno]
}

# ─── Step 4d: Security — ECR Repositories ────────────────────────

module "ecr" {
  source = "../../modules/security/ecr"

  environment = var.environment
  kms_key_arn = local.kms_general_key_arn

  repositories = [
    "novamart/payment-svc",
    "novamart/order-svc",
    "novamart/user-svc",
    "novamart/auth-svc",
    "novamart/frontend",
    "novamart/inventory-svc",
    "novamart/notification-svc",
    "novamart/bff",
  ]
}

# ─── Step 5: Ingress ─────────────────────────────────────────────

module "ingress" {
  source = "../../modules/ingress"

  environment            = var.environment
  cluster_name           = local.cluster_name
  aws_lb_controller_role_arn = local.aws_lb_controller_role_arn
  domain_name            = var.domain_name
  acme_email             = var.acme_email
  vpc_id                 = data.terraform_remote_state.networking.outputs.vpc_id

  depends_on = [module.namespaces, module.linkerd]
}

# ─── Step 6: ArgoCD ──────────────────────────────────────────────

module "argocd" {
  source = "../../modules/gitops/argocd"

  environment        = var.environment
  cluster_name       = local.cluster_name
  domain_name        = var.domain_name
  github_ssh_key     = var.argocd_github_ssh_key
  repo_url           = var.argocd_repo_url
  oidc_provider_arn  = local.oidc_provider_arn
  oidc_provider_url  = local.oidc_provider_url

  depends_on = [module.namespaces, module.ingress]
}

data "aws_caller_identity" "current" {}
```

---

## 5. Module Implementations

### 5.1 Namespaces Module

```hcl
# platform/modules/namespaces/main.tf

variable "environment" {
  type = string
}

variable "namespaces" {
  description = "Map of namespace name to config (labels, annotations)"
  type = map(object({
    labels = map(string)
  }))
}

resource "kubernetes_namespace" "ns" {
  for_each = var.namespaces

  metadata {
    name = each.key

    labels = merge(each.value.labels, {
      "novamart.com/environment" = var.environment
      "novamart.com/managed-by"  = "terraform"
      # Pod Security Standards — enforce restricted by default
      "pod-security.kubernetes.io/enforce" = lookup(
        each.value.labels, "novamart.com/tier", ""
      ) == "platform" ? "baseline" : "restricted"
      "pod-security.kubernetes.io/warn"    = "restricted"
      "pod-security.kubernetes.io/audit"   = "restricted"
    })

    annotations = {
      "novamart.com/created-by" = "platform-team"
    }
  }
}

output "namespace_names" {
  value = { for k, v in kubernetes_namespace.ns : k => v.metadata[0].name }
}
```

---

### 5.2 Linkerd Service Mesh

```hcl
# platform/modules/service-mesh/trust-anchor/main.tf
#
# Linkerd requires a trust anchor certificate (root CA) and
# issuer certificate. We generate these in Terraform and store
# them as Kubernetes secrets. In production, use cert-manager
# or Vault for rotation — this provides Day 1 bootstrap.

variable "environment" {
  type = string
}

variable "trust_anchor_validity_hours" {
  description = "Trust anchor (root CA) validity in hours"
  type        = number
  default     = 87600 # 10 years
}

variable "issuer_validity_hours" {
  description = "Issuer certificate validity in hours"
  type        = number
  default     = 8760 # 1 year
}

# ─── Trust Anchor (Root CA) ──────────────────────────────────────

resource "tls_private_key" "trust_anchor" {
  algorithm   = "ECDSA"
  ecdsa_curve = "P256"
}

resource "tls_self_signed_cert" "trust_anchor" {
  private_key_pem = tls_private_key.trust_anchor.private_key_pem

  subject {
    common_name = "root.linkerd.cluster.local"
  }

  validity_period_hours = var.trust_anchor_validity_hours
  is_ca_certificate     = true
  set_subject_key_id    = true

  allowed_uses = [
    "cert_signing",
    "crl_signing",
  ]
}

# ─── Issuer Certificate ─────────────────────────────────────────

resource "tls_private_key" "issuer" {
  algorithm   = "ECDSA"
  ecdsa_curve = "P256"
}

resource "tls_cert_request" "issuer" {
  private_key_pem = tls_private_key.issuer.private_key_pem

  subject {
    common_name = "identity.linkerd.cluster.local"
  }
}

resource "tls_locally_signed_cert" "issuer" {
  cert_request_pem   = tls_cert_request.issuer.cert_request_pem
  ca_private_key_pem = tls_private_key.trust_anchor.private_key_pem
  ca_cert_pem        = tls_self_signed_cert.trust_anchor.cert_pem

  validity_period_hours = var.issuer_validity_hours
  is_ca_certificate     = true
  set_subject_key_id    = true

  allowed_uses = [
    "cert_signing",
    "crl_signing",
  ]
}

output "trust_anchor_pem" {
  value     = tls_self_signed_cert.trust_anchor.cert_pem
  sensitive = true
}

output "issuer_cert_pem" {
  value     = tls_locally_signed_cert.issuer.cert_pem
  sensitive = true
}

output "issuer_key_pem" {
  value     = tls_private_key.issuer.private_key_pem
  sensitive = true
}

output "trust_anchor_expiry" {
  value = tls_self_signed_cert.trust_anchor.validity_end_time
}

output "issuer_expiry" {
  value = tls_locally_signed_cert.issuer.validity_end_time
}
```

```hcl
# platform/modules/service-mesh/linkerd/variables.tf

variable "environment" {
  type = string
}

variable "cluster_name" {
  type = string
}

variable "linkerd_version" {
  description = "Linkerd Helm chart version"
  type        = string
  default     = "1.16.11"  # Maps to Linkerd 2.14.x
}

variable "linkerd_viz_version" {
  description = "Linkerd Viz extension chart version"
  type        = string
  default     = "30.12.11"
}

variable "ha_enabled" {
  description = "Enable HA mode for Linkerd control plane"
  type        = bool
  default     = true
}
```

```hcl
# platform/modules/service-mesh/linkerd/main.tf

locals {
  name_prefix = "novamart-${var.environment}"
}

# ─── Trust Anchor Certificates ───────────────────────────────────

module "trust_anchor" {
  source      = "../trust-anchor"
  environment = var.environment
}

# ─── Linkerd CRDs (must install before control plane) ────────────

resource "helm_release" "linkerd_crds" {
  name             = "linkerd-crds"
  namespace        = "linkerd"
  repository       = "https://helm.linkerd.io/stable"
  chart            = "linkerd-crds"
  version          = var.linkerd_version
  create_namespace = false

  # CRDs should not be removed on helm uninstall
  skip_crds = false
}

# ─── Linkerd Control Plane ───────────────────────────────────────

resource "helm_release" "linkerd_control_plane" {
  name       = "linkerd-control-plane"
  namespace  = "linkerd"
  repository = "https://helm.linkerd.io/stable"
  chart      = "linkerd-control-plane"
  version    = var.linkerd_version

  # Trust anchor and issuer certificates
  set_sensitive {
    name  = "identityTrustAnchorsPEM"
    value = module.trust_anchor.trust_anchor_pem
  }

  set_sensitive {
    name  = "identity.issuer.tls.crtPEM"
    value = module.trust_anchor.issuer_cert_pem
  }

  set_sensitive {
    name  = "identity.issuer.tls.keyPEM"
    value = module.trust_anchor.issuer_key_pem
  }

  values = [yamlencode({
    # HA configuration
    controllerReplicas  = var.ha_enabled ? 3 : 1
    enablePodAntiAffinity = var.ha_enabled

    # Proxy (sidecar) defaults — applied to every meshed pod
    proxy = {
      resources = {
        cpu = {
          request = "100m"
          limit   = "500m"
        }
        memory = {
          request = "70Mi"
          limit   = "250Mi"
        }
      }
      # Log level for sidecars
      logLevel  = "warn,linkerd=info"
      logFormat = "json"
    }

    # Control plane component resources
    controllerResources = {
      cpu = {
        request = "250m"
        limit   = "1"
      }
      memory = {
        request = "256Mi"
        limit   = "512Mi"
      }
    }

    identityResources = {
      cpu = {
        request = "100m"
        limit   = "500m"
      }
      memory = {
        request = "128Mi"
        limit   = "256Mi"
      }
    }

    destinationResources = {
      cpu = {
        request = "250m"
        limit   = "1"
      }
      memory = {
        request = "256Mi"
        limit   = "512Mi"
      }
    }

    # Topology spread for AZ distribution
    controllerDeploymentStrategy = {
      type = "RollingUpdate"
      rollingUpdate = {
        maxUnavailable = 1
        maxSurge       = 1
      }
    }

    # Proxy injection configuration
    proxyInit = {
      resources = {
        cpu = {
          request = "10m"
          limit   = "100m"
        }
        memory = {
          request = "20Mi"
          limit   = "50Mi"
        }
      }
    }

    # Enable pod disruption budgets for HA
    enablePodDisruptionBudget = var.ha_enabled

    # Heartbeat — disable phoning home in production
    disableHeartBeat = true

    # Policy — default is to ALLOW all traffic (mesh provides mTLS,
    # not authorization). Network Policies handle authorization.
    policyController = {
      defaultAllowPolicy = "all-authenticated"
      # "all-authenticated" = only mTLS peers can communicate
      # This means non-meshed pods CANNOT reach meshed pods
      resources = {
        cpu = {
          request = "100m"
          limit   = "500m"
        }
        memory = {
          request = "128Mi"
          limit   = "256Mi"
        }
      }
    }

    # Monitoring integration — expose Prometheus metrics
    prometheusUrl = "http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090"
  })]

  depends_on = [helm_release.linkerd_crds]
}

# ─── Linkerd Viz Extension (Dashboard + Metrics) ─────────────────

resource "helm_release" "linkerd_viz" {
  name       = "linkerd-viz"
  namespace  = "linkerd-viz"
  repository = "https://helm.linkerd.io/stable"
  chart      = "linkerd-viz"
  version    = var.linkerd_viz_version

  create_namespace = true

  values = [yamlencode({
    # Use
