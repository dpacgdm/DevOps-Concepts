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
    # Use external Prometheus instead of Viz's built-in
    # (avoids running a second Prometheus instance)
    prometheusUrl = "http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090"
    prometheus = {
      enabled = false  # Don't install Viz's bundled Prometheus
    }

    # Grafana — use our external Grafana instance
    grafana = {
      enabled = false  # Don't install Viz's bundled Grafana
      externalUrl = "http://grafana.monitoring.svc.cluster.local:3000"
    }

    # Dashboard
    dashboard = {
      replicas = 2
      resources = {
        cpu = {
          request = "100m"
          limit   = "250m"
        }
        memory = {
          request = "128Mi"
          limit   = "256Mi"
        }
      }
      enforcedHostRegexp = "linkerd-viz\\.${var.cluster_name}\\.novamart\\.com"
    }

    # Tap — enables live traffic inspection (powerful for debugging)
    tap = {
      replicas = 2
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

    # Tap Injector
    tapInjector = {
      replicas = 2
      resources = {
        cpu = {
          request = "100m"
          limit   = "250m"
        }
        memory = {
          request = "64Mi"
          limit   = "128Mi"
        }
      }
    }

    # Metrics API
    metricsAPI = {
      replicas = 2
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

    # Default policy for Viz namespace
    defaultLogLevel  = "info"
    defaultLogFormat = "json"
  })]

  depends_on = [helm_release.linkerd_control_plane]
}
```

```hcl
# platform/modules/service-mesh/linkerd/outputs.tf

output "trust_anchor_expiry" {
  description = "Trust anchor certificate expiry — MUST rotate before this date"
  value       = module.trust_anchor.trust_anchor_expiry
}

output "issuer_expiry" {
  description = "Issuer certificate expiry — MUST rotate before this date"
  value       = module.trust_anchor.issuer_expiry
}

output "linkerd_version" {
  description = "Deployed Linkerd version"
  value       = var.linkerd_version
}

output "mesh_policy" {
  description = "Default mesh authorization policy"
  value       = "all-authenticated (mTLS required)"
}
```

---

### 5.3 Observability — Storage Buckets

```hcl
# platform/modules/observability/storage/main.tf

variable "environment" {
  type = string
}

variable "kms_key_arn" {
  type = string
}

data "aws_caller_identity" "current" {}

locals {
  name_prefix = "novamart-${var.environment}"
  account_id  = data.aws_caller_identity.current.account_id

  common_tags = {
    Component = "observability-storage"
  }
}

# ─── Thanos Long-Term Metrics Storage ────────────────────────────

resource "aws_s3_bucket" "thanos" {
  bucket = "${local.name_prefix}-thanos-${local.account_id}"

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-thanos"
    Purpose = "prometheus-long-term-metrics"
  })
}

resource "aws_s3_bucket_versioning" "thanos" {
  bucket = aws_s3_bucket.thanos.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "thanos" {
  bucket = aws_s3_bucket.thanos.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "thanos" {
  bucket = aws_s3_bucket.thanos.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "thanos" {
  bucket = aws_s3_bucket.thanos.id

  rule {
    id     = "compact-and-archive"
    status = "Enabled"

    # Thanos Compactor handles downsampling. S3 lifecycle handles tiering.
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
    transition {
      days          = 365
      storage_class = "GLACIER"
    }
    expiration {
      days = 730  # 2 years of metrics history
    }
  }
}

resource "aws_s3_bucket_policy" "thanos" {
  bucket = aws_s3_bucket.thanos.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyHTTP"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.thanos.arn,
          "${aws_s3_bucket.thanos.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}

# ─── Logs Long-Term Storage ──────────────────────────────────────

resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-app-logs-${local.account_id}"

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-app-logs"
    Purpose = "application-log-archive"
  })
}

resource "aws_s3_bucket_versioning" "logs" {
  bucket = aws_s3_bucket.logs.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "logs" {
  bucket = aws_s3_bucket.logs.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    id     = "archive-and-expire"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
    expiration {
      days = 365  # 1 year retention
    }
  }
}

resource "aws_s3_bucket_policy" "logs" {
  bucket = aws_s3_bucket.logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyHTTP"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.logs.arn,
          "${aws_s3_bucket.logs.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}

# ─── Tempo Trace Storage ─────────────────────────────────────────

resource "aws_s3_bucket" "traces" {
  bucket = "${local.name_prefix}-traces-${local.account_id}"

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-traces"
    Purpose = "distributed-trace-storage"
  })
}

resource "aws_s3_bucket_versioning" "traces" {
  bucket = aws_s3_bucket.traces.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "traces" {
  bucket = aws_s3_bucket.traces.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "traces" {
  bucket = aws_s3_bucket.traces.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "traces" {
  bucket = aws_s3_bucket.traces.id

  rule {
    id     = "archive-and-expire"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    expiration {
      days = 90  # 90 days of trace data
    }
  }
}

resource "aws_s3_bucket_policy" "traces" {
  bucket = aws_s3_bucket.traces.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyHTTP"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.traces.arn,
          "${aws_s3_bucket.traces.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}

# ─── Outputs ─────────────────────────────────────────────────────

output "thanos_bucket_id" {
  value = aws_s3_bucket.thanos.id
}

output "thanos_bucket_arn" {
  value = aws_s3_bucket.thanos.arn
}

output "logs_bucket_id" {
  value = aws_s3_bucket.logs.id
}

output "logs_bucket_arn" {
  value = aws_s3_bucket.logs.arn
}

output "traces_bucket_id" {
  value = aws_s3_bucket.traces.id
}

output "traces_bucket_arn" {
  value = aws_s3_bucket.traces.arn
}
```

---

### 5.4 Observability — Prometheus Stack

```hcl
# platform/modules/observability/prometheus/variables.tf

variable "environment" {
  type = string
}

variable "cluster_name" {
  type = string
}

variable "oidc_provider_arn" {
  type = string
}

variable "oidc_provider_url" {
  type = string
}

variable "kms_key_arn" {
  type = string
}

variable "thanos_s3_bucket" {
  description = "S3 bucket name for Thanos long-term storage"
  type        = string
}

variable "pagerduty_integration_key" {
  type      = string
  sensitive = true
}

variable "slack_webhook_url_warning" {
  type      = string
  sensitive = true
}

variable "slack_webhook_url_info" {
  type      = string
  sensitive = true
}

variable "prometheus_retention" {
  description = "Local Prometheus retention period"
  type        = string
  default     = "15d"
}

variable "prometheus_storage_size" {
  description = "Prometheus PV size"
  type        = string
  default     = "200Gi"
}

variable "chart_version" {
  description = "kube-prometheus-stack Helm chart version"
  type        = string
  default     = "57.1.1"
}
```

```hcl
# platform/modules/observability/prometheus/main.tf

locals {
  name_prefix = "novamart-${var.environment}"
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# ─── IRSA: Thanos Sidecar + Store Gateway + Compactor ────────────

resource "aws_iam_role" "thanos" {
  name_prefix = "${local.name_prefix}-thanos-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringLike = {
            "${var.oidc_provider_url}:sub" = "system:serviceaccount:monitoring:*thanos*"
            "${var.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      },
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${var.oidc_provider_url}:sub" = "system:serviceaccount:monitoring:prometheus-kube-prometheus-prometheus"
            "${var.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "thanos_s3" {
  name_prefix = "thanos-s3-"
  role        = aws_iam_role.thanos.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:ListBucket",
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:GetObjectTagging",
          "s3:PutObjectTagging"
        ]
        Resource = [
          "arn:aws:s3:::${var.thanos_s3_bucket}",
          "arn:aws:s3:::${var.thanos_s3_bucket}/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "kms:GenerateDataKey",
          "kms:Decrypt"
        ]
        Resource = var.kms_key_arn
      }
    ]
  })
}

# ─── Thanos Object Store Config Secret ───────────────────────────

resource "kubernetes_secret" "thanos_objstore" {
  metadata {
    name      = "thanos-objstore-config"
    namespace = "monitoring"
  }

  data = {
    "objstore.yml" = yamlencode({
      type = "S3"
      config = {
        bucket   = var.thanos_s3_bucket
        region   = data.aws_region.current.name
        endpoint = "s3.${data.aws_region.current.name}.amazonaws.com"
        # IRSA provides credentials — no static keys
      }
    })
  }
}

# ─── Alertmanager Secret (PagerDuty + Slack keys) ────────────────

resource "kubernetes_secret" "alertmanager_config" {
  metadata {
    name      = "alertmanager-secrets"
    namespace = "monitoring"
  }

  data = {
    pagerduty_key    = var.pagerduty_integration_key
    slack_url_warn   = var.slack_webhook_url_warning
    slack_url_info   = var.slack_webhook_url_info
  }
}

# ─── kube-prometheus-stack Helm Release ──────────────────────────

resource "helm_release" "kube_prometheus_stack" {
  name       = "prometheus"
  namespace  = "monitoring"
  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "kube-prometheus-stack"
  version    = var.chart_version

  timeout = 900  # 15 min — CRDs take time

  values = [
    file("${path.module}/values.yaml"),
  ]

  # Dynamic values that reference Terraform-managed resources
  set {
    name  = "prometheus.prometheusSpec.thanos.objectStorageConfig.existingSecret.name"
    value = kubernetes_secret.thanos_objstore.metadata[0].name
  }

  set {
    name  = "prometheus.prometheusSpec.thanos.objectStorageConfig.existingSecret.key"
    value = "objstore.yml"
  }

  set {
    name  = "prometheus.serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = aws_iam_role.thanos.arn
  }

  depends_on = [
    kubernetes_secret.thanos_objstore,
    kubernetes_secret.alertmanager_config,
  ]
}
```

```yaml
# platform/modules/observability/prometheus/values.yaml

# ═══════════════════════════════════════════════════════════════════
# kube-prometheus-stack Helm Values — NovaMart Production
# ═══════════════════════════════════════════════════════════════════

# ─── GLOBAL ──────────────────────────────────────────────────────
fullnameOverride: "prometheus"

defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: false              # EKS manages etcd — not our concern
    configReloaders: true
    general: true
    k8s: true
    kubeApiserverAvailability: true
    kubeApiserverBurnrate: true
    kubeApiserverHistogram: true
    kubeApiserverSlos: true
    kubeControllerManager: false  # EKS manages this
    kubelet: true
    kubeProxy: true
    kubePrometheusGeneral: true
    kubePrometheusNodeRecording: true
    kubeSchedulerAlerting: false  # EKS manages this
    kubeSchedulerRecording: false
    kubeStateMetrics: true
    network: true
    node: true
    nodeExporterAlerting: true
    nodeExporterRecording: true
    prometheus: true
    prometheusOperator: true

# ─── PROMETHEUS SERVER ───────────────────────────────────────────
prometheus:
  prometheusSpec:
    replicas: 2
    retention: 15d
    retentionSize: "180GB"

    # Storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3-encrypted
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 200Gi

    # Resources
    resources:
      requests:
        cpu: "1000m"
        memory: 4Gi
      limits:
        cpu: "2000m"
        memory: 8Gi

    # Pod anti-affinity — spread across AZs
    podAntiAffinity: "hard"
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: prometheus

    # Thanos sidecar for long-term storage
    thanos:
      image: quay.io/thanos/thanos:v0.34.1
      resources:
        requests:
          cpu: "200m"
          memory: 512Mi
        limits:
          cpu: "500m"
          memory: 1Gi

    # Scrape all ServiceMonitors across all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    serviceMonitorNamespaceSelector: {}
    podMonitorNamespaceSelector: {}
    ruleNamespaceSelector: {}

    # External labels for Thanos deduplication
    externalLabels:
      cluster: novamart-production
      environment: production

    # Scrape interval
    scrapeInterval: "30s"
    evaluationInterval: "30s"

    # Additional scrape configs for Linkerd
    additionalScrapeConfigs:
      - job_name: "linkerd-proxies"
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_container_name]
            action: keep
            regex: ^linkerd-proxy$
          - source_labels: [__meta_kubernetes_pod_container_port_name]
            action: keep
            regex: ^linkerd-admin$
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_label_linkerd_io_proxy_deployment]
            action: replace
            target_label: deployment

# ─── ALERTMANAGER ────────────────────────────────────────────────
alertmanager:
  alertmanagerSpec:
    replicas: 3
    podAntiAffinity: "hard"

    resources:
      requests:
        cpu: "100m"
        memory: 256Mi
      limits:
        cpu: "500m"
        memory: 512Mi

    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: alertmanager

    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3-encrypted
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

  config:
    global:
      resolve_timeout: 5m
      pagerduty_url: "https://events.pagerduty.com/v2/enqueue"

    # Inhibition rules — suppress noisy alerts during bigger incidents
    inhibit_rules:
      # If a node is down, suppress all pod alerts on that node
      - source_matchers:
          - severity = "critical"
          - alertname = "KubeNodeNotReady"
        target_matchers:
          - severity =~ "warning|info"
        equal: ["node"]

      # If cluster is unreachable, suppress all namespace-level alerts
      - source_matchers:
          - severity = "critical"
          - alertname = "KubeAPIDown"
        target_matchers:
          - severity =~ "warning|info"

      # If a deployment has zero replicas (intentional scale-down),
      # suppress pod-not-ready alerts
      - source_matchers:
          - alertname = "KubeDeploymentReplicasMismatch"
        target_matchers:
          - alertname = "KubePodNotReady"
        equal: ["namespace", "deployment"]

    route:
      receiver: "slack-info"  # Default catch-all
      group_by: ["namespace", "alertname"]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h

      routes:
        # Critical: PagerDuty (wake people up)
        - receiver: "pagerduty-critical"
          matchers:
            - severity = "critical"
          group_wait: 10s
          repeat_interval: 1h
          continue: true  # Also send to Slack for visibility

        # Critical: Also to Slack for visibility
        - receiver: "slack-warning"
          matchers:
            - severity = "critical"
          group_wait: 10s

        # Warning: Slack warning channel
        - receiver: "slack-warning"
          matchers:
            - severity = "warning"
          group_wait: 30s
          repeat_interval: 4h

        # Info: Slack info channel
        - receiver: "slack-info"
          matchers:
            - severity = "info"
          group_wait: 60s
          repeat_interval: 12h

        # Watchdog: heartbeat to ensure Alertmanager is alive
        - receiver: "pagerduty-heartbeat"
          matchers:
            - alertname = "Watchdog"
          repeat_interval: 1m

    receivers:
      - name: "pagerduty-critical"
        pagerduty_configs:
          - service_key_file: "/etc/alertmanager/secrets/alertmanager-secrets/pagerduty_key"
            severity: critical
            description: "{{ .CommonAnnotations.summary }}"
            details:
              firing: "{{ .Alerts.Firing | len }}"
              namespace: "{{ .CommonLabels.namespace }}"
              cluster: "novamart-production"

      - name: "pagerduty-heartbeat"
        pagerduty_configs:
          - service_key_file: "/etc/alertmanager/secrets/alertmanager-secrets/pagerduty_key"
            severity: info
            description: "Alertmanager Watchdog heartbeat"

      - name: "slack-warning"
        slack_configs:
          - api_url_file: "/etc/alertmanager/secrets/alertmanager-secrets/slack_url_warn"
            channel: "#alerts-warning"
            send_resolved: true
            title: '{{ template "slack.title" . }}'
            text: '{{ template "slack.text" . }}'
            color: '{{ if eq .Status "firing" }}warning{{ else }}good{{ end }}'

      - name: "slack-info"
        slack_configs:
          - api_url_file: "/etc/alertmanager/secrets/alertmanager-secrets/slack_url_info"
            channel: "#alerts-info"
            send_resolved: true
            title: '{{ template "slack.title" . }}'
            text: '{{ template "slack.text" . }}'

    templates:
      - "/etc/alertmanager/config/slack-templates.tmpl"

  # Mount the secrets
  alertmanagerSpec:
    secrets:
      - alertmanager-secrets

