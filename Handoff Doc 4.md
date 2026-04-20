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

After foundations (Phases 0-6), the user will be deployed as Senior DevOps/Platform Engineer at NovaMart.

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
              ┌────────────────▼────────────────┐
              │         AWS EKS Cluster          │
              │  ┌─────────────────────────────┐ │
              │  │      Istio Service Mesh      │ │
              │  │  ┌─────┐ ┌─────┐ ┌────────┐ │ │
              │  │  │ API │ │Cart │ │Payment │ │ │
              │  │  │ GW  │ │ Svc │ │  Svc   │ │ │
              │  │  └──┬──┘ └──┬──┘ └───┬────┘ │ │
              │  │     │       │        │      │ │
              │  │  ┌──▼──┐ ┌─▼────┐ ┌─▼───┐  │ │
              │  │  │User │ │Order │ │Notif │  │ │
              │  │  │ Svc │ │ Svc  │ │ Svc  │  │ │
              │  │  └─────┘ └──────┘ └──────┘  │ │
              │  └─────────────────────────────┘ │
              └────────────────┬────────────────┘
                               │
         ┌─────────┬───────────┼───────────┬──────────┐
         │         │           │           │          │
    ┌────▼───┐ ┌───▼────┐ ┌───▼───┐ ┌────▼────┐ ┌───▼────┐
    │AWS RDS │ │MongoDB │ │Redis  │ │RabbitMQ │ │  S3    │
    │Postgres│ │(Atlas) │ │Elasti │ │ + SQS   │ │Static  │
    │+ Proxy │ │        │ │Cache  │ │         │ │Assets  │
    └────────┘ └────────┘ └───────┘ └─────────┘ └────────┘
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
**Responsibilities:** Daily operations, incident response, CI/CD enablement, automation (Go/Python), SDLC integrations, architecture design, reliability (SLO/SLI), DR exercises, runbooks, postmortems, cost optimization

### Simulation Format (Phase 8):
- Monday morning scenarios (Slack, PagerDuty, Jira, PR reviews)
- Incidents (SEV1-SEV4) randomly injected
- Multi-week projects (migrations, new tooling)
- Cross-team interactions (engineering, QA, security, management)
- On-call rotations, vendor management, architecture reviews

---

# 🔄 COMPLETE ROADMAP

```
PART 1: FOUNDATIONS (Phases 0-6)
  Phase 0: Linux Deep Dive                    ████████████████████ 100% ✅
  Phase 1: Networking                          ████████████████████ 100% ✅
  Phase 2: Git, Docker, K8s                    ████████████████████ 100% ✅
  Phase 3: CI/CD Pipelines                     ████████████████████ 100% ✅
  Phase 4: IaC (Terraform/Ansible)             ████████████████████ 100% ✅
  Phase 5: Observability & SRE                 ████████████████░░░░  80%
    → Lesson 1: Prometheus + Metrics + PromQL + Alertmanager ✅
    → Lesson 2: Grafana + Loki + Long-term Storage (Thanos) ✅
    → Lesson 3: Distributed Tracing + OpenTelemetry ✅
    → Lesson 4: SLOs, SLIs, Error Budgets ✅ (Qs PENDING ANSWERS)
    → Lesson 5: Incident Management, On-Call, Postmortems (REMAINING)
  Phase 6: Security, Compliance, AWS Services  ░░░░░░░░░░░░░░░░░░░░   0%

PART 2: BUILD (Phase 7)
  Phase 7: Build NovaMart from scratch         ████████░░░░░░░░░░░░  40%
    → Lesson 1: Infrastructure Foundation ✅ (Built + Reviewed)
    → Lesson 2: Platform Services ✅ (Built, Review PENDING)
    → Lesson 3: CI/CD Pipeline (REMAINING)
    → Lesson 4: Application Onboarding (REMAINING)
    → Lesson 5: Operations Readiness (REMAINING)

PART 3: OPERATE
  Phase 8: NovaMart Production Simulation      ░░░░░░░░░░░░░░░░░░░░   0%

PART 4: INTERVIEW
  Phase 9: FAANG Interview Prep                ░░░░░░░░░░░░░░░░░░░░   0%

Overall: ~60%
```

---

