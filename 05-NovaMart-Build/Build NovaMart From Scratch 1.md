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
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    AWS ACCOUNT: 888888888888                                │
│                                    REGION: us-east-1                                        │
│                                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                           VPC: 10.0.0.0/16                                            │  │
│  │                    Secondary CIDR: 100.64.0.0/16 (Pod IPs)                            │  │
│  │                                                                                       │  │
│  │  ┌──────────────────────┬───────────────────────┬──────────────────────┐              │  │
│  │  │      AZ: us-east-1a  │    AZ: us-east-1b     │    AZ: us-east-1c    │              │  │
│  │  │                      │                       │                      │              │  │
│  │  │  ┌────────────────┐  │  ┌────────────────┐   │  ┌────────────────┐  │              │  │
│  │  │  │PUBLIC SUBNET   │  │  │PUBLIC SUBNET   │   │  │PUBLIC SUBNET   │  │              │  │
│  │  │  │10.0.0.0/22     │  │  │10.0.4.0/22     │   │  │10.0.8.0/22     │  │              │  │
│  │  │  │                │  │  │                │   │  │                │  │              │  │
│  │  │  │ ┌──────┐       │  │  │ ┌──────┐       │   │  │ ┌──────┐       │  │              │  │
│  │  │  │ │ NAT  │       │  │  │ │ NAT  │       │   │  │ │ NAT  │       │  │              │  │
│  │  │  │ │  GW  │       │  │  │ │  GW  │       │   │  │ │  GW  │       │  │              │  │
│  │  │  │ └──────┘       │  │  │ └──────┘       │   │  │ └──────┘       │  │              │  │
│  │  │  │    ALB         │  │  │    ALB         │   │  │    ALB         │  │              │  │
│  │  │  └────────────────┘  │  └────────────────┘   │  └────────────────┘  │              │  │
│  │  │                      │                       │                      │              │  │
│  │  │  ┌────────────────┐  │  ┌────────────────┐   │  ┌────────────────┐  │              │  │
│  │  │  │PRIVATE SUBNET  │  │  │PRIVATE SUBNET  │   │  │PRIVATE SUBNET  │  │              │  │
│  │  │  │10.0.16.0/20    │  │  │10.0.32.0/20    │   │  │10.0.48.0/20    │  │              │  │
│  │  │  │                │  │  │                │   │  │                │  │              │  │
│  │  │  │ ┌────────────┐ │  │  │ ┌────────────┐ │   │  │ ┌────────────┐ │  │              │  │
│  │  │  │ │  EKS NODES │ │  │  │ │  EKS NODES │ │   │  │ │  EKS NODES │ │  │              │  │
│  │  │  │ │ (m5.2xlarge│ │  │  │ │ (m5.2xlarge│ │   │  │ │ (m5.2xlarge│ │  │              │  │
│  │  │  │ │  + mixed)  │ │  │  │ │  + mixed)  │ │   │  │ │  + mixed)  │ │  │              │  │
│  │  │  │ └────────────┘ │  │  │ └────────────┘ │   │  │ └────────────┘ │  │              │  │
│  │  │  └────────────────┘  │  └────────────────┘   │  └────────────────┘  │              │  │
│  │  │                      │                       │                      │              │  │
│  │  │  ┌────────────────┐  │  ┌────────────────┐   │  ┌────────────────┐  │              │  │
│  │  │  │POD SUBNET      │  │  │POD SUBNET      │   │  │POD SUBNET      │  │              │  │
│  │  │  │100.64.0.0/18   │  │  │100.64.64.0/18  │   │  │100.64.128.0/18 │  │              │  │
│  │  │  │(16,384 pod IPs)│  │  │(16,384 pod IPs)│   │  │(16,384 pod IPs │  │              │  │
│  │  │  └────────────────┘  │  └────────────────┘   │  └────────────────┘  │              │  │
│  │  │                      │                       │                      │              │  │
│  │  │  ┌────────────────┐  │  ┌────────────────┐   │  ┌────────────────┐  │              │  │
│  │  │  │DATA SUBNET     │  │  │DATA SUBNET     │   │  │DATA SUBNET     │  │              │  │
│  │  │  │10.0.64.0/22    │  │  │10.0.68.0/22    │   │  │10.0.72.0/22    │  │              │  │
│  │  │  │                │  │  │                │   │  │                │  │              │  │
│  │  │  │  RDS Primary/  │  │  │  RDS Standby   │   │  │  ElastiCache   │  │              │  │
│  │  │  │  ElastiCache   │  │  │  Replica       │   │  │  Replica       │  │              │  │
│  │  │  └────────────────┘  │  └────────────────┘   │  └────────────────┘  │              │  │
│  │  └──────────────────────┴───────────────────────┴──────────────────────┘              │  │
│  │                                                                                       │  │
│  │  ┌──────────────────────────────────────────────────────────────┐                     │  │
│  │  │                    VPC ENDPOINTS                             │                     │  │
│  │  │  S3 (Gateway) │ ECR (Interface) │ STS (Interface)            │                     │  │
│  │  │  CloudWatch Logs │ Secrets Manager │ KMS                     │                     │  │
│  │  └──────────────────────────────────────────────────────────────┘                     │  │
│  └───────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────────────┐           │
│  │                         SECURITY BASELINE                                    │           │
│  │  CloudTrail → S3 (encrypted, object lock)                                    │           │
│  │  VPC Flow Logs → CloudWatch Logs + S3                                        │           │
│  │  KMS Keys: eks-secrets | rds-encryption | s3-encryption | general            │           │
│  │  S3 Buckets: terraform-state | cloudtrail | flow-logs | compliance           │           │
│  └──────────────────────────────────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
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
# ─── DATA TIER MODULE VARIABLES ─────────────────────────────────────

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project" {
  description = "Project name"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "data_subnet_ids" {
  description = "Subnet IDs for data tier (private data subnets)"
  type        = list(string)
}

variable "eks_node_security_group_id" {
  description = "Security group ID of EKS nodes (allowed to access data tier)"
  type        = string
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}

# ─── RDS VARIABLES ──────────────────────────────────────────────────

variable "rds_instances" {
  description = "Map of RDS instances to create"
  type = map(object({
    engine_version        = string
    instance_class        = string
    allocated_storage     = number
    max_allocated_storage = number
    database_name         = string
    multi_az              = bool
    backup_retention_period = number
    deletion_protection   = bool
  }))
  default = {
    payments = {
      engine_version        = "15.5"
      instance_class        = "db.r6g.xlarge"
      allocated_storage     = 100
      max_allocated_storage = 500
      database_name         = "payments"
      multi_az              = true
      backup_retention_period = 30
      deletion_protection   = true
    }
    orders = {
      engine_version        = "15.5"
      instance_class        = "db.r6g.large"
      allocated_storage     = 100
      max_allocated_storage = 500
      database_name         = "orders"
      multi_az              = true
      backup_retention_period = 30
      deletion_protection   = true
    }
    users = {
      engine_version        = "15.5"
      instance_class        = "db.r6g.large"
      allocated_storage     = 50
      max_allocated_storage = 200
      database_name         = "users"
      multi_az              = true
      backup_retention_period = 30
      deletion_protection   = true
    }
  }
}

variable "rds_kms_key_arn" {
  description = "KMS key ARN for RDS encryption"
  type        = string
}

variable "rds_master_password_management" {
  description = "Use AWS-managed master password via Secrets Manager"
  type        = bool
  default     = true
}

variable "rds_performance_insights_enabled" {
  description = "Enable Performance Insights"
  type        = bool
  default     = true
}

variable "rds_monitoring_interval" {
  description = "Enhanced Monitoring interval in seconds (0 to disable)"
  type        = number
  default     = 60
}

# ─── ELASTICACHE VARIABLES ──────────────────────────────────────────

variable "redis_node_type" {
  description = "ElastiCache Redis node type"
  type        = string
  default     = "cache.r6g.large"
}

variable "redis_num_cache_clusters" {
  description = "Number of cache clusters (nodes) in the replication group"
  type        = number
  default     = 3  # 1 primary + 2 replicas across 3 AZs
}

variable "redis_engine_version" {
  description = "Redis engine version"
  type        = string
  default     = "7.1"
}

variable "redis_parameter_group_family" {
  description = "Redis parameter group family"
  type        = string
  default     = "redis7"
}

variable "redis_kms_key_arn" {
  description = "KMS key ARN for ElastiCache encryption"
  type        = string
}

variable "redis_snapshot_retention_limit" {
  description = "Number of days to retain Redis snapshots"
  type        = number
  default     = 7
}

variable "redis_maintenance_window" {
  description = "Weekly maintenance window"
  type        = string
  default     = "sun:05:00-sun:07:00"
}

variable "redis_snapshot_window" {
  description = "Daily snapshot window"
  type        = string
  default     = "03:00-05:00"
}
```

### `modules/data-tier/main.tf`

```hcl
# ─── DATA TIER MODULE ───────────────────────────────────────────────
# RDS PostgreSQL + ElastiCache Redis
# All resources in data subnets, accessible only from EKS nodes
# ─────────────────────────────────────────────────────────────────────

locals {
  rds_common_tags = merge(var.tags, {
    Component = "data-tier"
    Tier      = "data"
  })
}

# ═══════════════════════════════════════════════════════════════════
# RDS POSTGRESQL
# ═══════════════════════════════════════════════════════════════════

# ─── Subnet Group ─────────────────────────────────────────────────
resource "aws_db_subnet_group" "main" {
  name       = "${var.project}-${var.environment}-data"
  subnet_ids = var.data_subnet_ids

  tags = merge(local.rds_common_tags, {
    Name = "${var.project}-${var.environment}-data-subnet-group"
  })
}

# ─── Security Group for RDS ──────────────────────────────────────
resource "aws_security_group" "rds" {
  name_prefix = "${var.project}-${var.environment}-rds-"
  vpc_id      = var.vpc_id
  description = "RDS PostgreSQL - data tier. Access from EKS nodes only."

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.eks_node_security_group_id]
    description     = "PostgreSQL from EKS pods"
  }

  # No broad egress — RDS doesn't need outbound internet
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [data.aws_vpc.current.cidr_block]
    description = "VPC internal only"
  }

  tags = merge(local.rds_common_tags, {
    Name = "${var.project}-${var.environment}-rds"
  })

  lifecycle {
    create_before_destroy = true
  }
}

data "aws_vpc" "current" {
  id = var.vpc_id
}

# ─── Parameter Group ─────────────────────────────────────────────
resource "aws_db_parameter_group" "postgres15" {
  name_prefix = "${var.project}-${var.environment}-pg15-"
  family      = "postgres15"
  description = "NovaMart PostgreSQL 15 parameters"

  # Connection management
  parameter {
    name  = "max_connections"
    value = "500"
    # 200 microservices × 2-3 connections per pod + headroom
    # RDS applies per-instance, not per-database
  }

  # Logging for compliance (PCI Req 10)
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
    # Log DDL statements (CREATE, ALTER, DROP)
    # "all" is too noisy for production
  }

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"
    # Log queries taking longer than 1 second (slow query detection)
  }

  # SSL enforcement
  parameter {
    name  = "rds.force_ssl"
    value = "1"
    # Enforce TLS for all connections — PCI Req 4
  }

  # Performance
  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements"
    apply_method = "pending-reboot"
    # Enables query performance tracking
  }

  parameter {
    name  = "pg_stat_statements.track"
    value = "all"
  }

  tags = local.rds_common_tags

  lifecycle {
    create_before_destroy = true
  }
}

# ─── Enhanced Monitoring IAM Role ────────────────────────────────
resource "aws_iam_role" "rds_monitoring" {
  name_prefix = "${var.project}-${var.environment}-rds-monitoring-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "monitoring.rds.amazonaws.com"
        }
      }
    ]
  })

  tags = local.rds_common_tags
}

resource "aws_iam_role_policy_attachment" "rds_monitoring" {
  role       = aws_iam_role.rds_monitoring.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}

