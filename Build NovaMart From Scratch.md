# Phase 7: Build NovaMart From Scratch

---

## The Shift

```
Phases 0-6: I taught. You learned. You answered questions.
Phase 7:    YOU build. I review. I challenge. You defend or fix.

This is not a lecture phase. This is a DESIGN + BUILD phase.

For each lesson, I will:
  1. Present REQUIREMENTS (what NovaMart needs)
  2. You DESIGN and BUILD (write Terraform, YAML, configs, docs)
  3. I REVIEW (critique every decision, find every gap)
  4. You ITERATE (fix issues, defend decisions)

Every shortcut will be caught.
Every missing production consideration will be flagged.
Every "I'll do that later" will be challenged.

You have 6 phases of knowledge. Now prove you can USE it.
```

```
PHASE 7 STRUCTURE:

  Lesson 1: Infrastructure Foundation
            VPC + EKS + Data Tier + Networking
            (The base everything else runs on)

  Lesson 2: Platform Services
            Observability + Security + Service Mesh
            (The platform that makes applications production-ready)

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

---

# Phase 7, Lesson 1: Infrastructure Foundation

## Your Mission

```
You are the Senior DevOps/Platform Engineer at NovaMart.
Day 1. The AWS account is empty. You need to build the foundation.

REQUIREMENTS:

  1. NETWORKING
     - Production VPC in us-east-1
     - 3 AZ deployment for high availability
     - Public, private, and data subnet tiers
     - EKS pods need real VPC IPs (VPC CNI)
     - Anticipate 2,000-5,000 pods at peak (Black Friday)
     - NAT Gateway for outbound internet access
     - VPC endpoints for AWS service access (cost optimization)
     - VPC Flow Logs for security forensics

  2. EKS CLUSTER
     - Production-grade EKS cluster
     - Managed node groups (not self-managed, not Fargate for core workloads)
     - Node sizing for 200+ microservices
     - Cluster autoscaling (Karpenter preferred)
     - IRSA enabled for pod-level AWS access
     - EKS add-ons (VPC CNI, CoreDNS, kube-proxy)
     - Encryption of secrets in etcd (KMS)
     - Private API endpoint (or public+private with restrictions)
     - Logging (control plane logs to CloudWatch)

  3. DATA TIER
     - RDS PostgreSQL (Multi-AZ) for payment-svc, order-svc, user-svc
     - ElastiCache Redis for session/cache
     - Networking: data tier accessible ONLY from EKS private subnets
     - Encryption at rest and in transit
     - Backup and retention policies

  4. SECURITY BASELINE
     - KMS keys (EKS secrets, RDS, S3, general encryption)
     - S3 buckets (Terraform state, CloudTrail, VPC Flow Logs, compliance)
     - CloudTrail (multi-region, data events for sensitive buckets)
     - Security Groups (ALB, EKS nodes, RDS, ElastiCache, VPC endpoints)

  5. TERRAFORM STRUCTURE
     - Module-based (not one giant main.tf)
     - Remote state (S3 + DynamoDB)
     - Environment separation strategy
     - Tagging strategy

CONSTRAINTS:
  - Everything in Terraform (no console clicks)
  - Production-ready (not "demo quality")
  - Cost-conscious (justify expensive choices)
  - Security-first (least privilege, encryption everywhere)

DELIVERABLES:
  1. Terraform module/file structure (show the directory tree)
  2. VPC module (full Terraform code)
  3. EKS module (full Terraform code)
  4. Data tier (RDS + ElastiCache Terraform)
  5. Security baseline (KMS, S3, CloudTrail, Security Groups)
  6. Architecture diagram (ASCII)
  7. Design decisions document (justify every non-obvious choice)
```

## What I'm Looking For

```
I will evaluate your build on:

  COMPLETENESS — Did you cover all requirements?
  PRODUCTION READINESS — Would this survive day 1 in production?
  SECURITY — Least privilege, encryption, network segmentation?
  COST AWARENESS — Did you justify expensive choices?
  OPERATIONAL MATURITY — Monitoring hooks, tagging, lifecycle policies?
  STRUCTURAL QUALITY — Module organization, naming, variable patterns?
  FAILURE MODE AWARENESS — What happens when components fail?

I will NOT accept:
  ❌ "I'll add monitoring later"
  ❌ "Security groups can be tightened later"
  ❌ "We can add encryption after launch"
  ❌ Hardcoded values that should be variables
  ❌ Missing tags
  ❌ Single-AZ anything in production
  ❌ Default security group usage
  ❌ Missing outputs that downstream modules need
```

---

## Go Build.

Write your complete infrastructure foundation. Show me the Terraform, the architecture diagram, and your design decisions.

I'll review everything you produce — structure, code, security, cost, and operational readiness. Every gap gets flagged. Every good decision gets acknowledged.

Take your time. This is the most important deliverable so far.


# NovaMart Infrastructure Foundation — Complete Build

---

## 1. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    AWS ACCOUNT: 888888888888                              │
│                                    REGION: us-east-1                                     │
│                                                                                          │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                           VPC: 10.0.0.0/16                                          │  │
│  │                    Secondary CIDR: 100.64.0.0/16 (Pod IPs)                          │  │
│  │                                                                                      │  │
│  │  ┌──────────────────────┬──────────────────────┬──────────────────────┐              │  │
│  │  │      AZ: us-east-1a  │    AZ: us-east-1b    │    AZ: us-east-1c   │              │  │
│  │  │                      │                       │                      │              │  │
│  │  │  ┌────────────────┐  │  ┌────────────────┐   │  ┌────────────────┐ │              │  │
│  │  │  │PUBLIC SUBNET   │  │  │PUBLIC SUBNET   │   │  │PUBLIC SUBNET   │ │              │  │
│  │  │  │10.0.0.0/22     │  │  │10.0.4.0/22     │   │  │10.0.8.0/22    │ │              │  │
│  │  │  │                │  │  │                │   │  │                │ │              │  │
│  │  │  │ ┌──────┐       │  │  │ ┌──────┐       │   │  │ ┌──────┐      │ │              │  │
│  │  │  │ │ NAT  │       │  │  │ │ NAT  │       │   │  │ │ NAT  │      │ │              │  │
│  │  │  │ │  GW  │       │  │  │ │  GW  │       │   │  │ │  GW  │      │ │              │  │
│  │  │  │ └──────┘       │  │  │ └──────┘       │   │  │ └──────┘      │ │              │  │
│  │  │  │    ALB         │  │  │    ALB          │   │  │    ALB        │ │              │  │
│  │  │  └────────────────┘  │  └────────────────┘   │  └────────────────┘ │              │  │
│  │  │                      │                       │                      │              │  │
│  │  │  ┌────────────────┐  │  ┌────────────────┐   │  ┌────────────────┐ │              │  │
│  │  │  │PRIVATE SUBNET  │  │  │PRIVATE SUBNET  │   │  │PRIVATE SUBNET  │ │              │  │
│  │  │  │10.0.16.0/20    │  │  │10.0.32.0/20    │   │  │10.0.48.0/20   │ │              │  │
│  │  │  │                │  │  │                │   │  │                │ │              │  │
│  │  │  │ ┌────────────┐ │  │  │ ┌────────────┐ │   │  │ ┌────────────┐│ │              │  │
│  │  │  │ │  EKS NODES │ │  │  │ │  EKS NODES │ │   │  │ │  EKS NODES ││ │              │  │
│  │  │  │ │ (m5.2xlarge│ │  │  │ │ (m5.2xlarge│ │   │  │ │ (m5.2xlarge││ │              │  │
│  │  │  │ │  + mixed)  │ │  │  │ │  + mixed)  │ │   │  │ │  + mixed)  ││ │              │  │
│  │  │  │ └────────────┘ │  │  │ └────────────┘ │   │  │ └────────────┘│ │              │  │
│  │  │  └────────────────┘  │  └────────────────┘   │  └────────────────┘ │              │  │
│  │  │                      │                       │                      │              │  │
│  │  │  ┌────────────────┐  │  ┌────────────────┐   │  ┌────────────────┐ │              │  │
│  │  │  │POD SUBNET      │  │  │POD SUBNET      │   │  │POD SUBNET     │ │              │  │
│  │  │  │100.64.0.0/18   │  │  │100.64.64.0/18  │   │  │100.64.128.0/18│ │              │  │
│  │  │  │(16,384 pod IPs)│  │  │(16,384 pod IPs)│   │  │(16,384 pod IPs│ │              │  │
│  │  │  └────────────────┘  │  └────────────────┘   │  └────────────────┘ │              │  │
│  │  │                      │                       │                      │              │  │
│  │  │  ┌────────────────┐  │  ┌────────────────┐   │  ┌────────────────┐ │              │  │
│  │  │  │DATA SUBNET     │  │  │DATA SUBNET     │   │  │DATA SUBNET    │ │              │  │
│  │  │  │10.0.64.0/22    │  │  │10.0.68.0/22    │   │  │10.0.72.0/22   │ │              │  │
│  │  │  │                │  │  │                │   │  │                │ │              │  │
│  │  │  │  RDS Primary/  │  │  │  RDS Standby   │   │  │  ElastiCache  │ │              │  │
│  │  │  │  ElastiCache   │  │  │  Replica        │   │  │  Replica      │ │              │  │
│  │  │  └────────────────┘  │  └────────────────┘   │  └────────────────┘ │              │  │
│  │  └──────────────────────┴──────────────────────┴──────────────────────┘              │  │
│  │                                                                                      │  │
│  │  ┌──────────────────────────────────────────────────────────────┐                    │  │
│  │  │                    VPC ENDPOINTS                              │                    │  │
│  │  │  S3 (Gateway) │ ECR (Interface) │ STS (Interface)            │                    │  │
│  │  │  CloudWatch Logs │ Secrets Manager │ KMS                     │                    │  │
│  │  └──────────────────────────────────────────────────────────────┘                    │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────┐        │
│  │                         SECURITY BASELINE                                     │        │
│  │  CloudTrail → S3 (encrypted, object lock)                                    │        │
│  │  VPC Flow Logs → CloudWatch Logs + S3                                        │        │
│  │  KMS Keys: eks-secrets | rds-encryption | s3-encryption | general            │        │
│  │  S3 Buckets: terraform-state | cloudtrail | flow-logs | compliance           │        │
│  └──────────────────────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Design Decisions Document

```markdown
# NovaMart Infrastructure — Design Decisions

