---

# ROUND 0: PHONE SCREEN — BREADTH CHECK

**Interviewer:** Senior SRE (Screening)
**Duration:** 30 minutes
**Format:** 20 questions. Short, precise answers. No essays.

---

*Ring ring.*

Hey, thanks for jumping on. I'm Alex, Senior SRE here. I've got your resume — looks solid. I've got about 30 minutes and a bunch of questions across the stack. I want to see how broad your knowledge is before we bring you on-site. Keep your answers tight — a few sentences each unless I ask you to go deeper. Ready?

Let's go.

---

**Q1.** A process is in `D` state and you can't kill it. What's happening and what are your options?

**Q2.** What's the difference between `net.core.somaxconn` and `net.ipv4.tcp_max_syn_backlog`?

**Q3.** You see 40,000 connections in `TIME_WAIT` on a server. Is this a problem? What do you do?

**Q4.** Explain the `ndots:5` problem in Kubernetes in two sentences.

**Q5.** What's the difference between a `ServiceMonitor` and a `PodMonitor` in Prometheus Operator?

**Q6.** Your Prometheus instance has 3 million active time series and queries are getting slow. What's your first move?

**Q7.** What is the difference between `histogram_quantile()` and a Summary's quantile? When does the distinction matter?

**Q8.** Explain tail sampling vs head sampling in one sentence each.

**Q9.** You have an SLO of 99.9% availability. How much error budget is that per 30-day window, in minutes?

**Q10.** What's the difference between `terraform taint` and `terraform apply -replace`?

**Q11.** Terraform apply failed halfway through. 3 out of 7 resources were created. What's the state of things and what do you do?

**Q12.** What's the difference between Ansible `import_tasks` and `include_tasks`? When does it matter?

**Q13.** A pod is in `CrashLoopBackOff`. Give me your first 4 commands, in order.

**Q14.** What's the difference between `externalTrafficPolicy: Cluster` and `externalTrafficPolicy: Local`?

**Q15.** Why should you never use `:latest` as an image tag in production?

**Q16.** What problem does a PodDisruptionBudget solve, and how can it become a problem itself?

**Q17.** Explain the difference between `RollingUpdate` strategy in a Deployment vs a StatefulSet.

**Q18.** What is IRSA and why does it exist? One sentence on the mechanism.

**Q19.** You `curl` an internal service and get a `503`. You `curl` it again and get `200`. What are the top 3 causes?

**Q20.** What's the single most important thing to get right when designing Loki labels, and why?



**Q1.** `D` state means uninterruptible sleep — the process is waiting on a kernel-level I/O operation (usually disk, sometimes NFS/network storage). You can't kill it because it won't process signals until the I/O call returns. Options: identify what it's waiting on (`cat /proc/<pid>/wchan`, `echo w > /proc/sysrq-trigger`), fix the underlying I/O issue (hung NFS mount, dying disk, stuck SCSI command). If nothing works, you're looking at a reboot. `kill -9` won't help — that's a common misconception.

**Q2.** `somaxconn` is the max size of the **accept queue** — fully established connections waiting for the application to call `accept()`. `tcp_max_syn_backlog` is the max size of the **SYN queue** — half-open connections that completed the first SYN but haven't finished the three-way handshake. Both can cause connection drops if undersized but at different stages — SYN backlog drops happen before the connection is established, somaxconn drops happen after. Under high connection rates you typically need to tune both.

**Q3.** It depends. 40K `TIME_WAIT` on a high-throughput proxy or load balancer can be totally normal — `TIME_WAIT` exists to prevent stale segments from corrupting new connections on the same tuple. It becomes a problem if you're exhausting ephemeral ports (check `net.ipv4.ip_local_port_range`). Fixes in order of preference: enable `net.ipv4.tcp_tw_reuse` (safe, lets kernel reuse TIME_WAIT sockets for outbound connections with timestamp protection), widen the ephemeral port range, use connection pooling/keepalives to reduce churn in the first place. Avoid `tcp_tw_recycle` — it's been removed from modern kernels because it breaks behind NAT.

**Q4.** With `ndots:5` (the Kubernetes default), any DNS query with fewer than 5 dots gets the search domains appended first — so `api.stripe.com` triggers 4-5 unnecessary cluster DNS lookups before the external query. Fix it per-pod with `dnsConfig.options ndots:2` or use FQDNs with a trailing dot.

**Q5.** `ServiceMonitor` selects targets based on Kubernetes `Service` objects — it discovers endpoints behind a Service. `PodMonitor` selects pods directly by label, bypassing Services entirely. Use `PodMonitor` when you have pods that don't need a Service (batch jobs, sidecars exposing metrics) or when the Service's label selectors don't align with what you want to scrape.

**Q6.** First: identify the cardinality culprits. Run `topk(10, count by (__name__)({__name__=~".+"}))` or use Prometheus's built-in `/tsdb-status` endpoint to find the highest cardinality metrics. Usually it's one or two metrics with an unbounded label (like `user_id`, `request_path`, or a poorly instrumented histogram). Fix the instrumentation at the source — relabeling to drop it at scrape time is the fast mitigation, but the real fix is fixing the metric definition. Don't throw hardware at a cardinality problem.

**Q7.** `histogram_quantile()` computes quantiles server-side from bucket counters — it's an approximation whose accuracy depends on bucket boundaries, but histograms are aggregatable across instances. Summary quantiles are computed client-side (in the application), so they're more accurate for a single instance but **cannot be aggregated** across instances. The distinction matters when you have multiple replicas — if you average p99s from 5 pod summaries, you get a statistically meaningless number. Use histograms when you need to aggregate across instances, which is almost always.

**Q8.** Head sampling: decide whether to keep or drop a trace at the start, before any spans are collected — fast and simple but you might drop interesting traces you didn't know were interesting yet. Tail sampling: collect all spans first, then decide after the trace completes whether to keep it — you can make smarter decisions (keep all error traces, keep slow traces) but it requires buffering complete traces, which is architecturally more complex and resource-intensive.

**Q9.** 30 days = 43,200 minutes. 0.1% error budget = **43.2 minutes** of allowed downtime per 30-day window. That's roughly 1.4 minutes per day. Tight enough that a single bad deploy with a slow rollback can eat half your budget.

**Q10.** `terraform taint` (deprecated) marks a resource as tainted in state, forcing destroy+recreate on next apply — it modifies state as a separate operation. `terraform apply -replace=<resource>` does the same thing but as part of the plan/apply cycle without modifying state separately. `-replace` is preferred because the intent is visible in the plan, it's a single operation, and it doesn't leave dirty state if you change your mind before applying.

**Q11.** Terraform state now reflects the 3 created resources. The 4 that failed don't exist in state or in reality (or partially exist if the failure was mid-resource-creation, which is rarer). First: `terraform plan` — Terraform will show you it needs to create the remaining 4. Review the plan, fix whatever caused the failure (permissions, quota, dependency issue), and re-run `terraform apply`. The 3 already created resources won't be touched. Don't manually delete things unless state is genuinely corrupted — Terraform's idempotency handles this. If something did get created but isn't in state (rare, usually provider bug), `terraform import` it.

**Q12.** `import_tasks` is **static** — tasks are parsed and included at playbook parse time. All tags, conditionals, and variables are resolved at parse time. `include_tasks` is **dynamic** — tasks are parsed and included at execution time when the task is reached. It matters when you're using loops or conditionally including task files — `include_tasks` can be looped over and conditionally included at runtime, `import_tasks` can't loop but gives you better `--list-tasks` visibility and tag inheritance. Rule of thumb: `import` for structural composition, `include` for dynamic/conditional behavior.

**Q13.**
1. `kubectl describe pod <pod> -n <ns>` — check Events section for OOM, image pull errors, probe failures, scheduling issues
2. `kubectl logs <pod> -n <ns> --previous` — see what the container printed before it crashed
3. `kubectl get pod <pod> -n <ns> -o yaml | grep -A5 lastState` — check exit code (137 = OOMKilled, 1 = app error, 127 = binary not found)
4. `kubectl get events -n <ns> --sort-by=.lastTimestamp` — broader namespace events for context (node pressure, quota limits)

