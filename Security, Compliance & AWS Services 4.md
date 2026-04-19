# Phase 6, Lesson 5: Compliance & Governance

---

## Why This Lesson Matters at NovaMart

```
NovaMart processes credit cards, stores personal data, and operates
at a scale where regulatory failure isn't a fine — it's an existential threat.

  PCI-DSS violation:  Lose ability to process credit cards → business ends
  GDPR violation:     Up to 4% of global revenue ($80M for NovaMart)
  SOC 2 failure:      Enterprise customers won't buy → revenue loss
  HIPAA (if applicable): $1.5M per violation category per year
  Data breach:        Average cost $4.45M + reputation + customer churn

Compliance is NOT a checkbox exercise performed once a year.
It's a CONTINUOUS ENGINEERING DISCIPLINE baked into infrastructure,
CI/CD, monitoring, and incident response.

This lesson covers:
  1. COMPLIANCE FRAMEWORKS — PCI-DSS, SOC 2, GDPR, CIS Benchmarks
  2. GOVERNANCE ARCHITECTURE — How NovaMart enforces compliance at scale
  3. AUDIT READINESS — Evidence collection, continuous compliance, audit prep
  4. ORGANIZATIONAL GOVERNANCE — Policies, change management, access reviews
```

```
THE DEVOPS ENGINEER'S ROLE IN COMPLIANCE:

  You are NOT a compliance officer.
  You ARE the person who IMPLEMENTS and AUTOMATES compliance.

  Compliance officer says: "We need encryption at rest for all data stores."
  You implement: KMS encryption on RDS, S3, EBS, etcd, ElastiCache.
  You automate: AWS Config rule that detects unencrypted resources.
  You prove: Terraform code, Config compliance dashboard, CloudTrail logs.

  Compliance officer says: "We need access reviews quarterly."
  You implement: IAM Access Analyzer, unused credential reports.
  You automate: Lambda that generates access review reports.
  You prove: S3-stored reports with timestamps, Jira tickets for remediation.

  The auditor doesn't care about your Terraform code.
  The auditor cares about EVIDENCE that controls exist and WORK.
  Your job: make evidence generation automatic and continuous.
```

---

## Part 1: Compliance Frameworks — What They Require and How NovaMart Meets Them

### PCI-DSS (Payment Card Industry Data Security Standard)

```
PCI-DSS applies because NovaMart processes credit card payments.
Version: PCI-DSS v4.0 (effective March 2024, mandatory March 2025)

PCI-DSS has 12 REQUIREMENTS organized in 6 GOALS:

┌─────────────────────────────────────────────────────────────────────┐
│                    PCI-DSS v4.0 REQUIREMENTS                         │
│                                                                      │
│  GOAL 1: BUILD AND MAINTAIN A SECURE NETWORK                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Req 1: Install and maintain network security controls        │    │
│  │   NovaMart: Security Groups, NACLs, NetworkPolicies,         │    │
│  │   Istio mTLS, WAF, Cloudflare                                │    │
│  │                                                              │    │
│  │ Req 2: Apply secure configurations to all system components  │    │
│  │   NovaMart: CIS-hardened AMIs, Pod Security Standards,       │    │
│  │   Gatekeeper policies, Ansible hardening playbooks           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  GOAL 2: PROTECT ACCOUNT DATA                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Req 3: Protect stored account data                           │    │
│  │   NovaMart: KMS encryption (RDS, S3, EBS, etcd),            │    │
│  │   Vault Transit for application-level encryption,            │    │
│  │   data retention policies, tokenization for PAN storage      │    │
│  │                                                              │    │
│  │ Req 4: Protect cardholder data with strong cryptography      │    │
│  │        during transmission over open, public networks        │    │
│  │   NovaMart: TLS 1.2+ everywhere, Istio mTLS (internal),     │    │
│  │   ACM certificates, cert-manager, HSTS headers              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  GOAL 3: MAINTAIN A VULNERABILITY MANAGEMENT PROGRAM                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Req 5: Protect all systems and networks from malicious       │    │
│  │        software                                              │    │
│  │   NovaMart: Trivy image scanning, Falco runtime detection,   │    │
│  │   GuardDuty malware protection, read-only rootfs             │    │
│  │                                                              │    │
│  │ Req 6: Develop and maintain secure systems and software      │    │
│  │   NovaMart: SonarQube (code quality), Snyk (dependencies),   │    │
│  │   secure SDLC, code review requirements, change management   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  GOAL 4: IMPLEMENT STRONG ACCESS CONTROL MEASURES                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Req 7: Restrict access to system components and cardholder   │    │
│  │        data by business need-to-know                         │    │
│  │   NovaMart: IAM least privilege, RBAC, IRSA, Vault policies, │    │
│  │   namespace isolation, NetworkPolicies                       │    │
│  │                                                              │    │
│  │ Req 8: Identify users and authenticate access to system      │    │
│  │        components                                            │    │
│  │   NovaMart: SSO (OIDC), MFA everywhere, unique accounts,    │    │
│  │   no shared credentials, service account management          │    │
│  │                                                              │    │
│  │ Req 9: Restrict physical access to cardholder data           │    │
│  │   NovaMart: AWS responsibility (data center security).       │    │
│  │   NovaMart responsibility: laptop encryption, office access  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  GOAL 5: REGULARLY MONITOR AND TEST NETWORKS                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Req 10: Log and monitor all access to system components      │    │
│  │         and cardholder data                                  │    │
│  │   NovaMart: CloudTrail, VPC Flow Logs, K8s audit logs,       │    │
│  │   Vault audit logs, application logs (Loki), Falco           │    │
│  │                                                              │    │
│  │ Req 11: Test security of systems and networks regularly      │    │
│  │   NovaMart: Penetration testing (annual + after changes),    │    │
│  │   vulnerability scanning (continuous), ASV scans (quarterly),│    │
│  │   internal network scans, wireless scanning                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  GOAL 6: MAINTAIN AN INFORMATION SECURITY POLICY                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Req 12: Support information security with organizational     │    │
│  │         policies and programs                                │    │
│  │   NovaMart: Security policy documents, risk assessments,     │    │
│  │   security awareness training, incident response plan,       │    │
│  │   vendor management program                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

```
PCI-DSS SCOPING — THE CRITICAL CONCEPT:

  PCI-DSS applies to the CDE (Cardholder Data Environment):
    - Systems that store, process, or transmit cardholder data
    - Systems connected to those systems
    - Systems that could affect security of the CDE

  NovaMart's CDE:
    ┌──────────────────────────────────────────────────────┐
    │  IN SCOPE (CDE):                                      │
    │  • payment-svc (processes card data)                  │
    │  • fraud-svc (analyzes transactions)                  │
    │  • RDS payment database                               │
    │  • Stripe integration (card processing)               │
    │  • EKS nodes running payment/fraud pods               │
    │  • Network segments connecting these components       │
    │  • ALB/WAF handling payment traffic                   │
    │  • Vault (stores payment credentials)                 │
    │                                                       │
    │  CONNECTED TO CDE (also in scope):                    │
    │  • order-svc (sends payment requests)                 │
    │  • api-gateway (routes to payment-svc)                │
    │  • Monitoring stack (accesses CDE metrics/logs)       │
    │  • CI/CD pipeline (deploys to CDE)                    │
    │  • EKS control plane                                  │
    │                                                       │
    │  OUT OF SCOPE:                                        │
    │  • search-svc (no cardholder data)                    │
    │  • notification-svc (no cardholder data — sends       │
    │    confirmation emails but no card numbers)           │
    │  • cart-svc (stores items, not card data)             │
    │  • Static assets (S3/CloudFront)                      │
    └──────────────────────────────────────────────────────┘

  SCOPE REDUCTION STRATEGY:
    1. TOKENIZATION: Replace real card numbers with tokens
       (Stripe handles this — card data never touches NovaMart)
    2. NETWORK SEGMENTATION: Isolate CDE with NetworkPolicies,
       Security Groups, separate namespaces, Istio AuthzPolicy
    3. MINIMIZE DATA: Don't store what you don't need.
       NovaMart stores Stripe payment tokens, NOT card numbers.
       This dramatically reduces PCI scope.

  If Stripe handles all card data and NovaMart only stores tokens:
    NovaMart is PCI-DSS SAQ A-EP (not full SAQ D)
    This means fewer requirements, simpler audits.
    BUT: payment-svc still sends data TO Stripe → in scope for transit.
```

### SOC 2 (Service Organization Control 2)

```
SOC 2 applies because NovaMart's enterprise customers require it.
"Show me your SOC 2 report" is the first security question from
any enterprise B2B customer or partner.

SOC 2 is based on 5 TRUST SERVICE CRITERIA (TSC):