# ─── GRAFANA ─────────────────────────────────────────────────────
grafana:
  replicas: 2

  resources:
    requests:
      cpu: "500m"
      memory: 512Mi
    limits:
      cpu: "1000m"
      memory: 1Gi

  persistence:
    enabled: true
    storageClassName: gp3-encrypted
    size: 20Gi

  # Admin password managed via External Secrets Operator
  admin:
    existingSecret: grafana-admin-credentials
    userKey: admin-user
    passwordKey: admin-password

  # Grafana.ini configuration
  grafana.ini:
    server:
      root_url: "https://grafana.novamart.com"
    auth:
      disable_login_form: false
    auth.generic_oauth:
      enabled: false  # Enable when SSO is configured
    security:
      admin_user: admin
      cookie_secure: true
      strict_transport_security: true
    log:
      mode: console
      level: info
    log.console:
      format: json
    analytics:
      reporting_enabled: false
      check_for_updates: false

  # Datasources provisioned automatically
  additionalDataSources:
    - name: Tempo
      type: tempo
      uid: tempo
      url: http://tempo-query-frontend.tracing.svc.cluster.local:3100
      access: proxy
      jsonData:
        tracesToLogsV2:
          datasourceUid: cloudwatch
          filterByTraceID: true
        tracesToMetrics:
          datasourceUid: prometheus
        serviceMap:
          datasourceUid: prometheus
        nodeGraph:
          enabled: true

    - name: Thanos
      type: prometheus
      uid: thanos
      url: http://thanos-query.monitoring.svc.cluster.local:9090
      access: proxy
      jsonData:
        timeInterval: "30s"

    - name: CloudWatch
      type: cloudwatch
      uid: cloudwatch
      jsonData:
        authType: default
        defaultRegion: us-east-1
      # Uses IRSA — no static credentials needed

    - name: Alertmanager
      type: alertmanager
      uid: alertmanager
      url: http://prometheus-alertmanager.monitoring.svc.cluster.local:9093
      access: proxy

  # Dashboard sidecar — auto-loads dashboards from ConfigMaps
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
      searchNamespace: ALL
      folderAnnotation: grafana_folder
      provider:
        foldersFromFilesStructure: true
    datasources:
      enabled: true
      label: grafana_datasource
      labelValue: "1"

  # Topology spread
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: grafana

# ─── KUBE-STATE-METRICS ─────────────────────────────────────────
kubeStateMetrics:
  enabled: true

kube-state-metrics:
  replicas: 2
  resources:
    requests:
      cpu: "100m"
      memory: 256Mi
    limits:
      cpu: "250m"
      memory: 512Mi
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: kube-state-metrics

# ─── NODE EXPORTER ───────────────────────────────────────────────
nodeExporter:
  enabled: true

prometheus-node-exporter:
  resources:
    requests:
      cpu: "100m"
      memory: 128Mi
    limits:
      cpu: "250m"
      memory: 256Mi

# ─── PROMETHEUS OPERATOR ─────────────────────────────────────────
prometheusOperator:
  resources:
    requests:
      cpu: "200m"
      memory: 256Mi
    limits:
      cpu: "500m"
      memory: 512Mi

  # Admission webhooks for PrometheusRule validation
  admissionWebhooks:
    enabled: true
    certManager:
      enabled: true  # Use cert-manager for webhook certs

# ─── DISABLE COMPONENTS WE DON'T NEED ───────────────────────────
kubeControllerManager:
  enabled: false  # EKS-managed
kubeScheduler:
  enabled: false  # EKS-managed
kubeEtcd:
  enabled: false  # EKS-managed
```

---

### 5.5 Observability — Fluent Bit

```hcl
# platform/modules/observability/fluent-bit/variables.tf

variable "environment" {
  type = string
}

variable "cluster_name" {
  type = string
}

variable "oidc_provider_arn" {
  type = string
}

variable "oidc_provider_url" {
  type = string
}

variable "aws_region" {
  type = string
}

variable "logs_s3_bucket" {
  description = "S3 bucket for long-term log archival"
  type        = string
}

variable "chart_version" {
  type    = string
  default = "0.43.0"
}

variable "hot_retention_days" {
  description = "CloudWatch Logs retention in days"
  type        = number
  default     = 30
}
```

```hcl
# platform/modules/observability/fluent-bit/main.tf

locals {
  name_prefix = "novamart-${var.environment}"
}

data "aws_caller_identity" "current" {}

# ─── IRSA for Fluent Bit ─────────────────────────────────────────

resource "aws_iam_role" "fluent_bit" {
  name_prefix = "${local.name_prefix}-fluent-bit-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${var.oidc_provider_url}:sub" = "system:serviceaccount:logging:fluent-bit"
            "${var.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "fluent_bit" {
  name_prefix = "fluent-bit-"
  role        = aws_iam_role.fluent_bit.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Resource = "arn:aws:logs:${var.aws_region}:${data.aws_caller_identity.current.account_id}:log-group:/novamart/*"
      },
      {
        Effect = "Allow"
        Action = [
          "s3:PutObject",
          "s3:GetBucketLocation"
        ]
        Resource = [
          "arn:aws:s3:::${var.logs_s3_bucket}",
          "arn:aws:s3:::${var.logs_s3_bucket}/*"
        ]
      }
    ]
  })
}

# ─── CloudWatch Log Groups (pre-created for retention control) ───

resource "aws_cloudwatch_log_group" "app_logs" {
  for_each = toset([
    "/novamart/${var.environment}/payments",
    "/novamart/${var.environment}/orders",
    "/novamart/${var.environment}/users",
    "/novamart/${var.environment}/frontend",
    "/novamart/${var.environment}/shared",
    "/novamart/${var.environment}/platform",
  ])

  name              = each.value
  retention_in_days = each.value == "/novamart/${var.environment}/payments" ? 90 : var.hot_retention_days
  # PCI: payments logs retained 90 days in CloudWatch (compliance requirement)

  tags = {
    Environment = var.environment
    Purpose     = "application-logs"
    PCI         = each.value == "/novamart/${var.environment}/payments" ? "in-scope" : "out-of-scope"
  }
}

# ─── Fluent Bit Helm Release ─────────────────────────────────────

resource "helm_release" "fluent_bit" {
  name       = "fluent-bit"
  namespace  = "logging"
  repository = "https://fluent.github.io/helm-charts"
  chart      = "fluent-bit"
  version    = var.chart_version

  values = [templatefile("${path.module}/values.yaml", {
    aws_region     = var.aws_region
    cluster_name   = var.cluster_name
    environment    = var.environment
    s3_bucket      = var.logs_s3_bucket
    irsa_role_arn  = aws_iam_role.fluent_bit.arn
  })]
}
```

```yaml
# platform/modules/observability/fluent-bit/values.yaml

# ═══════════════════════════════════════════════════════════════════
# Fluent Bit — NovaMart Log Pipeline
#
# Architecture:
#   Pod stdout → containerd log file → Fluent Bit (DaemonSet)
#     → Parse JSON
#     → Enrich with Kubernetes metadata
#     → Route by namespace
#     → Output to CloudWatch (hot) + S3 (cold)
# ═══════════════════════════════════════════════════════════════════

serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: ${irsa_role_arn}

resources:
  requests:
    cpu: "200m"
    memory: 128Mi
  limits:
    cpu: "500m"
    memory: 256Mi

# Tolerate ALL taints — DaemonSet must run on every node
tolerations:
  - operator: Exists

# Priority class — log collection is important
priorityClassName: system-node-critical

# Liveness/readiness probes
livenessProbe:
  httpGet:
    path: /api/v1/health
    port: http
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /api/v1/health
    port: http
  initialDelaySeconds: 10
  periodSeconds: 10

# Monitoring — Fluent Bit exposes Prometheus metrics
serviceMonitor:
  enabled: true
  namespace: logging
  interval: 30s

config:
  # ─── SERVICE ────────────────────────────────────────────────────
  service: |
    [SERVICE]
        Daemon        Off
        Flush         5
        Log_Level     info
        Parsers_File  /fluent-bit/etc/parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020
        Health_Check  On
        storage.path  /var/fluent-bit/state/
        storage.sync  normal
        storage.checksum off
        storage.max_chunks_up 128

  # ─── INPUT ──────────────────────────────────────────────────────
  inputs: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        multiline.parser  cri
        DB                /var/fluent-bit/state/flb_kube.db
        DB.Sync           Normal
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        Refresh_Interval  10
        Rotate_Wait       30
        Read_from_Head    False

  # ─── FILTERS ────────────────────────────────────────────────────
  filters: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On
        Labels              On
        Annotations         Off
        Buffer_Size         32KB

    # Add cluster metadata to every log line
    [FILTER]
        Name    modify
        Match   kube.*
        Add     cluster ${cluster_name}
        Add     environment ${environment}
        Add     region ${aws_region}

    # Extract trace_id from log body if present (OTel correlation)
    [FILTER]
        Name          parser
        Match         kube.*
        Key_Name      log_processed
        Parser        extract_trace_id
        Reserve_Data  On
        Preserve_Key  On

    # Remove noisy kube-system logs (reduce CloudWatch costs)
    [FILTER]
        Name    grep
        Match   kube.*
        Exclude $kubernetes['namespace_name'] ^(kube-system)$

  # ─── OUTPUTS ────────────────────────────────────────────────────
  outputs: |
    # PCI-scoped: payments namespace → dedicated log group
    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.var.log.containers.novamart-payments*
        region              ${aws_region}
        log_group_name      /novamart/${environment}/payments
        log_stream_prefix   pod/
        auto_create_group   false
        log_key             log_processed
        log_format          json/emf
        retry_limit         5

    # Orders namespace
    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.var.log.containers.novamart-orders*
        region              ${aws_region}
        log_group_name      /novamart/${environment}/orders
        log_stream_prefix   pod/
        auto_create_group   false
        log_key             log_processed
        log_format          json/emf
        retry_limit         5

    # Users namespace
    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.var.log.containers.novamart-users*
        region              ${aws_region}
        log_group_name      /novamart/${environment}/users
        log_stream_prefix   pod/
        auto_create_group   false
        log_key             log_processed
        log_format          json/emf
        retry_limit         5

    # Frontend namespace
    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.var.log.containers.novamart-frontend*
        region              ${aws_region}
        log_group_name      /novamart/${environment}/frontend
        log_stream_prefix   pod/
        auto_create_group   false
        log_key             log_processed
        log_format          json/emf
        retry_limit         5

    # Platform services → single platform log group
    [OUTPUT]
        Name                cloudwatch_logs
        Match               kube.var.log.containers.*
        region              ${aws_region}
        log_group_name      /novamart/${environment}/platform
        log_stream_prefix   pod/
        auto_create_group   false
        log_key             log_processed
        log_format          json/emf
        retry_limit         5

    # ALL logs → S3 for long-term archival + Athena queries
    [OUTPUT]
        Name                s3
        Match               kube.*
        region              ${aws_region}
        bucket              ${s3_bucket}
        total_file_size     100M
        upload_timeout      5m
        s3_key_format       /year=%Y/month=%m/day=%d/hour=%H/$TAG/%Y%m%d%H%M%S-$UUID.json.gz
        compression         gzip
        content_type        application/gzip
        store_dir           /var/fluent-bit/state/s3/
        retry_limit         3

  # ─── PARSERS ────────────────────────────────────────────────────
  customParsers: |
    [PARSER]
        Name        extract_trace_id
        Format      regex
        Regex       (?<trace_id>([0-9a-f]{32}|[0-9a-f\-]{36}))
        Time_Key    Off
```

---

### 5.6 Observability — Tempo (Trace Storage)

```hcl
# platform/modules/observability/tempo/variables.tf

variable "environment" {
  type = string
}

variable "cluster_name" {
  type = string
}

variable "oidc_provider_arn" {
  type = string
}

variable "oidc_provider_url" {
  type = string
}

variable "traces_s3_bucket" {
  type = string
}

variable "chart_version" {
  type    = string
  default = "1.7.2"
}
```

```hcl
# platform/modules/observability/tempo/main.tf

locals {
  name_prefix = "novamart-${var.environment}"
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# ─── IRSA for Tempo ──────────────────────────────────────────────

resource "aws_iam_role" "tempo" {
  name_prefix = "${local.name_prefix}-tempo-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringLike = {
            "${var.oidc_provider_url}:sub" = "system:serviceaccount:tracing:tempo*"
            "${var.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "tempo_s3" {
  name_prefix = "tempo-s3-"
  role        = aws_iam_role.tempo.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:ListBucket",
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = [
          "arn:aws:s3:::${var.traces_s3_bucket}",
          "arn:aws:s3:::${var.traces_s3_bucket}/*"
        ]
      }
    ]
  })
}

# ─── Tempo Helm Release (Distributed Mode) ───────────────────────

resource "helm_release" "tempo" {
  name       = "tempo"
  namespace  = "tracing"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "tempo-distributed"
  version    = var.chart_version

  values = [yamlencode({
    # Service account with IRSA
    serviceAccount = {
      annotations = {
        "eks.amazonaws.com/role-arn" = aws_iam_role.tempo.arn
      }
    }

    # S3 backend for traces
    storage = {
      trace = {
        backend = "s3"
        s3 = {
          bucket   = var.traces_s3_bucket
          region   = data.aws_region.current.name
          endpoint = "s3.${data.aws_region.current.name}.amazonaws.com"
        }
      }
    }

    # Distributor (receives traces from OTel Collector)
    distributor = {
      replicas = 3
      resources = {
        requests = { cpu = "500m", memory = "512Mi" }
        limits   = { cpu = "1",    memory = "1Gi" }
      }
      topologySpreadConstraints = [{
        maxSkew           = 1
        topologyKey       = "topology.kubernetes.io/zone"
        whenUnsatisfiable = "DoNotSchedule"
        labelSelector = {
          matchLabels = {
            "app.kubernetes.io/component" = "distributor"
          }
        }
      }]
    }

    # Ingester (writes to S3)
    ingester = {
      replicas = 3
      resources = {
        requests = { cpu = "500m", memory = "1Gi" }
        limits   = { cpu = "1",    memory = "2Gi" }
      }
      persistence = {
        enabled      = true
        size         = "20Gi"
        storageClass = "gp3-encrypted"
      }
      topologySpreadConstraints = [{
        maxSkew           = 1
        topologyKey       = "topology.kubernetes.io/zone"
        whenUnsatisfiable = "DoNotSchedule"
        labelSelector = {
          matchLabels = {
            "app.kubernetes.io/component" = "ingester"
          }
        }
      }]
    }

    # Querier (reads from S3 for Grafana queries)
    querier = {
      replicas = 2
      resources = {
        requests = { cpu = "500m", memory = "512Mi" }
        limits   = { cpu = "1",    memory = "1Gi" }
      }
    }

    # Query Frontend (caches query results)
    queryFrontend = {
      replicas = 2
      resources = {
        requests = { cpu = "250m", memory = "256Mi" }
        limits   = { cpu = "500m", memory = "512Mi" }
      }
    }

    # Compactor (compacts blocks in S3)
    compactor = {
      replicas = 1
      resources = {
        requests = { cpu = "500m", memory = "1Gi" }
        limits   = { cpu = "1",    memory = "2Gi" }
      }
    }

    # Retention
    overrides = {
      defaults = {
        global = {
          max_traces_per_user = 0  # No limit per tenant
        }
      }
    }

    # Metrics — expose Prometheus metrics for self-monitoring
    metricsGenerator = {
      enabled = true
      remoteWriteUrl = "http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090/api/v1/write"
    }

    # Global settings
    global = {
      clusterDomain = "cluster.local"
    }
  })]
}

