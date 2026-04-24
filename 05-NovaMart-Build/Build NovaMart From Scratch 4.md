# Phase 7 — Lesson 4: Application Onboarding

## 🎯 THE MISSION

You've built the infrastructure (L1), the platform (L2), and the deployment machinery (L3). Now you prove it all works by deploying the most critical service in NovaMart's fleet: **payment-service**.

This is the moment of truth. Every layer you've built gets exercised. If something was wired wrong in L1-L3, this is where it surfaces.

---

## WHY PAYMENT-SERVICE

Not random. Payment-service is the hardest onboarding because it touches **everything**:

```
┌───────────────────────────────────────────────────────────┐
│                  payment-service (Go)                     │
│                                                           │
│  INBOUND:   API Gateway → (Linkerd mTLS) → payment-svc    │
│  OUTBOUND:  RDS Postgres (payments DB)                    │
│             Redis (idempotency cache)                     │
│             SQS (order-events queue)                      │
│             External: Stripe/Adyen via egress proxy       │
│  SECRETS:   DB creds, API keys, signing keys              │
│  COMPLIANCE: PCI-DSS scope, audit logging mandatory       │
│  SLO:       99.95% availability, p99 < 500ms              │
└───────────────────────────────────────────────────────────┘
```

If you can onboard payment-service correctly, every other service is a simplified subset.

---

## 📋 DELIVERABLES

Build and commit everything to your repo. Here's what I expect:

### 1. Application Kubernetes Manifests (Kustomize)

```
gitops-repo/services/payment-service/
├── base/
│   ├── kustomization.yaml
│   ├── rollout.yaml              # Argo Rollout (NOT Deployment)
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   ├── configmap.yaml            # Non-sensitive config
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── networkpolicy.yaml
├── overlays/
│   ├── dev/
│   ├── staging/
│   └── production/
└── README.md
```

**What I'm looking for:**
- Rollout, not Deployment (you built Argo Rollouts in L3)
- Resource requests/limits that make sense for a Go service
- Probes that match payment-service behavior (think: DB dependency on startup)
- PDB that protects availability during deployments AND node drains
- HPA on the right metric (think: what drives payment-service scaling?)
- Environment-specific overrides that reflect real differences (replicas, resources, canary steps)

### 2. Security Layer (Terraform + K8s manifests)

```
terraform/environments/production/applications/
└── payment-service/
    ├── main.tf                   # IRSA role, SQS queue, KMS grants
    ├── variables.tf
    └── outputs.tf

gitops-repo/services/payment-service/base/
├── externalsecret.yaml           # DB creds, Stripe API key, signing key
├── networkpolicy.yaml            # Tight ingress + egress rules
└── (Kyverno policies if service-specific needed)
```

**What I'm looking for:**
- IRSA with least-privilege (what EXACTLY does payment-service need?)
- ExternalSecret pulling from AWS Secrets Manager (you built ESO in L2)
- NetworkPolicy that implements PCI-level isolation:
  - Ingress: ONLY from api-gateway pods
  - Egress: ONLY to payments DB, Redis, SQS endpoint, egress proxy, DNS, OTel collector
  - Everything else DENIED
- Think about what happens to in-flight transactions during secret rotation

### 3. Observability Layer

```
gitops-repo/services/payment-service/base/
├── servicemonitor.yaml           # Prometheus scraping
└── grafana-dashboard.configmap.yaml  # Four Golden Signals dashboard

gitops-repo/platform/prometheus/rules/
└── payment-service-slos.yaml     # SLI recording rules + burn rate alerts
```

**What I'm looking for:**
- ServiceMonitor that scrapes the right port/path
- OTel auto-instrumentation (how does this get enabled for a Go service? Trick question — think carefully)
- Grafana dashboard covering Four Golden Signals specific to payment processing
- SLO recording rules implementing the 99.95% availability and p99 < 500ms targets
- Burn rate alerts (you learned this in Phase 5 Lesson 4)
- Log format expectations (what fields MUST be in every payment-service log line?)
- Trace-to-log and metric-to-trace correlation wiring

### 4. Database Integration

```
gitops-repo/services/payment-service/base/
├── db-migration-job.yaml         # Schema migration (Flyway or golang-migrate)
└── (connection pooling config in configmap)
```

**What I'm looking for:**
- Migration strategy that's safe for zero-downtime deployments
- Connection pooling tuned for the pod count × HPA max
- What happens when a migration fails mid-apply?
- Read vs write connection routing (if applicable)

### 5. Argo Rollouts + ArgoCD

```
gitops-repo/services/payment-service/base/
├── rollout.yaml                  # Canary strategy with analysis
└── analysistemplate.yaml         # Prometheus-backed canary checks

gitops-repo/argocd/applications/
├── payment-service-dev.yaml
├── payment-service-staging.yaml
└── payment-service-production.yaml
```

**What I'm looking for:**
- Canary steps appropriate for a payment service (conservative)
- AnalysisTemplate querying real metrics (error rate, latency, not just "is pod running")
- ArgoCD Application with correct project, sync policy per environment
- What happens if analysis fails at step 3 of 6?

### 6. End-to-End Validation

```
docs/
├── payment-service-onboarding.md    # Step-by-step what was done
└── service-onboarding-template.md   # Generalized template for ANY new service
```

**The onboarding template is critical.** If onboarding payment-service took you 500 lines of YAML, and the next team needs to onboard inventory-service, they shouldn't need to reverse-engineer your work. Give them a template with:
- What to copy
- What to customize
- What Terraform to add
- What to request from Platform team vs self-service

### 7. Design Decisions

Continue your DD pattern:
```
docs/design-decisions/
└── DD-APP-001-through-XXX.md
```

Document every non-obvious choice (migration strategy, canary percentages, connection pool sizing, NetworkPolicy approach, etc.)

---

## ⚠️ TRAPS I'M WATCHING FOR

These are things I've seen engineers get wrong on exactly this type of task. I won't tell you what they are in detail — that defeats the purpose. But here are the categories:

1. **Go + OTel auto-instrumentation** — there's a fundamental difference from Java/Python. If you get this wrong, you'll have zero traces.

2. **Connection pool math** — if HPA scales to max and every pod opens max connections, does your RDS instance survive?

3. **NetworkPolicy + DNS** — a common omission that makes the pod fail to resolve anything.

4. **Canary analysis window** — if your canary gets 2 requests/minute in staging, can your AnalysisTemplate even produce a meaningful signal?

5. **Secret rotation without downtime** — if Secrets Manager rotates the DB password, what happens to running pods?

6. **Migration + Rollout ordering** — if the migration runs as part of the Rollout, and you rollback the Rollout, does the migration rollback? (Spoiler: no.)

7. **PCI audit logging** — payment-service needs to log every access to cardholder data. Where does this fit in your observability stack?

8. **Init container ordering** — if you have both a migration job and a "wait for dependencies" init, what runs first and why does it matter?

---

## 📐 GRADING CRITERIA

| Category | Weight | What I'm Evaluating |
|---|---|---|
| **Correctness** | 30% | Will this actually work on first apply? No broken references, no missing RBAC, no wrong ports |
| **Security** | 25% | PCI-appropriate isolation, least privilege, secret handling, audit trail |
| **Observability** | 20% | Can you detect payment failures within 2 minutes? Can you trace a failed payment end-to-end? |
| **Operability** | 15% | Canary safety, rollback works, migration safety, connection management |
| **Documentation** | 10% | Onboarding template is reusable, design decisions are clear |

---

## CONSTRAINTS

- **Language:** Go (payment-service is Go, per NovaMart stack)
- **Mesh:** Linkerd (you chose this in L2, not Istio)
- **GitOps:** ArgoCD + Kustomize (established in L3)
- **Secrets:** External Secrets Operator → AWS Secrets Manager (established in L2)
- **Canary:** Argo Rollouts (established in L3)
- **Database:** RDS PostgreSQL `novamart-payments` (established in L1)
- **Egress:** Squid proxy for external payment gateways (established in L2)

Everything must be consistent with what you've already built. If you reference something that doesn't exist in L1-L3, I'll flag it.

---

Go build. Commit to your repo. Tell me when it's ready for review.

# Phase 7, Lesson 4: Application Onboarding — COMPLETE DELIVERY

## Payment-Service: The Gold Standard Onboarding

---

## 1. ARCHITECTURE OVERVIEW

```
                    ┌──────────────────────────────────────────────────────────┐
                    │                    TRAFFIC FLOW                          │
                    │                                                          │
                    │   Client → ALB → Linkerd → api-gateway → payment-svc     │
                    │                                                          │
                    └──────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────────────────────────────┐
    │                         payment-service Pod                                  │
    │                                                                              │
    │  ┌───────────────────┐  ┌──────────────────────────────────────────────┐     │
    │  │  linkerd-proxy    │  │  payment-service (Go)                        │     │
    │  │  (sidecar, auto)  │  │                                              │     │
    │  │  - mTLS           │  │  :8080  → HTTP API (app traffic)             │     │
    │  │  - L7 metrics     │  │  :9090  → /metrics (Prometheus)              │     │
    │  │  - retries        │  │  :8080/healthz → liveness probe              │     │
    │  └───────────────────┘  │  :8080/readyz  → readiness probe             │     │
    │                         │  :8080/startupz → startup probe              │     │
    │  ┌───────────────────┐  │                                              │     │
    │  │  init: dep-check  │  │  Structured JSON logs → stdout               │     │
    │  │  (verify DB,Redis)│  │  OTel traces → manual SDK → Collector        │     │
    │  └───────────────────┘  └──────────────────────────────────────────────┘     │
    └──────────────────────────────────────────────────────────────────────────────┘
                    │                    │              │            │
         ┌──────────┘          ┌─────────┘     ┌───────┘     ┌──────┘
         ▼                     ▼               ▼              ▼
    ┌───────────┐     ┌──────────────┐   ┌──────────┐   ┌──────────────┐
    │ RDS Proxy │     │    Redis     │   │   SQS    │   │ Squid Egress │
    │ :5432     │     │ ElastiCache  │   │ order-   │   │ Proxy        │
    │     │     │     │ :6379        │   │ events   │   │ :3128        │
    │     ▼     │     └──────────────┘   │ + DLQ    │   │     │        │
    │ RDS PG    │                        └──────────┘   │     ▼        │
    │ payments  │                                       │ Stripe API   │
    └───────────┘                                       │ Adyen API    │
                                                        └──────────────┘

    ┌─────────────────────────────────────────────────────────────────────┐
    │                     OBSERVABILITY FLOW                              │
    │                                                                     │
    │  App metrics (:9090) ──── ServiceMonitor ───→ Prometheus            │
    │  Linkerd metrics ──────── auto-scraped ─────→ Prometheus            │
    │  App logs (stdout) ─────── Fluent Bit ──────→ Loki + CloudWatch     │
    │  Audit logs (stdout) ──── Fluent Bit filter → S3 (Object Lock)      │
    │  OTel traces (OTLP) ───── OTel Agent ───────→ OTel GW → Tempo       │
    │                                                                     │
    │  SLO burn rate alerts ──→ Alertmanager ──→ PagerDuty + Slack        │
    └─────────────────────────────────────────────────────────────────────┘
```

### Dependency Map

| Dependency | Type | Endpoint | Auth Method | Required at Startup |
|---|---|---|---|---|
| RDS Proxy (payments) | Persistent store | `payment-db-proxy.internal:5432` | IAM Auth via IRSA | Yes |
| Redis (ElastiCache) | Cache | `redis.novamart.internal:6379` | AUTH token from Secret | Yes |
| SQS (order-events) | Async messaging | AWS API (HTTPS) | IRSA | No (graceful degrade) |
| Stripe API | External payment | `api.stripe.com` via egress proxy | API key from Secret | No (graceful degrade) |
| OTel Collector | Telemetry | `otel-agent.observability:4317` | None (cluster-internal) | No |
| CoreDNS | DNS | `kube-dns.kube-system:53` | None | Yes |

### ArgoCD Sync Wave Ordering

```
Wave -3:  Namespace (if not exists) + ServiceAccount + IRSA annotation
Wave -2:  ExternalSecrets (must resolve to K8s Secrets before anything else)
Wave -1:  DB Migration Job (needs secrets, must complete before app starts)
Wave  0:  Rollout + Services + HPA + PDB + ConfigMap + NetworkPolicy
Wave  1:  ServiceMonitor + AnalysisTemplate + Grafana Dashboard
```

---

## 2. TERRAFORM — APPLICATION INFRASTRUCTURE

### `terraform/environments/production/applications/payment-service/main.tf`

```hcl
###############################################################################
# Payment Service — Application Infrastructure
# 
# Creates: IRSA role, SQS queue + DLQ, RDS Proxy, KMS grants,
#          Secrets Manager bootstrap entries
#
# Dependencies: eks (OIDC provider), networking (VPC, subnets, SGs),
#               data-tier (RDS instance, Redis), security (KMS keys)
###############################################################################

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    # Partial config — provided via -backend-config
    key = "applications/payment-service/terraform.tfstate"
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Service     = "payment-service"
      Team        = "payments"
      ManagedBy   = "terraform"
      CostCenter  = "payments"
      Compliance  = "pci-dss"
    }
  }
}

# ---------------------------------------------------------------------------
# Data sources — Pull from other state files
# ---------------------------------------------------------------------------

data "terraform_remote_state" "eks" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "eks/terraform.tfstate"
    region = var.aws_region
  }
}

data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "networking/terraform.tfstate"
    region = var.aws_region
  }
}

data "terraform_remote_state" "data_tier" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "data-tier/terraform.tfstate"
    region = var.aws_region
  }
}

data "terraform_remote_state" "security" {
  backend = "s3"
  config = {
    bucket = var.state_bucket
    key    = "security/terraform.tfstate"
    region = var.aws_region
  }
}

locals {
  cluster_name      = data.terraform_remote_state.eks.outputs.cluster_name
  oidc_provider_arn = data.terraform_remote_state.eks.outputs.oidc_provider_arn
  oidc_provider_url = data.terraform_remote_state.eks.outputs.oidc_provider_url
  vpc_id            = data.terraform_remote_state.networking.outputs.vpc_id
  private_subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
  data_subnet_ids   = data.terraform_remote_state.networking.outputs.data_subnet_ids

  rds_payments_endpoint       = data.terraform_remote_state.data_tier.outputs.rds_payments_endpoint
  rds_payments_instance_id    = data.terraform_remote_state.data_tier.outputs.rds_payments_instance_id
  rds_payments_port           = data.terraform_remote_state.data_tier.outputs.rds_payments_port
  rds_payments_sg_id          = data.terraform_remote_state.data_tier.outputs.rds_payments_security_group_id
  rds_payments_master_secret  = data.terraform_remote_state.data_tier.outputs.rds_payments_master_secret_arn
  redis_endpoint              = data.terraform_remote_state.data_tier.outputs.redis_primary_endpoint
  redis_auth_secret_arn       = data.terraform_remote_state.data_tier.outputs.redis_auth_secret_arn
  kms_general_key_arn         = data.terraform_remote_state.security.outputs.kms_general_key_arn
  kms_rds_key_arn             = data.terraform_remote_state.security.outputs.kms_rds_key_arn

  namespace          = "payments"
  service_account    = "payment-service"
  migrate_sa         = "payment-migrate"
  sqs_queue_name     = "novamart-${var.environment}-order-events"
  sqs_dlq_name       = "novamart-${var.environment}-order-events-dlq"
}

# ---------------------------------------------------------------------------
# 1. IRSA — Payment Service Role (Application)
# ---------------------------------------------------------------------------

resource "aws_iam_role" "payment_service" {
  name = "novamart-${var.environment}-payment-service"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = local.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${local.oidc_provider_url}:sub" = "system:serviceaccount:${local.namespace}:${local.service_account}"
            "${local.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = { Component = "irsa" }
}

# SQS — Send messages to order-events queue
resource "aws_iam_role_policy" "payment_sqs" {
  name = "sqs-order-events"
  role = aws_iam_role.payment_service.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "SendToOrderEvents"
        Effect   = "Allow"
        Action   = [
          "sqs:SendMessage",
          "sqs:GetQueueUrl",
          "sqs:GetQueueAttributes"    # For health checks
        ]
        Resource = aws_sqs_queue.order_events.arn
      }
    ]
  })
}

# Secrets Manager — Read payment secrets
resource "aws_iam_role_policy" "payment_secrets" {
  name = "secrets-read"
  role = aws_iam_role.payment_service.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadPaymentSecrets"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = [
          aws_secretsmanager_secret.stripe_credentials.arn,
          aws_secretsmanager_secret.payment_signing_key.arn,
          aws_secretsmanager_secret.db_app_credentials.arn,
          local.redis_auth_secret_arn
        ]
      }
    ]
  })
}

# KMS — Decrypt secrets
resource "aws_iam_role_policy" "payment_kms" {
  name = "kms-decrypt"
  role = aws_iam_role.payment_service.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DecryptSecrets"
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey"
        ]
        Resource = [local.kms_general_key_arn]
      }
    ]
  })
}

# RDS Proxy IAM Auth — Connect to DB via IAM (no password in pod)
resource "aws_iam_role_policy" "payment_rds_proxy" {
  name = "rds-proxy-connect"
  role = aws_iam_role.payment_service.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "RDSProxyConnect"
        Effect = "Allow"
        Action = "rds-db:connect"
        Resource = "arn:aws:rds-db:${var.aws_region}:${data.aws_caller_identity.current.account_id}:dbuser:${aws_db_proxy.payments.id}/payment_app"
      }
    ]
  })
}

data "aws_caller_identity" "current" {}

# ---------------------------------------------------------------------------
# 2. IRSA — Migration Role (DDL privileges, separate SA)
# ---------------------------------------------------------------------------

resource "aws_iam_role" "payment_migrate" {
  name = "novamart-${var.environment}-payment-migrate"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = local.oidc_provider_arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${local.oidc_provider_url}:sub" = "system:serviceaccount:${local.namespace}:${local.migrate_sa}"
            "${local.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = { Component = "irsa" }
}

resource "aws_iam_role_policy" "migrate_secrets" {
  name = "secrets-read"
  role = aws_iam_role.payment_migrate.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadMigrateCredentials"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = [aws_secretsmanager_secret.db_migrate_credentials.arn]
      },
      {
        Sid    = "DecryptSecrets"
        Effect = "Allow"
        Action = ["kms:Decrypt", "kms:DescribeKey"]
        Resource = [local.kms_general_key_arn]
      }
    ]
  })
}

# RDS Proxy connect for migrate user
resource "aws_iam_role_policy" "migrate_rds_proxy" {
  name = "rds-proxy-connect"
  role = aws_iam_role.payment_migrate.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "RDSProxyConnect"
        Effect = "Allow"
        Action = "rds-db:connect"
        Resource = "arn:aws:rds-db:${var.aws_region}:${data.aws_caller_identity.current.account_id}:dbuser:${aws_db_proxy.payments.id}/payment_migrate"
      }
    ]
  })
}

# ---------------------------------------------------------------------------
# 3. SQS — Order Events Queue + Dead Letter Queue
# ---------------------------------------------------------------------------

resource "aws_sqs_queue" "order_events_dlq" {
  name                       = local.sqs_dlq_name
  message_retention_seconds  = 1209600  # 14 days — DLQ keeps messages longer
  kms_master_key_id          = local.kms_general_key_arn
  kms_data_key_reuse_period_seconds = 300

  tags = { Component = "dlq" }
}

resource "aws_sqs_queue" "order_events" {
  name                       = local.sqs_queue_name
  visibility_timeout_seconds = 60       # 6x the expected processing time (10s)
  message_retention_seconds  = 345600   # 4 days
  receive_wait_time_seconds  = 20       # Long polling
  kms_master_key_id          = local.kms_general_key_arn
  kms_data_key_reuse_period_seconds = 300

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.order_events_dlq.arn
    maxReceiveCount     = 3    # After 3 failures, move to DLQ
  })

  tags = { Component = "queue" }
}

# Allow payment-service role to send messages
resource "aws_sqs_queue_policy" "order_events" {
  queue_url = aws_sqs_queue.order_events.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowPaymentServiceSend"
        Effect    = "Allow"
        Principal = { AWS = aws_iam_role.payment_service.arn }
        Action    = ["sqs:SendMessage", "sqs:GetQueueUrl", "sqs:GetQueueAttributes"]
        Resource  = aws_sqs_queue.order_events.arn
      },
      {
        Sid       = "AllowOrderServiceReceive"
        Effect    = "Allow"
        Principal = { AWS = var.order_service_role_arn }
        Action    = [
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "sqs:GetQueueUrl",
          "sqs:ChangeMessageVisibility"
        ]
        Resource  = aws_sqs_queue.order_events.arn
      }
    ]
  })
}

# DLQ alarm — if messages land here, something is broken
resource "aws_cloudwatch_metric_alarm" "dlq_messages" {
  alarm_name          = "novamart-${var.environment}-payment-dlq-not-empty"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Maximum"
  threshold           = 0
  alarm_description   = "Payment order-events DLQ has messages — investigate failed payment events"
  alarm_actions       = [var.sns_alerts_arn]
  ok_actions          = [var.sns_alerts_arn]

  dimensions = {
    QueueName = aws_sqs_queue.order_events_dlq.name
  }
}

# ---------------------------------------------------------------------------
# 4. RDS Proxy — Connection pooling + credential rotation transparency
# ---------------------------------------------------------------------------

resource "aws_security_group" "rds_proxy" {
  name_prefix = "novamart-${var.environment}-payment-rds-proxy-"
  vpc_id      = local.vpc_id
  description = "Security group for payment-service RDS Proxy"

  ingress {
    description     = "PostgreSQL from EKS pods"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [data.terraform_remote_state.eks.outputs.node_security_group_id]
  }

  egress {
    description     = "PostgreSQL to RDS"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [local.rds_payments_sg_id]
  }

  lifecycle {
    create_before_destroy = true
  }

  tags = { Name = "novamart-${var.environment}-payment-rds-proxy" }
}

# Allow RDS Proxy SG to reach RDS (add ingress rule on RDS SG)
resource "aws_security_group_rule" "rds_from_proxy" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.rds_proxy.id
  security_group_id        = local.rds_payments_sg_id
  description              = "PostgreSQL from RDS Proxy"
}

resource "aws_iam_role" "rds_proxy" {
  name = "novamart-${var.environment}-payment-rds-proxy"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = { Service = "rds.amazonaws.com" }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy" "rds_proxy_secrets" {
  name = "read-db-secrets"
  role = aws_iam_role.rds_proxy.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadDBSecrets"
        Effect = "Allow"
        Action = "secretsmanager:GetSecretValue"
        Resource = [
          aws_secretsmanager_secret.db_app_credentials.arn,
          aws_secretsmanager_secret.db_migrate_credentials.arn,
          local.rds_payments_master_secret
        ]
      },
      {
        Sid    = "DecryptSecrets"
        Effect = "Allow"
        Action = ["kms:Decrypt", "kms:DescribeKey"]
        Resource = [local.kms_general_key_arn, local.kms_rds_key_arn]
      }
    ]
  })
}

resource "aws_db_proxy" "payments" {
  name                   = "novamart-${var.environment}-payments"
  engine_family          = "POSTGRESQL"
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_subnet_ids         = local.data_subnet_ids
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  require_tls            = true
  idle_client_timeout    = 1800  # 30 minutes

  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"  # Force IAM auth — no password in pods
    secret_arn  = aws_secretsmanager_secret.db_app_credentials.arn
  }

  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"
    secret_arn  = aws_secretsmanager_secret.db_migrate_credentials.arn
  }

  debug_logging = var.environment != "production"  # Verbose logs in non-prod only

  tags = { Component = "rds-proxy" }
}

resource "aws_db_proxy_default_target_group" "payments" {
  db_proxy_name = aws_db_proxy.payments.name

  connection_pool_config {
    max_connections_percent      = 80   # Use 80% of RDS max_connections
    max_idle_connections_percent = 50
    connection_borrow_timeout    = 120  # Seconds to wait for connection
    session_pinning_filters      = ["EXCLUDE_VARIABLE_SETS"]
  }
}

resource "aws_db_proxy_target" "payments" {
  db_proxy_name          = aws_db_proxy.payments.name
  target_group_name      = aws_db_proxy_default_target_group.payments.name
  db_instance_identifier = local.rds_payments_instance_id
}

# CloudWatch alarm for RDS Proxy
resource "aws_cloudwatch_metric_alarm" "rds_proxy_connections" {
  alarm_name          = "novamart-${var.environment}-payment-rds-proxy-high-connections"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = 60
  statistic           = "Maximum"
  threshold           = var.rds_proxy_connection_alarm_threshold  # e.g., 500
  alarm_description   = "RDS Proxy connections above threshold — potential connection leak or scaling issue"
  alarm_actions       = [var.sns_alerts_arn]

  dimensions = {
    ProxyName = aws_db_proxy.payments.name
  }
}

# ---------------------------------------------------------------------------
# 5. Secrets Manager — Bootstrap entries
# ---------------------------------------------------------------------------

# DB App User (DML only: SELECT, INSERT, UPDATE, DELETE)
resource "aws_secretsmanager_secret" "db_app_credentials" {
  name       = "novamart/${var.environment}/payment-service/db-app"
  kms_key_id = local.kms_general_key_arn

  tags = { Component = "secret", SecretType = "database" }
}

resource "aws_secretsmanager_secret_version" "db_app_credentials" {
  secret_id = aws_secretsmanager_secret.db_app_credentials.id
  secret_string = jsonencode({
    username = "payment_app"
    password = var.db_app_initial_password  # Changed post-bootstrap, rotated by Lambda
    engine   = "postgres"
    host     = local.rds_payments_endpoint
    port     = local.rds_payments_port
    dbname   = "payments"
  })

  lifecycle {
    ignore_changes = [secret_string]  # After bootstrap, rotation Lambda manages this
  }
}

# DB Migration User (DDL: CREATE, ALTER, DROP + DML)
resource "aws_secretsmanager_secret" "db_migrate_credentials" {
  name       = "novamart/${var.environment}/payment-service/db-migrate"
  kms_key_id = local.kms_general_key_arn

  tags = { Component = "secret", SecretType = "database" }
}

resource "aws_secretsmanager_secret_version" "db_migrate_credentials" {
  secret_id = aws_secretsmanager_secret.db_migrate_credentials.id
  secret_string = jsonencode({
    username = "payment_migrate"
    password = var.db_migrate_initial_password
    engine   = "postgres"
    host     = local.rds_payments_endpoint
    port     = local.rds_payments_port
    dbname   = "payments"
  })

  lifecycle {
    ignore_changes = [secret_string]
  }
}

# Stripe API Credentials
resource "aws_secretsmanager_secret" "stripe_credentials" {
  name       = "novamart/${var.environment}/payment-service/stripe"
  kms_key_id = local.kms_general_key_arn

  tags = { Component = "secret", SecretType = "api-key", Compliance = "pci-dss" }
}

resource "aws_secretsmanager_secret_version" "stripe_credentials" {
  secret_id = aws_secretsmanager_secret.stripe_credentials.id
  secret_string = jsonencode({
    api_key        = var.stripe_api_key_initial
    webhook_secret = var.stripe_webhook_secret_initial
  })

  lifecycle {
    ignore_changes = [secret_string]
  }
}

# Payment Signing Key (for idempotency token signing / webhook verification)
resource "aws_secretsmanager_secret" "payment_signing_key" {
  name       = "novamart/${var.environment}/payment-service/signing-key"
  kms_key_id = local.kms_general_key_arn

  tags = { Component = "secret", SecretType = "signing-key", Compliance = "pci-dss" }
}

resource "aws_secretsmanager_secret_version" "payment_signing_key" {
  secret_id = aws_secretsmanager_secret.payment_signing_key.id
  secret_string = jsonencode({
    current  = var.signing_key_initial
    previous = ""  # Dual-key: app tries current first, falls back to previous during rotation
  })

  lifecycle {
    ignore_changes = [secret_string]
  }
}

# ---------------------------------------------------------------------------
# 6. Secret Rotation — Lambda for DB credentials
# ---------------------------------------------------------------------------

# Rotation Lambda — uses RDS Proxy-compatible single-user rotation
resource "aws_secretsmanager_secret_rotation" "db_app" {
  secret_id           = aws_secretsmanager_secret.db_app_credentials.id
  rotation_lambda_arn = var.db_rotation_lambda_arn  # Shared rotation Lambda from security module

  rotation_rules {
    automatically_after_days = 30
    duration                 = "2h"  # Window for rotation to complete
  }
}

resource "aws_secretsmanager_secret_rotation" "db_migrate" {
  secret_id           = aws_secretsmanager_secret.db_migrate_credentials.id
  rotation_lambda_arn = var.db_rotation_lambda_arn

  rotation_rules {
    automatically_after_days = 90  # Migrations run less frequently, rotate less often
    duration                 = "2h"
  }
}

# ---------------------------------------------------------------------------
# 7. RDS Proxy CloudWatch Dashboard (service-specific)
# ---------------------------------------------------------------------------

resource "aws_cloudwatch_dashboard" "payment_infra" {
  dashboard_name = "novamart-${var.environment}-payment-infra"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "RDS Proxy - Active Connections"
          metrics = [
            ["AWS/RDS", "DatabaseConnections", "ProxyName", aws_db_proxy.payments.name]
          ]
          period = 60
          stat   = "Maximum"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "SQS - Order Events"
          metrics = [
            ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", aws_sqs_queue.order_events.name],
            ["AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", aws_sqs_queue.order_events_dlq.name]
          ]
          period = 60
          stat   = "Maximum"
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 12
        height = 6
        properties = {
          title   = "RDS Proxy - Client Connection Errors"
          metrics = [
            ["AWS/RDS", "ClientConnectionsSetupFailedAuth", "ProxyName", aws_db_proxy.payments.name],
            ["AWS/RDS", "ClientConnectionsSetupSucceeded", "ProxyName", aws_db_proxy.payments.name]
          ]
          period = 60
          stat   = "Sum"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 6
        width  = 12
        height = 6
        properties = {
          title   = "SQS - Message Age (stale = stuck consumer)"
          metrics = [
            ["AWS/SQS", "ApproximateAgeOfOldestMessage", "QueueName", aws_sqs_queue.order_events.name]
          ]
          period = 60
          stat   = "Maximum"
        }
      }
    ]
  })
}
```

