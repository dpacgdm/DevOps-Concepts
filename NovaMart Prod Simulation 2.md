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

## FRIDAY Wk2 — REMAINDER (SUMMARY)

Derek's stockCache canary completed clean — 100% production by 2:15 PM. Zero CoreDNS impact with the jitter + ndots fixes. Three-tier fallback confirmed working in production: 99.97% tier 1 (live), 0.03% tier 2 (cached), 0% tier 3 (blind). Jake finished FluentBit reverse-engineering — values.yaml and ECR image ready, PR Monday. FBI CFAA referral submitted by outside counsel at 3:30 PM.

Sprint retrospective at 4:00 PM. Quiet. The team is tired but sharp. Tom formally presented R-6a close. Priya presented ILM results. You walked through the incident count and budget status. Sarah noted the board's 30-day follow-up is February 22.

Weekend: no pages. First clean weekend of the simulation.

---

## MONDAY 8:15 AM — WEEK 3, DAY 14

---

You arrive to a normal-looking Monday. Coffee, Slack scan, overnight messages.

```
OVERNIGHT / WEEKEND:

Sat — Tom: R-6b ArgoCD Helm chart draft PR (self-initiated,
  as promised). Looks substantial — handles StatefulSet 
  ordering with init containers.

Sun — Nothing.

Mon 7:30 AM — Jake: "FluentBit PR up: platform-infra#358. 
  ArgoCD Application + values.yaml + ECR image. Ready for 
  review. Deploying Tuesday if approved — gives us 3 days 
  buffer before Feb 2 enforce mode."

Mon 7:45 AM — Priya: "Search ArgoCD onboarding started. 
  Helm chart extracted from the cluster, values reconciled. 
  PR by Wednesday. Also started SLO draft with Lisa — 
  proposing 99.5% availability, p99 < 1.5s."

Mon 7:50 AM — Marcus: "Back from a real weekend. Feels 
  weird. Starting on the SA-2024-0119-NM response materials 
  today — drift detection documentation, operational evidence, 
  remediation timeline for Rachel."

Mon 8:00 AM — Derek: "stockCache 48h production metrics:
    Tier 1: 99.97%
    Tier 2: 0.03% (22 requests — brief inventory-service 
            GC pause Saturday 3 AM triggered CB for 4 seconds)
    Tier 3: 0%
    Cache staleness on tier 2 responses: avg 12 seconds
    
  The CB actually opened briefly on Saturday. The stockCache 
  caught it perfectly — 22 orders got cached stock levels 
  instead of blind accept. All 22 were accurate (verified 
  against inventory-service after recovery).
  
  This is working exactly as designed."

Mon 8:10 AM — Sarah: "Monday standup at 9:30. James won't 
  be there — he's at an all-day board follow-up offsite. 
  Normal standup, no drama expected.
  
  Also: two headcount recs posted Friday. Job descriptions 
  for 'Senior Platform Engineer' and 'Platform Engineer — 
  Security Focus.' Recruiting starts this week. James wants 
  your input on the job descriptions by Wednesday."

Mon 8:12 AM — Rachel: "SA-2024-0119-NM response draft 
  started. I'll need your technical sections by Wednesday:
    1. Drift detection architecture + test evidence
    2. Complete unmanaged infrastructure inventory
    3. Remediation timeline with current status
  Marcus is helping with #2. Please own #1 and #3."
```

---

Normal Monday. Planned work resuming. The security incident has settled into background legal/regulatory process. The team is executing on committed deliverables.

You review Jake's FluentBit PR (#358), approve Tom's R-6b Helm chart draft with comments on the init container ordering, and start drafting the drift detection documentation for Rachel.

---

### 10:45 AM — Standup Ends, Slack Lights Up

Standup was clean. Sprint goals reviewed, no blockers reported. Then:

```
Nina Petrov (10:42 AM) — #order-engineering:
"We have a problem. I just deployed the order-service 
inventory timeout adjustment (PLAT-917 — the p99 
investigation) and something is wrong.

The deploy went through ArgoCD normally. Canary passed. 
Promoted to 100%. But now I'm seeing orders failing with 
a new error I haven't seen before:

  'CircuitBreakerConfigurationException: Conflicting 
   configuration for circuit breaker instance 
   inventory-client. Property slowCallDurationThreshold 
   defined in both application.yml (2000ms) and 
   resilience4j-overrides.yml (1500ms).'

The order-service pods are crash-looping. ArgoCD shows 
the deploy as 'Healthy' because the readiness probe 
hasn't failed yet — it's an HTTP check on /health which 
returns 200 even though the circuit breaker bean fails 
to initialize on first request.

I'm seeing 100% failure rate on orders that hit the 
inventory check path. Orders that don't need inventory 
(digital products) are fine.

I didn't touch the circuit breaker config. I only changed 
the inventory client timeout from 2000ms to 1800ms based 
on the p99 analysis I've been doing for PLAT-917. But my 
values are in application.yml and Derek's circuit breaker 
values are in a different file?"
```

```
Derek (10:44 AM) — #order-engineering:
"Wait. My circuit breaker config is in 
resilience4j-overrides.yml — that was a design decision 
from the PR review. It's a separate config file that gets 
merged at startup. It has slowCallDurationThreshold: 2000ms.

Nina, did your change also set slowCallDurationThreshold? 
Or just the HTTP client timeout?"

Nina (10:45 AM):
"I set inventory_client_timeout_ms: 1800ms in application.yml. 
But I ALSO set slowCallDurationThreshold: 1500ms because 
the circuit breaker slow-call threshold should be lower 
than the HTTP timeout — otherwise you'll never detect slow 
calls. That was in the PLAT-917 ticket description.

I didn't realize there was already a slowCallDurationThreshold 
in a different file. It's not in application.yml where I 
was looking. Where did resilience4j-overrides.yml come from?"

Derek (10:46 AM):
"It was part of my CB PR (platform-infra#347). The 
overrides file was a clean separation — CB config in one 
file, application config in another. It passed review."
```

```
Lisa (10:47 AM) — #incidents:
"Order-service error rate:
  Physical product orders: 100% failure
  Digital product orders: 0% failure (no inventory check)
  Overall error rate: 67% (2/3 of orders are physical)
  
  Budget: 60.9% → burning fast.
  At 67% error rate we're burning ~4% per hour.
  
  This is SEV1 territory."
```

---

### 10:47 AM

**Two engineers' config changes colliding in production.** Nina didn't know about Derek's override file. Derek didn't know Nina was also setting the same property. The canary passed because the crash only happens on the first inventory-check request — the canary's synthetic health check didn't exercise that path.

**67% order failure rate. Physical products completely down. Budget burning at 4% per hour.**

**Clock is running. What do you do?**



**ACK PagerDuty (10:47 AM).** Wait — PagerDuty hasn't fired yet. Lisa caught this before alerting did. The error rate alert threshold is 10% for 5 minutes. We're at 67% but it's been less than 5 minutes since the deploy promoted to 100%.

Doesn't matter. Clock is running regardless.

---

**#incidents (10:47 AM):**

```
[User] (10:47 AM):
SEV1 — Order service. Physical product orders 100% failure.
67% overall error rate. CB config collision between Nina's 
PLAT-917 deploy and Derek's resilience4j-overrides.yml.

ROLLING BACK NOW. Then we fix.

@Nina — initiate rollback to previous revision immediately:
  argocd app rollback order-service

@Derek — stand by. This is a config merge conflict, not a 
code bug. We'll sort the config after rollback.

@Lisa — budget clock: how long has the 67% rate been active?
```

---

**Terminal 1 — Verify current state and rollback (10:48 AM):**

```bash
# Current order-service revision
argocd app get order-service --show-operation

# Pod status
kubectl -n production get pods -l app=order-service
```

```
NAME                              READY   STATUS    RESTARTS   AGE
order-service-7f89b6c4d2-abc12   2/2     Running   0          8m
order-service-7f89b6c4d2-def34   2/2     Running   0          8m
order-service-7f89b6c4d2-ghi56   2/2     Running   0          8m
order-service-7f89b6c4d2-jkl78   2/2     Running   0          8m
order-service-7f89b6c4d2-mno90   2/2     Running   0          8m
order-service-7f89b6c4d2-pqr12   2/2     Running   0          8m
```

Six pods, all "Running" — they're not crash-looping at the pod level. The error is a **runtime exception** on the inventory check path, not a startup crash. That's why the readiness probe passes and ArgoCD shows Healthy.

```
Nina (10:48 AM):
"Rolling back. ArgoCD rollback initiated.

...

Rollback in progress. Previous revision: sha-f8a91c3 
(Derek's stockCache + jitter deploy from Friday). 
New pods coming up."
```