# ─── RDS Instances ───────────────────────────────────────────────
resource "aws_db_instance" "main" {
  for_each = var.rds_instances

  identifier = "${var.project}-${var.environment}-${each.key}"

  # Engine
  engine         = "postgres"
  engine_version = each.value.engine_version

  # Sizing
  instance_class        = each.value.instance_class
  allocated_storage     = each.value.allocated_storage
  max_allocated_storage = each.value.max_allocated_storage
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = var.rds_kms_key_arn

  # Database
  db_name = each.value.database_name

  # Authentication — AWS-managed master password (never in Terraform state)
  username                    = "${each.key}_admin"
  manage_master_user_password = var.rds_master_password_management
  # This creates the password in Secrets Manager automatically
  # No password in Terraform state, auto-rotation available

  # Network
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false
  port                   = 5432

  # High Availability
  multi_az = each.value.multi_az

  # Parameters
  parameter_group_name = aws_db_parameter_group.postgres15.name

  # Backup
  backup_retention_period = each.value.backup_retention_period
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:05:00-sun:06:00"
  copy_tags_to_snapshot   = true

  # Deletion protection
  deletion_protection       = each.value.deletion_protection
  skip_final_snapshot       = false
  final_snapshot_identifier = "${var.project}-${var.environment}-${each.key}-final"

  # Monitoring
  performance_insights_enabled          = var.rds_performance_insights_enabled
  performance_insights_retention_period = 7  # Free tier: 7 days
  performance_insights_kms_key_id       = var.rds_kms_key_arn
  monitoring_interval                   = var.rds_monitoring_interval
  monitoring_role_arn                   = aws_iam_role.rds_monitoring.arn
  enabled_cloudwatch_logs_exports       = ["postgresql", "upgrade"]

  # Auto minor version upgrade (security patches)
  auto_minor_version_upgrade = true

  # IAM database authentication (for app access without passwords)
  iam_database_authentication_enabled = true

  tags = merge(local.rds_common_tags, {
    Name    = "${var.project}-${var.environment}-${each.key}"
    Service = each.key
  })

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [
      # Password managed by Secrets Manager, not Terraform
      password,
    ]
  }
}

# ═══════════════════════════════════════════════════════════════════
# ELASTICACHE REDIS
# ═══════════════════════════════════════════════════════════════════

# ─── Subnet Group ─────────────────────────────────────────────────
resource "aws_elasticache_subnet_group" "main" {
  name       = "${var.project}-${var.environment}-redis"
  subnet_ids = var.data_subnet_ids

  tags = local.rds_common_tags
}

# ─── Security Group for Redis ────────────────────────────────────
resource "aws_security_group" "redis" {
  name_prefix = "${var.project}-${var.environment}-redis-"
  vpc_id      = var.vpc_id
  description = "ElastiCache Redis. Access from EKS nodes only."

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [var.eks_node_security_group_id]
    description     = "Redis from EKS pods"
  }

  tags = merge(local.rds_common_tags, {
    Name = "${var.project}-${var.environment}-redis"
  })

  lifecycle {
    create_before_destroy = true
  }
}

# ─── Parameter Group ─────────────────────────────────────────────
resource "aws_elasticache_parameter_group" "redis7" {
  name   = "${var.project}-${var.environment}-redis7"
  family = var.redis_parameter_group_family

  # Memory management
  parameter {
    name  = "maxmemory-policy"
    value = "volatile-lru"
    # Evict keys with TTL set, least recently used first
    # "allkeys-lru" evicts ANY key — dangerous for session data
    # "volatile-lru" only evicts keys with explicit TTL
  }

  # Slow log for debugging
  parameter {
    name  = "slowlog-log-slower-than"
    value = "10000"
    # Log commands taking longer than 10ms (microseconds)
  }

  parameter {
    name  = "slowlog-max-len"
    value = "128"
  }

  # Connection management
  parameter {
    name  = "timeout"
    value = "300"
    # Close idle connections after 5 minutes
    # Prevents connection leaks from misbehaving apps
  }

  # Notifications
  parameter {
    name  = "notify-keyspace-events"
    value = "Ex"
    # E = keyevent events, x = expired events
    # Allows apps to subscribe to key expiration notifications
    # Useful for session expiry callbacks
  }

  tags = local.rds_common_tags
}

# ─── Redis Replication Group ─────────────────────────────────────
resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "${var.project}-${var.environment}-redis"
  description          = "NovaMart ${var.environment} Redis cluster"

  # Engine
  engine               = "redis"
  engine_version       = var.redis_engine_version
  node_type            = var.redis_node_type
  parameter_group_name = aws_elasticache_parameter_group.redis7.name

  # Topology: 1 primary + 2 replicas across 3 AZs
  num_cache_clusters      = var.redis_num_cache_clusters
  automatic_failover_enabled = true
  multi_az_enabled           = true

  # Network
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]
  port               = 6379

  # Encryption
  at_rest_encryption_enabled = true
  kms_key_id                 = var.redis_kms_key_arn
  transit_encryption_enabled = true
  # transit_encryption = TLS for data in flight (PCI Req 4)
  # Apps MUST connect via TLS when this is enabled

  # Auth
  auth_token = null
  # Using IAM authentication instead of auth token
  # (configured at application level with IRSA)

  # Maintenance
  maintenance_window         = var.redis_maintenance_window
  snapshot_window            = var.redis_snapshot_window
  snapshot_retention_limit   = var.redis_snapshot_retention_limit
  auto_minor_version_upgrade = true

  # Notifications
  notification_topic_arn = null  # Will connect to SNS in security-baseline module

  # Logging
  log_delivery_configuration {
    destination      = "/elasticache/${var.project}-${var.environment}/slow-log"
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    log_type         = "slow-log"
  }

  log_delivery_configuration {
    destination      = "/elasticache/${var.project}-${var.environment}/engine-log"
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    log_type         = "engine-log"
  }

  tags = merge(local.rds_common_tags, {
    Name    = "${var.project}-${var.environment}-redis"
    Service = "redis"
  })

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [
      # Cluster count managed by scaling, not Terraform, if auto-scaling enabled
      num_cache_clusters,
    ]
  }
}

# ─── CloudWatch Log Groups for Redis ─────────────────────────────
resource "aws_cloudwatch_log_group" "redis_slow_log" {
  name              = "/elasticache/${var.project}-${var.environment}/slow-log"
  retention_in_days = 30
  kms_key_id        = var.redis_kms_key_arn

  tags = local.rds_common_tags
}

resource "aws_cloudwatch_log_group" "redis_engine_log" {
  name              = "/elasticache/${var.project}-${var.environment}/engine-log"
  retention_in_days = 30
  kms_key_id        = var.redis_kms_key_arn

  tags = local.rds_common_tags
}
```

### `modules/data-tier/outputs.tf`

```hcl
# ─── DATA TIER OUTPUTS ──────────────────────────────────────────────

# ─── RDS Outputs ─────────────────────────────────────────────────
output "rds_endpoints" {
  description = "Map of RDS instance endpoints"
  value = {
    for key, instance in aws_db_instance.main : key => {
      endpoint = instance.endpoint
      address  = instance.address
      port     = instance.port
      name     = instance.db_name
    }
  }
}

output "rds_instance_ids" {
  description = "Map of RDS instance IDs"
  value = {
    for key, instance in aws_db_instance.main : key => instance.id
  }
}

output "rds_master_secret_arns" {
  description = "Map of Secrets Manager ARNs for RDS master passwords"
  value = {
    for key, instance in aws_db_instance.main : key => instance.master_user_secret[0].secret_arn
  }
}

output "rds_security_group_id" {
  description = "Security group ID for RDS instances"
  value       = aws_security_group.rds.id
}

# ─── Redis Outputs ───────────────────────────────────────────────
output "redis_primary_endpoint" {
  description = "Redis primary endpoint address"
  value       = aws_elasticache_replication_group.main.primary_endpoint_address
}

output "redis_reader_endpoint" {
  description = "Redis reader endpoint address"
  value       = aws_elasticache_replication_group.main.reader_endpoint_address
}

output "redis_port" {
  description = "Redis port"
  value       = aws_elasticache_replication_group.main.port
}

output "redis_security_group_id" {
  description = "Security group ID for Redis"
  value       = aws_security_group.redis.id
}

output "rds_subnet_group_name" {
  description = "RDS subnet group name"
  value       = aws_db_subnet_group.main.name
}
```

---

## Module 4: Security Baseline (KMS, S3, CloudTrail)

### `modules/security-baseline/variables.tf`

```hcl
# ─── SECURITY BASELINE MODULE VARIABLES ─────────────────────────────

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project" {
  description = "Project name"
  type        = string
}

variable "aws_account_id" {
  description = "AWS account ID"
  type        = string
}

variable "aws_region" {
  description = "AWS region"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID for VPC Flow Logs"
  type        = string
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}

# ─── CLOUDTRAIL VARIABLES ───────────────────────────────────────────

variable "cloudtrail_s3_data_event_buckets" {
  description = "List of S3 bucket ARNs for CloudTrail data event logging"
  type        = list(string)
  default     = []
}

variable "cloudtrail_log_retention_days" {
  description = "Days to retain CloudTrail logs in CloudWatch"
  type        = number
  default     = 90
}

# ─── S3 VARIABLES ───────────────────────────────────────────────────

variable "terraform_state_bucket_name" {
  description = "S3 bucket name for Terraform state (must be globally unique)"
  type        = string
}

variable "flow_log_retention_days" {
  description = "Days before transitioning flow logs to IA"
  type        = number
  default     = 90
}
```

### `modules/security-baseline/kms.tf`

```hcl
# ═══════════════════════════════════════════════════════════════════
# KMS KEYS — One per domain for separation of duties
# ═══════════════════════════════════════════════════════════════════

# ─── General Encryption Key (S3, CloudWatch, SNS) ────────────────
resource "aws_kms_key" "general" {
  description             = "NovaMart ${var.environment} general encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  rotation_period_in_days = 365

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EnableRootAccount"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.aws_account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "AllowKeyAdministration"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.aws_account_id}:role/OrganizationAdmin"
        }
        Action = [
          "kms:Create*",
          "kms:Describe*",
          "kms:Enable*",
          "kms:List*",
          "kms:Put*",
          "kms:Update*",
          "kms:Revoke*",
          "kms:Disable*",
          "kms:Get*",
          "kms:Delete*",
          "kms:TagResource",
          "kms:UntagResource",
          "kms:ScheduleKeyDeletion",
          "kms:CancelKeyDeletion"
        ]
        Resource = "*"
      },
      {
        Sid    = "AllowServiceUsage"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.aws_account_id}:root"
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "kms:ViaService" = [
              "s3.${var.aws_region}.amazonaws.com",
              "logs.${var.aws_region}.amazonaws.com",
              "sns.${var.aws_region}.amazonaws.com",
              "sqs.${var.aws_region}.amazonaws.com"
            ]
          }
        }
      },
      {
        Sid    = "AllowCloudWatchLogs"
        Effect = "Allow"
        Principal = {
          Service = "logs.${var.aws_region}.amazonaws.com"
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey"
        ]
        Resource = "*"
        Condition = {
          ArnLike = {
            "kms:EncryptionContext:aws:logs:arn" = "arn:aws:logs:${var.aws_region}:${var.aws_account_id}:*"
          }
        }
      }
    ]
  })

  tags = merge(var.tags, {
    Name    = "${var.project}-${var.environment}-general"
    KeyType = "general"
  })
}

resource "aws_kms_alias" "general" {
  name          = "alias/${var.project}-${var.environment}-general"
  target_key_id = aws_kms_key.general.key_id
}

# ─── EKS Secrets Encryption Key ─────────────────────────────────
resource "aws_kms_key" "eks" {
  description             = "NovaMart ${var.environment} EKS secrets encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  rotation_period_in_days = 365

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EnableRootAccount"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.aws_account_id}:root"
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
            "kms:CallerAccount" = var.aws_account_id
          }
        }
      }
    ]
  })

  tags = merge(var.tags, {
    Name    = "${var.project}-${var.environment}-eks"
    KeyType = "eks-secrets"
  })
}

resource "aws_kms_alias" "eks" {
  name          = "alias/${var.project}-${var.environment}-eks"
  target_key_id = aws_kms_key.eks.key_id
}

# ─── RDS Encryption Key ─────────────────────────────────────────
resource "aws_kms_key" "rds" {
  description             = "NovaMart ${var.environment} RDS encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  rotation_period_in_days = 365

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EnableRootAccount"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.aws_account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "AllowRDSService"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.aws_account_id}:root"
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "kms:ViaService" = "rds.${var.aws_region}.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = merge(var.tags, {
    Name    = "${var.project}-${var.environment}-rds"
    KeyType = "rds"
  })
}

resource "aws_kms_alias" "rds" {
  name          = "alias/${var.project}-${var.environment}-rds"
  target_key_id = aws_kms_key.rds.key_id
}

# ─── ElastiCache Encryption Key ─────────────────────────────────
resource "aws_kms_key" "elasticache" {
  description             = "NovaMart ${var.environment} ElastiCache encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  rotation_period_in_days = 365

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EnableRootAccount"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.aws_account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      }
    ]
  })

  tags = merge(var.tags, {
    Name    = "${var.project}-${var.environment}-elasticache"
    KeyType = "elasticache"
  })
}

resource "aws_kms_alias" "elasticache" {
  name          = "alias/${var.project}-${var.environment}-elasticache"
  target_key_id = aws_kms_key.elasticache.key_id
}
```

### `modules/security-baseline/s3.tf`

```hcl
# ═══════════════════════════════════════════════════════════════════
# S3 BUCKETS — Terraform state, CloudTrail, Flow Logs, Compliance
# ═══════════════════════════════════════════════════════════════════