┌─────────────────────────────────────────────────────────────────────┐
│                    SOC 2 TRUST SERVICE CRITERIA                       │
│                                                                      │
│  1. SECURITY (Required — always included)                           │
│     "The system is protected against unauthorized access"            │
│     Controls: Access control, encryption, monitoring, incident       │
│     response, vulnerability management, change management            │
│     NovaMart: IAM, RBAC, KMS, WAF, GuardDuty, Falco, etc.         │
│                                                                      │
│  2. AVAILABILITY (Included for NovaMart)                            │
│     "The system is available for operation as committed"             │
│     Controls: SLOs, monitoring, incident response, DR, backups,     │
│     capacity planning, redundancy                                    │
│     NovaMart: Multi-AZ, auto-scaling, SLOs, PagerDuty, runbooks    │
│                                                                      │
│  3. PROCESSING INTEGRITY (Included for NovaMart)                    │
│     "System processing is complete, valid, accurate, timely"         │
│     Controls: Input validation, error handling, reconciliation,      │
│     data processing monitoring                                       │
│     NovaMart: SLIs for correctness, order reconciliation jobs,      │
│     payment verification, data pipeline monitoring                   │
│                                                                      │
│  4. CONFIDENTIALITY (Included for NovaMart)                         │
│     "Information designated as confidential is protected"            │
│     Controls: Encryption, access control, data classification,       │
│     secure disposal                                                  │
│     NovaMart: KMS, Vault, network segmentation, data retention      │
│                                                                      │
│  5. PRIVACY (Included — NovaMart stores PII)                        │
│     "Personal information is collected, used, retained, disclosed    │
│      in conformity with commitments and criteria"                    │
│     Controls: Privacy policy, consent management, data subject       │
│     rights, data minimization                                        │
│     NovaMart: GDPR compliance, data subject access requests,        │
│     anonymization pipelines, consent management                      │
└─────────────────────────────────────────────────────────────────────┘

SOC 2 REPORT TYPES:
  Type I:  Controls exist at a point in time (snapshot)
           "On January 15, these controls were in place"
           Useful for first audit, faster to complete

  Type II: Controls operated EFFECTIVELY over a period (usually 12 months)
           "From Jan 1 to Dec 31, these controls were consistently effective"
           Required by most enterprise customers
           This is what NovaMart maintains annually

SOC 2 vs PCI-DSS:
  PCI-DSS: Prescriptive ("You MUST encrypt with AES-256")
  SOC 2:   Principles-based ("You must protect data. Show us HOW.")
  
  SOC 2 is more flexible but also more subjective.
  The auditor evaluates whether YOUR controls adequately address
  the trust criteria, not whether you checked specific boxes.
```

### GDPR (General Data Protection Regulation)

```
GDPR applies because NovaMart has EU customers (eu-west-1 region).
Even if NovaMart were US-only, processing EU resident data = GDPR applies.

KEY GDPR PRINCIPLES FOR DEVOPS:

  1. DATA MINIMIZATION
     Collect only what you need. Don't log PII unnecessarily.
     NovaMart: Log scrubbing pipelines, anonymization in non-prod,
     structured logging with PII fields identified and maskable.

  2. PURPOSE LIMITATION
     Data collected for one purpose can't be used for another.
     NovaMart: Data classification labels, access controls per purpose.

  3. STORAGE LIMITATION
     Don't keep data longer than necessary.
     NovaMart: Data retention policies enforced by automation:
       - CloudTrail logs: 2 years (compliance)
       - Application logs: 90 days (operational)
       - PII in databases: per data retention schedule
       - Backups: 30-day retention, then purge

  4. RIGHT TO ERASURE (Right to be forgotten)
     Users can request deletion of their personal data.
     NovaMart: Automated deletion pipeline:
       User request → Jira ticket → Lambda → cascading delete across:
       - User database (PostgreSQL)
       - Order history (anonymize, don't delete — financial records)
       - Search history (MongoDB — delete)
       - Notification logs (delete)
       - Backups (flag for exclusion in next restore)
     
     THE BACKUP PROBLEM: If user requests deletion but their data
     is in a 30-day-old backup that gets restored... the data comes back.
     Solution: Maintain a "deletion ledger" — on every restore, re-apply
     all deletions from the ledger. This is operationally complex.

  5. DATA PORTABILITY
     Users can request their data in a machine-readable format.
     NovaMart: Export API that generates JSON/CSV of user data.

  6. BREACH NOTIFICATION
     72 hours to notify supervisory authority after discovering breach.
     NovaMart: Incident response plan includes legal/DPO notification
     within 24 hours (giving 48 hours for regulatory filing).

  7. DATA PROTECTION BY DESIGN
     Security and privacy must be built in, not bolted on.
     NovaMart: This entire training program. Encryption, access control,
     monitoring, etc. are architectural decisions, not afterthoughts.
```

```
GDPR TECHNICAL IMPLEMENTATIONS:

DATA CLASSIFICATION:
  ┌──────────────────────────────────────────────────────────────┐
  │ Classification │ Examples                │ Controls           │
  ├────────────────┼─────────────────────────┼────────────────────┤
  │ PUBLIC         │ Product descriptions,   │ No special         │
  │                │ pricing                 │ controls           │
  │ INTERNAL       │ Internal docs, metrics  │ Auth required      │
  │ CONFIDENTIAL   │ Business data, orders   │ Encryption, RBAC   │
  │ RESTRICTED     │ PII, payment data,      │ KMS, Vault, audit  │
  │                │ health data             │ logging, DLP       │
  └────────────────┴─────────────────────────┴────────────────────┘

PII HANDLING IN LOGS:
  # Promtail pipeline to mask PII before ingestion to Loki
  pipelineStages:
    - replace:
        expression: '(\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b)'
        replace: '****-****-****-REDACTED'
    - replace:
        expression: '(\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,7}\b)'
        replace: 'REDACTED_EMAIL'
    - replace:
        expression: '(\b\d{3}[-.]?\d{2}[-.]?\d{4}\b)'
        replace: 'REDACTED_SSN'

  # IMPORTANT: Mask in the pipeline, NOT in the application.
  # Why: Applications are developed by many teams. If one team
  # forgets to mask, PII leaks into Loki (searchable, retained).
  # Centralized pipeline masking catches everything.

DATA RESIDENCY:
  EU customer data MUST stay in EU regions.
  NovaMart: eu-west-1 (Ireland) for EU customer data.
  
  Technical enforcement:
  - S3 bucket policy with aws:RequestedRegion condition
  - SCP blocking data service creation outside approved regions
  - RDS read replicas: only to EU regions
  - Separate EKS cluster in eu-west-1 for EU customer workloads
```

### CIS Benchmarks — Infrastructure Hardening

```
CIS (Center for Internet Security) publishes hardening benchmarks
for specific technologies. These are PRESCRIPTIVE checklists.

NovaMart-relevant CIS Benchmarks:
  - CIS AWS Foundations Benchmark v3.0
  - CIS Kubernetes Benchmark v1.8
  - CIS Amazon EKS Benchmark v1.4
  - CIS Docker Benchmark v1.6
  - CIS Amazon Linux 2 Benchmark

These are AUTOMATED via:
  Security Hub:  CIS AWS Foundations (automated checks)
  kube-bench:    CIS Kubernetes/EKS Benchmark
  Docker Bench:  CIS Docker Benchmark
  Inspector:     CIS host-level benchmarks
```

```bash
# kube-bench — automated CIS Kubernetes benchmark scanning
# Run as a Job in EKS to check cluster configuration

kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
  namespace: security
spec:
  template:
    spec:
      hostPID: true  # Needs host PID namespace to check kubelet config
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:v0.7.1
          command: ["kube-bench", "run", "--targets", "node,policies,managedservices"]
          # For EKS: use "node,policies,managedservices"
          # EKS manages the control plane — you can't check it
          volumeMounts:
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
            - name: etc-systemd
              mountPath: /etc/systemd
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: var-lib-kubelet
          hostPath:
            path: /var/lib/kubelet
        - name: etc-systemd
          hostPath:
            path: /etc/systemd
        - name: etc-kubernetes
          hostPath:
            path: /etc/kubernetes
  backoffLimit: 0
EOF

# Check results:
kubectl -n security logs job/kube-bench

# kube-bench output example:
# [PASS] 4.1.1 Ensure that the kubelet service file permissions are set to 600
# [FAIL] 4.1.5 Ensure that the --read-only-port argument is set to 0
# [WARN] 4.2.6 Ensure that the --protect-kernel-defaults argument is set to true
# 
# == Summary ==
# 15 checks PASS
# 3 checks FAIL
# 5 checks WARN
# 2 checks INFO
```

```yaml
# Run kube-bench as a CronJob for continuous compliance
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kube-bench-scheduled
  namespace: security
spec:
  schedule: "0 6 * * 1"  # Every Monday at 6 AM
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          hostPID: true
          containers:
            - name: kube-bench
              image: aquasec/kube-bench:v0.7.1
              command:
                - /bin/sh
                - -c
                - |
                  kube-bench run \
                    --targets node,policies,managedservices \
                    --json > /tmp/results.json
                  
                  # Count failures
                  FAILURES=$(cat /tmp/results.json | jq '[.Controls[].tests[].results[] | select(.status=="FAIL")] | length')
                  
                  # Send to monitoring
                  if [ "$FAILURES" -gt "0" ]; then
                    curl -X POST "$SLACK_WEBHOOK" \
                      -H 'Content-Type: application/json' \
                      -d "{\"text\": \"⚠️ kube-bench: $FAILURES CIS benchmark failures detected. Review results in S3.\"}"
                  fi
                  
                  # Archive results to S3
                  aws s3 cp /tmp/results.json \
                    "s3://novamart-compliance/kube-bench/$(date +%Y-%m-%d).json"
              env:
                - name: SLACK_WEBHOOK
                  valueFrom:
                    secretKeyRef:
                      name: slack-webhooks
                      key: security-channel
          restartPolicy: Never
          serviceAccountName: kube-bench-sa  # IRSA with S3 write access
```

---

## Part 2: Governance Architecture — Enforcing Compliance at Scale

### The Compliance-as-Code Pyramid

```
┌─────────────────────────────────────────────────────────────────────┐
│            COMPLIANCE-AS-CODE PYRAMID                                 │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  LAYER 1: PREVENTIVE CONTROLS (Stop bad things)             │    │
│  │                                                              │    │
│  │  SCPs → Account-level guardrails (can't be bypassed)        │    │
│  │  Gatekeeper/Kyverno → K8s admission policies                │    │
│  │  IAM Policies → Least privilege                              │    │
│  │  AWS Config Rules → Resource compliance                      │    │
│  │  Terraform Sentinel/OPA → IaC policy gates                  │    │
│  │  Branch protection → Code change governance                  │    │
│  │  NetworkPolicies → Network segmentation                      │    │
│  │                                                              │    │
│  │  BEST: Violations are IMPOSSIBLE. Can't create non-compliant │    │
│  │        resources because the platform won't allow it.        │    │
│  └──────────────────────────────┬──────────────────────────────┘    │
│                                 │                                    │
│  ┌──────────────────────────────▼──────────────────────────────┐    │
│  │  LAYER 2: DETECTIVE CONTROLS (Find bad things)               │    │
│  │                                                              │    │
│  │  Security Hub → Compliance scoring                           │    │
│  │  AWS Config → Continuous resource evaluation                 │    │
│  │  GuardDuty → Threat detection                                │    │
│  │  Falco → Runtime anomalies                                   │    │
│  │  CloudTrail → API audit trail                                │    │
│  │  kube-bench → CIS benchmark compliance                       │    │
│  │  Trivy → Vulnerability detection                             │    │
│  │  IAM Access Analyzer → Unintended access paths               │    │
│  │                                                              │    │
│  │  SECOND BEST: Violations are DETECTED quickly and            │    │
│  │  automatically. Alert → investigate → remediate.             │    │
│  └──────────────────────────────┬──────────────────────────────┘    │
│                                 │                                    │
│  ┌──────────────────────────────▼──────────────────────────────┐    │
│  │  LAYER 3: CORRECTIVE CONTROLS (Fix bad things)               │    │
│  │                                                              │    │
│  │  Config auto-remediation → Fix non-compliant resources       │    │
│  │  GuardDuty auto-response → Quarantine compromised instances  │    │
│  │  Falco auto-response → Kill compromised containers           │    │
│  │  Lambda functions → Auto-fix security groups, S3 policies    │    │
│  │  Incident response playbooks → Human-driven remediation      │    │
│  │                                                              │    │
│  │  FALLBACK: When prevention and detection aren't enough,      │    │
│  │  fix automatically or with clear procedures.                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  PRINCIPLE: Maximize Layer 1. Layer 2 catches what Layer 1 misses.   │
│  Layer 3 is for when Layer 2 finds something.                        │
│  If you're relying on Layer 3 daily, Layer 1 is insufficient.        │
└─────────────────────────────────────────────────────────────────────┘
```

### AWS Organizations — Multi-Account Governance

```
NovaMart uses AWS Organizations for account-level isolation:

┌─────────────────────────────────────────────────────────────────────┐
│                  AWS ORGANIZATIONS STRUCTURE                          │
│                                                                      │
│  Root                                                                │
│  ├── Management Account (billing, Organizations admin ONLY)          │
│  │   └── NO workloads here. Ever.                                   │
│  │                                                                   │
│  ├── OU: Security                                                    │
│  │   ├── security-audit (CloudTrail aggregation, Security Hub admin) │
│  │   ├── security-logging (centralized log archive)                  │
│  │   └── security-tooling (GuardDuty admin, Inspector)              │
│  │                                                                   │
│  ├── OU: Infrastructure                                              │
│  │   ├── network-hub (Transit Gateway, Direct Connect, DNS)          │
│  │   └── shared-services (ECR, Artifactory, monitoring)             │
│  │                                                                   │
│  ├── OU: Workloads                                                   │
│  │   ├── OU: Production                                              │
│  │   │   ├── novamart-prod-us-east-1                                │
│  │   │   ├── novamart-prod-us-west-2                                │
│  │   │   └── novamart-prod-eu-west-1                                │
│  │   ├── OU: Staging                                                 │
│  │   │   └── novamart-staging                                       │
│  │   └── OU: Development                                             │
│  │       └── novamart-dev                                           │
│  │                                                                   │
│  └── OU: Sandbox                                                     │
│      └── novamart-sandbox (experimentation, limited budget)          │
│                                                                      │
│  WHY MULTI-ACCOUNT?                                                  │
│  1. Blast radius: Compromise in dev can't reach production           │
│  2. Billing: Clear cost allocation per environment                   │
│  3. Limits: AWS service limits are per-account                       │
│  4. Compliance: CDE (PCI) isolation in dedicated account             │
│  5. IAM: Separate IAM boundaries per account                        │
└─────────────────────────────────────────────────────────────────────┘
```

### Service Control Policies — Organization-Level Guardrails

```
SCPs are the MOST POWERFUL preventive control in AWS.
They set MAXIMUM permissions for an entire account or OU.
Even the account root user cannot exceed SCP boundaries.

SCPs don't grant permissions — they set the CEILING.
IAM policies grant permissions UP TO the SCP ceiling.

  Effective permissions = IAM policy ∩ SCP ∩ Resource policy
  (Intersection of all three)
```

```hcl
# SCP: Deny actions in non-approved regions
resource "aws_organizations_policy" "region_restriction" {
  name        = "restrict-regions"
  description = "Deny all actions outside approved regions"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyNonApprovedRegions"
        Effect    = "Deny"
        NotAction = [
          # Global services that MUST work from any region
          "a4b:*", "budgets:*", "ce:*", "chime:*",
          "cloudfront:*", "cur:*", "globalaccelerator:*",
          "health:*", "iam:*", "importexport:*",
          "kms:*", "mobileanalytics:*", "organizations:*",
          "pricing:*", "route53:*", "route53domains:*",
          "route53-recovery-readiness:*", "s3:GetBucketLocation",
          "s3:ListAllMyBuckets", "shield:*", "sts:*",
          "support:*", "trustedadvisor:*", "waf-regional:*",
          "waf:*", "wafv2:*", "wellarchitected:*"
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" = [
              "us-east-1",
              "us-west-2",
              "eu-west-1"
            ]
          }
        }
      }
    ]
  })
}