# ─── Outputs ─────────────────────────────────────────────────────

output "distributor_endpoint" {
  description = "Tempo distributor endpoint for OTel Collector"
  value       = "tempo-distributor.tracing.svc.cluster.local:4317"
}
```

---

### 5.7 Observability — OTel Collector

```hcl
# platform/modules/observability/otel-collector/variables.tf

variable "environment" {
  type = string
}

variable "cluster_name" {
  type = string
}

variable "oidc_provider_arn" {
  type = string
}

variable "oidc_provider_url" {
  type = string
}

variable "tempo_endpoint" {
  description = "Tempo gRPC endpoint for trace export"
  type        = string
}

variable "xray_region" {
  type    = string
  default = "us-east-1"
}

variable "chart_version" {
  type    = string
  default = "0.84.0"
}
```

```hcl
# platform/modules/observability/otel-collector/main.tf

locals {
  name_prefix = "novamart-${var.environment}"
}

data "aws_caller_identity" "current" {}

# ─── IRSA for OTel Collector (X-Ray export) ──────────────────────

resource "aws_iam_role" "otel_collector" {
  name_prefix = "${local.name_prefix}-otel-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringLike = {
            "${var.oidc_provider_url}:sub" = "system:serviceaccount:logging:otel-*"
            "${var.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "otel_xray" {
  name_prefix = "otel-xray-"
  role        = aws_iam_role.otel_collector.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "xray:PutTraceSegments",
          "xray:PutTelemetryRecords",
          "xray:GetSamplingRules",
          "xray:GetSamplingTargets"
        ]
        Resource = "*"
      }
    ]
  })
}

# ─── OTel Collector — Agent (DaemonSet) ──────────────────────────

resource "helm_release" "otel_agent" {
  name       = "otel-agent"
  namespace  = "logging"
  repository = "https://open-telemetry.github.io/opentelemetry-helm-charts"
  chart      = "opentelemetry-collector"
  version    = var.chart_version

  values = [yamlencode({
    mode = "daemonset"

    serviceAccount = {
      annotations = {
        "eks.amazonaws.com/role-arn" = aws_iam_role.otel_collector.arn
      }
    }

    resources = {
      requests = { cpu = "200m", memory = "256Mi" }
      limits   = { cpu = "500m", memory = "512Mi" }
    }

    tolerations = [{ operator = "Exists" }]
    priorityClassName = "system-node-critical"

    # Ports
    ports = {
      otlp = {
        enabled     = true
        containerPort = 4317
        servicePort = 4317
        protocol    = "TCP"
      }
      otlp-http = {
        enabled     = true
        containerPort = 4318
        servicePort = 4318
        protocol    = "TCP"
      }
      prometheus = {
        enabled       = true
        containerPort = 8888
        servicePort   = 8888
        protocol      = "TCP"
      }
    }

    config = {
      receivers = {
        otlp = {
          protocols = {
            grpc = { endpoint = "0.0.0.0:4317" }
            http = { endpoint = "0.0.0.0:4318" }
          }
        }
      }

      processors = {
        batch = {
          timeout      = "5s"
          send_batch_size = 1024
        }

        memory_limiter = {
          check_interval  = "1s"
          limit_mib       = 400
          spike_limit_mib = 100
        }

        resource = {
          attributes = [
            {
              key    = "k8s.cluster.name"
              value  = var.cluster_name
              action = "upsert"
            },
            {
              key    = "deployment.environment"
              value  = var.environment
              action = "upsert"
            }
          ]
        }

        # Tail-based sampling at the agent level
        # (lightweight — full sampling at gateway)
        probabilistic_sampler = {
          sampling_percentage = 100  # Send everything to gateway
        }
      }

      exporters = {
        otlp = {
          endpoint = "otel-gateway.logging.svc.cluster.local:4317"
          tls = { insecure = true }  # In-cluster communication
        }
      }

      service = {
        pipelines = {
          traces = {
            receivers  = ["otlp"]
            processors = ["memory_limiter", "resource", "batch"]
            exporters  = ["otlp"]
          }
          metrics = {
            receivers  = ["otlp"]
            processors = ["memory_limiter", "resource", "batch"]
            exporters  = ["otlp"]
          }
        }

        telemetry = {
          metrics = { address = "0.0.0.0:8888" }
          logs    = { level = "info", encoding = "json" }
        }
      }
    }

    serviceMonitor = {
      enabled  = true
      interval = "30s"
    }
  })]
}

# ─── OTel Collector — Gateway (Deployment) ───────────────────────

resource "helm_release" "otel_gateway" {
  name       = "otel-gateway"
  namespace  = "logging"
  repository = "https://open-telemetry.github.io/opentelemetry-helm-charts"
  chart      = "opentelemetry-collector"
  version    = var.chart_version

  values = [yamlencode({
    mode = "deployment"
    replicaCount = 3

    serviceAccount = {
      annotations = {
        "eks.amazonaws.com/role-arn" = aws_iam_role.otel_collector.arn
      }
    }

    resources = {
      requests = { cpu = "500m",  memory = "1Gi" }
      limits   = { cpu = "1000m", memory = "2Gi" }
    }

    topologySpreadConstraints = [{
      maxSkew           = 1
      topologyKey       = "topology.kubernetes.io/zone"
      whenUnsatisfiable = "DoNotSchedule"
      labelSelector = {
        matchLabels = { "app.kubernetes.io/name" = "otel-gateway" }
      }
    }]

    ports = {
      otlp = {
        enabled       = true
        containerPort = 4317
        servicePort   = 4317
        protocol      = "TCP"
      }
      prometheus = {
        enabled       = true
        containerPort = 8888
        servicePort   = 8888
        protocol      = "TCP"
      }
    }

    config = {
      receivers = {
        otlp = {
          protocols = {
            grpc = { endpoint = "0.0.0.0:4317" }
          }
        }
      }

      processors = {
        batch = {
          timeout         = "10s"
          send_batch_size = 2048
        }

        memory_limiter = {
          check_interval  = "1s"
          limit_mib       = 1500
          spike_limit_mib = 300
        }

        # Tail-based sampling at the gateway (decision point)
        tail_sampling = {
          decision_wait = "10s"
          num_traces    = 50000
          expected_new_traces_per_sec = 1000
          policies = [
            {
              # Always keep error traces
              name = "errors-always"
              type = "status_code"
              status_code = { status_codes = ["ERROR"] }
            },
            {
              # Always keep slow traces (>2s)
              name = "slow-traces"
              type = "latency"
              latency = { threshold_ms = 2000 }
            },
            {
              # Sample 10% of successful traces
              name = "success-sampling"
              type = "probabilistic"
              probabilistic = { sampling_percentage = 10 }
            },
            {
              # Keep 1% of health checks (very noisy)
              name = "health-check-sampling"
              type = "and"
              and = {
                and_sub_policy = [
                  {
                    name = "health-path"
                    type = "string_attribute"
                    string_attribute = {
                      key    = "http.target"
                      values = ["/health", "/healthz", "/ready", "/readyz"]
                    }
                  },
                  {
                    name = "sample-1pct"
                    type = "probabilistic"
                    probabilistic = { sampling_percentage = 1 }
                  }
                ]
              }
            }
          ]
        }

        resource = {
          attributes = [
            {
              key    = "collector.tier"
              value  = "gateway"
              action = "upsert"
            }
          ]
        }
      }

      exporters = {
        # Primary: Grafana Tempo
        otlp_tempo = {
          endpoint = var.tempo_endpoint
          tls = { insecure = true }
        }

        # Secondary: AWS X-Ray
        awsxray = {
          region = var.xray_region
        }
      }

      service = {
        pipelines = {
          traces = {
            receivers  = ["otlp"]
            processors = ["memory_limiter", "tail_sampling", "resource", "batch"]
            exporters  = ["otlp_tempo", "awsxray"]
          }
          metrics = {
            receivers  = ["otlp"]
            processors = ["memory_limiter", "resource", "batch"]
            exporters  = ["otlp_tempo"]  # Metrics to Tempo for exemplars
          }
        }

        telemetry = {
          metrics = { address = "0.0.0.0:8888" }
          logs    = { level = "info", encoding = "json" }
        }
      }
    }

    serviceMonitor = {
      enabled  = true
      interval = "30s"
    }
  })]
}
```

---

### 5.8 Security — External Secrets Operator

```hcl
# platform/modules/security/external-secrets/main.tf

variable "environment" {
  type = string
}

variable "irsa_role_arn" {
  description = "IRSA role ARN for ESO (created in Lesson 1 EKS module)"
  type        = string
}

variable "aws_region" {
  type = string
}

variable "chart_version" {
  type    = string
  default = "0.9.13"
}

# ─── External Secrets Operator Helm Release ──────────────────────

resource "helm_release" "external_secrets" {
  name       = "external-secrets"
  namespace  = "external-secrets"
  repository = "https://charts.external-secrets.io"
  chart      = "external-secrets"
  version    = var.chart_version

  values = [yamlencode({
    replicaCount = 2

    serviceAccount = {
      create = true
      annotations = {
        "eks.amazonaws.com/role-arn" = var.irsa_role_arn
      }
    }

    resources = {
      requests = { cpu = "100m", memory = "128Mi" }
      limits   = { cpu = "250m", memory = "256Mi" }
    }

    webhook = {
      replicaCount = 2
      resources = {
        requests = { cpu = "50m",  memory = "64Mi" }
        limits   = { cpu = "100m", memory = "128Mi" }
      }
    }

    certController = {
      replicaCount = 1
      resources = {
        requests = { cpu = "50m",  memory = "64Mi" }
        limits   = { cpu = "100m", memory = "128Mi" }
      }
    }

    # Topology spread for HA
    topologySpreadConstraints = [{
      maxSkew           = 1
      topologyKey       = "topology.kubernetes.io/zone"
      whenUnsatisfiable = "DoNotSchedule"
      labelSelector = {
        matchLabels = { "app.kubernetes.io/name" = "external-secrets" }
      }
    }]

    # Prometheus metrics
    serviceMonitor = {
      enabled   = true
      interval  = "30s"
      namespace = "external-secrets"
    }

    # Pod disruption budget
    podDisruptionBudget = {
      enabled      = true
      minAvailable = 1
    }
  })]
}

# ─── ClusterSecretStore — AWS Secrets Manager ────────────────────

resource "kubectl_manifest" "cluster_secret_store" {
  yaml_body = yamlencode({
    apiVersion = "external-secrets.io/v1beta1"
    kind       = "ClusterSecretStore"
    metadata = {
      name = "aws-secrets-manager"
    }
    spec = {
      provider = {
        aws = {
          service = "SecretsManager"
          region  = var.aws_region
          auth = {
            jwt = {
              serviceAccountRef = {
                name      = "external-secrets"
                namespace = "external-secrets"
              }
            }
          }
        }
      }
      # Conditions — only allow access from specific namespaces
      conditions = [
        {
          namespaces = [
            "novamart-payments",
            "novamart-orders",
            "novamart-users",
            "novamart-frontend",
            "novamart-shared",
            "monitoring",
          ]
        }
      ]
    }
  })

  depends_on = [helm_release.external_secrets]
}

# ─── ClusterSecretStore — AWS Parameter Store ────────────────────

resource "kubectl_manifest" "cluster_secret_store_ssm" {
  yaml_body = yamlencode({
    apiVersion = "external-secrets.io/v1beta1"
    kind       = "ClusterSecretStore"
    metadata = {
      name = "aws-parameter-store"
    }
    spec = {
      provider = {
        aws = {
          service = "ParameterStore"
          region  = var.aws_region
          auth = {
            jwt = {
              serviceAccountRef = {
                name      = "external-secrets"
                namespace = "external-secrets"
              }
            }
          }
        }
      }
      conditions = [
        {
          namespaces = [
            "novamart-payments",
            "novamart-orders",
            "novamart-users",
            "novamart-frontend",
            "novamart-shared",
          ]
        }
      ]
    }
  })

  depends_on = [helm_release.external_secrets]
}

# ─── Example: Grafana Admin Credentials (ESO-managed) ────────────
# The actual secret must exist in AWS Secrets Manager at:
#   novamart/production/grafana/admin-credentials

resource "kubectl_manifest" "grafana_admin_secret" {
  yaml_body = yamlencode({
    apiVersion = "external-secrets.io/v1beta1"
    kind       = "ExternalSecret"
    metadata = {
      name      = "grafana-admin-credentials"
      namespace = "monitoring"
    }
    spec = {
      refreshInterval = "1h"
      secretStoreRef = {
        name = "aws-secrets-manager"
        kind = "ClusterSecretStore"
      }
      target = {
        name           = "grafana-admin-credentials"
        creationPolicy = "Owner"
      }
      data = [
        {
          secretKey = "admin-user"
          remoteRef = {
            key      = "novamart/production/grafana/admin-credentials"
            property = "username"
          }
        },
        {
          secretKey = "admin-password"
          remoteRef = {
            key      = "novamart/production/grafana/admin-credentials"
            property = "password"
          }
        }
      ]
    }
  })

  depends_on = [kubectl_manifest.cluster_secret_store]
}

# ─── Outputs ─────────────────────────────────────────────────────

output "cluster_secret_store_name" {
  value = "aws-secrets-manager"
}

output "parameter_store_name" {
  value = "aws-parameter-store"
}
```

---

### 5.9 Security — Kyverno

```hcl
# platform/modules/security/kyverno/variables.tf

variable "environment" {
  type = string
}

variable "ecr_registry" {
  description = "Allowed ECR registry URL"
  type        = string
}

variable "chart_version" {
  type    = string
  default = "3.1.4"
}

variable "enforcement_mode" {
  description = "Kyverno validation failure action: Enforce or Audit"
  type        = string
  default     = "Enforce"
  validation {
    condition     = contains(["Enforce", "Audit"], var.enforcement_mode)
    error_message = "Must be Enforce or Audit."
  }
}
```

```hcl
# platform/modules/security/kyverno/main.tf