## DD-001: VPC CIDR Strategy — Dual CIDR with Secondary Range for Pods

**Decision:** Use 10.0.0.0/16 as the primary CIDR for infrastructure
(nodes, databases, load balancers) and 100.64.0.0/16 as a secondary
CIDR exclusively for EKS pod IPs via VPC CNI custom networking.

**Rationale:**
- VPC CNI assigns real VPC IPs to every pod. At 5,000 peak pods,
  a single /16 would exhaust 7.6% of its address space on pods alone,
  competing with nodes, RDS, ENIs, and future growth.
- 100.64.0.0/16 is RFC 6598 (carrier-grade NAT) space — it won't
  conflict with corporate VPN ranges (typically 10.x or 172.x),
  enabling future VPC peering or Transit Gateway without CIDR collision.
- Three /18 pod subnets provide 16,384 IPs per AZ = 49,152 total
  pod IPs. At 5,000 peak pods, this gives 9.8x headroom.
- The ENI-per-node limit on m5.2xlarge is 15 ENIs × 15 IPs each = 225 
  pods per node. With /18 subnets, the pod IP pool is never the bottleneck.

**Alternatives considered:**
- Single CIDR (/16): Rejected — pod IP consumption would constrain 
  future infrastructure growth and create CIDR conflicts with peering.
- Prefix delegation: Considered but adds complexity. Secondary CIDR is
  simpler and equally effective at this scale.

**Cost impact:** Zero. Secondary CIDRs are free.

---

## DD-002: Three NAT Gateways (One Per AZ)

**Decision:** Deploy one NAT Gateway per AZ rather than sharing a single
NAT Gateway across AZs.

**Rationale:**
- Single NAT Gateway = single AZ. If that AZ goes down, ALL outbound
  internet access fails for every AZ. For a production e-commerce platform,
  this is unacceptable.
- Cross-AZ NAT Gateway traffic incurs $0.01/GB data transfer charges.
  With 200+ microservices making outbound calls, this adds up.
- NAT Gateway is $0.045/hr × 3 = $0.135/hr = ~$98/month for the
  gateway-hours. Compared to the risk of total outbound failure,
  this is trivially justified.

**Alternatives considered:**
- Single NAT Gateway: Rejected — unacceptable SPOF for production.
- NAT instances: Rejected — management overhead not justified; NAT 
  Gateway is managed, scales to 45 Gbps, and self-heals.

**Cost impact:** ~$98/month additional for 2 extra NAT Gateways.
Data processing charges ($0.045/GB) are identical regardless of count.

---

## DD-003: VPC Endpoints for AWS API Traffic

**Decision:** Deploy VPC endpoints for: S3 (Gateway), ECR API + ECR DKR 
(Interface), CloudWatch Logs (Interface), STS (Interface), Secrets Manager 
(Interface), KMS (Interface).

**Rationale:**
- Without VPC endpoints, every AWS API call from a pod traverses the NAT
  Gateway. ECR image pulls alone can generate hundreds of GB/month of
  NAT Gateway data processing charges at $0.045/GB.
- S3 Gateway endpoint is free. Interface endpoints cost $0.01/hr per AZ
  ($0.01 × 3 AZs × 730 hrs = ~$22/month per endpoint), but eliminate
  NAT Gateway data processing charges for that service.
- ECR is the highest-volume endpoint (image pulls for 200+ services).
  At ~500GB/month of image pulls, NAT processing would cost $22.50/month
  — the VPC endpoint pays for itself.
- STS and Secrets Manager endpoints also improve security: IRSA token 
  exchange and secret retrieval never leave the VPC.

**Cost impact:** ~$132/month for 6 interface endpoints (3 AZs).
Estimated NAT Gateway savings: ~$200-400/month. Net positive.

---

## DD-004: EKS API Endpoint — Public + Private with CIDR Restriction

**Decision:** Configure the EKS API endpoint as both public AND private,
with the public endpoint restricted to NovaMart's corporate IP ranges
and CI/CD runner IPs.

**Rationale:**
- Fully private endpoint requires a bastion host or VPN for all kubectl
  access, including CI/CD pipelines. This adds operational complexity.
- Public + private with CIDR restriction provides: (a) kubectl access
  from corporate network without VPN, (b) CI/CD access from known IPs,
  (c) pod-to-API-server traffic stays within VPC (via private endpoint).
- The public endpoint is authenticated (IAM/OIDC) AND network-restricted.
  An attacker without both valid credentials AND a whitelisted IP cannot
  reach the API server.

**Alternatives considered:**
- Fully private: Rejected for Day 1 — adds bastion complexity before
  VPN infrastructure is established. Will migrate to fully private in
  Phase 2 once VPN/SSM access is standardized.
- Fully public (unrestricted): Rejected — unnecessary exposure.

**Security tradeoff acknowledged:** The API server is reachable from 
specified public IPs. Mitigation: IAM authentication required, audit 
logging enabled, IP restriction limits attack surface.

---

## DD-005: EKS Managed Node Groups + Karpenter

**Decision:** Use a small managed node group for system workloads 
(CoreDNS, kube-proxy, Karpenter itself) and Karpenter for all 
application workload scaling.

**Rationale:**
- Karpenter cannot manage the nodes it runs on (chicken-and-egg).
  A small managed node group (~2 nodes) ensures Karpenter, CoreDNS,
  and system DaemonSets are always running.
- Karpenter provides: faster scaling (seconds vs minutes for Cluster
  Autoscaler), bin-packing optimization, mixed instance type selection,
  and Spot instance support — all critical for handling Black Friday 
  traffic spikes.
- Managed node groups handle AMI updates, node health, and lifecycle
  management for the system nodes, reducing operational burden for the
  most critical nodes in the cluster.

**Cost impact:** Karpenter itself is free (open source). The system
node group is 2× m5.large ($0.096/hr × 2 = ~$140/month) — minimal
fixed cost for cluster stability.

---

## DD-006: RDS Multi-AZ with Separate Instances per Service

**Decision:** Three separate RDS PostgreSQL instances (payments, orders,
users) rather than a shared instance with multiple databases.

**Rationale:**
- Blast radius isolation: a runaway query in the orders DB cannot
  affect payment transaction latency.
- Independent scaling: payments may need r6g.xlarge while users can
  run on r6g.large — right-sizing per workload.
- Independent maintenance windows: can upgrade the users DB without
  risking the payments DB.
- Independent backup/restore: can restore the orders DB to a point in
  time without affecting other databases.
- PCI-DSS scope: the payments DB has different compliance requirements
  than the users DB. Physical separation simplifies audit boundaries.

**Alternatives considered:**
- Single shared RDS instance: Rejected — blast radius and compliance.
- Aurora: Considered but deferred. Aurora's pricing model (per-IO)
  is harder to predict for a new platform. Starting with RDS PostgreSQL
  allows migration to Aurora later with minimal application changes.

**Cost impact:** 3 × db.r6g.large Multi-AZ = ~$1,100/month.
Shared instance would be ~$550/month. The $550/month premium buys
isolation, independent scaling, and compliance simplification.

---

## DD-007: Encryption Strategy — Dedicated KMS Keys per Domain

**Decision:** Four KMS keys: eks-secrets, rds-encryption, s3-encryption,
and general-encryption. Not one shared key.

**Rationale:**
- Key rotation policies differ: EKS secrets key rotates annually
  (automatic), RDS key has specific rotation constraints.
- Access policies differ: the EKS key should only be usable by the
  EKS service, the RDS key by the RDS service. Shared key = overly
  broad key policy.
- Audit clarity: CloudTrail KMS events show which key was used. 
  Separate keys make it immediately obvious which domain accessed
  which data.
- Blast radius: if a key is compromised or needs emergency rotation,
  only one domain is affected.

**Cost impact:** $1/month per key × 4 = $4/month. Trivial.

---

## DD-008: Terraform State — S3 + DynamoDB with Per-Module State Files

**Decision:** Terraform state stored in S3 with DynamoDB locking.
State split by logical domain: networking, eks, data, security.
Not a single monolithic state file.

**Rationale:**
- Monolithic state means every `terraform apply` locks everything.
  An engineer updating a security group blocks another engineer 
  updating an EKS add-on.
- Split state enables parallel work and limits blast radius of 
  `terraform destroy` accidents.
- State files are small and fast to load.

**State file layout:**
  - `novamart/production/networking/terraform.tfstate`
  - `novamart/production/eks/terraform.tfstate`
  - `novamart/production/data/terraform.tfstate`
  - `novamart/production/security/terraform.tfstate`

