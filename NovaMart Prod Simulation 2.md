## FRIDAY 7:00 AM — DAY 12

---

You wake up without an alarm. Six and a half hours of sleep. Phone check: clean. No pages. The longest quiet streak since Day 1.

```
OVERNIGHT:

11:15 PM — Rachel: "FBI CFAA referral package finalized. 
  Outside counsel submitting today. Carlos's attorney 
  will likely be contacted within 2-3 weeks. NDA civil 
  action: counsel recommends waiting for FBI response 
  before filing separately.
  
  Supervisory authority: no new communications. Both 
  filings acknowledged. SA-2024-0119-NM response 
  deadline: Feb 5 (12 days). I'll have a draft for 
  technical review by Monday."

11:45 PM — Tom: "Runbook complete. 22 pages. Covers:
  - Broker restart (rolling, with ISR verification)
  - Consumer lag debugging (threshold-based decision tree)
  - Connector failure recovery (common error codes + fixes)
  - Partition rebalancing (when to intervene vs wait)
  - ISR monitoring (alert conditions + response)
  - Disk usage alerts (thresholds + expansion procedure)
  - Linkerd mesh troubleshooting (sidecar injection, mTLS)
  - Grafana dashboard guide (what each panel means)
  
  Also includes 'Things Tom Knows That Aren't Written Down' 
  section with tribal knowledge about the pipeline behavior 
  under various failure modes. Marcus said this was the 
  most useful part.
  
  R-6a: everything is done or deploying today. Dashboard 
  import is the last item — Marcus is doing it at 8 AM."

6:00 AM — Marcus: "Early. Importing Tom's Grafana dashboards 
  now. Three dashboards:
    1. Kafka Cluster Overview (broker health, ISR, partitions)
    2. Kafka Connect Status (connectors, tasks, lag)
    3. Data Pipeline E2E (end-to-end latency, throughput, errors)
  
  All three imported. Looks clean. Panels are populating 
  with live data from the meshed pods.
  
  Tom's Kafka broker dashboard is actually better than our 
  standard template. Requesting permission to adopt his 
  layout for other StatefulSet monitoring."

6:15 AM — Derek: "stockCache staging soak: 14 hours clean.
  Cache hit rate: 99.7% (43,891 products cached)
  Tier 2 responses (cached stock): 0 (CB never opened)
  Tier 3 responses (blind accept): 0
  Cache refresh failures: 0
  Memory footprint: 4.1 MB
  
  Ready for production canary. Starting at 8 AM with 5%, 
  same protocol as the CB rollout."

6:20 AM — Jake: "Image admission 24-hour validation report:

  Production audit mode — 24 hours:
    Total pod creates: 847
    Compliant (ECR): 831 (98.1%)
    Non-compliant (flagged): 16 (1.9%)
      - Kibana: 8 pod restarts (Docker Hub image)
      - Grafana: 6 pod restarts (Docker Hub image)
      - Alex test pod: 1 (nginx:latest — terminated)
      - FluentBit: 1 (??? — checking this one)
    
  The FluentBit one surprised me. Our logging sidecar 
  is using fluent/fluent-bit:2.1 from Docker Hub. It's 
  injected by the FluentBit DaemonSet in the logging 
  namespace. That's 47 pods across the cluster, all 
  using a Docker Hub image.
  
  This wasn't on my ECR migration list because nobody 
  told me FluentBit was from Docker Hub. The DaemonSet 
  was set up before I joined.
  
  If we go to enforce mode Feb 2 without migrating 
  FluentBit, every node that restarts will lose its 
  logging sidecar. That's... bad."
```

---

### 7:15 AM — Coffee, Prioritize

Good morning. The R-6a milestone is effectively closed — Tom's runbook and dashboards are the last items and they're done. Derek's stockCache is soaking clean. Image admission found something interesting.

The FluentBit finding is significant. Not an incident, but it's a ticking dependency: enforce mode in 8 days, and 47 pods across every node are running Docker Hub images. If Jake hadn't caught this in audit mode, we'd have discovered it the hard way on Feb 2 when nodes start cycling and logging goes dark.

This is exactly why we deploy in audit mode first.

```
FRIDAY PLAN:
  8:00  R-6a milestone formal close
  8:00  Derek stockCache canary start
  8:30  Jake FluentBit ECR migration scoping
  AM    Normal operations + sprint close
  PM    Week retrospective, next sprint planning
```

