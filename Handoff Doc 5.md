# UPDATED HANDOFF DOCUMENT

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

# 🔄 COMPLETE ROADMAP (APPROVED v2)

```
══════════════════════════════════════════════════════════════════
  PART 1: FOUNDATIONS (Phases 0-6)
══════════════════════════════════════════════════════════════════

  Phase 0: Linux Deep Dive                     ████████████████████ 100% ✅
  Phase 1: Networking                          ████████████████████ 100% ✅
  Phase 2: Git, Docker, Kubernetes             ████████████████████ 100% ✅
  Phase 3: CI/CD Pipelines                     ████████████████████ 100% ✅
  Phase 4: IaC (Terraform + Ansible)           ████████████████████ 100% ✅
  Phase 5: Observability & SRE                 ████████████████████ 100% ✅
  Phase 6: Security, Compliance, Cloud         ████████████████████ 100% ✅

══════════════════════════════════════════════════════════════════
  PART 2: BUILD & OPERATE (Phases 7-8)
══════════════════════════════════════════════════════════════════

  Phase 7: Build NovaMart                      ████████████████████ 100% ✅
  Phase 8: Operate NovaMart (Simulation)       ████████████████████ 100% ✅

══════════════════════════════════════════════════════════════════
  PART 3: AUTOMATION & TOOLING (Phase 9)
══════════════════════════════════════════════════════════════════

  Phase 9: Production Automation & Internal Tooling  ░░░░░░░░░░░░░░░░░░░░  0%
    → Lesson 1: Bash for DevOps                      (NOT STARTED)
    → Lesson 2: Python for DevOps                    (NOT STARTED)
    → Lesson 3: Go for DevOps                        (NOT STARTED)
    → Lesson 4: Integration Patterns & Prod Tooling  (NOT STARTED)

══════════════════════════════════════════════════════════════════
  PART 4: ADVANCED TOPICS (Phase 10)
══════════════════════════════════════════════════════════════════

  Phase 10: Production Depth                         ░░░░░░░░░░░░░░░░░░░░  0%
    → Lesson 1: Database Operations for DevOps       (NOT STARTED)
    → Lesson 2: FinOps & Cost Engineering            (NOT STARTED)
    → Lesson 3: Chaos Engineering & DR               (NOT STARTED)
    → Lesson 4: Platform Engineering & DevEx         (NOT STARTED)

══════════════════════════════════════════════════════════════════
  PART 5: INTERVIEW PREP (Phase 11)
══════════════════════════════════════════════════════════════════

  Phase 11: FAANG Interview Preparation              ░░░░░░░░░░░░░░░░░░░░  0%
    → Lesson 1: System Design for Infrastructure     (NOT STARTED)
    → Lesson 2: Live Troubleshooting                 (NOT STARTED)
    → Lesson 3: Coding Challenges                    (NOT STARTED)
    → Lesson 4: Behavioral & Leadership              (NOT STARTED)
    → Lesson 5: Mock Interviews (Full Loops)         (NOT STARTED)

══════════════════════════════════════════════════════════════════

  Overall Progress:  ████████████████░░░░ ~69%
  Completed: 9 phases (Phases 0-8)
  Remaining: 3 phases, 13 lessons (Phases 9-11)

══════════════════════════════════════════════════════════════════
```

---

# 📚 DETAILED TOPIC COVERAGE — EVERYTHING TAUGHT

## PHASE 0: Linux Deep Dive (100% ✅)

### Lesson 1: Filesystem Hierarchy & Permissions
```
Topics: FHS (/etc, /var, /proc, /sys, /tmp, /opt), file permissions (rwx, octal, 
symbolic), ownership (chown, chgrp), special permissions (SUID, SGID, sticky bit),
umask, ACLs (getfacl, setfacl)
Failure modes: world-writable files, SUID exploitation, permission denied debugging
```

### Lesson 2: Process Management
```
Topics: Process lifecycle (fork, exec, exit, wait), PID 1 (init/systemd), 
process states (R, S, D, Z, T), signals (SIGTERM, SIGKILL, SIGHUP, SIGUSR1),
ps, top, htop, kill, nice/renice, nohup, job control (bg, fg, &)
Zombie processes, orphan processes, process trees, /proc filesystem
Failure modes: zombie accumulation, D-state (uninterruptible sleep), 
signal handling in containers
```

### Lesson 3: systemd
```
Topics: Units (service, timer, socket, mount, target), unit files, 
dependencies (Requires, Wants, After, Before), targets (multi-user, graphical),
systemctl commands, journalctl (filtering, persistence, disk management),
socket activation, timer units (replacing cron)
Failure modes: dependency cycles, failed units blocking boot, 
journal disk exhaustion, Java exit code 143 (SuccessExitStatus=143 fix)
```

### Lesson 4: Package Management & Boot Process
```
Topics: apt/yum/dnf, repositories, GPG verification, pinning, 
GRUB → kernel → initramfs → systemd → targets boot sequence,
kernel modules (lsmod, modprobe), dracut/update-initramfs
Failure modes: broken dependencies, repo GPG failures, boot failures
```

### Lesson 5: Storage & Filesystems
```
Topics: Block devices, partitions (MBR/GPT), filesystems (ext4, xfs, btrfs),
mount/umount, fstab, LVM (PV, VG, LV — extend, snapshot), 
RAID levels (0, 1, 5, 6, 10), inode exhaustion, disk I/O (iostat, iotop),
df, du, lsblk, blkid, swap
Failure modes: disk full, inode exhaustion, LVM snapshot overflow,
filesystem corruption, mount failures
```

### Lesson 6: Users, Groups, PAM, SSH
```
Topics: /etc/passwd, /etc/shadow, /etc/group, useradd/usermod/userdel,
PAM modules and configuration, SSH key-based auth, ssh-agent, 
sshd_config hardening, SSH tunneling (local, remote, dynamic/SOCKS),
SSH ProxyJump/bastion, authorized_keys management
Failure modes: locked accounts, PAM misconfiguration, SSH brute force,
key permission issues (0600)
```

### Lesson 7: Memory Management & Performance
```
Topics: Virtual memory, RSS vs VSZ, page cache, swap, OOM killer 
(oom_score_adj), free command interpretation, vmstat, sar, 
memory-mapped files, huge pages, NUMA
Failure modes: OOM kills, swap death spiral, memory leaks, 
cache vs actual free memory confusion
```