**Cross-state references via `terraform_remote_state` data sources.**

---

## DD-009: Tagging Strategy

**Decision:** Every resource gets 5 mandatory tags. Enforced via 
default_tags in the AWS provider and validated by a CI pipeline.

| Tag Key | Example Value | Purpose |
|---------|--------------|---------|
| Environment | production | Cost allocation, SCP conditions |
| Team | platform | Cost attribution, ownership |
| Service | eks-cluster | Resource identification |
| ManagedBy | terraform | Drift detection |
| CostCenter | PLATFORM-001 | Finance reporting |
```

---

## 3. Terraform Directory Structure

```
novamart-infrastructure/
├── README.md
├── docs/
│   ├── architecture-diagram.md
│   ├── design-decisions.md
│   ├── runbook-terraform-operations.md
│   └── cidr-allocation.md
│
├── modules/                          # Reusable modules
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── vpc-endpoints.tf
│   │   ├── flow-logs.tf
│   │   └── README.md
│   │
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── node-groups.tf
│   │   ├── addons.tf
│   │   ├── irsa.tf
│   │   ├── karpenter.tf
│   │   └── README.md
│   │
│   ├── rds/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   ├── elasticache/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   │
│   └── security-baseline/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── kms.tf
│       ├── s3.tf
│       ├── cloudtrail.tf
│       └── README.md
│
├── environments/
│   ├── production/
│   │   ├── networking/
│   │   │   ├── main.tf              # Calls modules/vpc
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── backend.tf
│   │   │   └── terraform.tfvars
│   │   │
│   │   ├── eks/
│   │   │   ├── main.tf              # Calls modules/eks
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── backend.tf
│   │   │   ├── data.tf              # Remote state refs
│   │   │   └── terraform.tfvars
│   │   │
│   │   ├── data/
│   │   │   ├── main.tf              # Calls modules/rds + elasticache
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── backend.tf
│   │   │   ├── data.tf
│   │   │   └── terraform.tfvars
│   │   │
│   │   └── security/
│   │       ├── main.tf              # Calls modules/security-baseline
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── backend.tf
│   │       └── terraform.tfvars
│   │
│   └── staging/                      # Same structure, different tfvars
│       └── ...
│
└── bootstrap/                        # Run once, manually
    ├── main.tf                       # S3 bucket + DynamoDB for state
    ├── variables.tf
    └── README.md
```

---

## 4. Terraform Code

### 4.1 Bootstrap — State Backend

```hcl
# bootstrap/main.tf
# Run ONCE manually: terraform init && terraform apply
# This creates the S3 bucket and DynamoDB table for remote state

terraform {
  required_version = ">= 1.7.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}

provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = "management"
      Team        = "platform"
      ManagedBy   = "terraform"
      Service     = "state-backend"
      CostCenter  = "PLATFORM-001"
    }
  }
}

data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "novamart-terraform-state-${data.aws_caller_identity.current.account_id}"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
      # Uses AWS-managed key for bootstrap — dedicated key created in security module
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "expire-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 90
    }

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "novamart-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }
}

output "state_bucket" {
  value = aws_s3_bucket.terraform_state.id
}

output "locks_table" {
  value = aws_dynamodb_table.terraform_locks.name
}
```

---

### 4.2 VPC Module

```hcl
# modules/vpc/variables.tf

variable "environment" {
  description = "Environment name (production, staging, dev)"
  type        = string
  validation {
    condition     = contains(["production", "staging", "dev"], var.environment)
    error_message = "Environment must be production, staging, or dev."
  }
}

variable "primary_cidr" {
  description = "Primary VPC CIDR block for infrastructure"
  type        = string
  default     = "10.0.0.0/16"
}

variable "secondary_cidr" {
  description = "Secondary CIDR block for EKS pod IPs (VPC CNI custom networking)"
  type        = string
  default     = "100.64.0.0/16"
}

variable "availability_zones" {
  description = "List of AZs to deploy into"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets (ALB, NAT GW)"
  type        = list(string)
  default     = ["10.0.0.0/22", "10.0.4.0/22", "10.0.8.0/22"]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets (EKS nodes)"
  type        = list(string)
  default     = ["10.0.16.0/20", "10.0.32.0/20", "10.0.48.0/20"]
}

variable "pod_subnet_cidrs" {
  description = "CIDR blocks for pod subnets (from secondary CIDR)"
  type        = list(string)
  default     = ["100.64.0.0/18", "100.64.64.0/18", "100.64.128.0/18"]
}

variable "data_subnet_cidrs" {
  description = "CIDR blocks for data subnets (RDS, ElastiCache)"
  type        = list(string)
  default     = ["10.0.64.0/22", "10.0.68.0/22", "10.0.72.0/22"]
}

variable "cluster_name" {
  description = "EKS cluster name — needed for subnet tagging"
  type        = string
}

variable "enable_vpc_endpoints" {
  description = "Create VPC endpoints for AWS services"
  type        = bool
  default     = true
}

variable "flow_log_retention_days" {
  description = "CloudWatch Logs retention for VPC Flow Logs"
  type        = number
  default     = 90
}

variable "flow_log_s3_bucket_arn" {
  description = "S3 bucket ARN for VPC Flow Log long-term storage"
  type        = string
  default     = ""
}

variable "tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/vpc/main.tf

locals {
  name_prefix = "novamart-${var.environment}"

  common_tags = merge(var.tags, {
    Module = "vpc"
  })
}

# ─────────────────────────────────────────────
# VPC
# ─────────────────────────────────────────────

resource "aws_vpc" "main" {
  cidr_block           = var.primary_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

resource "aws_vpc_ipv4_cidr_block_association" "secondary" {
  vpc_id     = aws_vpc.main.id
  cidr_block = var.secondary_cidr
}

# ─────────────────────────────────────────────
# INTERNET GATEWAY
# ─────────────────────────────────────────────

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-igw"
  })
}

# ─────────────────────────────────────────────
# PUBLIC SUBNETS
# ─────────────────────────────────────────────

resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name                                          = "${local.name_prefix}-public-${var.availability_zones[count.index]}"
    Tier                                          = "public"
    "kubernetes.io/role/elb"                      = "1"
    "kubernetes.io/cluster/${var.cluster_name}"    = "shared"
  })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# ─────────────────────────────────────────────
# NAT GATEWAYS (one per AZ)
# ─────────────────────────────────────────────

resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat-eip-${var.availability_zones[count.index]}"
  })

  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = length(var.availability_zones)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat-${var.availability_zones[count.index]}"
  })

  depends_on = [aws_internet_gateway.main]
}

# ─────────────────────────────────────────────
# PRIVATE SUBNETS (EKS Nodes)
# ─────────────────────────────────────────────

resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.common_tags, {
    Name                                          = "${local.name_prefix}-private-${var.availability_zones[count.index]}"
    Tier                                          = "private"
    "kubernetes.io/role/internal-elb"             = "1"
    "kubernetes.io/cluster/${var.cluster_name}"    = "shared"
    "karpenter.sh/discovery"                      = var.cluster_name
  })
}

resource "aws_route_table" "private" {
  count = length(var.availability_zones)

  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-rt-${var.availability_zones[count.index]}"
  })
}

resource "aws_route_table_association" "private" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# ─────────────────────────────────────────────
# POD SUBNETS (Secondary CIDR for VPC CNI)
# ─────────────────────────────────────────────

resource "aws_subnet" "pods" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.pod_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-pods-${var.availability_zones[count.index]}"
    Tier = "pods"
  })

  depends_on = [aws_vpc_ipv4_cidr_block_association.secondary]
}

resource "aws_route_table_association" "pods" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.pods[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# ─────────────────────────────────────────────
# DATA SUBNETS (RDS, ElastiCache)
# ─────────────────────────────────────────────

resource "aws_subnet" "data" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.data_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-data-${var.availability_zones[count.index]}"
    Tier = "data"
  })
}

resource "aws_route_table" "data" {
  vpc_id = aws_vpc.main.id

  # Data subnets have NO internet route — fully isolated
  # Traffic only flows to/from private subnets via local VPC routing

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-data-rt"
  })
}

resource "aws_route_table_association" "data" {
  count = length(var.availability_zones)

  subnet_id      = aws_subnet.data[count.index].id
  route_table_id = aws_route_table.data.id
}

resource "aws_db_subnet_group" "main" {
  name       = "${local.name_prefix}-db-subnet-group"
  subnet_ids = aws_subnet.data[*].id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-db-subnet-group"
  })
}

resource "aws_elasticache_subnet_group" "main" {
  name       = "${local.name_prefix}-cache-subnet-group"
  subnet_ids = aws_subnet.data[*].id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-cache-subnet-group"
  })
}

# ─────────────────────────────────────────────
# DEFAULT SECURITY GROUP — LOCK DOWN
# ─────────────────────────────────────────────

resource "aws_default_security_group" "default" {
  vpc_id = aws_vpc.main.id

  # No ingress rules — deny all
  # No egress rules — deny all
  # This ensures the default SG is never accidentally permissive

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-default-sg-DO-NOT-USE"
  })
}
```

```hcl
# modules/vpc/vpc-endpoints.tf