---

### 8:00 AM — R-6a Milestone Close

Post to #platform-engineering:

```
[User] (8:00 AM):
R-6a MILESTONE: KAFKA PIPELINE SECURED ✅

All deliverables complete:
  ✅ ECR image migration (Tue) — Docker Hub eliminated
  ✅ Linkerd mesh injection (Wed) — mTLS on all pods
  ✅ S3 bucket Terraform imports (Wed) — 3 buckets managed
  ✅ Monitoring dashboards (Thu) — 3 Grafana dashboards live
  ✅ Runbook (Thu) — 22 pages, including tribal knowledge

Committed date: Fri Jan 26. Delivered: Fri Jan 26.
Several components delivered 1-2 days early.

This was a board-committed deliverable. Tom Chen led the 
execution — ECR, mesh, Terraform, dashboards, and runbook 
in 4 working days. Marcus Webb partnered on every kubectl 
operation and Grafana review.

Next milestone: R-6b (Kafka FULLY ONBOARDED) — Feb 9.
Scope: ArgoCD Helm chart, full GitOps management, 
automated rollback, production readiness review.

Tom — outstanding work this week. The pipeline went from 
shadow infrastructure to production-grade in one sprint. 
That's the standard we're setting going forward.
```

```
Tom (8:04 AM):
"🎉 First milestone I've hit in months. Thank you for 
giving me the chance to do this right instead of just 
shutting it down.

R-6b planning: I'll draft the ArgoCD Helm chart this 
weekend (my choice, not asked). Monday I'll have a PR 
for review. The chart needs to handle the StatefulSet 
ordering (ZK before brokers before Connect) which is 
the tricky part."

Sarah (8:06 AM):
"Forwarding to James. Board-committed deliverable met 
on time. He'll want this for the 30-day follow-up.

Also: James approved Marcus's request to adopt Tom's 
dashboard layout as the StatefulSet monitoring standard. 
'If the person who built it in shadow can build something 
better than our standard, our standard needs updating.'"
```

---

### 8:10 AM — Jake FluentBit

DM to Jake:

> Good catch on FluentBit. This is exactly why audit mode exists — finding things before enforce mode breaks them.
>
> FluentBit is critical infrastructure. If logging goes dark, we're blind during incidents. This is priority one for your ECR migration punch list.
>
> Questions I need answered:
> 1. What FluentBit version and image tag are we running?
> 2. Is it deployed via Helm chart or raw manifests?
> 3. Who owns the logging namespace? (I suspect it's "Carlos set it up and nobody touches it" — another haunted forest)
> 4. Is there a staging equivalent to test the ECR image?
>
> Timeline: FluentBit ECR migration must be done before Feb 2 enforce mode. That gives you 7 days. Given that it's a DaemonSet on every node, the rollout needs to be careful — rolling update with a canary node first.

```
Jake (8:18 AM):
"Answers:

1. fluent/fluent-bit:2.1.10 — about 4 months old. 
   Current is 2.2.2.
2. Helm chart: fluent-bit (official, helm.fluent.io). 
   But it's NOT in ArgoCD. It was helm install'd 
   manually. No values file in git that I can find.
3. Nobody. The logging namespace has no CODEOWNERS, 
   no README, no ArgoCD Application. There's also 
   a fluentd deployment in there that looks dormant 
   (0 replicas).
4. No staging equivalent.

So: another haunted forest. The logging infrastructure 
is unmanaged, running Docker Hub images, with no GitOps 
and no owner.

I can push the image to ECR today. The tricky part is 
the Helm values — I need to reverse-engineer what's 
deployed to get the current config:

  helm get values fluent-bit -n logging

Then reproduce it in a values.yaml, check it into git, 
create an ArgoCD Application, and do the image swap as 
part of the ArgoCD sync.

Or I just swap the image tag in the existing Helm release 
and deal with ArgoCD onboarding separately. Faster but 
less clean."
```

DM to Jake:

> Do it properly. We just spent two weeks learning what happens when infrastructure runs outside GitOps. No more "quick swaps." Here's the approach:
>
> 1. Today: `helm get values`, push image to ECR, write the values.yaml
> 2. Monday: PR with ArgoCD Application + values.yaml
> 3. Tuesday: Deploy via ArgoCD sync (rolling DaemonSet update)
> 4. Wednesday-Thursday: Validate logging across all nodes
> 5. Friday Feb 2: Enforce mode with FluentBit already on ECR
>
> The dormant fluentd deployment — check if anything references it. If not, mark for cleanup. Don't delete it today.
>
> Add this to the drift detection findings. When Marcus's AWS audit found shadow EC2 instances, and your image admission found Docker Hub images — these are the same pattern. Infrastructure that was set up manually, worked, and was forgotten. The automation we're building is specifically designed to make this pattern impossible.

```
Jake (8:25 AM):
"On it. Clean path. I'll have the values.yaml and 
ECR image ready by EOD today, PR Monday morning."
```

---

### 8:30 AM — Derek Canary

```
Derek (8:30 AM):
"stockCache canary at 5%. Looking clean so far.

Cache stats at startup:
  Products cached: 43,891/43,891 (100% warm)
  Cache memory: 4.1 MB
  Warm-up time: 2.3 seconds (async, didn't block readiness)
  
First 30 minutes:
  Tier 1 (live inventory check): 2,847 requests
  Tier 2 (cached stock): 0 (CB hasn't opened)
  Tier 3 (blind accept): 0
  Cache hit rate: N/A (cache only used when CB opens)
  
Moving to 25% at 8:45."
```

---

### 9:07 AM — PagerDuty

Your phone buzzes.

```
🔴 PagerDuty: HighDNSErrorRate
   Source: CoreDNS
   Namespace: kube-system
   Severity: warning
   Message: DNS NXDOMAIN rate >5% for 3 minutes
   
   Current NXDOMAIN rate: 8.2%
   Affected: cluster-wide
   Started: 9:04 AM
```

Simultaneously, Slack starts lighting up:

```
Ryan Mitchell (9:07 AM) — #frontend:
"API calls from the frontend are intermittently failing. 
Getting connection errors to order-service and 
inventory-service. Not every request — maybe 1 in 10. 
Started about 3 minutes ago."

David Okafor (9:08 AM) — #payment-engineering:
"Seeing intermittent 503s from payment gateway calls. 
Stripe webhooks are also failing to resolve. Stripe 
endpoint: api.stripe.com — DNS resolution failing 
intermittently."

Priya (9:08 AM) — #platform-engineering:
"CoreDNS alert. I'm looking at it. Two CoreDNS pods 
in kube-system — one looks healthy, one is restarting."

Lisa (9:09 AM) — #incidents:
"Error rates climbing across all services:
  order-service: 3.2% (baseline 0.03%)
  payment-service: 1.8% (baseline 0.01%)
  inventory-service: 2.1% (baseline 0.02%)
  search-service: 4.7% (baseline 0.1%)
  
  Pattern: intermittent, affecting all services. 
  This looks like infrastructure, not application."
```

---

**9:09 AM. Your first DNS incident. The category with zero coverage until now.**

The symptoms are classic: intermittent failures across all services simultaneously, NXDOMAIN responses, one CoreDNS pod unhealthy. This isn't an application problem — it's name resolution.

But there are at least three possible root causes, and the Chaos Engine is active. The obvious answer (CoreDNS pod restart) might not be the whole story.

```
WHAT YOU KNOW:
  - CoreDNS NXDOMAIN rate: 8.2% (>5% threshold)
  - One of two CoreDNS pods restarting
  - All services affected intermittently (~1 in 10 requests)
  - External DNS also failing (Stripe api.stripe.com)
  - Started ~9:04 AM
  
WHAT YOU DON'T KNOW:
  - Why the CoreDNS pod is restarting
  - Whether both internal and external resolution are failing
  - Whether this is a pod issue or an upstream resolver issue
  - Whether any recent changes could have caused this
  - How Derek's stockCache canary is affected (inventory lookups)
```

**ACK PagerDuty (9:09 AM).** Clock started.

---

**Parallel triage — three terminals, one Slack window (9:09 AM):**

**Terminal 1 — CoreDNS pod status:**
```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
```

```
NAME                       READY   STATUS             RESTARTS   AGE    NODE
coredns-5d78c9869d-4xmpt   1/1     Running            0          12d    ip-10-0-41-15
coredns-5d78c9869d-9rvkz   0/1     CrashLoopBackOff   3          4m     ip-10-0-42-44
```

One pod CrashLoopBackOff. Three restarts in 4 minutes. The other pod is healthy — it's carrying all DNS traffic solo.

**Terminal 2 — why is it crashing:**
```bash
kubectl -n kube-system describe pod coredns-5d78c9869d-9rvkz | tail -20
```

```
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Fri, 26 Jan 2024 09:04:12
      Finished:     Fri, 26 Jan 2024 09:04:47
    Ready:          False
    Restart Count:  3
    Limits:
      cpu:     200m
      memory:  170Mi
    Requests:
      cpu:     100m
      memory:  70Mi
```

**OOMKilled at 170Mi limit.** CoreDNS is dying from memory pressure. The limit is 170Mi — that's the EKS default, and it's tight for a cluster this size.

**Terminal 3 — what's the surviving pod's memory:**
```bash
kubectl -n kube-system top pod coredns-5d78c9869d-4xmpt
```

```
NAME                        CPU(cores)   MEMORY(bytes)
coredns-5d78c9869d-4xmpt    189m         161Mi
```

**161Mi out of 170Mi limit.** The surviving pod is at 94.7% memory. It's about to OOM too. If it dies, we lose ALL cluster DNS. Every service goes down.

**This is not a "fix the crashed pod" situation. This is a "prevent total DNS failure in the next few minutes" situation.**

---

**#incidents post (9:10 AM) — 1 minute from ack:**

```
[User] (9:10 AM):
ACKed. CoreDNS incident. One of two CoreDNS pods OOMKilled. 
Surviving pod at 161/170Mi — minutes from OOM itself.

If the second pod dies: TOTAL cluster DNS failure. Every 
service goes down simultaneously.

IMMEDIATE: increasing CoreDNS memory limits to prevent 
second pod OOM. Investigating root cause of memory spike 
in parallel.

SEV2 — cluster-wide DNS degradation with risk of SEV1 
cascade.
```

---

**Immediate fix — save the surviving pod (9:11 AM):**

I can't just edit the pod — it's managed by a Deployment. I need to patch the Deployment's memory limits. This will trigger a rolling restart, but the critical thing is the surviving pod stays alive while the crashed one comes back with higher limits.

```bash
kubectl -n kube-system patch deployment coredns -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "coredns",
          "resources": {
            "limits": {"memory": "512Mi"},
            "requests": {"memory": "256Mi"}
          }
        }]
      }
    }
  }
}'
```

```
deployment.apps/coredns patched
```

The rolling update strategy will bring up a new pod with 512Mi before terminating the old one. The crashed pod will also be recreated with the new limits.

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -w
```

```
coredns-5d78c9869d-4xmpt   1/1     Running            0          12d
coredns-5d78c9869d-9rvkz   0/1     CrashLoopBackOff   3          5m
coredns-7f89b6c4d2-lq8mn   0/1     ContainerCreating  0          3s
```

New pod spinning up with 512Mi limit. Old pods will drain once the new one is ready.

**While that's happening — what caused the OOM? (9:12 AM)**

```bash
# CoreDNS logs from the surviving pod — what's it processing?
kubectl -n kube-system logs coredns-5d78c9869d-4xmpt --tail=100 | head -50
```

```
[INFO] 10.0.42.198:47382 - 29488 "A IN inventory-service.production.svc.cluster.local.production.svc.cluster.local. udp 83 false 512" NXDOMAIN qr,aa,rd 176 0.000089s
[INFO] 10.0.42.198:47382 - 29489 "A IN inventory-service.production.svc.cluster.local.svc.cluster.local. udp 77 false 512" NXDOMAIN qr,aa,rd 170 0.000076s
[INFO] 10.0.42.198:47382 - 29490 "A IN inventory-service.production.svc.cluster.local.cluster.local. udp 69 false 512" NXDOMAIN qr,aa,rd 162 0.000082s
[INFO] 10.0.42.198:47382 - 29491 "A IN inventory-service.production.svc.cluster.local.us-east-1.compute.internal. udp 82 false 512" NXDOMAIN qr,aa,rd 175 0.000091s
[INFO] 10.0.42.198:47382 - 29492 "A IN inventory-service.production.svc.cluster.local. udp 55 false 512" NOERROR qr,aa,rd 119 0.000045s
[INFO] 10.0.42.199:51003 - 29493 "A IN inventory-service.production.svc.cluster.local.production.svc.cluster.local. udp 83 false 512" NXDOMAIN qr,aa,rd 176 0.000089s
[INFO] 10.0.42.199:51003 - 29494 "A IN inventory-service.production.svc.cluster.local.svc.cluster.local. udp 77 false 512" NXDOMAIN qr,aa,rd 170 0.000076s
... [thousands of identical lines]
```

**There it is.**

Every lookup for `inventory-service.production.svc.cluster.local` is generating **five DNS queries** — four NXDOMAINs through the search path before the fifth succeeds. That's the `ndots:5` default.

But why inventory-service? And why NOW? These lookups have been happening since forever...

**Check the source IP (9:13 AM):**

```bash
# What pod is 10.0.42.198?
kubectl get pods -A -o wide | grep 10.0.42.198
```

```
production   order-service-cb-canary-7f4d2-xk9mz   2/2   Running   0   42m   10.0.42.198
```

```bash
kubectl get pods -A -o wide | grep 10.0.42.199
```

```
production   order-service-cb-canary-7f4d2-rm4qp   2/2   Running   0   42m   10.0.42.199
```

**Derek's stockCache canary pods.** The ones that scaled to 25% at 8:45 AM.

I check the timeline: Derek went from 5% (1 pod) to 25% (4 pods) at 8:45. Each pod's stockCache warms up by fetching all 43,891 products from inventory-service on startup. With `ndots:5`, each lookup generates 5 DNS queries. Four pods starting simultaneously:

```
4 pods × 43,891 products × 5 DNS queries = 877,820 DNS queries
In ~2.3 seconds = ~381,000 queries/second
```

Normal cluster DNS load is maybe 2,000-3,000 queries per second. The stockCache startup created a **125x spike** in DNS traffic. CoreDNS pod-2 OOM'd from the cache entries for all those queries. Pod-1 survived but is at 94.7% memory from handling the overflow.

**Root cause: stockCache warm-on-startup + ndots:5 query amplification + undersized CoreDNS memory limits.**

**Rule 1: Consequence branching.** The stockCache was a good decision (improve CB fallback). The warm-on-startup was a good decision (ensure cache is populated). But combined with ndots:5 and simultaneous pod startups during canary scaling, it created a DNS storm.

---

**Comms update (9:14 AM) — 5 minutes from ack:**

```
[User] (9:14 AM):
Root cause identified. 5 minutes.

Derek's stockCache canary scaled to 25% (4 pods) at 8:45. 
Each pod warms its cache by looking up 43,891 products 
from inventory-service on startup. With Kubernetes ndots:5 
default, each lookup generates 5 DNS queries (4 NXDOMAIN 
search path + 1 success).

4 pods × 43,891 × 5 = 877,820 DNS queries in ~2 seconds.
Normal load: ~3,000/sec. Spike: ~380,000/sec.

CoreDNS pod-2 OOMKilled (170Mi limit — EKS default, undersized).
CoreDNS pod-1 at 161/170Mi — minutes from cascading.

FIX IN PROGRESS:
  1. ✅ CoreDNS memory limit increased to 512Mi (patching now)
  2. ⏳ Hold Derek's canary at 25% (no further scaling)
  3. After DNS stabilizes: fix the ndots amplification in 
     stockCache pod spec

@Derek — HOLD at 25%. Do not scale further. The stockCache 
startup is causing a DNS storm. I'll explain in a minute.
```

```
Derek (9:14 AM):
"Holding. Was about to go to 50%. Stopped."
```

---

**Check CoreDNS recovery (9:15 AM):**

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

```
NAME                        READY   STATUS    RESTARTS   AGE
coredns-7f89b6c4d2-lq8mn   1/1     Running   0          78s
coredns-7f89b6c4d2-nt5px   1/1     Running   0          45s
```

Both new pods running with 512Mi limits. Old pods terminated.

```bash
kubectl -n kube-system top pod -l k8s-app=kube-dns
```

```
NAME                        CPU(cores)   MEMORY(bytes)
coredns-7f89b6c4d2-lq8mn   42m          89Mi
coredns-7f89b6c4d2-nt5px   38m          84Mi
```

**89Mi and 84Mi — healthy.** The DNS storm has passed (the stockCache pods finished warming up). Normal query load is well within 512Mi.

**Check service error rates (9:16 AM):**

```
Lisa (9:16 AM):
"Error rates dropping:
  order-service: 0.4% (was 3.2%) ↓
  payment-service: 0.1% (was 1.8%) ↓
  inventory-service: 0.2% (was 2.1%) ↓
  search-service: 0.3% (was 4.7%) ↓
  
  Recovering. Should be baseline within 2-3 minutes."
```

```
Ryan (9:17 AM):
"Frontend errors stopped. Everything looks normal again."
```

```
David Okafor (9:17 AM):
"Stripe webhook processing recovered. Payment flow clean."
```

---

**Now fix the root cause so this doesn't repeat when Derek scales further (9:17 AM):**

The problem is twofold:
1. CoreDNS memory limits were too low (fixed — 512Mi)
2. The stockCache startup generates a massive DNS burst because of ndots:5 query amplification

The permanent fix for #2 is adding a `dnsConfig` to the order-service pod spec that either:
- Uses FQDNs with trailing dots (skips search path entirely)
- Reduces ndots to 2 (reduces but doesn't eliminate search queries)
- Or — best option — the stockCache code should resolve inventory-service using the FQDN: `inventory-service.production.svc.cluster.local.` (with trailing dot)

But there's a faster tactical fix: stagger the cache warm-up across pods.

DM to Derek:

> Your stockCache warm-on-startup is the trigger. Here's what happened:
>
> Each pod resolves `inventory-service` 43,891 times on startup. Kubernetes ndots:5 means each resolution generates 5 DNS queries (4 NXDOMAIN attempts through the search path, then the real lookup). One pod warming up = 219K DNS queries in 2 seconds. Manageable.
>
> When you scaled to 25% at 8:45, FOUR pods started simultaneously. 877K DNS queries in 2 seconds. CoreDNS OOM'd.
>
> Two fixes needed. Do both:
>
> **Fix 1 (code change, today):** Add a random startup delay to the cache warm-up. `Thread.sleep(random(0, 30_000))` before the preload starts. This staggers warm-ups across pods so they don't all hit DNS simultaneously. Simple, effective, no architecture change.
>
> **Fix 2 (pod spec, today):** Add `dnsConfig` to the order-service Deployment:
> ```yaml
> dnsConfig:
>   options:
>   - name: ndots
>     value: "2"
> ```
> This reduces DNS search path queries. `inventory-service.production.svc.cluster.local` has 4 dots — with ndots:2, it resolves directly on the first try instead of going through the search path. Eliminates the 5x query amplification.
>
> **Why ndots:2 and not ndots:1:** We want short service names (`inventory-service`) to still use the search path for convenience. ndots:2 means names with 2+ dots resolve directly (all FQDN-style names), while names with 0-1 dots still search. Best of both worlds.
>
> Fix 1 prevents the burst. Fix 2 prevents the amplification. Together they reduce the DNS load from 877K queries to about 12K queries — a 73x reduction. CoreDNS won't even notice.
>
> After both fixes are in, you can resume scaling. But let me verify DNS is stable first.

```
Derek (9:20 AM):
"God. I never even thought about DNS. The startup looked 
fine in staging because there's only one pod and the 
inventory stub responds instantly.

Adding the jitter now. And the ndots — should I add that 
to just the order-service deployment or cluster-wide?"
```

DM to Derek:

> Just order-service for now. Cluster-wide ndots change is a bigger conversation — it could affect services that depend on the default search behavior. But you've identified something worth a broader fix. I'll open a ticket for cluster-wide ndots assessment.
>
> For right now: jitter + ndots:2 on order-service. PR it, I'll fast-track review.

---

**Verify everything is recovered (9:21 AM):**

```bash
# DNS resolution test from a pod
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup inventory-service.production.svc.cluster.local
```

```
Server:    172.20.0.10
Address:   172.20.0.10:53

Name:      inventory-service.production.svc.cluster.local
Address:   172.20.148.32
```

Clean resolution. No NXDOMAIN.

```bash
# External resolution test
kubectl run dns-test2 --image=busybox --rm -it --restart=Never -- \
  nslookup api.stripe.com
```

```
Server:    172.20.0.10
Address:   172.20.0.10:53

Non-authoritative answer:
Name:      api.stripe.com
Address:   52.4.xxx.xxx
```

External DNS also clean.

```bash
# CoreDNS metrics
kubectl -n kube-system exec coredns-7f89b6c4d2-lq8mn -- \
  wget -qO- http://localhost:9153/metrics | grep coredns_dns_requests_total
```

Query rate back to normal (~50/sec per pod). No errors.

---

**Red herring check (9:22 AM) — Rule 3:**

Priya messaged earlier that she was looking at CoreDNS. Let me check what she found:

```
Priya (9:12 AM):
"CoreDNS investigation: I see the OOM on pod-2. Also 
noticed the CoreDNS configmap was edited 2 days ago:

  kubectl -n kube-system get cm coredns -o yaml

  Last applied: Wed Jan 24 by marcus.webb

  Diff from default: added a forward zone for 
  'internal.novamart.com' pointing to 10.0.0.2 
  (VPC DNS resolver). This was for resolving the 
  internal ALB names during the AWS audit.

Could the configmap change be related?"
```

DM to Priya:

> Good investigation. The configmap change is NOT the cause — it's been running for 2 days without issues. The forward zone just adds a conditional forwarding rule which doesn't affect memory usage or query processing. Confirmed root cause is DNS query storm from stockCache startup (see #incidents).
>
> But good instinct checking recent changes. That's the right first step.

---

**#incidents resolution (9:23 AM):**

```
[User] (9:23 AM):
DNS incident RESOLVED. 14 minutes from ack.

TIMELINE:
  9:04  CoreDNS pod-2 OOMKilled
  9:07  Alert fires (NXDOMAIN rate >5%)
  9:09  ACKed
  9:11  CoreDNS memory limits patched (170Mi → 512Mi)
  9:14  Root cause identified: stockCache DNS storm
  9:15  New CoreDNS pods running, healthy
  9:16  Error rates dropping across all services
  9:21  DNS fully clean — internal and external
  9:23  Resolved

ROOT CAUSE:
  stockCache canary scale to 25% (4 pods) triggered 
  simultaneous cache warm-up: 877,820 DNS queries in 
  ~2 seconds (43,891 products × 5 ndots search queries 
  × 4 pods). CoreDNS memory limit was 170Mi (EKS default, 
  undersized). Pod OOM'd. Surviving pod at 94.7% memory.

  CONSEQUENCE CHAIN:
  Incident 1 (timeout) → circuit breaker → stockCache → 
  DNS storm. Every fix was individually correct. The 
  combination under simultaneous startup created the issue.

FIXES:
  ✅ CoreDNS memory: 170Mi → 512Mi (both pods)
  ⏳ stockCache startup jitter (Derek, today) — prevents burst
  ⏳ order-service ndots:2 (Derek, today) — prevents amplification
  
BUDGET IMPACT:
  ~12 minutes of elevated errors across all services.
  order-service: peak 3.2%, recovered within 5 min of fix
  Estimated budget burn: <0.5%

FOLLOW-UP:
  - Cluster-wide ndots assessment (ticket needed)
  - CoreDNS resource limit review for cluster size
  - Consider CoreDNS autoscaling (proportional to node count)
  - stockCache scaling protocol: stagger pod startups during canary

@Derek — resume canary AFTER jitter + ndots fixes are in. 
Not before.
```

---

**Post-incident thread cleanup (9:25 AM):**

```
Lisa (9:24 AM):
"All services baseline. Budget impact:
  order-service: 61.2% → 60.9% (0.3% consumed)
  payment-service: 86.0% → 85.9% (0.1% consumed)
  
  Minimal. The 512Mi fix was fast enough."
```

**DM to Lisa:**

> Add CoreDNS memory and query rate to the infrastructure monitoring dashboard. We should have an alert for CoreDNS memory >80% — that would have given us a 2-3 minute early warning before the OOM. The NXDOMAIN rate alert caught it, but a memory alert would have been earlier.

```
Lisa (9:26 AM):
"On it. Adding coredns_memory_usage_percent > 80% warning, 
> 90% critical. Also adding coredns_dns_requests_total 
rate-of-change alert — if query rate spikes >10x baseline 
in 1 minute, that's abnormal regardless of memory."
```

**DM to Derek:**

> One more thing. When you add the startup jitter, also add a log line: `"stockCache warm-up starting after {}ms jitter delay, {} products to fetch"`. And a metric: `stock_cache_warmup_duration_seconds` and `stock_cache_warmup_dns_queries_total`. If this ever causes problems again, we want to see it in the dashboard immediately.
>
> The ndots:2 change should be a separate commit from the jitter — different concerns, different review scope. The jitter is a quick fix. The ndots is a pod spec change that I want to apply thoughtfully.

```
Derek (9:30 AM):
"Both PRs up.

platform-infra#355: stockCache startup jitter (0-30s random delay)
  + warmup logging + warmup metrics

platform-infra#356: order-service dnsConfig ndots:2
  + comment explaining why

Requesting fast-track review."
```

I review both PRs immediately. #355 is clean — the jitter is `ThreadLocalRandom.current().nextInt(30_000)`, the logging and metrics are in the right places. #356 is a two-line YAML change with a good comment block explaining the ndots:5 amplification.

```
[User] (9:35 AM):
Both PRs approved and merged.
  #355: stockCache startup jitter ✅
  #356: order-service ndots:2 ✅

ArgoCD syncing. Derek — once the new pods are running 
with both fixes, you can resume scaling. Start at 25% 
again (the existing pods won't re-warm since the cache 
is already populated). Scale to 50% and watch CoreDNS 
memory. If it stays under 150Mi through 50%, you're 
clear to continue the normal protocol.

And Derek — this wasn't your bug. The stockCache design 
is correct. The warm-on-startup is correct. The gap was 
that neither of us thought about DNS query amplification 
during code review. I reviewed the PR and approved it 
without considering ndots impact. That's on both of us. 
The jitter + ndots fix makes it production-safe.
```

```
Derek (9:37 AM):
"New pods rolling out with both fixes. Cache warm-up 
staggered beautifully — pod 1 at 0s, pod 2 at 17s, 
pod 3 at 8s, pod 4 at 24s. CoreDNS didn't even twitch.

Resuming canary at 25%. Will scale through the rest 
of the protocol and report."
```

---

**Personal log entry (9:40 AM, mental note):**

```
Incident 11 — DNS/CoreDNS OOM from stockCache.

What went right:
  - Recognized surviving pod at 94.7% as imminent cascade risk
  - Fixed memory limit BEFORE investigating root cause 
    (stabilize first, diagnose second)
  - Root cause in 5 minutes (CoreDNS logs showed the 
    query pattern immediately)
  - Traced to stockCache via source IP correlation
  - Total resolution: 14 minutes
  - Budget impact: minimal (0.3% order, 0.1% payment)

What I missed:
  - During stockCache PR review, I didn't consider DNS 
    amplification from ndots:5. 43K lookups seemed fine. 
    43K × 5 × N pods simultaneously is not fine. 
    DNS impact of bulk resolution patterns should be part 
    of PR review mental model for any service that does 
    batch lookups on startup.

Broader lesson:
  ndots:5 is a cluster-wide default that silently 5x's 
  DNS load for any service using short names. It's a 
  known footgun. We need a cluster-wide policy on this. 
  Options: reduce default ndots, add FQDN trailing dots 
  to service configs, or deploy NodeLocal DNSCache to 
  absorb the load locally.

Consequence chain is now 5 links long:
  Incident 1 (800ms timeout) 
  → circuit breaker (fix) 
  → stockCache (improvement) 
  → DNS storm (consequence)
  → ndots:2 + jitter (fix)
  
  Every link was a correct decision. The system behavior 
  emerged from their interaction. This is why chaos 
  engineering matters.
```

---

### 9:45 AM — Back to Friday

DNS incident resolved. Fourteen minutes. Minimal budget impact. Two PRs merged. Derek resuming canary.

```
FRIDAY STATUS — 9:45 AM:

✅ R-6a milestone: CLOSED (8:00 AM)
✅ DNS incident: RESOLVED (9:23 AM, 14 min TTR)
⏳ Derek stockCache: canary resumed with fixes, scaling through protocol
⏳ Jake: FluentBit ECR migration scoping (values.yaml + image today)
⏳ FBI CFAA referral: Rachel/counsel submitting today

Remaining today:
  - Monitor Derek's canary through 100%
  - Sprint retrospective / next sprint planning
  - Jake FluentBit deliverables
  - Tie off any loose ends before weekend
```

The morning's first incident is done. Clean diagnosis, fast fix, consequence chain documented. Eleven incidents in, first DNS incident handled.

**The week is closing well.**

---

*That's Incident 11. DNS — CoreDNS OOM from ndots:5 query amplification during stockCache bulk startup. Category: Networking — DNS. The first of five planned DNS incidents.*

*Derek's canary continues. The day continues. Want to keep going through Friday, or shall we move on?*

**PagerDuty is waiting for acknowledgment. Error rates are climbing. The clock is running.**

**What do you do?**