# ─── Terraform State Bucket ──────────────────────────────────────
resource "aws_s3_bucket" "terraform_state" {
  bucket = var.terraform_state_bucket_name

  # NEVER destroy the state bucket
  force_destroy = false

  tags = merge(var.tags, {
    Name    = var.terraform_state_bucket_name
    Purpose = "terraform-state"
  })

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
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.general.arn
    }
    bucket_key_enabled = true  # Reduces KMS API calls (cost optimization)
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "${var.project}-${var.environment}-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.general.arn
  }

  tags = merge(var.tags, {
    Name    = "${var.project}-${var.environment}-terraform-locks"
    Purpose = "terraform-state-locking"
  })
}

# ─── CloudTrail Bucket ───────────────────────────────────────────
resource "aws_s3_bucket" "cloudtrail" {
  bucket = "${var.project}-cloudtrail-${var.aws_account_id}"

  force_destroy = false

  tags = merge(var.tags, {
    Name       = "${var.project}-cloudtrail-${var.aws_account_id}"
    Purpose    = "cloudtrail-audit-logs"
    Compliance = "pci-soc2"
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
      kms_master_key_id = aws_kms_key.general.arn
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
    # PCI: 1 year minimum. NovaMart: 2 years.
    expiration {
      days = 730
    }
  }
}

# Object Lock for tamper-proof compliance logs
resource "aws_s3_bucket_object_lock_configuration" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  rule {
    default_retention {
      mode = "COMPLIANCE"
      days = 365
    }
  }
}

# Bucket policy for CloudTrail service access
resource "aws_s3_bucket_policy" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AWSCloudTrailAclCheck"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.cloudtrail.arn
      },
      {
        Sid    = "AWSCloudTrailWrite"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.cloudtrail.arn}/cloudtrail/AWSLogs/${var.aws_account_id}/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" = "bucket-owner-full-control"
          }
        }
      },
      {
        Sid    = "DenyUnencryptedUploads"
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

# ─── VPC Flow Logs Bucket ────────────────────────────────────────
resource "aws_s3_bucket" "flow_logs" {
  bucket = "${var.project}-flow-logs-${var.aws_account_id}"

  force_destroy = false

  tags = merge(var.tags, {
    Name    = "${var.project}-flow-logs-${var.aws_account_id}"
    Purpose = "vpc-flow-logs"
  })
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
      kms_master_key_id = aws_kms_key.general.arn
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
    id     = "archive-flow-logs"
    status = "Enabled"

    transition {
      days          = var.flow_log_retention_days
      storage_class = "STANDARD_IA"
    }
    transition {
      days          = 365
      storage_class = "GLACIER"
    }
    expiration {
      days = 730
    }
  }
}

# ─── Compliance Evidence Bucket ──────────────────────────────────
resource "aws_s3_bucket" "compliance" {
  bucket = "${var.project}-compliance-${var.aws_account_id}"

  tags = merge(var.tags, {
    Name       = "${var.project}-compliance-${var.aws_account_id}"
    Purpose    = "compliance-evidence"
    Compliance = "pci-soc2-gdpr"
  })

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "compliance" {
  bucket = aws_s3_bucket.compliance.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "compliance" {
  bucket = aws_s3_bucket.compliance.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.general.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "compliance" {
  bucket = aws_s3_bucket.compliance.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_object_lock_configuration" "compliance" {
  bucket = aws_s3_bucket.compliance.id

  rule {
    default_retention {
      mode = "COMPLIANCE"
      days = 365
    }
  }
}
```

### `modules/security-baseline/cloudtrail.tf`

```hcl
# ═══════════════════════════════════════════════════════════════════
# CLOUDTRAIL — Multi-region, tamper-proof, PCI/SOC2 compliant
# ═══════════════════════════════════════════════════════════════════

# ─── CloudWatch Log Group for CloudTrail ─────────────────────────
resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "/cloudtrail/${var.project}-${var.environment}"
  retention_in_days = var.cloudtrail_log_retention_days
  kms_key_id        = aws_kms_key.general.arn

  tags = merge(var.tags, {
    Purpose    = "cloudtrail-realtime"
    Compliance = "pci-req10"
  })
}

# ─── IAM Role for CloudTrail → CloudWatch ────────────────────────
resource "aws_iam_role" "cloudtrail_cloudwatch" {
  name_prefix = "${var.project}-${var.environment}-ct-cw-"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
      }
    ]
  })

  tags = var.tags
}

resource "aws_iam_role_policy" "cloudtrail_cloudwatch" {
  name_prefix = "${var.project}-ct-cw-"
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

# ─── CloudTrail ──────────────────────────────────────────────────
resource "aws_cloudtrail" "main" {
  name                          = "${var.project}-${var.environment}"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  s3_key_prefix                 = "cloudtrail"
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  # Real-time alerting via CloudWatch
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_cloudwatch.arn

  # KMS encryption
  kms_key_id = aws_kms_key.general.arn

  # Management events (all API calls)
  event_selector {
    read_write_type           = "All"
    include_management_events = true

    # S3 data events for sensitive buckets only (cost-conscious)
    dynamic "data_resource" {
      for_each = length(var.cloudtrail_s3_data_event_buckets) > 0 ? [1] : []
      content {
        type   = "AWS::S3::Object"
        values = var.cloudtrail_s3_data_event_buckets
      }
    }
  }

  # Insights — detect unusual API activity patterns
  insight_selector {
    insight_type = "ApiCallRateInsight"
  }

  insight_selector {
    insight_type = "ApiErrorRateInsight"
  }

  tags = merge(var.tags, {
    Name       = "${var.project}-${var.environment}-cloudtrail"
    Compliance = "pci-soc2"
  })

  depends_on = [
    aws_s3_bucket_policy.cloudtrail
  ]
}

# ═══════════════════════════════════════════════════════════════════
# VPC FLOW LOGS
# ═══════════════════════════════════════════════════════════════════

resource "aws_flow_log" "main" {
  vpc_id               = var.vpc_id
  traffic_type         = "ALL"
  log_destination_type = "s3"
  log_destination      = aws_s3_bucket.flow_logs.arn

  max_aggregation_interval = 60  # 60 seconds for production forensic granularity

  # Custom log format with enhanced fields
  log_format = join(" ", [
    "$${version}",
    "$${account-id}",
    "$${interface-id}",
    "$${srcaddr}",
    "$${dstaddr}",
    "$${srcport}",
    "$${dstport}",
    "$${protocol}",
    "$${packets}",
    "$${bytes}",
    "$${start}",
    "$${end}",
    "$${action}",
    "$${log-status}",
    "$${vpc-id}",
    "$${subnet-id}",
    "$${az-id}",
    "$${pkt-srcaddr}",
    "$${pkt-dstaddr}",
    "$${region}",
    "$${pkt-src-aws-service}",
    "$${pkt-dst-aws-service}",
    "$${flow-direction}",
    "$${traffic-path}"
  ])

  tags = merge(var.tags, {
    Name = "${var.project}-${var.environment}-vpc-flow-logs"
  })
}
```

### `modules/security-baseline/outputs.tf`

```hcl
# ─── SECURITY BASELINE OUTPUTS ──────────────────────────────────────

# ─── KMS Key ARNs ───────────────────────────────────────────────
output "kms_key_arns" {
  description = "Map of KMS key ARNs by purpose"
  value = {
    general     = aws_kms_key.general.arn
    eks         = aws_kms_key.eks.arn
    rds         = aws_kms_key.rds.arn
    elasticache = aws_kms_key.elasticache.arn
  }
}

output "kms_key_ids" {
  description = "Map of KMS key IDs by purpose"
  value = {
    general     = aws_kms_key.general.key_id
    eks         = aws_kms_key.eks.key_id
    rds         = aws_kms_key.rds.key_id
    elasticache = aws_kms_key.elasticache.key_id
  }
}

# ─── S3 Bucket ARNs ─────────────────────────────────────────────
output "s3_bucket_arns" {
  description = "Map of S3 bucket ARNs"
  value = {
    terraform_state = aws_s3_bucket.terraform_state.arn
    cloudtrail      = aws_s3_bucket.cloudtrail.arn
    flow_logs       = aws_s3_bucket.flow_logs.arn
    compliance      = aws_s3_bucket.compliance.arn
  }
}

output "s3_bucket_ids" {
  description = "Map of S3 bucket IDs (names)"
  value = {
    terraform_state = aws_s3_bucket.terraform_state.id
    cloudtrail      = aws_s3_bucket.cloudtrail.id
    flow_logs       = aws_s3_bucket.flow_logs.id
    compliance      = aws_s3_bucket.compliance.id
  }
}

# ─── DynamoDB ────────────────────────────────────────────────────
output "terraform_lock_table_name" {
  description = "DynamoDB table name for Terraform state locking"
  value       = aws_dynamodb_table.terraform_locks.name
}

# ─── CloudTrail ──────────────────────────────────────────────────
output "cloudtrail_arn" {
  description = "CloudTrail ARN"
  value       = aws_cloudtrail.main.arn
}

output "cloudtrail_log_group_name" {
  description = "CloudWatch log group for CloudTrail"
  value       = aws_cloudwatch_log_group.cloudtrail.name
}

# ─── Flow Logs ───────────────────────────────────────────────────
output "flow_log_id" {
  description = "VPC Flow Log ID"
  value       = aws_flow_log.main.id
}
```

---

## Module 5: Security Groups

### `modules/security-groups/variables.tf`

```hcl
# ─── SECURITY GROUPS MODULE VARIABLES ────────────────────────────────

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "project" {
  description = "Project name"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
}

variable "cloudflare_ip_ranges" {
  description = "Cloudflare IP ranges for ALB ingress restriction"
  type        = list(string)
  # Updated by automation from https://api.cloudflare.com/client/v4/ips
  # Default includes current Cloudflare IPv4 ranges
  default = [
    "173.245.48.0/20",
    "103.21.244.0/22",
    "103.22.200.0/22",
    "103.31.4.0/22",
    "141.101.64.0/18",
    "108.162.192.0/18",
    "190.93.240.0/20",
    "188.114.96.0/20",
    "197.234.240.0/22",
    "198.41.128.0/17",
    "162.158.0.0/15",
    "104.16.0.0/13",
    "104.24.0.0/14",
    "172.64.0.0/13",
    "131.0.72.0/22"
  ]
}

variable "eks_control_plane_security_group_id" {
  description = "EKS control plane security group ID (EKS-managed)"
  type        = string
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}
```

### `modules/security-groups/main.tf`

```hcl
# ═══════════════════════════════════════════════════════════════════
# SECURITY GROUPS — SG-to-SG references for clean trust chain
#
# Trust chain: Cloudflare → ALB → EKS Nodes → Data Tier
# ═══════════════════════════════════════════════════════════════════

locals {
  sg_tags = merge(var.tags, {
    Component = "security-groups"
  })
}

# ─── ALB Security Group ─────────────────────────────────────────
resource "aws_security_group" "alb" {
  name_prefix = "${var.project}-${var.environment}-alb-"
  vpc_id      = var.vpc_id
  description = "Internet-facing ALB. HTTPS from Cloudflare only."

  tags = merge(local.sg_tags, {
    Name = "${var.project}-${var.environment}-alb"
    Tier = "public"
  })

  lifecycle {
    create_before_destroy = true
  }
}

# ALB ingress: HTTPS from Cloudflare IPs only
resource "aws_security_group_rule" "alb_ingress_https" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = var.cloudflare_ip_ranges
  security_group_id = aws_security_group.alb.id
  description       = "HTTPS from Cloudflare"
}

# ALB egress: to EKS nodes only
resource "aws_security_group_rule" "alb_egress_to_eks" {
  type                     = "egress"
  from_port                = 0
  to_port                  = 65535
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.eks_node.id
  security_group_id        = aws_security_group.alb.id
  description              = "To EKS worker nodes"
}

# ─── EKS Node Security Group ────────────────────────────────────
resource "aws_security_group" "eks_node" {
  name_prefix = "${var.project}-${var.environment}-eks-node-"
  vpc_id      = var.vpc_id
  description = "EKS worker nodes. Pod-to-pod and ALB traffic."

  tags = merge(local.sg_tags, {
    Name = "${var.project}-${var.environment}-eks-node"
    Tier = "private"
    # Required tag for EKS to discover this SG
    "kubernetes.io/cluster/${var.project}-${var.environment}" = "owned"
  })

  lifecycle {
    create_before_destroy = true
  }
}

# EKS nodes: ingress from ALB
resource "aws_security_group_rule" "eks_node_ingress_alb" {
  type                     = "ingress"
  from_port                = 0
  to_port                  = 65535
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.alb.id
  security_group_id        = aws_security_group.eks_node.id
  description              = "From ALB"
}

# EKS nodes: ingress from other nodes (pod-to-pod via VPC CNI)
resource "aws_security_group_rule" "eks_node_ingress_self_tcp" {
  type              = "ingress"
  from_port         = 0
  to_port           = 65535
  protocol          = "tcp"
  self              = true
  security_group_id = aws_security_group.eks_node.id
  description       = "Node-to-node TCP (pod communication)"
}