### Lesson 8: Kernel Tuning (sysctl)
```
Topics: sysctl interface, /proc/sys, key tunables:
  net.core.somaxconn, net.ipv4.tcp_max_syn_backlog,
  net.ipv4.ip_local_port_range, net.core.rmem_max/wmem_max,
  net.ipv4.tcp_tw_reuse, vm.swappiness, vm.overcommit_memory,
  fs.file-max, net.netfilter.nf_conntrack_max,
  net.ipv4.tcp_keepalive_time/intvl/probes
Persistence via /etc/sysctl.d/, production tuning profiles
Failure modes: conntrack exhaustion, SYN backlog overflow, 
port exhaustion, file descriptor limits
```

---

## PHASE 1: Networking (100% ✅)

### Lessons 1-3: Foundations
```
Topics: OSI/TCP-IP models, layer-by-layer debugging methodology,
IP addressing, CIDR notation, subnetting, AWS VPC design (public/private/data subnets),
DNS full resolution chain, record types (A, AAAA, CNAME, MX, NS, SOA, TXT, SRV, PTR),
CoreDNS in Kubernetes, ndots:5 problem and query amplification,
Route53 routing policies (simple, weighted, latency, failover, geolocation),
Route53 health checks, failover patterns
Failure modes: DNS TTL caching issues, Java DNS caching forever fix,
split-horizon DNS, stale cache
```

### Lessons 4-6: Protocols & Load Balancing
```
Topics: TCP 3-way handshake, 4-way teardown, all 11 TCP states,
CLOSE_WAIT (app bug), TIME_WAIT (normal but manage), tcp_tw_reuse,
congestion control (slow start, AIMD, Cubic, BBR), retransmission,
RST causes and debugging,
HTTP methods, status codes (2xx, 3xx, 4xx, 5xx — 502/503/504 for DevOps),
HTTP/1.1 vs HTTP/2 (multiplexing, server push, header compression) vs HTTP/3 (QUIC),
gRPC requires HTTP/2,
TLS 1.3 handshake (1-RTT), SNI, certificate chain validation,
ACM (AWS Certificate Manager), cert-manager, Let's Encrypt, ACME protocol,
L4 (NLB — TCP/UDP, connection-based) vs L7 (ALB — HTTP, content-based),
LB algorithms (round-robin, least-connections, IP hash, weighted),
health checks (active vs passive), connection draining,
Nginx reverse proxy config, upstream keepalive,
Timeout hierarchy: Client > LB idle > LB backend > Backend processing,
502 systematic debugging framework
Failure modes: CLOSE_WAIT accumulation, TIME_WAIT port exhaustion,
head-of-line blocking (HTTP/1.1), TLS certificate expiry,
SSL termination at wrong layer, timeout mismatch causing 502s
```

### Lessons 7-9: Firewalls & AWS Networking
```
Topics: iptables architecture (tables: filter/nat/mangle/raw, 
chains: INPUT/OUTPUT/FORWARD/PREROUTING/POSTROUTING),
first-match-wins rule processing, SNAT/DNAT/MASQUERADE,
Security Groups (stateful, allow-only, SG-to-SG references),
NACLs (stateless, ordered rules, ephemeral port trap),
VPC architecture (subnets, route tables, IGW, NAT Gateway),
VPC Peering (non-transitive), Transit Gateway (hub-and-spoke),
VPN (Site-to-Site, Client VPN), Direct Connect,
VPC Endpoints (Gateway = free for S3/DynamoDB, Interface = paid for everything else),
PrivateLink architecture,
tcpdump (capture and filter syntax), traceroute/mtr, ss (socket stats),
curl timing breakdown (-w format),
6-step systematic network debugging framework
Failure modes: Security Group allows but NACL blocks (ephemeral ports),
asymmetric routing with Transit Gateway, NAT Gateway single-AZ failure,
VPC endpoint missing for ECR/S3 in private subnets,
interface endpoint DNS resolution not enabled
```

### Lesson 10: Kubernetes Networking
```
Topics: K8s 4 networking requirements (every pod gets IP, no NAT),
CNI plugins (AWS VPC CNI, Calico, Cilium, Flannel, Weave),
Pod-to-pod same node (veth pairs → Linux bridge),
Pod-to-pod different nodes (VPC CNI direct routing, VXLAN overlay, BGP),
Service internals (iptables NAT rules with probability chains),
kube-proxy iptables mode (O(n)) vs IPVS mode (O(1) hash lookup),
All Service types (ClusterIP, NodePort, LoadBalancer, ExternalName, Headless),
externalTrafficPolicy (Cluster vs Local),
Ingress resources and controllers,
NetworkPolicies (ingress, egress, DNS egress gotcha),
VPC Flow Logs (format, debugging, Athena queries)
Failure modes: VPC CNI IP exhaustion (prefix delegation fix),
conntrack table full, IPVS session persistence causing latency,
NetworkPolicy label mismatch, LB subnet tags missing,
externalTrafficPolicy:Local + uneven pod distribution
```

---

## PHASE 2: Git, Docker, Kubernetes (100% ✅)

### Lesson 1 + 1B: Git Internals & Commands
```
Topics: Git object model (blob, tree, commit, tag), content-addressable storage,
SHA-1 hashing, deduplication, packfiles, refs, HEAD, tags,
three areas (working directory, staging/index, repository),
stash internals, merge strategies (fast-forward, three-way, rebase),
interactive rebase, branching strategies (trunk-based, GitHub Flow, GitFlow),
CI optimizations (shallow clone, sparse checkout, path-based triggers),
monorepo strategies, git bisect, branch protection rules,
fetch vs pull, reset (--soft/--mixed/--hard), revert vs reset,
revert a merge commit (-m 1, revert-the-revert trap),
cherry-pick, conflict resolution (ours/theirs SWAP during rebase),
git log (advanced filtering), blame, show, diff, restore, switch,
clean, Git LFS, .gitattributes, git hooks (client + server),
pre-commit framework, Husky, submodules vs subtrees, worktrees
Failure modes: force push to main (reflog recovery),
secrets committed to Git (rotate FIRST, git-filter-repo),
CI testing branch not merge result, .gitignore missing secrets/state
```

