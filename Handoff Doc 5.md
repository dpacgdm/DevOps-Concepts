# UPDATED HANDOFF DOCUMENT v3

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

# 🔄 COMPLETE ROADMAP (APPROVED v3)

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

  Phase 9: Production Automation & Internal Tooling  ██████████████████░░  90%
    → Lesson 1: Bash for DevOps                      ████████████████████ 100% ✅
    → Lesson 2: Python for DevOps                    ████████████████████ 100% ✅
    → Lesson 3: Go for DevOps                        ████████████████████ 100% ✅
    → Lesson 4: Integration Patterns & Prod Tooling  ██████████████████░░ TAUGHT — AWAITING ANSWERS

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

  Overall Progress:  ████████████████░░░░ ~76%
  Completed: 9 phases + 3 lessons (Phases 0-8, Phase 9 L1-L3)
  In Progress: Phase 9 Lesson 4 (taught, awaiting retention answers)
  Remaining: 2.1 phases, 9 lessons (Phase 9 L4 grading, Phases 10-11)

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

## PHASE 9: Production Automation & Internal Tooling (90% — In Progress)

### Lesson 1: Bash for DevOps (100% ✅)
```
Topics: set -euo pipefail (when each flag helps/hurts),
  trap (cleanup, signal handling, EXIT/ERR/INT/TERM),
  shellcheck (SC2086, SC2046, SC2034 — top violations),
  Logging framework (timestamp, level, function name),
  Argument parsing (getopts, positional args, usage functions),
  Lock files (flock) to prevent concurrent execution,
  Retry logic with exponential backoff,
  Dry-run mode (every destructive script needs this),
  Idempotency patterns,
  Text processing: awk (field extraction, pattern matching, BEGIN/END),
  sed (in-place editing, regex groups, production gotchas),
  jq (JSON: filter, map, select, reduce, @csv, --arg),
  yq (YAML manipulation for K8s manifests and Helm values),
  grep/egrep (regex patterns, -P for PCRE, -A/-B/-C context),
  Process management in scripts (background, wait, parallel, exec "$@"),
  PID files, signal forwarding in wrapper scripts,
  Container entrypoint scripts (envsubst, config generation,
    health check scripts, graceful shutdown handlers),
  Makefiles & Task runners (GNU Make, Justfile, Taskfile.yml),
  NovaMart scripts: EKS node drain wrapper, DB backup verification,
    cert expiry checker, deployment rollback automation,
    log collection for incidents, disk cleanup
Failure modes: unquoted variable word splitting, missing set -e,
  race conditions, trap not firing (subshell), locale-dependent behavior,
  heredoc tab issues, bash beyond 200 lines
```

### Lesson 2: Python for DevOps (100% ✅)
```
Topics: Project structure (pyproject.toml, src layout, venv, pipx),
  Dependency management (pip-compile, Poetry, PDM),
  Type hints (mypy — expected at senior level),
  boto3 deep dive (sessions, clients vs resources, paginators,
    waiters, error handling, retry config, STS AssumeRole,
    common services: EC2, S3, RDS, EKS, CloudWatch, SecretsManager, IAM),
  HTTP/API automation (requests sessions, retries via urllib3.Retry + HTTPAdapter,
    httpx async, rate limiting, pagination, API clients for PD/Jira/Slack),
  CLI frameworks (Click, Typer, Rich tables/progress/panels),
  Output formats (table, JSON, YAML — user choice),
  Jinja2 templating (config generation, K8s manifests, Helm values,
    template inheritance, custom filters),
  Testing (pytest fixtures/parametrize/markers/conftest, moto for AWS mocking,
    responses/httpretty for HTTP mocking, coverage + CI),
  Async patterns (asyncio + aiohttp, semaphore concurrency limiting),
  NovaMart tools: AWS cost reporter, EC2/EKS inventory tool,
    incident auto-responder (PD webhook → Slack → Jira → diagnostics),
    deployment orchestrator, secret rotation script,
    compliance checker (IAM/SG/encryption audit), log analyzer
Failure modes: no retry logic, credential leaks in logs,
  silent failures (bare except:), memory issues with large responses,
  timezone bugs (always UTC), missing pagination, boto3 region not set
```