resource "aws_security_group_rule" "eks_node_ingress_self_udp" {
  type              = "ingress"
  from_port         = 0
  to_port           = 65535
  protocol          = "udp"
  self              = true
  security_group_id = aws_security_group.eks_node.id
  description       = "Node-to-node UDP (DNS, VXLAN)"
}

# EKS nodes: ingress from EKS control plane (kubelet, webhooks)
resource "aws_security_group_rule" "eks_node_ingress_cp_kubelet" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = var.eks_control_plane_security_group_id
  security_group_id        = aws_security_group.eks_node.id
  description              = "EKS control plane to kubelet HTTPS"
}

resource "aws_security_group_rule" "eks_node_ingress_cp_webhooks" {
  type                     = "ingress"
  from_port                = 1025
  to_port                  = 65535
  protocol                 = "tcp"
  source_security_group_id = var.eks_control_plane_security_group_id
  security_group_id        = aws_security_group.eks_node.id
  description              = "EKS control plane to pods (webhooks, metrics)"
}

# EKS nodes: egress to internet via NAT Gateway
resource "aws_security_group_rule" "eks_node_egress_all" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.eks_node.id
  description       = "Outbound (via NAT Gateway for external dependencies)"
}

# ─── VPC Endpoints Security Group ────────────────────────────────
resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "${var.project}-${var.environment}-vpce-"
  vpc_id      = var.vpc_id
  description = "VPC Endpoints (ECR, KMS, STS, etc.)"

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_node.id]
    description     = "HTTPS from EKS nodes to VPC endpoints"
  }

  # Also allow from VPC CIDR for non-EKS services (Lambda, etc.)
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "HTTPS from VPC CIDR"
  }

  tags = merge(local.sg_tags, {
    Name = "${var.project}-${var.environment}-vpc-endpoints"
    Tier = "infrastructure"
  })

  lifecycle {
    create_before_destroy = true
  }
}
```

### `modules/security-groups/outputs.tf`

```hcl
# ─── SECURITY GROUPS OUTPUTS ────────────────────────────────────────

output "alb_security_group_id" {
  description = "ALB security group ID"
  value       = aws_security_group.alb.id
}

output "eks_node_security_group_id" {
  description = "EKS node security group ID"
  value       = aws_security_group.eks_node.id
}

output "vpc_endpoints_security_group_id" {
  description = "VPC endpoints security group ID"
  value       = aws_security_group.vpc_endpoints.id
}

output "security_group_ids" {
  description = "All security group IDs as a map"
  value = {
    alb           = aws_security_group.alb.id
    eks_node      = aws_security_group.eks_node.id
    vpc_endpoints = aws_security_group.vpc_endpoints.id
  }
}
```

---

## Root Module: `environments/production/`

### `environments/production/versions.tf`

# NovaMart Production Environment — Complete Compositions

---

## Remaining EKS Module Files (Not Yet Delivered)

Before wiring environments, these module files were referenced in the directory tree but not yet provided.

### `modules/eks/node-groups.tf` (Completion)

```hcl
# modules/eks/node-groups.tf
# (Continues from IAM role + policy attachments already delivered)

# ─────────────────────────────────────────────
# LAUNCH TEMPLATE (shared hardening baseline)
# ─────────────────────────────────────────────

resource "aws_launch_template" "system_nodes" {
  name_prefix = "${local.name_prefix}-system-"
  description = "Launch template for EKS system node group"

  # IMDSv2 enforcement — prevents SSRF-based credential theft
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2 ONLY
    http_put_response_hop_limit = 1           # Pods cannot reach IMDS (VPC CNI handles this)
    instance_metadata_tags      = "enabled"
  }

  # EBS root volume encryption
  block_device_mappings {
    device_name = "/dev/xvda"

    ebs {
      volume_size           = 100
      volume_type           = "gp3"
      iops                  = 3000
      throughput            = 125
      encrypted             = true
      kms_key_id            = var.eks_secrets_kms_key_arn
      delete_on_termination = true
    }
  }

  # Monitoring
  monitoring {
    enabled = true  # Detailed CloudWatch monitoring (1-minute intervals)
  }

  # Network — no public IP on nodes
  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.node.id]
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
      Name      = "${local.name_prefix}-system-node-vol"
      Encrypted = "true"
    })
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-system-lt"
  })

  lifecycle {
    create_before_destroy = true
  }
}

# ─────────────────────────────────────────────
# NODE SECURITY GROUP
# ─────────────────────────────────────────────

resource "aws_security_group" "node" {
  name_prefix = "${local.name_prefix}-eks-node-"
  description = "EKS worker node security group — pod-to-pod and control plane communication"
  vpc_id      = var.vpc_id

  tags = merge(local.common_tags, {
    Name                                          = "${local.name_prefix}-eks-node-sg"
    "kubernetes.io/cluster/${var.cluster_name}"    = "owned"
  })

  lifecycle {
    create_before_destroy = true
  }
}

# Node-to-node (pod-to-pod via VPC CNI)
resource "aws_security_group_rule" "node_self_ingress" {
  type              = "ingress"
  from_port         = 0
  to_port           = 65535
  protocol          = "-1"
  self              = true
  security_group_id = aws_security_group.node.id
  description       = "Node-to-node all traffic (pods, DNS, metrics)"
}

# Control plane → nodes (kubelet API, webhook callbacks)
resource "aws_security_group_rule" "node_cp_ingress_443" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.cluster.id
  security_group_id        = aws_security_group.node.id
  description              = "Control plane to kubelet HTTPS"
}

resource "aws_security_group_rule" "node_cp_ingress_high_ports" {
  type                     = "ingress"
  from_port                = 1025
  to_port                  = 65535
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.cluster.id
  security_group_id        = aws_security_group.node.id
  description              = "Control plane to pods (webhooks, admission controllers)"
}

# Nodes → control plane (API server)
resource "aws_security_group_rule" "cp_node_ingress" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.node.id
  security_group_id        = aws_security_group.cluster.id
  description              = "Worker nodes to EKS API server"
}

# Node egress — outbound to internet via NAT, VPC endpoints, and data tier
resource "aws_security_group_rule" "node_egress_all" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.node.id
  description       = "All outbound (NAT GW for internet, direct for VPC resources)"
}

# CoreDNS port (UDP 53) — explicit for clarity in audits
resource "aws_security_group_rule" "node_coredns_tcp" {
  type              = "ingress"
  from_port         = 53
  to_port           = 53
  protocol          = "tcp"
  self              = true
  security_group_id = aws_security_group.node.id
  description       = "CoreDNS TCP"
}

resource "aws_security_group_rule" "node_coredns_udp" {
  type              = "ingress"
  from_port         = 53
  to_port           = 53
  protocol          = "udp"
  self              = true
  security_group_id = aws_security_group.node.id
  description       = "CoreDNS UDP"
}

# ─────────────────────────────────────────────
# SYSTEM NODE GROUP
# Runs: CoreDNS, kube-proxy, Karpenter, DaemonSets
# This is the "always-on" foundation — never scaled to zero
# ─────────────────────────────────────────────

resource "aws_eks_node_group" "system" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${local.name_prefix}-system"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = var.system_node_desired
    min_size     = var.system_node_min
    max_size     = var.system_node_max
  }

  # Use our hardened launch template
  launch_template {
    id      = aws_launch_template.system_nodes.id
    version = aws_launch_template.system_nodes.latest_version
  }

  instance_types = var.system_node_instance_types

  # Kubernetes labels for scheduling
  labels = {
    "novamart.com/node-type" = "system"
    "novamart.com/lifecycle" = "on-demand"
  }

  # Taint system nodes — only system workloads tolerate this
  taint {
    key    = "CriticalAddonsOnly"
    value  = "true"
    effect = "NO_SCHEDULE"
  }

  update_config {
    max_unavailable_percentage = 33  # Rolling update: max 1/3 nodes at a time
  }

  tags = merge(local.common_tags, {
    Name                     = "${local.name_prefix}-system-ng"
    "novamart.com/node-type" = "system"
  })

  depends_on = [
    aws_iam_role_policy_attachment.node_policies,
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size,  # Let autoscaler manage desired count
    ]
  }
}
```

---

### `modules/eks/addons.tf`

```hcl
# modules/eks/addons.tf

# ═══════════════════════════════════════════════════════════════════
# EKS MANAGED ADD-ONS
#
# VPC CNI, CoreDNS, kube-proxy managed by EKS for:
#   - Automatic security patches
#   - Version compatibility guarantees
#   - Simplified upgrade path
# ═══════════════════════════════════════════════════════════════════

# ─────────────────────────────────────────────
# VPC CNI (aws-node DaemonSet)
# ─────────────────────────────────────────────

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
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:kube-system:aws-node"
          }
        }
      }
    ]
  })

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc-cni-irsa"
  })
}

resource "aws_iam_role_policy_attachment" "vpc_cni" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.vpc_cni.name
}

resource "aws_eks_addon" "vpc_cni" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "vpc-cni"

  # Pin to a known-good version — don't auto-upgrade in production
  addon_version            = var.vpc_cni_addon_version
  resolve_conflicts_on_update = "OVERWRITE"
  service_account_role_arn = aws_iam_role.vpc_cni.arn

  configuration_values = jsonencode({
    env = {
      # Enable custom networking — pods use secondary CIDR subnets
      AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG = "true"
      # ENI config label — maps nodes to pod subnets by AZ
      ENI_CONFIG_LABEL_DEF               = "topology.kubernetes.io/zone"
      # Enable prefix delegation for higher pod density per node
      ENABLE_PREFIX_DELEGATION           = "true"
      # Warm prefix target — pre-allocate IPs for fast pod startup
      WARM_PREFIX_TARGET                 = "1"
      # Minimum IPs to keep warm
      MINIMUM_IP_TARGET                  = "5"
      # Disable SNAT for pods — pods use their VPC IP directly
      # Required when using custom networking with secondary CIDR
      AWS_VPC_K8S_CNI_EXTERNALSNAT       = "true"
    }
    # Enable NetworkPolicy support (native VPC CNI network policies)
    enableNetworkPolicy = "true"
  })

  tags = local.common_tags

  depends_on = [
    aws_eks_node_group.system,
  ]
}

# ─────────────────────────────────────────────
# ENI CONFIG (maps AZs to pod subnets)
# Applied via kubectl after cluster bootstrap
# Defined here as a reference for the bootstrap script
# ─────────────────────────────────────────────

# NOTE: ENIConfig custom resources must be created in-cluster.
# This is handled by the bootstrap Helm chart or kubectl apply.
# The config maps each AZ to its corresponding pod subnet:
#
# apiVersion: crd.k8s.amazonaws.com/v1alpha1
# kind: ENIConfig
# metadata:
#   name: us-east-1a
# spec:
#   securityGroups:
#     - <node_security_group_id>
#   subnet: <pod_subnet_id_az_a>
#
# Repeated for us-east-1b and us-east-1c

# ─────────────────────────────────────────────
# COREDNS
# ─────────────────────────────────────────────

resource "aws_eks_addon" "coredns" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "coredns"

  addon_version               = var.coredns_addon_version
  resolve_conflicts_on_update = "OVERWRITE"

  configuration_values = jsonencode({
    replicaCount = 3  # One per AZ for HA DNS
    resources = {
      requests = {
        cpu    = "100m"
        memory = "128Mi"
      }
      limits = {
        cpu    = "250m"
        memory = "256Mi"
      }
    }
    # Tolerate system node taint so CoreDNS runs on system nodes
    tolerations = [
      {
        key      = "CriticalAddonsOnly"
        operator = "Equal"
        value    = "true"
        effect   = "NoSchedule"
      }
    ]
    affinity = {
      podAntiAffinity = {
        preferredDuringSchedulingIgnoredDuringExecution = [
          {
            weight = 100
            podAffinityTerm = {
              labelSelector = {
                matchExpressions = [
                  {
                    key      = "k8s-app"
                    operator = "In"
                    values   = ["kube-dns"]
                  }
                ]
              }
              topologyKey = "kubernetes.io/hostname"
            }
          }
        ]
      }
    }
    # TopologySpreadConstraints for AZ distribution
    topologySpreadConstraints = [
      {
        maxSkew           = 1
        topologyKey       = "topology.kubernetes.io/zone"
        whenUnsatisfiable = "DoNotSchedule"
        labelSelector = {
          matchLabels = {
            "k8s-app" = "kube-dns"
          }
        }
      }
    ]
  })

  tags = local.common_tags

  depends_on = [
    aws_eks_node_group.system,
  ]
}

# ─────────────────────────────────────────────
# KUBE-PROXY
# ─────────────────────────────────────────────

resource "aws_eks_addon" "kube_proxy" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "kube-proxy"

  addon_version               = var.kube_proxy_addon_version
  resolve_conflicts_on_update = "OVERWRITE"

  tags = local.common_tags

  depends_on = [
    aws_eks_node_group.system,
  ]
}

