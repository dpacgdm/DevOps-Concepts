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