### `terraform/environments/production/applications/payment-service/variables.tf`

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "state_bucket" {
  type        = string
  description = "S3 bucket for Terraform remote state"
}

# --- Initial secrets (set once, then managed by rotation) ---

variable "db_app_initial_password" {
  type      = string
  sensitive = true
  validation {
    condition     = length(var.db_app_initial_password) >= 20
    error_message = "DB password must be at least 20 characters."
  }
}

variable "db_migrate_initial_password" {
  type      = string
  sensitive = true
  validation {
    condition     = length(var.db_migrate_initial_password) >= 20
    error_message = "DB password must be at least 20 characters."
  }
}

variable "stripe_api_key_initial" {
  type      = string
  sensitive = true
}

variable "stripe_webhook_secret_initial" {
  type      = string
  sensitive = true
}

variable "signing_key_initial" {
  type      = string
  sensitive = true
  validation {
    condition     = length(var.signing_key_initial) >= 32
    error_message = "Signing key must be at least 32 characters."
  }
}

# --- Cross-service references ---

variable "order_service_role_arn" {
  type        = string
  description = "IAM role ARN for order-service to consume from SQS"
}

variable "db_rotation_lambda_arn" {
  type        = string
  description = "ARN of shared Secrets Manager rotation Lambda"
}

variable "sns_alerts_arn" {
  type        = string
  description = "SNS topic ARN for CloudWatch alarms"
}

# --- Thresholds ---

variable "rds_proxy_connection_alarm_threshold" {
  type    = number
  default = 500
  description = "Alarm when RDS Proxy active connections exceed this"
}
```

### `terraform/environments/production/applications/payment-service/outputs.tf`

```hcl
output "irsa_role_arn" {
  description = "IRSA role ARN for payment-service pods"
  value       = aws_iam_role.payment_service.arn
}

output "migrate_role_arn" {
  description = "IRSA role ARN for migration jobs"
  value       = aws_iam_role.payment_migrate.arn
}

output "sqs_queue_url" {
  description = "SQS order-events queue URL"
  value       = aws_sqs_queue.order_events.url
}

output "sqs_queue_arn" {
  description = "SQS order-events queue ARN"
  value       = aws_sqs_queue.order_events.arn
}

output "sqs_dlq_arn" {
  description = "SQS order-events DLQ ARN"
  value       = aws_sqs_queue.order_events_dlq.arn
}

output "rds_proxy_endpoint" {
  description = "RDS Proxy endpoint for payment-service connections"
  value       = aws_db_proxy.payments.endpoint
}

output "rds_proxy_arn" {
  description = "RDS Proxy ARN"
  value       = aws_db_proxy.payments.arn
}

output "secrets" {
  description = "Secrets Manager ARNs for ExternalSecrets references"
  value = {
    db_app     = aws_secretsmanager_secret.db_app_credentials.arn
    db_migrate = aws_secretsmanager_secret.db_migrate_credentials.arn
    stripe     = aws_secretsmanager_secret.stripe_credentials.arn
    signing    = aws_secretsmanager_secret.payment_signing_key.arn
  }
}
```

---

## 3. KUBERNETES MANIFESTS — GITOPS REPO

### `gitops-repo/services/payment-service/base/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

metadata:
  labels:
    app.kubernetes.io/part-of: novamart
    app.kubernetes.io/managed-by: kustomize

commonLabels:
  app: payment-service
  app.kubernetes.io/name: payment-service
  app.kubernetes.io/component: backend
  team: payments

resources:
  - namespace.yaml
  - serviceaccount.yaml
  - externalsecret.yaml
  - configmap.yaml
  - rollout.yaml
  - service.yaml
  - hpa.yaml
  - pdb.yaml
  - networkpolicy.yaml
  - servicemonitor.yaml
  - analysistemplate.yaml
  - grafana-dashboard-configmap.yaml
  - db-migration-job.yaml
```

### `gitops-repo/services/payment-service/base/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    # Linkerd mesh injection
    linkerd.io/inject: enabled
    # Pod Security Standards
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    # Kyverno policy scope
    novamart.io/compliance: pci-dss
    novamart.io/team: payments
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
```

### `gitops-repo/services/payment-service/base/serviceaccount.yaml`

```yaml
# --- Application ServiceAccount ---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service
  namespace: payments
  annotations:
    # IRSA — set per environment in overlay
    eks.amazonaws.com/role-arn: PLACEHOLDER_IRSA_ROLE_ARN
    argocd.argoproj.io/sync-wave: "-3"
  labels:
    app: payment-service
automountServiceAccountToken: true  # Needed for IRSA projected token

---
# --- Migration ServiceAccount (separate, different IRSA role) ---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-migrate
  namespace: payments
  annotations:
    eks.amazonaws.com/role-arn: PLACEHOLDER_MIGRATE_ROLE_ARN
    argocd.argoproj.io/sync-wave: "-3"
  labels:
    app: payment-service
    component: migration
automountServiceAccountToken: true
```

### `gitops-repo/services/payment-service/base/externalsecret.yaml`

```yaml
# --- Database Application Credentials ---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-db-credentials
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  refreshInterval: 1m  # Frequent refresh — catches rotation within 1 minute
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-db-credentials
    creationPolicy: Owner
    deletionPolicy: Retain  # Don't delete K8s secret if ExternalSecret is removed
    template:
      type: Opaque
      data:
        # RDS Proxy with IAM auth: pods use IAM, not password directly
        # But connection string still needed for golang-migrate and debugging
        DB_HOST: "{{ .db_host }}"
        DB_PORT: "{{ .db_port }}"
        DB_NAME: "{{ .db_name }}"
        DB_USER: "{{ .db_user }}"
        DB_PASSWORD: "{{ .db_password }}"
        # Full DSN for golang-migrate
        DB_DSN: "postgres://{{ .db_user }}:{{ .db_password }}@{{ .db_host }}:{{ .db_port }}/{{ .db_name }}?sslmode=require"
  data:
    - secretKey: db_host
      remoteRef:
        key: PLACEHOLDER_DB_SECRET_NAME
        property: host
    - secretKey: db_port
      remoteRef:
        key: PLACEHOLDER_DB_SECRET_NAME
        property: port
    - secretKey: db_name
      remoteRef:
        key: PLACEHOLDER_DB_SECRET_NAME
        property: dbname
    - secretKey: db_user
      remoteRef:
        key: PLACEHOLDER_DB_SECRET_NAME
        property: username
    - secretKey: db_password
      remoteRef:
        key: PLACEHOLDER_DB_SECRET_NAME
        property: password

---
# --- Database Migration Credentials ---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-db-migrate-credentials
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  refreshInterval: 5m  # Migrations run infrequently, no need for 1m
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-db-migrate-credentials
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      type: Opaque
      data:
        MIGRATE_DB_DSN: "postgres://{{ .username }}:{{ .password }}@{{ .host }}:{{ .port }}/{{ .dbname }}?sslmode=require"
  dataFrom:
    - extract:
        key: PLACEHOLDER_MIGRATE_SECRET_NAME

---
# --- Stripe API Credentials ---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-stripe-credentials
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-stripe-credentials
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      type: Opaque
      data:
        STRIPE_API_KEY: "{{ .api_key }}"
        STRIPE_WEBHOOK_SECRET: "{{ .webhook_secret }}"
  dataFrom:
    - extract:
        key: PLACEHOLDER_STRIPE_SECRET_NAME

---
# --- Payment Signing Key ---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-signing-key
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-signing-key
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      type: Opaque
      data:
        SIGNING_KEY_CURRENT: "{{ .current }}"
        SIGNING_KEY_PREVIOUS: "{{ .previous }}"
  dataFrom:
    - extract:
        key: PLACEHOLDER_SIGNING_SECRET_NAME

---
# --- Redis AUTH Token ---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-redis-credentials
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-redis-credentials
    creationPolicy: Owner
    deletionPolicy: Retain
  data:
    - secretKey: REDIS_AUTH_TOKEN
      remoteRef:
        key: PLACEHOLDER_REDIS_SECRET_NAME
        property: auth_token
```

### `gitops-repo/services/payment-service/base/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-service-config
  namespace: payments
data:
  # --- Server ---
  SERVER_PORT: "8080"
  METRICS_PORT: "9090"
  SHUTDOWN_TIMEOUT: "30s"  # Must be < terminationGracePeriodSeconds (45s)

  # --- Database (non-sensitive) ---
  DB_MAX_OPEN_CONNS: "10"       # Per pod. HPA max 20 pods × 10 = 200 ≤ RDS Proxy 80% limit
  DB_MAX_IDLE_CONNS: "5"
  DB_CONN_MAX_LIFETIME: "5m"    # Shorter than RDS Proxy idle timeout (30m)
  DB_CONN_MAX_IDLE_TIME: "1m"
  DB_CONNECT_TIMEOUT: "5s"
  DB_STATEMENT_TIMEOUT: "10s"   # Kill queries running > 10s
  DB_USE_IAM_AUTH: "true"       # Use IAM auth via RDS Proxy, not password

  # --- Redis ---
  REDIS_HOST: "redis.novamart.internal"
  REDIS_PORT: "6379"
  REDIS_DB: "0"
  REDIS_POOL_SIZE: "5"          # Per pod. 20 pods × 5 = 100 connections
  REDIS_READ_TIMEOUT: "500ms"
  REDIS_WRITE_TIMEOUT: "500ms"
  REDIS_DIAL_TIMEOUT: "2s"
  REDIS_TLS_ENABLED: "true"
  REDIS_IDEMPOTENCY_TTL: "24h" # Cache idempotency keys for 24 hours

  # --- SQS ---
  SQS_QUEUE_URL: "PLACEHOLDER_SQS_URL"  # Set per environment in overlay
  SQS_REGION: "us-east-1"
  SQS_MAX_RETRIES: "3"

  # --- External Payment Gateway ---
  PAYMENT_GATEWAY_PROVIDER: "stripe"
  PAYMENT_GATEWAY_TIMEOUT: "15s"
  PAYMENT_GATEWAY_RETRY_MAX: "2"
  PAYMENT_GATEWAY_RETRY_BACKOFF: "1s"
  # Egress proxy for PCI compliance — all external calls go through Squid
  HTTP_PROXY: "http://squid-egress-proxy.egress-proxy.svc.cluster.local:3128"
  HTTPS_PROXY: "http://squid-egress-proxy.egress-proxy.svc.cluster.local:3128"
  NO_PROXY: ".svc.cluster.local,.cluster.local,169.254.169.254,10.0.0.0/8,100.64.0.0/10"

  # --- OpenTelemetry ---
  OTEL_SERVICE_NAME: "payment-service"
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-agent.observability.svc.cluster.local:4317"
  OTEL_EXPORTER_OTLP_PROTOCOL: "grpc"
  OTEL_TRACES_SAMPLER: "parentbased_traceidratio"
  OTEL_TRACES_SAMPLER_ARG: "1.0"  # 100% at head — tail sampling at gateway decides final
  OTEL_RESOURCE_ATTRIBUTES: "service.namespace=payments,service.version=PLACEHOLDER_VERSION,deployment.environment=PLACEHOLDER_ENV"
  OTEL_PROPAGATORS: "tracecontext,baggage"  # W3C standard

  # --- Application ---
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  ENVIRONMENT: "PLACEHOLDER_ENV"

  # --- PCI Audit ---
  AUDIT_LOG_ENABLED: "true"
  AUDIT_LOG_FIELDS: "timestamp,trace_id,span_id,user_id,action,resource,outcome,ip,payment_method_type"
  # NEVER log: card numbers, CVV, full token — app must mask these
```

### `gitops-repo/services/payment-service/base/rollout.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "0"
    notifications.argoproj.io/subscribe.on-rollout-completed.slack: payments-deploys
    notifications.argoproj.io/subscribe.on-rollout-aborted.slack: payments-alerts
    notifications.argoproj.io/subscribe.on-analysis-run-failed.slack: payments-alerts