locals {
  interface_endpoints = var.enable_vpc_endpoints ? {
    ecr_api            = "com.amazonaws.${data.aws_region.current.name}.ecr.api"
    ecr_dkr            = "com.amazonaws.${data.aws_region.current.name}.ecr.dkr"
    cloudwatch_logs    = "com.amazonaws.${data.aws_region.current.name}.logs"
    sts                = "com.amazonaws.${data.aws_region.current.name}.sts"
    secretsmanager     = "com.amazonaws.${data.aws_region.current.name}.secretsmanager"
    kms                = "com.amazonaws.${data.aws_region.current.name}.kms"
  } : {}
}

data "aws_region" "current" {}

# ─────────────────────────────────────────────
# S3 GATEWAY ENDPOINT (free)
# ─────────────────────────────────────────────

resource "aws_vpc_endpoint" "s3" {
  count = var.enable_vpc_endpoints ? 1 : 0

  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${data.aws_region.current.name}.s3"

  vpc_endpoint_type = "Gateway"
  route_table_ids = concat(
    aws_route_table.private[*].id,
    [aws_route_table.data.id]
  )

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpce-s3"
  })
}

# ─────────────────────────────────────────────
# INTERFACE ENDPOINTS
# ─────────────────────────────────────────────

resource "aws_security_group" "vpc_endpoints" {
  count = var.enable_vpc_endpoints ? 1 : 0

  name_prefix = "${local.name_prefix}-vpce-"
  description = "Security group for VPC Interface Endpoints"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS from private subnets"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = concat(
      var.private_subnet_cidrs,
      var.pod_subnet_cidrs,
      var.data_subnet_cidrs
    )
  }

  # No egress needed — endpoints are AWS-managed

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpce-sg"
  })

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_vpc_endpoint" "interface" {
  for_each = local.interface_endpoints

  vpc_id              = aws_vpc.main.id
  service_name        = each.value
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true

  subnet_ids         = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.vpc_endpoints[0].id]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpce-${each.key}"
  })
}
```

```hcl
# modules/vpc/flow-logs.tf

resource "aws_flow_log" "cloudwatch" {
  vpc_id                   = aws_vpc.main.id
  traffic_type             = "ALL"
  log_destination_type     = "cloud-watch-logs"
  log_destination          = aws_cloudwatch_log_group.flow_logs.arn
  iam_role_arn             = aws_iam_role.flow_logs.arn
  max_aggregation_interval = 60

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-flow-log-cw"
  })
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/novamart/${var.environment}/vpc-flow-logs"
  retention_in_days = var.flow_log_retention_days
  # KMS encryption configured via security module's general key

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-flow-logs"
  })
}

resource "aws_iam_role" "flow_logs" {
  name_prefix = "${local.name_prefix}-flow-logs-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "aws:SourceAccount" = data.aws_caller_identity.current.account_id
          }
        }
      }
    ]
  })

  tags = local.common_tags
}

data "aws_caller_identity" "current" {}

resource "aws_iam_role_policy" "flow_logs" {
  name_prefix = "flow-logs-"
  role        = aws_iam_role.flow_logs.id

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
        Resource = "${aws_cloudwatch_log_group.flow_logs.arn}:*"
      }
    ]
  })
}

# S3 flow log for long-term retention (cheaper than CloudWatch)
resource "aws_flow_log" "s3" {
  count = var.flow_log_s3_bucket_arn != "" ? 1 : 0

  vpc_id                   = aws_vpc.main.id
  traffic_type             = "ALL"
  log_destination_type     = "s3"
  log_destination          = "${var.flow_log_s3_bucket_arn}/vpc-flow-logs/"
  max_aggregation_interval = 600 # 10-min aggregation for S3 (cost optimization)

  destination_options {
    file_format                = "parquet"
    hive_compatible_partitions = true
    per_hour_partition         = true
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-flow-log-s3"
  })
}
```

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "Primary VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs (EKS nodes)"
  value       = aws_subnet.private[*].id
}

output "pod_subnet_ids" {
  description = "List of pod subnet IDs (VPC CNI custom networking)"
  value       = aws_subnet.pods[*].id
}

output "data_subnet_ids" {
  description = "List of data subnet IDs (RDS, ElastiCache)"
  value       = aws_subnet.data[*].id
}

output "db_subnet_group_name" {
  description = "RDS subnet group name"
  value       = aws_db_subnet_group.main.name
}

output "elasticache_subnet_group_name" {
  description = "ElastiCache subnet group name"
  value       = aws_elasticache_subnet_group.main.name
}

output "nat_gateway_ips" {
  description = "NAT Gateway public IPs (for allowlisting)"
  value       = aws_eip.nat[*].public_ip
}

output "vpc_endpoint_sg_id" {
  description = "Security group ID for VPC endpoints"
  value       = try(aws_security_group.vpc_endpoints[0].id, "")
}

output "availability_zones" {
  description = "AZs used by this VPC"
  value       = var.availability_zones
}
```

---

### 4.3 EKS Module

```hcl
# modules/eks/variables.tf

variable "environment" {
  type = string
}

variable "cluster_name" {
  type = string
}

variable "cluster_version" {
  description = "Kubernetes version"
  type        = string
  default     = "1.29"
}

variable "vpc_id" {
  type = string
}

variable "private_subnet_ids" {
  description = "Subnets for EKS nodes"
  type        = list(string)
}

variable "pod_subnet_ids" {
  description = "Subnets for pod IPs (VPC CNI custom networking)"
  type        = list(string)
}

variable "api_public_access_cidrs" {
  description = "CIDR blocks allowed to access the public API endpoint"
  type        = list(string)
  default     = [] # Empty = no public access
}

variable "eks_secrets_kms_key_arn" {
  description = "KMS key ARN for encrypting Kubernetes secrets in etcd"
  type        = string
}

variable "system_node_instance_types" {
  description = "Instance types for the system node group"
  type        = list(string)
  default     = ["m5.large"]
}

variable "system_node_desired" {
  type    = number
  default = 2
}

variable "system_node_min" {
  type    = number
  default = 2
}

variable "system_node_max" {
  type    = number
  default = 4
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/eks/main.tf

locals {
  name_prefix = "novamart-${var.environment}"

  common_tags = merge(var.tags, {
    Module = "eks"
  })
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# ─────────────────────────────────────────────
# EKS CLUSTER IAM ROLE
# ─────────────────────────────────────────────

resource "aws_iam_role" "cluster" {
  name_prefix = "${local.name_prefix}-eks-cluster-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy_attachment" "cluster_policies" {
  for_each = toset([
    "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy",
    "arn:aws:iam::aws:policy/AmazonEKSVPCResourceController",
  ])

  policy_arn = each.value
  role       = aws_iam_role.cluster.name
}

# ─────────────────────────────────────────────
# EKS CLUSTER SECURITY GROUP
# ─────────────────────────────────────────────

resource "aws_security_group" "cluster" {
  name_prefix = "${local.name_prefix}-eks-cluster-"
  description = "EKS cluster control plane security group"
  vpc_id      = var.vpc_id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-eks-cluster-sg"
  })

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group_rule" "cluster_egress" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.cluster.id
  description       = "Allow all egress from control plane"
}

# ─────────────────────────────────────────────
# EKS CLUSTER
# ─────────────────────────────────────────────

resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  version  = var.cluster_version
  role_arn = aws_iam_role.cluster.arn

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = length(var.api_public_access_cidrs) > 0
    public_access_cidrs     = length(var.api_public_access_cidrs) > 0 ? var.api_public_access_cidrs : null
    security_group_ids      = [aws_security_group.cluster.id]
  }

  encryption_config {
    provider {
      key_arn = var.eks_secrets_kms_key_arn
    }
    resources = ["secrets"]
  }

  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  tags = merge(local.common_tags, {
    Name = var.cluster_name
  })

  depends_on = [
    aws_iam_role_policy_attachment.cluster_policies,
    aws_cloudwatch_log_group.cluster
  ]
}

resource "aws_cloudwatch_log_group" "cluster" {
  name              = "/aws/eks/${var.cluster_name}/cluster"
  retention_in_days = 90

  tags = local.common_tags
}

# ─────────────────────────────────────────────
# OIDC PROVIDER (for IRSA)
# ─────────────────────────────────────────────

data "tls_certificate" "cluster" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "cluster" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.cluster.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer

  tags = merge(local.common_tags, {
    Name = "${var.cluster_name}-oidc"
  })
}
```