### Lesson 3: Go for DevOps (100% ✅)
```
Topics: Why Go for DevOps (single binary, cross-compilation, concurrency,
    K8s ecosystem native, static typing),
  Project structure (cmd/, internal/, pkg/, go.mod/go.sum, go.work),
  CLI tools with Cobra + Viper (command hierarchy, persistent flags,
    flag binding, shell completions, man pages),
  K8s client-go (clientset, dynamic client, discovery client,
    informers/SharedInformerFactory, listers, contexts, cancellation,
    in-cluster vs kubeconfig, watch + retry patterns),
  Building K8s controllers/operators (Kubebuilder, reconciliation loop,
    status subresource, finalizers, leader election, RBAC markers, envtest),
  HTTP servers (net/http patterns, health endpoints, webhook receivers,
    admission webhooks, promhttp metrics, graceful shutdown),
  Concurrency (goroutines + channels, fan-out/fan-in, errgroup,
    context.Context, sync.WaitGroup/Mutex/Once, worker pool pattern),
  Error handling (errors.Is/As, wrapping with %w, custom types, sentinels),
  Distribution (GoReleaser, distroless containers, Homebrew tap),
  NovaMart tools: cluster audit CLI, deployment validator,
    custom admission webhook, Slack bot for incidents,
    K8s operator for NovaMartService CRD
Failure modes: goroutine leaks, nil pointer dereference,
  context misuse (Background mid-chain), race conditions (go test -race),
  client-go informer cache stale reads, connection pool exhaustion,
  binary size bloat (ldflags -s -w)
```