spec:
  replicas: 3  # Base — overridden per environment
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: payment-service

  strategy:
    canary:
      canaryService: payment-service-canary
      stableService: payment-service-stable
      # Traffic management via Linkerd TrafficSplit
      trafficRouting:
        plugins:
          argoproj-labs/linkerd:
            rootService: payment-service
            canaryService: payment-service-canary
            stableService: payment-service-stable
      # Conservative canary steps for payment service
      steps:
        - setWeight: 5
        - pause: { duration: 2m }      # Let initial traffic flow
        - analysis:
            templates:
              - templateName: payment-canary-analysis
            args:
              - name: service-name
                value: payment-service-canary
        - setWeight: 15
        - pause: { duration: 3m }
        - analysis:
            templates:
              - templateName: payment-canary-analysis
            args:
              - name: service-name
                value: payment-service-canary
        - setWeight: 30
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: payment-canary-analysis
            args:
              - name: service-name
                value: payment-service-canary
        - setWeight: 60
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: payment-canary-analysis
            args:
              - name: service-name
                value: payment-service-canary
        - setWeight: 100
      # Automatic rollback on failure
      abortScaleDownDelaySeconds: 30
      # Anti-affinity between canary and stable for blast radius isolation
      canaryMetadata:
        labels:
          role: canary
        annotations:
          prometheus.io/canary: "true"
      stableMetadata:
        labels:
          role: stable

  template:
    metadata:
      labels:
        app: payment-service
        version: "PLACEHOLDER_VERSION"
      annotations:
        linkerd.io/inject: enabled
        # OTel — Go does NOT support auto-instrumentation via Operator annotation
        # Go requires MANUAL SDK integration in application code
        # This annotation is intentionally ABSENT (unlike Java/Python/Node)
        #
        # For Java it would be:
        #   instrumentation.opentelemetry.io/inject-java: "true"
        # For Go, we instrument in application code using:
        #   go.opentelemetry.io/otel SDK
        #   go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
        #   go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc
        #   go.opentelemetry.io/contrib/instrumentation/database/sql/otelsql
        config.linkerd.io/proxy-cpu-request: "50m"
        config.linkerd.io/proxy-memory-request: "64Mi"
        config.linkerd.io/proxy-cpu-limit: "500m"
        config.linkerd.io/proxy-memory-limit: "256Mi"
        # Opaque ports — tell Linkerd these are NOT HTTP (skip protocol detection)
        config.linkerd.io/opaque-ports: "5432,6379"
    spec:
      serviceAccountName: payment-service
      terminationGracePeriodSeconds: 45  # > preStop (5s) + SHUTDOWN_TIMEOUT (30s) + buffer

      # Topology spread — distribute across AZs for HA
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: payment-service
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: payment-service

      # Don't co-locate with other payment pods on same node (best effort)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: payment-service
                topologyKey: kubernetes.io/hostname

      securityContext:
        runAsNonRoot: true
        runAsUser: 65534       # nobody
        runAsGroup: 65534
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault

      # --- Init Container: Dependency Check ---
      initContainers:
        - name: wait-for-dependencies
          image: PLACEHOLDER_ECR_REGISTRY/novamart/toolbox:1.0.0
          command:
            - /bin/sh
            - -c
            - |
              set -e
              echo "Checking database connectivity..."
              for i in $(seq 1 30); do
                if pg_isready -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -t 2; then
                  echo "Database is ready"
                  break
                fi
                if [ $i -eq 30 ]; then
                  echo "ERROR: Database not ready after 30 attempts"
                  exit 1
                fi
                echo "Waiting for database... attempt $i/30"
                sleep 2
              done

              echo "Checking Redis connectivity..."
              for i in $(seq 1 15); do
                if redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" --tls --no-auth-warning -a "$REDIS_AUTH_TOKEN" ping | grep -q PONG; then
                  echo "Redis is ready"
                  break
                fi
                if [ $i -eq 15 ]; then
                  echo "ERROR: Redis not ready after 15 attempts"
                  exit 1
                fi
                echo "Waiting for Redis... attempt $i/15"
                sleep 2
              done

              echo "All dependencies ready"
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_HOST
            - name: DB_PORT
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_PORT
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_USER
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_NAME
            - name: REDIS_HOST
              valueFrom:
                configMapKeyRef:
                  name: payment-service-config
                  key: REDIS_HOST
            - name: REDIS_PORT
              valueFrom:
                configMapKeyRef:
                  name: payment-service-config
                  key: REDIS_PORT
            - name: REDIS_AUTH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: payment-redis-credentials
                  key: REDIS_AUTH_TOKEN
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

      containers:
        - name: payment-service
          image: PLACEHOLDER_ECR_REGISTRY/novamart/payment-service:PLACEHOLDER_TAG
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP

          # --- Probes ---
          # Startup: generous — wait for DB connection pool, cache warm
          startupProbe:
            httpGet:
              path: /startupz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 3
            failureThreshold: 20   # 5 + (20×3) = 65s max startup time
            timeoutSeconds: 2

          # Readiness: must have working DB + Redis connections
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 3

          # Liveness: basic process health (NOT dependency checks)
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 3

          # --- Resources ---
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              # NO CPU limit — avoid CFS throttling on Go service
              # Go runtime is good at managing goroutines; throttling causes latency spikes
              memory: 512Mi

          # --- Environment Variables ---
          envFrom:
            - configMapRef:
                name: payment-service-config
          env:
            # Secrets as individual env vars (not envFrom — explicit is safer)
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_HOST
            - name: DB_PORT
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_PORT
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_NAME
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_PASSWORD
            - name: STRIPE_API_KEY
              valueFrom:
                secretKeyRef:
                  name: payment-stripe-credentials
                  key: STRIPE_API_KEY
            - name: STRIPE_WEBHOOK_SECRET
              valueFrom:
                secretKeyRef:
                  name: payment-stripe-credentials
                  key: STRIPE_WEBHOOK_SECRET
            - name: SIGNING_KEY_CURRENT
              valueFrom:
                secretKeyRef:
                  name: payment-signing-key
                  key: SIGNING_KEY_CURRENT
            - name: SIGNING_KEY_PREVIOUS
              valueFrom:
                secretKeyRef:
                  name: payment-signing-key
                  key: SIGNING_KEY_PREVIOUS
            - name: REDIS_AUTH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: payment-redis-credentials
                  key: REDIS_AUTH_TOKEN
            # Pod identity for logging/tracing
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            # Go runtime
            - name: GOMAXPROCS
              valueFrom:
                resourceFieldRef:
                  resource: requests.cpu
                  # automaxprocs library is better — set in app code
                  # This is fallback if library not imported

          # --- Lifecycle ---
          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    # Signal app to stop accepting new requests
                    # Wait for Linkerd proxy to drain connections
                    # Wait for endpoint removal to propagate (Linkerd + kube-proxy)
                    sleep 5

          # --- Security ---
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          # --- Volume Mounts ---
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: audit-config
              mountPath: /etc/payment-service/audit
              readOnly: true

      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
        - name: audit-config
          configMap:
            name: payment-audit-config
```

### `gitops-repo/services/payment-service/base/service.yaml`

```yaml
# --- Primary Service (routes to both stable + canary via Linkerd TrafficSplit) ---
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
    - name: metrics
      port: 9090
      targetPort: metrics
      protocol: TCP
  selector:
    app: payment-service

---
# --- Stable Service (Argo Rollouts canary routing) ---
apiVersion: v1
kind: Service
metadata:
  name: payment-service-stable
  namespace: payments
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
  selector:
    app: payment-service

---
# --- Canary Service (Argo Rollouts canary routing) ---
apiVersion: v1
kind: Service
metadata:
  name: payment-service-canary
  namespace: payments
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
  selector:
    app: payment-service
```

### `gitops-repo/services/payment-service/base/hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  scaleTargetRef:
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    name: payment-service
  minReplicas: 3      # Minimum for HA across 3 AZs
  maxReplicas: 20     # Max — constrained by DB connection math

  metrics:
    # Primary: RPS-based via custom metrics (requests per second per pod)
    # Payment processing is I/O bound (DB + external API), not CPU bound
    # CPU is a poor scaling signal for payment workloads
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "50"  # Scale up when average RPS per pod > 50

    # Secondary: CPU as safety net (in case custom metrics fail)
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Tertiary: Memory as safety net
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30    # React quickly to traffic spikes
      policies:
        - type: Pods
          value: 4                      # Add max 4 pods at a time
          periodSeconds: 60
        - type: Percent
          value: 50
          periodSeconds: 60
      selectPolicy: Min                 # Use the more conservative policy

    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 minutes before scaling down
      policies:                         # Slow scale-down — payment traffic is bursty
        - type: Pods
          value: 2
          periodSeconds: 120
      selectPolicy: Min
```

### `gitops-repo/services/payment-service/base/pdb.yaml`

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  # maxUnavailable: 1 is better than minAvailable for payment service
  # With minAvailable: N-1 and N pods running, PDB blocks ALL disruptions
  # if one pod is already down (e.g., during deployment)
  # maxUnavailable: 1 always allows exactly 1 pod to be disrupted
  maxUnavailable: 1
  selector:
    matchLabels:
      app: payment-service

  # unhealthyPodEvictionPolicy: AlwaysAllow (K8s 1.31+)
  # Prevents stuck PDB when pods are already unhealthy
  # Without this: if a pod is CrashLoopBackOff, PDB counts it as "unavailable"
  # and blocks node drain even though the pod isn't serving anyway
```

### `gitops-repo/services/payment-service/base/networkpolicy.yaml`

```yaml
# --- DEFAULT DENY ALL — PCI requirement: explicit allow only ---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  podSelector: {}  # All pods in namespace
  policyTypes:
    - Ingress
    - Egress

---
# --- Payment Service Ingress ---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-ingress
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Ingress
  ingress:
    # Allow traffic from api-gateway only
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: api-gateway
          podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - port: 8080
          protocol: TCP

    # Allow Prometheus scraping from observability namespace
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: observability
          podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - port: 9090
          protocol: TCP

    # Allow Linkerd control plane (tap, identity, destination)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: linkerd
      ports:
        - port: 4143     # Linkerd proxy inbound
        - port: 4191     # Linkerd proxy admin

---
# --- Payment Service Egress ---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-egress
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Egress
  egress:
    # DNS — CRITICAL: without this, pod can't resolve any hostname
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP

    # NodeLocal DNSCache (if deployed — runs on node network)
    - to:
        - ipBlock:
            cidr: 169.254.20.10/32  # NodeLocal DNS VIP
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP

    # RDS Proxy (data subnet CIDR)
    - to:
        - ipBlock:
            cidr: 10.0.48.0/20     # data-subnet-a CIDR
        - ipBlock:
            cidr: 10.0.64.0/20     # data-subnet-b CIDR
        - ipBlock:
            cidr: 10.0.80.0/20     # data-subnet-c CIDR
      ports:
        - port: 5432
          protocol: TCP

    # Redis (ElastiCache — data subnet)
    - to:
        - ipBlock:
            cidr: 10.0.48.0/20
        - ipBlock:
            cidr: 10.0.64.0/20
        - ipBlock:
            cidr: 10.0.80.0/20
      ports:
        - port: 6379
          protocol: TCP

    # SQS VPC Endpoint (interface endpoint in private subnets)
    - to:
        - ipBlock:
            cidr: 10.0.16.0/20     # private-subnet-a
        - ipBlock:
            cidr: 10.0.32.0/20     # private-subnet-b
        - ipBlock:
            cidr: 10.0.48.0/20     # private-subnet-c
      ports:
        - port: 443
          protocol: TCP

    # OTel Collector Agent (observability namespace)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: observability
          podSelector:
            matchLabels:
              app.kubernetes.io/name: otel-agent
      ports:
        - port: 4317          # OTLP gRPC
          protocol: TCP
        - port: 4318          # OTLP HTTP
          protocol: TCP

    # Egress proxy (Squid — for Stripe/Adyen API calls)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: egress-proxy
          podSelector:
            matchLabels:
              app: squid-egress-proxy
      ports:
        - port: 3128
          protocol: TCP

    # Linkerd control plane
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: linkerd
      ports:
        - port: 8443          # Linkerd destination
        - port: 8080          # Linkerd identity

    # STS VPC Endpoint (for IRSA token exchange)
    - to:
        - ipBlock:
            cidr: 10.0.16.0/20
        - ipBlock:
            cidr: 10.0.32.0/20
        - ipBlock:
            cidr: 10.0.48.0/20
      ports:
        - port: 443
          protocol: TCP

    # Secrets Manager VPC Endpoint
    # (ESO runs in platform namespace, but pod needs STS for IRSA)
    # STS egress already covered above

    # IMDS (instance metadata — needed for IRSA)
    - to:
        - ipBlock:
            cidr: 169.254.169.254/32
      ports:
        - port: 80
          protocol: TCP

---
# --- Migration Job Egress ---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-migrate-egress
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payment-service
      component: migration
  policyTypes:
    - Egress
  egress:
    # DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP

    # NodeLocal DNS
    - to:
        - ipBlock:
            cidr: 169.254.20.10/32
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP

    # RDS Proxy only
    - to:
        - ipBlock:
            cidr: 10.0.48.0/20
        - ipBlock:
            cidr: 10.0.64.0/20
        - ipBlock:
            cidr: 10.0.80.0/20
      ports:
        - port: 5432
          protocol: TCP

    # STS for IRSA
    - to:
        - ipBlock:
            cidr: 10.0.16.0/20
        - ipBlock:
            cidr: 10.0.32.0/20
        - ipBlock:
            cidr: 10.0.48.0/20
      ports:
        - port: 443
          protocol: TCP

    # IMDS
    - to:
        - ipBlock:
            cidr: 169.254.169.254/32
      ports:
        - port: 80
          protocol: TCP
```

### `gitops-repo/services/payment-service/base/db-migration-job.yaml`

```yaml
###############################################################################
# Database Migration Job
#
# DESIGN DECISIONS:
# 1. Separate Job, NOT init container — migrations must NOT re-run on every pod restart
#    Init container = runs on every pod start = dangerous for DDL
#    Job = runs once per deployment, can be monitored independently
#
# 2. Separate ServiceAccount — migration user has DDL privileges (CREATE, ALTER, DROP)
#    App user has DML only (SELECT, INSERT, UPDATE, DELETE)
#    Principle of least privilege: app can never accidentally DROP TABLE
#
# 3. golang-migrate (not Flyway) — Go ecosystem, lightweight, no JVM
#
# 4. ArgoCD sync wave -1 — runs AFTER secrets (wave -2), BEFORE app (wave 0)
#    If migration fails, ArgoCD sync fails, app doesn't deploy
#
# 5. Migrations MUST be backward-compatible (expand-contract pattern):
#    - Adding columns: add with DEFAULT, never NOT NULL without DEFAULT
#    - Removing columns: stop reading first (deploy app), then drop column (next deploy)
#    - Renaming: add new column, backfill, deploy app to use new, drop old
#    - This ensures rollback safety — old app code works with new schema
###############################################################################

apiVersion: batch/v1
kind: Job
metadata:
  name: payment-db-migrate-PLACEHOLDER_VERSION
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
    argocd.argoproj.io/hook: PreSync      # Run before main sync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation  # Clean up old job before creating new
  labels:
    app: payment-service
    component: migration
spec:
  backoffLimit: 3           # Retry 3 times on failure
  activeDeadlineSeconds: 300  # 5 minute timeout
  ttlSecondsAfterFinished: 86400  # Clean up after 24h

  template:
    metadata:
      labels:
        app: payment-service
        component: migration
      annotations:
        linkerd.io/inject: enabled
        # Linkerd sidecar must shut down after migration completes
        # Otherwise Job never completes (sidecar keeps running)
        config.linkerd.io/shutdown-grace-period: "10"
    spec:
      serviceAccountName: payment-migrate
      restartPolicy: OnFailure

      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        runAsGroup: 65534
        seccompProfile:
          type: RuntimeDefault

      initContainers:
        # Wait for DB before running migration
        - name: wait-for-db
          image: PLACEHOLDER_ECR_REGISTRY/novamart/toolbox:1.0.0
          command:
            - /bin/sh
            - -c
            - |
              for i in $(seq 1 30); do
                if pg_isready -h "$DB_HOST" -p "$DB_PORT" -t 2; then
                  echo "Database ready"
                  exit 0
                fi
                echo "Waiting for database... attempt $i/30"
                sleep 2
              done
              echo "Database not ready after 60s"
              exit 1
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: payment-db-migrate-credentials
                  key: DB_HOST
            - name: DB_PORT
              value: "5432"
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              memory: 64Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

      containers:
        - name: migrate
          image: PLACEHOLDER_ECR_REGISTRY/novamart/payment-service:PLACEHOLDER_TAG
          command:
            - /bin/sh
            - -c
            - |
              set -e

              echo "Starting migration..."
              echo "Database: $DB_HOST:5432/payments"

              # Run migrations
              /app/migrate -path /app/migrations -database "$MIGRATE_DB_DSN" up

              RESULT=$?
              if [ $RESULT -eq 0 ]; then
                echo "Migration completed successfully"
              else
                echo "Migration FAILED with exit code $RESULT"
                exit $RESULT
              fi

              # Shut down Linkerd sidecar so Job can complete
              # This is the correct way to handle sidecars in Jobs
              wget -q -O- --post-data="" http://localhost:4191/shutdown || true

          env:
            - name: MIGRATE_DB_DSN
              valueFrom:
                secretKeyRef:
                  name: payment-db-migrate-credentials
                  key: MIGRATE_DB_DSN
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: payment-db-credentials
                  key: DB_HOST

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 50Mi
```

### `gitops-repo/services/payment-service/base/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-service
  namespace: payments
  labels:
    # Must match Prometheus serviceMonitorSelector
    release: kube-prometheus-stack
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  selector:
    matchLabels:
      app: payment-service
  namespaceSelector:
    matchNames:
      - payments
  endpoints:
    - port: metrics           # Named port from Service
      path: /metrics
      interval: 15s           # Match Prometheus global scrape interval
      scrapeTimeout: 10s
      honorLabels: false      # Prometheus controls labels, not the app
      metricRelabelings:
        # Drop high-cardinality Go runtime metrics we don't need
        - sourceLabels: [__name__]
          regex: "go_gc_.*"
          action: drop
        # Keep standard Go metrics we do need
        - sourceLabels: [__name__]
          regex: "(go_goroutines|go_memstats_alloc_bytes|go_memstats_heap_.*|go_threads|process_.*)"
          action: keep
        # Keep all payment_* business metrics
        - sourceLabels: [__name__]
          regex: "payment_.*"
          action: keep
        # Keep all http_* request metrics
        - sourceLabels: [__name__]
          regex: "http_.*"
          action: keep
        # Keep all db_* connection metrics
        - sourceLabels: [__name__]
          regex: "db_.*"
          action: keep
        # Keep all redis_* metrics
        - sourceLabels: [__name__]
          regex: "redis_.*"
          action: keep
        # Keep all sqs_* metrics
        - sourceLabels: [__name__]
          regex: "sqs_.*"
          action: keep
        # Drop everything else
        - sourceLabels: [__name__]
          regex: ".*"
          action: drop
```

**WAIT — the metricRelabelings above have a bug.** The `keep` + `drop` ordering is wrong. Let me fix this. In Prometheus, relabelings are applied sequentially. If I `drop` at the end with `.*`, it drops everything including what I just kept. The correct approach:

```yaml
      metricRelabelings:
        # Drop verbose Go GC metrics (high cardinality, low value)
        - sourceLabels: [__name__]
          regex: "go_gc_duration_seconds.*"
          action: drop
        # Drop Go internal scheduling metrics
        - sourceLabels: [__name__]
          regex: "go_sched_.*"
          action: drop
        # Drop promhttp internal metrics
        - sourceLabels: [__name__]
          regex: "promhttp_.*"
          action: drop
        # Add environment label
        - targetLabel: environment
          replacement: "PLACEHOLDER_ENV"
```

### `gitops-repo/services/payment-service/base/analysistemplate.yaml`

```yaml
###############################################################################
# Canary Analysis Template
#
# Runs during each canary analysis step.
# Queries Prometheus for error rate and latency.
# If either degrades beyond threshold, canary is auto-aborted and rolled back.
#
# KEY DESIGN CHOICES:
# 1. Use Linkerd metrics (request_total, response_latency_ms) — always present
#    App-level metrics are also checked but Linkerd metrics are the safety net
# 2. Compare canary vs stable (relative) rather than absolute thresholds
#    This handles baseline shifts (e.g., if stable error rate is already 0.5%)
# 3. Low initial interval — payment errors are expensive, detect fast
# 4. failureLimit:1 — one failure = abort. No second chances for payment traffic.
###############################################################################

apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: payment-canary-analysis
  namespace: payments
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  args:
    - name: service-name
  metrics:
    # --- Error Rate: Canary error rate must be <= stable + 0.5% ---
    - name: error-rate
      interval: 30s
      # Wait before first measurement — let canary warm up
      initialDelay: 60s
      failureLimit: 1
      successCondition: "result[0] <= 0.005"  # Canary error rate <= 0.5% absolute
      failureCondition: "result[0] > 0.01"    # Abort immediately if > 1%
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.observability.svc.cluster.local:9090
          query: |
            sum(
              rate(
                response_total{
                  deployment="{{ args.service-name }}",
                  namespace="payments",
                  classification="failure"
                }[2m]
              )
            )
            /
            sum(
              rate(
                response_total{
                  deployment="{{ args.service-name }}",
                  namespace="payments"
                }[2m]
              )
            )

    # --- Latency: Canary p99 must be < 500ms (SLO target) ---
    - name: latency-p99
      interval: 30s
      initialDelay: 60s
      failureLimit: 1
      successCondition: "result[0] < 500"
      failureCondition: "result[0] > 750"     # Abort if p99 > 750ms
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.observability.svc.cluster.local:9090
          query: |
            histogram_quantile(0.99,
              sum(
                rate(
                  response_latency_ms_bucket{
                    deployment="{{ args.service-name }}",
                    namespace="payments"
                  }[2m]
                )
              ) by (le)
            )

    # --- App-level: Payment success rate ---
    - name: payment-success-rate
      interval: 60s
      initialDelay: 120s       # Payment transactions need more time to accumulate
      failureLimit: 2          # Slightly more lenient — app metric might have low volume
      # Inconclusive if no data (low traffic) — don't block on missing data
      inconclusiveLimit: 3
      successCondition: "result[0] >= 0.99 || isNaN(result[0])"  # NaN = no data = ok
      failureCondition: "result[0] < 0.95"
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.observability.svc.cluster.local:9090
          query: |
            sum(
              rate(
                payment_transactions_total{
                  namespace="payments",
                  pod=~".*canary.*",
                  status="success"
                }[5m]
              )
            )
            /
            sum(
              rate(
                payment_transactions_total{
                  namespace="payments",
                  pod=~".*canary.*"
                }[5m]
              )
            )

    # --- Saturation: No connection pool exhaustion ---
    - name: db-connection-saturation
      interval: 60s
      initialDelay: 60s
      failureLimit: 2
      successCondition: "result[0] < 0.9"    # < 90% pool utilization
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.observability.svc.cluster.local:9090
          query: |
            max(
              db_pool_active_connections{
                namespace="payments",
                pod=~".*canary.*"
              }
              /
              db_pool_max_connections{
                namespace="payments",
                pod=~".*canary.*"
              }
            )