locals {
  name_prefix = "novamart-${var.environment}"

  # Namespaces exempt from most policies (platform infrastructure)
  exempt_namespaces = [
    "kube-system",
    "kube-node-lease",
    "kube-public",
    "linkerd",
    "linkerd-viz",
    "monitoring",
    "logging",
    "tracing",
    "kyverno",
    "cert-manager",
    "external-secrets",
  ]
}

# ─── Kyverno Helm Release ────────────────────────────────────────

resource "helm_release" "kyverno" {
  name       = "kyverno"
  namespace  = "kyverno"
  repository = "https://kyverno.github.io/kyverno"
  chart      = "kyverno"
  version    = var.chart_version

  values = [yamlencode({
    replicaCount = 3

    resources = {
      requests = { cpu = "500m",  memory = "512Mi" }
      limits   = { cpu = "1000m", memory = "1Gi" }
    }

    # Background scanning — audit existing resources
    backgroundController = {
      enabled = true
      resources = {
        requests = { cpu = "100m", memory = "128Mi" }
        limits   = { cpu = "250m", memory = "256Mi" }
      }
    }

    # Reports controller
    reportsController = {
      enabled = true
      resources = {
        requests = { cpu = "100m", memory = "128Mi" }
        limits   = { cpu = "250m", memory = "256Mi" }
      }
    }

    # Cleanup controller
    cleanupController = {
      enabled = true
      resources = {
        requests = { cpu = "100m", memory = "128Mi" }
        limits   = { cpu = "250m", memory = "256Mi" }
      }
    }

    # Admission controller webhook configuration
    webhookConfiguration = {
      failurePolicy = "Fail"   # If Kyverno is down, BLOCK deployments
      # In production, this is safer than Ignore (which would allow
      # unvalidated resources to slip through)
    }

    # Topology spread
    topologySpreadConstraints = [{
      maxSkew           = 1
      topologyKey       = "topology.kubernetes.io/zone"
      whenUnsatisfiable = "DoNotSchedule"
      labelSelector = {
        matchLabels = { "app.kubernetes.io/name" = "kyverno" }
      }
    }]

    # Pod disruption budget — always keep at least 2 replicas
    podDisruptionBudget = {
      minAvailable = 2
    }

    # Monitoring
    serviceMonitor = {
      enabled  = true
      interval = "30s"
    }

    # Exclude system namespaces from webhook
    config = {
      webhooks = [{
        namespaceSelector = {
          matchExpressions = [{
            key      = "kubernetes.io/metadata.name"
            operator = "NotIn"
            values   = ["kube-system", "kyverno"]
          }]
        }
      }]
      # Resource filters — don't scan these resources
      resourceFilters = [
        "[Event,*,*]",
        "[*,kube-system,*]",
        "[*,kube-public,*]",
        "[*,kube-node-lease,*]",
        "[Node,*,*]",
        "[APIService,*,*]",
        "[TokenReview,*,*]",
        "[SubjectAccessReview,*,*]",
        "[SelfSubjectAccessReview,*,*]",
        "[Binding,*,*]",
        "[ReplicaSet,*,*]",
        "[ClusterRole,*,*]",
        "[ClusterRoleBinding,*,*]",
        "[ServiceAccount,*,*]",
      ]
    }
  })]
}

# ═══════════════════════════════════════════════════════════════════
# KYVERNO POLICIES
# ═══════════════════════════════════════════════════════════════════

# ─── Policy 1: Disallow Privileged Containers ────────────────────

resource "kubectl_manifest" "disallow_privileged" {
  yaml_body = yamlencode({
    apiVersion = "kyverno.io/v1"
    kind       = "ClusterPolicy"
    metadata = {
      name = "disallow-privileged-containers"
      annotations = {
        "policies.kyverno.io/title"       = "Disallow Privileged Containers"
        "policies.kyverno.io/category"    = "Pod Security Standards (Baseline)"
        "policies.kyverno.io/severity"    = "high"
        "policies.kyverno.io/description" = "Privileged containers can access all host resources. This policy prevents their creation."
      }
    }
    spec = {
      validationFailureAction = var.enforcement_mode
      background = true
      rules = [{
        name = "deny-privileged"
        match = {
          any = [{
            resources = { kinds = ["Pod"] }
          }]
        }
        exclude = {
          any = [{
            resources = {
              namespaces = local.exempt_namespaces
            }
          }]
        }
        validate = {
          message = "Privileged containers are not allowed. Set securityContext.privileged to false."
          pattern = {
            spec = {
              containers = [{
                securityContext = {
                  privileged = "false"
                }
              }]
              initContainers = [{
                securityContext = {
                  privileged = "false"
                }
              }]
            }
          }
        }
      }]
    }
  })

  depends_on = [helm_release.kyverno]
}

# ─── Policy 2: Disallow Latest Tag ──────────────────────────────

resource "kubectl_manifest" "disallow_latest_tag" {
  yaml_body = yamlencode({
    apiVersion = "kyverno.io/v1"
    kind       = "ClusterPolicy"
    metadata = {
      name = "disallow-latest-tag"
      annotations = {
        "policies.kyverno.io/title"       = "Disallow Latest Tag"
        "policies.kyverno.io/category"    = "Best Practices"
        "policies.kyverno.io/severity"    = "medium"
        "policies.kyverno.io/description" = "The ':latest' tag is mutable and non-deterministic. Require explicit image tags or digests."
      }
    }
    spec = {
      validationFailureAction = var.enforcement_mode
      background = true
      rules = [{
        name = "require-explicit-tag"
        match = {
          any = [{
            resources = { kinds = ["Pod"] }
          }]
        }
        exclude = {
          any = [{
            resources = {
              namespaces = local.exempt_namespaces
            }
          }]
        }
        validate = {
          message = "Image tag ':latest' is not allowed. Use a specific version tag or SHA digest."
          pattern = {
            spec = {
              containers = [{
                image = "!*:latest & *:*"
              }]
              initContainers = [{
                image = "!*:latest & *:*"
              }]
            }
          }
        }
      }]
    }
  })

  depends_on = [helm_release.kyverno]
}

# ─── Policy 3: Require Resource Limits ──────────────────────────

resource "kubectl_manifest" "require_resource_limits" {
  yaml_body = yamlencode({
    apiVersion = "kyverno.io/v1"
    kind       = "ClusterPolicy"
    metadata = {
      name = "require-resource-limits"
      annotations = {
        "policies.kyverno.io/title"       = "Require Resource Limits"
        "policies.kyverno.io/category"    = "Best Practices"
        "policies.kyverno.io/severity"    = "medium"
        "policies.kyverno.io/description" = "Every container must define CPU and memory requests and limits to prevent resource starvation."
      }
    }
    spec = {
      validationFailureAction = var.enforcement_mode
      background = true
      rules = [{
        name = "require-limits"
        match = {
          any = [{
            resources = { kinds = ["Pod"] }
          }]
        }
        exclude = {
          any = [{
            resources = {
              namespaces = local.exempt_namespaces
            }
          }]
        }
        validate = {
          message = "CPU and memory requests/limits are required for all containers."
          pattern = {
            spec = {
              containers = [{
                resources = {
                  requests = {
                    cpu    = "?*"
                    memory = "?*"
                  }
                  limits = {
                    cpu    = "?*"
                    memory = "?*"
                  }
                }
              }]
            }
          }
        }
      }]
    }
  })

  depends_on = [helm_release.kyverno]
}

# ─── Policy 4: Require Labels ───────────────────────────────────

resource "kubectl_manifest" "require_labels" {
  yaml_body = yamlencode({
    apiVersion = "kyverno.io/v1"
    kind       = "ClusterPolicy"
    metadata = {
      name = "require-labels"
      annotations = {
        "policies.kyverno.io/title"       = "Require Standard Labels"
        "policies.kyverno.io/category"    = "Best Practices"
        "policies.kyverno.io/severity"    = "medium"
        "policies.kyverno.io/description" = "All Deployments must have team, service, and version labels for ownership and observability."
      }
    }
    spec = {
      validationFailureAction = var.enforcement_mode
      background = true
      rules = [{
        name = "require-team-and-service"
        match = {
          any = [{
            resources = { kinds = ["Deployment", "StatefulSet", "DaemonSet"] }
          }]
        }
        exclude = {
          any = [{
            resources = {
              namespaces = local.exempt_namespaces
            }
          }]
        }
        validate = {
          message = "Labels 'novamart.com/team', 'novamart.com/service', and 'app.kubernetes.io/version' are required."
          pattern = {
            metadata = {
              labels = {
                "novamart.com/team"           = "?*"
                "novamart.com/service"        = "?*"
                "app.kubernetes.io/version"   = "?*"
              }
            }
          }
        }
      }]
    }
  })

  depends_on = [helm_release.kyverno]
}

# ─── Policy 5: Require Security Context ─────────────────────────

resource "kubectl_manifest" "require_security_context" {
  yaml_body = yamlencode({
    apiVersion = "kyverno.io/v1"
    kind       = "ClusterPolicy"
    metadata = {
      name = "require-run-as-non-root"
      annotations = {
        "policies.kyverno.io/title"       = "Require Non-Root User"
        "policies.kyverno.io/category"    = "Pod Security Standards (Restricted)"
        "policies.kyverno.io/severity"    = "high"
        "policies.kyverno.io/description" = "Containers must run as non-root and drop all capabilities."
      }
    }
    spec = {
      validationFailureAction = var.enforcement_mode
      background = true
      rules = [{
        name = "require-non-root"
        match = {
          any = [{
            resources = { kinds = ["Pod"] }
          }]
        }
        exclude = {
          any = [{
            resources = {
              namespaces = local.exempt_namespaces
            }
          }]
        }
        validate = {
          message = "Containers must run as non-root with readOnlyRootFilesystem and dropped capabilities."
          pattern = {
            spec = {
              securityContext = {
                runAsNonRoot = true
              }
              containers = [{
                securityContext = {
                  runAsNonRoot             = true
                  readOnlyRootFilesystem   = true
                  allowPrivilegeEscalation = false
                  capabilities = {
                    drop = ["ALL"]
                  }
                }
              }]
            }
          }
        }
      }]
    }
  })

  depends_on = [helm_release.kyverno]
}

# ─── Policy 6: Disallow HostPath Mounts ─────────────────────────

resource "kubectl_manifest" "disallow_host_path" {
  yaml_body = yamlencode({
    apiVersion = "kyverno.io/v1"
    kind       = "ClusterPolicy"
    metadata = {
      name = "disallow-host-path"
      annotations = {
        "policies.kyverno.io/title"       = "Disallow HostPath Volumes"
        "policies.kyverno.io/category"    = "Pod Security Standards (Baseline)"
        "policies.kyverno.io/severity"    = "high"
        "policies.kyverno.io/description" = "HostPath volumes allow containers to access the host filesystem, enabling container escape."
      }
    }
    spec = {
      validationFailureAction = var.enforcement_mode
      background = true
      rules = [{
        name = "deny-host-path"
        match = {
          any = [{
            resources = { kinds = ["Pod"] }
          }]
        }
        exclude = {
          any = [{
            resources = {
              namespaces = local.exempt_namespaces
            }
          }]
        }
        validate = {
          message = "HostPath volumes are not allowed."
          deny = {
            conditions = {
              any = [{
                key      = "{{ request.object.spec.volumes[?hostPath] | length(@) }}"
                operator = "GreaterThan"
                value    = 0
              }]
            }
          }
        }
      }]
    }
  })

  depends_on = [helm_release.kyverno]
}

# ─── Policy 7: Restrict Image Registries ────────────────────────

resource "kubectl_manifest" "restrict_registries" {
  yaml_body = yamlencode({
    apiVersion = "kyverno.io/v1"
    kind       = "ClusterPolicy"
    metadata = {
      name = "restrict-image-registries"
      annotations = {
        "policies.kyverno.io/title"       = "Restrict Image Registries"
        "policies.kyverno.io/category"    = "Supply Chain Security"
        "policies.kyverno.io/severity"    = "high"
        "policies.kyverno.io/description" = "Images may only come from approved registries (ECR, trusted public registries)."
      }
    }
    spec = {
      validationFailureAction = var.enforcement_mode
      background = true
      rules = [{
        name = "allowed-registries"
        match = {
          any = [{
            resources = { kinds = ["Pod"] }
          }]
        }
        exclude = {
          any = [{
            resources = {
              namespaces = local.exempt_namespaces
            }
          }]
        }
        validate = {
          message = "Images must come from approved registries: ECR (${var.ecr_registry}), or specific trusted public registries."
          pattern = {
            spec = {
              containers = [{
                image = "${var.ecr_registry}/* | docker.io/library/* | ghcr.io/linkerd/* | quay.io/prometheus/* | quay.io/thanos/* | grafana/*"
              }]
              initContainers = [{
                image = "${var.ecr_registry}/* | docker.io/library/* | ghcr.io/linkerd/* | quay.io/prometheus/* | quay.io/thanos/* | grafana/*"
              }]
            }
          }
        }
      }]
    }
  })

  depends_on = [helm_release.kyverno]
}

# ─── Mutation Policy: Auto-inject Labels ─────────────────────────

resource "kubectl_manifest" "mutate_default_security_context" {
  yaml_body = yamlencode({
    apiVersion = "kyverno.io/v1"
    kind       = "ClusterPolicy"
    metadata = {
      name = "mutate-default-security-context"
      annotations = {
        "policies.kyverno.io/title"       = "Default Security Context"
        "policies.kyverno.io/category"    = "Best Practices"
        "policies.kyverno.io/description" = "Auto-inject a secure default securityContext if one is not provided."
      }
    }
    spec = {
      rules = [{
        name = "set-default-security-context"
        match = {
          any = [{
            resources = { kinds = ["Pod"] }
          }]
        }
        exclude = {
          any = [{
            resources = {
              namespaces = local.exempt_namespaces
            }
          }]
        }
        mutate = {
          patchStrategicMerge = {
            spec = {
              securityContext = {
                "+(runAsNonRoot)"  = true
                "+(seccompProfile)" = {
                  type = "RuntimeDefault"
                }
              }
              containers = [{
                "(name)" = "*"
                securityContext = {
                  "+(allowPrivilegeEscalation)" = false
                  "+(readOnlyRootFilesystem)"   = true
                  "+(capabilities)" = {
                    drop = ["ALL"]
                  }
                }
              }]
            }
          }
        }
      }]
    }
  })

  depends_on = [helm_release.kyverno]
}

# ─── Generation Policy: Auto-create NetworkPolicy on new NS ─────