```hcl
# modules/eks/node-groups.tf

# ─────────────────────────────────────────────
# NODE IAM ROLE (shared by all managed node groups)
# ─────────────────────────────────────────────

resource "aws_iam_role" "node" {
  name_prefix = "${local.name_prefix}-eks-node-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy_attachment" "node_policies" {
  for_each = toset([
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
    "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore",
  ])

  policy_arn = each.value
  role       = aws_iam_role.node.name
}

# ─────────────────────────────────────────────
# NODE SECURITY GROUP
# ─────────────────────────────────────────────

resource "aws_security_group" "node" {
  name_prefix = "${local.name_prefix}-eks-node-"
  description = "EKS worker node security group"
  vpc_id      = var.vpc_id

  tags = merge(local.common_tags, {
    Name                                          = "${local.name_prefix}-eks-node-sg"
    "kubernetes.io/cluster/${var.cluster_name}"    = "owned"
  })

  lifecycle {
    create_before_destroy = true
  }
}

# Node-to-node communication (all ports)
resource "aws_security_group_rule" "node_to_node" {
  type                     = "ingress"
  from_port                = 0
  to_port                  = 65535
  protocol                 = "-1"
  source_security_group_id = aws_security_group.node.id
  security_group_id        = aws_security_group.node.id
  description              = "Node to node"
}

# Control plane to nodes (kubelet, webhooks)
resource "aws_security_group_rule" "control_plane_to_node" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.cluster.id
  security_group_id        = aws_security_group.node.id
  description              = "Control plane to node (webhooks)"
}

resource "aws_security_group_rule" "control_plane_to_node_kubelet" {
  type                     = "ingress"
  from_port                = 10250
  to_port                  = 10250
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.cluster.id
  security_group_id        = aws_security_group.node.id
  description              = "Control plane to kubelet"
}

# Nodes to control plane
resource "aws_security_group_rule" "node_to_control_plane" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.node.id
  security_group_id        = aws_security_group.cluster.id
  description              = "Nodes to control plane"
}

# Node egress (all — needed for NAT GW, VPC endpoints, etc.)
resource "aws_security_group_rule" "node_egress" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.node.id
  description       = "Node egress"
}

# CoreDNS port (nodes to nodes)
resource "aws_security_group_rule" "coredns_tcp" {
  type                     = "ingress"
  from_port                = 53
  to_port                  = 53
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.node.id
  security_group_id        = aws_security_group.node.id
  description              = "CoreDNS TCP"
}

resource "aws_security_group_rule" "coredns_udp" {
  type                     = "ingress"
  from_port                = 53
  to_port                  = 53
  protocol                 = "udp"
  source_security_group_id = aws_security_group.node.id
  security_group_id        = aws_security_group.node.id
  description              = "CoreDNS UDP"
}

# ─────────────────────────────────────────────
# LAUNCH TEMPLATE (IMDS hardening)
# ─────────────────────────────────────────────

resource "aws_launch_template" "system_nodes" {
  name_prefix = "${local.name_prefix}-system-"

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2 enforced
    http_put_response_hop_limit = 1           # Blocks pod access to IMDS
    instance_metadata_tags      = "disabled"
  }

  monitoring {
    enabled = true
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 100
      volume_type           = "gp3"
      iops                  = 3000
      throughput            = 125
      encrypted             = true
      delete_on_termination = true
    }
  }

  tag_specifications {
    resource_type = "instance"
    tags = merge(local.common_tags, {
      Name     = "${local.name_prefix}-system-node"
      NodeType = "system"
    })
  }

  tag_specifications {
    resource_type = "volume"
    tags = merge(local.common_tags, {
      Name = "${local.name_prefix}-system-node-vol"
    })
  }

  tags = local.common_tags
}

# ─────────────────────────────────────────────
# SYSTEM NODE GROUP (runs Karpenter, CoreDNS, kube-proxy)
# ─────────────────────────────────────────────

resource "aws_eks_node_group" "system" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${local.name_prefix}-system"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids

  instance_types = var.system_node_instance_types

  scaling_config {
    desired_size = var.system_node_desired
    min_size     = var.system_node_min
    max_size     = var.system_node_max
  }

  update_config {
    max_unavailable = 1
  }

  launch_template {
    id      = aws_launch_template.system_nodes.id
    version = aws_launch_template.system_nodes.latest_version
  }

  labels = {
    "novamart.com/node-type" = "system"
  }

  taint {
    key    = "CriticalAddonsOnly"
    value  = "true"
    effect = "NO_SCHEDULE"
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-system-nodegroup"
  })

 ```hcl
  depends_on = [
    aws_iam_role_policy_attachment.node_policies,
  ]

  lifecycle {
    ignore_changes = [scaling_config[0].desired_size]
  }
}
```

```hcl
# modules/eks/addons.tf

# ─────────────────────────────────────────────
# EKS MANAGED ADD-ONS
# ─────────────────────────────────────────────

resource "aws_eks_addon" "vpc_cni" {
  cluster_name             = aws_eks_cluster.main.name
  addon_name               = "vpc-cni"
  addon_version            = data.aws_eks_addon_version.vpc_cni.version
  resolve_conflicts_on_update = "OVERWRITE"
  service_account_role_arn = aws_iam_role.vpc_cni.arn

  configuration_values = jsonencode({
    env = {
      # Enable custom networking — pods use secondary CIDR subnets
      AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG = "true"
      ENI_CONFIG_LABEL_DEF               = "topology.kubernetes.io/zone"
      # Enable prefix delegation for higher pod density per node
      ENABLE_PREFIX_DELEGATION           = "true"
      WARM_PREFIX_TARGET                 = "1"
      # Enable NetworkPolicy enforcement
      ENABLE_NETWORK_POLICY              = "true"
    }
  })

  tags = local.common_tags

  depends_on = [aws_eks_node_group.system]
}

data "aws_eks_addon_version" "vpc_cni" {
  addon_name         = "vpc-cni"
  kubernetes_version = var.cluster_version
  most_recent        = true
}

# VPC CNI IRSA Role — needs ENI and IP management permissions
resource "aws_iam_role" "vpc_cni" {
  name_prefix = "${local.name_prefix}-vpc-cni-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.cluster.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:kube-system:aws-node"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy_attachment" "vpc_cni" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.vpc_cni.name
}

# CoreDNS
resource "aws_eks_addon" "coredns" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "coredns"
  addon_version               = data.aws_eks_addon_version.coredns.version
  resolve_conflicts_on_update = "OVERWRITE"

  configuration_values = jsonencode({
    tolerations = [
      {
        key      = "CriticalAddonsOnly"
        operator = "Exists"
        effect   = "NoSchedule"
      }
    ]
    nodeSelector = {
      "novamart.com/node-type" = "system"
    }
    replicaCount = 3  # One per AZ for HA
  })

  tags = local.common_tags

  depends_on = [aws_eks_node_group.system]
}

data "aws_eks_addon_version" "coredns" {
  addon_name         = "coredns"
  kubernetes_version = var.cluster_version
  most_recent        = true
}

# kube-proxy
resource "aws_eks_addon" "kube_proxy" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "kube-proxy"
  addon_version               = data.aws_eks_addon_version.kube_proxy.version
  resolve_conflicts_on_update = "OVERWRITE"

  tags = local.common_tags

  depends_on = [aws_eks_node_group.system]
}

data "aws_eks_addon_version" "kube_proxy" {
  addon_name         = "kube-proxy"
  kubernetes_version = var.cluster_version
  most_recent        = true
}

# EBS CSI Driver (for PersistentVolumes)
resource "aws_eks_addon" "ebs_csi" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "aws-ebs-csi-driver"
  addon_version               = data.aws_eks_addon_version.ebs_csi.version
  resolve_conflicts_on_update = "OVERWRITE"
  service_account_role_arn    = aws_iam_role.ebs_csi.arn

  tags = local.common_tags

  depends_on = [aws_eks_node_group.system]
}

data "aws_eks_addon_version" "ebs_csi" {
  addon_name         = "aws-ebs-csi-driver"
  kubernetes_version = var.cluster_version
  most_recent        = true
}

resource "aws_iam_role" "ebs_csi" {
  name_prefix = "${local.name_prefix}-ebs-csi-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.cluster.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:kube-system:ebs-csi-controller-sa"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy_attachment" "ebs_csi" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  role       = aws_iam_role.ebs_csi.name
}
```

```hcl
# modules/eks/karpenter.tf

# ─────────────────────────────────────────────
# KARPENTER IAM ROLE (IRSA)
# ─────────────────────────────────────────────

resource "aws_iam_role" "karpenter_controller" {
  name_prefix = "${local.name_prefix}-karpenter-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.cluster.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:kube-system:karpenter"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "karpenter_controller" {
  name_prefix = "karpenter-"
  role        = aws_iam_role.karpenter_controller.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowEC2Operations"
        Effect = "Allow"
        Action = [
          "ec2:CreateLaunchTemplate",
          "ec2:CreateFleet",
          "ec2:RunInstances",
          "ec2:CreateTags",
          "ec2:TerminateInstances",
          "ec2:DeleteLaunchTemplate",
          "ec2:DescribeLaunchTemplates",
          "ec2:DescribeInstances",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeSubnets",
          "ec2:DescribeInstanceTypes",
          "ec2:DescribeInstanceTypeOfferings",
          "ec2:DescribeAvailabilityZones",
          "ec2:DescribeImages",
          "ec2:DescribeSpotPriceHistory"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "aws:RequestedRegion" = data.aws_region.current.name
          }
        }
      },
      {
        Sid    = "AllowPassRole"
        Effect = "Allow"
        Action = "iam:PassRole"
        Resource = aws_iam_role.node.arn
      },
      {
        Sid    = "AllowSSMGetParameter"
        Effect = "Allow"
        Action = "ssm:GetParameter"
        Resource = "arn:aws:ssm:${data.aws_region.current.name}::parameter/aws/service/eks/optimized-ami/*"
      },
      {
        Sid    = "AllowPricing"
        Effect = "Allow"
        Action = [
          "pricing:GetProducts"
        ]
        Resource = "*"
      },
      {
        Sid    = "AllowSQS"
        Effect = "Allow"
        Action = [
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "sqs:GetQueueUrl",
          "sqs:ReceiveMessage"
        ]
        Resource = aws_sqs_queue.karpenter_interruption.arn
      }
    ]
  })
}

# ─────────────────────────────────────────────
# KARPENTER INSTANCE PROFILE (for nodes it provisions)
# ─────────────────────────────────────────────

resource "aws_iam_instance_profile" "karpenter_node" {
  name_prefix = "${local.name_prefix}-karpenter-node-"
  role        = aws_iam_role.node.name

  tags = local.common_tags
}