# 📚 DETAILED TOPIC COVERAGE — EVERYTHING TAUGHT SO FAR

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
CNI plugins (AWS VPC CNI — real VPC IPs, Calico — overlay/BGP + NetworkPolicy,
Cilium — eBPF + L7 policies, Flannel — simple overlay, Weave — encrypted),
Pod-to-pod same node (veth pairs → Linux bridge),
Pod-to-pod different nodes (VPC CNI direct routing, VXLAN overlay, BGP),
Service internals (iptables NAT rules with probability chains),
kube-proxy iptables mode (O(n)) vs IPVS mode (O(1) hash lookup),
All Service types (ClusterIP, NodePort, LoadBalancer, ExternalName, Headless),
externalTrafficPolicy (Cluster vs Local — source IP preservation tradeoff),
Ingress resources and controllers (Nginx, AWS LB Controller, Traefik, Istio),
NetworkPolicies (ingress, egress, DNS egress gotcha, label mismatch),
VPC Flow Logs (format, debugging, Athena queries)
Failure modes: VPC CNI IP exhaustion (prefix delegation fix),
conntrack table full (nf_conntrack_max or Cilium),
IPVS session persistence causing latency,
NetworkPolicy label mismatch = silent traffic deny,
LB subnet tags missing (kubernetes.io/role/elb),
externalTrafficPolicy:Local + uneven pod distribution
```

---

## PHASE 2: Git, Docker, Kubernetes (100% ✅)

### Lesson 1 + 1B: Git Internals & Commands
```
Topics: Git object model (blob, tree, commit, tag), content-addressable storage,
SHA-1 hashing, cat-file, deduplication, packfiles (delta compression),
refs (branches as files), HEAD (ref or detached), annotated vs lightweight tags,
three areas (working directory, staging/index, repository),
stash internals (merge commit structure),
fast-forward merge vs three-way merge vs rebase (new commits, new hashes),
interactive rebase (pick, reword, edit, squash, fixup, drop),
branching strategies (trunk-based development, GitHub Flow, GitFlow — avoid),
CI optimizations (shallow clone --depth 1, sparse checkout, path-based triggers),
monorepo strategies (dependency graph, Bazel/Nx/Turborepo),
git bisect (manual + automated with run),
branch protection rules (no force push, require PR),
fetch vs pull (fetch = safe download, pull = fetch + merge/rebase),
pull.rebase true configuration,
reset (--soft/--mixed/--hard) with visual comparison,
revert vs reset (pushed = revert, local = reset),
revert a merge commit (-m 1, revert-the-revert trap),
cherry-pick (single, range, --no-commit, conflict resolution),
conflict resolution workflow (ours/theirs SWAP during rebase),
git log (--oneline, --graph, --author, --since, -S pickaxe, -G regex, ranges),
git blame (-L range, -w), git show, git diff (--staged, --stat, --name-only),
restore vs switch (Git 2.23+ modern replacements for checkout),
git clean (-n, -f, -fd, -fdx),
Git LFS (pointers, GIT_LFS_SKIP_SMUDGE for CI),
.gitattributes (line endings, merge drivers, binary markers),
git hooks (client: pre-commit, commit-msg, pre-push; server: pre-receive, post-receive),
pre-commit framework (Python), Husky (Node.js),
submodules vs subtrees (tradeoffs, when to use each),
git worktrees (parallel branch checkouts)
Failure modes: force push to main (recovery via reflog),
secrets committed to Git (rotate FIRST, git-filter-repo, force push all),
CI testing branch not merge result, .gitignore missing secrets/state files
```

### Lesson 2 + 2B: Docker Internals & Production
```
Topics: Containers = process + namespaces + cgroups + union FS (NOT VMs),
Namespaces (PID, NET, MNT, UTS, IPC, USER — what each isolates),
Cgroups (CPU: --cpus, --cpu-shares; Memory: --memory, --memory-swap, OOM;
  I/O: --device-write-bps; PIDs: --pids-limit),
JVM + cgroup OOM issue (-XX:MaxRAMPercentage=75.0),
Container networking (bridge default, host, none, custom bridge with DNS, macvlan),
Docker DNS only on custom bridge networks,
Union filesystem / layers / copy-on-write,
Dockerfile best practices (pin versions, multi-stage, layer ordering,
  COPY deps before code, USER nonroot, --no-install-recommends, .dockerignore),
Multi-stage builds (builder pattern, distroless, scratch),
Image tagging strategy (never :latest, immutable tags, digest pinning),
Docker Compose (healthcheck, depends_on conditions, named volumes, profiles),
Docker security (non-root, --read-only + tmpfs, --cap-drop=ALL, 
  --no-new-privileges, Trivy/Grype scanning, Docker socket danger),
