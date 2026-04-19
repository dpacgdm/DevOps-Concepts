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