# ─────────────────────────────────────────────
# SPOT INTERRUPTION QUEUE
# ─────────────────────────────────────────────

resource "aws_sqs_queue" "karpenter_interruption" {
  name                      = "${var.cluster_name}-karpenter-interruption"
  message_retention_seconds = 300
  sqs_managed_sse_enabled   = true

  tags = merge(local.common_tags, {
    Name = "${var.cluster_name}-karpenter-interruption"
  })
}

resource "aws_sqs_queue_policy" "karpenter_interruption" {
  queue_url = aws_sqs_queue.karpenter_interruption.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowEventBridge"
        Effect = "Allow"
        Principal = {
          Service = ["events.amazonaws.com", "sqs.amazonaws.com"]
        }
        Action   = "sqs:SendMessage"
        Resource = aws_sqs_queue.karpenter_interruption.arn
      }
    ]
  })
}

# EventBridge rules for Spot interruption, instance state changes,
# and scheduled change health events
resource "aws_cloudwatch_event_rule" "karpenter_spot_interruption" {
  name_prefix = "karpenter-spot-"
  description = "Spot instance interruption warnings for Karpenter"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Spot Instance Interruption Warning"]
  })

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "karpenter_spot_interruption" {
  rule = aws_cloudwatch_event_rule.karpenter_spot_interruption.name
  arn  = aws_sqs_queue.karpenter_interruption.arn
}

resource "aws_cloudwatch_event_rule" "karpenter_instance_rebalance" {
  name_prefix = "karpenter-rebalance-"
  description = "Instance rebalance recommendations for Karpenter"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance Rebalance Recommendation"]
  })

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "karpenter_instance_rebalance" {
  rule = aws_cloudwatch_event_rule.karpenter_instance_rebalance.name
  arn  = aws_sqs_queue.karpenter_interruption.arn
}

resource "aws_cloudwatch_event_rule" "karpenter_instance_state_change" {
  name_prefix = "karpenter-state-"
  description = "Instance state change notifications for Karpenter"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance State-change Notification"]
  })

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "karpenter_instance_state_change" {
  rule = aws_cloudwatch_event_rule.karpenter_instance_state_change.name
  arn  = aws_sqs_queue.karpenter_interruption.arn
}

resource "aws_cloudwatch_event_rule" "karpenter_health_event" {
  name_prefix = "karpenter-health-"
  description = "AWS Health events for Karpenter"

  event_pattern = jsonencode({
    source      = ["aws.health"]
    detail-type = ["AWS Health Event"]
  })

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "karpenter_health_event" {
  rule = aws_cloudwatch_event_rule.karpenter_health_event.name
  arn  = aws_sqs_queue.karpenter_interruption.arn
}
```

```hcl
# modules/eks/irsa.tf

# ─────────────────────────────────────────────
# IRSA HELPER — reusable for application teams
# This creates IRSA roles that application Terraform can reference
# ─────────────────────────────────────────────

# AWS Load Balancer Controller IRSA
resource "aws_iam_role" "aws_lb_controller" {
  name_prefix = "${local.name_prefix}-aws-lb-ctrl-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.cluster.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:kube-system:aws-load-balancer-controller"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "aws_lb_controller" {
  name_prefix = "aws-lb-ctrl-"
  role        = aws_iam_role.aws_lb_controller.id

  # Using the official AWS LB Controller IAM policy
  # https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
  policy = file("${path.module}/policies/aws-lb-controller-policy.json")
}

# External Secrets Operator IRSA
resource "aws_iam_role" "external_secrets" {
  name_prefix = "${local.name_prefix}-ext-secrets-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.cluster.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:external-secrets:external-secrets"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "external_secrets" {
  name_prefix = "ext-secrets-"
  role        = aws_iam_role.external_secrets.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret",
          "secretsmanager:ListSecretVersionIds"
        ]
        Resource = "arn:aws:secretsmanager:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:secret:novamart/${var.environment}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "ssm:GetParameter",
          "ssm:GetParameters",
          "ssm:GetParametersByPath"
        ]
        Resource = "arn:aws:ssm:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:parameter/novamart/${var.environment}/*"
      }
    ]
  })
}
```

```hcl
# modules/eks/outputs.tf

output "cluster_name" {
  description = "EKS cluster name"
  value       = aws_eks_cluster.main.name
}

output "cluster_endpoint" {
  description = "EKS cluster API endpoint"
  value       = aws_eks_cluster.main.endpoint
}

output "cluster_certificate_authority" {
  description = "EKS cluster CA certificate (base64)"
  value       = aws_eks_cluster.main.certificate_authority[0].data
}

output "cluster_oidc_issuer" {
  description = "OIDC issuer URL"
  value       = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

output "oidc_provider_arn" {
  description = "OIDC provider ARN for IRSA"
  value       = aws_iam_openid_connect_provider.cluster.arn
}

output "node_role_arn" {
  description = "IAM role ARN for EKS worker nodes"
  value       = aws_iam_role.node.arn
}

output "node_security_group_id" {
  description = "Security group ID for EKS nodes"
  value       = aws_security_group.node.id
}

output "cluster_security_group_id" {
  description = "Security group ID for EKS control plane"
  value       = aws_security_group.cluster.id
}

output "karpenter_role_arn" {
  description = "IAM role ARN for Karpenter controller"
  value       = aws_iam_role.karpenter_controller.arn
}

output "karpenter_instance_profile_name" {
  description = "Instance profile name for Karpenter-provisioned nodes"
  value       = aws_iam_instance_profile.karpenter_node.name
}

output "karpenter_interruption_queue_name" {
  description = "SQS queue name for Karpenter interruption handling"
  value       = aws_sqs_queue.karpenter_interruption.name
}

output "aws_lb_controller_role_arn" {
  description = "IRSA role ARN for AWS Load Balancer Controller"
  value       = aws_iam_role.aws_lb_controller.arn
}

output "external_secrets_role_arn" {
  description = "IRSA role ARN for External Secrets Operator"
  value       = aws_iam_role.external_secrets.arn
}

output "cluster_version" {
  description = "EKS cluster Kubernetes version"
  value       = aws_eks_cluster.main.version
}
```

---

### 4.4 Data Tier

```hcl
# modules/rds/variables.tf

variable "environment" {
  type = string
}

variable "identifier" {
  description = "Unique identifier for this RDS instance (e.g., payments, orders, users)"
  type        = string
}

variable "engine_version" {
  description = "PostgreSQL engine version"
  type        = string
  default     = "16.2"
}

variable "instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.r6g.large"
}

variable "allocated_storage" {
  description = "Storage in GB"
  type        = number
  default     = 100
}

variable "max_allocated_storage" {
  description = "Maximum storage for autoscaling (GB)"
  type        = number
  default     = 500
}

variable "db_name" {
  description = "Initial database name"
  type        = string
}

variable "master_username" {
  description = "Master username"
  type        = string
  default     = "novamart_admin"
}

variable "db_subnet_group_name" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "allowed_security_group_ids" {
  description = "Security group IDs allowed to connect (EKS node SG)"
  type        = list(string)
}

variable "kms_key_arn" {
  description = "KMS key ARN for encryption at rest"
  type        = string
}

variable "backup_retention_period" {
  description = "Number of days to retain automated backups"
  type        = number
  default     = 30
}

variable "deletion_protection" {
  description = "Enable deletion protection"
  type        = bool
  default     = true
}

variable "performance_insights_enabled" {
  description = "Enable Performance Insights"
  type        = bool
  default     = true
}

variable "monitoring_interval" {
  description = "Enhanced monitoring interval in seconds (0 to disable)"
  type        = number
  default     = 60
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/rds/main.tf

locals {
  name_prefix = "novamart-${var.environment}"

  common_tags = merge(var.tags, {
    Module  = "rds"
    Service = var.identifier
  })
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# ─────────────────────────────────────────────
# SECURITY GROUP
# ─────────────────────────────────────────────

resource "aws_security_group" "rds" {
  name_prefix = "${local.name_prefix}-rds-${var.identifier}-"
  description = "Security group for RDS ${var.identifier}"
  vpc_id      = var.vpc_id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-rds-${var.identifier}-sg"
  })

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group_rule" "rds_ingress" {
  count = length(var.allowed_security_group_ids)

  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  source_security_group_id = var.allowed_security_group_ids[count.index]
  security_group_id        = aws_security_group.rds.id
  description              = "PostgreSQL from EKS nodes"
}

# No egress rules — RDS doesn't initiate outbound connections

# ─────────────────────────────────────────────
# PARAMETER GROUP
# ─────────────────────────────────────────────

resource "aws_db_parameter_group" "main" {
  name_prefix = "${local.name_prefix}-${var.identifier}-"
  family      = "postgres16"
  description = "Custom parameter group for ${var.identifier}"

  parameter {
    name  = "log_connections"
    value = "1"
  }

  parameter {
    name  = "log_disconnections"
    value = "1"
  }

  parameter {
    name  = "log_statement"
    value = "ddl"
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000" # Log queries taking >1 second
  }

  parameter {
    name         = "rds.force_ssl"
    value        = "1"
    apply_method = "pending-reboot"
  }

  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements"
  }

  tags = local.common_tags

  lifecycle {
    create_before_destroy = true
  }
}

# ─────────────────────────────────────────────
# ENHANCED MONITORING ROLE
# ─────────────────────────────────────────────