```

### `gitops-repo/services/payment-service/base/payment-audit-config.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-audit-config
  namespace: payments
data:
  # PCI-DSS Requirement 10: Track and monitor all access to cardholder data
  audit-policy.yaml: |
    # Defines which operations require audit logging
    # Application reads this config and emits structured audit log lines to stdout
    # Fluent Bit routes lines with "audit":true to S3 (Object Lock) + CloudWatch

    audit_events:
      # Payment operations
      - action: payment.create
        log_fields: [trace_id, user_id, amount_cents, currency, payment_method_type, outcome]
        # NEVER log: card_number, cvv, full_pan, security_code
        mask_fields: [card_number, expiry]
        retention_days: 2555    # 7 years (PCI + financial regulations)

      - action: payment.refund
        log_fields: [trace_id, user_id, original_payment_id, refund_amount, reason, outcome]
        retention_days: 2555

      - action: payment.void
        log_fields: [trace_id, user_id, payment_id, reason, outcome]
        retention_days: 2555

      # Sensitive data access
      - action: card.tokenize
        log_fields: [trace_id, user_id, card_last_four, card_brand, outcome]
        mask_fields: [card_number, cvv, expiry]
        retention_days: 2555

      - action: card.detokenize
        log_fields: [trace_id, user_id, token_id, reason, outcome]
        # Detokenization is high-risk — always alert
        alert_on_failure: true
        alert_on_success_rate_drop: true
        retention_days: 2555

      # Webhook processing
      - action: webhook.stripe.received
        log_fields: [trace_id, event_type, event_id, outcome]
        retention_days: 365

      - action: webhook.stripe.verified
        log_fields: [trace_id, event_type, event_id, signature_valid, outcome]
        retention_days: 365

      # Configuration changes
      - action: config.gateway.switch
        log_fields: [trace_id, user_id, from_provider, to_provider, reason]
        alert_on_event: true
        retention_days: 2555

    # Log format: every audit line MUST contain these fields
    required_fields:
      - timestamp           # RFC3339 with nanoseconds
      - trace_id            # OpenTelemetry trace ID for correlation
      - span_id             # OpenTelemetry span ID
      - audit               # boolean true — Fluent Bit routes on this
      - service             # "payment-service"
      - environment         # dev/staging/production
      - action              # from audit_events above
      - user_id             # who triggered the action (or "system" for webhooks)
      - source_ip           # client IP (from X-Forwarded-For)
      - outcome             # "success" or "failure"
      - duration_ms         # how long the operation took
      - pod_name            # which pod processed this
      - node_name           # which node

    # Example audit log line (JSON):
    # {
    #   "timestamp": "2024-01-15T10:30:45.123456789Z",
    #   "level": "info",
    #   "audit": true,
    #   "service": "payment-service",
    #   "environment": "production",
    #   "trace_id": "abc123def456",
    #   "span_id": "789ghi012",
    #   "action": "payment.create",
    #   "user_id": "user-12345",
    #   "source_ip": "203.0.113.42",
    #   "amount_cents": 9999,
    #   "currency": "USD",
    #   "payment_method_type": "card",
    #   "card_last_four": "4242",
    #   "outcome": "success",
    #   "duration_ms": 342,
    #   "pod_name": "payment-service-6b8f9c-xk2jl",
    #   "node_name": "ip-10-0-21-45.ec2.internal"
    # }
```

### `gitops-repo/services/payment-service/base/grafana-dashboard-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-service-dashboard
  namespace: observability
  labels:
    # Grafana sidecar discovers dashboards with this label
    grafana_dashboard: "1"
  annotations:
    argocd.argoproj.io/sync-wave: "1"
    grafana_folder: "Services/Payments"
data:
  payment-service.json: |
    {
      "annotations": {
        "list": [
          {
            "datasource": "Prometheus",
            "enable": true,
            "expr": "ALERTS{namespace=\"payments\", alertstate=\"firing\"}",
            "name": "Alerts",
            "tagKeys": "alertname",
            "type": "tags"
          },
          {
            "datasource": "Loki",
            "enable": true,
            "expr": "{namespace=\"payments\", app=\"payment-service\"} |= \"deployment\"",
            "name": "Deployments",
            "type": "tags"
          }
        ]
      },
      "templating": {
        "list": [
          {
            "name": "cluster",
            "type": "query",
            "datasource": "Prometheus",
            "query": "label_values(up{namespace=\"payments\"}, cluster)",
            "current": { "text": "All", "value": "$__all" },
            "includeAll": true
          },
          {
            "name": "environment",
            "type": "query",
            "datasource": "Prometheus",
            "query": "label_values(payment_transactions_total{namespace=\"payments\"}, environment)",
            "current": { "text": "production", "value": "production" }
          }
        ]
      },
      "panels": [
        {
          "_comment": "=== ROW 1: SLO OVERVIEW ===",
          "title": "SLO Status",
          "type": "row",
          "gridPos": { "h": 1, "w": 24, "x": 0, "y": 0 }
        },
        {
          "title": "Availability SLO (target: 99.95%)",
          "type": "gauge",
          "gridPos": { "h": 6, "w": 6, "x": 0, "y": 1 },
          "targets": [
            {
              "expr": "payment_sli:availability:ratio_rate30d{environment=~\"$environment\"}",
              "legendFormat": "30d availability"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "thresholds": {
                "steps": [
                  { "color": "red", "value": 0 },
                  { "color": "orange", "value": 0.999 },
                  { "color": "green", "value": 0.9995 }
                ]
              },
              "unit": "percentunit",
              "min": 0.99,
              "max": 1
            }
          }
        },
        {
          "title": "Latency SLO (target: p99 < 500ms)",
          "type": "gauge",
          "gridPos": { "h": 6, "w": 6, "x": 6, "y": 1 },
          "targets": [
            {
              "expr": "payment_sli:latency:ratio_rate30d{environment=~\"$environment\"}",
              "legendFormat": "30d latency compliance"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "thresholds": {
                "steps": [
                  { "color": "red", "value": 0 },
                  { "color": "orange", "value": 0.999 },
                  { "color": "green", "value": 0.9995 }
                ]
              },
              "unit": "percentunit",
              "min": 0.99,
              "max": 1
            }
          }
        },
        {
          "title": "Error Budget Remaining (Availability)",
          "type": "stat",
          "gridPos": { "h": 6, "w": 6, "x": 12, "y": 1 },
          "targets": [
            {
              "expr": "payment_error_budget:availability:remaining{environment=~\"$environment\"}",
              "legendFormat": "remaining"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "thresholds": {
                "steps": [
                  { "color": "red", "value": 0 },
                  { "color": "orange", "value": 0.25 },
                  { "color": "green", "value": 0.50 }
                ]
              },
              "unit": "percentunit"
            }
          }
        },
        {
          "title": "Burn Rate (1h)",
          "type": "stat",
          "gridPos": { "h": 6, "w": 6, "x": 18, "y": 1 },
          "targets": [
            {
              "expr": "payment_burn_rate:availability:1h{environment=~\"$environment\"}",
              "legendFormat": "1h burn rate"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "thresholds": {
                "steps": [
                  { "color": "green", "value": 0 },
                  { "color": "orange", "value": 1 },
                  { "color": "red", "value": 10 }
                ]
              }
            }
          }
        },
        {
          "_comment": "=== ROW 2: FOUR GOLDEN SIGNALS ===",
          "title": "Four Golden Signals",
          "type": "row",
          "gridPos": { "h": 1, "w": 24, "x": 0, "y": 7 }
        },
        {
          "title": "Request Rate (RPS)",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 },
          "targets": [
            {
              "expr": "sum(rate(http_server_request_duration_seconds_count{namespace=\"payments\", environment=~\"$environment\"}[5m]))",
              "legendFormat": "Total RPS"
            },
            {
              "expr": "sum(rate(http_server_request_duration_seconds_count{namespace=\"payments\", environment=~\"$environment\"}[5m])) by (http_route)",
              "legendFormat": "{{ http_route }}"
            }
          ]
        },
        {
          "title": "Error Rate (%)",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 8 },
          "targets": [
            {
              "expr": "sum(rate(http_server_request_duration_seconds_count{namespace=\"payments\", http_status_code=~\"5..\", environment=~\"$environment\"}[5m])) / sum(rate(http_server_request_duration_seconds_count{namespace=\"payments\", environment=~\"$environment\"}[5m])) * 100",
              "legendFormat": "5xx Error Rate %"
            },
            {
              "expr": "sum(rate(payment_transactions_total{namespace=\"payments\", status=\"failure\", environment=~\"$environment\"}[5m])) / sum(rate(payment_transactions_total{namespace=\"payments\", environment=~\"$environment\"}[5m])) * 100",
              "legendFormat": "Payment Failure Rate %"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "custom": {
                "thresholdsStyle": { "mode": "line" }
              },
              "thresholds": {
                "steps": [
                  { "color": "green", "value": 0 },
                  { "color": "red", "value": 0.5 }
                ]
              }
            }
          }
        },
        {
          "title": "Latency (p50, p95, p99)",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 16 },
          "targets": [
            {
              "expr": "histogram_quantile(0.50, sum(rate(http_server_request_duration_seconds_bucket{namespace=\"payments\", environment=~\"$environment\"}[5m])) by (le))",
              "legendFormat": "p50"
            },
            {
              "expr": "histogram_quantile(0.95, sum(rate(http_server_request_duration_seconds_bucket{namespace=\"payments\", environment=~\"$environment\"}[5m])) by (le))",
              "legendFormat": "p95"
            },
            {
              "expr": "histogram_quantile(0.99, sum(rate(http_server_request_duration_seconds_bucket{namespace=\"payments\", environment=~\"$environment\"}[5m])) by (le))",
              "legendFormat": "p99"
            }
          ],
          "fieldConfig": {
            "defaults": {
              "unit": "s",
              "custom": {
                "thresholdsStyle": { "mode": "line" }
              },
              "thresholds": {
                "steps": [
                  { "color": "green", "value": 0 },
                  { "color": "red", "value": 0.5 }
                ]
              }
            }
          }
        },
        {
          "title": "Saturation — DB Connection Pool",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 16 },
          "targets": [
            {
              "expr": "avg(db_pool_active_connections{namespace=\"payments\", environment=~\"$environment\"}) by (pod)",
              "legendFormat": "Active: {{ pod }}"
            },
            {
              "expr": "avg(db_pool_idle_connections{namespace=\"payments\", environment=~\"$environment\"}) by (pod)",
              "legendFormat": "Idle: {{ pod }}"
            },
            {
              "expr": "avg(db_pool_max_connections{namespace=\"payments\", environment=~\"$environment\"}) by (pod)",
              "legendFormat": "Max: {{ pod }}"
            }
          ]
        },
        {
          "_comment": "=== ROW 3: PAYMENT BUSINESS METRICS ===",
          "title": "Payment Business Metrics",
          "type": "row",
          "gridPos": { "h": 1, "w": 24, "x": 0, "y": 24 }
        },
        {
          "title": "Payment Transactions by Status",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 25 },
          "targets": [
            {
              "expr": "sum(rate(payment_transactions_total{namespace=\"payments\", environment=~\"$environment\"}[5m])) by (status)",
              "legendFormat": "{{ status }}"
            }
          ]
        },
        {
          "title": "Payment Transactions by Provider",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 25 },
          "targets": [
            {
              "expr": "sum(rate(payment_transactions_total{namespace=\"payments\", environment=~\"$environment\"}[5m])) by (provider)",
              "legendFormat": "{{ provider }}"
            }
          ]
        },
        {
          "title": "Gateway Latency by Provider (p99)",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 33 },
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum(rate(payment_gateway_duration_seconds_bucket{namespace=\"payments\", environment=~\"$environment\"}[5m])) by (le, provider))",
              "legendFormat": "p99: {{ provider }}"
            }
          ],
          "fieldConfig": { "defaults": { "unit": "s" } }
        },
        {
          "title": "Idempotency Cache Hit Rate",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 33 },
          "targets": [
            {
              "expr": "sum(rate(redis_idempotency_hits_total{namespace=\"payments\", environment=~\"$environment\"}[5m])) / (sum(rate(redis_idempotency_hits_total{namespace=\"payments\", environment=~\"$environment\"}[5m])) + sum(rate(redis_idempotency_misses_total{namespace=\"payments\", environment=~\"$environment\"}[5m])))",
              "legendFormat": "Hit Rate"
            }
          ],
          "fieldConfig": { "defaults": { "unit": "percentunit" } }
        },
        {
          "_comment": "=== ROW 4: INFRASTRUCTURE ===",
          "title": "Infrastructure",
          "type": "row",
          "gridPos": { "h": 1, "w": 24, "x": 0, "y": 41 }
        },
        {
          "title": "Pod CPU Usage",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 8, "x": 0, "y": 42 },
          "targets": [
            {
              "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"payments\", container=\"payment-service\"}[5m])) by (pod)",
              "legendFormat": "{{ pod }}"
            }
          ],
          "fieldConfig": { "defaults": { "unit": "short" } }
        },
        {
          "title": "Pod Memory Usage",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 8, "x": 8, "y": 42 },
          "targets": [
            {
              "expr": "container_memory_working_set_bytes{namespace=\"payments\", container=\"payment-service\"} / 1024 / 1024",
              "legendFormat": "{{ pod }}"
            }
          ],
          "fieldConfig": { "defaults": { "unit": "decmbytes" } }
        },
        {
          "title": "SQS Queue Depth + DLQ",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 8, "x": 16, "y": 42 },
          "targets": [
            {
              "expr": "sqs_queue_visible_messages{queue_name=~\".*order-events$\"}",
              "legendFormat": "Queue Depth"
            },
            {
              "expr": "sqs_queue_visible_messages{queue_name=~\".*order-events-dlq$\"}",
              "legendFormat": "DLQ Depth"
            }
          ]
        },
        {
          "title": "Redis Connection Pool + Latency",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 0, "y": 50 },
          "targets": [
            {
              "expr": "sum(redis_pool_active_connections{namespace=\"payments\", environment=~\"$environment\"}) by (pod)",
              "legendFormat": "Active: {{ pod }}"
            },
            {
              "expr": "histogram_quantile(0.99, sum(rate(redis_command_duration_seconds_bucket{namespace=\"payments\", environment=~\"$environment\"}[5m])) by (le))",
              "legendFormat": "p99 latency"
            }
          ]
        },
        {
          "title": "HPA Replicas vs Desired",
          "type": "timeseries",
          "gridPos": { "h": 8, "w": 12, "x": 12, "y": 50 },
          "targets": [
            {
              "expr": "kube_horizontalpodautoscaler_status_current_replicas{namespace=\"payments\", horizontalpodautoscaler=\"payment-service\"}",
              "legendFormat": "Current"
            },
            {
              "expr": "kube_horizontalpodautoscaler_status_desired_replicas{namespace=\"payments\", horizontalpodautoscaler=\"payment-service\"}",
              "legendFormat": "Desired"
            },
            {
              "expr": "kube_horizontalpodautoscaler_spec_max_replicas{namespace=\"payments\", horizontalpodautoscaler=\"payment-service\"}",
              "legendFormat": "Max"
            }
          ]
        },
        {
          "_comment": "=== ROW 5: LOGS + TRACES LINKS ===",
          "title": "Quick Links",
          "type": "row",
          "gridPos": { "h": 1, "w": 24, "x": 0, "y": 58 }
        },
        {
          "title": "Recent Errors (Loki)",
          "type": "logs",
          "gridPos": { "h": 8, "w": 24, "x": 0, "y": 59 },
          "targets": [
            {
              "datasource": "Loki",
              "expr": "{namespace=\"payments\", app=\"payment-service\"} | json | level=\"error\" | line_format \"{{.timestamp}} [{{.trace_id}}] {{.message}} {{.error}}\""
            }
          ]
        }
      ],
      "title": "Payment Service",
      "uid": "payment-service-dashboard",
      "version": 1,
      "time": { "from": "now-1h", "to": "now" },
      "refresh": "30s"
    }