### Lesson 2 + 2B: Docker Internals & Production
```
Topics: Containers = process + namespaces + cgroups + union FS,
Namespaces (PID, NET, MNT, UTS, IPC, USER),
Cgroups (CPU, Memory, I/O, PIDs), JVM + cgroup OOM issue,
Container networking (bridge, host, none, custom bridge, macvlan),
Union filesystem / layers / copy-on-write,
Dockerfile best practices (multi-stage, layer ordering, distroless),
Image tagging strategy (immutable tags, digest pinning),
Docker Compose (healthcheck, depends_on, profiles),
Docker security (non-root, read-only, cap-drop, Trivy/Grype),
Container runtimes (containerd, CRI-O, Podman), Dockershim removal,
ENTRYPOINT vs CMD, PID 1 zombie problem (tini, dumb-init),
Signal handling, Volume types, Log rotation (CRITICAL),
Logging drivers, Restart policies, Multi-architecture builds,
Image signing (Cosign/Sigstore, Kyverno verification),
DinD vs DooD vs Kaniko vs Buildah, docker inspect,
/proc visibility problem, GOMAXPROCS/UseContainerSupport,
BuildKit (cache mounts, secret mounts), ARG vs ENV,
Exit code meanings, troubleshooting playbook
Failure modes: OOM killed, layer cache busted, image size explosion,
works locally crashes in K8s (root vs non-root, read-only FS,
resource limits, stale :latest, network differences)
```

### Lesson 3 + 3B + 3C: Kubernetes Architecture & Production
```
Topics: Control plane (API server, etcd Raft, Scheduler, Controller Manager, CCM),
Worker node (kubelet, kube-proxy, container runtime),
Pod creation chain, Pod spec deep dive (init containers, probes, lifecycle hooks),
Termination sequence (preStop sleep trick for 502 prevention),
QoS classes, Deployments (rolling update, rollback),
StatefulSet (stable identity, ordered, headless service, PVC templates),
DaemonSet, Job, CronJob, ConfigMap and Secret (env vs volume, KMS, ESO, Reloader),
Namespaces (RBAC, ResourceQuota, LimitRange),
Scheduling (taints/tolerations, affinity, topologySpreadConstraints),
RBAC (Role/ClusterRole, bindings), Storage (PV, PVC, StorageClass, CSI, snapshots),
HPA (CPU, memory, custom, external, behavior policies),
VPA (modes, HPA conflict), Cluster Autoscaler vs Karpenter,
PDBs (deadlock trap), PriorityClasses, Node maintenance (cordon, drain),
IRSA (OIDC, projected token), Admission controllers (mutating, validating, webhooks),
Pod Security Standards/PSA, Helm (charts, hooks, diff, ArgoCD),
CRDs and Operators, Finalizers (stuck resources),
Ephemeral containers (kubectl debug), CPU throttling (CFS, nr_throttled),
CoreDNS scaling (NodeLocal DNSCache), EndpointSlice,
OOMKilled vs Evicted vs Preempted, Secret rotation,
Resource request tuning (VPA methodology), etcd failure modes, Runbook structure
Production scenarios: Pod stuck Pending, rollout stuck, pod eviction,
502 during deployment, PVC stuck, Helm rollback, namespace stuck Terminating,
HPA not scaling, service not reachable, node NotReady, CrashLoopBackOff
```

### Lesson 4: Kubernetes Operational Patterns
```
Topics: Istio service mesh (istiod, Envoy sidecars, VirtualService, DestinationRule,
Gateway, PeerAuthentication, AuthorizationPolicy, observability, Kiali),
ArgoCD GitOps (Application, syncPolicy, AppProject, App of Apps, ApplicationSet,
sync waves, sync windows),
CI/CD → ArgoCD flow (Jenkins → manifest repo → ArgoCD syncs),
Kustomize (base + overlays, patches, configMapGenerator, vs Helm),
Troubleshooting workflows: CrashLoopBackOff, service not reachable, node NotReady
```

---

## PHASE 3: CI/CD Pipelines (100% ✅)

```
Topics: Jenkins architecture (controller, agents, distributed builds),
Jenkinsfile (declarative vs scripted, shared libraries),
Pipeline design patterns (golden path templates, matrix builds),
Pipeline optimization (caching, parallelism, artifact reuse),
Pipeline security (credentials, RBAC, script approval),
SonarQube integration (quality gates, branch analysis),
Trivy/BlackDuck (security scanning in pipeline),
Artifactory (artifact management, promotion, cleanup policies),
ArgoCD advanced, Argo Rollouts (canary with analysis, blue-green),
Progressive delivery (feature flags, traffic shifting),
Pipeline debugging and failure patterns,
Webhook integrations (Bitbucket → Jenkins → Jira automation),
Monorepo CI strategies, Pipeline as Code best practices,
Cost optimization in CI (spot instances, caching, right-sizing)
```

---

## PHASE 4: IaC — Terraform + Ansible (100% ✅)

### Lessons 1-3: Terraform
```
Topics: Terraform architecture (init, plan, apply), HCL syntax,
providers, resources, data sources, variables, outputs, locals,
State management (S3 backend, DynamoDB locking, state structure),
State operations (mv, rm, import, taint, replace),
State file security, Modules (structure, versioning, registry),
Module composition, Workspaces,
Terraform in CI/CD (plan in PR, apply on merge, OIDC, artifact passing),
Atlantis and Terraform Cloud/Enterprise,
Testing (validate, tflint, tfsec/checkov, Terratest),
State refactoring (moved blocks, import blocks),
Terraform Cloud (Sentinel, cost estimation),
count vs for_each, dynamic blocks, lifecycle rules,
dependency graph resolution
Failure modes: state lock stuck, state corruption, drift,
dependency cycles, count/for_each index shift, partial apply
```

### Lessons 4-5: Ansible
```
Topics: Ansible architecture (agentless SSH), Ansible vs Terraform,
ansible.cfg (forks, pipelining, ControlMaster),
Static/dynamic inventory (AWS EC2 plugin), Variable precedence (22 levels),
group_vars/host_vars, Playbook anatomy (serial, max_fail_percentage),
Key modules (15+), Handlers, Jinja2 templates,
Conditionals, loops, register, Error handling (block/rescue/always),
Roles (full structure, production example), Ansible Galaxy,
Ansible Vault (encrypt, decrypt, rekey, vault-id, naming convention),
Vault vs External Secrets, Performance optimization (async, Mitogen),
Molecule testing (Docker + EC2, CI integration),
Terraform + Ansible integration patterns (4 patterns),
Tower/AWX (RBAC, workflows, credentials),
Delegation, import vs include, Lookup plugins, filters, tags
Failure modes: SSH timeout, Python missing, become failure,
idempotency broken, handler not running, variable collision,
slow execution, template rendering fails, partial failure,
secret leak, vault password lost, Molecule false positive
```

---

## PHASE 5: Observability & SRE (100% ✅)