```bash
kubectl -n production get pods -l app=order-service -w
```

```
order-service-6c4d2a8e-xxx01   0/2     ContainerCreating   0     3s
order-service-6c4d2a8e-xxx02   0/2     ContainerCreating   0     2s
order-service-6c4d2a8e-xxx03   0/2     ContainerCreating   0     1s
```

Rolling update. Old pods serving (with errors) while new pods come up.

```
Lisa (10:49 AM):
"Error rate timeline:
  10:38 AM — Nina's deploy promoted to 100%
  10:39 AM — error rate starts climbing
  10:42 AM — Nina reports in Slack
  10:47 AM — 67% error rate confirmed
  
  9 minutes at elevated error rate. First 3 minutes were 
  canary (low traffic). Full blast since 10:41.
  
  6 minutes at 67% error rate on full traffic.
  Budget impact: ~0.4% consumed per minute at this rate.
  Estimated total burn: 2.4% if we recover in the next minute.
  
  Order-service budget: 60.9% → ~58.5% (estimate)."
```

**2.4% budget burn in 6 minutes. Painful but not catastrophic — IF we recover now.**

---

**Rollback completing (10:50 AM):**

```bash
kubectl -n production get pods -l app=order-service
```

```
NAME                              READY   STATUS        RESTARTS   AGE
order-service-6c4d2a8e-xxx01     2/2     Running       0          62s
order-service-6c4d2a8e-xxx02     2/2     Running       0          61s
order-service-6c4d2a8e-xxx03     2/2     Running       0          60s
order-service-6c4d2a8e-xxx04     2/2     Running       0          55s
order-service-6c4d2a8e-xxx05     2/2     Running       0          53s
order-service-6c4d2a8e-xxx06     2/2     Running       0          50s
order-service-7f89b6c4d2-abc12   2/2     Terminating   0          10m
order-service-7f89b6c4d2-def34   2/2     Terminating   0          10m
```

New pods from the previous revision running. Old pods terminating.

```bash
# Quick sanity check — test an inventory-path order
kubectl -n production exec order-service-6c4d2a8e-xxx01 -- \
  curl -s localhost:8080/actuator/health | jq .
```

```json
{
  "status": "UP",
  "components": {
    "circuitBreaker": {
      "status": "UP",
      "details": {
        "inventory-client": {
          "state": "CLOSED",
          "failureRate": "0.0%"
        }
      }
    }
  }
}
```

Circuit breaker healthy. CLOSED state. No failures.

```
Lisa (10:51 AM):
"Error rate dropping:
  10:50 AM — 34% (old pods draining)
  10:51 AM — 8.2% (mostly recovered)
  10:51:30 — 0.12%
  
  Baseline. We're clean.
  
  Final budget impact: order-service 60.9% → 58.1% (2.8% consumed)
  
  That HURTS. We're at 58.1% with ~5 days left in the window.
  One more incident of any meaningful duration and we're 
  in freeze territory."
```

---

**#incidents update (10:52 AM):**

```
[User] (10:52 AM):
ROLLBACK COMPLETE. Order-service recovered. Error rate baseline.
TTR: 5 minutes from ack to recovery.

Budget: 60.9% → 58.1% (2.8% consumed in ~9 minutes of exposure)

Root cause clear. NOT a code bug. Config file merge collision:
  - Derek's CB config: resilience4j-overrides.yml 
    → slowCallDurationThreshold: 2000ms
  - Nina's PLAT-917: application.yml 
    → slowCallDurationThreshold: 1500ms
  - Spring Boot merged both files at startup
  - Resilience4j threw ConfigurationException on conflicting values
  - Exception only triggers on first inventory-check request
  - Health probe (/health) doesn't exercise inventory path
  - Canary passed because synthetic checks don't create real orders

Post-rollback: investigating config merge behavior and 
canary gap. Nobody deploys to order-service until this 
is resolved.
```

---

**Now — the real work. Understanding how this happened and preventing it. (10:53 AM)**

DM to Nina and Derek (group):