resource "aws_organizations_policy_attachment" "region_restriction" {
  policy_id = aws_organizations_policy.region_restriction.id
  target_id = aws_organizations_organizational_unit.workloads.id
}

# SCP: Prevent disabling security services
resource "aws_organizations_policy" "protect_security" {
  name    = "protect-security-services"
  type    = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyDisableGuardDuty"
        Effect = "Deny"
        Action = [
          "guardduty:DeleteDetector",
          "guardduty:DisassociateFromMasterAccount",
          "guardduty:UpdateDetector"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDisableCloudTrail"
        Effect = "Deny"
        Action = [
          "cloudtrail:DeleteTrail",
          "cloudtrail:StopLogging",
          "cloudtrail:UpdateTrail"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDisableConfig"
        Effect = "Deny"
        Action = [
          "config:DeleteConfigurationRecorder",
          "config:StopConfigurationRecorder",
          "config:DeleteDeliveryChannel"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDisableSecurityHub"
        Effect = "Deny"
        Action = [
          "securityhub:DisableSecurityHub",
          "securityhub:DeleteMembers",
          "securityhub:DisassociateFromMasterAccount"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyLeaveOrganization"
        Effect = "Deny"
        Action = [
          "organizations:LeaveOrganization"
        ]
        Resource = "*"
      }
    ]
  })
}

# SCP: Prevent privilege escalation
resource "aws_organizations_policy" "prevent_privilege_escalation" {
  name    = "prevent-privilege-escalation"
  type    = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyAdminPolicyAttachment"
        Effect = "Deny"
        Action = [
          "iam:AttachRolePolicy",
          "iam:AttachUserPolicy",
          "iam:AttachGroupPolicy"
        ]
        Resource = "*"
        Condition = {
          ArnEquals = {
            "iam:PolicyArn" = [
              "arn:aws:iam::aws:policy/AdministratorAccess",
              "arn:aws:iam::aws:policy/IAMFullAccess",
              "arn:aws:iam::aws:policy/PowerUserAccess"
            ]
          }
        }
      },
      {
        Sid    = "DenyCreateAdminUser"
        Effect = "Deny"
        Action = [
          "iam:CreateUser",
          "iam:CreateAccessKey"
        ]
        Resource = "*"
        Condition = {
          StringNotLike = {
            "aws:PrincipalArn" = [
              "arn:aws:iam::*:role/OrganizationAdmin",
              "arn:aws:iam::*:role/SecurityAdmin"
            ]
          }
        }
      }
    ]
  })
}