resource "aws_iam_role" "rds_monitoring" {
  count       = var.monitoring_interval > 0 ? 1 : 0
  name_prefix = "${local.name_prefix}-rds-monitoring-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "monitoring.rds.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy_attachment" "rds_monitoring" {
  count      = var.monitoring_interval > 0 ? 1 : 0
  role       = aws_iam_role.rds_monitoring[0].name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}

# ─────────────────────────────────────────────
# RDS INSTANCE
# ─────────────────────────────────────────────

resource "aws_db_instance" "main" {
  identifier = "${local.name_prefix}-${var.identifier}"

  # Engine
  engine               = "postgres"
  engine_version       = var.engine_version
  instance_class       = var.instance_class
  parameter_group_name = aws_db_parameter_group.main.name

  # Storage
  allocated_storage     = var.allocated_storage
  max_allocated_storage = var.max_allocated_storage
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = var.kms_key_arn

  # Database
  db_name  = var.db_name
  username = var.master_username
  # Password managed via Secrets Manager — set initial, rotate immediately
  manage_master_user_password   = true
  master_user_secret_kms_key_id = var.kms_key_arn

  # Network
  db_subnet_group_name   = var.db_subnet_group_name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false
  port                   = 5432

  # High Availability
  multi_az = var.environment == "production" ? true : false

  # Backup
  backup_retention_period   = var.backup_retention_period
  backup_window             = "03:00-04:00" # UTC — low traffic window
  maintenance_window        = "sun:04:00-sun:05:00"
  copy_tags_to_snapshot     = true
  delete_automated_backups  = false
  final_snapshot_identifier = "${local.name_prefix}-${var.identifier}-final-${formatdate("YYYY-MM-DD", timestamp())}"
  skip_final_snapshot       = false

  # Monitoring
  performance_insights_enabled          = var.performance_insights_enabled
  performance_insights_retention_period = var.performance_insights_enabled ? 731 : null # 2 years (free tier)
  performance_insights_kms_key_id       = var.performance_insights_enabled ? var.kms_key_arn : null
  monitoring_interval                   = var.monitoring_interval
  monitoring_role_arn                   = var.monitoring_interval > 0 ? aws_iam_role.rds_monitoring[0].arn : null
  enabled_cloudwatch_logs_exports       = ["postgresql", "upgrade"]

  # Protection
  deletion_protection = var.deletion_protection

  # Upgrades
  auto_minor_version_upgrade  = true
  allow_major_version_upgrade = false
  apply_immediately           = false

  tags = merge(local.common_tags, {
    Name     = "${local.name_prefix}-${var.identifier}"
    Backup   = "required"
    DataTier = "primary"
  })

  lifecycle {
    ignore_changes = [
      final_snapshot_identifier,
    ]
  }
}

# ─────────────────────────────────────────────
# CLOUDWATCH ALARMS
# ─────────────────────────────────────────────

resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${local.name_prefix}-rds-${var.identifier}-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "RDS ${var.identifier} CPU > 80% for 15 minutes"

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "free_storage_low" {
  alarm_name          = "${local.name_prefix}-rds-${var.identifier}-storage-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FreeStorageSpace"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 10737418240 # 10 GB in bytes
  alarm_description   = "RDS ${var.identifier} free storage < 10GB"

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "connection_count_high" {
  alarm_name          = "${local.name_prefix}-rds-${var.identifier}-connections-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 200 # db.r6g.large max_connections ~1700, alert at ~12%
  alarm_description   = "RDS ${var.identifier} connections > 200"

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "replica_lag" {
  count = var.environment == "production" ? 1 : 0

  alarm_name          = "${local.name_prefix}-rds-${var.identifier}-replica-lag"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "ReplicaLag"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 30 # 30 seconds
  alarm_description   = "RDS ${var.identifier} replica lag > 30s"

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }

  tags = local.common_tags
}
```

```hcl
# modules/rds/outputs.tf

output "endpoint" {
  description = "RDS instance endpoint (host:port)"
  value       = aws_db_instance.main.endpoint
}

output "address" {
  description = "RDS instance hostname"
  value       = aws_db_instance.main.address
}

output "port" {
  description = "RDS instance port"
  value       = aws_db_instance.main.port
}

output "db_name" {
  description = "Database name"
  value       = aws_db_instance.main.db_name
}

output "master_secret_arn" {
  description = "ARN of the Secrets Manager secret containing master credentials"
  value       = aws_db_instance.main.master_user_secret[0].secret_arn
}

output "security_group_id" {
  description = "Security group ID for the RDS instance"
  value       = aws_security_group.rds.id
}

output "identifier" {
  description = "RDS instance identifier"
  value       = aws_db_instance.main.identifier
}
```

```hcl
# modules/elasticache/main.tf

locals {
  name_prefix = "novamart-${var.environment}"
  common_tags = merge(var.tags, {
    Module  = "elasticache"
    Service = "redis"
  })
}

variable "environment" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "subnet_group_name" {
  type = string
}

variable "allowed_security_group_ids" {
  type = list(string)
}

variable "node_type" {
  type    = string
  default = "cache.r6g.large"
}

variable "num_cache_clusters" {
  description = "Number of cache clusters (nodes) in the replication group"
  type        = number
  default     = 3 # Primary + 2 replicas, one per AZ
}

variable "engine_version" {
  type    = string
  default = "7.1"
}

variable "kms_key_arn" {
  type = string
}

variable "tags" {
  type    = map(string)
  default = {}
}

# ─────────────────────────────────────────────
# SECURITY GROUP
# ─────────────────────────────────────────────

resource "aws_security_group" "redis" {
  name_prefix = "${local.name_prefix}-redis-"
  description = "Security group for ElastiCache Redis"
  vpc_id      = var.vpc_id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-redis-sg"
  })

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group_rule" "redis_ingress" {
  count = length(var.allowed_security_group_ids)

  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  source_security_group_id = var.allowed_security_group_ids[count.index]
  security_group_id        = aws_security_group.redis.id
  description              = "Redis from EKS nodes"
}

# ─────────────────────────────────────────────
# PARAMETER GROUP
# ─────────────────────────────────────────────

resource "aws_elasticache_parameter_group" "main" {
  name        = "${local.name_prefix}-redis-params"
  family      = "redis7"
  description = "NovaMart Redis parameter group"

  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  parameter {
    name  = "notify-keyspace-events"
    value = "Ex" # Expired events for session management
  }

  tags = local.common_tags
}

# ─────────────────────────────────────────────
# REPLICATION GROUP
# ─────────────────────────────────────────────

resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "${local.name_prefix}-redis"
  description          = "NovaMart ${var.environment} Redis cluster"

  node_type            = var.node_type
  num_cache_clusters   = var.num_cache_clusters
  engine_version       = var.engine_version
  port                 = 6379
  parameter_group_name = aws_elasticache_parameter_group.main.name

  # Network
  subnet_group_name  = var.subnet_group_name
  security_group_ids = [aws_security_group.redis.id]

  # Encryption
  at_rest_encryption_enabled = true
  kms_key_id                 = var.kms_key_arn
  transit_encryption_enabled = true
  # auth_token managed via Secrets Manager externally

  # HA
  automatic_failover_enabled = var.num_cache_clusters > 1
  multi_az_enabled           = var.num_cache_clusters > 1

  # Maintenance
  maintenance_window       = "sun:05:00-sun:06:00"
  snapshot_retention_limit = 7
  snapshot_window          = "02:00-03:00"
  auto_minor_version_upgrade = true

  # Notifications
  # notification_topic_arn = var.sns_topic_arn  # Configure when SNS module exists

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-redis"
  })

  lifecycle {
    ignore_changes = [num_cache_clusters]
  }
}

# ─────────────────────────────────────────────
# CLOUDWATCH ALARMS
# ─────────────────────────────────────────────

resource "aws_cloudwatch_metric_alarm" "redis_cpu" {
  alarm_name          = "${local.name_prefix}-redis-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "EngineCPUUtilization"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Average"
  threshold           = 75
  alarm_description   = "Redis CPU > 75% for 15 minutes"

  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.main.id
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "redis_memory" {
  alarm_name          = "${local.name_prefix}-redis-memory-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseMemoryUsagePercentage"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "Redis memory usage > 80%"

  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.main.id
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "redis_evictions" {
  alarm_name          = "${local.name_prefix}-redis-evictions"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "Evictions"
  namespace           = "AWS/ElastiCache"
  period              = 300
  statistic           = "Sum"
  threshold           = 1000
  alarm_description   = "Redis evictions > 1000 in 5 minutes — memory pressure"

  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.main.id
  }

  tags = local.common_tags
}

# ─────────────────────────────────────────────
# OUTPUTS
# ─────────────────────────────────────────────

output "primary_endpoint" {
  description = "Redis primary endpoint address"
  value       = aws_elasticache_replication_group.main.primary_endpoint_address
}

output "reader_endpoint" {
  description = "Redis reader endpoint address"
  value       = aws_elasticache_replication_group.main.reader_endpoint_address
}

output "port" {
  value = 6379
}

output "security_group_id" {
  value = aws_security_group.redis.id
}
```

---

### 4.5 Security Baseline

```hcl
# modules/security-baseline/kms.tf

locals {
  name_prefix = "novamart-${var.environment}"
  common_tags = merge(var.tags, {
    Module = "security-baseline"
  })
}

variable "environment" {
  type = string
}

variable "tags" {
  type    = map(string)
  default = {}
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# ─────────────────────────────────────────────
# KMS KEY: EKS Secrets Encryption
# ─────────────────────────────────────────────

resource "aws_kms_key" "eks_secrets" {
  description             = "Encrypts Kubernetes secrets in etcd for NovaMart ${var.environment}"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  multi_region            = false

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowRootAccountFullAccess"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "AllowEKSService"
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey",
          "kms:CreateGrant"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "kms:CallerAccount" = data.aws_caller_identity.current.account_id
          }
        }
      }
    ]
  })

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-eks-secrets-key"
    Purpose = "eks-secrets-encryption"
  })
}

