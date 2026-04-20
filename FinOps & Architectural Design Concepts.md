# Phase 7: Cost Optimization & FinOps
# Phase 8: Architectural Design from Scratch

---

# PHASE 7: COST OPTIMIZATION & FINOPS

## Phase 7 Context Document

```
══════════════════════════════════════════════════════════════════════
NOVAMART — COST PROFILE (as of Phase 7)
══════════════════════════════════════════════════════════════════════

Monthly AWS Spend: $187,000/month (trending to $240,000 by Q4)
Annual Budget: $2.0M (will exceed by $280K at current trajectory)

BREAKDOWN:
  EKS + EC2 (worker nodes):     $78,000/mo   (41.7%)
  RDS (PostgreSQL Multi-AZ):    $31,000/mo   (16.6%)
  Data Transfer:                $24,000/mo   (12.8%)
  S3 Storage:                   $14,000/mo    (7.5%)
  ElastiCache (Redis):          $11,000/mo    (5.9%)
  NAT Gateway:                  $9,500/mo     (5.1%)
  CloudFront:                   $7,000/mo     (3.7%)
  ECR:                          $3,500/mo     (1.9%)
  Monitoring (CloudWatch):      $4,200/mo     (2.2%)
  Other (Lambda, SQS, SNS):    $4,800/mo     (2.6%)

EKS CLUSTER DETAILS:
  Production cluster: 18 nodes (m5.2xlarge on-demand)
  Staging cluster: 8 nodes (m5.xlarge on-demand)
  Dev cluster: 6 nodes (m5.xlarge on-demand)
  Total: 32 nodes, ALL on-demand, ZERO Spot, ZERO Reserved

RESOURCE UTILIZATION (from Prometheus):
  Average cluster CPU utilization: 34%
  Average cluster memory utilization: 51%
  Peak CPU utilization (Black Friday): 87%
  Peak memory utilization (Black Friday): 72%

  Per-namespace breakdown:
    orders:          CPU req: 12 cores, actual use: 4.2 cores  (35%)
    payments:        CPU req: 8 cores, actual use: 5.1 cores   (64%)
    search:          CPU req: 16 cores, actual use: 3.8 cores  (24%)
    users:           CPU req: 6 cores, actual use: 1.9 cores   (32%)
    inventory:       CPU req: 10 cores, actual use: 2.1 cores  (21%)
    notifications:   CPU req: 4 cores, actual use: 0.6 cores   (15%)
    analytics:       CPU req: 8 cores, actual use: 6.2 cores   (78%)
    monitoring:      CPU req: 6 cores, actual use: 4.8 cores   (80%)

KEY PROBLEMS:
  1. CFO flagged AWS spend increasing 12% month-over-month
  2. Engineering says "we can't cut anything without risking reliability"
  3. No cost attribution — nobody knows which team drives which cost
  4. No Reserved Instances or Savings Plans purchased
  5. NAT Gateway data transfer costs are unexplained ($9,500/mo)
  6. S3 has 14TB of data, unclear how much is actively accessed
  7. Dev/staging environments run 24/7 identical to production
  8. CloudWatch costs growing 20% monthly due to custom metrics
══════════════════════════════════════════════════════════════════════
```

---

## Phase 7, Lesson 1: Kubernetes Resource Economics

### Lesson Preamble

```
This lesson focuses on the #1 source of Kubernetes cost waste: the gap
between REQUESTED resources and ACTUALLY USED resources. At NovaMart,
the cluster is 34% utilized but teams claim they can't reduce anything.
Your job: prove them wrong with data, implement right-sizing without
causing outages, and build the guardrails that prevent over-provisioning
from creeping back.

KEY CONCEPTS:
- Requests vs Limits vs Actual Usage — the cost triangle
- Quality of Service (QoS) classes and their cost implications
- Vertical Pod Autoscaler (VPA) — recommendations vs auto-mode
- Right-sizing methodology: observe → recommend → test → enforce
- Resource Quotas and LimitRanges as cost governance tools
- Pod Disruption Budgets and their interaction with cost optimization
- Node right-sizing and bin-packing efficiency
- Cluster Autoscaler vs Karpenter — cost-aware scheduling
```

### Questions

#### Q1: The Over-Provisioning Investigation 🔥

**Scenario:** The Platform team has been asked by the CFO to reduce EKS compute costs by 30% ($23,400/month). The VP of Engineering says "our services are already running lean — any reduction will cause outages on Black Friday." You have Prometheus metrics going back 90 days.

1. Using the NovaMart utilization data above, calculate the exact dollar amount of waste in CPU and memory over-provisioning. Show your math. What is the theoretical minimum cluster size that could serve current ACTUAL usage with appropriate headroom? How does this compare to the current 18-node production cluster?

2. The `search` namespace requests 16 CPU cores but uses 3.8. The search team lead says: "We had a Black Friday spike to 14 cores last year — we need the headroom." Pull apart this argument: what's correct about it, what's flawed, and what's the right way to handle burst capacity without permanently reserving 4x the normal usage? Include the exact HPA and VPA configuration.

3. Several pods have `resources.requests` set equal to `resources.limits` (Guaranteed QoS). Others have no resource requests at all (BestEffort QoS). Explain the cost implication of each QoS class, why both extremes are wrong for most workloads, and design the LimitRange and ResourceQuota for the `orders` namespace that enforces sensible defaults. Include the YAML.

4. The `notifications` namespace uses 15% of its requested CPU (0.6 of 4 cores). The team says they set requests high "just in case." Implement VPA in recommendation mode for this namespace, show the exact commands to extract actionable recommendations, and design the process for safely applying them without disrupting the service. Include rollback criteria.

---

#### Q2: Node-Level Cost Optimization 🔥

**Scenario:** All 18 production nodes are `m5.2xlarge` (8 vCPU, 32 GiB, $0.384/hr on-demand in us-east-1). The Cluster Autoscaler is configured but has `scale-down-utilization-threshold: 0.5` and `scale-down-delay-after-add: 10m`. Nodes are rarely removed even during low-traffic periods (midnight–6 AM, weekends).

1. NovaMart's workloads are a mix of CPU-bound (search, analytics) and memory-bound (Redis sidecars, JVM services). Explain why a single instance type (`m5.2xlarge`) for all workloads is suboptimal. Design the mixed instance type strategy using at least 3 different instance families. Include the rationale for each (compute-optimized, memory-optimized, general purpose), which workloads go where, and the Karpenter NodePool configuration.

2. The production cluster has 18 nodes but average utilization is 34% CPU. However, pods are spread unevenly — some nodes are at 80% and others at 15%. Explain what causes this and what Kubernetes scheduling mechanisms you'd use to improve bin-packing. Include specific scheduler configuration and pod topology spread constraints.

3. Design the Spot instance strategy for NovaMart's production cluster. Which workloads CAN go on Spot? Which MUST stay on On-Demand? How do you handle Spot interruptions gracefully? Include the Karpenter NodePool with disruption budget, and the PodDisruptionBudgets for critical services.

4. NovaMart runs 18 production nodes 24/7, but traffic from midnight to 6 AM is 8% of peak. Design the scaling schedule that safely reduces capacity during off-peak hours. What metrics trigger scale-down? What prevents aggressive scale-down from causing latency spikes when the morning traffic ramp begins? Include the Cluster Autoscaler or Karpenter configuration and the HPA settings.

---

#### Q3: Resource Governance & Guardrails 🔥

**Scenario:** After right-sizing, you've saved $18,000/month. But two months later, costs are creeping back up — teams are increasing requests "just in case" ahead of a product launch. The CFO wants guardrails that prevent this from recurring.

1. Design the ResourceQuota and LimitRange strategy for all NovaMart namespaces. The constraints: production namespaces collectively cannot exceed 80% of cluster capacity in requests. Each namespace has a proportional budget based on historical usage. Staging is capped at 40% of production's resources. Dev is capped at 25%. Include the YAML for the `orders` namespace (the largest consumer) and the `notifications` namespace (the smallest).

2. A developer submits a deployment with `requests.cpu: 4` and `limits.cpu: 4` for a service that typically uses 0.5 CPU. The ResourceQuota would allow it (the namespace has room). Design the Gatekeeper policy that prevents over-provisioning by enforcing a maximum ratio between requests and historical usage (from VPA recommendations). Include the ConstraintTemplate with Rego.

3. Build the cost monitoring and alerting stack. The CFO wants a weekly email showing cost-per-namespace and cost-per-team. The VP of Engineering wants real-time alerts when any namespace exceeds its budget. Design this using Kubecost (or OpenCost) + Prometheus + Grafana. Include the key PromQL queries, the Grafana dashboard design, and the alerting rules.