### Lesson 1: Prometheus + Metrics + PromQL + Alertmanager
```
Topics: Three pillars of observability, Monitoring vs Observability,
Prometheus architecture (pull-based, TSDB, WAL, service discovery),
Pull vs Push model, Four metric types (Counter, Gauge, Histogram, Summary),
Metric naming conventions, Labels as dimensional data,
Cardinality (the Prometheus killer, <2M series target),
kube-prometheus-stack (Helm chart with full production values),
Three K8s metrics sources (node-exporter, kube-state-metrics, cAdvisor),
Service discovery CRDs (ServiceMonitor, PodMonitor, PrometheusRule),
metricRelabelings, PromQL (instant/range vectors, Four Golden Signals queries,
K8s-specific queries, 7 gotchas), Recording rules,
Alertmanager (routing tree, group_by, inhibition, receivers),
Full alertmanager.yml for NovaMart, Watchdog/dead man's switch,
10 Prometheus failure modes
```

### Lesson 2: Grafana + Loki + Long-term Storage (Thanos)
```
Topics: Grafana HA architecture, data source provisioning as code,
Dashboard design (Four Golden Signals), template variables,
$__rate_interval, Annotations, Dashboard as Code/GitOps,
Grafana Unified Alerting, 8 Grafana failure modes,
Loki architecture (Distributor → Ingester → Object Storage),
Deployment modes (Monolithic, SimpleScalable, Distributed),
Full Helm config, Log collection comparison (Promtail vs Fluent Bit vs OTel),
Promtail configuration, Label design (THE critical decision),
Structured metadata (Loki 3.0+),
LogQL (stream selector, line filters, parsers, label filters, metric queries),
10 production LogQL queries, 10 Loki failure modes,
Metrics ↔ Logs ↔ Traces correlation,
Thanos vs Mimir vs Cortex comparison,
Thanos architecture (Sidecar, Store GW, Query, Query Frontend, Compactor),
Full component configs, S3 cost optimization,
Thanos downsampling, external_labels for dedup,
10 Thanos failure modes, NovaMart observability data flow
```

### Lesson 3: Distributed Tracing + OpenTelemetry
```
Topics: Trace anatomy (trace, span, root span, child span, context, baggage),
Context propagation (#1 failure mode), W3C TraceContext,
Propagation formats (W3C, B3, Jaeger, X-Ray), multi-format migration,
OpenTelemetry architecture (SDK, Collector, auto-instrumentation, OTLP),
OTel Collector pipeline (Receivers → Processors → Exporters),
Full production Collector config,
Deployment patterns (DaemonSet, Sidecar, Agent+Gateway),
loadbalancing exporter with routing_key:traceID,
OTel Operator + Instrumentation CRD + auto-instrumentation,
Manual instrumentation (Go example),
Sampling math, Head vs Tail sampling,
Tempo architecture (TraceQL), Tempo vs Jaeger,
Tempo Metrics Generator (RED metrics FROM traces),
Complete correlation model (Exemplars, Log→Trace, Trace→Logs, Trace→Metrics),
Complete incident investigation flow (4-step),
10 tracing failure modes
```

### Lesson 4: SLOs, SLIs, Error Budgets, Burn Rate Alerting
```
Topics: SLI/SLO/SLA definitions, Error budget math,
SLI types menu (availability, latency, freshness, correctness, throughput, durability),
Good vs Bad SLIs, NovaMart SLO definitions per service,
SLI implementation in Prometheus recording rules,
Latency SLI using histogram bucket ratio (NOT histogram_quantile),
Error budget remaining calculation, Error budget policies,
Burn rate concept and math,
Multi-window multi-burn-rate alerting (Google SRE book),
Full burn rate alert implementation in PrometheusRule,
Grafana SLO dashboard design, 6 SLO anti-patterns,
Sloth (SLO spec → Prometheus rules generator)
```

### Lesson 5: Incident Management, On-Call, Postmortems
```
Topics: Incident lifecycle, severity levels (SEV1-SEV4),
Incident commander role, communication templates,
On-call best practices, escalation policies,
PagerDuty configuration and routing,
Postmortem methodology (blameless), contributing factors analysis,
Action items (prevent, detect, mitigate — with owners and deadlines),
NovaMart incident response runbook templates,
On-call handoff procedures, toil measurement
```

---

## PHASE 6: Security, Compliance, Cloud (100% ✅)

```
Topics: AWS IAM deep dive (policies, roles, trust relationships, permission boundaries),
IAM best practices (least privilege, IRSA, no long-lived keys),
AWS Organizations, SCPs, SSO/Identity Center,
Secrets management (HashiCorp Vault architecture, dynamic secrets,
  External Secrets Operator, rotation patterns),
OPA/Gatekeeper (policy as code, Rego basics, constraint templates),
Kyverno policies (mutate, validate, generate),
Supply chain security (SBOM, Cosign, SLSA framework, Sigstore),
Container security (image scanning, runtime security, Falco),
Network security (mTLS, NetworkPolicies, WAF rules),
Compliance frameworks (SOC2, PCI-DSS, HIPAA — what DevOps owns),
Audit logging (CloudTrail, K8s audit logs, ELK),
Certificate management (cert-manager, ACM, rotation),
AWS service deep dives (relevant services for NovaMart stack)
```

---

## PHASE 7: Build NovaMart (100% ✅)

### Lesson 1: Infrastructure Foundation (Grade: A-)
```
BUILT: VPC (3-AZ, dual CIDR, 12 subnets, 3 NAT GWs, 7 VPC endpoints, Flow Logs),
  EKS (managed system node group, Karpenter, 4 add-ons with custom networking, IRSA),
  Data Tier (3× RDS PostgreSQL Multi-AZ, ElastiCache Redis),
  Security Baseline (4 KMS keys, 4 S3 buckets, CloudTrail, Object Lock),
  Terraform (module-based, split state per domain, bootstrap module),
  Documentation (9 design decisions, CIDR map, apply order runbook, cost estimate)

REVIEW: 5 Critical, 7 Significant, 7 Minor findings
  Key criticals: circular dependency, duplicate modules, orphaned SG module,
  Object Lock flag, Karpenter EC2NodeClass field
  Key praise: Dual CIDR (textbook), ADRs (professional-grade), default SG lockdown,
  managed passwords, Object Lock on CloudTrail, Flow Logs dual destination
```

### Lesson 2: Platform Services (Grade: A+)
```
BUILT + REDELIVERED (all issues resolved):
  Observability: kube-prometheus-stack, Thanos, Loki, Tempo, OTel (Agent+Gateway),
    Fluent Bit (triple output), Grafana (SSO, dashboards as code, SLO dashboard)
  Service Mesh: Linkerd with cert-manager auto-rotation
  GitOps: ArgoCD (App of Apps, sync waves, sync windows, notifications, emergency procedures)
  Policy: Kyverno (with exclusions, PolicyExceptions, webhook timeout)
  Security: External Secrets Operator, cert-manager (DNS01), ResourceQuota/LimitRange
  Extras: ECR pull-through cache, Squid egress proxy, platform validation script,
    emergency procedures doc, certificate rotation schedule, resource budget table

REVIEW: Initial 5 Critical, 9 Significant, 7 Minor — ALL fixed in redelivery
  Pattern identified: architecture solid, operational singleton/safety constraints missed initially
```