```

### `gitops-repo/platform/prometheus/rules/payment-service-slos.yaml`

```yaml
###############################################################################
# Payment Service SLO Recording Rules + Burn Rate Alerts
#
# SLO Targets:
#   Availability: 99.95% (error budget: 0.05% = 21.6 minutes/30 days)
#   Latency: 99.95% of requests < 500ms
#
# Implementation: Multi-window multi-burn-rate alerting (Google SRE book)
###############################################################################

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: payment-service-slos
  namespace: observability
  labels:
    release: kube-prometheus-stack
    app: payment-service
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  groups:
    # ===================================================================
    # SLI Recording Rules — compute good/total event rates
    # ===================================================================
    - name: payment-service-sli-recordings
      interval: 30s
      rules:
        # --- Availability SLI: proportion of non-5xx responses ---
        # Total requests
        - record: payment_sli:http_requests:rate5m
          expr: |
            sum(rate(http_server_request_duration_seconds_count{
              namespace="payments",
              job="payment-service"
            }[5m]))

        # Good requests (non-5xx)
        - record: payment_sli:http_requests_good:rate5m
          expr: |
            sum(rate(http_server_request_duration_seconds_count{
              namespace="payments",
              job="payment-service",
              http_status_code!~"5.."
            }[5m]))

        # --- Latency SLI: proportion of requests faster than 500ms ---
        - record: payment_sli:http_requests_fast:rate5m
          expr: |
            sum(rate(http_server_request_duration_seconds_bucket{
              namespace="payments",
              job="payment-service",
              le="0.5"
            }[5m]))

        # --- Current SLI ratios ---
        - record: payment_sli:availability:ratio_rate5m
          expr: |
            payment_sli:http_requests_good:rate5m
            /
            payment_sli:http_requests:rate5m

        - record: payment_sli:latency:ratio_rate5m
          expr: |
            payment_sli:http_requests_fast:rate5m
            /
            payment_sli:http_requests:rate5m

    # ===================================================================
    # Burn Rate Recording Rules — multi-window
    # ===================================================================
    - name: payment-service-burn-rate-recordings
      interval: 30s
      rules:
        # --- Availability burn rates ---
        # 5m window
        - record: payment_burn_rate:availability:5m
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service", http_status_code!~"5.."
              }[5m]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[5m]))
            )
            /
            (1 - 0.9995)
          # burn_rate = error_rate / error_budget
          # error_budget = 1 - 0.9995 = 0.0005

        # 30m window
        - record: payment_burn_rate:availability:30m
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service", http_status_code!~"5.."
              }[30m]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[30m]))
            )
            /
            (1 - 0.9995)

        # 1h window
        - record: payment_burn_rate:availability:1h
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service", http_status_code!~"5.."
              }[1h]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[1h]))
            )
            /
            (1 - 0.9995)

        # 6h window
        - record: payment_burn_rate:availability:6h
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service", http_status_code!~"5.."
              }[6h]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[6h]))
            )
            /
            (1 - 0.9995)

        # 3d window
        - record: payment_burn_rate:availability:3d
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service", http_status_code!~"5.."
              }[3d]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[3d]))
            )
            /
            (1 - 0.9995)

        # --- Latency burn rates ---
        - record: payment_burn_rate:latency:5m
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_bucket{
                namespace="payments", job="payment-service", le="0.5"
              }[5m]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[5m]))
            )
            /
            (1 - 0.9995)

        - record: payment_burn_rate:latency:30m
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_bucket{
                namespace="payments", job="payment-service", le="0.5"
              }[30m]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[30m]))
            )
            /
            (1 - 0.9995)

        - record: payment_burn_rate:latency:1h
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_bucket{
                namespace="payments", job="payment-service", le="0.5"
              }[1h]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[1h]))
            )
            /
            (1 - 0.9995)

        - record: payment_burn_rate:latency:6h
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_bucket{
                namespace="payments", job="payment-service", le="0.5"
              }[6h]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[6h]))
            )
            /
            (1 - 0.9995)

        - record: payment_burn_rate:latency:3d
          expr: |
            1 - (
              sum(rate(http_server_request_duration_seconds_bucket{
                namespace="payments", job="payment-service", le="0.5"
              }[3d]))
              /
              sum(rate(http_server_request_duration_seconds_count{
                namespace="payments", job="payment-service"
              }[3d]))
            )
            /
            (1 - 0.9995)

        # --- Error budget remaining (30-day window) ---
        - record: payment_error_budget:availability:remaining
          expr: |
            1 - (
              (1 - (
                sum(increase(http_server_request_duration_seconds_count{
                  namespace="payments", job="payment-service", http_status_code!~"5.."
                }[30d]))
                /
                sum(increase(http_server_request_duration_seconds_count{
                  namespace="payments", job="payment-service"
                }[30d]))
              ))
              /
              (1 - 0.9995)
            )

        - record: payment_error_budget:latency:remaining
          expr: |
            1 - (
              (1 - (
                sum(increase(http_server_request_duration_seconds_bucket{
                  namespace="payments", job="payment-service", le="0.5"
                }[30d]))
                /
                sum(increase(http_server_request_duration_seconds_count{
                  namespace="payments", job="payment-service"
                }[30d]))
              ))
              /
              (1 - 0.9995)
            )

        # --- 30d SLI for dashboard gauges ---
        - record: payment_sli:availability:ratio_rate30d
          expr: |
            sum(increase(http_server_request_duration_seconds_count{
              namespace="payments", job="payment-service", http_status_code!~"5.."
            }[30d]))
            /
            sum(increase(http_server_request_duration_seconds_count{
              namespace="payments", job="payment-service"
            }[30d]))

        - record: payment_sli:latency:ratio_rate30d
          expr: |
            sum(increase(http_server_request_duration_seconds_bucket{
              namespace="payments", job="payment-service", le="0.5"
            }[30d]))
            /
            sum(increase(http_server_request_duration_seconds_count{
              namespace="payments", job="payment-service"
            }[30d]))

    # ===================================================================
    # Burn Rate Alerts — Multi-window
    # ===================================================================
    - name: payment-service-slo-alerts
      rules:
        # ---------------------------------------------------------------
        # AVAILABILITY SLO ALERTS
        # ---------------------------------------------------------------

        # SEV1: 14.4x burn rate → exhausts budget in 2 hours
        # Short window: 5m (detect fast), Long window: 1h (confirm sustained)
        - alert: PaymentAvailabilityBudgetCritical
          expr: |
            payment_burn_rate:availability:5m > 14.4
            and
            payment_burn_rate:availability:1h > 14.4
          for: 2m
          labels:
            severity: critical
            service: payment-service
            slo: availability
            team: payments
          annotations:
            summary: "Payment availability SLO burning at {{ $value | printf \"%.1f\" }}x — budget exhausted in ~2h"
            description: |
              Current burn rate: {{ $value | printf "%.1f" }}x
              At this rate, 30-day error budget will be exhausted in approximately 2 hours.
              Error budget remaining: {{ with printf `payment_error_budget:availability:remaining` | query }}{{ . | first | value | printf "%.1f%%" }}{{ end }}
            runbook_url: "https://runbooks.novamart.internal/payments/availability-budget-critical"
            dashboard_url: "https://grafana.novamart.internal/d/payment-service-dashboard?orgId=1&viewPanel=1"

        # SEV2: 6x burn rate → exhausts budget in 5 hours
        # Short window: 30m, Long window: 6h
        - alert: PaymentAvailabilityBudgetHigh
          expr: |
            payment_burn_rate:availability:30m > 6
            and
            payment_burn_rate:availability:6h > 6
          for: 5m
          labels:
            severity: warning
            service: payment-service
            slo: availability
            team: payments
          annotations:
            summary: "Payment availability SLO burn rate elevated: {{ $value | printf \"%.1f\" }}x"
            description: |
              Current burn rate: {{ $value | printf "%.1f" }}x
              At this rate, 30-day error budget will be exhausted in approximately 5 hours.
            runbook_url: "https://runbooks.novamart.internal/payments/availability-budget-high"
            dashboard_url: "https://grafana.novamart.internal/d/payment-service-dashboard?orgId=1&viewPanel=1"

        # SEV3: 1x burn rate → exactly consuming budget at expected rate
        # Long window only: 6h and 3d (slow bleed)
        - alert: PaymentAvailabilityBudgetSlow
          expr: |
            payment_burn_rate:availability:6h > 1
            and
            payment_burn_rate:availability:3d > 1
          for: 30m
          labels:
            severity: info
            service: payment-service
            slo: availability
            team: payments
          annotations:
            summary: "Payment availability error budget slowly depleting"
            description: |
              Burn rate: {{ $value | printf "%.1f" }}x over 3 days.
              Not urgent but will exhaust budget before end of window if not addressed.
            runbook_url: "https://runbooks.novamart.internal/payments/availability-budget-slow"

        # ---------------------------------------------------------------
        # LATENCY SLO ALERTS
        # ---------------------------------------------------------------

        # SEV1: Fast burn
        - alert: PaymentLatencyBudgetCritical
          expr: |
            payment_burn_rate:latency:5m > 14.4
            and
            payment_burn_rate:latency:1h > 14.4
          for: 2m
          labels:
            severity: critical
            service: payment-service
            slo: latency
            team: payments
          annotations:
            summary: "Payment latency SLO critical — p99 significantly above 500ms"
            description: |
              Latency burn rate: {{ $value | printf "%.1f" }}x
              More than 0.05% of requests are exceeding the 500ms SLO target.
              Check: DB connection pool saturation, external gateway latency, pod resource pressure.
            runbook_url: "https://runbooks.novamart.internal/payments/latency-budget-critical"
            dashboard_url: "https://grafana.novamart.internal/d/payment-service-dashboard?orgId=1&viewPanel=5"

        # SEV2: Sustained elevated burn
        - alert: PaymentLatencyBudgetHigh
          expr: |
            payment_burn_rate:latency:30m > 6
            and
            payment_burn_rate:latency:6h > 6
          for: 5m
          labels:
            severity: warning
            service: payment-service
            slo: latency
            team: payments
          annotations:
            summary: "Payment latency SLO burn rate elevated: {{ $value | printf \"%.1f\" }}x"
            runbook_url: "https://runbooks.novamart.internal/payments/latency-budget-high"

        # ---------------------------------------------------------------
        # OPERATIONAL ALERTS (non-SLO but critical for payments)
        # ---------------------------------------------------------------

        - alert: PaymentDLQNotEmpty
          expr: |
            sqs_queue_visible_messages{queue_name=~".*order-events-dlq.*"} > 0
          for: 5m
          labels:
            severity: warning
            service: payment-service
            team: payments
          annotations:
            summary: "Payment DLQ has {{ $value }} messages — failed payment events"
            runbook_url: "https://runbooks.novamart.internal/payments/dlq-not-empty"

        - alert: PaymentDBConnectionPoolExhausted
          expr: |
            max(
              db_pool_active_connections{namespace="payments"}
              /
              db_pool_max_connections{namespace="payments"}
            ) > 0.9
          for: 2m
          labels:
            severity: critical
            service: payment-service
            team: payments
          annotations:
            summary: "Payment DB connection pool > 90% utilized"
            description: |
              Pod {{ $labels.pod }} has {{ $value | printf "%.0f%%" }} of connections in use.
              If HPA scales further, connections will be exhausted.
              Current pod count × max_conns_per_pod must not exceed RDS Proxy limit.
            runbook_url: "https://runbooks.novamart.internal/payments/db-pool-exhaustion"

        - alert: PaymentRedisDown
          expr: |
            redis_pool_active_connections{namespace="payments"} == 0
            and
            up{namespace="payments", job="payment-service"} == 1
          for: 1m
          labels:
            severity: critical
            service: payment-service
            team: payments
          annotations:
            summary: "Payment service has zero Redis connections — idempotency cache offline"
            description: |
              Without Redis, duplicate payments are possible.
              Check Redis connectivity, AUTH token, TLS configuration.
            runbook_url: "https://runbooks.novamart.internal/payments/redis-down"

        - alert: PaymentGatewayHighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(payment_gateway_duration_seconds_bucket{namespace="payments"}[5m])) by (le, provider)
            ) > 10
          for: 5m
          labels:
            severity: warning
            service: payment-service
            team: payments
          annotations:
            summary: "Payment gateway {{ $labels.provider }} p99 latency > 10s"
            description: |
              External payment gateway is slow. This affects user checkout experience.
              Check: egress proxy health, Stripe/Adyen status page, network connectivity.
            runbook_url: "https://runbooks.novamart.internal/payments/gateway-slow"

        - alert: PaymentHighRefundRate
          expr: |
            sum(rate(payment_transactions_total{namespace="payments", status="refund"}[1h]))
            /
            sum(rate(payment_transactions_total{namespace="payments", status="success"}[1h]))
            > 0.05
          for: 30m
          labels:
            severity: warning
            service: payment-service
            team: payments
          annotations:
            summary: "Refund rate > 5% over last hour — possible fraud or system issue"
            runbook_url: "https://runbooks.novamart.internal/payments/high-refund-rate"
```

---

## 4. ENVIRONMENT OVERLAYS

### `gitops-repo/services/payment-service/overlays/dev/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: payments

resources:
  - ../../base

patches:
  # --- Scale down for dev ---
  - target:
      kind: Rollout
      name: payment-service
    patch: |
      - op: replace
        path: /spec/replicas
        value: 1
      - op: replace
        path: /spec/strategy/canary/steps
        # Skip canary in dev — direct deploy
        value:
          - setWeight: 100

  - target:
      kind: HorizontalPodAutoscaler
      name: payment-service
    patch: |
      - op: replace
        path: /spec/minReplicas
        value: 1
      - op: replace
        path: /spec/maxReplicas
        value: 3

  # --- Smaller resources for dev ---
  - target:
      kind: Rollout
      name: payment-service
    patch: |
      - op: replace
        path: /spec/template/spec/containers/0/resources
        value:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            memory: 256Mi

  # --- Relaxed NetworkPolicy for dev (allow all ingress for debugging) ---
  - target:
      kind: NetworkPolicy
      name: payment-service-ingress
    patch: |
      - op: replace
        path: /spec/ingress
        value:
          - from:
              - namespaceSelector: {}
            ports:
              - port: 8080
                protocol: TCP
              - port: 9090
                protocol: TCP

  # --- Debug logging in dev ---
  - target:
      kind: ConfigMap
      name: payment-service-config
    patch: |
      - op: replace
        path: /data/LOG_LEVEL
        value: "debug"
      - op: replace
        path: /data/ENVIRONMENT
        value: "dev"
      - op: replace
        path: /data/SQS_QUEUE_URL
        value: "https://sqs.us-east-1.amazonaws.com/ACCOUNT_ID/novamart-dev-order-events"
      - op: replace
        path: /data/OTEL_TRACES_SAMPLER_ARG
        value: "1.0"
      - op: replace
        path: /data/OTEL_RESOURCE_ATTRIBUTES
        value: "service.namespace=payments,deployment.environment=dev"

  # --- No PDB in dev (single replica) ---
  - target:
      kind: PodDisruptionBudget
      name: payment-service
    patch: |
      $patch: delete

# IRSA annotations per environment
replacements:
  - source:
      kind: ConfigMap
      name: payment-env-config
      fieldPath: data.IRSA_ROLE_ARN
    targets:
      - select:
          kind: ServiceAccount
          name: payment-service
        fieldPaths:
          - metadata.annotations.[eks.amazonaws.com/role-arn]

configMapGenerator:
  - name: payment-env-config
    behavior: create
    literals:
      - IRSA_ROLE_ARN=arn:aws:iam::ACCOUNT_ID:role/novamart-dev-payment-service
      - MIGRATE_ROLE_ARN=arn:aws:iam::ACCOUNT_ID:role/novamart-dev-payment-migrate

images:
  - name: PLACEHOLDER_ECR_REGISTRY/novamart/payment-service
    newName: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/novamart/payment-service
    newTag: latest  # Dev uses latest — overwritten by CI
```

### `gitops-repo/services/payment-service/overlays/staging/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: payments

resources:
  - ../../base

patches:
  # --- Production-like but smaller scale ---
  - target:
      kind: Rollout
      name: payment-service
    patch: |
      - op: replace
        path: /spec/replicas
        value: 2
      # Faster canary in staging (still validates, but shorter pauses)
      - op: replace
        path: /spec/strategy/canary/steps
        value:
          - setWeight: 20
          - pause: { duration: 1m }
          - analysis:
              templates:
                - templateName: payment-canary-analysis
              args:
                - name: service-name
                  value: payment-service-canary
          - setWeight: 50
          - pause: { duration: 2m }
          - analysis:
              templates:
                - templateName: payment-canary-analysis
              args:
                - name: service-name
                  value: payment-service-canary
          - setWeight: 100

  - target:
      kind: HorizontalPodAutoscaler
      name: payment-service
    patch: |
      - op: replace
        path: /spec/minReplicas
        value: 2
      - op: replace
        path: /spec/maxReplicas
        value: 6

  - target:
      kind: ConfigMap
      name: payment-service-config
    patch: |
      - op: replace
        path: /data/LOG_LEVEL
        value: "info"
      - op: replace
        path: /data/ENVIRONMENT
        value: "staging"
      - op: replace
        path: /data/SQS_QUEUE_URL
        value: "https://sqs.us-east-1.amazonaws.com/ACCOUNT_ID/novamart-staging-order-events"
      - op: replace
        path: /data/OTEL_TRACES_SAMPLER_ARG
        value: "1.0"
      - op: replace
        path: /data/OTEL_RESOURCE_ATTRIBUTES
        value: "service.namespace=payments,deployment.environment=staging"

images:
  - name: PLACEHOLDER_ECR_REGISTRY/novamart/payment-service
    newName: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/novamart/payment-service
    newTag: PLACEHOLDER  # Set by CI pipeline
```

### `gitops-repo/services/payment-service/overlays/production/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: payments

resources:
  - ../../base

patches:
  # --- Full production scale ---
  - target:
      kind: Rollout
      name: payment-service
    patch: |
      - op: replace
        path: /spec/replicas
        value: 5

  # Full conservative canary (as defined in base — no override needed)

  - target:
      kind: HorizontalPodAutoscaler
      name: payment-service
    patch: |
      - op: replace
        path: /spec/minReplicas
        value: 5
      - op: replace
        path: /spec/maxReplicas
        value: 20

  # --- Production resources ---
  - target:
      kind: Rollout
      name: payment-service
    patch: |
      - op: replace
        path: /spec/template/spec/containers/0/resources
        value:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            memory: 1Gi

  - target:
      kind: ConfigMap
      name: payment-service-config
    patch: |
      - op: replace
        path: /data/LOG_LEVEL
        value: "info"
      - op: replace
        path: /data/ENVIRONMENT
        value: "production"
      - op: replace
        path: /data/SQS_QUEUE_URL
        value: "https://sqs.us-east-1.amazonaws.com/ACCOUNT_ID/novamart-production-order-events"
      # Production: 100% at head, tail sampling at gateway decides
      - op: replace
        path: /data/OTEL_TRACES_SAMPLER_ARG
        value: "1.0"
      - op: replace
        path: /data/OTEL_RESOURCE_ATTRIBUTES
        value: "service.namespace=payments,deployment.environment=production"
      # Tighter DB limits in production
      - op: replace
        path: /data/DB_MAX_OPEN_CONNS
        value: "10"
      - op: replace
        path: /data/DB_MAX_IDLE_CONNS
        value: "5"

images:
  - name: PLACEHOLDER_ECR_REGISTRY/novamart/payment-service
    newName: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/novamart/payment-service
    newTag: PLACEHOLDER  # Set by CI pipeline — NEVER latest in production
```

---

## 5. ARGOCD APPLICATION MANIFESTS

### `gitops-repo/argocd/applications/payment-service-dev.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-dev
  namespace: argocd
  labels:
    app: payment-service
    environment: dev
    team: payments
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: payments-deploys
    notifications.argoproj.io/subscribe.on-sync-failed.slack: payments-alerts
    notifications.argoproj.io/subscribe.on-health-degraded.slack: payments-alerts
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: payments   # AppProject with restricted permissions

  source:
    repoURL: https://bitbucket.org/novamart/gitops-repo.git
    targetRevision: main
    path: services/payment-service/overlays/dev

  destination:
    server: https://kubernetes.default.svc
    namespace: payments

  syncPolicy:
    automated:
      prune: true        # Remove resources deleted from Git
      selfHeal: true     # Revert manual changes
      allowEmpty: false   # Don't delete everything if path is empty
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground  # Wait for dependents to delete
      - PruneLast=true                     # Prune after all syncs complete
      - ApplyOutOfSyncOnly=true            # Only sync changed resources
      - ServerSideApply=true               # Better handling of large resources
      - RespectIgnoreDifferences=true
    retry:
      limit: 3
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:
    # HPA updates replicas — don't fight it
    - group: argoproj.io
      kind: Rollout
      jsonPointers:
        - /spec/replicas
    # Webhook CA bundle gets injected
    - group: admissionregistration.k8s.io
      kind: MutatingWebhookConfiguration
      jsonPointers:
        - /webhooks/0/clientConfig/caBundle
```

### `gitops-repo/argocd/applications/payment-service-staging.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-staging
  namespace: argocd
  labels:
    app: payment-service
    environment: staging
    team: payments
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: payments-deploys
    notifications.argoproj.io/subscribe.on-sync-failed.slack: payments-alerts
    notifications.argoproj.io/subscribe.on-health-degraded.slack: payments-alerts
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: payments

  source:
    repoURL: https://bitbucket.org/novamart/gitops-repo.git
    targetRevision: main
    path: services/payment-service/overlays/staging

  destination:
    server: https://kubernetes.default.svc
    namespace: payments

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
      - ServerSideApply=true
      - RespectIgnoreDifferences=true
    retry:
      limit: 3
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:
    - group: argoproj.io
      kind: Rollout
      jsonPointers:
        - /spec/replicas
```

### `gitops-repo/argocd/applications/payment-service-production.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-production
  namespace: argocd
  labels:
    app: payment-service
    environment: production
    team: payments
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: payments-deploys-prod
    notifications.argoproj.io/subscribe.on-sync-failed.slack: payments-alerts-critical
    notifications.argoproj.io/subscribe.on-health-degraded.slack: payments-alerts-critical
    notifications.argoproj.io/subscribe.on-sync-status-unknown.slack: payments-alerts-critical
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: payments

  source:
    repoURL: https://bitbucket.org/novamart/gitops-repo.git
    targetRevision: main
    path: services/payment-service/overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: payments

  syncPolicy:
    # PRODUCTION: NO automated sync
    # Sync is triggered manually by ArgoCD after Jenkins pipeline approval gate
    automated: null
    syncOptions:
      - CreateNamespace=false        # Namespace must pre-exist in production
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
      - ServerSideApply=true
      - RespectIgnoreDifferences=true
    retry:
      limit: 2                       # Fewer retries in production — fail fast, investigate
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 5m

  ignoreDifferences:
    - group: argoproj.io
      kind: Rollout
      jsonPointers:
        - /spec/replicas
```

### `gitops-repo/argocd/projects/payments.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments
  namespace: argocd
