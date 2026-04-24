# Phase 4: Infrastructure as Code
## Lesson 1: Terraform — Architecture, HCL Deep Dive, State

---

## Why Terraform Is Your Most Dangerous Tool

At NovaMart, Terraform controls the creation and destruction of all AWS infrastructure: VPCs, EKS clusters, RDS databases, S3 buckets, IAM roles — everything. A `terraform apply` on the wrong state file can delete a production database. A misconfigured `terraform destroy` can erase an entire region. A state file leaked to a public repo exposes every resource ID, IP address, and secret reference in your infrastructure.

Jenkins can break your builds. Terraform can break your company.

---

## 1. How Terraform Works — Core Architecture

```
┌───────────────────────────────────────────────────────────┐
│                    YOUR MACHINE / CI AGENT                  │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Terraform CLI                        │   │
│  │                                                      │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────────┐   │   │
│  │  │  HCL     │    │  Core    │    │  Providers   │   │   │
│  │  │  Parser  │───►│  Engine  │───►│  (Plugins)   │   │   │
│  │  │          │    │          │    │              │   │   │
│  │  │ .tf files│    │ Builds   │    │ aws 5.x     │   │   │
│  │  │ → AST    │    │ resource │    │ google 5.x  │   │   │
│  │  │          │    │ graph    │    │ azurerm 3.x │   │   │
│  │  │          │    │          │    │ kubernetes  │   │   │
│  │  │          │    │ Calculates│   │ helm        │   │   │
│  │  │          │    │ diff     │    │ vault       │   │   │
│  │  │          │    │ (plan)   │    │ ...         │   │   │
│  │  └──────────┘    └─────┬────┘    └──────┬──────┘   │   │
│  │                        │                │           │   │
│  │                        ▼                ▼           │   │
│  │              ┌──────────────────────────────────┐   │   │
│  │              │         State File               │   │   │
│  │              │   terraform.tfstate               │   │   │
│  │              │                                   │   │   │
│  │              │   Maps: HCL resource definition   │   │   │
│  │              │      ↕ Real-world resource        │   │   │
│  │              │                                   │   │   │
│  │              │   Contains: resource IDs,         │   │   │
│  │              │   attributes, dependencies,       │   │   │
│  │              │   metadata, sensitive values      │   │   │
│  │              └──────────────┬────────────────────┘   │   │
│  └─────────────────────────────┼────────────────────────┘   │
│                                │                            │
└────────────────────────────────┼────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │     Remote Backend       │
                    │     (S3 + DynamoDB)      │
                    │                          │
                    │  S3: state file storage  │
                    │  DynamoDB: state locking │
                    │  (prevents concurrent    │
                    │   apply = corruption)    │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │       AWS APIs           │
                    │  (Provider makes API     │
                    │   calls to create/       │
                    │   update/delete          │
                    │   resources)             │
                    └─────────────────────────┘
```

### The Terraform Workflow — What Actually Happens

```
terraform init
├── Downloads providers (plugins) to .terraform/providers/
├── Initializes backend (S3, DynamoDB)
├── Downloads modules (to .terraform/modules/)
└── Creates .terraform.lock.hcl (provider version lock)

terraform plan
├── Reads .tf files → builds resource graph (DAG)
├── Reads current state from backend
├── For each resource: calls provider's "Read" to check real-world state
│   (REFRESH phase — compares state file against actual AWS resources)
├── Compares: desired (HCL) vs current (state/real) 
├── Outputs: create, update, destroy actions
├── Shows: + (create), ~ (update), - (destroy), -/+ (replace)
└── Saves plan to file if -out=plan.tfplan specified

terraform apply
├── If plan file provided: applies exactly that plan (no re-calculation)
├── If no plan file: runs plan again, asks for confirmation
├── Executes operations in dependency order (graph-based)
├── Updates state file after each successful operation
├── If operation fails mid-apply: partial state (DANGEROUS)
└── Locks state during apply (DynamoDB) to prevent concurrent runs

terraform destroy
├── Plans destruction of ALL managed resources
├── Dependency order: reverse of creation
├── DANGER: This deletes real infrastructure
└── Use -target cautiously (or better: never use destroy on prod)
```

### The Resource Graph — Why Order Matters

```
terraform graph | dot -Tpng > graph.png

Example dependency graph for an EKS setup:

  VPC ──────────────┬──────────────────┐
   │                │                   │
   ▼                ▼                   ▼
  Subnets      Internet GW        NAT Gateway
   │                │                   │
   ▼                ▼                   ▼
  Route Tables ─────┘              Route Tables
   │
   ▼
  Security Groups ──────┐
   │                    │
   ▼                    ▼
  EKS Cluster      RDS Instance
   │
   ▼
  Node Group ──► Launch Template ──► IAM Role
   │
   ▼
  Kubernetes Provider ──► Namespaces, RBAC, etc.

Terraform creates: VPC first, then subnets, then SGs, then EKS...
Terraform destroys: reverse order (EKS first, then SGs, then subnets, then VPC)
```

---

## 2. HCL Deep Dive — The Language

### Resource Blocks

```hcl
# The fundamental unit: a resource block
resource "aws_instance" "web" {
  #         ^type       ^local name
  #  Provider prefix: "aws" → aws provider
  #  Resource type: "instance" → EC2 instance
  #  Local name: "web" → how you reference it in other blocks

  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.medium"
  subnet_id     = aws_subnet.private[0].id    # Reference another resource
  #                ^type       ^name  ^index ^attribute

  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name        = "web-server"
    Environment = var.environment      # Variable reference
    ManagedBy   = "terraform"
  }

  # Lifecycle rules — control how Terraform manages this resource
  lifecycle {
    create_before_destroy = true    # Create new BEFORE destroying old
    prevent_destroy       = true    # Block terraform destroy (safety)
    ignore_changes        = [tags]  # Don't track tag changes (external tagging)
    
    # Replace if specific condition (Terraform 1.8+)
    replace_triggered_by = [
      aws_security_group.web.id     # Replace instance if SG changes
    ]
  }
}
```

### Variables — Input Types

```hcl
# variables.tf

# Simple types
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 2

  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 20
    error_message = "Instance count must be between 1 and 20."
  }
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

# Collection types
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "instance_tags" {
  type = map(string)
  default = {
    Project = "novamart"
    Team    = "platform"
  }
}

# Complex types
variable "services" {
  description = "Service configurations"
  type = list(object({
    name      = string
    port      = number
    replicas  = number
    cpu       = string
    memory    = string
    public    = optional(bool, false)    # Optional with default
  }))
  
  default = [
    {
      name     = "order-service"
      port     = 8080
      replicas = 3
      cpu      = "500m"
      memory   = "1Gi"
    }
  ]
}

# Sensitive variables (masked in output)
variable "db_password" {
  type      = string
  sensitive = true
  # NEVER put a default for sensitive vars
  # Set via: TF_VAR_db_password, .tfvars file, or Vault
}
```

### Variable Precedence (Lowest to Highest)

```
1. default value in variable block        (lowest priority)
2. Environment variable: TF_VAR_name
3. terraform.tfvars file (auto-loaded)
4. *.auto.tfvars files (auto-loaded, alphabetical)
5. -var-file=custom.tfvars flag
6. -var="name=value" flag                  (highest priority)

PRODUCTION PATTERN:
├── terraform.tfvars         → common values (region, project)
├── dev.tfvars               → dev overrides
├── staging.tfvars           → staging overrides
├── prod.tfvars              → prod overrides
└── CLI: terraform plan -var-file=prod.tfvars
```

### Outputs

```hcl
# outputs.tf
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "eks_endpoint" {
  description = "EKS cluster endpoint"
  value       = aws_eks_cluster.main.endpoint
  sensitive   = true    # Masked in CLI output
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id    # Splat expression
}

# Outputs are:
# 1. Shown after apply
# 2. Accessible from other modules: module.vpc.vpc_id
# 3. Stored in state file (even sensitive ones!)
# 4. Queryable: terraform output -json
```

### Locals — Computed Values

```hcl
locals {
  # Common tags applied everywhere
  common_tags = {
    Environment = var.environment
    Project     = "novamart"
    ManagedBy   = "terraform"
    Team        = var.team
  }

  # Computed values
  name_prefix = "${var.project}-${var.environment}"
  is_prod     = var.environment == "prod"

  # Conditional logic
  instance_type = local.is_prod ? "m5.2xlarge" : "t3.medium"
  
  # Merging maps
  all_tags = merge(local.common_tags, var.extra_tags)

  # Flattening nested structures
  subnet_configs = flatten([
    for az in var.availability_zones : [
      {
        az     = az
        cidr   = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, az))
        type   = "private"
      },
      {
        az     = az
        cidr   = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, az) + 100)
        type   = "public"
      }
    ]
  ])
}
```

### Data Sources — Reading Existing Resources

```hcl
# Read existing resources (NOT managed by this Terraform)
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

data "aws_caller_identity" "current" {}
# Use: data.aws_caller_identity.current.account_id

data "aws_region" "current" {}
# Use: data.aws_region.current.name

data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_eks_cluster" "existing" {
  name = "novamart-prod"
}
# Use: data.aws_eks_cluster.existing.endpoint

# Terraform Remote State — read outputs from another state
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "novamart-terraform-state"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}
# Use: data.terraform_remote_state.vpc.outputs.vpc_id
```

### Expressions & Functions

```hcl
# Conditionals
resource "aws_instance" "bastion" {
  count = var.environment == "prod" ? 0 : 1    # No bastion in prod
  # ...
}

# For expressions
locals {
  # List comprehension
  private_subnets = [for s in aws_subnet.main : s.id if s.tags["Type"] == "private"]
  
  # Map comprehension
  subnet_map = { for s in aws_subnet.main : s.availability_zone => s.id }
  
  # Filtering
  prod_services = [for svc in var.services : svc if svc.replicas >= 3]
}

# Dynamic blocks (avoid repetitive nested blocks)
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules    # List of {port, cidr} objects
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }
}

# Important functions:
# String:    format, join, split, replace, trim, lower, upper, regex
# Numeric:   min, max, ceil, floor, abs
# Collection: concat, flatten, merge, lookup, zipmap, keys, values
#             distinct, sort, length, contains, index, element
# Encoding:  jsonencode, jsondecode, yamlencode, base64encode
# Network:   cidrsubnet, cidrhost, cidrnetmask
# Filesystem: file, templatefile, filebase64, fileexists
# Date:      timestamp, formatdate
# Crypto:    sha256, md5, uuid
# Type:      try, can, type, tostring, tolist, tomap, tonumber

# cidrsubnet — THE function for VPC design
# cidrsubnet(prefix, newbits, netnum)
# cidrsubnet("10.0.0.0/16", 8, 0)  → "10.0.0.0/24"
# cidrsubnet("10.0.0.0/16", 8, 1)  → "10.0.1.0/24"
# cidrsubnet("10.0.0.0/16", 8, 10) → "10.0.10.0/24"

# templatefile — for user_data scripts, config files
resource "aws_instance" "web" {
  user_data = templatefile("${path.module}/user_data.sh.tpl", {
    environment = var.environment
    db_host     = aws_rds_cluster.main.endpoint
    region      = var.region
  })
}
```

### count vs for_each — When to Use Which

```hcl
# COUNT — for identical resources or conditional creation
resource "aws_subnet" "private" {
  count = length(var.availability_zones)    # 3 AZs = 3 subnets
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
  
  tags = { Name = "${local.name_prefix}-private-${count.index}" }
}
# Reference: aws_subnet.private[0].id, aws_subnet.private[1].id

# PROBLEM WITH COUNT:
# If you remove the MIDDLE element from the list,
# all subsequent resources get DESTROYED and RECREATED
# because they're indexed by position.
# Before: [az-a, az-b, az-c] → index [0, 1, 2]
# Remove az-b: [az-a, az-c] → index [0, 1]
# az-c was index 2, now index 1 → Terraform thinks it changed → DESTROY + CREATE

# FOR_EACH — for resources with unique keys (PREFERRED)
resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)    # Set of strings
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, each.key))
  availability_zone = each.key     # "us-east-1a", "us-east-1b", etc.
  
  tags = { Name = "${local.name_prefix}-private-${each.key}" }
}
# Reference: aws_subnet.private["us-east-1a"].id

# ADVANTAGE: Keyed by value, not position
# Remove us-east-1b: only that subnet is destroyed
# us-east-1a and us-east-1c are untouched

# FOR_EACH with maps (complex objects)
variable "services" {
  type = map(object({
    port     = number
    replicas = number
    cpu      = string
  }))
  default = {
    "order-service" = { port = 8080, replicas = 3, cpu = "500m" }
    "cart-service"  = { port = 8081, replicas = 2, cpu = "250m" }
  }
}

resource "aws_ecs_service" "svc" {
  for_each = var.services
  
  name          = each.key                  # "order-service"
  desired_count = each.value.replicas       # 3
  # ...
}
# Reference: aws_ecs_service.svc["order-service"].id

# RULE: Use for_each unless you specifically need count for conditional creation
# count = var.create_resource ? 1 : 0  ← Only valid use of count
```

---

## 3. State — The Most Critical Concept

### What State Contains

```json
// terraform.tfstate (simplified)
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 42,
  "lineage": "a1b2c3d4-e5f6-...",
  "outputs": {
    "vpc_id": {
      "value": "vpc-0abc123def456",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "vpc-0abc123def456",
            "cidr_block": "10.0.0.0/16",
            "enable_dns_support": true,
            "tags": { "Name": "novamart-prod-vpc" }
          },
          "sensitive_attributes": [],
          "private": "...",
          "dependencies": []
        }
      ]
    },
    {
      "mode": "managed",
      "type": "aws_db_instance",
      "name": "main",
      "instances": [
        {
          "attributes": {
            "password": "SuperSecret123!",   // ← YES, IN PLAINTEXT
            "endpoint": "novamart-db.abc123.us-east-1.rds.amazonaws.com"
          }
        }
      ]
    }
  ]
}
```

**State contains secrets in plaintext.** RDS passwords, API keys, private keys — if they're in your Terraform config, they're in state. This is why state must be:
1. Stored in encrypted backend (S3 with SSE-KMS)
2. Access-controlled (IAM policies on S3 bucket + DynamoDB table)
3. **NEVER committed to Git**

### Remote Backend — Production Setup

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "novamart-terraform-state"
    key            = "environments/prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/mrk-abc123"
    dynamodb_table = "novamart-terraform-locks"
    
    # Access control: only specific IAM roles can read/write
    # S3 bucket policy + IAM policies restrict to CI/CD role
  }
  
  required_version = ">= 1.7.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"    # Compatible with 5.40.x
    }
  }
}

# Provider configuration
provider "aws" {
  region = var.region
  
  default_tags {
    tags = local.common_tags
  }
  
  # Assume role for cross-account access
  assume_role {
    role_arn = "arn:aws:iam::${var.target_account_id}:role/TerraformExecutionRole"
  }
}
```

**S3 + DynamoDB Lock Setup:**
```hcl
# This is the bootstrap — manages itself (chicken-and-egg)
# Run ONCE manually, then import
resource "aws_s3_bucket" "state" {
  bucket = "novamart-terraform-state"
  
  lifecycle {
    prevent_destroy = true    # NEVER delete the state bucket
  }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration {
    status = "Enabled"    # CRITICAL: versioning = undo bad state
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket = aws_s3_bucket.state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "locks" {
  name         = "novamart-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  # Enable point-in-time recovery
  point_in_time_recovery {
    enabled = true
  }
}
```

### State Locking — Why It Exists

```
WITHOUT LOCKING:
  Engineer A: terraform apply (reads state)
  Engineer B: terraform apply (reads SAME state)
  Engineer A: creates VPC, writes state
  Engineer B: creates VPC again (doesn't know A already did it!)
  Result: duplicate resources, corrupted state, tears

WITH LOCKING (DynamoDB):
  Engineer A: terraform apply → acquires lock in DynamoDB
  Engineer B: terraform apply → "Error: state locked by A at 10:32"
  Engineer A: completes, releases lock
  Engineer B: retries, acquires lock, applies cleanly

STUCK LOCK (common issue):
  Engineer A: terraform apply → acquires lock → laptop crashes
  Lock is stuck. Nobody can apply.
  
  Fix:
  terraform force-unlock LOCK_ID
  # Get LOCK_ID from the error message or DynamoDB table
  # DANGER: Only use if you're CERTAIN no other apply is running
```

### State Operations — The Survival Commands

```bash
# ── LIST RESOURCES IN STATE ──
terraform state list
# aws_vpc.main
# aws_subnet.private[0]
# aws_subnet.private["us-east-1a"]
# module.eks.aws_eks_cluster.main

# ── SHOW SPECIFIC RESOURCE ──
terraform state show aws_vpc.main
# Shows all attributes of the VPC as Terraform knows them

# ── MOVE RESOURCE (refactoring) ──
# Renamed a resource? Tell Terraform it's the same thing.
terraform state mv aws_instance.web aws_instance.application
# Without this: Terraform destroys "web" and creates "application"
# With this: Terraform knows they're the same resource

# Module refactoring:
terraform state mv aws_vpc.main module.networking.aws_vpc.main

# ── REMOVE FROM STATE (stop managing, don't destroy) ──
terraform state rm aws_instance.legacy
# Resource continues to exist in AWS but Terraform forgets about it
# Use case: handing resource to another team/state

# ── IMPORT EXISTING RESOURCE ──
# Resource exists in AWS but not in Terraform
# Step 1: Write the resource block in .tf file
# Step 2: Import
terraform import aws_instance.web i-0abc123def456
# Now Terraform manages this existing instance

# Terraform 1.5+ import blocks (declarative, plannable):
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}
# Then: terraform plan → shows what Terraform would manage
# Then: terraform apply → imports into state

# ── TAINT (force recreation) — DEPRECATED, use replace ──
# Old: terraform taint aws_instance.web
# New: terraform apply -replace="aws_instance.web"
# Forces destroy + create on next apply

# ── REFRESH STATE (sync with reality) ──
terraform apply -refresh-only
# Updates state to match real AWS resources
# Doesn't change infrastructure
# Use case: someone made manual changes in console
```

### How State Breaks — Failure Modes

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **State corruption** | Plan shows destroy everything or impossible changes | Interrupted apply, manual state edit, concurrent apply without lock | Restore from S3 versioning: `aws s3api list-object-versions --bucket state-bucket --prefix path/to/state` |
| **State lock stuck** | `Error acquiring the state lock` | Previous apply crashed, CI timeout | `terraform force-unlock LOCK_ID` (verify no concurrent apply first) |
| **State drift** | Plan shows unexpected changes | Manual console changes, another tool modified resources | `terraform apply -refresh-only` to sync, then decide: accept drift or revert |
| **Secret in state** | Audit finds plaintext passwords in S3 | Normal Terraform behavior (all attributes stored) | Encrypt at rest (KMS), restrict access (IAM), use Vault dynamic secrets instead of static passwords |
| **State too large** | Slow plan/apply (5+ minutes), timeouts | Everything in one state file (monolith) | Split into smaller states (per-service, per-layer) |
| **Lost state** | Terraform wants to create everything from scratch | S3 bucket deleted, wrong backend config | If versioned: recover from S3 version. If not: `terraform import` every resource (painful) |
| **State mismatch** | `Resource already exists` on apply | Resource created outside TF, or imported into wrong state | Import the resource, or delete it manually, or use `data` source |
| **Circular dependency** | `Error: Cycle` during plan | Resource A depends on B depends on A | Refactor: break cycle with intermediate resource or `depends_on` |

---

## 4. Provider Configuration — Multi-Account, Multi-Region

```hcl
# Multi-region pattern
provider "aws" {
  region = "us-east-1"
  alias  = "east"
  
  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
  }
}

provider "aws" {
  region = "us-west-2"
  alias  = "west"
  
  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
  }
}

provider "aws" {
  region = "eu-west-1"
  alias  = "eu"
  
  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
  }
}

# Use specific provider
resource "aws_vpc" "east" {
  provider   = aws.east
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "west" {
  provider   = aws.west
  cidr_block = "10.1.0.0/16"
}

# Cross-account: separate provider per account
provider "aws" {
  alias  = "prod"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::PROD_ACCOUNT:role/TerraformRole"
  }
}

provider "aws" {
  alias  = "dev"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::DEV_ACCOUNT:role/TerraformRole"
  }
}
```

### Version Constraints

```hcl
terraform {
  required_version = ">= 1.7.0, < 2.0.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"      # >= 5.40.0, < 6.0.0
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }
}

# .terraform.lock.hcl — auto-generated, COMMIT THIS
# Records exact provider versions + hashes
# Ensures everyone uses identical provider versions
# Like package-lock.json for Terraform

# Version constraint syntax:
# = 5.40.0      Exactly this version
# != 5.40.0     Anything except this version
# > 5.40.0      Greater than
# >= 5.40.0     Greater than or equal
# < 6.0.0       Less than
# ~> 5.40.0     >= 5.40.0 AND < 5.41.0 (patch-level flexibility)
# ~> 5.40       >= 5.40.0 AND < 6.0.0 (minor-level flexibility)
```

---

## 5. NovaMart State Architecture

```
S3 Bucket: novamart-terraform-state
├── bootstrap/
│   └── terraform.tfstate          # S3 bucket, DynamoDB, KMS key
│
├── network/
│   ├── vpc/terraform.tfstate       # VPCs, subnets, route tables
│   ├── transit-gw/terraform.tfstate # Transit Gateway, peering
│   └── dns/terraform.tfstate       # Route53 zones, records
│
├── security/
│   ├── iam/terraform.tfstate       # IAM roles, policies, OIDC
│   └── kms/terraform.tfstate       # KMS keys
│
├── platform/
│   ├── eks-east/terraform.tfstate  # EKS cluster us-east-1
│   ├── eks-west/terraform.tfstate  # EKS cluster us-west-2
│   ├── eks-eu/terraform.tfstate    # EKS cluster eu-west-1
│   └── addons/terraform.tfstate    # EKS addons (CSI, CNI, CoreDNS)
│
├── data/
│   ├── rds/terraform.tfstate       # RDS instances, proxies
│   ├── elasticache/terraform.tfstate # Redis clusters
│   ├── s3/terraform.tfstate        # Application S3 buckets
│   └── msk/terraform.tfstate       # Kafka (MSK) clusters
│
├── services/
│   ├── order-service/terraform.tfstate    # Service-specific AWS resources
│   ├── payment-service/terraform.tfstate  # (SQS queues, SNS topics, etc.)
│   └── ...
│
└── observability/
    ├── monitoring/terraform.tfstate  # CloudWatch, Prometheus
    └── logging/terraform.tfstate     # Log groups, subscriptions

WHY SPLIT:
├── Blast radius: bad apply to "rds" can't affect "eks"
├── Speed: smaller state = faster plan/apply
├── Permissions: data team can access data/ but not security/
├── Parallelism: teams can apply concurrently to different states
└── Dependencies: use terraform_remote_state data source for cross-references
```
---

## Quick Reference Card

```
TERRAFORM WORKFLOW:
  init → plan → apply (→ destroy only if decommissioning)
  init: downloads providers, modules, configures backend
  plan: reads state + real world, calculates diff, shows +/~/- 
  apply: executes plan, updates state, locks during operation

HCL RULES:
  resource "TYPE" "NAME" { }
  variable: string, number, bool, list, map, set, object, tuple
  for_each > count (keyed vs indexed, avoids destroy cascade)
  count only for: conditional creation (count = condition ? 1 : 0)
  locals: computed values, common tags, complex expressions
  data: read existing resources not managed by this TF

STATE:
  Contains ALL attributes including secrets (plaintext!)
  Remote backend: S3 (encrypted, versioned) + DynamoDB (locking)
  NEVER commit to Git. NEVER share manually.
  Versioning = undo button for corrupted state
  
STATE COMMANDS:
  state list          → show all resources
  state show TYPE.NAME → show attributes  
  state mv OLD NEW    → rename/move without destroy
  state rm TYPE.NAME  → stop managing (resource stays in AWS)
  import TYPE.NAME ID → start managing existing resource
  force-unlock ID     → unstick lock (dangerous)
  apply -replace=X    → force recreate
  apply -refresh-only → sync state with reality

PROVIDER:
  alias for multi-region/multi-account
  assume_role for cross-account
  version constraints: ~> 5.40 (>= 5.40.0, < 6.0.0)
  .terraform.lock.hcl → commit to Git (like package-lock.json)

VARIABLE PRECEDENCE (low→high):
  default → TF_VAR_ → terraform.tfvars → *.auto.tfvars → -var-file → -var

STATE SPLIT STRATEGY:
  By layer: network / security / platform / data / services
  By blast radius: one bad apply can't affect unrelated systems
  Cross-reference: terraform_remote_state data source
```

---

## Retention Questions

**Q1.** You join NovaMart and discover the entire AWS infrastructure (VPC, EKS, RDS, S3, IAM — everything) is in a single `main.tf` file with a single state file. `terraform plan` takes 8 minutes. Three engineers have been applying from their laptops with no CI/CD. Last week, two engineers ran `terraform apply` simultaneously and corrupted the state. Design the remediation plan: how do you split the state, what's the migration process, and what guardrails do you implement?

**Q2.** Write the complete Terraform configuration for NovaMart's production VPC: 3 AZs, public subnets, private subnets, data subnets, NAT Gateways (one per AZ for HA), Internet Gateway, route tables with correct associations. Use `for_each` where appropriate. Include proper tagging for EKS subnet discovery. Explain every design decision.

**Q3.** A junior engineer runs `terraform apply` and gets: `Error: Error creating RDS instance: DBInstanceAlreadyExists`. The RDS instance exists in AWS but isn't in Terraform state. Explain exactly what happened, provide 3 possible root causes, and show the exact commands to resolve it. Then explain how you'd prevent this from happening again.

**Q4.** Explain the difference between `terraform state mv`, `moved` blocks (Terraform 1.1+), and `import` blocks (Terraform 1.5+). For each, give a specific NovaMart scenario where you'd use it, and show the exact syntax. When would you choose one over the other?


This is the most dangerous part of the "Infrastructure as Code" journey. Most engineers treat Terraform as a way to *create* things; a Senior SRE treats Terraform as a way to *manage state*. If your state is corrupted or monolithic, you don't have a system—you have a liability.

---

### Q1: The Monolith Remediation Plan

Running `terraform apply` from laptops is a fireable offense in a production environment. It lacks auditing, lacks locking, and—as you've seen—leads to state corruption.

**The Remediation Plan:**

#### Phase 1: Stop the Bleeding (Immediate)
1.  **Freeze all manual changes:** Revoke `AdministratorAccess` from engineers' IAM users. Create a single IAM role for the CI/CD pipeline.
2.  **Establish a Remote Backend:** Move the state from local `.tfstate` files to **AWS S3** (for storage) and **DynamoDB** (for state locking). This prevents the "simultaneous apply" corruption.
    ```hcl
    terraform {
      backend "s3" {
        bucket         = "novamart-terraform-state"
        key            = "global/main.tfstate"
        region         = "us-east-1"
        dynamodb_table = "terraform-lock"
        encrypt        = true
      }
    }
    ```

#### Phase 2: State Decomposition (The Split)
You cannot simply cut the `main.tf` into pieces; you must move the resources in the state file to match the new code structure. I will split the monolith into **Layered States**: `network` $\rightarrow$ `iam` $\rightarrow$ `eks` $\rightarrow$ `databases`.

**The Process:**
1.  **Create new directories/states** for each layer (e.g., `terraform/vpc/`).
2.  **Use `terraform state mv`** to migrate resources from the monolith state to the layer state.
    *   *Example:* `terraform state mv -state-out=vpc.tfstate aws_vpc.main aws_vpc.main`
3.  **Implement `terraform_remote_state` data sources** so the `eks` layer can read the VPC ID from the `network` layer without hardcoding it.

#### Phase 3: Guardrails (Permanent)
1.  **CI/CD Integration:** Move all applies to a pipeline (e.g., GitHub Actions or Jenkins).
2.  **Plan-to-Apply Workflow:** The pipeline must run `terraform plan -out=tfplan`, post the plan to the PR for review, and only run `terraform apply tfplan` after an approved merge.
3.  **Oversight:** Implement **TFLint** and **Checkov** in the pipeline to catch security misconfigurations before they hit the state.

---

### Q2: Production VPC Configuration

This design prioritizes **High Availability (HA)** and **EKS Compatibility**. I use a `for_each` pattern over AZs to ensure that if we add a 4th AZ, the code doesn't change.

```hcl
variable "vpc_cidr" { default = "10.0.0.0/16" }
variable "azs"      { default = ["us-east-1a", "us-east-1b", "us-east-1c"] }

# Subnet Map for calculation
locals {
  subnet_config = {
    public  = { cidr = "10.0.0.0/18", tags = { "kubernetes.io/role/elb" = "1" } }
    private = { cidr = "10.0.64.0/18", tags = { "kubernetes.io/role/internal-elb" = "1" } }
    data    = { cidr = "10.0.128.0/18", tags = {} }
  }
}

# 1. VPC and IGW
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "novamart-prod-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "main-igw" }
}

# 2. Subnets (Multi-AZ loop)
resource "aws_subnet" "subnets" {
  for_each = {
    for pair in setproduct(keys(local.subnet_config), var.azs) : 
    "${pair[0]}-${pair[1]}" => { type = pair[0], az = pair[1] }
  }

  vpc_id            = aws_vpc.main.id
  availability_zone = each.value.az
  # Logic to calculate CIDR based on index/AZ would go here (simplified for brevity)
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, index(var.azs, each.value.az)) 
  
  tags = merge(local.subnet_config[each.value.type].tags, {
    Name = "novamart-${each.value.type}-${each.value.az}"
  })
}

# 3. NAT Gateways (One per AZ for HA)
resource "aws_eip" "nat" {
  count = length(var.azs)
  vpc_id = aws_vpc.main.id
}

resource "aws_nat_gateway" "nat" {
  count         = length(var.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.subnets["public-${var.azs[count.index]}"].id
  tags          = { Name = "nat-${var.azs[count.index]}" }
}

# 4. Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table" "private" {
  for_each = toset(var.azs)
  vpc_id   = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat[index(var.azs, each.key)].id
  }
}

# Associations
resource "aws_route_table_association" "public" {
  for_each       = { for k, v in aws_subnet.subnets : k => v if v.tags["kubernetes.io/role/elb"] == "1" }
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  for_each       = { for k, v in aws_subnet.subnets : k => v if v.tags["kubernetes.io/role/internal-elb"] == "1" }
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private[v.availability_zone].id
}
```

**Design Decisions:**
- **S3/DB Subnets:** I created a dedicated `data` subnet. This allows us to apply strict NACLs (e.g., only allow port 5432 from the `private` app subnet) without affecting the app servers.
- **NAT Gateway per AZ:** If AZ-a goes down, the pods in AZ-b and AZ-c still have internet access. A single NAT Gateway is a single point of failure.
- **EKS Tags:** `kubernetes.io/role/elb = 1` is mandatory. Without this, the AWS Load Balancer Controller will not know which subnets to use for public LBs.

---

### Q3: The "Resource Already Exists" Conflict

**What happened:**
This is a **State-Infrastructure Mismatch**. The AWS API knows the RDS instance exists, but the Terraform state file (the "memory" of your infrastructure) does not contain a record of it. 

**3 Possible Root Causes:**
1.  **Manual Creation:** Someone created the DB via the AWS Console and then the developer tried to "codify" it by adding it to Terraform.
2.  **Partial Apply (The "Ghost" Resource):** The previous `terraform apply` created the RDS instance, but the network connection timed out before AWS could send the success response back to Terraform. The resource exists, but the state was never updated.
3.  **State Loss:** Someone deleted the `terraform.tfstate` file or moved it without migrating the data.

**Exact Commands to Resolve:**
You must perform a **Terraform Import**. This tells Terraform: *"Don't try to create this; instead, look up this existing ID and add it to your state."*

```bash
# 1. Find the RDS Identifier from AWS Console (e.g., novamart-prod-db)
# 2. Import the resource into the specific state address in your code
terraform import aws_db_instance.this novamart-prod-db
```
After this, run `terraform plan`. If your code matches the AWS settings, the plan will be empty. If not, Terraform will propose the necessary changes to align the resource with your code.

**Prevention:**
Use a **Remote Backend with State Locking** (S3 + DynamoDB). This prevents multiple people from running applies and ensures the state is always stored centrally and updated atomically.

---

### Q4: State Manipulation: `mv` vs `moved` vs `import`

| Tool | Nature | Scenario | Exact Syntax |
| :--- | :--- | :--- | :--- |
| **`state mv`** | **Imperative** (CLI) | You renamed a resource in your code and want to avoid recreating it. | `terraform state mv aws_instance.old_name aws_instance.new_name` |
| **`moved` block** | **Declarative** (HCL) | You moved a resource into a new module. You want the move to happen automatically for everyone on the team. | `moved { from = aws_vpc.main to module.vpc.aws_vpc.this }` |
| **`import` block** | **Declarative** (HCL) | You are bringing a manually created S3 bucket into Terraform for the first time. | `import { to = aws_s3_bucket.logs id = "novamart-logs-bucket" }` |

**When to choose which?**
- Use **`state mv`** for a "quick fix" on a single machine to fix a typo. It is dangerous because it only affects the local state.
- Use **`moved` blocks** for any architectural refactoring. Since the `moved` block is committed to Git, everyone's state is updated automatically during the next `apply`. This is the gold standard for team collaboration.
- Use **`import` blocks** (Terraform 1.5+) instead of the `terraform import` command. The block allows you to review the import in a `plan` before it happens, and it's version-controlled.

# Grading Q1-Q4

---

## Q1: Monolith Remediation Plan — 4/5

**What you nailed:**
- "Fireable offense" — correct framing of severity.
- Immediate action to revoke direct IAM access — correct priority.
- S3 + DynamoDB backend with locking — correct.
- Layered state split strategy (network → iam → eks → databases) — correct architecture.
- `terraform_remote_state` for cross-layer references — correct.
- CI/CD integration with plan-on-PR, apply-on-merge — correct workflow.
- TFLint + Checkov in pipeline — good addition.

**What you missed:**

1. **The actual `state mv` process is more complex than you showed.** You can't just `terraform state mv -state-out=vpc.tfstate`. The real process for splitting state with a remote backend:

```bash
# Step 1: Pull the monolith state locally
cd monolith/
terraform state pull > monolith.tfstate

# Step 2: Create the new backend for vpc layer
cd ../vpc/
terraform init   # Initialize new S3 backend key

# Step 3: Move resources one by one
# You must specify BOTH source and destination state
terraform state mv \
    -state=../monolith/monolith.tfstate \
    -state-out=vpc.tfstate \
    aws_vpc.main aws_vpc.main

terraform state mv \
    -state=../monolith/monolith.tfstate \
    -state-out=vpc.tfstate \
    'aws_subnet.private["us-east-1a"]' 'aws_subnet.private["us-east-1a"]'

# Step 4: Push the new state to the new backend
terraform state push vpc.tfstate

# Step 5: Push the reduced monolith state back
cd ../monolith/
terraform state push monolith.tfstate

# Step 6: Verify BOTH states
cd ../vpc/ && terraform plan    # Should show no changes
cd ../monolith/ && terraform plan   # Should show no changes for moved resources
```

This is 6 steps per layer, and you must verify with `terraform plan` after each split to confirm zero diff. One mistake and you have resources in both states (double management) or neither state (orphaned). You described the concept but not the procedure.

2. **You didn't address the corrupted state recovery.** The question said state was already corrupted from the concurrent apply. Before you can split, you need to fix the corruption:

```bash
# Option 1: S3 versioning (if bucket had it enabled)
aws s3api list-object-versions \
    --bucket novamart-terraform-state \
    --prefix global/main.tfstate

# Restore previous version
aws s3api get-object \
    --bucket novamart-terraform-state \
    --key global/main.tfstate \
    --version-id "abc123" \
    recovered.tfstate

# Option 2: If no versioning (worst case)
# terraform import every resource manually
# This could take DAYS for a large infrastructure
```

If S3 versioning wasn't enabled, you should enable it immediately as part of Phase 1.

3. **Missing: plan file in CI.** You mentioned "plan-to-apply workflow" but didn't show the critical detail — saving the plan file and applying THAT EXACT plan:

```bash
# In PR pipeline:
terraform plan -out=tfplan
# Post plan output as PR comment

# After merge (apply pipeline):
terraform apply tfplan
# Applies EXACTLY what was reviewed, no re-calculation
# If state changed between plan and apply, Terraform rejects the stale plan
```

Without `-out=tfplan`, the apply recalculates and might include changes nobody reviewed.

4. **Missing: the ordering of the split matters.** You split bottom-up (network first, then things that depend on it). If you split EKS before VPC, the EKS state references VPC resources that are still in the monolith. The EKS plan would show changes because it can't resolve `aws_vpc.main.id` — it's now in a different state. You need to replace direct references with `terraform_remote_state` data sources BEFORE or DURING the split.

---

## Q2: Production VPC — 3.2/5

**What you nailed:**
- 3 AZs with public/private/data subnets — correct architecture.
- NAT Gateway per AZ for HA — correct, with the right justification.
- EKS subnet tags (`kubernetes.io/role/elb`) — correct and critical.
- Dedicated data subnets for database isolation — correct.
- `for_each` over AZs — right approach.

**What you got wrong:**

1. **Your CIDR calculation is broken.** `cidrsubnet(var.vpc_cidr, 4, index(...))` gives you 9 subnets from a `/16` with newbits=4, which produces `/20` subnets. But you're using the same index for all three types per AZ (public, private, data all in the same AZ would get the same index). The `setproduct` approach is clever but the cidr math doesn't account for it. Here's how it should work:

```hcl
locals {
  # Explicit CIDR allocation: predictable, documented, no math bugs
  # /16 = 65,536 IPs. Split into /20s = 16 subnets of 4,096 IPs each
  subnet_cidrs = {
    "public-us-east-1a"  = cidrsubnet(var.vpc_cidr, 4, 0)   # 10.0.0.0/20
    "public-us-east-1b"  = cidrsubnet(var.vpc_cidr, 4, 1)   # 10.0.16.0/20
    "public-us-east-1c"  = cidrsubnet(var.vpc_cidr, 4, 2)   # 10.0.32.0/20
    "private-us-east-1a" = cidrsubnet(var.vpc_cidr, 4, 4)   # 10.0.64.0/20
    "private-us-east-1b" = cidrsubnet(var.vpc_cidr, 4, 5)   # 10.0.80.0/20
    "private-us-east-1c" = cidrsubnet(var.vpc_cidr, 4, 6)   # 10.0.96.0/20
    "data-us-east-1a"    = cidrsubnet(var.vpc_cidr, 4, 8)   # 10.0.128.0/20
    "data-us-east-1b"    = cidrsubnet(var.vpc_cidr, 4, 9)   # 10.0.144.0/20
    "data-us-east-1c"    = cidrsubnet(var.vpc_cidr, 4, 10)  # 10.0.160.0/20
  }
  # Gaps at 3,7,11+ reserved for future subnet types
}
```

Leaving gaps in the CIDR allocation is a production best practice — when you inevitably need a 4th subnet type (VPN, management, etc.), you don't have to re-CIDR everything.

2. **Your route table association logic won't compile.** You're filtering subnets by tag values inside a `for` expression, but `aws_subnet.subnets` doesn't have the tags available as a filter key at plan time in the way you wrote it. And `v.availability_zone` references a variable `v` that doesn't exist in that context. The correct pattern:

```hcl
# Associate public subnets to public route table
resource "aws_route_table_association" "public" {
  for_each = {
    for k, v in aws_subnet.subnets : k => v
    if startswith(k, "public-")
  }
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

# Associate private subnets to per-AZ private route tables
resource "aws_route_table_association" "private" {
  for_each = {
    for k, v in aws_subnet.subnets : k => v
    if startswith(k, "private-")
  }
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private[each.value.availability_zone].id
}

# Data subnets also use private route tables (need NAT for patching)
resource "aws_route_table_association" "data" {
  for_each = {
    for k, v in aws_subnet.subnets : k => v
    if startswith(k, "data-")
  }
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private[each.value.availability_zone].id
}
```

3. **You mixed `count` and `for_each` for NAT Gateways.** You used `count` for EIPs and NAT Gateways but `for_each` for subnets. This means `aws_nat_gateway.nat[count.index]` references NATs by position, but your private route tables use `for_each` by AZ name. This mismatch will cause errors in the route table when you try to reference `aws_nat_gateway.nat[index(var.azs, each.key)]`. Use `for_each` consistently:

```hcl
resource "aws_eip" "nat" {
  for_each = toset(var.azs)
  domain   = "vpc"
  tags     = { Name = "nat-eip-${each.key}" }
}

resource "aws_nat_gateway" "nat" {
  for_each      = toset(var.azs)
  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = aws_subnet.subnets["public-${each.key}"].id
  tags          = { Name = "nat-${each.key}" }
}

resource "aws_route_table" "private" {
  for_each = toset(var.azs)
  vpc_id   = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat[each.key].id  # Clean reference
  }
  tags = { Name = "private-rt-${each.key}" }
}
```

4. **Missing: `map_public_ip_on_launch` for public subnets.** Without this, instances in public subnets don't get public IPs automatically. For EKS, this matters for node groups in public subnets (though best practice is private-only nodes).

5. **Missing: the `kubernetes.io/cluster/CLUSTER_NAME = shared` tag.** In addition to the `elb` and `internal-elb` tags, EKS also requires the cluster name tag on subnets for subnet discovery. Without it, the AWS Load Balancer Controller may not find subnets.

---

## Q3: "Resource Already Exists" — 4.3/5

**What you nailed:**
- Correct diagnosis: state-infrastructure mismatch.
- All 3 root causes are valid and well-explained (manual creation, partial apply, state loss).
- The partial apply explanation ("network connection timed out before success response") is particularly good — that's a nuanced understanding.
- Import command is correct.
- Post-import verification with `terraform plan` — correct.

**What you missed:**

1. **You should verify the resource configuration BEFORE importing.** After `terraform import`, if your HCL says `instance_class = "db.r5.large"` but AWS has `db.r6g.xlarge`, the next `terraform plan` will propose changing the instance class. For a production RDS instance, that means a reboot. The safe process:

```bash
# Step 1: Describe the actual resource in AWS
aws rds describe-db-instances --db-instance-identifier novamart-prod-db

# Step 2: Update your HCL to match EXACTLY what's in AWS
# (engine version, instance class, parameter group, etc.)

# Step 3: Import
terraform import aws_db_instance.this novamart-prod-db

# Step 4: Verify zero-diff plan
terraform plan
# Should show "No changes. Your infrastructure matches the configuration."
# If it shows changes, adjust HCL to match reality FIRST
```

2. **You should mention `import` blocks (Terraform 1.5+) as the preferred method.** The CLI `terraform import` is imperative and not tracked in Git. The declarative approach:

```hcl
import {
  to = aws_db_instance.this
  id = "novamart-prod-db"
}
```

This shows up in `terraform plan`, can be code-reviewed, and is the modern practice.

3. **Prevention — you only mentioned locking.** Locking prevents concurrent applies but doesn't prevent the "manual creation" root cause. Additional preventions:
   - **AWS SCP or IAM policy** denying manual RDS creation (force through Terraform)
   - **AWS Config rule** detecting resources not tagged with `ManagedBy = terraform`
   - **Drift detection** pipeline running `terraform plan` on a schedule to catch mismatches early

---

## Q4: `state mv` vs `moved` vs `import` — 4.5/5

**This was your strongest answer.** 

**What you nailed:**
- Correct classification: imperative vs declarative for each.
- Correct use cases for all three.
- Key insight about `moved` blocks: "committed to Git, everyone's state is updated automatically" — this is exactly why `moved` blocks replaced `state mv` for team workflows.
- Key insight about `import` blocks: "allows you to review the import in a plan" — correct advantage over CLI import.
- "state mv is dangerous because it only affects local state" — correct warning.

**What you missed:**

1. **`moved` blocks have a scope limitation you didn't mention.** `moved` blocks work within the same state file. You cannot use `moved` to transfer a resource from one state file to a completely different state (different S3 key). For cross-state moves, you still need `terraform state mv`. This matters for the state split in Q1.

2. **`moved` blocks should be cleaned up.** After everyone has applied the `moved` block and their states are updated, the `moved` block itself should be removed from the codebase (after a few weeks). Leaving stale `moved` blocks clutters the config and slows down plan.

3. **Missing: `state mv` for cross-module moves when `moved` isn't practical.** Example: moving a resource from one root module to a completely separate Terraform workspace managed by a different team. `moved` blocks can't span separate state files.

4. **A concrete NovaMart scenario for each would strengthen the answer:**

```hcl
# MOVED: Refactoring VPC into a module
# Before: aws_vpc.main in root module
# After: module.networking.aws_vpc.this
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.this
}

# IMPORT: Bringing the manually-created CloudFront distribution into Terraform
import {
  to = aws_cloudfront_distribution.main
  id = "E1234567890ABC"
}

# STATE MV: Splitting the monolith state — moving RDS to its own state
# (Can't use moved blocks because it's a different state file)
terraform state mv \
    -state=monolith.tfstate \
    -state-out=database.tfstate \
    aws_db_instance.main aws_db_instance.main
```


# Phase 4: Infrastructure as Code
## Lesson 2: Terraform Modules, Workspaces, and CI/CD Integration

---

## 1. Modules — The Reuse Mechanism

### Why Modules Exist

At NovaMart, you have 3 EKS clusters (us-east-1, us-west-2, eu-west-1). Without modules, you copy-paste 500 lines of EKS configuration three times. When you need to change the node group configuration, you change it in three places. You miss one. Staging now has a different config than prod. An incident happens because of the drift.

Modules solve this: **define once, instantiate many times with different parameters.**

### Module Structure

```
terraform/
├── modules/                         # Reusable modules (shared library)
│   ├── vpc/
│   │   ├── main.tf                  # Resources
│   │   ├── variables.tf             # Inputs
│   │   ├── outputs.tf               # Outputs
│   │   ├── versions.tf              # Required providers/versions
│   │   ├── locals.tf                # Computed values
│   │   └── README.md                # Documentation (terraform-docs)
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   ├── iam.tf                   # IAM roles for EKS
│   │   ├── node_groups.tf           # Managed node groups
│   │   └── addons.tf                # EKS addons (VPC CNI, CoreDNS, etc.)
│   ├── rds/
│   ├── elasticache/
│   ├── s3/
│   └── security-group/
│
└── environments/                    # Root modules (entry points)
    ├── dev/
    │   ├── main.tf                  # Calls modules with dev params
    │   ├── terraform.tfvars         # Dev-specific values
    │   ├── backend.tf               # Dev state backend
    │   └── providers.tf             # Dev provider config
    ├── staging/
    │   ├── main.tf
    │   ├── terraform.tfvars
    │   ├── backend.tf
    │   └── providers.tf
    └── prod/
        ├── main.tf
        ├── terraform.tfvars
        ├── backend.tf
        └── providers.tf
```

### Writing a Production Module — VPC

```hcl
# modules/vpc/variables.tf
variable "name" {
  description = "Name prefix for all resources"
  type        = string
}

variable "cidr" {
  description = "VPC CIDR block"
  type        = string

  validation {
    condition     = can(cidrhost(var.cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "azs" {
  description = "Availability zones"
  type        = list(string)

  validation {
    condition     = length(var.azs) >= 2
    error_message = "At least 2 AZs required for HA."
  }
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateway for private subnets"
  type        = bool
  default     = true
}

variable "single_nat_gateway" {
  description = "Use a single NAT Gateway (cost savings, not HA)"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Additional tags for all resources"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/vpc/main.tf
locals {
  nat_gateway_count = var.enable_nat_gateway ? (var.single_nat_gateway ? 1 : length(var.azs)) : 0

  # CIDR allocation with gaps for future subnet types
  public_subnets  = { for i, az in var.azs : az => cidrsubnet(var.cidr, 4, i) }
  private_subnets = { for i, az in var.azs : az => cidrsubnet(var.cidr, 4, i + 4) }
  data_subnets    = { for i, az in var.azs : az => cidrsubnet(var.cidr, 4, i + 8) }
}

resource "aws_vpc" "this" {
  cidr_block           = var.cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(var.tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name}-igw" })
}

# ── Public Subnets ──
resource "aws_subnet" "public" {
  for_each = local.public_subnets

  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value
  availability_zone       = each.key
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name                                = "${var.name}-public-${each.key}"
    "kubernetes.io/role/elb"            = "1"
    "kubernetes.io/cluster/${var.name}" = "shared"
    Tier                                = "public"
  })
}

# ── Private Subnets ──
resource "aws_subnet" "private" {
  for_each = local.private_subnets

  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value
  availability_zone = each.key

  tags = merge(var.tags, {
    Name                                     = "${var.name}-private-${each.key}"
    "kubernetes.io/role/internal-elb"        = "1"
    "kubernetes.io/cluster/${var.name}"      = "shared"
    Tier                                     = "private"
  })
}

# ── Data Subnets ──
resource "aws_subnet" "data" {
  for_each = local.data_subnets

  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value
  availability_zone = each.key

  tags = merge(var.tags, {
    Name = "${var.name}-data-${each.key}"
    Tier = "data"
  })
}

# ── NAT Gateways ──
resource "aws_eip" "nat" {
  for_each = var.enable_nat_gateway ? (
    var.single_nat_gateway ? { (var.azs[0]) = var.azs[0] } : tomap({ for az in var.azs : az => az })
  ) : {}

  domain = "vpc"
  tags   = merge(var.tags, { Name = "${var.name}-nat-eip-${each.key}" })
}

resource "aws_nat_gateway" "this" {
  for_each = aws_eip.nat

  allocation_id = each.value.id
  subnet_id     = aws_subnet.public[each.key].id

  tags = merge(var.tags, { Name = "${var.name}-nat-${each.key}" })

  depends_on = [aws_internet_gateway.this]
}

# ── Route Tables ──
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(var.tags, { Name = "${var.name}-public-rt" })
}

resource "aws_route_table" "private" {
  for_each = toset(var.azs)
  vpc_id   = aws_vpc.this.id

  tags = merge(var.tags, { Name = "${var.name}-private-rt-${each.key}" })
}

resource "aws_route" "private_nat" {
  for_each = var.enable_nat_gateway ? toset(var.azs) : toset([])

  route_table_id         = aws_route_table.private[each.key].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = var.single_nat_gateway ? aws_nat_gateway.this[var.azs[0]].id : aws_nat_gateway.this[each.key].id
}

# ── Route Table Associations ──
resource "aws_route_table_association" "public" {
  for_each       = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private[each.key].id
}

resource "aws_route_table_association" "data" {
  for_each       = aws_subnet.data
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private[each.key].id
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = [for s in aws_subnet.public : s.id]
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = [for s in aws_subnet.private : s.id]
}

output "data_subnet_ids" {
  description = "List of data subnet IDs"
  value       = [for s in aws_subnet.data : s.id]
}

output "private_subnet_map" {
  description = "Map of AZ to private subnet ID"
  value       = { for k, v in aws_subnet.private : k => v.id }
}

output "nat_gateway_ips" {
  description = "NAT Gateway public IPs"
  value       = [for eip in aws_eip.nat : eip.public_ip]
}
```

### Calling the Module

```hcl
# environments/prod/main.tf

module "vpc" {
  source = "../../modules/vpc"

  name               = "novamart-prod"
  cidr               = "10.0.0.0/16"
  azs                = ["us-east-1a", "us-east-1b", "us-east-1c"]
  enable_nat_gateway = true
  single_nat_gateway = false    # HA — one NAT per AZ in prod

  tags = {
    Environment = "prod"
    Project     = "novamart"
    ManagedBy   = "terraform"
  }
}

# Dev uses single NAT to save ~$100/mo
module "vpc" {
  source = "../../modules/vpc"

  name               = "novamart-dev"
  cidr               = "10.1.0.0/16"
  azs                = ["us-east-1a", "us-east-1b"]   # Only 2 AZs in dev
  enable_nat_gateway = true
  single_nat_gateway = true    # Cost savings — not HA, acceptable for dev

  tags = {
    Environment = "dev"
    Project     = "novamart"
    ManagedBy   = "terraform"
  }
}

# Reference outputs
resource "aws_eks_cluster" "main" {
  # ...
  vpc_config {
    subnet_ids = module.vpc.private_subnet_ids
  }
}
```

### Module Sources — Where Modules Come From

```hcl
# Local path (most common for internal modules)
module "vpc" {
  source = "../../modules/vpc"
}

# Terraform Registry (community or verified)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"    # ALWAYS pin version
}

# Git repository (private internal modules)
module "vpc" {
  source = "git::https://bitbucket.org/novamart/terraform-modules.git//vpc?ref=v2.3.0"
  #                                                                   ^module path ^version tag
}

# Git with SSH
module "vpc" {
  source = "git::ssh://git@bitbucket.org/novamart/terraform-modules.git//vpc?ref=v2.3.0"
}

# S3 bucket
module "vpc" {
  source = "s3::https://s3-us-east-1.amazonaws.com/novamart-terraform-modules/vpc/v2.3.0.zip"
}

# VERSIONING RULES:
# Internal modules: use Git tags (v1.0.0, v2.3.0)
# Registry modules: use version constraint (~> 5.5)
# NEVER use branch references (ref=main) in production — same problem as Jenkins @Library('lib') _
```

### Module Design Patterns

```
PATTERN 1: THIN WRAPPER (recommended for NovaMart)
  Module provides: opinionated defaults for your company
  User configures: just the essentials (name, environment, size)
  
  Example: NovaMart EKS module
  ├── Always uses VPC CNI
  ├── Always enables IRSA
  ├── Always creates Karpenter NodePools
  ├── Default node instance types: m5.xlarge (prod), t3.large (dev)
  ├── User provides: cluster name, VPC ID, subnet IDs
  └── 3 required variables, 10 optional with sensible defaults

PATTERN 2: COMPOSITION (modules calling modules)
  module "platform" {
    source = "./modules/platform"      # Platform module
    # Internally calls:
    #   module.vpc
    #   module.eks
    #   module.rds
    #   module.elasticache
    # Wires them together automatically
  }
  
  # Pro: Single module call creates entire environment
  # Con: Less flexibility, harder to debug, tighter coupling

PATTERN 3: FACTORY (generating resources dynamically)
  variable "services" {
    type = map(object({
      cpu    = string
      memory = string
      db     = optional(bool, false)
      queue  = optional(bool, false)
    }))
  }
  
  # Dynamically creates: SQS queue, RDS, IAM role per service
  module "service_infra" {
    source   = "./modules/service-infra"
    for_each = var.services
    
    name   = each.key
    cpu    = each.value.cpu
    memory = each.value.memory
    create_db    = each.value.db
    create_queue = each.value.queue
  }
```

### Module Anti-Patterns

```
ANTI-PATTERN 1: "The God Module"
  One module creates VPC + EKS + RDS + S3 + IAM + everything
  → Blast radius is everything. Can't test components independently.
  → Split by concern: network, compute, data, security.

ANTI-PATTERN 2: "The Wrapper Tax"
  Module that just passes every variable through to one resource
  → If your module has 40 variables and they all map 1:1 to resource attributes,
     you don't need a module. You need a resource.
  → Modules should ADD value: defaults, composition, validation, tagging.

ANTI-PATTERN 3: "Provider in Module"
  Defining provider blocks inside modules
  → Breaks multi-provider patterns, makes module inflexible
  → Providers should be defined in root module, passed to child modules

ANTI-PATTERN 4: "Unpinned Module"
  source = "git::...//vpc?ref=main"
  → Same as Jenkins @Library('lib') _ — breaks randomly when main changes
  → Pin: ref=v2.3.0

ANTI-PATTERN 5: "Module Without Outputs"
  Module creates VPC but doesn't output subnet IDs
  → Downstream modules can't reference the VPC
  → Always output IDs, ARNs, endpoints of created resources
```

---

## 2. Workspaces

### What Workspaces Are

```bash
# Workspaces = multiple state files with the same configuration
# Each workspace has its own state, but shares the same .tf code

terraform workspace list
# * default
#   dev
#   staging
#   prod

terraform workspace new dev
terraform workspace select dev
terraform workspace show
# dev

# In code, reference workspace:
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "m5.xlarge" : "t3.medium"
  tags = {
    Environment = terraform.workspace
  }
}
```

### Workspaces vs Directory Structure — The Great Debate

```
WORKSPACES (terraform workspace):
├── Same code, different state
├── Switch with: terraform workspace select prod
├── State stored at: env:/prod/terraform.tfstate (in S3)
├── Pro: Less code duplication
├── Con: Easy to forget which workspace you're in
├── Con: ALL environments use IDENTICAL code — no env-specific resources
├── Con: "terraform workspace select prod" then "terraform destroy" = disaster
└── Best for: Simple setups where environments are truly identical

DIRECTORY STRUCTURE (environments/dev, environments/prod):
├── Each environment has its own directory
├── Each directory has its own backend config (different state key)
├── Can share modules (DRY) but customize per environment
├── Pro: Impossible to accidentally apply to wrong environment
├── Pro: Each environment can have different resources
├── Pro: PR review shows exactly which environment is changing
├── Con: Some code duplication in root modules
└── Best for: Production environments with any differences between them

NOVAMART CHOICE: Directory structure
├── Dev has 2 AZs, prod has 3
├── Dev has single NAT, prod has per-AZ NAT
├── Staging has a perf-testing cluster, dev doesn't
├── Prod has multi-region, others don't
└── These differences make workspaces impractical
```

---

## 3. Terraform in CI/CD — The Production Pipeline

### The Golden Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│                   Terraform CI/CD Flow                         │
│                                                               │
│  PR Created/Updated                                           │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  1. terraform fmt -check -recursive  (formatting)        │  │
│  │  2. terraform init -backend=false    (syntax validation) │  │
│  │  3. terraform validate               (config validation) │  │
│  │  4. tflint                            (linting)          │  │
│  │  5. checkov / tfsec                   (security scan)    │  │
│  │  6. terraform plan                    (diff preview)     │  │
│  │  7. Post plan output as PR comment    (review)           │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  PR Merged to main                                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  8. terraform init                                       │  │
│  │  9. terraform plan -out=tfplan                           │  │
│  │  10. Manual approval (for prod)                          │  │
│  │  11. terraform apply tfplan                              │  │
│  │  12. Slack notification                                  │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### GitHub Actions Terraform Pipeline

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - 'terraform/environments/prod/**'
  push:
    branches: [main]
    paths:
      - 'terraform/environments/prod/**'

permissions:
  id-token: write
  contents: read
  pull-requests: write    # Post plan as PR comment

env:
  TF_VERSION: "1.7.5"
  WORKING_DIR: "terraform/environments/prod"

jobs:
  # ── VALIDATE (runs on PR) ──
  validate:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Init
        run: terraform init -backend=false
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Validate
        run: terraform validate
        working-directory: ${{ env.WORKING_DIR }}

      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest
      - run: |
          tflint --init
          tflint --recursive
        working-directory: ${{ env.WORKING_DIR }}

      - name: Checkov Security Scan
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: ${{ env.WORKING_DIR }}
          soft_fail: false
          skip_check: CKV_AWS_145    # Known accepted risk

  # ── PLAN (runs on PR) ──
  plan:
    needs: validate
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/TerraformPlanRole
          aws-region: us-east-1
          # NOTE: Plan role has READ-ONLY access to state + resources
          # Apply role has WRITE access — separate roles!

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -no-color -input=false \
            -out=tfplan \
            2>&1 | tee plan_output.txt
          echo "exitcode=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT
        working-directory: ${{ env.WORKING_DIR }}
        continue-on-error: true

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('${{ env.WORKING_DIR }}/plan_output.txt', 'utf8');
            const truncated = plan.length > 60000 
              ? plan.substring(0, 60000) + '\n\n... (truncated)'
              : plan;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `### Terraform Plan (prod)
            \`\`\`
            ${truncated}
            \`\`\`
            
            **Exit code:** ${{ steps.plan.outputs.exitcode }}
            `
            });

      - name: Fail if Plan Failed
        if: steps.plan.outputs.exitcode != '0'
        run: exit 1

  # ── APPLY (runs on merge to main) ──
  apply:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: production    # Requires approval + OIDC with environment claim
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/TerraformApplyRole
          aws-region: us-east-1
          # Apply role has WRITE access — separate from plan role

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Plan
        run: terraform plan -out=tfplan -input=false
        working-directory: ${{ env.WORKING_DIR }}

      - name: Terraform Apply
        run: terraform apply -input=false tfplan
        working-directory: ${{ env.WORKING_DIR }}

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && '✅' || '❌' }} Terraform apply to prod: ${{ job.status }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Separation of Plan and Apply IAM Roles

```
TerraformPlanRole:
├── S3: GetObject on state bucket (read state)
├── DynamoDB: GetItem, PutItem on lock table (acquire lock for plan)
├── AWS: Read-only on all services (describe, list, get)
│   ├── ec2:Describe*
│   ├── eks:Describe*, List*
│   ├── rds:Describe*
│   └── etc.
├── WHO: PR pipeline (any contributor can trigger)
└── RISK: Low — can't change anything

TerraformApplyRole:
├── S3: Get/PutObject on state bucket (read + write state)
├── DynamoDB: Get/PutItem/DeleteItem on lock table
├── AWS: Full write on managed services
│   ├── ec2:*, eks:*, rds:*, s3:*, iam:*
│   └── SCOPED to specific resources via conditions
├── WHO: Merge to main only (requires PR approval)
└── RISK: High — can create/destroy infrastructure

WHY SEPARATE:
├── Principle of least privilege
├── PR pipeline runs on every push — hundreds of times/day
├── Apply runs only on merge — a few times/day with approval
└── If PR pipeline is compromised, attacker gets read-only
```

### Drift Detection Pipeline

```yaml
# .github/workflows/drift-detection.yml
name: Terraform Drift Detection

on:
  schedule:
    - cron: '0 6 * * *'    # Daily at 6 AM UTC

jobs:
  drift-check:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        layer: [network, security, platform, data]
        environment: [prod]
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/TerraformPlanRole
          aws-region: us-east-1

      - name: Check for Drift
        id: drift
        working-directory: terraform/${{ matrix.layer }}
        run: |
          terraform init
          terraform plan -detailed-exitcode -input=false 2>&1 | tee drift.txt
          # Exit codes: 0=no changes, 1=error, 2=changes detected
          echo "exitcode=$?" >> $GITHUB_OUTPUT

      - name: Alert on Drift
        if: steps.drift.outputs.exitcode == '2'
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-Type: application/json' \
            -d '{
              "text": "⚠️ Terraform drift detected in ${{ matrix.layer }} (${{ matrix.environment }})\nSomeone made manual changes. Review and reconcile.",
              "attachments": [{
                "color": "warning",
                "text": "Run `terraform plan` in ${{ matrix.layer }} to see details."
              }]
            }'
```

---

## 4. Advanced Module Patterns

### Module Composition — NovaMart EKS Module

```hcl
# modules/eks/main.tf
resource "aws_eks_cluster" "this" {
  name     = var.cluster_name
  version  = var.kubernetes_version
  role_arn = aws_iam_role.cluster.arn

  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = var.endpoint_public_access
    security_group_ids      = [aws_security_group.cluster.id]
  }

  encryption_config {
    provider {
      key_arn = var.kms_key_arn
    }
    resources = ["secrets"]
  }

  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy,
    aws_iam_role_policy_attachment.cluster_AmazonEKSVPCResourceController,
  ]

  tags = var.tags
}

# ── EKS Addons ──
resource "aws_eks_addon" "this" {
  for_each = var.cluster_addons

  cluster_name             = aws_eks_cluster.this.name
  addon_name               = each.key
  addon_version            = each.value.version
  resolve_conflicts_on_update = "OVERWRITE"
  service_account_role_arn = try(each.value.service_account_role_arn, null)

  tags = var.tags
}

# ── OIDC Provider for IRSA ──
data "tls_certificate" "eks" {
  url = aws_eks_cluster.this.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.this.identity[0].oidc[0].issuer

  tags = var.tags
}
```

```hcl
# environments/prod/main.tf — Calling the module

module "eks" {
  source = "../../modules/eks"

  cluster_name       = "novamart-prod"
  kubernetes_version = "1.29"
  subnet_ids         = module.vpc.private_subnet_ids
  kms_key_arn        = module.kms.eks_key_arn

  endpoint_public_access = false    # Private-only in prod

  cluster_addons = {
    vpc-cni = {
      version                  = "v1.16.0-eksbuild.1"
      service_account_role_arn = module.irsa_vpc_cni.role_arn
    }
    coredns = {
      version = "v1.11.1-eksbuild.6"
    }
    kube-proxy = {
      version = "v1.29.0-eksbuild.1"
    }
    aws-ebs-csi-driver = {
      version                  = "v1.28.0-eksbuild.1"
      service_account_role_arn = module.irsa_ebs_csi.role_arn
    }
  }

  tags = local.common_tags
}

# IRSA module for EBS CSI driver
module "irsa_ebs_csi" {
  source = "../../modules/irsa"

  role_name          = "novamart-prod-ebs-csi"
  oidc_provider_arn  = module.eks.oidc_provider_arn
  oidc_provider_url  = module.eks.oidc_provider_url
  namespace          = "kube-system"
  service_account    = "ebs-csi-controller-sa"
  policy_arns        = ["arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"]
}
```

### terraform-docs — Auto-Generated Documentation

```bash
# Install: brew install terraform-docs

# Generate README from module code
terraform-docs markdown table ./modules/vpc > ./modules/vpc/README.md

# Output:
# ## Inputs
# | Name | Description | Type | Default | Required |
# |------|-------------|------|---------|:--------:|
# | name | Name prefix | string | n/a | yes |
# | cidr | VPC CIDR | string | n/a | yes |
# ...
# ## Outputs
# | Name | Description |
# |------|-------------|
# | vpc_id | VPC ID |
# ...

# Pre-commit hook to auto-update
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/terraform-docs/terraform-docs
    rev: v0.17.0
    hooks:
      - id: terraform-docs-go
        args: ["markdown", "table", "--output-file", "README.md"]
```

---

## 5. State Management Patterns at Scale

### Terraform Remote State Data Source

```hcl
# In the EKS layer, read VPC outputs
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "novamart-terraform-state"
    key    = "network/vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use the outputs
module "eks" {
  source     = "../../modules/eks"
  subnet_ids = data.terraform_remote_state.vpc.outputs.private_subnet_ids
  vpc_id     = data.terraform_remote_state.vpc.outputs.vpc_id
}

# ALTERNATIVE: Use SSM Parameter Store as the intermediary
# VPC layer writes:
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/novamart/prod/vpc/id"
  type  = "String"
  value = module.vpc.vpc_id
}

# EKS layer reads:
data "aws_ssm_parameter" "vpc_id" {
  name = "/novamart/prod/vpc/id"
}

# SSM ADVANTAGES over terraform_remote_state:
# 1. No direct dependency on state file location
# 2. Other tools (Ansible, scripts) can read SSM too
# 3. State file restructuring doesn't break consumers
# 4. Fine-grained IAM access (read only specific params)
```

### How Modules Break

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **Module version mismatch** | Different environments on different versions | Unpinned module source | Pin: `?ref=v2.3.0` in source |
| **Provider version conflict** | `Error: inconsistent dependency lock file` | Module requires provider version that conflicts with root | `required_providers` in module's versions.tf, align versions |
| **Circular dependency between layers** | `terraform plan` errors or infinite loops | EKS layer reads VPC state, VPC layer reads EKS state | One-directional dependencies only. VPC → EKS, never reverse. |
| **terraform_remote_state output missing** | `Error: Unsupported attribute` | VPC layer removed an output that EKS layer depends on | Treat module outputs like API contracts — never remove, deprecate first |
| **Module apply ordering** | EKS apply fails because VPC not yet created | CI applies all layers in parallel | Sequential pipeline: VPC → security → EKS → data → services |
| **State file too large** | Plan takes 10+ minutes | One state managing 500+ resources | Split by service or layer, target specific resources |
| **Provider in module** | Can't use module with different AWS accounts | `provider` block inside module | Pass provider from root via `providers` argument |


## Quick Reference Card

```
MODULES:
  Structure: main.tf, variables.tf, outputs.tf, versions.tf
  Source: local path, git (pin ref=vX.Y.Z), registry (pin version)
  Call: module "name" { source = "...", var = value }
  Reference: module.name.output_name
  
MODULE RULES:
  No provider blocks inside modules (pass from root)
  Always output IDs, ARNs, endpoints
  Pin versions (git tags or registry version constraint)
  Treat outputs as API contracts — don't break consumers
  
WORKSPACES vs DIRECTORIES:
  Workspaces: same code, different state (simple, dangerous)
  Directories: different code per env, shared modules (safe, recommended)
  NovaMart: directories (environments differ significantly)

CI/CD PIPELINE:
  PR: fmt → init → validate → tflint → checkov → plan → PR comment
  Merge: init → plan -out=tfplan → approval → apply tfplan → notify
  
IAM SEPARATION:
  PlanRole: read-only (state + AWS describe)
  ApplyRole: read-write (state + AWS full)
  Plan runs on every PR push. Apply runs on merge only.

DRIFT DETECTION:
  Daily cron: terraform plan -detailed-exitcode
  Exit 0=clean, 2=drift detected → alert Slack
  
STATE CROSS-REFERENCE:
  terraform_remote_state → direct state dependency
  SSM Parameter Store → decoupled, multi-tool friendly
  
MODULE SOURCES:
  Local: source = "../../modules/vpc"
  Git:   source = "git::https://...//vpc?ref=v2.3.0"
  Reg:   source = "terraform-aws-modules/vpc/aws" version="5.5.0"
```



## Retention Questions

**Q1.** NovaMart's platform team has written a VPC module, an EKS module, and an RDS module. Currently, each team that needs infrastructure writes their own root module calling these. The result: 15 different root modules, all slightly different, some with bugs. Design a "platform module" that composes VPC + EKS + RDS into a single call, while still allowing per-team customization. Show the module interface (variables), how it wires the internal modules together, and how a consuming team uses it. Address the tradeoff of convenience vs flexibility.

**Q2.** Write the complete GitHub Actions workflow for NovaMart's Terraform CI/CD, but this time handle the MONOREPO case: the repo has `terraform/network/`, `terraform/eks/`, `terraform/rds/`, and `terraform/services/*`. On a PR, only run plan for layers that changed. On merge, apply layers in dependency order (network → security → eks → data). Show path-based trigger filtering, dependency ordering, and how you prevent applying `eks` before `network` completes.

**Q3.** You discover that NovaMart's Terraform state for the production EKS cluster contains the RDS master password in plaintext (because it was set via `password = var.db_password` in the `aws_db_instance` resource). The security team demands remediation. How do you remove the password from state without destroying the database? What's the long-term fix to prevent secrets from ending up in state? Show exact commands and configuration changes.

**Q4.** Explain the difference between `terraform plan -detailed-exitcode`, `terraform plan -refresh-only`, and `terraform plan -target=RESOURCE`. For each, give a specific NovaMart production scenario where you'd use it, explain the risks, and show the exact command. Why is `-target` considered dangerous and when is it acceptable?

This is the transition from "writing Terraform" to "building a Platform." At NovaMart's scale, you cannot let individual teams manage the raw infrastructure. You must provide **Golden Paths**—standardized, pre-approved templates that embody the company's security and reliability standards, while leaving just enough "knobs" for developers to turn.

---

### Q1: The "Platform Module" (Composition Pattern)

I will implement a **Wrapper Module**. This module doesn't contain resources itself; it orchestrates the three sub-modules (VPC, EKS, RDS) into a single, cohesive stack.

#### 1. The Module Interface (`variables.tf`)
The goal is to provide sensible defaults for 90% of users but allow overrides for the 10% who need a bigger DB or more nodes.

```hcl
variable "environment" {
  type        = string
  description = "env name (e.g., prod, staging)"
}

variable "vpc_config" {
  type = object({
    cidr = string
    azs  = list(string)
  })
  default = { cidr = "10.0.0.0/16", azs = ["us-east-1a", "us-east-1b", "us-east-1c"] }
}

variable "eks_config" {
  type = object({
    node_type     = string
    min_nodes     = number
    max_nodes     = number
    desired_nodes = number
  })
  default = { node_type = "m5.large", min_nodes = 3, max_nodes = 10, desired_nodes = 3 }
}

variable "rds_config" {
  type = object({
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  default = { instance_class = "db.t3.medium", storage_gb = 20, multi_az = true }
}
```

#### 2. Wiring the Modules (`main.tf`)
The Wrapper Module handles the "plumbing"—passing the output of the VPC module into the EKS and RDS modules.

```hcl
module "vpc" {
  source = "../modules/vpc"
  cidr   = var.vpc_config.cidr
  azs    = var.vpc_config.azs
}

module "eks" {
  source     = "../modules/eks"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  node_type  = var.eks_config.node_type
  # ... other config
}

module "rds" {
  source     = "../modules/rds"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.data_subnets
  instance_class = var.rds_config.instance_class
  # ... other config
}
```

#### 3. Consuming the Module (Root Module)
A team now only needs to call one module.

```hcl
module "platform_stack" {
  source      = "git::https://github.com/novamart/terraform-modules.git//platform"
  environment = "prod"
  
  # Override only what is necessary
  rds_config = {
    instance_class = "db.r5.large" # Heavy DB for this team
    storage_gb     = 500
    multi_az       = true
  }
}
```

**The Trade-off: Convenience vs. Flexibility**
- **Convenience:** The "Time to First Pod" drops from days to minutes. Security is guaranteed because the `vpc` and `eks` modules are locked down.
- **Flexibility:** We lose "granular control." If a team needs a very specific networking quirk (e.g., a custom Transit Gateway attachment), they cannot do it through the wrapper.
- **Resolution:** I provide an "escape hatch." The wrapper module exposes the raw resources as outputs, allowing teams to add their own small resource blocks in their root module if absolutely necessary.

---

### Q2: Monorepo CI/CD with Dependency Ordering

The challenge here is to avoid "Plan Everything" while ensuring that `network` is applied before `eks`.

**The GitHub Actions Workflow (`.github/workflows/terraform.yml`):**

```yaml
name: Terraform CI/CD
on:
  pull_request:
    paths:
      - 'terraform/**'
  push:
    branches: [ main ]
    paths:
      - 'terraform/**'

jobs:
  # 1. Detection: Find which layers changed
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      layers: ${{ steps.filter.outputs.changes }}
    steps:
      - uses: actions/checkout@v3
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            network: 'terraform/network/**'
            security: 'terraform/security/**'
            eks: 'terraform/eks/**'
            data: 'terraform/rds/**'

  # 2. Plan: Run in parallel for changed layers
  plan:
    needs: detect-changes
    if: ${{ needs.detect-changes.outputs.layers != '[]' }}
    strategy:
      matrix:
        layer: ${{ fromJSON(needs.detect-changes.outputs.layers) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Terraform Plan
        run: |
          cd terraform/${{ matrix.layer }}
          terraform init
          terraform plan -out=tfplan

  # 3. Apply: Sequential execution based on hard-coded dependency order
  apply:
    needs: [detect-changes, plan]
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Apply Layers in Order
        run: |
          # Order is strictly defined here
          for layer in network security eks data; do
            if [[ "${{ needs.detect-changes.outputs.layers }}" == *"$layer"* ]]; then
              echo "Applying $layer..."
              cd terraform/$layer && terraform apply -auto-approve && cd -
            fi
          done
```

**Key Logic Explained:**
- **`paths-filter`**: Ensures we only run the plan for the actual folders changed.
- **Matrix Strategy**: The `plan` stage runs in parallel for speed.
- **The `for` loop in Apply**: This is the "SRE Guardrail." Even if `data` and `network` both changed, the loop forces the `network` apply to finish before the `eks` apply begins. This prevents the "VPC not found" error during EKS creation.

---

### Q3: Secret Leakage in Terraform State

**The Problem:** Terraform state is a JSON file that stores every attribute of every resource. `aws_db_instance` stores the `password` attribute in plaintext. Even if you use a variable, the *resulting* value is written to the `.tfstate` file.

**Immediate Remediation (Removing the secret):**
You cannot simply edit the JSON file manually (you'll break the state hash). You must change the value in the real world and then update the state.

1.  **Rotate the password in AWS:** Change the RDS password via the AWS Console or CLI.
2.  **Update the Variable:** Update the variable in your secret manager/`.tfvars` to the new password.
3.  **Force an Update:**
    `terraform apply -target=aws_db_instance.this`
    Terraform will see the difference between the state (old password) and the variable (new password), update the state to the new value, and the "leak" of the *old* password is gone (though it's still in the state history/backups).

**Long-Term Fix: The "Zero-Secret" Architecture**
Stop passing passwords into Terraform. Use **AWS Secrets Manager** and **Dynamic Referencing**.

**Correct Configuration:**
```hcl
# 1. Create a secret shell (no value)
resource "aws_secretsmanager_secret" "db_password" {
  name = "prod/db/password"
}

# 2. Tell RDS to manage the password itself (Rotation)
resource "aws_db_instance" "this" {
  # REMOVE: password = var.db_password
  manage_master_user_password = true # AWS generates and rotates the password
  # ... other config
}
```
By using `manage_master_user_password = true`, the password is never handled by Terraform. It is generated by AWS and stored internally in Secrets Manager. The `.tfstate` file will now only contain a reference to the secret, not the password itself.

---

### Q4: The `terraform plan` Toolbox

| Flag | What it does | NovaMart Scenario | Risk | Exact Command |
| :--- | :--- | :--- | :--- | :--- |
| **`-detailed-exitcode`** | Returns `0` (no change), `1` (change), or `2` (error). | **CI Automation.** Used in a script to decide if a "Deploy" stage should be triggered. | None. | `terraform plan -detailed-exitcode` |
| **`-refresh-only`** | Updates the state file to match AWS, but makes **no** changes to infra. | **Drift Detection.** Someone manually changed a Security Group rule in the Console. You want to sync the state without changing the rule. | Can overwrite local changes if you apply the refresh. | `terraform plan -refresh-only` |
| **`-target`** | Plans/Applies only a specific resource, ignoring everything else. | **Emergency Fix.** A critical bug in one SG rule needs a fix NOW, but the rest of the state has 100 failing changes you can't fix today. | **State Inconsistency.** It ignores dependencies. You might update a DB but forget to update the SG that allows the app to talk to it. | `terraform apply -target=aws_security_group.api_sg` |

**Why is `-target` dangerous?**
Terraform is a **declarative** tool; it's designed to manage the *entire* graph of resources. When you use `-target`, you are treating it as an **imperative** tool. You are telling Terraform: "Ignore the graph, just do this one thing." 

This often leads to "Ghost Changes." You fix the SG today using `-target`, but the next time a colleague runs a full `terraform apply`, Terraform will see that the SG was changed manually (or via target) and might try to revert it or conflict with it.

**When is it acceptable?**
Only during **Emergency Recovery** (the "Break-Glass" scenario). If the cluster is down and a full apply takes 20 minutes but a targeted apply takes 30 seconds, you use `-target` to restore service, and then immediately follow up with a full `terraform apply` to reconcile the rest of the state.


# Grading Q1-Q4

---

## Q1: Platform Module (Composition) — 4/5

**What you nailed:**
- Wrapper module pattern — correct architecture choice.
- Variable interface with `object` types and sensible defaults — clean API design.
- Internal wiring (VPC outputs → EKS/RDS inputs) — correct composition.
- "Escape hatch" concept for teams that need more control — mature platform thinking.
- "Time to First Pod" as a metric — that's how platform teams measure success.

**What you missed:**

1. **Your module doesn't enforce security invariants.** The whole point of a platform module is that teams CAN'T turn off the security controls. Your interface exposes `multi_az` as a configurable knob — what happens when a team sets `multi_az = false` in prod to "save costs"? The platform module should enforce:

```hcl
# Inside the platform module — not configurable by consumers
locals {
  # Enforce HA in prod, allow single-AZ in dev
  enforce_multi_az  = var.environment == "prod" ? true : var.rds_config.multi_az
  enforce_encryption = true  # Always — not a knob
  enforce_private_eks = var.environment == "prod" ? true : false
  
  # Minimum node count per environment
  min_nodes = var.environment == "prod" ? max(var.eks_config.min_nodes, 3) : var.eks_config.min_nodes
}

module "rds" {
  source   = "../modules/rds"
  multi_az = local.enforce_multi_az    # Overrides user input in prod
  storage_encrypted = local.enforce_encryption
  # ...
}
```

Without enforcement, the platform module is just a convenience wrapper, not a governance tool.

2. **Missing: validation blocks on the platform variables.** If someone passes `instance_class = "db.t2.micro"` for prod, that should fail at plan time, not at 3 AM:

```hcl
variable "rds_config" {
  type = object({
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  
  validation {
    condition     = !startswith(var.rds_config.instance_class, "db.t2")
    error_message = "db.t2 instances are not allowed. Use db.t3 or higher."
  }
  
  validation {
    condition     = var.rds_config.storage_gb >= 20
    error_message = "Minimum storage is 20GB."
  }
}
```

3. **Missing: outputs from the platform module.** Your consuming team needs to reference the EKS endpoint, the RDS endpoint, the VPC ID, etc. You showed the wiring but not what the platform module exposes back:

```hcl
# modules/platform/outputs.tf
output "eks_cluster_endpoint" {
  value = module.eks.cluster_endpoint
}
output "eks_cluster_name" {
  value = module.eks.cluster_name
}
output "rds_endpoint" {
  value     = module.rds.endpoint
  sensitive = true
}
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

4. **"Escape hatch" needs more specifics.** "Expose raw resources as outputs" is vague. The production pattern is to output IDs/ARNs and let teams create additional resources that reference them:

```hcl
# Team's root module
module "platform_stack" {
  source = "..."
}

# Team's custom addition — uses platform outputs
resource "aws_sqs_queue" "orders" {
  name = "order-queue"
  # Can reference platform outputs for VPC endpoints, etc.
}
```

This is cleaner than trying to make the platform module do everything.

---

## Q2: Monorepo CI/CD — 3.5/5

**What you nailed:**
- `dorny/paths-filter` for change detection — correct tool.
- Matrix strategy for parallel planning — correct.
- Hard-coded dependency order in apply — correct approach for Terraform (you can't infer it automatically).

**What you got wrong:**

1. **The `for` loop in apply has several problems.** First, `terraform apply -auto-approve` without a saved plan file means it recalculates the plan at apply time. If state changed between the plan job and the apply job, you're applying something nobody reviewed:

```bash
# WRONG: Recalculates plan at apply time
terraform apply -auto-approve

# RIGHT: Apply the exact plan that was reviewed
terraform apply tfplan
```

But now you have a problem: the plan was created in a separate job (the `plan` matrix job). You need to pass the plan file as an artifact:

```yaml
plan:
  steps:
    - name: Plan
      run: terraform plan -out=tfplan
    - uses: actions/upload-artifact@v4
      with:
        name: tfplan-${{ matrix.layer }}
        path: terraform/${{ matrix.layer }}/tfplan

apply-network:
  needs: plan
  if: contains(needs.detect-changes.outputs.layers, 'network')
  steps:
    - uses: actions/download-artifact@v4
      with:
        name: tfplan-network
    - run: terraform apply tfplan
```

2. **Your dependency ordering doesn't actually work as separate jobs with proper gates.** A `for` loop in a single job runs sequentially, but if `network` apply fails, the loop continues to `security`. You need separate jobs with `needs:`:

```yaml
apply-network:
  needs: [detect-changes, plan]
  if: github.event_name == 'push' && contains(...)
  steps:
    - run: terraform apply tfplan

apply-security:
  needs: [apply-network]    # Waits for network to complete
  if: github.event_name == 'push' && contains(...)
  steps:
    - run: terraform apply tfplan

apply-eks:
  needs: [apply-security]   # Waits for security
  steps:
    - run: terraform apply tfplan

apply-data:
  needs: [apply-eks]        # Waits for eks
  steps:
    - run: terraform apply tfplan
```

This gives you: visual dependency graph in GitHub UI, automatic skip if upstream fails, proper logging per layer, and the ability to re-run individual layers.

3. **The `paths-filter` output format is wrong.** `dorny/paths-filter` outputs individual boolean values per filter, not a JSON array. Your `fromJSON(needs.detect-changes.outputs.layers)` won't work as written. The correct pattern:

```yaml
detect-changes:
  outputs:
    network: ${{ steps.filter.outputs.network }}
    security: ${{ steps.filter.outputs.security }}
    eks: ${{ steps.filter.outputs.eks }}
    data: ${{ steps.filter.outputs.data }}
  steps:
    - uses: dorny/paths-filter@v2
      id: filter
      with:
        filters: |
          network: 'terraform/network/**'
          security: 'terraform/security/**'

# Then in downstream jobs:
apply-network:
  if: needs.detect-changes.outputs.network == 'true'
```

4. **Missing: what happens when `network` changes but `eks` doesn't.** If you change a subnet tag in `network/`, the EKS layer might need to be re-planned because it reads VPC outputs. But your paths-filter says "eks didn't change" so you skip it. The safe approach: if a upstream layer changed, always re-plan downstream layers:

```yaml
apply-eks:
  if: |
    needs.detect-changes.outputs.eks == 'true' || 
    needs.detect-changes.outputs.network == 'true' ||
    needs.detect-changes.outputs.security == 'true'
```

5. **Missing AWS credentials and init.** Each job needs OIDC setup and `terraform init` with the correct backend. Your apply step just runs `terraform apply` without initializing.

---

## Q3: Secret in State — 4.5/5

**Strongest answer this round.**

**What you nailed:**
- Correct understanding that Terraform stores ALL attributes in state, including secrets.
- `manage_master_user_password = true` — this is the correct modern solution. Excellent. Most engineers don't know this exists. This removes the password from Terraform's control entirely.
- Understanding that you can't manually edit the state JSON.
- Long-term "zero-secret architecture" framing is exactly right.

**What you missed:**

1. **Your immediate remediation is incomplete and partially wrong.** You said "use `-target` to force an update." But the problem isn't just the current password in state — it's the password in **state history**. S3 versioning means every previous version of the state file also contains the old password. The full remediation:

```bash
# Step 1: Rotate the password IMMEDIATELY (AWS Console or CLI)
aws rds modify-db-instance \
    --db-instance-identifier novamart-prod-db \
    --master-user-password "NEW_SECURE_PASSWORD" \
    --apply-immediately

# Step 2: Update all applications using the old password
# (connection strings, secrets in Vault/SecretsManager)

# Step 3: Update Terraform config to use manage_master_user_password
# Remove: password = var.db_password
# Add:    manage_master_user_password = true

# Step 4: Apply (this may require import or careful planning)
terraform plan   # Review what changes
terraform apply  # Apply the new config

# Step 5: Address state HISTORY
# Old passwords are in previous S3 versions of the state file
# Options:
#   a) Delete old versions (loses rollback ability)
#   b) Accept risk (state bucket is already encrypted + access-controlled)
#   c) Recreate the state (terraform state pull, sanitize, push — VERY risky)

# Step 6: Audit who/what has read access to the state bucket
# Any IAM principal with s3:GetObject on the state bucket
# has seen the plaintext password
```

2. **You didn't mention the broader pattern for other secrets.** RDS has `manage_master_user_password`, but what about API keys, TLS private keys, or other secrets that Terraform manages? The general pattern:

```hcl
# Generate secrets OUTSIDE Terraform, reference them
data "aws_secretsmanager_secret_version" "api_key" {
  secret_id = "prod/api/key"
}

# Use in resources
resource "aws_lambda_function" "api" {
  environment {
    variables = {
      API_KEY = data.aws_secretsmanager_secret_version.api_key.secret_string
    }
  }
}
# WARNING: Even data source values end up in state!
# The secret_string is stored in state.

# TRUE zero-state-secret pattern:
# Don't reference secrets in Terraform AT ALL
# Use External Secrets Operator (K8s) or 
# application-level Vault integration
# Terraform creates the Secrets Manager secret SHELL
# A separate process (Lambda rotation) populates the value
```

3. **Missing: `sensitive = true` on variables and outputs.** This doesn't prevent storage in state, but it prevents display in plan/apply output and in logs. It's defense-in-depth:

```hcl
variable "db_password" {
  type      = string
  sensitive = true    # Masked in CLI output
}

output "rds_endpoint" {
  value     = aws_db_instance.this.endpoint
  sensitive = true
}
```

---

## Q4: `plan` Flag Comparison — 4.2/5

**What you nailed:**
- All three flags correctly explained.
- `-detailed-exitcode` for CI automation — correct use case.
- `-refresh-only` for drift detection — correct.
- `-target` danger explained well: "treating declarative tool as imperative."
- "Ghost Changes" concept for target — accurate.
- Break-glass scenario for when `-target` is acceptable — correct.

**What you missed:**

1. **Exit codes for `-detailed-exitcode` are wrong.** You said: 0=no change, 1=change, 2=error. It's actually:
   - **0** = no changes
   - **1** = error
   - **2** = changes detected (success, but there are diffs)

This matters because in CI scripts, you'd check `if [ $? -eq 2 ]` to detect drift. If you swap 1 and 2, your drift detection alerts on errors instead of actual drift, and misses actual drift entirely.

2. **Missing: `-refresh=false` which pairs with `-target` in emergencies.** When you need the fastest possible apply in an emergency:

```bash
# Skip refresh (don't check every resource against AWS API)
# + target only the specific resource
# This turns a 5-minute plan into a 5-second plan
terraform apply -refresh=false -target=aws_security_group_rule.emergency_fix
```

The risk: if state is stale, you might overwrite a manual change. But in an emergency, speed matters.

3. **Your `-refresh-only` scenario needs the apply step.** You described it as "sync the state" but didn't complete the workflow:

```bash
# Step 1: Plan refresh-only (shows what state would change)
terraform plan -refresh-only

# Step 2: Review the proposed state changes
# "Is this drift acceptable or should we revert?"

# Step 3: If acceptable, apply the refresh
terraform apply -refresh-only

# Step 4: If NOT acceptable (someone broke something manually)
# Don't refresh — instead, run normal terraform apply
# to revert the manual change back to what's in code
terraform apply
```

4. **Missing a critical `-target` risk: dependency skipping.** The real danger isn't just "ghost changes." It's that `-target` skips dependency evaluation. Example:

```bash
# You target an RDS instance
terraform apply -target=aws_db_instance.main

# But RDS depends on a security group that also changed
# Terraform SKIPS the security group update
# Result: DB is updated but firewall rules are wrong
# New app version can't connect → outage
```

---

## Score Summary

| Question | Score | Key Gap |
|----------|-------|---------|
| Q1 | 4.0/5 | No security enforcement in module, no validation blocks, no outputs |
| Q2 | 3.5/5 | Apply without saved plan, for loop doesn't handle failures, paths-filter syntax wrong |
| Q3 | 4.5/5 | Excellent — `manage_master_user_password` is the correct modern answer. Minor: state history, broader secret pattern |
| Q4 | 4.2/5 | Exit codes wrong (1 vs 2 swapped), missing `-refresh=false` combo |
| **Average** | **4.05/5** | Consistent improvement. Q3 was excellent. Q2 pipeline implementation still shows the "concept vs execution" gap. |

**The trend:** Your architecture-level thinking is strong and improving (Q1's wrapper module, Q3's zero-secret architecture). Your CI/CD pipeline implementations still have syntax and workflow issues (Q2). This is because Terraform CI/CD has more moving parts than most people realize — artifact passing, conditional job execution, state locking coordination. The fix: build one of these pipelines for real and iterate through the failures.

---

Proceeding to **Phase 4, Lesson 3.**

---

# Phase 4: Infrastructure as Code
## Lesson 3: Terraform Advanced — Testing, Terraform Cloud, Refactoring at Scale

---

## 1. Terraform Testing

### Why Test Infrastructure Code?

```
WITHOUT TESTING:
  Engineer writes module → "terraform plan looks good" → merges
  Three months later: someone calls the module with 
  unexpected input → creates a public S3 bucket in prod
  
  Or: provider upgrade from 5.40 to 5.50 → module breaks
  → discovered when applying to prod → rollback under pressure

WITH TESTING:
  Unit tests catch: invalid variable combinations, 
    missing validations, logic errors in locals
  Integration tests catch: provider version incompatibilities,
    actual resource creation failures, cross-resource wiring issues
  Policy tests catch: security violations before apply
```

### Level 1: terraform validate + fmt

```bash
# The bare minimum — should be in every CI pipeline

# Format check (no changes, just verify)
terraform fmt -check -recursive -diff
# -diff shows what would change

# Validate syntax and internal consistency
terraform init -backend=false    # Don't need real backend for validation
terraform validate
# Checks: syntax errors, invalid references, type mismatches
# Does NOT check: whether resources are valid in AWS (no API calls)
```

### Level 2: TFLint — Linting

```hcl
# .tflint.hcl
config {
  module = true    # Also lint called modules
}

plugin "aws" {
  enabled = true
  version = "0.30.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

# Custom rules
rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_unused_declarations" {
  enabled = true
}

rule "terraform_documented_variables" {
  enabled = true
}

# AWS-specific rules (catches things validate can't)
# - Invalid instance types ("m5.xlarge" exists, "m5.huuuge" doesn't)
# - Invalid AMI patterns
# - Deprecated resource types
```

```bash
tflint --init
tflint --recursive
# Catches: typos in instance types, unused variables, naming violations
```

### Level 3: Checkov / tfsec — Security Policy Scanning

```bash
# Checkov scans Terraform for security misconfigurations
checkov -d . --framework terraform

# What it catches:
# CKV_AWS_145: S3 bucket not encrypted with KMS
# CKV_AWS_18:  S3 bucket doesn't have access logging
# CKV_AWS_19:  S3 bucket not encrypted at rest
# CKV_AWS_23:  Security group allows ingress 0.0.0.0/0
# CKV_AWS_79:  Instance metadata service v2 not enforced
# CKV_AWS_88:  EC2 instance has public IP
# CKV_AWS_126: RDS not encrypted at rest
# CKV_AWS_157: RDS doesn't have multi-AZ enabled

# Skip specific checks (with justification)
checkov -d . --skip-check CKV_AWS_145  # "We use SSE-S3, not KMS"

# Or inline skip:
resource "aws_s3_bucket" "logs" {
  #checkov:skip=CKV_AWS_145:Using SSE-S3 encryption, KMS not required for logs
  bucket = "novamart-access-logs"
}

# Custom policies (Python or YAML)
# Check that all resources have required tags
# policies/required_tags.yaml
metadata:
  name: "Ensure required tags"
  id: "CUSTOM_001"
  category: "CONVENTION"
definition:
  cond_type: "attribute"
  resource_types: ["aws_instance", "aws_s3_bucket", "aws_rds_cluster"]
  attribute: "tags.Environment"
  operator: "exists"
```

### Level 4: terraform test (Native, Terraform 1.6+)

```hcl
# tests/vpc_test.tftest.hcl

# Variables for the test
variables {
  name = "test-vpc"
  cidr = "10.99.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
  enable_nat_gateway = true
  single_nat_gateway = true
  tags = { Environment = "test" }
}

# Test 1: VPC is created with correct CIDR
run "creates_vpc" {
  command = plan    # Just plan, don't create real resources

  assert {
    condition     = aws_vpc.this.cidr_block == "10.99.0.0/16"
    error_message = "VPC CIDR doesn't match input"
  }

  assert {
    condition     = aws_vpc.this.enable_dns_support == true
    error_message = "DNS support should be enabled"
  }

  assert {
    condition     = aws_vpc.this.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }
}

# Test 2: Correct number of subnets per type
run "creates_correct_subnets" {
  command = plan

  assert {
    condition     = length(aws_subnet.public) == 2
    error_message = "Expected 2 public subnets for 2 AZs"
  }

  assert {
    condition     = length(aws_subnet.private) == 2
    error_message = "Expected 2 private subnets for 2 AZs"
  }

  assert {
    condition     = length(aws_subnet.data) == 2
    error_message = "Expected 2 data subnets for 2 AZs"
  }
}

# Test 3: Single NAT gateway when configured
run "single_nat_gateway" {
  command = plan

  assert {
    condition     = length(aws_nat_gateway.this) == 1
    error_message = "Expected single NAT gateway"
  }
}

# Test 4: EKS tags on subnets
run "eks_subnet_tags" {
  command = plan

  assert {
    condition     = aws_subnet.public["us-east-1a"].tags["kubernetes.io/role/elb"] == "1"
    error_message = "Public subnet missing EKS ELB tag"
  }

  assert {
    condition     = aws_subnet.private["us-east-1a"].tags["kubernetes.io/role/internal-elb"] == "1"
    error_message = "Private subnet missing EKS internal-elb tag"
  }
}

# Test 5: Validation rejects invalid input
run "rejects_single_az" {
  command = plan

  variables {
    azs = ["us-east-1a"]    # Only 1 AZ — should fail validation
  }

  expect_failures = [
    var.azs    # Expect the validation on var.azs to fail
  ]
}
```

```bash
# Run tests
terraform test
# Uses real provider but command=plan doesn't create resources
# command=apply actually creates and destroys resources (integration test)

# Run specific test file
terraform test -filter=tests/vpc_test.tftest.hcl
```

### Level 5: Terratest (Go, Integration Tests)

```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestVPCModule(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "name": "terratest-vpc",
            "cidr": "10.99.0.0/16",
            "azs":  []string{"us-east-1a", "us-east-1b"},
            "enable_nat_gateway": true,
            "single_nat_gateway": true,
            "tags": map[string]string{
                "Environment": "test",
            },
        },
        // Don't use shared state — isolated per test
        BackendConfig: map[string]interface{}{
            "bucket": "novamart-terratest-state",
            "key":    t.Name() + "/terraform.tfstate",
            "region": "us-east-1",
        },
    }

    // Clean up after test
    defer terraform.Destroy(t, terraformOptions)

    // Create real resources
    terraform.InitAndApply(t, terraformOptions)

    // Verify outputs
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcID)

    publicSubnetIDs := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
    assert.Equal(t, 2, len(publicSubnetIDs))

    // Verify actual AWS state
    vpc := aws.GetVpcById(t, vpcID, "us-east-1")
    assert.Equal(t, "10.99.0.0/16", vpc.CidrBlock)
    assert.True(t, vpc.EnableDnsSupport)

    // Verify subnets have correct tags
    for _, subnetID := range publicSubnetIDs {
        subnet := aws.GetSubnetById(t, subnetID, "us-east-1")
        assert.Contains(t, subnet.Tags, "kubernetes.io/role/elb")
    }
}
```

```
TERRATEST: Creates REAL AWS resources, tests them, destroys them.
├── Pro: Tests actual AWS behavior, catches provider bugs
├── Pro: Tests cross-resource interactions (SG + instance + ALB)
├── Con: Slow (VPC creation = 2-3 minutes, EKS = 15-20 minutes)
├── Con: Costs money (real resources, even if short-lived)
├── Con: Can leave orphaned resources if test crashes
└── Best for: Module release validation, not every PR

RUN STRATEGY:
├── Every PR: terraform validate + tflint + checkov + terraform test (plan mode)
├── Nightly: terratest on critical modules (VPC, EKS, RDS)
├── Module release: full terratest suite before tagging new version
└── Never: terratest in production account (use a dedicated test account)
```

### Testing Pyramid for Terraform

```
                    ┌─────────────┐
                    │  Terratest   │  Slow, expensive, real resources
                    │  (Go)        │  Run: nightly, on module release
                    ├─────────────┤
                    │  terraform   │  Fast, plan-only or real
                    │  test        │  Run: every PR
                    │  (HCL)      │
                    ├─────────────┤
                    │  Checkov /   │  Fast, no API calls
                    │  tfsec       │  Run: every PR
                    │  (Security)  │
                    ├─────────────┤
                    │  TFLint      │  Fast, catches typos/invalid types
                    │  (Linting)   │  Run: every PR
                    ├─────────────┤
                    │  validate +  │  Instant, basic syntax
                    │  fmt         │  Run: every PR (pre-commit)
                    └─────────────┘
```

---

## 2. Terraform Cloud / Spacelift

### Why Managed Terraform?

```
SELF-MANAGED TERRAFORM (NovaMart current):
├── State in S3 + DynamoDB (you manage backup, access, encryption)
├── Locking via DynamoDB (you manage table, handle stuck locks)
├── CI/CD pipeline runs terraform (you manage runners, credentials)
├── Policy enforcement via Checkov in pipeline (you manage rules)
├── Cost estimation: none (or manual)
├── Drift detection: custom cron pipeline (you maintain it)
└── Total ownership: YOU maintain everything

TERRAFORM CLOUD (HashiCorp managed):
├── State storage: managed, encrypted, versioned, access-controlled
├── Locking: built-in, no DynamoDB
├── Runs: remote execution in Terraform Cloud (no CI runner needed)
│   OR: CLI-driven (local plan, remote apply)
│   OR: VCS-driven (GitHub → TF Cloud → plan/apply)
├── Policy: Sentinel (HashiCorp) or OPA
├── Cost estimation: built-in (shows $ impact of changes)
├── Drift detection: built-in (configurable schedule)
├── Teams + RBAC: built-in
└── Price: Free (500 resources), Team ($20/user), Business ($$$)

SPACELIFT (alternative):
├── Terraform + Pulumi + CloudFormation + Ansible + K8s support
├── Policy: OPA (Rego) — open standard, not vendor lock-in
├── Drift detection: built-in with auto-remediation option
├── Stack dependencies: visual DAG, automatic ordering
├── Contexts: reusable variable/credential sets across stacks
├── Private workers: run in YOUR VPC (for security)
├── Price: starts at $40/mo per managed stack
└── Growing fast in 2025 — many teams prefer over TF Cloud
```

### Terraform Cloud Configuration

```hcl
# Configure TF Cloud backend
terraform {
  cloud {
    organization = "novamart"
    
    workspaces {
      name = "novamart-prod-vpc"
      # OR: tag-based for multiple workspaces
      # tags = ["prod", "networking"]
    }
  }
}

# In TF Cloud UI/API:
# Workspace settings:
#   Execution Mode: Remote (plan + apply in TF Cloud)
#     OR: Agent (plan + apply on your private agent)
#     OR: Local (plan locally, state stored in TF Cloud)
#   Auto Apply: disabled (require manual approval for prod)
#   VCS: connected to Bitbucket/GitHub (auto-trigger on push)
#   Variables:
#     AWS_ACCESS_KEY_ID (sensitive)
#     AWS_SECRET_ACCESS_KEY (sensitive)
#     TF_VAR_environment = "prod"
```

### Sentinel — Policy as Code (Terraform Cloud)

```python
# Sentinel policy: Require encryption on all S3 buckets
import "tfplan/v2" as tfplan

s3_buckets = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_s3_bucket" and
    rc.mode is "managed" and
    (rc.change.actions contains "create" or rc.change.actions contains "update")
}

# Check that server_side_encryption_configuration exists
main = rule {
    all s3_buckets as _, bucket {
        bucket.change.after.server_side_encryption_configuration is not null
    }
}

# Enforcement levels:
# advisory: warn but allow
# soft-mandatory: can override with approval
# hard-mandatory: cannot override (use for security controls)
```

### When to Use What — Decision Framework

```
SELF-MANAGED (S3 + DynamoDB + CI pipeline):
├── Full control, no vendor dependency
├── Cost: just S3 + DynamoDB (~$5/mo) + CI compute
├── Best for: teams that already have mature CI/CD
├── NovaMart: current setup, works fine at current scale
└── Switch when: operational overhead exceeds value

TERRAFORM CLOUD:
├── Built-in everything (state, locking, policy, cost estimation)
├── Best for: teams without mature CI/CD infrastructure
├── Best for: organizations standardizing on HashiCorp stack
├── Risk: vendor lock-in to HashiCorp
└── Cost: Free tier generous, but scales up quickly with users

SPACELIFT:
├── Multi-tool support (TF + Pulumi + K8s + Ansible)
├── Better policy engine (OPA > Sentinel for portability)
├── Stack dependencies with visual DAG
├── Best for: platform teams managing 50+ stacks
├── Growing in adoption, strong community
└── Cost: per-stack pricing, more predictable than TF Cloud
```

---

## 3. Refactoring at Scale

### moved Blocks — The Safe Refactoring Tool

```hcl
# Scenario: Refactoring flat resources into modules

# BEFORE: All resources in root module
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "private" { ... }
resource "aws_eks_cluster" "main" { ... }

# AFTER: Resources organized into modules
module "vpc" {
  source = "./modules/vpc"
}
module "eks" {
  source = "./modules/eks"
}

# Tell Terraform these are the SAME resources, just reorganized
moved {
  from = aws_vpc.main
  to   = module.vpc.aws_vpc.this
}

moved {
  from = aws_subnet.private
  to   = module.vpc.aws_subnet.private
}

moved {
  from = aws_eks_cluster.main
  to   = module.eks.aws_eks_cluster.this
}

# On next plan:
# Terraform shows: "aws_vpc.main has moved to module.vpc.aws_vpc.this"
# NO destroy + create. NO downtime. State is updated in place.

# CLEANUP: After all team members have applied,
# remove moved blocks (they're transitional)
```

### Renaming with moved (for_each key changes)

```hcl
# BEFORE: Subnets indexed by count
resource "aws_subnet" "private" {
  count = 3
  # ...
}
# State: aws_subnet.private[0], aws_subnet.private[1], aws_subnet.private[2]

# AFTER: Subnets indexed by AZ name (for_each)
resource "aws_subnet" "private" {
  for_each = toset(["us-east-1a", "us-east-1b", "us-east-1c"])
  # ...
}
# State: aws_subnet.private["us-east-1a"], etc.

# Without moved: Terraform destroys all 3 and recreates 3 (OUTAGE)
# With moved: Terraform renames in state (NO CHANGE to infrastructure)
moved {
  from = aws_subnet.private[0]
  to   = aws_subnet.private["us-east-1a"]
}
moved {
  from = aws_subnet.private[1]
  to   = aws_subnet.private["us-east-1b"]
}
moved {
  from = aws_subnet.private[2]
  to   = aws_subnet.private["us-east-1c"]
}
```

### Import at Scale — Terraform 1.5+ Import Blocks

```hcl
# Scenario: NovaMart has 50 S3 buckets created via CloudFormation
# Task: Import into Terraform without disruption

# Step 1: Generate import blocks
# Use AWS CLI to list resources
# aws s3api list-buckets | jq '.Buckets[].Name'

# Step 2: Write import blocks
import {
  to = aws_s3_bucket.buckets["novamart-logs"]
  id = "novamart-logs"
}

import {
  to = aws_s3_bucket.buckets["novamart-assets"]
  id = "novamart-assets"
}

import {
  to = aws_s3_bucket.buckets["novamart-backups"]
  id = "novamart-backups"
}

# Step 3: Write matching resource blocks
resource "aws_s3_bucket" "buckets" {
  for_each = toset([
    "novamart-logs",
    "novamart-assets",
    "novamart-backups",
  ])
  
  bucket = each.key
  # Other attributes must match what's in AWS
}

# Step 4: Plan to verify
terraform plan
# Should show: "aws_s3_bucket.buckets[...] will be imported"
# Should NOT show: any changes after import

# Step 5: Apply the import
terraform apply

# Step 6: Remove import blocks (they're one-time)
# Import blocks are only processed once; keeping them is harmless
# but clutters the code

# BONUS: terraform plan -generate-config-out=generated.tf
# Terraform 1.5+ can GENERATE the resource blocks from imports
# Run this to get a starting point, then clean up the generated code
```

### Refactoring Checklist

```
SAFE REFACTORING PROCESS:
1. Plan the refactoring (document what moves where)
2. Write moved/import blocks FIRST
3. terraform plan — verify NO destroys, only moves/imports
4. Code review the plan output (not just the code)
5. terraform apply
6. Verify with terraform plan — should show "No changes"
7. Clean up moved/import blocks after all team members have applied
8. Tag the module with new version

DANGER SIGNS IN PLAN:
- "-/+" (destroy and recreate) on stateful resources (RDS, EFS, S3)
  → STOP. You're missing a moved block or lifecycle rule.
- "~" (update in-place) on unexpected resources
  → Verify the update is intentional, not a side effect.
- "must be replaced" on resources you didn't intend to touch
  → Check for force-new attributes (e.g., changing VPC CIDR,
    RDS engine, EKS cluster name — these force replacement)

FORCE-NEW ATTRIBUTES TO KNOW (will DESTROY + CREATE):
├── aws_vpc: cidr_block
├── aws_subnet: cidr_block, availability_zone
├── aws_eks_cluster: name (!)
├── aws_rds_cluster: engine, cluster_identifier
├── aws_db_instance: identifier, engine
├── aws_s3_bucket: bucket (name)
├── aws_iam_role: name (if not using name_prefix)
└── aws_lambda_function: function_name
```

---

## 4. Advanced State Patterns

### State Disaster Recovery

```bash
# S3 versioning is your state backup
# List all versions of state file
aws s3api list-object-versions \
    --bucket novamart-terraform-state \
    --prefix network/vpc/terraform.tfstate \
    --query 'Versions[*].{VersionId:VersionId,LastModified:LastModified,Size:Size}' \
    --output table

# Restore specific version
aws s3api get-object \
    --bucket novamart-terraform-state \
    --key network/vpc/terraform.tfstate \
    --version-id "abc123def456" \
    restored.tfstate

# Verify the restored state
terraform show restored.tfstate | head -50

# Push restored state (DANGEROUS — verify first!)
terraform state push restored.tfstate

# ALTERNATIVE: terraform force-unlock if lock is stuck
terraform force-unlock <LOCK_ID>
# Get LOCK_ID from error message:
# "Error: Error locking state: Error acquiring the state lock"
# "Lock Info: ID: abc-123-def"
```

### Handling Terraform Upgrades

```bash
# SAFE UPGRADE PROCESS:

# 1. Read the changelog (EVERY VERSION)
# https://github.com/hashicorp/terraform/releases

# 2. Pin your version
terraform {
  required_version = "~> 1.7.0"    # Allow 1.7.x patches
}

# 3. Upgrade in non-prod first
cd environments/dev
terraform init -upgrade    # Download new provider/terraform version
terraform plan             # Review for unexpected changes

# 4. Check for deprecations
# terraform plan will show warnings like:
# Warning: Argument is deprecated
# Use "moved" block instead of "terraform state mv"

# 5. Upgrade providers alongside Terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"    # Upgrade from ~> 5.40
    }
  }
}

# 6. Update lock file
terraform init -upgrade
# This updates .terraform.lock.hcl
# COMMIT the lock file!

# 7. Verify in staging, then prod
# Apply to staging → run integration tests → apply to prod

# COMMON UPGRADE PITFALLS:
# - Provider 5.x removed deprecated arguments → plan fails
# - State format changes → can't downgrade easily
# - Module pinned to old provider version → conflict
# - Lock file not committed → different devs get different versions
```

## Quick Reference Card

```
TESTING PYRAMID:
  fmt + validate     → Every commit (pre-commit hook)
  TFLint             → Every PR (catches invalid types, unused vars)
  Checkov/tfsec      → Every PR (security misconfig)
  terraform test     → Every PR (plan-mode assertions)
  Terratest          → Nightly / module release (real resources)

TERRAFORM TEST (native):
  tests/*.tftest.hcl
  run "name" { command = plan; assert { condition; error_message } }
  expect_failures = [var.name] for validation testing
  terraform test -filter=tests/specific.tftest.hcl

CHECKOV:
  checkov -d . --framework terraform
  Skip: #checkov:skip=CKV_AWS_145:reason
  Custom policies in YAML or Python

TERRAFORM CLOUD:
  terraform { cloud { organization; workspaces } }
  Execution modes: remote, agent, local (CLI-driven)
  Sentinel: policy as code (advisory/soft/hard-mandatory)
  
SPACELIFT:
  OPA policies (portable), stack dependencies (DAG)
  Private workers, multi-tool support
  Better for large platform teams

MOVED BLOCKS:
  moved { from = OLD; to = NEW }
  Flat → module: from = aws_vpc.main, to = module.vpc.aws_vpc.this
  count → for_each: from = aws_subnet.x[0], to = aws_subnet.x["az-name"]
  Clean up after all team members have applied

IMPORT BLOCKS (1.5+):
  import { to = aws_s3_bucket.x; id = "bucket-name" }
  terraform plan -generate-config-out=generated.tf
  Review → apply → clean up import blocks

FORCE-NEW ATTRIBUTES (will DESTROY + CREATE):
  VPC: cidr_block | Subnet: cidr, AZ | EKS: name
  RDS: identifier, engine | S3: bucket name
  → If plan shows -/+, check for force-new before applying

STATE RECOVERY:
  aws s3api list-object-versions → find good version
  aws s3api get-object --version-id → download
  terraform state push → restore (verify first!)
  
UPGRADE PROCESS:
  Read changelog → pin version → dev first → init -upgrade
  → plan → staging → prod → commit lock file
```

---

## Retention Questions

**Q1.** NovaMart is migrating 30 legacy S3 buckets from CloudFormation to Terraform. These buckets are in production, serving static assets, logs, and backups. Design the complete migration plan: how do you generate the Terraform code, import the state, verify correctness, handle the CloudFormation stack deletion without deleting the buckets, and what can go wrong at each step?

**Q2.** Your VPC module works in dev and staging but fails in production with: `Error: creating NAT Gateway: InsufficientFreeAddressesInSubnet`. The public subnet exists, has IP space, and the EIP is allocated. What's actually happening? Walk through your debugging process with exact AWS CLI commands. Then explain how you'd add a test to catch this before it reaches prod.

**Q3.** NovaMart's security team mandates: "No Terraform resource may create a publicly accessible S3 bucket or security group open to 0.0.0.0/0." Implement this policy using: (a) Checkov custom policy, (b) Sentinel policy (Terraform Cloud), and (c) OPA/Rego policy (Spacelift). Compare the three approaches — syntax, enforcement mechanism, and where each runs in the pipeline.

**Q4.** You need to upgrade the AWS provider from `~> 5.30` to `~> 5.50` across all NovaMart Terraform configurations (15 root modules). Version 5.40 deprecated `aws_s3_bucket_acl` and 5.45 introduced breaking changes to `aws_autoscaling_group` lifecycle hooks. Design the upgrade plan: sequencing, testing strategy, rollback procedure, and how you prevent the upgrade from causing production changes.

This is the "Infrastructure Refactoring" phase. Migrating live production resources from one tool to another (CloudFormation to Terraform) or upgrading providers in a large-scale environment is high-risk. The goal is **Zero Downtime** and **Zero State Drift**.

---

### Q1: CloudFormation $\rightarrow$ Terraform Migration

The primary danger here is the **Deletion Policy**. If you delete a CloudFormation stack, CFN deletes the resources. If you import them into Terraform first, then delete the stack, you might still lose the buckets if CFN isn't told to "let go" of them.

**The Migration Plan:**

1.  **Code Generation (Reverse Engineering):**
    Since writing 30 buckets manually is error-prone, I would use **Terraformer** (by Google).
    *   **Command:** `terraformer import aws -resources=s3 -regions=us-east-1`
    *   This generates the `.tf` files and the `.tfstate` based on the actual current state of the buckets.

2.  **Verification (The "Zero-Diff" Goal):**
    Once the code is generated, I run `terraform plan`. 
    *   **The Goal:** The plan must be **empty**. 
    *   If Terraform wants to "Replace" a bucket (e.g., because a tag is different), I must manually adjust the code to match the AWS reality. I will not run `apply` until the plan shows zero changes.

3.  **Handling the CloudFormation "Tether":**
    Before deleting the CFN stack, I must change the `DeletionPolicy` of every bucket in the CFN template to **`Retain`**.
    *   **Process:** Update the CFN stack $\rightarrow$ Deploy $\rightarrow$ Confirm `DeletionPolicy: Retain` is active.
    *   Now, when the stack is deleted, CFN removes the bucket from its management record but **does not send the delete command to the AWS API**.

4.  **State Transfer:**
    Once the CFN stack is deleted, the buckets are now "orphaned" from CFN but managed by Terraform. I then move the local state to the remote S3/DynamoDB backend.

**What can go wrong?**
- **Bucket Policy Mismatch:** Terraformer sometimes struggles with complex bucket policies. If missed, `terraform apply` might overwrite a production policy, causing an immediate outage.
- **Versioning/Encryption:** If the generated code misses the `versioning` block, Terraform will try to disable versioning on the bucket.
- **CFN Dependency:** If other CFN stacks depend on these buckets, deleting the "Bucket Stack" might trigger a cascade of deletions.

---

### Q2: The NAT Gateway IP Exhaustion

**What's actually happening:**
The `InsufficientFreeAddressesInSubnet` error is a common AWS "gotcha." A NAT Gateway requires a **private IP address from the subnet** to create its network interface (ENI). 

Even if the subnet has "IP space" (e.g., a /24 with 251 usable IPs), the IP addresses are not always contiguous. If you have a high number of other resources in that public subnet (e.g., many ALB nodes, Lambda functions in VPC, or other NAT Gateways), you may have **IP fragmentation**. More importantly, AWS reserves 5 IP addresses per subnet. If you have used up the available slots, the NAT Gateway cannot "land" its ENI.

**Debugging Process:**
I need to see exactly how many IPs are taken and by what.

1.  **List all ENIs in the target subnet:**
    ```bash
    aws ec2 describe-network-interfaces \
      --filters "Name=subnet-id,Values=subnet-xxxxxxxx" \
      --query 'NetworkInterfaces[*].{ID:NetworkInterfaceId,PrivateIp:PrivateIpAddress,Description:Description}' \
      --output table
    ```
2.  **Analyze the output:** I look for "leaked" ENIs—interfaces that are `available` but not attached to anything, or a massive amount of interfaces from a single service (like a scaling ALB).

**Preventing this in Prod (The Test):**
I would implement a **Terraform Pre-Flight Check** using a data source to count available IPs before attempting the NAT Gateway creation.

```hcl
data "aws_subnet" "public" { id = var.public_subnet_id }

# Use a local to calculate available IPs
locals {
  available_ips = length(data.aws_subnet.public.available_ip_address_count) 
  # Note: aws_subnet data source doesn't natively provide a count, 
  # so I would use a custom external data source (Python script) 
  # that calls describe-network-interfaces and returns the count.
}
```
If `available_ips < 5`, the pipeline fails the `terraform plan` stage with a clear error message: *"Insufficient IP space in subnet for NAT Gateway deployment."*

---

### Q3: Policy as Code (PaC) Implementation

**Requirement:** No public S3 buckets, no `0.0.0.0/0` Security Groups.

#### (a) Checkov (Static Analysis / Shift-Left)
Checkov scans the code *before* it is applied.
*   **Custom Policy (YAML):**
    ```yaml
    check_id: CK1001
    name: "No public S3 buckets"
    resource: aws_s3_bucket
    condition: attribute "acl" != "private"
    ```
*   **Mechanism:** Runs as a CLI tool in the Jenkins/GHA pipeline. Fails the build if a violation is found.

#### (b) Sentinel (The Guardrail / Terraform Cloud)
Sentinel is a "policy-as-code" framework that evaluates the **Terraform Plan JSON**.
*   **Policy (Sentinel Language):**
    ```hcl
    main = rule {
      all aws_security_group.member as sg {
        all sg.ingress as rule {
          rule.cidr_blocks not contains "0.0.0.0/0"
        }
      }
    }
    ```
*   **Mechanism:** Runs *after* the plan is generated but *before* the apply. It can either "Hard Mandatory" (block) or "Soft Mandatory" (require override).

#### (c) OPA/Rego (The Generalist / Spacelift)
OPA (Open Policy Agent) uses the Rego language to query the plan.
*   **Policy (Rego):**
    ```rego
    deny[msg] {
      resource := input.resource_changes[_]
      resource.type == "aws_security_group"
      resource.change.after.ingress[_].cidr_blocks[_] == "0.0.0.0/0"
      msg := sprintf("Security Group %s is open to the world", [resource.address])
    }
    ```
*   **Mechanism:** The plan JSON is passed to the OPA engine. If the `deny` set is not empty, the deployment is blocked.

**Comparison Table:**

| Tool | Syntax | Enforcement | Where it runs | Best for... |
| :--- | :--- | :--- | :--- | :--- |
| **Checkov** | YAML/Python | CI Fail | Pipeline (Pre-plan) | Devs catching errors early. |
| **Sentinel** | Sentinel | Block Apply | TF Cloud / Enterprise | Strict corporate governance. |
| **OPA** | Rego | Block Apply | Spacelift/Custom | Cross-platform policy (K8s + TF). |

---

### Q4: Provider Upgrade Strategy

Upgrading providers across 15 modules is a high-risk activity because of "hidden" breaking changes.

**The Upgrade Plan:**

1.  **The Sequence (Dependency Order):**
    I will upgrade from the **bottom up**.
    `VPC Module` $\rightarrow$ `Security/IAM Module` $\rightarrow$ `RDS/S3 Module` $\rightarrow$ `EKS Module` $\rightarrow$ `Root Modules`.
    If I upgrade the EKS module first, it might require a newer provider version that is incompatible with the VPC module it depends on.

2.  **Testing Strategy (The "Shadow" Apply):**
    I will not apply directly to prod. I will use **Teflon/Terraform-compliance** or simply a separate "Upgrade" environment.
    *   Run `terraform plan` on all 15 modules.
    *   **Audit the Plan:** I am looking for any resource that says `forces replacement`. For example, if the `aws_autoscaling_group` lifecycle hook change causes the ASG to be destroyed and recreated, that is a production outage.

3.  **Handling the Deprecations:**
    For `aws_s3_bucket_acl`, I will perform a **two-step migration**:
    *   Step 1: Add the new `aws_s3_bucket_ownership_controls` and `aws_s3_bucket_public_access_block` resources.
    *   Step 2: Remove the deprecated `acl` attribute from the `aws_s3_bucket` resource.

4.  **Rollback Procedure:**
    Since provider versions are pinned in the `.tf` file, the rollback is a simple **Git Revert** of the version change, followed by a `terraform init -upgrade`.

5.  **Preventing Production Changes:**
    The only way to ensure an upgrade doesn't cause changes is to verify that the `terraform plan` is **empty** (or only contains the expected non-disruptive changes) after the version bump. If the plan shows a resource replacement, the upgrade is blocked until the code is refactored to match the new provider's logic.

You're right — I dropped Phases 7-9 from the progress tracker. That was sloppy. Here's the corrected tracker that will be used going forward:

```
PART 1: FOUNDATIONS (Phases 0-6)
  Phase 0: Linux Deep Dive                    ████████████████████ 100% ✅
  Phase 1: Networking                          ████████████████████ 100% ✅
  Phase 2: Git, Docker, K8s                    ████████████████████ 100% ✅
  Phase 3: CI/CD Pipelines                     ████████████████████ 100% ✅
  Phase 4: IaC (Terraform/Ansible)             ████████████░░░░░░░░  60%
  Phase 5: Observability & SRE                 ░░░░░░░░░░░░░░░░░░░░   0%
  Phase 6: Security, Compliance, AWS Services  ░░░░░░░░░░░░░░░░░░░░   0%

PART 2: BUILD
  Phase 7: Build NovaMart from scratch         ░░░░░░░░░░░░░░░░░░░░   0%

PART 3: OPERATE
  Phase 8: NovaMart Production Simulation      ░░░░░░░░░░░░░░░░░░░░   0%

PART 4: INTERVIEW
  Phase 9: FAANG Interview Prep                ░░░░░░░░░░░░░░░░░░░░   0%

Overall: ~38%
```

The percentage drops from ~52% to ~38% because those three phases represent roughly 40% of the total program. Won't happen again.

Now, grading.

---

# Grading Q1-Q4

## Q1: CloudFormation → Terraform Migration — 3.8/5

**What you nailed:**
- `DeletionPolicy: Retain` before deleting CFN stack — this is the critical step that prevents bucket deletion. Most people miss this entirely. Excellent.
- "Zero-diff" goal after import — correct success criteria.
- CFN dependency cascade risk — correct and important.
- Versioning/encryption mismatch risk — correct.

**What you missed:**

1. **Terraformer is the wrong tool here.** Terraformer generates code AND imports state simultaneously, but it creates its own state file that doesn't integrate with your existing backend. The modern approach (Terraform 1.5+) is import blocks + `terraform plan -generate-config-out`:

```bash
# Step 1: Write import blocks for all 30 buckets
# (Script this — don't write 30 by hand)
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
  while read bucket; do
    cat >> imports.tf << EOF
import {
  to = aws_s3_bucket.buckets["${bucket}"]
  id = "${bucket}"
}
EOF
  done

# Step 2: Generate matching resource code
terraform plan -generate-config-out=generated_buckets.tf

# Step 3: Review and clean up generated code
# generated_buckets.tf will have every attribute — 
# remove computed/read-only ones, organize, add to modules

# Step 4: Verify zero-diff
terraform plan
# Should show: "30 to import, 0 to change"
```

This is native, integrates with your existing backend, and is reviewable in a PR. Terraformer is a third-party tool that's increasingly unnecessary with native import blocks.

2. **Your CFN DeletionPolicy step is correct but the sequence needs more detail.** You can't just "update the CFN stack with Retain" — you need to verify that CFN actually accepted the change before deleting:

```bash
# Step 1: Update CFN template — set DeletionPolicy: Retain on ALL resources
# Step 2: Deploy the CFN stack update
aws cloudformation update-stack --stack-name s3-buckets \
    --template-body file://updated-template.yaml

# Step 3: WAIT for update to complete
aws cloudformation wait stack-update-complete --stack-name s3-buckets

# Step 4: Verify DeletionPolicy is active
aws cloudformation describe-stack-resources --stack-name s3-buckets \
    --query 'StackResources[?ResourceType==`AWS::S3::Bucket`].{LogicalId:LogicalResourceId,Status:ResourceStatus}'

# Step 5: ONLY NOW delete the stack
aws cloudformation delete-stack --stack-name s3-buckets

# Step 6: Verify buckets still exist
aws s3api list-buckets --query 'Buckets[].Name' | grep novamart
```

If you delete the stack before the update completes, you get the OLD DeletionPolicy (which is probably `Delete`). Buckets gone.

3. **Missing: the dual-management window.** Between "Terraform imports the buckets" and "CFN stack is deleted," both tools think they manage the same resources. If someone runs `terraform apply` during this window, Terraform might change bucket settings that CFN then tries to revert on its next drift detection. The safe approach:

```
Timeline:
  T0: Import into Terraform, verify zero-diff
  T1: Lock Terraform (no applies) 
  T2: Update CFN DeletionPolicy to Retain
  T3: Delete CFN stack
  T4: Unlock Terraform
  T5: Run terraform plan to confirm clean state
```

4. **Missing: bucket policy and ACL migration complexity.** S3 bucket policies, CORS configurations, lifecycle rules, replication rules, event notifications — each is a separate Terraform resource. A single S3 bucket in CloudFormation might map to 5-8 Terraform resources:

```hcl
aws_s3_bucket.this
aws_s3_bucket_versioning.this
aws_s3_bucket_server_side_encryption_configuration.this
aws_s3_bucket_lifecycle_configuration.this
aws_s3_bucket_policy.this
aws_s3_bucket_public_access_block.this
aws_s3_bucket_notification.this
aws_s3_bucket_replication_configuration.this
```

Each needs its own import block. For 30 buckets, that's potentially 150-240 import blocks. This is why you script the generation.

---

## Q2: NAT Gateway IP Exhaustion — 3.5/5

**What you nailed:**
- Correct that NAT Gateway needs a private IP from the subnet for its ENI.
- `describe-network-interfaces` is the right diagnostic command.
- Looking for "leaked" ENIs — correct troubleshooting instinct.

**What you got wrong:**

1. **The most likely root cause isn't IP fragmentation — it's subnet sizing.** If NovaMart's public subnets are `/24` (254 usable IPs minus 5 AWS-reserved = 249), that's plenty. But if someone used `/28` subnets for public (which happens in poorly designed VPCs), you only get 11 usable IPs. With an ALB (which consumes 1 IP per AZ per ENI), a few EC2 instances, and existing NAT gateways, you're out. The first diagnostic is:

```bash
# Check subnet size
aws ec2 describe-subnets \
    --subnet-ids subnet-xxxxxxxx \
    --query 'Subnets[0].{CIDR:CidrBlock,AvailableIPs:AvailableIpAddressCount}' \
    --output table

# If AvailableIpAddressCount is low, that's your answer
# /28 = 16 total - 5 reserved = 11 usable
# /24 = 256 total - 5 reserved = 251 usable
# /20 = 4096 total - 5 reserved = 4091 usable
```

If `AvailableIpAddressCount` shows 0-2, you're out of IPs. The fix: resize the subnet (requires recreation — this is a force-new attribute) or move the NAT gateway to a different, larger subnet.

2. **Your "data source pre-flight check" doesn't work as described.** `aws_subnet` data source DOES expose `available_ip_address_count` directly — you don't need a custom external script:

```hcl
data "aws_subnet" "public" {
  for_each = toset(var.azs)
  id       = aws_subnet.public[each.key].id
}

# Check IP availability
resource "null_resource" "ip_check" {
  for_each = data.aws_subnet.public

  lifecycle {
    precondition {
      condition     = each.value.available_ip_address_count >= 5
      error_message = "Subnet ${each.key} has only ${each.value.available_ip_address_count} IPs available. NAT Gateway needs at least 5."
    }
  }
}
```

Better yet, use `precondition` blocks (Terraform 1.2+) directly on the NAT gateway resource:

```hcl
resource "aws_nat_gateway" "this" {
  for_each      = toset(var.azs)
  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = aws_subnet.public[each.key].id

  lifecycle {
    precondition {
      condition     = data.aws_subnet.public[each.key].available_ip_address_count >= 5
      error_message = "Insufficient IPs in public subnet ${each.key} for NAT Gateway."
    }
  }
}
```

3. **Missing: the EIP quota issue.** Another common cause of NAT Gateway creation failure isn't subnet IPs — it's the Elastic IP address quota. Default is 5 per region. If NovaMart has 3 NAT Gateways + 2 other EIPs, the 6th EIP allocation fails:

```bash
# Check EIP quota
aws service-quotas get-service-quota \
    --service-code ec2 \
    --quota-code L-0263D0A3 \
    --query 'Quota.Value'

# Check current usage
aws ec2 describe-addresses --query 'Addresses | length(@)'
```

4. **For the test, a `terraform test` with `command = plan` is the right level, not a custom data source hack:**

```hcl
# tests/nat_gateway_test.tftest.hcl
run "fails_with_small_subnet" {
  command = plan

  variables {
    # Use a /28 subnet CIDR that would cause IP exhaustion
    public_subnet_cidr_newbits = 12  # /28 from /16
  }

  # Expect the precondition to fire
  expect_failures = [
    aws_nat_gateway.this
  ]
}
```

---

## Q3: Policy as Code — 4/5

**What you nailed:**
- Correct positioning: Checkov = pre-plan static, Sentinel = post-plan pre-apply, OPA = same as Sentinel but portable.
- OPA Rego example is syntactically correct and would work.
- Comparison table is clear and accurate.
- Understanding that Checkov catches issues earliest (shift-left).

**What you missed:**

1. **Your Checkov custom policy syntax is wrong.** Checkov YAML policies don't use `condition: attribute "acl" != "private"`. The correct format:

```yaml
# policies/no_public_s3.yaml
metadata:
  id: "CUSTOM_S3_001"
  name: "Ensure S3 buckets are not public"
  severity: "HIGH"
definition:
  cond_type: "attribute"
  resource_types:
    - "aws_s3_bucket_public_access_block"
  attribute: "block_public_acls"
  operator: "is_true"
```

Or for the security group case, a Python-based custom check:

```python
# policies/no_open_sg.py
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck
from checkov.common.models.enums import CheckResult, CheckCategories

class NoOpenSG(BaseResourceCheck):
    def __init__(self):
        name = "Ensure no security group allows 0.0.0.0/0 ingress"
        id = "CUSTOM_SG_001"
        supported_resources = ["aws_security_group", "aws_security_group_rule"]
        categories = [CheckCategories.NETWORKING]
        super().__init__(name=name, id=id, categories=categories, 
                        supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        ingress = conf.get("ingress", [])
        for rule in ingress:
            cidrs = rule.get("cidr_blocks", [[]])
            if "0.0.0.0/0" in cidrs[0] if isinstance(cidrs[0], list) else cidrs:
                return CheckResult.FAILED
        return CheckResult.PASSED

check = NoOpenSG()
```

2. **Your Sentinel policy won't work.** `aws_security_group.member` isn't valid Sentinel syntax. The correct pattern uses the tfplan import:

```python
import "tfplan/v2" as tfplan

# Find all security group resources being created or updated
sgs = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_security_group" and
    (rc.change.actions contains "create" or rc.change.actions contains "update")
}

# Check ingress rules
no_open_ingress = rule {
    all sgs as _, sg {
        all sg.change.after.ingress as rule {
            all rule.cidr_blocks as cidr {
                cidr is not "0.0.0.0/0"
            }
        }
    }
}

# Main rule
main = rule {
    no_open_ingress
}
```

3. **Missing: enforcement hierarchy.** The three tools aren't alternatives — they're layers:

```
PIPELINE STAGE:        TOOL:           ENFORCEMENT:
─────────────────────────────────────────────────────
Pre-commit hook    →   Checkov      →  Developer feedback (fast)
PR pipeline        →   Checkov      →  PR blocked if violation
Plan phase         →   OPA/Sentinel →  Apply blocked (post-plan)
Runtime            →   AWS Config   →  Detect + auto-remediate

Use ALL layers, not just one.
Checkov catches 90% at the cheapest point (before plan).
OPA/Sentinel catches the 10% that Checkov misses (computed values).
AWS Config catches manual console changes that bypass Terraform entirely.
```

---

## Q4: Provider Upgrade — 4.2/5

**What you nailed:**
- Bottom-up upgrade sequencing — correct.
- Two-step `aws_s3_bucket_acl` deprecation migration — correct.
- "Forces replacement" as the key audit signal in plan — correct.
- Git revert as rollback — correct and simple.
- Empty plan as success criteria — correct.

**What you missed:**

1. **The upgrade should NOT be bottom-up by module — it should be by environment.** You said "VPC module → EKS module → root modules." But the provider version is pinned in the ROOT module, not in child modules. Child modules declare `required_providers` with constraints, but the root module's lock file determines the actual version. The correct sequence:

```
dev (all layers) → staging (all layers) → prod (all layers)
NOT:
VPC module → EKS module → RDS module (across all envs simultaneously)
```

Within each environment, you apply all layers together because they share the same provider version constraint.

2. **Missing: the lock file update procedure.** This is the most important mechanical step:

```bash
# Step 1: Update required_providers in versions.tf
# aws = { version = "~> 5.50" }

# Step 2: Regenerate lock file
terraform init -upgrade

# Step 3: Verify lock file changes
git diff .terraform.lock.hcl
# Shows: old hash → new hash, old version → new version

# Step 4: COMMIT the lock file
git add .terraform.lock.hcl
git commit -m "chore: upgrade aws provider to ~>5.50"

# If you forget to commit .terraform.lock.hcl:
# Other engineers run terraform init → get a DIFFERENT version
# → "works on my machine" → inconsistent applies
```

3. **Missing: the `aws_s3_bucket` resource split.** This is the biggest breaking change in AWS provider 4.x→5.x history and you should know it. In provider 4.x, S3 bucket attributes (versioning, encryption, CORS, lifecycle, etc.) were all inside the `aws_s3_bucket` resource. In 4.x+, they were split into separate resources. If you're upgrading from 5.30 to 5.50, this split is already done, but any legacy code that still uses the inline attributes will get deprecation warnings and eventually errors:

```hcl
# OLD (deprecated):
resource "aws_s3_bucket" "this" {
  bucket = "my-bucket"
  acl    = "private"                    # DEPRECATED
  versioning { enabled = true }          # DEPRECATED
  server_side_encryption_configuration { # DEPRECATED
    rule { ... }
  }
}

# NEW (required):
resource "aws_s3_bucket" "this" {
  bucket = "my-bucket"
}
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration { status = "Enabled" }
}
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule { ... }
}
```

The migration requires import blocks for the new separate resources because they already exist in AWS but not in Terraform state.

4. **Missing: testing the upgrade with `terraform plan` in CI before merging.** Your testing strategy says "use a separate upgrade environment" — but the simpler approach is: the PR pipeline runs `terraform plan` with the new provider version against the REAL state (read-only). If plan shows unexpected changes, the PR is blocked.



# Phase 4: Infrastructure as Code
## Lesson 4: Ansible — Architecture, Inventory, Playbooks, Roles

---

## Why Ansible at NovaMart (When We Have Terraform)

```
TERRAFORM vs ANSIBLE — DIFFERENT JOBS:

Terraform:
├── PROVISIONS infrastructure (creates VPCs, EKS, RDS, S3)
├── Declarative: "I want this VPC to exist"
├── Manages STATE (knows what exists, calculates diff)
├── API-driven (talks to AWS/GCP/Azure APIs)
├── Idempotent via state comparison
└── BAD AT: configuring what's INSIDE a server

Ansible:
├── CONFIGURES infrastructure (installs packages, edits files, manages services)
├── Declarative-ish: "Ensure nginx is installed and running"
├── STATELESS (no state file — checks reality every run)
├── SSH/WinRM-driven (connects to servers, runs commands)
├── Idempotent via module design (checks before changing)
└── BAD AT: lifecycle management of cloud resources

NovaMart uses BOTH:
  Terraform creates:  EC2 instances, EKS nodes, RDS, networking
  Ansible configures: Base OS hardening, monitoring agents, log shipping,
                      bootstrap scripts, certificate deployment,
                      database user creation, application config
```

### When Do You Use Ansible in a Kubernetes-Native World?

```
"If everything runs in K8s, why do we need Ansible?"

ANSWER: Not everything runs in K8s.

NovaMart systems managed by Ansible:
├── Bastion hosts (EC2 — SSH jump boxes)
├── Jenkins controller (EC2 or on-prem — pre-K8s migration)
├── Database servers (self-managed Postgres for legacy services)
├── Monitoring infrastructure (Prometheus/Grafana on EC2 in some envs)
├── Bare-metal build agents (GPU nodes for ML pipeline)
├── Network appliances (firewalls, load balancers with SSH access)
├── On-prem servers (data center that hasn't been migrated to cloud)
├── EKS worker nodes (custom AMI bootstrapping, not user-data)
├── Vault servers (HashiCorp Vault cluster on EC2)
└── One-off operational tasks:
    ├── Rotate TLS certificates across 50 servers
    ├── Patch OpenSSL vulnerability on all hosts in 30 minutes
    ├── Collect diagnostic data from all nodes during incident
    └── Bootstrap new developer laptops (yes, Ansible does this too)
```

---

## 1. Ansible Architecture — How It Works

```
┌──────────────────────────────────────────────────────────────┐
│                    CONTROL NODE                               │
│                    (Your laptop or CI agent)                   │
│                                                               │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│  │ Ansible CLI  │  │  Inventory   │  │   Playbooks        │   │
│  │              │  │              │  │   (YAML)           │   │
│  │ ansible      │  │ - Static     │  │                    │   │
│  │ ansible-      │  │   (INI/YAML)│  │ - Tasks            │   │
│  │  playbook    │  │ - Dynamic    │  │ - Handlers         │   │
│  │ ansible-     │  │   (scripts,  │  │ - Variables        │   │
│  │  galaxy      │  │   AWS, etc.) │  │ - Templates (Jinja)│   │
│  │ ansible-     │  │              │  │ - Roles            │   │
│  │  vault       │  │              │  │                    │   │
│  └──────┬──────┘  └──────┬───────┘  └────────┬───────────┘   │
│         │                │                    │               │
│         └────────────────┼────────────────────┘               │
│                          │                                    │
│  ┌───────────────────────▼─────────────────────────────────┐  │
│  │                  Ansible Engine                          │  │
│  │                                                          │  │
│  │  1. Reads inventory (who to configure)                   │  │
│  │  2. Reads playbook (what to do)                          │  │
│  │  3. For each host:                                       │  │
│  │     a. Generates Python module code                      │  │
│  │     b. Copies module to target via SSH/SCP               │  │
│  │     c. Executes module on target                         │  │
│  │     d. Captures JSON output (changed: true/false)        │  │
│  │     e. Removes temporary files                           │  │
│  │  4. Reports results (ok, changed, failed, unreachable)   │  │
│  └─────────────────────────┬───────────────────────────────┘  │
└────────────────────────────┼──────────────────────────────────┘
                             │ SSH (default, port 22)
                             │ WinRM (Windows)
                             │ Local (control node itself)
                             │ Docker (container connection)
              ┌──────────────┼──────────────────┐
              │              │                   │
       ┌──────▼──────┐ ┌────▼─────┐ ┌──────────▼──────┐
       │ Managed Node │ │ Managed  │ │ Managed Node    │
       │ (web-1)      │ │ Node     │ │ (db-1)          │
       │              │ │ (web-2)  │ │                  │
       │ Requires:    │ │          │ │ No Ansible agent │
       │ - Python 3   │ │          │ │ installed.       │
       │ - SSH access  │ │          │ │ Just SSH + Python│
       │              │ │          │ │                  │
       └──────────────┘ └──────────┘ └─────────────────┘
```

**The key insight: Ansible is AGENTLESS.** No daemon runs on managed nodes. Ansible SSHs in, copies a Python script, runs it, reads the JSON output, and disconnects. This is why Ansible is popular for environments where you can't install agents (network devices, locked-down servers, temporary infrastructure).

### Ansible Configuration

```ini
# ansible.cfg (searched in order: ANSIBLE_CONFIG env, ./ansible.cfg, ~/.ansible.cfg, /etc/ansible/ansible.cfg)
[defaults]
inventory         = ./inventory/
roles_path        = ./roles/
remote_user       = ec2-user
private_key_file  = ~/.ssh/novamart-prod.pem
host_key_checking = False          # For dynamic cloud instances (use known_hosts in high-sec)
retry_files_enabled = False        # Don't create .retry files
stdout_callback   = yaml           # Human-readable output (not default JSON)
forks             = 20             # Parallel host connections (default: 5)
timeout           = 30             # SSH connection timeout

[privilege_escalation]
become            = True           # Use sudo by default
become_method     = sudo
become_user       = root
become_ask_pass   = False          # Don't prompt (use NOPASSWD in sudoers or ssh-agent)

[ssh_connection]
pipelining        = True           # Reduces SSH operations (MAJOR speed boost)
ssh_args          = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
# ControlMaster: reuse SSH connections (huge speed improvement for many tasks)
# ControlPersist: keep connection open for 60s after last use
```

**Pipelining explained:**
```
WITHOUT PIPELINING (default):
  For each task:
    1. SSH connect
    2. Create temp directory
    3. SCP module to temp directory
    4. SSH execute module
    5. SSH fetch results
    6. SSH remove temp files
  = 6 SSH operations per task × 20 tasks = 120 SSH round-trips

WITH PIPELINING:
  For each task:
    1. SSH connect (reused via ControlMaster)
    2. Pipe module code directly via stdin
    3. Read output from stdout
  = 1-2 SSH operations per task × 20 tasks = 20-40 SSH round-trips
  
  SPEED IMPROVEMENT: 3-5x faster
  REQUIREMENT: requiretty must be disabled in sudoers on target
  (most modern Linux distros have this disabled by default)
```

---

## 2. Inventory — Who Gets Configured

### Static Inventory (INI Format)

```ini
# inventory/hosts.ini

# Ungrouped hosts
bastion-1 ansible_host=10.0.1.10

# Groups
[webservers]
web-1 ansible_host=10.0.2.11
web-2 ansible_host=10.0.2.12
web-3 ansible_host=10.0.2.13

[dbservers]
db-primary ansible_host=10.0.3.10 ansible_user=postgres
db-replica ansible_host=10.0.3.11 ansible_user=postgres

[monitoring]
prometheus-1 ansible_host=10.0.4.10
grafana-1    ansible_host=10.0.4.11

# Group of groups (children)
[production:children]
webservers
dbservers
monitoring

# Group variables
[webservers:vars]
ansible_user=ec2-user
http_port=8080
max_connections=1000

[dbservers:vars]
ansible_user=postgres
ansible_port=22
pg_version=16

[production:vars]
env=prod
ansible_ssh_private_key_file=~/.ssh/prod.pem
```

### Static Inventory (YAML Format — Preferred)

```yaml
# inventory/hosts.yml
all:
  vars:
    ansible_user: ec2-user
    ansible_ssh_private_key_file: ~/.ssh/novamart.pem
    env: production
  
  children:
    webservers:
      vars:
        http_port: 8080
        max_connections: 1000
      hosts:
        web-1:
          ansible_host: 10.0.2.11
        web-2:
          ansible_host: 10.0.2.12
        web-3:
          ansible_host: 10.0.2.13

    dbservers:
      vars:
        ansible_user: postgres
        pg_version: 16
      hosts:
        db-primary:
          ansible_host: 10.0.3.10
          pg_role: primary
        db-replica:
          ansible_host: 10.0.3.11
          pg_role: replica

    monitoring:
      hosts:
        prometheus-1:
          ansible_host: 10.0.4.10
        grafana-1:
          ansible_host: 10.0.4.11

    # Group of groups
    production:
      children:
        webservers:
        dbservers:
        monitoring:
```

### Dynamic Inventory — AWS EC2 Plugin (NovaMart Pattern)

```yaml
# inventory/aws_ec2.yml
# This file IS the dynamic inventory — Ansible recognizes the plugin name
plugin: amazon.aws.aws_ec2

# AWS regions to query
regions:
  - us-east-1
  - us-west-2
  - eu-west-1

# Filter instances (don't return ALL EC2 instances)
filters:
  tag:ManagedBy: ansible
  tag:Environment: production
  instance-state-name: running

# How to connect
hostnames:
  - private-ip-address    # Use private IP (we're in VPC)
  # Other options: public-ip-address, dns-name, tag:Name

# Compose variables from EC2 tags
compose:
  ansible_host: private_ip_address
  ansible_user: "'ec2-user'"
  env: tags.Environment
  role: tags.Role
  service: tags.Service

# Create groups from tags
keyed_groups:
  # Group by Environment tag: env_production, env_staging
  - key: tags.Environment
    prefix: env
    separator: "_"

  # Group by Role tag: role_webserver, role_database
  - key: tags.Role
    prefix: role
    separator: "_"

  # Group by AWS region: us_east_1, us_west_2
  - key: placement.region
    prefix: aws_region
    separator: "_"

  # Group by instance type: instance_type_m5_xlarge
  - key: instance_type
    prefix: instance_type
    separator: "_"

# Groups based on conditions
groups:
  # All instances with "database" in their name
  databases: "'database' in tags.Name | default('')"
  
  # All instances in private subnets
  private: "not public_ip_address"
```

```bash
# Verify dynamic inventory
ansible-inventory -i inventory/aws_ec2.yml --list
# Shows all discovered hosts, groups, and variables

ansible-inventory -i inventory/aws_ec2.yml --graph
# Shows group hierarchy as tree

# Test connectivity
ansible -i inventory/aws_ec2.yml all -m ping
# Pings all discovered hosts via SSH
```

### Variable Precedence — The 22-Level Hierarchy

```
Ansible has 22 levels of variable precedence.
You DON'T need to memorize all 22.
You need to know the IMPORTANT ones and the PATTERN.

FROM LOWEST TO HIGHEST PRIORITY:
(only the ones that matter in practice)

 1. Role defaults           (roles/x/defaults/main.yml)     ← WEAKEST
 2. Inventory file group vars
 3. Inventory group_vars/   (group_vars/webservers.yml)
 4. Inventory host_vars/    (host_vars/web-1.yml)
 5. Play vars               (vars: in playbook)
 6. Role vars               (roles/x/vars/main.yml)
 7. Task vars               (vars: on a task)
 8. set_fact / registered vars
 9. Extra vars (-e)         (ansible-playbook -e "var=val")  ← STRONGEST

THE PATTERN:
  More specific = higher priority
  Role defaults < group vars < host vars < play vars < CLI extra vars
  
THE RULE:
  Put DEFAULTS in role defaults (easy to override)
  Put CONSTANTS in role vars (hard to override — intentional)
  Put SECRETS in Ansible Vault (encrypted)
  Put OVERRIDES in -e flag (CLI, highest priority, for emergency)
```

### group_vars and host_vars Directory Structure

```
inventory/
├── hosts.yml                    # Inventory file
├── group_vars/
│   ├── all.yml                  # Applies to ALL hosts
│   │   # common_packages:
│   │   #   - vim
│   │   #   - curl
│   │   #   - jq
│   ├── webservers.yml           # Applies to webservers group
│   │   # nginx_version: "1.25"
│   │   # worker_processes: auto
│   ├── dbservers.yml            # Applies to dbservers group
│   │   # pg_version: 16
│   │   # pg_max_connections: 200
│   ├── production.yml           # Applies to production group
│   │   # env: prod
│   │   # monitoring_enabled: true
│   └── production/              # Directory form (split into files)
│       ├── vars.yml             # Non-sensitive vars
│       └── vault.yml            # Encrypted vars (Ansible Vault)
│           # db_password: !vault |
│           #   $ANSIBLE_VAULT;1.1;AES256
│           #   ...encrypted...
└── host_vars/
    ├── db-primary.yml           # Applies to db-primary only
    │   # pg_role: primary
    │   # pg_wal_level: replica
    └── web-1.yml                # Applies to web-1 only
        # special_config: true
```

---

## 3. Playbooks — What to Do

### Playbook Anatomy

```yaml
# playbooks/site.yml — The master playbook
---
# Play 1: Common configuration for ALL servers
- name: Common baseline configuration
  hosts: all
  become: true    # Run as root (sudo)
  gather_facts: true    # Collect system info (OS, IP, memory, etc.)

  pre_tasks:
    - name: Update apt cache (Debian/Ubuntu)
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600    # Don't update if < 1 hour old
      when: ansible_os_family == "Debian"

  roles:
    - role: common            # Apply common role
    - role: security_hardening
    - role: monitoring_agent

  post_tasks:
    - name: Verify SSH is running
      ansible.builtin.service:
        name: sshd
        state: started
        enabled: true

# Play 2: Web server configuration
- name: Configure web servers
  hosts: webservers
  become: true
  serial: "25%"    # Rolling: only configure 25% of servers at a time
                    # If web-1 fails, web-2/3/4 don't get touched

  pre_tasks:
    - name: Drain from load balancer
      ansible.builtin.uri:
        url: "http://{{ lb_api }}/drain/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost    # Run on control node, not target

  roles:
    - role: nginx
      vars:
        nginx_version: "{{ nginx_version | default('1.25') }}"
    - role: app_deploy

  post_tasks:
    - name: Re-enable in load balancer
      ansible.builtin.uri:
        url: "http://{{ lb_api }}/enable/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost

    - name: Verify application health
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}:{{ http_port }}/health"
        status_code: 200
        timeout: 30
      retries: 5
      delay: 10
      register: health_check
      until: health_check.status == 200

# Play 3: Database configuration
- name: Configure database servers
  hosts: dbservers
  become: true
  serial: 1    # ONE AT A TIME for databases (never parallel)

  roles:
    - role: postgresql
```

### Key Modules — The Essential Toolkit

```yaml
# ── PACKAGE MANAGEMENT ──
- name: Install packages (apt)
  ansible.builtin.apt:
    name:
      - nginx
      - python3-pip
      - jq
      - curl
    state: present    # present (install), absent (remove), latest (upgrade)
    update_cache: true

- name: Install packages (yum/dnf)
  ansible.builtin.dnf:
    name:
      - httpd
      - python3
    state: present

# ── FILE MANAGEMENT ──
- name: Create directory
  ansible.builtin.file:
    path: /opt/novamart/config
    state: directory
    owner: appuser
    group: appgroup
    mode: '0755'

- name: Copy file from control node to target
  ansible.builtin.copy:
    src: files/nginx.conf        # Local file on control node
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: true    # Create backup before overwriting
  notify: Restart nginx    # Trigger handler if file changed

- name: Deploy template (Jinja2)
  ansible.builtin.template:
    src: templates/app.conf.j2
    dest: /etc/novamart/app.conf
    owner: appuser
    mode: '0640'
    validate: /usr/sbin/nginx -t -c %s    # Validate BEFORE deploying
  notify: Reload nginx

- name: Ensure line in file
  ansible.builtin.lineinfile:
    path: /etc/sysctl.conf
    regexp: '^net.core.somaxconn'
    line: 'net.core.somaxconn = 65535'
    state: present
  notify: Reload sysctl

- name: Add block to file
  ansible.builtin.blockinfile:
    path: /etc/hosts
    block: |
      10.0.2.11 web-1.novamart.internal
      10.0.2.12 web-2.novamart.internal
      10.0.2.13 web-3.novamart.internal
    marker: "# {mark} ANSIBLE MANAGED - NovaMart hosts"

# ── SERVICE MANAGEMENT ──
- name: Ensure nginx is running and enabled
  ansible.builtin.service:
    name: nginx
    state: started    # started, stopped, restarted, reloaded
    enabled: true     # Start on boot

- name: Manage systemd unit
  ansible.builtin.systemd:
    name: node_exporter
    state: started
    enabled: true
    daemon_reload: true    # Reload systemd if unit file changed

# ── USER MANAGEMENT ──
- name: Create application user
  ansible.builtin.user:
    name: appuser
    uid: 1001
    group: appgroup
    shell: /bin/bash
    home: /opt/novamart
    create_home: true
    system: true    # System account (no login)

# ── COMMAND EXECUTION ──
- name: Run a command
  ansible.builtin.command:
    cmd: /opt/novamart/bin/migrate.sh
    creates: /opt/novamart/.migrated    # Skip if this file exists (idempotent)
  register: migration_result

- name: Run shell command (with pipes, redirects)
  ansible.builtin.shell:
    cmd: "ps aux | grep nginx | wc -l"
  register: nginx_procs
  changed_when: false    # This command never "changes" anything

# command vs shell:
# command: safer, no shell interpretation (no pipes, no wildcards)
# shell: full shell features but risk of injection if using variables

# ── DOWNLOADS ──
- name: Download file
  ansible.builtin.get_url:
    url: "https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz"
    dest: /tmp/node_exporter.tar.gz
    checksum: "sha256:a550cd5c05f760b7934a2d0afad66d2e92e681482f5f57a917465b1fba3b02a6"
    mode: '0644'

- name: Extract archive
  ansible.builtin.unarchive:
    src: /tmp/node_exporter.tar.gz
    dest: /opt/
    remote_src: true    # File is already on the remote host

# ── WAIT CONDITIONS ──
- name: Wait for port to be available
  ansible.builtin.wait_for:
    host: "{{ ansible_host }}"
    port: 8080
    delay: 5
    timeout: 60
    state: started

- name: Wait for file to appear
  ansible.builtin.wait_for:
    path: /opt/novamart/.ready
    timeout: 120

# ── AWS MODULES ──
- name: Get EC2 instance facts
  amazon.aws.ec2_instance_info:
    filters:
      "tag:Environment": production
      "tag:Role": webserver
    region: us-east-1
  register: ec2_instances

- name: Create S3 bucket
  amazon.aws.s3_bucket:
    name: novamart-ansible-artifacts
    state: present
    region: us-east-1
    versioning: true
    encryption: AES256
```

### Handlers — React to Changes

```yaml
# Handlers are tasks that only run when NOTIFIED
# They run ONCE at the end of the play, regardless of how many tasks notify them

# In the playbook or role:
handlers:
  - name: Restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
    listen: "restart web server"    # Can be triggered by this alias too

  - name: Reload nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded

  - name: Reload sysctl
    ansible.builtin.command:
      cmd: sysctl -p

  - name: Restart node_exporter
    ansible.builtin.systemd:
      name: node_exporter
      state: restarted
      daemon_reload: true

# IMPORTANT HANDLER BEHAVIOR:
# 1. Handlers run at END of play, not immediately after task
#    → If task 3 notifies "Restart nginx" and task 8 fails,
#      nginx STILL gets restarted (handler already queued)
#    → Use meta: flush_handlers to run handlers immediately if needed
#
# 2. Handlers run ONCE even if notified multiple times
#    → Tasks 3, 5, and 7 all notify "Restart nginx"
#      → nginx restarted ONCE at end of play
#
# 3. Handlers do NOT run if play fails and --force-handlers not set
#    → Use --force-handlers to run handlers even on failure
```

### Jinja2 Templates

```jinja2
{# templates/nginx.conf.j2 #}
# Managed by Ansible — DO NOT EDIT MANUALLY
# Generated: {{ ansible_date_time.iso8601 }}
# Host: {{ inventory_hostname }}

user  nginx;
worker_processes  {{ ansible_processor_vcpus }};
pid   /run/nginx.pid;

events {
    worker_connections  {{ max_connections | default(1024) }};
}

http {
    # Upstream backends
    upstream app {
{% for host in groups['webservers'] %}
        server {{ hostvars[host]['ansible_host'] }}:{{ http_port }};
{% endfor %}
    }

    server {
        listen {{ http_port | default(80) }};
        server_name {{ server_name | default('_') }};

{% if ssl_enabled | default(false) %}
        listen 443 ssl;
        ssl_certificate     /etc/ssl/{{ ssl_cert_name }}.crt;
        ssl_certificate_key /etc/ssl/{{ ssl_cert_name }}.key;
{% endif %}

        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

{% if env == 'prod' %}
        # Production-only: rate limiting
        limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
        location /api/ {
            limit_req zone=api burst=50 nodelay;
            proxy_pass http://app;
        }
{% endif %}

        location /health {
            access_log off;
            return 200 'OK';
        }
    }
}
```

### Conditionals and Loops

```yaml
# ── CONDITIONALS ──
- name: Install on Debian only
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"

- name: Install on RHEL only
  ansible.builtin.dnf:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"

# Multiple conditions (AND)
- name: Configure production web servers
  ansible.builtin.template:
    src: prod.conf.j2
    dest: /etc/app/prod.conf
  when:
    - env == "prod"
    - inventory_hostname in groups['webservers']

# OR condition
- name: Alert if disk or memory critical
  ansible.builtin.debug:
    msg: "Resource critical on {{ inventory_hostname }}"
  when: >
    (ansible_memfree_mb < 512) or
    (ansible_mounts | selectattr('mount', 'equalto', '/') | map(attribute='size_available') | first < 1073741824)

# ── LOOPS ──
- name: Create multiple users
  ansible.builtin.user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - { name: deploy, uid: 1001, groups: "sudo,docker" }
    - { name: monitor, uid: 1002, groups: "docker" }
    - { name: backup, uid: 1003, groups: "" }

- name: Install multiple packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop: "{{ common_packages }}"
  # Better: pass list directly to name (fewer SSH round-trips)
  # ansible.builtin.apt:
  #   name: "{{ common_packages }}"
  #   state: present

# Loop with dict
- name: Create config files for each service
  ansible.builtin.template:
    src: "service.conf.j2"
    dest: "/etc/novamart/{{ item.key }}.conf"
    mode: '0644'
  loop: "{{ services | dict2items }}"
  # services:
  #   order-service: { port: 8080, workers: 4 }
  #   cart-service: { port: 8081, workers: 2 }
  notify: Restart services

# ── REGISTER & CONDITIONALS ──
- name: Check if application is installed
  ansible.builtin.stat:
    path: /opt/novamart/bin/app
  register: app_binary

- name: Install application if missing
  ansible.builtin.get_url:
    url: "https://releases.novamart.com/app-{{ app_version }}.tar.gz"
    dest: /tmp/app.tar.gz
  when: not app_binary.stat.exists
```

### Error Handling

```yaml
- name: Deploy with error handling
  block:
    # TRY block
    - name: Deploy new application version
      ansible.builtin.copy:
        src: "app-{{ app_version }}.jar"
        dest: /opt/novamart/app.jar
      notify: Restart application

    - name: Run database migration
      ansible.builtin.command:
        cmd: /opt/novamart/bin/migrate.sh
      register: migration

    - name: Verify health
      ansible.builtin.uri:
        url: "http://localhost:{{ http_port }}/health"
        status_code: 200
      retries: 5
      delay: 10

  rescue:
    # CATCH block — runs if ANY task in block fails
    - name: Rollback to previous version
      ansible.builtin.copy:
        src: "/opt/novamart/backup/app.jar"
        dest: /opt/novamart/app.jar
      notify: Restart application

    - name: Alert on deployment failure
      ansible.builtin.uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "❌ Deployment failed on {{ inventory_hostname }}: {{ ansible_failed_result.msg | default('unknown') }}"
      delegate_to: localhost

  always:
    # FINALLY block — runs regardless
    - name: Clean up temp files
      ansible.builtin.file:
        path: /tmp/deploy-staging/
        state: absent

    - name: Log deployment attempt
      ansible.builtin.lineinfile:
        path: /var/log/deployments.log
        line: "{{ ansible_date_time.iso8601 }} - {{ app_version }} - {{ 'SUCCESS' if not ansible_failed_task else 'FAILED' }}"
        create: true
```

---

## 4. Roles — The Reuse Mechanism

### Role Structure

```
roles/
└── nginx/
    ├── defaults/           # Default variables (LOWEST priority — easy to override)
    │   └── main.yml
    │       # nginx_version: "1.25"
    │       # nginx_worker_processes: auto
    │       # nginx_worker_connections: 1024
    │       # nginx_keepalive_timeout: 65
    │
    ├── vars/               # Role variables (HIGH priority — hard to override)
    │   └── main.yml
    │       # nginx_user: www-data  (Debian)
    │       # nginx_conf_path: /etc/nginx/nginx.conf
    │       # nginx_pid_path: /run/nginx.pid
    │
    ├── tasks/              # Task files
    │   ├── main.yml        # Entry point (includes others)
    │   ├── install.yml
    │   ├── configure.yml
    │   └── verify.yml
    │
    ├── handlers/           # Handler definitions
    │   └── main.yml
    │       # - name: Restart nginx
    │       #   service: name=nginx state=restarted
    │       # - name: Reload nginx
    │       #   service: name=nginx state=reloaded
    │
    ├── templates/          # Jinja2 templates
    │   ├── nginx.conf.j2
    │   └── default.conf.j2
    │
    ├── files/              # Static files (copied as-is)
    │   └── custom-error-pages/
    │       ├── 404.html
    │       └── 502.html
    │
    ├── meta/               # Role metadata
    │   └── main.yml
    │       # dependencies:
    │       #   - role: common
    │       #   - role: security_hardening
    │
    └── molecule/           # Tests (Molecule framework)
        └── default/
            ├── molecule.yml
            ├── converge.yml
            └── verify.yml
```

### Production Role Example — Node Exporter

```yaml
# roles/node_exporter/defaults/main.yml
---
node_exporter_version: "1.7.0"
node_exporter_user: node_exporter
node_exporter_group: node_exporter
node_exporter_port: 9100
node_exporter_listen_address: "0.0.0.0:{{ node_exporter_port }}"
node_exporter_textfile_dir: /var/lib/node_exporter/textfile
node_exporter_enabled_collectors:
  - systemd
  - textfile
  - filesystem
  - diskstats
  - netdev
  - meminfo
  - cpu
node_exporter_disabled_collectors:
  - infiniband
  - nfs
  - zfs
```

```yaml
# roles/node_exporter/tasks/main.yml
---
- name: Include install tasks
  ansible.builtin.include_tasks: install.yml

- name: Include configure tasks
  ansible.builtin.include_tasks: configure.yml

- name: Include verify tasks
  ansible.builtin.include_tasks: verify.yml
  tags: [verify]
```

```yaml
# roles/node_exporter/tasks/install.yml
---
- name: Create node_exporter group
  ansible.builtin.group:
    name: "{{ node_exporter_group }}"
    system: true

- name: Create node_exporter user
  ansible.builtin.user:
    name: "{{ node_exporter_user }}"
    group: "{{ node_exporter_group }}"
    shell: /usr/sbin/nologin
    system: true
    create_home: false

- name: Create textfile collector directory
  ansible.builtin.file:
    path: "{{ node_exporter_textfile_dir }}"
    state: directory
    owner: "{{ node_exporter_user }}"
    group: "{{ node_exporter_group }}"
    mode: '0755'

- name: Check current version
  ansible.builtin.command:
    cmd: /usr/local/bin/node_exporter --version
  register: current_version
  changed_when: false
  failed_when: false

- name: Download node_exporter
  ansible.builtin.get_url:
    url: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
    dest: "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
    checksum: "sha256:{{ node_exporter_checksum }}"
    mode: '0644'
  when: >
    current_version.rc != 0 or
    node_exporter_version not in current_version.stderr | default('')

- name: Extract node_exporter
  ansible.builtin.unarchive:
    src: "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
    dest: /tmp/
    remote_src: true
  when: >
    current_version.rc != 0 or
    node_exporter_version not in current_version.stderr | default('')

- name: Install node_exporter binary
  ansible.builtin.copy:
    src: "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
    dest: /usr/local/bin/node_exporter
    owner: root
    group: root
    mode: '0755'
    remote_src: true
  notify: Restart node_exporter
```

```yaml
# roles/node_exporter/tasks/configure.yml
---
- name: Deploy systemd unit file
  ansible.builtin.template:
    src: node_exporter.service.j2
    dest: /etc/systemd/system/node_exporter.service
    owner: root
    group: root
    mode: '0644'
  notify: Restart node_exporter

- name: Ensure node_exporter is started and enabled
  ansible.builtin.systemd:
    name: node_exporter
    state: started
    enabled: true
    daemon_reload: true
```

```yaml
# roles/node_exporter/tasks/verify.yml
---
- name: Verify node_exporter is listening
  ansible.builtin.wait_for:
    port: "{{ node_exporter_port }}"
    timeout: 30

- name: Verify metrics endpoint
  ansible.builtin.uri:
    url: "http://localhost:{{ node_exporter_port }}/metrics"
    return_content: true
  register: metrics_response
  failed_when: "'node_cpu_seconds_total' not in metrics_response.content"
```

```jinja2
{# roles/node_exporter/templates/node_exporter.service.j2 #}
# Managed by Ansible — DO NOT EDIT
[Unit]
Description=Prometheus Node Exporter
Documentation=https://prometheus.io/docs/guides/node-exporter/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User={{ node_exporter_user }}
Group={{ node_exporter_group }}
ExecStart=/usr/local/bin/node_exporter \
    --web.listen-address={{ node_exporter_listen_address }} \
    --collector.textfile.directory={{ node_exporter_textfile_dir }} \
{% for collector in node_exporter_enabled_collectors %}
    --collector.{{ collector }} \
{% endfor %}
{% for collector in node_exporter_disabled_collectors %}
    --no-collector.{{ collector }} \
{% endfor %}
    --log.level=info

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```yaml
# roles/node_exporter/handlers/main.yml
---
- name: Restart node_exporter
  ansible.builtin.systemd:
    name: node_exporter
    state: restarted
    daemon_reload: true
```

### Ansible Galaxy — Role Distribution

```bash
# Install community roles
ansible-galaxy install geerlingguy.docker
ansible-galaxy install geerlingguy.postgresql

# Install from requirements file (like package.json)
# requirements.yml
---
roles:
  - name: geerlingguy.docker
    version: 7.1.0
  - name: geerlingguy.postgresql
    version: 3.4.0
  - name: novamart.common
    src: git+https://bitbucket.org/novamart/ansible-role-common.git
    version: v2.3.0

collections:
  - name: amazon.aws
    version: ">=7.0.0"
  - name: community.general
    version: ">=8.0.0"

# Install all
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml

# Pin versions! Same reason as Terraform modules and Jenkins libraries.
```

---

## 5. How Ansible Breaks — Production Failure Modes

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **SSH timeout** | `UNREACHABLE! => connection timed out` | SG doesn't allow SSH, wrong private key, host down | Check SG rule for port 22, verify key, test: `ssh -i key user@host` |
| **Python missing** | `MODULE FAILURE: No python interpreter found` | Target has no Python (minimal AMI, Alpine container) | Install Python in user_data/bootstrap, or use `raw` module first |
| **Become failure** | `Missing sudo password` or `sudo: not found` | User not in sudoers, `requiretty` set, no NOPASSWD | Add to sudoers: `deploy ALL=(ALL) NOPASSWD:ALL`, disable requiretty |
| **Idempotency broken** | Task shows `changed` every run | Using `command`/`shell` without `creates`/`changed_when`, template has `{{ ansible_date_time }}` | Add `changed_when: false` for read-only commands, remove dynamic content from templates |
| **Handler not running** | Config updated but service not restarted | Handler name typo, handler in wrong file, play failed before handlers | Exact handler name match, `meta: flush_handlers`, `--force-handlers` |
| **Variable collision** | Wrong value applied | Same variable name in group_vars and role, precedence confusion | Use unique variable names prefixed by role: `nginx_port`, not `port` |
| **Slow execution** | 30 min for 50 hosts | Serial: 1, no pipelining, forks: 5, gathering facts every play | Increase forks, enable pipelining, `gather_facts: false` where unneeded, use `serial: "25%"` |
| **Template rendering fails** | `UndefinedError: 'var' is undefined` | Variable not defined for some hosts, missing default filter | Use `{{ var | default('fallback') }}` in templates |
| **Partial failure** | 3 of 10 hosts failed | Network blip, package repo down on those hosts | Review `.retry` file, use `--limit @playbook.retry`, investigate failed hosts |
| **Secret leak** | Password in plaintext in Ansible output | `debug: var=db_password`, or `-v` shows task args | Use `no_log: true` on tasks with secrets, don't debug sensitive vars |

---

## Quick Reference Card

```
ANSIBLE ARCHITECTURE:
  Control node → SSH → Managed nodes (agentless)
  No daemon on targets. Just Python + SSH.
  Copies Python module → executes → reads JSON output → cleans up

ANSIBLE.CFG:
  forks: 20 (parallel connections)
  pipelining: True (3-5x speed boost)
  ssh_args: ControlMaster=auto ControlPersist=60s
  gather_facts: disable when not needed

INVENTORY:
  Static: hosts.yml (YAML preferred over INI)
  Dynamic: aws_ec2.yml plugin (auto-discover from AWS tags)
  group_vars/GROUP.yml → variables for group
  host_vars/HOST.yml → variables for specific host
  Verify: ansible-inventory --list, ansible all -m ping

VARIABLE PRECEDENCE (simplified):
  role defaults < group_vars < host_vars < play vars < role vars < -e CLI
  Defaults = easy to override. Vars = hard to override. -e = always wins.

PLAYBOOK:
  hosts: target group
  become: true (sudo)
  serial: "25%" (rolling update)
  pre_tasks → roles → tasks → post_tasks → handlers
  
KEY MODULES:
  apt/dnf (packages), file (dirs/perms), copy (static files),
  template (jinja2), service/systemd (daemons), 
  command/shell (commands), user/group (accounts),
  uri (HTTP), wait_for (port/file), get_url (download)

HANDLERS:
  Triggered by "notify" keyword
  Run ONCE at END of play (not immediately)
  flush_handlers to run early
  --force-handlers to run even on failure

TEMPLATES (Jinja2):
  {{ variable }}, {% for %}, {% if %}
  {{ var | default('fallback') }}
  Validate before deploy: validate: /usr/sbin/nginx -t -c %s

ROLES:
  defaults/ (overridable) vs vars/ (constants)
  tasks/main.yml (entry point)
  handlers/main.yml
  templates/ (Jinja2) vs files/ (static)
  meta/main.yml (dependencies)
  Pin versions in requirements.yml

ERROR HANDLING:
  block/rescue/always (try/catch/finally)
  retries + delay + until (retry with condition)
  changed_when: false (idempotent command)
  no_log: true (hide secrets in output)
  failed_when: (custom failure condition)
```

---

## Retention Questions

**Q1.** NovaMart has 50 EC2 instances across 3 regions tagged with `Role: webserver`, `Role: database`, and `Role: monitoring`. Write the complete dynamic inventory configuration (AWS EC2 plugin) that: discovers all instances, groups them by Role and Region, sets the correct `ansible_user` based on OS (Amazon Linux = ec2-user, Ubuntu = ubuntu), and connects via private IP through a bastion host. Show the inventory YAML and the ansible.cfg needed.

**Q2.** You need to deploy a new version of Node Exporter to all 50 servers with zero downtime. The current version is 1.6.0 and you're upgrading to 1.7.0. Write the complete playbook that: checks current version, skips hosts already on 1.7.0, downloads and installs the new binary, updates the systemd unit file, restarts the service, and verifies the metrics endpoint is responding. Use proper error handling (block/rescue) and rolling deployment (serial).

**Q3.** An engineer runs a playbook and reports: "It takes 45 minutes to configure 50 hosts." You investigate and find: `forks: 5`, `gather_facts: true` on every play, no pipelining, `serial: 1`, and every task uses `ansible.builtin.command` instead of native modules. Explain each performance problem, provide the fix, and estimate the new runtime.

**Q4.** Explain the difference between `ansible.builtin.command`, `ansible.builtin.shell`, and `ansible.builtin.raw`. For each, give a NovaMart scenario where it's the right choice and a scenario where it's wrong. Then explain why using `command`/`shell` without `changed_when` or `creates` breaks idempotency, and show how to fix it.


# Phase 4, Lesson 4 — Ansible Retention Answers

---

## Q1: Dynamic Inventory with AWS EC2 Plugin

### The Inventory File

```yaml
# inventory/aws_ec2.yml
# Filename MUST end in aws_ec2.yml or aws_ec2.yaml for the plugin to auto-detect
plugin: amazon.aws.aws_ec2

# Discover across all 3 NovaMart regions
regions:
  - us-east-1
  - us-west-2
  - eu-west-1

# Only discover running instances with a Role tag
filters:
  tag:Role:
    - webserver
    - database
    - monitoring
  instance-state-name: running

# Use private IP — we're going through a bastion, not public internet
hostnames:
  - private-ip-address

# Compose variables per-host based on instance metadata
compose:
  # Set ansible_user based on the AMI's platform
  # Amazon Linux AMIs have "amzn" in the image_id description; Ubuntu has "ubuntu" in tags or platform_details
  ansible_user: >-
    {{
      'ubuntu' if 'ubuntu' in (platform_details | default('') | lower) or 'ubuntu' in (tags.get('OS', '') | lower)
      else 'ec2-user'
    }}
  # Private IP for connection
  ansible_host: private_ip_address
  # Region as a variable on every host
  ec2_region: placement.region

# Keyed groups: creates Ansible groups from instance metadata
keyed_groups:
  # Group by Role tag → groups: Role_webserver, Role_database, Role_monitoring
  - key: tags.Role
    prefix: role
    separator: "_"

  # Group by region → groups: region_us_east_1, region_us_west_2, region_eu_west_1
  - key: placement.region | regex_replace('-', '_')
    prefix: region
    separator: "_"

  # Group by OS for targeting
  - key: >-
      'ubuntu' if 'ubuntu' in (platform_details | default('') | lower) or 'ubuntu' in (tags.get('OS', '') | lower)
      else 'amazon_linux'
    prefix: os
    separator: "_"

# Optional: set strict to True so misconfigurations fail loudly instead of silently skipping
strict: true
```

**Why the `compose` section for `ansible_user` is tricky:** EC2 metadata doesn't have a clean "OS" field. The `platform_details` field may say `"Linux/UNIX"` for both Amazon Linux and Ubuntu. The reliable approaches are:

1. **Require an `OS` tag on every instance** (enforced via Terraform) — cleanest
2. **Use the AMI ID** to look up the image description — accurate but slow (extra API call per host)
3. **Use `platform_details` + tag fallback** — what I've shown above

For production, I'd enforce the `OS` tag via Terraform and simplify:

```yaml
compose:
  ansible_user: "'ubuntu' if tags.OS == 'Ubuntu' else 'ec2-user'"
```

### The ansible.cfg

```ini
# ansible.cfg — placed in the project root (or /etc/ansible/ansible.cfg)
[defaults]
# Point to the dynamic inventory file
inventory = inventory/aws_ec2.yml

# SSH key for EC2 instances
private_key_file = ~/.ssh/novamart-prod.pem

# Don't prompt for host key verification on first connect
# (In prod, use ssh_args with known_hosts instead)
host_key_checking = False

# Performance: run against 20 hosts in parallel (not the default 5)
forks = 20

# Performance: enable pipelining (reduces SSH round-trips per task)
# Requires requiretty to be disabled in /etc/sudoers on targets
pipelining = True

# Reduce fact-gathering overhead — cache facts for 24 hours
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_fact_cache
fact_caching_timeout = 86400

# Callback plugin for human-readable output
stdout_callback = yaml

# Retry files clutter the repo
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root

[ssh_connection]
# Bastion / Jump host configuration
# ALL connections route through the bastion's public IP
ssh_args = -o ProxyCommand="ssh -W %h:%p -q -i ~/.ssh/novamart-bastion.pem ec2-user@bastion.novamart.internal" -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no

# Use SCP for file transfers if SFTP fails (some hardened AMIs disable SFTP)
transfer_method = smart

# Speed: reuse SSH connections
control_path_dir = ~/.ansible/cp
```

**How the bastion connection works step by step:**

```
Control Node                  Bastion (public IP)           Target (private IP)
─────────────                 ───────────────────           ───────────────────
ansible-playbook runs
  └─→ SSH to bastion ────────→ bastion receives conn
       (ProxyCommand)            └─→ SSH forward to ──────→ target receives conn
                                      target private IP     └─→ executes module
                                                            └─→ returns result
                                                                  ↑
                                 ControlPersist=60s keeps this tunnel open
                                 for subsequent tasks (no re-auth per task)
```

### Verification

```bash
# Test that inventory discovers all 50 hosts
ansible-inventory -i inventory/aws_ec2.yml --graph

# Expected output:
# @all:
#   |--@role_webserver:
#   |  |--10.0.1.15
#   |  |--10.0.1.22
#   |  |--...
#   |--@role_database:
#   |  |--10.0.2.5
#   |  |--...
#   |--@role_monitoring:
#   |  |--10.0.3.8
#   |--@region_us_east_1:
#   |  |--10.0.1.15
#   |  |--...
#   |--@os_ubuntu:
#   |  |--10.0.1.15
#   |--@os_amazon_linux:
#   |  |--10.0.2.5

# Test connectivity through bastion
ansible all -m ping -i inventory/aws_ec2.yml --limit role_webserver

# Test that ansible_user is set correctly per OS
ansible-inventory -i inventory/aws_ec2.yml --host 10.0.1.15 | grep ansible_user
# Should show: "ansible_user": "ubuntu" (for Ubuntu hosts)
```

---

## Q2: Rolling Node Exporter Upgrade with Zero Downtime

```yaml
---
# playbook: upgrade_node_exporter.yml
# Purpose: Rolling upgrade of Node Exporter 1.6.0 → 1.7.0 across all 50 servers
# Safety: serial deployment, version pre-check, health verification, automatic rollback

- name: Rolling upgrade of Node Exporter to 1.7.0
  hosts: all
  become: true
  # Rolling deployment: 5 hosts at a time (10% of fleet)
  # If a batch fails, remaining batches are skipped — limits blast radius
  serial: 5
  # max_fail_percentage: if >20% of a batch fails, abort entire play
  max_fail_percentage: 20

  vars:
    node_exporter_version: "1.7.0"
    node_exporter_old_version: "1.6.0"
    node_exporter_url: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
    node_exporter_checksum: "sha256:PUT_ACTUAL_CHECKSUM_HERE"
    node_exporter_binary: "/usr/local/bin/node_exporter"
    node_exporter_port: 9100
    install_dir: "/tmp/node_exporter_upgrade"

  # Don't gather full facts — we don't need them and it saves ~2s per host
  gather_facts: false

  pre_tasks:
    - name: Gather only minimal facts (just OS family for conditional logic)
      ansible.builtin.setup:
        gather_subset:
          - '!all'
          - '!min'
          - distribution

    - name: Check current Node Exporter version
      ansible.builtin.command:
        cmd: "{{ node_exporter_binary }} --version"
      register: current_version
      changed_when: false  # This is a read-only check — never report "changed"
      failed_when: false   # If node_exporter isn't installed, don't fail here

    - name: Set version fact
      ansible.builtin.set_fact:
        installed_version: >-
          {{ current_version.stdout | default('') | regex_search('version (\d+\.\d+\.\d+)', '\1') | first | default('unknown') }}

    - name: Skip hosts already on target version
      ansible.builtin.meta: end_host
      when: installed_version == node_exporter_version

  tasks:
    - name: Upgrade Node Exporter with rollback on failure
      block:
        # === BACKUP ===
        - name: Backup current binary
          ansible.builtin.copy:
            src: "{{ node_exporter_binary }}"
            dest: "{{ node_exporter_binary }}.{{ node_exporter_old_version }}.bak"
            remote_src: true
            mode: '0755'

        # === DOWNLOAD ===
        - name: Create temp install directory
          ansible.builtin.file:
            path: "{{ install_dir }}"
            state: directory
            mode: '0755'

        - name: Download Node Exporter {{ node_exporter_version }}
          ansible.builtin.get_url:
            url: "{{ node_exporter_url }}"
            dest: "{{ install_dir }}/node_exporter.tar.gz"
            checksum: "{{ node_exporter_checksum }}"
            mode: '0644'
            timeout: 30

        - name: Extract binary
          ansible.builtin.unarchive:
            src: "{{ install_dir }}/node_exporter.tar.gz"
            dest: "{{ install_dir }}"
            remote_src: true

        # === INSTALL ===
        - name: Stop Node Exporter before binary swap
          ansible.builtin.systemd:
            name: node_exporter
            state: stopped

        - name: Install new binary
          ansible.builtin.copy:
            src: "{{ install_dir }}/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
            dest: "{{ node_exporter_binary }}"
            remote_src: true
            owner: root
            group: root
            mode: '0755'

        - name: Deploy updated systemd unit file
          ansible.builtin.template:
            src: templates/node_exporter.service.j2
            dest: /etc/systemd/system/node_exporter.service
            owner: root
            group: root
            mode: '0644'
          notify: reload systemd

        # Force handler to run NOW, not at end of play
        - name: Force systemd reload before restart
          ansible.builtin.meta: flush_handlers

        # === RESTART AND VERIFY ===
        - name: Start Node Exporter
          ansible.builtin.systemd:
            name: node_exporter
            state: started
            enabled: true

        - name: Wait for metrics endpoint to be responsive
          ansible.builtin.uri:
            url: "http://localhost:{{ node_exporter_port }}/metrics"
            method: GET
            status_code: 200
            return_content: true
          register: metrics_response
          retries: 5
          delay: 3
          until: metrics_response.status == 200

        - name: Verify new version is running
          ansible.builtin.command:
            cmd: "{{ node_exporter_binary }} --version"
          register: post_upgrade_version
          changed_when: false
          failed_when: "node_exporter_version not in post_upgrade_version.stdout"

        # === CLEANUP ===
        - name: Remove temp install directory
          ansible.builtin.file:
            path: "{{ install_dir }}"
            state: absent

      rescue:
        # === AUTOMATIC ROLLBACK ===
        - name: "ROLLBACK: Restore previous binary"
          ansible.builtin.copy:
            src: "{{ node_exporter_binary }}.{{ node_exporter_old_version }}.bak"
            dest: "{{ node_exporter_binary }}"
            remote_src: true
            mode: '0755'

        - name: "ROLLBACK: Restart Node Exporter with old binary"
          ansible.builtin.systemd:
            name: node_exporter
            state: restarted
            daemon_reload: true

        - name: "ROLLBACK: Verify old version is serving metrics"
          ansible.builtin.uri:
            url: "http://localhost:{{ node_exporter_port }}/metrics"
            status_code: 200
          retries: 3
          delay: 2

        - name: "ROLLBACK: Fail the host so serial batch accounting works"
          ansible.builtin.fail:
            msg: >
              Upgrade failed on {{ inventory_hostname }}.
              Rolled back to {{ node_exporter_old_version }}.
              Investigate before continuing.

  handlers:
    - name: reload systemd
      ansible.builtin.systemd:
        daemon_reload: true
```

**The systemd template:**

```ini
# templates/node_exporter.service.j2
[Unit]
Description=Prometheus Node Exporter v{{ node_exporter_version }}
Documentation=https://prometheus.io/docs/guides/node-exporter/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=node_exporter
Group=node_exporter
ExecStart={{ node_exporter_binary }} \
  --web.listen-address=:{{ node_exporter_port }} \
  --collector.filesystem.mount-points-exclude="^/(sys|proc|dev|host|etc)($$|/)" \
  --collector.netclass.ignored-devices="^(veth.*|docker.*|br-.*)$$"
Restart=always
RestartSec=5
SyslogIdentifier=node_exporter

# Hardening
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadOnlyPaths=/

[Install]
WantedBy=multi-user.target
```

**Why `serial: 5` and not `serial: 1` or `serial: 50`:**

```
serial: 1   → 50 hosts × ~30s each = 25 minutes. Safe but slow.
serial: 5   → 10 batches × ~30s = 5 minutes. 90% of fleet still healthy if a batch fails.
serial: 50  → All at once. If the new binary is broken, ALL monitoring is down. Never do this.
serial: "20%" → Also valid. Scales automatically with fleet size.
```

**The zero-downtime guarantee:** Node Exporter is stopped for approximately 2-3 seconds during binary swap and restart. Prometheus scrape intervals are typically 15-30 seconds. A 3-second gap in a 15-second scrape interval means at most one missed scrape — Prometheus handles this with staleness markers. No alerts fire for a single missed scrape. If your Prometheus alerting rules use `for: 5m`, you'd need the exporter down for 5+ minutes to trigger. This is why the `uri` health check with retries is critical — it catches a broken binary before moving to the next batch.

---

## Q3: Diagnosing and Fixing the 45-Minute Playbook

### Problem-by-Problem Breakdown

**Problem 1: `forks: 5` (default)**

```
What's happening:
  Ansible processes only 5 hosts in parallel. With 50 hosts, that's 10 sequential
  batches per task. If each task takes 2 seconds per host:
  
  50 hosts ÷ 5 forks = 10 rounds per task
  10 rounds × 2 seconds = 20 seconds per task
  vs.
  50 hosts ÷ 25 forks = 2 rounds per task
  2 rounds × 2 seconds = 4 seconds per task

Fix in ansible.cfg:
  [defaults]
  forks = 25
  # Why not 50? Your control node needs RAM per fork (~50-100MB each).
  # 25 forks ≈ 2.5GB RAM. 50 forks ≈ 5GB. Match to your CI runner capacity.

Impact: ~5x throughput improvement on parallel tasks.
```

**Problem 2: `gather_facts: true` on every play**

```
What's happening:
  gather_facts runs the 'setup' module on every host at the start of EVERY play.
  This SSHes into each host, executes ~30 system commands (uname, df, ip addr, 
  lsb_release, etc.), collects the output, and returns it as JSON.
  
  Time per host: 2-5 seconds
  50 hosts × 3 seconds × (let's say 4 plays) = 600 seconds = 10 minutes
  Just on fact gathering.

Fix — Option A: Disable where not needed:
  - name: Install packages
    hosts: all
    gather_facts: false   # We don't use any facts in this play
    tasks: [...]

Fix — Option B: Gather once, cache, and reuse:
  # ansible.cfg
  [defaults]
  gathering = smart                            # Only gather if not cached
  fact_caching = jsonfile                      # Cache backend
  fact_caching_connection = /tmp/facts_cache   # Cache location
  fact_caching_timeout = 3600                  # 1 hour TTL

Fix — Option C: Gather only what you need:
  - name: Deploy config
    hosts: all
    gather_facts: false
    pre_tasks:
      - ansible.builtin.setup:
          gather_subset:
            - '!all'        # Skip everything
            - network        # Only gather network facts (for ansible_default_ipv4)

Impact: Saves 8-10 minutes across 4 plays.
```

**Problem 3: No pipelining**

```
What's happening:
  Without pipelining, for EVERY task on EVERY host, Ansible:
    1. Opens SSH connection (or reuses ControlPersist)
    2. Creates a temp directory on remote: SSH round-trip #1
    3. SFTPs the module Python code to remote: SSH round-trip #2  
    4. Makes the module executable (chmod): SSH round-trip #3
    5. Executes the module: SSH round-trip #4
    6. Deletes the temp file: SSH round-trip #5
  
  That's 5 round-trips per task per host.
  
  With pipelining:
    1. Opens SSH connection
    2. Pipes the module code directly into the Python interpreter via stdin
  
  That's 1 round-trip per task per host.

Fix in ansible.cfg:
  [ssh_connection]
  pipelining = True

Prerequisite on targets:
  # /etc/sudoers must NOT have 'requiretty' enabled
  # If it does, pipelining breaks because sudo can't allocate a TTY over a pipe
  # Check with:
  ansible all -m command -a "grep requiretty /etc/sudoers" --become
  
  # Fix with (run ONCE before enabling pipelining):
  ansible all -m lineinfile -a "path=/etc/sudoers regexp='requiretty' state=absent" --become

Impact: 3-5x faster per task due to reduced SSH overhead.
```

**Problem 4: `serial: 1`**

```
What's happening:
  serial: 1 means Ansible processes ONE host at a time, SEQUENTIALLY.
  This COMPLETELY NEGATES the forks setting.
  
  Even with forks: 50, serial: 1 means:
    - Take host 1
    - Run ALL tasks on host 1
    - Take host 2
    - Run ALL tasks on host 2
    - ... repeat 50 times
  
  50 hosts × (let's say 30 seconds of tasks per host) = 25 minutes.
  
  serial is for ROLLING DEPLOYMENTS where you can't take all hosts out simultaneously.
  For configuration management (installing packages, setting configs), you don't 
  need serial at all.

Fix:
  # Remove serial entirely for configuration plays
  - name: Configure webservers
    hosts: role_webserver
    # serial: removed — let forks handle parallelism
    tasks: [...]

  # Keep serial ONLY for service-affecting rolling deploys
  - name: Rolling restart of nginx
    hosts: role_webserver
    serial: "25%"   # 25% of fleet at a time
    tasks:
      - name: Restart nginx
        ansible.builtin.systemd:
          name: nginx
          state: restarted

Impact: Changes from sequential (50 × T) to parallel (ceil(50/forks) × T).
```

**Problem 5: Using `command` instead of native modules**

```
What's happening — example:
  # BAD: Not idempotent, no change detection, slow
  - name: Install nginx
    ansible.builtin.command: yum install -y nginx
  
  - name: Create directory
    ansible.builtin.command: mkdir -p /opt/novamart/config
  
  - name: Copy config
    ansible.builtin.command: cp /tmp/nginx.conf /etc/nginx/nginx.conf
  
  - name: Start nginx
    ansible.builtin.command: systemctl start nginx

Problems with each:
  1. command always reports "changed" — even if nginx is already installed
  2. command runs EVERY time — mkdir runs even if directory exists
  3. No checksum comparison — copies file even if identical
  4. No service state awareness — tries to start even if already running
  
  Because every task reports "changed," you can never trust the playbook output 
  to tell you if something ACTUALLY changed. This is the death of idempotency.

Fix — use native modules:
  # GOOD: Idempotent, correct change detection, faster (no shell overhead)
  - name: Install nginx
    ansible.builtin.yum:
      name: nginx
      state: present    # Only installs if not present. Reports "ok" if already there.
  
  - name: Create directory
    ansible.builtin.file:
      path: /opt/novamart/config
      state: directory
      owner: root
      mode: '0755'       # Also ensures correct permissions — command doesn't
  
  - name: Copy config
    ansible.builtin.template:  # Or ansible.builtin.copy
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      owner: root
      mode: '0644'
    notify: restart nginx  # Only triggers handler if file ACTUALLY changed
  
  - name: Ensure nginx is running
    ansible.builtin.systemd:
      name: nginx
      state: started
      enabled: true       # Also ensures it starts on boot

Why native modules are faster:
  - command: forks a shell process → loads shell → interprets command → runs
  - native module: Python executes directly → calls OS APIs → no shell overhead
  - Native modules also do FEWER operations: yum module checks RPM database 
    first and short-circuits if package is present. command always runs yum.

Impact: Eliminates unnecessary operations on 2nd+ runs. First run: same speed.
  Subsequent runs: 10x faster (most tasks report "ok" and skip work).
```

### Estimated Runtime After All Fixes

```
BEFORE:
  forks=5, serial=1, gather_facts everywhere, no pipelining, all commands
  
  Effective parallelism: 1 (serial:1 overrides forks)
  50 hosts × sequential × 5 SSH round-trips per task × fact gathering every play
  ≈ 45 minutes

AFTER:
  forks=25, no serial (except rolling deploys), gather_facts:false or cached, 
  pipelining=true, native modules

  Effective parallelism: 25
  Tasks per round: ceil(50/25) = 2 rounds per task
  SSH overhead: 1 round-trip per task (pipelining)
  Fact gathering: 0 seconds (cached or disabled)
  Skipped work: native modules skip unchanged resources

  Estimated: 2-4 minutes for the same 50 hosts

  Breakdown:
    Facts:    0s (disabled/cached)     was: ~600s
    Tasks:    ~120s (2 rounds × 25 tasks × ~2.5s each)  was: ~2100s (serial)
    SSH:      minimal (pipelining + ControlPersist)      was: ~5x current
    Total:    ~2-4 minutes             was: 45 minutes
    
  Speedup: ~10-20x
```

---

## Q4: command vs shell vs raw — Deep Dive

### The Three Modules

```
                    command              shell                raw
                    ─────────           ──────               ────
Executes via:       execv() directly    /bin/sh -c "..."     Sends string over SSH
                    (no shell)          (full shell)         (no Python needed)

Shell features:     ✗ No pipes          ✓ Pipes              ✓ Everything
                    ✗ No redirects      ✓ Redirects          (it IS raw shell)
                    ✗ No env vars       ✓ $ENV_VARS
                    ✗ No glob           ✓ Wildcards

Python required     ✓ Yes               ✓ Yes                ✗ No
on target:

Injection safety:   Safer (no shell     Vulnerable to        No protection
                    interpretation)     shell injection       whatsoever
```

### Correct Usage Scenarios for Each

#### `ansible.builtin.command`

**Right choice — running a specific binary with known arguments:**

```yaml
# Scenario: Check the version of a binary (read-only, exact command)
- name: Get current Node Exporter version
  ansible.builtin.command:
    cmd: /usr/local/bin/node_exporter --version
  register: ne_version
  changed_when: false  # Read-only — never changed
```

Why `command` here: You're running one binary with one flag. No pipes, no redirects, no shell expansion needed. `command` is safer because if a variable contained `; rm -rf /`, `command` would pass it as a literal string argument, not interpret the semicolon.

**Wrong choice — when you need shell features:**

```yaml
# WRONG: This FAILS because command doesn't support pipes
- name: Count running nginx workers
  ansible.builtin.command:
    cmd: ps aux | grep nginx | wc -l
  # Error: tries to run binary literally named "ps aux | grep nginx | wc -l"
```

#### `ansible.builtin.shell`

**Right choice — when you genuinely need shell features:**

```yaml
# Scenario: Find and archive logs older than 7 days
- name: Archive old NovaMart application logs
  ansible.builtin.shell:
    cmd: |
      find /var/log/novamart/ -name "*.log" -mtime +7 -print0 | \
        xargs -0 tar -czf /backup/old_logs_$(date +%Y%m%d).tar.gz && \
        find /var/log/novamart/ -name "*.log" -mtime +7 -delete
    creates: "/backup/old_logs_{{ ansible_date_time.date | regex_replace('-', '') }}.tar.gz"
  # 'creates' provides idempotency: if the archive exists, skip the task
```

Why `shell` here: Pipes (`|`), command substitution (`$(date ...)`), glob patterns (`*.log`), and logical chaining (`&&`). None of these work with `command`.

**Wrong choice — when a native module exists:**

```yaml
# WRONG: Using shell to install a package
- name: Install nginx
  ansible.builtin.shell:
    cmd: "apt-get install -y nginx 2>&1 | tee /tmp/apt.log"
  # Problems:
  #   1. Always reports "changed" even if nginx was already installed
  #   2. Doesn't handle apt lock files
  #   3. Vulnerable to injection if variable interpolation is used
  #   4. No rollback capability

# RIGHT: Use the native module
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true
    cache_valid_time: 3600
```

#### `ansible.builtin.raw`

**Right choice — bootstrapping a host that has no Python:**

```yaml
# Scenario: NovaMart just provisioned new EC2 instances with a minimal AMI
# that doesn't have Python installed. All other modules REQUIRE Python on the target.
- name: Bootstrap Python on fresh instances
  hosts: new_servers
  gather_facts: false  # Can't gather facts without Python!
  tasks:
    - name: Install Python (raw — no Python needed on target)
      ansible.builtin.raw: |
        if command -v python3 &>/dev/null; then
          echo "Python already installed"
        else
          yum install -y python3 || apt-get install -y python3
        fi
      register: python_install
      changed_when: "'already installed' not in python_install.stdout"

    - name: Now gather facts (Python is available)
      ansible.builtin.setup:
```

Why `raw` here: The target has no Python. `command` and `shell` both require Ansible's Python module framework to be present on the target. `raw` just shoves a string through the SSH pipe and returns whatever comes back. It's the only option for Python-less targets.

**Wrong choice — for any normal task:**

```yaml
# WRONG: Using raw for normal operations
- name: Create a user
  ansible.builtin.raw: useradd -m -s /bin/bash deploy_user
  # Problems:
  #   1. No idempotency — runs useradd even if user exists (will ERROR)
  #   2. No proper return code handling
  #   3. No integration with Ansible's change tracking
  #   4. No diff mode support

# RIGHT: Use the native module
- name: Create deploy user
  ansible.builtin.user:
    name: deploy_user
    shell: /bin/bash
    create_home: true
    state: present
```

### Why command/shell Without `changed_when`/`creates` Breaks Idempotency

**The core problem:**

Ansible's execution model has a contract: every task returns one of three states:
- **ok** — resource is already in desired state, no action taken
- **changed** — resource was modified to reach desired state
- **failed** — could not reach desired state

Native modules implement this contract internally. The `yum` module checks if a package is installed before installing it. If already installed → `ok`. If it installs it → `changed`.

`command` and `shell` **cannot know what your command does.** From Ansible's perspective, it ran a command and it exited 0. Was something changed? Ansible has no idea. So it **always reports `changed`**.

```
Run 1: ansible-playbook site.yml
  TASK [Create directory] → changed  (actually created it)
  TASK [Set timezone]     → changed  (actually changed it)
  
Run 2: ansible-playbook site.yml  
  TASK [Create directory] → changed  (directory already existed — LYING)
  TASK [Set timezone]     → changed  (timezone was already correct — LYING)
```

**Why this is catastrophic:**

1. **You can't trust the output.** If every run shows 47 changed tasks, you can't tell if something actually changed. You lose visibility into drift.

2. **Handlers fire unnecessarily.** If a `command` task that "creates a config file" always reports `changed`, and it `notify: restart nginx`, nginx restarts **every single run** — for no reason. In production, that's unnecessary downtime.

3. **`--check` mode is useless.** Check mode (dry run) relies on `changed` reporting to predict what would happen. If everything is always "changed," check mode tells you nothing.

4. **Audit and compliance break.** "Show me that the last Ansible run made zero changes to production" is impossible if every run reports changes.

### The Fixes

**Fix 1: `changed_when` — tell Ansible how to determine if a change occurred:**

```yaml
# BAD
- name: Set timezone
  ansible.builtin.command:
    cmd: timedatectl set-timezone America/New_York

# GOOD
- name: Check current timezone
  ansible.builtin.command:
    cmd: timedatectl show --property=Timezone --value
  register: current_tz
  changed_when: false  # This is a read-only check

- name: Set timezone
  ansible.builtin.command:
    cmd: timedatectl set-timezone America/New_York
  when: current_tz.stdout != "America/New_York"
  changed_when: current_tz.stdout != "America/New_York"
```

**Fix 2: `creates`/`removes` — tell Ansible to check for a file as a proxy for state:**

```yaml
# BAD: Runs every time, always reports changed
- name: Initialize database
  ansible.builtin.command:
    cmd: /opt/novamart/bin/init_db.sh

# GOOD: Only runs if the marker file doesn't exist
- name: Initialize database
  ansible.builtin.command:
    cmd: /opt/novamart/bin/init_db.sh
    creates: /opt/novamart/data/.db_initialized
  # If /opt/novamart/data/.db_initialized exists → skip (reports "ok")
  # If it doesn't exist → run command (reports "changed")
  # Note: the init_db.sh script must create this file on success
```

**Fix 3: Best option — use the native module instead:**

```yaml
# Instead of: command: timedatectl set-timezone America/New_York
- name: Set timezone
  community.general.timezone:
    name: America/New_York
  # Idempotent by design. Checks current timezone, only changes if different.
  # Correctly reports ok/changed. Works with --check mode. No fixes needed.
```

**The decision tree:**

```
Need to run something on a target?
│
├─→ Does a native module exist? (apt, yum, file, copy, template, systemd, user...)
│   └─→ YES: Use it. Stop here.
│
├─→ Does target have Python?
│   ├─→ NO: Use raw (bootstrap Python, then switch to modules)
│   └─→ YES: ↓
│
├─→ Do you need pipes, redirects, env vars, globs?
│   ├─→ NO: Use command (safer)
│   └─→ YES: Use shell (less safe)
│
└─→ In ALL cases where you use command/shell:
    ├─→ Add changed_when (with logic) or changed_when: false (if read-only)
    ├─→ Add creates/removes if applicable
    └─→ Add a comment explaining WHY a native module isn't suitable
```

# Phase 4, Lesson 4 — Retention Grading

---

## Q1: Dynamic Inventory — Grade: 4.5/5 ⭐

**What you nailed:**

- Correct plugin format, filename convention (`aws_ec2.yml`), `strict: true` for fail-loud
- Honest handling of the OS detection problem — acknowledged the weakness, then provided the clean production fix (enforce `OS` tag via Terraform). That's senior thinking: don't hack around infrastructure problems, fix the source
- Bastion ProxyCommand syntax is correct, including ControlMaster/ControlPersist for connection reuse
- Fact caching in ansible.cfg — bonus, wasn't asked, shows performance awareness
- Verification commands at the end — operational habit

**What was slightly off:**

The `regex_replace('-', '_')` on region for group naming is clever, but there's a simpler approach that the plugin handles natively:

```yaml
keyed_groups:
  - key: placement.region
    prefix: region
    separator: "_"
# Plugin automatically produces: region_us_east_1
# The dots/dashes in the value get handled by the separator setting
# No regex needed
```

Not wrong — yours works — but unnecessary complexity.

**What was missing (minor):**

```
1. Plugin-level caching to reduce AWS API calls:
   cache: true
   cache_plugin: jsonfile
   cache_connection: /tmp/aws_inventory_cache
   cache_timeout: 300    # 5 minutes
   
   Why it matters: Without this, every ansible-playbook invocation
   makes DescribeInstances API calls to 3 regions. In CI running
   20 playbooks/day, that's 60 API calls/day minimum. With cache,
   it's 1 call per 5 minutes regardless of how many playbooks run.
   Also speeds up playbook startup from ~3-5s to instant.

2. IAM authentication for the control node itself:
   How does the control node authenticate to AWS to call DescribeInstances?
   - CI runner: Instance profile (IRSA if in K8s, EC2 instance profile if on EC2)
   - Local laptop: aws sso login → profile in ~/.aws/config
   - NEVER hardcoded access keys in ansible.cfg or env vars in CI
   
   You'd add to aws_ec2.yml:
   aws_profile: novamart-prod
   # Or rely on standard AWS credential chain (env → profile → instance profile)
```

Neither of these is a dealbreaker, but at NovaMart scale (3 regions, CI running constantly), the cache matters.

---

## Q2: Rolling Node Exporter Upgrade — Grade: 4.5/5 ⭐

**What you nailed:**

- `serial: 5` with `max_fail_percentage: 20` — correct blast radius control
- `meta: end_host` to skip already-upgraded hosts — efficient, avoids unnecessary work
- Binary backup before swap → rollback in rescue block — textbook safe upgrade
- `meta: flush_handlers` to force systemd daemon-reload before restart — catches the timing trap
- The explicit `fail` in rescue so the batch accounting registers the failure — most people miss this. Without it, rescue "recovers" the error and the host looks successful, so `max_fail_percentage` never triggers
- The Prometheus scrape interval analysis explaining why 2-3s downtime is acceptable — that's production reasoning, not textbook reasoning

**What was slightly off:**

```yaml
# Your gather_subset:
gather_subset:
  - '!all'
  - '!min'
  - distribution    # ← Not a valid subset name
```

Valid subset names are: `all`, `min`, `hardware`, `network`, `virtual`, `ohai`, `facter`. The `distribution` info (ansible_os_family, ansible_distribution) is part of the `min` subset, which is gathered by default unless you exclude it with `!min`.

```yaml
# Correct approach if you only want OS info:
- ansible.builtin.setup:
    gather_subset:
      - '!all'        # Disable everything
      # min is still included by default (has OS info)
      # This gives you ansible_os_family, ansible_distribution, etc.
      # without hardware, network, virtual detection
```

Or since you set `gather_facts: false` and don't actually USE any facts in the playbook (no conditionals on `ansible_os_family`), you could skip the `setup` task entirely. Your version check uses `command`, not facts.

**What was missing (minor):**

```yaml
# 1. Pre-flight notification
pre_tasks:
  # ...version check...
  
  - name: Notify team that upgrade is starting
    ansible.builtin.uri:
      url: "{{ slack_webhook }}"
      method: POST
      body_format: json
      body:
        text: "🔄 Node Exporter upgrade {{ node_exporter_old_version }} → {{ node_exporter_version }} starting across {{ ansible_play_hosts | length }} hosts"
    delegate_to: localhost
    run_once: true    # Only send ONE Slack message, not one per host

# 2. Post-play summary notification
post_tasks:
  - name: Notify team that batch completed
    ansible.builtin.uri:
      url: "{{ slack_webhook }}"
      method: POST
      body_format: json
      body:
        text: "✅ Node Exporter upgrade batch complete: {{ ansible_play_hosts | length }} hosts upgraded"
    delegate_to: localhost
    run_once: true
```

At NovaMart ($50K/min outage cost), any fleet-wide change should have visibility in the team channel. Not a correctness issue — an operational maturity issue.

---

## Q3: 45-Minute Playbook Diagnosis — Grade: 4.5/5 ⭐

**What you nailed:**

- All 5 problems correctly identified with accurate explanations
- `serial: 1` negating forks — this is the single biggest performance killer and you put it front and center with the correct explanation
- Pipelining prerequisite (`requiretty`) noted with the fix command
- Native modules vs command — correctly explained WHY native modules are faster (check-then-act vs always-act)
- Runtime estimate is realistic: 45 min → 2-4 min with a ~10-20x speedup

**What was missing:**

```
1. strategy: free
   Default strategy is 'linear': all hosts must complete task N 
   before ANY host starts task N+1.
   
   With 50 hosts and forks: 25:
     Round 1: hosts 1-25 start task 1
     Round 2: hosts 26-50 start task 1
     → ALL 50 must finish task 1 before task 2 begins
     → One slow host holds back the entire batch
   
   strategy: free lets faster hosts race ahead:
     Host 1 finishes task 1 → immediately starts task 2
     Host 47 is slow → doesn't block anyone
   
   Tradeoff: Output is harder to read (interleaved), and 
   you can't rely on task ordering across hosts.
   
   Good for: Configuration convergence (idempotent tasks)
   Bad for: Orchestration (where host A must complete before host B starts)

   - name: Configure all servers
     hosts: all
     strategy: free    # Each host proceeds independently
     tasks: [...]

2. async + poll for long-running tasks
   If one task takes 60 seconds (e.g., package installation):
   
   # Without async: 50 hosts × 60s ÷ 25 forks = 120s blocked
   - name: Install heavy package
     ansible.builtin.yum:
       name: java-17-openjdk-devel
       state: present
     async: 300       # Allow up to 5 minutes
     poll: 0          # Don't wait — fire and forget
     register: yum_job
   
   - name: Wait for package installation to complete
     ansible.builtin.async_status:
       jid: "{{ yum_job.ansible_job_id }}"
     register: job_result
     until: job_result.finished
     retries: 30
     delay: 10
   
   # All 50 hosts start installing simultaneously,
   # then we poll for completion. Saves time on long tasks.
```

Neither is critical — your 5 fixes already get the 10-20x improvement. These are the difference between "fast" and "maximally fast."

---

## Q4: command vs shell vs raw — Grade: 5/5 🎯

This is a perfect answer.

- Comparison table is clear and accurate
- Every module gets a correct "right" scenario and a "wrong" scenario with explanation
- The `raw` bootstrapping use case is exactly right, including `gather_facts: false` and `changed_when` for the raw task
- The idempotency explanation covers all four consequences: trust, handlers, check mode, audit
- All three fixes are correct with production examples
- The decision tree at the end is something I'd put in a team wiki

**One thing I'll highlight for your own reference** — you wrote this in the `shell` example:

```yaml
creates: "/backup/old_logs_{{ ansible_date_time.date | regex_replace('-', '') }}.tar.gz"
```

This is a subtle trap. `creates` is evaluated on the **target host**, but `ansible_date_time` is a **fact from the control node** (or the target, depending on when facts were gathered). If the control node is in UTC and the target is in US-East, the date might differ, and the `creates` check would fail to match. In practice this rarely bites, but worth knowing.

---

## Overall Score: 4.6/5

```
Q1: 4.5/5  — Comprehensive, honest about tricky parts, minor gaps (caching, IAM auth)
Q2: 4.5/5  — Production-grade playbook, Prometheus analysis is senior-level, minor gather_subset error
Q3: 4.5/5  — All 5 problems nailed, realistic estimate, missed strategy:free and async
Q4: 5.0/5  — Flawless. Decision tree is team-wiki quality.
```

**Pattern I'm seeing:** Your answers are consistently production-oriented. You don't just answer "what" — you explain "why it matters in production" and "what breaks if you don't." The Prometheus scrape interval analysis in Q2, the audit/compliance point in Q4 — these aren't textbook answers. These are answers from someone who's been on-call.

**The one gap to close:** You occasionally miss the "nice to have" operational touches — notifications, caching, strategy tuning. These are the difference between "correct" and "polished." At NovaMart, the Slack notification before a fleet-wide change isn't optional — it's the difference between your teammates knowing what's happening and someone opening a SEV3 because they see node_exporter metrics blip during your upgrade.

---

Moving to **Phase 4, Lesson 5: Ansible Advanced — Vault, Testing, Performance, Terraform Integration, Tower/AWX.**

---

# Phase 4, Lesson 5: Ansible Advanced — Vault, Performance, Testing, Integration Patterns

---

## 1. Ansible Vault — Secrets Management

### The Problem

```
Your playbook needs:
  - Database passwords
  - API keys
  - TLS private keys
  - SSH keys
  - Cloud credentials

You CANNOT put these in plaintext in Git.
You CANNOT put these in plaintext in group_vars.
You need encryption at rest in the repository.

Ansible Vault encrypts files or variables with AES-256.
The encrypted content lives IN your Git repo.
The decryption password is provided at runtime.
```

### Vault Operations

```bash
# ── ENCRYPTING FILES ──

# Encrypt an entire file
ansible-vault encrypt group_vars/production/vault.yml
# Prompts for vault password, encrypts file IN PLACE
# File now starts with: $ANSIBLE_VAULT;1.1;AES256

# Create a new encrypted file
ansible-vault create group_vars/production/vault.yml
# Opens $EDITOR, you type content, saves encrypted on close

# View encrypted file without decrypting on disk
ansible-vault view group_vars/production/vault.yml

# Edit encrypted file
ansible-vault edit group_vars/production/vault.yml
# Decrypts to temp file → opens editor → re-encrypts on save

# Decrypt file (rarely needed — don't leave decrypted files on disk)
ansible-vault decrypt group_vars/production/vault.yml

# Re-key (change the vault password — key rotation)
ansible-vault rekey group_vars/production/vault.yml
# Prompts for old password, then new password

# ── ENCRYPTING INDIVIDUAL STRINGS ──
# When you want SOME vars encrypted but not the entire file

ansible-vault encrypt_string 'SuperS3cretP@ss!' --name 'db_password'
# Output:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   62313365396662343061393464336163383764316462...

# Paste this directly into any vars file — the rest of the file stays plaintext
```

### Vault in Practice — The NovaMart Pattern

```yaml
# group_vars/production/vars.yml (PLAINTEXT — committed to Git)
---
env: production
app_port: 8080
db_host: prod-db.novamart.internal
db_name: novamart_prod
db_user: app_user
monitoring_enabled: true

# group_vars/production/vault.yml (ENCRYPTED — committed to Git)
---
vault_db_password: "SuperS3cretP@ss!"
vault_api_key: "sk-live-abc123def456"
vault_tls_private_key: |
  -----BEGIN PRIVATE KEY-----
  MIIEvgIBADANBgkqhkiG9w0BAQE...
  -----END PRIVATE KEY-----
vault_slack_webhook: "https://hooks.slack.com/services/T00/B00/xxxx"
```

```yaml
# NAMING CONVENTION — CRITICAL:
# Prefix all vault variables with 'vault_'
# Then reference them from plaintext files:

# group_vars/production/vars.yml
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"

# WHY this pattern:
# 1. grep -r "vault_" → instantly shows all secrets
# 2. You can see WHICH variables are secrets without decrypting
# 3. Debugging: if db_password is wrong, you know to check vault.yml
# 4. ansible-vault view shows the vault_ vars, 
#    but your playbooks use the non-prefixed names
```

### Providing the Vault Password

```bash
# Method 1: Interactive prompt (local development)
ansible-playbook site.yml --ask-vault-pass
# Prompts for password every run. Annoying but secure.

# Method 2: Password file (CI/CD)
echo 'MyVaultPassword123' > ~/.vault_pass
chmod 600 ~/.vault_pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# In ansible.cfg:
[defaults]
vault_password_file = ~/.vault_pass
# Now you never need to pass --vault-password-file

# Method 3: Script that retrieves password from external secret store (BEST)
#!/bin/bash
# vault_pass.sh — retrieves vault password from AWS Secrets Manager
aws secretsmanager get-secret-value \
  --secret-id novamart/ansible/vault-password \
  --query SecretString \
  --output text

# Make executable and reference:
chmod +x vault_pass.sh
# ansible.cfg:
vault_password_file = ./vault_pass.sh

# Method 4: Multiple vault IDs (different passwords for different environments)
ansible-vault encrypt --vault-id prod@prompt group_vars/production/vault.yml
ansible-vault encrypt --vault-id staging@prompt group_vars/staging/vault.yml

ansible-playbook site.yml \
  --vault-id prod@~/.vault_pass_prod \
  --vault-id staging@~/.vault_pass_staging
# Each vault file is encrypted with its own password
# Prod team doesn't need staging password and vice versa
```

### Vault vs External Secrets — When to Use What

```
                    Ansible Vault              HashiCorp Vault / AWS SM
                    ─────────────              ──────────────────────────
Encryption:         AES-256 (file-level)       AES-256 (service-level)
Where secrets       In Git repo (encrypted)    In external service
  live:
Access control:     Whoever has the vault      IAM policies, AppRoles,
                    password can decrypt ALL    fine-grained per-secret
Rotation:           Manual (rekey command)      Automatic (API-driven)
Audit:              Git log (who committed)     Full audit log (who 
                                                accessed what when)
Dynamic secrets:    ❌ No                      ✅ Yes (temp DB creds)
Best for:           Small teams, simple         Large orgs, compliance,
                    setups, bootstrap           dynamic credentials

NovaMart pattern:
  Ansible Vault: Bootstrap secrets (initial Vault server setup, 
                 cloud credentials for first Terraform run)
  HashiCorp Vault: Everything else (app secrets, DB passwords, 
                   TLS certs, API keys) — accessed at runtime
  
  THE GOAL: Minimize what's in Ansible Vault. Use it only for the 
  "chicken-and-egg" secrets needed to bootstrap the real secret store.
```

### Vault Failure Modes

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **Wrong vault password** | `ERROR! Decryption failed` | Password changed, wrong env password file | Verify password file, check vault-id |
| **Mixed encryption** | Some tasks decrypt, others fail | File encrypted with different vault-id, or partially encrypted | Use `ansible-vault rekey` to unify, or use `--vault-id` per environment |
| **Vault password in Git** | `.vault_pass` committed | Missing `.gitignore` entry | `echo '.vault_pass*' >> .gitignore`, rotate password immediately |
| **Secrets in ansible output** | Password visible in `-v` output | Missing `no_log: true` on tasks using secrets | Add `no_log: true` to every task that references vault variables |
| **Can't edit vault file** | `$EDITOR` not set or wrong | Environment variable issue | `export EDITOR=vim`, or `EDITOR=vim ansible-vault edit file.yml` |
| **Rekey across many files** | Need to change vault password | Key rotation, team member departure | Script: `find . -name 'vault.yml' -exec ansible-vault rekey {} \;` |

---

## 2. Performance Optimization — Making Ansible Fast

### The Performance Hierarchy

```
BIGGEST IMPACT → SMALLEST IMPACT:

1. Architecture: Don't use Ansible where it's not needed
   (Use K8s ConfigMaps, Helm values, cloud-init for what they do better)

2. Parallelism: forks, serial, strategy
   
3. Connection: pipelining, ControlPersist, connection reuse

4. Facts: gather_facts:false, caching, subset

5. Task design: native modules, avoid loops (use list parameters)

6. Execution: async, free strategy, mitogen

Let's cover #6 since #1-5 were in Lesson 4.
```

### Async — Fire and Forget for Long Tasks

```yaml
# Problem: Installing Java takes 60 seconds.
# With forks:25 and 50 hosts, that's 2 rounds × 60s = 120s BLOCKED.
# Other tasks can't start until Java installs on ALL hosts in the batch.

# Solution: async
- name: Install Java (async)
  ansible.builtin.yum:
    name: java-17-openjdk-devel
    state: present
  async: 600      # Maximum time allowed (10 minutes)
  poll: 0         # Don't wait — return immediately
  register: java_install

# ... other tasks can run here while Java installs ...

- name: Wait for Java installation
  ansible.builtin.async_status:
    jid: "{{ java_install.ansible_job_id }}"
  register: java_result
  until: java_result.finished
  retries: 60
  delay: 10       # Check every 10 seconds

# HOW IT WORKS:
# 1. Ansible starts the yum install on all 50 hosts
# 2. Returns immediately (poll: 0)
# 3. Each host runs yum in background, writes status to a JSON file
# 4. async_status reads that JSON file to check completion
# 5. You can run OTHER tasks in between

# WHEN TO USE:
# - Long-running tasks (package installs, large file downloads)
# - Tasks that are independent of each other
# - When you want maximum parallelism

# WHEN NOT TO USE:
# - Tasks that depend on previous task's output
# - Tasks that modify shared state
# - Short tasks (overhead of async > time saved)
```

### Strategy: Free vs Linear

```yaml
# Default: linear strategy
# ALL hosts complete task 1 → ALL hosts start task 2
# One slow host blocks the entire batch

# Free strategy: each host proceeds independently
- name: Configure servers independently
  hosts: all
  strategy: free    # Each host races through tasks at its own pace
  tasks:
    - name: Task 1
      ansible.builtin.yum:
        name: nginx
        state: present

    - name: Task 2
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf

    - name: Task 3
      ansible.builtin.service:
        name: nginx
        state: started

# With free strategy:
#   Host 1 (fast SSD): Task 1 → Task 2 → Task 3 (done in 10s)
#   Host 2 (slow disk): Task 1 ................. → Task 2 → Task 3 (done in 45s)
#   Host 1 doesn't wait for Host 2

# TRADEOFFS:
# ✓ Faster overall completion (limited by slowest host, not slowest task)
# ✗ Output is interleaved and hard to read
# ✗ Can't use serial (they conflict)
# ✗ Task ordering across hosts is not guaranteed
#   → DON'T use for orchestration where Host A must finish before Host B starts
```

### Mitogen — The Ansible Accelerator

```
Mitogen replaces Ansible's default SSH + temporary file execution model
with a persistent Python interpreter on the remote host that receives
code over an efficient binary channel.

DEFAULT ANSIBLE:
  For each task:
    SSH → create temp dir → SCP module → chmod → execute → read output → delete
  
MITOGEN:
  First task: SSH → install tiny bootstrap → start persistent interpreter
  Subsequent tasks: Send module code over existing channel → execute → return
  
  No temp files. No SCP. No chmod. No cleanup.
  The remote interpreter stays alive across tasks.

SPEED IMPROVEMENT: 2-7x faster than pipelining alone

INSTALL:
  pip install mitogen
  
  # ansible.cfg
  [defaults]
  strategy_plugins = /path/to/mitogen/ansible_mitogen/plugins/strategy
  strategy = mitogen_linear    # or mitogen_free

CAVEATS:
  - Not officially supported by Red Hat (Ansible Automation Platform)
  - Can lag behind new Ansible versions
  - Some edge cases with become methods other than sudo
  - Test thoroughly before deploying to production CI
  - At NovaMart: We use it on CI runners for speed, 
    but not in Ansible Tower (compatibility concerns)
```

### Performance Comparison

```
50 HOSTS, 20 TASKS, AVERAGE TASK: 3 SECONDS

Configuration                              Estimated Time
──────────────────────────────────────     ──────────────
forks:5, serial:1, no pipelining           ~50 min
forks:5, no serial, no pipelining          ~12 min
forks:25, no serial, no pipelining         ~5 min
forks:25, no serial, pipelining            ~2 min
forks:25, no serial, pipelining, cache     ~1.5 min
forks:25, strategy:free, mitogen           ~45 sec

The lesson: architecture and parallelism are 90% of the win.
Mitogen is the cherry on top.
```

---

## 3. Molecule — Testing Ansible Roles

### Why Test Ansible Code

```
"It worked on my machine" is bad enough for application code.
For infrastructure code it's CATASTROPHIC.

Without tests:
  1. You write a role
  2. You run it against staging
  3. It works (on Ubuntu 22.04 with Python 3.10)
  4. You run it against production
  5. Production is Amazon Linux 2 with Python 3.7
  6. Template fails because of Jinja2 version difference
  7. 20 servers are now in a broken state
  8. It's 2 AM

With Molecule:
  1. You write a role
  2. Molecule spins up a Docker container (or EC2 instance)
  3. Runs your role inside it
  4. Runs your verification tests
  5. Tears down the container
  6. All in CI, before merge
```

### Molecule Architecture

```
┌────────────────────────────────────────────────┐
│                  Molecule                        │
│                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ Create   │──▶│ Converge │──▶│ Verify   │    │
│  │          │   │          │   │          │    │
│  │ Spin up  │   │ Run the  │   │ Run      │    │
│  │ test     │   │ role     │   │ tests    │    │
│  │ instance │   │ against  │   │ against  │    │
│  │ (Docker/ │   │ instance │   │ instance │    │
│  │  EC2/    │   │          │   │ (verify  │    │
│  │  Vagrant)│   │          │   │  state)  │    │
│  └──────────┘   └──────────┘   └──────────┘    │
│       │              │              │            │
│       ▼              ▼              ▼            │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ Idempot. │   │ Side     │   │ Destroy  │    │
│  │          │   │ Effect   │   │          │    │
│  │ Run role │   │ Check    │   │ Tear     │    │
│  │ AGAIN    │   │          │   │ down     │    │
│  │ Verify 0 │   │ Lint     │   │ test     │    │
│  │ changes  │   │ Syntax   │   │ instance │    │
│  └──────────┘   └──────────┘   └──────────┘    │
└────────────────────────────────────────────────┘

FULL SEQUENCE (molecule test):
  dependency → lint → cleanup → destroy → syntax → create → 
  prepare → converge → idempotence → side_effect → verify → 
  cleanup → destroy
```

### Molecule Setup for a Role

```bash
# Initialize molecule in an existing role
cd roles/node_exporter
molecule init scenario --driver-name docker

# Creates:
# roles/node_exporter/molecule/default/
#   ├── molecule.yml       # Configuration
#   ├── converge.yml       # Playbook to run role
#   ├── verify.yml         # Verification tests
#   └── prepare.yml        # Pre-test setup (optional)
```

```yaml
# molecule/default/molecule.yml
---
dependency:
  name: galaxy
  options:
    requirements-file: requirements.yml

driver:
  name: docker

platforms:
  # Test on multiple OS versions
  - name: node-exporter-ubuntu2204
    image: geerlingguy/docker-ubuntu2204-ansible
    pre_build_image: true
    privileged: true    # Needed for systemd
    command: /lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    tmpfs:
      - /run
      - /tmp

  - name: node-exporter-amazonlinux2
    image: geerlingguy/docker-amazonlinux2-ansible
    pre_build_image: true
    privileged: true
    command: /lib/systemd/systemd
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host

provisioner:
  name: ansible
  config_options:
    defaults:
      callbacks_enabled: profile_tasks    # Show task timing
  inventory:
    host_vars:
      node-exporter-ubuntu2204:
        ansible_user: root
      node-exporter-amazonlinux2:
        ansible_user: root

verifier:
  name: ansible    # Use ansible tasks for verification (can also use testinfra)

lint: |
  set -e
  yamllint .
  ansible-lint
```

```yaml
# molecule/default/converge.yml
---
- name: Converge
  hosts: all
  become: true
  vars:
    node_exporter_version: "1.7.0"
    node_exporter_port: 9100
  roles:
    - role: node_exporter
```

```yaml
# molecule/default/verify.yml
---
- name: Verify node_exporter installation
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: Check node_exporter binary exists
      ansible.builtin.stat:
        path: /usr/local/bin/node_exporter
      register: binary
      failed_when: not binary.stat.exists

    - name: Check node_exporter binary is executable
      ansible.builtin.stat:
        path: /usr/local/bin/node_exporter
      register: binary_perms
      failed_when: not binary_perms.stat.executable

    - name: Check node_exporter user exists
      ansible.builtin.getent:
        database: passwd
        key: node_exporter
      register: user_check

    - name: Check node_exporter service is running
      ansible.builtin.service_facts:

    - name: Assert node_exporter service is active
      ansible.builtin.assert:
        that:
          - "'node_exporter.service' in ansible_facts.services"
          - "ansible_facts.services['node_exporter.service'].state == 'running'"
        fail_msg: "node_exporter service is not running"

    - name: Check metrics endpoint
      ansible.builtin.uri:
        url: "http://localhost:9100/metrics"
        return_content: true
      register: metrics

    - name: Assert key metrics are present
      ansible.builtin.assert:
        that:
          - "'node_cpu_seconds_total' in metrics.content"
          - "'node_memory_MemTotal_bytes' in metrics.content"
          - "'node_filesystem_avail_bytes' in metrics.content"
        fail_msg: "Expected metrics not found in node_exporter output"

    - name: Check systemd unit file content
      ansible.builtin.slurp:
        src: /etc/systemd/system/node_exporter.service
      register: unit_file

    - name: Assert unit file has security hardening
      ansible.builtin.assert:
        that:
          - "'NoNewPrivileges=yes' in (unit_file.content | b64decode)"
          - "'ProtectSystem=strict' in (unit_file.content | b64decode)"
        fail_msg: "Unit file missing security hardening directives"
```

### Running Molecule

```bash
# Full test sequence (create → converge → idempotence → verify → destroy)
molecule test

# Development workflow (faster iteration):
molecule create      # Spin up containers
molecule converge    # Run role
molecule verify      # Run tests
molecule converge    # Run again (check idempotence manually)
molecule login -h node-exporter-ubuntu2204    # SSH in to debug
molecule destroy     # Tear down

# Run on specific platform
molecule test --platform-name node-exporter-ubuntu2204

# The IDEMPOTENCE check is CRITICAL:
# Molecule runs the role TWICE.
# Second run must show 0 changed tasks.
# If anything shows "changed" on second run → idempotency broken → test FAILS.
```

### Molecule in CI (Jenkins)

```groovy
// Jenkinsfile for Ansible role testing
pipeline {
    agent { label 'docker' }

    stages {
        stage('Lint') {
            steps {
                sh '''
                    pip install yamllint ansible-lint
                    yamllint roles/node_exporter/
                    ansible-lint roles/node_exporter/
                '''
            }
        }

        stage('Molecule Test') {
            steps {
                sh '''
                    pip install molecule molecule-docker
                    cd roles/node_exporter
                    molecule test --all    # Test all scenarios
                '''
            }
        }
    }

    post {
        always {
            sh 'cd roles/node_exporter && molecule destroy || true'
        }
        failure {
            slackSend channel: '#platform-ci',
                      message: "❌ Ansible role test failed: ${env.BUILD_URL}"
        }
    }
}
```

### Molecule Limitations and Advanced Patterns

```
LIMITATION 1: Docker containers ≠ real servers
  - systemd in Docker is hacky (privileged mode, cgroup mounts)
  - No real kernel (shared with host)
  - Network stack differences
  - No real hardware (disk, NUMA, etc.)
  
  FIX: For critical roles, also test in EC2:
  driver:
    name: ec2    # molecule-ec2 plugin
  platforms:
    - name: test-ubuntu
      image: ami-0c55b159cbfafe1f0
      instance_type: t3.micro
      region: us-east-1
      vpc_subnet_id: subnet-xxx
      security_groups: [sg-xxx]
  # Slower (2-3 min to provision) but tests on real infra

LIMITATION 2: Can't test reboot scenarios
  Docker containers can't reboot. If your role configures 
  something that requires a reboot (kernel params, GRUB), 
  you need EC2 or Vagrant.

LIMITATION 3: Secrets
  Your role needs vault-encrypted variables, but CI needs 
  the vault password.
  FIX: Use molecule's provisioner vars to provide test values:
  provisioner:
    inventory:
      group_vars:
        all:
          vault_db_password: "test_password_not_real"
```

---

## 4. Ansible + Terraform Integration Patterns

### Pattern 1: Terraform Provisions, Ansible Configures

```
THE MOST COMMON PATTERN AT NOVAMART:

┌──────────┐         ┌──────────────┐         ┌──────────────┐
│Terraform │────────▶│  EC2 + Tags  │────────▶│   Ansible    │
│          │ creates │  + SG + Key  │ discovers│   dynamic    │
│ main.tf  │         │              │ via tags │   inventory  │
└──────────┘         └──────────────┘         └──────┬───────┘
                                                      │
                                              ┌───────▼────────┐
                                              │   Playbooks    │
                                              │   configure    │
                                              │   the server   │
                                              └────────────────┘

STEP 1: Terraform creates infrastructure
STEP 2: Terraform tags resources with Role, Environment, ManagedBy:ansible
STEP 3: Ansible dynamic inventory discovers hosts by tags
STEP 4: Ansible configures the hosts
```

```hcl
# Terraform — provision and tag
resource "aws_instance" "bastion" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = module.vpc.public_subnets[0]
  key_name      = aws_key_pair.ansible.key_name

  vpc_security_group_ids = [aws_security_group.bastion.id]

  tags = {
    Name        = "bastion-${var.environment}"
    Environment = var.environment
    Role        = "bastion"
    ManagedBy   = "ansible"
    OS          = "amazon-linux-2"
  }

  # Optional: run Ansible automatically after creation
  provisioner "local-exec" {
    command = <<-EOT
      # Wait for SSH to be available
      aws ec2 wait instance-status-ok --instance-ids ${self.id}
      
      # Run Ansible against just this host
      ansible-playbook \
        -i inventory/aws_ec2.yml \
        --limit ${self.private_ip} \
        playbooks/bastion.yml
    EOT
  }
}
```

### Pattern 2: Terraform Outputs → Ansible Variables

```hcl
# Terraform outputs
output "rds_endpoint" {
  value = aws_db_instance.main.endpoint
}

output "redis_endpoint" {
  value = aws_elasticache_cluster.main.cache_nodes[0].address
}

output "vpc_cidr" {
  value = module.vpc.vpc_cidr_block
}
```

```bash
# Script to bridge Terraform → Ansible
#!/bin/bash
# scripts/sync_terraform_vars.sh

# Pull Terraform outputs and convert to Ansible group_vars
cd terraform/production

terraform output -json | python3 -c "
import json, sys, yaml
outputs = json.load(sys.stdin)
ansible_vars = {k: v['value'] for k, v in outputs.items()}
print(yaml.dump(ansible_vars, default_flow_style=False))
" > ../../ansible/group_vars/production/terraform_outputs.yml

echo "Synced Terraform outputs to Ansible group_vars"
```

```yaml
# Result: group_vars/production/terraform_outputs.yml
# AUTO-GENERATED by sync_terraform_vars.sh — DO NOT EDIT
rds_endpoint: "prod-db.abc123.us-east-1.rds.amazonaws.com:5432"
redis_endpoint: "prod-redis.abc123.ng.0001.use1.cache.amazonaws.com"
vpc_cidr: "10.0.0.0/16"
```

### Pattern 3: Ansible as Terraform Provisioner (AVOID)

```hcl
# This pattern exists but is PROBLEMATIC:
resource "aws_instance" "web" {
  # ...
  
  provisioner "remote-exec" {
    inline = ["echo 'SSH is up'"]
  }
  
  provisioner "local-exec" {
    command = "ansible-playbook -i '${self.private_ip},' playbook.yml"
  }
}

# WHY TO AVOID:
# 1. Provisioners are NOT in Terraform state
#    → Terraform can't detect drift in the Ansible-managed config
# 2. If provisioner fails, resource is "tainted"
#    → Terraform wants to DESTROY AND RECREATE the instance
#    → You lose data, cause outage
# 3. Provisioners run on CREATE only (by default)
#    → Configuration changes require taint + apply (destroy + recreate)
# 4. Hard to test — can't run Ansible independently of Terraform

# THE RIGHT APPROACH:
# Keep Terraform and Ansible as SEPARATE workflows
# Terraform: provision → tag
# Ansible: discover by tag → configure
# Trigger both from CI pipeline, in order
```

### Pattern 4: The NovaMart CI/CD Integration

```
┌──────────────────────────────────────────────────────────────┐
│                    Jenkins Pipeline                            │
│                                                                │
│  Stage 1: Terraform Plan/Apply                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ terraform init → plan → apply                             │  │
│  │ Creates: EC2, SGs, IAM, etc.                              │  │
│  │ Tags: Role, Environment, ManagedBy:ansible                │  │
│  │ Outputs: endpoints, IPs, ARNs → saved as artifacts        │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                    │
│  Stage 2: Sync Terraform outputs to Ansible vars               │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │ terraform output -json → group_vars/env/terraform.yml     │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                    │
│  Stage 3: Ansible Configuration                                │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │ ansible-playbook -i inventory/aws_ec2.yml site.yml        │  │
│  │ Dynamic inventory discovers new/existing hosts            │  │
│  │ Configures: packages, users, services, monitoring         │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                    │
│  Stage 4: Verification                                         │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │ ansible-playbook verify.yml → health checks on all hosts  │  │
│  │ Smoke tests, port checks, endpoint validation             │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Ansible Tower / AWX — Enterprise Ansible

### What Tower/AWX Adds

```
Ansible CLI:
  ✓ Runs playbooks from command line
  ✗ No web UI
  ✗ No RBAC (whoever has SSH key can do anything)
  ✗ No job scheduling
  ✗ No audit trail
  ✗ No credential management (vault password in flat file)
  ✗ No approval workflows

Ansible Tower (commercial) / AWX (open source upstream):
  ✓ Web UI for running playbooks
  ✓ RBAC: "QA team can run deploys to staging but NOT production"
  ✓ Job scheduling (run hardening playbook every night)
  ✓ Full audit trail (who ran what, when, against which hosts, output)
  ✓ Credential store (SSH keys, vault passwords, cloud creds — encrypted, 
    never visible to users, injected at runtime)
  ✓ Approval workflows (prod deploy requires manager approval)
  ✓ Notifications (Slack, email, webhook on job completion)
  ✓ API (trigger playbooks from Jenkins, scripts, other tools)
  ✓ Inventory sync (auto-refresh from AWS, GCP, VMware)
  ✓ Survey forms (non-technical users can fill in variables)
```

### Tower/AWX Architecture

```
┌─────────────────────────────────────────────────────┐
│                 AWX / Tower                           │
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐  │
│  │ Web UI   │  │ REST API │  │  Task Engine        │  │
│  │ (React)  │  │          │  │  (runs playbooks    │  │
│  │          │  │          │  │   in containers)    │  │
│  └────┬─────┘  └────┬─────┘  └────────┬───────────┘  │
│       │              │                 │               │
│  ┌────▼──────────────▼─────────────────▼───────────┐  │
│  │              PostgreSQL                          │  │
│  │  (Jobs, inventories, credentials, audit logs)    │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │              Redis                               │  │
│  │  (Task queue, caching)                           │  │
│  └─────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘

Key concepts:
  Organization → Team → User (RBAC hierarchy)
  Project: Git repo containing playbooks
  Inventory: Static or dynamic (synced from cloud)
  Credential: SSH key, vault pass, cloud cred (encrypted, never exposed)
  Template: Playbook + Inventory + Credential + Variables = runnable job
  Workflow: Chain of templates (Template A → if success → Template B → ...)
```

### NovaMart AWX Usage

```yaml
# Example: AWX Workflow for server provisioning

Workflow: "Provision New Web Server"
  ├── Step 1: Terraform Apply (Job Template)
  │   Project: novamart-terraform
  │   Playbook: terraform_wrapper.yml  (playbook that runs terraform)
  │   Credential: AWS Production
  │   Survey: { instance_type: "m5.xlarge", count: 2 }
  │
  ├── Step 2: Wait for instances (Job Template)
  │   Playbook: wait_for_instances.yml
  │   On failure → Step 5 (cleanup)
  │
  ├── Step 3: Base configuration (Job Template)
  │   Playbook: site.yml
  │   Inventory: AWS EC2 Dynamic (auto-synced)
  │   Credential: SSH Key (ec2-user)
  │   On failure → Step 5 (cleanup)
  │
  ├── Step 4: Verification (Job Template)
  │   Playbook: verify.yml
  │   On failure → Step 5 (cleanup)
  │
  └── Step 5: Notify (Always)
      Playbook: notify.yml
      Notification: Slack #platform-ops

# RBAC:
#   Platform team: can create/edit templates and run against production
#   Dev team: can run pre-approved templates against staging only
#   QA team: can view job results, can't run anything
#   Manager: can approve production workflow runs
```

### Tower/AWX vs Jenkins for Ansible

```
                        Tower/AWX                Jenkins + Ansible
                        ─────────                ─────────────────
Purpose:                Run Ansible              Run EVERYTHING
Ansible-specific        ✓ Native support         ✗ Just shell steps
  features:             (credentials, RBAC,      (you build everything)
                         inventory sync)         
RBAC:                   Fine-grained per          Folder/job-level only
                        playbook/inventory
Credential mgmt:        Built-in vault            Jenkins credentials
                                                  (less Ansible-aware)
Inventory:              Auto-synced dynamic       You manage it
Audit:                  Per-task output            Build logs only
                        with RBAC
Cost:                   Tower: $$$                Jenkins: free
                        AWX: free (no support)    (but maintenance cost)
When to use:            Ansible is primary         Ansible is one tool
                        automation tool            among many in CI/CD

NovaMart approach:
  Jenkins: CI/CD pipeline orchestrator (build, test, deploy)
  AWX: Ansible-specific operations (server config, patching, ad-hoc tasks)
  Jenkins triggers AWX via API for Ansible steps in the pipeline
```

---

## 6. Advanced Ansible Patterns

### Delegation and Local Actions

```yaml
# Run a task on a DIFFERENT host than the current play target
- name: Remove server from load balancer before maintenance
  ansible.builtin.uri:
    url: "http://lb.novamart.internal/api/drain/{{ inventory_hostname }}"
    method: POST
  delegate_to: localhost    # Run on control node, not target server
  # Target server is still the "context" — {{ inventory_hostname }} 
  # refers to the TARGET, not the delegate

# Run once for the entire play (not once per host)
- name: Send deployment start notification
  ansible.builtin.uri:
    url: "{{ slack_webhook }}"
    method: POST
    body_format: json
    body:
      text: "Deployment starting for {{ ansible_play_hosts | length }} hosts"
  delegate_to: localhost
  run_once: true    # Only ONE Slack message, not one per host
```

### Dynamic Includes vs Static Imports

```yaml
# IMPORT (static) — processed at playbook PARSE time
- name: Import database tasks
  ansible.builtin.import_tasks: database.yml
  # ✓ Tags, when, and other directives apply to EACH task inside
  # ✓ Shows up in --list-tasks
  # ✗ Cannot use variables in the filename
  # ✗ Cannot use loop

# INCLUDE (dynamic) — processed at EXECUTION time
- name: Include OS-specific tasks
  ansible.builtin.include_tasks: "{{ ansible_os_family | lower }}.yml"
  # ✓ Can use variables in filename
  # ✓ Can use loop
  # ✗ Tags on include don't propagate to included tasks
  # ✗ Doesn't show in --list-tasks
  # ✗ Handlers inside includes have scoping issues

# RULE OF THUMB:
# Use import_tasks for static, known task files
# Use include_tasks when the filename depends on runtime variables
```

### Lookup Plugins and Filters

```yaml
# ── LOOKUP PLUGINS: fetch data from external sources ──

# Read a file from the control node
- name: Set SSH public key
  ansible.builtin.authorized_key:
    user: deploy
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

# Read from environment variable
- name: Set API key from CI environment
  ansible.builtin.set_fact:
    api_key: "{{ lookup('env', 'NOVAMART_API_KEY') }}"

# Read from AWS SSM Parameter Store
- name: Get database password from SSM
  ansible.builtin.set_fact:
    db_password: "{{ lookup('amazon.aws.ssm_parameter', '/novamart/prod/db_password') }}"

# Read from HashiCorp Vault
- name: Get secret from Vault
  ansible.builtin.set_fact:
    db_password: "{{ lookup('community.hashi_vault.hashi_vault', 
                    'secret/data/novamart/production/db 
                     auth_method=aws 
                     url=https://vault.novamart.internal:8200') }}"

# Generate a password
- name: Generate random password
  ansible.builtin.set_fact:
    temp_password: "{{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}"

# ── USEFUL FILTERS ──

# Default value (prevent undefined errors)
"{{ database_port | default(5432) }}"

# Type conversion
"{{ max_connections | int }}"
"{{ enable_ssl | bool }}"

# String manipulation
"{{ hostname | upper }}"
"{{ 'Hello World' | regex_replace('World', 'NovaMart') }}"

# List operations
"{{ groups['webservers'] | length }}"                    # Count
"{{ groups['webservers'] | join(',') }}"                 # Join
"{{ [1,2,3,4,5] | select('greaterthan', 3) | list }}"  # Filter

# IP address operations
"{{ '10.0.1.5/24' | ansible.utils.ipaddr('network') }}"  # 10.0.1.0
"{{ '10.0.1.5/24' | ansible.utils.ipaddr('netmask') }}"  # 255.255.255.0

# Hash/encrypt
"{{ 'password' | password_hash('sha512') }}"    # For user module
```

### Tags — Selective Execution

```yaml
# Apply tags to tasks
- name: Install packages
  ansible.builtin.apt:
    name: nginx
    state: present
  tags: [install, nginx]

- name: Configure nginx
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  tags: [configure, nginx]

- name: Start nginx
  ansible.builtin.service:
    name: nginx
    state: started
  tags: [service, nginx]

# Run only tagged tasks:
ansible-playbook site.yml --tags "configure"       # Only config tasks
ansible-playbook site.yml --tags "nginx"            # All nginx tasks
ansible-playbook site.yml --skip-tags "install"     # Everything except install

# Special tags:
#   always — task runs even if you filter with --tags (unless explicitly --skip-tags always)
#   never  — task never runs unless explicitly --tags never

- name: DANGEROUS - reset all data
  ansible.builtin.command: /opt/novamart/bin/reset_db.sh
  tags: [never, reset]
  # Only runs if: ansible-playbook site.yml --tags reset
```

---

## How All of This Breaks — Advanced Failure Modes

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **Vault password lost** | Can't decrypt any encrypted files | Password not stored securely, person left company | Store vault pass in AWS Secrets Manager, enable multi-vault-id per env |
| **Molecule test passes, prod fails** | Role works in Docker, breaks on real servers | Docker ≠ real server (systemd hack, kernel differences, SELinux) | Add EC2-based molecule scenario for critical roles |
| **Ansible + Terraform race** | Ansible runs before Terraform finishes | CI pipeline stages not properly sequenced | Explicit dependency: `needs: terraform-apply` in CI, plus `wait_for` SSH in playbook |
| **AWX credential exposure** | User sees "extra vars" containing secrets | Secrets passed as extra_vars instead of AWX credentials | Use AWX credential types, not survey/extra_vars for secrets |
| **Import vs Include confusion** | Tags don't propagate, `--list-tasks` incomplete | Using `include_tasks` where `import_tasks` was appropriate | Use import for static, include only when filename is dynamic |
| **Lookup plugin failure** | `ERROR! lookup plugin 'aws_ssm' not found` | Collection not installed | `ansible-galaxy collection install amazon.aws` |
| **Async task orphaned** | Background task runs forever, no one checks result | Missing `async_status` check, or play fails before poll | Always pair async with async_status, add `async` timeout |
| **Delegation confusion** | Task runs on wrong host, variables from wrong context | `delegate_to` changes WHERE, not the variable context | Remember: `{{ inventory_hostname }}` is still the TARGET host even with delegate_to |


---

## Quick Reference Card

```
ANSIBLE VAULT:
  encrypt/decrypt/view/edit/rekey/encrypt_string
  Naming: vault_<var> in vault.yml → reference as <var> in vars.yml
  CI: vault_password_file = ./script_that_reads_secrets_manager.sh
  Multi-env: --vault-id prod@file --vault-id staging@file
  ALWAYS: no_log: true on tasks using secrets

PERFORMANCE:
  forks: 25+ | pipelining: True | gather_facts: false/cached
  strategy: free (independent hosts) | async/poll (long tasks)
  Mitogen: 2-7x faster (but test compatibility)
  Native modules > command/shell (always)
  Pass lists to modules (apt name: [list]) > loop

MOLECULE:
  molecule init scenario --driver-name docker
  molecule test (full: create→converge→idempotence→verify→destroy)
  molecule converge + verify (dev iteration)
  Verify: assert service running, port listening, config correct
  IDEMPOTENCE: 2nd run must show 0 changed

TERRAFORM + ANSIBLE:
  RIGHT: Terraform provisions + tags → Ansible discovers + configures
  WRONG: Terraform provisioner "local-exec" ansible-playbook
  Bridge: terraform output -json → group_vars/env/terraform.yml
  CI: terraform apply → sync vars → ansible-playbook → verify

TOWER/AWX:
  Template = Playbook + Inventory + Credential
  Workflow = Chain of templates with conditions
  RBAC: Org → Team → User → Permission per template
  Use for: scheduled runs, RBAC, audit, non-CLI users
  Jenkins triggers AWX API for Ansible steps

DELEGATION:
  delegate_to: localhost (run on control node)
  run_once: true (one execution for entire play)
  delegate_to changes WHERE, not variable CONTEXT

IMPORTS vs INCLUDES:
  import_tasks: static, parsed at load time, tags propagate
  include_tasks: dynamic, parsed at runtime, can use variables in filename
```
Set 1:
Q1. A NovaMart engineer commits a plaintext database password to group_vars/production/vars.yml, pushes it to Bitbucket, and then realizes the mistake. Walk through the COMPLETE incident response: immediate actions (first 5 minutes), remediation (next hour), and prevention (permanent). Include the exact Ansible Vault commands to encrypt the file, the Git commands to scrub history, and the process changes to ensure this never happens again. What do you do about the password that was already exposed?

Q2. NovaMart's platform team wants to roll out a kernel parameter change (net.core.somaxconn = 65535) across all 200 EC2 instances managed by Ansible. The fleet is mixed: 120 Amazon Linux 2, 50 Ubuntu 22.04, and 30 RHEL 9. Write the complete role (tasks, handlers, templates, molecule tests) that handles all three OS families, is fully idempotent, verifies the change took effect, and includes a molecule test that proves idempotency. Explain your serial strategy and what happens if 5 hosts fail mid-rollout.

Q3. You're designing the Ansible + Terraform integration pipeline for NovaMart's new disaster recovery region (eu-central-1). Terraform will create 15 EC2 instances (bastions, Vault servers, monitoring). Ansible must configure them. Design the complete Jenkins pipeline (stages, error handling, notifications) and explain: How does Ansible know about the new instances? How are Terraform outputs passed to Ansible? What happens if Terraform succeeds but Ansible fails on 3 of 15 hosts? How do you handle secrets (Vault password, SSH keys, AWS credentials) across both tools in CI?

Q4. Your Molecule tests for the nginx role pass in CI (Docker-based), but the role fails when applied to production Amazon Linux 2 instances. The error is: TASK [nginx : Start nginx] fatal: [web-14]: FAILED! => {"msg": "Could not find the requested service nginx: host"}. Debug this systematically. What are the possible causes? How would you modify the Molecule setup to catch this class of failure before it reaches production? What's the fundamental limitation of Docker-based Molecule testing that this exposes?

Set 2:
Q1. Vault + CI/CD + Runtime Secret Hygiene
NovaMart has:

Jenkins running Ansible playbooks
AWX for scheduled ops tasks
HashiCorp Vault as the long-term secret store
A small set of bootstrap secrets still stored in Ansible Vault
Design the full secret flow for production:

where the Ansible Vault password lives
how Jenkins retrieves it
how AWX retrieves it
how playbooks avoid leaking secrets in logs
how bootstrap secrets differ from runtime app secrets
how rotation works when a platform engineer leaves the company
Your answer must include:

repository layout
ansible.cfg / vault-password script pattern
no_log usage
failure modes if .vault_pass gets committed
Q2. Performance Tuning a Slow Ansible Estate
A patching playbook takes 70 minutes across 120 EC2 instances. You investigate and find:

forks = 10
serial: 1
gather_facts: true on 5 plays
no fact caching
strategy: linear
package installs are long-running
multiple tasks use loops over packages instead of passing lists
dynamic inventory calls AWS every run with no inventory cache
bastion SSH is used but no ControlPersist
Explain:

every performance bottleneck
the exact fix for each
which fixes give the largest gains
what you would not optimize yet
a realistic post-fix runtime estimate
Then provide:

corrected ansible.cfg
one corrected play example using strategy: free, async package install, and reduced fact gathering
Q3. Molecule Test Strategy for a Critical Role
You own a vault_server Ansible role that:

installs Vault on EC2
configures systemd
writes TLS config
opens firewall rules
requires a restart on config change
behaves differently on Ubuntu 22.04 and Amazon Linux 2
Design the complete Molecule test strategy:

scenario layout
Docker vs EC2 tradeoffs
what to verify in converge
what idempotence must prove
what Docker cannot validate
when to add an EC2-backed Molecule scenario
how CI should run fast tests vs slow tests
Include:

directory structure
sample molecule.yml approach
sample assertions
at least 5 production failure modes Molecule would catch and 3 it would miss
Q4. Terraform + Ansible Integration at NovaMart
NovaMart provisions EC2 instances, security groups, IAM roles, and RDS with Terraform, then configures instances with Ansible.

You need to design a production-safe integration pattern that:

avoids Terraform provisioners
passes Terraform outputs to Ansible
uses dynamic inventory by tags
ensures Ansible does not run before instances are actually reachable
supports re-running Ansible without re-running Terraform
supports rollback and verification
Provide:

pipeline stages
tagging strategy
output synchronization pattern
waiting/reachability logic
why local-exec/remote-exec is a trap
what drift each tool can and cannot detect
Q5. AWX vs Jenkins — Real Separation of Duties
At NovaMart, one team wants to run everything from Jenkins. Another wants AWX for all operational automation.

Give the correct split of responsibilities between Jenkins and AWX for:

CI/CD application delivery
scheduled patching
ad-hoc operational tasks
credential management
inventory sync
RBAC
audit trail
approvals for production changes
Then answer this scenario:
A developer wants to restart a staging service on a set of EC2 hosts but must never touch production. Show how AWX should enforce this safely, and explain why Jenkins job permissions are weaker for this use case.

# SET 1 — Answers

---

## Set 1, Q1: Plaintext Password Committed to Git — Full Incident Response

### Phase 1: Immediate Actions (First 5 Minutes)

**The password is burned. This is the most important thing to internalize.** Even if you delete the commit, scrub history, and force-push — anyone who cloned the repo, any CI runner that fetched, any Bitbucket mirror, any backup system — they all have the plaintext. The password must be rotated regardless of what you do to Git.

```bash
# MINUTE 0-1: Rotate the exposed credential IMMEDIATELY
# This is step ONE, not step five. Every second it's valid is exposure.

# Identify what database the password is for
cat group_vars/production/vars.yml | grep -i pass
# Output: db_password: "SuperSecret123!"

# Connect to AWS and rotate the RDS master password
aws rds modify-db-instance \
  --db-instance-identifier novamart-prod-db \
  --master-user-password "$(openssl rand -base64 32)" \
  --apply-immediately

# If it's an application-level DB user (not master), connect and rotate:
mysql -h novamart-prod-db.xxxxx.us-east-1.rds.amazonaws.com -u admin -p <<EOF
ALTER USER 'novamart_app'@'%' IDENTIFIED BY '$(openssl rand -base64 32)';
FLUSH PRIVILEGES;
EOF

# Store the NEW password in HashiCorp Vault (the real secret store)
vault kv put secret/novamart/production/database \
  password="$(vault kv get -field=password secret/novamart/production/database 2>/dev/null || openssl rand -base64 32)"

# MINUTE 1-2: Revoke Bitbucket access to limit further exposure
# If the repo is public (worst case), make it private immediately
# If internal, check who has cloned recently

# MINUTE 2-3: Notify the security team
# This is a security incident — don't try to handle it silently
# Post in #security-incidents with:
#   - What was exposed: RDS production database password
#   - When: commit SHA abc123, pushed at 14:32 UTC
#   - Blast radius: anyone with repo read access
#   - Status: password already rotated
```

**Why rotation comes before Git cleanup:** Git history scrubbing takes 15-30 minutes. If an attacker has a clone, they already have the password. Rotation makes the exposed password worthless instantly. Everything else is cleanup.

### Phase 2: Remediation (Next Hour)

**Step 1: Encrypt the file with Ansible Vault**

```bash
# Encrypt the entire vars file
ansible-vault encrypt group_vars/production/vars.yml \
  --vault-id production@~/.vault_pass_prod

# Or better: extract ONLY secrets into a vault-encrypted file
# and keep non-secret vars in plaintext (easier to review in PRs)

# Create the encrypted secrets file
cat > group_vars/production/vault.yml << 'EOF'
vault_db_password: "NEW_PASSWORD_FROM_HASHICORP_VAULT"
vault_api_key: "other-secret-here"
EOF

ansible-vault encrypt group_vars/production/vault.yml \
  --vault-id production@~/.vault_pass_prod

# Reference vaulted vars from the main vars file
cat > group_vars/production/vars.yml << 'EOF'
# Non-secret configuration
db_host: novamart-prod-db.xxxxx.us-east-1.rds.amazonaws.com
db_port: 3306
db_name: novamart

# Secret references (actual values in vault.yml)
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
EOF

# Verify encryption
head -1 group_vars/production/vault.yml
# Output: $ANSIBLE_VAULT;1.1;AES256

# Verify decryption works
ansible-vault view group_vars/production/vault.yml \
  --vault-id production@~/.vault_pass_prod
```

**Step 2: Scrub the password from Git history**

```bash
# Option A: git-filter-repo (preferred — faster, safer than BFG or filter-branch)
# Install if needed: pip install git-filter-repo

# First, clone a FRESH copy (git-filter-repo requires this)
git clone --mirror git@bitbucket.org:novamart/ansible-config.git
cd ansible-config.git

# Remove the file from ALL history
git filter-repo --path group_vars/production/vars.yml --invert-paths

# OR if you want to keep the file but replace the secret in history:
git filter-repo --blob-callback '
  if b"SuperSecret123!" in blob.data:
    blob.data = blob.data.replace(b"SuperSecret123!", b"REDACTED")
'

# Force-push the rewritten history
git push --force --all
git push --force --tags

# Option B: BFG Repo Cleaner (simpler syntax, Java-based)
# Create a file with the passwords to scrub
echo "SuperSecret123!" > passwords_to_remove.txt

bfg --replace-text passwords_to_remove.txt ansible-config.git
cd ansible-config.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force --all

# Shred the passwords file
shred -u passwords_to_remove.txt
```

**Step 3: Force all team members to re-clone**

```bash
# After force-push, existing clones still have the old history
# Notify the team:
# "Force push completed on ansible-config. Delete your local clone 
#  and re-clone. Do NOT git pull — it will merge the old history back in."

# In Bitbucket: Settings → Branch permissions → 
# Temporarily allow force push, then re-restrict after cleanup
```

**Step 4: Audit exposure**

```bash
# Check Bitbucket audit log for who accessed the repo
# Bitbucket Cloud: Admin → Audit log → filter by repo
# Bitbucket Server: /rest/audit/latest/events

# Check CI systems — Jenkins workspace may have the plaintext cached
# On every Jenkins agent:
find /var/lib/jenkins/workspace -name "vars.yml" -exec grep -l "SuperSecret123" {} \;
# Delete any matches

# Check if Ansible logged the password during any run
grep -r "SuperSecret123" /var/log/jenkins/ /var/log/ansible/
```

### Phase 3: Prevention (Permanent)

**1. Pre-commit hook to detect secrets:**

```bash
# .pre-commit-config.yaml (in the repo root)
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/ansible-community/ansible-lint
    rev: v6.22.0
    hooks:
      - id: ansible-lint

# Install hooks for the team
pre-commit install

# Test it catches plaintext secrets
echo 'password: "test123"' >> test.yml
git add test.yml
git commit -m "test"
# Output: gitleaks has detected secrets in your code. Commit blocked.
```

**2. Gitleaks configuration for Ansible-specific patterns:**

```toml
# .gitleaks.toml
title = "NovaMart Gitleaks Config"

[[rules]]
id = "ansible-plaintext-password"
description = "Plaintext password in Ansible vars"
regex = '''(?i)(password|passwd|secret|api_key|token)\s*[:=]\s*["']?[^\s{$]'''
# Matches: password: "plaintext" but NOT password: "{{ vault_db_password }}"
# The [^\s{$] excludes Jinja2 variable references
path = '''group_vars/.*\.yml$'''

[[allowlist]]
paths = ['''.*vault\.yml$''']
# Don't scan vault-encrypted files (they're binary-looking)
```

**3. CI pipeline gate in Jenkinsfile:**

```groovy
stage('Secret Scan') {
    steps {
        sh 'gitleaks detect --source . --verbose --no-git'
        // If any plaintext secrets found, this exits non-zero → build fails
    }
}
```

**4. Bitbucket branch permission — block pushes to main without CI passing:**

```
Repository Settings → Branch permissions → main:
  ✓ Require passing builds (includes the secret scan stage)
  ✓ Require pull request (no direct pushes)
  ✓ Require 2 approvals (second pair of eyes on vars changes)
```

**5. Ansible Vault password file in .gitignore:**

```bash
echo ".vault_pass*" >> .gitignore
echo "*.vault_password" >> .gitignore
git add .gitignore
git commit -m "chore: prevent vault password files from being committed"
```

**6. Vault password delivery via script (not file):**

```bash
#!/bin/bash
# vault_pass_client.sh — retrieves vault password from HashiCorp Vault
# This script is set as vault_password_file in ansible.cfg
# The actual vault password never touches disk

vault kv get -field=ansible_vault_password secret/novamart/ansible/vault_pass
```

```ini
# ansible.cfg
[defaults]
vault_password_file = ./vault_pass_client.sh
```

### Summary: What Happened to the Exposed Password

```
Timeline:
  T+0min:   Password committed to Bitbucket in plaintext
  T+2min:   Engineer realizes mistake
  T+3min:   RDS password rotated → exposed password is NOW WORTHLESS
  T+5min:   Security team notified, incident ticket opened
  T+15min:  Ansible Vault encryption applied to vars file
  T+30min:  Git history scrubbed with git-filter-repo
  T+35min:  Force-push to Bitbucket, team notified to re-clone
  T+45min:  CI workspaces and logs audited for remnants
  T+60min:  Pre-commit hooks and CI gates deployed
  T+90min:  Incident post-mortem written

The password that was exposed is DEAD. Even if someone has the old Git history,
the password no longer works because it was rotated in minute 3.
```

---

## Set 1, Q2: Kernel Parameter Rollout Across Mixed Fleet

### Role Structure

```
roles/kernel_tuning/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── sysctl.yml
│   └── verify.yml
├── templates/
│   └── 99-novamart-tuning.conf.j2
├── molecule/
│   ├── default/
│   │   ├── converge.yml
│   │   ├── molecule.yml
│   │   └── verify.yml
│   └── multi-os/
│       ├── converge.yml
│       ├── molecule.yml
│       └── verify.yml
├── meta/
│   └── main.yml
└── README.md
```

### defaults/main.yml

```yaml
---
# Kernel parameters to apply
# Defined as a dictionary so it's extensible without code changes
kernel_params:
  net.core.somaxconn: 65535
  net.ipv4.tcp_max_syn_backlog: 65535
  net.core.netdev_max_backlog: 65535

# Sysctl config file path — drop-in file, don't edit /etc/sysctl.conf
sysctl_config_file: /etc/sysctl.d/99-novamart-tuning.conf

# Whether to apply params immediately (true) or wait for reboot
apply_immediately: true
```

### tasks/main.yml

```yaml
---
# Entrypoint — OS family detection and parameter application

- name: Gather OS family facts only
  ansible.builtin.setup:
    gather_subset:
      - '!all'
      - '!min'
  # min subset is gathered by default — gives us ansible_os_family

- name: Validate target OS is supported
  ansible.builtin.assert:
    that:
      - ansible_os_family in ['RedHat', 'Debian']
    fail_msg: >
      Unsupported OS family '{{ ansible_os_family }}' on {{ inventory_hostname }}.
      This role supports RedHat (Amazon Linux 2, RHEL 9) and Debian (Ubuntu 22.04).

- name: Include sysctl configuration tasks
  ansible.builtin.include_tasks: sysctl.yml

- name: Include verification tasks
  ansible.builtin.include_tasks: verify.yml
```

### tasks/sysctl.yml

```yaml
---
# Apply kernel parameters using sysctl module (idempotent, OS-agnostic)

# Install procps if missing (provides sysctl binary)
# Amazon Linux 2 and RHEL have it by default; Ubuntu minimal might not
- name: Ensure procps is installed (provides sysctl)
  ansible.builtin.package:
    name: procps
    state: present

# Ensure the sysctl.d directory exists (it should, but be defensive)
- name: Ensure sysctl.d directory exists
  ansible.builtin.file:
    path: /etc/sysctl.d
    state: directory
    owner: root
    group: root
    mode: '0755'

# Deploy the sysctl config file from template
- name: Deploy NovaMart kernel tuning configuration
  ansible.builtin.template:
    src: 99-novamart-tuning.conf.j2
    dest: "{{ sysctl_config_file }}"
    owner: root
    group: root
    mode: '0644'
    validate: '/sbin/sysctl -p %s'
  notify: reload sysctl

# Apply each parameter using the native sysctl module
# This is belt-and-suspenders: the template + handler covers file persistence,
# the sysctl module covers runtime application
- name: Apply kernel parameters at runtime
  ansible.posix.sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    sysctl_file: "{{ sysctl_config_file }}"
    state: present
    reload: "{{ apply_immediately }}"
  loop: "{{ kernel_params | dict2items }}"
  loop_control:
    label: "{{ item.key }}={{ item.value }}"
```

### templates/99-novamart-tuning.conf.j2

```ini
# Managed by Ansible — do not edit manually
# Role: kernel_tuning
# Applied: {{ ansible_date_time.iso8601 }}
# Host: {{ inventory_hostname }}

{% for param, value in kernel_params.items() %}
{{ param }} = {{ value }}
{% endfor %}
```

### handlers/main.yml

```yaml
---
- name: reload sysctl
  ansible.builtin.command:
    cmd: sysctl --system
  changed_when: true
  # sysctl --system loads ALL files in /etc/sysctl.d/*.conf
  # This is more thorough than sysctl -p (which only loads /etc/sysctl.conf)
  listen: reload sysctl
```

### tasks/verify.yml

```yaml
---
# Post-application verification — every parameter must match desired state

- name: Verify kernel parameters are active
  ansible.builtin.command:
    cmd: "sysctl -n {{ item.key }}"
  register: sysctl_verify
  loop: "{{ kernel_params | dict2items }}"
  loop_control:
    label: "{{ item.key }}"
  changed_when: false
  failed_when: sysctl_verify.stdout | int != item.value | int

- name: Verify parameters survive reboot (persist in config file)
  ansible.builtin.command:
    cmd: "grep -c '{{ item.key }}' {{ sysctl_config_file }}"
  register: persist_verify
  loop: "{{ kernel_params | dict2items }}"
  loop_control:
    label: "{{ item.key }}"
  changed_when: false
  failed_when: persist_verify.stdout | int < 1
```

### Molecule Tests

**molecule/default/molecule.yml**

```yaml
---
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: amazonlinux2
    image: amazonlinux:2
    command: /sbin/init
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    privileged: true
    pre_build_image: true

  - name: ubuntu2204
    image: ubuntu:22.04
    command: /sbin/init
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    privileged: true
    pre_build_image: true

  - name: rhel9
    image: redhat/ubi9:latest
    command: /sbin/init
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    privileged: true
    pre_build_image: true

provisioner:
  name: ansible
  playbooks:
    converge: converge.yml
    verify: verify.yml

verifier:
  name: ansible

scenario:
  name: default
  test_sequence:
    - dependency
    - lint
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence    # KEY: runs converge AGAIN — must show 0 changed
    - verify
    - cleanup
    - destroy
```

**molecule/default/converge.yml**

```yaml
---
- name: Converge
  hosts: all
  become: true
  roles:
    - role: kernel_tuning
```

**molecule/default/verify.yml**

```yaml
---
- name: Verify kernel parameters
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: Check net.core.somaxconn is set to 65535
      ansible.builtin.command:
        cmd: sysctl -n net.core.somaxconn
      register: somaxconn
      changed_when: false
      failed_when: somaxconn.stdout | trim != '65535'

    - name: Check config file exists and contains the parameter
      ansible.builtin.lineinfile:
        path: /etc/sysctl.d/99-novamart-tuning.conf
        regexp: '^net\.core\.somaxconn'
        state: absent
      check_mode: true
      register: config_check
      failed_when: not config_check.found
      # In check_mode with state:absent, 'found' tells us if the line exists
      # without actually removing it

    - name: Verify idempotence — run role again and check no changes
      ansible.builtin.debug:
        msg: >
          If idempotence step passed above, the role made 0 changes
          on the second run. This proves the sysctl module correctly
          detects existing values and skips.
```

**Important caveat about Docker and sysctl:** Docker containers share the host kernel. `sysctl -w net.core.somaxconn=65535` inside a container **actually changes the host's kernel parameter** if the container is privileged, or silently fails if not. This is a known limitation. The Molecule Docker test validates:
- File creation and template rendering
- Package installation
- Task idempotency (second run = 0 changes)
- OS-family branching logic

It does NOT validate that the kernel parameter actually took effect in an isolated way. For that, you need the EC2-backed scenario (see Set 2, Q3 for that design).

### Playbook with Serial Strategy and Failure Handling

```yaml
---
# site.yml — Apply kernel tuning across all 200 hosts
- name: Roll out kernel parameter changes
  hosts: all
  become: true
  serial:
    - 1        # First batch: 1 host (canary)
    - 5        # Second batch: 5 hosts (small group)
    - "25%"    # Remaining batches: 25% of remaining fleet at a time
  max_fail_percentage: 10

  pre_tasks:
    - name: Notify deployment start
      ansible.builtin.uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "🔧 Kernel tuning rollout starting: batch {{ ansible_play_batch }} of {{ ansible_play_hosts | length }} hosts"
      delegate_to: localhost
      run_once: true
      tags: [notify]

  roles:
    - role: kernel_tuning

  post_tasks:
    - name: Notify batch completion
      ansible.builtin.uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "✅ Kernel tuning batch complete: {{ ansible_play_hosts | length }} hosts configured"
      delegate_to: localhost
      run_once: true
      tags: [notify]
```

### What Happens When 5 Hosts Fail Mid-Rollout

```
Scenario: serial: [1, 5, "25%"], max_fail_percentage: 10, fleet: 200 hosts

Batch 1 (canary):  1 host → succeeds → continue
Batch 2:           5 hosts → all succeed → continue
Batch 3:          49 hosts → 5 fail (10.2%) → EXCEEDS 10% threshold
                   → Ansible ABORTS the play
                   → Remaining 145 hosts are NEVER touched
                   → Failed hosts have PARTIAL state

What state are the 5 failed hosts in?
  - If failure was in 'sysctl.yml' during parameter application:
    → Config file may be deployed but runtime params not applied
    → Fix: Re-run playbook with --limit @kernel_tuning.retry
    
  - If failure was in 'verify.yml':
    → Parameters are applied but verification detected a mismatch
    → This means a deeper OS issue (read-only sysctl, SELinux, etc.)
    → Fix: Manual investigation, then re-run

  - The .retry file:
    → Ansible writes failed hosts to kernel_tuning.retry
    → Re-run: ansible-playbook site.yml --limit @kernel_tuning.retry

The 195 successful hosts:
  → Parameters are applied and verified
  → Idempotent: re-running the playbook on them changes nothing

The 145 untouched hosts:
  → Still on old parameters
  → Safe: they were never modified
  → After fixing the 5 failures, re-run without --limit to catch remaining
```

---

## Set 1, Q3: Terraform + Ansible Integration Pipeline for DR Region

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        Jenkins Pipeline                          │
│                                                                  │
│  ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐  │
│  │Terraform│──→│ Wait for │──→│ Ansible  │──→│  Smoke Test  │  │
│  │  Apply  │   │ Readiness│   │Configure │   │  & Notify    │  │
│  └─────────┘   └──────────┘   └──────────┘   └──────────────┘  │
│       │              ↑              │                             │
│       │         SSH ready?          │                             │
│       ↓         Port 22 open?      ↓                             │
│  Outputs:       cloud-init done?   Inventory:                    │
│  instance IPs                      aws_ec2 plugin                │
│  bastion IP                        discovers by tags             │
│  SG IDs                                                          │
│  RDS endpoint                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Tagging Strategy (Terraform Side)

```hcl
# Terraform tags every instance with metadata Ansible needs
locals {
  common_tags = {
    Environment  = "dr"
    Region       = "eu-central-1"
    ManagedBy    = "terraform"
    ConfiguredBy = "ansible"
    Project      = "novamart"
  }
}

resource "aws_instance" "bastion" {
  count         = 2
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.common_tags, {
    Name = "dr-bastion-${count.index}"
    Role = "bastion"
    OS   = "amazon_linux"
  })
}

resource "aws_instance" "vault" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"
  subnet_id     = aws_subnet.private[count.index].id

  tags = merge(local.common_tags, {
    Name = "dr-vault-${count.index}"
    Role = "vault_server"
    OS   = "ubuntu"
  })
}

resource "aws_instance" "monitoring" {
  count         = 3
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.large"
  subnet_id     = aws_subnet.private[count.index].id

  tags = merge(local.common_tags, {
    Name = "dr-monitoring-${count.index}"
    Role = "monitoring"
    OS   = "amazon_linux"
  })
}

# Terraform outputs — consumed by Ansible
output "bastion_public_ips" {
  value = aws_instance.bastion[*].public_ip
}

output "vault_private_ips" {
  value = aws_instance.vault[*].private_ip
}

output "monitoring_private_ips" {
  value = aws_instance.monitoring[*].private_ip
}

output "rds_endpoint" {
  value     = aws_db_instance.dr.endpoint
  sensitive = true
}

output "all_instance_ids" {
  value = concat(
    aws_instance.bastion[*].id,
    aws_instance.vault[*].id,
    aws_instance.monitoring[*].id
  )
}
```

### Ansible Dynamic Inventory

```yaml
# inventory/dr_aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - eu-central-1

filters:
  tag:Environment: dr
  tag:Project: novamart
  instance-state-name: running

cache: true
cache_plugin: jsonfile
cache_connection: /tmp/dr_inventory_cache
cache_timeout: 120

hostnames:
  - private-ip-address

compose:
  ansible_user: "'ubuntu' if tags.OS == 'ubuntu' else 'ec2-user'"
  ansible_host: private_ip_address

keyed_groups:
  - key: tags.Role
    prefix: role
    separator: "_"

strict: true
```

### Output Synchronization: Terraform → Ansible

```bash
# Method: Terraform outputs written to a vars file consumed by Ansible
# This avoids Terraform provisioners entirely

# In the pipeline script:
terraform output -json > /tmp/tf_outputs.json

# Convert to Ansible extra-vars format
python3 << 'PYEOF'
import json
with open('/tmp/tf_outputs.json') as f:
    outputs = json.load(f)

ansible_vars = {}
for key, val in outputs.items():
    ansible_vars[f"tf_{key}"] = val['value']

with open('terraform_outputs.yml', 'w') as f:
    import yaml
    yaml.dump(ansible_vars, f, default_flow_style=False)
PYEOF

# Result: terraform_outputs.yml contains:
# tf_bastion_public_ips:
#   - 52.59.1.100
#   - 52.59.1.101
# tf_rds_endpoint: novamart-dr.xxxxx.eu-central-1.rds.amazonaws.com
# tf_vault_private_ips:
#   - 10.1.1.10
#   - 10.1.1.11
#   - 10.1.1.12

# Ansible consumes this with --extra-vars
ansible-playbook site.yml -e @terraform_outputs.yml
```

### Jenkinsfile — Complete Pipeline

```groovy
pipeline {
    agent { label 'infra-runner' }

    environment {
        AWS_REGION           = 'eu-central-1'
        TF_WORKSPACE         = 'dr'
        ANSIBLE_VAULT_PASSWORD_FILE = credentials('ansible-vault-password')
        // Jenkins credential 'ansible-vault-password' is a secret file
        // injected at runtime — never in repo
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
        // Only one DR deployment at a time
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Terraform Init & Plan') {
            steps {
                dir('terraform/dr') {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                     credentialsId: 'aws-dr-deployer']]) {
                        sh '''
                            terraform init -backend-config=backend-dr.hcl
                            terraform plan -out=dr.tfplan -detailed-exitcode
                        '''
                        // Exit code 2 = changes detected
                        // Exit code 0 = no changes
                        // Exit code 1 = error
                    }
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                dir('terraform/dr') {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                     credentialsId: 'aws-dr-deployer']]) {
                        sh 'terraform apply -auto-approve dr.tfplan'

                        // Export outputs for Ansible
                        sh '''
                            terraform output -json > /tmp/tf_outputs.json
                            python3 scripts/tf_to_ansible_vars.py \
                                /tmp/tf_outputs.json \
                                ${WORKSPACE}/ansible/terraform_outputs.yml
                        '''

                        // Capture all instance IDs for readiness check
                        sh '''
                            terraform output -json all_instance_ids | \
                                jq -r '.[]' > /tmp/instance_ids.txt
                        '''
                    }
                }
            }
        }

        stage('Wait for Instance Readiness') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                 credentialsId: 'aws-dr-deployer']]) {
                    sh '''
                        echo "Waiting for all instances to pass status checks..."
                        
                        # AWS-level: wait for instance status checks (hypervisor + OS)
                        while read instance_id; do
                            aws ec2 wait instance-status-ok \
                                --instance-ids "$instance_id" \
                                --region ${AWS_REGION}
                            echo "✓ $instance_id is status-ok"
                        done < /tmp/instance_ids.txt

                        # SSH-level: wait for SSH to actually accept connections
                        # cloud-init may still be running even after status-ok
                        BASTION_IP=$(python3 -c "
import yaml
with open('ansible/terraform_outputs.yml') as f:
    d = yaml.safe_load(f)
print(d['tf_bastion_public_ips'][0])
")
                        
                        echo "Bastion IP: $BASTION_IP"
                        
                        # Wait for bastion SSH
                        for i in $(seq 1 30); do
                            if ssh -o StrictHostKeyChecking=no \
                                   -o ConnectTimeout=5 \
                                   -i ${SSH_KEY} \
                                   ec2-user@${BASTION_IP} \
                                   "cloud-init status --wait && echo READY" 2>/dev/null | grep -q READY; then
                                echo "✓ Bastion SSH ready and cloud-init complete"
                                break
                            fi
                            echo "Waiting for bastion SSH... attempt $i/30"
                            sleep 10
                        done

                        # Wait for all private instances via bastion
                        python3 -c "
import yaml
with open('ansible/terraform_outputs.yml') as f:
    d = yaml.safe_load(f)
for ip in d.get('tf_vault_private_ips', []) + d.get('tf_monitoring_private_ips', []):
    print(ip)
" | while read private_ip; do
                            for i in $(seq 1 30); do
                                if ssh -o StrictHostKeyChecking=no \
                                       -o ConnectTimeout=5 \
                                       -o ProxyCommand="ssh -W %h:%p -i ${SSH_KEY} ec2-user@${BASTION_IP}" \
                                       -i ${SSH_KEY} \
                                       ec2-user@${private_ip} \
                                       "echo READY" 2>/dev/null | grep -q READY; then
                                    echo "✓ $private_ip SSH ready"
                                    break
                                fi
                                echo "Waiting for $private_ip... attempt $i/30"
                                sleep 10
                            done
                        done
                    '''
                }
            }
        }

        stage('Ansible Configure') {
            steps {
                dir('ansible') {
                    withCredentials([
                        [$class: 'AmazonWebServicesCredentialsBinding',
                         credentialsId: 'aws-dr-deployer'],
                        sshUserPrivateKey(credentialsId: 'novamart-dr-ssh-key',
                                         keyFileVariable: 'SSH_KEY')
                    ]) {
                        sh '''
                            export ANSIBLE_PRIVATE_KEY_FILE=${SSH_KEY}
                            
                            # Invalidate inventory cache to pick up new instances
                            rm -rf /tmp/dr_inventory_cache

                            # Verify inventory discovers all 15 instances
                            DISCOVERED=$(ansible-inventory -i inventory/dr_aws_ec2.yml \
                                --list | python3 -c "
import json, sys
data = json.load(sys.stdin)
hosts = data.get('_meta', {}).get('hostvars', {})
print(len(hosts))
")
                            
                            if [ "$DISCOVERED" -lt 15 ]; then
                                echo "ERROR: Expected 15 instances, discovered $DISCOVERED"
                                exit 1
                            fi
                            echo "✓ Discovered $DISCOVERED instances"

                            # Run the playbook
                            ansible-playbook site.yml \
                                -i inventory/dr_aws_ec2.yml \
                                -e @terraform_outputs.yml \
                                --vault-password-file ${ANSIBLE_VAULT_PASSWORD_FILE} \
                                -v
                        '''
                    }
                }
            }
        }

        stage('Smoke Test') {
            steps {
                dir('ansible') {
                    sh '''
                        ansible-playbook smoke_test.yml \
                            -i inventory/dr_aws_ec2.yml \
                            -e @terraform_outputs.yml \
                            --vault-password-file ${ANSIBLE_VAULT_PASSWORD_FILE}
                    '''
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#dr-deployments',
                color: 'good',
                message: "✅ DR region eu-central-1 provisioned and configured successfully.\n" +
                         "Terraform: 15 instances created\n" +
                         "Ansible: All hosts configured\n" +
                         "Build: ${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: '#dr-deployments',
                color: 'danger',
                message: "❌ DR deployment FAILED at stage: ${env.STAGE_NAME}\n" +
                         "Build: ${env.BUILD_URL}\n" +
                         "Action required: Check logs and determine if rollback needed."
            )
        }
        always {
            // Clean up sensitive files
            sh 'rm -f /tmp/tf_outputs.json /tmp/instance_ids.txt'
            archiveArtifacts artifacts: 'terraform/dr/dr.tfplan', allowEmptyArchive: true
        }
    }
}
```

### What Happens If Terraform Succeeds But Ansible Fails on 3 of 15 Hosts

```
Scenario: Terraform creates all 15 instances. Ansible configures 12 successfully.
3 Vault servers fail during TLS certificate deployment.

State of the world:
  - 15 instances exist in AWS (Terraform state is clean)
  - 12 instances are fully configured
  - 3 instances are PARTIALLY configured (some tasks ran, some didn't)
  - Terraform doesn't know about the Ansible failure
  - Ansible wrote the 3 failed hosts to site.retry

Recovery procedure:
  1. DO NOT re-run Terraform. Infrastructure is fine.
  2. Investigate the 3 failures:
     ansible-playbook site.yml \
       -i inventory/dr_aws_ec2.yml \
       --limit @site.retry \
       -e @terraform_outputs.yml \
       -v --diff
     
  3. If the failure was transient (SSH timeout, package mirror down):
     → Re-run fixes it because Ansible is idempotent
     → Already-completed tasks report "ok", failed tasks retry
  
  4. If the failure was structural (wrong AMI, missing IAM permission):
     → Fix the root cause in Terraform or Ansible
     → Re-run the limited playbook

  5. If a host is truly broken (corrupt filesystem, hardware):
     → terraform taint aws_instance.vault[2]
     → terraform apply  (replaces just that instance)
     → Re-run Ansible against the new instance

Key design principle: Ansible and Terraform are INDEPENDENTLY re-runnable.
  - Re-running Terraform when Ansible fails: unnecessary, wasteful, potentially dangerous
  - Re-running Ansible when Terraform hasn't changed: perfectly safe (idempotent)
  - Re-running Ansible with --limit: only touches failed hosts
```

### Secrets Handling Across Both Tools in CI

```
┌─────────────────────────────────────────────────────────────┐
│                    Jenkins Credentials Store                  │
│  (or HashiCorp Vault with Jenkins Vault plugin)             │
│                                                              │
│  aws-dr-deployer        → AWS_ACCESS_KEY_ID/SECRET          │
│  novamart-dr-ssh-key    → SSH private key for EC2           │
│  ansible-vault-password → Ansible Vault decryption password  │
│  slack-webhook          → Slack notification URL             │
└─────────────────────────────────────────────────────────────┘
         │
         │ injected at runtime via withCredentials()
         │ available only in the pipeline process
         │ never written to workspace or logs
         ↓
┌─────────────────────────────────────────────────────────────┐
│                    Pipeline Runtime                           │
│                                                              │
│  Terraform:                                                  │
│    AWS creds → environment variables (AWS_ACCESS_KEY_ID)     │
│    State encryption → S3 backend with SSE-KMS                │
│    Sensitive outputs → marked sensitive = true               │
│                                                              │
│  Ansible:                                                    │
│    SSH key → ANSIBLE_PRIVATE_KEY_FILE env var                │
│    Vault password → --vault-password-file (tmpfs)            │
│    Secrets in playbooks → ansible-vault encrypted files      │
│    Runtime secrets → no_log: true on tasks with passwords    │
│    HashiCorp Vault lookups → hashi_vault lookup plugin       │
└─────────────────────────────────────────────────────────────┘

What NEVER happens:
  ✗ AWS credentials in terraform.tfvars or env vars in Jenkinsfile
  ✗ SSH keys committed to repo
  ✗ Vault password in ansible.cfg
  ✗ Sensitive Terraform outputs piped to console
  ✗ Ansible tasks logging password values (no_log: true)
```

---

## Set 1, Q4: Molecule Docker vs Production — The systemd Trap

### The Error

```
TASK [nginx : Start nginx] fatal: [web-14]: FAILED! => 
{"msg": "Could not find the requested service nginx: host"}
```

### Systematic Debugging

**Possible Cause 1: nginx is not installed (package name differs)**

```bash
# Amazon Linux 2 uses 'amazon-linux-extras' for nginx, not yum directly
# If the role does:
- name: Install nginx
  ansible.builtin.yum:
    name: nginx
    state: present
# This WORKS in Docker (base amazonlinux:2 image has different repo config)
# but FAILS on real EC2 because nginx isn't in the default AL2 repos

# The fix:
- name: Enable nginx repo on Amazon Linux 2
  ansible.builtin.command:
    cmd: amazon-linux-extras install nginx1 -y
    creates: /usr/sbin/nginx
  when: ansible_distribution == "Amazon"
```

**Possible Cause 2: systemd is not running (most likely in Docker)**

Wait — the error is on **production**, not Docker. The Molecule tests PASS in Docker. Let me re-read.

The tests pass in Docker. The role fails on real EC2. The error is `"Could not find the requested service nginx: host"`. This means **the systemd service unit doesn't exist** on the target.

```bash
# Debug on the failing host:
ssh web-14

# Check if nginx binary exists
which nginx
# /usr/sbin/nginx — YES, it's installed

# Check if the service unit exists
systemctl list-unit-files | grep nginx
# Empty — NO service unit

# Check if the package installed a unit file
rpm -ql nginx | grep service
# Nothing — the package on AL2 may not include a systemd unit

# OR check if it's using a different service name
systemctl list-unit-files | grep -i "nginx\|httpd"
# Maybe it's registered as 'nginx.service' in a different path
```

**Possible Cause 3: The service name differs between OS/package versions**

```bash
# Amazon Linux 2 nginx from amazon-linux-extras:
# Service name: nginx
# Unit file: /usr/lib/systemd/system/nginx.service

# Amazon Linux 2 nginx from EPEL:
# Might install as 'nginx' or might not include a systemd unit

# If installed from source or a custom RPM:
# No systemd unit at all — you must provide one
```

**Root Cause Analysis:**

The most likely scenario: **The Docker-based Molecule test uses a different nginx package source than production.** In Docker:
- The `amazonlinux:2` image may have different repos enabled
- `yum install nginx` pulls from a repo that includes a systemd unit
- The test passes because the unit exists

In production EC2:
- `amazon-linux-extras install nginx1` is the correct method
- OR the repo configuration differs
- The nginx binary installs but without a systemd unit file
- `ansible.builtin.systemd: name=nginx state=started` fails because there's no unit

**The fix in the role:**

```yaml
- name: Deploy nginx systemd unit file
  ansible.builtin.template:
    src: nginx.service.j2
    dest: /etc/systemd/system/nginx.service
    owner: root
    group: root
    mode: '0644'
  notify:
    - reload systemd
    - restart nginx

# Don't rely on the package providing the unit file.
# Always deploy your own — you control it, you can tune it.
```

### How to Modify Molecule to Catch This

**Problem: Docker containers don't run systemd by default.** When Molecule tests run in Docker:

```
Docker container:
  - PID 1 is /bin/bash or /sbin/init (faked)
  - systemd is NOT running as the init system
  - systemctl commands either:
    a) Fail silently (if systemd is somewhat available)
    b) Get mocked by the container setup
    c) "Work" because privileged mode + cgroup mount fakes it
  
Production EC2:
  - PID 1 is /usr/lib/systemd/systemd (real)
  - systemctl commands interact with real service manager
  - If a unit file is missing, systemd returns a real error
```

**Fix 1: Make Docker-based Molecule tests more realistic:**

```yaml
# molecule/default/molecule.yml
platforms:
  - name: amazonlinux2
    image: amazonlinux:2
    command: /usr/sbin/init    # Run systemd as PID 1
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    privileged: true           # Required for systemd in Docker
    # cgroupns_mode: host      # For Docker with cgroups v2
    pre_build_image: true
    
    # Even with this setup, systemd in Docker is FRAGILE.
    # Some unit behaviors differ from real VMs.
```

**Fix 2: Add explicit service unit verification in Molecule verify:**

```yaml
# molecule/default/verify.yml
- name: Verify nginx deployment
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: Verify nginx systemd unit file exists
      ansible.builtin.stat:
        path: /etc/systemd/system/nginx.service
      register: unit_file
      failed_when: not unit_file.stat.exists

    - name: Verify nginx service is recognized by systemd
      ansible.builtin.command:
        cmd: systemctl list-unit-files nginx.service
      register: unit_list
      changed_when: false
      failed_when: "'nginx.service' not in unit_list.stdout"

    - name: Verify nginx is running
      ansible.builtin.command:
        cmd: systemctl is-active nginx
      register: nginx_active
      changed_when: false
      failed_when: nginx_active.stdout != 'active'

    - name: Verify nginx is responding on port 80
      ansible.builtin.uri:
        url: http://localhost:80
        status_code: [200, 301, 403]
      register: nginx_http
```

**Fix 3: Add an EC2-backed Molecule scenario for pre-production validation:**

```yaml
# molecule/ec2/molecule.yml
dependency:
  name: galaxy
driver:
  name: ec2
platforms:
  - name: mol-nginx-al2
    image: ami-0123456789abcdef0  # Actual Amazon Linux 2 AMI
    instance_type: t3.micro
    vpc_subnet_id: subnet-xxxxx
    security_group_name: molecule-testing
    key_name: molecule-test-key
    assign_public_ip: true
    tags:
      ManagedBy: molecule
      Purpose: testing

provisioner:
  name: ansible
  connection_options:
    ansible_user: ec2-user
    ansible_ssh_private_key_file: ~/.ssh/molecule-test-key.pem

verifier:
  name: ansible

scenario:
  name: ec2
  test_sequence:
    - dependency
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - verify
    - cleanup
    - destroy
```

### The Fundamental Limitation of Docker-Based Molecule Testing

```
What Docker CAN validate:
  ✓ Package installation logic
  ✓ Template rendering
  ✓ File permissions and ownership
  ✓ Task idempotency (second run = 0 changed)
  ✓ Variable interpolation
  ✓ Conditional logic (when: ansible_os_family == ...)
  ✓ Handler notification chains
  ✓ Syntax and lint compliance

What Docker CANNOT validate:
  ✗ systemd behavior (units, targets, dependencies, restart policies)
    → Docker's PID 1 is not real systemd; behavior differs
  ✗ Kernel parameters (sysctl)
    → Container shares host kernel; changes either affect host or fail silently
  ✗ Firewall rules (iptables/nftables/firewalld)
    → Containers use the host's network namespace (or bridged — neither matches production)
  ✗ SELinux/AppArmor policies
    → Docker typically runs with SELinux disabled or in permissive mode
  ✗ Network configuration (real NICs, routes, DNS resolution)
    → Docker networking != EC2 VPC networking
  ✗ Disk/mount behavior (fstab, LVM, EBS volumes)
    → Docker uses overlay filesystem, not real block devices
  ✗ Hardware interaction (NUMA, CPU governors, PCI devices)
    → Docker shares host hardware abstraction
  ✗ Cloud-init interaction
    → Docker doesn't run cloud-init
  ✗ IAM instance profile / metadata service
    → No IMDS endpoint in Docker

The rule:
  Docker Molecule = fast feedback on LOGIC (run in every PR, <2 min)
  EC2 Molecule = real validation on BEHAVIOR (run nightly or pre-release, ~10 min)
  Production canary = final proof (serial: 1 in the real playbook)
  
  All three layers are needed. Docker alone gives false confidence.
```

---

# SET 2 — Answers

---

## Set 2, Q1: Vault + CI/CD + Runtime Secret Hygiene

### Repository Layout

```
novamart-ansible/
├── ansible.cfg
├── vault_pass_client.sh            # Executable script, NOT a file with the password
├── .gitignore                      # Contains: .vault_pass*, *.vault_password, vault_pass_client.sh
│
├── inventory/
│   └── aws_ec2.yml                 # Dynamic inventory plugin
│
├── group_vars/
│   ├── all/
│   │   ├── vars.yml                # Non-secret: dns_domain, ntp_servers, log_level
│   │   └── vault.yml               # Ansible-vault encrypted: bootstrap secrets
│   ├── production/
│   │   ├── vars.yml                # Non-secret: instance sizes, replica counts
│   │   └── vault.yml               # Encrypted: vault_db_bootstrap_password, vault_tls_key
│   └── staging/
│       ├── vars.yml
│       └── vault.yml
│
├── roles/
│   └── ...
├── playbooks/
│   └── ...
└── Jenkinsfile
```

### Where the Ansible Vault Password Lives

```
                    ┌─────────────────────────────┐
                    │     HashiCorp Vault          │
                    │                              │
                    │  secret/novamart/ansible/    │
                    │    vault_password: "xxxx"    │
                    │    ssh_key: "-----BEGIN..."  │
                    │                              │
                    └──────────┬──────────────────┘
                               │
                    ┌──────────┴──────────────────┐
                    │                              │
              ┌─────▼─────┐                 ┌──────▼──────┐
              │  Jenkins   │                │    AWX       │
              │            │                │              │
              │ vault_pass │                │ Credential   │
              │ _client.sh │                │ Type: Vault  │
              │ uses AppRole│               │ uses AppRole │
              └────────────┘                └─────────────┘

The Ansible Vault password NEVER:
  ✗ Exists as a file on disk (except briefly in tmpfs during execution)
  ✗ Is committed to the repository
  ✗ Is hardcoded in ansible.cfg
  ✗ Is passed as a command-line argument (visible in ps aux)
```

### vault_pass_client.sh — The Script Pattern

```bash
#!/bin/bash
# vault_pass_client.sh
# This script is called by Ansible when it needs to decrypt vault-encrypted files.
# It retrieves the Ansible Vault password from HashiCorp Vault.
#
# NOTE: This file is in .gitignore. It's deployed to CI runners and operator
# workstations via a bootstrap playbook, NOT stored in the repo.

set -euo pipefail

# Method 1: HashiCorp Vault (production — Jenkins and AWX)
if command -v vault &>/dev/null && [ -n "${VAULT_ADDR:-}" ]; then
    vault kv get -field=ansible_vault_password \
        secret/novamart/ansible/vault_pass
    exit 0
fi

# Method 2: macOS Keychain (developer laptops)
if [[ "$(uname)" == "Darwin" ]]; then
    security find-generic-password -a "ansible-vault" -s "novamart" -w
    exit 0
fi

# Method 3: Pass (Linux developer workstations)
if command -v pass &>/dev/null; then
    pass show novamart/ansible-vault-password
    exit 0
fi

echo "ERROR: No vault password source available" >&2
exit 1
```

### ansible.cfg Vault Configuration

```ini
[defaults]
vault_password_file = ./vault_pass_client.sh
# Or for multiple vault IDs:
vault_identity_list = production@vault_pass_client.sh, staging@vault_pass_client.sh

# CRITICAL: Prevent accidental secret display
display_args_to_stdout = False
no_target_syslog = True
```

### How Jenkins Retrieves the Vault Password

```groovy
// Jenkinsfile
pipeline {
    agent { label 'infra-runner' }

    environment {
        // Jenkins injects HashiCorp Vault credentials via the Vault plugin
        VAULT_ADDR  = 'https://vault.novamart.internal:8200'
        VAULT_TOKEN = credentials('jenkins-vault-approle-token')
        // This uses AppRole auth — Jenkins has a role_id + secret_id
        // that grants read-only access to secret/novamart/ansible/*
    }

    stages {
        stage('Run Playbook') {
            steps {
                sh '''
                    # vault_pass_client.sh uses VAULT_ADDR and VAULT_TOKEN
                    # from the environment to authenticate
                    ansible-playbook playbooks/deploy.yml \
                        -i inventory/aws_ec2.yml
                    # ansible.cfg points vault_password_file to the script
                    # The script calls HashiCorp Vault, retrieves the password,
                    # prints it to stdout, Ansible reads it. Done.
                '''
            }
        }
    }
}
```

### How AWX Retrieves the Vault Password

```
AWX Credential Types:
  1. "Vault" credential type (built-in)
     - Stores the Ansible Vault password
     - AWX injects it at runtime via --vault-password-file <tmpfile>
     - The tmpfile is on a tmpfs mount, deleted after play completion

  2. "HashiCorp Vault Secret Lookup" credential type
     - AWX calls HashiCorp Vault directly
     - Uses AppRole auth: role_id stored in AWX, secret_id rotated automatically
     - Retrieved secrets injected as extra_vars or environment variables

AWX Configuration:
  Settings → Credentials → Add:
    Name: NovaMart Ansible Vault Password
    Type: Vault
    Vault Password: (retrieved from HashiCorp Vault, pasted once)
    Vault Identifier: production
  
  Job Template → Credentials:
    Add: "NovaMart Ansible Vault Password"
    Add: "AWS DR Deployer" (Machine credential with SSH key)
    Add: "HashiCorp Vault Lookup" (for runtime secrets)
```

### no_log Usage — Preventing Secret Leakage in Output

```yaml
# Any task that handles secrets MUST use no_log: true
- name: Set database password
  ansible.builtin.template:
    src: db_config.j2
    dest: /etc/novamart/database.yml
    mode: '0600'
  no_log: true
  # Without no_log, Ansible prints the FULL template contents
  # including the password in the task diff output

# For debug tasks during development, use a conditional
- name: Debug database connection (non-production only)
  ansible.builtin.debug:
    msg: "DB host: {{ db_host }}, DB user: {{ db_user }}"
    # NEVER: msg: "DB password: {{ db_password }}"
  when: env != 'production'
  no_log: "{{ env == 'production' }}"

# Callback plugin to catch accidental leaks:
# ansible.cfg:
# [defaults]
# callback_whitelist = ansible.posix.profile_tasks
# 
# And in CI, pipe output through a scrubber:
# ansible-playbook site.yml 2>&1 | sed 's/SuperSecret123/REDACTED/g'
# (belt-and-suspenders — no_log should already prevent this)
```

### Bootstrap Secrets vs Runtime App Secrets

```
BOOTSTRAP SECRETS (in Ansible Vault):
  Purpose: Secrets Ansible needs to CONFIGURE infrastructure
  Examples:
    - TLS certificate private keys (deployed to Vault servers)
    - Initial database admin password (used to create app users)
    - LDAP bind password (for configuring sssd)
    - HashiCorp Vault unseal keys (if auto-unseal isn't configured)
  
  Lifecycle:
    - Created once during initial setup
    - Rotated infrequently (quarterly or on personnel change)
    - Encrypted in the repo with ansible-vault
    - Decrypted at runtime by the vault_pass_client.sh script
  
  Who manages: Platform/SRE team

RUNTIME APP SECRETS (in HashiCorp Vault):
  Purpose: Secrets the APPLICATION reads at runtime
  Examples:
    - Database connection strings
    - API keys for third-party services (Stripe, Twilio)
    - JWT signing keys
    - S3 access credentials
  
  Lifecycle:
    - Created by Terraform or Ansible during provisioning
    - Rotated frequently (daily for DB passwords via Vault dynamic secrets)
    - Never stored in Git — only in HashiCorp Vault
    - Application reads directly from Vault at startup (or via sidecar)
  
  Who manages: Vault auto-rotation + application team

The handoff:
  Ansible uses BOOTSTRAP secrets to configure Vault server
  → Vault server generates RUNTIME secrets for applications
  → Applications read from Vault, never from Ansible vars
```

### Rotation When a Platform Engineer Leaves

```
IMMEDIATE (within 1 hour of departure):
  1. Revoke the engineer's HashiCorp Vault token and all associated leases
     vault token revoke -accessor <engineer_accessor>
  
  2. Disable their LDAP/SSO account (blocks Vault login)
     # This is an IAM/IdP action, not Ansible
  
  3. Remove their SSH key from all bastion authorized_keys
     # Run the ssh_keys role with their key removed from the vars
     ansible-playbook playbooks/ssh_keys.yml -i inventory/aws_ec2.yml

  4. Rotate the Ansible Vault password:
     # Generate new password
     NEW_PASS=$(openssl rand -base64 32)
     
     # Re-encrypt all vault files with the new password
     # First, decrypt with old password, then encrypt with new
     ansible-vault rekey group_vars/production/vault.yml \
         --vault-password-file old_vault_pass.sh \
         --new-vault-password-file <(echo "$NEW_PASS")
     
     # Repeat for all vault.yml files
     find . -name "vault.yml" -exec \
         ansible-vault rekey {} \
         --vault-password-file old_vault_pass.sh \
         --new-vault-password-file <(echo "$NEW_PASS") \;
     
     # Store new password in HashiCorp Vault
     vault kv put secret/novamart/ansible/vault_pass \
         ansible_vault_password="$NEW_PASS"
     
     # Update AWX credential with new password
     # AWX API or UI: Credentials → NovaMart Ansible Vault Password → update
     
     # Commit rekeyed files
     git add -A
     git commit -m "security: rekey vault files - personnel change"
     git push

WITHIN 24 HOURS:
  5. Rotate all bootstrap secrets the engineer had access to:
     - RDS master passwords
     - TLS private keys (regenerate certs)
     - Any API keys stored in vault.yml
  
  6. Rotate Jenkins/AWX service account credentials:
     - If the engineer configured the AppRole secret_id, regenerate it
     - Update Jenkins credential store and AWX credential store

  7. Audit HashiCorp Vault audit log for the engineer's recent access:
     vault audit list
     # Check what secrets they read in the last 90 days
     # Rotate any secret they accessed

WITHIN 1 WEEK:
  8. Review and rotate any shared credentials (WiFi, third-party SaaS logins)
  9. Post-mortem: document what was rotated and any gaps
```

### Failure Mode: What If .vault_pass Gets Committed

```
If the vault_pass_client.sh file is committed (it shouldn't be — it's in .gitignore):
  Impact: LOW — the script itself doesn't contain secrets.
  It's a SCRIPT that retrieves secrets from HashiCorp Vault.
  Without a valid VAULT_TOKEN, the script returns nothing.
  
If a .vault_pass FILE (containing the actual password) is committed:
  Impact: CRITICAL — anyone with repo access can decrypt ALL vault files.
  
  Response (same as Q1 incident response):
    T+0: Rotate the Ansible Vault password (rekey ALL vault files)
    T+1: Scrub the file from Git history (git-filter-repo)
    T+2: Force push, notify team to re-clone
    T+3: Rotate all secrets that were protected by the vault password
          (because an attacker could have decrypted the vault files
           using the committed password and read every secret)
  
  Prevention:
    - .gitignore with .vault_pass* pattern
    - Pre-commit hook (gitleaks) that catches high-entropy strings
    - Use vault_pass_client.sh SCRIPT, never a static FILE
    - The script pattern means there IS no file to accidentally commit
```

## Set 2, Q2: Performance Tuning a Slow Ansible Estate (continued)

### Every Bottleneck and Its Fix

**1. `forks = 10` — too few for 120 hosts**

```
Problem: 120 hosts ÷ 10 forks = 12 sequential rounds per task.
Fix: forks = 30-50 (depends on control node RAM — ~100MB per fork).

  forks = 10:  120 ÷ 10 = 12 rounds per task
  forks = 40:  120 ÷ 40 = 3 rounds per task
  
  Speedup: ~4x on parallel tasks
  
  Constraint: Control node needs ~4GB RAM for 40 forks.
  CI runners (t3.large = 8GB) can handle 50 forks comfortably.
```

**2. `serial: 1` — the parallelism killer**

```
Problem: serial: 1 processes ONE host at a time regardless of forks setting.
  120 hosts × sequential = all tasks run 120 times in series.
  This is the SINGLE BIGGEST bottleneck. Nothing else matters until this is fixed.

Fix: Remove serial entirely for patching playbooks.
  Patching is not a rolling deployment — all hosts should be patched in parallel.
  
  If you need rolling behavior (e.g., restarting a service behind an LB):
    serial: "25%"  → 4 batches of 30
  
  For pure patching with no service restart:
    Remove serial entirely → forks controls parallelism

  Speedup: 120x (from serial:1) down to ceil(120/forks) rounds
```

**3. `gather_facts: true` on 5 plays**

```
Problem: 5 plays × 120 hosts × ~3 seconds per gather = 30 minutes just on facts.
  Even with forks=40: 5 plays × 3 rounds × 3 seconds = 45 seconds.
  But with serial:1: 5 plays × 120 hosts × 3 seconds = 1800 seconds = 30 minutes.

Fix — three layers:
  Layer 1: Disable where not needed
    - name: Patch servers
      hosts: all
      gather_facts: false   # We don't use ansible_os_family here
  
  Layer 2: Gather once with caching
    # ansible.cfg
    gathering = smart
    fact_caching = jsonfile
    fact_caching_connection = /tmp/ansible_fact_cache
    fact_caching_timeout = 3600
    
    # First play gathers and caches. Subsequent plays read from cache.
    # 120 hosts × 3s = 360s once, then 0s for remaining 4 plays.
    # With forks=40: 3 rounds × 3s = 9s once.

  Layer 3: Gather minimal subsets when facts ARE needed
    - name: Gather only OS family
      ansible.builtin.setup:
        gather_subset:
          - '!all'
          # min subset auto-included: gives ansible_os_family, ansible_distribution

  Speedup: From ~30 minutes (serial:1) to ~9 seconds (parallel + cached)
```

**4. No fact caching**

```
Problem: Every playbook invocation re-gathers facts from all 120 hosts.
  If the patching playbook calls 3 child playbooks via include, that's 3 gathers.
  If a CI pipeline runs the playbook then a verification playbook, that's 2 gathers.

Fix: Already covered above — jsonfile caching with 1-hour TTL.
  
  Alternative for large fleets: Redis fact caching
    # ansible.cfg
    fact_caching = redis
    fact_caching_connection = redis.novamart.internal:6379:0
    fact_caching_timeout = 3600
    
    # Advantage: shared across multiple control nodes / CI runners
    # Multiple engineers running playbooks share the same cache

  Speedup: Eliminates redundant gathers across multiple playbook runs
```

**5. `strategy: linear` (default)**

```
Problem: Linear strategy waits for ALL hosts to finish task N before 
  ANY host starts task N+1.
  
  If 39 hosts finish task 3 in 2 seconds but 1 host takes 30 seconds
  (slow network, loaded disk), all 39 sit idle for 28 seconds.
  
  With 20 tasks: 20 × 28s wasted = 9+ minutes of idle time.

Fix: strategy: free for independent configuration tasks
  - name: Patch all servers
    hosts: all
    strategy: free    # Each host races ahead independently
    gather_facts: false
    tasks: [...]
  
  Tradeoff:
    ✓ Faster — no host waits for the slowest
    ✗ Output is interleaved (harder to read)
    ✗ Can't use serial with free (serial implies ordering)
    ✗ Can't rely on cross-host task ordering
  
  When NOT to use free:
    - Rolling restarts behind a load balancer (need serial)
    - Tasks that depend on another host completing first (orchestration)
  
  Speedup: 10-40% depending on host performance variance
```

**6. Package installs are long-running**

```
Problem: yum update or apt upgrade can take 60-120 seconds per host.
  With linear strategy: all hosts blocked until the slowest finishes.
  With 120 hosts and forks=40: 3 rounds × 120s = 360s just for patching.

Fix: async + poll pattern for long-running tasks
  - name: Apply all security patches
    ansible.builtin.yum:
      name: '*'
      state: latest
      security: true
    async: 600          # Allow up to 10 minutes
    poll: 0             # Don't wait — fire and move on
    register: patch_job

  - name: Wait for patching to complete on all hosts
    ansible.builtin.async_status:
      jid: "{{ patch_job.ansible_job_id }}"
    register: patch_result
    until: patch_result.finished
    retries: 60
    delay: 10
  
  What this does:
    1. Task 1 fires yum update on ALL 120 hosts simultaneously (within forks limit)
    2. Task 1 returns immediately (poll: 0)
    3. Ansible moves to task 2 on all hosts
    4. Task 2 polls each host until yum finishes
    
    All 120 hosts patch IN PARALLEL, even within a single round.
    Total time ≈ slowest single host, not sum of all hosts.

  Speedup: From 3 rounds × 120s = 360s to ~120s (one slow host)
```

**7. Loops over packages instead of passing lists**

```
Problem:
  # BAD: 10 packages = 10 separate SSH round-trips × 10 yum transactions
  - name: Install required packages
    ansible.builtin.yum:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - python3
      - htop
      - jq
      - curl
      - wget
      - vim
      - tree
      - git
      - tmux
  # Each iteration: SSH → Python → yum check → yum install → return
  # 10 iterations × 120 hosts ÷ 40 forks = 30 rounds × ~5s = 150 seconds

Fix:
  # GOOD: 1 SSH round-trip, 1 yum transaction for all packages
  - name: Install required packages
    ansible.builtin.yum:
      name:
        - nginx
        - python3
        - htop
        - jq
        - curl
        - wget
        - vim
        - tree
        - git
        - tmux
      state: present
  # 1 iteration: SSH → Python → yum check all → yum install missing → return
  # 1 round × 120 hosts ÷ 40 forks = 3 rounds × ~8s = 24 seconds

  The yum/apt modules accept a LIST natively. This is a single transaction.
  
  Speedup: ~6x for package installation tasks (10 round-trips → 1)
```

**8. Dynamic inventory calls AWS every run with no inventory cache**

```
Problem: Every ansible-playbook invocation calls ec2:DescribeInstances 
  across all regions. This takes 3-8 seconds and counts against API rate limits.
  
  If the pipeline runs 5 playbooks sequentially: 5 × 5s = 25s wasted on inventory.
  If 10 engineers are running playbooks simultaneously: API throttling risk.

Fix: Enable plugin-level inventory caching
  # inventory/aws_ec2.yml
  plugin: amazon.aws.aws_ec2
  regions:
    - us-east-1
    - us-west-2
    - eu-west-1
  
  cache: true
  cache_plugin: jsonfile
  cache_connection: /tmp/aws_inventory_cache
  cache_timeout: 300    # 5 minutes
  
  # First run: calls AWS API, caches result
  # Subsequent runs within 5 minutes: reads from cache instantly
  # Force refresh: ansible-inventory --flush-cache

  Speedup: Eliminates 3-8s startup delay on cached runs
```

**9. Bastion SSH with no ControlPersist**

```
Problem: Without ControlPersist, every task on every host opens a NEW SSH 
  connection through the bastion. SSH handshake through a bastion:
  
  Control → Bastion: TCP handshake + SSH handshake (~200ms)
  Bastion → Target: TCP handshake + SSH handshake (~100ms)
  Total: ~300ms per connection
  
  20 tasks × 120 hosts = 2400 connections × 300ms = 720 seconds = 12 minutes
  Just on SSH connection overhead.

Fix in ansible.cfg:
  [ssh_connection]
  ssh_args = -o ProxyCommand="ssh -W %h:%p -q -i ~/.ssh/bastion.pem ec2-user@bastion.novamart.internal" -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
  control_path_dir = ~/.ansible/cp
  control_path = %(directory)s/%%h-%%r
  pipelining = True
  
  ControlMaster=auto: First SSH to a host creates a master socket
  ControlPersist=60s: Socket stays open for 60 seconds after last use
  
  Result: First task to each host: ~300ms (full handshake)
          Tasks 2-20 to each host: ~5ms (reuse master socket)
  
  20 tasks × 120 hosts: 120 full connections + 2280 reused = 
    120 × 300ms + 2280 × 5ms = 36s + 11.4s = ~48 seconds
  
  Speedup: From ~12 minutes to ~48 seconds (15x improvement)
```

### Which Fixes Give the Largest Gains

```
RANK  FIX                        TIME SAVED       EFFORT
─────────────────────────────────────────────────────────
#1    Remove serial: 1           ~50 minutes       1 line change
#2    ControlPersist + pipelining ~11 minutes       ansible.cfg edit
#3    Increase forks to 40       ~8 minutes        ansible.cfg edit
#4    Fact caching / disable     ~8 minutes        ansible.cfg + play edits
#5    Package list (not loop)    ~3 minutes        Task refactor
#6    strategy: free             ~2-5 minutes      1 line per play
#7    async for package installs ~2-3 minutes      Task refactor
#8    Inventory caching          ~30 seconds       Inventory file edit

Total saved: ~70 minutes → we get to the target runtime
```

### What I Would NOT Optimize Yet

```
1. Mitogen (Ansible strategy plugin replacement)
   - Replaces Ansible's SSH + module execution with a custom protocol
   - 3-7x speedup on task execution
   - WHY NOT YET: Adds a dependency, not all modules are compatible,
     debugging becomes harder. Only if the above fixes aren't enough.

2. Pull mode (ansible-pull)
   - Hosts pull their own config from Git on a cron schedule
   - Eliminates SSH bottleneck entirely
   - WHY NOT YET: Different operational model. Requires git on all hosts,
     changes debugging workflow. Major architecture change for a patching playbook.

3. Custom connection plugins
   - WHY NOT YET: ControlPersist + pipelining gets 95% of the benefit.

Rule: Fix configuration before changing architecture.
```

### Realistic Post-Fix Runtime Estimate

```
BEFORE: 70 minutes
  serial: 1 → everything sequential
  No caching → redundant fact gathering and API calls
  No pipelining → 5 SSH round-trips per task per host
  Loops → 10x task count for package installs

AFTER (with all fixes applied):
  Parallel execution: forks=40, no serial (for patching)
  Fact gathering: cached, gathered once in ~9 seconds
  SSH: pipelined + ControlPersist, ~5ms per task after first
  Package installs: single transaction + async
  Strategy: free (hosts race ahead)
  Inventory: cached for 5 minutes

  Estimated breakdown:
    Inventory resolution:    0s (cached)
    Fact gathering:          9s (3 rounds × 3s, cached for remaining plays)
    Package installation:    ~90s (async, limited by slowest host)
    Configuration tasks:     ~60s (20 tasks × 3 rounds × ~1s with pipelining)
    Service restarts:        ~30s (using serial: "25%" for rolling restart only)
    Verification:            ~20s
    ─────────────────────────────
    Total:                   ~3.5 minutes

  Speedup: 70 min → 3.5 min = 20x improvement
```

### Corrected ansible.cfg

```ini
[defaults]
# Parallelism
forks = 40
strategy = free

# Fact management
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_fact_cache
fact_caching_timeout = 3600

# Output
stdout_callback = yaml
display_args_to_stdout = False
retry_files_enabled = True
retry_files_save_path = /tmp/ansible_retries

# Security
no_log = False
no_target_syslog = True
host_key_checking = False

# Performance
internal_poll_interval = 0.001

[ssh_connection]
# Pipelining: 1 SSH round-trip per task instead of 5
pipelining = True

# Connection multiplexing through bastion
ssh_args = -o ProxyCommand="ssh -W %h:%p -q -i ~/.ssh/bastion.pem ec2-user@bastion.novamart.internal" -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no

# Connection reuse paths
control_path_dir = ~/.ansible/cp
control_path = %(directory)s/%%C

# Transfer method
transfer_method = smart

[privilege_escalation]
become = True
become_method = sudo
become_user = root
```

### Corrected Play Example

```yaml
---
- name: Security patching — all hosts
  hosts: all
  become: true
  strategy: free
  gather_facts: false

  pre_tasks:
    # Gather minimal facts, cached for subsequent plays
    - name: Gather OS facts only
      ansible.builtin.setup:
        gather_subset:
          - '!all'

    - name: Notify patching start
      ansible.builtin.uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "🔒 Security patching starting across {{ ansible_play_hosts | length }} hosts"
      delegate_to: localhost
      run_once: true

  tasks:
    # Single transaction for all prerequisite packages (not a loop)
    - name: Install prerequisite packages
      ansible.builtin.yum:
        name:
          - yum-utils
          - python3
          - aide
        state: present
      when: ansible_os_family == "RedHat"

    - name: Install prerequisite packages (Debian)
      ansible.builtin.apt:
        name:
          - apt-utils
          - python3
          - aide
        state: present
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

    # Async patching — fire on all hosts, then wait
    - name: Apply security patches (RedHat family)
      ansible.builtin.yum:
        name: '*'
        state: latest
        security: true
      async: 600
      poll: 0
      register: patch_job_rh
      when: ansible_os_family == "RedHat"

    - name: Apply security patches (Debian family)
      ansible.builtin.apt:
        upgrade: safe
        update_cache: true
      async: 600
      poll: 0
      register: patch_job_deb
      when: ansible_os_family == "Debian"

    - name: Wait for RedHat patching to complete
      ansible.builtin.async_status:
        jid: "{{ patch_job_rh.ansible_job_id }}"
      register: rh_result
      until: rh_result.finished
      retries: 60
      delay: 10
      when: patch_job_rh is not skipped

    - name: Wait for Debian patching to complete
      ansible.builtin.async_status:
        jid: "{{ patch_job_deb.ansible_job_id }}"
      register: deb_result
      until: deb_result.finished
      retries: 60
      delay: 10
      when: patch_job_deb is not skipped

    # Check if reboot is required
    - name: Check if reboot needed (RedHat)
      ansible.builtin.command:
        cmd: needs-restarting -r
      register: reboot_needed_rh
      changed_when: false
      failed_when: false
      when: ansible_os_family == "RedHat"

    - name: Check if reboot needed (Debian)
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_needed_deb
      when: ansible_os_family == "Debian"

    - name: Set reboot fact
      ansible.builtin.set_fact:
        needs_reboot: >-
          {{ (reboot_needed_rh.rc | default(0) == 1) or 
             (reboot_needed_deb.stat.exists | default(false)) }}

  post_tasks:
    - name: Report patching results
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: patching complete. Reboot needed: {{ needs_reboot }}"

# Separate play for reboots — this one uses serial for safety
- name: Rolling reboot of hosts that need it
  hosts: all
  become: true
  serial: "20%"
  max_fail_percentage: 10
  gather_facts: false

  tasks:
    - name: Reboot if needed
      ansible.builtin.reboot:
        reboot_timeout: 300
        msg: "Rebooting for kernel/security patches"
      when: needs_reboot | default(false) | bool

    - name: Wait for host to be responsive
      ansible.builtin.wait_for_connection:
        delay: 10
        timeout: 120
      when: needs_reboot | default(false) | bool

    - name: Verify services are running post-reboot
      ansible.builtin.systemd:
        name: "{{ item }}"
        state: started
      loop:
        - sshd
        - node_exporter
      when: needs_reboot | default(false) | bool
```

---

## Set 2, Q3: Molecule Test Strategy for vault_server Role

### Directory Structure

```
roles/vault_server/
├── defaults/
│   └── main.yml              # Default variables (Vault version, ports, paths)
├── handlers/
│   └── main.yml              # restart vault, reload systemd
├── tasks/
│   ├── main.yml              # Entrypoint — includes OS-specific tasks
│   ├── install.yml           # Download + install Vault binary
│   ├── configure.yml         # TLS certs, vault.hcl, systemd unit
│   ├── firewall_redhat.yml   # firewalld rules (RHEL/AL2)
│   └── firewall_debian.yml   # ufw rules (Ubuntu)
├── templates/
│   ├── vault.hcl.j2          # Vault server configuration
│   ├── vault.service.j2      # systemd unit file
│   └── vault-tls.conf.j2     # TLS configuration
├── files/
│   └── vault_health_check.sh # Health check script
├── meta/
│   └── main.yml
├── molecule/
│   ├── default/              # Docker-based: fast, runs in every PR
│   │   ├── molecule.yml
│   │   ├── converge.yml
│   │   ├── prepare.yml
│   │   └── verify.yml
│   ├── multi-os/             # Docker-based: tests both OS families
│   │   ├── molecule.yml
│   │   ├── converge.yml
│   │   └── verify.yml
│   └── ec2/                  # EC2-backed: full integration, runs nightly
│       ├── molecule.yml
│       ├── converge.yml
│       ├── prepare.yml
│       └── verify.yml
└── README.md
```

### Docker vs EC2 Tradeoffs

```
                         Docker                    EC2
                         ──────                    ───
Speed                    30-90 seconds             5-10 minutes
Cost                     Free (runs on CI runner)  ~$0.02 per test run
systemd testing          Fake (privileged + init)  Real systemd
Firewall testing         Cannot validate           Real firewalld/ufw
TLS/networking           Localhost only             Real VPC networking
Kernel parameters        Shares host kernel         Isolated kernel
SELinux                  Disabled/fake              Real enforcement
Cloud-init               Not available              Real cloud-init
Instance profiles/IMDS   Not available              Real metadata service
AMI compatibility        Base Docker image          Actual production AMI
Idempotency testing      ✓ Full support             ✓ Full support
Template rendering       ✓ Full support             ✓ Full support
OS family branching      ✓ Full support             ✓ Full support

Decision matrix:
  PR pipeline  → Docker (fast feedback, logic validation)
  Nightly CI   → EC2 (full integration, catches systemd/firewall/TLS issues)
  Pre-release  → EC2 against production AMI (final proof)
```

### molecule/default/molecule.yml (Docker — Fast)

```yaml
---
dependency:
  name: galaxy
  requirements-file: requirements.yml

driver:
  name: docker

platforms:
  - name: vault-ubuntu2204
    image: ubuntu:22.04
    command: /sbin/init
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    privileged: true
    pre_build_image: true
    groups:
      - vault_servers

  - name: vault-amazonlinux2
    image: amazonlinux:2
    command: /sbin/init
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    privileged: true
    pre_build_image: true
    groups:
      - vault_servers

provisioner:
  name: ansible
  playbooks:
    converge: converge.yml
    prepare: prepare.yml
    verify: verify.yml
  inventory:
    group_vars:
      vault_servers:
        vault_version: "1.15.4"
        vault_tls_disable: true  # TLS testing requires real certs; skip in Docker
        vault_ui: true
        vault_api_port: 8200
        vault_cluster_port: 8201

verifier:
  name: ansible

scenario:
  name: default
  test_sequence:
    - dependency
    - lint
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - verify
    - cleanup
    - destroy
```

### molecule/default/prepare.yml

```yaml
---
- name: Prepare containers for Vault installation
  hosts: all
  become: true
  gather_facts: true
  tasks:
    # Docker containers are minimal — install basics
    - name: Install prerequisites (RedHat)
      ansible.builtin.yum:
        name:
          - systemd
          - python3
          - unzip
          - sudo
          - iproute
        state: present
      when: ansible_os_family == "RedHat"

    - name: Install prerequisites (Debian)
      ansible.builtin.apt:
        name:
          - systemd
          - python3
          - unzip
          - sudo
          - iproute2
          - gpg
        state: present
        update_cache: true
      when: ansible_os_family == "Debian"

    # Create vault user (the role expects it or creates it)
    - name: Ensure vault group exists
      ansible.builtin.group:
        name: vault
        system: true

    - name: Ensure vault user exists
      ansible.builtin.user:
        name: vault
        group: vault
        system: true
        shell: /usr/sbin/nologin
        create_home: false
```

### molecule/default/converge.yml

```yaml
---
- name: Converge
  hosts: vault_servers
  become: true
  roles:
    - role: vault_server
```

### molecule/default/verify.yml — What to Verify

```yaml
---
- name: Verify Vault server installation
  hosts: vault_servers
  become: true
  gather_facts: false
  tasks:
    # === BINARY INSTALLATION ===
    - name: Verify Vault binary exists
      ansible.builtin.stat:
        path: /usr/local/bin/vault
      register: vault_binary
      failed_when: not vault_binary.stat.exists

    - name: Verify Vault binary is executable
      ansible.builtin.stat:
        path: /usr/local/bin/vault
      register: vault_perms
      failed_when: not vault_perms.stat.executable

    - name: Verify Vault version matches expected
      ansible.builtin.command:
        cmd: /usr/local/bin/vault version
      register: vault_ver
      changed_when: false
      failed_when: "'1.15.4' not in vault_ver.stdout"

    # === SYSTEMD CONFIGURATION ===
    - name: Verify systemd unit file exists
      ansible.builtin.stat:
        path: /etc/systemd/system/vault.service
      register: unit_file
      failed_when: not unit_file.stat.exists

    - name: Verify unit file permissions (0644, root:root)
      ansible.builtin.stat:
        path: /etc/systemd/system/vault.service
      register: unit_perms
      failed_when: >
        unit_perms.stat.mode != '0644' or
        unit_perms.stat.pw_name != 'root'

    - name: Verify systemd recognizes vault service
      ansible.builtin.command:
        cmd: systemctl list-unit-files vault.service
      register: systemd_list
      changed_when: false
      failed_when: "'vault.service' not in systemd_list.stdout"

    # === CONFIGURATION FILES ===
    - name: Verify vault.hcl exists with correct permissions
      ansible.builtin.stat:
        path: /etc/vault.d/vault.hcl
      register: vault_config
      failed_when: >
        not vault_config.stat.exists or
        vault_config.stat.mode != '0640' or
        vault_config.stat.pw_name != 'vault'

    - name: Verify vault.hcl contains expected listener config
      ansible.builtin.command:
        cmd: grep -c 'api_addr' /etc/vault.d/vault.hcl
      register: config_check
      changed_when: false
      failed_when: config_check.stdout | int < 1

    # === DIRECTORY STRUCTURE ===
    - name: Verify Vault data directory
      ansible.builtin.stat:
        path: /opt/vault/data
      register: data_dir
      failed_when: >
        not data_dir.stat.exists or
        not data_dir.stat.isdir or
        data_dir.stat.pw_name != 'vault' or
        data_dir.stat.mode != '0750'

    # === SERVICE STATE (Docker caveat: may not fully work) ===
    - name: Check if Vault service is enabled
      ansible.builtin.command:
        cmd: systemctl is-enabled vault
      register: vault_enabled
      changed_when: false
      failed_when: vault_enabled.stdout != 'enabled'

    # === PROCESS AND PORT (if service started successfully in Docker) ===
    - name: Check if Vault API port is configured
      ansible.builtin.command:
        cmd: grep -c '8200' /etc/vault.d/vault.hcl
      register: port_check
      changed_when: false
      failed_when: port_check.stdout | int < 1

    # === HEALTH CHECK SCRIPT ===
    - name: Verify health check script is deployed
      ansible.builtin.stat:
        path: /usr/local/bin/vault_health_check.sh
      register: health_script
      failed_when: >
        not health_script.stat.exists or
        not health_script.stat.executable
```

### What Idempotence Must Prove

```
The idempotence step in the test_sequence runs converge.yml a SECOND time.

It MUST show:
  changed=0

This proves:
  1. Binary installation: doesn't re-download if already present
  2. User/group creation: doesn't recreate existing users
  3. Configuration templates: identical content → no change reported
  4. Systemd unit: identical file → no change → handler NOT triggered
  5. Directory creation: existing dirs with correct perms → "ok"
  6. Firewall rules: existing rules → no modification
  7. Service state: already started + enabled → "ok"

If idempotence shows changed > 0, it means:
  - A task is using command/shell without changed_when
  - A handler fires unnecessarily (template always reports changed)
  - A task doesn't check current state before acting
  
  This MUST be fixed — a non-idempotent role causes:
    - Unnecessary service restarts on every Ansible run
    - False "changed" counts that mask real drift
    - Handlers that fire when they shouldn't (Vault restart = brief outage)
```

### molecule/ec2/molecule.yml — Full Integration

```yaml
---
dependency:
  name: galaxy
  requirements-file: requirements.yml

driver:
  name: ec2

platforms:
  - name: mol-vault-ubuntu
    image: ami-0abcdef1234567890  # Ubuntu 22.04 production AMI
    instance_type: t3.small
    vpc_subnet_id: subnet-xxxxx
    security_group_name: molecule-vault-testing
    key_name: molecule-test-key
    assign_public_ip: true
    tags:
      ManagedBy: molecule
      Purpose: testing
      Role: vault_server
    groups:
      - vault_servers

  - name: mol-vault-al2
    image: ami-0fedcba9876543210  # Amazon Linux 2 production AMI
    instance_type: t3.small
    vpc_subnet_id: subnet-xxxxx
    security_group_name: molecule-vault-testing
    key_name: molecule-test-key
    assign_public_ip: true
    tags:
      ManagedBy: molecule
      Purpose: testing
      Role: vault_server
    groups:
      - vault_servers

provisioner:
  name: ansible
  connection_options:
    ansible_user: ec2-user  # Overridden per-platform if needed
  playbooks:
    converge: converge.yml
    prepare: prepare.yml
    verify: verify.yml
  inventory:
    group_vars:
      vault_servers:
        vault_version: "1.15.4"
        vault_tls_disable: false   # EC2: test with REAL TLS
        vault_tls_cert_file: /etc/vault.d/tls/vault.crt
        vault_tls_key_file: /etc/vault.d/tls/vault.key

verifier:
  name: ansible

scenario:
  name: ec2
  test_sequence:
    - dependency
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - verify
    - cleanup
    - destroy
```

### EC2 verify.yml — Additional Tests Docker Can't Do

```yaml
---
- name: Verify Vault server (full integration — EC2 only)
  hosts: vault_servers
  become: true
  gather_facts: false
  tasks:
    # All Docker verify checks PLUS:

    # === REAL SYSTEMD BEHAVIOR ===
    - name: Verify Vault service is running (real systemd)
      ansible.builtin.systemd:
        name: vault
      register: vault_service
      failed_when: vault_service.status.ActiveState != 'active'

    - name: Verify Vault survives a restart
      ansible.builtin.systemd:
        name: vault
        state: restarted
      register: restart_result

    - name: Wait for Vault to be responsive after restart
      ansible.builtin.uri:
        url: "https://localhost:8200/v1/sys/health"
        validate_certs: false
        status_code: [200, 429, 472, 473, 501, 503]
        # 200=initialized+unsealed, 429=standby, 472=DR secondary
        # 473=perf standby, 501=not initialized, 503=sealed
      register: health
      retries: 10
      delay: 3
      until: health.status in [200, 501, 503]

    # === REAL FIREWALL RULES ===
    - name: Verify firewall allows Vault API port (RedHat)
      ansible.builtin.command:
        cmd: firewall-cmd --list-ports
      register: fw_ports
      changed_when: false
      failed_when: "'8200/tcp' not in fw_ports.stdout"
      when: ansible_os_family == "RedHat"

    - name: Verify firewall allows Vault API port (Ubuntu)
      ansible.builtin.command:
        cmd: ufw status
      register: ufw_status
      changed_when: false
      failed_when: "'8200/tcp' not in ufw_status.stdout"
      when: ansible_os_family == "Debian"

    # === REAL TLS ===
    - name: Verify TLS certificate is valid
      ansible.builtin.command:
        cmd: openssl x509 -in /etc/vault.d/tls/vault.crt -noout -dates
      register: cert_dates
      changed_when: false

    - name: Verify Vault responds over HTTPS
      ansible.builtin.uri:
        url: "https://{{ ansible_default_ipv4.address }}:8200/v1/sys/health"
        validate_certs: false
        status_code: [200, 501, 503]
      register: tls_health

    # === REAL PROCESS ISOLATION ===
    - name: Verify Vault runs as vault user (not root)
      ansible.builtin.command:
        cmd: ps -o user= -p $(pgrep vault)
      register: vault_user
      changed_when: false
      failed_when: vault_user.stdout | trim != 'vault'

    # === REAL PORT BINDING ===
    - name: Verify Vault is listening on expected ports
      ansible.builtin.command:
        cmd: ss -tlnp
      register: listening_ports
      changed_when: false
      failed_when: >
        ':8200' not in listening_ports.stdout or
        ':8201' not in listening_ports.stdout
```

### CI: Fast Tests vs Slow Tests

```groovy
// Jenkinsfile
pipeline {
    agent { label 'infra-runner' }

    stages {
        stage('Lint') {
            steps {
                sh 'ansible-lint roles/vault_server/'
                sh 'yamllint roles/vault_server/'
            }
        }

        stage('Molecule Docker (Fast)') {
            // Runs on EVERY PR — <2 minutes
            steps {
                sh 'cd roles/vault_server && molecule test -s default'
            }
        }

        stage('Molecule EC2 (Slow)') {
            // Runs ONLY on: merge to main, nightly schedule, manual trigger
            when {
                anyOf {
                    branch 'main'
                    triggeredBy 'TimerTrigger'
                    triggeredBy cause: 'UserIdCause'
                }
            }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                 credentialsId: 'molecule-ec2-testing']]) {
                    sh 'cd roles/vault_server && molecule test -s ec2'
                }
            }
        }
    }

    post {
        failure {
            slackSend(
                channel: '#ansible-ci',
                color: 'danger',
                message: "❌ vault_server role tests failed: ${env.BUILD_URL}"
            )
        }
        always {
            // Always destroy EC2 instances even if tests fail
            sh 'cd roles/vault_server && molecule destroy -s ec2 || true'
        }
    }
}
```

### 5 Production Failure Modes Molecule WOULD Catch

```
1. WRONG FILE PERMISSIONS ON vault.hcl
   Symptom: Vault refuses to start — "permission denied reading config"
   Molecule catches: verify.yml checks mode='0640', owner=vault
   
2. MISSING SYSTEMD UNIT FILE
   Symptom: "Could not find the requested service vault"
   Molecule catches: verify.yml checks stat on /etc/systemd/system/vault.service

3. BROKEN JINJA2 TEMPLATE
   Symptom: vault.hcl has syntax errors from bad template rendering
   Molecule catches: converge fails at template task with Jinja2 error

4. NON-IDEMPOTENT HANDLER (Vault restarts every run)
   Symptom: Every Ansible run restarts Vault → brief outage
   Molecule catches: idempotence step shows changed > 0

5. OS-FAMILY BRANCHING BUG (Ubuntu task runs on Amazon Linux)
   Symptom: apt module called on RHEL → task failure
   Molecule catches: multi-os scenario runs both platforms, one would fail
```

### 3 Production Failure Modes Molecule Would MISS

```
1. FIREWALL BLOCKS VAULT PORT IN VPC SECURITY GROUP
   Why missed: Molecule (even EC2) uses a permissive test security group.
   Production uses tightly scoped SGs managed by Terraform.
   A firewalld rule opening port 8200 means nothing if the SG blocks it.
   Caught by: Smoke test in the deployment pipeline (curl Vault API from another host)

2. VAULT FAILS TO GET TLS CERT FROM ACM/LET'S ENCRYPT
   Why missed: Molecule uses self-signed test certs.
   Production uses real certs from ACM Private CA or Let's Encrypt.
   If the cert issuing process fails (IAM permission, DNS challenge), Vault starts without TLS.
   Caught by: Integration test that validates the cert chain end-to-end

3. VAULT CAN'T REACH STORAGE BACKEND (Consul/DynamoDB)
   Why missed: Molecule tests Vault with file storage backend (local).
   Production uses Consul or DynamoDB for HA storage.
   If the IAM role for DynamoDB is wrong, or Consul ACLs block Vault, startup fails.
   Caught by: EC2 Molecule test WITH a real storage backend (expensive, slow — nightly only)
```

---

## Set 2, Q4: Terraform + Ansible Integration Pattern

### Why local-exec/remote-exec Is a Trap

```
resource "aws_instance" "web" {
  ami           = "ami-xxxxx"
  instance_type = "t3.micro"

  # THE TRAP:
  provisioner "remote-exec" {
    inline = ["sudo yum install -y nginx"]
  }
}

Why this is dangerous:

1. NOT IDEMPOTENT — Terraform provisioners run only on resource CREATION.
   If you need to change nginx config later, Terraform won't re-run the provisioner.
   You must taint + destroy + recreate the instance. That's downtime.

2. NO RETRY LOGIC — If SSH fails during provisioning (cloud-init still running,
   network not ready), the provisioner fails, Terraform marks the resource as 
   TAINTED, and your next apply DESTROYS and recreates the instance.

3. NO CHANGE TRACKING — Terraform state doesn't know what the provisioner did.
   There's no diff, no plan output, no way to see "this instance has nginx 
   installed" in the state. It's a black box.

4. TIGHT COUPLING — Terraform now needs SSH access to instances.
   Your CI runner needs the SSH key, the security group must allow SSH from 
   the CI runner, and you've created a dependency between infrastructure 
   provisioning and configuration management.

5. NO PARTIAL RE-RUN — If Terraform creates 15 instances and the provisioner 
   fails on instance 12, you can't re-run JUST the provisioner. You must 
   taint instance 12, destroy it, and recreate it. Ansible can re-run on 
   just the failed host with --limit.

6. BLOCKING — Terraform waits for the provisioner to complete before moving 
   to the next resource. 15 instances with 2-minute provisioners = 30 minutes 
   of sequential blocking (unless you use count + null_resource tricks).

The correct separation:
   Terraform: Create the instance, tag it, output its ID
   Ansible: Configure the instance, independently, re-runnably
```

### Pipeline Stages

```
┌─────────┐    ┌──────────┐    ┌───────────┐    ┌─────────┐    ┌────────┐    ┌──────┐
│Terraform│───→│  Output  │───→│   Wait    │───→│ Ansible │───→│ Smoke  │───→│Notify│
│  Apply  │    │  Sync    │    │  Ready    │    │Configure│    │ Tests  │    │      │
└─────────┘    └──────────┘    └───────────┘    └─────────┘    └────────┘    └──────┘
     │              │               │                │              │
     │         TF outputs      EC2 status ok     Dynamic inv    curl + API
     │         → YAML file     SSH reachable     by tags         health checks
     │                         cloud-init done
     │
     │  On failure: TF state is clean. Instances exist.
     │  Fix Ansible, re-run from "Ansible Configure" stage.
     │  No need to re-run Terraform.
```

### Tagging Strategy

```hcl
# Terraform tags control what Ansible discovers and how it groups hosts

locals {
  # Standard tags on EVERY instance
  base_tags = {
    Project      = "novamart"
    Environment  = var.environment       # "production", "staging"
    ManagedBy    = "terraform"
    ConfiguredBy = "ansible"
    ProvisionedAt = timestamp()
  }
}

resource "aws_instance" "vault" {
  count = 3
  # ...
  tags = merge(local.base_tags, {
    Name = "${var.environment}-vault-${count.index}"
    Role = "vault_server"               # Ansible groups by this
    OS   = "ubuntu"                     # Ansible sets ansible_user from this
    AnsibleReady = "true"               # Set to "true" AFTER cloud-init
    # Initially set to "false" — a cloud-init script sets it to "true"
    # when the instance is fully bootstrapped
  })
}

# Cloud-init template that signals readiness
resource "aws_instance" "vault" {
  # ...
  user_data = <<-USERDATA
    #!/bin/bash
    # Standard cloud-init bootstrap
    yum update -y
    yum install -y python3
    
    # Signal that this instance is ready for Ansible
    INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
    aws ec2 create-tags \
      --resources $INSTANCE_ID \
      --tags Key=AnsibleReady,Value=true \
      --region ${var.region}
  USERDATA
}
```

### Ansible Dynamic Inventory Using Tags

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1

filters:
  tag:Project: novamart
  tag:Environment: production
  tag:AnsibleReady: "true"        # Only discover instances that are READY
  instance-state-name: running

cache: true
cache_plugin: jsonfile
cache_connection: /tmp/novamart_inventory_cache
cache_timeout: 120

hostnames:
  - private-ip-address

compose:
  ansible_user: "'ubuntu' if tags.OS == 'ubuntu' else 'ec2-user'"
  ansible_host: private_ip_address

keyed_groups:
  - key: tags.Role
    prefix: role
    separator: "_"
  - key: tags.Environment
    prefix: env
    separator: "_"

strict: true
```

### Output Synchronization Pattern

```bash
#!/bin/bash
# scripts/tf_to_ansible_vars.py — called by Jenkins pipeline

import json
import yaml
import sys

tf_output_file = sys.argv[1]  # /tmp/tf_outputs.json
ansible_vars_file = sys.argv[2]  # ansible/terraform_outputs.yml

with open(tf_output_file) as f:
    tf_outputs = json.load(f)

ansible_vars = {}
for key, value in tf_outputs.items():
    # Prefix with tf_ to avoid variable name collisions
    ansible_vars[f"tf_{key}"] = value["value"]

with open(ansible_vars_file, 'w') as f:
    yaml.dump(ansible_vars, f, default_flow_style=False)

print(f"Written {len(ansible_vars)} variables to {ansible_vars_file}")
```

```bash
# Pipeline usage:
terraform -chdir=terraform/prod output -json > /tmp/tf_outputs.json
python3 scripts/tf_to_ansible_vars.py /tmp/tf_outputs.json ansible/terraform_outputs.yml

# Ansible consumes:
ansible-playbook site.yml \
  -i inventory/aws_ec2.yml \
  -e @terraform_outputs.yml
```

### Waiting / Reachability Logic

```bash
#!/bin/bash
# scripts/wait_for_instances.sh
# Called AFTER terraform apply, BEFORE ansible-playbook

set -euo pipefail

REGION="${1:-us-east-1}"
ENVIRONMENT="${2:-production}"
MAX_WAIT=300  # 5 minutes
POLL_INTERVAL=10

echo "Waiting for instances to be ready..."

# Phase 1: AWS-level instance status checks
INSTANCE_IDS=$(aws ec2 describe-instances \
  --region "$REGION" \
  --filters \
    "Name=tag:Project,Values=novamart" \
    "Name=tag:Environment,Values=$ENVIRONMENT" \
    "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text)

echo "Found instances: $INSTANCE_IDS"

for id in $INSTANCE_IDS; do
  echo "Waiting for $id status checks..."
  aws ec2 wait instance-status-ok --instance-ids "$id" --region "$REGION"
  echo "✓ $id passed status checks"
done

# Phase 2: Wait for cloud-init to complete (AnsibleReady tag)
ELAPSED=0
while true; do
  NOT_READY=$(aws ec2 describe-instances \
    --region "$REGION" \
    --filters \
      "Name=tag:Project,Values=novamart" \
      "Name=tag:Environment,Values=$ENVIRONMENT" \
      "Name=instance-state-name,Values=running" \
    --query 'Reservations[].Instances[?Tags[?Key==`AnsibleReady` && Value!=`true`]].InstanceId' \
    --output text)

  if [ -z "$NOT_READY" ]; then
    echo "✓ All instances have AnsibleReady=true"
    break
  fi

  if [ "$ELAPSED" -ge "$MAX_WAIT" ]; then
    echo "ERROR: Timeout waiting for instances: $NOT_READY"
    exit 1
  fi

  echo "Waiting for AnsibleReady on: $NOT_READY ($ELAPSED/${MAX_WAIT}s)"
  sleep "$POLL_INTERVAL"
  ELAPSED=$((ELAPSED + POLL_INTERVAL))
done

# Phase 3: SSH connectivity test via bastion
BASTION_IP=$(aws ec2 describe-instances \
  --region "$REGION" \
  --filters \
    "Name=tag:Role,Values=bastion" \
    "Name=tag:Environment,Values=$ENVIRONMENT" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

PRIVATE_IPS=$(aws ec2 describe-instances \
  --region "$REGION" \
  --filters \
    "Name=tag:Project,Values=novamart" \
    "Name=tag:Environment,Values=$ENVIRONMENT" \
    "Name=tag:AnsibleReady,Values=true" \
  --query 'Reservations[].Instances[].PrivateIpAddress' \
  --output text)

for ip in $PRIVATE_IPS; do
  for attempt in $(seq 1 10); do
    if ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 \
           -o ProxyCommand="ssh -W %h:%p -q ec2-user@$BASTION_IP" \
           ec2-user@"$ip" "echo READY" 2>/dev/null | grep -q READY; then
      echo "✓ $ip SSH ready"
      break
    fi
    if [ "$attempt" -eq 10 ]; then
      echo "ERROR: Cannot reach $ip via SSH after 10 attempts"
      exit 1
    fi
    sleep 5
  done
done

echo "All instances ready for Ansible configuration."
```

### Re-Running Ansible Without Re-Running Terraform

```
This is the KEY advantage of the decoupled pattern:

  # Scenario 1: Ansible role update (new nginx config)
  # Just re-run Ansible — Terraform state unchanged
  ansible-playbook site.yml -i inventory/aws_ec2.yml -e @terraform_outputs.yml

  # Scenario 2: Fix a failure on specific hosts
  ansible-playbook site.yml -i inventory/aws_ec2.yml --limit @site.retry

  # Scenario 3: Run Ansible against a subset
  ansible-playbook site.yml -i inventory/aws_ec2.yml --limit role_vault_server

  # Scenario 4: Day-2 operations (patching, cert rotation)
  ansible-playbook playbooks/patch.yml -i inventory/aws_ec2.yml
  # Terraform is not involved at all

  # The terraform_outputs.yml file is committed to the repo after initial 
  # provisioning, so Ansible always has access to TF outputs without 
  # re-running terraform output.
```

### Rollback and Verification

```
ANSIBLE ROLLBACK:
  Option A: Git revert the playbook change, re-run
    git revert HEAD
    ansible-playbook site.yml -i inventory/aws_ec2.yml
    # Ansible converges to the PREVIOUS state (previous config files, 
    # previous package versions if pinned)
  
  Option B: Version-pinned deployments
    # Role deploys a specific version of config/binary
    # Rollback = change the version variable and re-run
    ansible-playbook site.yml -e vault_version=1.15.3  # old version
  
TERRAFORM ROLLBACK:
  git revert the Terraform change, re-apply
  Terraform destroys new resources, recreates old ones
  CAUTION: This destroys instances → Ansible must re-run on new instances

VERIFICATION:
  # After Ansible run, execute smoke tests
  ansible-playbook smoke_test.yml -i inventory/aws_ec2.yml
  
  # smoke_test.yml checks:
  # - HTTP endpoints responding
  # - Vault sealed/unsealed status
  # - Node exporter metrics endpoint
  # - Log files being written
  # - DNS resolution working
```

### What Drift Each Tool Can and Cannot Detect

```
                        Terraform               Ansible
                        ─────────               ───────
DETECTS:
  Security group rules  ✓ terraform plan        ✗ Not its domain
  Instance count        ✓ terraform plan        ✗ (sees what exists via inventory)
  RDS config            ✓ terraform plan        ✗ Not its domain
  IAM policies          ✓ terraform plan        ✗ Not its domain
  Nginx config file     ✗ Not its domain        ✓ template diff on re-run
  Installed packages    ✗ Not its domain        ✓ yum module checks state
  systemd unit state    ✗ Not its domain        ✓ systemd module checks state
  File permissions      ✗ Not its domain        ✓ file module checks state
  Kernel parameters     ✗ Not its domain        ✓ sysctl module checks state
  User accounts         ✗ Not its domain        ✓ user module checks state

CANNOT DETECT:
  Terraform:
    - Manual SSH changes to instances (someone edited nginx.conf by hand)
    - Configuration inside the OS (packages, files, services)
    - Application-level state
  
  Ansible:
    - Cloud resource changes (someone deleted a security group rule in console)
    - Infrastructure-level drift (subnet CIDR changed, RDS instance class changed)
    - IAM changes
  
  BOTH MISS:
    - Application logic bugs
    - Data corruption
    - Performance degradation without state change
  
  TO CATCH EVERYTHING:
    Terraform plan (scheduled)     → infrastructure drift
    Ansible --check --diff         → configuration drift  
    Prometheus + alerting          → runtime/performance drift
    AWS Config rules               → compliance drift
    
    All four layers running continuously = comprehensive drift detection
```

---

## Set 2, Q5: AWX vs Jenkins — Separation of Duties

### Responsibility Matrix

```
FUNCTION                          JENKINS              AWX
──────────────────────────────────────────────────────────────
CI/CD Application Delivery        ✓ PRIMARY            ✗
  Build, test, push images,       Jenkins pipelines
  deploy to K8s                   are code-driven

Scheduled Patching                ✗                    ✓ PRIMARY
  Monthly OS patches,             Ansible is the       AWX schedules,
  security updates                config tool          runs, audits

Ad-hoc Operational Tasks          ✗                    ✓ PRIMARY
  Restart a service,              AWX has survey       One-click with
  flush a cache,                  forms for operators  guardrails
  rotate a cert                   who shouldn't write
                                  playbooks

Credential Management             △ BASIC              ✓ SUPERIOR
  Jenkins: credentials store      AWX: credential      AWX credentials
  (flat, no type system)          types with           are typed, 
  Secrets are env vars            injection patterns   injected per-job,
  visible in logs if              and integration      never in logs
  not careful                     with external vaults

Inventory Sync                    ✗ MANUAL             ✓ PRIMARY
  Jenkins: you write the          AWX: built-in        Auto-syncs
  inventory fetch script          inventory sources    from AWS EC2,
  yourself                        with scheduling      GCP, Azure

RBAC                              △ WEAK               ✓ STRONG
  Jenkins: folder-level           AWX: organization    Per-inventory,
  permissions, role-based         → team → user        per-credential,
  but coarse                      granular per         per-job-template
                                  resource type

Audit Trail                       △ BUILD LOGS         ✓ PURPOSE-BUILT
  Jenkins: build log has          AWX: every job       Who ran what,
  everything but not              execution logged     on which hosts,
  structured for audit            with: who, what,     with which
                                  when, which hosts,   credentials,
                                  what changed,        full diff output
                                  approval chain

Approvals for Prod Changes        △ MANUAL GATES       ✓ WORKFLOW
  Jenkins: input step or          AWX: workflow         Approval node
  manual trigger                  templates with       → human approves
  No structured approval          approval nodes,      → next step runs
  chain                           notification on      → audit logged
                                  pending approvals
```

### The Key Insight: Why Both Exist

```
Jenkins = DEVELOPER tool
  - Triggered by code changes (git push)
  - Builds, tests, deploys applications
  - Pipeline-as-code in Jenkinsfile
  - Engineers write and maintain pipelines
  - Audience: development team

AWX = OPERATOR tool
  - Triggered by schedules or button clicks
  - Configures, patches, operates infrastructure
  - Playbooks selected from a catalog
  - Engineers write playbooks; operators RUN them via AWX UI
  - Audience: operations team, on-call engineers, NOC

Overlap zone:
  Jenkins CAN run Ansible playbooks (and does, for initial provisioning)
  AWX CAN be triggered by API (and is, for post-deploy config)
  
  Rule: If the trigger is a CODE CHANGE → Jenkins
        If the trigger is an OPERATIONAL NEED → AWX
```

### Scenario: Developer Restarts Staging Service Without Touching Production

**The requirement:** A developer needs to restart `novamart-api` on staging EC2 hosts. They must NEVER be able to touch production.

**AWX Implementation:**

```
Step 1: Create separate inventories
  ┌─────────────────────────────────────────┐
  │ AWX Inventories                          │
  │                                          │
  │ "NovaMart Staging"                       │
  │   Source: AWS EC2 dynamic                │
  │   Filter: tag:Environment = staging      │
  │   Sync: every 5 minutes                 │
  │                                          │
  │ "NovaMart Production"                    │
  │   Source: AWS EC2 dynamic                │
  │   Filter: tag:Environment = production   │
  │   Sync: every 5 minutes                 │
  └─────────────────────────────────────────┘

Step 2: Create the job template
  ┌─────────────────────────────────────────┐
  │ Job Template: "Restart Service"          │
  │                                          │
  │ Playbook: playbooks/restart_service.yml  │
  │ Inventory: (selected at launch time)     │
  │ Credentials: SSH key for staging         │
  │                                          │
  │ Survey (form presented to user):         │
  │   Service Name: [____________]           │
  │   (dropdown: novamart-api, nginx,        │
  │    node_exporter, vault)                 │
  │                                          │
  │ Limit: (optional host pattern)           │
  └─────────────────────────────────────────┘

Step 3: RBAC — this is where AWX enforces safety
  ┌─────────────────────────────────────────┐
  │ AWX Organizations & Teams               │
  │                                          │
  │ Organization: NovaMart                   │
  │   Team: Developers                       │
  │     Members: dev-alice, dev-bob          │
  │     Permissions:                         │
  │       "NovaMart Staging" inventory → USE │
  │       "Restart Service" template → EXEC  │
  │       "NovaMart Production" inventory    │
  │         → NO ACCESS (not granted)        │
  │                                          │
  │   Team: SRE                              │
  │     Members: sre-carol, sre-dave         │
  │     Permissions:                         │
  │       "NovaMart Staging" → ADMIN         │
  │       "NovaMart Production" → USE        │
  │       "Restart Service" template → EXEC  │
  │       (Production requires approval      │
  │        workflow — see below)             │
  │                                          │
  │   Team: Platform                         │
  │     Permissions: ADMIN on everything     │
  └─────────────────────────────────────────┘
```

**What happens when dev-alice tries to restart staging:**

```
1. Alice logs into AWX UI
2. Clicks "Restart Service" job template
3. Survey form appears:
   - Service Name: novamart-api (dropdown)
4. Inventory selection: 
   - She can ONLY see "NovaMart Staging" 
   - "NovaMart Production" is not in her list (no permission)
5. She clicks "Launch"
6. AWX:
   - Runs the playbook against staging inventory only
   - Uses staging SSH credentials (she never sees the key)
   - Logs: who=dev-alice, what=restart_service, when=2024-01-15T14:32:00,
           inventory=staging, service=novamart-api, result=success
7. Alice sees the output in real-time in the AWX UI
```

**What happens when dev-alice tries to touch production:**

```
She can't. The "NovaMart Production" inventory is not granted to the 
Developers team. When she launches the job template:
  - Production inventory doesn't appear in the dropdown
  - If she tries the API: 403 Forbidden
  - AWX audit log: "Permission denied: dev-alice attempted to access 
    inventory 'NovaMart Production'"
```

**For SRE team accessing production (with approval):**

```yaml
# AWX Workflow Template: "Production Service Restart"
#
# Node 1: Approval
#   Type: Approval node
#   Approvers: Platform team
#   Timeout: 30 minutes
#   Notification: Slack #prod-approvals
#
# Node 2: Execute (runs only after approval)
#   Job Template: "Restart Service"
#   Inventory: NovaMart Production
#   Credentials: Production SSH key
#
# Node 3: Notify
#   Type: Notification
#   Channel: #production-changes
#   Message: "Service restart completed by {user} on production"

Workflow:
  sre-carol requests restart
    → Slack: "🔔 sre-carol requests production restart of novamart-api. Approve?"
    → platform-admin clicks "Approve" in AWX
    → Playbook runs against production
    → Slack: "✅ novamart-api restarted on production by sre-carol (approved by platform-admin)"
    → Full audit trail in AWX
```

### Why Jenkins Permissions Are Weaker for This Use Case

```
JENKINS RBAC MODEL:
  ┌──────────────────────────────────────────────────────────┐
  │ Jenkins permissions are FOLDER-BASED and JOB-BASED:      │
  │                                                          │
  │ /jenkins/jobs/                                           │
  │   ├── staging/                                           │
  │   │   └── restart-service    ← dev-alice: Build permission│
  │   └── production/                                        │
  │       └── restart-service    ← dev-alice: No permission   │
  │                                                          │
  │ Problem 1: TWO SEPARATE JOBS for the same playbook       │
  │   The staging and production restart jobs are duplicates  │
  │   with different parameters. When the playbook changes,  │
  │   you must update BOTH jobs. Drift between them = risk.  │
  │                                                          │
  │ Problem 2: INVENTORY IS A PARAMETER, NOT A PERMISSION    │
  │   If you use ONE job with a "target" parameter:          │
  │     - dev-alice selects "staging" → OK                   │
  │     - dev-alice types "production" → Jenkins runs it!    │
  │     - Jenkins doesn't validate parameter VALUES           │
  │     - You'd need a Groovy sandbox script to check,       │
  │       which is fragile and bypassable                    │
  │                                                          │
  │ Problem 3: CREDENTIAL SCOPE IS COARSE                    │
  │   Jenkins credentials are scoped to folders or global.    │
  │   If dev-alice has Build permission on staging folder,    │
  │   she can see which credentials are AVAILABLE (though     │
  │   not their values). She could reference the production   │
  │   SSH key ID in a pipeline script if she has write access │
  │   to the Jenkinsfile.                                    │
  │                                                          │
  │ Problem 4: NO SURVEY FORMS                               │
  │   Jenkins "parameterized builds" are free-text fields.    │
  │   No dropdown validation, no enforced choices.            │
  │   A developer can type ANY host pattern as a parameter.   │
  │   Jenkins won't stop them — it just passes the string     │
  │   to ansible-playbook --limit.                           │
  │                                                          │
  │ Problem 5: AUDIT IS UNSTRUCTURED                         │
  │   Jenkins build log: a giant text blob.                   │
  │   "Who restarted what on which hosts at what time?"       │
  │   → Parse 500 lines of console output.                   │
  │   AWX: structured JSON record with user, inventory,      │
  │   job template, hosts affected, changed count,            │
  │   and approval chain.                                    │
  └──────────────────────────────────────────────────────────┘

AWX RBAC MODEL:
  ┌──────────────────────────────────────────────────────────┐
  │ AWX permissions are RESOURCE-BASED:                       │
  │                                                          │
  │ ONE job template: "Restart Service"                       │
  │ Permission check at LAUNCH TIME:                          │
  │                                                          │
  │   1. Does user have EXECUTE on this job template? → check │
  │   2. Does user have USE on the selected inventory? → check│
  │   3. Does user have USE on the required credential? → chk │
  │   4. Is the inventory allowed for this template?  → check │
  │                                                          │
  │   ALL FOUR must pass. If any fails → 403.                │
  │                                                          │
  │ dev-alice:                                               │
  │   ✓ EXECUTE on "Restart Service"                         │
  │   ✓ USE on "NovaMart Staging" inventory                  │
  │   ✗ No permission on "NovaMart Production" inventory     │
  │   → She literally cannot select production as a target    │
  │   → It doesn't appear in her UI                          │
  │   → API returns 403                                      │
  │   → No workaround exists without an admin granting access │
  │                                                          │
  │ The survey form enforces VALID CHOICES:                   │
  │   Service name: dropdown with fixed options               │
  │   She can't type an arbitrary command                     │
  │   She can't inject "--limit production" into a field      │
  │                                                          │
  │ Credentials are INVISIBLE:                               │
  │   dev-alice has USE permission on staging SSH credential   │
  │   She can't SEE the key, can't COPY it, can't REFERENCE  │
  │   it in another context. AWX injects it into the          │
  │   playbook execution environment and removes it after.    │
  └──────────────────────────────────────────────────────────┘

SUMMARY:

  Jenkins approach to "dev can't touch prod":
    → Duplicate jobs per environment
    → Folder-level permissions (coarse)
    → Parameter values not validated by RBAC
    → Credentials potentially referenceable across jobs
    → Audit = text parsing
    → Workarounds exist for a determined developer

  AWX approach to "dev can't touch prod":
    → Single job template, permission checked at launch
    → Resource-level permissions (granular)
    → Inventory access is a first-class permission object
    → Credentials injected and invisible
    → Audit = structured, queryable records
    → No workaround without admin intervention

  Verdict: For OPERATIONAL tasks (restart, patch, rotate),
  AWX is the correct tool. Jenkins is the wrong tool used
  because "we already have Jenkins."
```

### Complete Side-by-Side Example

```
SCENARIO: dev-alice wants to restart novamart-api on 3 staging hosts

═══════════════════════════════════════════════════════════════
                        JENKINS WAY
═══════════════════════════════════════════════════════════════

1. Someone creates a parameterized Jenkins job:
   - Parameter: ENVIRONMENT (string, default: "staging")
   - Parameter: SERVICE_NAME (string, default: "novamart-api")  
   - Parameter: LIMIT (string, optional host pattern)

2. Jenkinsfile:
   pipeline {
     parameters {
       choice(name: 'ENVIRONMENT', choices: ['staging', 'production'])
       string(name: 'SERVICE_NAME', defaultValue: 'novamart-api')
       string(name: 'LIMIT', defaultValue: '')
     }
     stages {
       stage('Restart') {
         steps {
           // DANGER: Nothing prevents alice from choosing 'production'
           // if she has Build permission on this job
           sh """
             ansible-playbook playbooks/restart_service.yml \
               -i inventory/${params.ENVIRONMENT}_aws_ec2.yml \
               -e service_name=${params.SERVICE_NAME} \
               ${params.LIMIT ? "--limit ${params.LIMIT}" : ''}
           """
         }
       }
     }
   }

3. Attempted safeguard (fragile):
   stage('Validate') {
     steps {
       script {
         if (params.ENVIRONMENT == 'production' && !currentBuild.rawBuild.getCause(hudson.model.Cause.UserIdCause).getUserId().startsWith('sre-')) {
           error("Only SRE team can target production")
         }
         // PROBLEMS:
         // - Groovy sandbox may block getCause()
         // - Username check is string-based — easily spoofed
         // - If someone edits the Jenkinsfile in a branch, they can remove this check
         // - Parameter injection: LIMIT="--inventory production_aws_ec2.yml --"
       }
     }
   }

4. Audit: 
   Build #47 console output: 800 lines of text
   To find who ran what: grep through logs manually
   
═══════════════════════════════════════════════════════════════
                        AWX WAY
═══════════════════════════════════════════════════════════════

1. Admin creates ONE job template: "Restart Service"
   - Playbook: playbooks/restart_service.yml
   - Ask inventory on launch: YES
   - Survey enabled:
     Service Name: [novamart-api ▾] (dropdown, fixed choices)

2. RBAC configured:
   Team "Developers":
     - "Restart Service" template → Execute
     - "NovaMart Staging" inventory → Use
     - "Staging SSH Key" credential → Use
     (no production permissions granted — they don't exist for this team)

3. dev-alice experience:
   - Opens AWX → Templates → "Restart Service" → Launch
   - Inventory dropdown shows: "NovaMart Staging" (only option)
   - Survey shows: Service Name: [novamart-api ▾]
   - Clicks Launch
   - Watches real-time output
   - Done

4. What alice CANNOT do:
   - Select production inventory (not in her permission set)
   - Type a custom host pattern (survey is a dropdown, not free text)
   - Reference production credentials (invisible to her)
   - Modify the playbook being run (template is locked)
   - Bypass the survey (survey is required on the template)

5. Audit:
   AWX → Jobs → #1247:
     Who: dev-alice
     When: 2024-01-15 14:32:07 UTC
     Template: Restart Service
     Inventory: NovaMart Staging (3 hosts)
     Extra vars: {"service_name": "novamart-api"}
     Status: Successful
     Changed: 3
     Duration: 12s
   
   Queryable via API:
   curl -s https://awx.novamart.internal/api/v2/jobs/?created_by__username=dev-alice | jq '.results[] | {id, name, status, started}'
```

### When Jenkins SHOULD Run Ansible (Not AWX)

```
Jenkins runs Ansible when:
  1. INITIAL PROVISIONING after Terraform
     → Part of the CI/CD pipeline (code change → infra change → config)
     → Trigger: git push to infrastructure repo
     → Jenkins orchestrates: TF plan → TF apply → wait → Ansible configure
     
  2. APPLICATION DEPLOYMENT CONFIG
     → Deploy new app version, update nginx vhost, rotate feature flags
     → Trigger: git push to application repo
     → Part of the CD pipeline, not an operational task

  3. INTEGRATION TESTING
     → Molecule tests run in Jenkins as part of role CI
     → Trigger: PR to ansible-roles repo

AWX runs Ansible when:
  1. SCHEDULED OPERATIONS
     → Patching (monthly), cert rotation (quarterly), compliance scans (weekly)
     → Trigger: AWX schedule
     
  2. AD-HOC OPERATIONS
     → Restart service, flush cache, add SSH key, investigate incident
     → Trigger: human clicks button in AWX UI
     
  3. SELF-SERVICE OPERATIONS
     → Developer restarts staging service
     → NOC engineer runs diagnostic playbook
     → Trigger: human with limited permissions

  4. DRIFT REMEDIATION
     → AWX runs ansible-playbook --check --diff on schedule
     → If drift detected, notifies team or auto-remediates
     → Trigger: AWX schedule

The boundary:
  CODE CHANGE triggers Jenkins → Jenkins may call Ansible
  OPERATIONAL NEED triggers AWX → AWX runs Ansible directly
  
  They NEVER overlap on the same task.
  If both Jenkins and AWX can restart a service, 
  you have a governance problem.
```