### Lesson 3: CI/CD Pipeline (Grade: A)
```
BUILT: Bitbucket → Jenkins (EKS) → Kaniko → ECR → GitOps repo → ArgoCD → Argo Rollouts
  Jenkins on EKS (JCasC, ephemeral agents, SSO, shared library, backup),
  Agent images (Go, Java — non-root, all tools baked in),
  Shared library golden path (13-stage pipeline template),
  Security scanning (Trivy, Cosign keyless signing),
  GitOps repo (Kustomize base + overlays),
  Argo Rollouts (canary with AnalysisTemplates),
  SonarQube, Karpenter CI NodePool, Network policies,
  Design decisions, developer onboarding guide, validation script
```

### Lesson 4: Application Onboarding (Grade: Reviewed)
```
BUILT: Complete onboarding of payment-service as reference implementation
  Terraform data layer (RDS, Redis, connection pool math),
  K8s manifests (Deployment, Service, HPA, PDB, NetworkPolicies),
  OTel instrumentation (Go auto + manual),
  Canary deployment strategy with AnalysisTemplates,
  Database migrations (Flyway, init container ordering),
  Secrets management (ESO, rotation),
  SLO definitions and Prometheus rules,
  PCI compliance (audit logging, encryption, network isolation),
  Developer onboarding template (reusable for all services)
```

### Lesson 5: Operations Readiness
```
BUILT: SLOs in practice, full runbooks, DR exercise plans,
  on-call handoff documentation, operational validation,
  chaos engineering scenarios, GameDay templates
```

---

## PHASE 8: Operate NovaMart — Production Simulation (100% ✅)

```
Completed: Multi-week production simulation including:
  - Monday morning scenarios (Slack, PagerDuty, Jira, PR reviews)
  - Incidents (SEV1-SEV4) — investigated, mitigated, postmortem'd
  - Multi-week projects (migrations, new tooling rollouts)
  - Cross-team interactions (engineering, QA, security, management)
  - On-call rotations, vendor management, architecture reviews
  - Cost optimization exercises
  - DR exercises and GameDays
```

---

# 📊 RETENTION SCORE HISTORY

```
Phase 0 Lessons:                    Consistently 4/5 to 5/5
Phase 1 Lessons:                    Consistently 3.5/4 to 4/4
Phase 2 Lesson 1 (Git):            3.5/4
Phase 2 Lesson 1B (Git Commands):  4/4
Phase 2 Lesson 2 (Docker):         4/4
Phase 2 Lesson 3+3B+3C (K8s):      5/5
Phase 2 Lesson 4 (K8s Ops):        4/4
Phase 3:                            Completed as block
Phase 4 Lesson 4 (Ansible Basic):  4.6/5
Phase 4 Lesson 5 (Ansible Adv):    4.6/5
Phase 5 Lesson 1 (Prometheus):     4.8/5
Phase 5 Lesson 2 (Grafana/Loki):   4.8/5
Phase 5 Lesson 3 (Tracing/OTel):   4.8/5
Phase 5 Lesson 4 (SLOs):           4.8/5
Phase 7 Lesson 1 (Infra):          Grade A-
Phase 7 Lesson 2 (Platform):       Grade A+
Phase 7 Lesson 3 (CI/CD):          Grade A
Phase 7 Lesson 4 (App Onboard):    Grade A
Phase 7 Lesson 5 (Ops Ready):      Grade A
Phase 8 (Simulation):              Completed — demonstrated production readiness
```

---

# 👤 USER PROFILE

### Strengths:
- Strong diagnostic reasoning — challenges premises, follows evidence
- Understands "why" not just "what"
- SRE mindset: inventory before cleanup, prevent recurrence, document
- Connects concepts across domains (Linux → containers → K8s → production)
- Incident response instincts are sharp — sequences actions correctly under time pressure
- Closes every loop (rotation, scrub, audit, prevent)
- Production-oriented answers — explains "why it matters" not just "what"
- Three consecutive 4.8/5 scores on Phase 5 — gaps at mechanism precision level, not conceptual
- Architectural understanding and operational instincts are senior-level
- Phase 7 builds demonstrate strong Terraform module design, security-first thinking, cost awareness
- Design decisions documents are professional-grade ADRs
- Phase 8 simulation demonstrated ability to operate under pressure
- Identified the automation gap in the roadmap (shows ownership of own learning)

### Gaps to Watch:
- Occasionally misses "production gotcha" in favor of textbook answer
- Sometimes misses idempotency edge cases in own code
- Occasionally misses operational touches (Slack before fleet-wide changes, caching, approval gates)
- Tool-specific precision gaps (mechanism details vs concepts)
- Uses working path vs optimal path sometimes (logs-first vs exemplar shortcut)
- Phase 7 pattern: architecture solid, operational singleton/safety constraints missed initially
- **Untested: Production automation code (Go, Python, Bash)** ← Phase 9 will reveal true gaps
- **Untested: Interview articulation** ← Phase 11 will test ability to explain under pressure

---

# 📋 PHASE-BY-PHASE LESSON PLAN (REMAINING)

## Phase 9: Production Automation & Internal Tooling

