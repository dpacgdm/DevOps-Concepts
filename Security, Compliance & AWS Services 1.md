# Phase 6: Security, Compliance & AWS Services

## Phase 6 Lesson Plan

```
Lesson 1: AWS IAM Deep Dive — Identity, Policies, Organizations, SSO
Lesson 2: Secrets Management — KMS, Vault, Secrets Manager, Certificates
Lesson 3: Kubernetes Security — OPA/Gatekeeper, PSA, Falco, Supply Chain
Lesson 4: Network Security & AWS Security Services — WAF, Shield, GuardDuty, CloudTrail
Lesson 5: Compliance & Governance — SOC2, PCI-DSS, AWS Config, Landing Zones
```

---

# Phase 6, Lesson 1: AWS IAM Deep Dive

## Why IAM Is the Foundation of Everything

```
Every AWS API call — from launching an EC2 instance to reading an S3 
object to deploying an EKS pod — goes through IAM.

If IAM is misconfigured:
  - Overly permissive: an attacker who compromises ONE service 
    can pivot to EVERYTHING
  - Overly restrictive: engineers can't do their jobs, so they 
    create workarounds that are WORSE than the original risk
  - Misconfigured trust: cross-account access becomes a backdoor
  - No audit trail: you can't prove compliance or investigate breaches

IAM is the single most important AWS service to understand deeply.
It's also the one most people get wrong.

At NovaMart:
  - 200+ microservices, each needing specific AWS permissions
  - 3 AWS accounts (dev, staging, prod) minimum
  - 150+ engineers who need varying levels of access
  - PCI-DSS compliance requires least-privilege and audit trails
  - A single IAM misconfiguration in the payment service could 
    expose customer financial data
```

---

## Part 1: IAM Fundamentals — The Request Authorization Model

### How Every AWS API Call Is Authorized

```
┌─────────────────────────────────────────────────────────────────────┐
│                  AWS API REQUEST FLOW                                │
│                                                                     │
│  1. AUTHENTICATION: "Who are you?"                                  │
│     ├── IAM User: Access Key ID + Secret Access Key                 │
│     ├── IAM Role: Temporary credentials from STS                    │
│     ├── Federated: SAML/OIDC token → STS → temporary creds          │
│     └── Anonymous: No credentials (public access)                   │
│                                                                     │
│  2. REQUEST CONTEXT assembled:                                      │
│     ├── Principal (who)                                             │
│     ├── Action (what API call: s3:GetObject, ec2:RunInstances)     │
│     ├── Resource (which ARN)                                        │
│     ├── Conditions (source IP, time, MFA, tags, etc.)              │
│     └── Environment (account, region, service)                     │
│                                                                     │
│  3. POLICY EVALUATION (order matters!):                             │
│                                                                     │
│     ┌──────────────────────────────┐                                │
│     │ Explicit DENY in any policy? │──YES──→ ❌ DENIED              │
│     └──────────────┬───────────────┘                                │
│                    NO                                               │
│                    ▼                                                 │
│     ┌──────────────────────────────┐                                │
│     │ SCP allows it?              │──NO───→ ❌ DENIED               │
│     │ (Organizations)             │                                 │
│     └──────────────┬───────────────┘                                │
│                    YES                                              │
│                    ▼                                                 │
│     ┌──────────────────────────────┐                                │
│     │ Resource-based policy        │                                │
│     │ grants cross-account access? │──YES──→ ✅ ALLOWED             │
│     └──────────────┬───────────────┘  (if same account, continue)  │
│                    │                                                │
│                    ▼                                                 │
│     ┌──────────────────────────────┐                                │
│     │ Identity-based policy        │                                │
│     │ (user/role policy) allows?   │──NO───→ ❌ DENIED              │
│     └──────────────┬───────────────┘                                │
│                    YES                                              │
│                    ▼                                                 │
│     ┌──────────────────────────────┐                                │
│     │ Permissions boundary         │                                │
│     │ allows it?                  │──NO───→ ❌ DENIED               │
│     └──────────────┬───────────────┘                                │
│                    YES                                              │
│                    ▼                                                 │
│     ┌──────────────────────────────┐                                │
│     │ Session policy allows?       │──NO───→ ❌ DENIED              │
│     │ (if using STS)              │                                 │
│     └──────────────┬───────────────┘                                │
│                    YES                                              │
│                    ▼                                                 │
│                 ✅ ALLOWED                                           │
│                                                                     │
│  KEY INSIGHT: Default is DENY. Every gate must say YES.             │
│  Any single DENY anywhere = denied. Period.                        │
│  This is "deny by default, allow by exception."                    │
└─────────────────────────────────────────────────────────────────────┘
```

### The Critical Rule

```
EXPLICIT DENY > EXPLICIT ALLOW > IMPLICIT DENY (default)

This means:
  - If ANY policy says Deny → denied (even if 10 others say Allow)
  - If no policy says Allow → denied (implicit deny = default state)
  - Only if at least one policy says Allow AND nothing says Deny → allowed

This is why IAM is "additive for allows, absolute for denies."
```

---

## Part 2: IAM Policy Language

### Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadPaymentBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::novamart-payment-data",
        "arn:aws:s3:::novamart-payment-data/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/team": "payments"
        },
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    }
  ]
}
```

### Every Field Explained

```
Version: Always "2012-10-17" (the current policy language version)
         "2008-10-17" exists but lacks features. Always use 2012.

Statement: Array of permission rules. Each evaluated independently.

Sid: Statement ID. Optional but recommended for readability and 
     programmatic reference.

Effect: "Allow" or "Deny". No other values.

Action: The AWS API actions. Format: <service>:<action>
  Wildcards:
    "s3:*"              — all S3 actions
    "s3:Get*"           — all S3 Get actions (GetObject, GetBucketPolicy, etc.)
    "*"                 — ALL actions on ALL services (NEVER use in prod)

  NotAction: The inverse — "everything EXCEPT these actions"
    Use case: "Allow everything except IAM changes"
    ⚠️ Dangerous — new services/actions are automatically included

Resource: The ARN(s) this applies to.
  Format: arn:aws:<service>:<region>:<account>:<resource>
  
  S3 gotcha — TWO resources needed:
    "arn:aws:s3:::bucket-name"      → for ListBucket (bucket-level)
    "arn:aws:s3:::bucket-name/*"    → for GetObject (object-level)
    Missing either one = Access Denied

  "*" = all resources (avoid in production)

Condition: Additional constraints. ALL conditions must be true (AND logic).
  Multiple values within a condition key = OR logic.
  Multiple condition keys = AND logic.
```

### Condition Keys You Must Know

```
┌──────────────────────────────────────┬────────────────────────────────────────┐
│ Condition Key                        │ Use Case                               │
├──────────────────────────────────────┼────────────────────────────────────────┤
│ aws:SourceIp                         │ Restrict to corporate IP range         │
│ aws:SourceVpc                        │ Restrict to specific VPC               │
│ aws:SourceVpce                       │ Restrict to specific VPC endpoint      │
│ aws:PrincipalTag/<key>               │ ABAC — tag on the calling principal    │
│ aws:ResourceTag/<key>                │ Tag on the target resource             │
│ aws:RequestedRegion                  │ Restrict to specific regions           │
│ aws:PrincipalOrgID                   │ Restrict to your AWS Organization      │
│ aws:MultiFactorAuthPresent           │ Require MFA for sensitive actions      │
│ aws:SecureTransport                  │ Require HTTPS (deny HTTP)              │
│ s3:prefix                            │ S3 key prefix filtering                │
│ ec2:ResourceTag/<key>                │ EC2 tag-based access control           │
│ kms:ViaService                       │ KMS key used only through a service    │
│ sts:ExternalId                       │ Cross-account confused deputy fix      │
│ aws:PrincipalArn                     │ Specific role/user ARN                 │
│ aws:CalledVia                        │ Service that made the call on behalf   │
│ aws:PrincipalIsAWSService            │ Is the caller an AWS service?          │
│ elasticmapreduce:ResourceTag/<key>   │ Service-specific tag conditions        │
└──────────────────────────────────────┴────────────────────────────────────────┘
```

### Policy Types — Where Policies Live

```
┌──────────────────────┬───────────────────────────────────────────────────┐
│ Policy Type          │ Attached To / Scope                               │
├──────────────────────┼───────────────────────────────────────────────────┤
│ Identity-based       │ IAM users, groups, roles                          │
│ (managed/inline)     │ "What can THIS principal do?"                     │
│                      │ Managed = reusable, versioned, preferred          │
│                      │ Inline = embedded directly, use sparingly         │
├──────────────────────┼───────────────────────────────────────────────────┤
│ Resource-based       │ S3 buckets, SQS queues, KMS keys, Lambda, etc.  │
│                      │ "Who can access THIS resource?"                   │
│                      │ Has a Principal field (identity policies don't)   │
│                      │ ONLY way to grant cross-account without role      │
│                      │ assumption (S3, KMS, SQS, SNS, Lambda)           │
├──────────────────────┼───────────────────────────────────────────────────┤
│ Permissions boundary │ IAM users, roles                                  │
│                      │ MAXIMUM permissions ceiling                       │
│                      │ Even if identity policy allows, boundary must too │
│                      │ Used to delegate safe IAM admin                   │
├──────────────────────┼───────────────────────────────────────────────────┤
│ SCP (Service Control │ AWS Organization OUs / Accounts                   │
│ Policy)              │ MAXIMUM permissions for entire account            │
│                      │ Cannot GRANT — can only RESTRICT                  │
│                      │ Even root user is bound by SCPs                   │
├──────────────────────┼───────────────────────────────────────────────────┤
│ Session policy       │ STS AssumeRole / GetFederationToken               │
│                      │ Further restricts the assumed session             │
│                      │ Rarely used directly (programmatic only)          │
├──────────────────────┼───────────────────────────────────────────────────┤
│ VPC Endpoint policy  │ VPC endpoints (Gateway/Interface)                 │
│                      │ Controls which principals/resources can use       │
│                      │ the endpoint                                      │
│                      │ Default: allow all — ALWAYS restrict in prod     │
└──────────────────────┴───────────────────────────────────────────────────┘
```

### The Intersection Model

```
EFFECTIVE PERMISSIONS = 
  Identity policy
    ∩ Permissions boundary (if set)
    ∩ SCP (if in Organization)
    ∩ Session policy (if assumed role)
    ∩ Resource-based policy (additive for same account,
                             independent for cross-account)
    — Explicit denies (override everything)

Think of it as concentric circles:
  ┌─────────────────────────────────────────────┐
  │ SCP (outermost — account-level ceiling)     │
  │   ┌───────────────────────────────────────┐ │
  │   │ Permissions boundary (role ceiling)   │ │
  │   │   ┌───────────────────────────────┐   │ │
  │   │   │ Identity policy (actual perms)│   │ │
  │   │   │   ┌───────────────────────┐   │   │ │
  │   │   │   │ Session policy (temp) │   │   │ │
  │   │   │   └───────────────────────┘   │   │ │
  │   │   └───────────────────────────────┘   │ │
  │   └───────────────────────────────────────┘ │
  └─────────────────────────────────────────────┘
  
  You can only do what ALL circles allow.
  Exception: Resource-based policies can "reach through" 
  permissions boundaries for same-account access.
```

---

## Part 3: IAM Roles — The Correct Way to Grant Access

### Why Roles, Not Users

```
IAM Users have LONG-LIVED credentials:
  - Access Key + Secret Key
  - Can be leaked in Git, logs, .env files, Slack
  - Must be rotated manually
  - Exist forever until deleted
  - ONE compromised key = persistent access until revoked

IAM Roles provide TEMPORARY credentials:
  - Generated by STS (Security Token Service)
  - Expire automatically (15 min to 12 hours)
  - No long-lived secrets to leak
  - Credential rotation is automatic
  - ONE compromised session = limited time window

RULE: In production, EVERYTHING uses roles.
  - EC2 instances → Instance profiles (roles)
  - EKS pods → IRSA (roles)
  - Lambda → Execution roles
  - CI/CD → OIDC federation (roles)
  - Humans → SSO federated roles
  
IAM Users should only exist for:
  - Emergency "break glass" access (MFA-protected, alarmed)
  - Legacy systems that can't use roles (migration target)
  - Service accounts for tools that ONLY support access keys
```

### Role Trust Policies — Who Can Assume This Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```
The TRUST POLICY answers: "Who is allowed to BECOME this role?"
The PERMISSIONS POLICY answers: "What can this role DO?"

These are separate documents. A common mistake is putting permissions 
in the trust policy or vice versa.

Principal types in trust policies:
┌──────────────────────────────────────────────────────────────────┐
│ "Principal": {"AWS": "arn:aws:iam::123456789012:root"}          │
│   → Any principal in account 123456789012                        │
│                                                                  │
│ "Principal": {"AWS": "arn:aws:iam::123456789012:role/jenkins"}  │
│   → Only the jenkins role in that account                        │
│                                                                  │
│ "Principal": {"Service": "ec2.amazonaws.com"}                   │
│   → EC2 service (instance profiles)                              │
│                                                                  │
│ "Principal": {"Service": "eks.amazonaws.com"}                   │
│   → EKS service                                                  │
│                                                                  │
│ "Principal": {"Federated": "arn:aws:iam::123:oidc-provider/..."}│
│   → OIDC identity provider (IRSA, GitHub Actions, etc.)          │
│                                                                  │
│ "Principal": {"Federated": "arn:aws:iam::123:saml-provider/..."}│
│   → SAML identity provider (SSO, Active Directory)               │
│                                                                  │
│ "Principal": "*"                                                 │
│   → ANYONE (⚠️ EXTREMELY DANGEROUS — use with conditions only)  │
└──────────────────────────────────────────────────────────────────┘
```

### IRSA — IAM Roles for Service Accounts (Critical for EKS)

```
IRSA is how Kubernetes pods get AWS permissions without
embedding credentials.

THE OLD WAY (bad):
  - Attach IAM role to EC2 worker node
  - ALL pods on that node share the same permissions
  - payment-svc can access search-svc's S3 bucket
  - Violates least privilege at the pod level

THE IRSA WAY (correct):
  - Each K8s ServiceAccount maps to a specific IAM role
  - Pod assumes ONLY its own role
  - payment-svc can ONLY access payment-related resources
  - Least privilege at the pod level
```

```
HOW IRSA WORKS:

┌──────────┐  1. Pod starts with    ┌───────────────┐
│ Pod with │────ServiceAccount────→│ K8s API Server │
│ SA       │  annotated with IAM   │ (projects OIDC │
│ payment- │  role ARN             │  token into pod)│
│ svc-sa   │                       └───────┬────────┘
└────┬─────┘                               │
     │                                     │ 2. Projected token
     │  3. Pod reads token                 │    mounted at
     │     from mounted path               │    /var/run/secrets/
     │                                     │    eks.amazonaws.com/
     │                                     │    serviceaccount/token
     ▼                                     │
┌────────────┐                             │
│ AWS SDK    │  4. SDK calls               │
│ in the pod │─────STS AssumeRoleWithWebIdentity
│            │     with the OIDC token     │
└────┬───────┘                             │
     │                                     │
     ▼                                     │
┌─────────┐  5. STS validates token  ┌─────▼──────────┐
│ AWS STS │←──against OIDC provider──│ EKS OIDC       │
│         │   (is this token from    │ Provider        │
│         │    my cluster? Is the    │ (registered     │
│         │    SA allowed to assume  │  in IAM)        │
│         │    this role?)           │                 │
└────┬────┘                          └────────────────┘
     │
     │  6. STS returns temporary credentials
     │     (AccessKeyId, SecretAccessKey, SessionToken)
     │     Valid for up to 12 hours
     ▼
┌──────────┐
│ AWS API  │  7. Pod calls AWS APIs with temporary creds
│ (S3, SQS,│     No long-lived secrets anywhere
│  RDS...) │
└──────────┘
```

### IRSA Setup — Complete Production Example

```bash
# Step 1: Create OIDC provider for EKS cluster (one-time per cluster)
eksctl utils associate-iam-oidc-provider \
  --cluster novamart-prod \
  --approve

# Or via Terraform:
```

```hcl
# Terraform — OIDC provider
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer
}
```

```hcl
# Step 2: Create IAM role with trust policy for the specific ServiceAccount

locals {
  oidc_provider_arn = aws_iam_openid_connect_provider.eks.arn
  oidc_provider_url = replace(aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")
}

# IAM Role for payment-svc
resource "aws_iam_role" "payment_svc" {
  name = "novamart-prod-payment-svc"

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
            # CRITICAL: Both conditions required
            "${local.oidc_provider_url}:sub" = "system:serviceaccount:payments:payment-svc-sa"
            "${local.oidc_provider_url}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = {
    Service     = "payment-svc"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# Permissions: what payment-svc can actually do
resource "aws_iam_role_policy" "payment_svc" {
  name = "payment-svc-permissions"
  role = aws_iam_role.payment_svc.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowSQSPaymentQueue"
        Effect = "Allow"
        Action = [
          "sqs:SendMessage",
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes"
        ]
        Resource = "arn:aws:sqs:us-east-1:123456789012:payment-processing"
      },
      {
        Sid    = "AllowKMSDecrypt"
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "arn:aws:kms:us-east-1:123456789012:key/payment-key-id"
      },
      {
        Sid    = "AllowSecretsManagerPayment"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = "arn:aws:secretsmanager:us-east-1:123456789012:secret:payment/*"
      }
    ]
  })
}
```

```yaml
# Step 3: Kubernetes ServiceAccount with annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-svc-sa
  namespace: payments
  annotations:
    # THIS annotation tells IRSA which IAM role to assume
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/novamart-prod-payment-svc
---
# Step 4: Pod/Deployment references the ServiceAccount
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-svc
  namespace: payments
spec:
  template:
    spec:
      serviceAccountName: payment-svc-sa  # ← Uses the annotated SA
      containers:
        - name: payment-svc
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/payment-svc:v2.34.2
          # No AWS credentials in env vars or mounted secrets
          # The AWS SDK automatically uses the IRSA-projected token
```

### IRSA Failure Modes

```
FAILURE 1: "AccessDenied" — Pod can't assume role
  CAUSE: Trust policy condition mismatch
  CHECK: The :sub condition must EXACTLY match:
    "system:serviceaccount:<namespace>:<sa-name>"
  COMMON MISTAKE: Wrong namespace or SA name in trust policy
  DEBUG:
    # Verify what the pod's token claims
    kubectl -n payments exec deploy/payment-svc -- \
      cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | \
      cut -d. -f2 | base64 -d 2>/dev/null | jq .
    # Check "sub" field matches trust policy

FAILURE 2: "No credentials" — SDK doesn't find IRSA token
  CAUSE: ServiceAccount annotation missing or pod doesn't reference SA
  CHECK:
    kubectl -n payments get sa payment-svc-sa -o yaml
    # Verify eks.amazonaws.com/role-arn annotation exists
    kubectl -n payments get pod <pod> -o yaml | grep serviceAccountName

FAILURE 3: OIDC provider not registered
  CAUSE: Forgot to create the OIDC provider for the cluster
  CHECK:
    aws iam list-open-id-connect-providers
    # Must list your EKS cluster's OIDC issuer URL

FAILURE 4: Stale OIDC thumbprint
  CAUSE: AWS rotated the OIDC signing certificate
  SYMPTOM: Pods that used to work suddenly get AccessDenied
  FIX: Update the thumbprint in the OIDC provider config
  NOTE: AWS announced in 2023 that they no longer validate thumbprints 
        for EKS OIDC providers — but other providers still need them

FAILURE 5: Token expiry
  CAUSE: Token audience or expiry mismatch
  NOTE: Projected tokens have configurable expiry (default 86400s)
        AWS SDK automatically refreshes, but very old SDKs might not
  FIX: Update AWS SDK version

FAILURE 6: Node-level role interference
  CAUSE: Pod falls back to node instance role because IRSA isn't configured
  DANGER: Pod gets TOO MANY permissions (node role has broad access)
  FIX: Block instance metadata from pods:
```

```yaml
# Block IMDS access from pods (force them to use IRSA)
# This prevents pods from falling back to the node's instance role
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-instance-metadata
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32  # Instance Metadata Service
```

```
BETTER APPROACH: Configure the aws-node DaemonSet to disable 
IMDS for pods:

kubectl -n kube-system set env daemonset/aws-node \
  AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true

# Or in EKS managed node group:
# httpPutResponseHopLimit: 1 (blocks containerized access to IMDSv2)
# This is the recommended approach — blocks at network level
```

---

## Part 4: AWS Organizations — Multi-Account Strategy

### Why Multiple Accounts?

```
SINGLE ACCOUNT (bad for production):
  - One blast radius: IAM misconfiguration affects everything
  - Cost attribution is difficult
  - Compliance boundaries are blurred
  - Service quotas are shared
  - Security isolation is impossible