resource "kubectl_manifest" "generate_default_deny" {
  yaml_body = yamlencode({
    apiVersion = "kyverno.io/v1"
    kind       = "ClusterPolicy"
    metadata = {
      name = "generate-default-deny-network-policy"
      annotations = {
        "policies.kyverno.io/title"       = "Generate Default Deny NetworkPolicy"
        "policies.kyverno.io/category"    = "Network Security"
        "policies.kyverno.io/description" = "Automatically creates a default-deny NetworkPolicy when a new namespace with the novamart.com/tier=application label is created."
      }
    }
    spec = {
      rules = [{
        name = "default-deny-on-ns-create"
        match = {
          any = [{
            resources = {
              kinds = ["Namespace"]
              selector = {
                matchLabels = {
                  "novamart.com/tier" = "application"
                }
              }
            }
          }]
        }
        generate = {
          synchronize = true
          apiVersion  = "networking.k8s.io/v1"
          kind        = "NetworkPolicy"
          name        = "default-deny-all"
          namespace   = "{{request.object.metadata.name}}"
          data = {
            spec = {
              podSelector = {}
              policyTypes = ["Ingress", "Egress"]
              # No ingress or egress rules = deny all
            }
          }
        }
      }]
    }
  })

  depends_on = [helm_release.kyverno]
}
```

---

### 5.10 Security — Network Policies

```hcl
# platform/modules/security/network-policies/variables.tf

variable "environment" {
  type = string
}
```

```hcl
# platform/modules/security/network-policies/main.tf

# ═══════════════════════════════════════════════════════════════════
# NETWORK POLICIES
#
# Strategy:
#   1. Default-deny generated by Kyverno for all app namespaces
#   2. Explicit allow rules defined here per namespace
#   3. DNS egress allowed for all pods (required for service discovery)
#   4. Monitoring ingress allowed for all pods (Prometheus scraping)
#   5. Mesh control plane communication allowed
#
# Note: These policies use Kubernetes-native NetworkPolicy.
# Linkerd also provides its own Server/ServerAuthorization policies
# for L7 authorization — those are defined per-service in ArgoCD.
# ═══════════════════════════════════════════════════════════════════

# ─── DNS Egress — ALL application namespaces ─────────────────────

resource "kubectl_manifest" "allow_dns" {
  for_each = toset([
    "novamart-payments",
    "novamart-orders",
    "novamart-users",
    "novamart-frontend",
    "novamart-shared",
  ])

  yaml_body = yamlencode({
    apiVersion = "networking.k8s.io/v1"
    kind       = "NetworkPolicy"
    metadata = {
      name      = "allow-dns-egress"
      namespace = each.key
      labels = {
        "novamart.com/policy-type" = "base"
      }
    }
    spec = {
      podSelector = {}
      policyTypes = ["Egress"]
      egress = [
        {
          # CoreDNS in kube-system
          ports = [
            { port = 53, protocol = "UDP" },
            { port = 53, protocol = "TCP" },
          ]
          to = [{
            namespaceSelector = {
              matchLabels = {
                "kubernetes.io/metadata.name" = "kube-system"
              }
            }
          }]
        }
      ]
    }
  })
}

# ─── Prometheus Scraping — ALL application namespaces ────────────

resource "kubectl_manifest" "allow_monitoring_ingress" {
  for_each = toset([
    "novamart-payments",
    "novamart-orders",
    "novamart-users",
    "novamart-frontend",
    "novamart-shared",
  ])

  yaml_body = yamlencode({
    apiVersion = "networking.k8s.io/v1"
    kind       = "NetworkPolicy"
    metadata = {
      name      = "allow-prometheus-scrape"
      namespace = each.key
      labels = {
        "novamart.com/policy-type" = "monitoring"
      }
    }
    spec = {
      podSelector = {}
      policyTypes = ["Ingress"]
      ingress = [{
        from = [{
          namespaceSelector = {
            matchLabels = {
              "kubernetes.io/metadata.name" = "monitoring"
            }
          }
        }]
        ports = [
          { port = 9090, protocol = "TCP" },  # Prometheus metrics
          { port = 8080, protocol = "TCP" },  # Common app metrics port
          { port = 4191, protocol = "TCP" },  # Linkerd proxy metrics
        ]
      }]
    }
  })
}

# ─── Linkerd Control Plane Communication ─────────────────────────

resource "kubectl_manifest" "allow_linkerd" {
  for_each = toset([
    "novamart-payments",
    "novamart-orders",
    "novamart-users",
    "novamart-frontend",
    "novamart-shared",
  ])

  yaml_body = yamlencode({
    apiVersion = "networking.k8s.io/v1"
    kind       = "NetworkPolicy"
    metadata = {
      name      = "allow-linkerd-control-plane"
      namespace = each.key
      labels = {
        "novamart.com/policy-type" = "mesh"
      }
    }
    spec = {
      podSelector = {}
      policyTypes = ["Ingress", "Egress"]
      ingress = [
        {
          # Allow Linkerd proxy injection and tap
          from = [{
            namespaceSelector = {
              matchLabels = {
                "linkerd.io/is-control-plane" = "true"
              }
            }
          }]
        },
        {
          # Allow pod-to-pod within mesh (mTLS secured by Linkerd)
          from = [{
            namespaceSelector = {
              matchLabels = {
                "linkerd.io/inject" = "enabled"
              }
            }
          }]
          ports = [
            { port = 4143, protocol = "TCP" },  # Linkerd inbound proxy
          ]
        }
      ]
      egress = [
        {
          # Allow reaching Linkerd control plane
          to = [{
            namespaceSelector = {
              matchLabels = {
                "linkerd.io/is-control-plane" = "true"
              }
            }
          }]
          ports = [
            { port = 8443, protocol = "TCP" },  # Linkerd destination
            { port = 8090, protocol = "TCP" },  # Linkerd identity
          ]
        },
        {
          # Allow reaching meshed services in other namespaces
          to = [{
            namespaceSelector = {
              matchLabels = {
                "linkerd.io/inject" = "enabled"
              }
            }
          }]
          ports = [
            { port = 4143, protocol = "TCP" },
          ]
        }
      ]
    }
  })
}

# ═══════════════════════════════════════════════════════════════════
# APPLICATION-SPECIFIC POLICIES
# ═══════════════════════════════════════════════════════════════════

# ─── Payments Namespace (PCI-scoped — strictest rules) ───────────

resource "kubectl_manifest" "payments_egress" {
  yaml_body = yamlencode({
    apiVersion = "networking.k8s.io/v1"
    kind       = "NetworkPolicy"
    metadata = {
      name      = "payments-allowed-egress"
      namespace = "novamart-payments"
      labels = {
        "novamart.com/policy-type" = "application"
        "novamart.com/pci-scope"   = "true"
      }
    }
    spec = {
      podSelector = {
        matchLabels = {
          "novamart.com/service" = "payment-svc"
        }
      }
      policyTypes = ["Egress"]
      egress = [
        {
          # Payment DB (PostgreSQL on port 5432)
          to = [{
            ipBlock = {
              # Data subnet CIDRs — RDS endpoints resolve here
              cidr = "10.0.64.0/22"
            }
          }, {
            ipBlock = { cidr = "10.0.68.0/22" }
          }, {
            ipBlock = { cidr = "10.0.72.0/22" }
          }]
          ports = [
            { port = 5432, protocol = "TCP" }
          ]
        },
        {
          # Redis (ElastiCache on port 6379)
          to = [{
            ipBlock = { cidr = "10.0.64.0/22" }
          }, {
            ipBlock = { cidr = "10.0.68.0/22" }
          }, {
            ipBlock = { cidr = "10.0.72.0/22" }
          }]
          ports = [
            { port = 6379, protocol = "TCP" }
          ]
        },
        {
          # Payment gateway (external API — via NAT Gateway)
          # Specific IP ranges for payment processors
          to = [{
            ipBlock = {
              cidr   = "0.0.0.0/0"
              except = [
                "10.0.0.0/8",
                "172.16.0.0/12",
                "192.168.0.0/16",
                "100.64.0.0/16",
              ]
            }
          }]
          ports = [
            { port = 443, protocol = "TCP" }
          ]
        },
        {
          # OTel Collector (send traces)
          to = [{
            namespaceSelector = {
              matchLabels = {
                "kubernetes.io/metadata.name" = "logging"
              }
            }
          }]
          ports = [
            { port = 4317, protocol = "TCP" },
            { port = 4318, protocol = "TCP" },
          ]
        }
      ]
    }
  })
}

# Payments namespace: allowed ingress (ONLY from frontend/orders via mesh)
resource "kubectl_manifest" "payments_ingress" {
  yaml_body = yamlencode({
    apiVersion = "networking.k8s.io/v1"
    kind       = "NetworkPolicy"
    metadata = {
      name      = "payments-allowed-ingress"
      namespace = "novamart-payments"
      labels = {
        "novamart.com/policy-type" = "application"
        "novamart.com/pci-scope"   = "true"
      }
    }
    spec = {
      podSelector = {
        matchLabels = {
          "novamart.com/service" = "payment-svc"
        }
      }
      policyTypes = ["Ingress"]
      ingress = [
        {
          # Only orders and frontend can call payment-svc
          from = [
            {
              namespaceSelector = {
                matchLabels = {
                  "novamart.com/domain" = "orders"
                }
              }
            },
            {
              namespaceSelector = {
                matchLabels = {
                  "novamart.com/domain" = "frontend"
                }
              }
            }
          ]
          ports = [
            { port = 4143, protocol = "TCP" },  # Through Linkerd proxy
          ]
        }
      ]
    }
  })
}

# ─── Orders Namespace ────────────────────────────────────────────

resource "kubectl_manifest" "orders_egress" {
  yaml_body = yamlencode({
    apiVersion = "networking.k8s.io/v1"
    kind       = "NetworkPolicy"
    metadata = {
      name      = "orders-allowed-egress"
      namespace = "novamart-orders"
      labels = {
        "novamart.com/policy-type" = "application"
      }
    }
    spec = {
      podSelector = {}
      policyTypes = ["Egress"]
      egress = [
        {
          # Orders DB
          to = [{
            ipBlock = { cidr = "10.0.64.0/22" }
          }, {
            ipBlock = { cidr = "10.0.68.0/22" }
          }, {
            ipBlock = { cidr = "10.0.72.0/22" }
          }]
          ports = [{ port = 5432, protocol = "TCP" }]
        },
        {
          # Redis
          to = [{
            ipBlock = { cidr = "10.0.64.0/22" }
          }, {
            ipBlock = { cidr = "10.0.68.0/22" }
          }, {
            ipBlock = { cidr = "10.0.72.0/22" }
          }]
          ports = [{ port = 6379, protocol = "TCP" }]
        },
        {
          # Payment service (cross-namespace via mesh)
          to = [{
            namespaceSelector = {
              matchLabels = { "novamart.com/domain" = "payments" }
            }
          }]
          ports = [{ port = 4143, protocol = "TCP" }]
        },
        {
          # Inventory service (same namespace)
          to = [{
            podSelector = {
              matchLabels = { "novamart.com/service" = "inventory-svc" }
            }
          }]
        },
        {
          # OTel Collector
          to = [{
            namespaceSelector = {
              matchLabels = { "kubernetes.io/metadata.name" = "logging" }
            }
          }]
          ports = [
            { port = 4317, protocol = "TCP" },
            { port = 4318, protocol = "TCP" },
          ]
        }
      ]
    }
  })
}

# ─── Users Namespace ─────────────────────────────────────────────

resource "kubectl_manifest" "users_egress" {
  yaml_body = yamlencode({
    apiVersion = "networking.k8s.io/v1"
    kind       = "NetworkPolicy"
    metadata = {
      name      = "users-allowed-egress"
      namespace = "novamart-users"
      labels = {
        "novamart.com/policy-type" = "application"
      }
    }
    spec = {
      podSelector = {}
      policyTypes = ["Egress"]
      egress = [
        {
          # Users DB
          to = [{
            ipBlock = { cidr = "10.0.64.0/22" }
          }, {
            ipBlock = { cidr = "10.0.68.0/22" }
          }, {
            ipBlock = { cidr = "10.0.72.0/22" }
          }]
          ports = [{ port = 5432, protocol = "TCP" }]
        },
        {
          # Redis (sessions)
          to = [{
            ipBlock = { cidr = "10.0.64.0/22" }
          }, {
            ipBlock = { cidr = "10.0.68.0/22" }
          }, {
            ipBlock = { cidr = "10.0.72.0/22" }
          }]
          ports = [{ port = 6379, protocol = "TCP" }]
        },
        {
          # OTel Collector
          to = [{
            namespaceSelector = {
              matchLabels = { "kubernetes.io/metadata.name" = "logging" }
            }
          }]
          ports = [
            { port = 4317, protocol = "TCP" },
            { port = 4318, protocol = "TCP" },
          ]
        },
        {
          # External SMTP (email via SES)
          to = [{
            ipBlock = {
              cidr   = "0.0.0.0/0"
              except = ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16", "100.64.0.0/16"]
            }
          }]
          ports = [{ port = 465, protocol = "TCP" }]
        }
      ]
    }
  })
}
```

---

### 5.11 Security — ECR Repositories

```hcl
# platform/modules/security/ecr/main.tf

variable "environment" {
  type = string
}

variable "kms_key_arn" {
  type = string
}

variable "repositories" {
  description = "List of ECR repository names to create"
  type        = list(string)
}

variable "image_retention_count" {
  description = "Number of tagged images to retain per repository"
  type        = number
  default     = 30
}

variable "scan_on_push" {
  description = "Enable image scanning on push"
  type        = bool
  default     = true
}

# ─── ECR Repositories ────────────────────────────────────────────

resource "aws_ecr_repository" "repos" {
  for_each = toset(var.repositories)

  name                 = each.key
  image_tag_mutability = "IMMUTABLE"  # Prevent tag overwrites (supply chain security)

  image_scanning_configuration {
    scan_on_push = var.scan_on_push
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = var.kms_key_arn
  }

  tags = {
    Environment = var.environment
    Service     = split("/", each.key)[1]  # Extract service name
    ManagedBy   = "terraform"
  }
}

# ─── Lifecycle Policy (keep last N tagged images) ────────────────

resource "aws_ecr_lifecycle_policy" "repos" {
  for_each   = toset(var.repositories)
  repository = aws_ecr_repository.repos[each.key].name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Remove untagged images after 1 day"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 1
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2
        description  = "Keep last ${var.image_retention_count} tagged images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v", "sha-", "main-", "release-"]
          countType     = "imageCountMoreThan"
          countNumber   = var.image_retention_count
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}

# ─── Repository Policy — Restrict Push Access ───────────────────

resource "aws_ecr_repository_policy" "repos" {
  for_each   = toset(var.repositories)
  repository = aws_ecr_repository.repos[each.key].name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowPullFromEKS"
        Effect = "Allow"
        Principal = {
          AWS = "*"
        }
        Action = [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability"
        ]
        Condition = {
          StringEquals = {
            "aws:PrincipalOrgID" = data.aws_organizations_organization.current.id
          }
        }
      }
    ]
  })
}

data "aws_organizations_organization" "current" {}

# ─── Enhanced Scanning (Inspector) ───────────────────────────────

resource "aws_ecr_registry_scanning_configuration" "enhanced" {
  scan_type = "ENHANCED"

  rule {
    scan_frequency = "CONTINUOUS_SCAN"
    repository_filter {
      filter      = "*"
      filter_type = "WILDCARD"
    }
  }
}

# ─── Outputs ─────────────────────────────────────────────────────

output "repository_urls" {
  description = "Map of repository name to URL"
  value = {
    for k, v in aws_ecr_repository.repos : k => v.repository_url
  }
}