### Lesson 1: Bash for DevOps
```
Topics:
  Shell scripting production standards:
    set -euo pipefail (and when each flag helps/hurts)
    trap (cleanup, signal handling, EXIT/ERR/INT/TERM)
    shellcheck (SC2086, SC2046, SC2034 — top violations)
  Production script patterns:
    Logging framework (timestamp, level, function name)
    Argument parsing (getopts, positional args, usage functions)
    Lock files (flock) to prevent concurrent execution
    Retry logic with exponential backoff
    Dry-run mode (every destructive script needs this)
    Idempotency patterns
  Text processing mastery:
    awk (field extraction, pattern matching, BEGIN/END blocks)
    sed (in-place editing, regex groups, production gotchas)
    jq (JSON: filter, map, select, reduce, @csv, --arg)
    yq (YAML manipulation for K8s manifests and Helm values)
    grep/egrep (regex patterns, -P for PCRE, -A/-B/-C context)
  Process management in scripts:
    Background processes, wait, parallel execution
    Signal forwarding in wrapper scripts (exec "$@")
    PID files and process guarding
  Container entrypoint scripts:
    Environment variable substitution (envsubst)
    Configuration file generation from env vars
    Health check scripts, readiness gate scripts
    Graceful shutdown handlers
  Makefiles & Task runners:
    GNU Make for DevOps workflows (.PHONY, variables, targets)
    Justfile (modern alternative)
    Taskfile.yml (Go-based, YAML syntax)
    Common targets: lint, test, build, deploy, clean
  Real NovaMart scripts:
    EKS node drain wrapper (with PDB validation, Slack notification)
    Database backup verification (RDS snapshot → restore → validate → cleanup)
    Certificate expiry checker (scan all certs, alert on <30 days)
    Deployment rollback automation (ArgoCD + Slack + Jira)
    Log collection for incident response (multi-source, archive, S3 upload)
    Disk cleanup (safe pruning with thresholds, logging, dry-run)
  Anti-patterns:
    Bash beyond 200 lines (rewrite in Python/Go)
    No error handling, no logging, no dry-run
    Parsing ls output, unquoted variables, eval usage
    Hardcoded paths/credentials, missing shebang
  Failure modes:
    Unquoted variable word splitting causes data loss
    Missing set -e allows silent failures
    Race conditions in concurrent scripts
    trap not firing (subshell context)
    Locale-dependent sort/grep behavior (LC_ALL=C)
    heredoc tab issues (<<- requires tabs not spaces)
```

### Lesson 2: Python for DevOps
```
Topics:
  Project structure and packaging:
    pyproject.toml (modern packaging), src layout
    Virtual environments (venv, pipx for CLI tools)
    Dependency management (pip-compile, Poetry, PDM)
    Type hints (mypy) — expected at senior level
  boto3 deep dive:
    Sessions, clients vs resources (clients preferred)
    Paginators (never manual next_token loops)
    Waiters (built-in + custom), error handling (botocore.exceptions)
    Retry configuration (adaptive mode)
    STS AssumeRole for cross-account
    Common services: EC2, S3, RDS, EKS, CloudWatch, SecretsManager,
      IAM, STS, Route53, ELB, AutoScaling
  HTTP/API automation:
    requests (sessions, auth, retries via urllib3.Retry + HTTPAdapter)
    httpx (async support)
    Rate limiting (ratelimit library, token bucket)
    Pagination handling for REST APIs
    API clients for PagerDuty, Jira, Slack, Confluence
  CLI frameworks:
    Click (groups, commands, options, arguments, context)
    Typer (modern, type-hint based)
    Rich (tables, progress bars, panels, syntax highlighting)
    Output formats (table, JSON, YAML — let user choose)
  Jinja2 templating:
    Config file generation (nginx, prometheus, alertmanager configs)
    K8s manifest generation, Helm values generation
    Template inheritance, custom filters
  Testing:
    pytest (fixtures, parametrize, markers, conftest.py)
    moto (AWS service mocking — every boto3 call tested without AWS)
    responses/httpretty (HTTP mocking)
    Coverage targets and CI integration
  Async patterns:
    asyncio + aiohttp for concurrent API calls
    Semaphore for concurrency limiting
    When async helps vs overkill
  Real NovaMart tools:
    AWS cost reporter (CUR analysis, team allocation, anomaly detection)
    EC2/EKS inventory tool (cross-account, cross-region, output table/JSON)
    Incident auto-responder (PagerDuty webhook → create Slack channel →
      assign commander → create Jira → gather initial diagnostics)
    Deployment orchestrator (multi-service coordinated rollout)
    Secret rotation script (Vault/ASM → verify → rotate → validate)
    Compliance checker (IAM audit, SG audit, encryption audit)
    Log analyzer (parse structured logs, extract error patterns, correlate)
  Failure modes:
    No retry logic (transient failures crash script)
    Credential leaks in logs (always mask secrets)
    Silent failures (bare except:, swallowed errors)
    Memory issues with large API responses (use pagination + streaming)
    Timezone bugs (always use UTC internally)
    Missing pagination (only getting first page of results)
    boto3 client not respecting region (explicit region_name)
```

### Lesson 3: Go for DevOps
```
Topics:
  Why Go for DevOps:
    Single binary distribution (no runtime dependencies)
    Fast compilation, cross-compilation (GOOS/GOARCH)
    Concurrency primitives (goroutines, channels)
    K8s ecosystem is Go-native (client-go, controllers, operators)
    Static typing catches bugs at compile time
  Project structure:
    cmd/ (entrypoints), internal/ (private), pkg/ (public)
    go.mod/go.sum, Go workspace (go.work for monorepo)
    Makefile targets (build, test, lint, release)
  CLI tools with Cobra + Viper:
    Command hierarchy, persistent flags, flag binding
    Viper config (file, env, flags — priority order)
    Shell completions generation
    Man page generation
  K8s client-go:
    Clientset, dynamic client, discovery client
    Informers (SharedInformerFactory, event handlers, resync)
    Listers (cached reads from informer store)
    Contexts and cancellation
    In-cluster config vs kubeconfig
    Watch + retry patterns
  Building K8s controllers/operators:
    Kubebuilder scaffolding
    Reconciliation loop pattern
    Status subresource updates
    Finalizers in controllers
    Leader election for HA
    RBAC generation from markers
    Testing controllers (envtest)
  HTTP servers:
    net/http patterns (handlers, middleware, mux)
    Health endpoints (/healthz, /readyz, /metrics)
    Webhook receivers (Bitbucket, PagerDuty, generic)
    Admission webhooks (mutating + validating)
    Prometheus metrics exposition (promhttp)
    Graceful shutdown (context, signal handling)
  Concurrency patterns:
    goroutines + channels (fan-out/fan-in)
    errgroup (bounded concurrency with error propagation)
    context.Context (timeout, cancellation, value propagation)
    sync.WaitGroup, sync.Mutex, sync.Once
    Worker pool pattern
  Error handling:
    errors.Is, errors.As, error wrapping (fmt.Errorf %w)
    Custom error types, sentinel errors
    Don't panic — handle errors explicitly
  Distribution:
    GoReleaser (multi-arch, checksums, changelogs)
    Distroless containers (gcr.io/distroless/static-debian12)
    Homebrew tap generation
  Real NovaMart tools:
    Cluster audit CLI (resource usage, orphaned resources, policy violations)
    Deployment validator (pre-deploy checks: images exist, secrets present,
      quota available, PDB won't block)
    Custom admission webhook (enforce NovaMart standards: labels, resource
      limits, image registry, probe requirements)
    Slack bot for incidents (slash commands: /incident create, /runbook,
      /rollback, /oncall)
    K8s operator for NovaMart CRD (NovaMartService — automates onboarding:
      namespace, RBAC, quotas, network policies, monitoring)
  Failure modes:
    Goroutine leaks (missing context cancellation, channel not closed)
    Nil pointer dereference (uninitialized pointers, interface nil check trap)
    Context misuse (context.Background() mid-chain breaks cancellation)
    Race conditions (go test -race, sync primitives)
    Client-go informer cache stale reads vs direct API calls
    Connection pool exhaustion (http.Client reuse, transport settings)
    Binary size bloat (ldflags -s -w, UPX cautiously)
```