# ─────────────────────────────────────────────
# EBS CSI DRIVER (for persistent volumes)
# ─────────────────────────────────────────────

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
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:kube-system:ebs-csi-controller-sa"
          }
        }
      }
    ]
  })

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-ebs-csi-irsa"
  })
}

resource "aws_iam_role_policy_attachment" "ebs_csi" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  role       = aws_iam_role.ebs_csi.name
}

# Custom KMS policy for EBS CSI — allows encrypting PVs with our KMS key
resource "aws_iam_role_policy" "ebs_csi_kms" {
  name_prefix = "ebs-csi-kms-"
  role        = aws_iam_role.ebs_csi.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "kms:CreateGrant",
          "kms:ListGrants",
          "kms:RevokeGrant"
        ]
        Resource = var.eks_secrets_kms_key_arn
        Condition = {
          Bool = {
            "kms:GrantIsForAWSResource" = "true"
          }
        }
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey"
        ]
        Resource = var.eks_secrets_kms_key_arn
      }
    ]
  })
}

resource "aws_eks_addon" "ebs_csi" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "aws-ebs-csi-driver"

  addon_version               = var.ebs_csi_addon_version
  resolve_conflicts_on_update = "OVERWRITE"
  service_account_role_arn    = aws_iam_role.ebs_csi.arn

  tags = local.common_tags

  depends_on = [
    aws_eks_node_group.system,
  ]
}
```

---

### `modules/eks/karpenter.tf`

```hcl
# modules/eks/karpenter.tf

# ═══════════════════════════════════════════════════════════════════
# KARPENTER — Just-in-time node provisioning
#
# Architecture:
#   - Karpenter controller runs on the system managed node group
#   - Karpenter provisions/deprovisions application workload nodes
#   - Uses EC2 Fleet API for optimal instance selection
#   - Supports Spot for non-critical, On-Demand for critical
# ═══════════════════════════════════════════════════════════════════

# ─────────────────────────────────────────────
# KARPENTER IAM: IRSA for the controller pod
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
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:kube-system:karpenter"
          }
        }
      }
    ]
  })

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-karpenter-irsa"
  })
}

resource "aws_iam_role_policy" "karpenter_controller" {
  name_prefix = "karpenter-"
  role        = aws_iam_role.karpenter_controller.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EC2NodeManagement"
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
        Sid    = "EC2TerminateScoped"
        Effect = "Allow"
        Action = "ec2:TerminateInstances"
        Resource = "*"
        Condition = {
          StringEquals = {
            "ec2:ResourceTag/karpenter.sh/discovery" = var.cluster_name
          }
        }
      },
      {
        Sid    = "PassNodeRole"
        Effect = "Allow"
        Action = "iam:PassRole"
        Resource = aws_iam_role.node.arn
      },
      {
        Sid    = "EKSClusterAccess"
        Effect = "Allow"
        Action = [
          "eks:DescribeCluster"
        ]
        Resource = aws_eks_cluster.main.arn
      },
      {
        Sid    = "SSMGetAMI"
        Effect = "Allow"
        Action = "ssm:GetParameter"
        Resource = "arn:aws:ssm:${data.aws_region.current.name}::parameter/aws/service/eks/optimized-ami/*"
      },
      {
        Sid    = "PricingAPI"
        Effect = "Allow"
        Action = [
          "pricing:GetProducts"
        ]
        Resource = "*"
      },
      {
        Sid    = "SQSInterruptionQueue"
        Effect = "Allow"
        Action = [
          "sqs:DeleteMessage",
          "sqs:GetQueueUrl",
          "sqs:ReceiveMessage",
          "sqs:GetQueueAttributes"
        ]
        Resource = aws_sqs_queue.karpenter_interruption.arn
      }
    ]
  })
}

# ─────────────────────────────────────────────
# KARPENTER NODE IAM: Instance Profile
# (Reuses the node IAM role from node-groups.tf)
# ─────────────────────────────────────────────

resource "aws_iam_instance_profile" "karpenter_node" {
  name_prefix = "${local.name_prefix}-karpenter-node-"
  role        = aws_iam_role.node.name

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-karpenter-node-profile"
  })
}

# ─────────────────────────────────────────────
# SPOT INTERRUPTION HANDLING
# Karpenter listens to SQS for spot interruption
# and rebalance recommendation events
# ─────────────────────────────────────────────

resource "aws_sqs_queue" "karpenter_interruption" {
  name                       = "${local.name_prefix}-karpenter-interruption"
  message_retention_seconds  = 300
  sqs_managed_sse_enabled    = true
  receive_wait_time_seconds  = 20  # Long polling

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-karpenter-interruption"
  })
}

resource "aws_sqs_queue_policy" "karpenter_interruption" {
  queue_url = aws_sqs_queue.karpenter_interruption.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowEC2Events"
        Effect = "Allow"
        Principal = {
          Service = [
            "events.amazonaws.com",
            "sqs.amazonaws.com"
          ]
        }
        Action   = "sqs:SendMessage"
        Resource = aws_sqs_queue.karpenter_interruption.arn
      }
    ]
  })
}

# EventBridge rules for Spot interruption, rebalance, state change, health
resource "aws_cloudwatch_event_rule" "spot_interruption" {
  name_prefix = "${local.name_prefix}-spot-int-"
  description = "EC2 Spot Instance Interruption Warning"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Spot Instance Interruption Warning"]
  })

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "spot_interruption" {
  rule = aws_cloudwatch_event_rule.spot_interruption.name
  arn  = aws_sqs_queue.karpenter_interruption.arn
}

resource "aws_cloudwatch_event_rule" "instance_rebalance" {
  name_prefix = "${local.name_prefix}-rebalance-"
  description = "EC2 Instance Rebalance Recommendation"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance Rebalance Recommendation"]
  })

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "instance_rebalance" {
  rule = aws_cloudwatch_event_rule.instance_rebalance.name
  arn  = aws_sqs_queue.karpenter_interruption.arn
}

resource "aws_cloudwatch_event_rule" "instance_state_change" {
  name_prefix = "${local.name_prefix}-state-chg-"
  description = "EC2 Instance State-change Notification"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance State-change Notification"]
  })

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "instance_state_change" {
  rule = aws_cloudwatch_event_rule.instance_state_change.name
  arn  = aws_sqs_queue.karpenter_interruption.arn
}

resource "aws_cloudwatch_event_rule" "scheduled_change" {
  name_prefix = "${local.name_prefix}-health-"
  description = "AWS Health Events for EC2"

  event_pattern = jsonencode({
    source      = ["aws.health"]
    detail-type = ["AWS Health Event"]
    detail = {
      service = ["EC2"]
    }
  })

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "scheduled_change" {
  rule = aws_cloudwatch_event_rule.scheduled_change.name
  arn  = aws_sqs_queue.karpenter_interruption.arn
}

# ─────────────────────────────────────────────
# KARPENTER ACCESS ENTRY
# Allows Karpenter-provisioned nodes to join the cluster
# (EKS Access Entry replaces the deprecated aws-auth ConfigMap)
# ─────────────────────────────────────────────

resource "aws_eks_access_entry" "karpenter_node" {
  cluster_name  = aws_eks_cluster.main.name
  principal_arn = aws_iam_role.node.arn
  type          = "EC2_LINUX"

  tags = local.common_tags
}
```

---

### `modules/eks/irsa.tf`

```hcl
# modules/eks/irsa.tf

# ═══════════════════════════════════════════════════════════════════
# IRSA HELPER — Reusable pattern for creating service account roles
#
# The OIDC provider is created in main.tf.
# This file provides:
#   1. AWS Load Balancer Controller IRSA (needed for ALB Ingress)
#   2. External Secrets Operator IRSA (needed for Secrets Manager)
#   3. A generic output for downstream IRSA creation
# ═══════════════════════════════════════════════════════════════════

# ─────────────────────────────────────────────
# AWS LOAD BALANCER CONTROLLER
# ─────────────────────────────────────────────

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
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:kube-system:aws-load-balancer-controller"
          }
        }
      }
    ]
  })

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-aws-lb-controller-irsa"
  })
}

resource "aws_iam_role_policy" "aws_lb_controller" {
  name_prefix = "aws-lb-ctrl-"
  role        = aws_iam_role.aws_lb_controller.id

  # Full ALB controller policy — covers ALB, NLB, TargetGroup creation
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "iam:CreateServiceLinkedRole"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "iam:AWSServiceName" = "elasticloadbalancing.amazonaws.com"
          }
        }
      },
      {
        Effect = "Allow"
        Action = [
          "ec2:DescribeAccountAttributes",
          "ec2:DescribeAddresses",
          "ec2:DescribeAvailabilityZones",
          "ec2:DescribeInternetGateways",
          "ec2:DescribeVpcs",
          "ec2:DescribeVpcPeeringConnections",
          "ec2:DescribeSubnets",
          "ec2:DescribeSecurityGroups",
          "ec2:DescribeInstances",
          "ec2:DescribeNetworkInterfaces",
          "ec2:DescribeTags",
          "ec2:DescribeCoipPools",
          "ec2:GetCoipPoolUsage",
          "ec2:DescribeVpcEndpoints",
          "elasticloadbalancing:DescribeLoadBalancers",
          "elasticloadbalancing:DescribeLoadBalancerAttributes",
          "elasticloadbalancing:DescribeListeners",
          "elasticloadbalancing:DescribeListenerCertificates",
          "elasticloadbalancing:DescribeSSLPolicies",
          "elasticloadbalancing:DescribeRules",
          "elasticloadbalancing:DescribeTargetGroups",
          "elasticloadbalancing:DescribeTargetGroupAttributes",
          "elasticloadbalancing:DescribeTargetHealth",
          "elasticloadbalancing:DescribeTags",
          "elasticloadbalancing:DescribeTrustStores"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "cognito-idp:DescribeUserPoolClient",
          "acm:ListCertificates",
          "acm:DescribeCertificate",
          "iam:ListServerCertificates",
          "iam:GetServerCertificate",
          "waf-regional:GetWebACL",
          "waf-regional:GetWebACLForResource",
          "waf-regional:AssociateWebACL",
          "waf-regional:DisassociateWebACL",
          "wafv2:GetWebACL",
          "wafv2:GetWebACLForResource",
          "wafv2:AssociateWebACL",
          "wafv2:DisassociateWebACL",
          "shield:GetSubscriptionState",
          "shield:DescribeProtection",
          "shield:CreateProtection",
          "shield:DeleteProtection"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ec2:AuthorizeSecurityGroupIngress",
          "ec2:RevokeSecurityGroupIngress",
          "ec2:CreateSecurityGroup",
          "ec2:CreateTags",
          "ec2:DeleteTags",
          "ec2:DeleteSecurityGroup"
        ]
        Resource = "*"
        Condition = {
          "Null" = {
            "aws:ResourceTag/kubernetes.io/cluster/${var.cluster_name}" = "false"
          }
        }
      },
      {
        Effect = "Allow"
        Action = [
          "elasticloadbalancing:CreateLoadBalancer",
          "elasticloadbalancing:CreateTargetGroup"
        ]
        Resource = "*"
        Condition = {
          "Null" = {
            "aws:RequestTag/elbv2.k8s.aws/cluster" = "false"
          }
        }
      },
      {
        Effect = "Allow"
        Action = [
          "elasticloadbalancing:AddTags",
          "elasticloadbalancing:RemoveTags",
          "elasticloadbalancing:DeleteLoadBalancer",
          "elasticloadbalancing:DeleteTargetGroup",
          "elasticloadbalancing:ModifyLoadBalancerAttributes",
          "elasticloadbalancing:ModifyTargetGroupAttributes",
          "elasticloadbalancing:SetIpAddressType",
          "elasticloadbalancing:SetSecurityGroups",
          "elasticloadbalancing:SetSubnets",
          "elasticloadbalancing:CreateListener",
          "elasticloadbalancing:DeleteListener",
          "elasticloadbalancing:ModifyListener",
          "elasticloadbalancing:CreateRule",
          "elasticloadbalancing:DeleteRule",
          "elasticloadbalancing:ModifyRule",
          "elasticloadbalancing:RegisterTargets",
          "elasticloadbalancing:DeregisterTargets"
        ]
        Resource = "*"
        Condition = {
          "Null" = {
            "aws:ResourceTag/elbv2.k8s.aws/cluster" = "false"
          }
        }
      }
    ]
  })
}

# ─────────────────────────────────────────────
# EXTERNAL SECRETS OPERATOR
# ─────────────────────────────────────────────

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
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
            "${replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:external-secrets:external-secrets"
          }
        }
      }
    ]
  })

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-external-secrets-irsa"
  })
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
        # Scoped to NovaMart secrets only
        Resource = "arn:aws:secretsmanager:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:secret:novamart/*"
      },
      {
        Effect = "Allow"
        Action = [
          "ssm:GetParameter",
          "ssm:GetParameters",
          "ssm:GetParametersByPath"
        ]
        Resource = "arn:aws:ssm:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:parameter/novamart/*"
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey"
        ]
        Resource = var.eks_secrets_kms_key_arn
      }
    ]
  })
}
```

---

### `modules/eks/variables.tf` (Additions for Add-on Versions)

```hcl
# Append to existing modules/eks/variables.tf