output "registry_id" {
  value = values(aws_ecr_repository.repos)[0].registry_id
}
```

---

### 5.12 Ingress — ALB Controller + cert-manager

```hcl
# platform/modules/ingress/variables.tf

variable "environment" {
  type = string
}

variable "cluster_name" {
  type = string
}

variable "aws_lb_controller_role_arn" {
  type = string
}

variable "domain_name" {
  type = string
}

variable "acme_email" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "cert_manager_chart_version" {
  type    = string
  default = "1.14.4"
}

variable "aws_lb_controller_chart_version" {
  type    = string
  default = "1.7.1"
}
```

```hcl
# platform/modules/ingress/main.tf

# ─── cert-manager ────────────────────────────────────────────────

resource "helm_release" "cert_manager" {
  name       = "cert-manager"
  namespace  = "cert-manager"
  repository = "https://charts.jetstack.io"
  chart      = "cert-manager"
  version    = var.cert_manager_chart_version

  values = [yamlencode({
    installCRDs = true

    replicaCount = 2

    resources = {
      requests = { cpu = "100m", memory = "128Mi" }
      limits   = { cpu = "250m", memory = "256Mi" }
    }

    webhook = {
      replicaCount = 2
      resources = {
        requests = { cpu = "50m",  memory = "64Mi" }
        limits   = { cpu = "100m", memory = "128Mi" }
      }
    }

    cainjector = {
      replicaCount = 2
      resources = {
        requests = { cpu = "50m",  memory = "64Mi" }
        limits   = { cpu = "100m", memory = "128Mi" }
      }
    }

    topologySpreadConstraints = [{
      maxSkew           = 1
      topologyKey       = "topology.kubernetes.io/zone"
      whenUnsatisfiable = "DoNotSchedule"
      labelSelector = {
        matchLabels = { "app.kubernetes.io/name" = "cert-manager" }
      }
    }]

    # Prometheus metrics
    prometheus = {
      enabled = true
      servicemonitor = {
        enabled  = true
        interval = "30s"
      }
    }

    # Pod disruption budget
    podDisruptionBudget = {
      enabled      = true
      minAvailable = 1
    }

    # DNS-01 challenge solver (for wildcard certs)
    # Requires Route53 permissions — add IRSA if needed
    # For now, using HTTP-01 challenge
  })]
}

# ─── ClusterIssuer — Let's Encrypt Production ───────────────────

resource "kubectl_manifest" "letsencrypt_prod" {
  yaml_body = yamlencode({
    apiVersion = "cert-manager.io/v1"
    kind       = "ClusterIssuer"
    metadata = {
      name = "letsencrypt-production"
    }
    spec = {
      acme = {
        server = "https://acme-v02.api.letsencrypt.org/directory"
        email  = var.acme_email
        privateKeySecretRef = {
          name = "letsencrypt-production-key"
        }
        solvers = [
          {
            http01 = {
              ingress = {
                class = "alb"
              }
            }
          }
        ]
      }
    }
  })

  depends_on = [helm_release.cert_manager]
}

# ─── ClusterIssuer — Let's Encrypt Staging (for testing) ────────

resource "kubectl_manifest" "letsencrypt_staging" {
  yaml_body = yamlencode({
    apiVersion = "cert-manager.io/v1"
    kind       = "ClusterIssuer"
    metadata = {
      name = "letsencrypt-staging"
    }
    spec = {
      acme = {
        server = "https://acme-staging-v02.api.letsencrypt.org/directory"
        email  = var.acme_email
        privateKeySecretRef = {
          name = "letsencrypt-staging-key"
        }
        solvers = [
          {
            http01 = {
              ingress = {
                class = "alb"
              }
            }
          }
        ]
      }
    }
  })

  depends_on = [helm_release.cert_manager]
}

# ─── AWS Load Balancer Controller ────────────────────────────────

resource "helm_release" "aws_lb_controller" {
  name       = "aws-load-balancer-controller"
  namespace  = "ingress"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  version    = var.aws_lb_controller_chart_version

  values = [yamlencode({
    clusterName = var.cluster_name
    region      = "us-east-1"
    vpcId       = var.vpc_id

    serviceAccount = {
      create = true
      annotations = {
        "eks.amazonaws.com/role-arn" = var.aws_lb_controller_role_arn
      }
    }

    replicaCount = 2

    resources = {
      requests = { cpu = "100m", memory = "128Mi" }
      limits   = { cpu = "250m", memory = "256Mi" }
    }

    topologySpreadConstraints = [{
      maxSkew           = 1
      topologyKey       = "topology.kubernetes.io/zone"
      whenUnsatisfiable = "DoNotSchedule"
      labelSelector = {
        matchLabels = {
          "app.kubernetes.io/name" = "aws-load-balancer-controller"
        }
      }
    }]

    # Pod disruption budget
    podDisruptionBudget = {
      minAvailable = 1
    }

    # Enable Shield, WAF, and WAFv2 annotation support
    enableShield    = true
    enableWaf       = false  # WAF Classic — deprecated
    enableWafv2     = true

    # Default tags on all ALBs created by the controller
    defaultTags = {
      Environment = var.environment
      ManagedBy   = "aws-lb-controller"
      Project     = "novamart"
      CostCenter  = "PLATFORM-001"
    }

    # Ingress class
    ingressClass = "alb"
    createIngressClassResource = true

    # Service monitor
    serviceMonitor = {
      enabled   = true
      interval  = "30s"
      namespace = "ingress"
    }
  })]
}

# ─── Outputs ─────────────────────────────────────────────────────

output "ingress_class" {
  value = "alb"
}

output "cluster_issuer_production" {
  value = "letsencrypt-production"
}

output "cluster_issuer_staging" {
  value = "letsencrypt-staging"
}
```

---

### 5.13 GitOps — ArgoCD

```hcl
# platform/modules/gitops/argocd/variables.tf

variable "environment" {
  type = string
}

variable "cluster_name" {
  type = string
}

variable "domain_name" {
  type = string
}

variable "github_ssh_key" {
  type      = string
  sensitive = true
}

variable "repo_url" {
  type = string
}

variable "oidc_provider_arn" {
  type = string
}

variable "oidc_provider_url" {
  type = string
}

variable "chart_version" {
  type    = string
  default = "6.7.3"
}
```

```hcl
# platform/modules/gitops/argocd/main.tf

locals {
  name_prefix = "novamart-${var.environment}"
}

data "aws_caller_identity" "current" {}

# ─── IRSA for ArgoCD (ECR image pull for repo server) ────────────

resource "aws_iam_role" "argocd" {
  name_prefix = "${local.name_prefix}-argocd-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${var.oidc_provider_url}:sub" = "system:serviceaccount:argocd:argocd-repo-server"
            "${var.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "argocd_ecr" {
  name_prefix = "argocd-ecr-"
  role        = aws_iam_role.argocd.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:DescribeImages"
        ]
        Resource = "*"
      }
    ]
  })
}

# ─── GitHub SSH Key as Kubernetes Secret ─────────────────────────

resource "kubernetes_secret" "argocd_repo_creds" {
  metadata {
    name      = "argocd-repo-creds"
    namespace = "argocd"
    labels = {
      "argocd.argoproj.io/secret-type" = "repo-creds"
    }
  }

  data = {
    type          = "git"
    url           = var.repo_url
    sshPrivateKey = var.github_ssh_key
  }

  type = "Opaque"
}

# ─── ArgoCD Helm Release (HA Mode) ──────────────────────────────

resource "helm_release" "argocd" {
  name       = "argocd"
  namespace  = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  version    = var.chart_version

  timeout = 600

  values = [yamlencode({
    global = {
      domain = "argocd.${var.domain_name}"
      logging = {
        format = "json"
        level  = "info"
      }
    }

    # ─── Server ──────────────────────────────────────────────
    server = {
      replicas = 3

      resources = {
        requests = { cpu = "250m", memory = "256Mi" }
        limits   = { cpu = "500m", memory = "512Mi" }
      }

      topologySpreadConstraints = [{
        maxSkew           = 1
        topologyKey       = "topology.kubernetes.io/zone"
        whenUnsatisfiable = "DoNotSchedule"
        labelSelector = {
          matchLabels = { "app.kubernetes.io/component" = "server" }
        }
      }]

      # Ingress — exposed via ALB
      ingress = {
        enabled    = true
        ingressClassName = "alb"
        annotations = {
          "alb.ingress.kubernetes.io/scheme"          = "internet-facing"
          "alb.ingress.kubernetes.io/target-type"     = "ip"
          "alb.ingress.kubernetes.io/listen-ports"    = "[{\"HTTPS\": 443}]"
          "alb.ingress.kubernetes.io/ssl-policy"      = "ELBSecurityPolicy-TLS13-1-2-2021-06"
          "alb.ingress.kubernetes.io/certificate-arn"  = ""  # ACM cert ARN — populate via tfvars
          "alb.ingress.kubernetes.io/wafv2-acl-arn"   = ""  # WAF ARN — populate via tfvars
          "alb.ingress.kubernetes.io/group.name"       = "novamart-public"
          "alb.ingress.kubernetes.io/healthcheck-path" = "/healthz"
        }
        hosts = ["argocd.${var.domain_name}"]
        tls = [{
          hosts      = ["argocd.${var.domain_name}"]
          secretName = "argocd-tls"
        }]
      }

      # Disable admin password — use SSO or initial admin secret
      # The initial admin password is auto-generated and stored in
      # argocd-initial-admin-secret. Rotate immediately after first login.
      config = {
        # RBAC
        "policy.default" = "role:readonly"
        "policy.csv" = <<-EOT
          p, role:platform-admin, applications, *, */*, allow
          p, role:platform-admin, clusters, get, *, allow
          p, role:platform-admin, repositories, *, *, allow
          p, role:platform-admin, projects, *, *, allow
          p, role:platform-admin, logs, get, *, allow
          p, role:platform-admin, exec, create, */*, allow

          p, role:app-developer, applications, get, */*, allow
          p, role:app-developer, applications, sync, */*, allow
          p, role:app-developer, logs, get, */*, allow
          p, role:app-developer, applications, action/*, */*, allow

          p, role:app-viewer, applications, get, */*, allow
          p, role:app-viewer, logs, get, */*, allow

          g, platform-team, role:platform-admin
          g, dev-team, role:app-developer
          g, stakeholders, role:app-viewer
        EOT

        # Resource tracking
        "application.resourceTrackingMethod" = "annotation"

        # Status badge
        "statusbadge.enabled" = "true"

        # Dex (SSO) — configure when IdP is available
        "dex.config" = ""
      }
    }

    # ─── Repo Server ─────────────────────────────────────────
    repoServer = {
      replicas = 3

      serviceAccount = {
        annotations = {
          "eks.amazonaws.com/role-arn" = aws_iam_role.argocd.arn
        }
      }

      resources = {
        requests = { cpu = "500m", memory = "512Mi" }
        limits   = { cpu = "1",    memory = "1Gi" }
      }

      topologySpreadConstraints = [{
        maxSkew           = 1
        topologyKey       = "topology.kubernetes.io/zone"
        whenUnsatisfiable = "DoNotSchedule"
        labelSelector = {
          matchLabels = { "app.kubernetes.io/component" = "repo-server" }
        }
      }]

      # Enable Helm support
      env = [
        {
          name  = "HELM_CACHE_HOME"
          value = "/tmp/helm-cache"
        },
        {
          name  = "HELM_CONFIG_HOME"
          value = "/tmp/helm-config"
        }
      ]
    }

    # ─── Application Controller ──────────────────────────────
    controller = {
      replicas = 2  # Active-passive HA

      resources = {
        requests = { cpu = "500m",  memory = "512Mi" }
        limits   = { cpu = "1000m", memory = "1Gi" }
      }

      topologySpreadConstraints = [{
        maxSkew           = 1
        topologyKey       = "topology.kubernetes.io/zone"
        whenUnsatisfiable = "DoNotSchedule"
        labelSelector = {
          matchLabels = { "app.kubernetes.io/component" = "controller" }
        }
      }]

      # Metrics
      metrics = {
        enabled = true
        serviceMonitor = {
          enabled  = true
          interval = "30s"
        }
      }
    }

    # ─── Redis (ArgoCD's internal cache) ─────────────────────
    redis-ha = {
      enabled = true
      replicas = 3
      resources = {
        requests = { cpu = "100m", memory = "128Mi" }
        limits   = { cpu = "250m", memory = "256Mi" }
      }
    }

    # ─── ApplicationSet Controller ───────────────────────────
    applicationSet = {
      replicas = 2
      resources = {
        requests = { cpu = "100m", memory = "128Mi" }
        limits   = { cpu = "250m", memory = "256Mi" }
      }
    }

    # ─── Notifications ───────────────────────────────────────
    notifications = {
      enabled = true
      resources = {
        requests = { cpu = "50m",  memory = "64Mi" }
        limits   = { cpu = "100m", memory = "128Mi" }
      }
    }

    # ─── Metrics ─────────────────────────────────────────────
    monitoring = {
      enabled = true
      serviceMonitor = {
        enabled  = true
        interval = "30s"
      }
    }
  })]

  depends_on = [kubernetes_secret.argocd_repo_creds]
}

# ─── ArgoCD Projects ─────────────────────────────────────────────

resource "kubectl_manifest" "project_platform" {
  yaml_body = yamlencode({
    apiVersion = "argoproj.io/v1alpha1"
    kind       = "AppProject"
    metadata = {
      name      = "platform"
      namespace = "argocd"
    }
    spec = {
      description = "Platform infrastructure services"
      sourceRepos = [var.repo_url]
      destinations = [
        {
          server    = "https://kubernetes.default.svc"
          namespace = "monitoring"
        },
        {
          server    = "https://kubernetes.default.svc"
          namespace = "logging"
        },
        {
          server    = "https://kubernetes.default.svc"
          namespace = "tracing"
        },
        {
          server    = "https://kubernetes.default.svc"
          namespace = "linkerd"
        },
        {
          server    = "https://kubernetes.default.svc"
          namespace = "ingress"
        },
      ]
      clusterResourceWhitelist = [
        {
          group = "*"
          kind  = "Namespace"
        },
        {
          group = "apiextensions.k8s.io"
          kind  = "CustomResourceDefinition"
        },
        {
          group = "rbac.authorization.k8s.io"
          kind  = "*"
        },
        {
          group = "networking.k8s.io"
          kind  = "IngressClass"
        }
      ]

      namespaceResourceWhitelist = [
        { group = "*", kind = "*" }
      ]

      # Only platform team can manage this project
      roles = [
        {
          name     = "platform-admin"
          policies = [
            "p, proj:platform:platform-admin, applications, *, platform/*, allow"
          ]
          groups = ["platform-team"]
        }
      ]

      # Sync windows — platform changes only during business hours
      syncWindows = [
        {
          kind         = "allow"
          schedule     = "0 8-18 * * 1-5"  # Mon-Fri 8am-6pm UTC
          duration     = "10h"
          applications = ["*"]
          namespaces   = ["*"]
        },
        {
          kind         = "deny"
          schedule     = "0 0 * * *"        # Deny all other times
          duration     = "24h"
          applications = ["*"]
          namespaces   = ["*"]
          manualSync   = true               # Allow manual sync during deny windows
        }
      ]
    }
  })

  depends_on = [helm_release.argocd]
}