### Lesson 4: Integration Patterns & Production Tooling
```
Topics:
  Pre-commit hooks ecosystem:
    pre-commit framework (Python-based, .pre-commit-config.yaml)
    Hooks: shellcheck, yamllint, terraform fmt/validate, golangci-lint,
      detect-secrets, hadolint, markdownlint, commitlint
    CI enforcement (pre-commit run --all-files in pipeline)
  Building the NovaMart platform CLI:
    novactl (Go + Cobra): unified developer interface
    Commands: service create, service deploy, service status,
      cluster info, secrets rotate, runbook show, incident create
    Plugin architecture for extensibility
  Webhook handlers:
    Bitbucket → custom validation → Slack notification
    PagerDuty → Slack channel creation → Jira ticket
    ArgoCD → deployment tracking → metrics
    Generic pattern: receiver → validate → transform → dispatch
  ChatOps:
    Slack bot architecture (Socket Mode vs Events API)
    Command handlers: deploy, rollback, status, oncall, runbook
    Approval workflows via Slack
    Audit trail for all ChatOps actions
  K8s CronJobs for operational tasks:
    Certificate expiry scanner
    Orphaned resource cleaner
    Cost report generator
    Compliance audit (weekly)
    Backup verification
  Toil measurement and elimination:
    What is toil (manual, repetitive, automatable, no enduring value)
    Toil taxonomy and tracking
    Automation priority framework (frequency × time × risk)
    Toil budget (≤50% per Google SRE book)
  When to use what — decision framework:
    Bash: <200 lines, text processing, glue, entrypoints, one-off
    Python: API automation, data processing, complex logic, prototyping
    Go: CLI tools, K8s integration, long-running services, performance-critical
    Decision tree with examples
  Failure modes:
    Webhook security (no signature verification → anyone can trigger)
    ChatOps without audit trail (no accountability)
    CronJob overlap (concurrencyPolicy not set → parallel runs)
    Platform CLI version drift (no auto-update mechanism)
    Pre-commit hooks too slow (developers bypass with --no-verify)
```

---

## Phase 10: Production Depth

### Lesson 1: Database Operations for DevOps
```
Topics:
  PostgreSQL operations:
    EXPLAIN/EXPLAIN ANALYZE (reading query plans)
    Vacuum (autovacuum tuning, bloat, dead tuples)
    Locks (row-level, table-level, lock monitoring, deadlock detection)
    Connection management (PgBouncer vs RDS Proxy, pool sizing math)
    Replication (streaming, logical, replication lag monitoring)
    Performance Insights (AWS), pg_stat_statements
    Index management (unused indexes, missing indexes, pg_stat_user_indexes)
  Schema migration strategies:
    Expand-contract pattern (backward compatible deployments)
    Blue-green schema migrations
    Flyway/Liquibase in CI/CD pipelines
    Migration rollback strategies
    Zero-downtime migrations (add column, backfill, rename)
    Large table migrations (pt-online-schema-change patterns)
  Backup and recovery:
    RDS automated backups, manual snapshots, PITR
    Cross-region backup replication
    Restore testing automation (weekly restore → validate → cleanup)
    RTO/RPO validation
  Redis operations:
    Memory management (maxmemory, eviction policies)
    Key patterns and anti-patterns (large keys, hot keys)
    Cluster mode vs replication group
    ElastiCache metrics that matter
    Failover testing
  Failure modes:
    Connection pool exhaustion (symptoms, investigation, fix)
    Long-running queries blocking vacuum
    Replication lag during heavy writes
    OOM on analytics queries
    Split-brain in failover
    Migration rollback failure (already dropped column)
    Redis maxmemory reached, eviction cascade
```

### Lesson 2: FinOps & Cost Engineering
```
Topics:
  AWS cost model:
    On-demand, Reserved Instances, Savings Plans, Spot
    When to use each (decision framework)
    Compute Savings Plans vs EC2 Savings Plans vs RIs
  Cost visibility:
    Cost Explorer, CUR (Cost and Usage Report)
    Cost allocation tags (aws:eks:cluster-name, team, environment, service)
    Kubecost / OpenCost for K8s cost attribution
    Cost anomaly detection and alerting (AWS + custom)
  Right-sizing methodology:
    VPA recommendations → CloudWatch metrics → action
    Instance type comparison (Graviton ARM vs x86, cost savings)
    EBS optimization (GP3 vs GP2 vs IO2, IOPS/throughput provisioning)
    RDS instance right-sizing
  Spot strategies:
    Karpenter consolidation and spot diversification
    Spot interruption handling (2-minute warning, rebalancing)
    Spot capacity pools, allocation strategy
    Mixed instance policies
  Storage optimization:
    S3 lifecycle policies (Standard → IA → Glacier → Deep Archive)
    S3 Intelligent-Tiering
    EBS snapshot lifecycle (DLM policies)
    ECR image lifecycle policies
  Data transfer costs (the hidden killer):
    NAT Gateway processing ($0.045/GB — often largest bill item)
    Cross-AZ traffic ($0.01/GB each way)
    VPC endpoints to eliminate NAT costs
    CloudFront vs direct S3
    Data transfer between regions
  FinOps practices:
    Showback/chargeback per team
    Monthly cost review cadence
    Budget alerts and anomaly detection
    Reserved capacity planning
    Waste detection (idle resources, unattached EBS, unused EIPs)
  NovaMart exercise:
    Build cost dashboard (Kubecost + CUR + CloudWatch)
    Identify $X/month savings opportunities
    Propose Savings Plan strategy
    Implement cost allocation tags across infrastructure
  Failure modes:
    Savings Plan over-commitment (locked in, can't reduce)
    Spot interruption during deployment
    Cost tags not propagated (invisible spend)
    NAT Gateway costs explode (missing VPC endpoint)
    S3 lifecycle deletes data still needed
    Dev environments running 24/7 at production size
```