**Q14.** `externalTrafficPolicy: Cluster` (default) — traffic from a NodePort/LB can land on any node, kube-proxy SNATs and forwards to any pod on any node. You lose the client's source IP but get even load distribution. `externalTrafficPolicy: Local` — traffic only goes to pods on the node it arrived at. Preserves source IP, avoids the extra hop, but if no pod is running on that node, the health check fails and the LB stops sending traffic there. Risk: uneven distribution if pods aren't spread evenly. Pair it with topology spread constraints.

**Q15.** Three reasons: `:latest` is mutable, so the same tag can point to different images at different times — you lose reproducibility and auditability of what's actually running. Second, Kubernetes uses `imagePullPolicy: Always` for `:latest` by default, meaning every pod restart hits the registry. Third, rollbacks become meaningless — you can't "roll back" to `:latest` because it's the same tag. Use immutable tags tied to git SHAs or semantic versions.

**Q16.** PDB defines the minimum number of pods that must remain available (or maximum unavailable) during voluntary disruptions — node drains, cluster upgrades, spot evictions. Without it, a node drain could kill all replicas of a service simultaneously. **How it becomes a problem:** if you set `minAvailable` too high (like equal to replica count) or your pods are already unhealthy and below the PDB threshold, it blocks node drains entirely. I've seen cluster upgrades stall for hours because a PDB was protecting a Deployment whose pods were already crashlooping — the drain can't proceed, the node can't be recycled, nothing moves.

**Q17.** Deployment `RollingUpdate` creates new pods (from a new ReplicaSet) and terminates old ones based on `maxSurge` and `maxUnavailable` — pods are interchangeable, no ordering guarantees, any pod can come up or go down in any order. StatefulSet `RollingUpdate` updates pods **in reverse ordinal order** (highest to lowest), one at a time, waiting for each pod to be Running and Ready before moving to the next. This is because StatefulSets have identity (stable network names, persistent volumes) — order matters for things like databases where you want to update replicas before the primary.

**Q18.** IRSA (IAM Roles for Service Accounts) lets Kubernetes pods assume AWS IAM roles without using node-level instance profiles — it works by projecting a signed OIDC token into the pod which AWS STS exchanges for temporary IAM credentials. It exists because without it, every pod on a node shares the node's IAM role, which violates least-privilege — one compromised pod gets the permissions of every workload on that node.

**Q19.** Top 3 causes for intermittent 503s:
1. **Pod not ready / rolling deploy in progress** — you hit the request during a window where the backend pod was terminating or hadn't passed readiness checks yet. The service sent traffic to a pod that wasn't ready to serve.
2. **Load balancer / ingress retry hitting a different backend** — one backend pod is unhealthy or overloaded, first request hit that pod (503), second request got routed to a healthy one (200). Check if you have outlier detection / circuit breaking configured.
3. **Resource pressure / rate limiting** — the target pod was momentarily CPU-throttled or hitting a connection/memory limit, returned 503, then recovered. Check resource limits and throttling metrics.

**Q20.** **Keep label cardinality low.** Loki indexes on labels, not on log content — every unique label combination creates a new stream. If you put high-cardinality values as labels (user IDs, request IDs, pod UIDs), you explode the number of streams, which kills ingestion performance and makes the index unmanageable. Use labels for things like `namespace`, `app`, `environment`, `level` — things with bounded, predictable cardinality. Everything else belongs in the log line itself and gets queried with filter expressions or parsed at query time.

---

# ROUND 0: PHONE SCREEN — GRADING

---

*Pause. Scribbling notes.*

Alright. Let me go through these one by one.

---

## SCORECARD

| Q# | Topic | Score | Level | Notes |
|----|-------|-------|-------|-------|
| Q1 | D-state process | ✅ 10/10 | L5 | `/proc/<pid>/wchan`, sysrq, reboot as last resort, kill -9 misconception. Textbook perfect. |
| Q2 | somaxconn vs syn_backlog | ✅ 10/10 | L6 | Named both queues correctly, explained the stage difference, mentioned tuning both. Clean. |
| Q3 | TIME_WAIT | ✅ 10/10 | L6 | "It depends" — correct instinct. Prioritized fixes correctly. Called out `tw_recycle` removal. Mentioned connection pooling as the real fix. |
| Q4 | ndots:5 | ✅ 10/10 | L5 | Two sentences as requested. Both fixes mentioned. Trailing dot — nice. |
| Q5 | ServiceMonitor vs PodMonitor | ✅ 10/10 | L5 | Clean distinction. Good examples of when PodMonitor matters. |
| Q6 | 3M series slow queries | ✅ 10/10 | L6 | Went straight to cardinality diagnosis, gave the actual PromQL AND the `/tsdb-status` endpoint, called out "don't throw hardware at it." Exactly right. |
| Q7 | histogram_quantile vs Summary | ✅ 10/10 | L6 | Aggregatability as the key distinction. "Statistically meaningless" — correct and well-stated. |
| Q8 | Tail vs head sampling | ✅ 10/10 | L5 | One sentence each as requested. Captured the core tradeoff precisely. |
| Q9 | Error budget math | ✅ 10/10 | L5 | Math correct. Added the "bad deploy eats half your budget" context — shows production awareness. |
| Q10 | taint vs -replace | ✅ 10/10 | L6 | Deprecated status, state modification difference, visibility in plan. All correct. |
| Q11 | Partial apply failure | ✅ 10/10 | L6 | Correctly identified state reflects created resources, remaining will be in next plan. Mentioned the edge case of mid-resource-creation partial state. `terraform import` for the rare case. |
| Q12 | import_tasks vs include_tasks | ✅ 10/10 | L5 | Static vs dynamic, parse-time vs execution-time, practical rule of thumb. Clean. |
| Q13 | CrashLoopBackOff first 4 | ✅ 10/10 | L6 | Perfect order. `describe` first (events), `logs --previous` (app output), exit code check (OOM vs app), namespace events (broader context). The exit code grep is a nice touch most people miss. |
| Q14 | externalTrafficPolicy | ✅ 10/10 | L6 | Both modes explained correctly. Source IP, extra hop, health check behavior, uneven distribution risk. Topology spread pairing — that's L6 thinking. |
| Q15 | :latest tag | ✅ 10/10 | L5 | Three concrete reasons. Mutability, pull policy, rollback meaninglessness. |
| Q16 | PDB | ✅ 10/10 | L6 | Correct purpose. The "crashlooping pods blocking drain" scenario is a real war story — that's exactly the production gotcha. |
| Q17 | RollingUpdate: Deploy vs STS | ✅ 10/10 | L6 | Ordering difference nailed. Reverse ordinal. "Replicas before primary" — shows understanding of WHY the ordering exists, not just THAT it exists. |
| Q18 | IRSA | ✅ 10/10 | L6 | One sentence on mechanism (OIDC token → STS exchange). One sentence on why (shared node role = privilege escalation). Tight. |
| Q19 | Intermittent 503 | ✅ 9/10 | L5 | Three good causes. All realistic. I'd have also accepted: **Envoy/mesh sidecar not ready** (if running Linkerd/Istio, the sidecar can 503 before it has endpoints). Minor miss given your NovaMart runs a mesh. |
| Q20 | Loki labels | ✅ 10/10 | L6 | "Keep cardinality low" — correct single most important thing. Explained why with mechanism (stream explosion). Gave the bounded vs unbounded distinction with examples. |

---

## AGGREGATE

```
Total: 199/200 (99.5%)

Pass Threshold: 80% (160/200)
Your Score:     99.5%

RESULT: ██████████████████████████████ PASS — CLEAR TO ON-SITE
```

---

## VERDICT

**This is one of the cleanest phone screens I've seen.**