resource "kubectl_manifest" "project_applications" {
  yaml_body = yamlencode({
    apiVersion = "argoproj.io/v1alpha1"
    kind       = "AppProject"
    metadata = {
      name      = "applications"
      namespace = "argocd"
    }
    spec = {
      description = "NovaMart application workloads"
      sourceRepos = [var.repo_url]
      destinations = [
        {
          server    = "https://kubernetes.default.svc"
          namespace = "novamart-payments"
        },
        {
          server    = "https://kubernetes.default.svc"
          namespace = "novamart-orders"
        },
        {
          server    = "https://kubernetes.default.svc"
          namespace = "novamart-users"
        },
        {
          server    = "https://kubernetes.default.svc"
          namespace = "novamart-frontend"
        },
        {
          server    = "https://kubernetes.default.svc"
          namespace = "novamart-shared"
        },
      ]

      # Applications project can NOT create cluster-scoped resources
      clusterResourceWhitelist = []

      # Restrict what resource types applications can create
      namespaceResourceWhitelist = [
        { group = "",              kind = "ConfigMap" },
        { group = "",              kind = "Secret" },
        { group = "",              kind = "Service" },
        { group = "",              kind = "ServiceAccount" },
        { group = "",              kind = "PersistentVolumeClaim" },
        { group = "apps",          kind = "Deployment" },
        { group = "apps",          kind = "StatefulSet" },
        { group = "apps",          kind = "DaemonSet" },
        { group = "batch",         kind = "Job" },
        { group = "batch",         kind = "CronJob" },
        { group = "autoscaling",   kind = "HorizontalPodAutoscaler" },
        { group = "policy",        kind = "PodDisruptionBudget" },
        { group = "networking.k8s.io", kind = "Ingress" },
        { group = "networking.k8s.io", kind = "NetworkPolicy" },
        { group = "external-secrets.io", kind = "ExternalSecret" },
        { group = "linkerd.io",    kind = "ServiceProfile" },
        { group = "monitoring.coreos.com", kind = "ServiceMonitor" },
        { group = "monitoring.coreos.com", kind = "PodMonitor" },
        { group = "monitoring.coreos.com", kind = "PrometheusRule" },
      ]

      # Deny creating resources in other namespaces
      namespaceResourceBlacklist = [
        { group = "rbac.authorization.k8s.io", kind = "*" },
      ]

      # Roles for app teams
      roles = [
        {
          name     = "app-admin"
          policies = [
            "p, proj:applications:app-admin, applications, *, applications/*, allow"
          ]
          groups = ["dev-team"]
        },
        {
          name     = "app-viewer"
          policies = [
            "p, proj:applications:app-viewer, applications, get, applications/*, allow"
          ]
          groups = ["stakeholders"]
        }
      ]

      # Sync policy — manual sync required in production
      # (auto-sync enabled per-app in staging only)
    }
  })

  depends_on = [helm_release.argocd]
}

# ─── App of Apps — Root Application ─────────────────────────────

resource "kubectl_manifest" "root_application" {
  yaml_body = yamlencode({
    apiVersion = "argoproj.io/v1alpha1"
    kind       = "Application"
    metadata = {
      name      = "root"
      namespace = "argocd"
      labels = {
        "novamart.com/tier" = "gitops"
      }
      finalizers = [
        "resources-finalizer.argocd.argoproj.io"
      ]
    }
    spec = {
      project = "platform"
      source = {
        repoURL        = var.repo_url
        targetRevision = "main"
        path           = "argocd/apps"
        # This directory contains one Application manifest per service
        # ArgoCD watches this directory and creates/deletes applications
        # as manifests are added/removed
      }
      destination = {
        server    = "https://kubernetes.default.svc"
        namespace = "argocd"
      }
      syncPolicy = {
        automated = {
          prune      = true       # Delete apps removed from Git
          selfHeal   = true       # Re-sync if manual changes detected
          allowEmpty = false      # Don't sync if source is empty
        }
        retry = {
          limit = 5
          backoff = {
            duration    = "5s"
            factor      = 2
            maxDuration = "3m"
          }
        }
        syncOptions = [
          "CreateNamespace=false",
          "PrunePropagationPolicy=foreground",
          "PruneLast=true",
        ]
      }
    }
  })

  depends_on = [
    helm_release.argocd,
    kubectl_manifest.project_platform,
    kubectl_manifest.project_applications,
  ]
}

# ─── Outputs ─────────────────────────────────────────────────────

output "argocd_server_url" {
  value = "https://argocd.${var.domain_name}"
}

output "argocd_initial_admin_secret" {
  description = "Kubernetes secret containing the initial admin password. Rotate after first login."
  value       = "argocd-initial-admin-secret"
}

output "argocd_projects" {
  value = {
    platform     = "platform"
    applications = "applications"
  }
}
```

---

### 5.14 Custom Alert Rules

```yaml
# platform/modules/observability/prometheus/alerts/cluster-alerts.yaml
# Loaded via kube-prometheus-stack additionalPrometheusRules

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: novamart-cluster-alerts
  namespace: monitoring
  labels:
    release: prometheus
    novamart.com/alert-tier: cluster
spec:
  groups:
    - name: novamart.cluster.critical
      interval: 30s
      rules:
        - alert: ClusterNodeCountLow
          expr: count(kube_node_status_condition{condition="Ready",status="true"}) < 3
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Cluster has fewer than 3 ready nodes"
            description: "Only {{ $value }} nodes are Ready. Minimum 3 required for HA."
            runbook_url: "https://wiki.novamart.com/runbooks/cluster-node-count-low"

        - alert: ClusterCPUOvercommitted
          expr: |
            sum(kube_pod_container_resource_requests{resource="cpu"})
            / sum(kube_node_status_allocatable{resource="cpu"})
            > 0.9
          for: 15m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Cluster CPU requests exceed 90% of allocatable"
            description: "CPU overcommit ratio is {{ $value | humanizePercentage }}. New pods may not schedule."
            runbook_url: "https://wiki.novamart.com/runbooks/cluster-cpu-overcommitted"

        - alert: ClusterMemoryOvercommitted
          expr: |
            sum(kube_pod_container_resource_requests{resource="memory"})
            / sum(kube_node_status_allocatable{resource="memory"})
            > 0.9
          for: 15m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Cluster memory requests exceed 90% of allocatable"
            description: "Memory overcommit ratio is {{ $value | humanizePercentage }}."
            runbook_url: "https://wiki.novamart.com/runbooks/cluster-memory-overcommitted"

        - alert: PersistentVolumeSpaceLow
          expr: |
            kubelet_volume_stats_available_bytes
            / kubelet_volume_stats_capacity_bytes
            < 0.1
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "PV {{ $labels.persistentvolumeclaim }} has <10% space remaining"
            description: "Namespace {{ $labels.namespace }}, PVC {{ $labels.persistentvolumeclaim }} at {{ $value | humanizePercentage }} free."

    - name: novamart.cluster.warning
      interval: 30s
      rules:
        - alert: KarpenterNodeNotReady
          expr: |
            karpenter_nodes_terminating > 5
          for: 10m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Multiple Karpenter nodes terminating simultaneously"
            description: "{{ $value }} nodes are terminating. May indicate spot interruption wave."

        - alert: PodSchedulingDelayed
          expr: |
            sum(kube_pod_status_phase{phase="Pending"}) > 10
          for: 10m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "{{ $value }} pods stuck in Pending for >10 minutes"
            description: "Possible resource exhaustion or scheduling constraints."

        - alert: NamespaceResourceQuotaNearing
          expr: |
            kube_resourcequota{type="used"}
            / kube_resourcequota{type="hard"}
            > 0.8
          for: 15m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Namespace {{ $labels.namespace }} using >80% of {{ $labels.resource }} quota"
```

```yaml
# platform/modules/observability/prometheus/alerts/node-alerts.yaml

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: novamart-node-alerts
  namespace: monitoring
  labels:
    release: prometheus
    novamart.com/alert-tier: node
spec:
  groups:
    - name: novamart.node.critical
      interval: 30s
      rules:
        - alert: NodeDiskPressure
          expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
          for: 2m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Node {{ $labels.node }} has disk pressure"
            runbook_url: "https://wiki.novamart.com/runbooks/node-disk-pressure"

        - alert: NodeMemoryPressure
          expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
          for: 2m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Node {{ $labels.node }} has memory pressure"

        - alert: NodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Node {{ $labels.node }} is not Ready for >5 minutes"
            runbook_url: "https://wiki.novamart.com/runbooks/node-not-ready"

    - name: novamart.node.warning
      interval: 30s
      rules:
        - alert: NodeCPUHigh
          expr: |
            100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
          for: 15m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Node {{ $labels.instance }} CPU >85% for 15 minutes"

        - alert: NodeMemoryHigh
          expr: |
            (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
          for: 15m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Node {{ $labels.instance }} memory usage >85%"

        - alert: NodeDiskUsageHigh
          expr: |
            (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 80
          for: 15m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Node {{ $labels.instance }} root disk usage >80%"

        - alert: NodeClockSkew
          expr: |
            abs(node_timex_offset_seconds) > 0.05
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Node {{ $labels.instance }} clock skew >50ms"
            description: "Clock skew affects certificate validation, log correlation, and distributed systems."
```

```yaml
# platform/modules/observability/prometheus/alerts/pod-alerts.yaml

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: novamart-pod-alerts
  namespace: monitoring
  labels:
    release: prometheus
    novamart.com/alert-tier: pod
spec:
  groups:
    - name: novamart.pod.critical
      interval: 30s
      rules:
        - alert: PodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 3
          for: 5m
          labels:
            severity: critical
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash-looping"
            description: "Container {{ $labels.container }} restarted {{ $value | humanize }} times in last 15 minutes."
            runbook_url: "https://wiki.novamart.com/runbooks/pod-crash-looping"

        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{condition="true"} == 0
            AND ON(pod, namespace) kube_pod_status_phase{phase="Running"} == 1
          for: 10m
          labels:
            severity: critical
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is Running but not Ready for >10 minutes"

        - alert: ContainerOOMKilled
          expr: |
            kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
          for: 0m
          labels:
            severity: critical
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "Container {{ $labels.container }} in {{ $labels.namespace }}/{{ $labels.pod }} was OOM-killed"
            description: "Increase memory limits for this container."

    - name: novamart.pod.warning
      interval: 30s
      rules:
        - alert: ContainerCPUThrottled
          expr: |
            rate(container_cpu_cfs_throttled_seconds_total[5m]) > 0.25
          for: 15m
          labels:
            severity: warning
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "Container {{ $labels.container }} in {{ $labels.namespace }}/{{ $labels.pod }} is CPU-throttled"
            description: "CPU throttling >25% of the time. Consider increasing CPU limits."

        - alert: ContainerMemoryNearLimit
          expr: |
            container_memory_working_set_bytes
            / container_spec_memory_limit_bytes
            > 0.9
          for: 15m
          labels:
            severity: warning
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "Container {{ $labels.container }} memory at >90% of limit"
            description: "OOM kill imminent if traffic increases."

        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_spec_replicas
            != kube_deployment_status_available_replicas
          for: 10m
          labels:
            severity: warning
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has unavailable replicas"
            description: "Desired: {{ $labels.spec_replicas }}, Available: {{ $value }}"

        - alert: HorizontalPodAutoscalerMaxed
          expr: |
            kube_horizontalpodautoscaler_status_current_replicas
            == kube_horizontalpodautoscaler_spec_max_replicas
          for: 30m
          labels:
            severity: warning
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} at max replicas for >30 minutes"
            description: "Consider increasing max replicas or optimizing the service."
```

```yaml
# platform/modules/observability/prometheus/alerts/platform-alerts.yaml

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: novamart-platform-alerts
  namespace: monitoring
  labels:
    release: prometheus
    novamart.com/alert-tier: platform
spec:
  groups:
    - name: novamart.platform.critical
      interval: 30s
      rules:
        # ─── Monitoring Self-Monitoring ──────────────────────────
        - alert: PrometheusTargetDown
          expr: up == 0
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Prometheus target {{ $labels.job }} is down"
            description: "Instance {{ $labels.instance }} in job {{ $labels.job }} has been unreachable for >5 minutes."

        - alert: AlertmanagerClusterDown
          expr: count(alertmanager_cluster_members) < 3
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Alertmanager cluster has fewer than 3 members"
            description: "Alert delivery may be unreliable."

        - alert: PrometheusStorageFull
          expr: |
            prometheus_tsdb_storage_blocks_bytes
            / (prometheus_tsdb_retention_limit_bytes > 0 or vector(200 * 1024 * 1024 * 1024))
            > 0.9
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Prometheus TSDB storage is >90% full"

        - alert: ThanosSidecarUnhealthy
          expr: thanos_sidecar_prometheus_up != 1
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Thanos sidecar cannot reach Prometheus"
            description: "Long-term metrics storage is failing. Recent metrics will not be uploaded to S3."

        # ─── Linkerd Service Mesh ────────────────────────────────
        - alert: LinkerdControlPlaneDown
          expr: |
            absent(up{job="linkerd-controller"} == 1)
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Linkerd control plane is unreachable"
            description: "mTLS certificate rotation and service discovery may be impaired."

        - alert: LinkerdIdentityExpiring
          expr: |
            identity_cert_expiry_timestamp_seconds - time() < 86400
          for: 1h
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Linkerd identity certificate expires in <24 hours"
            description: "mTLS will stop working when the certificate expires. Rotate immediately."
            runbook_url: "https://wiki.novamart.com/runbooks/linkerd-cert-rotation"

        # ─── Fluent Bit ──────────────────────────────────────────
        - alert: FluentBitOutputErrors
          expr: |
            rate(fluentbit_output_errors_total[5m]) > 0
          for: 10m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Fluent Bit output errors for >10 minutes"
            description: "Logs may be lost. Output plugin {{ $labels.name }} is failing."

        - alert: FluentBitBufferFull
          expr: |
            fluentbit_input_bytes_total - fluentbit_output_bytes_total > 50 * 1024 * 1024
          for: 10m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Fluent Bit buffer growing — output lag"
            description: "Buffer has >50MB of unsent data. Logs may be delayed."

        # ─── ArgoCD ──────────────────────────────────────────────
        - alert: ArgoCDSyncFailed
          expr: |
            argocd_app_info{sync_status="OutOfSync",health_status!="Healthy"} == 1
          for: 30m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "ArgoCD app {{ $labels.name }} is out of sync for >30 minutes"
            description: "Project: {{ $labels.project }}, Status: {{ $labels.health_status }}"

        - alert: ArgoCDAppDegraded
          expr: |
            argocd_app_info{health_status="Degraded"} == 1
          for: 10m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "ArgoCD app {{ $labels.name }} is in Degraded state"

        # ─── External Secrets ────────────────────────────────────
        - alert: ExternalSecretSyncFailed
          expr: |
            externalsecret_status_condition{condition="Ready",status="False"} == 1
          for: 10m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "ExternalSecret {{ $labels.namespace }}/{{ $labels.name }} sync failed"
            description: "Application may be using stale secrets."

        # ─── Kyverno ─────────────────────────────────────────────
        - alert: KyvernoAdmissionBlocked
          expr: |
            rate(kyverno_admission_requests{resource_request_operation="CREATE",action="block"}[5m]) > 5
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Kyverno blocked >5 resource creations per minute"
            description: "High rate of policy violations. A team may be struggling with deployment."

        - alert: KyvernoWebhookDown
          expr: |
            absent(up{job="kyverno"} == 1)
          for: 2m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Kyverno admission webhook is down"
            description: "With failurePolicy=Fail, ALL deployments will be blocked. Immediate action required."
            runbook_url: "https://wiki.novamart.com/runbooks/kyverno-webhook-down"

    - name: novamart.linkerd.service
      interval: 30s
      rules:
        # ─── Service-Level Indicators (from Linkerd mesh) ────────
        - alert: ServiceHighErrorRate
          expr: |
            sum(rate(response_total{classification="failure",direction="inbound"}[5m])) by (deployment, namespace)
            / sum(rate(response_total{direction="inbound"}[5m])) by (deployment, namespace)
            > 0.05
          for: 5m
          labels:
            severity: critical
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "{{ $labels.namespace }}/{{ $labels.deployment }} error rate >5%"
            description: "{{ $value | humanizePercentage }} of inbound requests are failing."
            runbook_url: "https://wiki.novamart.com/runbooks/service-high-error-rate"

        - alert: ServiceHighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(response_latency_ms_bucket{direction="inbound"}[5m])) by (le, deployment, namespace)
            ) > 2000
          for: 5m
          labels:
            severity: warning
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "{{ $labels.namespace }}/{{ $labels.deployment }} p99 latency >2 seconds"
            description: "p99 latency is {{ $value | humanize }}ms."

        - alert: ServiceTrafficDrop
          expr: |
            sum(rate(response_total{direction="inbound"}[5m])) by (deployment, namespace)
            < sum(rate(response_total{direction="inbound"}[5m] offset 1h)) by (deployment, namespace) * 0.5
          for: 10m
          labels:
            severity: warning
            team: "{{ $labels.namespace }}"
          annotations:
            summary: "{{ $labels.namespace }}/{{ $labels.deployment }} traffic dropped >50% vs 1 hour ago"
            description: "Possible upstream failure or routing change."