### Lesson 4: Integration Patterns & Production Tooling (TAUGHT — AWAITING ANSWERS)
```
Topics taught:
  Pre-commit hooks ecosystem:
    .pre-commit-config.yaml (full production config with 15+ hooks),
    Hooks: trailing-whitespace, check-yaml, detect-secrets, shellcheck,
      ruff (lint+format), mypy, golangci-lint, go-fmt, go-vet,
      terraform_fmt/validate/tflint/tfsec/terraform_docs,
      hadolint, yamllint, markdownlint, conventional-pre-commit,
    .yamllint.yml (K8s-friendly config),
    CI enforcement (pre-commit run --all-files in pipeline),
    4 failure modes (hooks too slow → bypass, stale baseline,
      not installed on new machines, version drift)

  NovaMart Platform CLI (novactl):
    Full command architecture (service, cluster, secrets, cost,
      incident, compliance, completion subcommands),
    Service scaffolding golden path (Cobra command, flags, validation),
    Scaffold template engine (Go embed.FS, template.FuncMap,
      conditional templates based on data layer, ServiceSpec model,
      tier-based resource defaults),
    Example deployment.yaml.tmpl with NovaMart standards

  Webhook handler patterns:
    Integration architecture diagram (Bitbucket, AlertManager, PagerDuty,
      ArgoCD → novatools-webhook → Slack, Jira, K8s),
    Idempotency via Redis SETNX deduplication,
    Fail-open strategy on Redis errors,
    Event fingerprinting per provider

  ChatOps — Slack bot:
    Socket Mode vs Events API (Socket Mode works behind VPN),
    Full bot implementation (Go + slack-go/socketmode),
    CommandRouter pattern (group:subcommand dispatch),
    Handlers: deploy status, rollback (with prod approval flow),
      oncall (PagerDuty query), runbook lookup, cluster nodes,
      incident create (Slack channel + topic + Jira + context),
    5 failure modes (no audit trail, no approval flow,
      bot token too powerful, blocking handlers, no rate limiting)

  K8s CronJobs for operational tasks:
    5 production CronJob specs:
      cert-expiry-scanner (daily), orphan-resource-cleaner (weekly),
      compliance-audit (weekly), cost-report (daily),
      backup-verify (daily),
    All with: concurrencyPolicy:Forbid, activeDeadlineSeconds,
      startingDeadlineSeconds, IRSA, resource limits, pinned images,
    5 failure modes (missing concurrencyPolicy, no deadline,
      no startingDeadlineSeconds, silent failures without monitoring,
      :latest tag drift),
    PrometheusRule for CronJob monitoring

  Toil measurement and elimination:
    Google SRE toil definition (manual, repetitive, automatable,
      tactical, no enduring value, scales linearly),
    Target: ≤50% engineering time on toil,
    NovaMart toil tracking sheet (9 tasks, monthly hours),
    Priority formula: Frequency × Time × Risk × (1 / Difficulty),
    Scored priority table with automation decisions

  When to use what — decision framework:
    Language decision tree (Bash <100 lines/glue, Python APIs/data,
      Go CLI/K8s/long-running, Terraform IaC),
    Automation pattern decision matrix (8 trigger types mapped to
      pattern, language, and example),
    5 anti-patterns:
      1. Mega Script (decompose into single-purpose tools)
      2. Automate Everything Immediately (use priority score)
      3. Internal Tool With No Tests (same rigor as product code)
      4. ChatOps for Everything (convenience layer, not SPOF)
      5. Webhook Does Too Much (ack + enqueue, process async)

  Platform operations map:
    Developer workflow (create → pre-commit → CI → PR → ArgoCD),
    Day-2 operations (5 CronJobs),
    Incident response flow (AlertManager → auto-remediate → PD → Slack),
    Self-service goals (deploy, rotate, logs without platform team)

  Platform maturity model:
    Level 1: Everything manual (>80% toil)
    Level 2: Basic automation (~60% toil)
    Level 3: Platform team with tooling (~35% toil) ← NovaMart target
    Level 4: Self-service platform (<25% toil)
    Level 5: Fully autonomous operations (<15% toil)

RETENTION QUESTIONS PENDING (4 questions):
  Q1: Design novactl service deploy (7-step orchestration with rollback)
  Q2: Debug a buggy webhook handler (find every bug in code)
  Q3: Write webhook deduplicator + async processor (Go, with tests)
  Q4: Toil analysis (priority scoring, automation decisions, savings projection)
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
Phase 9 Lesson 1 (Bash):           Completed ✅
Phase 9 Lesson 2 (Python):         Completed ✅
Phase 9 Lesson 3 (Go):             Completed ✅
Phase 9 Lesson 4 (Integration):    AWAITING RETENTION ANSWERS
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
- **Phase 9 Lessons 1-3 completed: demonstrated Bash, Python, Go production automation skills**

### Gaps to Watch:
- Occasionally misses "production gotcha" in favor of textbook answer
- Sometimes misses idempotency edge cases in own code
- Occasionally misses operational touches (Slack before fleet-wide changes, caching, approval gates)
- Tool-specific precision gaps (mechanism details vs concepts)
- Uses working path vs optimal path sometimes (logs-first vs exemplar shortcut)
- Phase 7 pattern: architecture solid, operational singleton/safety constraints missed initially
- **Phase 9 Lesson 4 retention answers pending — will reveal integration thinking quality**
- **Untested: Interview articulation** ← Phase 11 will test ability to explain under pressure

---

# 📋 PHASE-BY-PHASE LESSON PLAN (REMAINING)

## Phase 9: Production Automation & Internal Tooling (Lesson 4 in progress)

### Lesson 4: Integration Patterns & Production Tooling — RETENTION ANSWERS PENDING
```
4 retention questions asked:
  Q1: Design novactl service deploy (end-to-end 7-step orchestration)
  Q2: Debug buggy webhook handler (find every bug)
  Q3: Write webhook deduplicator + async processor (Go + tests)
  Q4: Toil analysis with priority scoring and automation recommendations

Status: Grade answers → complete Phase 9 → proceed to Phase 10
```

---

## Phase 10: Production Depth

### Lesson 1: Database Operations for DevOps
```
Topics:
  PostgreSQL operations (EXPLAIN, vacuum, locks, connection pooling,
    replication, Performance Insights, pg_stat_statements, index mgmt)
  Schema migration strategies (expand-contract, blue-green, Flyway,
    zero-downtime, large table migrations)
  Backup and recovery (RDS automated, PITR, cross-region, restore testing)
  Redis operations (memory mgmt, eviction, cluster mode, failover)
  Failure modes (pool exhaustion, vacuum blocked, replication lag,
    OOM on analytics, split-brain, migration rollback, Redis eviction)