spec:
  description: "Payment service team — PCI-scoped applications"

  # Only these repos can be sources
  sourceRepos:
    - https://bitbucket.org/novamart/gitops-repo.git

  # Only deploy to payments namespace
  destinations:
    - namespace: payments
      server: https://kubernetes.default.svc
    # Dashboard goes to observability namespace
    - namespace: observability
      server: https://kubernetes.default.svc

  # Restrict what resources the team can create
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace

  namespaceResourceWhitelist:
    - group: ""
      kind: "*"
    - group: apps
      kind: "*"
    - group: argoproj.io
      kind: "*"
    - group: autoscaling
      kind: "*"
    - group: batch
      kind: "*"
    - group: external-secrets.io
      kind: "*"
    - group: monitoring.coreos.com
      kind: "*"
    - group: networking.k8s.io
      kind: NetworkPolicy
    - group: policy
      kind: PodDisruptionBudget

  # Block dangerous resources
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota       # Platform team manages quotas
    - group: ""
      kind: LimitRange           # Platform team manages limits
    - group: rbac.authorization.k8s.io
      kind: "*"                  # No self-RBAC escalation

  roles:
    - name: payments-team
      description: "Payments team members"
      policies:
        - p, proj:payments:payments-team, applications, get, payments/*, allow
        - p, proj:payments:payments-team, applications, sync, payments/*, allow
        - p, proj:payments:payments-team, applications, action/*, payments/*, allow
        - p, proj:payments:payments-team, logs, get, payments/*, allow
        # Deny delete in production — only platform team can delete
        - p, proj:payments:payments-team, applications, delete, payments/payment-service-production, deny
      groups:
        - payments-engineers   # OIDC group from SSO
```

---

## 6. FLUENT BIT — AUDIT LOG ROUTING

### `gitops-repo/platform/fluent-bit/filters/payment-audit.yaml`

```yaml
# Added to Fluent Bit ConfigMap (platform layer)
# Routes payment audit logs to S3 with Object Lock for PCI compliance

apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-payment-audit-filter
  namespace: observability
data:
  payment-audit.conf: |
    # ============================================================
    # FILTER: Parse JSON logs from payment-service
    # ============================================================
    [FILTER]
        Name         kubernetes
        Match        kube.payments.*
        Kube_Tag_Prefix  kube.payments.
        Merge_Log    On
        Keep_Log     Off
        K8S-Logging.Parser  On

    # ============================================================
    # FILTER: Route audit logs to separate output
    # Uses "rewrite_tag" to fork audit logs to their own stream
    # ============================================================
    [FILTER]
        Name          rewrite_tag
        Match         kube.payments.*
        Rule          $audit ^(true)$ audit.payments.${tag} false
        # 'false' = keep original record in main stream too
        # Audit logs go to BOTH Loki (searchable) AND S3 (immutable archive)

    # ============================================================
    # OUTPUT: Audit logs to S3 (Object Lock bucket — immutable)
    # ============================================================
    [OUTPUT]
        Name                         s3
        Match                        audit.payments.*
        bucket                       novamart-production-compliance-audit
        region                       us-east-1
        s3_key_format                /payments/year=%Y/month=%m/day=%d/hour=%H/$UUID.json.gz
        s3_key_format_tag_delimiters ._
        total_file_size              50M
        upload_timeout               5m
        use_put_object               On
        compression                  gzip
        content_type                 application/gzip
        store_dir                    /tmp/fluent-bit-s3-audit
        # S3 Object Lock is configured at bucket level (done in L1 Terraform)
        # Objects are immutable once written — 7-year retention for PCI

    # ============================================================
    # OUTPUT: Audit logs also to CloudWatch (hot search, 90-day retention)
    # ============================================================
    [OUTPUT]
        Name                 cloudwatch_logs
        Match                audit.payments.*
        region               us-east-1
        log_group_name       /novamart/production/audit/payments
        log_stream_prefix    payment-service-
        auto_create_group    On
        log_retention_days   90
```

---

## 7. DESIGN DECISIONS

### `docs/design-decisions/DD-APP-001-rds-proxy.md`

```markdown
# DD-APP-001: RDS Proxy for Payment Service

## Status: Accepted
## Date: 2024-01-15

## Context
Payment-service connects to RDS PostgreSQL. In Kubernetes, pod churn is frequent
(deployments, HPA scaling, node replacement). Each new pod establishing a new DB 
connection is expensive (TLS handshake + auth + connection setup ≈ 50-200ms).
With HPA scaling 5→20 pods × 10 connections each = 200 connections rapidly.

## Decision
Use AWS RDS Proxy between pods and RDS.

## Rationale
1. **Connection pooling**: RDS Proxy multiplexes many app connections over fewer DB connections
2. **IAM Auth**: Pods authenticate via IRSA → no DB passwords in pods/secrets
   - RDS Proxy handles the actual DB auth using Secrets Manager
   - Even if a pod is compromised, attacker gets a scoped IAM token, not a DB password
3. **Credential rotation transparency**: When Secrets Manager rotates DB password,
   RDS Proxy picks it up. Running pods are NOT affected (they use IAM, not password)
4. **Failover handling**: On Multi-AZ failover, RDS Proxy maintains connection pool
   to new primary. App sees brief pause, not connection errors.
5. **Connection surge protection**: During HPA scale-up, RDS Proxy queues connection
   requests instead of overwhelming the DB. `connection_borrow_timeout=120s` gives
   time for the surge to settle.

## Alternatives Considered
- **Direct RDS connection**: Simpler but creates connection storms on scaling events.
  No credential rotation transparency. Password in K8s Secret.
- **PgBouncer sidecar**: Lower cost than RDS Proxy ($0.015/vCPU/hr) but adds operational
  complexity (another container to manage, monitor, upgrade). No IAM auth integration.
- **Application-level pooling only**: Works for steady state but doesn't protect
  against scaling surges or rotation.

## Cost Impact
RDS Proxy: ~$73/month (db.r6g.large equivalent). Justified by:
- Reduced connection-related errors during deployments
- Eliminates secret rotation downtime
- Reduces blast radius of credential compromise

## Connection Math
```
Production steady state:
  minReplicas=5 × DB_MAX_OPEN_CONNS=10 = 50 connections
  maxReplicas=20 × DB_MAX_OPEN_CONNS=10 = 200 connections

RDS db.r6g.large max_connections ≈ 1600
RDS Proxy max_connections_percent = 80% → 1280

200 app connections << 1280 proxy limit ✓
Proxy can multiplex 200 app connections over ~50-100 actual DB connections ✓
```
```

### `docs/design-decisions/DD-APP-002-go-otel-manual.md`

```markdown
# DD-APP-002: Manual OTel Instrumentation for Go Services

## Status: Accepted
## Date: 2024-01-15

## Context
OpenTelemetry auto-instrumentation (via OTel Operator) works by injecting agents 
that hook into runtime internals. This works well for:
- **Java**: JVMTI agent (bytecode instrumentation)
- **Python**: sys.settrace / import hooks  
- **Node.js**: require() patching

Go is fundamentally different:
- **Compiled to native binary** — no runtime to hook into
- **No bytecode** — no agent injection point
- Go auto-instrumentation via OTel Operator uses **eBPF** (experimental, alpha quality)
  - Requires privileged containers (conflicts with restricted PSA)
  - Limited library coverage
  - Production-readiness: NOT recommended (as of 2024)

## Decision
Payment-service (Go) uses **manual OTel SDK instrumentation** in application code.
No `instrumentation.opentelemetry.io/inject-go` annotation.

```markdown
## Implementation
Application code must import and use:
- `go.opentelemetry.io/otel` — core API
- `go.opentelemetry.io/otel/sdk/trace` — SDK trace provider
- `go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp` — HTTP middleware
- `go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc` — gRPC interceptors
- `go.opentelemetry.io/contrib/instrumentation/database/sql/otelsql` — DB tracing
- `go.opentelemetry.io/contrib/instrumentation/github.com/go-redis/redis/otelredis` — Redis tracing
- `go.opentelemetry.io/contrib/instrumentation/github.com/aws/aws-sdk-go-v2/otelaws` — AWS SDK tracing

The application initializes a TracerProvider at startup pointing to the OTel Agent:

```go
exporter, _ := otlptracegrpc.New(ctx,
    otlptracegrpc.WithEndpoint(os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")),
    otlptracegrpc.WithInsecure(), // Cluster-internal, Linkerd mTLS wraps it
)

tp := sdktrace.NewTracerProvider(
    sdktrace.WithBatcher(exporter),
    sdktrace.WithResource(resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceNameKey.String("payment-service"),
        semconv.DeploymentEnvironmentKey.String(os.Getenv("ENVIRONMENT")),
    )),
    sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(1.0))),
)
otel.SetTracerProvider(tp)
otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
    propagation.TraceContext{},
    propagation.Baggage{},
))
```

HTTP handlers wrapped with otelhttp middleware:
```go
handler := otelhttp.NewHandler(mux, "payment-api")
```

Every outbound call (DB, Redis, SQS, Stripe) must propagate context:
```go
// CORRECT — context flows through
result, err := db.QueryContext(ctx, "SELECT ...")

// WRONG — breaks trace chain
result, err := db.QueryContext(context.Background(), "SELECT ...")
```

## Trade-offs
- **Pro**: Full control over span names, attributes, events
- **Pro**: No privileged containers needed
- **Pro**: Production-proven, stable API
- **Con**: Requires developer discipline — every new HTTP client/DB call must pass ctx
- **Con**: Developers must add instrumentation libraries when adding new dependencies
- **Mitigation**: Code review checklist includes "context propagation verified"
- **Mitigation**: Linting rule to flag `context.Background()` in non-init code paths

## Impact on Other Services
- Java services (order-service, user-service): USE auto-instrumentation annotation
- Python services (notification-service): USE auto-instrumentation annotation
- Go services (api-gateway, payment-service): MANUAL instrumentation required
```

### `docs/design-decisions/DD-APP-003-migration-strategy.md`

```markdown
# DD-APP-003: Database Migration Strategy

## Status: Accepted
## Date: 2024-01-15

## Context
Schema changes must be applied safely in a zero-downtime deployment environment.
The database schema and application code evolve independently.

## Decision
1. Migrations run as a Kubernetes **Job** (ArgoCD PreSync hook), NOT as an init container
2. Migrations use **golang-migrate** (lightweight, Go ecosystem)
3. Migrations MUST follow the **expand-contract** pattern
4. Separate migration ServiceAccount with DDL privileges
5. Application ServiceAccount has DML only (SELECT, INSERT, UPDATE, DELETE)

## Why Job, Not Init Container

| Aspect | Init Container | Job (PreSync Hook) |
|---|---|---|
| Runs when? | Every pod start (restart, reschedule, scale-up) | Once per deployment |
| Idempotency risk | Re-runs on every pod restart — must be perfectly idempotent | Runs once, completes, done |
| Monitoring | Hidden inside pod lifecycle | Visible as independent Job, own logs |
| Failure isolation | Blocks pod startup | Fails independently, blocks ArgoCD sync |
| Rollback | Runs again if pod restarts after rollback | Doesn't re-run on rollback |
| Resource usage | Blocks pod resources during migration | Separate resource allocation |

The critical issue: **if migration runs as init container and you rollback the Rollout,
new pods start with the init container, which runs the migration FORWARD again.** 
Rollback only affects the application container image tag, not the init container behavior.

With PreSync Job:
- Migration runs once before the new ReplicaSet is created
- If canary fails and rolls back, the migration is NOT reversed
- This is CORRECT because expand-contract migrations are backward-compatible

## Expand-Contract Pattern

### Adding a column
```
Migration V1: ALTER TABLE payments ADD COLUMN new_field VARCHAR(255) DEFAULT '';
              -- DEFAULT ensures existing rows are valid
              -- Old app code ignores the column (doesn't SELECT or INSERT it)
Deploy V1:    New app code starts using new_field
              -- Both old and new pods work (old ignores it, new uses it)
```

### Removing a column
```
Deploy V1:    New app code stops reading/writing old_field
              -- Old pods during canary still work (column exists)
              -- After full rollout, no code uses old_field
Migration V2: ALTER TABLE payments DROP COLUMN old_field;
              -- Safe: no running code references it
              -- This migration ships with the NEXT release
```

### Renaming a column
```
Migration V1: ALTER TABLE payments ADD COLUMN new_name VARCHAR(255);
              UPDATE payments SET new_name = old_name;  -- Backfill
              -- Now both columns exist with same data
Deploy V1:    App reads from new_name, writes to BOTH old_name and new_name
              -- Old canary pods still work (read old_name)
Deploy V2:    App reads/writes ONLY new_name, stops touching old_name
Migration V3: ALTER TABLE payments DROP COLUMN old_name;
```

## Failed Migration Recovery

If a migration fails mid-apply:
1. golang-migrate tracks version in `schema_migrations` table
2. Failed version is marked as `dirty=true`
3. ArgoCD PreSync hook fails → sync stops → app stays on old version
4. Operator investigates, fixes migration SQL
5. To retry: `migrate -database $DSN force <version>` then re-apply
6. NEVER manually edit schema_migrations unless you understand the state

## Alternatives Considered
- **Flyway**: Industry standard, but requires JVM. Overhead for Go services.
- **Atlas (Ariga)**: Declarative schema management. More powerful but more complex.
  Good candidate for future migration if team grows.
- **Liquibase**: XML-based, heavy. Rejected.
```

### `docs/design-decisions/DD-APP-004-canary-strategy.md`

```markdown
# DD-APP-004: Conservative Canary Strategy for Payment Service

## Status: Accepted
## Date: 2024-01-15

## Context
Payment-service processes real money. A bad deployment that processes payments
incorrectly or fails silently costs real revenue AND customer trust.

## Decision
5-step canary with progressive traffic shifting:
5% → 15% → 30% → 60% → 100%
With automated Prometheus-backed analysis at each step.

## Step Design Rationale

| Step | Weight | Pause | Analysis | Rationale |
|---|---|---|---|---|
| 1 | 5% | 2m | Yes | Minimum blast radius. Catches crashes immediately. |
| 2 | 15% | 3m | Yes | Increased traffic gives analysis statistical significance |
| 3 | 30% | 5m | Yes | Meaningful traffic. If errors exist, they show here. |
| 4 | 60% | 5m | Yes | Majority traffic. Final validation before full cutover. |
| 5 | 100% | - | - | Full rollout. Monitoring continues via SLO alerts. |

Total deployment time: ~20-25 minutes (production).
For comparison: simple backend services use 10/30/60/100 with 1-2m pauses (~10 min).

## Analysis Metrics
1. **Error rate** (Linkerd metrics) — most reliable, mesh-level, always present
2. **p99 latency** (Linkerd metrics) — detect degradation before users complain
3. **Payment success rate** (app metric) — business-level validation
4. **DB connection saturation** (app metric) — catches connection leaks

## Analysis Failure Behavior
- `failureLimit: 1` for error rate and latency — ONE bad measurement = abort
- `failureLimit: 2` for payment success rate — allows one transient failure
- `inconclusiveLimit: 3` for payment success rate — handles low traffic (NaN result)

## Low Traffic Problem
In staging, payment-service might get 2 requests/minute.
Over a 2-minute analysis window, that's ~4 requests.
A single 5xx gives a 25% error rate — instant abort.

Mitigations:
- `inconclusiveLimit` allows NaN results (no data = don't block)
- Staging uses shorter canary steps (faster feedback loop)
- Staging `for:` values are shorter in AnalysisTemplate
- Load testing in staging before production promotion is part of the CI pipeline

## Rollback Behavior
If analysis fails at step 3 of 6:
1. Argo Rollouts immediately sets canary weight to 0%
2. All traffic shifts to stable ReplicaSet
3. Canary ReplicaSet scales to 0
4. Slack notification sent to `payments-alerts`
5. AnalysisRun preserved for investigation (shows which metric failed)
6. Previous stable version continues serving 100%
7. No manual intervention needed

What does NOT happen:
- Database migration is NOT rolled back (expand-contract pattern — safe)
- Secrets are NOT rolled back
- Config changes in ConfigMap ARE rolled back (they're part of the Rollout)
```

### `docs/design-decisions/DD-APP-005-connection-pool-math.md`

```markdown
# DD-APP-005: Connection Pool Sizing

## Status: Accepted
## Date: 2024-01-15

## Context
Connection pools must be sized so that at maximum HPA scale, the total connections
from all pods don't exceed the database or cache limit.

## Database Connection Math

```
RDS Instance: db.r6g.large
  max_connections (PostgreSQL default for this instance) ≈ 1600

RDS Proxy:
  max_connections_percent = 80% → 1280 effective connections
  Proxy multiplexes: 200 app connections can be served by ~50-100 actual DB connections
  connection_borrow_timeout = 120s (queuing under pressure)

Application:
  DB_MAX_OPEN_CONNS = 10 per pod
  DB_MAX_IDLE_CONNS = 5 per pod
  DB_CONN_MAX_LIFETIME = 5m (recycle connections, prevent stale)
  DB_CONN_MAX_IDLE_TIME = 1m (release idle connections quickly)

Worst case (HPA max):
  20 pods × 10 connections = 200 app connections
  RDS Proxy limit: 1280 → 200 is well within limit ✓

Headroom:
  200 / 1280 = 15.6% utilization at max scale
  This leaves room for migration job (separate pool), monitoring connections,
  and manual debugging sessions.

Alert threshold:
  DB connection pool per-pod > 90% (9 of 10) → warn
  Total connections to RDS Proxy > 500 → CloudWatch alarm
```

## Redis Connection Math

```
ElastiCache: cache.r6g.large
  max_connections ≈ 65,000

Application:
  REDIS_POOL_SIZE = 5 per pod

Worst case:
  20 pods × 5 connections = 100
  100 / 65,000 = 0.15% ← effectively unlimited headroom

  Redis is not the bottleneck. Pool size of 5 is conservative.
```

## SQS Connection Math

```
SQS is API-based (HTTPS), not connection-pooled.
AWS SDK uses HTTP connection pooling internally (default MaxIdleConns=100 per host).
VPC endpoint ensures traffic stays in-network.
No connection limit concern for SQS.
```

## Key Constraint
**If DB_MAX_OPEN_CONNS is increased, HPA maxReplicas must be re-evaluated.**

Example: if someone sets DB_MAX_OPEN_CONNS=25:
  20 pods × 25 = 500 connections → still within 1280 limit ✓
  But if maxReplicas is also increased to 50:
  50 × 25 = 1250 → dangerously close to 1280 limit ✗

**This is why DB_MAX_OPEN_CONNS is in ConfigMap (visible, reviewable) and not hardcoded.**
```

### `docs/design-decisions/DD-APP-006-secret-rotation.md`

```markdown
# DD-APP-006: Secret Rotation Without Downtime

## Status: Accepted
## Date: 2024-01-15

## Context
PCI-DSS requires credential rotation. Rotation must not cause downtime.

## Database Credentials

### Flow
1. AWS Secrets Manager rotation Lambda triggers every 30 days
2. Lambda creates new password in RDS using `ALTER ROLE payment_app PASSWORD '...'`
3. Lambda updates Secret in Secrets Manager (AWSCURRENT ← new, AWSPREVIOUS ← old)
4. RDS Proxy detects Secret change, starts using new credentials for new DB connections
5. Existing connections continue working (they're already authenticated)
6. ExternalSecret (refreshInterval: 1m) picks up new secret within 1 minute
7. New K8s Secret is created/updated
8. Running pods still use old credentials (env vars don't hot-reload)
9. On next pod restart (deployment, scaling, etc.), pods get new credentials
10. Gap between steps 2-9: pods use OLD credentials

### Why This Is Safe
- **RDS Proxy handles the gap**: Pods connect to RDS Proxy using IAM Auth.
  The password in the Secret is used by RDS Proxy internally, not by pods directly.
  When DB_USE_IAM_AUTH=true, pods authenticate to RDS Proxy via IAM token.
  RDS Proxy authenticates to RDS using the Secret.
  So step 2 only affects RDS Proxy ↔ RDS, not Pod ↔ RDS Proxy.
- **RDS Proxy rotation transparency**: RDS Proxy is designed for this.
  It reads AWSCURRENT from Secrets Manager and smoothly transitions connections.
- **No pod restart needed**: Because pods use IAM auth, the DB password rotation
  is completely transparent to the application.

### Without RDS Proxy (Why It Matters)
If pods connected directly to RDS with password auth:
1. Lambda rotates password
2. Existing connections with OLD password keep working (already authenticated)
3. NEW connections fail (pod has old password in memory from env var)
4. ExternalSecret refreshes K8s Secret (1 minute)
5. But env vars are NOT hot-reloaded — pods still have old password
6. Must restart pods (rolling restart) to pick up new password
7. Window of new connection failures: 1-3 minutes

This is why RDS Proxy + IAM Auth is critical for zero-downtime rotation.

## Stripe API Key Rotation

Stripe doesn't support automatic rotation. Process:
1. Generate new API key in Stripe Dashboard
2. Update Secret in AWS Secrets Manager (both old and new temporarily)
3. ESO syncs to K8s Secret (1 minute)
4. Rolling restart of payment-service: `kubectl rollout restart rollout/payment-service -n payments`
5. Verify new pods process payments successfully
6. Revoke old API key in Stripe Dashboard

**Enhancement opportunity**: Mount Stripe secret as volume instead of env var.
Volumes hot-reload (~60-90s). App watches file for changes. No restart needed.
Trade-off: more complex app code vs simpler rotation.

## Signing Key Rotation (Dual-Key Pattern)

The signing key is used for idempotency token signing. During rotation:
1. Update Secret: `current` = new key, `previous` = old key
2. App verifies tokens: try `current` first, fallback to `previous`
3. New tokens are signed with `current`
4. After TTL expires (24h), all tokens signed with old key have expired
5. `previous` can be cleared on next rotation

This dual-key pattern is standard for signing key rotation.

## Redis AUTH Token Rotation

ElastiCache supports dual-password during rotation:
1. Set new AUTH token with `--auth-token` and `--auth-token-update-strategy ROTATE`
2. ElastiCache accepts BOTH old and new tokens during transition window
3. Update Secret in Secrets Manager
4. ESO syncs, pods restart (or use volume mount for hot-reload)
5. After all pods have new token, finalize with `SET` strategy (old token invalidated)
```

### `docs/design-decisions/DD-APP-007-networkpolicy-design.md`

```markdown
# DD-APP-007: NetworkPolicy Design for PCI Compliance

## Status: Accepted
## Date: 2024-01-15

## Context
PCI-DSS Requirement 1: "Install and maintain network security controls"
Payment-service is in PCI scope. Network access must be explicitly allowed.

## Design

### Default Deny
First resource: deny ALL ingress and egress for the entire payments namespace.
Every connection must be explicitly allowed.

### Ingress Rules
| Source | Port | Reason |
|---|---|---|
| api-gateway (namespace + pod selector) | 8080 | Application traffic |
| Prometheus (observability namespace) | 9090 | Metrics scraping |
| Linkerd control plane (linkerd namespace) | 4143, 4191 | Mesh proxy communication |

### Egress Rules
| Destination | Port | Reason |
|---|---|---|
| CoreDNS (kube-system) | 53/UDP, 53/TCP | DNS resolution |
| NodeLocal DNSCache (169.254.20.10) | 53/UDP, 53/TCP | Optimized DNS |
| RDS Proxy (data subnet CIDRs) | 5432 | Database |
| Redis (data subnet CIDRs) | 6379 | Idempotency cache |
| SQS VPC Endpoint (private subnet CIDRs) | 443 | Order events |
| OTel Agent (observability namespace) | 4317, 4318 | Tracing |
| Squid Egress Proxy (egress-proxy namespace) | 3128 | External payment API |
| Linkerd control plane (linkerd namespace) | 8443, 8080 | Mesh control |
| STS VPC Endpoint (private subnet CIDRs) | 443 | IRSA token exchange |
| IMDS (169.254.169.254) | 80 | Instance metadata for IRSA |

### Why IP CIDRs for Data Tier
NetworkPolicy cannot select pods outside the cluster (RDS, ElastiCache, SQS endpoints
are not pods). We use `ipBlock` with data subnet CIDRs.

Risk: any pod in the data subnet CIDR range on those ports is reachable.
Mitigation: Security Groups provide the second layer (SG on RDS only allows from
EKS node SG on port 5432).

### Migration Job
Separate, tighter egress policy:
- DNS + RDS Proxy + STS + IMDS only
- No Redis, no SQS, no external API, no OTel
- Least privilege: migration job only needs database access

## Common Gotchas Avoided
1. **DNS egress missing**: Without explicit DNS egress, pods can't resolve ANY hostname.
   This is the #1 NetworkPolicy mistake.
2. **NodeLocal DNSCache**: If deployed, pods talk to 169.254.20.10, not kube-dns directly.
   Must allow BOTH for safety.
3. **IMDS for IRSA**: IRSA projected tokens require IMDS access for the credential chain.
   Blocking 169.254.169.254 breaks IRSA silently.
4. **Linkerd ports**: Without explicit Linkerd egress, the proxy can't reach the control
   plane for certificates and service discovery. mTLS breaks silently.
5. **STS endpoint**: IRSA tokens are exchanged via STS. Without STS egress, IRSA fails
   with "unable to assume role" errors that are confusing to debug.
```

### `docs/design-decisions/DD-APP-008-hpa-scaling-metric.md`

```markdown
# DD-APP-008: HPA Scaling Metric Selection

## Status: Accepted
## Date: 2024-01-15

## Context
Payment-service is I/O-bound (database queries, external API calls to Stripe/Adyen,
Redis lookups). CPU utilization is a poor scaling signal because:
- During peak load, CPU may only be at 30% while all goroutines are blocked on I/O
- By the time CPU hits 70%, the service is already failing (queue depth, timeouts)

## Decision
Primary scaling metric: **HTTP requests per second per pod** (custom metric)
Secondary: CPU utilization as safety net (if custom metrics pipeline fails)
Tertiary: Memory utilization as safety net

## Implementation
The application exposes `http_server_request_duration_seconds_count` (counter).
Prometheus scrapes it. A custom metrics adapter (prometheus-adapter or KEDA) exposes
it as a Kubernetes custom metric `http_requests_per_second`.

prometheus-adapter config:
```yaml
rules:
  - seriesQuery: 'http_server_request_duration_seconds_count{namespace="payments",job="payment-service"}'
    resources:
      overrides:
        namespace: {resource: "namespace"}
        pod: {resource: "pod"}
    name:
      matches: "^(.*)_seconds_count$"
      as: "http_requests_per_second"
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

## Target: 50 RPS per pod

Based on load testing:
- Each pod handles ~100 RPS at p99 < 500ms (SLO target)
- At 50 RPS, pods are at ~50% capacity
- This gives headroom for burst absorption while HPA provisions new pods
- HPA scale-up takes ~30s (metric detection) + ~60s (Karpenter provision if needed)
- During that 90s, existing pods handle 2x their target load (100 RPS) within SLO

## Scale-Down Protection
- stabilizationWindowSeconds: 300 (5 minutes) — payment traffic is bursty
- maxPods decrease: 2 per 120s — slow ramp-down prevents oscillation
- selectPolicy: Min — use the most conservative scale-down policy

## Alternative Considered: KEDA
KEDA could scale on SQS queue depth or custom Prometheus queries.
Rejected for now: prometheus-adapter is simpler, already deployed for other services.
KEDA is a good option if we need SQS-based scaling for consumer services later.
```

---

## 8. SERVICE ONBOARDING TEMPLATE

### `docs/service-onboarding-template.md`

```markdown
# Service Onboarding Template — NovaMart Platform

## Overview
This template guides engineering teams through onboarding a new service to the 
NovaMart platform. Follow each section in order.

**Estimated time:** 2-4 hours for a standard service, 4-8 hours for PCI-scoped services.

**Who does what:**
- **Service team**: Application code, Dockerfile, Jenkinsfile, Kustomize base/overlays
- **Platform team**: IRSA role, Terraform infra, ArgoCD Application, namespace setup
- **Security team**: Review NetworkPolicies, approve PCI scope changes

---

## Pre-Onboarding Checklist

- [ ] Service name decided (lowercase, hyphenated: `inventory-service`)
- [ ] Team name and Slack channels identified
- [ ] Language/framework known (Go, Java Spring Boot, Python)
- [ ] Dependencies mapped (databases, caches, queues, external APIs)
- [ ] SLO targets defined (availability %, latency target)
- [ ] PCI scope determined (does it touch payment/cardholder data?)
- [ ] Container image builds and passes security scan in CI
- [ ] Application exposes:
  - [ ] `/healthz` (liveness — process health only)
  - [ ] `/readyz` (readiness — dependency health)
  - [ ] `/startupz` (startup — initialization complete)
  - [ ] `/metrics` (Prometheus format on separate port)
  - [ ] Structured JSON logs to stdout
  - [ ] OTel traces (manual for Go, auto-instrumentation annotation for Java/Python/Node)

---

## Step 1: Terraform (Platform Team)

Request from Platform team via Jira (template: `PLATFORM-SERVICE-ONBOARDING`):

```
Service Name: <name>
Namespace: <namespace>
Language: <Go|Java|Python|Node>
Dependencies:
  - Database: <RDS instance name, or "none">
  - Cache: <Redis, or "none">
  - Queue: <SQS queue name, or "none">
  - External APIs: <list, or "none">
Secrets needed:
  - <list of Secrets Manager paths>
SLO targets:
  - Availability: <99.9% | 99.95% | 99.99%>
  - Latency: <p99 < Xms>
PCI scope: <yes|no>
```

Platform team creates:
1. IRSA role with least-privilege policies
2. SQS queue + DLQ (if needed)
3. RDS Proxy (if DB access needed)
4. Secrets Manager entries (bootstrap)
5. KMS grants
6. CloudWatch alarms for infrastructure components

**Output:** IRSA role ARN, SQS queue URL, RDS Proxy endpoint, Secret ARNs

---

## Step 2: GitOps Repo Structure (Service Team)

Copy from template:

```bash
cp -r gitops-repo/services/_template gitops-repo/services/<service-name>
```

Template structure:
```
services/<service-name>/
├── base/
│   ├── kustomization.yaml      # Update app name, labels
│   ├── namespace.yaml          # Update namespace name
│   ├── serviceaccount.yaml     # Update IRSA ARN placeholder
│   ├── externalsecret.yaml     # Update secret references
│   ├── configmap.yaml          # Service-specific config
│   ├── rollout.yaml            # Adjust resources, probes, env vars
│   ├── service.yaml            # Update port names if needed
│   ├── hpa.yaml                # Adjust min/max replicas, metrics
│   ├── pdb.yaml                # Usually unchanged
│   ├── networkpolicy.yaml      # CUSTOMIZE per service dependencies
│   ├── servicemonitor.yaml     # Update port/path if non-standard
│   └── analysistemplate.yaml   # Update metric queries
├── overlays/
│   ├── dev/
│   │   └── kustomization.yaml  # Dev overrides
│   ├── staging/
│   │   └── kustomization.yaml  # Staging overrides
│   └── production/
│       └── kustomization.yaml  # Production overrides
└── README.md                   # Service-specific docs
```

### Customization Guide

| File | What to Change | Common Mistakes |
|---|---|---|
| `rollout.yaml` | Image, ports, env vars, resources, probes | Wrong probe paths, missing preStop |
| `configmap.yaml` | All service-specific config | Putting secrets in ConfigMap |
| `externalsecret.yaml` | Secret references from Terraform output | Wrong secretStoreRef name |
| `networkpolicy.yaml` | Ingress sources, egress destinations | **Missing DNS egress** |
| `hpa.yaml` | Min/max replicas, scaling metric | Setting maxReplicas without connection pool math |
| `analysistemplate.yaml` | Metric names, thresholds | Using app metrics that don't exist yet |

### Language-Specific Notes

**Go services:**
- NO auto-instrumentation annotation (manual OTel SDK)
- No CPU limit (avoid CFS throttling)
- Import `automaxprocs` library for correct GOMAXPROCS
- Memory limit = 2× expected working set

**Java services:**
- ADD annotation: `instrumentation.opentelemetry.io/inject-java: "true"`
- SET CPU limit (JVM needs it for thread pool sizing)
- SET `-XX:MaxRAMPercentage=75.0` in JAVA_OPTS
- Memory limit = heap + non-heap + overhead (~1.5× heap)
- Startup probe failureThreshold higher (JVM startup is slow)

**Python services:**
- ADD annotation: `instrumentation.opentelemetry.io/inject-python: "true"`
- Consider Gunicorn workers = 2× CPU request + 1
- Memory limit = per-worker memory × workers + overhead

---

## Step 3: CI Pipeline (Service Team)

Create `Jenkinsfile` in service repo:

```groovy
@Library('novamart-shared-library') _

novamartPipeline(
    service: '<service-name>',
    language: 'go',              // or 'java', 'python'
    namespace: '<namespace>',
    team: '<team-name>',
    slackChannel: '<team>-deploys',

    // Canary strategy
    canarySteps: [10, 30, 60, 100],  // Standard service
    // canarySteps: [5, 15, 30, 60, 100],  // Payment/critical service

    // Optional overrides
    // sonarExclusions: 'vendor/**,generated/**',
    // trivySeverity: 'CRITICAL,HIGH',
    // integrationTestEnabled: true,
    // integrationTestCommand: 'make integration-test',
)
```

---

## Step 4: ArgoCD Application (Platform Team)

Platform team creates ArgoCD Application manifests:
- `argocd/applications/<service-name>-dev.yaml` (auto-sync)
- `argocd/applications/<service-name>-staging.yaml` (auto-sync)
- `argocd/applications/<service-name>-production.yaml` (manual sync)

Using the AppProject for the team (or creating one if new team).

---

## Step 5: Observability (Service Team + Platform Team)

### Service Team:
1. Implement metrics in application code:
   - HTTP request duration histogram
   - Business-specific counters (e.g., `order_created_total`)
   - Dependency health gauges (DB pool, cache connections)
2. Ensure structured JSON logs with fields:
   - `timestamp`, `level`, `message`, `trace_id`, `span_id`, `service`, `environment`
3. Implement tracing (manual for Go, auto for Java/Python)

### Platform Team:
1. Create SLO recording rules in Prometheus
2. Create burn rate alerts
3. Create Grafana dashboard (Four Golden Signals + service-specific panels)
4. Add service to on-call routing in Alertmanager

---

## Step 6: Database Migration (If Applicable)

1. Add migration files to service repo under `migrations/`
2. Use naming convention: `V001_create_initial_schema.up.sql`, `V001_create_initial_schema.down.sql`
3. Ensure all migrations are backward-compatible (expand-contract)
4. Migration Job in `base/db-migration-job.yaml` handles execution
5. Test migrations locally: `migrate -path ./migrations -database $DSN up`

---

## Step 7: Validation

After deploying to dev:

```bash
# 1. Verify pods are running
kubectl get pods -n <namespace>

# 2. Verify service is reachable
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
curl http://localhost:8080/healthz

# 3. Verify metrics are being scraped
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n observability
# Query: up{namespace="<namespace>", job="<service-name>"}

# 4. Verify traces are flowing
# Check Grafana → Explore → Tempo → service.name = "<service-name>"

# 5. Verify logs are collected
# Check Grafana → Explore → Loki → {namespace="<namespace>", app="<service-name>"}

# 6. Verify NetworkPolicy
kubectl exec -it <pod> -n <namespace> -- wget -qO- http://some-other-service.default
# Should be blocked by NetworkPolicy

# 7. Verify ArgoCD sync
argocd app get <service-name>-dev

# 8. Trigger a deployment and verify canary
# Push a change, watch ArgoCD + Argo Rollouts
kubectl argo rollouts get rollout <service-name> -n <namespace> --watch
```

---

## Step 8: Production Readiness Review

Before promoting to production, complete this checklist:

- [ ] Service passes all CI stages (build, test, scan, quality gate)
- [ ] Deployed successfully to dev and staging
- [ ] SLO dashboard shows data
- [ ] Burn rate alerts are configured and tested
- [ ] Canary analysis works (trigger a failing canary to verify auto-rollback)
- [ ] NetworkPolicy reviewed by security team (PCI services only)
- [ ] Runbook created for common failure modes
- [ ] On-call team knows the service exists and has context
- [ ] Load test completed (matches expected production traffic)
- [ ] Connection pool math documented and validated
- [ ] Secret rotation tested (rotate a secret, verify no downtime)
- [ ] Rollback tested (deploy a known-bad version, verify auto-rollback)

---

## Post-Onboarding

- Add service to team's Grafana folder
- Add service to weekly SLO review
- Schedule first on-call handoff
- Update architecture diagram
- Close PLATFORM-SERVICE-ONBOARDING Jira ticket
```

---

## 9. PAYMENT SERVICE SPECIFIC README

### `gitops-repo/services/payment-service/README.md`

```markdown
# Payment Service — Deployment Documentation

## Overview
Payment-service processes financial transactions for NovaMart.
**PCI-DSS scope**: Yes — handles payment method tokens and transaction data.

## Architecture
- **Language**: Go 1.22
- **Framework**: Custom HTTP server with chi router
- **Database**: PostgreSQL (via RDS Proxy, IAM Auth)
- **Cache**: Redis (ElastiCache, TLS + AUTH)
- **Queue**: SQS (order-events)
- **External**: Stripe API (via egress proxy)
- **Mesh**: Linkerd (automatic mTLS)
- **Tracing**: Manual OTel SDK (Go doesn't support auto-instrumentation)

## SLO Targets
| SLO | Target | Error Budget (30d) |
|---|---|---|
| Availability | 99.95% | 21.6 minutes |
| Latency (p99) | < 500ms | 0.05% of requests |

## Deployment
Deployments are automated via Jenkins → ArgoCD pipeline.
- **Dev**: Auto-sync, no canary
- **Staging**: Auto-sync, abbreviated canary (20/50/100)
- **Production**: Manual sync, full canary (5/15/30/60/100), ~25 min total

### Manual Production Deployment
```bash
# After Jenkins pipeline completes staging validation:
argocd app sync payment-service-production

# Watch rollout:
kubectl argo rollouts get rollout payment-service -n payments --watch

# Emergency abort:
kubectl argo rollouts abort payment-service -n payments
```

## Connection Pool Limits

| Resource | Per Pod | Max Pods (HPA) | Total Max | Backend Limit | Utilization |
|---|---|---|---|---|---|
| PostgreSQL | 10 | 20 | 200 | 1280 (Proxy) | 15.6% |
| Redis | 5 | 20 | 100 | 65,000 | 0.15% |

**WARNING**: Changing DB_MAX_OPEN_CONNS or HPA maxReplicas requires recalculating this table.

## Secrets

| Secret | Source | Rotation | Impact on Pods |
|---|---|---|---|
| DB credentials | Secrets Manager | 30 days (auto) | Transparent (IAM Auth) |
| Stripe API key | Secrets Manager | Manual | Requires pod restart |
| Signing key | Secrets Manager | Manual | Dual-key pattern, no downtime |
| Redis AUTH | Secrets Manager | Manual | Dual-password, no downtime |

## Database Migrations
- Tool: golang-migrate
- Location: `migrations/` in payment-service repo
- Execution: ArgoCD PreSync Job (runs before app deployment)
- Pattern: expand-contract (always backward-compatible)
- Migration user: `payment_migrate` (DDL privileges)
- Application user: `payment_app` (DML only)

## Monitoring

### Dashboards
- [Payment Service Dashboard](https://grafana.novamart.internal/d/payment-service-dashboard)
- [Payment Infrastructure](https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:name=novamart-production-payment-infra)

### Key Alerts
| Alert | Severity | Action |
|---|---|---|
| PaymentAvailabilityBudgetCritical | critical | Page on-call, check error logs |
| PaymentLatencyBudgetCritical | critical | Check DB pool, external gateway |
| PaymentDBConnectionPoolExhausted | critical | Check HPA, possible connection leak |
| PaymentRedisDown | critical | Check Redis, possible network/auth issue |
| PaymentDLQNotEmpty | warning | Check failed events, possible consumer bug |
| PaymentGatewayHighLatency | warning | Check Stripe status, egress proxy |
| PaymentHighRefundRate | warning | Possible fraud or system issue |

### Runbooks
- [Availability Budget Critical](https://runbooks.novamart.internal/payments/availability-budget-critical)
- [Latency Budget Critical](https://runbooks.novamart.internal/payments/latency-budget-critical)
- [DB Pool Exhaustion](https://runbooks.novamart.internal/payments/db-pool-exhaustion)
- [Redis Down](https://runbooks.novamart.internal/payments/redis-down)
- [DLQ Not Empty](https://runbooks.novamart.internal/payments/dlq-not-empty)
- [Gateway Slow](https://runbooks.novamart.internal/payments/gateway-slow)

## Troubleshooting Quick Reference

### Payment processing failures
```bash
# Check recent errors
kubectl logs -l app=payment-service -n payments --tail=100 | jq 'select(.level=="error")'

# Check trace for specific transaction
# In Grafana → Explore → Tempo → { resource.service.name = "payment-service" && status = error }

# Check Stripe connectivity
kubectl exec -it deploy/payment-service -n payments -- wget -qO- --timeout=5 http://squid-egress-proxy.egress-proxy:3128
```

### Pod not starting
```bash
# Check events
kubectl describe pod -l app=payment-service -n payments

# Check init container (dependency check)
kubectl logs -l app=payment-service -n payments -c wait-for-dependencies

# Check ExternalSecrets
kubectl get externalsecret -n payments
kubectl describe externalsecret payment-db-credentials -n payments
```

### Canary stuck or failing
```bash
# Check rollout status
kubectl argo rollouts get rollout payment-service -n payments

# Check analysis run
kubectl argo rollouts list analysisruns -n payments
kubectl describe analysisrun <name> -n payments

# Manual abort
kubectl argo rollouts abort payment-service -n payments

# Manual promote (DANGER — skips remaining analysis)
kubectl argo rollouts promote payment-service -n payments
```
```

---

## 10. VALIDATION SCRIPT

### `scripts/validate-payment-service.sh`

```bash
#!/usr/bin/env bash
###############################################################################
# Payment Service Onboarding Validation Script
#
# Verifies all components are correctly deployed and connected.
# Run after deploying to any environment.
#
# Usage: ./scripts/validate-payment-service.sh <environment>
# Example: ./scripts/validate-payment-service.sh dev
###############################################################################

set -euo pipefail

ENV="${1:?Usage: $0 <dev|staging|production>}"
NAMESPACE="payments"
SERVICE="payment-service"
PASS=0
FAIL=0
WARN=0

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

check_pass() { echo -e "  ${GREEN}✓${NC} $1"; ((PASS++)); }
check_fail() { echo -e "  ${RED}✗${NC} $1"; ((FAIL++)); }
check_warn() { echo -e "  ${YELLOW}!${NC} $1"; ((WARN++)); }

echo "============================================="
echo " Payment Service Validation — ${ENV}"
echo "============================================="
echo ""

# --- 1. Namespace ---
echo "▸ Namespace"
if kubectl get namespace "$NAMESPACE" &>/dev/null; then
  check_pass "Namespace ${NAMESPACE} exists"
  
  # Check labels
  INJECT=$(kubectl get ns "$NAMESPACE" -o jsonpath='{.metadata.labels.linkerd\.io/inject}')
  if [ "$INJECT" = "enabled" ]; then
    check_pass "Linkerd injection enabled"
  else
    check_fail "Linkerd injection NOT enabled (label: linkerd.io/inject)"
  fi

  PSA=$(kubectl get ns "$NAMESPACE" -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}')
  if [ "$PSA" = "restricted" ]; then
    check_pass "Pod Security Standards: restricted"
  else
    check_fail "Pod Security Standards NOT restricted (got: ${PSA:-none})"
  fi
else
  check_fail "Namespace ${NAMESPACE} does not exist"
fi
echo ""

# --- 2. ServiceAccount + IRSA ---
echo "▸ ServiceAccount + IRSA"
if kubectl get sa "$SERVICE" -n "$NAMESPACE" &>/dev/null; then
  check_pass "ServiceAccount ${SERVICE} exists"
  
  ROLE_ARN=$(kubectl get sa "$SERVICE" -n "$NAMESPACE" -o jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}')
  if [ -n "$ROLE_ARN" ]; then
    check_pass "IRSA annotation present: ${ROLE_ARN}"
  else
    check_fail "IRSA annotation MISSING on ServiceAccount"
  fi
else
  check_fail "ServiceAccount ${SERVICE} does not exist"
fi

if kubectl get sa "payment-migrate" -n "$NAMESPACE" &>/dev/null; then
  check_pass "Migration ServiceAccount exists"
else
  check_fail "Migration ServiceAccount payment-migrate does not exist"
fi
echo ""

# --- 3. ExternalSecrets ---
echo "▸ ExternalSecrets"
for ES in payment-db-credentials payment-db-migrate-credentials payment-stripe-credentials payment-signing-key payment-redis-credentials; do
  if kubectl get externalsecret "$ES" -n "$NAMESPACE" &>/dev/null; then
    STATUS=$(kubectl get externalsecret "$ES" -n "$NAMESPACE" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
    if [ "$STATUS" = "True" ]; then
      check_pass "ExternalSecret ${ES}: Ready"
    else
      REASON=$(kubectl get externalsecret "$ES" -n "$NAMESPACE" -o jsonpath='{.status.conditions[?(@.type=="Ready")].reason}')
      check_fail "ExternalSecret ${ES}: NOT Ready (${REASON})"
    fi
  else
    check_fail "ExternalSecret ${ES} does not exist"
  fi
done
echo ""

# --- 4. Rollout ---
echo "▸ Rollout"
if kubectl get rollout "$SERVICE" -n "$NAMESPACE" &>/dev/null; then
  check_pass "Rollout ${SERVICE} exists"
  
  PHASE=$(kubectl get rollout "$SERVICE" -n "$NAMESPACE" -o jsonpath='{.status.phase}')
  if [ "$PHASE" = "Healthy" ]; then
    check_pass "Rollout phase: Healthy"
  else
    check_warn "Rollout phase: ${PHASE} (expected: Healthy)"
  fi

  READY=$(kubectl get rollout "$SERVICE" -n "$NAMESPACE" -o jsonpath='{.status.readyReplicas}')
  DESIRED=$(kubectl get rollout "$SERVICE" -n "$NAMESPACE" -o jsonpath='{.spec.replicas}')
  if [ "${READY:-0}" -ge "${DESIRED:-1}" ]; then
    check_pass "Replicas: ${READY}/${DESIRED} ready"
  else
    check_fail "Replicas: ${READY:-0}/${DESIRED} ready (not all pods ready)"
  fi
else
  check_fail "Rollout ${SERVICE} does not exist"
fi
echo ""

# --- 5. Pods ---
echo "▸ Pods"
RUNNING_PODS=$(kubectl get pods -n "$NAMESPACE" -l app="$SERVICE" --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
if [ "$RUNNING_PODS" -gt 0 ]; then
  check_pass "${RUNNING_PODS} pod(s) running"
  
  # Check Linkerd sidecar
  POD_NAME=$(kubectl get pods -n "$NAMESPACE" -l app="$SERVICE" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')
  CONTAINERS=$(kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o jsonpath='{.spec.containers[*].name}')
  if echo "$CONTAINERS" | grep -q "linkerd-proxy"; then
    check_pass "Linkerd sidecar injected"
  else
    check_fail "Linkerd sidecar NOT injected"
  fi

  # Check security context
  RUN_AS=$(kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o jsonpath='{.spec.securityContext.runAsNonRoot}')
  if [ "$RUN_AS" = "true" ]; then
    check_pass "runAsNonRoot: true"
  else
    check_fail "runAsNonRoot NOT set"
  fi

  # Check readOnlyRootFilesystem
  READONLY=$(kubectl get pod "$POD_NAME" -n "$NAMESPACE" -o jsonpath='{.spec.containers[?(@.name=="payment-service")].securityContext.readOnlyRootFilesystem}')
  if [ "$READONLY" = "true" ]; then
    check_pass "readOnlyRootFilesystem: true"
  else
    check_fail "readOnlyRootFilesystem NOT set"
  fi
else
  check_fail "No running pods found"
fi
echo ""

# --- 6. Services ---
echo "▸ Services"
for SVC in "$SERVICE" "${SERVICE}-stable" "${SERVICE}-canary"; do
  if kubectl get svc "$SVC" -n "$NAMESPACE" &>/dev/null; then
    check_pass "Service ${SVC} exists"
  else
    check_fail "Service ${SVC} does not exist"
  fi
done
echo ""

# --- 7. HPA ---
echo "▸ HPA"
if kubectl get hpa "$SERVICE" -n "$NAMESPACE" &>/dev/null; then
  check_pass "HPA exists"
  MIN=$(kubectl get hpa "$SERVICE" -n "$NAMESPACE" -o jsonpath='{.spec.minReplicas}')
  MAX=$(kubectl get hpa "$SERVICE" -n "$NAMESPACE" -o jsonpath='{.spec.maxReplicas}')
  check_pass "HPA range: ${MIN}-${MAX} replicas"
  
  # Connection pool math validation
  DB_CONNS=$(kubectl get configmap "${SERVICE}-config" -n "$NAMESPACE" -o jsonpath='{.data.DB_MAX_OPEN_CONNS}' 2>/dev/null || echo "unknown")
  if [ "$DB_CONNS" != "unknown" ]; then
    TOTAL=$((MAX * DB_CONNS))
    if [ "$TOTAL" -le 1280 ]; then
      check_pass "Connection pool math: ${MAX} pods × ${DB_CONNS} conns = ${TOTAL} (limit: 1280)"
    else
      check_fail "Connection pool math: ${MAX} pods × ${DB_CONNS} conns = ${TOTAL} EXCEEDS limit 1280"
    fi
  fi
else
  check_fail "HPA does not exist"
fi
echo ""

# --- 8. PDB ---
echo "▸ PDB"
if [ "$ENV" = "dev" ]; then
  check_warn "PDB not expected in dev (single replica)"
else
  if kubectl get pdb "$SERVICE" -n "$NAMESPACE" &>/dev/null; then
    check_pass "PDB exists"
  else
    check_fail "PDB does not exist (required for ${ENV})"
  fi
fi
echo ""

# --- 9. NetworkPolicies ---
echo "▸ NetworkPolicies"
NP_COUNT=$(kubectl get networkpolicy -n "$NAMESPACE" --no-headers 2>/dev/null | wc -l)
if [ "$NP_COUNT" -ge 3 ]; then
  check_pass "${NP_COUNT} NetworkPolicies found"
  
  # Check default deny exists
  if kubectl get networkpolicy "default-deny-all" -n "$NAMESPACE" &>/dev/null; then
    check_pass "Default deny policy exists"
  else
    check_fail "Default deny policy MISSING — PCI violation"
  fi
else
  check_fail "Expected at least 3 NetworkPolicies, found ${NP_COUNT}"
fi
echo ""

# --- 10. ServiceMonitor ---
echo "▸ Observability"
if kubectl get servicemonitor "$SERVICE" -n "$NAMESPACE" &>/dev/null; then
  check_pass "ServiceMonitor exists"
else
  check_fail "ServiceMonitor does not exist"
fi

# Check Prometheus scraping
if command -v curl &>/dev/null; then
  PROM_URL="http://localhost:9090"
  # Try port-forward check if accessible
  UP_RESULT=$(kubectl exec -n observability deploy/kube-prometheus-stack-prometheus -c prometheus -- \
    wget -qO- "http://localhost:9090/api/v1/query?query=up{namespace=\"${NAMESPACE}\",job=\"${SERVICE}\"}" 2>/dev/null || echo "")
  if echo "$UP_RESULT" | grep -q '"value":\[.*,"1"\]'; then
    check_pass "Prometheus scraping active (up=1)"
  else
    check_warn "Cannot verify Prometheus scraping (check manually)"
  fi
fi

# Check Grafana dashboard ConfigMap
if kubectl get configmap "${SERVICE}-dashboard" -n observability &>/dev/null; then
  check_pass "Grafana dashboard ConfigMap exists"
else
  check_fail "Grafana dashboard ConfigMap does not exist"
fi

# Check PrometheusRule (SLOs)
if kubectl get prometheusrule "${SERVICE}-slos" -n observability &>/dev/null; then
  check_pass "SLO PrometheusRule exists"
else
  check_fail "SLO PrometheusRule does not exist"
fi
echo ""

# --- 11. AnalysisTemplate ---
echo "▸ Canary Analysis"
if kubectl get analysistemplate "payment-canary-analysis" -n "$NAMESPACE" &>/dev/null; then
  check_pass "AnalysisTemplate exists"
else
  check_fail "AnalysisTemplate does not exist"
fi
echo ""

# --- 12. ArgoCD Application ---
echo "▸ ArgoCD"
if command -v argocd &>/dev/null || kubectl get application "${SERVICE}-${ENV}" -n argocd &>/dev/null 2>&1; then
  if kubectl get application "${SERVICE}-${ENV}" -n argocd &>/dev/null; then
    check_pass "ArgoCD Application ${SERVICE}-${ENV} exists"
    
    SYNC=$(kubectl get application "${SERVICE}-${ENV}" -n argocd -o jsonpath='{.status.sync.status}')
    HEALTH=$(kubectl get application "${SERVICE}-${ENV}" -n argocd -o jsonpath='{.status.health.status}')
    
    if [ "$SYNC" = "Synced" ]; then
      check_pass "ArgoCD sync status: Synced"
    else
      check_warn "ArgoCD sync status: ${SYNC}"
    fi
    
    if [ "$HEALTH" = "Healthy" ]; then
      check_pass "ArgoCD health: Healthy"
    else
      check_warn "ArgoCD health: ${HEALTH}"
    fi
  else
    check_fail "ArgoCD Application ${SERVICE}-${ENV} does not exist"
  fi
else
  check_warn "Cannot verify ArgoCD (CLI not available and no API access)"
fi
echo ""

# --- 13. Probes (verify endpoint responds) ---
echo "▸ Health Endpoints"
if [ "$RUNNING_PODS" -gt 0 ]; then
  POD_NAME=$(kubectl get pods -n "$NAMESPACE" -l app="$SERVICE" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')
  
  for ENDPOINT in healthz readyz startupz metrics; do
    PORT=8080
    [ "$ENDPOINT" = "metrics" ] && PORT=9090
    
    HTTP_CODE=$(kubectl exec "$POD_NAME" -n "$NAMESPACE" -c "$SERVICE" -- \
      wget -qO/dev/null -S "http://localhost:${PORT}/${ENDPOINT}" 2>&1 | grep "HTTP/" | awk '{print $2}' || echo "000")
    
    if [ "$HTTP_CODE" = "200" ]; then
      check_pass "/${ENDPOINT} responds 200"
    else
      check_fail "/${ENDPOINT} responds ${HTTP_CODE} (expected 200)"
    fi
  done
else
  check_warn "Cannot verify health endpoints (no running pods)"
fi
echo ""

# --- 14. DNS Resolution Test ---
echo "▸ DNS Resolution"
if [ "$RUNNING_PODS" -gt 0 ]; then
  POD_NAME=$(kubectl get pods -n "$NAMESPACE" -l app="$SERVICE" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')
  
  for HOST in "kube-dns.kube-system.svc.cluster.local" "otel-agent.observability.svc.cluster.local"; do
    if kubectl exec "$POD_NAME" -n "$NAMESPACE" -c "$SERVICE" -- \
      nslookup "$HOST" >/dev/null 2>&1; then
      check_pass "DNS resolves: ${HOST}"
    else
      check_fail "DNS FAILS for: ${HOST} (check NetworkPolicy DNS egress)"
    fi
  done
fi
echo ""

# --- Summary ---
echo "============================================="
echo " RESULTS"
echo "============================================="
echo -e " ${GREEN}Passed: ${PASS}${NC}"
echo -e " ${RED}Failed: ${FAIL}${NC}"
echo -e " ${YELLOW}Warnings: ${WARN}${NC}"
echo ""

if [ "$FAIL" -gt 0 ]; then
  echo -e "${RED}VALIDATION FAILED — ${FAIL} checks failed${NC}"
  exit 1
elif [ "$WARN" -gt 0 ]; then
  echo -e "${YELLOW}VALIDATION PASSED WITH WARNINGS — review manually${NC}"
  exit 0
else
  echo -e "${GREEN}VALIDATION PASSED — all checks green${NC}"
  exit 0
fi
```

---

## 11. COMPLETE FILE TREE

```
terraform/environments/production/applications/payment-service/
├── main.tf                     # IRSA, SQS, RDS Proxy, Secrets Manager, CloudWatch
├── variables.tf                # All input variables with validation
└── outputs.tf                  # ARNs and endpoints for downstream use

gitops-repo/
├── services/
│   └── payment-service/
│       ├── base/
│       │   ├── kustomization.yaml
│       │   ├── namespace.yaml
│       │   ├── serviceaccount.yaml          # App SA + Migrate SA (separate IRSA)
│       │   ├── externalsecret.yaml          # 5 ExternalSecrets (DB, DB-migrate, Stripe, Signing, Redis)
│       │   ├── configmap.yaml               # Non-sensitive config (ports, pool sizes, OTel, proxy)
│       │   ├── payment-audit-config.yaml    # PCI audit event definitions
│       │   ├── rollout.yaml                 # Argo Rollout with canary, probes, security
│       │   ├── service.yaml                 # Primary + Stable + Canary services
│       │   ├── hpa.yaml                     # RPS-based + CPU/Memory safety nets
│       │   ├── pdb.yaml                     # maxUnavailable:1
│       │   ├── networkpolicy.yaml           # Default deny + explicit ingress/egress
│       │   ├── db-migration-job.yaml        # PreSync Job with golang-migrate
│       │   ├── servicemonitor.yaml          # Prometheus scraping config
│       │   ├── analysistemplate.yaml        # Canary analysis (error rate, latency, business)
│       │   └── grafana-dashboard-configmap.yaml  # Four Golden Signals + SLO + Business
│       ├── overlays/
│       │   ├── dev/
│       │   │   └── kustomization.yaml       # 1 replica, debug logging, relaxed network, no PDB
│       │   ├── staging/
│       │   │   └── kustomization.yaml       # 2 replicas, abbreviated canary
│       │   └── production/
│       │       └── kustomization.yaml       # 5 replicas, full canary, production resources
│       └── README.md                        # Service deployment documentation
├── argocd/
│   ├── applications/
│   │   ├── payment-service-dev.yaml         # Auto-sync
│   │   ├── payment-service-staging.yaml     # Auto-sync
│   │   └── payment-service-production.yaml  # Manual sync
│   └── projects/
│       └── payments.yaml                    # AppProject with RBAC
└── platform/
    ├── fluent-bit/filters/
    │   └── payment-audit.yaml               # Audit log routing to S3 + CloudWatch
    └── prometheus/rules/
        └── payment-service-slos.yaml        # SLI recordings + burn rate alerts

docs/
├── design-decisions/
│   ├── DD-APP-001-rds-proxy.md              # Why RDS Proxy, connection math
│   ├── DD-APP-002-go-otel-manual.md         # Why manual OTel for Go
│   ├── DD-APP-003-migration-strategy.md     # Why Job not init, expand-contract
│   ├── DD-APP-004-canary-strategy.md        # Step design, analysis failure behavior
│   ├── DD-APP-005-connection-pool-math.md   # Full pool sizing calculations
│   ├── DD-APP-006-secret-rotation.md        # Rotation strategies per secret type
│   ├── DD-APP-007-networkpolicy-design.md   # PCI network isolation, common gotchas
│   └── DD-APP-008-hpa-scaling-metric.md     # Why RPS not CPU for I/O-bound services
├── payment-service-onboarding.md            # What was done for payment-service
└── service-onboarding-template.md           # Reusable template for any new service

scripts/
└── validate-payment-service.sh              # Automated validation (14 check categories)
```

---

## 12. TRAP RESPONSE SUMMARY

Let me explicitly address every trap mentioned in the assignment:

### Trap 1: Go + OTel Auto-Instrumentation ✅
**Handled in:** `rollout.yaml` (explicit comment block explaining why annotation is ABSENT), `DD-APP-002`

Go compiles to native binary. No JVM/runtime to hook into. OTel Operator auto-instrumentation for Go uses eBPF (alpha, requires privileged containers, conflicts with restricted PSA). Payment-service requires **manual OTel SDK instrumentation** in application code. The `instrumentation.opentelemetry.io/inject-go` annotation is intentionally absent with a documented reason.

### Trap 2: Connection Pool Math ✅
**Handled in:** `DD-APP-005`, `configmap.yaml`, `hpa.yaml`, `validate-payment-service.sh`

```
HPA max (20) × DB_MAX_OPEN_CONNS (10) = 200 connections
RDS Proxy limit: 80% of 1600 = 1280
200 / 1280 = 15.6% utilization ✓

HPA max (20) × REDIS_POOL_SIZE (5) = 100 connections
ElastiCache limit: 65,000
100 / 65,000 = 0.15% ✓
```

Validation script checks this math automatically. README documents the constraint.

### Trap 3: NetworkPolicy + DNS ✅
**Handled in:** `networkpolicy.yaml` (explicit DNS egress rules for both CoreDNS AND NodeLocal DNSCache), `DD-APP-007`

Both CoreDNS (kube-system) and NodeLocal DNSCache (169.254.20.10/32) have explicit egress rules. Also includes IMDS (169.254.169.254) for IRSA and STS VPC endpoint for token exchange.

### Trap 4: Canary Analysis Window in Low Traffic ✅
**Handled in:** `analysistemplate.yaml`, `DD-APP-004`

- `inconclusiveLimit: 3` for payment success rate — NaN (no data) doesn't block
- `successCondition` includes `|| isNaN(result[0])` to handle missing data
- Staging uses shorter canary steps (1-2 minute pauses vs 2-5 minutes)
- `initialDelay: 120s` on payment success metric gives time for transactions to accumulate
- Design decision documents the low-traffic problem and mitigations

### Trap 5: Secret Rotation Without Downtime ✅
**Handled in:** `DD-APP-006`, Terraform (RDS Proxy + IAM Auth + rotation Lambda)

- **DB credentials**: RDS Proxy + IAM Auth makes password rotation transparent to pods. Pods never hold the DB password — they authenticate via IRSA token.
- **Stripe API key**: Requires pod restart (env var). Enhancement path: volume mount + file watcher.
- **Signing key**: Dual-key pattern (current + previous). Application tries current first, falls back.
- **Redis AUTH**: ElastiCache dual-password support during rotation window.

### Trap 6: Migration + Rollout Ordering ✅
**Handled in:** `db-migration-job.yaml` (ArgoCD PreSync Hook), `DD-APP-003`

Migration runs as **ArgoCD PreSync Hook**, not init container. If Rollout is rolled back:
- Migration is NOT reversed (PreSync already completed)
- This is CORRECT because expand-contract migrations are backward-compatible
- Old app code works with new schema (that's the whole point of expand-contract)
- Migration Job and Rollout have different ServiceAccounts (DDL vs DML)

### Trap 7: PCI Audit Logging ✅
**Handled in:** `payment-audit-config.yaml`, `fluent-bit/filters/payment-audit.yaml`

- Application emits structured JSON audit logs with `"audit": true` field
- Fluent Bit `rewrite_tag` filter forks audit logs to separate stream
- Fluent Bit routes audit stream to **two destinations**:
  1. **S3** (Object Lock bucket, gzip compressed, 7-year retention) — immutable compliance archive
  2. **CloudWatch** (90-day retention) — hot search for incident investigation
- Audit logs also flow to Loki via main pipeline (searchable via Grafana)
- Audit config defines exactly which fields to log per operation type
- **Masking rules**: card_number, CVV, expiry are NEVER logged — application enforces this
- Required fields per audit line: timestamp, trace_id, span_id, user_id, action, outcome, source_ip, pod_name, duration_ms
- Every audit log line carries trace_id for correlation with distributed traces

### Trap 8: Init Container Ordering ✅
**Handled in:** `rollout.yaml` (init containers), `db-migration-job.yaml` (separate Job)

There is NO init container for migration. Migration is a separate PreSync Job (sync wave -1). The only init container in the Rollout is `wait-for-dependencies` which verifies:
1. Database is reachable (pg_isready with 30 retries × 2s = 60s max)
2. Redis is reachable (redis-cli PING with 15 retries × 2s = 30s max)

The ordering is:
```
Sync Wave -3:  Namespace + ServiceAccounts
Sync Wave -2:  ExternalSecrets → K8s Secrets created
Sync Wave -1:  Migration Job (PreSync hook) — uses secrets, runs DDL
Sync Wave  0:  Rollout created
                 → init container: wait-for-dependencies (verify DB + Redis reachable)
                 → main container: payment-service starts
Sync Wave  1:  ServiceMonitor + AnalysisTemplate + Dashboard
```

If someone added BOTH a migration init container AND a dependency-check init container to the same pod:
- Init containers run **in order** (array index 0, then 1, etc.)
- Migration init would run first, then dependency check — which is **backwards**
- The dependency check should run BEFORE migration (what if DB is down?)
- This is why migration is a separate Job: avoids init container ordering headaches entirely

---


## 14. QUICK REFERENCE CARD

```
╔══════════════════════════════════════════════════════════════════════╗
║           PAYMENT-SERVICE ONBOARDING — QUICK REFERENCE               ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  NAMESPACE:     payments                                             ║
║  SERVICE:       payment-service                                      ║
║  LANGUAGE:      Go 1.22 (manual OTel — NO auto-instrumentation)      ║
║  MESH:          Linkerd (auto-inject via namespace label)            ║
║  SLO:           99.95% availability, p99 < 500ms                     ║
║                                                                      ║
║  SYNC WAVES:    -3(ns+sa) → -2(secrets) → -1(migrate) → 0(app) → 1   ║
║                                                                      ║
║  TERRAFORM:     applications/payment-service/                        ║
║    Creates:     IRSA(app+migrate), SQS+DLQ, RDS Proxy,               ║
║                 Secrets Manager(4), CloudWatch alarms+dashboard      ║
║                                                                      ║
║  CONNECTIONS:                                                        ║
║    DB:    10/pod × 20 max pods = 200 (limit: 1280) ✓                 ║
║    Redis: 5/pod  × 20 max pods = 100 (limit: 65000) ✓                ║
║                                                                      ║
║  CANARY (prod): 5% → 15% → 30% → 60% → 100% (~25 min)                ║
║  CANARY (stg):  20% → 50% → 100% (~5 min)                            ║
║  CANARY (dev):  100% (no canary)                                     ║
║                                                                      ║
║  ANALYSIS:      error-rate(<0.5%), p99(<500ms),                      ║
║                 payment-success(>99%), db-pool(<90%)                 ║
║                                                                      ║
║  SECRETS:                                                            ║
║    DB creds:     IAM Auth (rotation transparent)                     ║
║    Stripe key:   Restart required (env var)                          ║
║    Signing key:  Dual-key (zero downtime)                            ║
║    Redis AUTH:   Dual-password (zero downtime)                       ║
║                                                                      ║
║  MIGRATION:     PreSync Job (not init container)                     ║
║                 expand-contract pattern (backward-compatible)        ║
║                 payment_migrate SA (DDL) ≠ payment_app SA (DML)      ║
║                                                                      ║
║  NETWORKPOLICY: Default deny ALL. Explicit allow:                    ║
║    IN:  api-gateway:8080, prometheus:9090, linkerd:4143/4191         ║
║    OUT: DNS(53), DB(5432), Redis(6379), SQS(443), OTel(4317),        ║
║         Squid(3128), Linkerd(8443/8080), STS(443), IMDS(80)          ║
║                                                                      ║
║  AUDIT LOGS:    "audit":true → Fluent Bit → S3(7yr) + CW(90d)        ║
║                 NEVER log: card_number, CVV, full PAN                ║
║                                                                      ║
║  DEPLOY:        argocd app sync payment-service-production           ║
║  WATCH:         kubectl argo rollouts get rollout payment-service    ║
║                   -n payments --watch                                ║
║  ABORT:         kubectl argo rollouts abort payment-service          ║
║                   -n payments                                        ║
║  VALIDATE:      ./scripts/validate-payment-service.sh production     ║
║                                                                      ║
║  DASHBOARDS:    grafana.novamart.internal/d/payment-service-dashboard║
║  ALERTS:        7 alerts (2 SLO-critical, 2 SLO-warning,             ║
║                 1 SLO-info, 5 operational)                           ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```