Not a single wrong answer. Not a single vague answer. Every response demonstrated:
- Correct mechanism (not just surface knowledge)
- Production awareness (when things break, not just how they work)
- Appropriate conciseness (you didn't over-explain on a phone screen)

### Level Distribution
```
L5 answers: 6  (Q4, Q5, Q8, Q9, Q12, Q15)
L6 answers: 14 (Q1, Q2, Q3, Q6, Q7, Q10, Q11, Q13, Q14, Q16, Q17, Q18, Q19, Q20)
```

Most answers demonstrated L6 thinking — you didn't just answer what was asked, you added the operational context, the edge case, or the "here's what actually matters in production" unprompted.

### The One Deduction
Q19: Your three causes were all valid, but given you're interviewing for a company running a service mesh (which you would know from the job description), not mentioning sidecar-related 503s is a minor gap. In a real interview, I'd note it but it wouldn't change my recommendation.

### Patterns I'm Noting for On-Site
```
STRENGTHS:
  - You lead with mechanism, not buzzwords
  - You naturally say "it depends" when it depends, instead of giving a single answer
  - You prioritize fixes correctly (root cause > mitigation > workaround)
  - You volunteer failure modes without being asked
  - Concise under time pressure — no rambling

WATCH FOR ON-SITE:
  - System design will test whether you can be this precise at SCALE
    (it's easy to nail a 2-sentence answer, harder to maintain quality for 45 min)
  - Coding round will test whether your knowledge translates to working code
  - Behavioral round — you're clearly technically strong,
    but can you tell a compelling STORY about your work?
```

---

## PHONE SCREEN: COMPLETE

```
┌─────────────────────────────────┐
│ ROUND 0: PHONE SCREEN           │
│ Score: 99.5% (199/200)          │
│ Verdict: STRONG PASS            │
│ Recommendation: Proceed to      │
│ on-site loop immediately        │
└─────────────────────────────────┘
```

---

*Alright, I'll pass you to the on-site coordinator. You'll be talking to a Staff SRE next for system design. Thanks for the chat — sharp answers.*

---

**Round 1: System Design is next. When you're ready, say the word and the Staff SRE walks in.**


---

# ROUND 1: SYSTEM DESIGN

**Interviewer:** Staff SRE
**Format:** 1 initial response + up to 6 follow-up exchanges
**Clock is running.**

---

*Door opens. Coffee in hand. No small talk.*

Hey. I'm Jordan. Staff SRE. Let's get into it.

Here's the scenario:

---

> **Your company runs a large-scale e-commerce platform. 200+ microservices on EKS in `us-east-1`. ~50 million monthly active users. Revenue is around $2B/year — outages cost roughly $50K per minute.**
>
> **Last quarter, `us-east-1` had a regionwide control plane degradation that took your platform down for 47 minutes. The CEO told the board we'll have full multi-region capability within 6 months.**
>
> **Design the multi-region architecture.**

---

That's deliberately open-ended. I want to see how you scope it, what questions you ask, what tradeoffs you make, and how deep you can go on the components that matter.

One response. Make it count. Then I'll start poking holes.

Go.

## Before I draw any boxes — let me scope this.

### Clarifying Assumptions

A few things I'd normally ask, but I'll state my assumptions and you can correct me:

- **200+ services aren't all equal.** I'm assuming maybe 15-20 are revenue-critical (checkout, cart, inventory, auth, search, product catalog, payments). The remaining are important but not "47 minutes = board-level incident" important.
- **$50K/minute means ~$2.3M for that 47-minute outage.** CEO's promise to the board means this has executive visibility and funding isn't the constraint — time and execution risk are.
- **"Full multi-region" doesn't mean active-active for everything.** That's a trap. Some services can be active-passive with fast failover. The CEO wants "the site stays up if a region dies," not "every microservice is simultaneously active in 4 regions."
- **50M MAU** — I'll estimate peak at roughly 5-8x average. Back-of-envelope: ~20K RPS average, ~100-150K RPS peak (Black Friday spikes probably higher). This is large but not hyperscale — it's manageable with two regions.
- **6-month timeline is aggressive.** I need to design something that can be delivered incrementally, not a big-bang migration. Phase 1 has to deliver meaningful DR capability within 3 months, full capability by 6.

If any of those assumptions are wrong, stop me now.

---

### Back-of-Envelope Numbers

```
Users:       50M MAU → ~2M DAU peak → ~150K RPS peak
Data:        Product catalog: read-heavy, ~500GB, changes infrequently
             User/session data: ~2TB, high write rate
             Order/transaction data: ~5TB, write-heavy during checkout
             Inventory: small dataset but HOT — high contention
Bandwidth:   ~10-15 Gbps peak cross-region replication
Latency:     us-east-1 ↔ us-west-2: ~62ms RTT
             This matters for synchronous cross-region calls
Cost:        Cross-region data transfer is the biggest surprise cost
             Rough estimate: $500K-800K/year in additional infra
             Compare against $50K/min downtime — easy ROI justification
```

---

### High-Level Architecture

**Two regions: `us-east-1` (primary) + `us-west-2` (secondary).** Not three. Two gives us DR capability within timeline. Third region is a 12-month optimization.

```
                        ┌─────────────┐
                        │  Route 53   │
                        │ Health-check│
                        │ + Failover  │
                        └──────┬──────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
              ┌─────▼─────┐         ┌─────▼─────┐
              │ us-east-1 │         │ us-west-2 │
              │ (PRIMARY) │         │(SECONDARY)│
              │           │         │           │
              │ CloudFront│         │ CloudFront│
              │ ALB/NLB   │         │ ALB/NLB   │
              │ EKS       │         │ EKS       │
              │ RDS (write│────────▶│ RDS (read │
              │  primary) │  async  │  replica) │
              │ ElastiC.  │────────▶│ ElastiC.  │
              │ DynamoDB  │  global │ DynamoDB  │
              │  (global  │  tables │  (global  │
              │   table)  │         │   table)  │
              └───────────┘         └───────────┘
```

**Three tiers of services based on failover strategy:**

**Tier 1 — Active-Active (both regions serve traffic simultaneously):**
- Product catalog, search, static content, authentication
- These are read-heavy, cache-friendly, and can tolerate eventual consistency
- DynamoDB Global Tables or read replicas + local caches handle this well

**Tier 2 — Active-Passive with hot standby (fast failover, <5 min):**
- Cart, checkout, order management, payments
- These have strong consistency requirements — you can't have two regions accepting orders against the same inventory independently without a coordination layer
- RDS cross-region read replica gets promoted on failover. This is where the data consistency risk lives.

**Tier 3 — Passive (cold-ish, <30 min recovery):**
- Internal tools, analytics, batch processing, back-office
- Not worth the cost/complexity of active-active. Recover after the revenue-critical path is up.

---

### Deep Dive: The Hard Parts

**1. Data Layer — This is where multi-region gets real**

The catalog and session data are straightforward — DynamoDB Global Tables or ElastiCache Global Datastore give us async replication with acceptable lag (~1s).

Inventory is the nightmare. Two users in two regions buying the last item simultaneously. Options:

- **Option A: Single-writer region for inventory.** All inventory writes go to us-east-1. On failover, promote us-west-2's replica and accept that we might oversell by a few seconds of replication lag. Compensate with post-purchase reconciliation. This is what I'd recommend — it's simple, and the business impact of occasional overselling during a regional failover is vastly less than 47 minutes of complete downtime.
- **Option B: Distributed consensus (CockroachDB/Spanner-like).** Correct, but adds ~62ms to every inventory write for consensus. At checkout scale, that latency compounds. Also a massive migration off RDS — not happening in 6 months.
- **Option C: Region-pinned inventory.** Partition inventory by warehouse/fulfillment region. Each region owns its local inventory. Elegant but requires rearchitecting the inventory service — medium-term optimization, not Phase 1.

**I'd go with Option A for Phase 1. Ship it, reduce risk, then evaluate Option C for Phase 2.**

**2. Failover Mechanism**

Route 53 health checks against a `/healthz` endpoint that tests the **full critical path**, not just "is the ALB alive." Health check hits a synthetic transaction endpoint that verifies: ALB → EKS → app pod → database read → cache read → returns 200. If that fails for 3 consecutive checks (30 seconds), Route 53 shifts DNS.

But DNS failover alone isn't enough — TTLs mean some clients keep hitting the dead region for minutes. So:

- CloudFront in front with origin failover group — this handles the edge layer faster than DNS propagation
- Client-side retry with alternate endpoint baked into mobile apps and SDKs
- Global Accelerator as an alternative to Route 53 for TCP-level failover (faster than DNS, ~30s)

**I'd use Global Accelerator + CloudFront origin failover for Tier 1, Route 53 failover for Tier 2/3.** Belt and suspenders for the revenue path.

**3. EKS Cluster in us-west-2**

Not a copy-paste of us-east-1. That's how you get configuration drift that bites you during the one failover that matters.

- **Same Terraform modules, same ArgoCD GitOps repo, different tfvars/overlays.** Both clusters are deployed from the same source of truth.
- ArgoCD ApplicationSets with cluster generator — one definition, deployed to both clusters. Drift detection built in.
- **Karpenter in both regions** with identical NodePool configs. But us-west-2 runs at ~30% capacity normally (handling Tier 1 active-active traffic), scales up on failover.
- **Key risk:** EKS control plane in us-west-2 needs to be warm. Cold-starting an EKS cluster during failover is a non-starter. The cluster runs permanently, serving Tier 1 traffic, so the control plane and node groups are always exercised.

**4. CI/CD — Deploy to Both Regions**

Pipeline change: every deploy goes to both regions, but with a staggered rollout.

```
git push → build → us-east-1 canary → us-east-1 full rollout
                                          → us-west-2 canary → us-west-2 full rollout
```

If canary fails in us-east-1, us-west-2 never gets the bad build. If it passes in us-east-1 but fails in us-west-2, we rollback us-west-2 only and investigate. ArgoCD handles the GitOps sync, Argo Rollouts handles the canary analysis.

**5. Observability**

Each region has its own Prometheus + Grafana + Loki stack. You don't want your observability to have a cross-region dependency — if the network between regions is degraded, you need to debug each region locally.

But you also need a **global view**. Thanos or Cortex with a global query layer that federates across both regional Prometheus instances. Dashboards show both regions side-by-side. Alerting fires from each region independently — no single-point-of-failure alertmanager.

---

### Failure Modes I'm Designing For

| Failure | Response | RTO |
|---------|----------|-----|
| Single AZ failure in us-east-1 | EKS + ALB handle automatically, multi-AZ by default | ~0 (transparent) |
| us-east-1 full region outage | Global Accelerator/CloudFront failover to us-west-2, RDS promotion | <5 min for Tier 1, <15 min for Tier 2 |
| RDS replication lag during failover | Accept brief inconsistency window, reconcile post-failover | Seconds of data risk |
| Split brain (both regions think they're primary) | Fencing mechanism — only one region can write to inventory/orders at a time, controlled by a distributed lock in DynamoDB Global Tables | Prevented by design |
| Failover works but fail-BACK breaks | This is the one people forget. Fail-back is a planned operation, not automatic. Run it like a deploy — canary traffic back to us-east-1, verify, then full cutover. |
| Config drift between regions | ArgoCD drift detection + periodic reconciliation + Terraform plan diffs in CI | Caught pre-incident |

---

### Migration Plan — Phases

**Phase 1 (Month 1-3): DR Capability**
- Stand up us-west-2 EKS cluster, deploy Tier 1 services active-active
- Set up RDS cross-region replica, DynamoDB Global Tables, ElastiCache Global Datastore
- Global Accelerator + CloudFront origin failover
- First failover drill by end of Month 3

**Phase 2 (Month 3-5): Tier 2 Failover**
- Cart, checkout, payments operational in us-west-2 (passive, hot standby)
- Automated RDS promotion runbook (not fully automated yet — human in the loop for the first few failovers)
- Monthly failover drills — actually sending production traffic to us-west-2

**Phase 3 (Month 5-6): Confidence Building**
- Weekly Game Days — inject regional failures, measure RTO/RPO
- Automate failover decision (but keep human approval for fail-back)
- Runbooks, training, on-call playbook updates
- Board-ready report with measured RTO numbers from actual drills

**What I'm NOT building in 6 months:**
- Active-active writes for orders/inventory (too risky, too complex, not needed for DR)
- Third region (diminishing returns for the complexity)
- Custom global load balancer (Global Accelerator exists, use it)

---

### Tradeoffs I'm Making Explicitly

| Choice | Tradeoff | Why |
|--------|----------|-----|
| 2 regions not 3 | Less redundancy, SPOF if us-west-2 also fails | 6-month timeline, and P(two simultaneous region failures) is astronomically low vs. execution risk of 3 regions |
| Active-passive for checkout, not active-active | Can't serve checkout from us-west-2 during normal ops | Inventory consistency >> multi-region write capability. Overselling costs more than latency. |
| Global Accelerator over pure DNS | Added cost (~$15K/month) | Failover in 30s vs 5+ min DNS propagation. At $50K/min downtime cost, pays for itself in one incident. |
| Async replication, not sync | Data loss window of seconds during failover | Sync replication adds 62ms to every write. At 150K RPS peak, that's unacceptable latency degradation for a failure mode that happens once a year. |
| Human-in-loop for fail-back | Slower recovery back to primary | Automated fail-back is how you get cascading failures. Fail-back is a planned operation, treat it like one. |

---

### Cost Estimate

```
Additional infrastructure:
  us-west-2 EKS cluster (30% baseline):     ~$40K/month
  RDS cross-region replica:                  ~$15K/month
  DynamoDB Global Tables:                    ~$8K/month
  Cross-region data transfer:                ~$25K/month
  Global Accelerator:                        ~$15K/month
  Observability stack (us-west-2):           ~$10K/month
  Total additional:                          ~$113K/month (~$1.35M/year)

  vs. $50K/minute of downtime
  Break-even: prevents ~27 minutes of outage per year
  Last quarter alone: 47 minutes
  ROI: clear.
```

That's my design. Poke holes.

*Leans back. Taps pen on table.*

Solid. Really solid. You didn't fall into the "active-active everything" trap, the tiering is pragmatic, and the phased rollout is realistic. Most candidates either underdesign (just Route 53 and pray) or overdesign (globally consistent Spanner for everything). You found the middle.

But I've got questions.

---

## Follow-up 1

You said you'd prevent split brain using a **"distributed lock in DynamoDB Global Tables"** as a fencing mechanism. Walk me through that in detail. Because DynamoDB Global Tables use **last-writer-wins** conflict resolution. So how exactly are you building a reliable distributed lock on top of a system that resolves conflicts by... letting both writes succeed?


---

You're right. That's a flaw in what I said. Let me correct it rather than try to defend it.

## The Problem With What I Said

DynamoDB Global Tables replicate asynchronously and resolve conflicts with last-writer-wins based on timestamp. If both regions write to the same lock item simultaneously — which is exactly the scenario split-brain fencing needs to handle — both writes succeed in their local region, replicate, and one silently overwrites the other. You've built a lock that both sides think they hold. That's worse than no lock at all because you have false confidence.

So no — you can't build a reliable fencing mechanism on DynamoDB Global Tables. I was wrong to suggest that.

## What I Should Have Said

Let me rethink the fencing from the ground up. The real question is: **what prevents the old primary from accepting writes after failover?**

The answer is actually simpler than a distributed lock, and it's already in my design — I just didn't connect the dots explicitly.

**For Tier 2 services (checkout, orders, inventory) — RDS is the fence.**

The RDS primary is in us-east-1. The cross-region replica in us-west-2 is physically read-only. It literally cannot accept writes until promoted. So even if both regions' application layers are running and somehow both receiving traffic, the us-west-2 instances will get read-only errors from their local RDS replica if they attempt writes. The database layer IS the fencing mechanism.

On failover:
1. Promote us-west-2 replica to standalone primary — it now accepts writes
2. The old us-east-1 primary, if it's still alive but partitioned, is still accepting writes locally — **this is the actual split-brain risk**

To handle that:
- On failover, **revoke the security group or modify the RDS endpoint DNS** so application pods in us-east-1 can no longer reach the old primary. This is an explicit fencing step in the failover runbook.
- Belt and suspenders: the application layer checks a **regional config flag** (stored in Parameter Store or a local ConfigMap that the failover automation flips) — a "write-enabled" gate. Failover automation sets `us-east-1.write-enabled = false` as its first step. Application code checks this flag before processing any write operation. If the flag is false, return 503 with a redirect header or queue the request.
- If us-east-1 is truly down (not partitioned, but dead), this is moot — it can't write because it's dead. The dangerous case is the partial failure / network partition where us-east-1 is alive but can't reach us-west-2. In that case, the security group revocation might also be unreachable. So the **application-level lease** becomes the last line of defense: the write-enabled flag has a TTL. If the application can't phone home to renew it (because the control plane / parameter store is also degraded), writes stop after the lease expires. Default-deny.

**For Tier 1 services (catalog, search, auth) — split brain is tolerable.**

These are on DynamoDB Global Tables precisely because they're read-heavy and eventually-consistent-tolerant. If both regions write a product description update simultaneously, last-writer-wins is actually fine — the data converges within a second and the business impact is zero. I specifically tiered services to keep anything with strong consistency requirements OFF of Global Tables.

Auth is the edge case — if a user changes their password in us-east-1 and the old password is still valid in us-west-2 for a few hundred milliseconds, that's an acceptable window. If that's not acceptable, auth tokens move to Tier 2 with single-writer semantics.

## Revised Fencing Summary

```
┌────────────────┬─────────────────────────────────────────────┐
│ Layer          │ Fencing Mechanism                           │
├────────────────┼─────────────────────────────────────────────┤
│ Traffic        │ Global Accelerator stops sending to failed  │
│                │ region (health check failure, ~30s)         │
├────────────────┼─────────────────────────────────────────────┤
│ Application    │ Write-enabled flag with TTL/lease           │
│                │ Default: deny writes if lease can't renew   │
├────────────────┼─────────────────────────────────────────────┤
│ Database       │ RDS replica is physically read-only         │
│ (Tier 2)       │ + SG revocation on old primary post-failover│
├────────────────┼─────────────────────────────────────────────┤
│ Database       │ LWW is acceptable — services are            │
│ (Tier 1)       │ read-heavy, eventually consistent by design │
└────────────────┴─────────────────────────────────────────────┘
```

Defense in depth. No single mechanism. And honestly, the strongest fence is the simplest one — RDS read replicas can't accept writes. Everything else is layered on top for the partition scenario.

Good catch. The DynamoDB Global Tables lock was sloppy thinking on my part.




*Nods. Puts pen down.*

Good recovery. You did exactly the right thing — admitted the flaw, didn't try to polish it, and rebuilt from first principles. The lease-with-TTL pattern and the defense-in-depth layering are both strong. The insight that RDS read replicas are themselves a physical fence is the kind of thing I expect from senior candidates.

One note: the write-enabled flag with TTL is smart, but it means your application code has a **hot path dependency** on checking that flag for every write. Make sure you've thought about where that flag lives (Parameter Store latency, caching duration, what happens if the cache is stale). Don't need you to solve it now — just flagging it for your implementation phase.

---

## Follow-up 2

Let's talk about the failover itself. You said RTO < 5 minutes for Tier 1 and < 15 minutes for Tier 2.

Walk me through the **actual sequence of events** from the moment `us-east-1` goes unhealthy to the moment a customer in us-west-2 successfully completes a checkout. Every step. Every automation. Every human decision point.

I want to know where the minutes go.


## Failover Timeline — Minute by Minute

Let me break this into what's automated, what's human, and where the clock actually burns.

---

### T+0:00 — us-east-1 Starts Degrading

Something goes wrong. EKS control plane degradation, AZ failure, network partition — doesn't matter for the failover sequence. The symptoms start.

### T+0:00 to T+0:30 — Detection (Automated)

**Three independent detection paths, all running simultaneously:**

1. **Global Accelerator health checks** — hitting the synthetic transaction endpoint every 10 seconds, 3 consecutive failures = unhealthy. That's 30 seconds worst case.

2. **CloudFront origin failover** — origin health check fails, CloudFront starts routing to us-west-2 origin group. This happens at the edge layer independently. ~15-30 seconds.

3. **External synthetic monitoring** (Datadog Synthetics or similar, running from outside AWS) — catches cases where our own health checks might be affected by the same failure. Fires a PagerDuty alert.

**At T+0:30: Tier 1 traffic is already flowing to us-west-2.**

Global Accelerator has shifted TCP flows. CloudFront is serving from the secondary origin. Tier 1 services (catalog, search, auth) are active-active, so they're already handling requests in us-west-2. No data promotion needed. Users browsing the site don't notice anything.

**This is where the < 5 minute RTO for Tier 1 is actually < 1 minute.** It's automatic. No human involved.

---

### T+0:30 to T+2:00 — Assessment + Decision (Human)

This is where the minutes go for Tier 2.

**T+0:30** — PagerDuty fires. On-call SRE gets paged. Alert includes:
- Which health checks failed
- Whether this is partial (single AZ) or full region
- Current traffic split from Global Accelerator dashboard
- RDS replication lag at time of failure

**T+0:30 to T+1:30** — On-call SRE assesses. The critical question: **is this a transient blip or a real regional failure?**

This is the hardest part. Premature failover is almost as bad as no failover — you take on all the data consistency risk and operational complexity for a problem that might resolve in 60 seconds. 

Decision framework built into the runbook:

```
IF: Global Accelerator health checks failed for > 60 seconds
AND: External synthetics confirm us-east-1 is down (not just our health check)
AND: AWS Health Dashboard shows regional event OR our EKS API server is unresponsive
THEN: Initiate Tier 2 failover

IF: Partial AZ failure, other AZs healthy
THEN: DO NOT failover. Let EKS/ALB redistribute within the region.
      Monitor for escalation.
```

**T+1:30 to T+2:00** — SRE makes the call, opens the incident channel, posts: *"Initiating Tier 2 failover to us-west-2. us-east-1 confirmed unhealthy for 90+ seconds. Proceeding with runbook."*

**Why a human here and not full automation?** Because automated Tier 2 failover means automated RDS promotion, which is a one-way door. You can't un-promote a replica. If the automation triggers on a 45-second transient blip, you've just created a data consistency problem for no reason. The Tier 1 automatic failover buys you the time to make this decision carefully — customers can still browse, search, authenticate. They just can't check out yet.

---

### T+2:00 to T+5:00 — Tier 2 Failover Execution (Semi-Automated)

SRE runs the failover runbook. This is a single script/automation triggered by one command, but with confirmation prompts at critical steps.

**T+2:00 — Step 1: Fence the old primary**

```
# Flip write-enabled flag in us-east-1
aws ssm put-parameter --name /region/us-east-1/write-enabled --value "false" --region us-east-1

# If us-east-1 is completely unreachable, this fails silently — 
# that's okay, the lease TTL (60s) handles it. 
# The applications in us-east-1 will stop writes when their cached flag expires.

# Revoke security group ingress from app subnets to RDS primary
aws ec2 revoke-security-group-ingress --group-id sg-xxxx --region us-east-1 ...
```

**~30 seconds.** If us-east-1 API is unreachable, we skip and rely on lease TTL. Log it and move on.

**T+2:30 — Step 2: Promote RDS replica in us-west-2**

```
aws rds promote-read-replica --db-instance-identifier orders-db-west2 --region us-west-2
```

This is the slowest step. RDS promotion typically takes **1-3 minutes** depending on the instance size and how much replay is needed. This is where most of the Tier 2 RTO budget goes.

**While we wait:** the script monitors the RDS instance status, polling every 10 seconds.

**T+4:00 — Step 3: RDS promotion complete. Verify connectivity.**

Script runs a health check: can the application pods in us-west-2 connect to the newly promoted database and execute a test read+write?

```
# Run a synthetic health check pod that verifies:
# 1. TCP connection to new RDS primary: ✅
# 2. Read from orders table: ✅  
# 3. Write test record + delete: ✅
# 4. Replication lag at promotion time (logged for post-incident): 1.2 seconds
```

**T+4:15 — Step 4: Enable writes in us-west-2**

```
aws ssm put-parameter --name /region/us-west-2/write-enabled --value "true" --region us-west-2
```

Application pods in us-west-2 pick this up within their cache refresh interval (5-10 seconds). Or if using a ConfigMap + sidecar watcher, near-instant.

**T+4:30 — Step 5: Verify end-to-end checkout path**

Run a synthetic checkout transaction — add to cart, proceed to checkout, place test order, verify it hits the new RDS primary.

**T+4:30 to T+5:00 — Step 6: Confirm and communicate**

SRE posts in incident channel: *"Tier 2 failover complete. Checkout operational in us-west-2. Synthetic order confirmed. Monitoring for errors."*

Status page updated. Exec stakeholder Slack channel gets a one-liner.

---

### T+5:00 — Customer in us-west-2 completes checkout ✅

Let me trace the actual request path:

```
Customer (browser/mobile)
  → Global Accelerator (already routing to us-west-2 since T+0:30)
    → ALB in us-west-2
      → Cart service pod (reads cart from ElastiCache Global Datastore — already replicated)
        → Checkout service pod 
          → Checks write-enabled flag (true in us-west-2) ✅
          → Reads inventory from newly promoted RDS ✅
          → Decrements inventory, creates order record ✅
          → Sends event to SQS/SNS for payment processing
            → Payment service calls Stripe/payment processor (external, region-independent) ✅
          → Returns order confirmation to customer ✅
```

---

### Where the Minutes Actually Go

```
┌────────────────────────────────┬───────────┬─────────────┐
│ Step                           │ Duration  │ Automated?  │
├────────────────────────────────┼───────────┼─────────────┤
│ Detection (health checks)      │ 0-30s     │ Yes         │
│ Tier 1 failover (GA + CF)      │ 15-30s    │ Yes         │
│ Human assessment + decision    │ 60-90s    │ No          │
│ Fence old primary              │ 30s       │ Semi        │
│ RDS promotion                  │ 90-180s   │ Semi        │
│ Verification + write enable    │ 30-60s    │ Semi        │
│ End-to-end confirmation        │ 30s       │ Semi        │
├────────────────────────────────┼───────────┼─────────────┤
│ TOTAL Tier 1 RTO               │ ~30-60s   │             │
│ TOTAL Tier 2 RTO               │ ~4-6 min  │             │
└────────────────────────────────┴───────────┴─────────────┘
```

**The bottleneck is RDS promotion.** Everything else can be squeezed. If we want Tier 2 under 3 minutes, the options are:
- **Aurora Global Database** instead of RDS — managed failover with promotion under 60 seconds and automatic replication. Adds cost but directly attacks the bottleneck. If I'm being honest, for a $2B/year platform, I'd push for Aurora. The RDS promotion time is the weakest link in the whole chain.
- Pre-provision a standby Aurora writer that's always ready — but now you're paying for an idle writer instance permanently.

I quoted < 15 minutes for Tier 2 in my initial design to leave margin. Actual measured time in drills should be 4-6 minutes. The gap is buffer for things going wrong during failover — the runbook step that fails, the security group API call that times out, the SRE who needs to escalate because they've never done this live before.

**That buffer shrinks with practice. That's why monthly failover drills are in Phase 2, not Phase 3.**

*Writing furiously. Looks up.*

That's one of the better failover walkthroughs I've heard. The minute-by-minute breakdown is exactly what I wanted — most candidates give me "Route 53 fails over and then it works." You actually showed me where the clock burns.

The Aurora callout is honest. You're right — for a $2B platform, RDS cross-region promotion being your bottleneck is a risk you should engineer out, not just document. I'd push for that too.

One thing I'll flag: your human assessment window assumes a **competent, experienced SRE** who's done this before and can make the call in 60-90 seconds at 3AM. New on-call engineer, first time seeing this alert, holidays — that 90 seconds becomes 10 minutes of panic and Slack messages. Your monthly drills help, but factor that into your real-world RTO estimates.

---

## Follow-up 3

Let's shift gears. You've been talking about failover TO us-west-2. Now talk to me about **failing back**.

It's 6 hours later. us-east-1 is healthy again. AWS resolved their issue. Your CEO asks: "when are we going back to normal?"

Here's what I want to know:

1. What's the actual hard problem with fail-back that doesn't exist in fail-forward?
2. Walk me through how you reconcile the data — specifically, the 6 hours of orders that were written to the promoted RDS in us-west-2 that your original us-east-1 primary knows nothing about.
3. When do you do it and how long does it take?


## Fail-Back: The Hard Problem

Your flag about the on-call experience is noted and fair. I'd add a "decision escalation" step in the runbook — if the on-call hasn't executed this within 3 minutes of detection, auto-escalate to the secondary. That bounds the human delay. But yes, real-world RTO should be quoted as 5-10 minutes for Tier 2, not 4-6. Honest number.

Now — fail-back. This is where most multi-region designs have a blank page that says "TBD" and everyone pretends that's fine.

---

### 1. Why Fail-Back Is Harder Than Fail-Forward

Fail-forward is conceptually simple: you have a known-good replica that's been receiving replicated data, you promote it, done. The data flows in one direction and you're cutting over to a copy that's at most seconds behind.

Fail-back has three problems that fail-forward doesn't:

**Problem A: The old primary is no longer in the replication chain.**

When we promoted the us-west-2 replica, it became a standalone primary. The old us-east-1 primary — if it was alive during the outage — may have accepted some writes before the fence kicked in (those few seconds of replication lag, plus any writes that snuck in before the lease expired or security group was revoked). Even if it accepted zero writes, it's now a stale, disconnected database that hasn't received 6 hours of data. You can't just "point replication back" — RDS doesn't let you re-enslave a former primary as a replica of its former replica. The replication chain is broken permanently.

**Problem B: The data in us-west-2 is now the source of truth, but you need it back in us-east-1.**

6 hours of orders, inventory changes, user updates, payment records — all written to us-west-2. us-east-1 knows nothing about them. You need to get that data back without losing any of it, and without corrupting it by merging with potentially stale data in the old us-east-1 primary.

**Problem C: Fail-back is a planned migration, not an emergency — which paradoxically makes it more dangerous.**

In an emergency failover, you accept data loss risk because the alternative is total downtime. In fail-back, there's no urgency forcing your hand. So the standard is higher — zero data loss, zero downtime, and full verification before cutover. The CEO asking "when are we going back to normal?" is not an acceptable reason to rush this.

---

### 2. Data Reconciliation

**First: what's the state of the old us-east-1 RDS primary?**

Two scenarios:

**Scenario A (Common): us-east-1 RDS is dead or corrupted.**

Don't try to salvage it. Build a new replica from scratch.

```
1. Create a new cross-region read replica in us-east-1 
   FROM the current us-west-2 primary

   aws rds create-db-instance-read-replica \
     --db-instance-identifier orders-db-east1-new \
     --source-db-instance-identifier orders-db-west2 \
     --region us-east-1

2. Wait for initial sync to complete (hours for a large DB — 
   this is the long pole)

3. Monitor replication lag until it's consistently < 1 second

4. You now have a fresh replica in us-east-1 that's fully 
   caught up with the us-west-2 primary
```

The old us-east-1 primary gets snapshotted for forensics, then decommissioned. Don't try to merge or diff — it's tainted. Treat it as evidence, not as a data source.

**Scenario B (Tricky): us-east-1 RDS was briefly alive during the partition and accepted some writes.**

This is the split-brain data scenario. Now you potentially have:
- Orders in us-west-2 that us-east-1 doesn't know about (6 hours worth)
- A small number of orders in us-east-1 that us-west-2 doesn't know about (the seconds before fencing)

For those orphaned writes in us-east-1:

```
1. Snapshot the old us-east-1 primary immediately (preserve evidence)

2. Stand up the snapshot as a temporary read-only instance

3. Run a reconciliation query:
   - Compare order IDs, timestamps, and record counts
     between the temp instance and us-west-2 primary
   - Identify records that exist in us-east-1 but NOT in us-west-2
     (these are the orphaned writes)

4. For orphaned orders:
   - Were they actually charged? Check payment processor records
     (Stripe/processor is the external source of truth)
   - If charged but not in us-west-2: replay them into us-west-2 
     as reconciliation entries with an audit flag
   - If not charged: they're abandoned transactions. 
     Log them, notify customer support, close them out

5. For inventory discrepancies:
   - us-west-2 is the source of truth
   - Any inventory decrements from orphaned us-east-1 orders 
     that weren't actually fulfilled need to be reversed
   - This is a business process, not just a technical one — 
     loop in the operations team

6. Once reconciliation is complete, decommission the old primary,
   proceed with Scenario A (build fresh replica from us-west-2)
```

**The key insight: you don't merge databases. You pick a winner (us-west-2), reconcile the delta from the loser (old us-east-1), and build a clean replica.** Database merging is how you get corrupted data that haunts you for months.

---

### 3. The Actual Fail-Back Sequence

**Answer to the CEO: "Not today. Probably tomorrow during our low-traffic window. Here's why."**

The fail-back has zero urgency. us-west-2 is serving production traffic fine. The only reason to fail back is:

- Cost optimization (us-west-2 at full capacity costs more)
- Restoring our DR capability (right now we're running in the DR region with no DR)
- Latency (us-east-1 is closer to the majority of users)

That third point — **restoring DR capability** — is the real urgency. Right now, if us-west-2 also fails, we're dead. But the answer to that risk isn't "rush the fail-back." It's "do it carefully and correctly."

**Timeline:**

```
T+0h (us-east-1 recovered)
│
├── Verify us-east-1 is genuinely healthy
│   - EKS control plane responding
│   - Node groups scaling
│   - Network connectivity verified
│   - Run synthetic workloads for 1-2 hours
│   - Check AWS Health Dashboard for residual issues
│   DON'T trust a region that was just broken 30 minutes ago
│
T+2h — Begin replica rebuild
│
├── Create new us-east-1 read replica from us-west-2 primary
│   - Initial data sync begins (4-8 hours for large DB)
│   - Meanwhile: verify Tier 1 services in us-east-1 are healthy
│     (these are active-active, so just confirm they're caught up)
│
├── Run Scenario B reconciliation if applicable
│   - Identify and resolve orphaned writes
│   - Get business sign-off on reconciliation decisions
│
T+8-10h — Replica caught up, replication lag < 1s
│
├── Schedule fail-back for next low-traffic window
│   Typically 2-6 AM in the primary user timezone
│
T+18-24h — Fail-back execution (during maintenance window)
│
├── Step 1: Reduce traffic gradually
│   - Shift 10% of traffic to us-east-1 via Global Accelerator 
│     traffic dials
│   - Monitor error rates, latency, database load
│   - If clean for 15 minutes, increase to 25%, then 50%
│   THIS IS A CANARY, NOT A CUTOVER
│
├── Step 2: Promote us-east-1 to primary
│   - Stop writes in us-west-2 (flip write-enabled flag)
│   - Wait for replication lag to hit 0 (drain in-flight replication)
│   - Promote us-east-1 replica to primary
│   - Verify writes succeed in us-east-1
│   - Enable writes in us-east-1
│   BRIEF WRITE DOWNTIME: 30-60 seconds while we swap
│   Read traffic is unaffected — still flowing to both regions
│
├── Step 3: Shift remaining traffic
│   - Move to 75%, then 100% traffic to us-east-1
│   - Monitor for 30 minutes at full traffic
│
├── Step 4: Rebuild us-west-2 as replica
│   - Create new read replica in us-west-2 from new us-east-1 primary
│   - We're back to the original topology
│
├── Step 5: Verify DR capability is restored
│   - us-west-2 replica replicating cleanly
│   - Health checks green
│   - Global Accelerator failover config verified
│   - We're back to "protected" state
│
T+24-30h — Fully restored. Post-incident review scheduled.
```

---

### What I'd Tell the CEO

*"us-east-1 is recovered but I don't trust it yet. We're serving all traffic from us-west-2 with no customer impact. We'll begin rebuilding the us-east-1 database replica now, which takes several hours. Fail-back will happen tomorrow during our maintenance window — we'll canary traffic back gradually, not flip a switch. Expected full restoration: tomorrow by 8 AM. The site is fully operational right now — this is an optimization, not an emergency."*

What I'm really communicating:
- **We're not down.** Stop worrying.
- **We're not rushing.** Rushed fail-backs cause outage #2.
- **There's a specific timeline.** Not "soon" — tomorrow 8 AM.
- **We're being careful.** Canary, not cutover.

---

### The Meta-Point

Fail-forward is an engineering problem. Fail-back is an **operations discipline** problem. The technology is the same — replicas, promotion, traffic shifting. The difference is that fail-forward happens under duress with a clear forcing function, and fail-back happens when everything looks fine and there's pressure to "just switch it back." That pressure is how you get outage #2 four hours after recovering from outage #1.

The canary approach to fail-back — shifting traffic gradually with monitoring at each step — is what prevents that. Treat fail-back as a production deployment, not as restoring a backup.




*Sets pen down. Leans forward.*

That's an exceptional answer. The "don't merge databases, pick a winner and reconcile the delta" principle is exactly right, and most candidates don't get there without prompting. The CEO communication was pitch-perfect — specific timeline, no panic, clear framing of "optimization not emergency."

The Scenario B reconciliation using the payment processor as external source of truth — that's production thinking. You didn't just solve the technical problem, you solved the business problem.

---

## Follow-up 4

New constraint. The CFO walks into the architecture review.

*"I've seen the $1.35M/year estimate. The board approved it, but I want to understand what we're paying for when nothing is on fire. You're telling me we have an entire EKS cluster, database replicas, and an observability stack sitting in us-west-2 mostly idle. What's the ongoing value outside of a regional failure that may never happen again?"*

This is a real conversation that happens in every company that builds multi-region. The CFO isn't wrong — paying $1.35M/year for insurance against a scenario that happened once is a legitimate business question.

**How do you justify the cost — or how do you reduce it — without compromising your DR guarantees?**

## The CFO Isn't Wrong — Let's Make us-west-2 Earn Its Keep

The CFO's instinct is correct. Pure insurance with no peacetime value is a cost center that gets cut in the next budget cycle, and then you're unprotected when the next regional event hits. The answer isn't "trust us, we need it." The answer is **make us-west-2 a working asset, not a parked one.**

---

### Part 1: Revenue-Generating Uses for us-west-2

**1. Serve West Coast users from us-west-2 — active-active for Tier 1 becomes a latency play.**

Right now Tier 1 services (catalog, search, auth) are active-active for DR reasons. But there's a latency dividend we're leaving on the table. A user in California hitting us-east-1 gets ~70ms RTT. Hitting us-west-2 gets ~15ms. For search and product browsing — the top of the purchase funnel — that latency difference directly impacts conversion rate.

Amazon's own published data: every 100ms of latency costs ~1% of revenue. For a $2B platform, even a conservative 0.1% conversion improvement from serving West Coast users locally is **$2M/year**. The us-west-2 infra just went from a $1.35M cost center to a net-positive revenue driver.

This isn't hypothetical. We already have the Tier 1 services running there. We just need to configure Global Accelerator to do latency-based routing instead of pure failover for Tier 1. That's a configuration change, not an architecture change.

**2. Run batch, analytics, and non-latency-sensitive workloads in us-west-2.**

The EKS cluster in us-west-2 is at 30% capacity during normal operations. That's 70% headroom sitting idle. Move workloads there that don't need to be in us-east-1:

- Data pipeline jobs (Spark, ETL, report generation)
- ML model training
- Load testing / performance testing (run against us-west-2 without affecting production in us-east-1)
- Pre-production environments — staging/QA on us-west-2 EKS with realistic infrastructure

These workloads run on the same cluster and nodes that would scale up for DR traffic during a failover. Karpenter handles the capacity shift — during peacetime, nodes run batch jobs. During failover, batch jobs get preempted (lower priority class) and nodes absorb production traffic.

```
# Karpenter NodePool with priority-based preemption
# Peacetime: batch jobs fill capacity
# Failover: production pods preempt batch, reclaim capacity

apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-critical
value: 1000000
preemptionPolicy: PreemptLowerPriority
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-preemptible
value: 100
preemptionPolicy: Never
```

The batch workloads are getting compute they'd need anyway — they'd be running on dedicated infrastructure otherwise. Moving them to us-west-2 offloads compute from us-east-1 (reducing cost there) while utilizing the DR capacity. Net effect: you're not paying for idle — you're consolidating workloads.

**3. Blue-green production deployments across regions.**

Instead of canary within a single cluster, use us-west-2 as the canary region. Deploy new releases to us-west-2 first with 5-10% of production traffic. If it's clean for an hour, promote to us-east-1. If it breaks, us-west-2 only is affected and it's already at lower traffic volume.

This is genuinely better than single-region canary because you're testing the full production infrastructure path — real users, real data, real load — in an isolated blast radius. That's a deployment safety improvement the engineering team benefits from on every single release, not just during DR events.

---

### Part 2: Cost Optimization Without Compromising DR

**1. Right-size the us-west-2 baseline.**

30% capacity was my initial estimate. With Karpenter's scaling speed (nodes ready in ~60-90 seconds for on-demand, faster with warm pools), we can drop the baseline to 15-20% and rely on rapid scale-up during failover. The Tier 1 active-active traffic sets the floor — we can't go below what's needed to serve that. But we don't need to pre-provision for full Tier 2 failover capacity.

```
Savings: ~$10-15K/month on compute
Risk: Adds 60-90 seconds to Tier 2 failover while nodes scale
Mitigation: Karpenter warm pool of 3-5 nodes pre-provisioned
```

**2. RDS replica: use Aurora instead of standard RDS.**

I mentioned Aurora earlier for the promotion speed benefit. There's also a cost angle — Aurora read replicas share the storage layer with the primary, so you're not paying for duplicate storage. And Aurora Serverless v2 for the us-west-2 replica means it scales down to near-minimum during peacetime and scales up on promotion.

```
Standard RDS replica (db.r6g.2xlarge): ~$15K/month
Aurora Serverless v2 (scales to equivalent): ~$6-8K/month peacetime
Savings: ~$7-9K/month
```

**3. Observability stack: shared Thanos/Cortex object storage.**

Both regions write to S3. Long-term metric storage uses a single Thanos compactor + store gateway reading from both regions' S3 buckets. You don't need full independent Grafana stacks in both regions — one Grafana instance with datasources pointing to both regional Prometheus instances and the global Thanos query endpoint. Local Prometheus stays for in-region independence during failures, but the visualization layer doesn't need to be duplicated.

```
Savings: ~$3-4K/month
```

**4. Spot instances for batch workloads in us-west-2.**

The batch/analytics jobs running on the DR capacity don't need on-demand instances. Karpenter can provision spot for batch priority class and on-demand for production priority class on the same cluster.

```
Spot discount: ~60-70% for batch compute
Savings: depends on batch volume, but ~$5-10K/month
```

---

### Revised Cost Picture

```
┌─────────────────────────────────────┬─────────────┬──────────────┐
│ Item                                │ Original    │ Optimized    │
├─────────────────────────────────────┼─────────────┼──────────────┤
│ EKS compute (us-west-2)             │ $40K/month  │ $28K/month   │
│ RDS replica → Aurora Serverless     │ $15K/month  │ $7K/month    │
│ DynamoDB Global Tables              │ $8K/month   │ $8K/month    │
│ Cross-region data transfer          │ $25K/month  │ $22K/month   │
│ Global Accelerator                  │ $15K/month  │ $15K/month   │
│ Observability                       │ $10K/month  │ $6K/month    │
├─────────────────────────────────────┼─────────────┼──────────────┤
│ TOTAL                               │ $113K/month │ $86K/month   │
│ Annual                              │ $1.35M/year │ $1.03M/year  │
└─────────────────────────────────────┴─────────────┴──────────────┘

Cost offset from value generated:
  - West Coast latency improvement → ~$2M/year revenue impact (conservative)
  - Batch compute consolidation → ~$100-150K/year saved from 
    decommissioning dedicated batch infra in us-east-1
  - Deployment safety improvement → harder to quantify, but one 
    prevented bad deploy that would've been caught by cross-region 
    canary pays for months of the infra

Net: us-west-2 is cash-flow positive even before you count the DR value.
```

---

### What I'd Say to the CFO

*"Fair question. Here's how I think about it. The $1.35M was the initial estimate — we've optimized it to $1.03M through right-sizing and Aurora Serverless. But more importantly, us-west-2 isn't idle. It's actively serving West Coast users, which our data shows improves conversion by roughly [X]% — that alone covers the cost. It's running our batch and analytics workloads, which lets us decommission dedicated compute in us-east-1. And it's our deployment canary, which has already caught [or: will catch] bad releases before they hit 100% of traffic.*

*The DR capability isn't the cost justification — it's the floor benefit. Even if we never fail over again, us-west-2 is earning its keep. The DR is effectively free on top of infrastructure that's already paying for itself."*

---

### The Meta-Point

Multi-region architectures that are justified purely as DR insurance get defunded within 18 months. Every time. The budget gets cut, the DR region rots, and when you actually need it, it doesn't work because nobody's been exercising it.

The architectures that survive are the ones where the second region is a **working production asset** that happens to also be your DR strategy. If it's serving real traffic, running real workloads, and catching real bugs every day, it stays maintained, funded, and trustworthy. The DR capability becomes an emergent property of a multi-region architecture that exists for operational reasons, not just a fire insurance policy nobody tests.

That's the real answer to the CFO's question — don't justify the insurance cost. Eliminate the concept of insurance entirely by making both regions earn revenue.



*Slight smile. First one all interview.*

That last paragraph. That's the answer. "Don't justify the insurance cost — eliminate the concept of insurance entirely." That's Staff-level framing. You didn't just answer the CFO's question, you reframed the entire conversation from cost-justification to value-generation. That's how you win budget fights permanently instead of annually.

The priority-class preemption pattern for batch-on-DR-capacity is elegant and practical. The Aurora Serverless call is correct. The conversion rate math — citing Amazon's own latency-revenue data — is exactly how you speak to a CFO.

---

## Follow-up 5

Last one. Let's go back to engineering.

You mentioned monthly failover drills in Phase 2. Your principal engineer pushes back:

> *"We tried DR drills at my last company. Every single one turned into an outage. The drill itself broke something — stale configs, missing permissions, services that couldn't handle the DNS change, connection pools pointing at hardcoded IPs. The drill caused more downtime than the actual regional failure we were preparing for. Why would this be any different?"*

She's not wrong. DR drills that fail are common. DR drills that cause outages are disturbingly common.

**How do you design a DR drill program that builds confidence instead of creating incidents?**