MULTI-ACCOUNT (NovaMart's approach):
  - Blast radius containment: dev can't touch prod
  - Clear cost attribution per account
  - Compliance scope minimization (PCI only in payment account)
  - Independent service quotas
  - Strong security boundaries (account = hardest boundary in AWS)
```

### NovaMart Account Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS ORGANIZATION                                  │
│                    (Management Account)                              │
│                    - Billing only                                    │
│                    - NO workloads here                               │
│                    - SCPs managed from here                          │
│                                                                     │
├──────────────┬──────────────┬──────────────┬───────────────────────┤
│              │              │              │                       │
│   Security   │  Shared      │  Workloads   │  Sandbox              │
│   OU         │  Services OU │  OU          │  OU                   │
│              │              │              │                       │
│ ┌──────────┐│┌────────────┐│┌────────────┐│┌───────────────┐      │
│ │ Security ││ │ Networking ││ │ Production ││ │ Dev-sandbox-1 │      │
│ │ Account  │││ Account    ││ │ Account    │││ (per engineer) │      │
│ │          │││            ││ │            ││└───────────────┘      │
│ │ GuardDuty│││ Transit GW ││ │ EKS        ││┌───────────────┐      │
│ │ CloudTrl ││ │ DNS (R53)  ││ │ RDS        │││ Dev-sandbox-2 │      │
│ │ Sec Hub  │││ VPN/DX     ││ │ S3         ││└───────────────┘      │
│ │ Config   │││ Shared VPCs││ │ SQS        ││                       │
│ └──────────┘│└────────────┘│ └────────────┘│                       │
│              │              │┌────────────┐│                       │
│ ┌──────────┐│┌────────────┐││ Staging    ││                       │
│ │ Log      │││ CI/CD      │││ Account    ││                       │
│ │ Archive  ││ │ Account    │││            ││                       │
│ │ Account  │││            │││ Mirror of  ││                       │
│ │          │││ Jenkins     │││ production ││                       │
│ │ CloudTrl │││ Artifactory│││ (smaller)  ││                       │
│ │ all accts│││ ECR        ││└────────────┘│                       │
│ │ VPC Flow │││            ││┌────────────┐│                       │
│ └──────────┘│└────────────┘││ Dev        ││                       │
│              │              ││ Account    ││                       │
│              │              │└────────────┘│                       │
└──────────────┴──────────────┴──────────────┴───────────────────────┘
```

### Service Control Policies (SCPs)

```
SCPs are the GUARDRAILS for the entire organization.

Key concepts:
  - SCPs do NOT grant permissions — they set MAXIMUM boundaries
  - Even the root user of a member account is bound by SCPs
  - SCPs on the management account have NO effect
    (this is why the management account should have NO workloads)
  - SCPs are inherited: OU SCP → child OU SCP → account
  - Effective SCP = intersection of all inherited SCPs
```

### Production SCPs for NovaMart

```json
// SCP 1: Deny region outside approved regions
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedRegions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2",
            "eu-west-1"
          ]
        },
        "ArnNotLike": {
          "aws:PrincipalARN": [
            "arn:aws:iam::*:role/OrganizationAdmin"
          ]
        }
      }
    }
  ]
}
```

```json
// SCP 2: Protect CloudTrail and GuardDuty
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyCloudTrailModification",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/SecurityAdmin"
        }
      }
    },
    {
      "Sid": "DenyGuardDutyDisable",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "guardduty:UpdateDetector"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/SecurityAdmin"
        }
      }
    }
  ]
}
```

```json
// SCP 3: Prevent leaving the Organization
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeaveOrg",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```

```json
// SCP 4: Enforce encryption
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedS3",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": [
            "aws:kms",
            "AES256"
          ]
        },
        "Null": {
          "s3:x-amz-server-side-encryption": "false"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedEBS",
      "Effect": "Deny",
      "Action": "ec2:CreateVolume",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "ec2:Encrypted": "false"
        }
      }
    }
  ]
}
```

```json
// SCP 5: Deny dangerous services/actions in production
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDangerousActions",
      "Effect": "Deny",
      "Action": [
        "ec2:CreateDefaultVpc",
        "iam:CreateUser",
        "iam:CreateAccessKey",
        "sts:GetFederationToken",
        "organizations:*",
        "account:*"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalARN": [
            "arn:aws:iam::*:role/OrganizationAdmin",
            "arn:aws:iam::*:role/BreakGlass"
          ]
        }
      }
    }
  ]
}
```

### SCP Failure Modes

```
FAILURE 1: SCP locks out everyone including admins
  CAUSE: Deny policy without exception for admin role
  SYMPTOM: Even the account's admin role gets Access Denied
  FIX: ALWAYS include a condition exception for an admin/break-glass role
  RECOVERY: SCPs can only be modified from the management account
    → This is why management account access is critical and 
      heavily guarded

FAILURE 2: SCP breaks AWS service actions
  CAUSE: Overly broad deny that blocks AWS service-linked roles
  EXAMPLE: Denying "iam:*" prevents AWS services from creating 
    service-linked roles needed for ELB, RDS, etc.
  FIX: Use conditions to exclude AWS service principals:
    "ArnNotLike": {"aws:PrincipalARN": "arn:aws:iam::*:role/aws-service-role/*"}

FAILURE 3: New region needed but SCP blocks it
  CAUSE: Region restriction SCP doesn't include the new region
  SYMPTOM: Terraform apply fails for resources in new region
  FIX: Update SCP first, test, then deploy to new region
  LESSON: SCPs should be in Terraform and go through PR review

FAILURE 4: SCP size limit exceeded
  LIMIT: 5,120 characters per SCP
  CAUSE: Too many granular deny rules
  FIX: Use wildcards wisely, split into multiple SCPs
  NOTE: Maximum 5 SCPs per OU/account attachment
```

---

## Part 5: AWS IAM Identity Center (SSO)

### How Humans Access AWS at NovaMart

```
THE WRONG WAY:
  - Each engineer has an IAM user per account
  - 150 engineers × 6 accounts = 900 IAM users to manage
  - Password rotation, MFA enrollment, access key management per user
  - Engineer leaves → must delete users in ALL accounts
  - No centralized audit

THE RIGHT WAY (IAM Identity Center):
  - Single identity provider (Okta, Azure AD, or AWS built-in)
  - Engineers authenticate once → get temporary access to any account
  - Permission sets define what they can do in each account
  - Centralized provisioning and deprovisioning
  - Single audit trail across all accounts

┌───────────────┐     SAML/OIDC      ┌─────────────────────┐
│ Identity       │──────────────────→│ IAM Identity Center │
│ Provider       │                   │ (AWS SSO)           │
│ (Okta/AzureAD)│                   │                     │
│                │                   │ Permission Sets:    │
│ Users & Groups │                   │ - AdminAccess       │
│ managed here   │                   │ - ReadOnly          │
│                │                   │ - DevOps            │
│                │                   │ - Developer         │
└───────────────┘                   │ - SecurityAudit     │
                                    └──────────┬──────────┘
                                               │
                    ┌──────────────────────────┼──────────────┐
                    │                          │              │
                    ▼                          ▼              ▼
             ┌─────────────┐          ┌──────────────┐ ┌───────────┐
             │ Prod Account│          │ Staging Acct │ │ Dev Acct  │
             │             │          │              │ │           │
             │ Eng: DevOps │          │ Eng: DevOps  │ │ Eng: Admin│
             │     ReadOnly│          │     Developer│ │    DevOps │
             │             │          │     Admin    │ │           │
             │ Mgr: Admin  │          │              │ │           │
             └─────────────┘          └──────────────┘ └───────────┘
```

### Permission Sets — NovaMart Examples

```json
// Permission Set: DevOps
// Assigned to: Platform team in Prod, Staging, Dev accounts
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EKSFullAccess",
      "Effect": "Allow",
      "Action": [
        "eks:*",
        "ecr:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EC2ReadAndLimited",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*",
        "ec2:CreateTags",
        "ec2:DeleteTags"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::novamart-*",
        "arn:aws:s3:::novamart-*/*"
      ]
    },
    {
      "Sid": "ObservabilityAccess",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:*",
        "logs:*",
        "xray:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SecretsRead",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret",
        "secretsmanager:ListSecrets"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyIAMModification",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:DeleteUser",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/aws-reserved/sso.amazonaws.com/*"
        }
      }
    }
  ]
}
```

### CLI Access via Identity Center

```bash
# Configure SSO profile
aws configure sso
#  SSO session name: novamart
#  SSO start URL: https://novamart.awsapps.com/start
#  SSO Region: us-east-1
#  SSO registration scopes: sso:account:access

# Login (opens browser for authentication)
aws sso login --profile novamart-prod

# Use the profile
aws s3 ls --profile novamart-prod
aws eks update-kubeconfig --name novamart-prod --profile novamart-prod

# Credentials are TEMPORARY — auto-refresh via SSO session
# No access keys stored in ~/.aws/credentials

# Multiple accounts:
# ~/.aws/config:
[profile novamart-prod]
sso_session = novamart
sso_account_id = 111111111111
sso_role_name = DevOps
region = us-east-1

[profile novamart-staging]
sso_session = novamart
sso_account_id = 222222222222
sso_role_name = DevOps
region = us-east-1

[profile novamart-dev]
sso_session = novamart
sso_account_id = 333333333333
sso_role_name = Admin
region = us-east-1

[sso-session novamart]
sso_start_url = https://novamart.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access
```

---

## Part 6: Least Privilege in Practice

### The Principle

```
Grant ONLY the permissions needed to perform the task.
No more. Review regularly. Revoke when no longer needed.

This sounds simple. In practice, it's the hardest security 
challenge in AWS because:

1. Engineers want "it just works" → they request broad permissions
2. Debugging permission errors is painful → admins grant wildcard
3. Services evolve → permissions accumulate but are never removed
4. Cross-service interactions → hard to predict which actions are needed
5. AWS adds new actions constantly → wildcards grant more than intended
```

### ABAC — Attribute-Based Access Control

```
RBAC (traditional):
  "DevOps team can access EKS"
  → Create role, attach policy, assign role to team
  → 200 microservices × 3 environments = 600 roles to manage

ABAC (tag-based):
  "Anyone tagged team=payments can access resources tagged team=payments"
  → ONE policy handles all teams
  → New teams get access by tagging, not by creating new roles

EXAMPLE:
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSameTeamAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::novamart-*/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/team": "${aws:PrincipalTag/team}"
        }
      }
    }
  ]
}
```

```
This single policy means:
  - payment-svc (tagged team=payments) can ONLY access objects tagged team=payments
  - order-svc (tagged team=orders) can ONLY access objects tagged team=orders
  - No new policies needed when a new team is created — just tag correctly

NovaMart ABAC tags:
  team:        payments, orders, search, platform, data
  environment: production, staging, dev
  service:     payment-svc, order-svc, etc.
  compliance:  pci, hipaa, standard
```

### IAM Access Analyzer

```
ACCESS ANALYZER finds:
1. External access: Resources shared outside your account/org
   - S3 bucket with public access
   - IAM role assumable by another account
   - KMS key with cross-account grant
   - Lambda with public invocation

2. Unused access: Permissions granted but never used
   - Roles not assumed in 90 days
   - Actions in policy never called
   - Services in policy never accessed

3. Policy validation: Syntax and security errors
   - Overly permissive wildcards
   - Missing condition keys
   - Redundant statements

4. Policy generation: Generate least-privilege from CloudTrail
   - Analyze 90 days of actual API calls
   - Generate a policy with ONLY the actions actually used
   - THIS IS THE KILLER FEATURE for achieving least privilege
```

```bash
# Generate policy from actual usage (last 90 days)
aws accessanalyzer start-policy-generation \
  --policy-generation-details '{
    "principalArn": "arn:aws:iam::123456789012:role/novamart-prod-payment-svc",
    "cloudTrailDetails": [
      {
        "trailArn": "arn:aws:cloudtrail:us-east-1:123456789012:trail/org-trail",
        "startTime": "2024-01-01T00:00:00Z",
        "endTime": "2024-03-31T00:00:00Z",
        "accessRole": "arn:aws:iam::123456789012:role/AccessAnalyzerRole"
      }
    ]
  }'

# Check generation status
aws accessanalyzer get-generated-policy --job-id <job-id>

# Output: A policy with ONLY the actions payment-svc actually called
# in the last 90 days. This is your least-privilege baseline.
```

### Permission Boundaries — Delegating Safe IAM Admin

```
PROBLEM: Platform team needs to create IAM roles for new services.
         But you don't want them creating roles with AdminAccess.

SOLUTION: Permission boundary = maximum ceiling for any role they create.

The platform team can create roles, BUT every role they create MUST 
have the permission boundary attached. The boundary limits what those 
roles can ever do, regardless of what policies are attached later.
```

```json
// Permission boundary: maximum allowed for service roles
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServiceActions",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "sqs:*",
        "sns:*",
        "dynamodb:*",
        "secretsmanager:GetSecretValue",
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "logs:*",
        "cloudwatch:PutMetricData",
        "xray:PutTraceSegments"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyBoundaryModification",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteRolePermissionsBoundary",
        "iam:PutRolePermissionsBoundary"
      ],
      "Resource": "*"
    }
  ]
}
```

```json
// Platform team's IAM creation policy — MUST attach boundary
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCreateRolesWithBoundary",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy"
      ],
      "Resource": "arn:aws:iam::123456789012:role/svc-*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/ServiceRoleBoundary"
        }
      }
    }
  ]
}
```

```
Now the platform team:
  ✅ Can create roles prefixed with svc-*
  ✅ Must attach the ServiceRoleBoundary 
  ❌ Cannot create roles without the boundary (condition enforces it)
  ❌ Cannot remove or change the boundary (denied in boundary itself)
  ❌ Cannot grant IAM, Organizations, or other dangerous permissions
     (boundary doesn't include them)
```

---

## Part 7: Cross-Account Access Patterns

### Pattern 1: Role Assumption (Most Common)

```
Account A (CI/CD)                    Account B (Production)
┌─────────────────┐                 ┌─────────────────────┐
│ Jenkins Role     │───AssumeRole──→│ Deploy Role          │
│                  │                │                      │
│ Policy:          │                │ Trust Policy:        │
│ "Allow           │                │ "Allow Principal     │
│  sts:AssumeRole  │                │  from Account A's    │
│  on Deploy Role  │                │  Jenkins Role"       │
│  in Account B"   │                │                      │
│                  │                │ Permissions:          │
│                  │                │ "EKS deploy, ECR pull"│
└─────────────────┘                └─────────────────────┘
```

```hcl
# In Account B (Production) — the role to be assumed
resource "aws_iam_role" "cross_account_deploy" {
  name = "novamart-cross-account-deploy"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::444444444444:role/jenkins-controller"
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "sts:ExternalId" = "novamart-deploy-2024"  # Confused deputy prevention
          }
        }
      }
    ]
  })
}

# In Account A (CI/CD) — permission to assume the role
resource "aws_iam_role_policy" "jenkins_cross_account" {
  name = "assume-prod-deploy-role"
  role = aws_iam_role.jenkins_controller.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "sts:AssumeRole"
        Resource = "arn:aws:iam::111111111111:role/novamart-cross-account-deploy"
      }
    ]
  })
}
```

### The Confused Deputy Problem

```
WITHOUT ExternalId:

Third-party service "MonitorCorp" asks you to create a role 
they can assume to monitor your account.

You create a role trusting MonitorCorp's account.
MonitorCorp's account ID: 999999999999

But MonitorCorp also has another customer "EvilCorp."
EvilCorp tells MonitorCorp: "Please monitor account 123456789012" 
(your account!)

MonitorCorp now assumes your role on behalf of EvilCorp.
EvilCorp has accessed your account through MonitorCorp.

WITH ExternalId:
  Your trust policy requires ExternalId = "unique-secret-string"
  MonitorCorp includes this ExternalId when assuming on YOUR behalf
  EvilCorp doesn't know your ExternalId
  MonitorCorp can't assume your role on EvilCorp's behalf

RULE: ALWAYS use ExternalId for third-party cross-account access.
```

### Pattern 2: Resource-Based Policies (No Role Assumption Needed)

```json
// S3 bucket policy — grant cross-account access directly
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCICDAccountRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::444444444444:role/jenkins-controller"
      },
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::novamart-artifacts",
        "arn:aws:s3:::novamart-artifacts/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-novamart123"
        }
      }
    }
  ]
}
```

```
When to use which pattern:

Resource-based policy (S3, KMS, SQS, SNS, Lambda):
  ✅ Simpler — no role assumption step
  ✅ The calling principal retains its original identity
  ✅ Good for: "Account A's service writes to Account B's S3"
  ❌ Only works for services that support resource policies

Role assumption (everything else):
  ✅ Works for ALL AWS services
  ✅ Caller "becomes" the target role — clear permission boundary
  ✅ Good for: CLI access, broad service access, temp credentials
  ❌ Extra step (must call AssumeRole first)
  ❌ Caller loses original identity (becomes the assumed role)
```

---

## Part 8: CI/CD Authentication — No Long-Lived Secrets

### OIDC Federation for CI/CD

```
THE OLD WAY:
  Create IAM user → Generate access keys → Store in Jenkins credentials
  Problems:
    - Keys never expire
    - Keys must be rotated manually
    - Key leak in build logs = permanent access until revoked
    - Audit trail shows "user" not "which pipeline run"

THE NEW WAY (OIDC Federation):
  CI system provides a JWT → AWS STS validates it → Temporary creds
  Benefits:
    - No stored secrets
    - Credentials expire in minutes/hours
    - Each pipeline run gets unique credentials
    - Audit trail includes pipeline metadata
```

```
┌────────────────┐                    ┌─────────────┐
│ Jenkins/GitHub │   1. JWT token     │ AWS STS     │
│ Actions/GitLab │──────────────────→│             │
│                │   (signed by CI    │ Validates   │
│                │    provider's      │ against     │
│                │    OIDC endpoint)  │ registered  │
│                │                    │ OIDC        │
│                │   2. Temp creds    │ provider    │
│                │←──────────────────│             │
│                │   (15min-12hr)     └─────────────┘
│                │
│  3. Use temp   │
│     creds for  │
│     AWS APIs   │
└────────────────┘
```

```hcl
# GitHub Actions OIDC setup (Terraform)

# Register GitHub as an OIDC provider
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# Role for GitHub Actions to assume
resource "aws_iam_role" "github_actions_deploy" {
  name = "github-actions-deploy"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            # CRITICAL: Restrict to specific repo and branch
            "token.actions.githubusercontent.com:sub" = "repo:novamart/*:ref:refs/heads/main"
          }
        }
      }
    ]
  })
}
```

```yaml
# GitHub Actions workflow using OIDC
name: Deploy to EKS
on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: us-east-1
          # No access keys needed!
      
      - run: aws eks update-kubeconfig --name novamart-prod
      - run: kubectl apply -f manifests/
```

```groovy
// Jenkins OIDC — using the AWS Credentials plugin
// Jenkins must have OIDC provider configured
pipeline {
  agent any
  
  environment {
    AWS_ROLE = "arn:aws:iam::123456789012:role/jenkins-deploy"
  }
  
  stages {
    stage('Deploy') {
      steps {
        withCredentials([
          [$class: 'AmazonWebServicesCredentialsBinding',
           credentialsId: 'aws-oidc-prod',
           roleArn: env.AWS_ROLE,
           roleSessionName: "jenkins-${BUILD_NUMBER}"]
        ]) {
          sh 'aws eks update-kubeconfig --name novamart-prod'
          sh 'kubectl apply -f manifests/'
        }
      }
    }
  }
}
```

---

## Part 9: IAM Debugging — When "Access Denied" Strikes

### Systematic Debugging Framework

```
"Access Denied" is the most common and most frustrating AWS error.
Here's the systematic approach:

STEP 1: IDENTIFY THE EXACT ERROR
  The error message often includes:
    - Which action was denied
    - Which resource was targeted
    - Which principal was making the call
    - An encoded authorization message (decode it!)

  # Decode the authorization failure message
  aws sts decode-authorization-message \
    --encoded-message <the-encoded-message> | jq '.DecodedMessage | fromjson'
  
  # This reveals the EXACT policy evaluation details

STEP 2: VERIFY THE PRINCIPAL
  # Who am I?
  aws

```
STEP 2: VERIFY THE PRINCIPAL
  # Who am I?
  aws sts get-caller-identity
  # Output:
  # {
  #   "UserId": "AROA...:session-name",
  #   "Account": "123456789012",
  #   "Arn": "arn:aws:sts::123456789012:assumed-role/payment-svc/session"
  # }
  
  Common surprises:
    - You're in the wrong account
    - You're using the wrong role (node role instead of IRSA)
    - Your session has expired
    - You're using cached credentials from a different profile

STEP 3: CHECK IDENTITY POLICIES
  # List all policies attached to the role
  aws iam list-attached-role-policies --role-name payment-svc
  aws iam list-role-policies --role-name payment-svc  # inline policies
  
  # Get the actual policy document
  aws iam get-role-policy --role-name payment-svc --policy-name permissions
  
  # For managed policies:
  aws iam get-policy-version \
    --policy-arn arn:aws:iam::123456789012:policy/payment-svc-policy \
    --version-id $(aws iam get-policy --policy-arn arn:aws:iam::123456789012:policy/payment-svc-policy --query 'Policy.DefaultVersionId' --output text)

STEP 4: CHECK RESOURCE POLICIES
  # S3 bucket policy
  aws s3api get-bucket-policy --bucket novamart-payment-data | jq '.Policy | fromjson'
  
  # KMS key policy
  aws kms get-key-policy --key-id <key-id> --policy-name default
  
  # SQS queue policy
  aws sqs get-queue-attributes --queue-url <url> --attribute-names Policy

STEP 5: CHECK BOUNDARIES AND SCPS
  # Permissions boundary
  aws iam get-role --role-name payment-svc | jq '.Role.PermissionsBoundary'
  
  # SCPs (must check from management account or use Organizations API)
  aws organizations list-policies-for-target \
    --target-id <account-id> \
    --filter SERVICE_CONTROL_POLICY

STEP 6: CHECK CONDITIONS
  Common condition failures:
    - SourceIP doesn't match (NAT Gateway IP vs developer's IP)
    - SourceVpc condition but calling from outside VPC
    - PrincipalTag not set on the role
    - ResourceTag not set on the target resource
    - MFA required but not present (role sessions DON'T have MFA)
    - aws:SecureTransport but using HTTP endpoint
    - VPC endpoint policy blocking the action

STEP 7: USE POLICY SIMULATOR
  # Simulate without making the actual call
  aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789012:role/payment-svc \
    --action-names s3:GetObject \
    --resource-arns arn:aws:s3:::novamart-payment-data/transactions/2024.csv
  
  # Output: "allowed" or "implicitDeny" or "explicitDeny"
  # Plus which policy caused the result

STEP 8: CHECK CLOUDTRAIL
  # Find the exact API call and its authorization details
  aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=GetObject \
    --start-time "2024-01-18T02:00:00Z" \
    --end-time "2024-01-18T03:00:00Z" | jq '.Events[0].CloudTrailEvent | fromjson'
  
  # Look at: errorCode, errorMessage, userIdentity, requestParameters
```

### The Top 10 "Access Denied" Causes at NovaMart

```
┌───┬──────────────────────────────────────┬────────────────────────────────────┐
│ # │ Cause                                │ Fix                                │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│ 1 │ S3: Missing bucket-level ARN         │ Need BOTH arn:...:bucket AND       │
│   │ (only have object-level)             │ arn:...:bucket/* in Resource       │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│ 2 │ KMS: No kms:Decrypt permission       │ S3 SSE-KMS encrypted objects       │
│   │ for encrypted objects                │ need KMS permissions too           │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│ 3 │ IRSA: Wrong namespace/SA in trust    │ Trust policy :sub must exactly     │
│   │ policy condition                     │ match system:serviceaccount:ns:sa  │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│ 4 │ SCP blocking the action              │ Check SCPs on OU and account       │
│   │                                      │ from management account            │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│ 5 │ Wrong account                        │ aws sts get-caller-identity FIRST  │
│   │                                      │ before any debugging               │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│ 6 │ VPC endpoint policy denying          │ Default is allow-all but if        │
│   │                                      │ customized, must include principal │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│ 7 │ Cross-account: identity policy       │ Cross-account needs BOTH:          │
│   │ allows but resource policy doesn't   │ caller's identity policy +         │
│   │ (or vice versa)                      │ target's resource policy           │
│   │                                      │ (unless using role assumption)     │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│ 8 │ Permissions boundary too restrictive │ Boundary must ALSO allow the       │
│   │                                      │ action (intersection model)        │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│ 9 │ Wildcard in wrong place              │ ec2:Describe* ≠ ec2:*Describe     │
│   │                                      │ Action format is exact             │
├───┼──────────────────────────────────────┼────────────────────────────────────┤
│10 │ Implicit deny mistaken for explicit  │ Check if ANY policy has matching   │
│   │ deny                                 │ Deny — different debugging path    │
└───┴──────────────────────────────────────┴────────────────────────────────────┘
```

### Production Debugging Scenario

```
SCENARIO: payment-svc pod can't read from S3 bucket novamart-payment-data

STEP 1: Confirm the error
  Pod logs show: "AccessDenied: Access Denied (Service: S3, Status Code: 403)"

STEP 2: Verify principal
  kubectl -n payments exec deploy/payment-svc -- \
    aws sts get-caller-identity
  
  EXPECTED: arn:aws:sts::123456789012:assumed-role/novamart-prod-payment-svc/...
  
  IF YOU SEE: arn:aws:sts::123456789012:assumed-role/eks-node-group-role/...
  → IRSA is NOT working. Pod is falling back to node role.
  → Check: SA annotation, pod serviceAccountName, OIDC provider

STEP 3: Simulate the call
  aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789012:role/novamart-prod-payment-svc \
    --action-names s3:GetObject \
    --resource-arns arn:aws:s3:::novamart-payment-data/transactions/2024-01.csv
  
  Result: "implicitDeny" → role policy doesn't include this action/resource
  Result: "explicitDeny" → some policy actively denies this (SCP? boundary?)

STEP 4: Check the bucket policy
  aws s3api get-bucket-policy --bucket novamart-payment-data | jq '.Policy | fromjson'
  
  Look for: Deny statements, Principal restrictions, Condition blocks
  
  COMMON TRAP: Bucket policy has:
    "Condition": {"StringEquals": {"aws:SourceVpc": "vpc-abc123"}}
  → Only allows access FROM within the VPC
  → If using a VPC endpoint, this works
  → If NOT using a VPC endpoint (e.g., NAT Gateway), SourceVpc 
    condition won't match — calls go over internet, not endpoint

STEP 5: Check KMS (if bucket uses SSE-KMS)
  aws s3api head-object --bucket novamart-payment-data \
    --key transactions/2024-01.csv
  
  Look at: ServerSideEncryption, SSEKMSKeyId
  
  If encrypted with KMS:
    Role needs BOTH s3:GetObject AND kms:Decrypt
    KMS key policy must allow the role as well
    
    aws kms get-key-policy --key-id <key-id> --policy-name default | jq .
    # Verify the payment-svc role is in the Principal list

RESOLUTION:
  Added kms:Decrypt to the role policy for the specific KMS key ARN.
  Root cause: S3 bucket encryption was changed from AES256 to SSE-KMS 
  last week, but the service role wasn't updated.
```

---

## Part 10: IAM Security Best Practices — NovaMart Checklist

```
┌──────────────────────────────────────────────────────────────────────┐
│              IAM SECURITY CHECKLIST — PRODUCTION                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│ CREDENTIALS                                                          │
│ ☐ No IAM users for service workloads (use roles everywhere)         │
│ ☐ No long-lived access keys in production accounts                   │
│ ☐ Break-glass IAM user exists with MFA + CloudWatch alarm on usage  │
│ ☐ CI/CD uses OIDC federation, not access keys                       │
│ ☐ All human access via IAM Identity Center (SSO)                    │
│ ☐ MFA enforced for all human users (in IdP)                         │
│ ☐ Access keys older than 90 days → alarm + auto-disable             │
│                                                                      │
│ LEAST PRIVILEGE                                                      │
│ ☐ No wildcard (*) actions in production policies                    │
│ ☐ No wildcard (*) resources in production policies                  │
│ ☐ IRSA configured for every EKS service (pod-level isolation)       │
│ ☐ Instance metadata blocked for pods (force IRSA usage)             │
│ ☐ IAM Access Analyzer enabled (external access findings)            │
│ ☐ IAM Access Analyzer unused access findings reviewed monthly       │
│ ☐ Permission boundaries on all delegated role creation              │
│ ☐ Quarterly access review: revoke unused permissions                │
│                                                                      │
│ ORGANIZATION                                                         │
│ ☐ Multi-account strategy (separate prod/staging/dev)                │
│ ☐ Management account has NO workloads                               │
│ ☐ SCPs enforce region restrictions                                  │
│ ☐ SCPs protect CloudTrail, GuardDuty, Config                       │
│ ☐ SCPs prevent IAM user creation in member accounts                 │
│ ☐ SCPs enforce encryption (S3, EBS, RDS)                            │
│ ☐ SCP prevents leaving the organization                             │
│                                                                      │
│ AUDIT                                                                │
│ ☐ CloudTrail enabled in ALL accounts, ALL regions                   │
│ ☐ CloudTrail logs shipped to security account (immutable)           │
│ ☐ IAM credential report generated weekly                            │
│ ☐ Root user usage → immediate alarm + investigation                 │
│ ☐ All IAM changes → CloudWatch alarm → Slack                       │
│ ☐ Cross-account role assumptions logged and reviewed                │
│                                                                      │
│ EMERGENCY                                                            │
│ ☐ Break-glass procedure documented and tested quarterly             │
│ ☐ Break-glass credentials stored securely (physical safe or         │
│   dedicated hardware token, NOT in a digital password manager       │
│   that could be compromised in the same incident)                   │
│ ☐ Root account MFA on hardware token (not SMS, not TOTP app)        │
│ ☐ Root account email is a distribution list, not personal email     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### IAM Monitoring Alerts

```yaml
# CloudWatch metric filter + alarm for root account usage
resource "aws_cloudwatch_log_metric_filter" "root_usage" {
  name           = "root-account-usage"
  pattern        = '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }'
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name          = "RootAccountUsage"
    namespace     = "SecurityMetrics"
    value         = "1"
    default_value = "0"
  }
}

resource "aws_cloudwatch_metric_alarm" "root_usage" {
  alarm_name          = "root-account-usage"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "RootAccountUsage"
  namespace           = "SecurityMetrics"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "CRITICAL: Root account used. Investigate immediately."
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

```yaml
# Alert for IAM policy changes
resource "aws_cloudwatch_log_metric_filter" "iam_changes" {
  name           = "iam-policy-changes"
  pattern        = '{ ($.eventName = "PutRolePolicy") || ($.eventName = "AttachRolePolicy") || ($.eventName = "DetachRolePolicy") || ($.eventName = "DeleteRolePolicy") || ($.eventName = "CreateRole") || ($.eventName = "DeleteRole") || ($.eventName = "AttachUserPolicy") || ($.eventName = "CreateUser") || ($.eventName = "CreateAccessKey") }'
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name          = "IAMPolicyChanges"
    namespace     = "SecurityMetrics"
    value         = "1"
    default_value = "0"
  }
}

resource "aws_cloudwatch_metric_alarm" "iam_changes" {
  alarm_name          = "iam-policy-changes"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "IAMPolicyChanges"
  namespace           = "SecurityMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "IAM policy change detected. Verify this is expected."
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

```yaml
# Alert for access key creation (should be rare/never in prod)
resource "aws_cloudwatch_log_metric_filter" "access_key_created" {
  name           = "access-key-created"
  pattern        = '{ $.eventName = "CreateAccessKey" }'
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name          = "AccessKeyCreated"
    namespace     = "SecurityMetrics"
    value         = "1"
    default_value = "0"
  }
}
```

---

## Part 11: IAM Failure Modes — Production War Stories

```
FAILURE MODE 1: Credential leak in Git
────────────────────────────────────────
  CAUSE: Developer commits .env file or hardcodes access keys
  DETECTION: GitGuardian, Trufflehog, AWS GuardDuty (detects compromised keys)
  TIMELINE: Automated scrapers find leaked keys within MINUTES of push
  RESPONSE (in this exact order):
    1. REVOKE the credentials immediately (don't investigate first)
       aws iam delete-access-key --user-name <user> --access-key-id <key>
    2. Check CloudTrail: what did the attacker do with the key?
       aws cloudtrail lookup-events \
         --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=<key-id> \
         --start-time <leak-time>
    3. Remediate any actions taken (instances launched, data accessed)
    4. Rotate ALL secrets the compromised principal had access to
    5. Remove from Git history (git-filter-repo)
    6. Add pre-commit hooks to prevent recurrence

FAILURE MODE 2: Overprivileged role exploited
──────────────────────────────────────────────
  CAUSE: Service role has "s3:*" on Resource "*" 
         → attacker compromises service → accesses ALL S3 data
  DETECTION: GuardDuty anomaly detection, CloudTrail unusual API patterns
  PREVENTION: Least privilege + Access Analyzer + regular reviews
  EXAMPLE: payment-svc compromised via SSRF vulnerability
    → With broad role: attacker accesses customer PII in ANY bucket
    → With least privilege: attacker can only access payment-specific data
    → Blast radius reduced from "everything" to "one service's data"

FAILURE MODE 3: SCP locks out the account
──────────────────────────────────────────
  CAUSE: Overly broad SCP deny without admin exception
  SYMPTOM: NOBODY in the account can perform ANY action
  RECOVERY: Must fix from management account (member account is locked)
  PREVENTION: 
    - Always include ArnNotLike exception for break-glass role
    - Test SCPs in sandbox OU first
    - Use "Allow" SCPs as guardrails (allow list) rather than 
      "Deny everything then allow specific things"

FAILURE MODE 4: IAM propagation delay
─────────────────────────────────────────
  CAUSE: IAM is eventually consistent (global service)
  SYMPTOM: Policy change made but still getting Access Denied
  TIMELINE: Usually seconds, but can take up to 60 seconds
  GOTCHA: Terraform apply succeeds (policy created) but immediately 
          testing fails (policy not propagated yet)
  FIX: Add sleep/retry in automation, or use waiters
  NOTE: This is especially painful in CI pipelines — create role, 
        immediately assume it → fails

FAILURE MODE 5: Resource policy overrides everything (S3 public)
────────────────────────────────────────────────────────────────
  CAUSE: S3 bucket policy with Principal: "*" 
  EFFECT: The bucket is publicly accessible regardless of IAM policies
  DETECTION: IAM Access Analyzer, S3 Block Public Access settings
  PREVENTION:
    - Account-level S3 Block Public Access (enabled by default now)
    - SCP to prevent disabling Block Public Access:
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDisablingS3PublicAccessBlock",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketPublicAccessBlock",
        "s3:PutAccountPublicAccessBlock"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/SecurityAdmin"
        }
      }
    }
  ]
}
```

```
FAILURE MODE 6: Tag-based ABAC bypass
─────────────────────────────────────────
  CAUSE: User can tag resources → changes their own access scope
  EXAMPLE: Policy says "access resources tagged team=yours"
           User tags a sensitive resource with their team tag
           Now they have access to sensitive resource
  PREVENTION: Separate "who can tag" from "who can access tagged resources"
  FIX: Deny tagging permissions or restrict tag key/value modifications:

FAILURE MODE 7: Cross-account role chaining confusion
─────────────────────────────────────────────────────
  CAUSE: Role A assumes Role B assumes Role C
  GOTCHA: Session duration is limited to 1 hour for chained assumptions
          (even if Role C allows 12-hour sessions)
  SYMPTOM: Long-running jobs fail after 1 hour
  FIX: Minimize role chaining, or use direct trust relationships

FAILURE MODE 8: Service-linked role deletion blocks service
──────────────────────────────────────────────────────────
  CAUSE: Someone deletes AWSServiceRoleForElasticLoadBalancing
  SYMPTOM: Can't create new ALBs
  FIX: AWS recreates it automatically when needed, but 
       there can be a delay and confusing errors in between
  PREVENTION: SCP to deny deletion of service-linked roles
```


## Quick Reference Card

```
IAM AUTHORIZATION FLOW
──────────────────────
Explicit Deny > Explicit Allow > Implicit Deny (default)
ALL gates must allow: SCP ∩ Boundary ∩ Identity ∩ Session ∩ Resource policy
Any single Deny = DENIED. Period.

PRINCIPAL TYPES
───────────────
{"AWS": "arn:..."}          → IAM user/role/account
{"Service": "ec2.amazonaws.com"}  → AWS service
{"Federated": "arn:...oidc/..."} → OIDC provider (IRSA, GitHub)
{"Federated": "arn:...saml/..."} → SAML provider (SSO)

IRSA CHECKLIST
──────────────
1. OIDC provider registered for EKS cluster
2. IAM role trust policy: sts:AssumeRoleWithWebIdentity
   Condition: :sub = system:serviceaccount:<ns>:<sa>
3. K8s ServiceAccount: eks.amazonaws.com/role-arn annotation
4. Deployment: serviceAccountName references the SA
5. Block IMDS (169.254.169.254) to force IRSA usage

DEBUGGING "ACCESS DENIED"
─────────────────────────
1. aws sts get-caller-identity (WHO am I?)
2. aws sts decode-authorization-message (WHAT was denied?)
3. Check identity policy (actions + resources)
4. Check resource policy (principal + conditions)
5. Check SCP and permissions boundary
6. Check conditions (SourceVpc, tags, MFA, encryption)
7. aws iam simulate-principal-policy (test without doing)
8. CloudTrail (find the exact failed call)

CROSS-ACCOUNT
─────────────
Role assumption: Need trust policy (target) + identity policy (caller)
Resource policy: Need resource policy with cross-account Principal
  + aws:PrincipalOrgID condition (prevent access from outside org)
ExternalId: ALWAYS use for third-party access (confused deputy)

SCPS — REMEMBER
───────────────
SCPs don't GRANT — they RESTRICT (ceiling, not floor)
Management account is NOT bound by SCPs
ALWAYS exempt a break-glass role in Deny SCPs
Test in sandbox OU before applying to production OU

SECURITY ALARMS (must-have)
──────────────────────────
Root account usage → IMMEDIATE page
IAM policy changes → Slack notification
Access key creation → Slack notification  
S3 public access change → IMMEDIATE page
CloudTrail stopped → IMMEDIATE page
```


---

## Retention Questions — Phase 6, Lesson 1

### Q1: IRSA Debugging Under Pressure 🔥

**Scenario:** Monday morning. payment-svc is throwing `AccessDenied` errors when trying to read from SQS queue `payment-processing`. This worked fine last week. No code changes were deployed to payment-svc. The on-call engineer tried restarting the pods — no change.

1. Walk through your debugging steps in order, with exact commands. Start from confirming the principal identity through to root cause.
2. During debugging, you discover `aws sts get-caller-identity` from inside the pod returns the **node instance role**, not the IRSA role. List ALL possible causes for this (at least 5) and the check command for each.
3. The root cause turns out to be: a Terraform apply last Friday updated the EKS OIDC provider thumbprint but introduced a typo. The OIDC validation is now failing silently, so ALL IRSA roles in the cluster are broken — pods fall back to node role. How many services are affected and what's your mitigation plan? This is now a SEV1.
4. Write the SCP that would prevent someone from accidentally deleting or modifying the OIDC provider configuration.

### Q2: Policy Evaluation Challenge 🔥

**Scenario:** A NovaMart engineer reports that `order-svc` running in the production account (111111111111) cannot write to S3 bucket `novamart-order-archive` in the log archive account (555555555555).

The following policies exist:

**Identity policy on order-svc role (production account):**
```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject"],
  "Resource": "arn:aws:s3:::novamart-order-archive/*"
}
```

**S3 bucket policy (log archive account):**
```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::111111111111:root"},
  "Action": ["s3:PutObject"],
  "Resource": "arn:aws:s3:::novamart-order-archive/*",
  "Condition": {
    "StringEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

**SCP on production account's OU:**
```json
{
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": ["aws:kms", "AES256"]
    },
    "Null": {
      "s3:x-amz-server-side-encryption": "false"
    }
  }
}
```

1. The engineer is uploading with `--server-side-encryption aws:kms`. Should this work? Trace through the **complete** policy evaluation chain and explain the result at each gate.
2. The engineer then tries without any encryption flag. What happens? Trace again.
3. The engineer's upload with `aws:kms` is STILL failing. Name three things outside the policies shown above that could cause this. For each, give the exact debugging command.
4. Fix the cross-account setup properly. Show the corrected policies AND explain why `aws:PrincipalOrgID` should be added.

### Q3: Organization Design Under Constraints 🔥

**Scenario:** NovaMart is undergoing PCI-DSS audit. The auditor asks:

"Show me that your payment processing system is isolated from non-payment workloads, that developers cannot access production payment data, and that all access to the payment environment is logged and immutable."

1. Design the AWS account structure that satisfies this requirement. Show which services run where and explain WHY the isolation matters for PCI scope.
2. Write the specific SCPs for the payment production account that enforce: (a) no IAM user creation, (b) no access from outside the organization, (c) CloudTrail cannot be disabled, (d) only approved regions.
3. The auditor asks: "Can a developer with staging access pivot to production payment data?" Walk through exactly how your design prevents this, referencing specific IAM controls at each boundary.
4. The current break-glass procedure stores the root account MFA seed in a password manager shared by 4 senior engineers. The auditor flags this. What's wrong and what's the fix?

### Q4: Blast Radius Scenario 🔥

**Scenario:** An attacker has compromised a pod running `notification-svc` through a Server-Side Request Forgery (SSRF) vulnerability. The attacker can execute arbitrary AWS API calls using the pod's credentials.

1. With NovaMart's IRSA setup, what is the attacker's blast radius? What CAN and CANNOT they access? Be specific about what `notification-svc` would typically need access to.
2. If IRSA was NOT configured and all pods used the node instance role, what additional access would the attacker have? Why is this catastrophically worse?
3. The attacker tries to escalate privileges. List three specific escalation paths they might attempt and the IAM controls that block each one.
4. You detect this compromise via GuardDuty. Write the first 5 actions of your incident response, in order, with exact commands. Remember: this is a security incident, not just an availability incident.

---

# Phase 6, Lesson 1 — Security, Compliance, AWS Services

---

## Q1: IRSA Debugging Under Pressure

### 1. Debugging Steps — In Order, With Exact Commands

**Step 1: Confirm what identity the pod is using RIGHT NOW**

```bash
# Exec into a payment-svc pod
kubectl -n payments exec -it deploy/payment-svc -- /bin/sh

# Check the identity
aws sts get-caller-identity
```

**Expected (working IRSA):**
```json
{
  "UserId": "AROA1234567890EXAMPLE:botocore-session-1234567890",
  "Account": "111111111111",
  "Arn": "arn:aws:sts::111111111111:assumed-role/payment-svc-irsa-role/botocore-session-1234567890"
}
```

**What you actually get (broken IRSA):**
```json
{
  "UserId": "AROA1234567890EXAMPLE:i-0abc123def456789",
  "Account": "111111111111",
  "Arn": "arn:aws:sts::111111111111:assumed-role/eks-node-role/i-0abc123def456789"
}
```

**This tells us:** The pod is using the **node instance role**, not the IRSA role. The STS AssumeRoleWithWebIdentity flow is failing somewhere.

**Step 2: Verify the IRSA environment variables are present in the pod**

```bash
# Still inside the pod
env | grep AWS_

# Expected:
# AWS_ROLE_ARN=arn:aws:iam::111111111111:role/payment-svc-irsa-role
# AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
# AWS_DEFAULT_REGION=us-east-1 (maybe)

# If these are MISSING → ServiceAccount annotation or webhook issue
# If these are PRESENT → the AssumeRoleWithWebIdentity call is failing
```

**Step 3: Verify the projected token file exists and is valid**

```bash
# Check the token file exists
ls -la /var/run/secrets/eks.amazonaws.com/serviceaccount/token

# Decode the JWT token (it's a base64-encoded JWT)
cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | cut -d. -f2 | base64 -d 2>/dev/null | jq .
```

**Expected JWT payload:**
```json
{
  "aud": ["sts.amazonaws.com"],
  "exp": 1705401600,
  "iat": 1705315200,
  "iss": "https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE",
  "kubernetes.io": {
    "namespace": "payments",
    "pod": {"name": "payment-svc-abc123"},
    "serviceaccount": {"name": "payment-svc-sa", "uid": "..."}
  },
  "sub": "system:serviceaccount:payments:payment-svc-sa"
}
```

**Check:**
- `iss` — matches the cluster's OIDC issuer URL?
- `aud` — includes `sts.amazonaws.com`?
- `exp` — not expired?
- `sub` — matches the ServiceAccount name?

**Step 4: Verify the ServiceAccount has the IRSA annotation**

```bash
# Outside the pod (kubectl context)
kubectl -n payments get sa payment-svc-sa -o yaml
```

**Expected:**
```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111111111111:role/payment-svc-irsa-role
```

**If missing:** The mutating webhook won't inject `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE`.

**Step 5: Verify the IAM role's trust policy allows this ServiceAccount**

```bash
aws iam get-role --role-name payment-svc-irsa-role --query 'Role.AssumeRolePolicyDocument' | jq .
```

**Expected trust policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::111111111111:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:payments:payment-svc-sa",
        "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
      }
    }
  }]
}
```

**Check:** Does the `Federated` principal match the cluster's actual OIDC provider ARN?

**Step 6: Verify the OIDC provider exists in IAM and has the correct thumbprint**

```bash
# Get the cluster's OIDC issuer URL
aws eks describe-cluster --name novamart-prod --query 'cluster.identity.oidc.issuer' --output text
# Output: https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE

# Extract the OIDC ID
OIDC_ID=$(aws eks describe-cluster --name novamart-prod --query 'cluster.identity.oidc.issuer' --output text | sed 's|https://||')

# Check if the OIDC provider exists in IAM
aws iam list-open-id-connect-providers | jq '.OpenIDConnectProviderList[] | select(.Arn | contains("EXAMPLED539D4633E53DE1B71EXAMPLE"))'

# Get the provider details including thumbprint
aws iam get-open-id-connect-provider --open-id-connect-provider-arn \
  "arn:aws:iam::111111111111:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
```

**Check the thumbprint:**
```bash
# Get the ACTUAL current thumbprint from the OIDC endpoint
OIDC_URL="oidc.eks.us-east-1.amazonaws.com"
echo | openssl s_client -servername "$OIDC_URL" -connect "$OIDC_URL:443" 2>/dev/null | \
  openssl x509 -fingerprint -sha1 -noout | \
  sed 's/://g' | awk -F= '{print tolower($2)}'

# Compare with what IAM has stored:
aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn "arn:aws:iam::111111111111:oidc-provider/$OIDC_ID" \
  --query 'ThumbprintList[0]' --output text
```

**If these don't match → OIDC validation fails → AssumeRoleWithWebIdentity silently fails → falls back to node role. THIS IS THE ROOT CAUSE.**

**Step 7: Attempt the AssumeRoleWithWebIdentity manually to see the exact error**

```bash
# Inside the pod:
aws sts assume-role-with-web-identity \
  --role-arn "$AWS_ROLE_ARN" \
  --role-session-name "debug-session" \
  --web-identity-token "$(cat $AWS_WEB_IDENTITY_TOKEN_FILE)" \
  --duration-seconds 3600 2>&1

# If OIDC thumbprint is wrong, you'll get:
# "An error occurred (InvalidIdentityToken) when calling the
#  AssumeRoleWithWebIdentity operation: Couldn't retrieve verification
#  key from your identity provider, please reference
#  AssumeRoleWithWebIdentity documentation for requirements"
```

**This is the smoking gun.** The error message is specific and points directly to the OIDC provider configuration.

### 2. ALL Possible Causes When get-caller-identity Returns Node Role (≥5)

```
┌───┬─────────────────────────────────────────────┬──────────────────────────────────────────┐
│ # │ Cause                                       │ Check Command                            │
├───┼─────────────────────────────────────────────┼──────────────────────────────────────────┤
│ 1 │ IRSA env vars not injected — mutating       │ kubectl -n payments exec deploy/          │
│   │ admission webhook (pod-identity-webhook)     │ payment-svc -- env | grep AWS_            │
│   │ not running or not configured                │                                          │
│   │                                             │ kubectl -n kube-system get pods -l        │
│   │                                             │   app=eks-pod-identity-webhook            │
│   │                                             │ kubectl get mutatingwebhookconfigurations │
│   │                                             │   | grep pod-identity                     │
├───┼─────────────────────────────────────────────┼──────────────────────────────────────────┤
│ 2 │ ServiceAccount missing IRSA annotation       │ kubectl -n payments get sa payment-svc-sa │
│   │ (eks.amazonaws.com/role-arn)                 │   -o jsonpath='{.metadata.annotations}'  │
│   │                                             │                                          │
│   │ Pods created BEFORE annotation was added     │ Check pod creation time vs annotation    │
│   │ won't have the env vars — need pod restart   │ time. Env vars are injected at pod       │
│   │                                             │ creation, not dynamically.               │
├───┼─────────────────────────────────────────────┼──────────────────────────────────────────┤
│ 3 │ IAM OIDC provider thumbprint mismatch       │ # Get actual thumbprint:                 │
│   │ (Terraform typo, cert rotation, AWS change) │ echo | openssl s_client -servername       │
│   │                                             │   oidc.eks.us-east-1.amazonaws.com       │
│   │                                             │   -connect oidc.eks...:443 2>/dev/null   │
│   │                                             │   | openssl x509 -fingerprint -sha1      │
│   │                                             │   -noout                                 │
│   │                                             │ # Get stored thumbprint:                 │
│   │                                             │ aws iam get-open-id-connect-provider     │
│   │                                             │   --oidc-arn <arn> --query Thumbprint*   │
├───┼─────────────────────────────────────────────┼──────────────────────────────────────────┤
│ 4 │ Trust policy condition mismatch — sub or    │ aws iam get-role --role-name              │
│   │ aud doesn't match the token claims          │   payment-svc-irsa-role --query           │
│   │                                             │   Role.AssumeRolePolicyDocument          │
│   │ Common: namespace or SA name typo in        │                                          │
│   │ condition, or aud is "sts.amazonaws.com"    │ # Decode token to compare:               │
│   │ in token but trust policy expects           │ cat /var/run/.../token | cut -d. -f2 |   │
│   │ something else                              │   base64 -d | jq '.sub, .aud'           │
├───┼─────────────────────────────────────────────┼──────────────────────────────────────────┤
│ 5 │ OIDC provider deleted or recreated with     │ aws iam list-open-id-connect-providers   │
│   │ different ARN — trust policy references     │   | jq .                                 │
│   │ old ARN that no longer exists               │                                          │
│   │                                             │ # Compare the OIDC ID in the trust       │
│   │                                             │ # policy vs the actual cluster OIDC ID:  │
│   │                                             │ aws eks describe-cluster --name           │
│   │                                             │   novamart-prod --query                  │
│   │                                             │   cluster.identity.oidc.issuer           │
├───┼─────────────────────────────────────────────┼──────────────────────────────────────────┤
│ 6 │ Projected service account token volume      │ kubectl -n payments get pod               │
│   │ not mounted — pod spec missing the volume   │   payment-svc-xyz -o yaml | grep -A 10   │
│   │ or the webhook failed to inject it          │   "projected"                            │
│   │                                             │                                          │
│   │                                             │ # Check inside pod:                      │
│   │                                             │ ls -la /var/run/secrets/                 │
│   │                                             │   eks.amazonaws.com/serviceaccount/      │
├───┼─────────────────────────────────────────────┼──────────────────────────────────────────┤
│ 7 │ Token expired and kubelet not refreshing    │ # Inside the pod — check token expiry:   │
│   │ it (rare, usually kubelet bug or clock      │ cat /var/run/.../token | cut -d. -f2 |   │
│   │ skew on the node)                           │   base64 -d | jq '.exp' |               │
│   │                                             │   xargs -I{} date -d @{}                │
│   │                                             │                                          │
│   │                                             │ # Check node time sync:                  │
│   │                                             │ kubectl debug node/<node> -it             │
│   │                                             │   --image=busybox -- date                │
├───┼─────────────────────────────────────────────┼──────────────────────────────────────────┤
│ 8 │ AWS SDK version too old — doesn't support   │ # Inside the pod:                        │
│   │ AssumeRoleWithWebIdentity credential chain  │ python3 -c "import boto3;                │
│   │ (older SDKs skip the IRSA credential        │   print(boto3.__version__)"              │
│   │ provider entirely)                          │                                          │
│   │                                             │ # AWS SDK for Python needs >= 1.21.0     │
│   │                                             │ # AWS CLI needs >= 1.16.232              │
│   │                                             │ aws --version                            │
├───┼─────────────────────────────────────────────┼──────────────────────────────────────────┤
│ 9 │ Container overrides AWS_* env vars —        │ # Check deployment spec for hardcoded    │
│   │ hardcoded AWS_ACCESS_KEY_ID or              │ # credentials that take precedence:      │
│   │ AWS_PROFILE in the deployment spec          │ kubectl -n payments get deploy             │
│   │ takes precedence over IRSA token chain      │   payment-svc -o yaml | grep -i          │
│   │                                             │   "AWS_ACCESS_KEY\|AWS_SECRET\|          │
│   │                                             │    AWS_PROFILE\|AWS_DEFAULT_PROFILE"     │
└───┴─────────────────────────────────────────────┴──────────────────────────────────────────┘
```

### 3. OIDC Thumbprint Typo — SEV1 Response

**Blast radius assessment:**

```bash
# How many services use IRSA? Count all ServiceAccounts with IRSA annotation
kubectl get sa --all-namespaces -o json | \
  jq '[.items[] | select(.metadata.annotations["eks.amazonaws.com/role-arn"] != null) | {namespace: .metadata.namespace, name: .metadata.name, role: .metadata.annotations["eks.amazonaws.com/role-arn"]}]'

# Count affected pods
kubectl get pods --all-namespaces -o json | \
  jq '[.items[] | select(.spec.volumes[]? | .projected?.sources[]? | .serviceAccountToken?.audience == "sts.amazonaws.com") | {namespace: .metadata.namespace, name: .metadata.name}] | length'
```

**EVERY pod using IRSA across the ENTIRE cluster is now using the node role instead of its scoped role.** This means:

```
Impact:
├── payment-svc:     Using node role → may lack SQS permissions → 503s on payments
├── order-svc:       Using node role → may lack RDS/S3 permissions → order failures
├── fraud-svc:       Using node role → may lack SageMaker permissions → fraud checks skipped
├── notification-svc: Using node role → may lack SES/SNS permissions → silent notification failure
├── ALL services:    Now have node role permissions → OVER-PRIVILEGED
│                    If node role has broad EC2/EKS permissions, every pod
│                    can now access things it shouldn't (security breach)
└── Duration:        Since Friday → 3 days of broken IRSA → some services
                     may have been silently failing or silently over-privileged
```

**Mitigation plan:**

**Minute 0-2: Declare SEV1 and communicate**

```
🔴 SEV1 DECLARED — Cluster-wide IRSA failure

Impact: ALL pods using IRSA are falling back to node instance role.
- Some services are losing access to required AWS resources (AccessDenied)
- ALL services are over-privileged (using node role instead of scoped role)
- This has been in effect since Friday's Terraform apply
Root cause: OIDC provider thumbprint typo in Terraform

IC: [your name]
War room: #inc-20240116-irsa-cluster
Immediate action: fixing the OIDC thumbprint

@security-team ALERT: All pods have been running with node instance role 
since Friday. Potential privilege escalation window. Need CloudTrail audit 
of all API calls from the node role ARN since Friday.
```

**Minute 2-5: Fix the OIDC provider thumbprint**

```bash
# Get the correct thumbprint
OIDC_URL="oidc.eks.us-east-1.amazonaws.com"
CORRECT_THUMBPRINT=$(echo | openssl s_client -servername "$OIDC_URL" \
  -connect "$OIDC_URL:443" 2>/dev/null | \
  openssl x509 -fingerprint -sha1 -noout | \
  sed 's/://g' | awk -F= '{print tolower($2)}')

echo "Correct thumbprint: $CORRECT_THUMBPRINT"

# Get the OIDC provider ARN
OIDC_ARN=$(aws iam list-open-id-connect-providers --query \
  'OpenIDConnectProviderList[?contains(Arn, `eks`)].Arn' --output text)

# Update the thumbprint
aws iam update-open-id-connect-provider-thumbprint \
  --open-id-connect-provider-arn "$OIDC_ARN" \
  --thumbprint-list "$CORRECT_THUMBPRINT"

# Verify the update
aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn "$OIDC_ARN" \
  --query 'ThumbprintList'
# Should show the correct thumbprint
```

**⚠️ Why fix directly via AWS CLI instead of Terraform:** Terraform apply will fix it, but terraform plan + review + apply takes 10-20 minutes. Direct CLI fix takes 30 seconds. We'll reconcile Terraform state after the incident. Every minute of broken IRSA = broken payments = $50K/min.

**Minute 5-8: Restart pods to force new token acquisition**

```bash
# IRSA tokens are cached by the SDK. Pods need to re-attempt
# AssumeRoleWithWebIdentity with the now-valid OIDC provider.
# Most SDKs refresh credentials automatically, but to be sure:

# Restart critical services first (revenue-impacting)
kubectl -n payments rollout restart deploy/payment-svc
kubectl -n payments rollout status deploy/payment-svc --timeout=120s

kubectl -n orders rollout restart deploy/order-svc
kubectl -n orders rollout status deploy/order-svc --timeout=120s

# Then restart all other IRSA-dependent workloads
for ns in $(kubectl get sa --all-namespaces -o json | \
  jq -r '[.items[] | select(.metadata.annotations["eks.amazonaws.com/role-arn"] != null) | .metadata.namespace] | unique[]'); do
  echo "Restarting deployments in namespace: $ns"
  kubectl -n "$ns" rollout restart deploy
done
```

**Minute 8-12: Verify IRSA is working**

```bash
# Check payment-svc identity
kubectl -n payments exec deploy/payment-svc -- aws sts get-caller-identity
# Should show: assumed-role/payment-svc-irsa-role/...

# Check order-svc identity
kubectl -n orders exec deploy/order-svc -- aws sts get-caller-identity
# Should show: assumed-role/order-svc-irsa-role/...

# Test the actual SQS access that was failing
kubectl -n payments exec deploy/payment-svc -- \
  aws sqs receive-message --queue-url https://sqs.us-east-1.amazonaws.com/111111111111/payment-processing --max-number-of-messages 1
# Should succeed
```

**Minute 12-30: Security audit**

```bash
# CRITICAL: Audit what happened during the 3-day window
# All pods were using the node role — check for unauthorized API calls

# Get the node role ARN
NODE_ROLE_ARN="arn:aws:iam::111111111111:role/eks-node-role"

# Search CloudTrail for API calls from the node role that
# don't match normal node behavior
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::IAM::Role \
  --start-time "$(date -u -d '3 days ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --query 'Events[?contains(CloudTrailEvent, `eks-node-role`)]' | head -100

# More targeted: look for unusual API calls from the node role
# that would indicate exploitation
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue \
  --start-time "$(date -u -d '3 days ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --query 'Events[?contains(CloudTrailEvent, `eks-node-role`)]'

# Check for any IAM modifications during the window
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=iam.amazonaws.com \
  --start-time "$(date -u -d '3 days ago' +%Y-%m-%dT%H:%M:%SZ)" | \
  jq '.Events[] | {time: .EventTime, event: .EventName, user: .Username}'
```

**Minute 30+: Fix Terraform to prevent recurrence**

```bash
# Fix the Terraform code to use the correct thumbprint
# Better: use the aws_eks_cluster data source to derive it automatically

# In Terraform, DON'T hardcode the thumbprint:
```

```hcl
# WRONG — hardcoded thumbprint that can have typos
resource "aws_iam_openid_connect_provider" "eks" {
  url             = aws_eks_cluster.novamart.identity[0].oidc[0].issuer
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["9e99a48a9960b14926bb7f3b02e22da2b0ab7280"]  # TYPO HERE
}

# CORRECT — dynamically computed thumbprint
data "tls_certificate" "eks" {
  url = aws_eks_cluster.novamart.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  url             = aws_eks_cluster.novamart.identity[0].oidc[0].issuer
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
}
```

```bash
# Apply the fix
cd terraform/eks
terraform plan -target=aws_iam_openid_connect_provider.eks
# Verify the plan shows the correct thumbprint
terraform apply -target=aws_iam_openid_connect_provider.eks
```

### 4. SCP to Protect the OIDC Provider

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectEKSOIDCProvider",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteOpenIDConnectProvider",
        "iam:UpdateOpenIDConnectProviderThumbprint",
        "iam:RemoveClientIDFromOpenIDConnectProvider"
      ],
      "Resource": "arn:aws:iam::*:oidc-provider/oidc.eks.*.amazonaws.com/*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/NovaMartTerraformRole",
            "arn:aws:iam::*:role/BreakGlassRole"
          ]
        }
      }
    }
  ]
}
```

**Why this SCP works:**
- **Denies** deletion, thumbprint updates, and client ID removal on any EKS OIDC provider
- **Condition exempts** only the Terraform automation role and break-glass role
- Even an account admin cannot accidentally modify the OIDC provider
- Applied at the OU level, covers all accounts with EKS clusters

**What it doesn't catch:** The Terraform role itself making the typo (which is what happened). For that, add a **CI-level check:**

```bash
# In the Terraform CI pipeline, add a post-plan validation step:
# verify_oidc_thumbprint.sh

#!/usr/bin/env bash
set -euo pipefail

PLAN_FILE="tfplan.json"
terraform show -json tfplan > "$PLAN_FILE"

# Check if the plan modifies the OIDC provider thumbprint
OIDC_CHANGES=$(jq '[.resource_changes[] | select(.type == "aws_iam_openid_connect_provider") | select(.change.actions | contains(["update"]))] | length' "$PLAN_FILE")

if [ "$OIDC_CHANGES" -gt 0 ]; then
  echo "⚠️  OIDC provider thumbprint change detected!"
  echo "This affects ALL IRSA roles in the cluster."
  echo "Changes:"
  jq '.resource_changes[] | select(.type == "aws_iam_openid_connect_provider") | .change.after.thumbprint_list' "$PLAN_FILE"
  
  # Auto-verify the thumbprint against the live endpoint
  PLANNED_THUMBPRINT=$(jq -r '.resource_changes[] | select(.type == "aws_iam_openid_connect_provider") | .change.after.thumbprint_list[0]' "$PLAN_FILE")
  
  OIDC_URL=$(jq -r '.resource_changes[] | select(.type == "aws_iam_openid_connect_provider") | .change.after.url' "$PLAN_FILE" | sed 's|https://||')
  
  ACTUAL_THUMBPRINT=$(echo | openssl s_client -servername "$OIDC_URL" -connect "$OIDC_URL:443" 2>/dev/null | openssl x509 -fingerprint -sha1 -noout | sed 's/://g' | awk -F= '{print tolower($2)}')
  
  if [ "$PLANNED_THUMBPRINT" != "$ACTUAL_THUMBPRINT" ]; then
    echo "❌ THUMBPRINT MISMATCH!"
    echo "Planned:  $PLANNED_THUMBPRINT"
    echo "Actual:   $ACTUAL_THUMBPRINT"
    echo "This will break ALL IRSA roles. Blocking apply."
    exit 1
  fi
  
  echo "✅ Thumbprint matches live endpoint."
fi
```

---

## Q2: Policy Evaluation Challenge

### 1. Upload With `--server-side-encryption aws:kms` — Should This Work?

**Tracing through the complete IAM policy evaluation chain:**

```
Cross-account access evaluation flow:
  Request: s3:PutObject to novamart-order-archive/*
  From: order-svc role in account 111111111111 (production)
  To: S3 bucket in account 555555555555 (log archive)
  With: x-amz-server-side-encryption = aws:kms

Gate 1: SCP (applied to production account's OU)
  ├── SCP Deny rule checks:
  │   Condition: StringNotEquals s3:x-amz-server-side-encryption: ["aws:kms", "AES256"]
  │             AND Null s3:x-amz-server-side-encryption: "false"
  │
  │   The request HAS s3:x-amz-server-side-encryption = "aws:kms"
  │   → Null check: "false" means "deny if the key IS present AND doesn't match"
  │   → StringNotEquals: "aws:kms" IS in the allowed list ["aws:kms", "AES256"]
  │   → Condition evaluates to FALSE (not-equals fails because it equals)
  │   → Deny does NOT apply
  │
  └── Result: ✅ ALLOWED to pass through

Gate 2: Identity policy (on order-svc role, production account)
  ├── Effect: Allow
  │   Action: s3:PutObject ← matches
  │   Resource: arn:aws:s3:::novamart-order-archive/* ← matches
  │   No conditions
  │
  └── Result: ✅ ALLOWED by identity policy

Gate 3: Resource policy (S3 bucket policy, log archive account)
  ├── Effect: Allow
  │   Principal: arn:aws:iam::111111111111:root ← matches (any principal in account 111111111111)
  │   Action: s3:PutObject ← matches
  │   Resource: arn:aws:s3:::novamart-order-archive/* ← matches
  │   Condition: StringEquals s3:x-amz-server-side-encryption: "aws:kms"
  │   → Request has "aws:kms" → ✅ condition met
  │
  └── Result: ✅ ALLOWED by bucket policy

Cross-account rule: For cross-account access, BOTH the identity policy
AND the resource policy must explicitly allow the action.
  Identity policy: ✅ Allow
  Bucket policy: ✅ Allow

Final result: ✅ THIS SHOULD WORK
```

### 2. Upload Without Any Encryption Flag

```
Request: s3:PutObject to novamart-order-archive/*
With: NO x-amz-server-side-encryption header

Gate 1: SCP
  ├── Condition: StringNotEquals s3:x-amz-server-side-encryption: ["aws:kms", "AES256"]
  │             AND Null s3:x-amz-server-side-encryption: "false"
  │
  │   The request does NOT have s3:x-amz-server-side-encryption at all
  │   → Null check: "false" means "deny when the key IS present"
  │   → Since the key is ABSENT (null), Null:"false" evaluates to FALSE
  │   → The AND condition requires BOTH to be true
  │   → One is false → entire condition is FALSE
  │   → Deny does NOT apply
  │
  │   WAIT — let me re-read this more carefully.
  │
  │   The SCP is:
  │   {
  │     "Effect": "Deny",
  │     "Condition": {
  │       "StringNotEquals": {
  │         "s3:x-amz-server-side-encryption": ["aws:kms", "AES256"]
  │       },
  │       "Null": {
  │         "s3:x-amz-server-side-encryption": "false"
  │       }
  │     }
  │   }
  │
  │   The Null condition with value "false" means:
  │   "Condition is true when the key is NOT null (i.e., IS present)"
  │
  │   When the encryption header is ABSENT:
  │   → Null:"false" → "Is the key NOT null?" → No, it IS null → FALSE
  │   → Since both conditions must be true (AND), and one is false:
  │   → Deny does NOT apply
  │
  └── Result: ✅ SCP allows it through (doesn't deny)

Gate 2: Identity policy
  ├── s3:PutObject to arn:aws:s3:::novamart-order-archive/*
  │   No conditions → matches
  │
  └── Result: ✅ ALLOWED

Gate 3: Bucket policy
  ├── Condition: StringEquals s3:x-amz-server-side-encryption: "aws:kms"
  │   Request has NO encryption header
  │   → StringEquals fails → condition not met
  │   → This Allow statement does NOT match
  │
  │   Are there other statements? Only this one shown.
  │   No matching Allow in bucket policy.
  │
  └── Result: ❌ NO ALLOW — implicit deny

Cross-account rule: Both must allow.
  Identity policy: ✅ Allow
  Bucket policy: ❌ No matching Allow (condition failed)

Final result: ❌ ACCESS DENIED
```

**The request without encryption is denied by the bucket policy's condition, not the SCP.** The SCP actually has a gap — it allows unencrypted uploads through because the `Null:"false"` condition means it only evaluates the `StringNotEquals` when the header IS present. To fix this, the SCP should have a second statement:

```json
{
  "Sid": "DenyUnencryptedUploads",
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "*",
  "Condition": {
    "Null": {
      "s3:x-amz-server-side-encryption": "true"
    }
  }
}
```

### 3. Upload WITH aws:kms Still Failing — Three Non-Policy Causes

**Cause 1: KMS key cross-account access not configured**

When uploading with `aws:kms`, the S3 service calls KMS to generate a data encryption key. If the request doesn't specify a KMS key ID, S3 uses the bucket's default KMS key (or the AWS-managed `aws/s3` key). If a CMK is required, the order-svc role in account 111111111111 needs `kms:GenerateDataKey` permission on the KMS key in account 555555555555.

```bash
# Check what KMS key the bucket uses by default
aws s3api get-bucket-encryption \
  --bucket novamart-order-archive \
  --profile log-archive-account
# Look for: KMSMasterKeyID

# Check if order-svc role can use that KMS key
aws kms describe-key --key-id <key-id-from-above> --profile log-archive-account
# Check the key policy for cross-account access

# Test KMS access directly
aws kms generate-data-key \
  --key-id <key-id> \
  --key-spec AES_256 \
  --profile production-account
# If AccessDenied → KMS key policy doesn't allow cross-account
```

**Fix: Add to the KMS key policy in the log archive account:**

```json
{
  "Sid": "AllowProductionAccountEncrypt",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::111111111111:role/order-svc-irsa-role"
  },
  "Action": [
    "kms:GenerateDataKey",
    "kms:Decrypt"
  ],
  "Resource": "*"
}
```

**AND add to the order-svc identity policy:**

```json
{
  "Effect": "Allow",
  "Action": [
    "kms:GenerateDataKey",
    "kms:Decrypt"
  ],
  "Resource": "arn:aws:kms:us-east-1:555555555555:key/<key-id>"
}
```

**Cause 2: S3 VPC Endpoint policy blocking the request**

If the production VPC uses an S3 VPC Gateway Endpoint (which it should, for security), the endpoint may have a policy that doesn't allow access to buckets in other accounts.

```bash
# Find the S3 VPC endpoint
aws ec2 describe-vpc-endpoints \
  --filters "Name=service-name,Values=com.amazonaws.us-east-1.s3" \
  --query 'VpcEndpoints[?VpcId==`vpc-prod-id`].{Id:VpcEndpointId, Policy:PolicyDocument}'

# Check if the endpoint policy allows access to the cross-account bucket
# Default policy is "Allow all" but it's often restricted in production
```

**If the endpoint policy is restricted:**

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::novamart-*",
      "arn:aws:s3:::novamart-*/*"
    ]
  }]
}
```

This would need `arn:aws:s3:::novamart-order-archive/*` to be included (it likely is if the prefix matches, but verify).

**Cause 3: S3 bucket has `aws:SourceVpc` or `aws:SourceIp` condition restricting access**

```bash
# Get the full bucket policy (there may be additional statements not shown)
aws s3api get-bucket-policy --bucket novamart-order-archive --profile log-archive-account | jq '.Policy | fromjson'

# Look for Deny statements with conditions like:
# "aws:SourceVpc" != "vpc-xxx"
# "aws:SourceIp" not in "10.0.0.0/8"
# These would deny requests from the production VPC if not matched
```

### 4. Corrected Cross-Account Setup

**Identity policy (order-svc role, production account 111111111111):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountS3Write",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::novamart-order-archive/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "AllowCrossAccountKMS",
      "Effect": "Allow",
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:555555555555:key/mrk-archive-key-id"
    }
  ]
}
```

**Bucket policy (log archive account 555555555555):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgAccountsWrite",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::novamart-order-archive/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms",
          "aws:PrincipalOrgID": "o-novamart123"
        }
      }
    },
    {
      "Sid": "DenyNonEncryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::novamart-order-archive/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyMissingEncryption",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::novamart-order-archive/*",
      "Condition": {
        "Null": {
          "s3:x-amz-server-side-encryption": "true"
        }
      }
    }
  ]
}
```

**Why `aws:PrincipalOrgID` should be added:**

The original bucket policy used `"Principal": {"AWS": "arn:aws:iam::111111111111:root"}` which has two problems:

1. **Hardcodes the account ID** — if NovaMart adds a new production account, someone must manually update the bucket policy. Easy to forget.

2. **Trusts the entire account** — any role in account 111111111111 can write to this bucket, not just order-svc. If an attacker compromises ANY role in the production account, they can write (and potentially overwrite) archived data.

`aws:PrincipalOrgID` solves the first problem:

```
"aws:PrincipalOrgID": "o-novamart123"
```

This means "any principal from any account in the NovaMart AWS Organization." New accounts automatically have access. Accounts that leave the org automatically lose access. No hardcoded account IDs to maintain.

**For maximum security, combine both:**

```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalOrgID": "o-novamart123",
    "aws:PrincipalArn": "arn:aws:iam::111111111111:role/order-svc-irsa-role"
  }
}
```

This restricts to both the org AND the specific role — defense in depth.

---

## Q3: Organization Design Under Constraints

### 1. AWS Account Structure for PCI-DSS

```
NovaMart AWS Organization (o-novamart123)
│
├── Root Account (management only — no workloads)
│   └── Organizations, SCPs, Billing, SSO
│
├── OU: Security
│   ├── Account: Security Tooling (222222222222)
│   │   └── GuardDuty delegated admin, Security Hub, Config aggregator,
│   │       CloudTrail org trail (logs TO log archive), forensics tooling
│   │
│   └── Account: Log Archive (555555555555)
│       └── CloudTrail logs, VPC flow logs, Config snapshots
│           S3 buckets with Object Lock (WORM — immutable logs)
│           NO HUMAN ACCESS except break-glass
│
├── OU: Infrastructure
│   ├── Account: Shared Services (333333333333)
│   │   └── Transit Gateway, DNS (Route 53), shared ECR,
│   │       CI/CD runners, Artifact storage
│   │
│   └── Account: Network (444444444444)
│       └── Transit Gateway attachments, VPN, Direct Connect,
│           centralized egress (NAT Gateways)
│
├── OU: Workloads
│   ├── OU: Non-Production
│   │   ├── Account: Dev (666666666666)
│   │   │   └── All non-payment services dev environments
│   │   │
│   │   └── Account: Staging (777777777777)
│   │       └── All non-payment services staging environments
│   │
│   └── OU: Production
│       └── Account: Production General (111111111111)
│           └── order-svc, cart-svc, search-svc, notification-svc, ml-model-svc
│               EKS cluster, RDS, ElastiCache, S3
│
├── OU: PCI (SEPARATE OU — different SCPs, stricter controls)
│   ├── Account: PCI Production (888888888888) ← PAYMENT WORKLOADS ONLY
│   │   └── payment-svc, fraud-svc
│   │       Dedicated EKS cluster (or dedicated node group with taints)
│   │       Dedicated RDS for payment data
│   │       KMS CMK for payment data encryption
│   │       VPC peered to production (restricted security groups)
│   │       NO non-payment workloads
│   │
│   ├── Account: PCI Staging (999999999999)
│   │   └── payment-svc staging, fraud-svc staging
│   │       Mirrors PCI production architecture
│   │       Uses synthetic/tokenized data (NO real card numbers)
│   │
│   └── Account: PCI Dev (101010101010)
│       └── payment-svc dev, fraud-svc dev
│
└── OU: Sandbox
    └── Account: Sandbox (121212121212)
        └── Developer experimentation, no production data
```

**Why this isolation matters for PCI scope:**

```
┌─────────────────────────────────────────────────────────────────┐
│ PCI-DSS SCOPE REDUCTION                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Without isolation:                                              │
│   ALL of NovaMart's infrastructure is "in scope" for PCI       │
│   audit because cardholder data COULD flow to any system.       │
│   200+ EC2 instances need PCI compliance. Cost: enormous.       │
│                                                                 │
│ With account-level isolation:                                    │
│   Only the PCI OU accounts are in scope.                        │
│   payment-svc + fraud-svc + their infrastructure.               │
│   Maybe 20-30 instances instead of 200+.                        │
│   The auditor verifies the BOUNDARY (SCPs, network controls)    │
│   and then only audits what's inside.                           │
│                                                                 │
│ The boundary is enforced by:                                    │
│   1. Separate AWS account = separate IAM boundary               │
│   2. SCPs on PCI OU = stricter than general production          │
│   3. Network isolation = VPC peering with security groups       │
│      that only allow specific ports (no broad access)           │
│   4. No shared credentials between PCI and non-PCI              │
│   5. Separate EKS cluster (or node group) = no pod-to-pod      │
│      communication between PCI and non-PCI workloads            │
│                                                                 │
│ This is the "reduce PCI scope" strategy — the #1 cost           │
│ optimization in PCI compliance.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. SCPs for PCI Production Account

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyIAMUserCreation",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:CreateLoginProfile",
        "iam:CreateAccessKey",
        "iam:AttachUserPolicy",
        "iam:PutUserPolicy"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyAccessFromOutsideOrg",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "o-novamart123"
        },
        "Bool": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    },
    {
      "Sid": "DenyCloudTrailModification",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail",
        "cloudtrail:PutEventSelectors"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyUnapprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "sts:*",
        "organizations:*",
        "support:*",
        "budgets:*",
        "s3:GetBucketLocation",
        "s3:ListAllMyBuckets",
        "cloudfront:*",
        "route53:*",
        "waf:*",
        "wafv2:*",
        "waf-regional:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2",
            "eu-west-1"
          ]
        }
      }
    },
    {
      "Sid": "DenyS3PublicAccess",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketPublicAccessBlock",
        "s3:PutAccountPublicAccessBlock"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:PublicAccessBlockConfiguration/BlockPublicAcls": "true"
        }
      }
    },
    {
      "Sid": "DenyLeavingOrg",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    },
    {
      "Sid": "ProtectLogArchiveBucket",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteBucket",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "s3:PutLifecycleConfiguration"
      ],
      "Resource": [
        "arn:aws:s3:::novamart-cloudtrail-logs",
        "arn:aws:s3:::novamart-cloudtrail-logs/*"
      ]
    }
  ]
}
```

**Key design decisions:**

- **No IAM users:** All human access via SSO (IAM Identity Center). IAM users are credentials that can be leaked. IRSA for services.
- **Org-only access:** Prevents any external account from assuming roles, even if someone misconfigures a trust policy. `PrincipalIsAWSService: false` exempts AWS services (like S3 replication, CloudTrail).
- **CloudTrail immutable:** Even an account admin can't stop logging. The auditor specifically asked for "immutable logging."
- **Region restriction:** Using `NotAction` pattern to exempt global services (IAM, STS, Route53, CloudFront) from the region restriction while blocking compute/storage in non-approved regions.

### 3. Developer Staging-to-Production Pivot Prevention

**The auditor's question: "Can a developer with staging access pivot to production payment data?"**

**Walk-through of every boundary:**

```
Developer authenticates via SSO (IAM Identity Center)
    │
    │ Boundary 1: SSO Permission Set Assignment
    │   Developer has "DeveloperAccess" permission set
    │   assigned ONLY to accounts: Dev (666), Staging (777), PCI-Dev (101)
    │   They are NOT assigned to PCI-Production (888) at all.
    │
    │   Even if they try to switch roles to PCI-Production,
    │   SSO has no permission set for them in that account.
    │
    │   Check:
    │   aws sso-admin list-account-assignments \
    │     --instance-arn <sso-instance-arn> \
    │     --account-id 888888888888 \
    │     --permission-set-arn <developer-permission-set-arn>
    │   Expected: empty (no assignments)
    │
    ▼
    │ Boundary 2: No cross-account role trust
    │   Roles in PCI-Production (888) only trust:
    │   - The CI/CD automation role in Shared Services (333)
    │   - The break-glass role (emergency only)
    │   - SSO with "PCI-Admin" permission set (assigned to 2 senior engineers only)
    │
    │   The developer's SSO role in staging (777) is NOT trusted
    │   by any role in PCI-Production (888).
    │
    │   Check:
    │   aws iam list-roles --query 'Roles[*].AssumeRolePolicyDocument' \
    │     --profile pci-production | grep "777777777777"
    │   Expected: no results (staging account not in any trust policy)
    │
    ▼
    │ Boundary 3: Network isolation
    │   PCI-Production VPC is peered to Production General VPC
    │   but with restricted security groups:
    │   - Only payment-svc pods (specific CIDR) can reach PCI RDS
    │   - Port 5432 (PostgreSQL) only, no SSH, no other ports
    │   - Staging VPC has NO peering/connectivity to PCI VPC at all
    │
    │   Even if a developer compromised a staging pod, they cannot
    │   reach the PCI production network.
    │
    │   Check:
    │   aws ec2 describe-vpc-peering-connections \
    │     --filters "Name=requester-vpc-info.vpc-id,Values=vpc-staging-id" \
    │     --query 'VpcPeeringConnections[?AccepterVpcInfo.VpcId==`vpc-pci-prod-id`]'
    │   Expected: empty (no peering between staging and PCI production)
    │
    ▼
    │ Boundary 4: SCP prevents external access to PCI account
    │   The "DenyAccessFromOutsideOrg" SCP blocks any principal
    │   not in the organization. But more importantly, the trust
    │   policies on PCI roles DON'T trust staging roles — so even
    │   within the org, there's no path from staging → PCI production.
    │
    ▼
    │ Boundary 5: Data-level isolation
    │   PCI production RDS uses a KMS CMK that only the PCI account
    │   IAM roles can decrypt. Even if a developer somehow got a
    │   snapshot of the PCI database, they couldn't decrypt it
    │   without access to the KMS key — which they don't have.
    │
    │   Check:
    │   aws kms get-key-policy --key-id <pci-cmk-id> --policy-name default \
    │     --profile pci-production | jq .
    │   Expected: only PCI account roles in the key policy
    │
    ▼
    │ Boundary 6: Kubernetes-level isolation
    │   PCI workloads run on a separate EKS cluster (or dedicated
    │   node group with taints + tolerations + NodeSelector).
    │   NetworkPolicy in the PCI namespace blocks all ingress
    │   except from the PCI API gateway.
    │
    │   kubectl --context pci-production get networkpolicy -n payments
    │   Expected: default-deny-all + allow-from-pci-gateway-only
    │
    RESULT: Developer CANNOT pivot from staging to production payment data.
    
    Six independent boundaries must ALL be bypassed:
    SSO assignment → IAM trust → Network → SCP → KMS → Kubernetes
    
    This is defense in depth — any single boundary prevents the pivot.
```

### 4. Break-Glass Root Account MFA — What's Wrong and the Fix

**What's wrong:**

```
Problem 1: SHARED MFA SEED
  4 engineers share the TOTP seed. This means:
  - No individual accountability (CloudTrail shows "root" but not WHICH human)
  - You can't revoke access for one person without rotating for all 4
  - If any one of their devices is compromised, root is compromised
  - Violates PCI-DSS Requirement 8.5: "Do not use group, shared, or
    generic IDs, passwords, or other authentication methods"

Problem 2: PASSWORD MANAGER
  - MFA seed in a password manager = MFA and password in the same store
  - If the password manager is breached, the attacker has BOTH factors
  - This defeats the entire purpose of MFA (two INDEPENDENT factors)
  - The MFA seed should never be in a digital system accessible over network

Problem 3: 4 PEOPLE
  - Root account access should be needed almost never (1-2x per year)
  - 4 people with access = 4x the attack surface
  - Each person's phone, laptop, and accounts is a potential entry point
```

**The fix:**

```
┌─────────────────────────────────────────────────────────────────┐
│ ROOT ACCOUNT BREAK-GLASS PROCEDURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 1. HARDWARE MFA TOKEN (not software TOTP)                       │
│    - Purchase 2x YubiKey 5 FIPS or similar hardware token       │
│    - Register BOTH as MFA devices on the root account           │
│    - Hardware tokens can't be cloned, shared digitally,         │
│      or compromised remotely                                    │
│                                                                 │
│ 2. PHYSICAL SEPARATION (two different locations)                │
│    - Primary YubiKey: company safe / physical lockbox at HQ     │
│    - Backup YubiKey: separate physical location (e.g., bank     │
│      safe deposit box or second office)                         │
│    - Root account password: printed on paper, sealed in         │
│      tamper-evident envelope, stored in DIFFERENT safe          │
│      from the YubiKey                                           │
│    - This means: no single person and no single physical        │
│      location has both factors                                  │
│                                                                 │
│ 3. DUAL-PERSON REQUIREMENT                                      │
│    - Accessing the root account requires TWO senior engineers    │
│    - Person A: has combination to the safe with the password     │
│    - Person B: has key to the lockbox with the YubiKey           │
│    - Neither person alone can access root                       │
│    - Log who opened what in a physical access log               │
│                                                                 │
│ 4. AUDIT TRAIL                                                  │
│    - CloudTrail logs all root account API calls                 │
│    - SNS alert fires immediately when root credentials are used │
│    - Physical access log documents who, when, why               │
│    - Post-use review: was root access necessary? Could the      │
│      action be delegated to a lesser-privileged role?           │
│                                                                 │
│ 5. MINIMIZE ROOT USAGE                                          │
│    - Almost everything can be done without root                 │
│    - Root is only needed for:                                   │
│      • Changing account email                                   │
│      • Changing root password/MFA                               │
│      • Closing the account                                      │
│      • Restoring IAM permissions if locked out                  │
│      • Enabling/disabling MFA delete on S3                      │
│    - If you're using root more than 2x/year, you're doing       │
│      something wrong                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Alert when root is used:**

```yaml
# CloudWatch Events rule (EventBridge) for root login
{
  "source": ["aws.signin"],
  "detail-type": ["AWS Console Sign In via CloudTrail"],
  "detail": {
    "userIdentity": {
      "type": ["Root"]
    }
  }
}
# → SNS topic → PagerDuty page to security team

# Also for any root API call:
{
  "source": ["aws.iam", "aws.s3", "aws.ec2"],
  "detail": {
    "userIdentity": {
      "type": ["Root"]
    }
  }
}
```

```bash
# Verify root has hardware MFA enabled:
aws iam get-account-summary --query 'SummaryMap.AccountMFAEnabled'
# Must return 1

# List MFA devices on root:
aws iam list-mfa-devices --user-name root
# Should show hardware device, not virtual
```

---

## Q4: Blast Radius Scenario — SSRF Compromise

### 1. Attacker's Blast Radius with IRSA

**What `notification-svc` typically needs access to:**

```
┌──────────────────────────────────────────────────────────────┐
│ notification-svc IRSA Role — Legitimate Permissions          │
├──────────────────────────────────────────────────────────────┤
│ SES: ses:SendEmail, ses:SendRawEmail                         │
│   → Send transactional emails (order confirmations, etc.)    │
│                                                              │
│ SNS: sns:Publish                                             │
│   → Send SMS notifications, push notifications               │
│   → Specific topic ARNs only                                 │
│                                                              │
│ SQS: sqs:ReceiveMessage, sqs:DeleteMessage                   │
│   → Read from notification queue                             │
│   → Specific queue ARN only                                  │
│                                                              │
│ DynamoDB: dynamodb:GetItem, dynamodb:PutItem                 │
│   → Notification preferences, delivery status tracking       │
│   → Specific table only                                      │
│                                                              │
│ (Maybe) S3: s3:GetObject                                     │
│   → Read email templates from a template bucket              │
│   → Specific bucket/prefix only                              │
└──────────────────────────────────────────────────────────────┘
```

**What the attacker CAN do:**

```


```
WITH IRSA (scoped role):

CAN:
├── Send emails via SES as NovaMart (phishing FROM NovaMart's verified domain)
├── Send SMS via SNS (cost: could run up a bill, but limited to specific topics)
├── Read messages from the notification queue (data exposure: contains
│   customer email addresses, order IDs, notification content)
├── Read/write notification preferences in DynamoDB (PII exposure:
│   email addresses, phone numbers, user preferences)
├── Read email templates from S3 (low sensitivity)
└── Call aws sts get-caller-identity to identify the role
    (reconnaissance — now they know the account ID and role name)

CANNOT:
├── Access payment data (different account, different role)
├── Access RDS databases (no rds-db:connect permission, and network
│   security groups block non-app ports anyway)
├── Modify IAM roles or policies (no iam:* permissions)
├── Access other services' SQS queues (resource-scoped to notification queue only)
├── Access EC2 instances or EKS API (no ec2:*, eks:* permissions)
├── Read secrets from Secrets Manager or Parameter Store
│   (no secretsmanager:* or ssm:GetParameter permissions)
├── Access S3 buckets other than the template bucket
├── Modify infrastructure (no cloudformation:*, no terraform state access)
├── Move laterally to other pods' roles (IRSA tokens are per-ServiceAccount)
└── Access the node's IMDS to get the node role (IRSA + IMDS hop limit = 1
    blocks containers from reaching the node metadata service)
```

**Blast radius summary with IRSA:**

```
Data exposure:   Customer emails, phone numbers, notification preferences
                 from DynamoDB + SQS messages (PII, but NOT payment data)
Financial risk:  SES/SNS abuse for spam/phishing using NovaMart's identity
Lateral movement: NONE — scoped role has no path to other services
Infrastructure:  NONE — no ability to modify compute, networking, or IAM
```

### 2. Without IRSA — Node Instance Role — Catastrophically Worse

**If all pods use the node instance role, the attacker gets the node's permissions:**

```
Node instance role typically has:

ec2:Describe*                  → Full visibility into all EC2 instances, VPCs,
                                 security groups, subnets (reconnaissance goldmine)
ecr:GetAuthorizationToken      → Pull ANY container image (source code exposure)
ecr:BatchGetImage              → Read all container images in the registry
eks:DescribeCluster            → Get cluster endpoint, OIDC config, K8s API details
s3:GetObject on EKS assets     → Read EKS bootstrap scripts, potentially credentials
ssm:GetParameter (sometimes)   → Read Parameter Store values (may contain secrets)
logs:CreateLogStream           → Write to CloudWatch Logs (cover tracks)
logs:PutLogEvents              → Inject false log entries
```

**Plus the node can reach the IMDS (Instance Metadata Service):**

```bash
# The attacker inside the pod WITHOUT IMDS restriction:
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Returns: eks-node-role

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eks-node-role
# Returns: AccessKeyId, SecretAccessKey, Token
# These are REAL credentials for the NODE ROLE

# With IRSA + proper IMDS configuration:
# HttpPutResponseHopLimit = 1 on the launch template
# → Container is 2 hops from IMDS → request never reaches IMDS
# → This attack path is BLOCKED
```

**What the attacker gains without IRSA:**

```
WITHOUT IRSA (node role):

CAN (in addition to everything above):
├── Enumerate ALL AWS resources in the account (ec2:Describe*)
├── Pull ALL container images (ecr:BatchGetImage)
│   → Read application source code for ALL services
│   → Find hardcoded secrets, API keys, database connection strings
├── Discover ALL EKS clusters and their configurations
├── Access IMDS for the node
│   → Get node role credentials
│   → Get instance user-data (may contain bootstrap secrets)
│   → Get VPC CIDR, subnet info, security group IDs
├── Potentially access Parameter Store / Secrets Manager
│   (if node role has ssm:GetParameter — common for EKS bootstrapping)
│   → Read database passwords, API keys, encryption keys
├── Potentially access S3 buckets used by other services
│   (if node role has broad s3:GetObject)
│   → Read customer data, backups, configs
├── Access CloudWatch Logs for all services on the node
│   → Read application logs (may contain PII, tokens, trace IDs)
├── Move laterally by reading other pods' environment variables
│   through the Kubernetes API (if node role has eks: permissions)
└── Potentially escalate by finding credentials in pulled images
    or parameter store that grant access to OTHER accounts

BLAST RADIUS COMPARISON:
┌────────────────────┬─────────────────────┬──────────────────────┐
│ Dimension          │ With IRSA           │ Without IRSA         │
├────────────────────┼─────────────────────┼──────────────────────┤
│ Data exposure      │ notification PII    │ ALL customer data    │
│ Lateral movement   │ None                │ Full account recon   │
│ Source code access  │ None                │ ALL container images │
│ Secret exposure    │ None                │ Secrets Manager/SSM  │
│ Infrastructure     │ None                │ Full EC2/VPC/EKS map │
│ Cross-service      │ None                │ All services on node │
│ Blast radius       │ 1 service           │ Entire AWS account   │
│ PCI impact         │ None (no payment    │ FULL PCI BREACH      │
│                    │ data accessible)    │ (if payment-svc on   │
│                    │                     │ same node)           │
└────────────────────┴─────────────────────┴──────────────────────┘
```

### 3. Privilege Escalation Paths and IAM Controls That Block Them

**Escalation Path 1: Create a new IAM role with admin permissions**

```
Attack: The attacker tries to create a new role with AdministratorAccess,
        then assume it.

aws iam create-role --role-name backdoor-admin \
  --assume-role-policy-document '{"Statement":[{"Effect":"Allow",
    "Principal":{"AWS":"*"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name backdoor-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws sts assume-role --role-arn arn:aws:iam::111111111111:role/backdoor-admin \
  --role-session-name pwned

Blocking controls:
├── IRSA role has NO iam:CreateRole, iam:AttachRolePolicy permissions
│   → AccessDenied immediately
├── SCP: permissions boundary enforcement
│   Even if the role somehow had iam:CreateRole, an SCP can require
│   all created roles to have a permissions boundary:
```

```json
{
  "Sid": "EnforcePermissionsBoundary",
  "Effect": "Deny",
  "Action": [
    "iam:CreateRole",
    "iam:AttachRolePolicy",
    "iam:PutRolePolicy"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotLike": {
      "iam:PermissionsBoundary": "arn:aws:iam::*:policy/NovaMartPermissionsBoundary"
    }
  }
}
```

```
├── Even if the attacker somehow creates a role, the permissions boundary
│   caps its effective permissions to a safe subset
└── CloudTrail + GuardDuty alert on IAM role creation from unexpected principal
```

**Escalation Path 2: Access the Kubernetes API to read Secrets and pivot to other ServiceAccounts**

```
Attack: The attacker uses the pod's Kubernetes ServiceAccount token to
        access the Kubernetes API and read Secrets from other namespaces
        (which may contain database passwords, API keys).

kubectl --token=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) \
  get secrets --all-namespaces

Blocking controls:
├── RBAC: notification-svc's ServiceAccount should have minimal K8s RBAC
│   Default: pods can only read their own namespace's resources
│   Proper config: no cluster-wide Secret access
│
│   Check:
│   kubectl auth can-i list secrets --all-namespaces \
│     --as=system:serviceaccount:notifications:notification-svc-sa
│   Expected: "no"
│
├── NetworkPolicy: if the pod can't reach the K8s API server, this fails
│   (though most clusters allow pod → API server by default)
│
├── Pod Security Admission: prevents mounting of other SA tokens
│
└── If using automountServiceAccountToken: false on pods that don't
    need K8s API access, there's no token to use at all:
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notification-svc-sa
  namespace: notifications
automountServiceAccountToken: false  # No K8s API token mounted
```

**Escalation Path 3: SSRF to the IMDS to steal the node role credentials**

```
Attack: The attacker uses the SSRF vulnerability to make a request from
        the pod to the EC2 Instance Metadata Service (IMDS) at
        169.254.169.254 to steal the node's IAM credentials.

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

Blocking controls (three independent layers):
├── Layer 1: IMDSv2 enforcement
│   Node launch template requires IMDSv2 (token-based):
│   HttpTokens: required
│   Attacker must first GET a token with PUT request:
│     curl -X PUT http://169.254.169.254/latest/api/token \
│       -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
│   Even if they do this, Layer 2 blocks them:
│
├── Layer 2: HttpPutResponseHopLimit = 1
│   The launch template sets:
│     MetadataOptions:
│       HttpTokens: required
│       HttpPutResponseHopLimit: 1
│   Container is behind a network namespace (2 hops from IMDS).
│   The IMDSv2 token response has a TTL of 1 hop.
│   By the time it reaches the container, the TTL is 0 → dropped.
│   The container CANNOT get an IMDS token → CANNOT get credentials.
│
│   Terraform:
```

```hcl
resource "aws_launch_template" "eks_nodes" {
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"    # IMDSv2 only
    http_put_response_hop_limit = 1             # Blocks containers
    instance_metadata_tags      = "disabled"
  }
}
```

```
├── Layer 3: NetworkPolicy blocking IMDS IP
│   Even if hop limit is misconfigured, block at the network level:
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-imds
  namespace: notifications
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to: []
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    # Allow all EXCEPT IMDS
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32
```

```
└── All three must fail for the attacker to reach IMDS.
    Defense in depth: any single layer prevents the escalation.
```

### 4. Incident Response — First 5 Actions

**This is a SECURITY incident, not an availability incident. The priorities are different:**

```
Availability incident priority: Restore service → Investigate → Prevent
Security incident priority:     Contain → Preserve evidence → Eradicate → Restore → Investigate
```

**Action 1 (Minute 0-2): Contain — Isolate the compromised pod WITHOUT destroying evidence**

```bash
# DO NOT kubectl delete the pod — that destroys forensic evidence
# Instead, isolate it by cutting network access

# Step 1: Apply a NetworkPolicy that blocks ALL traffic from the pod
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-compromised-pod
  namespace: notifications
spec:
  podSelector:
    matchLabels:
      app: notification-svc
      # If you know the specific pod, use more specific labels
  policyTypes:
    - Ingress
    - Egress
  # No ingress or egress rules = all traffic blocked
EOF

# Step 2: Verify isolation
kubectl -n notifications exec deploy/notification-svc -- \
  curl -s --connect-timeout 2 http://order-svc.orders:8080/health
# Should timeout — pod is isolated

# Step 3: Cordon the node to prevent new pods from being scheduled there
NODE=$(kubectl -n notifications get pod -l app=notification-svc \
  -o jsonpath='{.items[0].spec.nodeName}')
kubectl cordon "$NODE"
```

**Why isolate instead of kill:** If we delete the pod, we lose:
- The process memory (may contain attacker tools, malware)
- The filesystem state (what files were modified)
- Network connections (what the attacker was connected to)
- The IRSA credentials the attacker was using (we need to see their activity)

**Action 2 (Minute 2-5): Revoke the compromised credentials immediately**

```bash
# The attacker has IRSA credentials — revoke them by adding a
# deny-all inline policy to the IRSA role
# This immediately invalidates ALL active sessions using this role

aws iam put-role-policy \
  --role-name notification-svc-irsa-role \
  --policy-name EmergencyDenyAll \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "DenyEverything",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*"
    }]
  }'

# Verify the policy is applied
aws iam get-role-policy \
  --role-name notification-svc-irsa-role \
  --policy-name EmergencyDenyAll

# This is INSTANT — all API calls from this role now return AccessDenied
# Including the attacker's active session tokens
```

**⚠️ This also breaks notification-svc for legitimate pods.** That's acceptable — containing the breach is more important than sending emails. Post a note:

```
🔴 SECURITY INCIDENT — notification-svc

notification-svc IRSA role has been emergency-revoked.
Email/SMS notifications are DOWN until further notice.
This is intentional — we are containing a security breach.

DO NOT restore the role or create new pods for notification-svc
until the security team clears it.

IC: [your name]
Security lead: [security team lead]
War room: #sec-inc-20240116-ssrf
```

**Action 3 (Minute 5-10): Preserve evidence — capture forensic data from the pod**

```bash
# Capture pod state before anything changes
POD_NAME=$(kubectl -n notifications get pod -l app=notification-svc \
  -o jsonpath='{.items[0].metadata.name}')

# Full pod description
kubectl -n notifications describe pod "$POD_NAME" > "/tmp/forensics/${POD_NAME}-describe.txt"

# Pod logs (may show attacker commands if they used exec)
kubectl -n notifications logs "$POD_NAME" --all-containers > "/tmp/forensics/${POD_NAME}-logs.txt"
kubectl -n notifications logs "$POD_NAME" --all-containers --previous > "/tmp/forensics/${POD_NAME}-logs-previous.txt" 2>/dev/null

# Environment variables (shows IRSA config, any other secrets)
kubectl -n notifications exec "$POD_NAME" -- env > "/tmp/forensics/${POD_NAME}-env.txt"

# Process list (may show attacker's processes)
kubectl -n notifications exec "$POD_NAME" -- ps auxwww > "/tmp/forensics/${POD_NAME}-processes.txt"

# Network connections (may show active C2 connections)
kubectl -n notifications exec "$POD_NAME" -- netstat -tlnp > "/tmp/forensics/${POD_NAME}-netstat.txt" 2>/dev/null || \
kubectl -n notifications exec "$POD_NAME" -- ss -tlnp > "/tmp/forensics/${POD_NAME}-ss.txt"

# Filesystem changes (compare to the known container image)
kubectl -n notifications exec "$POD_NAME" -- find / -newer /proc -type f 2>/dev/null > "/tmp/forensics/${POD_NAME}-modified-files.txt"

# Copy any suspicious files out
kubectl -n notifications cp "$POD_NAME":/tmp/ "/tmp/forensics/pod-tmp-dir/" 2>/dev/null
```

**Action 4 (Minute 10-15): Assess blast radius — what did the attacker access?**

```bash
# Query CloudTrail for ALL API calls made by the notification-svc IRSA role
# in the last 24 hours (extend if compromise may be older)

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::STS::AssumedRole \
  --start-time "$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --max-results 100 \
  --query 'Events[?contains(CloudTrailEvent, `notification-svc-irsa-role`)].[EventTime, EventName, CloudTrailEvent]' \
  --output json > /tmp/forensics/cloudtrail-notification-role.json

# Parse for suspicious API calls
cat /tmp/forensics/cloudtrail-notification-role.json | jq -r '.[] | .[0] + " | " + .[1]' | sort

# Look specifically for:
# - sts:GetCallerIdentity (reconnaissance)
# - iam:* (privilege escalation attempts)
# - ec2:Describe* (infrastructure recon)
# - s3:ListBuckets, s3:GetObject on unexpected buckets
# - ses:SendEmail to external addresses (data exfil via email)
# - sqs:ReceiveMessage (reading customer data from queue)

# Check GuardDuty for findings related to this role/instance
aws guardduty list-findings \
  --detector-id "$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)" \
  --finding-criteria '{
    "Criterion": {
      "resource.accessKeyDetails.userName": {
        "Eq": ["notification-svc-irsa-role"]
      },
      "updatedAt": {
        "GreaterThanOrEqual": '$(date -u -d '24 hours ago' +%s000)'
      }
    }
  }'

# Check for data exfiltration — did the attacker send emails via SES?
aws ses get-send-statistics --query 'SendDataPoints[?Timestamp>=`'"$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)"'`]'

# Check SQS — were messages read (potential customer data exposure)?
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ReceiveMessage \
  --start-time "$(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)" | \
  jq '.Events[] | select(.CloudTrailEvent | contains("notification-svc-irsa-role"))'
```

**Action 5 (Minute 15-20): Notify stakeholders and begin coordinated response**

```
# Notifications in order:

1. Security team (already in war room from Action 2)
2. VP of Engineering — this is a confirmed security breach
3. Legal/compliance — may trigger PCI-DSS breach notification requirements
   (depends on what data was accessed)
4. Customer data assessment — did PII leak?
```

```
Message to #sec-inc-20240116-ssrf:

📋 SECURITY INCIDENT STATUS — Minute 15

CONTAINMENT:
✅ Compromised pod network-isolated (NetworkPolicy applied)
✅ IRSA role emergency-revoked (deny-all inline policy)
✅ Node cordoned (no new pods scheduled)
✅ Forensic evidence captured (pod state, logs, processes, network)

BLAST RADIUS ASSESSMENT (preliminary):
- Compromised role: notification-svc-irsa-role
- Permissions: SES, SNS, SQS (notification queue), DynamoDB (preferences)
- CloudTrail review: [IN PROGRESS]
- GuardDuty findings: [CHECKING]

WHAT WE KNOW:
- Attack vector: SSRF vulnerability in notification-svc
- Attacker had access to: notification-svc IRSA role credentials
- IRSA contained the blast radius — NO access to payment data,
  databases, or other services' resources
- Duration of compromise: [UNKNOWN — checking CloudTrail for first suspicious call]

WHAT WE DON'T KNOW YET:
- When the compromise started (initial access time)
- What data was exfiltrated (SQS messages, DynamoDB records)
- Whether the attacker established persistence (checked: no IAM changes,
  checking for K8s cronjobs/deployments)
- Whether other pods/services are compromised (checking)

NEXT ACTIONS:
1. Complete CloudTrail analysis for full timeline
2. Check for persistence mechanisms (K8s resources, Lambda, etc.)
3. Patch the SSRF vulnerability in notification-svc
4. Rebuild notification-svc from known-good image after patching
5. Legal assessment: does PII exposure require customer notification?

@legal @vp-engineering @ciso — please join #sec-inc-20240116-ssrf
```

**Post-containment steps (not in first 5 but critical):**

```bash
# After clearing with security team:

# 1. Deploy patched notification-svc (fix the SSRF vulnerability)
# Build from known-good source, NOT from the compromised image

# 2. Create a NEW IRSA role (don't just remove the deny-all from the old one)
# The old role's ARN may be logged by the attacker for future use

# 3. Rotate any secrets that notification-svc had access to
# Even though IRSA scoped the blast radius, rotate SES credentials,
# SNS topics, SQS queue policies as a precaution

# 4. Remove the network isolation and uncordon the node
kubectl -n notifications delete networkpolicy isolate-compromised-pod
kubectl uncordon "$NODE"

# 5. Delete the compromised pod (AFTER forensics are complete)
kubectl -n notifications delete pod "$POD_NAME"
```

---

## Summary

```
Q1: IRSA Debugging
  Debug ladder: get-caller-identity → env vars → token file → 
    ServiceAccount annotation → trust policy → OIDC provider → manual STS call
  Node role fallback = 9+ possible causes (webhook, annotation, thumbprint,
    trust policy, OIDC deleted, token volume, token expired, SDK version, env override)
  OIDC thumbprint typo = cluster-wide SEV1 → fix CLI first, Terraform later
  Prevention: dynamic thumbprint via tls_certificate data source + SCP + CI validation

Q2: IAM Policy Evaluation
  Cross-account: BOTH identity policy AND resource policy must Allow
  SCP: Null condition with "false" = "deny when key IS present"
  KMS, VPC endpoints, and bucket conditions are the hidden blockers
  aws:PrincipalOrgID replaces hardcoded account IDs

Q3: PCI Organization Design
  Separate OU for PCI = scope reduction (20 instances vs 200)
  6 boundaries prevent staging→production pivot (SSO, IAM trust,
    network, SCP, KMS, Kubernetes)
  Root break-glass: hardware MFA + physical separation + dual-person

Q4: SSRF Blast Radius
  IRSA: blast radius = 1 service (notification PII only)
  Node role: blast radius = entire AWS account (catastrophic)
  3 escalation paths blocked by: no IAM perms, K8s RBAC, IMDS hop limit
  Security IR: Contain → Preserve evidence → Revoke → Assess → Notify
  NEVER delete the compromised pod before capturing forensics
```

---

# Phase 6, Lesson 2: Secrets Management

## Why Secrets Management Is a Solved Problem That Everyone Gets Wrong

```
SECRETS IN THE WILD (real incidents):

2023: Tesla — Kubernetes dashboard exposed, AWS credentials leaked
      → Attacker spun up crypto miners on Tesla's AWS account

2022: Toyota — 300,000 customer records exposed
      → Access key hardcoded in public GitHub repo for 5 years

2021: Twitch — Entire source code leaked including server configs
      → Plaintext secrets in configuration files

2019: Capital One — 100 million records
      → SSRF vulnerability + overprivileged IAM role
      → (Exactly the Q4 scenario you just answered)

PATTERN: The breach is almost never "they cracked our encryption."
         It's "the secret was lying around in plaintext somewhere
         it shouldn't have been."

At NovaMart:
  - Database passwords for 200+ microservices
  - Stripe API keys (touches money)
  - JWT signing keys (authentication bypass if leaked)
  - TLS private keys (impersonation if leaked)
  - Internal service tokens
  - Third-party API keys (Twilio, SendGrid, Datadog)
  - Encryption keys for PII at rest

  One leaked secret = breach. The question is scope.
```

---

## Part 1: AWS KMS — The Foundation of Encryption at Rest

### What KMS Actually Does

```
KMS does NOT store your data. It stores and manages KEYS
that encrypt your data.

Two fundamental operations:

ENCRYPT:
  Your service → sends plaintext data key request → KMS
  KMS → generates Data Key → returns:
    1. Plaintext data key (use it to encrypt, then throw it away)
    2. Encrypted data key (store this alongside your encrypted data)

DECRYPT:
  Your service → sends encrypted data key → KMS
  KMS → decrypts the data key using the CMK → returns plaintext data key
  Your service → uses plaintext data key to decrypt the data
  → throws away the plaintext key

This is called ENVELOPE ENCRYPTION:
  ┌────────────────────────────────────────────────┐
  │                                                │
  │  CMK (Customer Master Key) — lives in KMS      │
  │  │                                             │
  │  │ encrypts/decrypts                           │
  │  ▼                                             │
  │  Data Key (DEK) — generated by KMS             │
  │  │                                             │
  │  │ encrypts/decrypts                           │
  │  ▼                                             │
  │  Your actual data (S3 objects, EBS volumes,     │
  │  RDS databases, DynamoDB tables, etc.)         │
  │                                                │
  └────────────────────────────────────────────────┘

Why envelope encryption?
  1. CMK never leaves KMS hardware (FIPS 140-2 validated HSMs)
  2. Data key does the actual encryption (fast, local to your service)
  3. You can encrypt GBs of data without sending it all to KMS
  4. Rotating the CMK doesn't require re-encrypting all data
     (old data keys still work, new data gets new data keys)
```

### KMS Key Types

```
┌────────────────────────┬─────────────────────────────────────────────┐
│ Key Type               │ Details                                      │
├────────────────────────┼─────────────────────────────────────────────┤
│ AWS Owned Keys         │ AWS manages entirely. Free. No visibility.  │
│ (aws/service)          │ Used by default for many services.          │
│                        │ You can't see, manage, rotate, or audit     │
│                        │ usage. NOT suitable for compliance.         │
├────────────────────────┼─────────────────────────────────────────────┤
│ AWS Managed Keys       │ AWS creates and manages in YOUR account.    │
│ (aws/s3, aws/ebs,     │ Free (no monthly fee, pay per use).        │
│  aws/rds, etc.)        │ Auto-rotates every year.                   │
│                        │ Visible in your KMS console.               │
│                        │ You can see CloudTrail usage.              │
│                        │ You CANNOT change key policy.              │
│                        │ Cannot be used cross-account.              │
├────────────────────────┼─────────────────────────────────────────────┤
│ Customer Managed Keys  │ YOU create, manage, and control.            │
│ (CMK)                  │ $1/month + $0.03/10K API calls.            │
│                        │ Full key policy control.                    │
│                        │ Can be used cross-account.                 │
│                        │ Can enable/disable, schedule deletion.      │
│                        │ Custom rotation period (90-2560 days).     │
│                        │ REQUIRED for PCI, HIPAA, SOC2 compliance.  │
│                        │                                             │
│                        │ NovaMart: Use CMKs for all sensitive data. │
├────────────────────────┼─────────────────────────────────────────────┤
│ Multi-Region Keys      │ Same key material replicated across regions.│
│ (MRK)                  │ Prefix: mrk-                               │
│                        │ Can decrypt in any region without cross-    │
│                        │ region API calls.                          │
│                        │ Use for: DR, global services, cross-region │
│                        │ replication.                                │
│                        │ Higher cost than single-region.            │
├────────────────────────┼─────────────────────────────────────────────┤
│ External Key Material  │ You provide the key material (BYOK).        │
│                        │ You're responsible for durability.          │
│                        │ Can set expiration date.                   │
│                        │ Use case: regulatory requirement to        │
│                        │ control key material lifecycle.            │
│                        │ Rare at NovaMart.                          │
├────────────────────────┼─────────────────────────────────────────────┤
│ Custom Key Store       │ Keys backed by CloudHSM cluster you own.   │
│ (CloudHSM backing)     │ FIPS 140-2 Level 3 (vs Level 2 for KMS). │
│                        │ Most expensive. Most control.              │
│                        │ Use case: regulatory mandate for L3 HSM.  │
│                        │ Very rare.                                 │
└────────────────────────┴─────────────────────────────────────────────┘
```

### KMS Key Policy — The Critical Control

```
KMS key policies are RESOURCE-BASED policies, like S3 bucket policies.
Unlike most AWS resources, KMS keys are LOCKED DOWN by default —
even the account root cannot use a key unless the key policy allows it.

This is unique to KMS. For S3, the account root always has access.
For KMS, if you create a key with a policy that doesn't include root,
even root is locked out. The key becomes unusable.
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RootAccess",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::888888888888:root"},
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "KeyAdministration",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::888888888888:role/KMSAdminRole"
      },
      "Action": [
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
      ],
      "Resource": "*"
    },
    {
      "Sid": "KeyUsage",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::888888888888:role/payment-svc-irsa-role"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowS3Service",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::888888888888:root"},
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com",
          "kms:CallerAccount": "888888888888"
        }
      }
    },
    {
      "Sid": "AllowKeyGrants",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::888888888888:role/payment-svc-irsa-role"
      },
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": "true"
        }
      }
    }
  ]
}
```

```
KEY POLICY CONCEPTS:

1. Root statement: ALWAYS include root access.
   Without it, if the admin role is deleted, the key is PERMANENTLY
   locked out. Root access is your safety net.

2. Separation of duties:
   KMSAdminRole can MANAGE the key (create, enable, disable, delete)
   but CANNOT USE it (no Encrypt, Decrypt, GenerateDataKey)
   
   payment-svc can USE the key but CANNOT MANAGE it
   (can't disable, delete, or change key policy)
   
   This prevents a compromised service from deleting its own
   encryption key (ransomware attack).

3. Grants: Used by AWS services (EBS, RDS, S3) to get temporary
   key usage permissions. The GrantIsForAWSResource condition
   ensures grants can only be created for AWS service use,
   not for arbitrary principals.

4. kms:ViaService: Restricts key usage to when it's called
   through a specific AWS service. Even if an attacker gets
   kms:Decrypt permission, they can only decrypt through S3,
   not by calling KMS directly.
```

### KMS at NovaMart — Terraform

```hcl
# KMS key for payment service data
resource "aws_kms_key" "payment_data" {
  description             = "Encryption key for payment service data"
  deletion_window_in_days = 30    # 7-30 days. Use 30 for production.
  enable_key_rotation     = true  # Auto-rotate every year
  rotation_period_in_days = 365   # Default. Can set 90-2560.
  multi_region            = false # Single region unless DR needs it
  
  key_usage               = "ENCRYPT_DECRYPT"  # vs SIGN_VERIFY
  customer_master_key_spec = "SYMMETRIC_DEFAULT"  # AES-256-GCM

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "RootAccess"
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
        Action    = "kms:*"
        Resource  = "*"
      },
      {
        Sid       = "PaymentServiceUsage"
        Effect    = "Allow"
        Principal = { AWS = aws_iam_role.payment_svc.arn }
        Action    = [
          "kms:Decrypt",
          "kms:GenerateDataKey",
          "kms:DescribeKey"
        ]
        Resource = "*"
      },
      {
        Sid       = "RDSServiceUsage"
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
        Action    = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:CreateGrant",
          "kms:ListGrants",
          "kms:DescribeKey"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "kms:ViaService"     = "rds.us-east-1.amazonaws.com"
            "kms:CallerAccount"  = data.aws_caller_identity.current.account_id
          }
        }
      }
    ]
  })

  tags = {
    Environment = "production"
    Service     = "payment-svc"
    Compliance  = "pci"
    ManagedBy   = "terraform"
  }
}

resource "aws_kms_alias" "payment_data" {
  name          = "alias/novamart-payment-data"
  target_key_id = aws_kms_key.payment_data.key_id
}
```

### KMS Failure Modes

```
FAILURE 1: Key policy locked out everyone
  CAUSE: Key policy doesn't include root access, admin role deleted
  RESULT: Key is PERMANENTLY unusable — all data encrypted with it
          is PERMANENTLY inaccessible
  PREVENTION: ALWAYS include root in key policy
  RECOVERY: AWS Support MAY be able to help (contact immediately)
            No guarantee. This is a "break glass and pray" situation.

FAILURE 2: Key scheduled for deletion during active use
  CAUSE: Someone ran ScheduleKeyDeletion; after waiting period, key deleted
  SYMPTOM: Services start failing with "key not found" errors
  PREVENTION: 
    - 30-day deletion window (maximum)
    - CloudWatch alarm on ScheduleKeyDeletion API call
    - SCP to deny key deletion except for security admin
  RECOVERY: Cancel deletion if still in waiting period:
    aws kms cancel-key-deletion --key-id <key-id>
    aws kms enable-key --key-id <key-id>

FAILURE 3: KMS request throttling
  CAUSE: Exceeding KMS API limits (5,500-30,000 req/s per region
         depending on key type)
  SYMPTOM: ThrottlingException, services intermittently fail
  COMMON TRIGGER: High-volume service encrypting/decrypting per-request
    without data key caching
  FIX: Cache data keys locally (envelope encryption pattern):
    - Call GenerateDataKey once
    - Cache the plaintext data key in memory
    - Use it for multiple encrypt operations
    - Re-generate periodically (every hour or every N operations)
  ALSO: Use different keys for different services (quota is per key)

FAILURE 4: Cross-account access denied
  CAUSE: Key policy allows cross-account, but caller's identity policy
         doesn't include kms:Decrypt, or kms:ViaService blocks direct calls
  DEBUG: Same pattern as Q2 — check BOTH sides

FAILURE 5: Automatic key rotation breaks custom encryption
  CAUSE: KMS auto-rotation creates new backing key material but keeps
         the same key ID. AWS-integrated services handle this seamlessly.
         Custom encryption code that hardcodes key material metadata breaks.
  FIX: Always reference the key by alias or ID, never by backing key version.
       AWS handles the mapping from key ID → correct backing key for decrypt.

FAILURE 6: Multi-region key replication lag
  CAUSE: Key material replication across regions isn't instant
  SYMPTOM: Encrypt in us-east-1, immediately decrypt in eu-west-1 → fails
  FIX: MRKs replicate quickly (seconds) but build in retry logic
```

---

## Part 2: AWS Secrets Manager

### What Secrets Manager Does

```
Secrets Manager is a MANAGED SERVICE for storing and rotating secrets.

KEY FEATURES:
  - Stores secrets encrypted with KMS (you choose the key)
  - Automatic rotation with Lambda functions
  - Versioning (current, previous, pending)
  - Cross-account access via resource policy
  - Fine-grained IAM access control
  - CloudTrail audit of all access
  - Integrates with RDS, Redshift, DocumentDB natively

SECRETS MANAGER vs PARAMETER STORE:
┌──────────────────────┬────────────────────────┬────────────────────────┐
│ Feature              │ Secrets Manager        │ SSM Parameter Store    │
├──────────────────────┼────────────────────────┼────────────────────────┤
│ Cost                 │ $0.40/secret/month     │ Free (standard)        │
│                      │ + $0.05/10K API calls  │ $0.05/10K (advanced)   │
├──────────────────────┼────────────────────────┼────────────────────────┤
│ Auto rotation        │ Built-in (Lambda)      │ No native rotation     │
├──────────────────────┼────────────────────────┼────────────────────────┤
│ Cross-account        │ Resource policy (easy) │ No resource policy     │
│                      │                        │ (need role assumption) │
├──────────────────────┼────────────────────────┼────────────────────────┤
│ Secret size          │ 64KB                   │ 8KB (adv: 8KB)         │
├──────────────────────┼────────────────────────┼────────────────────────┤
│ Versioning           │ Staging labels         │ History (param store)  │
│                      │ AWSCURRENT, AWSPREVIOUS│ Version numbers        │
├──────────────────────┼────────────────────────┼────────────────────────┤
│ RDS integration      │ Native rotation Lambda │ Manual                 │
├──────────────────────┼────────────────────────┼────────────────────────┤
│ Best for             │ Credentials that need  │ Configuration values,  │
│                      │ rotation, DB passwords,│ feature flags, non-    │
│                      │ API keys, compliance   │ sensitive configs      │
└──────────────────────┴────────────────────────┴────────────────────────┘

NovaMart rule:
  - Secrets Manager: ALL credentials (DB passwords, API keys, tokens)
  - Parameter Store: Non-sensitive configuration (feature flags, endpoints)
```

### Secrets Manager with RDS — Automatic Rotation

```hcl
# Create the RDS master password in Secrets Manager
resource "aws_secretsmanager_secret" "rds_payment" {
  name        = "payment/rds/master"
  description = "Payment service RDS master credentials"
  kms_key_id  = aws_kms_key.payment_data.arn

  # Force delete without recovery window (NEVER in production)
  # recovery_window_in_days = 0  # Only for dev/testing
  
  tags = {
    Service    = "payment-svc"
    Compliance = "pci"
  }
}

resource "aws_secretsmanager_secret_version" "rds_payment" {
  secret_id = aws_secretsmanager_secret.rds_payment.id
  secret_string = jsonencode({
    username = "payment_admin"
    password = random_password.rds_payment.result
    engine   = "postgres"
    host     = aws_db_instance.payment.address
    port     = 5432
    dbname   = "payments"
  })
}

# Enable automatic rotation
resource "aws_secretsmanager_secret_rotation" "rds_payment" {
  secret_id           = aws_secretsmanager_secret.rds_payment.id
  rotation_lambda_arn = aws_lambda_function.secret_rotation.arn

  rotation_rules {
    automatically_after_days = 30  # Rotate every 30 days
    # For RDS: uses multi-user rotation strategy
    # Creates a new user, grants same permissions, swaps
  }
}

# RDS instance using the Secrets Manager secret
resource "aws_db_instance" "payment" {
  identifier     = "novamart-payment"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r6g.xlarge"

  manage_master_user_password   = true  # AWS manages rotation automatically
  master_user_secret_kms_key_id = aws_kms_key.payment_data.arn
  
  # OR use Secrets Manager directly:
  # username = jsondecode(aws_secretsmanager_secret_version.rds_payment.secret_string)["username"]
  # password = jsondecode(aws_secretsmanager_secret_version.rds_payment.secret_string)["password"]
  
  storage_encrypted = true
  kms_key_id        = aws_kms_key.payment_data.arn
}
```

### Secret Rotation — How It Actually Works

```
ROTATION LIFECYCLE (4 steps, executed by Lambda):

Step 1: CREATE SECRET
  Lambda generates new password
  Stores it in Secrets Manager with AWSPENDING staging label
  
Step 2: SET SECRET  
  Lambda connects to the database (or service)
  Creates new user OR changes password
  
Step 3: TEST SECRET
  Lambda connects using the NEW credentials
  Verifies they work (SELECT 1 or equivalent)
  
Step 4: FINISH SECRET
  Move AWSPENDING → AWSCURRENT
  Move old AWSCURRENT → AWSPREVIOUS
  New apps get new credentials, old sessions still work with AWSPREVIOUS

ROTATION STRATEGIES:

Single-user rotation:
  Changes password on the SAME user
  Brief window where old sessions fail (password changed but app 
  hasn't refreshed yet)
  Risk: App outage if refresh fails

Multi-user rotation (RECOMMENDED):
  TWO users: user_A and user_B
  Rotation alternates: update A's password, then B's password
  While A is being rotated, B serves traffic (and vice versa)
  Zero downtime rotation
  Requires TWO database users with identical permissions

APPLICATION HANDLING:
  Your app MUST handle credential refresh:
  
  Option A: Re-fetch from Secrets Manager periodically
    - Every 5-15 minutes, call GetSecretValue
    - Cache in memory, not on disk
    - Use AWSCURRENT staging label
  
  Option B: Use AWS SDK credential providers
    - Java: SecretsManagerRotationReadyMultiUserClient  
    - Most SDKs have built-in Secrets Manager integration
  
  Option C: External Secrets Operator (K8s)
    - Syncs Secrets Manager → K8s Secret automatically
    - Reloader detects Secret change → restarts pods
    - NovaMart's approach (covered in Phase 2)
```

### Secrets Manager Failure Modes

```
FAILURE 1: App doesn't refresh after rotation
  CAUSE: Application caches DB password at startup, never re-fetches
  SYMPTOM: After rotation, app can't connect to database
  FIX: Implement credential refresh (Option A/B/C above)
  TEST: Rotate manually and verify app handles it:
    aws secretsmanager rotate-secret --secret-id payment/rds/master

FAILURE 2: Lambda rotation function fails silently
  CAUSE: Lambda has wrong VPC config, can't reach RDS or Secrets Manager
  SYMPTOM: Secret never rotates, compliance violation
  PREVENTION: 
    - Lambda in same VPC as RDS, with security group access
    - VPC endpoint for Secrets Manager (Lambda can't reach public API 
      from inside VPC without NAT or endpoint)
    - CloudWatch alarm on rotation failure:
      aws cloudwatch put-metric-alarm \
        --alarm-name secret-rotation-failure \
        --metric-name RotationFailed \
        --namespace AWS/SecretsManager

FAILURE 3: AWSPENDING stuck
  CAUSE: Rotation Lambda created AWSPENDING but failed at SET or TEST step
  SYMPTOM: GetSecretValue returns old credentials (AWSCURRENT is unchanged)
           but AWSPENDING exists with new, possibly partially-applied creds
  FIX: 
    # Check pending version
    aws secretsmanager describe-secret --secret-id payment/rds/master
    # If stuck, remove pending and re-rotate:
    aws secretsmanager update-secret-version-stage \
      --secret-id payment/rds/master \
      --version-stage AWSPENDING \
      --remove-from-version-id <pending-version-id>
    # Then trigger fresh rotation

FAILURE 4: Cross-account access denied
  CAUSE: Secret resource policy doesn't allow the external account
  FIX: Add resource policy:
    aws secretsmanager put-resource-policy \
      --secret-id payment/rds/master \
      --resource-policy '...'

FAILURE 5: KMS key rotation breaks Secrets Manager
  CAUSE: The KMS key used to encrypt the secret is deleted or disabled
  SYMPTOM: GetSecretValue returns KMS error
  FIX: Re-enable or restore the KMS key
  PREVENTION: Never disable/delete a KMS key without checking what 
              Secrets Manager secrets use it
```

---

## Part 3: HashiCorp Vault — The Enterprise Secrets Solution

### Why Vault When You Have Secrets Manager?

```
Secrets Manager strengths:
  - Fully managed (no infrastructure to run)
  - Native AWS integration (RDS rotation, IRSA access)
  - Simple for AWS-only environments

But NovaMart needs Vault because:
  1. Multi-cloud: GCP for analytics/ML. Need secrets that work everywhere.
  2. Dynamic secrets: Vault generates short-lived DB credentials per request
     (not just rotating — CREATING unique per-pod credentials)
  3. PKI: Vault as internal CA for mTLS certificate issuance
  4. Secret engines: HashiCorp, AWS, GCP, database, PKI, transit, SSH
  5. Audit log: Comprehensive audit trail (who accessed what, when)
  6. Policies: Fine-grained access control with policy-as-code
  7. Namespaces: Multi-tenant secret isolation (Enterprise)

NOVAMART STRATEGY:
  - Vault (primary): Database creds, PKI, cross-cloud secrets
  - Secrets Manager (AWS-native): RDS auto-rotation, IRSA-accessed secrets
  - External Secrets Operator: Bridges both into Kubernetes Secrets
```

### Vault Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    VAULT ARCHITECTURE                             │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    VAULT CLUSTER (HA)                        │ │
│  │                                                             │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │ │
│  │  │ Active   │  │ Standby  │  │ Standby  │                 │ │
│  │  │ Node     │  │ Node     │  │ Node     │                 │ │
│  │  │          │  │          │  │          │                 │ │
│  │  │ API +    │  │ Forwards │  │ Forwards │                 │ │
│  │  │ Processing│ │ to active│  │ to active│                 │ │
│  │  └────┬─────┘  └──────────┘  └──────────┘                 │ │
│  │       │                                                     │ │
│  │       ▼                                                     │ │
│  │  ┌─────────────────────────────────────┐                   │ │
│  │  │         STORAGE BACKEND              │                   │ │
│  │  │                                     │                   │ │
│  │  │  Integrated Raft (recommended)      │                   │ │
│  │  │  OR: Consul, DynamoDB, S3           │                   │ │
│  │  │                                     │                   │ │
│  │  │  All data encrypted at rest         │                   │ │
│  │  │  with Vault's master key            │                   │ │
│  │  └─────────────────────────────────────┘                   │ │
│  │                                                             │ │
│  │  SECRET ENGINES:           AUTH METHODS:                    │ │
│  │  ├── KV v2 (static)       ├── Kubernetes (ServiceAccount)  │ │
│  │  ├── Database (dynamic)   ├── AWS IAM (IRSA)              │ │
│  │  ├── AWS (dynamic IAM)    ├── OIDC/JWT                    │ │
│  │  ├── PKI (certificates)   ├── AppRole (CI/CD)             │ │
│  │  ├── Transit (encryption) ├── LDAP/AD                     │ │
│  │  └── SSH (signed keys)    └── Userpass (humans)           │ │
│  │                                                             │ │
│  │  AUDIT DEVICES:                                            │ │
│  │  ├── File (local)                                          │ │
│  │  ├── Syslog                                                │ │
│  │  └── Socket (send to SIEM)                                 │ │
│  │                                                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  SEAL/UNSEAL:                                                    │
│  Vault starts SEALED — cannot process requests                   │
│  Must be UNSEALED using unseal keys (Shamir's Secret Sharing)   │
│  OR auto-unseal using AWS KMS (NovaMart's approach)             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Vault Deployment on EKS — Production Configuration

```yaml
# values.yaml for Vault Helm chart
global:
  enabled: true
  tlsDisable: false  # ALWAYS use TLS in production

server:
  image:
    repository: hashicorp/vault
    tag: "1.15.4"

  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 512Mi
      cpu: 500m

  # HA with Raft integrated storage
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      config: |
        ui = true

        listener "tcp" {
          tls_disable = 0
          address     = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_cert_file = "/vault/tls/tls.crt"
          tls_key_file  = "/vault/tls/tls.key"
        }

        storage "raft" {
          path = "/vault/data"
          
          retry_join {
            leader_api_addr = "https://vault-0.vault-internal:8200"
            leader_tls_servername = "vault.vault.svc"
          }
          retry_join {
            leader_api_addr = "https://vault-1.vault-internal:8200"
            leader_tls_servername = "vault.vault.svc"
          }
          retry_join {
            leader_api_addr = "https://vault-2.vault-internal:8200"
            leader_tls_servername = "vault.vault.svc"
          }
        }

        # Auto-unseal with AWS KMS (no manual unseal needed)
        seal "awskms" {
          region     = "us-east-1"
          kms_key_id = "alias/novamart-vault-unseal"
        }

        telemetry {
          prometheus_retention_time = "30s"
          disable_hostname = true
        }

  # Service account for IRSA (Vault needs KMS access for auto-unseal)
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/vault-server

  # Persistent storage for Raft
  dataStorage:
    enabled: true
    size: 20Gi
    storageClass: gp3-encrypted

  # Anti-affinity — spread across AZs
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: vault
          topologyKey: topology.kubernetes.io/zone

  # Audit logging
  auditStorage:
    enabled: true
    size: 10Gi

# Vault Agent Injector — injects secrets into pods via sidecar
injector:
  enabled: true
  resources:
    requests:
      memory: 64Mi
      cpu: 50m
    limits:
      memory: 128Mi
      cpu: 100m
```

### Vault Authentication — Kubernetes Auth Method

```bash
# Enable Kubernetes auth method
vault auth enable kubernetes

# Configure it to use the EKS cluster
vault write auth/kubernetes/config \
  kubernetes_host="https://$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}')" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create a role for payment-svc
vault write auth/kubernetes/role/payment-svc \
  bound_service_account_names=payment-svc-sa \
  bound_service_account_namespaces=payments \
  policies=payment-svc-policy \
  ttl=1h
```

### Vault Policies

```hcl
# payment-svc-policy.hcl
# Payment service can only access payment secrets

# Static secrets (KV v2)
path "secret/data/payment/*" {
  capabilities = ["read"]
}

path "secret/metadata/payment/*" {
  capabilities = ["list", "read"]
}

# Dynamic database credentials
path "database/creds/payment-readonly" {
  capabilities = ["read"]
}

path "database/creds/payment-readwrite" {
  capabilities = ["read"]
}

# PKI — request certificates for payment-svc
path "pki_int/issue/payment-svc" {
  capabilities = ["create", "update"]
}

# Transit — encrypt/decrypt payment data
path "transit/encrypt/payment-key" {
  capabilities = ["update"]
}

path "transit/decrypt/payment-key" {
  capabilities = ["update"]
}

# Deny everything else — explicit
path "secret/data/order/*" {
  capabilities = ["deny"]
}

path "secret/data/user/*" {
  capabilities = ["deny"]
}
```

### Dynamic Secrets — The Killer Feature

```
STATIC SECRETS (traditional):
  Create password → store in vault → give to app → app uses it forever
  Problems:
    - Password shared across all pods/instances
    - One pod compromised → attacker has THE password
    - Rotation disrupts all consumers
    - Revocation requires rotation (affects everyone)

DYNAMIC SECRETS (Vault):
  App requests credentials → Vault creates UNIQUE credentials
  → Vault gives to app with TTL → credentials auto-expire
  → If pod is compromised, revoke ONLY that pod's credentials
  
  Each pod gets its OWN database user with its OWN password.
  If you have 10 payment-svc pods, there are 10 different
  database users, each with a 1-hour TTL.
```

```bash
# Configure the database secret engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/payment-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles="payment-readonly,payment-readwrite" \
  connection_url="postgresql://{{username}}:{{password}}@payment-db.cluster-xxx.us-east-1.rds.amazonaws.com:5432/payments?sslmode=require" \
  username="vault_admin" \
  password="initial-password"

# Create a role for read-only access
vault write database/roles/payment-readonly \
  db_name=payment-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\"; \
    DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Create a role for read-write access
vault write database/roles/payment-readwrite \
  db_name=payment-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\"; \
    DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Application requests credentials:
vault read database/creds/payment-readonly
# Returns:
# Key                Value
# ---                -----
# lease_id           database/creds/payment-readonly/abc123
# lease_duration     1h
# username           v-payment-readonly-abc123-1705401600
# password           A1b2C3d4E5f6G7h8

# After 1 hour, this user is automatically deleted from PostgreSQL
# Application must request new credentials before TTL expires
```

### Vault Agent Sidecar — Injecting Secrets into Pods

```yaml
# Pod annotations for Vault Agent Injector
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-svc
  namespace: payments
spec:
  template:
    metadata:
      annotations:
        # Enable Vault injection
        vault.hashicorp.com/agent-inject: "true"
        
        # Vault role to authenticate as
        vault.hashicorp.com/role: "payment-svc"
        
        # Inject database credentials
        vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/payment-readwrite"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "database/creds/payment-readwrite" -}}
          {
            "username": "{{ .Data.username }}",
            "password": "{{ .Data.password }}"
          }
          {{- end }}
        
        # Inject static API keys
        vault.hashicorp.com/agent-inject-secret-stripe: "secret/data/payment/stripe"
        vault.hashicorp.com/agent-inject-template-stripe: |
          {{- with secret "secret/data/payment/stripe" -}}
          {{ .Data.data.api_key }}
          {{- end }}

        # Auto-renew dynamic secrets before TTL expires
        vault.hashicorp.com/agent-inject-command-db-creds: "/bin/sh -c 'kill -HUP $(pidof payment-svc)'"
        
    spec:
      serviceAccountName: payment-svc-sa
      containers:
        - name: payment-svc
          image: novamart/payment-svc:v2.34.2
          # Secrets are mounted at /vault/secrets/
          # /vault/secrets/db-creds → database credentials JSON
          # /vault/secrets/stripe → Stripe API key
```

```
HOW VAULT AGENT SIDECAR WORKS:

┌─────────────────────────────────────────────┐
│ Pod                                         │
│                                             │
│ ┌─────────────────┐  ┌──────────────────┐  │
│ │ Vault Agent     │  │ payment-svc      │  │
│ │ (init + sidecar)│  │ (main container) │  │
│ │                 │  │                  │  │
│ │ 1. Auth to Vault│  │ 3. Reads secrets │  │
│ │    via K8s SA   │  │    from shared   │  │
│ │                 │  │    volume at     │  │
│ │ 2. Fetches      │  │    /vault/secrets│  │
│ │    secrets from │  │                  │  │
│ │    Vault server │  │ 4. When secrets  │  │
│ │                 │  │    are renewed,  │  │
│ │ 5. Renews leases│  │    agent sends   │  │
│ │    before TTL   │  │    SIGHUP to app │  │
│ │    expires      │  │    (or app polls │  │
│ │                 │  │    the file)     │  │
│ │ 6. Writes new   │  │                  │  │
│ │    secrets to   │  │                  │  │
│ │    shared volume│  │                  │  │
│ └────────┬────────┘  └──────────────────┘  │
│          │                    ▲             │
│          │   Shared Volume    │             │
│          └────────────────────┘             │
│           /vault/secrets/                   │
└─────────────────────────────────────────────┘

LIFECYCLE:
  Init container (vault-agent-init):
    → Authenticates to Vault
    → Fetches initial secrets
    → Writes to /vault/secrets/
    → Exits

  Sidecar container (vault-agent):
    → Runs continuously alongside the app
    → Monitors lease TTLs
    → Renews leases before expiration
    → Re-fetches secrets if rotated
    → Updates files in /vault/secrets/
    → Optionally signals the app (SIGHUP, command)

  Application:
    → Reads secrets from /vault/secrets/ at startup
    → Either watches files for changes (inotify)
    → Or handles SIGHUP to re-read
    → Or polls the file periodically
```

### Vault vs External Secrets Operator — When to Use Which

```
TWO WAYS TO GET VAULT SECRETS INTO KUBERNETES:

Method 1: Vault Agent Sidecar (Vault-native)
  ┌─────────┐        ┌───────────┐
  │ Pod with │──auth──│ Vault     │
  │ Vault    │←secrets│ Server    │
  │ Agent    │        │           │
  │ sidecar  │        └───────────┘
  └─────────┘
  
  ✅ Dynamic secrets (per-pod DB credentials)
  ✅ Automatic lease renewal
  ✅ No K8s Secret object (secrets never in etcd)
  ✅ Works with ANY Vault secret engine
  ❌ Sidecar per pod (resource overhead)
  ❌ Vault-specific annotations (vendor lock-in)
  ❌ App must read from file, not env var

Method 2: External Secrets Operator (ESO)
  ┌─────────────────┐       ┌───────────┐
  │ External Secrets │──sync─│ Vault     │
  │ Operator         │←──────│ Server    │
  │                  │       └───────────┘
  │ Creates K8s      │       ┌───────────┐
  │ Secret objects   │──sync─│ AWS       │
  │ from external    │←──────│ Secrets   │
  │ sources          │       │ Manager   │
  └────────┬────────┘       └───────────┘
           │
           ▼
  ┌─────────────────┐
  │ K8s Secret      │ ← Standard K8s Secret, usable as env var or volume
  │ (in etcd)       │
  └─────────────────┘
  
  ✅ Standard K8s Secrets (env vars work)
  ✅ Multi-source (Vault, AWS SM, GCP SM, Azure KV)
  ✅ No sidecar needed
  ✅ Works with Reloader for automatic pod restarts
  ❌ Secrets stored in etcd (must enable KMS encryption)
  ❌ No dynamic per-pod credentials
  ❌ Sync delay (polling interval)

NOVAMART STRATEGY:
  Vault Agent: payment-svc, fraud-svc (dynamic DB creds, PCI compliance)
  ESO:         Everything else (simpler, standard K8s patterns)
```

### External Secrets Operator — Production Configuration

```yaml
# SecretStore — connects ESO to Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.vault.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: "external-secrets-sa"
            namespace: "external-secrets"
      caProvider:
        type: ConfigMap
        name: vault-ca
        namespace: vault
        key: ca.crt

---
# SecretStore — connects ESO to AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: "external-secrets-sa"
            namespace: "external-secrets"

---
# ExternalSecret — syncs a specific secret from Vault to K8s
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-svc-db-credentials
  namespace: orders
spec:
  refreshInterval: 5m  # How often to check for updates
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: order-svc-db-credentials  # K8s Secret name
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        DB_HOST: "{{ .db_host }}"
        DB_PORT: "{{ .db_port }}"
        DB_USERNAME: "{{ .username }}"
        DB_PASSWORD: "{{ .password }}"
        DB_CONNECTION_STRING: "postgresql://{{ .username }}:{{ .password }}@{{ .db_host }}:{{ .db_port }}/orders?sslmode=require"
  data:
    - secretKey: db_host
      remoteRef:
        key: secret/data/order/database
        property: host
    - secretKey: db_port
      remoteRef:
        key: secret/data/order/database
        property: port
    - secretKey: username
      remoteRef:
        key: secret/data/order/database
        property: username
    - secretKey: password
      remoteRef:
        key: secret/data/order/database
        property: password

---
# ExternalSecret — syncs from AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: notification-svc-ses-credentials
  namespace: notifications
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: ses-credentials
    creationPolicy: Owner
  data:
    - secretKey: SES_API_KEY
      remoteRef:
        key: notification/ses-credentials
        property: api_key
    - secretKey: SES_API_SECRET
      remoteRef:
        key: notification/ses-credentials
        property: api_secret
```

```yaml
# Reloader — automatically restarts pods when secrets change
# (Installed via Helm: stakater/reloader)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-svc
  namespace: orders
  annotations:
    # Reloader watches this secret — restarts pods on change
    reloader.stakater.com/auto: "true"
    # OR be specific:
    # secret.reloader.stakater.com/reload: "order-svc-db-credentials"
spec:
  template:
    spec:
      containers:
        - name: order-svc
          envFrom:
            - secretRef:
                name: order-svc-db-credentials
```

### Vault Failure Modes

```
FAILURE 1: Vault sealed after restart
  CAUSE: Vault pod restarted but auto-unseal failed (KMS access issue)
  SYMPTOM: All secret reads return 503 "Vault is sealed"
  DEBUG:
    kubectl -n vault exec vault-0 -- vault status
    # Look for: Sealed = true
  FIX: 
    # If auto-unseal KMS key is accessible:
    kubectl -n vault exec vault-0 -- vault operator unseal
    # If IRSA is broken → fix IRSA first (chicken-and-egg problem!)
  PREVENTION: 
    - Test auto-unseal regularly
    - Keep manual unseal keys in secure offline storage
    - Monitor seal status:
      vault_core_unsealed == 0 → ALERT

FAILURE 2: Raft consensus lost
  CAUSE: 2 of 3 Vault nodes lost simultaneously (AZ failure)
  SYMPTOM: Vault becomes read-only or unresponsive
  IMPACT: No new secrets, no lease renewals, no authentication
  FIX: Restore quorum by recovering at least 2 nodes
  PREVENTION: 
    - 3 or 5 nodes across 3 AZs (pod anti-affinity)
    - Regular Raft snapshots:
      vault operator raft snapshot save /tmp/vault-snapshot.snap
    - Store snapshots in S3 (encrypted)

FAILURE 3: Dynamic secret lease expiry during outage
  CAUSE: Vault outage → leases can't be renewed → DB credentials expire
  SYMPTOM: Apps lose DB access when leases expire, even after Vault recovers
  FIX: Apps must handle credential refresh — detect auth failure, 
       re-authenticate, get new credentials
  MITIGATION: Set generous max_ttl (24h) so leases survive brief outages

FAILURE 4: Vault Agent sidecar crash
  CAUSE: Vault Agent OOM, bug, or connectivity issue
  SYMPTOM: Secrets at /vault/secrets/ become stale, app may serve
           with expired credentials
  FIX: 
    - Resource limits on the sidecar (memory, CPU)
    - Liveness probe on the agent:
      vault.hashicorp.com/agent-inject-liveness-probe: "true"
    - App should fail-open or fail-closed based on secret criticality

FAILURE 5: Orphaned dynamic credentials
  CAUSE: Pod deleted but Vault didn't receive revocation request
  SYMPTOM: DB user lingers after pod is gone (security issue)
  FIX: Vault TTL handles this — orphaned creds expire at TTL
  PREVENTION: Set reasonable TTL (1h, not 7d)
  AUDIT: Periodically check for DB users that don't match running pods:
    # List Vault leases
    vault list sys/leases/lookup/database/creds/payment-readwrite
    # Compare with running pods
    kubectl -n payments get pods

FAILURE 6: Vault policy too restrictive — blocks legitimate access
  CAUSE: Policy doesn't include a path the app needs
  SYMPTOM: "permission denied" from Vault (not AWS)
  DEBUG:
    # Check what policies the token has:
    vault token lookup
    # Test policy access:
    vault policy read payment-svc-policy
    # Try the read manually:
    vault read secret/data/payment/stripe
  FIX: Update the policy (through GitOps, not manually)
```

---

## Part 4: Certificate Management

### The TLS Certificate Lifecycle Problem

```
NovaMart has hundreds of TLS certificates:
  - Public-facing: *.novamart.com (ALB, CloudFront)
  - Internal: service-to-service mTLS (Istio)
  - Infrastructure: Vault, Prometheus, Grafana
  - Database: RDS SSL connections
  - Client certificates: mutual TLS for partners

CERTIFICATES EXPIRE. When they do:
  - Public: Customers see browser warnings, can't access site
  - Internal: Service-to-service calls fail (503, connection refused)
  - Infrastructure: Monitoring goes blind, secrets stop syncing
  - Database: App can't connect to DB

Certificate expiry is the #1 preventable outage category.
It's also the most embarrassing — you had 90 days to renew.
```

### ACM — AWS Certificate Manager (Public Certificates)

```
ACM provides FREE public TLS certificates for AWS services:
  - ALB / NLB
  - CloudFront
  - API Gateway
  - Elastic Beanstalk

ACM handles:
  ✅ Certificate issuance (free, unlimited)
  ✅ Automatic renewal (before expiry)
  ✅ Private key management (you never see the private key)
  ✅ Integration with AWS services (one-click attachment)

ACM does NOT:
  ❌ Issue certificates for non-AWS services (on-premise, other cloud)
  ❌ Let you export private keys
  ❌ Work with EC2 directly (must use ALB/NLB in front)
  ❌ Issue certificates for internal/private domains (use ACM Private CA)
```

```hcl
# ACM certificate for NovaMart public domain
resource "aws_acm_certificate" "novamart" {
  domain_name       = "novamart.com"
  validation_method = "DNS"

  subject_alternative_names = [
    "*.novamart.com",
    "api.novamart.com",
    "admin.novamart.com"
  ]

  lifecycle {
    create_before_destroy = true  # Ensure new cert before old one destroyed
  }

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# DNS validation (Route53)
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.novamart.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.main.zone_id
}

resource "aws_acm_certificate_validation" "novamart" {
  certificate_arn         = aws_acm_certificate.novamart.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# Attach to ALB
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.novamart.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }
}
```

### cert-manager — Kubernetes Certificate Automation

```
cert-manager handles certificates INSIDE Kubernetes:
  - Ingress TLS certificates (Let's Encrypt, ACM PCA)
  - Internal service certificates
  - Istio gateway certificates (for non-mesh ingress)
  - Custom certificate issuance for any pod

cert-manager automates the full lifecycle:
  Request → Issue → Store (K8s Secret) → Renew → Repeat

┌─────────────────────────────────────────────────────────────────┐
│                  cert-manager ARCHITECTURE                        │
│                                                                  │
│  ┌──────────────┐                                               │
│  │ Certificate  │  "I need a cert for api.novamart.com"         │
│  │ Resource     │                                                │
│  └──────┬───────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                                │
│  │ cert-manager │  Processes the request                         │
│  │ controller   │                                                │
│  └──────┬───────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐     ┌─────────────────────┐                   │
│  │ Issuer /     │────→│ CA Backend          │                    │
│  │ ClusterIssuer│     │ - Let's Encrypt     │                    │
│  └──────────────┘     │ - Vault PKI         │                    │
│                       │ - ACM Private CA    │                    │
│         │             │ - Self-signed       │                    │
│         ▼             └─────────────────────┘                    │
│  ┌──────────────┐                                                │
│  │ K8s Secret   │  Contains: tls.crt + tls.key                  │
│  │ (TLS type)   │  Automatically renewed before expiry          │
│  └──────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
```

```yaml
# ClusterIssuer using Let's Encrypt (for public-facing certificates)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@novamart.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      # DNS-01 challenge (required for wildcard certs)
      - dns01:
          route53:
            region: us-east-1
            # Uses IRSA — no static credentials
        selector:
          dnsZones:
            - "novamart.com"

---
# ClusterIssuer using Vault PKI (for internal mTLS certificates)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-pki
spec:
  vault:
    path: pki_int/sign/novamart-internal
    server: https://vault.vault.svc:8200
    caBundle: <base64-encoded-vault-ca>
    auth:
      kubernetes:
        role: cert-manager
        mountPath: /v1/auth/kubernetes
        serviceAccountRef:
          name: cert-manager
          
---
# Certificate resource — requesting a cert for an internal service
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: payment-svc-tls
  namespace: payments
spec:
  secretName: payment-svc-tls
  duration: 720h      # 30 days
  renewBefore: 168h   # Renew 7 days before expiry
  isCA: false
  privateKey:
    algorithm: ECDSA
    size: 256
  usages:
    - server auth
    - client auth
  dnsNames:
    - payment-svc.payments.svc.cluster.local
    - payment-svc.payments.svc
    - payment-svc
  issuerRef:
    name: vault-pki
    kind: ClusterIssuer
    group: cert-manager.io
```

### Vault PKI — Internal Certificate Authority

```bash
# Set up a root CA and intermediate CA in Vault

# Step 1: Enable PKI secret engine for root CA
vault secrets enable -path=pki pki
vault secrets tune -max-lease-ttl=87600h pki  # 10 years

# Generate root CA
vault write -field=certificate pki/root/generate/internal \
  common_name="NovaMart Root CA" \
  ttl=87600h > root_ca.crt

# Step 2: Enable PKI for intermediate CA
vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int  # 5 years

# Generate intermediate CSR
vault write -format=json pki_int/intermediate/generate/internal \
  common_name="NovaMart Intermediate CA" | jq -r '.data.csr' > intermediate.csr

# Sign intermediate with root
vault write -format=json pki/root/sign-intermediate \
  csr=@intermediate.csr \
  format=pem_bundle \
  ttl=43800h | jq -r '.data.certificate' > intermediate.crt

# Import signed intermediate
vault write pki_int/intermediate/set-signed certificate=@intermediate.crt

# Step 3: Create a role for issuing service certificates
vault write pki_int/roles/novamart-internal \
  allowed_domains="svc.cluster.local,novamart.internal" \
  allow_subdomains=true \
  allow_bare_domains=false \
  max_ttl=720h \
  key_type=ec \
  key_bits=256 \
  require_cn=false \
  generate_lease=true

# Step 4: Test — issue a certificate
vault write pki_int/issue/novamart-internal \
  common_name="payment-svc.payments.svc.cluster.local" \
  alt_names="payment-svc.payments.svc" \
  ttl=720h
```

```
WHY TWO-TIER CA (Root + Intermediate)?

  Root CA:
    - Signs ONLY the intermediate CA certificate
    - Long lifetime (10 years)
    - If compromised → re-issue EVERYTHING
    - Kept as secure as possible (minimal usage)

  Intermediate CA:
    - Signs all service certificates
    - Shorter lifetime (5 years)
    - If compromised → revoke intermediate, issue new one
    - Root CA signs new intermediate → services trust it
    - The ROOT is NOT compromised, so root doesn't change

  This is standard PKI hierarchy. If you used a single CA
  and it was compromised, you'd have to replace the root CA
  on every client/service — far more disruptive.
```

### Certificate Monitoring — Never Get Surprised by Expiry

```yaml
# Prometheus alerts for certificate expiry
groups:
  - name: certificate-alerts
    rules:
      # cert-manager certificates
      - alert: CertificateExpiringSoon
        expr: |
          certmanager_certificate_expiration_timestamp_seconds - time() < 7 * 24 * 3600
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Certificate {{ $labels.name }} in {{ $labels.namespace }} expires in less than 7 days"
          runbook: "https://wiki.novamart.internal/runbooks/certificate-renewal"
          
      - alert: CertificateExpiryCritical
        expr: |
          certmanager_certificate_expiration_timestamp_seconds - time() < 24 * 3600
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "CRITICAL: Certificate {{ $labels.name }} expires in less than 24 hours"

      # cert-manager not ready (renewal failing)
      - alert: CertificateNotReady
        expr: |
          certmanager_certificate_ready_status{condition="True"} == 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Certificate {{ $labels.name }} is not ready — renewal may be failing"

      # External certificates (probe from blackbox exporter)
      - alert: ExternalCertExpiring
        expr: |
          probe_ssl_earliest_cert_expiry - time() < 30 * 24 * 3600
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "External cert for {{ $labels.instance }} expires in < 30 days"
```

```yaml
# Blackbox exporter config — probe external TLS endpoints
modules:
  tls_check:
    prober: tcp
    timeout: 5s
    tcp:
      tls: true
      tls_config:
        insecure_skip_verify: false

# ServiceMonitor target for external endpoints
apiVersion: monitoring.coreos.com/v1
kind: Probe
metadata:
  name: external-tls-check
  namespace: monitoring
spec:
  interval: 1h
  module: tls_check
  prober:
    url: blackbox-exporter.monitoring:9115
  targets:
    staticConfig:
      targets:
        - api.novamart.com:443
        - admin.novamart.com:443
        - grafana.novamart.internal:443
        - vault.novamart.internal:8200
```

### Certificate Failure Modes

```
FAILURE 1: Let's Encrypt rate limit hit
  CAUSE: Too many certificate requests (5 certs per domain per week)
  SYMPTOM: cert-manager logs show "too many certificates already issued"
  FIX: Use staging endpoint for testing, production only for real certs
  PREVENTION: Use wildcard cert (*.novamart.com) — ONE cert, all subdomains

FAILURE 2: DNS-01 challenge fails
  CAUSE: cert-manager can't create Route53 TXT record (IRSA permissions,
         wrong hosted zone, DNS propagation delay)
  SYMPTOM: Certificate stuck in "Pending" state
  DEBUG:
    kubectl describe certificate <name>
    kubectl describe certificaterequest <name>
    kubectl describe order <name>
    kubectl describe challenge <name>
    # The challenge resource shows the exact failure
  FIX: Check IRSA role has route53:ChangeResourceRecordSets permission

FAILURE 3: Vault PKI intermediate CA expired
  CAUSE: Intermediate CA certificate expired (5 year lifetime)
  SYMPTOM: ALL internal certs fail to renew simultaneously
  IMPACT: Mass mTLS failure — services can't communicate
  PREVENTION: Alert at 6 months before intermediate expiry
  FIX: Generate new intermediate, sign with root, distribute

FAILURE 4: Certificate/private key mismatch
  CAUSE: cert-manager stores cert in one Secret, app expects different format
  SYMPTOM: "tls: private key does not match public key" in app logs
  FIX: Verify Secret contains matching tls.crt and tls.key:
    kubectl -n payments get secret payment-svc-tls -o json | \
      jq -r '.data["tls.crt"]' | base64 -d | openssl x509 -noout -modulus | md5sum
    kubectl -n payments get secret payment-svc-tls -o json | \
      jq -r '.data["tls.key"]' | base64 -d | openssl rsa -noout -modulus | md5sum
    # Both md5sums must match

FAILURE 5: Certificate chain incomplete
  CAUSE: Server sends leaf cert but not intermediate CA cert
  SYMPTOM: Clients with different trust stores get different results
           (Chrome works, curl doesn't, Java app fails)
  FIX: Ensure cert bundle includes full chain:
    cat leaf.crt intermediate.crt > fullchain.crt
  cert-manager handles this correctly, but manual certs often miss it

FAILURE 6: Clock skew breaks certificate validation
  CAUSE: Server or client clock is wrong (VM clock drift, NTP failure)
  SYMPTOM: "certificate is not yet valid" or "certificate has expired"
           even though cert dates look correct
  FIX: Fix NTP on the affected node:
    timedatectl status
    systemctl restart chronyd
```

---

## Part 5: Secrets Management Anti-Patterns and NovaMart Strategy

### The Anti-Patterns

```
ANTI-PATTERN 1: Secrets in environment variables (partially)
  RISK: env vars show up in:
    - /proc/<pid>/environ (any process on the container can read)
    - Docker inspect
    - kubectl describe pod (if set directly in pod spec)
    - Core dumps
    - Child process inheritance
  REALITY: This is still the most COMMON pattern in Kubernetes
  MITIGATION:
    - Use K8s Secrets (not configMaps) for sensitive values
    - Enable etcd encryption (KMS envelope encryption)
    - Prefer volume mounts over env vars for highly sensitive secrets
    - Never put secrets directly in Deployment spec (always reference Secret)
  NUANCE: For most secrets, envFrom a K8s Secret is acceptable.
          For PCI/highly sensitive: use Vault Agent file mount.

ANTI-PATTERN 2: Secrets in container images
  RISK: Anyone with ECR access can pull the image and extract secrets
  DETECTION: Image scanning (Trivy, Grype) for known secret patterns
  FIX: NEVER bake secrets into images. Always inject at runtime.
  ALSO: Docker build args (ARG) are visible in image history:
    docker history <image> → shows build args in plaintext

ANTI-PATTERN 3: Secrets in Git (even "private" repos)
  RISK: Git history is permanent, private repos become public (acq, leak)
  DETECTION: git-secrets, trufflehog, GitGuardian, gitleaks
  PREVENTION: Pre-commit hooks that scan for secrets
  RECOVERY: Rotate secret FIRST, then clean Git history with git-filter-repo

ANTI-PATTERN 4: Same secret everywhere (shared credentials)
  RISK: One compromise → everything breached
  FIX: Per-service credentials (Vault dynamic secrets are ideal)
  MINIMUM: Per-environment credentials (dev ≠ staging ≠ production)

ANTI-PATTERN 5: No rotation (set and forget)
  RISK: Compromised credentials remain valid forever
  FIX: Automated rotation (Secrets Manager, Vault dynamic secrets)
  COMPLIANCE: PCI-DSS requires credential rotation (at minimum annually)

ANTI-PATTERN 6: Secrets in Terraform state
  RISK: Terraform state contains all resource attributes in plaintext,
        including passwords passed as variables
  FIX: 
    - Use Secrets Manager for the password, reference it in Terraform
    - Enable S3 encryption for state backend
    - Use manage_master_user_password for RDS (AWS manages the secret)
    - Never terraform output a secret value
```

### NovaMart Secrets Architecture — Complete Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│              NOVAMART SECRETS FLOW                                    │
│                                                                      │
│  SECRET CREATION:                                                    │
│  ┌────────────┐  ┌────────────────┐  ┌─────────────────────────┐    │
│  │ Terraform  │  │ Vault CLI/API  │  │ AWS Console/CLI         │    │
│  │ (infra     │  │ (app secrets,  │  │ (RDS managed passwords, │    │
│  │  secrets)  │  │  dynamic creds)│  │  Secrets Manager)       │    │
│  └─────┬──────┘  └───────┬────────┘  └───────────┬─────────────┘    │
│        │                 │                        │                  │
│        ▼                 ▼                        ▼                  │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                 SECRET STORES                                │    │
│  │                                                              │    │
│  │  ┌─────────────────┐  ┌──────────────────────────────────┐  │    │
│  │  │ HashiCorp Vault │  │ AWS Secrets Manager               │  │    │
│  │  │                 │  │                                   │  │    │
│  │  │ • Dynamic DB    │  │ • RDS auto-rotation               │  │    │
│  │  │   credentials   │  │ • AWS-native service creds       │  │    │
│  │  │ • PKI (internal │  │ • Third-party API keys           │  │    │
│  │  │   CA + certs)   │  │   (rotation via Lambda)          │  │    │
│  │  │ • Transit       │  │                                   │  │    │
│  │  │   encryption    │  │                                   │  │    │
│  │  │ • Static KV     │  │                                   │  │    │
│  │  │   (cross-cloud) │  │                                   │  │    │
│  │  └────────┬────────┘  └──────────────┬────────────────────┘  │    │
│  │           │                          │                       │    │
│  └───────────┼──────────────────────────┼───────────────────────┘    │
│              │                          │                            │
│  SECRET DELIVERY TO KUBERNETES:                                      │
│              │                          │                            │
│  ┌───────────▼──────────────────────────▼───────────────────────┐    │
│  │           External Secrets Operator                           │    │
│  │  Syncs Vault + AWS SM → K8s Secrets (polling, 5-15min)       │    │
│  └──────────────────────────┬───────────────────────────────────┘    │
│                             │                                        │
│  ┌──────────────────────────▼───────────────────────────────────┐    │
│  │                    K8s Secrets (etcd)                          │    │
│  │           Encrypted with KMS envelope encryption              │    │
│  └──────────────────────────┬───────────────────────────────────┘    │
│                             │                                        │
│  ┌──────────────────────────▼───────────────────────────────────┐    │
│  │                    APPLICATION PODS                            │    │
│  │                                                               │    │
│  │  Non-PCI services:        PCI services (payment-svc):        │    │
│  │  envFrom K8s Secret       Vault Agent sidecar                │    │
│  │  (via ESO)                (dynamic per-pod DB creds)         │    │
│  │                           File mount at /vault/secrets/       │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  SECRET ROTATION:                                                    │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Vault dynamic:  Automatic (TTL-based, per-pod)              │    │
│  │  Vault static:   Manual or CI-triggered rotation             │    │
│  │  AWS SM + RDS:   Automatic (Lambda rotation, 30-day)         │    │
│  │  K8s Secrets:    ESO re-syncs → Reloader restarts pods       │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  AUDIT:                                                              │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Vault:    Audit device → file/syslog → Loki                │    │
│  │  AWS SM:   CloudTrail → S3 (immutable) + CloudWatch alarm   │    │
│  │  KMS:      CloudTrail → every encrypt/decrypt logged        │    │
│  │  K8s:      Audit logs → who read which Secret, when         │    │
│  └──────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

### Kubernetes etcd Encryption — Protecting Secrets at Rest

```yaml
# EKS: Enable envelope encryption for Secrets in etcd
# This is configured at the EKS cluster level

# Terraform
resource "aws_eks_cluster" "main" {
  name     = "novamart-prod"
  role_arn = aws_iam_role.eks_cluster.arn

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks_secrets.arn
    }
    resources = ["secrets"]  # Encrypts all K8s Secrets in etcd
  }
}

# Without this:
# K8s Secrets are base64 encoded (NOT encrypted) in etcd
# Anyone with etcd access can read all secrets in plaintext
# This is a COMPLIANCE REQUIREMENT for PCI/SOC2

# With this:
# K8s Secrets are encrypted with the KMS key before storing in etcd
# Even etcd backups are encrypted
# Only the EKS control plane (via KMS) can decrypt
```

## Quick Reference Card

```
KMS
────
Envelope encryption: CMK (in KMS) encrypts Data Key, Data Key encrypts data
ALWAYS include root in key policy (lock-out prevention)
Separate admin (manage) from usage (encrypt/decrypt) in key policy
kms:ViaService — restrict key use to specific AWS services
Enable rotation: 365 days default, 90-2560 configurable
CMK for compliance (PCI, SOC2). AWS managed keys can't cross accounts.

SECRETS MANAGER
────────────────
$0.40/secret/month + API calls. Use for credentials that rotate.
Auto-rotation: Lambda-based, 4-step lifecycle (create→set→test→finish)
Multi-user rotation: zero-downtime (alternate between 2 DB users)
AWSCURRENT / AWSPREVIOUS staging labels
App MUST handle credential refresh (don't cache at startup forever)
manage_master_user_password = true for RDS (simplest approach)

VAULT
─────
Dynamic secrets: per-pod, TTL-based, auto-revoke. Killer feature.
K8s auth: ServiceAccount → Vault token → secret access
Vault Agent sidecar: for PCI/dynamic (file mount, auto-renew)
ESO: for everything else (K8s-native, multi-source)
Auto-unseal with AWS KMS (avoid manual unseal in production)
Raft with 3 nodes across 3 AZs. Regular snapshots.
Policies: path-based, capabilities (read, create, update, delete, deny)

CERTIFICATES
────────────
ACM: Free public certs, auto-renewal, ALB/CloudFront only
cert-manager: K8s certificate automation (Let's Encrypt, Vault PKI)
Vault PKI: Internal CA (root + intermediate hierarchy)
MONITOR: Alert at 30d (warning), 7d (warning), 24h (critical)
NEVER let certs expire — most preventable outage category

ANTI-PATTERNS
─────────────
❌ Secrets in Git (even private repos)
❌ Secrets in container images (ARG visible in history)
❌ Same credential for all environments
❌ No rotation (set and forget)
❌ Secrets in Terraform state (use manage_master_user_password)
❌ Unencrypted etcd (base64 ≠ encryption)
```

---

## Retention Questions — Phase 6, Lesson 2

### Q1: Secret Rotation Gone Wrong 🔥

**Scenario:** Tuesday 6 AM. PagerDuty alert: `PaymentServiceDatabaseConnectionFailure`. payment-svc pods are all returning 500 errors. CloudWatch shows the Secrets Manager rotation Lambda ran at 5:45 AM.

1. Walk through your investigation — what do you check first, second, third? Give exact commands. The rotation Lambda's CloudWatch logs show all 4 steps completed "successfully."
2. You discover: the new password was set in Secrets Manager (AWSCURRENT updated) and in PostgreSQL (password changed), BUT payment-svc is still using the OLD password. The pods haven't restarted. What broke in the delivery chain? List ALL possible causes (at least 4) and the check for each.
3. How would you mitigate this RIGHT NOW (restore payment processing) without rolling back the rotation? Give two approaches with exact commands.
4. Design the system that prevents this from ever happening again. Cover: how the secret flows from Secrets Manager to the pod, what triggers a pod restart, and how you verify the new password works before the old one is invalidated.

### Q2: KMS Key Policy Disaster 🔥

**Scenario:** A junior engineer was tasked with restricting the payment KMS key to only the payment team. They deployed this key policy via Terraform:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PaymentTeamOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::888888888888:role/payment-svc-irsa-role"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

1. What is catastrophically wrong with this key policy? Identify ALL the problems (there are at least 3).
2. The Terraform apply succeeded. Now nobody — not even the account root user — can modify or manage this key. Why? What specific IAM principle makes this true?
3. How do you recover? Walk through the exact recovery process. (Hint: this is one of the hardest recovery scenarios in AWS.)
4. Write the corrected key policy that follows the principle of separation of duties, includes the safety nets, and supports the NovaMart architecture.

### Q3: Vault Outage During Peak Traffic 🔥

**Scenario:** Black Friday. 3x normal traffic. Vault's Raft cluster loses quorum — 2 of 3 nodes failed simultaneously (both were in us-east-1a, which had a network partition). All Vault operations are failing.

1. What is the immediate impact on NovaMart's services? Think through EACH secret delivery mechanism (Vault Agent sidecars, ESO, cert-manager) and explain what happens to each.
2. The payment-svc pods have Vault Agent sidecars with dynamic DB credentials that have a 1-hour TTL. The credentials were last refreshed 40 minutes ago. You have 20 minutes before payment processing fails entirely. What do you do?
3. After restoring Vault (recovering the Raft cluster), you discover that 50+ dynamic database credentials were orphaned — the leases expired during the outage, but Vault couldn't revoke the PostgreSQL users. What's the security implication and how do you clean up?
4. Design the architecture changes that make Vault resilient to this exact scenario. Include HA, backup, and graceful degradation strategies.

### Q4: Complete Secret Compromise Response 🔥

**Scenario:** GitGuardian alerts: a developer pushed a commit to a feature branch containing `payment/rds/master` credentials from Secrets Manager — the full JSON blob with username, password, host, and port for the production payment database. The commit was pushed 4 hours ago. The repository is private but has 85 contributors with read access.

1. Write your first 10 actions in order. Be precise about what you do FIRST — the order matters critically. Include exact commands.
2. After revoking the credential, how do you determine if the credential was used maliciously during the 4-hour window? Give specific CloudTrail and RDS queries.
3. The compromised credential is the RDS master user. Rotating it requires a brief RDS restart. It's 2 PM (peak traffic). The CISO says "rotate now." The VP of Product says "wait until midnight." Who's right and why?
4. What systemic changes prevent this class of incident? Cover: pre-commit hooks, CI scanning, secret architecture, and developer education. Give specific tool configurations, not just "use pre-commit hooks."

---

# Phase 6, Lesson 2 — Answers: Secrets, KMS, Vault & Incident Response

---

## Q1: Secret Rotation Gone Wrong

### Part 1: Investigation Sequence

**MOST IMPORTANT FIRST**: Confirm scope and financial impact. payment-svc returning 500s at NovaMart = **$50K/min bleeding right now.** Start the incident clock.

```bash
# STEP 0: Declare incident, start timer (run from your laptop/jump box)
# Post to #incident-payment in Slack:
# "SEV1: payment-svc 500s. Suspected secret rotation issue. IC: <you>. Start: 06:02 UTC"

# STEP 1: Confirm the symptom — are ALL pods failing or just some?
# Run from: kubectl context pointed at prod EKS cluster
kubectl get pods -n payment -l app=payment-svc -o wide
kubectl logs -n payment -l app=payment-svc --tail=50 | grep -i "password\|auth\|login\|FATAL\|connection refused"

# What you're looking for: "password authentication failed for user" or 
# "FATAL: password authentication failed" — confirms credential mismatch
# vs "connection refused" which would be a network/RDS issue (different problem)
```

```bash
# STEP 2: Confirm the rotation Lambda actually ran and what it changed
# Run from: AWS CLI with security/audit account access
aws logs filter-log-events \
  --log-group-name "/aws/lambda/SecretsManager-payment-rds-rotation" \
  --start-time $(date -d '2 hours ago' +%s)000 \
  --filter-pattern "?finishSecret ?setSecret ?testSecret ?createSecret" \
  --region us-east-1

# The 4 rotation steps are: createSecret → setSecret → testSecret → finishSecret
# "All completed successfully" is suspicious — testSecret SHOULD have caught a mismatch
```

```bash
# STEP 3: Verify what Secrets Manager currently holds — check version stages
aws secretsmanager describe-secret \
  --secret-id payment/rds/master \
  --region us-east-1 \
  --query '{Versions: VersionIdsToStages}' --output json

# Expected output: two versions — one with AWSCURRENT, one with AWSPREVIOUS
# This tells you the rotation completed the staging label swap
```

```bash
# STEP 4: Verify the password in Secrets Manager actually works against RDS
# Run from: a bastion host or pod with psql and network access to the RDS instance
# First, get the current secret value
aws secretsmanager get-secret-value \
  --secret-id payment/rds/master \
  --version-stage AWSCURRENT \
  --region us-east-1 \
  --query 'SecretString' --output text | jq -r '.password'

# Then test it (DO NOT log the password — pipe directly)
PGPASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id payment/rds/master \
  --version-stage AWSCURRENT \
  --region us-east-1 \
  --query 'SecretString' --output text | jq -r '.password') \
psql -h payment-db.cluster-xxxx.us-east-1.rds.amazonaws.com \
  -U payment_master -d payments -c "SELECT 1;"

# If this works: Secrets Manager + RDS are in sync. Problem is delivery to pods.
# If this fails: rotation Lambda's testSecret step lied — investigate Lambda code.
```

```bash
# STEP 5: Check what the PODS actually have
# What's in the Kubernetes Secret object right now?
kubectl get secret payment-db-credentials -n payment -o jsonpath='{.data.password}' | base64 -d

# Compare this to the AWSCURRENT value from Step 4
# If they differ → the Kubernetes Secret was NOT updated after rotation
# This is your smoking gun: the delivery chain broke between Secrets Manager and K8s
```

**Investigation order rationale:**
1. Confirm symptom scope (maybe it's partial — different mitigation)
2. Confirm root cause hypothesis (rotation actually ran)
3. Verify state at each link in the chain: Lambda → Secrets Manager → RDS → K8s Secret → Pod
4. The chain link where values diverge is where the break is

### Part 2: Delivery Chain Failure — All Possible Causes

The secret traveled: `Secrets Manager → (???) → Kubernetes Secret → Pod mount/env → Application connection pool`

Something broke in `(???)`. Here are **all** the failure modes:

| # | Failure Mode | Check Command | Why It Happens |
|---|---|---|---|
| **1** | **ESO `ExternalSecret` refreshInterval hasn't elapsed** | `kubectl get externalsecret payment-db-credentials -n payment -o jsonpath='{.spec.refreshInterval}'` | If refreshInterval is `1h` and rotation happened 15 min ago, ESO hasn't polled yet. Default is often `1h`. |
| **2** | **ESO SecretStore auth is broken** — the IRSA role for ESO can't call `GetSecretValue` | `kubectl get secretstore -n payment -o yaml` then check the IRSA role: `kubectl describe sa external-secrets -n payment` and `aws iam get-role-policy` for the ESO role | ESO silently fails to sync if it can't authenticate. Check `kubectl get externalsecret -n payment` — status will show `SecretSyncedError`. |
| **3** | **ESO is healthy BUT the pod reads the secret as an environment variable, not a volume mount** | `kubectl get pod <pod> -n payment -o jsonpath='{.spec.containers[0].env}' \| jq .` — look for `valueFrom.secretKeyRef` vs checking `volumeMounts` | **Env vars are injected at pod creation time and NEVER update.** Even if the K8s Secret changes, the env var is static. The pod must restart. Volume-mounted secrets auto-update (kubelet sync period, ~60-120s). |
| **4** | **Application connection pool caches credentials** — even if the file/env is updated, the app holds open connections with the old password | Inspect app config: does it use PgBouncer? HikariCP? Check `max-lifetime` setting. `kubectl exec <pod> -n payment -- cat /app/config/datasource.yaml` | JDBC connection pools (HikariCP, c3p0) keep connections alive. New connections use new creds, but if `maxLifetime` is 30min+, old connections persist. The pool must be refreshed. |
| **5** | **No pod restart trigger exists** — nobody configured a mechanism to restart pods when the K8s Secret changes | `kubectl get deployment payment-svc -n payment -o yaml \| grep -A5 "annotations\|reloader\|checksum"` | Kubernetes **does not restart pods when a Secret changes.** You need either: Reloader, a checksum annotation in the pod template, or an ESO webhook trigger. If none exist, pods run forever with stale secrets. |
| **6** | **ESO is running but the `ExternalSecret` targets the wrong secret path or version** | `kubectl get externalsecret payment-db-credentials -n payment -o jsonpath='{.spec.data}'` | If the ExternalSecret has `versionStage: AWSPREVIOUS` hardcoded or the `key` path is wrong, ESO faithfully syncs the wrong value. |
| **7** | **Kubelet volume sync delay** — even with volume mounts, kubelet only syncs projected/secret volumes on its sync period | `kubelet --sync-frequency` defaults to `1m`, but secret updates can take up to `ttl + sync-frequency` | Up to `kubelet-sync-frequency (60s) + configmap/secret cache TTL (default 0 but often set to 60s)` = up to ~2 min. At 5:45 rotation, this should have resolved by 6:00, so this alone doesn't explain the 6 AM alert. More likely cause #3 or #5. |

**The most likely root cause for NovaMart:** A combination of **#3 (env var injection)** and **#5 (no restart trigger)**. The payment-svc was deployed with `env.valueFrom.secretKeyRef`, and no Reloader/checksum mechanism exists. The K8s Secret may or may not have updated (depends on ESO config), but it doesn't matter — the pods would need a restart regardless.

### Part 3: Immediate Mitigation (Restore Payment Processing)

**Do NOT roll back the rotation.** The old password is now `AWSPREVIOUS` in Secrets Manager and **the RDS password has already been changed to the new one.** Rolling back Secrets Manager labels without rolling back the RDS password makes things worse.

**Approach 1: Rolling restart of payment-svc pods (Preferred — fastest)**

```bash
# Forces all pods to restart and re-read the K8s Secret
# If the K8s Secret already has the new password (ESO synced), this fixes it immediately
# Run from: kubectl context with prod access

# FIRST: Verify the K8s Secret has the CURRENT password
NEW_PW=$(aws secretsmanager get-secret-value \
  --secret-id payment/rds/master \
  --version-stage AWSCURRENT \
  --region us-east-1 \
  --query 'SecretString' --output text | jq -r '.password')
K8S_PW=$(kubectl get secret payment-db-credentials -n payment \
  -o jsonpath='{.data.password}' | base64 -d)

if [ "$NEW_PW" = "$K8S_PW" ]; then
  echo "K8s Secret matches Secrets Manager AWSCURRENT — rolling restart will fix it"
  kubectl rollout restart deployment/payment-svc -n payment
  kubectl rollout status deployment/payment-svc -n payment --timeout=120s
else
  echo "K8s Secret is STALE — must update K8s Secret first (see Approach 2)"
fi
```

```bash
# VERIFY after restart:
kubectl logs -n payment -l app=payment-svc --tail=10 | grep -i "connected\|ready\|pool"
# Check pod readiness
kubectl get pods -n payment -l app=payment-svc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

**Approach 2: Force-sync the K8s Secret, THEN rolling restart**

```bash
# If ESO hasn't synced yet — force it
# Option A: Annotate the ExternalSecret to trigger immediate reconciliation
kubectl annotate externalsecret payment-db-credentials \
  -n payment \
  force-sync=$(date +%s) --overwrite

# Verify the sync happened:
kubectl get externalsecret payment-db-credentials -n payment \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].message}'
# Should show: "Secret was synced"

# Option B: If ESO is broken or too slow — manually patch the K8s Secret directly
# THIS IS AN EMERGENCY MEASURE — ArgoCD/ESO will overwrite it on next sync, which is fine
# because by then ESO should be syncing the correct value
SECRET_JSON=$(aws secretsmanager get-secret-value \
  --secret-id payment/rds/master \
  --version-stage AWSCURRENT \
  --region us-east-1 \
  --query 'SecretString' --output text)

kubectl create secret generic payment-db-credentials \
  -n payment \
  --from-literal=username=$(echo "$SECRET_JSON" | jq -r '.username') \
  --from-literal=password=$(echo "$SECRET_JSON" | jq -r '.password') \
  --from-literal=host=$(echo "$SECRET_JSON" | jq -r '.host') \
  --from-literal=port=$(echo "$SECRET_JSON" | jq -r '.port') \
  --dry-run=client -o yaml | kubectl apply -f -

# NOW rolling restart to pick up the new values
kubectl rollout restart deployment/payment-svc -n payment
kubectl rollout status deployment/payment-svc -n payment --timeout=120s
```

```bash
# VERIFY payment processing is restored:
# 1. Check pods are Ready
kubectl get pods -n payment -l app=payment-svc

# 2. Check application logs for successful DB connections
kubectl logs -n payment -l app=payment-svc --tail=5 | grep -i "connected\|pool initialized"

# 3. Check actual payment flow (hit health/readiness endpoint)
kubectl exec -n payment deploy/payment-svc -- wget -qO- http://localhost:8080/health/ready

# 4. Verify in Grafana: payment_svc_http_requests_total{status="500"} should drop to 0
#    payment_svc_db_connection_pool_active should show healthy connections
```

**Rollback plan if rolling restart makes things worse:** `kubectl rollout undo deployment/payment-svc -n payment` — but note that undoing won't help because the OLD pods also can't connect (old password is invalidated in RDS). If both approaches fail, you need to update the RDS password manually to something both old and new pods can use — which means a Secrets Manager manual update.

### Part 4: Prevention Architecture — End-to-End Secret Delivery

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    SECRET ROTATION DELIVERY PIPELINE                         │
│                                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐    ┌─────────────┐  │
│  │  Secrets Mgr │───▶│  Rotation   │───▶│  Secrets Mgr │───▶│    ESO      │  │
│  │  (trigger)   │    │  Lambda     │    │  (AWSCURRENT)│    │  (sync)     │  │
│  └─────────────┘    └──────┬──────┘    └──────────────┘    └──────┬──────┘  │
│                            │                                       │         │
│                   ┌────────▼────────┐                    ┌────────▼───────┐  │
│                   │  testSecret     │                    │  K8s Secret    │  │
│                   │  step validates │                    │  updated       │  │
│                   │  NEW cred works │                    └────────┬───────┘  │
│                   │  AND old cred   │                             │          │
│                   │  still works    │                    ┌────────▼───────┐  │
│                   │  (dual-auth     │                    │  Reloader      │  │
│                   │   window)       │                    │  detects hash  │  │
│                   └─────────────────┘                    │  change        │  │
│                                                          └────────┬───────┘  │
│                                                                   │          │
│                                                          ┌────────▼───────┐  │
│                                                          │  Rolling       │  │
│                                                          │  restart of    │  │
│                                                          │  payment-svc   │  │
│                                                          │  (RollingUpdate│  │
│                                                          │   strategy)    │  │
│                                                          └────────┬───────┘  │
│                                                                   │          │
│                                                          ┌────────▼───────┐  │
│                                                          │  Readiness     │  │
│                                                          │  probe confirms│  │
│                                                          │  DB connection │  │
│                                                          │  works         │  │
│                                                          └────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Layer 1: Secrets Manager Rotation Lambda — Dual-Auth Window**

The key insight: the rotation Lambda's 4 steps must maintain a **dual-auth window** where BOTH old and new passwords work simultaneously. This is what gives the delivery chain time to propagate.

```python
# Lambda rotation handler — critical: the ALTERNATING USER strategy
# For RDS, use the "alternating users" rotation strategy, NOT "single user"
#
# Single user: changes password on the same user → instant mismatch risk
# Alternating users: maintains TWO database users (payment_app, payment_app_clone)
#   - AWSCURRENT points to payment_app (password A)
#   - Rotation creates/updates payment_app_clone with password B
#   - After testSecret validates, swaps AWSCURRENT to payment_app_clone
#   - Next rotation: updates payment_app with password C, swaps back
#   - AT ALL TIMES, the previous user+password still works until overwritten
#
# AWS provides this as a built-in Lambda:
# arn:aws:serverlessrepo:us-east-1:297356227824:applications/SecretsManagerRDSPostgreSQLRotationMultiUser
```

```hcl
# Terraform: Secrets Manager rotation with multi-user strategy
resource "aws_secretsmanager_secret_rotation" "payment_db" {
  secret_id           = aws_secretsmanager_secret.payment_db.id
  rotation_lambda_arn = aws_lambda_function.rotation.arn

  rotation_rules {
    automatically_after_days = 30
    # Duration: how long both old and new credentials are valid
    # Must be longer than ESO sync + pod restart time
    # ESO sync: 5 min + pod rolling restart: 3 min + buffer = set 1 hour minimum
    duration = "2h"
  }
}

# The master secret (used by Lambda to create/alter the app users)
resource "aws_secretsmanager_secret" "payment_db_master" {
  name        = "payment/rds/master"
  description = "RDS master credentials — used only by rotation Lambda"
  kms_key_id  = aws_kms_key.payment_secrets.arn
}

# The application secret (what payment-svc actually uses)
resource "aws_secretsmanager_secret" "payment_db_app" {
  name        = "payment/rds/app"
  description = "Rotated application credentials for payment-svc"
  kms_key_id  = aws_kms_key.payment_secrets.arn
}
```

**Layer 2: External Secrets Operator — Aggressive Polling + Status Monitoring**

```yaml
# ExternalSecret with short refresh interval and explicit status tracking
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-db-credentials
  namespace: payment
spec:
  refreshInterval: 5m    # Poll every 5 minutes — NOT 1 hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-db-credentials
    creationPolicy: Owner
    template:
      metadata:
        annotations:
          # This annotation changes on every sync, which Reloader watches
          secret-rotated-at: "{{ .Annotations.reconciled }}"
  data:
    - secretKey: username
      remoteRef:
        key: payment/rds/app
        property: username
    - secretKey: password
      remoteRef:
        key: payment/rds/app
        property: password
    - secretKey: host
      remoteRef:
        key: payment/rds/app
        property: host
    - secretKey: port
      remoteRef:
        key: payment/rds/app
        property: port
---
# Alert if ESO sync fails
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: eso-sync-alerts
  namespace: monitoring
spec:
  groups:
    - name: eso.rules
      rules:
        - alert: ExternalSecretSyncFailed
          expr: |
            external_secrets_sync_calls_error{name="payment-db-credentials"} > 0
          for: 10m
          labels:
            severity: critical
            team: payment
          annotations:
            summary: "ExternalSecret payment-db-credentials sync failing for 10m"
            runbook: "https://runbooks.novamart.internal/eso-sync-failure"
```

**Layer 3: Stakater Reloader — Automatic Pod Restart on Secret Change**

```yaml
# Install Reloader (or use annotations on the Deployment)
# Reloader watches for Secret/ConfigMap changes and triggers rolling restarts
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-svc
  namespace: payment
  annotations:
    # Reloader annotation — triggers rolling restart when this Secret changes
    reloader.stakater.com/auto: "true"
    # OR be explicit about which secret:
    # secret.reloader.stakater.com/reload: "payment-db-credentials"
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # Never take more than 1 pod down during rotation
      maxSurge: 2             # Bring up 2 new pods before killing old ones
  template:
    metadata:
      labels:
        app: payment-svc
    spec:
      containers:
        - name: payment-svc
          # CRITICAL: Mount as volume, NOT environment variable
          # Volume mounts get kubelet sync updates; env vars are static at pod start
          volumeMounts:
            - name: db-credentials
              mountPath: /etc/secrets/db
              readOnly: true
          # Readiness probe that validates DB connectivity
          readinessProbe:
            httpGet:
              path: /health/ready   # This endpoint MUST check DB connection pool
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3     # Pod marked NotReady after 30s of DB failure
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
      volumes:
        - name: db-credentials
          secret:
            secretName: payment-db-credentials
```

**Layer 4: Application-Level — Connection Pool Refresh**

```yaml
# Application configuration (Spring Boot / HikariCP example)
# The application MUST re-read credentials from the mounted file, not cache them
spring:
  datasource:
    # Read from volume mount path, NOT hardcoded or env var
    url: jdbc:postgresql://${file:/etc/secrets/db/host}:${file:/etc/secrets/db/port}/payments
    username: ${file:/etc/secrets/db/username}
    password: ${file:/etc/secrets/db/password}
  hikari:
    maximum-pool-size: 20
    minimum-idle: 5
    max-lifetime: 1800000       # 30 minutes — connections cycle, picking up new creds
    connection-test-query: "SELECT 1"
    # If using file watcher pattern, set shorter max-lifetime during rotation
```

**Layer 5: End-to-End Verification — Canary Test After Rotation**

```yaml
# EventBridge rule that triggers verification after rotation
# This catches the case where ALL layers "succeed" but auth actually fails
resource "aws_cloudwatch_event_rule" "rotation_complete" {
  name        = "payment-secret-rotation-complete"
  description = "Triggers verification after secret rotation"

  event_pattern = jsonencode({
    source      = ["aws.secretsmanager"]
    detail-type = ["AWS API Call via CloudTrail"]
    detail = {
      eventName = ["RotationSucceeded"]
      requestParameters = {
        secretId = ["payment/rds/app"]
      }
    }
  })
}

# Lambda that:
# 1. Waits 10 minutes (for ESO sync + pod restart)
# 2. Calls payment-svc /health/ready endpoint
# 3. Queries payment_svc_db_connection_errors_total metric
# 4. If either fails → pages on-call with "ROTATION DELIVERY FAILURE"
```

**Complete timeline for healthy rotation:**

```
T+0:00  Secrets Manager triggers rotation Lambda
T+0:01  createSecret: new password generated, stored as AWSPENDING
T+0:02  setSecret: password set on RDS (ALTER USER)
T+0:03  testSecret: Lambda validates AWSPENDING works against RDS
T+0:04  finishSecret: AWSPENDING promoted to AWSCURRENT, old becomes AWSPREVIOUS
        *** DUAL-AUTH WINDOW OPEN: both passwords work for 2 hours ***
T+5:00  ESO polls Secrets Manager, detects change, updates K8s Secret
T+5:01  Reloader detects K8s Secret hash change, triggers rolling restart
T+5:03  New pods start, mount new credentials via volume, connect to RDS
T+5:04  Readiness probe confirms DB connection works
T+5:10  Old pods terminated (RollingUpdate complete)
T+15:00 EventBridge verification Lambda confirms payment-svc healthy
T+120:00 Dual-auth window closes (old password would be overwritten on next rotation)
```

---

## Q2: KMS Key Policy Disaster

### Part 1: Everything Wrong With This Policy

**Problem 1 (CATASTROPHIC): No root account principal.** The policy does not include:
```json
{
  "Sid": "EnableRootAccountAccess",
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::888888888888:root"},
  "Action": "kms:*",
  "Resource": "*"
}
```

KMS is **unique among all AWS services**: the key policy is the **primary authorizer.** Without the root principal in the key policy, IAM policies are completely ignored for this key. This is unlike S3, SQS, etc., where IAM policies can grant access independently of resource policies. For KMS, if the key policy doesn't grant access (either directly or via the root principal delegation), **no IAM policy can grant it.**

**Problem 2 (CATASTROPHIC): No key administration actions.** The policy only grants data-plane actions (`Encrypt`, `Decrypt`, `GenerateDataKey*`, `DescribeKey`). Nobody has:
- `kms:PutKeyPolicy` — can't fix the policy
- `kms:EnableKey` / `kms:DisableKey` — can't manage lifecycle
- `kms:ScheduleKeyDeletion` — can't even delete the key
- `kms:CreateGrant` — can't delegate access
- `kms:EnableKeyRotation` — can't enable automatic key rotation

**Problem 3 (OPERATIONAL): Single principal.** Only the IRSA role has access. If that role is deleted, modified, or the IRSA trust breaks, **no entity in the AWS account can use this key for anything.** No human can decrypt data encrypted with this key. No admin can troubleshoot.

**Problem 4 (AUDIT): No separation of duties.** Key administrators (who manage the key lifecycle) should be separate from key users (who encrypt/decrypt). This policy has neither — it has one role with partial user permissions and zero admin permissions.

**Problem 5 (SUBTLE): `DescribeKey` without `kms:ListKeys` or `kms:ListAliases`.** The IRSA role can use the key if it knows the key ID/ARN, but can't discover it. Minor compared to the others, but shows the policy wasn't thought through.

### Part 2: Why Even Root Can't Fix This

This is the **KMS key policy trust model**, which is unique in all of AWS:

```
┌─────────────────────────────────────────────────────────────┐
│            KMS KEY POLICY AUTHORIZATION MODEL                │
│                                                             │
│  Every other AWS resource:                                  │
│  ┌──────────┐     ┌─────────────┐                          │
│  │IAM Policy│ OR  │Resource     │ = Access Granted          │
│  │  allows  │     │Policy allows│                           │
│  └──────────┘     └─────────────┘                          │
│                                                             │
│  KMS keys (unique):                                         │
│  ┌──────────────────────┐                                   │
│  │  Key Policy MUST     │──── AND ───▶ Access Granted       │
│  │  grant access        │                                   │
│  │  (directly or via    │    ┌──────────────┐              │
│  │   root delegation)   │────│ If root      │              │
│  └──────────────────────┘    │ principal in  │              │
│                              │ key policy →  │              │
│                              │ IAM policies  │              │
│                              │ can ALSO      │              │
│                              │ grant access  │              │
│                              └──────────────┘              │
│                                                             │
│  The root delegation statement:                             │
│  "Principal": {"AWS": "arn:aws:iam::ACCOUNT:root"}         │
│  "Action": "kms:*"                                          │
│                                                             │
│  This statement does TWO things:                            │
│  1. Allows the root user direct access                      │
│  2. ENABLES IAM policies to grant KMS access to other      │
│     principals (the "delegation" effect)                    │
│                                                             │
│  WITHOUT this statement:                                    │
│  - IAM policies granting kms:* are SILENTLY IGNORED        │
│  - Only principals EXPLICITLY in the key policy have access │
│  - If no principal in the key policy can PutKeyPolicy,      │
│    the key is PERMANENTLY LOCKED                            │
└─────────────────────────────────────────────────────────────┘
```

In the junior engineer's policy:
- The root principal is not listed → IAM policies are inert for this key
- The only listed principal (`payment-svc-irsa-role`) doesn't have `kms:PutKeyPolicy` → nobody can change the key policy
- **The key is now unmanageable.** Even the account root user logging in with MFA cannot modify this key because the key policy doesn't grant root any access, and IAM policies don't apply.

### Part 3: Recovery Process

This is one of the **hardest recovery scenarios in AWS** because you cannot fix it through normal API calls.

```
┌─────────────────────────────────────────────────────────────┐
│                KMS KEY POLICY RECOVERY                       │
│                                                             │
│  Normal KMS key management:                                 │
│    IAM User/Role → kms:PutKeyPolicy → Fix it               │
│    ❌ BLOCKED: Key policy doesn't allow ANY PutKeyPolicy    │
│                                                             │
│  Root user override:                                        │
│    Root login → kms:PutKeyPolicy → Fix it                   │
│    ❌ BLOCKED: Root not in key policy, IAM doesn't apply    │
│                                                             │
│  ONLY recovery path:                                        │
│    AWS Support request FROM the root user account           │
│    → AWS re-adds root principal to key policy               │
│    → You can then fix the rest via normal API calls         │
└─────────────────────────────────────────────────────────────┘
```

**Step-by-step recovery:**

```bash
# STEP 1: Log in as the root user of account 888888888888
# This MUST be the root user (email + password + MFA), not an IAM role
# Navigate to: AWS Console → Support → Create Case

# STEP 2: Create AWS Support case (requires Business or Enterprise support plan)
# Category: Account and Billing → Service: KMS
# Subject: "KMS key policy locked — no principals can manage key"
# Body must include:
# - Key ARN: arn:aws:kms:us-east-1:888888888888:key/<key-id>
# - Region: us-east-1
# - Explanation: Key policy was modified to remove root account access,
#   no remaining principal has kms:PutKeyPolicy permission
# - Request: Re-add default key policy granting root account access

# AWS will verify you ARE the root account holder and will add:
# Principal: arn:aws:iam::888888888888:root, Action: kms:* 
# back to the key policy

# STEP 3: Once AWS Support restores root access (may take hours to days):
# Verify you can now see the key policy
aws kms get-key-policy \
  --key-id arn:aws:kms:us-east-1:888888888888:key/<key-id> \
  --policy-name default \
  --region us-east-1 \
  --output text

# STEP 4: Apply the corrected key policy (see Part 4 below)
aws kms put-key-policy \
  --key-id arn:aws:kms:us-east-1:888888888888:key/<key-id> \
  --policy-name default \
  --policy file://corrected-key-policy.json \
  --region us-east-1

# STEP 5: Verify all intended principals can access the key
aws kms describe-key \
  --key-id arn:aws:kms:us-east-1:888888888888:key/<key-id> \
  --region us-east-1

# STEP 6: Import the key back into Terraform state management
terraform import aws_kms_key.payment <key-id>

# STEP 7: Apply the corrected Terraform to ensure policy is in code
terraform plan -target=aws_kms_key.payment
terraform apply -target=aws_kms_key.payment
```

**While waiting for AWS Support** (this is the production impact analysis):
- **All data encrypted with this key is still accessible** — the `payment-svc-irsa-role` can still `Decrypt` and `Encrypt`. Operations continue.
- **Key rotation cannot be enabled or disabled** — not an emergency but a compliance gap.
- **The key cannot be deleted, disabled, or managed** — if you need to revoke access to the key (e.g., during a security incident), you cannot. This is the real danger.
- **No new grants can be created** — if a new service needs to decrypt payment data, it's blocked.

**Blast radius during recovery window:** Medium. Existing operations continue, but all key management is frozen. If the IRSA role breaks during this window, all payment data becomes inaccessible with no recovery path other than AWS Support.

### Part 4: Corrected Key Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableRootAccountAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::888888888888:root"
      },
      "Action": "kms:*",
      "Resource": "*",
      "Condition": {}
    },
    {
      "Sid": "KeyAdministrators",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::888888888888:role/SecurityTeamAdmin",
          "arn:aws:iam::888888888888:role/InfraTeamAdmin"
        ]
      },
      "Action": [
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
        "kms:CancelKeyDeletion",
        "kms:ReplicateKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "KeyAdministratorsDenyDataOps",
      "Effect": "Deny",
      "Principal": {
        "AWS": [
          "arn:aws:iam::888888888888:role/SecurityTeamAdmin",
          "arn:aws:iam::888888888888:role/InfraTeamAdmin"
        ]
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PaymentServiceDataOperations",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::888888888888:role/payment-svc-irsa-role"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey*",
        "kms:DescribeKey",
        "kms:ReEncrypt*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "rds.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "SecretsManagerAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::888888888888:role/payment-svc-irsa-role"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "AllowKeyRotationLambda",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::888888888888:role/secrets-rotation-lambda-role"
      },
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "AllowGrantsForAWSServices",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::888888888888:role/payment-svc-irsa-role"
      },
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": "true"
        }
      }
    }
  ]
}
```

**Separation of duties in this policy:**

| Principal | Can Manage Key? | Can Encrypt/Decrypt? | Rationale |
|---|---|---|---|
| Root (888888888888) | ✅ Full control | ✅ (emergency only) | Safety net — NEVER locked out. Enables IAM policy delegation. |
| SecurityTeamAdmin | ✅ Full admin | ❌ Explicit Deny | Can rotate key, change policy, enable/disable. CANNOT read payment data. |
| InfraTeamAdmin | ✅ Full admin | ❌ Explicit Deny | Same as Security — operational management without data access. |
| payment-svc-irsa-role | ❌ No admin | ✅ Encrypt/Decrypt (via RDS + Secrets Manager only) | Can use the key through AWS services, but cannot manage or export it. |
| secrets-rotation-lambda | ❌ No admin | ✅ Decrypt/Encrypt (via Secrets Manager only) | Rotation Lambda needs to decrypt old secret, encrypt new one. |

**Terraform for the corrected key with CI guardrails:**

```hcl
resource "aws_kms_key" "payment" {
  description             = "Payment service encryption key - PCI scope"
  deletion_window_in_days = 30    # Maximum window — gives time to recover
  enable_key_rotation     = true  # Automatic annual rotation
  is_enabled              = true
  
  # CRITICAL: key policy applied inline ensures Terraform always manages it
  policy = data.aws_iam_policy_document.payment_key_policy.json

  tags = {
    Service     = "payment-svc"
    Environment = "production"
    Compliance  = "pci-dss"
    ManagedBy   = "terraform"
  }

  # Lifecycle: prevent accidental deletion
  lifecycle {
    prevent_destroy = true
  }
}

# Alias for human-readable reference
resource "aws_kms_alias" "payment" {
  name          = "alias/novamart-payment-prod"
  target_key_id = aws_kms_key.payment.key_id
}
```

**CI prevention layer — Checkov policy to catch this class of mistake:**

```python
# custom_checks/kms_root_access.py
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck
from checkov.common.models.enums import CheckResult, CheckCategories
import json

class KMSKeyPolicyHasRootAccess(BaseResourceCheck):
    def __init__(self):
        name = "Ensure KMS key policy includes root account access"
        id = "CKV_NOVAMART_KMS_001"
        supported_resources = ["aws_kms_key"]
        categories = [CheckCategories.ENCRYPTION]
        super().__init__(name=name, id=id, categories=categories,
                         supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        policy = conf.get("policy", [None])[0]
        if policy is None:
            # No inline policy — AWS default includes root. PASS but warn.
            return CheckResult.PASSED
        
        try:
            policy_doc = json.loads(policy) if isinstance(policy, str) else policy
        except (json.JSONDecodeError, TypeError):
            return CheckResult.UNKNOWN

        for statement in policy_doc.get("Statement", []):
            principal = statement.get("Principal", {})
            aws_principals = principal.get("AWS", [])
            if isinstance(aws_principals, str):
                aws_principals = [aws_principals]
            
            for p in aws_principals:
                if p.endswith(":root") and statement.get("Action") == "kms:*":
                    return CheckResult.PASSED
                if p.endswith(":root") and "kms:*" in statement.get("Action", []):
                    return CheckResult.PASSED

        return CheckResult.FAILED

check = KMSKeyPolicyHasRootAccess()
```

---

## Q3: Vault Outage During Peak Traffic

### Part 1: Impact Analysis Per Secret Delivery Mechanism

```
┌──────────────────────────────────────────────────────────────┐
│           VAULT QUORUM LOSS — IMPACT MATRIX                  │
│                                                              │
│  Mechanism        │ Immediate Impact    │ Time Bomb          │
│  ─────────────────┼─────────────────────┼──────────────────  │
│  Vault Agent      │ ✅ Cached secrets   │ ⏰ TTL expiry:     │
│  Sidecars         │    still served     │    credentials     │
│  (dynamic DB      │    from local disk  │    stop working    │
│   creds)          │    template cache   │    at TTL+grace    │
│                   │                     │    (e.g. 60 min)   │
│  ─────────────────┼─────────────────────┼──────────────────  │
│  External Secrets │ ✅ K8s Secrets      │ ⏰ Secrets become  │
│  Operator (ESO)   │    already exist    │    stale but       │
│  (static secrets) │    in-cluster       │    remain usable   │
│                   │    Pods unaffected  │    until manually   │
│                   │                     │    rotated in Vault │
│  ─────────────────┼─────────────────────┼──────────────────  │
│  cert-manager     │ ✅ Existing certs   │ ⏰ New cert        │
│  (Vault PKI       │    valid until      │    issuance fails  │
│   issuer)         │    expiry           │    Istio mTLS      │
│                   │                     │    renewal fails    │
│                   │                     │    (typ. 24h TTL)   │
│  ─────────────────┼─────────────────────┼──────────────────  │
│  New pod starts   │ ❌ FAILS IMMEDIATELY│ Pod stuck in       │
│  (any mechanism)  │    init container   │ CrashLoopBackOff   │
│                   │    or sidecar can't │ Can't scale up on  │
│                   │    fetch secrets    │ Black Friday!       │
│  ─────────────────┼─────────────────────┼──────────────────  │
│  HPA scaling      │ ❌ CRITICALLY       │ New replicas fail  │
│  events           │    BROKEN           │ to start → cluster │
│                   │                     │ can't scale for    │
│                   │                     │ Black Friday load  │
│  ─────────────────┼─────────────────────┼──────────────────  │
│  ArgoCD deploys   │ ❌ Any deployment   │ Rollback also      │
│                   │    that creates new │ won't work if it   │
│                   │    pods FAILS       │ needs new pods     │
└──────────────────────────────────────────────────────────────┘
```

**Critical cascading failure on Black Friday:**
1. Vault dies → HPA can't scale new pods → traffic increases → existing pods overloaded
2. Existing pods start OOMKilling or hitting connection limits → replacement pods can't start (need Vault)
3. Cascading failure across all Vault-dependent services: payment-svc, order-svc, fraud-svc
4. **$50K/min outage is now multiplied across multiple services**

The **new pod startup failure** is the most dangerous impact because it blocks both scaling AND recovery. Existing pods with cached secrets are a ticking clock, but the inability to scale is an immediate Black Friday killer.

### Part 2: 20-Minute Countdown — Payment Processing Rescue

**Clock: 20 minutes until payment-svc dynamic DB credentials expire.**

```bash
# MINUTE 0-2: Triage and declare
# IC declaration: "SEV1: Vault quorum lost. payment-svc DB creds expire in 20 min.
# Black Friday traffic. All hands."

# PRIORITY 1: Extend the life of existing credentials (buy time)
# The dynamic DB creds were created by Vault in PostgreSQL.
# Vault SETS the password — the password doesn't expire in PostgreSQL.
# The TTL is a VAULT concept — Vault would revoke (DROP USER) at expiry.
# But Vault is DOWN, so it CAN'T revoke. The credentials will keep working
# in PostgreSQL PAST the TTL as long as Vault doesn't come back and revoke them.

# WAIT — this is critical understanding:
# Dynamic secret TTL expiry when Vault is down:
# - Vault Agent sidecar: will STOP serving the cached secret template once
#   the lease_duration + renewal grace period passes (agent considers it invalid)
# - PostgreSQL: the actual password STILL WORKS because Vault can't revoke it
# - The failure is at the CLIENT SIDE: Vault Agent refuses to serve expired secrets

# MINUTE 2-5: Configure Vault Agent to serve stale secrets
# If Vault Agent is configured with exit_after_auth=false and has a cache:
# Check the Vault Agent template configuration in the sidecar

# OPTION A (FASTEST): Bypass Vault Agent — inject credentials directly
# Get the current credentials from any running payment-svc pod
kubectl exec -n payment deploy/payment-svc -c vault-agent -- \
  cat /vault/secrets/db-credentials.json
# Save these credentials — they still work in PostgreSQL

# Create a static K8s Secret with these known-good credentials
kubectl create secret generic payment-db-emergency \
  -n payment \
  --from-literal=username='v-payment-xxxx' \
  --from-literal=password='<password-from-above>' \
  --from-literal=host='payment-db.cluster-xxxx.us-east-1.rds.amazonaws.com' \
  --from-literal=port='5432' \
  --dry-run=client -o yaml | kubectl apply -f -
```

```bash
# MINUTE 5-10: Patch the deployment to use static secret instead of Vault Agent
# This is the emergency bypass — remove Vault dependency entirely

# Save current deployment spec for rollback
kubectl get deployment payment-svc -n payment -o yaml > /tmp/payment-svc-backup.yaml

# Patch: add volume mount for emergency secret, and set env to use it
# The exact patch depends on how the app reads creds, but conceptually:
kubectl patch deployment payment-svc -n payment --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {
      "name": "emergency-db-creds",
      "secret": {
        "secretName": "payment-db-emergency"
      }
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/-",
    "value": {
      "name": "emergency-db-creds",
      "mountPath": "/vault/secrets",
      "readOnly": true
    }
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/env/-",
    "value": {
      "name": "DB_CREDENTIALS_SOURCE",
      "value": "file"
    }
  }
]'

# If the app reads from /vault/secrets/db-credentials.json, create the file format:
kubectl create secret generic payment-db-emergency \
  -n payment \
  --from-literal=db-credentials.json='{"username":"v-payment-xxxx","password":"<pw>","host":"payment-db.cluster-xxxx.us-east-1.rds.amazonaws.com","port":"5432"}' \
  --dry-run=client -o yaml | kubectl apply -f -
```

```bash
# MINUTE 10-15: Remove or disable the Vault Agent sidecar injection
# Annotate pods to skip Vault Agent injection on restart
kubectl patch deployment payment-svc -n payment --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/metadata/annotations/vault.hashicorp.com~1agent-inject",
    "value": "false"
  }
]'

# This triggers a rolling restart without Vault Agent sidecars
# New pods use the static K8s Secret instead
kubectl rollout status deployment/payment-svc -n payment --timeout=300s
```

```bash
# MINUTE 15-18: Verify payment processing works
kubectl get pods -n payment -l app=payment-svc
kubectl logs -n payment -l app=payment-svc --tail=10 | grep -i "connected\|pool\|ready"

# Test a payment flow
kubectl exec -n payment deploy/payment-svc -- \
  wget -qO- http://localhost:8080/health/ready

# Check metrics
# In Grafana: payment_svc_http_requests_total{status="200"} should be climbing
# payment_svc_db_connection_pool_active should show healthy count
```

```bash
# MINUTE 18-20: Confirm HPA can now scale
# New pods no longer need Vault → HPA scaling works again
kubectl get hpa -n payment
# Force a scale-up test:
kubectl scale deployment payment-svc -n payment --replicas=10
kubectl get pods -n payment -l app=payment-svc -w
# Verify new pods start successfully WITHOUT Vault Agent
```

**SIMULTANEOUSLY (parallel workstream): Restore Vault**

```bash
# On the surviving Vault node (the one NOT in us-east-1a):
# Check Raft peer status
vault operator raft list-peers

# If both failed nodes are unreachable, you need to force a single-node quorum
# THIS IS DANGEROUS — only do this if you cannot recover the other nodes

# Option 1: Wait for us-east-1a network partition to resolve (preferred)
# Monitor: ping/traceroute to the failed nodes

# Option 2: Force new cluster from surviving node
vault operator raft remove-peer <node-2-id>
vault operator raft remove-peer <node-3-id>
# Surviving node becomes single-node cluster (has quorum with 1/1)
# Then add new nodes in different AZs

# After Vault is restored — DON'T immediately revert the payment-svc bypass
# Verify Vault is stable, then schedule a maintenance window to switch back
```

### Part 3: Orphaned Credential Cleanup

**Security Implication:**

```
┌──────────────────────────────────────────────────────────────┐
│              ORPHANED DYNAMIC CREDENTIALS                     │
│                                                              │
│  Normal lifecycle:                                           │
│  Vault creates user → TTL expires → Vault runs:             │
│    DROP USER "v-payment-xxxx";                               │
│  ✅ User gone, credentials useless                           │
│                                                              │
│  During outage:                                              │
│  Vault creates user → TTL expires → Vault is DOWN →          │
│    DROP USER never runs                                      │
│  ⚠️  User STILL EXISTS in PostgreSQL with valid password     │
│  ⚠️  These are zombie accounts: nobody's using them,         │
│     nobody's monitoring them, but they CAN be used           │
│                                                              │
│  Risk: If any of these passwords were logged, cached,        │
│  or intercepted, an attacker has valid PostgreSQL             │
│  credentials that Vault doesn't know about and won't revoke  │
│                                                              │
│  Blast radius: 50+ PostgreSQL users across multiple          │
│  databases, each with whatever privileges the Vault          │
│  dynamic role grants (likely SELECT/INSERT/UPDATE/DELETE     │
│  on payment tables)                                          │
└──────────────────────────────────────────────────────────────┘
```

**Cleanup procedure:**

```bash
# STEP 1: Get list of all Vault-created users in PostgreSQL
# Vault dynamic users follow a naming pattern: v-<role>-<random>-<timestamp>
# Run from: bastion with psql access to payment RDS

psql -h payment-db.cluster-xxxx.us-east-1.rds.amazonaws.com \
  -U payment_master -d payments -c "
SELECT usename, valuntil, 
       (SELECT count(*) FROM pg_stat_activity WHERE usename = pg_user.usename) as active_connections
FROM pg_user 
WHERE usename LIKE 'v-%' 
ORDER BY valuntil ASC;"

# Expected: 50+ rows of v-payment-xxxxx users
```

```bash
# STEP 2: Cross-reference with Vault's known leases (after Vault is restored)
vault list sys/leases/lookup/database/creds/payment-readonly
vault list sys/leases/lookup/database/creds/payment-readwrite

# Users that appear in PostgreSQL but NOT in Vault's lease list are orphaned
# Vault may have already lost track of them during the outage
```

```bash
# STEP 3: Revoke all leases in Vault (forces Vault to clean up what it can)
# WARNING: This will also revoke ACTIVE credentials for running pods
# Schedule this AFTER you've switched payment-svc to static creds (Part 2 bypass)

vault lease revoke -prefix database/creds/payment-readonly
vault lease revoke -prefix database/creds/payment-readwrite

# Verify Vault executed the revocations
vault audit list  # Check audit log for revocation entries
```

```bash
# STEP 4: Force-kill remaining orphaned PostgreSQL users that Vault couldn't clean up
# These are users where Vault's revocation failed (user had active connections, etc.)

psql -h payment-db.cluster-xxxx.us-east-1.rds.amazonaws.com \
  -U payment_master -d payments <<'EOF'
-- Terminate all connections from Vault-created users
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE usename LIKE 'v-%' 
  AND pid <> pg_backend_pid();

-- Revoke all privileges and drop the users
DO $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN SELECT usename FROM pg_user WHERE usename LIKE 'v-%'
    LOOP
        EXECUTE format('REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM %I', r.usename);
        EXECUTE format('REVOKE ALL PRIVILEGES ON DATABASE payments FROM %I', r.usename);
        EXECUTE format('DROP USER IF EXISTS %I', r.usename);
        RAISE NOTICE 'Dropped orphaned user: %', r.usename;
    END LOOP;
END
$$;
EOF
```

```bash
# STEP 5: Verify cleanup
psql -h payment-db.cluster-xxxx.us-east-1.rds.amazonaws.com \
  -U payment_master -d payments -c "
SELECT count(*) as remaining_vault_users 
FROM pg_user 
WHERE usename LIKE 'v-%';"
# Expected: 0

# STEP 6: Check RDS audit logs for any suspicious activity during the outage window
# Look for connections from orphaned users that shouldn't have been active
aws rds download-db-log-file-portion \
  --db-instance-identifier payment-db-primary \
  --log-file-name error/postgresql.log.2024-11-29-05 \
  --region us-east-1 \
  --output text | grep -E "v-payment.*connection authorized"
```

### Part 4: Architecture Changes for Vault Resilience

```
┌──────────────────────────────────────────────────────────────────────────┐
│                 VAULT HA ARCHITECTURE — POST-INCIDENT                    │
│                                                                          │
│  us-east-1a          us-east-1b          us-east-1c                     │
│  ┌──────────┐        ┌──────────┐        ┌──────────┐                   │
│  │ Vault    │        │ Vault    │        │ Vault    │                   │
│  │ Node 1   │◄──────▶│ Node 2   │◄──────▶│ Node 3   │                   │
│  │ (Raft)   │  Raft  │ (Leader) │  Raft  │ (Raft)   │                   │
│  └──────────┘  Repl  └──────────┘  Repl  └──────────┘                   │
│       │                    │                    │                         │
│       │         ┌──────────▼──────────┐         │                        │
│       │         │  NLB (cross-AZ)     │         │                        │
│       │         │  Active health      │         │                        │
│       │         │  checks (8200)      │         │                        │
│       │         └─────────────────────┘         │                        │
│       │                                          │                        │
│  ┌────▼────┐                              ┌─────▼────┐                   │
│  │Vault    │                              │Vault     │                   │
│  │Node 4   │   ◄── Additional Raft voter  │Node 5    │  ◄── Non-voter   │
│  │(Raft)   │       5-node = tolerates 2   │(read     │      replica for │
│  └─────────┘       node failures          │replica)  │      read scaling│
│                                           └──────────┘                   │
│                                                                          │
│  DR Region (us-west-2):                                                  │
│  ┌─────────────────────────────────────────┐                            │
│  │  Vault DR Cluster (Performance Standby) │                            │
│  │  - Raft snapshot replication every 1h   │                            │
│  │  - Can be promoted if us-east-1 fails   │                            │
│  │  - NOT active — no split-brain risk     │                            │
│  └─────────────────────────────────────────┘                            │
└──────────────────────────────────────────────────────────────────────────┘
```

**Change 1: 5-node Raft cluster across 3 AZs (minimum)**

```hcl
# Vault server configuration — 5-node Raft cluster
# Each node gets this config with node-specific values

# vault-node-1.hcl (us-east-1a)
storage "raft" {
  path    = "/vault/data"
  node_id = "vault-node-1"

  retry_join {
    leader_api_addr = "https://vault-node-2.vault-internal:8200"
  }
  retry_join {
    leader_api_addr = "https://vault-node-3.vault-internal:8200"
  }
  retry_join {
    leader_api_addr = "https://vault-node-4.vault-internal:8200"
  }
  retry_join {
    leader_api_addr = "https://vault-node-5.vault-internal:8200"
  }

  # Autopilot manages cluster health automatically
  autopilot {
    cleanup_dead_servers           = true
    dead_server_last_contact_limit = "10m"
    min_quorum                     = 3
    # server_stabilization_time: wait before promoting new voter
    # prevents flapping during network instability
    server_stabilization_time      = "30s"
  }
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/vault/tls/tls.crt"
  tls_key_file  = "/vault/tls/tls.key"
  
  # Telemetry for Prometheus scraping
  telemetry {
    unauthenticated_metrics_access = true
  }
}

# Enable Prometheus metrics
telemetry {
  prometheus_retention_time = "30s"
  disable_hostname          = true
}

seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "alias/novamart-vault-unseal"
  # Auto-unseal via KMS — nodes recover without manual intervention
  # CRITICAL: Without auto-unseal, a restarted node stays sealed
  # and you need an operator to manually unseal at 3 AM on Black Friday
}

api_addr     = "https://vault-node-1.vault-internal:8200"
cluster_addr = "https://vault-node-1.vault-internal:8201"
```

**Why 5 nodes matters:**
- 3-node Raft: quorum = 2, tolerates **1** failure. Lose 2 nodes (like our scenario) = quorum loss.
- 5-node Raft: quorum = 3, tolerates **2** failures. The exact scenario that caused this outage is now survivable.
- 7-node: tolerates 3 failures but adds latency on writes (more replication). Overkill for most deployments.

**AZ distribution for 5 nodes:**
```
us-east-1a: Node 1, Node 4   (2 nodes)
us-east-1b: Node 2, Node 5   (2 nodes)  
us-east-1c: Node 3            (1 node)

Any single AZ failure loses at most 2 nodes → quorum maintained (3/5 alive)
```

**Change 2: Kubernetes topology spread constraints to ENFORCE AZ distribution**

```yaml
# Vault StatefulSet with anti-affinity and topology spread
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: vault
  namespace: vault
spec:
  replicas: 5
  serviceName: vault-internal
  template:
    spec:
      topologySpreadConstraints:
        # HARD constraint: max 2 vault pods per AZ
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: vault
      affinity:
        podAntiAffinity:
          # HARD constraint: never 2 vault pods on same node
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: vault
              topologyKey: kubernetes.io/hostname
      containers:
        - name: vault
          image: hashicorp/vault:1.15.4
          resources:
            requests:
              memory: "8Gi"
              cpu: "2000m"
            limits:
              memory: "8Gi"    # No burst — Vault must never OOM
              cpu: "4000m"
          ports:
            - containerPort: 8200
              name: api
            - containerPort: 8201
              name: cluster
          readinessProbe:
            httpGet:
              path: /v1/sys/health
              port: 8200
              scheme: HTTPS
              # ?standbyok=true → standbys report ready (they serve reads)
              # ?sealedcode=503 → sealed nodes are NOT ready
            httpHeaders:
              - name: Host
                value: "vault.vault.svc.cluster.local"
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /v1/sys/health
              port: 8200
              scheme: HTTPS
              httpHeaders:
                - name: Host
                  value: "vault.vault.svc.cluster.local"
            initialDelaySeconds: 30
            periodSeconds: 10
            # Higher failure threshold — don't kill Vault during temporary issues
            failureThreshold: 5
      # Use EBS gp3 for Raft storage — consistent IOPS regardless of volume size
      volumeClaimTemplates:
        - metadata:
            name: data
          spec:
            accessModes: ["ReadWriteOnce"]
            storageClassName: gp3-encrypted
            resources:
              requests:
                storage: 50Gi
```

**Change 3: Automated Raft snapshots for disaster recovery**

```bash
#!/bin/bash
# /opt/vault/scripts/raft-snapshot.sh
# Runs as a CronJob every hour

set -euo pipefail

VAULT_ADDR="https://vault.vault.svc.cluster.local:8200"
SNAPSHOT_DIR="/tmp/vault-snapshots"
S3_BUCKET="novamart-vault-backups-${AWS_REGION}"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_FILE="${SNAPSHOT_DIR}/vault-raft-${TIMESTAMP}.snap"

mkdir -p "${SNAPSHOT_DIR}"

# Take the Raft snapshot
vault operator raft snapshot save "${SNAPSHOT_FILE}"

# Verify snapshot integrity
SNAPSHOT_SIZE=$(stat -c%s "${SNAPSHOT_FILE}")
if [ "${SNAPSHOT_SIZE}" -lt 1024 ]; then
  echo "ERROR: Snapshot too small (${SNAPSHOT_SIZE} bytes) — likely corrupt"
  exit 1
fi

# Upload to S3 with server-side encryption using Vault's KMS key
aws s3 cp "${SNAPSHOT_FILE}" \
  "s3://${S3_BUCKET}/raft-snapshots/vault-raft-${TIMESTAMP}.snap" \
  --sse aws:kms \
  --sse-kms-key-id alias/novamart-vault-backup \
  --region "${AWS_REGION}"

# Cross-region replication for DR
aws s3 cp "${SNAPSHOT_FILE}" \
  "s3://novamart-vault-backups-us-west-2/raft-snapshots/vault-raft-${TIMESTAMP}.snap" \
  --sse aws:kms \
  --sse-kms-key-id alias/novamart-vault-backup-dr \
  --region us-west-2

# Verify upload
aws s3api head-object \
  --bucket "${S3_BUCKET}" \
  --key "raft-snapshots/vault-raft-${TIMESTAMP}.snap" \
  --region "${AWS_REGION}" > /dev/null

echo "Snapshot uploaded successfully: ${SNAPSHOT_FILE} (${SNAPSHOT_SIZE} bytes)"

# Cleanup local snapshots older than 24 hours
find "${SNAPSHOT_DIR}" -name "vault-raft-*.snap" -mmin +1440 -delete

# Cleanup S3 snapshots older than 30 days (lifecycle policy also handles this)
# But belt-and-suspenders approach
```

```yaml
# CronJob for automated snapshots
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-raft-snapshot
  namespace: vault
spec:
  schedule: "0 * * * *"    # Every hour
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 3
      template:
        spec:
          serviceAccountName: vault-snapshot-sa  # IRSA role with S3 + KMS access
          containers:
            - name: snapshot
              image: hashicorp/vault:1.15.4
              command: ["/bin/bash", "/scripts/raft-snapshot.sh"]
              env:
                - name: VAULT_ADDR
                  value: "https://vault-active.vault.svc.cluster.local:8200"
                - name: VAULT_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: vault-snapshot-token
                      key: token
                - name: AWS_REGION
                  value: "us-east-1"
              volumeMounts:
                - name: scripts
                  mountPath: /scripts
                - name: tmp
                  mountPath: /tmp
          volumes:
            - name: scripts
              configMap:
                name: vault-snapshot-scripts
            - name: tmp
              emptyDir:
                sizeLimit: 5Gi
          restartPolicy: OnFailure
```

**Change 4: Graceful degradation — secret caching with extended grace periods**

```yaml
# Vault Agent sidecar configuration with aggressive caching
# This is the KEY change: pods survive Vault outages much longer

# vault-agent-config.hcl (injected via annotation or ConfigMap)
auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config = {
      role = "payment-svc"
    }
  }
  
  sink "file" {
    config = {
      path = "/home/vault/.vault-token"
    }
  }
}

cache {
  use_auto_auth_token = true
  # Persist the cache to disk — survives agent restarts
  persist = {
    type = "kubernetes"
    path = "/vault/agent-cache"
    keep_after_import = true
    exit_on_err = false
  }
}

# Template with explicit error handling
template {
  source      = "/vault/templates/db-credentials.tpl"
  destination = "/vault/secrets/db-credentials.json"

  # CRITICAL: These settings control behavior when Vault is unreachable
  error_on_missing_key = false

  # Wait before rendering — prevents thrashing during intermittent connectivity
  wait {
    min = "5s"
    max = "30s"
  }

  # If Vault is down and the lease expires, DON'T delete the file
  # Keep serving stale secrets — stale > absent
  sandbox_path = "/vault/secrets"
}

# Vault Agent won't exit if it can't reach Vault
exit_after_auth = false

# Long TTL tokens with generous renewal grace period
vault {
  address = "https://vault-active.vault.svc.cluster.local:8200"
  
  retry {
    num_retries = 12           # Retry for 12 * backoff ≈ 30+ minutes
    backoff     = "250ms"
    max_backoff = "5m"
  }
}
```

**Change 5: Longer credential TTLs for critical services on high-traffic days**

```bash
# Pre-Black Friday operational checklist:
# Increase dynamic credential TTL to survive longer outages

# Temporarily increase TTL for payment-svc DB role
vault write database/roles/payment-readwrite \
  db_name=payment-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT payment_app_role TO \"{{name}}\";" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="4h" \           # Normal: 1h. Black Friday: 4h
  max_ttl="8h"                 # Normal: 2h. Black Friday: 8h

# Force credential renewal on all running pods BEFORE Black Friday
# So they start with a fresh 4h TTL
kubectl rollout restart deployment/payment-svc -n payment

# VERIFY the new TTL:
kubectl exec -n payment deploy/payment-svc -c vault-agent -- \
  cat /vault/secrets/db-credentials.json | jq '.lease_duration'
# Should show 14400 (4 hours in seconds)
```

**Change 6: Monitoring and alerting for Vault health**

```yaml
# PrometheusRule for Vault cluster health
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: vault-cluster-alerts
  namespace: monitoring
spec:
  groups:
    - name: vault.raft.rules
      rules:
        # Alert if fewer than 5 peers in the Raft cluster
        - alert: VaultRaftPeersBelowTarget
          expr: vault_raft_peers < 5
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Vault Raft cluster has {{ $value }} peers (expected 5)"

        # CRITICAL: Alert if quorum is at risk (3 or fewer peers)
        - alert: VaultRaftQuorumAtRisk
          expr: vault_raft_peers <= 3
          for: 1m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Vault Raft quorum at risk: only {{ $value }} peers"
            runbook: "https://runbooks.novamart.internal/vault-quorum-risk"

        # Alert on leader changes (instability)
        - alert: VaultLeaderFlapping
          expr: increase(vault_raft_leader_lastContact_count[5m]) > 10
          for: 2m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Vault leader contact instability detected"

        # Alert if Vault Agent sidecars can't renew tokens
        - alert: VaultAgentTokenRenewalFailing
          expr: |
            increase(vault_agent_auth_failure[10m]) > 0
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Vault Agent auth failures — pods may lose secrets"

        # Alert on approaching credential expiry
        - alert: VaultDynamicCredentialExpiryApproaching
          expr: |
            (vault_secret_lease_remaining_seconds{service="payment-svc"} / 
             vault_secret_lease_total_seconds{service="payment-svc"}) < 0.25
          for: 5m
          labels:
            severity: warning
            team: payment
          annotations:
            summary: "payment-svc DB credentials at 75% of TTL — renewal may be failing"
```

**Summary decision tree for Vault outages:**

```
Vault unreachable?
├── Is quorum lost or just a single node?
│   ├── Single node: Raft self-heals. Monitor vault_raft_peers. No action.
│   └── Quorum lost:
│       ├── Can surviving nodes reach each other?
│       │   ├── YES: Wait for partitioned nodes to rejoin (auto-heal)
│       │   └── NO: Force single-node quorum on healthiest surviving node
│       │       vault operator raft remove-peer <dead-node-id>
│       │
│       ├── Are dynamic credential TTLs about to expire?
│       │   ├── >30 min remaining: Focus on Vault recovery
│       │   └── <30 min remaining: PARALLEL WORKSTREAMS:
│       │       ├── Stream 1: Emergency static secret bypass (Part 2)
│       │       └── Stream 2: Vault cluster recovery
│       │
│       └── Is this a peak traffic event (Black Friday)?
│           ├── YES: Block ALL deployments. Freeze ArgoCD syncs:
│           │   kubectl patch app --type merge -p '{"spec":{"syncPolicy":null}}' -n argocd --all
│           │   New pods can't start without Vault → prevent any pod churn
│           └── NO: Standard incident response, schedule recovery
│
└── Vault reachable but slow (high latency)?
    ├── Check: vault_core_handle_request latency
    ├── Common cause: too many lease renewals, storage backend IOPS
    └── Mitigation: increase Raft storage IOPS, check EBS gp3 baseline
```

---

## Q4: Complete Secret Compromise Response

### Part 1: First 10 Actions — Exact Order

**The order is: CONTAIN → PRESERVE → REVOKE → ASSESS. NOT revoke first.**

If you revoke first without containment, an attacker with a copy of the credential still has network access to the database. If you contain first, even if they have the credential, they can't reach the target.

```bash
# ACTION 1 (T+0 min): Restrict network access to the RDS instance
# WHY FIRST: Even if the attacker has the credential, they need NETWORK access
# to use it. Restricting security groups is instant and blocks exploitation.
# Run from: AWS CLI with production VPC admin access

# Get current security group for payment RDS
aws rds describe-db-instances \
  --db-instance-identifier payment-db-primary \
  --query 'DBInstances[0].VpcSecurityGroups' \
  --region us-east-1

# Restrict the security group to ONLY known application CIDR ranges
# Remove any 0.0.0.0/0 or overly broad rules
# SAVE THE CURRENT RULES FIRST for rollback:
aws ec2 describe-security-group-rules \
  --filters Name=group-id,Values=sg-payment-rds-xxxx \
  --region us-east-1 > /tmp/sg-payment-rds-backup.json

# If there's a broad ingress rule, revoke it:
aws ec2 revoke-security-group-ingress \
  --group-id sg-payment-rds-xxxx \
  --protocol tcp \
  --port 5432 \
  --cidr 10.0.0.0/8 \
  --region us-east-1

# Replace with tight CIDR: only the EKS pod CIDR and bastion subnet
aws ec2 authorize-security-group-ingress \
  --group-id sg-payment-rds-xxxx \
  --protocol tcp \
  --port 5432 \
  --cidr 10.42.0.0/16 \
  --region us-east-1
# (10.42.0.0/16 = EKS pod CIDR for payment namespace)
```

```bash
# ACTION 2 (T+2 min): Enable enhanced RDS audit logging if not already enabled
# WHY SECOND: You need to capture any ongoing malicious activity RIGHT NOW.
# If pgaudit isn't enabled, you're flying blind on what the attacker is doing.

aws rds modify-db-parameter-group \
  --db-parameter-group-name payment-db-params \
  --parameters "ParameterName=pgaudit.log,ParameterValue=all,ApplyMethod=immediate" \
  --region us-east-1

# Also enable log_connections and log_disconnections if not set:
aws rds modify-db-parameter-group \
  --db-parameter-group-name payment-db-params \
  --parameters \
    "ParameterName=log_connections,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=log_disconnections,ParameterValue=1,ApplyMethod=immediate" \
  --region us-east-1
```

```bash
# ACTION 3 (T+4 min): Preserve evidence — snapshot the RDS instance NOW
# WHY: Before any rotation/changes, capture the current state for forensics.
# If data was exfiltrated or modified, this snapshot is your evidence baseline.

aws rds create-db-snapshot \
  --db-instance-identifier payment-db-primary \
  --db-snapshot-identifier "payment-db-incident-$(date +%Y%m%d-%H%M%S)" \
  --region us-east-1

# Tag the snapshot as forensic evidence
aws rds add-tags-to-resource \
  --resource-name "arn:aws:rds:us-east-1:888888888888:snapshot:payment-db-incident-$(date +%Y%m%d-%H%M%S)" \
  --tags Key=Incident,Value=SECRET-LEAK-$(date +%Y%m%d) Key=DoNotDelete,Value=true \
  --region us-east-1
```

```bash
# ACTION 4 (T+6 min): Preserve the Git evidence — prevent force-push destruction
# WHY: The developer or someone might panic and force-push to remove the commit.
# That destroys your audit trail. Protect the branch NOW.

# Add branch protection rule to prevent force-push on the feature branch
# Using GitHub CLI (or GitLab equivalent):
gh api repos/novamart/payment-svc/branches/feature-xyz/protection \
  --method PUT \
  -f "required_status_checks=null" \
  -F "enforce_admins=true" \
  -F "required_pull_request_reviews=null" \
  -f "restrictions=null" \
  -F "allow_force_pushes=false" \
  -F "allow_deletions=false"

# Also: clone the repo locally to preserve the commit
git clone --mirror git@github.com:novamart/payment-svc.git /tmp/payment-svc-evidence-$(date +%s).git

# Save the specific commit SHA for the incident record
LEAKED_COMMIT=$(git -C /tmp/payment-svc-evidence-*.git log --all --oneline --diff-filter=A \
  --grep="payment/rds/master" | head -1 | awk '{print $1}')
echo "Leaked commit: ${LEAKED_COMMIT}" >> /tmp/incident-evidence-log.txt
```

```bash
# ACTION 5 (T+8 min): Rotate the compromised credential in Secrets Manager
# NOW we rotate — network is contained, evidence is preserved

# Trigger immediate rotation
aws secretsmanager rotate-secret \
  --secret-id payment/rds/master \
  --region us-east-1

# If rotation Lambda doesn't exist or takes too long — manual rotation:
NEW_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=')

# Update RDS directly
aws rds modify-db-instance \
  --db-instance-identifier payment-db-primary \
  --master-user-password "${NEW_PASSWORD}" \
  --apply-immediately \
  --region us-east-1

# NOTE: For RDS master user password change via modify-db-instance:
# - This does NOT require a restart for PostgreSQL (unlike MySQL which may)
# - Existing connections using the OLD password remain active until they disconnect
# - New connections must use the new password
# - There may be a brief period (seconds) during the change where connections fail

# Update Secrets Manager to match
aws secretsmanager update-secret \
  --secret-id payment/rds/master \
  --secret-string "{\"username\":\"payment_master\",\"password\":\"${NEW_PASSWORD}\",\"host\":\"payment-db.cluster-xxxx.us-east-1.rds.amazonaws.com\",\"port\":\"5432\",\"dbname\":\"payments\"}" \
  --region us-east-1

# Clear the variable — don't leave the password in shell history
unset NEW_PASSWORD
history -d $(history 1 | awk '{print $1}')
```

```bash
# ACTION 6 (T+12 min): Force-kill any active sessions using the OLD credentials
# The old master password may still have active connections

psql -h payment-db.cluster-xxxx.us-east-1.rds.amazonaws.com \
  -U payment_master -d payments -c "
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE usename = 'payment_master' 
  AND pid <> pg_backend_pid()
  AND backend_start < NOW() - INTERVAL '5 minutes';"
# Kills sessions older than 5 min (established before the password change)
```

```bash
# ACTION 7 (T+14 min): Propagate new credentials to running services
# Trigger ESO sync + pod restart (same procedure as Q1 Part 3)

kubectl annotate externalsecret payment-db-credentials \
  -n payment \
  force-sync=$(date +%s) --overwrite

# If Reloader is configured, pods will restart automatically
# Otherwise force it:
kubectl rollout restart deployment/payment-svc -n payment
kubectl rollout status deployment/payment-svc -n payment --timeout=180s

# Verify pods connect with new credentials:
kubectl logs -n payment -l app=payment-svc --tail=10 | grep -i "connected\|pool"
```

```bash
# ACTION 8 (T+18 min): Audit who had access to the leaked credential
# 85 contributors had read access to the repo. Enumerate them.

gh api repos/novamart/payment-svc/collaborators --paginate \
  --jq '.[].login' > /tmp/repo-collaborators.txt
wc -l /tmp/repo-collaborators.txt
# Also check team access:
gh api repos/novamart/payment-svc/teams --jq '.[].name'

# Check: did anyone clone/fork/download the repo in the last 4 hours?
# GitHub audit log (requires org admin):
gh api orgs/novamart/audit-log \
  --method GET \
  -f phrase="action:git.clone repo:novamart/payment-svc" \
  -f after="$(date -d '4 hours ago' -u +%Y-%m-%dT%H:%M:%SZ)" \
  --paginate
```

```bash
# ACTION 9 (T+20 min): Remove the secret from Git history
# BFG Repo-Cleaner is faster and safer than git-filter-branch

# On a clean machine (NOT the evidence copy):
git clone --mirror git@github.com:novamart/payment-svc.git /tmp/payment-svc-clean.git

# Create a file with the exact secret values to remove
echo "payment_master_password_value_here" > /tmp/passwords-to-remove.txt
echo "payment-db.cluster-xxxx.us-east-1.rds.amazonaws.com" >> /tmp/passwords-to-remove.txt

# Run BFG to replace secrets with ***REMOVED***
java -jar bfg-1.14.0.jar --replace-text /tmp/passwords-to-remove.txt \
  /tmp/payment-svc-clean.git

cd /tmp/payment-svc-clean.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Force-push the cleaned history
# WARNING: This rewrites all commit SHAs from the leaked commit forward
# All open PRs will need to be rebased
git push --force

# Securely delete the passwords file
shred -u /tmp/passwords-to-remove.txt
```

```bash
# ACTION 10 (T+25 min): Notify stakeholders and open formal incident ticket

# Security team notification
# Slack: #security-incidents
# Email: security@novamart.com, ciso@novamart.com
# Include:
# - Incident ID
# - What was exposed: RDS master credentials (username, password, host, port)
# - Exposure window: 4 hours (commit time to detection)
# - Scope: 85 potential viewers, private repo
# - Actions taken: network restricted, credential rotated, sessions killed, 
#   services restarted, git history cleaned
# - Forensic evidence preserved: RDS snapshot, git mirror copy
# - PCI implications: payment database is PCI-scoped — this is a potential
#   PCI DSS Requirement 8 violation (credential exposure)
# - Next steps: forensic analysis of 4-hour window (Part 2)

# If PCI QSA notification is required (depends on your PCI DSS SAQ/ROC level):
# Notify your QSA within the contractual window (typically 24-72 hours)
```

### Part 2: Forensic Analysis of the 4-Hour Window

```bash
# CLOUDTRAIL: Check for API calls using the compromised credential
# The master RDS credential is used via psql/JDBC, NOT via AWS API
# So CloudTrail won't show database queries. But check for:

# 1. Did anyone use Secrets Manager to READ this secret (besides normal automation)?
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue \
  --start-time "$(date -d '5 hours ago' -u +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --region us-east-1 \
  --query 'Events[?contains(CloudTrailEvent, `payment/rds/master`)]' \
  --output json > /tmp/secret-access-audit.json

# Parse: look for unusual source IPs, unknown IAM principals, or console access
cat /tmp/secret-access-audit.json | jq '.[] | {
  eventTime: (.CloudTrailEvent | fromjson | .eventTime),
  sourceIP: (.CloudTrailEvent | fromjson | .sourceIPAddress),
  userAgent: (.CloudTrailEvent | fromjson | .userAgent),
  principal: (.CloudTrailEvent | fromjson | .userIdentity.arn)
}'

# Red flags:
# - sourceIPAddress from outside known AWS/VPN ranges
# - userAgent showing manual CLI/console vs automation
# - Principal you don't recognize
```

```bash
# 2. RDS connection logs — check for connections from unexpected sources
# Download RDS PostgreSQL logs from the 4-hour window

for hour in 01 02 03 04 05; do
  aws rds download-db-log-file-portion \
    --db-instance-identifier payment-db-primary \
    --log-file-name "error/postgresql.log.$(date +%Y-%m-%d)-${hour}" \
    --starting-token 0 \
    --region us-east-1 \
    --output text >> /tmp/rds-logs-incident.txt
done

# Search for connections using the master user from unexpected IPs
grep "payment_master" /tmp/rds-logs-incident.txt | \
  grep "connection authorized" | \
  grep -v "10.42\." | \    # Exclude known EKS pod CIDR
  grep -v "10.0.1\."       # Exclude known bastion subnet

# If ANY connections come from IPs outside known ranges → confirmed compromise
```

```bash
# 3. RDS Performance Insights — check for unusual query patterns
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier "db-XXXXXXXXXXXXXXXXXXXX" \
  --metric-queries '[
    {
      "Metric": "db.sql.queries",
      "GroupBy": {
        "Group": "db.user",
        "Dimensions": ["db.user.name"],
        "Limit": 10
      }
    }
  ]' \
  --start-time "$(date -d '5 hours ago' -u +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period-in-seconds 300 \
  --region us-east-1

# Look for:
# - Unusually high query counts from payment_master
# - Queries during off-hours (if the 4-hour window includes off-hours)
# - SELECT * patterns (data exfiltration)
# - pg_dump or COPY TO patterns (bulk data export)
```

```bash
# 4. VPC Flow Logs — check for data exfiltration from the RDS instance
aws logs filter-log-events \
  --log-group-name "novamart-vpc-flow-logs" \
  --start-time $(date -d '5 hours ago' +%s)000 \
  --filter-pattern "{ ($.dstPort = 5432) && ($.srcAddr != \"10.42.*\") }" \
  --region us-east-1

# Also check for large outbound data transfers FROM the RDS subnet
aws logs filter-log-events \
  --log-group-name "novamart-vpc-flow-logs" \
  --start-time $(date -d '5 hours ago' +%s)000 \
  --filter-pattern "{ ($.srcAddr = \"10.0.100.*\") && ($.bytes > 1000000) }" \
  --region us-east-1
# 10.0.100.* = RDS subnet. >1MB transfers = suspicious for DB exfiltration

# 5. If pgaudit was enabled: check for privilege escalation attempts
grep -E "GRANT|ALTER ROLE|CREATE ROLE|COPY|pg_dump" /tmp/rds-logs-incident.txt
```

### Part 3: Rotate Now vs Wait — Who's Right?

**The CISO is right. Rotate now.** Here's the exact reasoning:

```
┌─────────────────────────────────────────────────────────────────┐
│                 DECISION ANALYSIS                                │
│                                                                 │
│  Option A: Rotate now (2 PM, peak traffic)                      │
│  ├── Risk: Brief connection interruption during password change  │
│  ├── Duration: 10-30 seconds for RDS master password change     │
│  │   (PostgreSQL applies immediately, no restart required)       │
│  ├── Impact: Some payment requests may fail during transition    │
│  ├── Cost: ~$8K-$25K (30 sec of degraded service × $50K/min)   │
│  └── Mitigation: Application retry logic + circuit breaker      │
│                                                                 │
│  Option B: Wait until midnight                                   │
│  ├── Risk: 10 MORE HOURS of attacker access to production       │
│  │   payment database containing PCI cardholder data             │
│  ├── Impact if exploited during those 10 hours:                 │
│  │   ├── Complete payment DB exfiltration                       │
│  │   ├── PCI DSS breach notification (all cardholders)          │
│  │   ├── PCI compliance revocation                              │
│  │   ├── Card brand fines ($50K-$500K per brand)                │
│  │   ├── Customer lawsuits                                      │
│  │   ├── Brand damage (unquantifiable)                          │
│  │   └── Regulatory penalties (GDPR if EU customers: 4% revenue)│
│  └── The VP is optimizing for uptime. The CISO is optimizing    │
│      for security. For a PCI-scoped credential leak,            │
│      SECURITY WINS. PCI DSS Requirement 8.2.4 requires          │
│      immediate credential change when compromise is suspected.  │
│                                                                 │
│  VERDICT: Rotate NOW. The cost of a 30-second blip is           │
│  negligible compared to 10 hours of exposure.                   │
│                                                                 │
│  IMPORTANT NUANCE about the "brief RDS restart" premise:        │
│  For PostgreSQL, master password changes via modify-db-instance  │
│  do NOT require a restart. The premise of the question assumed   │
│  a restart, but for PostgreSQL on RDS, apply-immediately works  │
│  without downtime. For MySQL/Aurora MySQL, a restart MAY be      │
│  needed — in that case, do a failover to the read replica:      │
│  aws rds failover-db-cluster --db-cluster-identifier payment-db │
│  Password change happens on the old primary, failover switches  │
│  traffic to the replica (seconds of downtime, not minutes).     │
└─────────────────────────────────────────────────────────────────┘
```

**The compromise:** If the VP won't budge, execute the rotation with a maintenance page for the 30-second window. But frame it correctly: "We're choosing 30 seconds of planned degradation over 10 hours of data breach risk on a PCI system. PCI DSS requires us to act."

### Part 4: Systemic Prevention — Layered Defense

**Layer 1: Pre-commit hooks (Developer Workstation)**

Catches secrets before they enter Git history. This is the cheapest defense — prevents the incident entirely.

```yaml
# .pre-commit-config.yaml (committed to every repo)
repos:
  # Gitleaks — fast, low false-positive secret scanner
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.1
    hooks:
      - id: gitleaks
        name: gitleaks
        description: Detect hardcoded secrets
        entry: gitleaks protect --staged --verbose --redact
        language: golang
        pass_filenames: false

  # Also detect high-entropy strings and AWS patterns
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
        # Baseline file: tracks known false positives so devs don't get alert fatigue
```

```toml
# .gitleaks.toml — custom rules for NovaMart patterns
title = "NovaMart Gitleaks Configuration"

[extend]
useDefault = true   # Include all default rules (AWS keys, generic tokens, etc.)

# Custom rule: detect Secrets Manager JSON blobs
[[rules]]
id = "novamart-secrets-manager-json"
description = "Secrets Manager JSON blob (contains host+password+username)"
regex = '''\"password\"\s*:\s*\"[^\"]{8,}\".*\"host\"\s*:\s*\"[^\"]*\.rds\.amazonaws\.com\"'''
tags = ["secret", "rds", "pci"]
severity = "CRITICAL"

# Custom rule: detect RDS connection strings
[[rules]]
id = "novamart-rds-connection-string"
description = "RDS connection string with embedded password"
regex = '''postgresql://[^:]+:[^@]+@[^/]*\.rds\.amazonaws\.com'''
tags = ["secret", "rds"]
severity = "CRITICAL"

# Allowlist: paths that are expected to contain secret-like patterns
[allowlist]
paths = [
  '''\.secrets\.baseline$''',
  '''test/fixtures/.*\.json$''',
  '''docs/examples/.*'''
]
```

```bash
# Bootstrap script for new developer onboarding
#!/bin/bash
# setup-pre-commit.sh — run once per developer machine

set -euo pipefail

# Install pre-commit framework
pip install pre-commit

# Install gitleaks
brew install gitleaks  # macOS
# or: apt-get install gitleaks  # Linux

# Install hooks for this repo
pre-commit install

# Run against all existing files to verify clean baseline
pre-commit run --all-files

# Generate initial secrets baseline (for detect-secrets)
detect-secrets scan --all-files > .secrets.baseline
echo "Review .secrets.baseline and commit it"

echo "✅ Pre-commit hooks installed. Secrets will be blocked before commit."
```

**What this catches that other layers miss:** Prevents the secret from ever entering Git. No other layer can undo a `git push` — BFG cleanup is expensive, and the credential is still cached in GitHub's backend for up to 90 days.

**What this DOESN'T catch:** Developer bypasses `--no-verify`, new machines without hooks installed, non-standard Git clients.

**Layer 2: CI Pipeline Scanning (Server-Side)**

Catches everything pre-commit misses — developers who skip hooks, new repos without hooks, and commits through the GitHub web UI.

```yaml
# .github/workflows/secret-scan.yml
name: Secret Scanning
on:
  push:
    branches: ['**']
  pull_request:
    branches: [main, develop]

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # Full history — scan all commits in PR

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
        with:
          args: detect --source . --verbose --redact --log-opts="${{ github.event.pull_request.base.sha }}..${{ github.sha }}"

      # If gitleaks finds a secret — auto-notify security team
      - name: Alert Security Team on Failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'C0SECURITY'
          payload: |
            {
              "text": "🚨 SECRET DETECTED in ${{ github.repository }} by ${{ github.actor }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "🚨 *Secret detected in commit*\n*Repo:* ${{ github.repository }}\n*Author:* ${{ github.actor }}\n*Branch:* ${{ github.ref_name }}\n*Commit:* ${{ github.sha }}\n*Action:* Pipeline blocked. Security team notified."
                  }
                }
              ]
            }
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_SECURITY_BOT_TOKEN }}

      # Block the PR — cannot merge with secrets present
      - name: Fail Check
        if: failure()
        run: |
          echo "::error::SECRET DETECTED — this PR cannot be merged. Contact #security for assistance."
          exit 1
```

**What this catches that pre-commit misses:** Developers who `--no-verify`, web UI commits, automated tools that push directly.

**What this DOESN'T catch:** Secrets already in Git history from before scanning was enabled. Requires baseline scan.

**Layer 3: GitHub Native Secret Scanning + Push Protection**

```bash
# Enable GitHub Advanced Security features (org-level)
# This catches secrets PUSHED to GitHub, even before CI runs
# Also scans existing history

gh api orgs/novamart/settings -X PATCH \
  -f "advanced_security_enabled_for_new_repositories=true" \
  -f "secret_scanning_enabled_for_new_repositories=true" \
  -f "secret_scanning_push_protection_enabled_for_new_repositories=true"

# Enable on existing repos
gh api repos/novamart/payment-svc -X PATCH \
  -F "security_and_analysis[secret_scanning][status]=enabled" \
  -F "security_and_analysis[secret_scanning_push_protection][status]=enabled"

# Push protection BLOCKS the push at GitHub's server before it's even stored
# This is the only defense against pre-commit bypass + CI race condition
```

**Layer 4: Secret Architecture — Eliminate Secrets in Code Entirely**

The root cause isn't "we didn't scan" — it's "the developer had the production secret on their machine in the first place." Fix the architecture:

```
┌─────────────────────────────────────────────────────────────┐
│       SECRET ARCHITECTURE — ZERO DEVELOPER ACCESS            │
│                                                              │
│  BEFORE (broken):                                            │
│  Developer → aws secretsmanager get-secret-value → clipboard │
│  → paste into config file → commit → push → INCIDENT         │
│                                                              │
│  AFTER (correct):                                            │
│  Developer: NEVER has production secrets                      │
│  ├── Dev environment: uses LOCAL secrets (different DB)       │
│  ├── Staging: ESO injects staging secrets (different creds)  │
│  └── Prod: ESO injects prod secrets (no human access)        │
│                                                              │
│  How to enforce:                                             │
│  1. IAM policy: Deny GetSecretValue for production secrets   │
│     from developer roles                                     │
│  2. Separate AWS accounts: devs can't even reach prod acct   │
│  3. Secrets Manager resource policy: restrict to IRSA roles  │
│  4. SCP: Deny secrets access from non-production accounts    │
└─────────────────────────────────────────────────────────────┘
```

```json
// IAM policy: block developer access to production secrets
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyProdSecretAccess",
      "Effect": "Deny",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:GetResourcePolicy",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:888888888888:secret:payment/*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::888888888888:role/payment-svc-irsa-role",
            "arn:aws:iam::888888888888:role/secrets-rotation-lambda-role",
            "arn:aws:iam::888888888888:role/external-secrets-operator-role",
            "arn:aws:iam::888888888888:role/BreakGlassSecurityRole"
          ]
        }
      }
    }
  ]
}
```

```hcl
# Secrets Manager resource policy — defense in depth
resource "aws_secretsmanager_secret_policy" "payment_db" {
  secret_arn = aws_secretsmanager_secret.payment_db.arn

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "RestrictToServiceRoles"
        Effect    = "Deny"
        Principal = "*"
        Action    = "secretsmanager:GetSecretValue"
        Resource  = "*"
        Condition = {
          StringNotEquals = {
            "aws:PrincipalArn" = [
              "arn:aws:iam::888888888888:role/payment-svc-irsa-role",
              "arn:aws:iam::888888888888:role/secrets-rotation-lambda-role",
              "arn:aws:iam::888888888888:role/external-secrets-operator-role"
            ]
          }
        }
      }
    ]
  })
}
```

**Layer 5: Developer Education — Specific, Not Generic**

```markdown
## NovaMart Security Training — Module: Secret Handling

### What you MUST know:
1. **NEVER** run `aws secretsmanager get-secret-value` for production secrets
   - You don't need prod secrets for development. Use dev secrets.
   - If you think you need a prod secret, you're solving the wrong problem.
   - Ask in #platform-help — we'll find the right approach.

2. **If you accidentally see/copy a production secret:**
   - Do NOT paste it anywhere (file, Slack, email, Jira ticket)
   - Clear your clipboard: `pbcopy < /dev/null` (macOS) or `xclip -i /dev/null` (Linux)
   - Report it in #security-incidents — no blame, we need to know

3. **Local development uses docker-compose with local DB:**
   ```bash
   # Local development — uses LOCAL PostgreSQL, no production credentials
   docker-compose up -d
   # Connection: postgresql://payment_dev:localdevpassword@localhost:5432/payments_dev
   ```

4. **Pre-commit hooks are mandatory. Verify they're installed:**
   ```bash
   pre-commit run --all-files
   # If this command doesn't work, run: make setup-dev
   ```

### Quarterly exercise:
- Security team pushes a test secret to a branch
- Measure: How fast does pre-commit catch it? CI catch it? GitGuardian catch it?
- Review results in team retrospective
```

**Prevention layers summary:**

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DEFENSE IN DEPTH — SECRET LEAK PREVENTION         │
│                                                                      │
│  Layer             │ Tool                │ Catches What Others Miss   │
│  ──────────────────┼─────────────────────┼────────────────────────── │
│  1. Architecture   │ Account isolation   │ Devs can't GET the secret │
│                    │ + resource policies │ in the first place         │
│  2. Pre-commit     │ Gitleaks + detect-  │ Blocks before commit      │
│                    │ secrets             │ (fastest feedback)         │
│  3. Push protection│ GitHub native       │ Blocks at git push even   │
│                    │                     │ if pre-commit is skipped   │
│  4. CI scan        │ Gitleaks in GHA     │ Catches web UI commits,   │
│                    │                     │ automated pushes           │
│  5. Post-push      │ GitGuardian /       │ Catches secrets in ALL    │
│     monitoring     │ GitHub secret scan  │ existing history, forks,  │
│                    │                     │ and mirrors                │
│  6. Runtime        │ Secrets Manager     │ Auto-rotation means even  │
│     rotation       │ rotation + ESO      │ leaked creds expire       │
│                    │                     │ before exploitation        │
│  7. Network        │ Security groups +   │ Even with valid creds,    │
│     containment    │ PrivateLink         │ attacker can't reach DB   │
│                    │                     │ from outside VPC           │
└──────────────────────────────────────────────────────────────────────┘

Layer 1 prevents 95% of incidents. Layers 2-5 are for the remaining 5%.
Layers 6-7 limit blast radius when all prevention fails.
```

---

## Summary: Key Principles From This Lesson

```
┌──────────────────────────────────────────────────────────────────┐
│  1. SECRET DELIVERY IS A CHAIN — every link must be monitored    │
│     Secrets Manager → ESO → K8s Secret → Pod Volume → App Pool  │
│     Failure at ANY link = outage. Test the full chain.           │
│                                                                  │
│  2. KMS ROOT PRINCIPAL IS NON-NEGOTIABLE                         │
│     Without it, the key is permanently unmanageable.             │
│     Only AWS Support can recover. Put it in every key policy.    │
│     CI check: Checkov custom rule to enforce this.               │
│                                                                  │
│  3. VAULT OUTAGE ≠ JUST VAULT PROBLEM                           │
│     It blocks HPA scaling, pod restarts, deployments, AND        │
│     cert renewal. Graceful degradation (caching, static          │
│     fallback) is mandatory for critical-path services.           │
│                                                                  │
│  4. INCIDENT RESPONSE ORDER: CONTAIN → PRESERVE → REVOKE        │
│     Revoking first without containment leaves the attack         │
│     surface open. Preserving evidence before changes enables     │
│     forensics. Get the order right.                              │
│                                                                  │
│  5. ARCHITECTURE > SCANNING                                      │
│     If developers can't access production secrets, they can't    │
│     leak them. Account isolation + resource policies + IRSA      │
│     eliminate the class of incident, not just detect it.          │
└──────────────────────────────────────────────────────────────────┘
```