### Lesson 3: Chaos Engineering & DR
```
Topics:
  Chaos engineering methodology:
    Steady-state hypothesis (define "normal" with metrics)
    Experiment design (what, blast radius, abort conditions)
    Run, observe, analyze, improve
    Chaos maturity model (from manual to automated)
  Tools:
    Litmus Chaos (K8s native, ChaosEngine, ChaosExperiment CRDs)
    AWS Fault Injection Simulator (FIS — managed, IAM controlled)
    Chaos Mesh (K8s native, fine-grained network/IO/stress)
    Gremlin (SaaS, enterprise, easy to start)
  GameDay planning and execution:
    Stakeholder alignment, scope definition
    Communication plan (who knows, who doesn't)
    Runbook validation through chaos
    Post-GameDay analysis and improvements
    NovaMart: design and execute 3 GameDays:
      1. AZ failure (cordon all nodes in one AZ)
      2. Database failover (RDS Multi-AZ forced failover)
      3. Service mesh failure (kill Linkerd control plane)
  DR patterns:
    Active-active (multi-region, Route53 latency routing)
    Active-passive (standby region, failover routing)
    Pilot light (minimal infra in DR, scale on failover)
    Warm standby (scaled-down but running)
    DR pattern selection (RTO/RPO requirements → pattern)
  DR exercises:
    RDS failover test (measure actual RTO)
    Region failover simulation
    AZ failure simulation (NACLs to simulate AZ loss)
    Backup restore validation
    DNS failover testing
  RTO/RPO:
    Definitions, calculation methodology
    Testing and validation cadence
    Documentation and reporting
  Failure modes:
    Chaos experiment escapes blast radius
    GameDay reveals missing runbooks
    DR failover works but failback doesn't
    RDS Multi-AZ failover takes longer than expected
    Data loss during region failover (replication lag)
    DNS TTL prevents quick failover
    DR environment configuration drift from production
```

### Lesson 4: Platform Engineering & Developer Experience
```
Topics:
  Internal Developer Platforms (IDP):
    What, why, maturity model (ad-hoc → self-service → product)
    Build vs buy decision framework
    Platform as product (users = developers)
  Backstage / Port:
    Service catalog (ownership, dependencies, documentation)
    Software templates (scaffolding new services)
    TechDocs (docs-as-code in Backstage)
    Plugins ecosystem
  Golden paths:
    Opinionated defaults vs escape hatches
    NovaMart golden path: service template → CI/CD → observability →
      security → deployment → all automated
    Measuring golden path adoption
  DORA metrics:
    Deployment frequency
    Lead time for changes
    Change failure rate
    Mean time to recovery (MTTR)
    How to measure each in NovaMart stack
    Targets by performance tier (elite, high, medium, low)
  Developer experience:
    Developer surveys and feedback loops
    Friction logs (identify pain points)
    Time-to-first-deploy metric
    Self-service infrastructure
    Documentation standards
  Multi-tenancy in K8s:
    Namespace-per-team, cluster-per-team (tradeoffs)
    Resource quotas, limit ranges, network policies
    RBAC per team, audit logging
    Hierarchical namespaces (HNC)
  Failure modes:
    Platform team becomes bottleneck (no self-service)
    Golden path too rigid (developers bypass it)
    DORA metrics gamed (small PRs, no real improvement)
    Backstage adoption stalls (stale catalog, no ownership)
    Multi-tenancy noisy neighbor (resource contention)
```

---

## Phase 11: FAANG Interview Preparation

### Lesson 1: System Design for Infrastructure
```
Topics:
  Framework: requirements → constraints → architecture →
    tradeoffs → scaling → failure modes → monitoring
  Practice problems:
    Design a CI/CD platform for 500 microservices
    Design a multi-region deployment strategy
    Design an observability platform at scale
    Design a secrets management system
    Design a cost optimization platform
    Design a Kubernetes multi-tenant platform
  Whiteboard/diagramming practice
  Communication during design (narrate tradeoffs)
  Time management (45 min structure)
```

### Lesson 2: Live Troubleshooting
```
Topics:
  Timed incident scenarios (15-30 min):
    "Your K8s cluster is down" — systematic investigation
    "Latency spiked 10x" — investigation methodology
    "Deployment broke production" — rollback + RCA
    "Database connections exhausted" — diagnose + fix
    "Disk full on nodes" — prioritize + fix + prevent
  Communication during troubleshooting:
    Narrate your thinking out loud
    Ask clarifying questions (what changed recently?)
    State assumptions explicitly
    Prioritize (impact first, root cause second)
```

### Lesson 3: Coding Challenges
```
Topics:
  Write a deployment CLI tool (Go, 45 min)
  Write an AWS cleanup script (Python, 30 min)
  Write a log parser / metric extractor (Bash/Python)
  Debug a broken Terraform module (find the bugs)
  Debug a broken K8s manifest (find the misconfigs)
  Code review exercises (evaluate someone else's PR)
  Focus: clean code, error handling, edge cases, testability
```

### Lesson 4: Behavioral & Leadership
```
Topics:
  STAR format mastery (Situation, Task, Action, Result)
  Amazon Leadership Principles mapped to DevOps scenarios
  Google "Googleyness" and collaboration signals
  Meta infrastructure culture
  Story bank:
    Incident you led (SEV1, communication, resolution)
    System you designed (tradeoffs, scale)
    Conflict you resolved (technical disagreement)
    Mistake you made (what you learned)
    Process you improved (before/after metrics)
  Questions to ask interviewers (infrastructure-specific)
  Negotiation basics (levels, comp structure, competing offers)
```

### Lesson 5: Mock Interviews (Full Loops)
```
Topics:
  Mock phone screen (45 min — mix of technical + behavioral)
  Mock system design (60 min — full whiteboard exercise)
  Mock coding (45 min — build a tool under pressure)
  Mock behavioral (45 min — STAR stories, follow-ups)
  Feedback and iteration after each mock
  Interviewer calibration (what they're actually scoring)
```

---

# 📊 WHERE TO RESUME

```
╔═══════════════════════════════════════════════════════════════╗
║  CURRENT POSITION: Phase 9, Lesson 1 — Bash for DevOps      ║
║  STATUS: NOT STARTED                                         ║
║                                                              ║
║  Action: Begin Phase 9, Lesson 1 teaching                    ║
║                                                              ║
║  Remaining lessons: 13                                       ║
║    Phase 9:  4 lessons (Bash, Python, Go, Integration)       ║
║    Phase 10: 4 lessons (DB Ops, FinOps, Chaos/DR, Platform)  ║
║    Phase 11: 5 lessons (SysDesign, Troubleshoot, Code,       ║
║              Behavioral, Mocks)                              ║
╚═══════════════════════════════════════════════════════════════╝
```

---

**END OF HANDOFF DOCUMENT v2 — APPROVED ROADMAP**