```

---

### 5.15 Grafana Dashboards (as ConfigMaps)

```hcl
# platform/modules/observability/prometheus/dashboards.tf
# Dashboards are loaded automatically by Grafana's sidecar
# via the grafana_dashboard=1 label

resource "kubernetes_config_map" "cluster_overview_dashboard" {
  metadata {
    name      = "grafana-dashboard-cluster-overview"
    namespace = "monitoring"
    labels = {
      grafana_dashboard = "1"
    }
    annotations = {
      grafana_folder = "NovaMart/Cluster"
    }
  }

  data = {
    "cluster-overview.json" = file("${path.module}/dashboards/cluster-overview.json")
  }
}

resource "kubernetes_config_map" "node_health_dashboard" {
  metadata {
    name      = "grafana-dashboard-node-health"
    namespace = "monitoring"
    labels = {
      grafana_dashboard = "1"
    }
    annotations = {
      grafana_folder = "NovaMart/Cluster"
    }
  }

  data = {
    "node-health.json" = file("${path.module}/dashboards/node-health.json")
  }
}

resource "kubernetes_config_map" "namespace_resources_dashboard" {
  metadata {
    name      = "grafana-dashboard-namespace-resources"
    namespace = "monitoring"
    labels = {
      grafana_dashboard = "1"
    }
    annotations = {
      grafana_folder = "NovaMart/Namespaces"
    }
  }

  data = {
    "namespace-resources.json" = file("${path.module}/dashboards/namespace-resources.json")
  }
}

resource "kubernetes_config_map" "linkerd_service_dashboard" {
  metadata {
    name      = "grafana-dashboard-linkerd-service"
    namespace = "monitoring"
    labels = {
      grafana_dashboard = "1"
    }
    annotations = {
      grafana_folder = "NovaMart/Service Mesh"
    }
  }

  data = {
    "linkerd-service.json" = file("${path.module}/dashboards/linkerd-service.json")
  }
}

resource "kubernetes_config_map" "platform_health_dashboard" {
  metadata {
    name      = "grafana-dashboard-platform-health"
    namespace = "monitoring"
    labels = {
      grafana_dashboard = "1"
    }
    annotations = {
      grafana_folder = "NovaMart/Platform"
    }
  }

  # Platform Health dashboard — monitors all platform services
  data = {
    "platform-health.json" = jsonencode({
      annotations = { list = [] }
      editable    = false
      title       = "NovaMart Platform Health"
      uid         = "novamart-platform-health"
      version     = 1
      time        = { from = "now-1h", to = "now" }
      refresh     = "30s"

      templating = {
        list = [{
          name       = "datasource"
          type       = "datasource"
          query      = "prometheus"
          current    = { text = "Prometheus", value = "Prometheus" }
          hide       = 0
        }]
      }

      panels = [
        {
          title       = "Platform Services Status"
          type        = "stat"
          gridPos     = { h = 4, w = 24, x = 0, y = 0 }
          targets = [{
            expr = "up{job=~\"prometheus|alertmanager|grafana|linkerd.*|fluent-bit|otel.*|kyverno|argocd.*|external-secrets|cert-manager|tempo.*\"}"
            legendFormat = "{{ job }}"
          }]
          fieldConfig = {
            defaults = {
              mappings = [
                { type = "value", options = { "0" = { text = "DOWN", color = "red" } } },
                { type = "value", options = { "1" = { text = "UP", color = "green" } } }
              ]
            }
          }
        },
        {
          title   = "Prometheus Memory Usage"
          type    = "timeseries"
          gridPos = { h = 8, w = 12, x = 0, y = 4 }
          targets = [{
            expr         = "process_resident_memory_bytes{job=\"prometheus\"}"
            legendFormat = "{{ pod }}"
          }]
        },
        {
          title   = "Alertmanager Notifications/min"
          type    = "timeseries"
          gridPos = { h = 8, w = 12, x = 12, y = 4 }
          targets = [{
            expr         = "rate(alertmanager_notifications_total[5m]) * 60"
            legendFormat = "{{ integration }}"
          }]
        },
        {
          title   = "Fluent Bit Throughput (records/sec)"
          type    = "timeseries"
          gridPos = { h = 8, w = 12, x = 0, y = 12 }
          targets = [{
            expr         = "rate(fluentbit_output_proc_records_total[5m])"
            legendFormat = "{{ name }}"
          }]
        },
        {
          title   = "Linkerd mTLS Coverage"
          type    = "gauge"
          gridPos = { h = 8, w = 12, x = 12, y = 12 }
          targets = [{
            expr = <<-EOT
              sum(rate(response_total{tls="true",direction="inbound"}[5m]))
              / sum(rate(response_total{direction="inbound"}[5m]))
            EOT
          }]
          fieldConfig = {
            defaults = {
              unit = "percentunit"
              thresholds = {
                steps = [
                  { value = 0,    color = "red" },
                  { value = 0.95, color = "yellow" },
                  { value = 0.99, color = "green" },
                ]
              }
            }
          }
        },
        {
          title   = "Kyverno Policy Violations"
          type    = "timeseries"
          gridPos = { h = 8, w = 12, x = 0, y = 20 }
          targets = [{
            expr         = "rate(kyverno_admission_requests{action=\"block\"}[5m]) * 60"
            legendFormat = "{{ resource_namespace }}"
          }]
        },
        {
          title   = "ArgoCD Sync Status"
          type    = "stat"
          gridPos = { h = 8, w = 12, x = 12, y = 20 }
          targets = [{
            expr         = "argocd_app_info"
            legendFormat = "{{ name }}: {{ sync_status }}/{{ health_status }}"
          }]
        }
      ]
    })
  }
}
```

---

### 5.16 gp3-encrypted StorageClass

```hcl
# platform/modules/observability/prometheus/storage-class.tf

resource "kubernetes_storage_class" "gp3_encrypted" {
  metadata {
    name = "gp3-encrypted"
    annotations = {
      "storageclass.kubernetes.io/is-default-class" = "true"
    }
  }

  storage_provisioner = "ebs.csi.aws.com"

  parameters = {
    type      = "gp3"
    iops      = "3000"
    throughput = "125"
    encrypted = "true"
    # KMS key ID for EBS encryption — uses the EKS secrets key
    # which is already authorized for EBS CSI via IRSA
  }

  reclaim_policy         = "Retain"  # Don't delete PVs when PVC is deleted
  volume_binding_mode    = "WaitForFirstConsumer"  # Schedule-aware provisioning
  allow_volume_expansion = true

  mount_options = []
}
```

---

## 6. Platform Apply Order & Outputs

```hcl
# platform/environments/production/outputs.tf

# ─── Observability ───────────────────────────────────────────────
output "grafana_url" {
  value = "https://grafana.${var.domain_name}"
}

output "prometheus_endpoint" {
  value = "http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090"
}

output "alertmanager_endpoint" {
  value = "http://prometheus-alertmanager.monitoring.svc.cluster.local:9093"
}

output "tempo_endpoint" {
  value = module.tempo.distributor_endpoint
}

# ─── Service Mesh ────────────────────────────────────────────────
output "linkerd_trust_anchor_expiry" {
  value       = module.linkerd.trust_anchor_expiry
  description = "CRITICAL: Rotate trust anchor before this date"
}

output "linkerd_issuer_expiry" {
  value       = module.linkerd.issuer_expiry
  description = "CRITICAL: Rotate issuer cert before this date"
}

# ─── Security ────────────────────────────────────────────────────
output "cluster_secret_store" {
  value = module.external_secrets.cluster_secret_store_name
}

output "ecr_repository_urls" {
  value = module.ecr.repository_urls
}

# ─── GitOps ──────────────────────────────────────────────────────
output "argocd_url" {
  value = module.argocd.argocd_server_url
}

output "argocd_initial_admin_secret" {
  value       = module.argocd.argocd_initial_admin_secret
  description = "Run: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d"
}

# ─── Ingress ─────────────────────────────────────────────────────
output "ingress_class" {
  value = module.ingress.ingress_class
}

output "cluster_issuer" {
  value = module.ingress.cluster_issuer_production
}
```

```markdown
# docs/runbook-platform-operations.md

# ═══════════════════════════════════════════════════════════════════
# PLATFORM SERVICES — APPLY ORDER
# ═══════════════════════════════════════════════════════════════════
#
# PREREQUISITES:
#   - Lesson 1 infrastructure fully deployed
#   - kubectl configured for novamart-production cluster
#   - Helm v3.14+
#   - Sensitive variables populated in terraform.tfvars or env vars:
#     - TF_VAR_pagerduty_integration_key
#     - TF_VAR_slack_webhook_url_warning
#     - TF_VAR_slack_webhook_url_info
#     - TF_VAR_argocd_github_ssh_key
#
# DEPENDENCY GRAPH:
#
#   ┌──────────────┐
#   │  Namespaces   │
#   └──────┬───────┘
#          │
#   ┌──────▼───────┐
#   │   Linkerd     │  (mesh must be ready before apps inject sidecars)
#   └──────┬───────┘
#          │
#   ┌──────▼───────────────────────────────────────┐
#   │  Observability (Prometheus, Fluent Bit,       │
#   │  OTel Collector, Tempo) — parallel OK         │
#   └──────┬───────────────────────────────────────┘
#          │
#   ┌──────▼───────────────────────────────────────┐
#   │  Security (ESO, Kyverno, Network Policies,    │
#   │  ECR) — parallel OK                           │
#   └──────┬───────────────────────────────────────┘
#          │
#   ┌──────▼───────┐
#   │   Ingress     │  (cert-manager + ALB Controller)
#   └──────┬───────┘
#          │
#   ┌──────▼───────┐
#   │   ArgoCD      │  (depends on Ingress for web UI)
#   └──────────────┘
#
# ESTIMATED TIME: ~15-20 minutes
#
# APPLY:
#   cd platform/environments/production
#   terraform init -backend-config="bucket=${STATE_BUCKET}" \
#                  -backend-config="dynamodb_table=${LOCK_TABLE}"
#   terraform plan -out=platform.plan
#   terraform apply platform.plan
#
# POST-APPLY VALIDATION:
#
#   # Linkerd
#   linkerd check
#   linkerd viz stat deploy -n novamart-payments
#
#   # Prometheus
#   kubectl get pods -n monitoring
#   kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
#   # Open http://localhost:9090/targets — verify all targets are UP
#
#   # Grafana
#   kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
#   # Open http://localhost:3000 — verify dashboards load
#
#   # Fluent Bit
#   kubectl get pods -n logging -l app.kubernetes.io/name=fluent-bit
#   aws logs describe-log-groups --log-group-name-prefix /novamart/
#
#   # ArgoCD
#   kubectl port-forward svc/argocd-server -n argocd 8080:443
#   # Get initial admin password:
#   kubectl -n argocd get secret argocd-initial-admin-secret \
#     -o jsonpath='{.data.password}' | base64 -d
#
#   # Kyverno
#   kubectl get clusterpolicies
#   kubectl get policyreports -A
#
#   # External Secrets
#   kubectl get clustersecretstores
#   kubectl get externalsecrets -A
#
#   # Network Policies
#   kubectl get networkpolicies -A
#
# ═══════════════════════════════════════════════════════════════════
# CERTIFICATE ROTATION SCHEDULE
# ═══════════════════════════════════════════════════════════════════
#
# Linkerd Trust Anchor: Rotates every 10 years (manual process)
#   - Check: terraform output linkerd_trust_anchor_expiry
#   - Runbook: https://linkerd.io/2/tasks/manually-rotating-control-plane-tls-credentials/
#
# Linkerd Issuer: Rotates every 1 year (AUTOMATE THIS)
#   - Set calendar reminder for 30 days before expiry
#   - Alertmanager rule LinkerdIdentityExpiring fires at 24 hours
#   - Process: Re-run Terraform to regenerate, helm upgrade Linkerd
#
# Let's Encrypt Certificates: cert-manager auto-renews at 30 days before expiry
#   - Monitor: cert-manager_certificate_expiration_timestamp_seconds
#
# ═══════════════════════════════════════════════════════════════════
# PLATFORM MONTHLY MAINTENANCE CHECKLIST
# ═══════════════════════════════════════════════════════════════════
#
# [ ] Check Helm chart versions for updates (security patches)
# [ ] Review Kyverno policy reports for violation trends
# [ ] Review Prometheus disk usage and retention
# [ ] Verify Thanos compaction is running (check Grafana platform dashboard)
# [ ] Review CloudWatch Logs costs — adjust retention if needed
# [ ] Check Linkerd issuer certificate expiry
# [ ] Verify ArgoCD sync status — no apps stuck OutOfSync
# [ ] Review and resolve any ExternalSecret sync failures
# [ ] Check ECR image scanning results for critical vulnerabilities
# [ ] Test alerting pipeline — trigger a test alert
```