variable "vpc_cni_addon_version" {
  description = "VPC CNI add-on version"
  type        = string
  default     = "v1.18.1-eksbuild.1"
}

variable "coredns_addon_version" {
  description = "CoreDNS add-on version"
  type        = string
  default     = "v1.11.1-eksbuild.8"
}

variable "kube_proxy_addon_version" {
  description = "kube-proxy add-on version"
  type        = string
  default     = "v1.29.1-eksbuild.2"
}

variable "ebs_csi_addon_version" {
  description = "EBS CSI Driver add-on version"
  type        = string
  default     = "v1.28.0-eksbuild.1"
}
```

### `modules/eks/outputs.tf` (Complete)

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
  description = "EKS cluster CA certificate (base64 encoded)"
  value       = aws_eks_cluster.main.certificate_authority[0].data
}

output "cluster_version" {
  description = "EKS cluster Kubernetes version"
  value       = aws_eks_cluster.main.version
}

output "cluster_arn" {
  description = "EKS cluster ARN"
  value       = aws_eks_cluster.main.arn
}

output "cluster_security_group_id" {
  description = "EKS cluster (control plane) security group ID"
  value       = aws_security_group.cluster.id
}

output "node_security_group_id" {
  description = "EKS worker node security group ID"
  value       = aws_security_group.node.id
}

output "node_iam_role_arn" {
  description = "IAM role ARN for EKS worker nodes"
  value       = aws_iam_role.node.arn
}

output "node_iam_role_name" {
  description = "IAM role name for EKS worker nodes"
  value       = aws_iam_role.node.name
}

output "oidc_provider_arn" {
  description = "OIDC provider ARN for IRSA"
  value       = aws_iam_openid_connect_provider.cluster.arn
}

output "oidc_provider_url" {
  description = "OIDC provider URL (without https://)"
  value       = replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")
}

output "karpenter_controller_role_arn" {
  description = "Karpenter controller IRSA role ARN"
  value       = aws_iam_role.karpenter_controller.arn
}

output "karpenter_instance_profile_name" {
  description = "Instance profile name for Karpenter-provisioned nodes"
  value       = aws_iam_instance_profile.karpenter_node.name
}

output "karpenter_interruption_queue_name" {
  description = "SQS queue name for Karpenter spot interruption handling"
  value       = aws_sqs_queue.karpenter_interruption.name
}

output "aws_lb_controller_role_arn" {
  description = "AWS Load Balancer Controller IRSA role ARN"
  value       = aws_iam_role.aws_lb_controller.arn
}

output "external_secrets_role_arn" {
  description = "External Secrets Operator IRSA role ARN"
  value       = aws_iam_role.external_secrets.arn
}

output "system_node_group_name" {
  description = "System node group name"
  value       = aws_eks_node_group.system.node_group_name
}

output "cluster_primary_security_group_id" {
  description = "EKS-managed primary cluster security group (auto-created by EKS)"
  value       = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
}
```

---

## Environment Compositions

### `environments/production/versions.tf`

```hcl
# environments/production/versions.tf
# Shared version constraints — symlinked or copied into each subdirectory

terraform {
  required_version = ">= 1.7.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}
```

### `environments/production/provider.tf`

```hcl
# environments/production/provider.tf
# Shared provider config — symlinked or copied into each subdirectory

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

  # Safety: prevent accidental operations on wrong account
  # Uncomment and set when account ID is known
  # allowed_account_ids = ["888888888888"]
}
```

---

### `environments/production/security/`

```hcl
# environments/production/security/backend.tf

terraform {
  backend "s3" {
    bucket         = "novamart-terraform-state-888888888888"
    key            = "novamart/production/security/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "novamart-terraform-locks"
    encrypt        = true
  }
}
```

```hcl
# environments/production/security/main.tf

module "security_baseline" {
  source = "../../../modules/security-baseline"

  environment    = "production"
  project        = "novamart"
  aws_account_id = data.aws_caller_identity.current.account_id
  aws_region     = data.aws_region.current.name

  # VPC ID injected after networking is created
  # For initial bootstrap, use a placeholder and update after networking apply
  vpc_id = var.vpc_id

  terraform_state_bucket_name = "novamart-tfstate-production-${data.aws_caller_identity.current.account_id}"

  cloudtrail_log_retention_days = 90
  flow_log_retention_days       = 90

  # Data event logging for sensitive buckets
  cloudtrail_s3_data_event_buckets = [
    "arn:aws:s3:::novamart-tfstate-production-${data.aws_caller_identity.current.account_id}",
  ]

  tags = {
    SecurityZone = "management"
  }
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
```

```hcl
# environments/production/security/variables.tf

variable "vpc_id" {
  description = "VPC ID — set after networking module is applied"
  type        = string
  default     = ""  # Empty on first run; populated via tfvars after networking
}
```

```hcl
# environments/production/security/outputs.tf

output "kms_key_arns" {
  description = "KMS key ARNs for use by other modules"
  value       = module.security_baseline.kms_key_arns
}

output "kms_key_ids" {
  description = "KMS key IDs"
  value       = module.security_baseline.kms_key_ids
}

output "s3_bucket_arns" {
  description = "S3 bucket ARNs"
  value       = module.security_baseline.s3_bucket_arns
}

output "s3_bucket_ids" {
  description = "S3 bucket names"
  value       = module.security_baseline.s3_bucket_ids
}

output "terraform_lock_table_name" {
  description = "DynamoDB lock table"
  value       = module.security_baseline.terraform_lock_table_name
}

output "cloudtrail_arn" {
  value = module.security_baseline.cloudtrail_arn
}
```

```hcl
# environments/production/security/terraform.tfvars

# First apply: leave vpc_id empty (CloudTrail + KMS don't need it)
# Second apply (after networking): set the vpc_id for flow logs
# vpc_id = "vpc-0123456789abcdef0"
```

---

### `environments/production/networking/`

```hcl
# environments/production/networking/backend.tf

terraform {
  backend "s3" {
    bucket         = "novamart-terraform-state-888888888888"
    key            = "novamart/production/networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "novamart-terraform-locks"
    encrypt        = true
  }
}
```

```hcl
# environments/production/networking/data.tf

# Pull outputs from security module
data "terraform_remote_state" "security" {
  backend = "s3"
  config = {
    bucket = "novamart-terraform-state-888888888888"
    key    = "novamart/production/security/terraform.tfstate"
    region = "us-east-1"
  }
}
```

```hcl
# environments/production/networking/main.tf

module "vpc" {
  source = "../../../modules/vpc"

  environment = "production"
  cluster_name = "novamart-production"

  primary_cidr   = "10.0.0.0/16"
  secondary_cidr = "100.64.0.0/16"

  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

  public_subnet_cidrs  = ["10.0.0.0/22", "10.0.4.0/22", "10.0.8.0/22"]
  private_subnet_cidrs = ["10.0.16.0/20", "10.0.32.0/20", "10.0.48.0/20"]
  pod_subnet_cidrs     = ["100.64.0.0/18", "100.64.64.0/18", "100.64.128.0/18"]
  data_subnet_cidrs    = ["10.0.64.0/22", "10.0.68.0/22", "10.0.72.0/22"]

  enable_vpc_endpoints = true

  flow_log_retention_days = 90
  flow_log_s3_bucket_arn  = data.terraform_remote_state.security.outputs.s3_bucket_arns["flow_logs"]

  tags = {
    NetworkTier = "production"
  }
}
```

```hcl
# environments/production/networking/outputs.tf

output "vpc_id" {
  value = module.vpc.vpc_id
}

output "vpc_cidr" {
  value = module.vpc.vpc_cidr
}

output "public_subnet_ids" {
  value = module.vpc.public_subnet_ids
}

output "private_subnet_ids" {
  value = module.vpc.private_subnet_ids
}

output "pod_subnet_ids" {
  value = module.vpc.pod_subnet_ids
}

output "data_subnet_ids" {
  value = module.vpc.data_subnet_ids
}

output "db_subnet_group_name" {
  value = module.vpc.db_subnet_group_name
}

output "elasticache_subnet_group_name" {
  value = module.vpc.elasticache_subnet_group_name
}

output "nat_gateway_ips" {
  value = module.vpc.nat_gateway_ips
}

output "availability_zones" {
  value = module.vpc.availability_zones
}
```

---

### `environments/production/eks/`

```hcl
# environments/production/eks/backend.tf

terraform {
  backend "s3" {
    bucket         = "novamart-terraform-state-888888888888"
    key            = "novamart/production/eks/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "novamart-terraform-locks"
    encrypt        = true
  }
}
```

```hcl
# environments/production/eks/data.tf

data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "novamart-terraform-state-888888888888"
    key    = "novamart/production/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

data "terraform_remote_state" "security" {
  backend = "s3"
  config = {
    bucket = "novamart-terraform-state-888888888888"
    key    = "novamart/production/security/terraform.tfstate"
    region = "us-east-1"
  }
}
```

```hcl
# environments/production/eks/main.tf

module "eks" {
  source = "../../../modules/eks"

  environment  = "production"
  cluster_name = "novamart-production"
  cluster_version = "1.29"

  vpc_id             = data.terraform_remote_state.networking.outputs.vpc_id
  private_subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
  pod_subnet_ids     = data.terraform_remote_state.networking.outputs.pod_subnet_ids

  eks_secrets_kms_key_arn = data.terraform_remote_state.security.outputs.kms_key_arns["eks"]

  # Public API endpoint restricted to corporate + CI/CD IPs
  api_public_access_cidrs = var.api_public_access_cidrs

  # System node group sizing
  system_node_instance_types = ["m5.large"]
  system_node_desired        = 2
  system_node_min            = 2
  system_node_max            = 4

  # Add-on versions — pinned for production stability
  vpc_cni_addon_version    = "v1.18.1-eksbuild.1"
  coredns_addon_version    = "v1.11.1-eksbuild.8"
  kube_proxy_addon_version = "v1.29.1-eksbuild.2"
  ebs_csi_addon_version    = "v1.28.0-eksbuild.1"

  tags = {
    ClusterTier = "production"
  }
}
```

```hcl
# environments/production/eks/variables.tf

variable "api_public_access_cidrs" {
  description = "CIDR blocks allowed to reach EKS public API endpoint"
  type        = list(string)
  # Corporate VPN + CI/CD runners
  # NEVER use 0.0.0.0/0 in production
}
```

```hcl
# environments/production/eks/terraform.tfvars

api_public_access_cidrs = [
  "203.0.113.0/24",   # NovaMart corporate VPN egress
  "198.51.100.0/28",  # GitHub Actions self-hosted runner NAT
]
```

```hcl
# environments/production/eks/outputs.tf

output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_endpoint" {
  value     = module.eks.cluster_endpoint
  sensitive = true
}

output "cluster_certificate_authority" {
  value     = module.eks.cluster_certificate_authority
  sensitive = true
}

output "cluster_version" {
  value = module.eks.cluster_version
}

output "node_security_group_id" {
  value = module.eks.node_security_group_id
}

output "cluster_security_group_id" {
  value = module.eks.cluster_security_group_id
}

output "oidc_provider_arn" {
  value = module.eks.oidc_provider_arn
}

output "oidc_provider_url" {
  value = module.eks.oidc_provider_url
}

output "karpenter_controller_role_arn" {
  value = module.eks.karpenter_controller_role_arn
}

output "karpenter_instance_profile_name" {
  value = module.eks.karpenter_instance_profile_name
}

output "karpenter_interruption_queue_name" {
  value = module.eks.karpenter_interruption_queue_name
}

output "aws_lb_controller_role_arn" {
  value = module.eks.aws_lb_controller_role_arn
}

output "external_secrets_role_arn" {
  value = module.eks.external_secrets_role_arn
}

output "node_iam_role_arn" {
  value = module.eks.node_iam_role_arn
}
```

---

### `environments/production/data/`

```hcl
# environments/production/data/backend.tf

terraform {
  backend "s3" {
    bucket         = "novamart-terraform-state-888888888888"
    key            = "novamart/production/data/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "novamart-terraform-locks"
    encrypt        = true
  }
}
```

```hcl
# environments/production/data/data.tf

data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "novamart-terraform-state-888888888888"
    key    = "novamart/production/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

data "terraform_remote_state" "eks" {
  backend = "s3"
  config = {
    bucket = "novamart-terraform-state-888888888888"
    key    = "novamart/production/eks/terraform.tfstate"
    region = "us-east-1"
  }
}

data "terraform_remote_state" "security" {
  backend = "s3"
  config = {
    bucket = "novamart-terraform-state-888888888888"
    key    = "novamart/production/security/terraform.tfstate"
    region = "us-east-1"
  }
}
```