resource "aws_kms_alias" "eks_secrets" {
  name          = "alias/${local.name_prefix}-eks-secrets"
  target_key_id = aws_kms_key.eks_secrets.key_id
}

# ─────────────────────────────────────────────
# KMS KEY: RDS Encryption
# ─────────────────────────────────────────────

resource "aws_kms_key" "rds" {
  description             = "Encrypts RDS instances for NovaMart ${var.environment}"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowRootAccountFullAccess"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "AllowRDSService"
        Effect = "Allow"
        Principal = {
          Service = "rds.amazonaws.com"
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey",
          "kms:CreateGrant"
        ]
        Resource = "*"
      }
    ]
  })

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-rds-key"
    Purpose = "rds-encryption"
  })
}

resource "aws_kms_alias" "rds" {
  name          = "alias/${local.name_prefix}-rds"
  target_key_id = aws_kms_key.rds.key_id
}

# ─────────────────────────────────────────────
# KMS KEY: S3 Encryption
# ─────────────────────────────────────────────

resource "aws_kms_key" "s3" {
  description             = "Encrypts S3 buckets for NovaMart ${var.environment}"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowRootAccountFullAccess"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "AllowS3Service"
        Effect = "Allow"
        Principal = {
          Service = "s3.amazonaws.com"
        }
        Action = [
          "kms:GenerateDataKey",
          "kms:Decrypt"
        ]
        Resource = "*"
      },
      {
        Sid    = "AllowCloudTrailEncrypt"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "kms:GenerateDataKey*"
        Resource = "*"
        Condition = {
          StringEquals = {
            "aws:SourceArn" = "arn:aws:cloudtrail:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:trail/${local.name_prefix}-trail"
          }
        }
      },
      {
        Sid    = "AllowFlowLogsEncrypt"
        Effect = "Allow"
        Principal = {
          Service = "delivery.logs.amazonaws.com"
        }
        Action   = "kms:GenerateDataKey*"
        Resource = "*"
      }
    ]
  })

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-s3-key"
    Purpose = "s3-encryption"
  })
}

resource "aws_kms_alias" "s3" {
  name          = "alias/${local.name_prefix}-s3"
  target_key_id = aws_kms_key.s3.key_id
}

# ─────────────────────────────────────────────
# KMS KEY: General Purpose
# ─────────────────────────────────────────────

resource "aws_kms_key" "general" {
  description             = "General-purpose encryption key for NovaMart ${var.environment}"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-general-key"
    Purpose = "general-encryption"
  })
}

resource "aws_kms_alias" "general" {
  name          = "alias/${local.name_prefix}-general"
  target_key_id = aws_kms_key.general.key_id
}
```

```hcl
# modules/security-baseline/s3.tf

# ─────────────────────────────────────────────
# CLOUDTRAIL BUCKET
# ─────────────────────────────────────────────

resource "aws_s3_bucket" "cloudtrail" {
  bucket = "${local.name_prefix}-cloudtrail-${data.aws_caller_identity.current.account_id}"

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-cloudtrail"
    Purpose = "audit-logs"
  })

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  rule {
    id     = "archive-and-expire"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 365
      storage_class = "GLACIER"
    }

    expiration {
      days = 2555 # 7 years — regulatory compliance
    }
  }
}

resource "aws_s3_bucket_object_lock_configuration" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  rule {
    default_retention {
      mode = "COMPLIANCE"
      days = 365 # 1 year — cannot be deleted or overwritten
    }
  }
}

resource "aws_s3_bucket_policy" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudTrailACLCheck"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.cloudtrail.arn
        Condition = {
          StringEquals = {
            "aws:SourceArn" = "arn:aws:cloudtrail:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:trail/${local.name_prefix}-trail"
          }
        }
      },
      {
        Sid    = "AllowCloudTrailWrite"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.cloudtrail.arn}/AWSLogs/${data.aws_caller_identity.current.account_id}/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" = "bucket-owner-full-control"
            "aws:SourceArn" = "arn:aws:cloudtrail:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:trail/${local.name_prefix}-trail"
          }
        }
      },
      {
        Sid    = "DenyUnencryptedPut"
        Effect = "Deny"
        Principal = "*"
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.cloudtrail.arn}/*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      },
      {
        Sid    = "DenyHTTP"
        Effect = "Deny"
        Principal = "*"
        Action   = "s3:*"
        Resource = [
          aws_s3_bucket.cloudtrail.arn,
          "${aws_s3_bucket.cloudtrail.arn}/*"
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

# ─────────────────────────────────────────────
# VPC FLOW LOGS BUCKET
# ─────────────────────────────────────────────

resource "aws_s3_bucket" "flow_logs" {
  bucket = "${local.name_prefix}-flow-logs-${data.aws_caller_identity.current.account_id}"

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-flow-logs"
    Purpose = "network-forensics"
  })

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "flow_logs" {
  bucket = aws_s3_bucket.flow_logs.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "flow_logs" {
  bucket = aws_s3_bucket.flow_logs.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "flow_logs" {
  bucket = aws_s3_bucket.flow_logs.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "flow_logs" {
  bucket = aws_s3_bucket.flow_logs.id

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
      days = 365
    }
  }
}

resource "aws_s3_bucket_policy" "flow_logs" {
  bucket = aws_s3_bucket.flow_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowFlowLogDelivery"
        Effect = "Allow"
        Principal = {
          Service = "delivery.logs.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.flow_logs.arn}/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-acl"    = "bucket-owner-full-control"
            "aws:SourceAccount" = data.aws_caller_identity.current.account_id
          }
        }
      },
      {
        Sid    = "AllowFlowLogACLCheck"
        Effect = "Allow"
        Principal = {
          Service = "delivery.logs.amazonaws.com"
        }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.flow_logs.arn
      },
      {
        Sid    = "DenyHTTP"
        Effect = "Deny"
        Principal = "*"
        Action   = "s3:*"
        Resource = [
          aws_s3_bucket.flow_logs.arn,
          "${aws_s3_bucket.flow_logs.arn}/*"
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
```

```hcl
# modules/security-baseline/cloudtrail.tf

# ─────────────────────────────────────────────
# CLOUDTRAIL
# ─────────────────────────────────────────────

resource "aws_cloudtrail" "main" {
  name                       = "${local.name_prefix}-trail"
  s3_bucket_name             = aws_s3_bucket.cloudtrail.id
  is_multi_region_trail      = true
  include_global_service_events = true
  enable_logging             = true
  enable_log_file_validation = true
  kms_key_id                 = aws_kms_key.s3.arn

  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_cloudwatch.arn

  # Data events for sensitive S3 buckets
  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = [
        "${aws_s3_bucket.cloudtrail.arn}/",
      ]
    }
  }

  # Insights — detect unusual API activity
  insight_selector {
    insight_type = "ApiCallRateInsight"
  }

  insight_selector {
    insight_type = "ApiErrorRateInsight"
  }

  tags = merge(local.common_tags, {
    Name    = "${local.name_prefix}-trail"
    Purpose = "audit"
  })

  depends_on = [aws_s3_bucket_policy.cloudtrail]
}

resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "/novamart/${var.environment}/cloudtrail"
  retention_in_days = 90

  tags = local.common_tags
}

resource "aws_iam_role" "cloudtrail_cloudwatch" {
  name_prefix = "${local.name_prefix}-cloudtrail-cw-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "cloudtrail_cloudwatch" {
  name_prefix = "cloudtrail-cw-"
  role        = aws_iam_role.cloudtrail_cloudwatch.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
      }
    ]
  })
}

# ─────────────────────────────────────────────
# CLOUDWATCH ALARM: CloudTrail Delivery Failure
# ─────────────────────────────────────────────

resource "aws_cloudwatch_metric_alarm" "cloudtrail_delivery" {
  alarm_name          = "${local.name_prefix}-cloudtrail-delivery-failure"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "CallCount"
  namespace           = "AWS/CloudTrail"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "CloudTrail delivery failures detected — audit logging may be compromised"
  treat_missing_data  = "breaching" # Missing data = trail may be stopped

  tags = local.common_tags
}

# ─────────────────────────────────────────────
# ACCOUNT-LEVEL SECURITY SETTINGS
# ─────────────────────────────────────────────

resource "aws_s3_account_public_access_block" "account" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_ebs_encryption_by_default" "enabled" {
  enabled = true
}
```

```hcl
# modules/security-baseline/outputs.tf

output "eks_secrets_kms_key_arn" {
  description = "KMS key ARN for EKS secrets encryption"
  value       = aws_kms_key.eks_secrets.arn
}

output "rds_kms_key_arn" {
  description = "KMS key ARN for RDS encryption"
  value       = aws_kms_key.rds.arn
}

output "s3_kms_key_arn" {
  description = "KMS key ARN for S3 encryption"
  value       = aws_kms_key.s3.arn
}

output "general_kms_key_arn" {
  description = "KMS key ARN for general-