> Both of you stop. Nobody is in trouble. This is a process failure, not a people failure. Let me explain what happened.
>
> The order-service has two config files that both define circuit breaker properties:
> - `application.yml` — the main Spring Boot config that Nina (and most developers) knows about
> - `resilience4j-overrides.yml` — a separate config file Derek created during the CB implementation to keep CB config isolated
>
> Both files are valid individually. Spring Boot merges them at startup. When they define the SAME property (`slowCallDurationThreshold`), Resilience4j detects the conflict and throws a `CircuitBreakerConfigurationException`.
>
> **Derek:** when you created the overrides file during the CB PR, did you document its existence anywhere? In the README, the deploy guide, or the application config comments?
>
> **Nina:** when you were investigating PLAT-917, did you search the entire codebase for `slowCallDurationThreshold` before adding yours?

```
Derek (10:55 AM):
"Honestly? No. I documented it in the PR description and 
the code review comments. I didn't add a README note or a 
comment in application.yml pointing to the overrides file.

I assumed anyone changing CB config would see the overrides 
file in the same directory. But Nina wasn't changing CB 
config — she was changing inventory timeout config. She 
had no reason to look for a separate CB file."

Nina (10:56 AM):
"I searched application.yml for 'slowCall' and got zero 
results. I assumed it wasn't configured and added it as 
part of my timeout adjustment. I didn't know to search 
other files because I didn't know other config files 
existed for this service.

I also didn't review Derek's CB PR because it merged 
while I was on PTO. When I came back, I saw the CB was 
deployed but I didn't review the implementation details."
```

---

This is a textbook configuration management problem with two contributing factors:

**Factor 1: Discoverability.** Derek's override file was invisible to Nina because it wasn't referenced from the main config and wasn't documented outside the PR.

**Factor 2: Canary gap.** The canary's health check doesn't exercise the inventory path. The failure mode is a runtime exception on a specific code path, not a startup crash or health check failure.

Both are fixable.

DM to Nina and Derek (group):