4. NovaMart is evaluating Kubecost Enterprise vs OpenCost (open source). The CFO wants the cheapest option; the VP of Engineering wants accurate showback reporting. Compare the two tools: what does each provide, what are the accuracy limitations of each, and what is your recommendation with justification? Include the cost of Kubecost Enterprise itself in your analysis.

---

#### Q4: The Cost Optimization Incident 🔥

**Scenario:** Friday 4 PM. An engineer applied VPA recommendations to the `payments` namespace, reducing CPU requests from 8 cores to 3.2 cores across all payment-svc pods. By 5 PM (pre-weekend traffic spike), payment-svc p99 latency goes from 150ms to 2.8 seconds. Transactions are timing out. Revenue impact: ~$12,000/hour.

1. Diagnose the root cause. The VPA recommended 3.2 cores based on 90-day average usage. Why is this recommendation dangerous for the payments service specifically? What percentile should VPA target for latency-sensitive services, and how do you configure it?

2. Your immediate action to restore service. Then your longer-term fix that balances cost savings with reliability for the payments namespace. Include the exact kubectl commands for immediate remediation and the VPA configuration for the permanent fix.

3. This incident reveals a process gap. Right-sizing was applied to a revenue-critical service on a Friday afternoon before a traffic spike. Design the change management process for resource changes to production services. Include: timing restrictions, blast radius limits, rollback criteria, and approval gates.

4. After the incident, recalculate the actual safe savings for the payments namespace. Given that payments-svc needs to handle peak load (which is 5.1 cores average but spikes to 7.2 cores during checkout surges), what should the requests and limits be? Show the math, explain the headroom calculation, and reconcile this with cost targets.

---

## Phase 7, Lesson 2: AWS Cost Management

### Lesson Preamble

```
This lesson moves beyond Kubernetes to the AWS bill itself. NovaMart
is spending $187K/month with zero Reserved Instances, zero Savings Plans,
and unexplained data transfer costs. 41% of the bill is EC2 — but the
other 59% has optimization opportunities that most engineers ignore.

KEY CONCEPTS:
- Reserved Instances vs Savings Plans — when to use which
- Compute Savings Plans vs EC2 Instance Savings Plans
- RDS Reserved Instances and Aurora Serverless cost modeling
- Data transfer costs — the hidden AWS tax
- NAT Gateway costs and VPC endpoint alternatives
- S3 storage classes and lifecycle policies
- CloudWatch cost optimization (metrics, logs, insights)
- AWS Cost Explorer, Cost Anomaly Detection, Budgets
- The "cloud tax" — services that cost more than they should
```

### Questions

#### Q1: Savings Plans Strategy 🔥

**Scenario:** NovaMart has been running on AWS for 2 years with zero commitments (all on-demand). The CFO has approved committing to a 1-year Savings Plan but wants to understand the risk. Finance needs the analysis by Wednesday.

1. Using NovaMart's current spend ($78K/month on EKS/EC2), calculate the savings from: (a) Compute Savings Plan at 1-year no-upfront, (b) EC2 Instance Savings Plan at 1-year partial-upfront, (c) 3-year Compute Savings Plan all-upfront. For each option, state the commitment amount, the savings percentage, the annual savings in dollars, and the risk if NovaMart's usage pattern changes (migration to Graviton, region change, or traffic decline).

2. NovaMart is considering migrating some workloads to Graviton (ARM-based) instances for additional savings. How does this affect the Savings Plan strategy? Which type of Savings Plan accommodates a Graviton migration, and which locks you into x86? Design the phased commitment: what percentage of current usage should be committed vs. left on-demand to preserve flexibility?

3. RDS costs $31K/month across 3 Multi-AZ PostgreSQL instances (payments: db.r5.2xlarge, orders: db.r5.xlarge, users: db.r5.xlarge). All on-demand. Calculate the savings from RDS Reserved Instances. One complication: the users-db is being evaluated for migration to Aurora Serverless v2. How does this uncertainty affect your RI recommendation? What do you commit, and what do you leave flexible?

4. The CFO asks: "If we commit $600K/year in Savings Plans and our traffic drops 30% because a competitor launches, how much money do we lose?" Answer this precisely. Then design the commitment strategy that protects against downside risk while still capturing most of the savings. Include the concept of "coverage ratio" and where NovaMart should target it.

---

#### Q2: Data Transfer — The Hidden Tax 🔥

**Scenario:** NovaMart's data transfer line item is $24,000/month and growing 15% monthly. The networking team says "that's just what it costs to run in AWS." You disagree.

1. Break down every possible source of AWS data transfer charges for NovaMart's architecture (EKS on EC2, RDS, ElastiCache, S3, CloudFront, NAT Gateway, cross-AZ, VPC endpoints). For each source, explain when it generates charges and when it doesn't. Most engineers get at least 2 of these wrong.

2. NAT Gateway costs $9,500/month. Decompose this into the per-GB processing charge and the per-hour charge. Using NovaMart's architecture, identify what traffic is flowing through the NAT Gateway that SHOULDN'T be. Design the VPC endpoint strategy that reduces NAT Gateway costs. Include exact Terraform for each VPC endpoint and the expected cost reduction.

3. Cross-AZ data transfer is charged at $0.01/GB per direction. NovaMart's EKS pods communicate across AZs frequently (service mesh traffic, pod-to-RDS, pod-to-Redis). Calculate the cross-AZ cost given: 500GB/day of inter-service traffic, 30% of which crosses AZ boundaries. Then design the topology-aware routing strategy that reduces cross-AZ traffic. Include Kubernetes topology spread constraints and Istio/service mesh locality-aware routing configuration.

4. S3 costs $14,000/month. NovaMart has 14TB of data. At standard storage rates, 14TB should cost ~$330/month. The remaining ~$13,670 is request costs and data transfer. Investigate: what access patterns cause S3 costs to balloon like this? Design the S3 lifecycle policy, storage class strategy, and access pattern optimization that brings S3 costs under $2,000/month. Include the Terraform for lifecycle rules.

---

#### Q3: Environment Cost Rationalization 🔥

**Scenario:** Dev and staging clusters run 24/7 and are nearly identical to production in size. Dev: 6 nodes ($0.192/hr × 6 × 730 = $8,410/month). Staging: 8 nodes ($0.192/hr × 8 × 730 = $11,213/month). Combined: $19,623/month for non-production environments. The VP of Engineering says "we need production-parity for testing."

1. Dismantle the "production-parity" argument. What specifically needs to be identical to production for testing to be valid, and what doesn't? Design the right-sized dev and staging environments that maintain testing validity while reducing costs by at least 60%. Include instance types, node counts, scaling schedules, and what tradeoffs you're making.

2. Staging RDS instances are identical to production (Multi-AZ, same instance type). Design the staging database strategy that costs 70% less while still providing valid pre-production testing. Address: what breaks if staging DB is smaller than production? How do you test performance-sensitive queries? What about data volume differences?