Container runtimes (Docker Engine → containerd → runc, CRI-O, Podman),
Dockershim removal from K8s 1.24,
ENTRYPOINT vs CMD (exec form vs shell form),
PID 1 zombie reaping problem (tini, dumb-init, --init),
Signal handling (SIGTERM forwarding, exec in shell scripts, STOPSIGNAL),
Volume types (named volumes, bind mounts, tmpfs — when to use each),
Volume drivers (NFS, etc.), read-only volumes,
Docker disk management (system df, prune, builder prune),
Log rotation (max-size, max-file in daemon.json — CRITICAL),
Logging drivers (json-file, journald, syslog, fluentd, awslogs),
K8s logging: DON'T change Docker log driver (kubectl logs depends on it),
Restart policies and exponential backoff,
Multi-architecture builds (buildx, --platform, ARM/Graviton cost savings),
Image signing (Cosign/Sigstore, keyless signing, Kyverno verification policy, DCT),
DinD vs DooD vs Kaniko vs Buildah (security tradeoffs),
docker inspect (state, OOMKilled, network, env), diff, cp, export, events,
Compose production patterns (override files, profiles, depends_on conditions),
/proc visibility problem (meminfo/cpuinfo show HOST, not container),
GOMAXPROCS (Go), UseContainerSupport (Java), UV_THREADPOOL_SIZE (Node),
Advanced networking (multi-network isolation, aliases, container network mode),
Docker vs K8s health checks (Docker HEALTHCHECK ignored in K8s),
BuildKit (cache mounts, secret mounts, SSH mounts, registry cache),
ARG vs ENV (ARG visible in history — never pass secrets),
Exit code meanings (0, 1, 126, 127, 137, 139, 143),
Troubleshooting playbook (won't start, high disk, networking, image pull, not responding)
Failure modes: OOM killed (JVM heap + non-heap), layer cache busted every build,
image size explosion, works locally crashes in K8s (root vs non-root, read-only FS, 
resource limits, stale cached :latest image, network differences)
```

### Lesson 3 + 3B + 3C: Kubernetes Architecture & Production
```
Topics: Control plane (API server, etcd — Raft consensus, Scheduler — filter→score→bind, 
Controller Manager — 30+ reconciliation loops, Cloud Controller Manager),
Worker node (kubelet, kube-proxy, container runtime),
Pod creation chain (kubectl → API server auth→RBAC→admission→etcd → scheduler → kubelet → kube-proxy),
Pod spec deep dive (init containers, resource requests vs limits,
  liveness/readiness/startup probes, lifecycle hooks preStop/postStart),
Termination sequence (parallel: endpoint removal + preStop, then SIGTERM, 
  then terminationGracePeriodSeconds, then SIGKILL),
preStop sleep trick to prevent 502s during deployment,
QoS classes (Guaranteed, Burstable, BestEffort — eviction order),
Deployments (rolling update: maxSurge + maxUnavailable, rollback, progressDeadlineSeconds),
StatefulSet (stable identity, ordered create/delete, headless service, volumeClaimTemplates),
DaemonSet, Job (backoffLimit, parallelism), CronJob (concurrencyPolicy, startingDeadlineSeconds),
ConfigMap and Secret (env vs volume mount, env NOT updated, volume updated ~60-90s,
  base64 NOT encryption, KMS envelope encryption, External Secrets Operator, Reloader),
Namespaces (RBAC boundary, ResourceQuota, LimitRange),
Scheduling (taints/tolerations, nodeAffinity, podAntiAffinity, topologySpreadConstraints),
RBAC (Role/ClusterRole, RoleBinding/ClusterRoleBinding, kubectl auth can-i),
Storage (PV, PVC, StorageClass, CSI drivers, WaitForFirstConsumer, VolumeSnapshots),
HPA (CPU, memory, custom metrics, external metrics, behavior policies),
VPA (Off/Auto/Initial, DO NOT use HPA CPU + VPA together),
Cluster Autoscaler vs Karpenter (right-sized, 60s provision, consolidation, drift),
PodDisruptionBudgets (minAvailable/maxUnavailable, PDB deadlock),
PriorityClasses (preemption, system-cluster-critical),
Node maintenance (cordon, uncordon, drain), kubelet eviction thresholds,
IRSA (OIDC provider, IAM trust policy, projected token),
Admission controllers (mutating then validating, webhooks, failurePolicy),
Pod Security Standards/PSA (Privileged, Baseline, Restricted),
Helm (chart structure, values, Go templates, hooks, helm diff, ArgoCD integration),
CRDs and Operators (reconciliation pattern, popular operators),
Finalizers (block deletion, stuck resources, removal risks),
Ephemeral containers (kubectl debug), kubectl power tools,
Image pull policies, imagePullSecrets, ECR cross-account,
Container patterns (sidecar, ambassador, adapter, init),
CPU throttling (CFS 100ms windows, nr_throttled, the CPU limits debate),
ContainerCreating/ImagePullBackOff debugging,
CoreDNS scaling (NodeLocal DNSCache, dns-autoscaler, autopath, ndots reduction),
EndpointSlice, probe anti-patterns, OOMKilled vs Evicted vs Preempted vs Deleted,
Secret rotation (dual-write, External Secrets Operator, Reloader),
Resource request tuning methodology (VPA recommend → observe → set → iterate),
etcd failure modes (disk I/O, fragmentation, compaction/defrag),
Runbook structure (severity, symptoms, impact, diagnosis, resolution, escalation)

Production scenarios: Pod stuck Pending, deployment rollout stuck, pod eviction,
502 during deployment, PVC stuck Pending, Helm rollback, namespace stuck Terminating,
HPA not scaling, service not reachable, node NotReady, CrashLoopBackOff
```

### Lesson 4: Kubernetes Operational Patterns
```
Topics: Istio service mesh (control plane: istiod, data plane: Envoy sidecars),
Istio resources (VirtualService, DestinationRule, Gateway, PeerAuthentication, AuthorizationPolicy),
Istio observability (tracing, metrics, access logs, Kiali),
ArgoCD GitOps (Application, syncPolicy, AppProject, App of Apps, ApplicationSet,
  sync waves, sync windows),
CI/CD → ArgoCD flow (Jenkins builds → updates manifests repo → ArgoCD syncs),
Kustomize (base + overlays, patches, configMapGenerator, Kustomize vs Helm),
Troubleshooting workflows: CrashLoopBackOff (7-step), service not reachable (7-step),
  node NotReady (7-step with EKS-specific)
```

---

## PHASE 3: CI/CD Pipelines (100% ✅)

```
Topics covered across all Phase 3 lessons:
Jenkins architecture (controller, agents, distributed builds),
Jenkinsfile (declarative vs scripted, shared libraries),
Pipeline design patterns (golden path templates, matrix builds),
Pipeline optimization (caching, parallelism, artifact reuse),
Pipeline security (credentials, RBAC, script approval),
SonarQube integration (quality gates, branch analysis),
Trivy/BlackDuck (security scanning in pipeline),
Artifactory (artifact management, promotion, cleanup policies),
ArgoCD advanced (deepened from Phase 2),
Argo Rollouts (canary with analysis, blue-green),
Progressive delivery (feature flags, traffic shifting),
Pipeline debugging and failure patterns,
Webhook integrations (Bitbucket → Jenkins → Jira automation),
Monorepo CI strategies (deepened from Phase 2),
Pipeline as Code best practices,
Cost optimization in CI (spot instances, caching, right-sizing)
```

---

## PHASE 4: IaC — Terraform + Ansible (100% ✅)

### Lessons 1-3: Terraform
```
Topics: Terraform architecture (init, plan, apply cycle), HCL syntax,
providers, resources, data sources, variables, outputs, locals,
State management (S3 backend, DynamoDB locking, state structure),
State operations (mv, rm, import, taint, untaint, replace),
State file security (encryption, access control, never in Git),
Modules (structure, inputs/outputs, versioning, registry),
Module composition patterns, NovaMart module library,
Workspaces (CLI workspaces for env separation — vs separate backends debate),
Terraform in CI/CD (plan in PR, apply on merge, OIDC auth, artifact passing),
Atlantis and Terraform Cloud/Enterprise,
Testing (terraform validate, tflint, tfsec/checkov, Terratest, plan-based testing),
State refactoring (moved blocks, import blocks, state surgery),
Terraform Cloud (remote execution, policy as code with Sentinel, cost estimation),
Provider version constraints, dependency lock file,
count vs for_each, dynamic blocks, lifecycle rules (prevent_destroy, ignore_changes),
Terraform graph and dependency resolution
Failure modes: state lock stuck, state corruption, provider auth,
drift detection, dependency cycles, count/for_each index shift,
module version mismatch, partial apply
```

### Lessons 4-5: Ansible
```
Topics: Ansible architecture (control node, managed nodes, agentless SSH),
Ansible vs Terraform (provision vs configure), when to use Ansible in K8s world,
ansible.cfg (forks, pipelining, ControlMaster/ControlPersist, ssh_args),
Pipelining explained (3-5x speed, requiretty prerequisite),
Static inventory (INI and YAML), dynamic inventory (AWS EC2 plugin with full config),
Variable precedence (22 levels simplified to important 9),
group_vars / host_vars directory structure,
Playbook anatomy (plays, pre/post tasks, serial, max_fail_percentage),
Key modules (15+ modules: apt/dnf, file, copy, template, service, systemd,
  command, shell, user, group, uri, wait_for, get_url, unarchive, lineinfile, blockinfile),
Handlers (notify, listen, execution timing, flush_handlers, --force-handlers),
Jinja2 templates (loops, conditionals, facts, validate before deploy),
Conditionals (when), loops (loop, dict2items), register + conditionals,
Error handling (block/rescue/always with Slack alerting and rollback),
Roles (structure, defaults vs vars, tasks, handlers, templates, files, meta),
Full production role example (node_exporter),
Ansible Galaxy (requirements.yml, version pinning),
Ansible Vault (encrypt, decrypt, rekey, encrypt_string, vault-id),
Vault naming convention (vault_ prefix pattern),
Vault password methods (prompt, file, script from Secrets Manager, multi-vault-id),
Vault vs External Secrets (comparison + NovaMart strategy),
Performance optimization (async/poll, strategy:free, Mitogen 2-7x, performance comparison table),
Molecule testing (architecture, Docker + EC2 scenarios, converge/idempotence/verify),
Molecule in CI (Jenkins pipeline), Molecule limitations (Docker ≠ real servers),
Terraform + Ansible integration patterns:
  Pattern 1: Tag → Discover (correct), Pattern 2: Outputs → Vars (bridge script),
  Pattern 3: Provisioner (AVOID — why it's bad), Pattern 4: NovaMart CI pipeline,
Tower/AWX (architecture, RBAC, workflows, credentials, Tower vs Jenkins comparison),
Delegation (delegate_to, run_once), import vs include (static vs dynamic),
Lookup plugins (file, env, SSM, HashiCorp Vault, password), filters, tags
Failure modes: SSH timeout, Python missing, become failure, idempotency broken,
handler not running, variable collision, slow execution, template rendering fails,
partial failure, secret leak, vault password lost, Molecule false positive,
Terraform+Ansible race condition, AWX credential exposure, async orphaned,
delegation context confusion
```

---

## PHASE 5: Observability & SRE (80% — IN PROGRESS)

### Lesson 1: Prometheus + Metrics + PromQL + Alertmanager ✅
```
Topics: Three pillars of observability (metrics, logs, traces) with correlation example,
Monitoring vs Observability distinction (known vs unknown failure modes),
How the three pillars correlate during incident investigation,
Prometheus architecture (pull-based, TSDB with head block + persistent blocks,
  WAL for crash recovery, service discovery, scrape engine, PromQL engine, rule engine),
Pull vs Push model (5 advantages of pull, Pushgateway exception for short-lived jobs),
Four metric types:
  Counter (only increases, use rate()/increase(), handles resets),
  Gauge (up/down, raw value meaningful, deriv() for trends),
  Histogram (bucketed observations, _bucket/_sum/_count, histogram_quantile(),
    bucket design critical for SLOs, SERVER-side quantile calculation, AGGREGATABLE),
  Summary (client-side quantiles, NOT aggregatable — use histograms instead),
Metric naming conventions (<namespace>_<subsystem>_<name>_<unit>, base units),
Labels as dimensional data model,
Cardinality (the Prometheus killer — unique label combinations = time series,
  NEVER use user_id/IP/request_id as labels, monitor with prometheus_tsdb_head_series,
  target < 2M active series per instance),
kube-prometheus-stack (Helm chart: Prometheus Operator + Alertmanager + Grafana +
  node-exporter + kube-state-metrics + default alerting rules),
Full production Helm values (retention, storage, resources, HA replicas, anti-affinity,
  serviceMonitorSelector, externalLabels for multi-cluster),
Three K8s metrics sources distinction:
  node-exporter (node hardware), kube-state-metrics (K8s objects), cAdvisor/kubelet (containers),
Service discovery CRDs:
  ServiceMonitor (scrape pods behind Service), PodMonitor (scrape pods directly),
  PrometheusRule (alerting + recording rules with full YAML examples),
  metricRelabelings (drop high-cardinality metrics),
PromQL fundamentals:
  Instant vector, range vector, scalar,
  Four Golden Signals in PromQL (25+ production queries),
  Kubernetes-specific queries, Node-level queries,
7 PromQL gotchas,
Recording rules (naming convention level:metric:operations),
Alertmanager architecture and configuration:
  Routing tree (first match wins), group_by, group_wait/group_interval/repeat_interval,
  Inhibition rules, Receivers (PagerDuty/Slack/email),
  Full alertmanager.yml with NovaMart routing hierarchy,
  Alert annotations (summary, description, runbook_url, dashboard_url),
Watchdog / dead man's switch pattern,
10 Prometheus failure modes
```

### Lesson 2: Grafana + Loki + Long-term Storage (Thanos) ✅
```
Topics: Grafana architecture and deployment (HA with PostgreSQL, NOT SQLite),
Grafana data source provisioning as code (Prometheus, Loki, Tempo with correlation config),
Dashboard design (Four Golden Signals layout),
Template variables and chaining (cluster → namespace → service),
$__rate_interval (auto-adjusts to scrape interval + dashboard resolution),
Annotations (overlay deploys/incidents on graphs),
Dashboard as Code / GitOps (ConfigMap sidecar, editable: false in prod),
Grafana Unified Alerting (v8+, used for Loki-based log alerts only),
8 Grafana failure modes with fixes,
Loki architecture (Distributor → Ingester → Object Storage, index labels only),
Loki deployment modes (Monolithic, SimpleScalable, Distributed),
Full Loki Helm deployment config (SimpleScalable with S3, limits, retention),
Log collection comparison (Promtail vs Fluent Bit vs OTel Collector),
Promtail configuration (kubernetes_sd, relabel_configs, pipeline_stages),
Label design — THE critical decision (bounded only: app, namespace, level, cluster;
  NEVER: user_id, request_id, trace_id, IP; pod name gotcha),
  Structured metadata for unbounded values (Loki 3.0+),
LogQL log queries (stream selector, line filters |= != |~ !~, parsers: json/logfmt/regexp/pattern,
  label filters on extracted fields, line_format, label_format),
  Pipeline stage ordering (line filter FIRST for performance),
LogQL metric queries (count_over_time, rate, quantile_over_time with unwrap, bytes_rate),
10 production LogQL queries for NovaMart,
10 Loki failure modes (429s, stream limit, query timeout, missing logs, duplicates,
  ingester OOM, chunk flush failures, slow queries, index corruption, clock skew),
Cardinal Loki performance rule: NEVER use {} with no stream selector,
Metrics ↔ Logs ↔ Traces correlation (shared labels, derivedFields, tracesToMetrics),
Long-term storage problem (Prometheus single-node, limited retention, no global query),
Thanos vs Mimir vs Cortex comparison table,
Thanos architecture deep dive:
  Sidecar (uploads blocks + serves StoreAPI), Store Gateway (serves from S3),
  Query (fan-out + dedup), Query Frontend (split + cache + retry),
  Compactor (SINGLETON — never >1 replica, compaction + downsampling + retention),
Full Thanos component configs (sidecar in Prometheus Operator, Store GW, Query, Compactor, Query Frontend),
Object storage config (S3, IRSA, SSE, lifecycle rules),
S3 cost optimization (Standard-IA after 30d, NEVER Glacier for queried data),
Thanos downsampling (raw=30d, 5m=90d, 1h=365d),
disableCompaction: true on Prometheus when using Thanos,
external_labels (cluster + replica) for global dedup,
10 Thanos failure modes,
NovaMart full end-to-end observability data flow diagram
```

### Lesson 3: Distributed Tracing + OpenTelemetry ✅
```
Topics: Why traces exist (what metrics and logs can't tell you),
Trace anatomy (trace, span, root span, child span, span context, baggage),
Context propagation (THE most critical concept, #1 tracing failure mode),
W3C TraceContext standard (traceparent header format),
Propagation formats (W3C, B3/Zipkin, B3 Single, Jaeger uber-trace-id, AWS X-Ray),
Propagation break problem (format mismatch = disconnected traces),
Multi-format propagation during migration,
OpenTelemetry architecture (SDK, Collector, auto-instrumentation, OTLP),
OTel Collector pipeline (Receivers → Processors → Exporters),
Full production Collector config (otlp, jaeger receivers; batch, memory_limiter,
  k8sattributes, resource, filter, tail_sampling processors;
  otlp/tempo, prometheusremotewrite, loki exporters),
Collector deployment patterns:
  DaemonSet (Agent), Sidecar, Agent+Gateway (NovaMart's choice),
  Why Gateway needed for tail sampling (all spans in one place),
  loadbalancing exporter with routing_key: traceID,
OTel Collector K8s deployment (OpenTelemetryCollector CRD via Operator),
Auto-instrumentation (OTel Operator + Instrumentation CRD + pod annotations,
  Java/Python/Node.js/Go, what auto-inst captures vs what needs manual),
Manual instrumentation (Go example with spans, events, error handling,
  ALWAYS pass ctx, never context.Background() mid-chain),
Sampling math (1TB/day raw, 10% sampling = 100GB/day),
Head sampling vs Tail sampling:
  Head: decision at start, simple, misses errors,
  Tail: decision after complete, intelligent, needs memory + gateway,
NovaMart strategy: 100% head → tail at gateway,
Tempo architecture (object storage only, minimal index, TraceQL),
Tempo vs Jaeger comparison (cost, operations, query model, integration),
Tempo deployment config (SimpleScalable, S3 storage, metrics generator),
Tempo Metrics Generator (creates RED metrics FROM traces → Prometheus,
  service graph metrics, span metrics),
TraceQL query language (service filter, status, duration, attributes, aggregation),
Complete correlation model:
  Exemplars (metrics → traces: histogram sample carries traceID),
  Log → Trace (derivedFields regex on traceID in Loki datasource),
  Trace → Logs (tracesToLogs config, filter by traceID + time),
  Trace → Metrics (tracesToMetrics config, jump to service dashboard),
Complete incident investigation flow (4-step: metrics → exemplar → trace → logs → fix),
10 tracing failure modes (broken traces, missing spans, duplicates, span explosion,
  traces not in Tempo, latency from tracing, traceID not in logs,
  sampling confusion, Collector OOM, clock skew)
```

### Lesson 4: SLOs, SLIs, Error Budgets, Burn Rate Alerting ✅ (Retention Qs PENDING ANSWERS)
```
Topics: Why SLOs exist (solving "everything is emergency" and "ship and pray"),
SLI/SLO/SLA definitions,
Error budget = 1 - SLO target,
Error budget math (time and request-based),
SLI types menu (availability, latency, freshness, correctness, throughput, durability),
Good vs Bad SLIs (user-facing vs internal metrics),
NovaMart SLO definitions per service,
SLI implementation in Prometheus recording rules,
Latency SLI using histogram bucket ratio (NOT histogram_quantile),
Error budget remaining calculation,
Error budget policies (velocity tiers, freeze triggers, exceptions),
Burn rate concept and math,
Multi-window multi-burn-rate alerting (Google SRE book),
Full burn rate alert implementation in PrometheusRule,
Grafana SLO dashboard design,
6 SLO anti-patterns,
Sloth (SLO spec → Prometheus rules generator)
```

---

## PHASE 7: Build NovaMart (40% — IN PROGRESS)

### Lesson 1: Infrastructure Foundation ✅ (Built + Reviewed, Grade: A-)

```
BUILT:
  Architecture: VPC (3-AZ, dual CIDR 10.0.0.0/16 + 100.64.0.0/16),
    12 subnets (public/private/pod/data), 3 NAT Gateways, 7 VPC endpoints,
    VPC Flow Logs (CloudWatch + S3)
  EKS: Production cluster, managed system node group (tainted CriticalAddonsOnly),
    Karpenter (IAM + SQS + EventBridge for spot interruption),
    4 add-ons (VPC CNI with custom networking, CoreDNS ×3, kube-proxy, EBS CSI),
    IRSA (OIDC + roles for VPC CNI, EBS CSI, Karpenter, LB Controller, External Secrets),
    Launch template (IMDSv2, encrypted EBS, detailed monitoring)
  Data Tier: 3× RDS PostgreSQL Multi-AZ (payments/orders/users),
    ElastiCache Redis (3-node replication group), managed master passwords,
    Enhanced Monitoring, Performance Insights, parameter groups with SSL forced
  Security Baseline: 4 KMS keys (EKS, RDS, ElastiCache, General),
    4 S3 buckets (state, CloudTrail, flow logs, compliance),
    CloudTrail (multi-region, insights, data events),
    Object Lock on CloudTrail/compliance buckets,
    Account-level EBS encryption default
  Terraform: Module-based structure, remote S3+DynamoDB state,
    split state per domain (security/networking/eks/data),
    bootstrap module, environment compositions with remote_state references
  Documentation: 9 design decisions (DD-001 through DD-009),
    CIDR allocation map, apply order runbook, cost estimate, rollback procedures

REVIEW FINDINGS (5 Critical, 7 Significant, 7 Minor):
  CRITICAL:
    1. Circular dependency: security ↔ networking for flow logs (missing count conditional)
    2. Duplicate modules: modules/rds + modules/elasticache vs modules/data-tier (dead code)
    3. Orphaned security-groups module (conflicts with EKS module's own SGs)
    4. S3 Object Lock requires object_lock_enabled=true at bucket creation
    5. Karpenter EC2NodeClass uses instance profile name not role name
  SIGNIFICANT:
    1. No .terraform.lock.hcl strategy
    2. No .gitignore file
    3. Backend config hardcoded in every backend.tf (should use partial config)
    4. No CIDR count validation on VPC variables
    5. EKS add-on versions will drift (no update check automation)
    6. No SNS topic for alarm actions (alarms fire into void)
    7. Redis AUTH token not configured (TLS but no authentication)
  PRAISED:
    - Dual CIDR strategy (textbook correct, RFC 6598 choice)
    - Design decisions document (professional-grade ADRs)
    - Default SG lockdown (90% of engineers miss this)
    - manage_master_user_password (no password in state)
    - S3 Object Lock on CloudTrail (PCI/SOC2 compliant)
    - Flow logs dual destination (CloudWatch hot + S3 cold with Parquet)
    - System node group + Karpenter pattern
    - CloudTrail Insights
    - EBS encryption by default
```

### Lesson 2: Platform Services ✅ (Built, Review PENDING)

```
BUILT:
  Service Mesh: Linkerd 2.x (chosen over Istio — DD-PS-001 justification:
    10MB vs 70MB sidecar memory, 1ms vs 3-5ms p99 latency, simpler ops),
    Trust anchor PKI (10yr root CA, 1yr issuer), CRDs + control plane + Viz extension,
    mTLS default policy "all-authenticated", HA (3 replicas, PDB, anti-affinity)

  Observability — Metrics: kube-prometheus-stack (Prometheus ×2 HA,
    Alertmanager ×3, Grafana ×2, kube-state-metrics ×2, node-exporter DaemonSet),
    Thanos sidecar → S3 long-term storage (compactor, store gateway, query),
    15-day local retention, 2-year S3 retention with downsampling,
    IRSA for Thanos S3 access, gp3-encrypted StorageClass

  Observability — Logging: Fluent Bit DaemonSet (chosen over Fluentd — DD-PS-003),
    Namespace-based routing (payments → separate CW log group for PCI),
    Dual output: CloudWatch (hot, 30-day) + S3 (cold, 1-year, gzip),
    Kubernetes metadata enrichment, trace_id extraction from log body,
    IRSA for CloudWatch Logs + S3, pre-created log groups with per-NS retention

  Observability — Tracing: Grafana Tempo distributed mode (S3 backend),
    OTel Collector Agent (DaemonSet) + Gateway (Deployment ×3),
    Tail-based sampling at gateway (100% errors, 10% success, 1% health checks),
    Dual export: Tempo (primary) + AWS X-Ray (secondary),
    Trace-to-log and trace-to-metric correlation in Grafana datasources

  Alerting: Alertmanager with PagerDuty (critical) + Slack (warning/info) routing,
    Inhibition rules (node-down suppresses pod alerts, API-down suppresses all),
    4 alert rule sets: cluster, node, pod, platform self-monitoring,
    Service-level alerts from Linkerd RED metrics (error rate, latency, traffic drop),
    Platform self-monitoring (Prometheus, Thanos, Fluent Bit, Kyverno, ArgoCD, ESO, Linkerd)

  Security — Secrets: External Secrets Operator with ClusterSecretStore
    (Secrets Manager + Parameter Store), namespace-scoped access conditions,
    Grafana admin credentials managed via ESO example

  Security — Policies: Kyverno (chosen over OPA/Gatekeeper — DD-PS-002),
    8 policies: disallow-privileged, disallow-latest-tag, require-resource-limits,
    require-labels, require-run-as-non-root, disallow-host-path,
    restrict-image-registries, mutation (default security context),
    Generation policy (auto-create default-deny NetworkPolicy on new NS),
    Webhook failurePolicy=Fail (blocks all deploys if Kyverno is down)

  Security — Network Policies: Default-deny via Kyverno generation,
    DNS egress (all app NS), Prometheus scrape ingress (all app NS),
    Linkerd control plane + mesh communication,
    Per-service policies: payments (DB + Redis + payment gateway + OTel only),
    orders (DB + Redis + payment-svc + inventory-svc + OTel),
    users (DB + Redis + OTel + SMTP)

  Security — ECR: Immutable tags, KMS encryption, enhanced scanning (continuous),
    Lifecycle policies (expire untagged after 1 day, keep last 30 tagged),
    Repository policies (org-scoped pull access)

  Ingress: cert-manager (Let's Encrypt production + staging ClusterIssuers),
    AWS Load Balancer Controller (IRSA, WAFv2 enabled, default tags on ALBs)

  GitOps: ArgoCD HA (server ×3, repo-server ×3, controller ×2, Redis-HA ×3),
    RBAC (platform-admin, app-developer, app-viewer roles),
    Two AppProjects: platform (auto-sync, sync windows for business hours),
    applications (manual sync in prod, restricted resource whitelist),
    App-of-Apps root application, IRSA for ECR access

  Grafana Dashboards: ConfigMap-based (sidecar-loaded),
    cluster-overview, node-health, namespace-resources, linkerd-service,
    platform-health (self-monitoring)

  Namespace Strategy: 15 namespaces (9 platform + 6 application),
    Pod Security Standards enforced (restricted for app, baseline for platform),
    PCI isolation for novamart-payments (separate logs, strict NetworkPolicy, IRSA)

  Design Decisions: DD-PS-001 through DD-PS-009
    (Linkerd vs Istio, Kyverno vs OPA, Fluent Bit vs Fluentd,
    Tempo vs Jaeger, Terraform vs ArgoCD boundary, namespace strategy,
    Prometheus retention strategy, alert routing strategy, resource budget)

  Resource Budget: ~18 vCPU + 32Gi fixed, ~500m/node + 512Mi/node per-node,
    ~100m/pod + 70Mi/pod mesh overhead, ~12% of 2000-vCPU cluster at scale

  Operational: Platform apply order runbook, certificate rotation schedule,
    monthly maintenance checklist, post-apply validation commands
```

---

## RETENTION SCORE HISTORY

```
Phase 0 Lessons: Consistently 4/5 to 5/5
Phase 1 Lessons: Consistently 3.5/4 to 4/4
Phase 2 Lesson 1 (Git): 3.5/4
Phase 2 Lesson 1B (Git Commands): 4/4
Phase 2 Lesson 2 (Docker): 4/4
Phase 2 Lesson 3+3B+3C (K8s): 5/5
Phase 2 Lesson 4 (K8s Ops): 4/4 (Q6-Q9 only, Q1-Q4 skipped)
Phase 3: Scores not individually tracked (completed as block)
Phase 4 Lesson 4 (Ansible Basics): 4.6/5 (Q1: 4.5, Q2: 4.5, Q3: 4.5, Q4: 5.0)
Phase 4 Lesson 5 (Ansible Advanced): 4.6/5 (Q1: 5.0, Q2: 4.5, Q3: 4.5, Q4: 4.5)
Phase 5 Lesson 1 (Prometheus): 4.8/5 (Q1: 4.8, Q2: 5.0, Q3: 5.0, Q4: 4.5)
Phase 5 Lesson 2 (Grafana/Loki/Thanos): 4.8/5 (Q1: 4.8, Q2: 4.9, Q3: 4.9, Q4: 4.6)
Phase 5 Lesson 3 (Tracing/OTel): 4.8/5 (Q1: 4.9, Q2: 4.8, Q3: 4.9, Q4: 4.7)
Phase 5 Lesson 4 (SLOs/Error Budgets): PENDING — retention questions given, answers not yet received
Phase 7 Lesson 1 (Infra Foundation): GRADE A- (built + reviewed, no retention Qs — design/build phase)
Phase 7 Lesson 2 (Platform Services): BUILT, REVIEW PENDING
```

### User's Strengths:
- Strong diagnostic reasoning (challenges premises, follows evidence)
- Understands "why" not just "what"
- SRE mindset: inventory before cleanup, prevent recurrence, document
- Connects concepts across domains (Linux → containers → K8s)
- Incident response instincts are sharp — sequences actions correctly under time pressure
- Closes every loop (rotation, scrub, audit, prevent)
- Production-oriented answers — explains "why it matters in production" not just "what"
- Three consecutive 4.8/5 scores on Phase 5 — gaps consistently at "mechanism precision" level, not conceptual
- Architectural understanding and operational instincts are senior-level
- Phase 7 builds demonstrate strong Terraform module design, security-first thinking, and cost awareness
- Design decisions documents are professional-grade ADRs with alternatives and cost impact
- Dual CIDR VPC strategy, Karpenter architecture, and Linkerd choice show depth of understanding

### User's Gaps to Watch:
- Occasionally misses the "production gotcha" in favor of the textbook answer
- Sometimes misses idempotency edge cases in own code
- Occasionally misses "nice to have" operational touches (Slack notifications before fleet-wide changes, caching for API calls, approval gates for production changes)
- Q3 Phase 1 Lesson 10: missed IPVS session persistence
- Needs to consistently close answers with prevention (branch protection, monitoring, alerts)
- DR state backend location (store state in primary region, not DR region) — organizational pattern gap
- Tool-specific precision gaps (tcpdump vs gRPC binary, memory_limiter headroom ratios)
- Uses working path vs optimal path (e.g., logs-first correlation instead of exemplar shortcut)
- Promtail match stage selector syntax (only accepts stream selectors, not full LogQL pipelines)
- rate() internal mechanism precision (how step vs range interact — conclusion correct but mechanism slightly off)
- Occasionally assumes app-specific metrics exist without flagging availability uncertainty
- metricRelabeling keep action logic (accidentally drops all other metrics from target)
- **Phase 7 Build Gaps (from Lesson 1 review):**
  - Module duplication (3 SG definitions, 2 data tier implementations)
  - Several resources would fail first apply (Object Lock, flow log conditional, launch template SG conflict)
  - Monitoring defined but not connected (alarms → no SNS topic)
  - Some manual steps in runbook should be automated (ENIConfigs, Karpenter NodePool)

---

## WHERE TO RESUME

**Current position:** Phase 7, Lesson 2 (Platform Services) was fully built. Review has NOT yet been performed.

**Action required:**
1. **Review Phase 7 Lesson 2** — Same brutal review format as Lesson 1 (Critical/Significant/Minor issues, applause, completeness scorecard, priority action items)
2. After review, proceed to **Phase 7 Lesson 3: CI/CD Pipeline**

**Outstanding items from earlier phases (can be addressed later or skipped):**
- Phase 5 Lesson 4 retention questions still unanswered
- Phase 5 Lesson 5 (Incident Management) not delivered
- Phase 6 (Security, Compliance, AWS) not started
- User jumped ahead to Phase 7 Build — this is acceptable as the build phase integrates Phase 5-6 concepts practically

**The 4 retention questions for Phase 5 Lesson 4 (still pending, reproduce if user wants to answer):**

### Q1: SLO Design Under Constraints 🔥
VP of Product demands 99.999% availability for payment service. Current: 99.97% over 90 days.
1. Calculate error budget for 99.999% over 30-day window (minutes of downtime)
2. Explain to VP (non-technical) why 99.999% is wrong — three specific technical constraints
3. What SLO would you recommend? Justify with current data and NovaMart architecture
4. Write exact Prometheus recording rules for availability SLI and latency SLI (99.9% < 2s)

### Q2: Burn Rate Alert Investigation 🔥
Monday 9 AM ticket: OrderServiceSlowBurnRate_Ticket — 1x burn rate for 3 days, error budget at 38%.
1. Is this actionable? Justify
2. PromQL queries to find WHERE budget is consumed (endpoints, error codes, time periods)
3. Error rate 0.12% vs 0.1% budget, all 503s from inventory-svc, bursts every 4 hours — root cause?
4. Immediate and permanent fix

### Q3: SLI Measurement Trap 🔥
search-svc reports 99.95% availability (above 99.9% target), but users complain search returns empty results. 200 OK with empty results.
1. Why is the SLI lying? What principle is violated?
2. Design corrected SLIs (at least 3 dimensions) with Prometheus implementation
3. Search team says "can't add correctness SLI" — two practical approaches
4. PromQL for proportion of searches returning ≥1 result, excluding non-existent products

### Q4: Error Budget Policy Enforcement 🔥
Day 18 of 30-day window, payment-svc budget: 3.8 of 4.32 min consumed (12% remaining). Critical feature launch tomorrow, CEO committed to partner.
1. What should happen per error budget policy?
2. Three counter-arguments to "it's been tested in staging, low risk"
3. Compromise proposal respecting both policy and business commitment
4. VP overrides policy and orders deploy — what do you do?

**Phase 7 remaining lessons:**
```
Lesson 3: CI/CD Pipeline
          End-to-end from commit to production
          (The deployment machinery)

Lesson 4: Application Onboarding
          Deploy payment-svc with full security + observability
          (Prove the platform works)

Lesson 5: Operations Readiness
          SLOs, alerting, runbooks, DR, documentation
          (Prove you can operate what you built)
```

Then continue: Phase 8 (Operate NovaMart) → Phase 9 (Interview Prep).

---

**END OF HANDOFF DOCUMENT**