```hcl
# environments/production/data/main.tf

module "data_tier" {
  source = "../../../modules/data-tier"

  environment = "production"
  project     = "novamart"

  vpc_id          = data.terraform_remote_state.networking.outputs.vpc_id
  data_subnet_ids = data.terraform_remote_state.networking.outputs.data_subnet_ids

  eks_node_security_group_id = data.terraform_remote_state.eks.outputs.node_security_group_id

  # RDS configuration — separate instance per service
  rds_instances = {
    payments = {
      engine_version          = "15.5"
      instance_class          = "db.r6g.xlarge"  # Payments gets more power
      allocated_storage       = 100
      max_allocated_storage   = 500
      database_name           = "payments"
      multi_az                = true
      backup_retention_period = 30  # Maximum for regulatory
      deletion_protection     = true
    }
    orders = {
      engine_version          = "15.5"
      instance_class          = "db.r6g.large"
      allocated_storage       = 100
      max_allocated_storage   = 500
      database_name           = "orders"
      multi_az                = true
      backup_retention_period = 30
      deletion_protection     = true
    }
    users = {
      engine_version          = "15.5"
      instance_class          = "db.r6g.large"
      allocated_storage       = 50
      max_allocated_storage   = 200
      database_name           = "users"
      multi_az                = true
      backup_retention_period = 30
      deletion_protection     = true
    }
  }

  rds_kms_key_arn                  = data.terraform_remote_state.security.outputs.kms_key_arns["rds"]
  rds_master_password_management   = true  # AWS manages password in Secrets Manager
  rds_performance_insights_enabled = true
  rds_monitoring_interval          = 60    # 60-second Enhanced Monitoring

  # Redis configuration
  redis_node_type            = "cache.r6g.large"
  redis_num_cache_clusters   = 3  # 1 primary + 2 replicas across 3 AZs
  redis_engine_version       = "7.1"
  redis_kms_key_arn          = data.terraform_remote_state.security.outputs.kms_key_arns["elasticache"]
  redis_snapshot_retention_limit = 7
  redis_maintenance_window   = "sun:05:00-sun:07:00"
  redis_snapshot_window      = "03:00-05:00"

  tags = {
    DataTier = "production"
  }
}
```

```hcl
# environments/production/data/outputs.tf

output "rds_endpoints" {
  description = "RDS endpoints by service"
  value       = module.data_tier.rds_endpoints
  sensitive   = true
}

output "rds_master_secret_arns" {
  description = "Secrets Manager ARNs for RDS passwords"
  value       = module.data_tier.rds_master_secret_arns
}

output "rds_security_group_id" {
  value = module.data_tier.rds_security_group_id
}

output "redis_primary_endpoint" {
  value     = module.data_tier.redis_primary_endpoint
  sensitive = true
}

output "redis_reader_endpoint" {
  value     = module.data_tier.redis_reader_endpoint
  sensitive = true
}

output "redis_port" {
  value = module.data_tier.redis_port
}

output "redis_security_group_id" {
  value = module.data_tier.redis_security_group_id
}
```

```hcl
# environments/production/data/terraform.tfvars

# No overrides needed — defaults in main.tf are production-tuned
# This file exists for consistency and future per-environment tuning
```

---

## Apply Order & Bootstrap Runbook

```hcl
# docs/runbook-terraform-operations.md (referenced in directory tree)

# ═══════════════════════════════════════════════════════════════════
# NOVAMART INFRASTRUCTURE — APPLY ORDER
# ═══════════════════════════════════════════════════════════════════
#
# DEPENDENCY GRAPH:
#
#   ┌─────────────┐
#   │  bootstrap   │  ← Run once manually (creates S3 + DynamoDB)
#   └──────┬──────┘
#          │
#   ┌──────▼──────┐
#   │  security    │  ← KMS keys, S3 buckets, CloudTrail
#   └──────┬──────┘    (no VPC dependency for first apply)
#          │
#   ┌──────▼──────┐
#   │ networking   │  ← VPC, subnets, NAT GW, endpoints, flow logs
#   └──────┬──────┘
#          │
#   ┌──────▼──────┐    ← Re-apply security with vpc_id for flow logs
#   │  security    │
#   │  (2nd pass)  │
#   └──────┬──────┘
#          │
#   ┌──────▼──────┐
#   │     eks      │  ← Cluster, node groups, add-ons, Karpenter IAM
#   └──────┬──────┘
#          │
#   ┌──────▼──────┐
#   │    data      │  ← RDS instances, ElastiCache Redis
#   └─────────────┘
#
# TOTAL ESTIMATED TIME: ~45-60 minutes
#   bootstrap:   2 min
#   security:    3 min
#   networking: 10 min  (NAT GWs are slow)
#   security:    1 min  (2nd pass — just adds flow logs)
#   eks:        15 min  (cluster creation ~10min, node group ~5min)
#   data:       20 min  (RDS Multi-AZ instances are slow)

# ═══════════════════════════════════════════════════════════════════
# STEP 0: PREREQUISITES
# ═══════════════════════════════════════════════════════════════════

# Required tools:
#   - Terraform >= 1.7.0
#   - AWS CLI v2
#   - aws-iam-authenticator (for EKS kubectl access)
#   - kubectl >= 1.29
#   - helm >= 3.14

# Verify AWS identity
aws sts get-caller-identity
# Expected: correct account (888888888888) and role (OrganizationAdmin or PlatformAdmin)

# Verify Terraform version
terraform version
# Expected: >= 1.7.0

# ═══════════════════════════════════════════════════════════════════
# STEP 1: BOOTSTRAP (Run ONCE — never again)
# ═══════════════════════════════════════════════════════════════════

cd bootstrap/
terraform init
terraform plan -out=bootstrap.plan
# REVIEW the plan carefully — this creates the state backend
terraform apply bootstrap.plan

# Capture outputs
export STATE_BUCKET=$(terraform output -raw state_bucket)
export LOCK_TABLE=$(terraform output -raw locks_table)
echo "State bucket: ${STATE_BUCKET}"
echo "Lock table:   ${LOCK_TABLE}"

# IMPORTANT: After bootstrap, migrate bootstrap's OWN state to S3
# (Optional but recommended — prevents local state file loss)
# Add backend.tf to bootstrap/ and run terraform init -migrate-state

# ═══════════════════════════════════════════════════════════════════
# STEP 2: SECURITY BASELINE (First Pass — no VPC yet)
# ═══════════════════════════════════════════════════════════════════

cd ../environments/production/security/

# Ensure terraform.tfvars has vpc_id = "" for first pass
# (CloudTrail, KMS, S3 don't require VPC)

terraform init \
  -backend-config="bucket=${STATE_BUCKET}" \
  -backend-config="dynamodb_table=${LOCK_TABLE}"

terraform plan -out=security.plan
# REVIEW: Expect ~20 resources (4 KMS keys, 4 S3 buckets, CloudTrail, IAM roles)
terraform apply security.plan

# Verify KMS keys exist
aws kms list-aliases --query 'Aliases[?starts_with(AliasName, `alias/novamart`)]' \
  --output table

# ═══════════════════════════════════════════════════════════════════
# STEP 3: NETWORKING
# ═══════════════════════════════════════════════════════════════════

cd ../networking/

terraform init \
  -backend-config="bucket=${STATE_BUCKET}" \
  -backend-config="dynamodb_table=${LOCK_TABLE}"

terraform plan -out=networking.plan
# REVIEW: Expect ~50 resources
#   - 1 VPC + 1 secondary CIDR
#   - 12 subnets (3 public + 3 private + 3 pod + 3 data)
#   - 3 NAT Gateways + 3 EIPs
#   - 6 route tables + associations
#   - 1 internet gateway
#   - 7 VPC endpoints (1 gateway + 6 interface)
#   - 2 flow logs (CloudWatch + S3)
#   - Subnet groups (RDS + ElastiCache)

terraform apply networking.plan
# ⏱ ~10 minutes (NAT Gateways take 2-3 min each)

# Verify VPC
export VPC_ID=$(terraform output -raw vpc_id)
echo "VPC ID: ${VPC_ID}"
aws ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'Subnets[].[SubnetId,AvailabilityZone,CidrBlock,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# ═══════════════════════════════════════════════════════════════════
# STEP 4: SECURITY BASELINE (Second Pass — add VPC Flow Logs)
# ═══════════════════════════════════════════════════════════════════

cd ../security/

# Update terraform.tfvars with the VPC ID
echo "vpc_id = \"${VPC_ID}\"" > terraform.tfvars

terraform plan -out=security-v2.plan
# REVIEW: Expect 1-2 new resources (VPC flow log)
terraform apply security-v2.plan

# ═══════════════════════════════════════════════════════════════════
# STEP 5: EKS CLUSTER
# ═══════════════════════════════════════════════════════════════════

cd ../eks/

terraform init \
  -backend-config="bucket=${STATE_BUCKET}" \
  -backend-config="dynamodb_table=${LOCK_TABLE}"

terraform plan -out=eks.plan
# REVIEW: Expect ~30 resources
#   - EKS cluster + IAM role
#   - OIDC provider
#   - System node group + launch template
#   - Security groups (cluster + node)
#   - 4 EKS add-ons (VPC CNI, CoreDNS, kube-proxy, EBS CSI)
#   - Karpenter IAM (role, policy, instance profile, SQS, EventBridge rules)
#   - IRSA roles (LB controller, External Secrets)
#   - CloudWatch log group

terraform apply eks.plan
# ⏱ ~15 minutes (cluster ~10min, node group ~5min)

# Verify cluster
export CLUSTER_NAME=$(terraform output -raw cluster_name)
aws eks describe-cluster --name ${CLUSTER_NAME} \
  --query 'cluster.[name,status,version,endpoint]' --output table

# Configure kubectl
aws eks update-kubeconfig --name ${CLUSTER_NAME} --region us-east-1 --alias novamart-prod

# Verify nodes
kubectl get nodes -o wide
# Expected: 2 system nodes in Ready state

# Verify add-ons
kubectl get pods -n kube-system
# Expected: aws-node, coredns (3 replicas), kube-proxy, ebs-csi-controller

# ═══════════════════════════════════════════════════════════════════
# STEP 6: CREATE ENI CONFIGS (VPC CNI Custom Networking)
# Must be applied BEFORE Karpenter provisions app nodes
# ═══════════════════════════════════════════════════════════════════

# Get pod subnet IDs and node security group
POD_SUBNETS=$(cd ../networking && terraform output -json pod_subnet_ids)
NODE_SG=$(terraform output -raw node_security_group_id)

# Apply ENIConfig for each AZ
cat <<EOF | kubectl apply -f -
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1a
spec:
  securityGroups:
    - ${NODE_SG}
  subnet: $(echo $POD_SUBNETS | jq -r '.[0]')
---
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1b
spec:
  securityGroups:
    - ${NODE_SG}
  subnet: $(echo $POD_SUBNETS | jq -r '.[1]')
---
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1c
spec:
  securityGroups:
    - ${NODE_SG}
  subnet: $(echo $POD_SUBNETS | jq -r '.[2]')
EOF

# Verify ENIConfigs
kubectl get eniconfigs
# Expected: 3 ENIConfigs, one per AZ

# ═══════════════════════════════════════════════════════════════════
# STEP 7: INSTALL KARPENTER (Helm — after EKS cluster is ready)
# ═══════════════════════════════════════════════════════════════════

# Karpenter Helm chart (controller runs on system nodes)
KARPENTER_VERSION="v0.35.0"

helm registry logout public.ecr.aws || true

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version ${KARPENTER_VERSION} \
  --namespace kube-system \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=$(terraform output -raw karpenter_interruption_queue_name)" \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=$(terraform output -raw karpenter_controller_role_arn)" \
  --set "controller.resources.requests.cpu=250m" \
  --set "controller.resources.requests.memory=256Mi" \
  --set "controller.resources.limits.cpu=1" \
  --set "controller.resources.limits.memory=1Gi" \
  --set "replicas=2" \
  --set "tolerations[0].key=CriticalAddonsOnly" \
  --set "tolerations[0].operator=Equal" \
  --set "tolerations[0].value=true" \
  --set "tolerations[0].effect=NoSchedule" \
  --set "affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution[0].topologyKey=kubernetes.io/hostname" \
  --set "topologySpreadConstraints[0].maxSkew=1" \
  --set "topologySpreadConstraints[0].topologyKey=topology.kubernetes.io/zone" \
  --set "topologySpreadConstraints[0].whenUnsatisfiable=DoNotSchedule" \
  --wait --timeout 5m

# Verify Karpenter
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
# Expected: 2 karpenter pods Running

# ═══════════════════════════════════════════════════════════════════
# STEP 8: APPLY KARPENTER NODE POOL + EC2 NODE CLASS
# ═══════════════════════════════════════════════════════════════════

INSTANCE_PROFILE=$(terraform output -raw karpenter_instance_profile_name)

cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: general-workloads
spec:
  template:
    metadata:
      labels:
        novamart.com/node-type: application
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]  # Start with on-demand; add spot for non-critical later
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["m", "r", "c"]  # General, memory-optimized, compute
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["4"]  # 5th gen+ only (m5, r6g, c6i, etc.)
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["large", "xlarge", "2xlarge", "4xlarge"]
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["us-east-1a", "us-east-1b", "us-east-1c"]
      nodeClassRef:
        name: default
      # Kubelet configuration
      kubelet:
        maxPods: 110
        systemReserved:
          cpu: "100m"
          memory: "256Mi"
          ephemeral-storage: "1Gi"
        kubeReserved:
          cpu: "200m"
          memory: "512Mi"
          ephemeral-storage: "1Gi"
  # Consolidation: Karpenter replaces underutilized nodes
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h  # 30 days — force node refresh for AMI updates
  # Limits: prevent runaway scaling
  limits:
    cpu: "2000"       # Max 2000 vCPUs across all Karpenter nodes
    memory: "4000Gi"  # Max 4TB RAM
  # Weight: prefer this pool (higher = more preferred)
  weight: 50
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: "${INSTANCE_PROFILE}"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
  securityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/${CLUSTER_NAME}: owned
  # EBS root volume
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        iops: 3000
        throughput: 125
        encrypted: true
        deleteOnTermination: true
  # IMDSv2 enforcement
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 1
    httpTokens: required
  # Tags on launched instances
  tags:
    Environment: production
    ManagedBy: karpenter
    Team: platform
    Project: novamart
    CostCenter: PLATFORM-001
EOF

# Verify NodePool
kubectl get nodepools
kubectl get ec2nodeclasses

# ═══════════════════════════════════════════════════════════════════
# STEP 9: DATA TIER
# ═══════════════════════════════════════════════════════════════════

cd ../data/

terraform init \
  -backend-config="bucket=${STATE_BUCKET}" \
  -backend-config="dynamodb_table=${LOCK_TABLE}"

terraform plan -out=data.plan
# REVIEW: Expect ~15 resources
#   - 3 RDS instances (payments, orders, users) — Multi-AZ
#   - 1 RDS subnet group + 1 security group + 1 parameter group
#   - 1 Enhanced Monitoring IAM role
#   - 1 Redis replication group (3 nodes)
#   - 1 Redis subnet group + 1 security group + 1 parameter group
#   - 2 CloudWatch log groups (Redis slow-log, engine-log)

terraform apply data.plan
# ⏱ ~20 minutes (RDS Multi-AZ instances are slow to provision)

# Verify RDS
aws rds describe-db-instances \
  --query 'DBInstances[?starts_with(DBInstanceIdentifier, `novamart`)].[DBInstanceIdentifier,DBInstanceStatus,MultiAZ,Engine,EngineVersion]' \
  --output table

# Verify Redis
aws elasticache describe-replication-groups \
  --query 'ReplicationGroups[?starts_with(ReplicationGroupId, `novamart`)].[ReplicationGroupId,Status,AutomaticFailover,MultiAZ]' \
  --output table

# Verify Secrets Manager has RDS passwords
aws secretsmanager list-secrets \
  --query 'SecretList[?starts_with(Name, `rds!`)].Name' --output table

# ═══════════════════════════════════════════════════════════════════
# STEP 10: POST-DEPLOY VALIDATION
# ═══════════════════════════════════════════════════════════════════

echo "========================================="
echo "  POST-DEPLOYMENT VALIDATION CHECKLIST"
echo "========================================="

echo "--- VPC ---"
aws ec2 describe-vpcs --vpc-ids ${VPC_ID} --query 'Vpcs[0].State' --output text
# Expected: available

echo "--- Subnets ---"
aws ec2 describe-subnets --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'length(Subnets)' --output text
# Expected: 12

echo "--- NAT Gateways ---"
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=${VPC_ID}" \
  --query 'NatGateways[].State' --output text
# Expected: available available available

echo "--- VPC Endpoints ---"
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'VpcEndpoints[].[ServiceName,State]' --output table
# Expected: 7 endpoints, all available

echo "--- EKS Cluster ---"
aws eks describe-cluster --name ${CLUSTER_NAME} --query 'cluster.status' --output text
# Expected: ACTIVE

echo "--- EKS Nodes ---"
kubectl get nodes
# Expected: 2 system nodes, Ready

echo "--- System Pods ---"
kubectl get pods -n kube-system --field-selector=status.phase!=Running 2>/dev/null
# Expected: No results (all running)

echo "--- Karpenter ---"
kubectl get nodepools
# Expected: general-workloads

echo "--- RDS ---"
aws rds describe-db-instances \
  --query 'DBInstances[?starts_with(DBInstanceIdentifier,`novamart-production`)].DBInstanceStatus' \
  --output text
# Expected: available available available

echo "--- Redis ---"
aws elasticache describe-replication-groups \
  --replication-group-id novamart-production-redis \
  --query 'ReplicationGroups[0].Status' --output text
# Expected: available

echo "--- CloudTrail ---"
aws cloudtrail get-trail-status --name novamart-production \
  --query '[IsLogging,LatestDeliveryTime]' --output text
# Expected: True <recent_timestamp>

echo "--- KMS Keys ---"
aws kms list-aliases --query 'Aliases[?starts_with(AliasName,`alias/novamart-production`)].AliasName' \
  --output text
# Expected: 4 aliases

echo "--- Flow Logs ---"
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=${VPC_ID}" \
  --query 'FlowLogs[].[FlowLogId,FlowLogStatus,LogDestinationType]' --output table
# Expected: 2 flow logs (cloudwatch-logs + s3), both ACTIVE

echo "========================================="
echo "  VALIDATION COMPLETE"
echo "========================================="

# ═══════════════════════════════════════════════════════════════════
# ROLLBACK PROCEDURES
# ═══════════════════════════════════════════════════════════════════
#
# CRITICAL: Rollback order is REVERSE of apply order
#
#   1. data      → terraform destroy (RDS has deletion_protection — disable first)
#   2. eks       → terraform destroy (node groups drain, cluster deletes)
#   3. networking → terraform destroy (VPC, subnets, NAT GWs)
#   4. security  → terraform destroy (KMS keys enter 30-day deletion window)
#
# WARNINGS:
#   - RDS has deletion_protection = true. You MUST disable it first:
#     aws rds modify-db-instance --db-instance-identifier <id> --no-deletion-protection
#   - RDS will create a final snapshot (configured in Terraform)
#   - KMS keys have 30-day deletion window — recoverable
#   - S3 buckets have prevent_destroy — must remove lifecycle rule first
#   - CloudTrail Object Lock prevents deletion for 365 days
#
# NEVER run terraform destroy in production without:
#   1. Change ticket approved
#   2. Data backup verified
#   3. Stakeholder sign-off
#   4. Runbook reviewed

# ═══════════════════════════════════════════════════════════════════
# COST ESTIMATE (Monthly)
# ═══════════════════════════════════════════════════════════════════
#
# ┌─────────────────────────────────────┬──────────────┐
# │ Component                           │ Est. Monthly │
# ├─────────────────────────────────────┼──────────────┤
# │ NAT Gateways (3x)                   │ $98          │
# │ NAT Gateway data processing*        │ $200-800     │
# │ VPC Endpoints (6 interface, 3 AZ)   │ $132         │
# │ EKS Control Plane                   │ $73          │
# │ System Node Group (2x m5.large)     │ $140         │
# │ Karpenter Nodes (variable)**        │ $2,000-8,000 │
# │ RDS payments (db.r6g.xlarge M-AZ)   │ $550         │
# │ RDS orders (db.r6g.large M-AZ)      │ $370         │
# │ RDS users (db.r6g.large M-AZ)       │ $370         │
# │ RDS storage (250 GB total gp3)      │ $30          │
# │ RDS Performance Insights            │ Free (7-day) │
# │ ElastiCache (3x cache.r6g.large)    │ $550         │
# │ CloudWatch Logs                     │ $50-100      │
# │ S3 (state, trails, flow logs)       │ $10-30       │
# │ KMS Keys (4x)                       │ $4           │
# │ DynamoDB (state locking)            │ <$1          │
# ├─────────────────────────────────────┼──────────────┤
# │ BASELINE (before app workloads)     │ ~$2,400      │
# │ WITH APP WORKLOADS (normal)         │ ~$4,500      │
# │ PEAK (Black Friday)                 │ ~$10,000     │
# └─────────────────────────────────────┴──────────────┘
#
# * NAT data processing highly variable — VPC endpoints offset most
# ** Karpenter nodes scale to zero when no workloads scheduled
#
# COST OPTIMIZATION LEVERS:
#   1. Spot instances for non-critical workloads (Karpenter NodePool)
#   2. Reserved Instances for RDS (1yr RI saves ~40%)
#   3. Savings Plans for EC2 (1yr compute SP saves ~30%)
#   4. GP3 storage over GP2 (already using GP3)
#   5. VPC endpoints avoid NAT data processing charges
#   6. S3 lifecycle policies move old data to Glacier
#   7. CloudWatch log retention limits (90 days hot, then S3)
```