3. Implement automated environment scheduling. Dev environments should run only during business hours (8 AM – 8 PM weekdays, team's timezone). Staging should run 24/7 but scale down to minimum at night. Design the automation using AWS Lambda + EventBridge (or Karpenter node scaling) that implements this. Include the exact Terraform/CloudFormation, the Lambda function code, and the exception handling (what if a dev needs the environment at midnight?).

4. A developer proposes using ephemeral preview environments — spin up a complete environment per pull request, run tests, tear it down. Calculate the cost of this approach assuming 40 PRs/day, each environment runs for 2 hours, each environment costs $3/hour. Compare this to the persistent dev environment cost. What's the breakeven point, and under what conditions is each approach cheaper?

---

#### Q4: The Cost Anomaly Investigation 🔥

**Scenario:** Tuesday morning. AWS Cost Anomaly Detection fires an alert: CloudWatch costs spiked from $4,200/month to a projected $11,000/month over the past week. Nobody made any intentional changes.

1. CloudWatch charges for: PutMetricData API calls, custom metrics, log ingestion, log storage, Logs Insights queries, dashboards, and alarms. Design the investigation process to identify which component caused the spike. Include the exact AWS CLI commands and Cost Explorer filters.

2. The investigation reveals: a new Prometheus ServiceMonitor was added that scrapes a high-cardinality metric (HTTP request duration by URL path, user ID, and status code) every 15 seconds. This generates 2.4 million unique metric series. Calculate: (a) the CloudWatch cost if these metrics are shipped via CloudWatch agent, (b) the Prometheus storage cost if kept in-cluster, (c) the Grafana Cloud cost for the same volume. Which is cheapest and why?

3. The developer who added the ServiceMonitor says: "I need per-user latency metrics for debugging." This is a legitimate need but the implementation is wrong. Design the correct observability approach that gives per-user debugging capability without high-cardinality explosion. Include: what metrics to keep, what to convert to logs/traces, and the relabeling configuration that drops high-cardinality labels before storage.

4. After resolving this incident, design the cost governance for observability. How do you prevent engineers from creating cost explosions with metrics? Include: Prometheus relabeling rules to drop high-cardinality series, recording rules to pre-aggregate, retention policies, and Gatekeeper policies that limit ServiceMonitor configurations.

---

## Phase 7, Lesson 3: FinOps Culture & Organizational Integration

### Lesson Preamble

```
Technical optimization is necessary but insufficient. NovaMart's cost
problem is also an organizational problem: no one owns costs, no one
sees costs, and engineers make provisioning decisions without cost
awareness. This lesson builds the organizational system that makes
cost optimization sustainable.

KEY CONCEPTS:
- FinOps framework: Inform → Optimize → Operate
- Cost attribution and showback/chargeback models
- Tagging strategy and enforcement
- Cost-aware engineering culture
- Budget management and forecasting
- Executive reporting and communication
- Making the business case for infrastructure investment
```

### Questions

#### Q1: Cost Attribution & Tagging Strategy 🔥

**Scenario:** The CFO asks: "Which product team is responsible for the $187K/month AWS bill?" Currently, nobody can answer this. There's no consistent tagging strategy, Kubernetes namespaces don't map cleanly to cost centers, and shared services (monitoring, ingress, databases) are unattributed.

1. Design the complete tagging strategy for NovaMart. Define the mandatory tags, who owns them, how they're enforced, and how they map to Kubernetes namespaces. Include the Terraform tag enforcement (aws_default_tags), the SCP that denies untagged resource creation, and the AWS Config rule that detects tag drift.

2. Shared services (ALB, NAT Gateway, monitoring stack, service mesh control plane) are used by all teams but owned by the platform team. Design the cost allocation model: what percentage goes to each product team, what stays with platform, and how do you calculate the split? Address the political challenge: the search team will argue they shouldn't pay for the payments team's database.

3. Kubernetes cost attribution is harder than AWS tagging because pods share nodes. Explain the challenge of attributing node costs to namespaces when a single node runs pods from 5 different namespaces. How does Kubecost/OpenCost solve this? What are the accuracy limitations? Design the allocation model and the Kubecost configuration.

4. Build the weekly cost report that goes to: (a) the CFO (wants total spend, trend, forecast, action items — one page), (b) engineering leads (want namespace-level breakdown, anomalies, optimization recommendations), (c) individual developers (want their team's spend and what they can do about it). Design all three reports — what data, what format, what delivery mechanism.

---

#### Q2: Building Cost-Aware Engineering Culture 🔥

**Scenario:** After implementing cost visibility, you discover that engineers don't look at the dashboards. Cost reports go unread. The platform team is the only group that cares about optimization. The VP of Engineering wants to make cost awareness part of engineering culture without creating resentment or slowing down development.

1. Design the "cost context in the developer workflow" system. Engineers should see cost implications BEFORE they deploy, not after. Include: cost estimation in PR reviews (what does this Terraform change cost?), cost badges on deployments (this deployment costs $X/month), and cost impact in CI/CD pipeline output. Include the Infracost integration for Terraform PRs and the Kubecost cost prediction for Kubernetes manifests.

2. The VP proposes giving each team a monthly cloud budget and letting them self-manage. Design this system: how are budgets set, what happens when a team approaches their limit, what happens when they exceed it, and how do you handle legitimate budget overruns (traffic growth, new features)? Address the gaming risks — teams sandbagging budgets, teams blaming shared services, teams deferring optimization to next quarter.

3. An engineer argues: "My time costs $80/hour. If I spend 4 hours optimizing to save $200/month, the payback period is 1.6 months. But if I spend those 4 hours building features, the revenue impact is higher." This is a legitimate argument. How do you counter it? When IS optimization worth engineering time, and when isn't it? Design the decision framework with concrete thresholds.

4. Design the quarterly FinOps review process. Who attends, what's reviewed, what decisions are made, and what's the output? Include the agenda, the metrics reviewed, the action item format, and how you track follow-through. Make this practical enough that NovaMart could implement it next month.

---

## Phase 7, Lesson 4: Cost Optimization Under Pressure — Integrated Scenarios

### Questions

#### Q1: The Budget Crisis 🔥

**Scenario:** Q3 results are in. NovaMart's AWS spend was $561K for the quarter against a $500K budget. The CFO mandates a 25% cost reduction ($140K annually) within 60 days or the engineering team faces headcount reductions. The VP of Engineering asks you to lead the effort.

1. Design the 60-day cost reduction plan. Categorize every optimization into: Quick Wins (Week 1-2, no risk), Medium Effort (Week 2-4, some risk), Strategic Changes (Week 4-8, significant effort). For each item, specify: the expected monthly savings, the implementation effort (hours), the risk level, and the rollback plan. Your total must exceed $11,700/month ($140K/year).

2. The search team refuses to cooperate. They say "our service is already optimized" but they're the most over-provisioned namespace (24% utilization). The team lead says reducing resources will cause search to fail during peak traffic. You need to get their buy-in. How do you present the data, address their concerns, and get agreement — without escalating to management? (This is a human problem, not a technical one.)

3. During the optimization sprint, the analytics team discovers that their Prometheus instance is storing 90 days of full-resolution metrics consuming 800GB of EBS storage ($80/month in gp3 but $2,400/month when you include IOPS and throughput). Design the metrics retention strategy: what resolution for what time range, how to implement downsampling, and what tool (Thanos, Cortex, Grafana Mimir) to use. Calculate the storage savings.

4. After the 60-day sprint, you've achieved $13,200/month in savings. The CFO is satisfied but asks: "How do I know costs won't creep back up?" Design the continuous cost governance system: automated alerts, monthly reviews, budget guardrails, and the reporting cadence that makes cost optimization a permanent practice, not a one-time project.

---

#### Q2: Cost vs Reliability Tradeoff 🔥

**Scenario:** The SRE team pushes back on cost optimization. They say: "Every dollar you cut from infrastructure increases our risk of an outage. Our SLA guarantees 99.95% uptime. One major outage costs $500K in lost revenue. The math doesn't support cutting $140K/year in infrastructure if it causes even one additional outage."

1. This argument has merit but is also used to resist ANY optimization. Dismantle it precisely: what is the ACTUAL relationship between resource headroom and reliability? At what point does additional headroom stop improving reliability? Use NovaMart's utilization data to show where the diminishing returns curve is.

2. Design the framework for making cost-vs-reliability tradeoffs. For each optimization proposed, how do you quantify the reliability risk? Include: the error budget model (if SLA is 99.95%, how many minutes of downtime per month are acceptable?), the risk scoring for each optimization category, and the approval process for optimizations that touch reliability-sensitive services.

3. The `payments` service is the most reliability-sensitive (direct revenue impact). Currently over-provisioned by 40%. Design the optimization plan for payments that captures savings WITHOUT touching the error budget. Include: the minimum safe headroom, the canary deployment strategy for resource changes, the automated rollback triggers, and the monitoring that detects degradation before customers do.

4. Calculate the total cost of an outage for NovaMart (revenue loss, SLA credits, customer trust, engineering time for incident response). Compare this to the annual savings from your optimization plan. At what point does the optimization math stop making sense? Present this to both the CFO and the SRE team in a way that gets both to agree.

---

# PHASE 8: ARCHITECTURAL DESIGN

## Phase 8 Context Document

```
══════════════════════════════════════════════════════════════════════
Phase 8 shifts perspective. You are no longer operating NovaMart's
existing platform. You are DESIGNING platforms from scratch, migrating
legacy systems, and making the foundational architectural decisions
that determine the next 3-5 years of a company's infrastructure.

This phase tests:
  - Greenfield architecture design (new company, blank AWS account)
  - Migration planning (monolith to microservices, on-prem to cloud)
  - Multi-account and multi-environment strategy
  - Resilience and disaster recovery architecture
  - Technical decision-making under uncertainty
  - Communicating architecture to non-technical stakeholders
  - Capstone: full platform design integrating everything from Phases 1-8

The scenarios are deliberately ambiguous. Real architectural decisions
are made with incomplete information. You must state your assumptions,
justify your choices, and acknowledge the tradeoffs you're making.
══════════════════════════════════════════════════════════════════════
```

---

## Phase 8, Lesson 1: Greenfield Platform Design

### Lesson Preamble

```
You are the first infrastructure/platform hire at a startup called
"MedRelay" — a healthcare SaaS company that provides telemedicine
scheduling, medical record transfers, and prescription management.

MedRelay's constraints:
  - HIPAA compliance required (healthcare data)
  - 3 backend engineers, 2 frontend engineers, 1 you (infra)
  - Series A funded: $4M in the bank, 18-month runway
  - Current state: monolith Rails app running on Heroku ($2K/month)
  - Growth: 500 users today, projecting 50,000 in 12 months
  - Technical founders who want "the right architecture from day one"
  - Target: SOC 2 Type II certification within 12 months

You must design the infrastructure from a blank AWS account to
production-ready, HIPAA-compliant, cost-effective platform.

KEY CONSTRAINT: With $4M and 18-month runway, every dollar spent on
infrastructure is a dollar not spent on product development. Over-
engineering is as dangerous as under-engineering. Premature
Kubernetes adoption has killed more startups than monoliths have.
```

### Questions

#### Q1: The Technology Decision 🔥

**Scenario:** The CTO (technical co-founder) says: "I want us on Kubernetes from day one. NovaMart uses EKS and they're crushing it. We should too." You need to advise whether this is the right call.

1. Make the case AGAINST Kubernetes for MedRelay at their current stage (500 users, 6 engineers, 18-month runway). Be specific: what is the operational cost of running EKS (in dollars AND engineering hours), what's the learning curve tax, and what does the team give up by spending time on K8s instead of product? Compare the monthly cost of: (a) staying on Heroku, (b) ECS Fargate, (c) EKS. Include the hidden costs (engineer time, learning curve, maintenance burden).

2. Now make the case FOR Kubernetes. At what user count / traffic level / team size does EKS become the right choice? What signals should MedRelay watch for that indicate "it's time to migrate"? Design the decision framework with specific, measurable thresholds.

3. The CTO is convinced: they want ECS Fargate now with a migration path to EKS later. Design the ECS architecture for MedRelay that is HIPAA-compliant from day one. Include: VPC design, ECS cluster configuration, RDS (encrypted, Multi-AZ), ALB with WAF, CloudTrail, and the logging/audit requirements for HIPAA. Include Terraform for the core infrastructure.

4. Design the migration path from ECS to EKS that MedRelay can execute in 12-18 months when they outgrow ECS. What should they build NOW (on ECS) that makes the future migration easier? What patterns should they avoid that would make migration harder? Include: container image standards, configuration management patterns, service discovery approaches, and CI/CD design that works for both ECS and EKS.

---

#### Q2: HIPAA-Compliant Architecture from Scratch 🔥

**Scenario:** MedRelay handles Protected Health Information (PHI). HIPAA requires encryption at rest and in transit, audit logging of all PHI access, access controls, and the ability to produce audit reports. The compliance officer (part-time consultant) gives you a checklist of 47 HIPAA technical safeguards.

1. Design the AWS account structure for HIPAA compliance. Should MedRelay use a single account or multi-account? (They're 6 engineers.) If multi-account, what's the minimum viable account structure? Include: AWS Organizations, SCPs, and the specific accounts needed. Justify the complexity against the team size.

2. Design the data architecture for PHI. Where is PHI stored (RDS, S3, ElastiCache)? How is it encrypted (KMS key hierarchy)? Who has access (IAM roles, not users)? How is access audited? Include the KMS key policy, the RDS encryption configuration, the S3 bucket policy with mandatory encryption, and the CloudTrail configuration for data event logging.

3. HIPAA requires a Business Associate Agreement (BAA) with every vendor that handles PHI. AWS has a BAA, but not all AWS services are HIPAA-eligible. List the AWS services MedRelay's architecture uses and identify which are HIPAA-eligible and which are NOT. For any non-eligible service in the architecture, provide the HIPAA-eligible alternative.

4. Design the audit and compliance monitoring stack. The compliance officer needs to produce evidence of: (a) all PHI access for the last 12 months, (b) all user authentication events, (c) all infrastructure changes, (d) encryption status of all data stores. Design the system using AWS Config, CloudTrail, SecurityHub, and custom automation. Include the AWS Config rules for HIPAA conformance pack.

---

#### Q3: CI/CD for a Small Team 🔥

**Scenario:** MedRelay has 3 backend engineers and no dedicated DevOps person (you're splitting time between infrastructure and feature work). The CI/CD pipeline needs to be simple enough that any engineer can understand and modify it, robust enough for HIPAA compliance, and cheap enough for a startup budget.

1. Design the end-to-end CI/CD pipeline for MedRelay. From git push to production deployment. The constraints: must include automated testing, container scanning, infrastructure as code, and audit trail (HIPAA requirement). Compare: GitHub Actions, GitLab CI, Bitbucket Pipelines, and AWS CodePipeline. Recommend one with justification. Include the pipeline YAML.

2. MedRelay deploys the monolith Rails app as a single container. The deployment strategy needs to be zero-downtime (patients are using the telemedicine platform). Design the deployment strategy for ECS Fargate: blue/green, rolling, or canary? Include the ALB configuration, health check design, and rollback procedure. Justify your choice against the other options.

3. The compliance officer requires that every deployment to production is traceable to: a specific commit, a specific engineer, a passing test suite, and a passed security scan. Design the "deployment provenance" system that meets this requirement. Include: how commits are linked to deployments, how test results are stored, how scan results are attached, and how an auditor can trace from "this version is running in production" back to every artifact that proves it's safe.

4. One of the engineers wants to implement GitOps with ArgoCD. For a 6-person team on ECS (not even Kubernetes), is this appropriate? Make the case for and against, then recommend the right level of deployment automation for MedRelay's current stage. What's the simplest system that meets HIPAA requirements?

---

#### Q4: Scaling Architecture Decisions 🔥

**Scenario:** 8 months in. MedRelay has 15,000 users and is growing 30% month-over-month. The Rails monolith is hitting performance limits. The database is at 70% CPU during peak hours. Response times are climbing. The CTO wants to discuss "breaking up the monolith."

1. The CTO proposes extracting 3 microservices: scheduling, medical-records, and prescriptions. Evaluate this decision: is it the right time? What are the criteria for when to extract a microservice vs. when to optimize the monolith? For MedRelay's specific situation (15K users, 6 engineers, single database), what would you recommend and why?

2. Regardless of the microservices decision, the database is the immediate bottleneck. Design the database scaling strategy: read replicas, connection pooling, query optimization, or Aurora migration? Each has different cost, complexity, and timeline implications. Evaluate all four, recommend the sequence, and justify why you'd do them in that order.

3. The medical-records feature requires file storage (PDFs, images, DICOM files). Currently stored as BLOBs in PostgreSQL. This is contributing to the database performance problem. Design the migration to S3-based document storage with: HIPAA-compliant encryption, signed URLs for temporary access, lifecycle policies for archival, and the database schema change to reference S3 objects instead of BLOBs. Include the migration plan that avoids downtime.

4. The CTO says: "We're going to need Kubernetes within 6 months based on our growth." Design the ECS-to-EKS migration plan. Include: the prerequisite work (containerization improvements, service discovery migration, secrets management migration), the migration sequence (which service moves first), the parallel-run strategy (old and new running simultaneously), the cutover plan, and the rollback plan. Timeline this realistically for a team of 8 engineers (they've hired 2 more).

---

## Phase 8, Lesson 2: Migration & Modernization

### Lesson Preamble

```
You've been hired as a Senior Platform Engineer at "ShipFast" — a
logistics company with 200 engineers. ShipFast has:

  - 40+ microservices running on EC2 instances managed by Ansible
  - Jenkins CI/CD (version 2.x, 300+ pipelines, 5 years of tech debt)
  - PostgreSQL on self-managed EC2 (not RDS)
  - Redis on self-managed EC2 (not ElastiCache)
  - Monitoring: Nagios + custom scripts
  - No container adoption yet
  - Deployments are SSH + rsync to EC2 instances
  - 3 environments: dev, staging, prod (all manually provisioned)
  - Monthly deployment frequency (fear of deploying more often)
  - Last major outage: 3 weeks ago (bad deployment, 4-hour recovery)

The VP of Engineering has budget approval for a "modernization initiative"
and wants to move to containers + Kubernetes within 18 months.

Your job: design the migration from this legacy state to a modern
container platform without disrupting a running business.
```

### Questions

#### Q1: Migration Strategy Design 🔥

**Scenario:** The VP says: "I want everything on Kubernetes in 18 months." 200 engineers are watching. The last major infrastructure change (migrating from physical servers to EC2) took 2 years and caused 6 major outages.

1. The VP wants a big-bang migration. Explain why this will fail. Then design the phased migration strategy. How do you sequence 40+ services? What criteria determine which service migrates first? What's the minimum viable "platform" that needs to exist before the first service migrates? Include the 18-month timeline with milestones.

2. The first 3 months should focus on foundations, not migration. Design what gets built in the "Phase 0" period: the EKS cluster, the CI/CD pipeline, the observability stack, the networking, and the security baseline. Include the Terraform module structure and the critical design decisions (cluster per environment vs. namespace isolation, managed node groups vs. Karpenter, etc.).

3. You need a "migration factory" — a repeatable process that any team can use to containerize and migrate their service. Design the migration template: the Dockerfile standards, the Helm chart template, the CI/CD pipeline template, the testing strategy for validating the containerized version against the EC2 version, and the traffic cutover procedure. Include the documentation that goes to the 200 engineers.

4. 6 months in, 8 services have been migrated. 32 remain on EC2. You're running a hybrid environment where some services are on EKS and others are on EC2, and they need to communicate. Design the networking strategy for the hybrid period: service discovery across EKS and EC2, load balancing, DNS, and security. How do you ensure that the hybrid state doesn't create more operational complexity than either pure state?

---

#### Q2: Database Modernization 🔥

**Scenario:** ShipFast runs PostgreSQL 11 on a self-managed EC2 instance (r5.4xlarge). The database has 2TB of data, handles 15,000 transactions/second at peak, and hasn't been upgraded in 3 years. There are no read replicas. Backups are pg_dump to S3, running nightly. The DBA left 6 months ago.

1. This database is a ticking time bomb. List every risk (at least 7) and prioritize them by likelihood × impact. Then design the immediate stabilization plan (before any migration) that reduces the top 3 risks within 2 weeks. Include exact commands and configurations.

2. Design the migration from self-managed PostgreSQL on EC2 to RDS PostgreSQL (or Aurora). This is a 2TB database serving production traffic — the migration must be near-zero-downtime. Compare the options: pg_dump/restore (downtime), AWS DMS (CDC-based), logical replication, and Aurora PostgreSQL migration. For each, state the expected downtime, the complexity, the risk, and your recommendation.

3. The application uses PostgreSQL-specific features: advisory locks, listen/notify, custom extensions (PostGIS, pg_trgm). How do you validate that RDS/Aurora supports all of these BEFORE migration? Design the compatibility testing process. What's the plan if a critical extension isn't supported?

4. After migrating to RDS, design the operational runbook for the database. The DBA is gone and won't be replaced (the VP says "RDS is managed, we don't need a DBA"). This is dangerously wrong — explain why, then design the minimal database operations that the platform team must own: monitoring, alerting, backup verification, performance tuning, and upgrade planning. Include the CloudWatch alarms and the RDS Performance Insights queries.

---

#### Q3: CI/CD Modernization 🔥

**Scenario:** ShipFast's Jenkins instance has 300+ pipelines, some untouched for 3 years. Builds take 45 minutes on average. The Jenkins server is a single EC2 instance with no HA. Plugins haven't been updated in 18 months. 3 of the 300 pipelines are for the build server itself and nobody understands them.

1. The VP wants to replace Jenkins with a modern CI/CD platform. Evaluate: GitHub Actions, GitLab CI, ArgoCD + Tekton, and AWS CodePipeline. For ShipFast's specific situation (200 engineers, 40+ services, migrating to K8s), recommend one with detailed justification. Address: migration effort, learning curve, cost, Kubernetes integration, and the 300 existing pipelines.

2. You can't migrate 300 pipelines at once. Design the migration strategy: which pipelines move first, how do you run old Jenkins and new platform in parallel, how do you handle pipelines that depend on Jenkins-specific plugins, and what's the decommission criteria for Jenkins? Include the timeline.

3. Build times are 45 minutes. This is killing developer productivity (200 engineers × 5 builds/day × 45 min wait = 750 hours/day of wait time). Design the build optimization strategy: Docker layer caching, dependency caching, parallel test execution, and selective builds (only build/test what changed). Calculate the expected reduction in build time and the dollar value of recovered engineering productivity.

4. The deployment frequency is monthly because teams are afraid to deploy. Design the path from monthly to daily deployments: what technical changes (automated testing, canary deployments, feature flags), what process changes (deploy trains, deployment windows), and what cultural changes (blameless post-mortems, deployment celebrations) are needed? Include the metrics that track progress.

---

#### Q4: Observability Migration 🔥

**Scenario:** ShipFast uses Nagios for monitoring. Alerts are noisy (300+ alerts/week, 90% are false positives or informational). Engineers have alert fatigue and ignore most notifications. The on-call rotation has 30% turnover because engineers hate being on-call. There are no distributed traces, no centralized logging, and metrics are host-based (CPU, memory, disk) not application-based.

1. Design the modern observability stack for ShipFast. Cover all three pillars: metrics (Prometheus + Grafana vs. Datadog vs. CloudWatch), logs (ELK vs. Loki vs. CloudWatch Logs), traces (Jaeger vs. Tempo vs. X-Ray). For each pillar, compare the options considering: cost at ShipFast's scale, operational burden, Kubernetes integration, and the learning curve for 200 engineers. Recommend the stack with justification.

2. The alert noise problem must be fixed before anything else. Design the alert rationalization process: how do you audit 300+ Nagios alerts, which ones to keep, which to delete, which to convert to dashboards? Include the criteria for what qualifies as a PagerDuty-level alert vs. a Slack notification vs. a dashboard metric. How many alerts should a well-monitored 40-service platform have?

3. Implement SLOs for ShipFast. They've never had SLOs. Design the SLO framework: which services get SLOs first, what SLIs to measure, what targets to set, and how error budgets are calculated and communicated to teams. Include the Prometheus recording rules and the Grafana dashboard for the top 3 services.

4. The on-call rotation has 30% turnover. This is a people problem as much as a technical problem. Design the improvements: better runbooks (what goes in them), better escalation policies, better on-call compensation, better tooling (what does the on-call engineer need at 3 AM that they don't have today?), and better post-incident process. Include an example runbook for ShipFast's most critical service.

---

## Phase 8, Lesson 3: Multi-Account & Multi-Environment Strategy

### Lesson Preamble

```
NovaMart has grown. They now have 4 product teams, a platform team,
a data team, and a security team. All sharing a single AWS account.
Last week, a developer in the data team accidentally deleted a
production S3 bucket while testing a Terraform module (they were
in the wrong AWS profile). This incident triggered the decision to
implement a multi-account strategy.

You are designing the AWS Organization structure from the current
single-account mess to a properly segmented multi-account architecture.
```

### Questions

#### Q1: Organization Design 🔥

**Scenario:** NovaMart currently runs everything in a single AWS account: production, staging, dev, CI/CD, logging, and security tooling. 47 IAM users, 23 IAM roles, and nobody knows what half of them do.

1. Design the AWS Organization structure. How many accounts, what purpose for each, how are they organized into OUs? Include: security account, logging account, shared services account, production workload accounts, non-production accounts, and sandbox accounts. Justify why each account exists and what blast radius it limits. Include the OU hierarchy diagram.

2. Design the SCP strategy for each OU. What actions are denied at each level? Include SCPs for: preventing region sprawl (only us-east-1 and us-west-2), preventing public S3 buckets, preventing CloudTrail deletion, preventing root user usage, and restricting instance types in non-production. Include the JSON for at least 3 critical SCPs.

3. Design the identity strategy. NovaMart uses AWS IAM Identity Center (SSO) with Okta as the IdP. Map out: the permission sets for each role (developer, SRE, security engineer, data engineer, read-only), which accounts each role can access, and the temporary elevation process for production access. Include the permission set YAML and the access matrix.

4. The migration from single-account to multi-account is complex. Design the migration plan: what moves first, how do you handle resources that reference account IDs (IAM roles, S3 bucket policies, KMS keys), how do you migrate Terraform state, and how do you ensure zero downtime during the transition? Include the sequence and the specific challenges for each resource type.

---

#### Q2: Network Architecture Across Accounts 🔥

**Scenario:** With 8+ AWS accounts, each needing VPCs, the network architecture becomes critical. Services in the production account need to reach RDS in the data account. The CI/CD account needs to push images to ECR in the shared services account. Developers need to reach staging environments.

1. Design the network topology: Transit Gateway vs. VPC Peering vs. PrivateLink for each connection pattern. When is each appropriate? What are the cost differences? Include the full network diagram showing all accounts, VPCs, subnets, and connections.

2. Design the IP address management (IPAM) strategy. With 8+ VPCs, how do you prevent CIDR overlap? How do you plan for future growth? Include the CIDR allocation table for all accounts and VPCs, the AWS VPC IPAM configuration, and the process for requesting new CIDR blocks.

3. DNS resolution across accounts is a common pain point. Design the Route53 architecture: which account owns the public hosted zone, how do private hosted zones work across accounts (Route53 Resolver), and how does service discovery work for EKS services that need to reach resources in other accounts? Include the Terraform configuration.

4. A developer in the sandbox account should NEVER be able to reach production databases. Design the network segmentation that enforces this: Transit Gateway route tables, security groups, NACLs, and any additional controls. Then verify: if a developer tries every possible network path, can they reach production? Walk through each potential path and prove it's blocked.

---

## Phase 8, Lesson 4: Resilience & Disaster Recovery

### Lesson Preamble

```
NovaMart's current DR strategy is "hope nothing bad happens." There's
no documented RPO or RTO. Backups exist but have never been tested.
The EKS cluster runs in a single region (us-east-1). The CTO has
asked for a DR plan after reading about a competitor's 12-hour outage
that cost them $2M in lost revenue.
```

### Questions

#### Q1: DR Strategy Design 🔥

**Scenario:** The CTO asks: "If us-east-1 goes down completely, how long until we're back online?" Your honest answer: "I don't know. Probably days." The CTO says: "Make it hours."

1. Define RPO and RTO targets for each NovaMart service tier. Tier 1 (payments, orders): RPO ?, RTO ? Tier 2 (search, users): RPO ?, RTO ? Tier 3 (analytics, notifications): RPO ?, RTO ? Justify each target with business impact analysis. Include the cost of each level of DR (hot standby vs. warm vs. cold vs. backup-only) and recommend the appropriate DR strategy for each tier.

2. Design the multi-region DR architecture for NovaMart. Cover: EKS cluster in the DR region (active-passive vs. active-active), RDS cross-region replicas, S3 cross-region replication, Route53 failover routing, and certificate management. Include the Terraform for the DR region infrastructure and the estimated monthly cost of maintaining it.

3. The DR plan is worthless if it's never tested. Design the DR testing program: what to test, how often, how to test without impacting production, and how to measure success. Include the "game day" runbook for a simulated us-east-1 failure. What do you expect to break during the first test? (Be honest — list at least 5 things that will fail.)

4. The CTO proposes active-active multi-region. Explain the complexity and cost implications compared to active-passive. For NovaMart's specific situation (e-commerce platform, PostgreSQL database, eventual consistency challenges), is active-active justified? What are the data consistency challenges? Design the solution if active-active IS chosen, addressing: database replication strategy, conflict resolution, and request routing.

---

#### Q2: Backup & Recovery Verification 🔥

**Scenario:** Post-incident review reveals that NovaMart has never tested restoring from backup. RDS automated backups are enabled. S3 versioning is on. EKS cluster state is "in Git" (GitOps). But nobody has verified that a full restore actually works.

1. Design the backup verification program. For each data store (RDS, S3, Redis, EKS cluster state), define: what's backed up, where it's stored, how to verify it's restorable, and the automated test that proves it works. Include the Lambda function or CronJob that automatically restores to a test environment weekly and verifies data integrity.

2. The payments database is the crown jewel. If it's lost, NovaMart may not survive as a business. Design the backup strategy with: automated daily backups, cross-region replication, point-in-time recovery testing, and the runbook for "payments database is completely gone — restore from backup." Include the exact commands and the expected recovery time.

3. EKS cluster state recovery is often overlooked. If the EKS cluster is destroyed, what's in Git (manifests, configs) and what ISN'T (PVCs, running state, certificates, secrets)? Design the cluster recreation runbook that goes from "the cluster doesn't exist" to "all services are running and serving traffic." Include what you learn from this exercise about what SHOULD be in Git but currently isn't.

4. Kubernetes secrets are often the forgotten backup. If the cluster is destroyed, secrets in etcd are gone. If you're using External Secrets Operator pulling from Secrets Manager, recovery is straightforward. But what about: TLS certificates (cert-manager), service mesh certificates, ArgoCD credentials, and any secrets that were created manually via kubectl? Audit NovaMart's secret management and identify every gap in recoverability. Design the fix.

---

## Phase 8, Lesson 5: Capstone — Full Platform Design

### Lesson Preamble

```
This is the capstone. You are designing an ENTIRE platform from scratch,
integrating everything from Phases 1-8. There is no "right answer" —
there are tradeoffs, and your job is to make them explicitly and
defend them.

SCENARIO: You are the founding Platform Engineer at "UrbanFleet" — a
Series B startup ($25M raised) building an autonomous vehicle fleet
management platform. UrbanFleet's product:

  - Real-time tracking of 10,000 vehicles across 5 cities
  - Ingests telemetry data: GPS, speed, battery, sensor data
    (50 million events/day, growing 40% quarterly)
  - Driver and fleet manager web dashboard
  - Mobile app for drivers
  - REST API for partner integrations
  - ML pipeline for route optimization and predictive maintenance

The team: 45 engineers (25 backend, 10 frontend, 5 ML, 3 mobile, 2 QA)
Current state: Running on GCP (GKE) but switching to AWS for
  partnership reasons. Must be migrated within 6 months.

Compliance: SOC 2 Type II (already certified on GCP — must maintain
  on AWS). Vehicle telemetry data may fall under IoT regulations
  depending on city.
```

### Questions

#### Q1: Foundation Architecture 🔥

Design the complete AWS architecture for UrbanFleet. This is an open-ended design question. Your answer must cover:

1. **Compute platform:** EKS architecture, node groups, instance types, autoscaling strategy. Justify your choices against alternatives (ECS, Lambda). Address the real-time telemetry ingestion requirement — 50M events/day means ~580 events/second average, but vehicle telemetry is bursty (rush hours could be 5-10x average). What instance types handle this? How does the cluster scale?

2. **Data architecture:** Where does telemetry data land (Kinesis? MSK? SQS?)? Where is it stored long-term (S3? TimescaleDB? DynamoDB?)? How is it queried for the dashboard (real-time) vs. the ML pipeline (batch)? Design the data flow from vehicle → cloud → storage → API → dashboard. Include throughput calculations.

3. **Network architecture:** Multi-account strategy, VPC design, ingress (API Gateway vs. ALB vs. NLB for WebSocket telemetry), egress, and inter-service communication. The vehicle-to-cloud connection uses MQTT — how does this fit into the AWS architecture?

4. **Cost estimation:** Provide a monthly cost estimate for the full architecture at current scale (50M events/day, 10K vehicles) and at 3x scale. Break down by service category. Identify the top 3 cost drivers and the optimization levers for each.

---

#### Q2: Migration from GCP 🔥

UrbanFleet currently runs on GCP with GKE, Cloud SQL (PostgreSQL), Cloud Pub/Sub, BigQuery, and GCS. The migration to AWS must happen within 6 months without customer-facing downtime.

1. Map every GCP service to its AWS equivalent. For each, identify the migration complexity (drop-in replacement vs. significant rework). Which migrations are easy? Which are hard? Which require application code changes?

2. Design the 6-month migration plan. What moves first? How do you handle the period when parts of the system are on GCP and parts are on AWS? Include: data migration strategy (2TB in Cloud SQL, 15TB in GCS, real-time events in Pub/Sub), DNS cutover, and the parallel-running period.

3. BigQuery to what? UrbanFleet's ML team uses BigQuery heavily for ad-hoc analysis and ML training data preparation. Evaluate: Redshift, Athena + S3, EMR + Spark, and Redshift Serverless. Recommend one with justification. Address the ML team's workflow: they write SQL queries in notebooks. What changes for them?

4. The mobile app and partner API can't go down during migration. Design the traffic routing strategy that allows gradual migration: some traffic to GCP, some to AWS, with the ability to roll back. Include: DNS configuration, health checks, and the metrics that tell you "AWS is ready to take 100% of traffic."

---

#### Q3: Security & Compliance 🔥

UrbanFleet has SOC 2 Type II on GCP. Moving to AWS means re-establishing compliance controls.

1. Map SOC 2 controls to AWS services. For each control category (access control, change management, availability, confidentiality, privacy), identify the AWS service that satisfies it. What was handled by GCP-native features that you now need to implement yourself?

2. Vehicle telemetry data includes GPS location — this is PII under GDPR (if UrbanFleet operates in Europe) and potentially under state privacy laws (CCPA). Design the data classification and protection scheme: what's PII, what's not, how is PII encrypted differently, how is access to PII audited, and how do you handle a "right to be forgotten" request for a vehicle's location history?

3. The partner API handles authentication for external companies. Design the API security architecture: API Gateway with usage plans, OAuth2/JWT authentication, rate limiting, WAF rules, and the threat model for the API. What are the top 5 attack vectors for a fleet management API specifically?

4. UrbanFleet's ML pipeline processes raw telemetry and outputs route recommendations that control real vehicles. A compromised ML model could route vehicles dangerously. Design the ML pipeline security: model signing, training data integrity, inference endpoint security, and the human-in-the-loop safeguards for model deployment. This is a safety-critical system — treat it accordingly.

---

#### Q4: Operational Readiness 🔥

You've designed the architecture, planned the migration, and addressed security. Now: make it operationally ready.

1. Design the observability stack for UrbanFleet. The key challenge: correlating vehicle telemetry events with backend service behavior. When a fleet manager reports "vehicle 4521 location hasn't updated for 10 minutes," the on-call engineer needs to trace: did the vehicle stop sending? Did the MQTT broker drop the message? Did the processing pipeline lag? Did the database write fail? Did the API cache serve stale data? Design the tracing and correlation strategy end-to-end.

2. Define SLOs for UrbanFleet's critical user journeys: (a) vehicle location updates visible on dashboard within 5 seconds of the event, (b) API response time p99 < 200ms, (c) historical telemetry query returns within 10 seconds for 30-day range. For each SLO, define the SLI, the measurement method, the error budget, and what happens when the budget is exhausted.

3. Design the on-call structure for a 45-person engineering team. Not everyone should be on-call. Who is? How are rotations structured? What's the escalation path? What runbooks exist for the top 5 most likely incidents? Include one complete runbook for "telemetry pipeline is backed up — vehicle locations are 5+ minutes stale."

4. Write the "Day 2 Operations" document. This is the document that lists everything that needs regular human attention AFTER the platform is running: certificate renewals, dependency updates, Kubernetes version upgrades, database maintenance windows, cost reviews, security patching, compliance audits, capacity planning, and DR testing. For each item, specify the frequency, the owner, and the process. This is the document that prevents "it worked when we set it up and then nobody touched it for 2 years."

---

# Integration Notes

```
══════════════════════════════════════════════════════════════════
CURRICULUM SEQUENCE (Updated)
══════════════════════════════════════════════════════════════════

Phase 1: Linux Fundamentals
Phase 2: Git & Version Control
Phase 3: Docker & Containers
Phase 4: Networking
Phase 5: CI/CD Pipelines
Phase 5b: Infrastructure as Code
Phase 6: Kubernetes + Security + Observability & SRE
Phase 7: Cost Optimization & FinOps              ← NEW
Phase 8: Architectural Design                     ← NEW

DEPENDENCY CHAIN:
  Phase 7 requires: Phases 5b (Terraform), 6 (K8s + AWS)
  Phase 8 requires: ALL previous phases (this is the integration layer)

  Phase 8 Lesson 5 (Capstone) is explicitly designed as a
  "final exam" that requires knowledge from every previous phase.

TEACHING PROGRESSION:
  Phases 1-6: "Here is the platform. Operate it. Fix it. Secure it."
  Phase 7:    "The platform costs too much. Optimize it."
  Phase 8:    "There is no platform. Design it from scratch."

  This progression mirrors career growth:
    Junior:  Operate existing systems (Phases 1-5)
    Mid:     Secure and optimize systems (Phase 6-7)
    Senior:  Design new systems (Phase 8, Lessons 1-3)
    Staff/Principal: Design systems for complex orgs (Phase 8, L4-5)

```
══════════════════════════════════════════════════════════════════
GRADING RUBRICS FOR PHASES 7-8
══════════════════════════════════════════════════════════════════

PHASE 7 GRADING CRITERIA:

  5/5 — Answer includes:
    ✓ Exact dollar calculations with shown math
    ✓ Comparison of at least 2 alternatives with cost breakdown
    ✓ Risk assessment for each optimization
    ✓ Implementation plan with rollback strategy
    ✓ Organizational considerations (who approves, who's impacted)
    ✓ Monitoring to verify savings are realized
    ✓ Long-term sustainability (guardrails against cost creep)

  4/5 — Answer includes most of above but:
    - Math is approximate rather than exact
    - OR missing rollback strategy
    - OR missing organizational dimension
    - OR missing monitoring/verification

  3/5 — Answer identifies correct optimizations but:
    - No dollar amounts
    - No risk assessment
    - Generic recommendations without NovaMart-specific analysis
    - No implementation plan

  2/5 — Answer is vague:
    - "Use Spot instances to save money"
    - "Right-size your pods"
    - No specifics, no math, no plan

  1/5 — Answer is wrong:
    - Recommends optimizations that would cause outages
    - Math errors that lead to wrong conclusions
    - Ignores business constraints


PHASE 8 GRADING CRITERIA:

  5/5 — Answer includes:
    ✓ Complete architecture with ALL components addressed
    ✓ Explicit tradeoffs stated for every major decision
    ✓ Alternatives considered and rejected WITH reasoning
    ✓ Cost estimation for the proposed architecture
    ✓ Migration/implementation plan with realistic timeline
    ✓ Failure modes identified and mitigated
    ✓ Organizational fit (team size, skill level, budget)
    ✓ Evolution path (how does this architecture grow?)
    ✓ Security and compliance addressed as first-class concerns
    ✓ Operational readiness (not just "build it" but "run it")

  4/5 — Answer includes most of above but:
    - Missing cost estimation
    - OR tradeoffs mentioned but not deeply analyzed
    - OR migration plan is vague on sequencing
    - OR operational concerns are afterthought

  3/5 — Answer provides a reasonable architecture but:
    - No tradeoff analysis ("just use EKS" without justification)
    - No cost estimation
    - No migration plan
    - Ignores team size or budget constraints
    - Security bolted on rather than designed in

  2/5 — Answer is a technology wishlist:
    - Lists tools without architecture
    - "Use Kubernetes, Terraform, ArgoCD, Istio, Prometheus"
    - No design, no reasoning, no tradeoffs

  1/5 — Answer is architecturally unsound:
    - Proposes patterns that won't work at stated scale
    - Ignores compliance requirements
    - Would bankrupt the startup
    - Ignores stated constraints entirely

══════════════════════════════════════════════════════════════════
ANSWER QUALITY EXPECTATIONS BY PHASE
══════════════════════════════════════════════════════════════════

Phase 7 answers should read like a CONSULTANT'S DELIVERABLE to a CFO.
  - Executive summary with bottom-line savings number
  - Detailed analysis with math
  - Risk assessment
  - Implementation roadmap
  - ROI calculation

Phase 8 answers should read like an ARCHITECTURE DECISION RECORD.
  - Context: what problem are we solving
  - Decision: what we chose
  - Alternatives: what we considered and rejected
  - Consequences: what we gain and what we lose
  - Implementation: how we execute
  - Review criteria: how we know if we were right

══════════════════════════════════════════════════════════════════
CROSS-PHASE INTEGRATION POINTS
══════════════════════════════════════════════════════════════════

Phase 7 builds on:
  - Phase 5b (Terraform): All infrastructure changes for cost
    optimization must be expressed as IaC
  - Phase 6, Lesson 1 (Kubernetes): Pod resource management,
    HPA, VPA, node autoscaling
  - Phase 6, Lesson 2 (Secrets): Cost of secrets management
    services (Secrets Manager vs SSM Parameter Store pricing)
  - Phase 6, Lesson 3 (Security): Security controls have cost
    implications (Shield Advanced = $3K/month, WAF per-rule
    pricing, GuardDuty per-event pricing)
  - Phase 6, Lesson 4 (Network): Data transfer costs are
    directly tied to network architecture decisions

Phase 8 builds on:
  - EVERYTHING. Phase 8 is the integration layer.
  - Phase 1 (Linux): Base OS decisions for containers
  - Phase 2 (Git): Branching strategy affects CI/CD design
  - Phase 3 (Docker): Container image strategy
  - Phase 4 (Networking): VPC design, DNS, load balancing
  - Phase 5 (CI/CD): Pipeline architecture
  - Phase 5b (IaC): Terraform module structure, state management
  - Phase 6 (K8s + Security + Observability): Cluster design,
    security baseline, monitoring stack
  - Phase 7 (Cost): Every architectural decision has a cost
    implication that must be quantified

══════════════════════════════════════════════════════════════════
SCENARIO COMPANIES SUMMARY
══════════════════════════════════════════════════════════════════

NovaMart (Phases 1-7):
  - E-commerce platform
  - Established company with existing infrastructure
  - 18-node EKS cluster, $187K/month AWS spend
  - Focus: operate, secure, optimize

MedRelay (Phase 8, Lesson 1):
  - Healthcare SaaS startup
  - 6 engineers, $4M runway
  - HIPAA compliance
  - Focus: greenfield design with constraints

ShipFast (Phase 8, Lessons 2-3):
  - Logistics company, 200 engineers
  - Legacy infrastructure (EC2, Ansible, Jenkins, Nagios)
  - Focus: migration and modernization

UrbanFleet (Phase 8, Lessons 4-5 / Capstone):
  - Autonomous vehicle fleet management
  - Series B, 45 engineers
  - GCP to AWS migration
  - Real-time IoT + ML + compliance
  - Focus: full-stack architecture design

Each company represents a different archetype:
  NovaMart  = "optimize what exists"
  MedRelay  = "build from nothing with tight constraints"
  ShipFast  = "modernize legacy at scale"
  UrbanFleet = "complex multi-domain architecture"

Together they cover the four most common scenarios a platform
engineer will face in their career.

══════════════════════════════════════════════════════════════════
SKILLS PROGRESSION MAP (FULL CURRICULUM)
══════════════════════════════════════════════════════════════════

Phase 1-3 (Linux + Git + Docker):
  YOU CAN: Build and containerize applications
  YOU CAN: Navigate and troubleshoot Linux systems
  YOU CAN: Manage code and collaborate with version control
  JOB LEVEL: Junior DevOps / Junior SRE

Phase 4-5 (Networking + CI/CD + IaC):
  YOU CAN: Design networks and automate infrastructure
  YOU CAN: Build deployment pipelines end-to-end
  YOU CAN: Express infrastructure as code
  JOB LEVEL: Mid-level DevOps / Mid-level Platform Engineer

Phase 6 (Kubernetes + Security + Observability):
  YOU CAN: Operate and secure Kubernetes clusters
  YOU CAN: Respond to security incidents
  YOU CAN: Design monitoring and alerting systems
  YOU CAN: Implement defense-in-depth security
  JOB LEVEL: Senior DevOps / Senior Platform Engineer

Phase 7 (Cost Optimization):
  YOU CAN: Optimize cloud spend without sacrificing reliability
  YOU CAN: Build cost governance systems
  YOU CAN: Communicate cost tradeoffs to business stakeholders
  YOU CAN: Make data-driven resource allocation decisions
  JOB LEVEL: Senior Platform Engineer / Staff Engineer

Phase 8 (Architectural Design):
  YOU CAN: Design platforms from scratch
  YOU CAN: Plan and execute large-scale migrations
  YOU CAN: Make architectural decisions under uncertainty
  YOU CAN: Balance competing concerns across the full stack
  YOU CAN: Communicate architecture to technical and non-technical audiences
  JOB LEVEL: Staff Engineer / Principal Engineer

══════════════════════════════════════════════════════════════════
COMPLETION CRITERIA
══════════════════════════════════════════════════════════════════

A student has "completed" the curriculum when they can:

  1. Answer any Phase 8 Capstone question at a 4.5+/5 level
  2. Design a production platform from a blank AWS account
  3. Migrate a legacy system to a modern stack with a credible plan
  4. Respond to a security incident with the correct first 10 actions
  5. Optimize a $200K/month AWS bill by 25% with exact math
  6. Communicate technical decisions to a non-technical CFO/CTO
  7. Acknowledge tradeoffs and state what they're giving up
  8. Identify what they DON'T know and how they'd find out

That last point is crucial. The goal isn't omniscience.
The goal is: competence + self-awareness + the ability to learn
fast when you encounter something new.

══════════════════════════════════════════════════════════════════
POST-CURRICULUM RECOMMENDATIONS
══════════════════════════════════════════════════════════════════

After completing all 8 phases, the student should:

  1. BUILD a real platform:
     - Personal AWS account (free tier + ~$50-100/month)
     - Deploy a 3-tier app on EKS
     - Implement everything from Phases 1-8
     - Break it. Fix it. Break it differently.

  2. CONTRIBUTE to open source:
     - Contribute to Kubernetes, Terraform providers, or
       security tooling
     - This builds real-world skills AND professional visibility

  3. GET CERTIFIED (in this order):
     - CKA (Certified Kubernetes Administrator)
     - AWS Solutions Architect Professional
     - CKS (Certified Kubernetes Security Specialist)
     - These validate knowledge but don't replace it

  4. JOIN an on-call rotation:
     - Volunteer for on-call at work
     - This is where theoretical knowledge becomes operational skill
     - No course can replace the 3 AM adrenaline

  5. WRITE about what you've learned:
     - Blog posts, internal documentation, conference talks
     - Teaching others is the fastest way to find gaps in
       your own understanding

  6. STAY CURRENT:
     - Subscribe to: LWKD (Last Week in Kubernetes Development),
       AWS Week in Review, tl;dr sec newsletter
     - Follow: Kubernetes SIGs, AWS re:Invent sessions,
       CNCF project updates
     - The landscape changes every 6 months. Stagnation is decay.

══════════════════════════════════════════════════════════════════
```

---

## Phase 7 & 8 Summary

```
┌──────────────────────────────────────────────────────────────────┐
│                    PHASE 7: COST OPTIMIZATION                     │
├──────────┬───────────────────────────────────────────────────────┤
│ Lesson 1 │ Kubernetes Resource Economics                         │
│          │ Right-sizing, VPA, HPA, QoS, bin-packing, Karpenter  │
│          │ 4 scenario questions, each with 4 sub-parts           │
├──────────┼───────────────────────────────────────────────────────┤
│ Lesson 2 │ AWS Cost Management                                   │
│          │ Savings Plans, data transfer, NAT Gateway, S3, RDS   │
│          │ 4 scenario questions, each with 4 sub-parts           │
├──────────┼───────────────────────────────────────────────────────┤
│ Lesson 3 │ FinOps Culture & Organization                         │
│          │ Tagging, attribution, showback, engineering culture   │
│          │ 2 scenario questions, each with 4 sub-parts           │
├──────────┼───────────────────────────────────────────────────────┤
│ Lesson 4 │ Cost Optimization Under Pressure                      │
│          │ Budget crisis, cost vs reliability tradeoffs          │
│          │ 2 scenario questions, each with 4 sub-parts           │
├──────────┴───────────────────────────────────────────────────────┤
│ TOTAL: 12 scenarios, 48 sub-questions                            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                  PHASE 8: ARCHITECTURAL DESIGN                    │
├──────────┬───────────────────────────────────────────────────────┤
│ Lesson 1 │ Greenfield Platform Design (MedRelay)                 │
│          │ Technology decisions, HIPAA, small-team CI/CD,        │
│          │ scaling architecture                                  │
│          │ 4 scenario questions, each with 4 sub-parts           │
├──────────┼───────────────────────────────────────────────────────┤
│ Lesson 2 │ Migration & Modernization (ShipFast)                  │
│          │ EC2→K8s migration, DB modernization, CI/CD migration, │
│          │ observability migration                               │
│          │ 4 scenario questions, each with 4 sub-parts           │
├──────────┼───────────────────────────────────────────────────────┤
│ Lesson 3 │ Multi-Account & Multi-Environment (NovaMart)          │
│          │ AWS Organizations, SCPs, network across accounts,     │
│          │ identity strategy                                     │
│          │ 2 scenario questions, each with 4 sub-parts           │
├──────────┼───────────────────────────────────────────────────────┤
│ Lesson 4 │ Resilience & Disaster Recovery (NovaMart)             │
│          │ DR strategy, multi-region, backup verification,       │
│          │ secret recovery                                       │
│          │ 2 scenario questions, each with 4 sub-parts           │
├──────────┼───────────────────────────────────────────────────────┤
│ Lesson 5 │ CAPSTONE — Full Platform Design (UrbanFleet)          │
│          │ Complete architecture, GCP migration, security,       │
│          │ operational readiness                                 │
│          │ 4 scenario questions, each with 4 sub-parts           │
├──────────┴───────────────────────────────────────────────────────┤
│ TOTAL: 16 scenarios, 64 sub-questions                            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                 FULL CURRICULUM STATISTICS                         │
├──────────────────────────────────────────────────────────────────┤
│ Phases:           8                                               │
│ Companies:        4 (NovaMart, MedRelay, ShipFast, UrbanFleet)   │
│ Phase 7-8 Scenarios: 28                                           │
│ Phase 7-8 Sub-questions: 112                                      │
│ Career levels covered: Junior → Principal                         │
│ AWS services covered: 40+                                         │
│ K8s concepts covered: 30+                                         │
│ Compliance frameworks: HIPAA, SOC2, GDPR, CCPA                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## What These Two Phases Add to the Curriculum

**Phase 7 closes the "engineering decisions have financial consequences" gap.** Before Phase 7, a student could design a technically excellent platform that bankrupts the company. After Phase 7, they can design a technically excellent platform that the CFO approves. This is the difference between an engineer who builds what's cool and an engineer who builds what's sustainable.

**Phase 8 closes the "operate vs. architect" gap.** Before Phase 8, a student can operate NovaMart's existing platform exceptionally well. After Phase 8, they can walk into a company that has NOTHING and build the platform themselves. They can also walk into a company with a legacy mess and design the path to modern infrastructure. These are the two most valuable skills in the industry — and they're the skills that separate senior engineers ($180K) from staff/principal engineers ($300K+).

**The Capstone (Phase 8, Lesson 5) is the integration test.** UrbanFleet combines: real-time data ingestion, ML pipelines, IoT protocols, multi-cloud migration, compliance requirements, cost constraints, and operational readiness. A student who can design UrbanFleet's platform at a 4.5+/5 level has demonstrated competence across every domain in the curriculum. That's the graduation criterion.