> Here's what we're going to do. Three things, in order:
>
> **1. Merge the config files (today, Nina + Derek together).**
> There should be ONE source of truth for order-service configuration. The resilience4j-overrides.yml was well-intentioned separation of concerns, but in practice it created a hidden config surface that other developers can't discover. Merge everything into application.yml with clear section headers:
> ```yaml
> # === Circuit Breaker Configuration ===
> # Owner: Derek Huang (PLAT-915)
> # WARNING: Changes require CB team review
> resilience4j:
>   circuitbreaker:
>     instances:
>       inventory-client:
>         slowCallDurationThreshold: 1800ms  # Nina's value — see below
>         ...
>
> # === Inventory Client Configuration ===
> # Owner: Nina Petrov (PLAT-917)
> inventory:
>   client:
>     timeout_ms: 1800
> ```
>
> The `slowCallDurationThreshold` should be `1800ms` — Nina is right that it should be lower than the HTTP timeout (otherwise slow calls never trigger the CB). Derek's original 2000ms was equal to the HTTP timeout, which means the CB would never open on slow calls — they'd always timeout first. Nina's 1500ms was too aggressive given the new 1800ms HTTP timeout. Split the difference at the HTTP timeout minus a buffer: `1800ms - 200ms = 1600ms`.
>
> Actually, let me think about this more carefully. The HTTP timeout is 1800ms. The CB `slowCallDurationThreshold` is the boundary between "normal" and "slow." If you set it at 1600ms, then any call between 1600-1800ms counts as "slow" for the CB but still succeeds. If enough calls are in that window, the CB opens. That's the correct behavior — the CB should open BEFORE everything starts timing out, not after.
>
> `slowCallDurationThreshold: 1600ms` with `timeout_ms: 1800ms`. Derek, Nina — does that logic make sense to both of you?
>
> **2. Fix the canary (today, PR from Nina).**
> The canary health check needs to exercise the inventory path. Add a synthetic inventory check to the canary analysis:
> ```yaml
> # In canary AnalysisTemplate (Lisa's PLAT-916 work)
> - name: inventory-path-health
>   provider:
>     prometheus:
>       query: |
>         rate(order_service_inventory_check_errors_total{canary="true"}[2m])
>         / rate(order_service_inventory_check_total{canary="true"}[2m])
>       condition: "result < 0.05"  # <5% inventory check error rate
> ```
> This ensures the canary fails if the inventory check path is broken, even if /health returns 200.
>
> **3. Add a config validation startup check (this week, Derek).**
> The order-service should fail FAST on conflicting config, not on the first real request. Add a `@PostConstruct` bean that validates CB config consistency at startup:
> ```java
> @PostConstruct
> void validateCircuitBreakerConfig() {
>     // Load from all config sources
>     // Throw on conflicts
>     // Log resolved values
> }
> ```
> This makes the failure mode a startup crash (which the readiness probe catches) instead of a runtime exception (which it doesn't).

```
Derek (11:00 AM):
"1600ms makes sense. The CB should detect the degradation 
trend before clients start timing out. That's the whole 
point.

I'll merge the config files with Nina this morning. One 
file, clear sections, clear ownership comments. The 
separate overrides file was a mistake — it solved a 
review-time problem (clean PR diff) by creating a 
runtime problem (invisible config). Bad tradeoff.

@PostConstruct validation — great idea. I'll add it today. 
If the bean fails, Spring won't start, readiness probe 
fails, ArgoCD stops the rollout. Clean failure mode."

Nina (11:01 AM):
"Agreed on 1600ms. And I should have searched the whole 
repo, not just application.yml. My assumption that 'the 
main config file has everything' was wrong.

Canary fix PR: I'll update Lisa's AnalysisTemplate to 
add the inventory-path health check. This is actually a 
gap we should have caught when we first deployed the CB — 
the canary should always exercise every critical code path, 
not just the health endpoint.

Question: should I also add a payment-path canary check? 
Same gap exists there — if payment-service integration 
breaks, /health still returns 200."
```

DM to Nina:

> Yes. Add canary checks for every external dependency path: inventory, payment, and search. If any of them break during a canary deploy, the canary should fail. Good thinking to generalize this.
>
> And Nina — your PLAT-917 analysis was correct. The p99 investigation found that 1800ms is the right timeout. The circuit breaker threshold at 1600ms is the right relationship. The only problem was the config collision, not your engineering judgment.

---

**The corrective deploy (11:30 AM):**

Nina and Derek pair for 30 minutes. They merge the config files, set the correct values, add the `@PostConstruct` validation, and submit a single PR:

```
platform-infra#359: order-service config consolidation

Changes:
  - Merge resilience4j-overrides.yml into application.yml
  - Delete resilience4j-overrides.yml
  - Set slowCallDurationThreshold: 1600ms (was: conflicting 2000/1500)
  - Set inventory_client_timeout_ms: 1800ms (was: 2000)
  - Add @PostConstruct CB config validation
  - Add section headers + ownership comments in application.yml

Co-authored-by: Nina Petrov, Derek Huang
Reviewed-by: [User]
```

I review the PR. Clean diff. Config file is well-organized with clear sections. The `@PostConstruct` validator catches:
- Duplicate property definitions across config sources
- `slowCallDurationThreshold` >= `timeout_ms` (would make CB slow-call detection useless)
- Missing required CB properties

One comment:

```
Comment (ApplicationConfig.java, line 34):
Add a startup log line that prints all resolved CB 
configuration values:

  log.info("Circuit breaker config resolved: " +
    "slowCallDuration={}ms, httpTimeout={}ms, " +
    "failureRate={}%, slidingWindow={}", 
    slowCall, timeout, failureRate, window);

This makes config merge behavior visible in pod logs 
without having to dig into multiple files. During 
incidents, `kubectl logs` should immediately show 
what config the pod is running with.
```

Approved with that comment.

```
Nina (11:45 AM):
"PR merged. ArgoCD syncing. Canary starting.

This time I'm watching the inventory-check error rate, 
not just /health. Derek is watching the CB metrics.

Also: canary AnalysisTemplate update is in a separate PR 
(platform-infra#360) — adds inventory, payment, and search 
dependency health checks to the canary analysis. Lisa is 
reviewing."
```

```
Lisa (11:48 AM):
"PR #360 looks good. The dependency health checks use 
existing Prometheus metrics — no new instrumentation 
needed. Approving. This closes the gap that let today's 
incident through the canary.

Should I add these checks to payment-service and 
search-service canary templates too? Same gap exists 
everywhere."
```

DM to Lisa:

> Yes. Every service should have canary checks that exercise its critical dependency paths. The /health endpoint is necessary but not sufficient. This is a platform-wide pattern, not just order-service. File a ticket and do it this sprint — it's directly relevant to the budget situation.

---

**Canary progression (11:50 AM - 12:30 PM):**

```
Nina (11:50 AM):
"Canary at 5%. Inventory check path: 100% success. 
CB config validation passed at startup (log line confirms 
resolved values). Clean."

Nina (12:00 PM):
"Canary at 25%. All metrics clean. Inventory path exercised 
412 times, zero errors. CB state: CLOSED."

Nina (12:15 PM):
"Canary at 50%. p99: 380ms. Error rate: 0.01%. 
Inventory checks: 2,847 successes, 0 failures.
Payment checks: 1,204 successes, 0 failures.
Promoting to 75%."

Nina (12:30 PM):
"Canary at 100%. Full production. All metrics nominal.

PLAT-917 is finally deployed correctly:
  inventory_client_timeout_ms: 1800ms (was 2000ms)
  slowCallDurationThreshold: 1600ms (was 2000ms)
  
  p99 inventory calls: 1.02s (unchanged — the timeout 
  reduction just tightens the failure boundary)
  
  CB hasn't opened because p99 is well under 1600ms. 
  It will open if inventory degrades past 1600ms, which 
  is exactly the behavior we want."
```

---

**#incidents resolution (12:32 PM):**

```
[User] (12:32 PM):
Incident 12 RESOLVED. Config collision — CB override file.

TIMELINE:
  10:38  Nina's PLAT-917 deploy promoted to 100%
  10:39  Error rate climbing (inventory check path)
  10:42  Nina reports in #order-engineering
  10:47  SEV1 declared. Rollback initiated.
  10:50  Old pods running, error rate dropping
  10:51  Baseline restored. 5 min TTR from ack.
  11:00  Root cause explained, fix plan agreed
  11:30  PR #359: config consolidation (Nina + Derek pair)
  11:45  Corrective deploy canary started
  12:30  100% production, all metrics clean ✅

ROOT CAUSE:
  Config file collision. Derek's CB config in 
  resilience4j-overrides.yml defined slowCallDurationThreshold: 2000ms.
  Nina's PLAT-917 in application.yml defined 
  slowCallDurationThreshold: 1500ms. Spring Boot merged both.
  Resilience4j threw ConfigurationException on the conflict.

  Exception only triggered on inventory-check code path 
  (not on /health). Canary passed because health probe 
  doesn't exercise dependency paths.

CONTRIBUTING FACTORS:
  1. Override file not discoverable (no README, no reference 
     from main config, no comment pointing to it)
  2. Canary health check only tests /health endpoint, not 
     dependency paths (inventory, payment, search)
  3. No startup validation of config consistency

FIXES (all deployed):
  ✅ Config files merged into single application.yml with 
     section headers and ownership comments
  ✅ @PostConstruct CB config validator (fails startup on conflicts)
  ✅ Correct values: timeout 1800ms, CB slow threshold 1600ms
  ⏳ Canary AnalysisTemplate: dependency path health checks 
     (PR #360, Lisa reviewing — covers inventory, payment, search)
  ⏳ Extend dependency canary checks to all services (Lisa, this sprint)

BUDGET IMPACT:
  order-service: 60.9% → 58.1% → holding (corrective deploy clean)
  
  We're at 58.1% with 5 days in the window. Uncomfortably 
  thin. The CB + stockCache improvements are earning ~0.04%/day 
  back, so projected window end: ~58.3%.
  
  RECOMMENDATION: Soft deploy freeze on order-service for 
  remainder of this error budget window. Exceptions: 
  corrective deploys and security fixes only. Any new 
  feature deploys wait for the next window.
  
  @Sarah @Lisa — requesting alignment on this.

PROCESS LESSONS:
  1. Multi-file config is a hidden coupling surface. Services 
     should have ONE config source of truth, or explicit merge 
     validation at startup.
  2. Canary health checks must exercise dependency paths, not 
     just the health endpoint. /health returning 200 while 67% 
     of real requests fail is a canary design deficiency.
  3. When two engineers independently modify related config, 
     the CODEOWNERS file should require cross-review. Adding 
     resilience4j-overrides.yml to CODEOWNERS with both Derek 
     and Nina as reviewers would have caught this.
```

```
Sarah (12:35 PM):
"Soft freeze approved. Order-service: corrective and 
security only until the budget window resets. I'll 
communicate to the order team.

Also: this incident is a good example for the board 
30-day follow-up. It shows the canary gap (identified 
and fixed) and the config management improvement 
(deployed same day). The system is getting better in 
real-time."

Lisa (12:37 PM):
"Budget confirmed at 58.1%. Soft freeze is the right call. 
I'm updating the budget dashboard to show the freeze status.

PR #360 approved and merged. Canary AnalysisTemplate now 
includes dependency health checks for order-service. I'll 
extend to payment-service and search-service by Wednesday."
```

---

**DM to Derek (12:40 PM):**

> Derek — I need you to hear something. In two weeks, you've shipped a circuit breaker, a stockCache with three-tier fallback, fixed a DNS amplification issue, and today consolidated a config collision in 45 minutes of pair programming. Every one of these has made the system materially better. The override file was a learning moment — you've already internalized it (your own words: "solved a review-time problem by creating a runtime problem").
>
> The reason today's incident was SEV1 for 5 minutes instead of SEV1 for 45 minutes is because Nina knew to rollback immediately and you knew exactly where the conflict was. Fast diagnosis, fast fix, fast deploy. That's the team working.

```
Derek (12:42 PM):
"Thanks. I'm adding a personal rule: if I create a new 
config file in any service, the PR must include a comment 
in the MAIN config file pointing to it, a README update, 
and a CODEOWNERS entry. No more invisible config surfaces.

Also: the @PostConstruct validator caught a second issue 
during testing that we didn't notice — the slidingWindowType 
was set to COUNT_BASED in the old override file but the 
documentation I wrote during the CB review says TIME_BASED 
is recommended. We fixed it in the merge. So the config 
consolidation actually improved correctness beyond just 
preventing conflicts."
```

---

**Personal log (12:45 PM, mental note):**

```
Incident 12 — Config collision. CB override file vs application.yml.

What went right:
  - Nina reported immediately (10:42, 3 min after error started)
  - Rollback decision was instant (no debugging in prod)
  - 5-minute TTR from my ack
  - Root cause explained clearly to both engineers
  - Nina and Derek paired on the fix (no blame, constructive)
  - Corrective deploy through full canary in 45 minutes
  - Three systemic fixes deployed same day (config merge, 
    startup validator, canary dependency checks)
  - Extended canary fix to all services (Lisa)

What went right that I didn't do:
  - Nina's PLAT-917 analysis was correct (1800ms timeout)
  - Derek immediately identified the override file as the conflict
  - Lisa caught the budget impact and recommended freeze
  - All three engineers self-organized the fix

What I should learn:
  - During Derek's original CB PR review, I should have 
    flagged the separate config file as a discoverability risk.
    I reviewed the CB logic but not the operational ergonomics 
    of where the config lives. Config file layout is a review 
    concern, not just config values.
  - The canary gap (dependency paths not exercised) has existed 
    since we deployed the CB two weeks ago. Lisa's original 
    AnalysisTemplate (PLAT-916) only checks overall error rate, 
    not per-dependency health. I approved that template without 
    asking "does this canary catch every failure mode?"

Consequence chain continues:
  Incident 1 (timeout) → CB → stockCache → DNS storm → 
  ndots fix → PLAT-917 timeout adjustment → config collision
  
  Six links. Every single one is a consequence of Incident 1.
  The original 800ms timeout has spawned a chain of fixes 
  and incidents spanning two weeks.

Category: Deploy/Release failures (config collision variant).
```

---

### 12:50 PM — Rest of Monday

Incident resolved. Order-service on soft freeze. Canary checks improved across services. Config management pattern established.

```
MONDAY STATUS — 12:50 PM:

✅ Incident 12 (config collision): RESOLVED, 5 min TTR
✅ R-6a: closed Friday ✅
✅ FluentBit: Jake PR ready, deploying Tuesday
✅ Tom R-6b: Helm chart PR in review
✅ Priya: Search ArgoCD onboarding in progress
✅ Derek: stockCache in production, config consolidated
✅ Nina: PLAT-917 deployed correctly
✅ Lisa: Canary dependency checks deployed + extending to all services

⏳ Rachel SA-2024-0119-NM response: due Feb 5 (drift detection docs needed)
⏳ Headcount job descriptions: due Wednesday
⏳ Order-service: soft freeze until budget window reset
⏳ Image admission enforce mode: Feb 2 (4 days)
⏳ FluentBit ECR migration: Tuesday (Jake)
⏳ R-6b milestone: Feb 9

Budget:
  Order-service: 58.1% (soft freeze)
  Payment-service: 86.0% (velocity freeze)
```

The afternoon is planned work: drift detection documentation for Rachel, headcount job description input for Sarah, and reviewing Tom's R-6b Helm chart. No more fires today.

---

**Twelve incidents. Two weeks. The system keeps testing itself.**