---

## CIDR Allocation Reference

```markdown
# docs/cidr-allocation.md

# ═══════════════════════════════════════════════════════════════════
# NOVAMART CIDR ALLOCATION MAP
# ═══════════════════════════════════════════════════════════════════
#
# PRIMARY CIDR: 10.0.0.0/16 (65,536 IPs)
# ─────────────────────────────────────────────────────────────────
#
# ┌─────────────────┬──────────────────┬─────────┬──────────────┐
# │ Subnet Tier     │ CIDR             │ Size    │ Usable IPs   │
# ├─────────────────┼──────────────────┼─────────┼──────────────┤
# │ Public AZ-a     │ 10.0.0.0/22      │ /22     │ 1,019        │
# │ Public AZ-b     │ 10.0.4.0/22      │ /22     │ 1,019        │
# │ Public AZ-c     │ 10.0.8.0/22      │ /22     │ 1,019        │
# │ (reserved)      │ 10.0.12.0/22     │ /22     │ reserved     │
# ├─────────────────┼──────────────────┼─────────┼──────────────┤
# │ Private AZ-a    │ 10.0.16.0/20     │ /20     │ 4,091        │
# │ Private AZ-b    │ 10.0.32.0/20     │ /20     │ 4,091        │
# │ Private AZ-c    │ 10.0.48.0/20     │ /20     │ 4,091        │
# ├─────────────────┼──────────────────┼─────────┼──────────────┤
# │ Data AZ-a       │ 10.0.64.0/22     │ /22     │ 1,019        │
# │ Data AZ-b       │ 10.0.68.0/22     │ /22     │ 1,019        │
# │ Data AZ-c       │ 10.0.72.0/22     │ /22     │ 1,019        │
# │ (reserved)      │ 10.0.76.0/22     │ /22     │ reserved     │
# ├─────────────────┼──────────────────┼─────────┼──────────────┤
# │ UNALLOCATED     │ 10.0.80.0/20     │         │ future use   │
# │                 │ 10.0.96.0/19     │         │              │
# │                 │ 10.0.128.0/17    │         │              │
# └─────────────────┴──────────────────┴─────────┴──────────────┘
#
# Allocated:  ~28,500 IPs (43% of /16)
# Reserved:   ~2,000 IPs (for future subnet tiers)
# Unallocated: ~35,000 IPs (for future growth)
#
# SECONDARY CIDR: 100.64.0.0/16 (65,536 IPs) — Pod IPs Only
# ─────────────────────────────────────────────────────────────────
#
# ┌─────────────────┬──────────────────┬─────────┬──────────────┐
# │ Subnet Tier     │ CIDR             │ Size    │ Usable IPs   │
# ├─────────────────┼──────────────────┼─────────┼──────────────┤
# │ Pods AZ-a       │ 100.64.0.0/18    │ /18     │ 16,379       │
# │ Pods AZ-b       │ 100.64.64.0/18   │ /18     │ 16,379       │
# │ Pods AZ-c       │ 100.64.128.0/18  │ /18     │ 16,379       │
# │ (reserved)      │ 100.64.192.0/18  │ /18     │ reserved     │
# └─────────────────┴──────────────────┴─────────┴──────────────┘
#
# Total pod capacity: 49,137 IPs across 3 AZs
# Peak requirement:   5,000 pods (Black Friday)
# Headroom:           9.8x — sufficient for multi-year growth
#
# ENVIRONMENT STRATEGY (Future):
# ─────────────────────────────────────────────────────────────────
#
# │ Environment │ Primary CIDR    │ Secondary CIDR    │
# │ production  │ 10.0.0.0/16     │ 100.64.0.0/16     │
# │ staging     │ 10.1.0.0/16     │ 100.65.0.0/16     │
# │ dev         │ 10.2.0.0/16     │ 100.66.0.0/16     │
#
# Non-overlapping CIDRs enable VPC peering between environments
# if cross-environment access is needed (e.g., staging → prod read replica)
```