```

### Lesson 2: FinOps & Cost Engineering
```
Topics:
  AWS cost model (On-demand, RI, Savings Plans, Spot, decision framework)
  Cost visibility (Cost Explorer, CUR, cost allocation tags, Kubecost/OpenCost)
  Right-sizing (VPA recommendations, Graviton, EBS optimization, RDS)
  Spot strategies (Karpenter, interruption handling, diversification)
  Storage optimization (S3 lifecycle, Intelligent-Tiering, EBS DLM, ECR lifecycle)
  Data transfer costs (NAT Gateway, cross-AZ, VPC endpoints, CloudFront)
  FinOps practices (showback/chargeback, budgets, anomaly detection, waste detection)
  NovaMart exercise (cost dashboard, savings opportunities, SP strategy, tags)
```

### Lesson 3: Chaos Engineering & DR
```
Topics:
  Chaos methodology (steady-state hypothesis, experiment design, maturity model)
  Tools (Litmus Chaos, AWS FIS, Chaos Mesh, Gremlin)
  GameDay planning (3 NovaMart GameDays: AZ failure, DB failover, mesh failure)
  DR patterns (active-active, active-passive, pilot light, warm standby)
  DR exercises (RDS failover, region failover, AZ failure, backup restore, DNS failover)
  RTO/RPO (definitions, calculation, testing, documentation)
```

### Lesson 4: Platform Engineering & Developer Experience
```
Topics:
  Internal Developer Platforms (IDP maturity model, build vs buy)
  Backstage/Port (service catalog, software templates, TechDocs, plugins)
  Golden paths (opinionated defaults, escape hatches, adoption metrics)
  DORA metrics (4 metrics, how to measure in NovaMart, performance tiers)
  Developer experience (surveys, friction logs, time-to-first-deploy, self-service)
  Multi-tenancy in K8s (namespace-per-team, quotas, RBAC, HNC)
```

---

## Phase 11: FAANG Interview Preparation

### Lesson 1: System Design for Infrastructure
```
Topics:
  Framework: requirements → constraints → architecture →
    tradeoffs → scaling → failure modes → monitoring
  Practice problems (6 infrastructure design problems)
  Whiteboard/diagramming, communication, time management
```

### Lesson 2: Live Troubleshooting
```
Topics:
  Timed incident scenarios (15-30 min each, 5 scenarios)
  Communication during troubleshooting (narrate thinking, ask clarifying questions)
```

### Lesson 3: Coding Challenges
```
Topics:
  Timed coding exercises (Go CLI, Python cleanup, Bash parser)
  Debug broken Terraform/K8s manifests
  Code review exercises
  Focus: clean code, error handling, edge cases, testability
```

### Lesson 4: Behavioral & Leadership
```
Topics:
  STAR format, Amazon LPs, Google Googleyness, Meta infra culture
  Story bank (5 story types), questions for interviewers, negotiation basics
```

### Lesson 5: Mock Interviews (Full Loops)
```
Topics:
  Mock phone screen, system design, coding, behavioral (45-60 min each)
  Feedback, iteration, interviewer calibration
```

---

# 📊 WHERE TO RESUME

```
╔═══════════════════════════════════════════════════════════════╗
║  CURRENT POSITION: Phase 9, Lesson 4 — Integration Patterns  ║
║  STATUS: TAUGHT — AWAITING RETENTION ANSWERS                 ║
║                                                              ║
║  Action: Grade retention answers for Q1-Q4 when submitted    ║
║          → Complete Phase 9 → Begin Phase 10, Lesson 1       ║
║                                                              ║
║  Remaining after grading: 9 lessons                          ║
║    Phase 10: 4 lessons (DB Ops, FinOps, Chaos/DR, Platform)  ║
║    Phase 11: 5 lessons (SysDesign, Troubleshoot, Code,       ║
║              Behavioral, Mocks)                              ║
╚═══════════════════════════════════════════════════════════════╝
```

---

**END OF HANDOFF DOCUMENT v3 — Phase 9 Lessons 1-3 Complete, Lesson 4 Awaiting Grading**