# SCP: Deny unencrypted resource creation
resource "aws_organizations_policy" "require_encryption" {
  name    = "require-encryption"
  type    = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyUnencryptedS3Upload"
        Effect = "Deny"
        Action = ["s3:PutObject"]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = ["aws:kms", "AES256"]
          }
          Null = {
            "s3:x-amz-server-side-encryption" = "true"
          }
        }
      },
      {
        Sid    = "DenyUnencryptedEBS"
        Effect = "Deny"
        Action = ["ec2:CreateVolume"]
        Resource = "*"
        Condition = {
          Bool = {
            "ec2:Encrypted" = "false"
          }
        }
      },
      {
        Sid    = "DenyUnencryptedRDS"
        Effect = "Deny"
        Action = ["rds:CreateDBInstance", "rds:CreateDBCluster"]
        Resource = "*"
        Condition = {
          Bool = {
            "rds:StorageEncrypted" = "false"
          }
        }
      }
    ]
  })
}
```

```
SCP TESTING — CRITICAL:

  SCPs can LOCK YOU OUT of your own account if misconfigured.

  ALWAYS test SCPs:
  1. Apply to sandbox OU first (not production)
  2. Verify with test IAM role:
     aws sts assume-role --role-arn <test-role-in-sandbox>
     # Then try actions that should be denied
  3. Only after validation → apply to staging OU → production OU
  4. Keep an "emergency" OU with no SCPs attached
     - If an SCP breaks something critical, move the account
       to the emergency OU to temporarily remove all SCPs
  5. NEVER test SCPs in the management account
     (SCPs don't apply to the management account anyway)
```

### IAM Access Analyzer — Continuous Access Review

```
IAM Access Analyzer continuously monitors for unintended access:

TYPE 1: EXTERNAL ACCESS ANALYZER
  Finds resources shared with external entities (outside your org):
  - S3 buckets with public or cross-account access
  - IAM roles with trust policies allowing external principals
  - KMS keys shared with other accounts
  - Lambda functions with public resource policies
  - SQS queues with external access

TYPE 2: UNUSED ACCESS ANALYZER (newer, very useful)
  Finds unused permissions within your own account:
  - IAM roles not used in 90 days
  - Access keys not used in 90 days
  - Permissions granted but never used (over-provisioned roles)
  - Service-linked roles with excessive permissions
```

```hcl
# IAM Access Analyzer configuration

# External access analyzer
resource "aws_accessanalyzer_analyzer" "external" {
  analyzer_name = "novamart-external-access"
  type          = "ORGANIZATION"  # Organization-wide analysis
  # ACCOUNT for single account, ORGANIZATION for all accounts

  tags = {
    Environment = "production"
  }
}

# Unused access analyzer
resource "aws_accessanalyzer_analyzer" "unused" {
  analyzer_name = "novamart-unused-access"
  type          = "ORGANIZATION_UNUSED_ACCESS"

  configuration {
    unused_access {
      unused_access_age = 90  # Flag access unused for 90+ days
    }
  }
}

# Archive rule — suppress known-good findings
resource "aws_accessanalyzer_archive_rule" "cross_account_ecr" {
  analyzer_name = aws_accessanalyzer_analyzer.external.analyzer_name
  rule_name     = "suppress-ecr-cross-region"

  filter {
    criteria = "resourceType"
    eq       = ["AWS::ECR::Repository"]
  }
  filter {
    criteria = "principal.AWS"
    contains = [data.aws_caller_identity.current.account_id]
    # Suppress findings for cross-region ECR replication (same account)
  }
}

# Alert on new findings
resource "aws_cloudwatch_event_rule" "access_analyzer_findings" {
  name = "access-analyzer-new-finding"

  event_pattern = jsonencode({
    source      = ["aws.access-analyzer"]
    detail-type = ["Access Analyzer Finding"]
    detail = {
      status = ["ACTIVE"]
    }
  })
}

resource "aws_cloudwatch_event_target" "access_analyzer_sns" {
  rule      = aws_cloudwatch_event_rule.access_analyzer_findings.name
  target_id = "notify-security"
  arn       = aws_sns_topic.security_alerts.arn
}
```

### Automated Access Reviews

```python
# Lambda: Quarterly access review report generator
# Triggered by EventBridge schedule: cron(0 9 1 */3 * *)  (First day of each quarter)

import boto3
import json
from datetime import datetime, timedelta

iam = boto3.client('iam')
analyzer = boto3.client('accessanalyzer')
s3 = boto3.client('s3')
ses = boto3.client('ses')

def handler(event, context):
    report = {
        'generated_at': datetime.utcnow().isoformat(),
        'report_type': 'quarterly_access_review',
        'sections': {}
    }

    # Section 1: Unused IAM users (no activity in 90 days)
    unused_users = []
    credential_report = iam.generate_credential_report()
    # Wait for report... (simplified)
    response = iam.get_credential_report()
    # Parse CSV response for last_activity dates
    for user_record in parse_credential_report(response['Content']):
        if user_record['password_last_used'] == 'N/A' and \
           user_record['access_key_1_last_used'] == 'N/A':
            unused_users.append(user_record['user'])
        elif days_since(user_record['password_last_used']) > 90:
            unused_users.append(user_record['user'])
    report['sections']['unused_users'] = unused_users

    # Section 2: Unused access keys
    unused_keys = []
    paginator = iam.get_paginator('list_users')
    for page in paginator.paginate():
        for user in page['Users']:
            keys = iam.list_access_keys(UserName=user['UserName'])
            for key in keys['AccessKeyMetadata']:
                last_used = iam.get_access_key_last_used(
                    AccessKeyId=key['AccessKeyId']
                )
                if 'LastUsedDate' not in last_used['AccessKeyLastUsed'] or \
                   (datetime.utcnow() - last_used['AccessKeyLastUsed']['LastUsedDate'].replace(tzinfo=None)).days > 90:
                    unused_keys.append({
                        'user': user['UserName'],
                        'key_id': key['AccessKeyId'],
                        'created': key['CreateDate'].isoformat(),
                        'status': key['Status']
                    })
    report['sections']['unused_access_keys'] = unused_keys

    # Section 3: Over-provisioned roles (from Access Analyzer)
    unused_access_findings = []
    paginator = analyzer.get_paginator('list_findings_v2')
    for page in paginator.paginate(
        analyzerArn='arn:aws:access-analyzer:us-east-1:888888888888:analyzer/novamart-unused-access',
        filter={'status': {'eq': ['ACTIVE']}}
    ):
        for finding in page['findings']:
            unused_access_findings.append({
                'resource': finding['resource'],
                'resource_type': finding['resourceType'],
                'finding_type': finding['findingType'],
                'details': finding.get('findingDetails', {})
            })
    report['sections']['unused_access_findings'] = unused_access_findings

    # Section 4: External access findings
    external_findings = []
    paginator = analyzer.get_paginator('list_findings_v2')
    for page in paginator.paginate(
        analyzerArn='arn:aws:access-analyzer:us-east-1:888888888888:analyzer/novamart-external-access',
        filter={'status': {'eq': ['ACTIVE']}}
    ):
        for finding in page['findings']:
            external_findings.append({
                'resource': finding['resource'],
                'resource_type': finding['resourceType'],
                'principal': finding.get('principal', {}),
                'action': finding.get('action', [])
            })
    report['sections']['external_access_findings'] = external_findings

    # Section 5: Summary and action items
    report['sections']['summary'] = {
        'unused_users_count': len(unused_users),
        'unused_keys_count': len(unused_keys),
        'unused_access_findings_count': len(unused_access_findings),
        'external_access_findings_count': len(external_findings),
        'action_required': (
            len(unused_users) > 0 or
            len(unused_keys) > 0 or
            len(external_findings) > 0
        )
    }

    # Store report in S3
    report_key = f"access-reviews/{datetime.utcnow().strftime('%Y-Q%q')}/report.json"
    s3.put_object(
        Bucket='novamart-compliance',
        Key=report_key,
        Body=json.dumps(report, indent=2, default=str),
        ServerSideEncryption='aws:kms'
    )

    # Create Jira tickets for action items
    if report['sections']['summary']['action_required']:
        # Integration with Jira API to create review tickets
        create_jira_tickets(report)

    # Email report to security team
    ses.send_email(
        Source='compliance@novamart.com',
        Destination={'ToAddresses': ['security-team@novamart.com']},
        Message={
            'Subject': {'Data': f"Quarterly Access Review - {datetime.utcnow().strftime('%Y Q%q')}"},
            'Body': {'Text': {'Data': generate_summary_text(report)}}
        }
    )

    return report['sections']['summary']
```

---

## Part 3: Audit Readiness — Evidence Collection and Continuous Compliance

### The Evidence Problem

```
AUDITORS NEED EVIDENCE. Not promises. Not architecture diagrams.
EVIDENCE that controls existed and OPERATED EFFECTIVELY over the audit period.

TYPES OF EVIDENCE:
  1. Configuration evidence — "Show me encryption is enabled on RDS"
  2. Operational evidence — "Show me it was enabled for the entire year"
  3. Process evidence — "Show me your change management process"
  4. Testing evidence — "Show me your penetration test results"
  5. Incident evidence — "Show me your incident response and postmortems"

THE MANUAL EVIDENCE TRAP:
  If your evidence collection requires an engineer to:
    - Log into AWS console and take screenshots
    - Run CLI commands and save output to files
    - Write a document explaining each control

  Then you will:
    - Spend 2-4 weeks on audit prep (engineering time wasted)
    - Miss evidence for controls that changed mid-year
    - Have gaps that the auditor will flag

THE AUTOMATED EVIDENCE GOAL:
  Evidence is CONTINUOUSLY GENERATED and STORED as artifacts.
  When the auditor asks for evidence, you query a database.
  Audit prep takes hours, not weeks.
```

### NovaMart's Continuous Compliance Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│         NOVAMART CONTINUOUS COMPLIANCE ARCHITECTURE                    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  EVIDENCE SOURCES                                              │ │
│  │                                                                │ │
│  │  AWS Config          → Resource configuration over time         │ │
│  │  Security Hub        → Compliance scores over time             │ │
│  │  CloudTrail          → API audit trail                         │ │
│  │  Gatekeeper audits   → K8s policy compliance                   │ │
│  │  kube-bench          → CIS benchmark results                   │ │
│  │  Trivy scans         → Vulnerability scan results              │ │
│  │  Terraform plans     → Infrastructure change evidence          │ │
│  │  ArgoCD sync history → Deployment change evidence              │ │
│  │  PagerDuty           → Incident history and response times     │ │
│  │  Postmortems         → Incident analysis documentation         │ │
│  │  Access reviews      → Quarterly IAM review results            │ │
│  │  Pen test reports    → Annual penetration test results         │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                          │                                           │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  EVIDENCE STORE (S3 — immutable, encrypted, versioned)         │ │
│  │                                                                │ │
│  │  s3://novamart-compliance/                                     │ │
│  │  ├── config-snapshots/          (AWS Config, 6-hourly)         │ │
│  │  ├── security-hub/              (compliance scores, daily)     │ │
│  │  ├── cloudtrail/                (API logs, continuous)         │ │
│  │  ├── kube-bench/                (CIS results, weekly)          │ │
│  │  ├── trivy-scans/               (vuln scans, per build)       │ │
│  │  ├── access-reviews/            (IAM reviews, quarterly)      │ │
│  │  ├── pen-tests/                 (results, annual)             │ │
│  │  ├── incident-postmortems/      (after each incident)         │ │
│  │  ├── change-records/            (Terraform plans + approvals) │ │
│  │  └── training-records/          (security training completion)│ │
│  │                                                                │ │
│  │  S3 features:                                                  │ │
│  │  • Object Lock (COMPLIANCE mode — immutable for audit period)  │ │
│  │  • Versioning (track changes)                                  │ │
│  │  • KMS encryption (data protection)                            │ │
│  │  • Access logging (who accessed evidence)                      │ │
│  │  • Lifecycle (archive after audit period, retain per policy)   │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                          │                                           │
│                          ▼                                           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  COMPLIANCE DASHBOARD (Grafana)                                │ │
│  │                                                                │ │
│  │  Panel 1: Security Hub compliance scores by standard           │ │
│  │  Panel 2: Open findings by severity and age                    │ │
│  │  Panel 3: Config rule compliance % over time                   │ │
│  │  Panel 4: Mean time to remediate (MTTR) for findings           │ │
│  │  Panel 5: Gatekeeper violations trend                          │ │
│  │  Panel 6: Access review completion status                      │ │
│  │  Panel 7: Certificate expiry timeline                          │ │
│  │  Panel 8: Encryption coverage (% of resources encrypted)       │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Mapping Controls to Evidence — The Control Matrix

```
THE CONTROL MATRIX maps each compliance requirement to:
  1. The control that addresses it
  2. The tool that implements the control
  3. The evidence that proves the control works
  4. The frequency of evidence collection

┌────────────────────┬─────────────────────┬────────────────────┬──────────────────────┬───────────┐
│ Requirement        │ Control             │ Implementation     │ Evidence             │ Frequency │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ PCI Req 1:         │ Network             │ Security Groups,   │ Config rules,        │ Continuous│
│ Network controls   │ segmentation        │ NACLs, NetworkPol  │ VPC Flow Logs        │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ PCI Req 3:         │ Encryption at       │ KMS (RDS, S3,      │ Config rule:         │ Continuous│
│ Protect stored     │ rest                │ EBS, etcd)         │ encrypted-volumes,   │           │
│ data               │                     │                    │ s3-bucket-encryption │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ PCI Req 4:         │ Encryption in       │ TLS 1.2+, Istio   │ ALB access logs,     │ Continuous│
│ Protect data in    │ transit             │ mTLS, ACM certs    │ cert-manager status, │           │
│ transit            │                     │                    │ TLS scan results     │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ PCI Req 6:         │ Secure SDLC,        │ SonarQube, Trivy,  │ Jenkins build logs,  │ Per build │
│ Secure software    │ vulnerability       │ code review,       │ scan reports,        │           │
│                    │ scanning            │ branch protection  │ PR history           │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ PCI Req 7:         │ Least privilege     │ IAM policies,      │ Access Analyzer,     │ Quarterly │
│ Need-to-know       │ access control      │ RBAC, IRSA,        │ access review        │           │
│ access             │                     │ Vault policies     │ reports              │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ PCI Req 8:         │ Strong authN,       │ SSO + MFA,         │ CloudTrail login     │ Continuous│
│ Identify + authN   │ unique accounts     │ no shared creds,   │ events, IAM          │           │
│                    │                     │ OIDC federation    │ credential report    │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ PCI Req 10:        │ Comprehensive       │ CloudTrail, K8s    │ CloudTrail logs,     │ Continuous│
│ Logging            │ logging             │ audit, VPC Flow,   │ log integrity        │           │
│                    │                     │ Vault audit        │ validation, Loki     │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ PCI Req 11:        │ Regular testing     │ Pen tests, ASV     │ Pen test report,     │ Annual/   │
│ Test security      │                     │ scans, vuln scans  │ scan results (S3)    │ Quarterly │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ SOC 2 Avail:       │ SLOs, monitoring,   │ Prometheus, SLIs,  │ SLO dashboards,      │ Continuous│
│ Availability       │ incident response   │ PagerDuty,         │ incident reports,    │           │
│                    │                     │ runbooks           │ postmortems          │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ SOC 2 Change:      │ Change management   │ GitOps (ArgoCD),   │ Git history, PR      │ Per change│
│ Change controls    │ process             │ PR reviews,        │ reviews, ArgoCD      │           │
│                    │                     │ Terraform plans    │ sync history         │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ GDPR Art 17:       │ Right to erasure    │ Deletion pipeline  │ Deletion request     │ Per       │
│ Right to be        │                     │ (Lambda +          │ logs, completion      │ request   │
│ forgotten          │                     │  cascading delete) │ confirmations        │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ GDPR Art 32:       │ Appropriate         │ Encryption, access │ Config rules,        │ Continuous│
│ Security of        │ security measures   │ control, monitoring│ Security Hub score   │           │
│ processing         │                     │                    │                      │           │
└────────────────────┴─────────────────────┴────────────────────┴──────────────────────┴───────────┘
```

---

## Part 4: Change Management — Governance of Changes

### Why Change Management Matters

```
MOST OUTAGES ARE CAUSED BY CHANGES.

  Google SRE data: ~70% of outages are caused by changes
  to a running system (deploys, config changes, infra changes).

  Compliance frameworks (PCI, SOC 2) require:
    - Documented change management process
    - Separation of duties (developer ≠ deployer to production)
    - Approval before production changes
    - Rollback capability
    - Change audit trail

  NovaMart's GitOps model provides this naturally:
    Developer → PR (code change) → Reviewer (approval) →
    Merge → ArgoCD (automated deploy) → Production
    
    Every step is logged, auditable, and reversible.
```

### NovaMart Change Management Process

```
┌─────────────────────────────────────────────────────────────────────┐
│              NOVAMART CHANGE MANAGEMENT PROCESS                      │
│                                                                      │
│  CHANGE TYPES:                                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ Standard Change (pre-approved, low risk):                      │ │
│  │   • Application deployment via CI/CD (passing all gates)       │ │
│  │   • Scaling adjustments (HPA threshold changes)                │ │
│  │   • Dependency updates (non-security, non-major version)       │ │
│  │   Process: PR → 1 reviewer → merge → auto-deploy               │ │
│  │   Approval: Peer review only                                   │ │
│  │                                                                │ │
│  │ Normal Change (moderate risk):                                 │ │
│  │   • New service deployment                                     │ │
│  │   • Infrastructure changes (Terraform)                         │ │
│  │   • Database schema migrations                                 │ │
│  │   • Gatekeeper policy changes                                  │ │
│  │   • Security group modifications                               │ │
│  │   Process: PR → 2 reviewers → CAB approval → merge → deploy    │ │
│  │   Approval: Peer review + team lead                            │ │
│  │                                                                │ │
│  │ Emergency Change (incident response):                          │ │
│  │   • Hotfix for active incident                                 │ │
│  │   • Security patch for critical CVE                            │ │
│  │   • Configuration rollback                                     │ │
│  │   Process: PR → 1 reviewer → merge → deploy → RETROACTIVE CAB │ │
│  │   Approval: On-call lead (retroactive review within 48 hours)  │ │
│  │                                                                │ │
│  │ Major Change (high risk, high impact):                         │ │
│  │   • Kubernetes version upgrade                                 │ │
│  │   • Database migration (engine change, major version)          │ │
│  │   • Network architecture changes (VPC, Transit Gateway)        │ │
│  │   • New compliance-affecting service                           │ │
│  │   Process: RFC → Architecture review → CAB → scheduled window  │ │
│  │   Approval: Architecture board + VP Engineering                │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  CHANGE ADVISORY BOARD (CAB):                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ Meets: Weekly (Tuesday 10 AM) + on-demand for emergencies     │ │
│  │ Members: Platform lead, Security lead, SRE lead, QA lead      │ │
│  │ Reviews: All Normal and Major changes for the week             │ │
│  │ Outputs: Approved, Deferred, Rejected (with reasons)           │ │
│  │ Records: Meeting notes in Confluence, Jira ticket updates      │ │
│  │                                                                │ │
│  │ CAB does NOT slow down standard deployments.                   │ │
│  │ CI/CD pipeline handles standard changes automatically.         │ │
│  │ CAB reviews infrastructure, policy, and high-risk changes.     │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  CHANGE RECORD (Jira ticket template):                              │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ • Description: What is being changed and why                   │ │
│  │ • Risk assessment: Impact if change fails                      │ │
│  │ • Rollback plan: Exact steps to undo the change                │ │
│  │ • Testing evidence: What was tested and results                │ │
│  │ • Dependencies: Other changes or teams affected                │ │
│  │ • Schedule: When will the change be applied                    │ │
│  │ • Verification: How to confirm the change succeeded            │ │
│  │ • Approvals: PR link, reviewer names, CAB decision             │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  EVIDENCE TRAIL (automatic via GitOps):                             │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ Git commit → WHO made the change (author + committer)          │ │
│  │ PR review  → WHO approved (reviewer signatures)                │ │
│  │ CI logs    → WHAT was tested (scan results, test results)      │ │
│  │ ArgoCD     → WHEN it was deployed (sync timestamp)             │ │
│  │ CloudTrail → HOW infrastructure changed (API calls)            │ │
│  │ Prometheus → WHETHER it worked (SLI before/after)              │ │
│  │                                                                │ │
│  │ An auditor can trace ANY production change from:               │ │
│  │   Jira ticket → PR → CI build → ArgoCD sync → running pod      │ │
│  │ Every link is timestamped, immutable, and attributable.        │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Separation of Duties — The Compliance Requirement

```
SEPARATION OF DUTIES means no single person can:
  - Write code AND deploy it to production (without review)
  - Create an IAM policy AND attach it (without approval)
  - Modify a security control AND disable the monitoring of that control

NovaMart implements this through:

  CODE CHANGES:
    Developer writes code → different person reviews PR →
    CI pipeline deploys (automated, no human in the loop for deploy)
    
    Enforced by: Branch protection rules
      - Require at least 1 (standard) or 2 (infrastructure) approvals
      - Cannot approve your own PR
      - Dismiss stale approvals when new commits are pushed
      - Require status checks to pass (CI must succeed)

  INFRASTRUCTURE CHANGES:
    Engineer writes Terraform → different engineer reviews →
    Atlantis/CI runs plan → reviewer approves plan output →
    Atlantis applies (or CI applies on merge)
    
    Enforced by: Atlantis require_approval: true
    
    ADDITIONAL: For PCI-scoped infrastructure:
      Two reviewers required (one must be from security or platform team)

  IAM CHANGES:
    All IAM changes go through Terraform (no console access).
    IAM Terraform modules require security team review.
    SCP prevents IAM changes outside of approved roles.

  EMERGENCY EXCEPTION:
    On-call engineer can deploy hotfix with single approval.
    MUST file retroactive change record within 48 hours.
    Retroactive CAB review at next meeting.
    If the hotfix bypassed controls, document WHY and assess risk.
```

```hcl
# Bitbucket branch protection (NovaMart uses Bitbucket)
# Equivalent configuration as code:

# For application repos:
# Settings → Branch permissions → main branch
# - Require at least 1 approval
# - No self-approval
# - Require passing CI builds
# - No direct pushes (force push disabled)
# - No deletions

# For infrastructure repos (Terraform):
# - Require at least 2 approvals
# - One must be from @platform-team or @security-team
# - Require passing plan + policy checks
# - No direct pushes

# Terraform via Atlantis — enforces plan-then-apply with approval
# atlantis.yaml (repo-level config)
version: 3
projects:
  - name: production
    dir: environments/production
    workspace: default
    terraform_version: v1.7.0
    autoplan:
      when_modified: ["*.tf", "*.tfvars", "modules/**/*.tf"]
      enabled: true
    apply_requirements:
      - approved    # PR must be approved
      - mergeable   # PR must be mergeable (CI passing)
    workflow: production

workflows:
  production:
    plan:
      steps:
        - init
        - plan
        - run: conftest test $PLANFILE --policy ../policies/ -o json
        # OPA policy check on the Terraform plan
    apply:
      steps:
        - apply
```

### Change Freeze / Deployment Windows

```
CHANGE FREEZE: Periods when non-emergency changes are blocked.

NovaMart's change freezes:
  1. Black Friday → Cyber Monday (Wed before Thanksgiving → Tuesday after)
     Duration: ~6 days
     Exception: Critical security patches, revenue-impacting hotfixes
     
  2. End of quarter (last 3 business days)
     Duration: 3 days
     Reason: Financial reconciliation period
     
  3. Major holiday weekends
     Duration: Varies
     Reason: Reduced on-call capacity

IMPLEMENTATION: ArgoCD sync windows
```

```yaml
# ArgoCD sync window — prevent deploys during change freeze
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  syncWindows:
    # Allow syncs during business hours only
    - kind: allow
      schedule: "0 8 * * 1-5"  # Mon-Fri 8 AM
      duration: 12h              # Until 8 PM
      
    # Block ALL syncs during Black Friday freeze
    - kind: deny
      schedule: "0 0 20 11 *"   # Nov 20 (approximate)
      duration: 168h             # 7 days
      manualSync: true           # Even manual syncs blocked
      
    # Exception: Allow security namespace during freeze
    - kind: allow
      schedule: "* * * * *"     # Always
      duration: 24h
      namespaces:
        - "security"
      # Security patches can deploy anytime

    # Block Friday evening deploys (reduced on-call)
    - kind: deny
      schedule: "0 17 * * 5"   # Friday 5 PM
      duration: 63h             # Until Monday 8 AM
```

---

## Part 5: Incident Documentation and Compliance

### Postmortems as Compliance Evidence

```
SOC 2 and PCI-DSS both require:
  - Documented incident response process
  - Evidence of incident handling
  - Root cause analysis
  - Remediation tracking

Postmortems serve BOTH operational AND compliance purposes.

POSTMORTEM TEMPLATE (NovaMart standard):
```

```markdown
# Incident Postmortem: [INCIDENT-ID] [Title]

## Metadata
- **Date:** YYYY-MM-DD
- **Severity:** SEV1 / SEV2 / SEV3 / SEV4
- **Duration:** X hours Y minutes
- **Impact:** [User-facing impact description]
- **Detection:** [How was it detected — monitoring, customer report, etc.]
- **Incident Commander:** [Name]
- **Author:** [Name]
- **Review Date:** [Date postmortem was reviewed by team]
- **Status:** Draft / Reviewed / Action Items Complete

## Executive Summary
[2-3 sentences: what happened, impact, resolution]

## Timeline (UTC)
| Time | Event |
|------|-------|
| 14:00 | Monitoring alert: payment-svc error rate > 5% |
| 14:02 | On-call acknowledged PagerDuty alert |
| 14:05 | IC declared, Slack incident channel created |
| 14:10 | Root cause identified: database connection pool exhausted |
| 14:15 | Mitigation: scaled payment-svc replicas from 5 to 15 |
| 14:20 | Error rate dropped below 1% |
| 14:30 | Permanent fix: increased connection pool max from 10 to 50 |
| 14:45 | Deployed fix to production |
| 15:00 | Monitoring confirmed stable. Incident resolved. |

## Root Cause
[Detailed technical explanation. Be specific.]
The connection pool was configured with max_connections=10 per pod.
During a traffic spike (3x normal), each pod exhausted its connections.
New requests were queued, timing out after 30 seconds, producing 503 errors.

## Impact
- **Users affected:** ~15,000 (estimated from error rate × traffic)
- **Revenue impact:** ~$45,000 (18 minutes × ~$2,500/min for payment flow)
- **SLO impact:** Error budget consumed: 8.5 minutes of 4.32 available
- **Data impact:** No data loss. No data exposure.
- **Compliance impact:** No PCI scope breach. Availability SLO breached.

## Detection
- **How detected:** Prometheus alert — payment-svc error ratio > 1% for 2 min
- **Time to detect:** 2 minutes (alert fired at 14:00, condition started ~13:58)
- **Detection gap:** None — alert worked as designed

## Resolution
- **Immediate:** Scaled pods to handle increased connection demand
- **Permanent:** Increased connection pool, added connection pool monitoring

## Lessons Learned

### What went well
- Alert fired within 2 minutes of impact start
- IC was declared quickly, communication was clear
- Root cause was identified in 5 minutes

### What went poorly
- Connection pool configuration was never load-tested at 3x traffic
- No alert on connection pool utilization (only on error rate)
- Runbook for payment-svc didn't cover connection exhaustion

### Where we got lucky
- Traffic spike was organic (marketing campaign), not sustained
- If it had been Black Friday (10x), scaling pods wouldn't have helped
  because RDS max_connections would have been hit

## Action Items
| ID | Action | Owner | Priority | Due Date | Status |
|----|--------|-------|----------|----------|--------|
| 1 | Add connection pool utilization metric + alert | @platform | P1 | 2024-01-22 | Done |
| 2 | Load test payment-svc at 5x and 10x traffic | @payments | P1 | 2024-01-29 | In Progress |
| 3 | Update payment-svc runbook with connection pool section | @platform | P2 | 2024-01-29 | Open |
| 4 | Review all services for connection pool configs | @platform | P2 | 2024-02-05 | Open |
| 5 | Implement RDS connection pool monitoring (RDS Proxy) | @platform | P3 | 2024-02-28 | Open |

## Blameless Note
This postmortem is blameless. The connection pool default was set during
initial service creation and was never revisited as traffic grew. This is
a systemic gap in our capacity planning process, not an individual failure.
```

```
POSTMORTEM COMPLIANCE REQUIREMENTS:

  PCI-DSS Req 12.10: Incident response plan must be tested annually
  SOC 2: Incident response process must be documented and evidenced
  
  NovaMart stores postmortems:
    1. Confluence (team-readable, linked from Jira incident ticket)
    2. S3 compliance bucket (immutable archive for auditors)
    3. Action items tracked in Jira with due dates
    
  AUDITOR QUESTION: "Show me your last 5 incidents and how you handled them."
  ANSWER: Jira query for incident tickets → each links to postmortem →
          postmortem shows timeline, root cause, action items →
          action items tracked to completion in Jira.
  
  This is a complete, auditable chain.

BLAMELESS CULTURE AND COMPLIANCE:
  "Blameless" doesn't mean "accountable to no one."
  It means: we investigate SYSTEMS, not PEOPLE.
  
  The auditor doesn't care who caused the incident.
  The auditor cares that:
    1. You detected it (monitoring evidence)
    2. You responded appropriately (timeline, communications)
    3. You found the root cause (technical analysis)
    4. You fixed it and verified the fix (action items, completion)
    5. You learned from it (prevention measures)
```

### Compliance Monitoring and Alerting

```yaml
# Compliance-specific Prometheus alerts

groups:
  - name: compliance-alerts
    rules:
      # Security Hub compliance score dropping
      - alert: SecurityHubComplianceDropped
        expr: |
          aws_securityhub_compliance_score{standard="aws-foundational"} < 90
        for: 1h
        labels:
          severity: warning
          compliance: "soc2"
        annotations:
          summary: "Security Hub compliance score dropped below 90%"
          runbook: "https://wiki.novamart.internal/runbooks/compliance-score-drop"

      # CloudTrail logging gap
      - alert: CloudTrailLoggingGap
        expr: |
          time() - aws_cloudtrail_last_delivery_timestamp > 3600
        for: 15m
        labels:
          severity: critical
          compliance: "pci-dss-req10"
        annotations:
          summary: "CloudTrail has not delivered logs in over 1 hour"
          description: "PCI Req 10 violation — audit logging gap"

      # Config recorder stopped
      - alert: AWSConfigRecorderStopped
        expr: |
          aws_config_recorder_status == 0
        for: 5m
        labels:
          severity: critical
          compliance: "soc2"
        annotations:
          summary: "AWS Config recorder is not running"

      # Encryption compliance
      - alert: UnencryptedResourceDetected
        expr: |
          aws_config_rule_compliance{rule_name="encrypted-volumes"} == 0 or
          aws_config_rule_compliance{rule_name="s3-bucket-encryption"} == 0
        for: 30m
        labels:
          severity: warning
          compliance: "pci-dss-req3"
        annotations:
          summary: "Unencrypted resource detected — PCI Req 3 violation"

      # Access review overdue
      - alert: AccessReviewOverdue
        expr: |
          (time() - novamart_last_access_review_timestamp) > (90 * 86400)
        for: 1d
        labels:
          severity: warning
          compliance: "soc2,pci-dss-req7"
        annotations:
          summary: "Quarterly access review is overdue"

      # Gatekeeper audit violations in PCI namespace
      - alert: PCINamespaceViolations
        expr: |
          sum(gatekeeper_violations{namespace=~"payments|fraud"}) > 0
        for: 15m
        labels:
          severity: critical
          compliance: "pci-dss"
        annotations:
          summary: "Gatekeeper violations detected in PCI-scoped namespace"
```

### Audit Preparation Checklist

```
WHEN THE AUDITOR ARRIVES (SOC 2 Type II or PCI-DSS Assessment):

PRE-AUDIT (2 weeks before):
  □ Generate Security Hub compliance report for audit period
  □ Generate AWS Config compliance timeline for audit period
  □ Export CloudTrail logs for audit period (verify integrity)
  □ Compile all postmortem reports from audit period
  □ Generate quarterly access review reports
  □ Run kube-bench and archive results
  □ Verify all Jira action items from previous audit are closed
  □ Prepare network diagrams (current, not 6 months old)
  □ Compile penetration test reports
  □ Generate certificate inventory and expiry report
  □ Document all change freezes and exceptions during audit period
  □ Prepare list of all third-party vendors with access

DURING AUDIT:
  □ Designate a technical liaison (platform engineer) for auditor
  □ Provide read-only access to:
    - Security Hub console
    - AWS Config console
    - CloudTrail console
    - Grafana compliance dashboard
    - Jira (filtered to compliance-tagged tickets)
  □ Be prepared to demonstrate:
    - A deployment going through CI/CD (live demo)
    - An incident alert → response → resolution (from recent incident)
    - Access review process (show the Lambda report)
    - Encryption evidence (Config rules, KMS key policies)
    - Network segmentation (VPC diagram, NetworkPolicy demo)
    - Log integrity (CloudTrail digest verification)

COMMON AUDITOR QUESTIONS YOU MUST ANSWER:
  "Show me how a code change gets to production"
    → PR → CI (scans, tests) → ArgoCD → production
    → Show Bitbucket PR with reviews, Jenkins build with scans, ArgoCD sync

  "Show me who has access to production infrastructure"
    → IAM roles and policies (Terraform), SSO group membership,
       RBAC in Kubernetes, Vault policies
    → Show access review report (quarterly Lambda output)

  "Show me your last incident and how you handled it"
    → Jira incident ticket → PagerDuty timeline → postmortem
    → Show action items and their completion status

  "Show me encryption is enabled on all data stores"
    → AWS Config rule: encrypted-volumes (COMPLIANT for all)
    → KMS key policies, RDS encryption config, S3 bucket encryption
    → Show Config compliance timeline (100% for audit period)

  "Show me your vulnerability management process"
    → Trivy scans in CI (Jenkins build logs)
    → ECR Enhanced Scanning (console or API)
    → Snyk dashboard (dependency monitoring)
    → Patch timeline: CVE published → image rebuilt → deployed

  "Show me separation of duties"
    → Branch protection: developer ≠ reviewer
    → ArgoCD: automated deploy (no human login to production)
    → Terraform: plan reviewed by different person than author
    → IAM: admin role ≠ deployment role
```

---

## Compliance Failure Modes

```
FAILURE 1: Compliance drift — score degrades slowly
  CAUSE: New resources created without compliance checks,
         Config rules disabled during troubleshooting and not re-enabled,
         engineers bypass Gatekeeper for "temporary" exceptions that persist
  SYMPTOM: Security Hub score drops from 95% to 78% over 3 months
  IMPACT: Audit finding, potential PCI non-compliance
  FIX: 
    - Alert on score drops (covered in alerts above)
    - Weekly compliance review in team standup
    - SCP-level prevention (can't create unencrypted resources)
  PREVENTION: Shift from detective (find problems) to preventive
              (make problems impossible)

FAILURE 2: Evidence gap during audit period
  CAUSE: CloudTrail was accidentally stopped for 3 days,
         Config recorder was paused during a Terraform migration
  SYMPTOM: Auditor finds 3-day gap in logging → PCI Req 10 violation
  IMPACT: Qualified audit opinion (not clean), remediation required
  FIX: 
    - SCP prevents stopping CloudTrail/Config (covered in SCPs above)
    - Alert on logging gaps (covered in alerts above)
    - Monitor CloudTrail delivery with <1 hour tolerance
  REALITY: A 3-day logging gap can turn a clean SOC 2 report into
           a qualified report. This is career-impacting for the CISO.

FAILURE 3: Access review not completed on time
  CAUSE: Quarterly access review Lambda failed silently,
         security team was busy with incidents
  SYMPTOM: No access review evidence for Q3
  IMPACT: SOC 2 finding, PCI Req 7 gap
  FIX: 
    - Monitor the Lambda: alert on failure
    - Jira ticket auto-created with due date
    - Escalation: if not completed by day 7, alert manager
  PREVENTION: Automate everything possible. The review REPORT
              should be automatic. Only the REMEDIATION is human.

FAILURE 4: Postmortem not completed for a significant incident
  CAUSE: Team resolved the incident, never wrote the postmortem
  SYMPTOM: Auditor asks "show me your incident response" → gap
  IMPACT: SOC 2 finding — incident response process not followed
  FIX:
    - Jira workflow: incident ticket CANNOT be closed without
      postmortem link
    - Automated reminder: 48 hours after incident resolution,
      if no postmortem → Slack reminder → manager escalation at 7 days
  PREVENTION: Make postmortem a required step in the incident lifecycle,
              enforced by tooling, not culture alone.

FAILURE 5: Scope creep — new service handles PCI data without assessment
  CAUSE: Order-svc team adds a feature that displays last 4 digits of
         card number. Nobody realizes this puts order-svc in PCI scope.
  SYMPTOM: PCI assessor discovers card data in order-svc during audit
  IMPACT: Expanded PCI scope, potentially non-compliant services in CDE
  FIX:
    - Data classification labels on all data fields
    - Code review checklist includes: "Does this change handle PII or
      payment data?"
    - Gatekeeper policy: pods in PCI namespaces require specific labels
      and security contexts
  PREVENTION: Architecture review for any feature that touches user
              financial data. Compliance team member in review.

FAILURE 6: Third-party vendor introduces compliance risk
  CAUSE: New SaaS tool integrated without security review,
         processes customer data in a non-compliant region
  SYMPTOM: Auditor discovers data flowing to unapproved vendor
  IMPACT: GDPR violation (data residency), SOC 2 vendor management gap
  FIX:
    - Vendor security assessment required before integration
    - Vendor must provide SOC 2 report or equivalent
    - Data processing agreement (DPA) for GDPR
    - Technical controls: VPC endpoints, network restrictions
  PREVENTION: Security review gate in procurement process.
              No vendor gets production access without assessment.
```

## Quick Reference Card

```
PCI-DSS
───────
12 requirements, 6 goals. Applies to CDE (cardholder data environment).
SCOPE REDUCTION: Tokenization (Stripe), segmentation (NetworkPolicies).
Key requirements for DevOps: 1 (network), 3 (encryption), 6 (secure SDLC),
  7 (access control), 8 (authentication), 10 (logging), 11 (testing).
Annual assessment (QSA or SAQ), quarterly ASV scans, annual pen test.

SOC 2
─────
5 Trust Service Criteria: Security (required), Availability, Processing
  Integrity, Confidentiality, Privacy.
Type I: Point-in-time. Type II: Period of time (12 months). Type II required.
Principles-based (not prescriptive). Show HOW you meet criteria.
Enterprise customers require SOC 2 Type II report.

GDPR
────
Data minimization, purpose limitation, storage limitation, right to erasure.
72-hour breach notification. Data residency (EU data in EU).
PII masking in logs (pipeline-level, not application-level).
Deletion pipeline with backup ledger for restores.

GOVERNANCE
──────────
SCPs: Account-level ceiling. Prevent region sprawl, security service
  disabling, privilege escalation, unencrypted resources.
  Test in sandbox OU first. Keep emergency OU with no SCPs.
Organizations: Multi-account. Management (billing only), Security,
  Infrastructure, Workloads (prod/staging/dev), Sandbox.
Access Analyzer: External access + unused access. Quarterly reviews automated.

CHANGE MANAGEMENT
─────────────────
4 types: Standard (auto), Normal (CAB), Emergency (retro-CAB), Major (arch review).
Separation of duties: developer ≠ reviewer ≠ deployer (GitOps handles this).
Change freeze: Black Friday, EOQ, holiday weekends. ArgoCD sync windows.
Evidence trail: Git → PR → CI → ArgoCD → CloudTrail (fully auditable).

AUDIT READINESS
───────────────
Evidence = continuously generated, S3-stored, immutable (Object Lock).
Control matrix: Requirement → Control → Tool → Evidence → Frequency.
Compliance dashboard: Security Hub scores, Config rules, finding trends.
Postmortems: Required for all incidents, stored as compliance evidence.
Key auditor questions: deployment process, access control, incident response,
  encryption, vulnerability management, separation of duties.

COMPLIANCE ALERTING
───────────────────
Alert on: Security Hub score drops, CloudTrail gaps, Config recorder stopped,
  unencrypted resources, overdue access reviews, PCI namespace violations.
Label alerts with compliance framework (pci-dss-req3, soc2, gdpr).
```


---

## Retention Questions

### Q1: PCI Scope Creep Incident 🔥

**Scenario:** During a routine code review, you notice that the `order-svc` team has added a new feature: displaying the customer's full card number (masked except last 4) on the order confirmation page. The card number is fetched from Stripe's API and briefly passes through `order-svc` before rendering. Currently, `order-svc` is classified as "connected to CDE" but NOT in the CDE itself.

1. What is the immediate compliance impact? Explain how this change affects NovaMart's PCI-DSS scope. Be specific about which requirements now apply to `order-svc` that didn't before.
2. The product manager says "We're only showing the masked number, and it's from Stripe's API — we never store it." Does this matter for PCI scoping? Why or why not? Explain the PCI concept that applies here.
3. What is your recommended technical architecture to achieve the product goal (showing last 4 digits on confirmation) WITHOUT expanding PCI scope? Include the specific implementation pattern.
4. Design the preventive controls that catch PCI scope changes BEFORE they reach production. Cover: code review, CI pipeline, architecture review, and runtime detection.

### Q2: SOC 2 Audit Evidence Gap 🔥

**Scenario:** NovaMart's SOC 2 Type II audit period is January 1 - December 31. The auditor arrives in February and discovers:

- AWS Config recorder was stopped for 12 days in March (Terraform migration)
- No access review evidence for Q2 (the Lambda failed, nobody noticed)
- A SEV1 incident in August has no postmortem document
- Security Hub compliance score dropped to 72% in July (unencrypted EBS volume created by a developer), recovered to 94% by August

1. For each of the 4 gaps, explain: (a) which SOC 2 Trust Service Criteria is affected, (b) the severity (will it result in a qualified opinion or just an observation?), and (c) what you can provide as compensating evidence.
2. The Config recorder gap is the most serious. The auditor asks: "How do I know your infrastructure was compliant during those 12 days?" What can you provide? (Hint: Config isn't the only evidence source.)
3. Design the monitoring and alerting that would have caught ALL FOUR of these gaps in real-time. Give specific alerts with thresholds.
4. The auditor says the overall finding picture suggests "compliance is not a continuous priority." How do you respond? Frame your answer as both technical evidence and organizational commitment.

### Q3: Multi-Framework Compliance Conflict 🔥

**Scenario:** NovaMart's legal team has signed a contract with a new EU enterprise customer that requires:
- GDPR compliance (data residency in EU)
- SOC 2 Type II report
- PCI-DSS compliance for payment processing
- Data must be deletable within 30 days of request (GDPR right to erasure)
- All data must be retained for 7 years (financial regulations for the customer's industry)

1. The 30-day deletion requirement and 7-year retention requirement directly conflict. How do you resolve this technically? Design the data architecture that satisfies BOTH requirements simultaneously.
2. The EU data residency requirement means customer data must stay in `eu-west-1`. But NovaMart's primary observability stack (Prometheus, Loki, Tempo) is in `us-east-1`. Application logs from the EU cluster contain customer identifiers (order IDs, email addresses in error messages). Is NovaMart in violation? How do you fix this?
3. The customer requires that their data be encrypted with a key THEY control (BYOK — Bring Your Own Key). Design how this works in NovaMart's architecture (KMS, RDS, S3) for a single customer's data within a multi-tenant system.
4. Write the SCP that enforces EU data residency for the EU production account. Include the specific services that must be restricted and the exceptions for global services.

### Q4: Change Management Under Pressure 🔥

**Scenario:** Thursday 4 PM. A critical vulnerability (CVSS 9.8) is published affecting the Istio proxy (Envoy) version running in all NovaMart clusters. The vulnerability allows remote code execution through crafted HTTP/2 requests. Your Istio version is affected. The fix requires upgrading Istio from 1.20.0 to 1.20.2.

The complication: NovaMart is in the Black Friday change freeze (started Wednesday). The CAB has explicitly denied all non-emergency changes. The change freeze policy says emergency changes require VP Engineering approval and a retroactive CAB review.

1. Walk through the complete change management process for this emergency change. Include: who you contact, what approvals you need, what documentation you create BEFORE making the change, and the exact timeline.
2. The Istio upgrade requires restarting all Envoy sidecars (every pod in the mesh). This means a rolling restart of ~400 pods across 3 clusters during Black Friday traffic. Design the rollout plan that minimizes customer impact. Include PDB considerations, canary testing, and rollback criteria.
3. During the rollout in the first cluster (`us-east-1`), 3 pods fail to start after the Istio upgrade — their init containers timeout connecting to `istiod`. Traffic to those services is degraded. Do you continue the rollout to the other clusters or stop? Justify your decision with specific criteria.
4. After the incident, the VP asks: "Should we have waited until Monday?" Write the risk analysis that justifies your decision. Include: CVSS score context, exploit availability, NovaMart's attack surface during Black Friday, and the cost of waiting vs the cost of the change freeze violation.

---
