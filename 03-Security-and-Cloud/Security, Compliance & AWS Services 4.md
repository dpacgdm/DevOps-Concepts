--- START OF FILE Paste April 25, 2026 - 12:42PM ---

# Phase 6, Lesson 5: Compliance & Governance

---

## Why This Lesson Matters at NovaMart

```text
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

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                  THE DEVOPS ENGINEER'S ROLE IN COMPLIANCE                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ You are NOT a compliance officer.                                           │
│ You ARE the person who IMPLEMENTS and AUTOMATES compliance.                 │
│                                                                             │
│ Compliance officer says: "We need encryption at rest for all data stores."  │
│ You implement: KMS encryption on RDS, S3, EBS, etcd, ElastiCache.           │
│ You automate: AWS Config rule that detects unencrypted resources.           │
│ You prove: Terraform code, Config compliance dashboard, CloudTrail logs.    │
│                                                                             │
│ Compliance officer says: "We need access reviews quarterly."                │
│ You implement: IAM Access Analyzer, unused credential reports.              │
│ You automate: Lambda that generates access review reports.                  │
│ You prove: S3-stored reports with timestamps, Jira tickets for remediation. │
│                                                                             │
│ The auditor doesn't care about your Terraform code.                         │
│ The auditor cares about EVIDENCE that controls exist and WORK.              │
│ Your job: make evidence generation automatic and continuous.                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Compliance Frameworks — What They Require and How NovaMart Meets Them

### PCI-DSS (Payment Card Industry Data Security Standard)

```text
PCI-DSS applies because NovaMart processes credit card payments.
Version: PCI-DSS v4.0 (effective March 2024, mandatory March 2025)

PCI-DSS has 12 REQUIREMENTS organized in 6 GOALS:

┌─────────────────────────────────────────────────────────────────────────────┐
│                          PCI-DSS v4.0 REQUIREMENTS                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ GOAL 1: BUILD AND MAINTAIN A SECURE NETWORK                                 │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Req 1: Install and maintain network security controls                   │ │
│ │   NovaMart: Security Groups, NACLs, NetworkPolicies,                    │ │
│ │   Istio mTLS, WAF, Cloudflare                                           │ │
│ │                                                                         │ │
│ │ Req 2: Apply secure configurations to all system components             │ │
│ │   NovaMart: CIS-hardened AMIs, Pod Security Standards,                  │ │
│ │   Gatekeeper policies, Ansible hardening playbooks                      │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│ GOAL 2: PROTECT ACCOUNT DATA                                                │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Req 3: Protect stored account data                                      │ │
│ │   NovaMart: KMS encryption (RDS, S3, EBS, etcd),                        │ │
│ │   Vault Transit for application-level encryption,                       │ │
│ │   data retention policies, tokenization for PAN storage                 │ │
│ │                                                                         │ │
│ │ Req 4: Protect cardholder data with strong cryptography                 │ │
│ │        during transmission over open, public networks                   │ │
│ │   NovaMart: TLS 1.2+ everywhere, Istio mTLS (internal),                 │ │
│ │   ACM certificates, cert-manager, HSTS headers                          │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│ GOAL 3: MAINTAIN A VULNERABILITY MANAGEMENT PROGRAM                         │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Req 5: Protect all systems and networks from malicious                  │ │
│ │        software                                                         │ │
│ │   NovaMart: Trivy image scanning, Falco runtime detection,              │ │
│ │   GuardDuty malware protection, read-only rootfs                        │ │
│ │                                                                         │ │
│ │ Req 6: Develop and maintain secure systems and software                 │ │
│ │   NovaMart: SonarQube (code quality), Snyk (dependencies),              │ │
│ │   secure SDLC, code review requirements, change management              │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│ GOAL 4: IMPLEMENT STRONG ACCESS CONTROL MEASURES                            │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Req 7: Restrict access to system components and cardholder              │ │
│ │        data by business need-to-know                                    │ │
│ │   NovaMart: IAM least privilege, RBAC, IRSA, Vault policies,            │ │
│ │   namespace isolation, NetworkPolicies                                  │ │
│ │                                                                         │ │
│ │ Req 8: Identify users and authenticate access to system                 │ │
│ │        components                                                       │ │
│ │   NovaMart: SSO (OIDC), MFA everywhere, unique accounts,                │ │
│ │   no shared credentials, service account management                     │ │
│ │                                                                         │ │
│ │ Req 9: Restrict physical access to cardholder data                      │ │
│ │   NovaMart: AWS responsibility (data center security).                  │ │
│ │   NovaMart responsibility: laptop encryption, office access             │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│ GOAL 5: REGULARLY MONITOR AND TEST NETWORKS                                 │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Req 10: Log and monitor all access to system components                 │ │
│ │         and cardholder data                                             │ │
│ │   NovaMart: CloudTrail, VPC Flow Logs, K8s audit logs,                  │ │
│ │   Vault audit logs, application logs (Loki), Falco                      │ │
│ │                                                                         │ │
│ │ Req 11: Test security of systems and networks regularly                 │ │
│ │   NovaMart: Penetration testing (annual + after changes),               │ │
│ │   vulnerability scanning (continuous), ASV scans (quarterly),           │ │
│ │   internal network scans, wireless scanning                             │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│ GOAL 6: MAINTAIN AN INFORMATION SECURITY POLICY                             │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Req 12: Support information security with organizational                │ │
│ │         policies and programs                                           │ │
│ │   NovaMart: Security policy documents, risk assessments,                │ │
│ │   security awareness training, incident response plan,                  │ │
│ │   vendor management program                                             │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

```text
PCI-DSS SCOPING — THE CRITICAL CONCEPT:

  PCI-DSS applies to the CDE (Cardholder Data Environment):
    - Systems that store, process, or transmit cardholder data
    - Systems connected to those systems
    - Systems that could affect security of the CDE

  NovaMart's CDE:
  ┌──────────────────────────────────────────────────────────────┐
  │                        PCI-DSS SCOPE                         │
  ├──────────────────────────────────────────────────────────────┤
  │ IN SCOPE (CDE):                                              │
  │ • payment-svc (processes card data)                          │
  │ • fraud-svc (analyzes transactions)                          │
  │ • RDS payment database                                       │
  │ • Stripe integration (card processing)                       │
  │ • EKS nodes running payment/fraud pods                       │
  │ • Network segments connecting these components               │
  │ • ALB/WAF handling payment traffic                           │
  │ • Vault (stores payment credentials)                         │
  │                                                              │
  │ CONNECTED TO CDE (also in scope):                            │
  │ • order-svc (sends payment requests)                         │
  │ • api-gateway (routes to payment-svc)                        │
  │ • Monitoring stack (accesses CDE metrics/logs)               │
  │ • CI/CD pipeline (deploys to CDE)                            │
  │ • EKS control plane                                          │
  │                                                              │
  │ OUT OF SCOPE:                                                │
  │ • search-svc (no cardholder data)                            │
  │ • notification-svc (no cardholder data — sends               │
  │   confirmation emails but no card numbers)                   │
  │ • cart-svc (stores items, not card data)                     │
  │ • Static assets (S3/CloudFront)                              │
  └──────────────────────────────────────────────────────────────┘

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

```text
SOC 2 applies because NovaMart's enterprise customers require it.
"Show me your SOC 2 report" is the first security question from
any enterprise B2B customer or partner.

SOC 2 is based on 5 TRUST SERVICE CRITERIA (TSC):

┌─────────────────────────────────────────────────────────────────────────────┐
│                         SOC 2 TRUST SERVICE CRITERIA                        │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. SECURITY (Required — always included)                                    │
│    "The system is protected against unauthorized access"                    │
│    Controls: Access control, encryption, monitoring, incident               │
│    response, vulnerability management, change management                    │
│    NovaMart: IAM, RBAC, KMS, WAF, GuardDuty, Falco, etc.                    │
│                                                                             │
│ 2. AVAILABILITY (Included for NovaMart)                                     │
│    "The system is available for operation as committed"                     │
│    Controls: SLOs, monitoring, incident response, DR, backups,              │
│    capacity planning, redundancy                                            │
│    NovaMart: Multi-AZ, auto-scaling, SLOs, PagerDuty, runbooks              │
│                                                                             │
│ 3. PROCESSING INTEGRITY (Included for NovaMart)                             │
│    "System processing is complete, valid, accurate, timely"                 │
│    Controls: Input validation, error handling, reconciliation,              │
│    data processing monitoring                                               │
│    NovaMart: SLIs for correctness, order reconciliation jobs,               │
│    payment verification, data pipeline monitoring                           │
│                                                                             │
│ 4. CONFIDENTIALITY (Included for NovaMart)                                  │
│    "Information designated as confidential is protected"                    │
│    Controls: Encryption, access control, data classification,               │
│    secure disposal                                                          │
│    NovaMart: KMS, Vault, network segmentation, data retention               │
│                                                                             │
│ 5. PRIVACY (Included — NovaMart stores PII)                                 │
│    "Personal information is collected, used, retained, disclosed            │
│     in conformity with commitments and criteria"                            │
│    Controls: Privacy policy, consent management, data subject               │
│    rights, data minimization                                                │
│    NovaMart: GDPR compliance, data subject access requests,                 │
│    anonymization pipelines, consent management                              │
└─────────────────────────────────────────────────────────────────────────────┘

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

```text
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

```text
GDPR TECHNICAL IMPLEMENTATIONS:

DATA CLASSIFICATION:
┌────────────────┬─────────────────────────┬────────────────────────────────┐
│ Classification │ Examples                │ Controls                       │
├────────────────┼─────────────────────────┼────────────────────────────────┤
│ PUBLIC         │ Product descriptions,   │ No special controls            │
│                │ pricing                 │                                │
│ INTERNAL       │ Internal docs, metrics  │ Auth required                  │
│ CONFIDENTIAL   │ Business data, orders   │ Encryption, RBAC               │
│ RESTRICTED     │ PII, payment data,      │ KMS, Vault, audit logging, DLP │
│                │ health data             │                                │
└────────────────┴─────────────────────────┴────────────────────────────────┘

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

```text
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
          command:["kube-bench", "run", "--targets", "node,policies,managedservices"]
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
#[FAIL] 4.1.5 Ensure that the --read-only-port argument is set to 0
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
                  if[ "$FAILURES" -gt "0" ]; then
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

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                          COMPLIANCE-AS-CODE PYRAMID                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ LAYER 1: PREVENTIVE CONTROLS (Stop bad things)                          │ │
│ │                                                                         │ │
│ │ SCPs → Account-level guardrails (can't be bypassed)                     │ │
│ │ Gatekeeper/Kyverno → K8s admission policies                             │ │
│ │ IAM Policies → Least privilege                                          │ │
│ │ AWS Config Rules → Resource compliance                                  │ │
│ │ Terraform Sentinel/OPA → IaC policy gates                               │ │
│ │ Branch protection → Code change governance                              │ │
│ │ NetworkPolicies → Network segmentation                                  │ │
│ │                                                                         │ │
│ │ BEST: Violations are IMPOSSIBLE. Can't create non-compliant             │ │
│ │       resources because the platform won't allow it.                    │ │
│ └─────────────────────────────┬───────────────────────────────────────────┘ │
│                               │                                             │
│ ┌─────────────────────────────▼───────────────────────────────────────────┐ │
│ │ LAYER 2: DETECTIVE CONTROLS (Find bad things)                           │ │
│ │                                                                         │ │
│ │ Security Hub → Compliance scoring                                       │ │
│ │ AWS Config → Continuous resource evaluation                             │ │
│ │ GuardDuty → Threat detection                                            │ │
│ │ Falco → Runtime anomalies                                               │ │
│ │ CloudTrail → API audit trail                                            │ │
│ │ kube-bench → CIS benchmark compliance                                   │ │
│ │ Trivy → Vulnerability detection                                         │ │
│ │ IAM Access Analyzer → Unintended access paths                           │ │
│ │                                                                         │ │
│ │ SECOND BEST: Violations are DETECTED quickly and                        │ │
│ │ automatically. Alert → investigate → remediate.                         │ │
│ └─────────────────────────────┬───────────────────────────────────────────┘ │
│                               │                                             │
│ ┌─────────────────────────────▼───────────────────────────────────────────┐ │
│ │ LAYER 3: CORRECTIVE CONTROLS (Fix bad things)                           │ │
│ │                                                                         │ │
│ │ Config auto-remediation → Fix non-compliant resources                   │ │
│ │ GuardDuty auto-response → Quarantine compromised instances              │ │
│ │ Falco auto-response → Kill compromised containers                       │ │
│ │ Lambda functions → Auto-fix security groups, S3 policies                │ │
│ │ Incident response playbooks → Human-driven remediation                  │ │
│ │                                                                         │ │
│ │ FALLBACK: When prevention and detection aren't enough,                  │ │
│ │ fix automatically or with clear procedures.                             │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│ PRINCIPLE: Maximize Layer 1. Layer 2 catches what Layer 1 misses.           │
│ Layer 3 is for when Layer 2 finds something.                                │
│ If you're relying on Layer 3 daily, Layer 1 is insufficient.                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### AWS Organizations — Multi-Account Governance

```text
NovaMart uses AWS Organizations for account-level isolation:

┌─────────────────────────────────────────────────────────────────────────────┐
│                         AWS ORGANIZATIONS STRUCTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ Root                                                                        │
│ ├── Management Account (billing, Organizations admin ONLY)                  │
│ │   └── NO workloads here. Ever.                                            │
│ │                                                                           │
│ ├── OU: Security                                                            │
│ │   ├── security-audit (CloudTrail aggregation, Security Hub admin)         │
│ │   ├── security-logging (centralized log archive)                          │
│ │   └── security-tooling (GuardDuty admin, Inspector)                       │
│ │                                                                           │
│ ├── OU: Infrastructure                                                      │
│ │   ├── network-hub (Transit Gateway, Direct Connect, DNS)                  │
│ │   └── shared-services (ECR, Artifactory, monitoring)                      │
│ │                                                                           │
│ ├── OU: Workloads                                                           │
│ │   ├── OU: Production                                                      │
│ │   │   ├── novamart-prod-us-east-1                                         │
│ │   │   ├── novamart-prod-us-west-2                                         │
│ │   │   └── novamart-prod-eu-west-1                                         │
│ │   ├── OU: Staging                                                         │
│ │   │   └── novamart-staging                                                │
│ │   └── OU: Development                                                     │
│ │       └── novamart-dev                                                    │
│ │                                                                           │
│ └── OU: Sandbox                                                             │
│     └── novamart-sandbox (experimentation, limited budget)                  │
│                                                                             │
│ WHY MULTI-ACCOUNT?                                                          │
│ 1. Blast radius: Compromise in dev can't reach production                   │
│ 2. Billing: Clear cost allocation per environment                           │
│ 3. Limits: AWS service limits are per-account                               │
│ 4. Compliance: CDE (PCI) isolation in dedicated account                     │
│ 5. IAM: Separate IAM boundaries per account                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Service Control Policies — Organization-Level Guardrails

```text
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
    Statement =[
      {
        Sid       = "DenyNonApprovedRegions"
        Effect    = "Deny"
        NotAction =[
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
            "aws:RequestedRegion" =[
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
    Statement =[
      {
        Sid    = "DenyDisableGuardDuty"
        Effect = "Deny"
        Action =[
          "guardduty:DeleteDetector",
          "guardduty:DisassociateFromMasterAccount",
          "guardduty:UpdateDetector"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDisableCloudTrail"
        Effect = "Deny"
        Action =[
          "cloudtrail:DeleteTrail",
          "cloudtrail:StopLogging",
          "cloudtrail:UpdateTrail"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDisableConfig"
        Effect = "Deny"
        Action =[
          "config:DeleteConfigurationRecorder",
          "config:StopConfigurationRecorder",
          "config:DeleteDeliveryChannel"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyDisableSecurityHub"
        Effect = "Deny"
        Action =[
          "securityhub:DisableSecurityHub",
          "securityhub:DeleteMembers",
          "securityhub:DisassociateFromMasterAccount"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyLeaveOrganization"
        Effect = "Deny"
        Action =[
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
    Statement =[
      {
        Sid    = "DenyAdminPolicyAttachment"
        Effect = "Deny"
        Action =[
          "iam:AttachRolePolicy",
          "iam:AttachUserPolicy",
          "iam:AttachGroupPolicy"
        ]
        Resource = "*"
        Condition = {
          ArnEquals = {
            "iam:PolicyArn" =[
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
        Action =[
          "iam:CreateUser",
          "iam:CreateAccessKey"
        ]
        Resource = "*"
        Condition = {
          StringNotLike = {
            "aws:PrincipalArn" =[
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
    Statement =[
      {
        Sid    = "DenyUnencryptedS3Upload"
        Effect = "Deny"
        Action =["s3:PutObject"]
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
        Action =["rds:CreateDBInstance", "rds:CreateDBCluster"]
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

```text
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

```text
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
    unused_users =[]
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
    unused_keys =[]
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
    unused_access_findings =[]
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
    external_findings =[]
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

```text
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

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 NOVAMART CONTINUOUS COMPLIANCE ARCHITECTURE                 │
├─────────────────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ EVIDENCE SOURCES                                                        │ │
│ │                                                                         │ │
│ │ AWS Config          → Resource configuration over time                  │ │
│ │ Security Hub        → Compliance scores over time                       │ │
│ │ CloudTrail          → API audit trail                                   │ │
│ │ Gatekeeper audits   → K8s policy compliance                             │ │
│ │ kube-bench          → CIS benchmark results                             │ │
│ │ Trivy scans         → Vulnerability scan results                        │ │
│ │ Terraform plans     → Infrastructure change evidence                    │ │
│ │ ArgoCD sync history → Deployment change evidence                        │ │
│ │ PagerDuty           → Incident history and response times               │ │
│ │ Postmortems         → Incident analysis documentation                   │ │
│ │ Access reviews      → Quarterly IAM review results                      │ │
│ │ Pen test reports    → Annual penetration test results                   │ │
│ └────────────────────────────────────┬────────────────────────────────────┘ │
│                                      │                                      │
│                                      ▼                                      │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ EVIDENCE STORE (S3 — immutable, encrypted, versioned)                   │ │
│ │                                                                         │ │
│ │ s3://novamart-compliance/                                               │ │
│ │ ├── config-snapshots/          (AWS Config, 6-hourly)                   │ │
│ │ ├── security-hub/              (compliance scores, daily)               │ │
│ │ ├── cloudtrail/                (API logs, continuous)                   │ │
│ │ ├── kube-bench/                (CIS results, weekly)                    │ │
│ │ ├── trivy-scans/               (vuln scans, per build)                  │ │
│ │ ├── access-reviews/            (IAM reviews, quarterly)                 │ │
│ │ ├── pen-tests/                 (results, annual)                        │ │
│ │ ├── incident-postmortems/      (after each incident)                    │ │
│ │ ├── change-records/            (Terraform plans + approvals)            │ │
│ │ └── training-records/          (security training completion)           │ │
│ │                                                                         │ │
│ │ S3 features:                                                            │ │
│ │ • Object Lock (COMPLIANCE mode — immutable for audit period)            │ │
│ │ • Versioning (track changes)                                            │ │
│ │ • KMS encryption (data protection)                                      │ │
│ │ • Access logging (who accessed evidence)                                │ │
│ │ • Lifecycle (archive after audit period, retain per policy)             │ │
│ └────────────────────────────────────┬────────────────────────────────────┘ │
│                                      │                                      │
│                                      ▼                                      │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ COMPLIANCE DASHBOARD (Grafana)                                          │ │
│ │                                                                         │ │
│ │ Panel 1: Security Hub compliance scores by standard                     │ │
│ │ Panel 2: Open findings by severity and age                              │ │
│ │ Panel 3: Config rule compliance % over time                             │ │
│ │ Panel 4: Mean time to remediate (MTTR) for findings                     │ │
│ │ Panel 5: Gatekeeper violations trend                                    │ │
│ │ Panel 6: Access review completion status                                │ │
│ │ Panel 7: Certificate expiry timeline                                    │ │
│ │ Panel 8: Encryption coverage (% of resources encrypted)                 │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Mapping Controls to Evidence — The Control Matrix

```text
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
│ PCI Req 4:         │ Encryption in       │ TLS 1.2+, Istio    │ ALB access logs,     │ Continuous│
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
│ SOC 2 Change:      │ Change controls     │ GitOps (ArgoCD),   │ Git history, PR      │ Per change│
│ Change controls    │                     │ PR reviews,        │ reviews, ArgoCD      │           │
│                    │                     │ Terraform plans    │ sync history         │           │
├────────────────────┼─────────────────────┼────────────────────┼──────────────────────┼───────────┤
│ GDPR Art 17:       │ Right to erasure    │ Deletion pipeline  │ Deletion request     │ Per       │
│ Right to be        │                     │ (Lambda +          │ logs, completion     │ request   │
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

```text
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

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                      NOVAMART CHANGE MANAGEMENT PROCESS                     │
├─────────────────────────────────────────────────────────────────────────────┤
│ CHANGE TYPES:                                                               │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Standard Change (pre-approved, low risk):                               │ │
│ │   • Application deployment via CI/CD (passing all gates)                │ │
│ │   • Scaling adjustments (HPA threshold changes)                         │ │
│ │   • Dependency updates (non-security, non-major version)                │ │
│ │   Process: PR → 1 reviewer → merge → auto-deploy                        │ │
│ │   Approval: Peer review only                                            │ │
│ │                                                                         │ │
│ │ Normal Change (moderate risk):                                          │ │
│ │   • New service deployment                                              │ │
│ │   • Infrastructure changes (Terraform)                                  │ │
│ │   • Database schema migrations                                          │ │
│ │   • Gatekeeper policy changes                                           │ │
│ │   • Security group modifications                                        │ │
│ │   Process: PR → 2 reviewers → CAB approval → merge → deploy             │ │
│ │   Approval: Peer review + team lead                                     │ │
│ │                                                                         │ │
│ │ Emergency Change (incident response):                                   │ │
│ │   • Hotfix for active incident                                          │ │
│ │   • Security patch for critical CVE                                     │ │
│ │   • Configuration rollback                                              │ │
│ │   Process: PR → 1 reviewer → merge → deploy → RETROACTIVE CAB           │ │
│ │   Approval: On-call lead (retroactive review within 48 hours)           │ │
│ │                                                                         │ │
│ │ Major Change (high risk, high impact):                                  │ │
│ │   • Kubernetes version upgrade                                          │ │
│ │   • Database migration (engine change, major version)                   │ │
│ │   • Network architecture changes (VPC, Transit Gateway)                 │ │
│ │   • New compliance-affecting service                                    │ │
│ │   Process: RFC → Architecture review → CAB → scheduled window           │ │
│ │   Approval: Architecture board + VP Engineering                         │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│ CHANGE ADVISORY BOARD (CAB):                                                │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Meets: Weekly (Tuesday 10 AM) + on-demand for emergencies               │ │
│ │ Members: Platform lead, Security lead, SRE lead, QA lead                │ │
│ │ Reviews: All Normal and Major changes for the week                      │ │
│ │ Outputs: Approved, Deferred, Rejected (with reasons)                    │ │
│ │ Records: Meeting notes in Confluence, Jira ticket updates               │ │
│ │                                                                         │ │
│ │ CAB does NOT slow down standard deployments.                            │ │
│ │ CI/CD pipeline handles standard changes automatically.                  │ │
│ │ CAB reviews infrastructure, policy, and high-risk changes.              │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│ CHANGE RECORD (Jira ticket template):                                       │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ • Description: What is being changed and why                            │ │
│ │ • Risk assessment: Impact if change fails                               │ │
│ │ • Rollback plan: Exact steps to undo the change                         │ │
│ │ • Testing evidence: What was tested and results                         │ │
│ │ • Dependencies: Other changes or teams affected                         │ │
│ │ • Schedule: When will the change be applied                             │ │
│ │ • Verification: How to confirm the change succeeded                     │ │
│ │ • Approvals: PR link, reviewer names, CAB decision                      │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│ EVIDENCE TRAIL (automatic via GitOps):                                      │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Git commit → WHO made the change (author + committer)                   │ │
│ │ PR review  → WHO approved (reviewer signatures)                         │ │
│ │ CI logs    → WHAT was tested (scan results, test results)               │ │
│ │ ArgoCD     → WHEN it was deployed (sync timestamp)                      │ │
│ │ CloudTrail → HOW infrastructure changed (API calls)                     │ │
│ │ Prometheus → WHETHER it worked (SLI before/after)                       │ │
│ │                                                                         │ │
│ │ An auditor can trace ANY production change from:                        │ │
│ │   Jira ticket → PR → CI build → ArgoCD sync → running pod               │ │
│ │ Every link is timestamped, immutable, and attributable.                 │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Separation of Duties — The Compliance Requirement

```text
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

```text
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

```text
SOC 2 and PCI-DSS both require:
  - Documented incident response process
  - Evidence of incident handling
  - Root cause analysis
  - Remediation tracking

Postmortems serve BOTH operational AND compliance purposes.

POSTMORTEM TEMPLATE (NovaMart standard):
```

```markdown
# Incident Postmortem:[INCIDENT-ID] [Title]

## Metadata
- **Date:** YYYY-MM-DD
- **Severity:** SEV1 / SEV2 / SEV3 / SEV4
- **Duration:** X hours Y minutes
- **Impact:** [User-facing impact description]
- **Detection:**[How was it detected — monitoring, customer report, etc.]
- **Incident Commander:** [Name]
- **Author:** [Name]
- **Review Date:**[Date postmortem was reviewed by team]
- **Status:** Draft / Reviewed / Action Items Complete

## Executive Summary[2-3 sentences: what happened, impact, resolution]

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

```text
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

```text
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

```text
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

```text
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
# NovaMart Compliance, Governance & Change Management

---

## Q1: PCI Scope Creep Incident 🔥

### 1. Immediate Compliance Impact

**This is a PCI scope expansion event. It must be treated as a compliance incident.**

Before this change, NovaMart's PCI-DSS scoping looked like this:

```text
BEFORE:
┌──────────────────────────────────────────────────────────────┐
│                        PCI-DSS SCOPE                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ CDE (Cardholder Data Environment):                           │
│   └── Stripe (handles all card data — NovaMart never         │
│       touches PAN, CVV, or expiry)                           │
│                                                              │
│ Connected-to-CDE:                                            │
│   └── order-svc (sends payment intents to Stripe API,        │
│       receives payment confirmation tokens — never           │
│       handles raw cardholder data)                           │
│                                                              │
│ Out of Scope:                                                │
│   └── search-svc, inventory-svc, notification-svc, etc.      │
│                                                              │
│ PCI Compliance Method: SAQ-A or SAQ A-EP                     │
│ (Stripe handles all cardholder data processing)              │
└──────────────────────────────────────────────────────────────┘
```

**AFTER this code change:**

The `order-svc` now receives the PAN (Primary Account Number) from Stripe's API — even in masked form — processes it in memory, and renders it in an HTTP response. This means:

```text
AFTER:
┌──────────────────────────────────────────────────────────────┐
│                  PCI-DSS SCOPE (EXPANDED)                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ CDE (Cardholder Data Environment):                           │
│   ├── Stripe                                                 │
│   └── order-svc  ← MOVED INTO CDE                            │
│                                                              │
│ Connected-to-CDE (expanded blast radius):                    │
│   ├── ALB / Ingress controller (carries response with PAN)   │
│   ├── Any service mesh proxy (Envoy sidecar)                 │
│   ├── Monitoring stack (if it logs HTTP response bodies)     │
│   ├── The EKS node running order-svc pods                    │
│   └── The network segment order-svc runs on                  │
│                                                              │
│ PCI Compliance Method: REQUIRES SAQ D or full ROC            │
│ (NovaMart now processes cardholder data)                     │
└──────────────────────────────────────────────────────────────┘
```

**Specific PCI-DSS requirements that NOW apply to `order-svc` that didn't before:**

| Requirement | What It Mandates | Impact on order-svc |
|---|---|---|
| **Req 3: Protect stored cardholder data** | PAN must be rendered unreadable anywhere it's stored (encryption, truncation, hashing, tokenization) | Even if "in memory only," the PAN exists in process memory, potentially in core dumps, heap dumps, and swap space |
| **Req 3.4** | Render PAN unreadable using strong cryptography | The masked PAN transiting through order-svc must be protected |
| **Req 4: Encrypt transmission** | Encrypt cardholder data across open/public networks | The HTTP response containing the masked PAN must be encrypted end-to-end (TLS) — which it likely is, but must now be VERIFIED and documented |
| **Req 6.5** | Secure coding practices for applications handling cardholder data | order-svc now requires PCI-specific secure code review, SAST/DAST scanning for PAN leakage |
| **Req 10: Track and monitor access** | Log all access to cardholder data | Every request/response involving the masked PAN must be audited — but the PAN itself must NOT appear in logs |
| **Req 11.3** | Penetration testing | order-svc now requires annual penetration testing specifically targeting cardholder data flows |
| **Req 12.8** | Service provider management | The connection between order-svc and Stripe must be formally documented in NovaMart's PCI responsibility matrix |

**The cascade effect is massive.** Moving order-svc into the CDE doesn't just affect order-svc — it pulls in every system that touches order-svc's network traffic, every log aggregator that might capture the response, and every monitoring tool that might scrape metrics containing the PAN. This is why PCI scope creep is so dangerous — one code change can expand the audit boundary to include dozens of systems.

**NovaMart's PCI compliance method also changes.** If they were previously using SAQ A-EP (because Stripe handled all card data and NovaMart's systems never processed it), they now likely need SAQ D or a full Report on Compliance (ROC) — which is significantly more expensive and time-consuming (typically $50K-$200K for a QSA assessment).

---

### 2. "We Only Show the Masked Number" — Does This Matter?

**The product manager's argument has two claims. Both are wrong in ways that matter for PCI scoping.**

**Claim 1: "We're only showing the masked number"**

PCI-DSS defines cardholder data as the PAN (Primary Account Number). Under Requirement 3.3, displaying the PAN is permitted only when masked — showing at most the first 6 and last 4 digits. So the *display* of a masked PAN is technically compliant with Requirement 3.3.

**BUT — and this is the critical point — the PCI concept that applies here is SCOPE, not just data handling.**

PCI-DSS scoping is determined by whether a system **stores, processes, or transmits** cardholder data. The key word is **processes**:

> **PCI-DSS Glossary — "Process":** Any manipulation, handling, or use of cardholder data within a system, including but not limited to: receiving, transforming, transmitting, or displaying cardholder data.

`order-svc` receives the PAN from Stripe's API (even masked), holds it in memory, and includes it in an HTTP response. **That is processing.** The fact that the PAN is masked doesn't remove order-svc from scope — it means order-svc is handling cardholder data in a compliant FORMAT, but it's still handling cardholder data. The system is still in scope.

**Claim 2: "It's from Stripe's API — we never store it"**

This is the more dangerous misunderstanding. The PCI concept that applies is **the Shared Responsibility Model for PCI**:

```text
Stripe's Responsibility:
  ✓ Securely storing the full PAN
  ✓ Returning the masked PAN via their API
  ✓ Their API being PCI-compliant

NovaMart's Responsibility (NEW):
  ✗ The code that receives the PAN from Stripe
  ✗ The memory space where the PAN exists
  ✗ The HTTP response that carries the PAN to the browser
  ✗ Any log, cache, or monitoring system that might capture the PAN
  ✗ The network path the response travels
  ✗ The browser that renders the PAN (client-side is customer's scope,
    but the TRANSMISSION to the browser is NovaMart's scope)
```

**"We never store it" is irrelevant.** PCI scope is triggered by store OR process OR transmit. Transmitting through memory and over HTTP is processing and transmitting. The data doesn't need to touch a database to bring a system into scope.

Furthermore, even "briefly in memory" creates concrete risks:
- **Core dumps / heap dumps** could contain the PAN
- **Application logs** might accidentally log the API response body
- **APM/tracing tools** (Jaeger, Datadog) often capture HTTP response bodies — the PAN would appear in traces
- **Error handling** — if the request fails and the error is logged with context, the PAN could be in the error log
- **Service mesh** — Envoy access logs could capture the response body

**The PM's framing reveals a common and dangerous misconception:** PCI compliance is not about what you INTEND to do with the data. It's about what the data TOUCHES. Every system the PAN passes through, even transiently, is in scope.

---

### 3. Architecture That Achieves the Goal WITHOUT Expanding PCI Scope

**The goal:** Show the customer their card's last 4 digits on the order confirmation page.

**The principle:** The card information must flow directly from Stripe to the customer's browser, never passing through NovaMart's backend systems.

**Solution: Client-Side Tokenized Rendering (Stripe Elements / Stripe.js)**

```text
CORRECT ARCHITECTURE:
                                                              
  Browser                  NovaMart Backend           Stripe
  ───────                  ────────────────           ──────
     │                            │                      │
     │  1. Load order page        │                      │
     │ ────────────────────────►  │                      │
     │                            │                      │
     │  2. HTML + Stripe.js       │                      │
     │     (includes payment      │                      │
     │      intent client         │                      │
     │      secret)               │                      │
     │ ◄────────────────────────  │                      │
     │                            │                      │
     │  3. Stripe.js calls        │                      │
     │     Stripe API DIRECTLY    │                      │
     │     from browser           │                      │
     │ ───────────────────────────────────────────────►  │
     │                            │                      │
     │  4. Stripe returns         │                      │
     │     card.last4 directly    │                      │
     │     to browser             │                      │
     │ ◄───────────────────────────────────────────────  │
     │                            │                      │
     │  5. Browser renders        │                      │
     │     "Card ending 4242"     │                      │
     │                            │                      │
     │  NovaMart backend          │                      │
     │  NEVER SEES THE PAN        │                      │
```

**Implementation:**

```javascript
// Frontend — Order Confirmation Page
// Uses Stripe.js to retrieve payment method details directly from Stripe

// Step 1: Backend provides the payment_intent ID (NOT card data)
// This is already in order-svc's database — it's a Stripe token, not cardholder data
const orderData = await fetch('/api/v1/orders/12345');
const { stripe_payment_intent_id } = await orderData.json();

// Step 2: Stripe.js retrieves the payment method details DIRECTLY from Stripe
// This call goes from the BROWSER to STRIPE — never touches NovaMart's servers
const stripe = Stripe('pk_live_xxxxx');  // Public key — safe to expose
const paymentIntent = await stripe.retrievePaymentIntent(clientSecret);

// Step 3: Render the last 4 digits
const last4 = paymentIntent.payment_method.card.last4;
const brand = paymentIntent.payment_method.card.brand;
document.getElementById('card-info').textContent = 
  `Paid with ${brand} ending in ${last4}`;
```

```python
# Backend — order-svc (UNCHANGED from current PCI scope)
# Only returns the payment_intent client_secret, NOT card data

@app.route('/api/v1/orders/<order_id>')
def get_order(order_id):
    order = db.query(Order).get(order_id)
    return jsonify({
        'order_id': order.id,
        'total': order.total,
        'status': order.status,
        'stripe_payment_intent_client_secret': order.stripe_pi_client_secret,
        # NO card data here — only the Stripe reference token
    })
```

**Why this works for PCI scope:**

- `order-svc` never receives, processes, or transmits cardholder data — it only handles Stripe tokens (payment intent IDs and client secrets)
- The card's last 4 digits flow from Stripe → Browser, never touching NovaMart's infrastructure
- NovaMart remains on SAQ A-EP (or even SAQ A if they fully use Stripe's hosted elements)
- The `stripe_payment_intent_client_secret` is NOT cardholder data — it's a Stripe API token that cannot be used to retrieve the full PAN

**Alternative if Stripe.js direct retrieval isn't possible:**

Store the `last4` and `brand` in NovaMart's database at the time of payment creation. Stripe's PaymentIntent webhook includes `payment_method.card.last4` and `payment_method.card.brand` — these are NOT cardholder data under PCI-DSS because they cannot be used to reconstruct the full PAN. The PCI Council explicitly states that the last 4 digits alone are not considered PAN and do not trigger PCI scope.

```python
# At payment creation time, store truncated card info
# This is SAFE — last4 and brand are NOT cardholder data per PCI
@app.route('/webhooks/stripe', methods=['POST'])
def stripe_webhook(event):
    if event['type'] == 'payment_intent.succeeded':
        pi = event['data']['object']
        order = db.query(Order).filter_by(stripe_pi_id=pi['id']).first()
        order.card_last4 = pi['charges']['data'][0]['payment_method_details']['card']['last4']
        order.card_brand = pi['charges']['data'][0]['payment_method_details']['card']['brand']
        db.commit()
```

---

### 4. Preventive Controls for PCI Scope Changes

**Layer 1: Code Review — PCI-Aware Review Checklist**

```markdown
# .github/PULL_REQUEST_TEMPLATE/pci-review.md (or Bitbucket equivalent)
# Auto-triggered when changes touch payment-related services

## PCI Scope Review Checklist

### Mandatory for changes to: order-svc, payment-svc, checkout-svc

- [ ] Does this change introduce handling of any cardholder data?
  - PAN (full or partial card number)
  - CVV/CVC
  - Expiration date
  - Cardholder name (in combination with PAN)
  - Track data (magnetic stripe)

- [ ] Does this change call Stripe/payment APIs that return card data?
  - If YES: does the response pass through NovaMart's backend?
  - If YES: **STOP — requires Security Architecture Review**

- [ ] Does this change modify HTTP responses to include card-related fields?

- [ ] Does this change add new logging that could capture card data?

- [ ] Does this change add new API endpoints that accept card-related input?

If ANY box above is checked YES, this PR requires:
  1. Security team review (@novamart/security-team)
  2. PCI scope impact assessment (link to template)
  3. Approval from the PCI compliance owner
```

```yaml
# CODEOWNERS — enforce security review for payment-related code
# Any file in payment-related services requires security team approval
services/order-svc/src/**/payment*    @novamart/security-team
services/order-svc/src/**/card*       @novamart/security-team
services/order-svc/src/**/stripe*     @novamart/security-team
services/payment-svc/**               @novamart/security-team
services/checkout-svc/**              @novamart/security-team
**/api/**/payment*                    @novamart/security-team
```

**Layer 2: CI Pipeline — Automated PCI Scope Detection**

```yaml
# Bitbucket Pipeline step — runs on every PR to payment-related services
- step:
    name: PCI Scope Scanner
    script:
      # Check 1: Scan for cardholder data patterns in code
      - |
        echo "=== Scanning for PCI-sensitive patterns ==="
        
        # Regex patterns for cardholder data handling
        PATTERNS=(
          'card[._-]?num'
          'pan[^a-z]'
          'credit[._-]?card'
          'card[._-]?number'
          'account[._-]?number'
          'payment_method\.card\.'
          'payment_method_details\.card\.'
          '\.card\.number'
          '\.card\.cvc'
          '\.card\.exp'
          'cardholder'
          'track[12][._-]?data'
        )
        
        FOUND=false
        for pattern in "${PATTERNS[@]}"; do
          MATCHES=$(git diff origin/main...HEAD -- '*.java' '*.py' '*.js' '*.ts' '*.go' \
            | grep -iP "^\+.*$pattern" || true)
          if [ -n "$MATCHES" ]; then
            echo "⚠️  PCI-SENSITIVE PATTERN FOUND: $pattern"
            echo "$MATCHES"
            FOUND=true
          fi
        done
        
        if[ "$FOUND" = true ]; then
          echo ""
          echo "⛔ PCI SCOPE ALERT: This PR introduces patterns associated with cardholder data handling."
          echo "This PR REQUIRES security team review before merge."
          echo "See: https://wiki.novamart.internal/pci-scope-review-process"
          exit 1
        fi

      # Check 2: Scan for Stripe API calls that return card data
      - |
        echo "=== Scanning for Stripe API calls returning card data ==="
        
        # These Stripe API calls return cardholder data:
        STRIPE_RISKY_CALLS=(
          'stripe\.paymentMethods\.retrieve'
          'stripe\.customers\.retrievePaymentMethod'
          'stripe\.charges\.retrieve'
          'payment_method_details'
          'stripe\.tokens\.retrieve'
        )
        
        for call in "${STRIPE_RISKY_CALLS[@]}"; do
          MATCHES=$(git diff origin/main...HEAD | grep -iP "^\+.*$call" || true)
          if[ -n "$MATCHES" ]; then
            echo "⛔ STRIPE API CALL RETURNS CARD DATA: $call"
            echo "$MATCHES"
            echo "If this data passes through NovaMart's backend, PCI scope expands."
            exit 1
          fi
        done
        
        echo "✅ No PCI-sensitive Stripe API calls detected"

      # Check 3: Verify no new HTTP response fields contain card data
      - |
        echo "=== Scanning for card data in API response schemas ==="
        
        # Check OpenAPI specs for card-related response fields
        find . -name "*.yaml" -o -name "*.json" | \
          xargs grep -l "openapi\|swagger" 2>/dev/null | \
          while read spec; do
            if grep -iP 'card|pan|credit.*number|account.*number' "$spec" | \
               grep -iv 'last4\|brand\|fingerprint'; then
              echo "⛔ API spec $spec contains card data fields in response"
              exit 1
            fi
          done
        
        echo "✅ No card data in API response schemas"
```

**Layer 3: Architecture Review Gate**

```yaml
# Gatekeeper policy — prevent deployments that add cardholder data environment variables
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockpcienvvars
spec:
  crd:
    spec:
      names:
        kind: K8sBlockPCIEnvVars
      validation:
        openAPIV3Schema:
          type: object
          properties:
            blockedPatterns:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblockpcienvvars

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          env := container.env[_]
          pattern := input.parameters.blockedPatterns[_]
          regex.match(pattern, lower(env.name))
          msg := sprintf(
            "Container '%s' has env var '%s' matching PCI-sensitive pattern '%s'. Cardholder data must not be passed via environment variables. Contact security team.",
            [container.name, env.name, pattern]
          )
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          env := container.env[_]
          pattern := input.parameters.blockedPatterns[_]
          env.valueFrom.secretKeyRef
          regex.match(pattern, lower(env.valueFrom.secretKeyRef.key))
          msg := sprintf(
            "Container '%s' references secret key '%s' matching PCI-sensitive pattern. Contact security team.",
            [container.name, env.valueFrom.secretKeyRef.key]
          )
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockPCIEnvVars
metadata:
  name: block-pci-env-vars
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet"]
  parameters:
    blockedPatterns:
      - "card.*num"
      - "pan$"
      - "^pan[^a-z]"
      - "credit.*card"
      - "cvv"
      - "cvc"
      - "cardholder"
      - "track.*data"
      - "card.*expir"
```

**Layer 4: Runtime Detection**

```yaml
# Falco rule — detect card data patterns in network traffic
- rule: Potential PAN in HTTP Response
  desc: >
    Detects patterns matching credit card numbers in outbound
    HTTP responses from non-CDE services
  condition: >
    evt.type in (write, sendto) and
    fd.type = ipv4 and
    container and
    not k8s.ns.name in (payments) and
    (evt.buffer matches "(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13}|6(?:011|5[0-9]{2})[0-9]{12})")
  output: >
    CRITICAL: Potential credit card number detected in network traffic
    from non-CDE service (command=%proc.cmdline connection=%fd.name
    container=%container.name pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: CRITICAL
  tags: [pci, cardholder_data]
```

```yaml
# Prometheus alert — detect unexpected traffic patterns to Stripe's card-returning endpoints
# If order-svc starts making calls to Stripe endpoints it didn't call before
- alert: UnexpectedStripeAPICallPattern
  expr: |
    increase(http_client_requests_total{
      service="order-svc",
      target_host=~".*stripe.com.*",
      target_path=~".*(payment_methods|cards|tokens).*"
    }[1h]) > 0
    unless
    increase(http_client_requests_total{
      service="order-svc",
      target_host=~".*stripe.com.*",
      target_path=~".*(payment_methods|cards|tokens).*"
    }[1h] offset 7d) > 0
  for: 5m
  labels:
    severity: critical
    compliance: pci
  annotations:
    summary: "order-svc is calling Stripe card-data endpoints it didn't call last week"
    description: "Potential PCI scope expansion. Investigate immediately."
```

**Layer 5: Quarterly Architecture Review**

```text
PCI SCOPE REVIEW — QUARTERLY PROCESS
═══════════════════════════════════════
1. Data Flow Diagram audit: re-trace all flows involving payment data
2. Stripe integration audit: verify which API endpoints are called
3. Log audit: sample 1000 log lines from order-svc, search for PAN patterns
4. Network audit: inspect service mesh traffic between payment-adjacent services
5. New service review: any new service deployed in the last quarter that 
   touches the payment flow?
6. Output: Updated PCI scope document, signed by Security Lead and CTO
```

---

## Q2: SOC 2 Audit Evidence Gap 🔥

### 1. Analysis of Each Gap

**Gap 1: AWS Config Recorder Stopped for 12 Days in March**

**(a) Trust Service Criteria affected:** 
- **CC7.1 (Monitoring Infrastructure):** "The entity uses monitoring activities to identify changes to configurations that result in the introduction of new vulnerabilities."
- **CC6.1 (Logical Access Security):** Config tracks IAM and security group changes — without it, unauthorized access changes could go undetected.
- **CC8.1 (Change Management):** Config records infrastructure changes — gap means infrastructure changes during those 12 days are unauditable.

**(b) Severity:** **HIGH — potential qualified opinion (exception).** A 12-day gap in continuous monitoring is significant. The auditor cannot provide reasonable assurance that controls were operating effectively during that period. This is likely to result in a **qualification** (exception) in the SOC 2 report, not just an observation. The distinction: if the auditor believes the control was designed correctly but had a temporary failure in operating effectiveness, it's an exception. If they believe the control was never designed to be continuous, it's a deficiency — which is worse.

**(c) Compensating evidence:**
- CloudTrail logs for the 12-day period (CloudTrail was presumably still running — verify this FIRST)
- GuardDuty findings for the period (continuous, independent of Config)
- VPC Flow Logs for the period
- Terraform state file diffs showing what changed during that period
- Git history of infrastructure-as-code repositories
- Any screenshots or exports from Security Hub during that period
- AWS CloudWatch metrics showing the Config recorder status metric (proves when it stopped and restarted, which demonstrates you have monitoring — even though it failed to alert)

**Gap 2: No Access Review Evidence for Q2**

**(a) Trust Service Criteria affected:**
- **CC6.2 (Access Provisioning):** "Prior to issuing system credentials, the entity registers authorized users and validates their identity."
- **CC6.3 (Access Removal):** "The entity removes access to protected information assets when appropriate."
- **CC6.1:** Periodic access reviews are a core control for SOC 2 logical access.

**(b) Severity:** **MEDIUM — likely an exception.** A single missed quarterly review (out of 4) demonstrates the control exists but had an operational failure. The auditor will want to see that Q1, Q3, and Q4 reviews were completed. If those are clean, this is a point exception, not a systemic failure. If multiple quarters are missing, it becomes a deficiency.

**(c) Compensating evidence:**
- Q1, Q3, and Q4 access review artifacts (proves the control exists)
- CloudTrail logs showing no IAM changes during Q2 that would have been caught by a review (proves no actual harm)
- AWS IAM Access Analyzer findings for Q2 (continuous analysis even without the manual review)
- IAM credential reports generated during Q2
- Any JIT (just-in-time) access logs from your access management system
- The Lambda failure logs and CloudWatch alarm history (proves you had monitoring — it just failed to alert on the Lambda failure)

**Gap 3: SEV1 Incident in August with No Postmortem**

**(a) Trust Service Criteria affected:**
- **CC7.3 (Incident Response):** "The entity evaluates security events and determines if they are incidents."
- **CC7.4 (Incident Management):** "The entity responds to identified security incidents by executing a defined incident response program."
- **CC7.5 (Incident Recovery):** "The entity identifies, develops, and implements activities to recover from identified security incidents."

**(b) Severity:** **MEDIUM-HIGH — exception with management response required.** SEV1 incidents without postmortems indicate the incident response process is incomplete. The auditor will want to see: was the incident handled correctly in real-time (yes, presumably), but was the learning loop closed (no). This is likely an exception with a recommendation for remediation.

**(c) Compensating evidence:**
- Incident channel logs (Slack, PagerDuty timeline) showing the incident was detected, triaged, and resolved
- Any real-time notes or status updates during the incident
- JIRA/ticket showing the incident was tracked
- Write the postmortem NOW. It's late but better than nothing. Note in the document that it's being written retroactively and explain the process gap
- Evidence of postmortems for ALL OTHER SEV1/SEV2 incidents during the audit period (proves this is an exception, not a pattern)

**Gap 4: Security Hub Score Drop to 72% in July**

**(a) Trust Service Criteria affected:**
- **CC6.1 (Security):** Unencrypted EBS volume violates encryption-at-rest controls
- **CC7.1 (Monitoring):** The fact that it was detected and remediated within 30 days actually demonstrates that monitoring works
- **CC8.1 (Change Management):** The unencrypted volume was created by a developer — this suggests change control gaps

**(b) Severity:** **LOW — observation, not exception.** This is actually the BEST of the four findings from the auditor's perspective. Why? Because:
- Security Hub detected the non-compliant resource (control is working)
- The score recovered to 94% within a month (remediation happened)
- This demonstrates the "detect → respond → remediate" cycle working as designed
- The temporary dip shows a real environment, not a sanitized one

The auditor may note it as an observation with a recommendation to add preventive controls (SCP to block unencrypted EBS creation), but it's unlikely to result in a qualification.

**(c) Compensating evidence:**
- Security Hub findings showing the exact timeline: when the unencrypted volume was created, when it was detected, when it was remediated
- The remediation ticket/PR
- The corrective action taken (SCP to prevent unencrypted EBS creation going forward)
- Evidence that the developer was informed of the policy

---

### 2. Proving Compliance During the 12-Day Config Gap

**Before I provide evidence, I need to verify the integrity of my evidence sources:**

```bash
# FIRST: Verify CloudTrail was logging during those 12 days
aws cloudtrail get-trail-status --name novamart-production-trail
# Check: LatestDeliveryTime, IsLogging

# Verify CloudTrail log file integrity for the March period
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:888888888888:trail/novamart-production-trail \
  --start-time "2024-03-01T00:00:00Z" \
  --end-time "2024-03-31T23:59:59Z"
# This validates digest files — proves logs weren't tampered with
```

**Evidence Package for the Auditor:**

**Evidence 1: CloudTrail — Complete API Call Record**

```bash
# Export ALL CloudTrail events during the 12-day gap
aws cloudtrail lookup-events \
  --start-time "2024-03-05T00:00:00Z" \
  --end-time "2024-03-17T23:59:59Z" \
  --max-results 1000 \
  --output json > march-gap-cloudtrail.json

# Specifically show: no unauthorized IAM changes
aws cloudtrail lookup-events \
  --start-time "2024-03-05T00:00:00Z" \
  --end-time "2024-03-17T23:59:59Z" \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=iam.amazonaws.com \
  --output json > march-gap-iam-events.json

# Show: no security group changes
aws cloudtrail lookup-events \
  --start-time "2024-03-05T00:00:00Z" \
  --end-time "2024-03-17T23:59:59Z" \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress \
  --output json > march-gap-sg-events.json
```

Present to auditor: "CloudTrail provides a complete API-level audit trail for the 12-day period. Here are all IAM changes, security group changes, and infrastructure modifications. CloudTrail log integrity is validated via digest files."

**Evidence 2: GuardDuty — Continuous Threat Monitoring**

```bash
# Show GuardDuty was active during the gap
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --finding-criteria '{
    "Criterion": {
      "createdAt": {
        "GreaterThanOrEqual": 1709596800000,
        "LessThanOrEqual": 1710633600000
      }
    }
  }'
# Ideally: zero findings (no threats detected)
# If findings exist: show they were handled
```

Present to auditor: "GuardDuty operated continuously during the Config gap, providing threat detection independent of Config. No critical findings were generated during this period."

**Evidence 3: Terraform State and Git History**

```bash
# Show what infrastructure changes were made during the gap
git log --after="2024-03-05" --before="2024-03-17" \
  --oneline -- 'terraform/' 'infrastructure/'

# Show Terraform state changes
# (Terraform Cloud/S3 state has versioning)
aws s3api list-object-versions \
  --bucket novamart-terraform-state \
  --prefix production/ \
  --query "Versions[?LastModified>=\`2024-03-05\` && LastModified<=\`2024-03-17\`]"
```

Present to auditor: "All infrastructure changes are managed through Terraform with state stored in S3 with versioning. Here are all changes made during the 12-day period, each traceable to a git commit and pull request."

**Evidence 4: Security Hub Historical Findings**

```bash
# Security Hub findings for the gap period
aws securityhub get-findings \
  --filters '{
    "CreatedAt":[
      {
        "Start": "2024-03-05T00:00:00Z",
        "End": "2024-03-17T23:59:59Z"
      }
    ]
  }' \
  --sort-criteria '{"Field": "CreatedAt", "SortOrder": "asc"}' \
  --max-results 100
```

**Evidence 5: The Config Gap Explanation**

```markdown
## Config Recorder Gap — Incident Report

**Duration:** March 5–17, 2024 (12 days)
**Root Cause:** During a Terraform state migration from S3 backend 
to Terraform Cloud, the Config recorder resource was temporarily 
removed from state and re-imported. The re-import failed silently, 
leaving the recorder in a stopped state.

**Detection:** Detected on March 17 during routine compliance dashboard review.
**Remediation:** Recorder restarted within 2 hours of detection. 
Full Config evaluation ran on restart, producing a complete snapshot 
of all resource compliance as of March 17.

**Corrective Actions Taken:**
1. CloudWatch alarm added for Config recorder status (see Evidence 6)
2. Terraform pipeline now validates Config recorder status post-apply
3. Weekly automated compliance check verifies all monitoring tools are active

**Compensating Controls Active During Gap:**
- CloudTrail: ✅ Active (verified via log digest validation)
- GuardDuty: ✅ Active (zero critical findings)
- Security Hub: ✅ Active (receiving findings from other sources)
- VPC Flow Logs: ✅ Active
```

---

### 3. Monitoring and Alerting to Catch All Four Gaps

**Alert 1: Config Recorder Stopped**

```yaml
# CloudWatch Alarm — fires immediately when Config recorder stops
resource "aws_cloudwatch_metric_alarm" "config_recorder_stopped" {
  alarm_name          = "config-recorder-stopped-CRITICAL"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ConfigurationRecorderStatus"
  namespace           = "AWS/Config"
  period              = 300     # 5 minutes
  statistic           = "Minimum"
  threshold           = 1
  alarm_description   = "AWS Config recorder has stopped. SOC2 CC7.1 control gap."
  alarm_actions       =[
    aws_sns_topic.security_critical.arn,
    aws_sns_topic.pagerduty_security.arn
  ]
  treat_missing_data  = "breaching"  # If no data, assume it's stopped
}
```

```bash
# Additionally: Lambda that checks Config recorder status every hour
# More reliable than CloudWatch metrics which can have gaps

aws configservice describe-configuration-recorder-status \
  --query 'ConfigurationRecordersStatus[0].recording'
# Must return: true
```

```python
# Lambda function — hourly Config health check
import boto3

def handler(event, context):
    config = boto3.client('config')
    status = config.describe_configuration_recorder_status()
    
    for recorder in status['ConfigurationRecordersStatus']:
        if not recorder['recording']:
            # CRITICAL: Config is not recording
            sns = boto3.client('sns')
            sns.publish(
                TopicArn=SECURITY_TOPIC,
                Subject='🔴 CRITICAL: AWS Config Recorder STOPPED',
                Message=f"""
AWS Config recorder '{recorder["name"]}' is NOT recording.
Last status: {recorder.get("lastStatus", "UNKNOWN")}
Last status change: {recorder.get("lastStatusChangeTime", "UNKNOWN")}

This is a SOC 2 compliance gap. Every minute Config is stopped,
infrastructure changes are not being recorded.

ACTION REQUIRED: Restart Config recorder immediately.
aws configservice start-configuration-recorder --configuration-recorder-name {recorder["name"]}
                """
            )
            
            # Also try to auto-remediate
            try:
                config.start_configuration_recorder(
                    ConfigurationRecorderName=recorder['name']
                )
                sns.publish(
                    TopicArn=SECURITY_TOPIC,
                    Subject='✅ AWS Config Recorder AUTO-RESTARTED',
                    Message=f"Config recorder auto-restarted by compliance Lambda."
                )
            except Exception as e:
                sns.publish(
                    TopicArn=SECURITY_TOPIC,
                    Subject='⛔ AWS Config Recorder RESTART FAILED',
                    Message=f"Auto-restart failed: {e}. Manual intervention required."
                )
```

**Alert 2: Access Review Lambda Failed**

```yaml
# CloudWatch Alarm — detect Lambda failure
resource "aws_cloudwatch_metric_alarm" "access_review_lambda_failed" {
  alarm_name          = "access-review-lambda-failed"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 86400   # 24 hours
  statistic           = "Sum"
  threshold           = 0
  dimensions = {
    FunctionName = "quarterly-access-review"
  }
  alarm_description   = "Access review Lambda failed. SOC2 CC6.2 evidence at risk."
  alarm_actions       =[aws_sns_topic.security_alerts.arn]
}

# ALSO: Alarm on missing invocation — catches "Lambda never ran"
resource "aws_cloudwatch_metric_alarm" "access_review_not_run" {
  alarm_name          = "access-review-not-executed"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "Invocations"
  namespace           = "AWS/Lambda"
  period              = 7776000  # 90 days (quarterly)
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "Access review Lambda has not run this quarter."
  alarm_actions       =[aws_sns_topic.security_alerts.arn]
  treat_missing_data  = "breaching"
}
```

```python
# Better approach: a "compliance heartbeat" system
# After each compliance task completes, it writes a heartbeat
# A separate monitor checks that all heartbeats are current

COMPLIANCE_TASKS = {
    'access_review': {'frequency_days': 90, 'owner': 'security-team'},
    'config_recorder_check': {'frequency_days': 1, 'owner': 'platform-team'},
    'incident_postmortem_audit': {'frequency_days': 30, 'owner': 'sre-team'},
    'security_hub_review': {'frequency_days': 7, 'owner': 'security-team'},
}

def check_compliance_heartbeats(event, context):
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('compliance-heartbeats')
    
    for task, config in COMPLIANCE_TASKS.items():
        response = table.get_item(Key={'task_name': task})
        if 'Item' not in response:
            alert(f"Compliance task '{task}' has NEVER run")
            continue
        
        last_run = datetime.fromisoformat(response['Item']['last_run'])
        days_since = (datetime.utcnow() - last_run).days
        
        if days_since > config['frequency_days']:
            alert(
                f"Compliance task '{task}' is OVERDUE. "
                f"Last run: {days_since} days ago. "
                f"Required frequency: every {config['frequency_days']} days. "
                f"Owner: {config['owner']}"
            )
```

**Alert 3: SEV1 Incident Without Postmortem**

```python
# Integration with incident management (PagerDuty/Jira)
# When a SEV1 incident is resolved, auto-create a postmortem ticket
# Alert if the ticket isn't completed within 5 business days

def on_incident_resolved(event):
    if event['severity'] == 'SEV1':
        # Create postmortem ticket
        jira.create_issue(
            project='POSTMORTEM',
            summary=f"Postmortem Required: {event['title']}",
            description=f"""
            Incident: {event['id']}
            Resolved: {event['resolved_at']}
            
            Postmortem MUST be completed within 5 business days.
            Template: https://wiki.novamart.internal/postmortem-template
            
            This is a SOC 2 requirement (CC7.4).
            """,
            due_date=business_days_from_now(5),
            assignee=event['incident_commander']
        )
        
        # Schedule a follow-up check
        schedule_check(
            task='postmortem_completion_check',
            incident_id=event['id'],
            check_date=business_days_from_now(6)
        )

def postmortem_completion_check(incident_id):
    ticket = jira.search(f"project=POSTMORTEM AND labels=incident-{incident_id}")
    if ticket.status != 'Done':
        alert(
            f"🔴 COMPLIANCE: SEV1 incident {incident_id} postmortem is OVERDUE. "
            f"Ticket: {ticket.key}. Assigned to: {ticket.assignee}. "
            f"This is a SOC 2 CC7.4 requirement."
        )
        # Escalate to Engineering Director
        notify(ENGINEERING_DIRECTOR, message=f"Overdue postmortem: {ticket.key}")
```

**Alert 4: Security Hub Compliance Score Drop**

```yaml
# EventBridge rule — fires when Security Hub compliance status changes
resource "aws_cloudwatch_event_rule" "security_hub_compliance_drop" {
  name        = "security-hub-compliance-score-drop"
  description = "Detect significant drop in Security Hub compliance score"
  
  event_pattern = jsonencode({
    source      = ["aws.securityhub"]
    detail-type = ["Security Hub Findings - Imported"]
    detail = {
      findings = {
        Compliance = {
          Status = ["FAILED"]
        }
        Severity = {
          Label = ["CRITICAL", "HIGH"]
        }
      }
    }
  })
}
```

```yaml
# Prometheus alert — aggregate compliance score monitoring
- alert: SecurityHubComplianceScoreLow
  expr: |
    aws_securityhub_compliance_score < 90
  for: 30m
  labels:
    severity: warning
    compliance: soc2
  annotations:
    summary: "Security Hub compliance score dropped below 90%"
    description: "Current score: {{ $value }}%. Investigate failed checks in Security Hub console."

- alert: SecurityHubComplianceScoreCritical
  expr: |
    aws_securityhub_compliance_score < 80
  for: 10m
  labels:
    severity: critical
    compliance: soc2
  annotations:
    summary: "Security Hub compliance score critically low: {{ $value }}%"
```

---

### 4. Responding to "Compliance Is Not a Continuous Priority"

**This is the most important question in Q2 because it's not a technical problem — it's a credibility problem.**

The auditor's comment is an assessment of organizational maturity, not just technical controls. A purely technical response ("we added more alerts") will reinforce their concern. The response must address BOTH dimensions.

---

**Part 1: Technical Evidence of Continuous Compliance**

> "I appreciate the directness, and I want to address this head-on with both evidence and actions.
>
> First, the technical evidence that compliance IS monitored continuously:
>
> **Controls that operated without interruption during the entire audit period:**
> - CloudTrail: 365/365 days active, log integrity validated via digest files for every day of the audit period [Exhibit A: CloudTrail validation report]
> - GuardDuty: 365/365 days active, $TOTAL findings generated, $CRITICAL resolved within SLA [Exhibit B: GuardDuty finding timeline]
> - Security Hub: Active throughout the period, with the compliance score tracked weekly [Exhibit C: Security Hub score trend graph]
> - Automated image scanning (ECR + Trivy): Every deployment scanned, zero critical CVEs deployed to production [Exhibit D: scan results]
> - VPC Flow Logs: Continuous, no gaps
> - Kubernetes audit logs: Continuous, no gaps
>
> **The four gaps you identified are real. Here's the pattern:**
> - Config recorder: Operational failure during a migration (process gap, not design gap)
> - Access review: Automation failure with insufficient monitoring (monitoring gap)
> - Postmortem: Process adherence failure (culture gap)
> - Security Hub score: Detection worked correctly; developer created non-compliant resource, it was detected and remediated within 30 days (this is actually the control WORKING)"

**Part 2: Organizational Commitment**

> "You're right that four gaps in a single audit period indicate a systemic issue, even if each individual gap has a reasonable explanation. Here's what we've done and are doing:
>
> **Corrective Actions Already Implemented:**
>
> 1. **Compliance Heartbeat System** [Exhibit E]: Every compliance control now reports a 'heartbeat' to a central DynamoDB table. A daily Lambda checks all heartbeats and alerts if any control hasn't reported within its expected frequency. This catches the Config and access review gaps proactively.
>
> 2. **Automated Postmortem Enforcement**[Exhibit F]: SEV1 incidents now auto-generate a postmortem ticket with a 5-business-day deadline. If not completed, it escalates to the Engineering Director. The system has been active since September — 4 of 4 postmortems completed on time since implementation.
>
> 3. **Compliance Dashboard**[Exhibit G]: Real-time Grafana dashboard showing the status of every SOC 2 control. Reviewed weekly by the Security team and monthly by the VP of Engineering. I can show you the meeting notes and attendance records.
>
> 4. **SCP Guardrails** [Exhibit H]: Preventive controls added — unencrypted EBS volumes can no longer be created in production accounts. The developer who created the unencrypted volume would now be blocked at the API level.
>
> **Organizational Changes:**
>
> 5. Compliance review is now a standing agenda item in the weekly engineering leadership meeting [Exhibit I: meeting minutes].
>
> 6. Every engineer's onboarding includes a 2-hour compliance training module covering SOC 2 obligations [Exhibit J: training records].
>
> 7. The quarterly access review now has a backup — if the Lambda fails, a calendar reminder triggers a manual review within 5 business days[Exhibit K: calendar entries and manual review records for Q3 and Q4].
>
> **My honest assessment:** The March Config gap was a genuine process failure that we should have caught faster. The access review gap was an automation failure compounded by insufficient monitoring. The postmortem gap was a culture gap that we've since addressed. The Security Hub dip was actually our controls working correctly — we detected the issue and fixed it.
>
> I won't claim we're perfect. But I can demonstrate that each gap was identified, root-caused, and resulted in a systemic improvement. Compliance IS a continuous priority — these four items represent the exceptions that prove the rule, and each exception has made our compliance posture stronger."

---

## Q3: Multi-Framework Compliance Conflict 🔥

### 1. Resolving the 30-Day Deletion vs 7-Year Retention Conflict

**This is one of the most common compliance conflicts in practice. The resolution requires data classification and architectural separation.**

**The key insight:** GDPR's right to erasure applies to **personal data** (data that identifies a natural person). Financial retention requirements apply to **transaction records**. The solution is to separate the two:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DATA CLASSIFICATION                            │
├─────────────────────────────────────────────────────────────────────────────┤
│ PERSONAL DATA (GDPR-deletable):                                             │
│   - Customer name                                                           │
│   - Email address                                                           │
│   - Phone number                                                            │
│   - Shipping address                                                        │
│   - IP address                                                              │
│   - Device fingerprint                                                      │
│   - Browsing/search history                                                 │
│   - Customer account profile                                                │
│                                                                             │
│ TRANSACTION RECORDS (Retention-required):                                   │
│   - Order ID                                                                │
│   - Order date                                                              │
│   - Product SKUs and quantities                                             │
│   - Transaction amount                                                      │
│   - Payment reference (Stripe token — NOT card data)                        │
│   - Tax calculations                                                        │
│   - Invoice number                                                          │
│                                                                             │
│ DUAL-CATEGORY (requires special handling):                                  │
│   - Shipping address ON an order (personal data + transaction record)       │
│   - Customer name ON an invoice (personal data + financial record)          │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Technical Architecture:**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DATA ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ ┌───────────────┐           ┌───────────────────┐                           │
│ │ CUSTOMER DB   │           │ TRANSACTION DB    │                           │
│ │ (Deletable)   │──────────►│ (7-year retain)   │                           │
│ │               │           │                   │                           │
│ │ customer_id   │  FK:      │ order_id          │                           │
│ │ name          │  customer │ customer_ref*     │                           │
│ │ email         │  _id      │ amount            │                           │
│ │ phone         │           │ product_skus      │                           │
│ │ address       │           │ tax               │                           │
│ │ preferences   │           │ stripe_pi_id      │                           │
│ │               │           │ anonymized_addr** │                           │
│ │ TTL: Delete   │           │ invoice_number    │                           │
│ │ on request    │           │                   │                           │
│ │               │           │ TTL: 7 years      │                           │
│ └───────────────┘           └───────────────────┘                           │
│                                                                             │
│ * customer_ref = pseudonymized ID (not reversible                           │
│   after deletion)                                                           │
│ ** anonymized_addr = country + postal code prefix only                      │
│    (sufficient for tax/audit, not personally identifiable)                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**GDPR Deletion Process:**

```python
def handle_gdpr_deletion_request(customer_id):
    """
    Process a GDPR Article 17 (Right to Erasure) request.
    Deletes personal data while preserving anonymized transaction records.
    """
    
    # Step 1: Verify the request is legitimate
    customer = customer_db.get(customer_id)
    if not customer:
        raise NotFoundError("Customer not found")
    
    # Step 2: Generate a pseudonymized reference for transaction records
    # This is a one-way hash — cannot be reversed to identify the customer
    pseudo_ref = sha256(f"{customer_id}:{DELETION_SALT}").hexdigest()[:16]
    
    # Step 3: Anonymize transaction records (keep for 7-year retention)
    transactions = transaction_db.query(customer_id=customer_id)
    for txn in transactions:
        txn.customer_ref = pseudo_ref          # Replace customer ID with hash
        txn.shipping_name = "REDACTED"         # Remove name
        txn.shipping_address = anonymize_address(txn.shipping_address)
        # Keep: country, postal code prefix (for tax compliance)
        # Remove: street, city, full postal code
        txn.customer_email = None              # Remove email
        txn.customer_phone = None              # Remove phone
        transaction_db.save(txn)
    
    # Step 4: Delete all personal data
    customer_db.delete(customer_id)
    
    # Step 5: Delete from all secondary stores
    redis_cache.delete(f"customer:{customer_id}")
    elasticsearch.delete_by_query(index="customers", body={
        "query": {"term": {"customer_id": customer_id}}
    })
    
    # Step 6: Delete from S3 (profile images, documents)
    s3.delete_objects(
        Bucket='customer-data',
        Delete={'Objects':[
            {'Key': f'customers/{customer_id}/'}  # Delete entire prefix
        ]}
    )
    
    # Step 7: Request deletion from third parties
    stripe.customers.delete(customer.stripe_customer_id)
    sendgrid.contacts.delete(customer.email)
    
    # Step 8: Audit trail (GDPR requires proof of deletion)
    audit_log.record({
        'action': 'GDPR_DELETION',
        'customer_pseudo_ref': pseudo_ref,  # NOT the original customer_id
        'timestamp': datetime.utcnow(),
        'data_deleted':['customer_db', 'cache', 'search', 's3', 'stripe', 'sendgrid'],
        'transaction_records_anonymized': len(transactions),
        'completed_within_days': 1  # Must be <30
    })
    
    return {
        'status': 'completed',
        'pseudo_ref': pseudo_ref,
        'records_anonymized': len(transactions)
    }

def anonymize_address(address):
    """Keep only what's needed for tax/financial compliance."""
    return {
        'country': address.get('country'),
        'postal_prefix': address.get('postal_code', '')[:3],
        'street': None,
        'city': None,
        'state': address.get('state'),  # Needed for US tax calculations
    }
```

**Why this satisfies BOTH requirements:**

- **GDPR:** All personal data (name, email, phone, full address, browsing history) is deleted within 30 days. The remaining transaction records contain only a pseudonymized reference that cannot be reversed to identify the person. Under GDPR Recital 26, data that cannot identify a natural person is no longer personal data.

- **7-Year Financial Retention:** Transaction records are preserved with all financially relevant information: amounts, dates, product details, tax calculations, invoice numbers. The anonymized address retains enough detail for tax jurisdiction validation. Auditors can verify financial records without being able to identify individual customers.

- **The key legal principle:** GDPR Article 17(3)(b) explicitly exempts data that must be retained for "compliance with a legal obligation." However, this exemption applies narrowly — you retain only what's legally required for the financial regulation and delete everything else. The anonymization approach is the technical implementation of this legal principle.

---

### 2. EU Data Residency vs US-Based Observability Stack

**Yes, NovaMart is potentially in violation.** GDPR Article 44 restricts transfers of personal data outside the EU/EEA. If application logs from the EU cluster contain customer identifiers (order IDs, email addresses in error messages) and these logs are shipped to Prometheus/Loki/Tempo in us-east-1, that constitutes a transfer of personal data to a third country.

**Verify the severity first:**

```bash
# Check: what personal data is in the logs?
# Sample EU cluster logs for PII patterns
kubectl logs -n orders deploy/order-svc --since=1h -c order-svc | \
  grep -iE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' | head -20

kubectl logs -n orders deploy/order-svc --since=1h -c order-svc | \
  grep -iP '\b\d{5,}\b' | head -20  # Order IDs that could be linked to customers

kubectl logs -n users deploy/user-svc --since=1h -c user-svc | \
  grep -iE 'customer|user.*id|email|name|address' | head -20
```

**If personal data is found in logs (which it almost certainly is), the fix has three layers:**

**Fix 1: Deploy a separate observability stack in eu-west-1**

```hcl
# Dedicated monitoring namespace in the EU cluster
# Prometheus, Loki, Tempo, Grafana — all in eu-west-1

module "eu_observability" {
  source = "../modules/observability-stack"
  
  region             = "eu-west-1"
  cluster_name       = "novamart-eu-prod"
  prometheus_retention = "30d"
  loki_retention     = "90d"
  tempo_retention    = "14d"
  
  # S3 bucket for long-term storage — MUST be in eu-west-1
  storage_bucket     = aws_s3_bucket.eu_observability.id
  
  # Grafana can be accessed from us-east-1 for unified dashboards
  # BUT: Grafana queries data in eu-west-1, data never leaves EU
  grafana_cross_region_access = true
}

resource "aws_s3_bucket" "eu_observability" {
  bucket = "novamart-eu-observability"
  
  # CRITICAL: Bucket must be in eu-west-1
  # No cross-region replication to US regions
}
```

**Fix 2: Scrub PII from logs before ingestion**

```yaml
# Promtail / Fluentd / Vector pipeline stage — strip PII before shipping

# Vector configuration (runs as DaemonSet in EU cluster)
[transforms.strip_pii]
  type = "remap"
  inputs =["kubernetes_logs"]
  source = '''
    # Redact email addresses
    .message = replace(.message, r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}', "[EMAIL_REDACTED]")
    
    # Redact customer names (if structured logging with known fields)
    if exists(.customer_name) {
      .customer_name = "REDACTED"
    }
    
    # Redact phone numbers
    .message = replace(.message, r'\+?[0-9]{10,15}', "[PHONE_REDACTED]")
    
    # Redact IP addresses
    .message = replace(.message, r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b', "[IP_REDACTED]")
    
    # Preserve order IDs but hash them (for correlation without PII)
    if exists(.order_id) {
      .order_id_hash = sha2(.order_id, variant: "SHA-256")
      del(.order_id)
    }
  '''

# If logs MUST be shipped to us-east-1 for unified monitoring,
# they must be scrubbed BEFORE leaving eu-west-1[sinks.us_east_loki]
  type = "loki"
  inputs = ["strip_pii"]  # Only scrubbed logs go to US
  endpoint = "https://loki.us-east-1.novamart.internal"
```

**Fix 3: Grafana federation — query in EU, view from US**

```yaml
# Grafana in us-east-1 can query Prometheus/Loki in eu-west-1
# Data stays in EU; only query results (aggregated metrics, not raw PII) cross the border

# Grafana datasource configuration
apiVersion: 1
datasources:
  - name: EU-Prometheus
    type: prometheus
    url: https://prometheus.eu-west-1.novamart.internal
    access: proxy  # Grafana server proxies the query
    jsonData:
      httpHeaderName1: Authorization
    secureJsonData:
      httpHeaderValue1: Bearer ${EU_PROMETHEUS_TOKEN}
    
  - name: EU-Loki
    type: loki
    url: https://loki.eu-west-1.novamart.internal
    access: proxy
```

**The aggregated metrics (request rates, error rates, latency percentiles) that Grafana dashboards display are NOT personal data** — they're statistical summaries. A query result like "p99 latency for order-svc in eu-west-1 is 245ms" contains no PII and can be viewed from any region.

Raw logs with customer identifiers MUST stay in eu-west-1 and be accessed by EU-based Grafana/Loki only.

---

### 3. BYOK (Bring Your Own Key) for a Single Customer in Multi-Tenant

**Architecture: Per-Customer KMS Key with Envelope Encryption**

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              BYOK ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ Customer's AWS Account                NovaMart's AWS Account                │
│ ──────────────────────                ──────────────────────                │
│                                                                             │
│ ┌────────────────────┐                ┌──────────────────────────┐          │
│ │ Customer's KMS     │                │ NovaMart's KMS           │          │
│ │ Key (CMK)          │◄──────────────►│ (for all OTHER customers)│          │
│ │                    │ Cross-account  │                          │          │
│ │ Customer controls  │ grant          │                          │          │
│ │ key policy         │                │                          │          │
│ └─────────┬──────────┘                └─────────────┬────────────┘          │
│           │ Encrypts this                           │ Encrypts this         │
│           │ customer's data                         │ customer's data       │
│           ▼                                         ▼                       │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │                       RDS / S3 / ElastiCache                            │ │
│ │                                                                         │ │
│ │ Tenant A data: encrypted with Customer A's key                          │ │
│ │ Tenant B data: encrypted with NovaMart's key                            │ │
│ │ Tenant C data: encrypted with NovaMart's key                            │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Implementation for RDS:**

RDS encrypts the entire database instance with a single KMS key — you can't encrypt individual rows with different keys. So for true BYOK per-customer, you need **application-level encryption** on top of RDS's instance-level encryption.

```python
# Application-level envelope encryption per customer
# NovaMart's data access layer wraps all reads/writes for BYOK customers

import boto3
from cryptography.fernet import Fernet
import base64

class BYOKEncryptionService:
    """
    Per-customer envelope encryption using customer-owned KMS keys.
    
    Architecture:
    1. Customer provides their KMS key ARN (from their AWS account)
    2. Customer grants NovaMart's IAM role permission to use the key
       for Encrypt and Decrypt operations
    3. NovaMart generates a data encryption key (DEK) using customer's CMK
    4. DEK encrypts customer's data at the application level
    5. Encrypted DEK is stored alongside the encrypted data
    6. Customer can revoke access at any time by removing the KMS grant
    """
    
    def __init__(self):
        self.kms = boto3.client('kms')
        self._dek_cache = {}  # Cache DEKs in memory (short TTL)
    
    def get_customer_key_config(self, customer_id):
        """Look up whether this customer uses BYOK and which key."""
        config = db.query(CustomerEncryptionConfig).get(customer_id)
        if config and config.byok_enabled:
            return {
                'byok': True,
                'kms_key_arn': config.customer_kms_key_arn,
                # e.g., arn:aws:kms:eu-west-1:CUSTOMER_ACCOUNT:key/mrk-xxx
            }
        return {
            'byok': False,
            'kms_key_arn': NOVAMART_DEFAULT_KMS_KEY_ARN
        }
    
    def encrypt_field(self, customer_id, plaintext):
        """Encrypt a single field using the customer's key."""
        key_config = self.get_customer_key_config(customer_id)
        
        # Generate a data encryption key using the customer's CMK
        # (or NovaMart's default key if not BYOK)
        response = self.kms.generate_data_key(
            KeyId=key_config['kms_key_arn'],
            KeySpec='AES_256',
            EncryptionContext={
                'customer_id': customer_id,
                'service': 'novamart-order-svc',
                'purpose': 'field-encryption'
            }
        )
        
        # Use the plaintext DEK to encrypt the data
        plaintext_dek = response['Plaintext']
        encrypted_dek = response['CiphertextBlob']
        
        fernet = Fernet(base64.urlsafe_b64encode(plaintext_dek[:32]))
        encrypted_data = fernet.encrypt(plaintext.encode())
        
        # Return both: encrypted data + encrypted DEK
        # The DEK can only be decrypted with the customer's KMS key
        return {
            'encrypted_data': base64.b64encode(encrypted_data).decode(),
            'encrypted_dek': base64.b64encode(encrypted_dek).decode(),
            'key_arn': key_config['kms_key_arn'],
            'encryption_context': {
                'customer_id': customer_id,
                'service': 'novamart-order-svc',
                'purpose': 'field-encryption'
            }
        }
    
    def decrypt_field(self, customer_id, encrypted_record):
        """Decrypt a field using the customer's key."""
        encrypted_dek = base64.b64decode(encrypted_record['encrypted_dek'])
        
        # Ask KMS to decrypt the DEK using the customer's CMK
        response = self.kms.decrypt(
            CiphertextBlob=encrypted_dek,
            EncryptionContext=encrypted_record['encryption_context']
        )
        
        plaintext_dek = response['Plaintext']
        fernet = Fernet(base64.urlsafe_b64encode(plaintext_dek[:32]))
        
        encrypted_data = base64.b64decode(encrypted_record['encrypted_data'])
        return fernet.decrypt(encrypted_data).decode()


# Usage in the order service:
byok = BYOKEncryptionService()

# Writing an order for a BYOK customer
def create_order(customer_id, order_data):
    encrypted_shipping = byok.encrypt_field(
        customer_id, 
        json.dumps(order_data['shipping_address'])
    )
    encrypted_email = byok.encrypt_field(
        customer_id,
        order_data['email']
    )
    
    order = Order(
        order_id=generate_order_id(),
        customer_id=customer_id,
        amount=order_data['amount'],           # NOT encrypted (financial record)
        product_skus=order_data['skus'],       # NOT encrypted (financial record)
        shipping_address_enc=json.dumps(encrypted_shipping),  # Encrypted with customer key
        email_enc=json.dumps(encrypted_email),                # Encrypted with customer key
        encryption_version='v1',
        byok_enabled=True
    )
    db.save(order)
```

**Customer's KMS Key Policy (in their account):**

```json
{
    "Version": "2012-10-17",
    "Statement":[
        {
            "Sid": "AllowCustomerAdminAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::CUSTOMER_ACCOUNT:root"
            },
            "Action": "kms:*",
            "Resource": "*"
        },
        {
            "Sid": "AllowNovaMartEncryptDecrypt",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::888888888888:role/novamart-order-svc-irsa"
            },
            "Action":[
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:GenerateDataKey",
                "kms:DescribeKey"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "kms:EncryptionContext:service": "novamart-order-svc",
                    "kms:EncryptionContext:customer_id": "CUSTOMER_ID"
                },
                "StringLike": {
                    "kms:ViaService":[
                        "rds.eu-west-1.amazonaws.com",
                        "s3.eu-west-1.amazonaws.com"
                    ]
                }
            }
        },
        {
            "Sid": "CustomerCanRevokeAtAnyTime",
            "Effect": "Deny",
            "Principal": {
                "AWS": "arn:aws:iam::888888888888:root"
            },
            "Action":[
                "kms:CreateGrant",
                "kms:PutKeyPolicy",
                "kms:RetireGrant"
            ],
            "Resource": "*"
        }
    ]
}
```

**Critical property:** The customer can revoke NovaMart's access at ANY time by removing the `AllowNovaMartEncryptDecrypt` statement from their key policy. When they do:
- NovaMart can still read the encrypted blobs from the database
- NovaMart CANNOT decrypt them (KMS will deny the Decrypt call)
- The data is effectively cryptographically deleted from NovaMart's perspective
- The customer retains the ability to decrypt if they ever need the data back

**For S3 (document storage):**

```hcl
# S3 bucket for the EU customer uses their KMS key for SSE-KMS
resource "aws_s3_bucket_server_side_encryption_configuration" "eu_customer_docs" {
  bucket = aws_s3_bucket.customer_documents.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.customer_kms_key_arn  # Customer's key
    }
    bucket_key_enabled = true  # Reduces KMS API costs
  }
}

# S3 bucket policy — enforce encryption with the customer's key for their prefix
resource "aws_s3_bucket_policy" "enforce_customer_key" {
  bucket = aws_s3_bucket.customer_documents.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement =[
      {
        Sid       = "EnforceCustomerBYOK"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.customer_documents.arn}/customers/${var.customer_id}/*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption-aws-kms-key-id" = var.customer_kms_key_arn
          }
        }
      }
    ]
  })
}
```

**For multi-region BYOK (customer wants key in eu-west-1):**

```hcl
# Customer should create a Multi-Region KMS key
# The primary key is in eu-west-1, replica can be in eu-west-2 for DR
# NovaMart accesses the key via the eu-west-1 endpoint

# NovaMart's IAM role needs cross-account KMS access
resource "aws_iam_role_policy" "byok_cross_account_kms" {
  name = "byok-customer-kms-access"
  role = aws_iam_role.order_svc_irsa.name
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action =[
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:GenerateDataKey",
          "kms:DescribeKey"
        ]
        Resource =[
          "arn:aws:kms:eu-west-1:CUSTOMER_ACCOUNT:key/*"
        ]
        Condition = {
          StringEquals = {
            "kms:EncryptionContext:customer_id" = var.customer_id
          }
        }
      }
    ]
  })
}
```

---

### 4. SCP for EU Data Residency Enforcement

```hcl
resource "aws_organizations_policy" "eu_data_residency" {
  name        = "eu-data-residency-enforcement"
  description = "Restricts EU production account to eu-west-1 region only, with exceptions for global services"
  type        = "SERVICE_CONTROL_POLICY"
  
  content = jsonencode({
    Version = "2012-10-17"
    Statement =[
      {
        Sid    = "DenyNonEURegions"
        Effect = "Deny"
        Action = "*"
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" =[
              "eu-west-1",
              "eu-west-2"    # DR region within EU
            ]
          }
          # Exceptions for global services that don't have regional endpoints
          # These services operate globally but data stays in configured region
          ArnNotLike = {
            "aws:PrincipalARN" =[
              "arn:aws:iam::*:role/OrganizationAdminRole"
            ]
          }
        }
      },
      {
        Sid    = "AllowGlobalServices"
        Effect = "Allow"
        Action =[
          # IAM is global — must be allowed from any region
          "iam:*",
          # STS is global
          "sts:*",
          # Organizations is global
          "organizations:*",
          # Route53 is global
          "route53:*",
          # CloudFront is global (edge locations are worldwide,
          # but origin must be in EU — enforced separately)
          "cloudfront:*",
          # WAF global (for CloudFront)
          "wafv2:*",
          # ACM for CloudFront (must be us-east-1 for CF, but
          # the cert is just metadata — no customer data)
          "acm:*",
          # Support is global
          "support:*",
          # Billing is global
          "ce:*",
          "cur:*",
          # Health/Trusted Advisor
          "health:*",
          "trustedadvisor:*"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyS3CrossRegionReplication"
        Effect = "Deny"
        Action = [
          "s3:PutReplicationConfiguration"
        ]
        Resource = "*"
        Condition = {
          StringNotLike = {
            # Only allow replication to other EU regions
            "s3:ReplicationDestination" =[
              "arn:aws:s3:::*-eu-*",
              "arn:aws:s3:::novamart-eu-*"
            ]
          }
        }
      },
      {
        Sid    = "DenyRDSSnapshotCopyOutsideEU"
        Effect = "Deny"
        Action =[
          "rds:CopyDBSnapshot",
          "rds:CopyDBClusterSnapshot"
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "rds:DatabaseEngine" = "aurora-postgresql"
          }
          # This condition is tricky — RDS snapshot copy region
          # is determined by the API endpoint, not a parameter.
          # The region restriction in statement 1 handles this
          # since the copy API call goes to the DESTINATION region
        }
      },
      {
        Sid    = "DenyEBSSnapshotCopyOutsideEU"
        Effect = "Deny"
        Action =[
          "ec2:CopySnapshot"
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "ec2:Region" =[
              "eu-west-1",
              "eu-west-2"
            ]
          }
        }
      },
      {
        Sid    = "DenyKinesisOutsideEU"
        Effect = "Deny"
        Action =[
          "kinesis:*",
          "firehose:*"
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" =[
              "eu-west-1",
              "eu-west-2"
            ]
          }
        }
      },
      {
        Sid    = "DenyCloudWatchCrossRegionExport"
        Effect = "Deny"
        Action = [
          "logs:PutDestination",
          "logs:CreateExportTask"
        ]
        Resource = "*"
        Condition = {
          StringNotLike = {
            # Only allow log exports to EU S3 buckets
            "logs:DestinationArn" =[
              "arn:aws:s3:::novamart-eu-*"
            ]
          }
        }
      }
    ]
  })
}

# Attach to the EU Production OU
resource "aws_organizations_policy_attachment" "eu_prod" {
  policy_id = aws_organizations_policy.eu_data_residency.id
  target_id = aws_organizations_organizational_unit.eu_production.id
}
```

**Services that MUST be restricted and their specific concerns:**

```text
┌──────────────────────┬─────────────────────────────┬────────────────────────────────────────────────┐
│ SERVICE              │ RESIDENCY RISK              │ SCP COVERAGE                                   │
├──────────────────────┼─────────────────────────────┼────────────────────────────────────────────────┤
│ EC2/EKS              │ Instances in wrong region   │ ✅ Blocked by DenyNonEURegions                 │
│ RDS                  │ Database in wrong region    │ ✅ Blocked by DenyNonEURegions                 │
│ RDS Snapshots        │ Snapshot copy to US region  │ ✅ Blocked by region restriction               │
│ S3                   │ Bucket in wrong region      │ ✅ Blocked by DenyNonEURegions                 │
│ S3 Replication       │ CRR to US region            │ ✅ Blocked by DenyS3CrossRegionReplication     │
│ EBS Snapshots        │ Copy to US region           │ ✅ Blocked by DenyEBSSnapshotCopyOutsideEU     │
│ CloudWatch Logs      │ Export to US bucket         │ ✅ Blocked by DenyCloudWatchCrossRegionExport  │
│ Kinesis/Firehose     │ Stream in wrong region      │ ✅ Blocked by DenyKinesisOutsideEU             │
│ Lambda               │ Function in wrong region    │ ✅ Blocked by DenyNonEURegions                 │
│ SQS/SNS              │ Queue/Topic in wrong region │ ✅ Blocked by DenyNonEURegions                 │
│ DynamoDB             │ Table in wrong region       │ ✅ Blocked by DenyNonEURegions                 │
│ ElastiCache          │ Cluster in wrong region     │ ✅ Blocked by DenyNonEURegions                 │
│ Secrets Manager      │ Secret in wrong region      │ ✅ Blocked by DenyNonEURegions                 │
└──────────────────────┴─────────────────────────────┴────────────────────────────────────────────────┘

EXCEPTIONS (Global services — data doesn't reside in a region):
──────────────────────────────────────────────────────────────────────
IAM                     Global by design                   ✅ Allowed
Route53                 Global DNS                         ✅ Allowed
CloudFront              Edge caching (origin in EU)        ✅ Allowed (but origin
                                                              must be eu-west-1)
ACM (us-east-1)         Required for CloudFront certs      ✅ Allowed (cert metadata
                                                              only, no customer data)
WAFv2 (global scope)    Required for CloudFront            ✅ Allowed
```

**Additional runtime validation — AWS Config rule to detect any resources created outside EU:**

```hcl
resource "aws_config_config_rule" "eu_region_only" {
  name = "eu-region-resource-check"
  
  source {
    owner             = "CUSTOM_LAMBDA"
    source_identifier = aws_lambda_function.region_checker.arn
    
    source_detail {
      event_source = "aws.config"
      message_type = "ConfigurationItemChangeNotification"
    }
  }
  
  scope {
    compliance_resource_types =[
      "AWS::EC2::Instance",
      "AWS::RDS::DBInstance",
      "AWS::S3::Bucket",
      "AWS::Lambda::Function",
      "AWS::EKS::Cluster",
      "AWS::DynamoDB::Table",
      "AWS::SQS::Queue",
      "AWS::SNS::Topic",
      "AWS::ElastiCache::CacheCluster"
    ]
  }
}
```

---

## Q4: Change Management Under Pressure 🔥

### 1. Complete Change Management Process

**Timeline:**

```text
Thursday 4:00 PM — CVE Published (CVSS 9.8, Istio Envoy RCE)
         4:05 PM — Security team assesses impact
         4:15 PM — Emergency change request initiated
         4:30 PM — VP Engineering approves
         4:45 PM — Change documentation complete
         5:00 PM — Canary rollout begins (Cluster 1, non-critical namespace)
         6:00 PM — Canary verified, full Cluster 1 rollout
         8:00 PM — Cluster 1 complete, Cluster 2 begins
        10:00 PM — Cluster 2 complete, Cluster 3 begins
Friday  12:00 AM — All clusters patched
         9:00 AM — Post-change verification report
Monday   2:00 PM — Retroactive CAB review
```

**Step-by-Step Process:**

**4:00 PM — CVE Assessment (You + Security Engineer)**

```bash
# Verify NovaMart is affected
istioctl version
# Client: 1.20.0
# Control plane: 1.20.0
# Data plane: 1.20.0

# Check the CVE details
# CVE affects Envoy proxy in Istio 1.20.0 and 1.20.1
# Fixed in 1.20.2
# Attack vector: crafted HTTP/2 request → RCE in the Envoy sidecar
# No authentication required — any internet-facing service is exploitable
```

**4:05 PM — Risk Assessment Document (written BEFORE contacting VP)**

```markdown
## Emergency Change Request: Istio Upgrade 1.20.0 → 1.20.2

### Vulnerability Summary
- **CVE:** CVE-2024-XXXX
- **CVSS:** 9.8 (Critical)  
- **Affected Component:** Envoy proxy (Istio sidecar) in all meshed pods
- **Attack Vector:** Remote, unauthenticated, via crafted HTTP/2 request
- **Impact:** Remote Code Execution in the Envoy sidecar container
- **Exploit Availability:**[Check: is there a public PoC? Check NVD, ExploitDB, GitHub]

### NovaMart Exposure
- **Affected pods:** ~400 pods across 3 clusters (all meshed services)
- **Internet-facing services:** order-svc, search-svc, api-gateway
  (directly exploitable from the internet)
- **Current traffic:** Black Friday — peak traffic, maximum exposure
- **Attack surface:** Any HTTP/2 request to any internet-facing service
  could achieve code execution inside the Envoy sidecar

### Risk of NOT Patching (waiting until Monday)
- **Duration of exposure:** 72+ hours during peak traffic
- **Likelihood of exploitation:** HIGH — Black Friday attracts attackers,
  critical Istio CVEs typically have PoCs within 24-48 hours
- **Impact if exploited:** Attacker gains code execution inside the
  service mesh. From the Envoy sidecar, they can:
  - Intercept all service-to-service traffic (mTLS terminates at Envoy)
  - Steal service account tokens
  - Pivot to any service in the mesh
  - Exfiltrate customer data (orders, payment tokens, PII)
- **Estimated cost of breach:** $500K-$2M (incident response, 
  customer notification, regulatory fines, reputation damage)

### Risk of Patching Now (during change freeze)
- **Istio 1.20.0 → 1.20.2 is a patch release** (no breaking changes)
- **Rolling restart of ~400 pods** (PDBs ensure service availability)
- **Estimated customer impact:** Brief increase in p99 latency during
  pod restarts (~50ms for 2-3 seconds per pod). No dropped requests
  if PDBs are configured correctly.
- **Rollback plan:** `istioctl upgrade --set revision=1-20-0` (10 min)
- **Estimated cost of disruption:** Minimal (rolling restart, no downtime)

### Recommendation
**PATCH NOW.** The risk of a 72-hour exposure window during Black Friday
with a CVSS 9.8 RCE far exceeds the risk of a controlled rolling restart
during the change freeze.

### Approvals Required
- [ ] VP Engineering (emergency change freeze override)
- [ ] Security Lead (risk acceptance)
-[ ] On-call SRE (execution approval)
```

**4:15 PM — Contact VP Engineering**

```text
Channel: Direct call (not Slack — this is urgent and needs verbal approval)

Script:
"Hi [VP Name], we have an emergency. A CVSS 9.8 RCE vulnerability was
published today affecting our Istio proxy — every pod in our mesh is
exploitable via HTTP/2 from the internet. We're on Black Friday traffic.

The fix is an Istio patch upgrade (1.20.0 to 1.20.2), which requires
rolling restart of ~400 pods. I have the risk assessment doc — the risk
of NOT patching during Black Friday weekend is significantly higher than
the risk of the rolling restart.

I need your approval to override the change freeze for this emergency.
I'll have full documentation ready in 15 minutes and will present to
CAB retroactively on Monday."
```

**4:30 PM — VP Approves (get written confirmation)**

```text
Slack DM from VP (screenshot and save):
"Approved. Proceed with Istio emergency upgrade per the risk assessment.
Keep me updated every hour during rollout. —[VP Name]"
```

**4:30-4:45 PM — Complete Pre-Change Documentation**

```markdown
## Emergency Change Record

**Change ID:** EMG-2024-047
**Date:** [Thursday, Black Friday week]
**Requestor:** [Your Name]
**Approver:** [VP Name]
**Change Freeze Override:** Yes — emergency security patch

### Pre-Change Checklist
- [x] Risk assessment document completed
- [x] VP Engineering approval obtained (written, timestamped)
- [x] Security Lead approval obtained
- [x] Rollout plan with canary stages documented
- [x] Rollback plan documented and tested in staging
- [x] PodDisruptionBudgets verified for all critical services
- [x] Monitoring dashboards open for all 3 clusters
- [x] On-call SRE briefed and standing by
- [x] #incident-bridge channel created for real-time updates

### Communication Plan
- 4:45 PM: Post to #engineering-all: "Emergency Istio security patch 
  starting. Rolling restarts expected. No customer impact anticipated.
  Updates in #incident-bridge."
- Hourly updates to VP during rollout
- Post to #security-incidents with CVE details
```

---

### 2. Rollout Plan for 400 Pods During Black Friday

**Phase 0: Pre-Flight (4:45 PM - 5:00 PM)**

```bash
# Verify PDBs exist for all critical services
for ns in orders payments search users inventory; do
  echo "=== $ns ==="
  kubectl get pdb -n $ns
done

# Expected output — every critical service should have a PDB like:
# NAME              MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
# order-svc-pdb     N-1             1                 1
# payment-svc-pdb   N-1             1                 1
```

```bash
# Verify current Istio health
istioctl proxy-status
# All proxies should show SYNCED

# Take a snapshot of current metrics baseline
# (to compare during rollout for anomaly detection)
curl -s "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.99,rate(istio_request_duration_milliseconds_bucket[5m]))" \
  > /tmp/pre-upgrade-p99-baseline.json
```

```bash
# Test the upgrade in staging FIRST (even under time pressure)
# Staging is not in the change freeze — it's non-production
istioctl upgrade --set revision=1-20-2 --context staging-cluster
kubectl rollout restart deployment -n orders --context staging-cluster
# Wait 10 minutes, verify staging is healthy
```

**Phase 1: Canary — Single Non-Critical Namespace (5:00 PM - 5:30 PM)**

```bash
# Start with the notifications namespace in Cluster 1
# Lowest traffic, least customer impact, good canary signal

# Step 1: Upgrade Istio control plane
istioctl upgrade \
  --set values.pilot.image=docker.io/istio/pilot:1.20.2 \
  --set values.global.proxy.image=docker.io/istio/proxyv2:1.20.2 \
  --context us-east-1-cluster

# Step 2: Verify istiod is healthy
kubectl get pods -n istio-system --context us-east-1-cluster
kubectl logs -n istio-system -l app=istiod --tail=50 --context us-east-1-cluster

# Step 3: Restart only the notification namespace pods
kubectl rollout restart deployment -n notifications --context us-east-1-cluster

# Step 4: Verify new proxy version
istioctl proxy-status --context us-east-1-cluster | grep notifications
# Should show 1.20.2 for notification pods

# Step 5: Watch metrics for 15 minutes
# Key metrics to monitor:
# - istio_request_duration_milliseconds_bucket (latency)
# - istio_requests_total (throughput — should not drop)
# - envoy_cluster_upstream_cx_connect_fail (connection failures)
# - pilot_proxy_convergence_time (proxy config sync time)
```

```yaml
# Canary success criteria (must ALL pass before proceeding):
canary_criteria:
  - metric: error_rate
    threshold: "< 0.5%"     # No increase in error rate
    window: 15m
  - metric: p99_latency
    threshold: "< 2x baseline"  # Latency may spike briefly during restart
    window: 15m                  # but must settle within 15 min
  - metric: throughput
    threshold: "> 95% of baseline"  # No significant traffic drop
    window: 15m
  - metric: proxy_sync
    threshold: "all SYNCED"     # All proxies converged on new config
    window: 5m
```

**Phase 2: Cluster 1 Full Rollout (5:30 PM - 8:00 PM)**

```bash
# Roll through namespaces in order of criticality (least critical first)
NAMESPACE_ORDER=(
  "analytics"        # Internal analytics — lowest customer impact
  "inventory"        # Inventory checks — degraded mode available
  "search"           # Search — degraded experience but not revenue-blocking
  "users"            # User service — degraded mode available  
  "orders"           # Orders — revenue-critical, second-to-last
  "payments"         # Payments — most critical, LAST
)

for ns in "${NAMESPACE_ORDER[@]}"; do
  echo "$(date): Starting rollout for namespace: $ns"
  
  # Restart deployments in this namespace
  kubectl rollout restart deployment -n $ns --context us-east-1-cluster
  
  # Wait for rollout to complete
  for deploy in $(kubectl get deploy -n $ns -o name --context us-east-1-cluster); do
    kubectl rollout status $deploy -n $ns --context us-east-1-cluster --timeout=300s
    if [ $? -ne 0 ]; then
      echo "⛔ ROLLOUT FAILED: $deploy in $ns"
      echo "Pausing rollout. Investigating."
      # DO NOT CONTINUE — investigate the failure
      exit 1
    fi
  done
  
  # Verify proxy versions
  istioctl proxy-status --context us-east-1-cluster | grep $ns
  
  # Watch metrics for 10 minutes between namespaces
  echo "$(date): Namespace $ns complete. Monitoring for 10 minutes..."
  sleep 600
  
  # Check canary criteria
  ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=sum(rate(istio_requests_total{response_code=~\"5..\",namespace=\"$ns\"}[5m]))/sum(rate(istio_requests_total{namespace=\"$ns\"}[5m]))" | jq '.data.result[0].value[1]')
  
  if (( $(echo "$ERROR_RATE > 0.005" | bc -l) )); then
    echo "⛔ ERROR RATE ELEVATED in $ns: $ERROR_RATE — PAUSING ROLLOUT"
    exit 1
  fi
  
  echo "✅ $ns healthy. Proceeding to next namespace."
done
```

**Phase 3: Cluster 2 (8:00 PM - 10:00 PM)**

```bash
# Same process as Cluster 1, but we have higher confidence now
# Can reduce inter-namespace monitoring window from 10 min to 5 min

# First: upgrade Istio control plane in Cluster 2
istioctl upgrade \
  --set values.pilot.image=docker.io/istio/pilot:1.20.2 \
  --set values.global.proxy.image=docker.io/istio/proxyv2:1.20.2 \
  --context us-west-2-cluster

# Then: same namespace-by-namespace rollout
for ns in "${NAMESPACE_ORDER[@]}"; do
  # ... same process as Phase 2
done
```

**Phase 4: Cluster 3 (10:00 PM - 12:00 AM)**

```bash
# Same process. By now we've done this twice successfully.
# Confidence is high but DO NOT skip verification steps.
```

**Phase 5: Post-Change Verification (Friday 9:00 AM)**

```bash
# Full verification across all clusters
for cluster in us-east-1-cluster us-west-2-cluster eu-west-1-cluster; do
  echo "=== Verifying $cluster ==="
  
  # All proxies on 1.20.2
  istioctl proxy-status --context $cluster | grep -v "1.20.2" | grep -v "NAME"
  # Should return NOTHING (all proxies on 1.20.2)
  
  # No elevated error rates
  # No degraded latency
  # All PDBs satisfied
  kubectl get pdb -A --context $cluster | grep -v "ALLOWED"
done

# CVE verification — confirm the vulnerability is patched
istioctl proxy-config log order-svc-xxx.orders --context us-east-1-cluster | \
  grep "Envoy version"
# Should show the patched Envoy version
```

---

### 3. Three Pods Fail After Upgrade — Continue or Stop?

**STOP THE ROLLOUT. Do not proceed to other clusters.**

Here is the specific decision framework:

```text
DECISION CRITERIA:
═══════════════════════════════════════════════════════════════

CONTINUE rollout to other clusters IF:
  ✅ Failure is isolated to a known, unrelated issue
     (e.g., pod was already unhealthy before the upgrade)
  ✅ The failing pods are in non-critical namespaces
  ✅ All other pods in the cluster upgraded successfully
  ✅ The failure is understood and has a workaround
  ✅ Error rate has not increased for successfully upgraded pods

STOP rollout IF (any of these):
  ⛔ Failure is in the Istio control plane (istiod) — STOP
  ⛔ Multiple pods across different namespaces fail — STOP
  ⛔ The failure pattern suggests a systemic compatibility issue — STOP
  ⛔ Error rate is elevated for already-upgraded pods — STOP
  ⛔ The failure is not understood — STOP
```

**In this specific case: 3 pods fail with init container timeout connecting to istiod. This is STOP.**

**Reasoning:**

1. **Init container timeout connecting to istiod is a control plane issue.** It means the Istio control plane upgrade may have introduced an incompatibility or the control plane is overloaded. If istiod is struggling to serve configuration to 3 pods in Cluster 1, it will likely struggle with 400+ pods across 3 clusters.

2. **The pattern is not isolated.** 3 pods across potentially multiple namespaces failing with the same symptom (istiod connectivity) indicates a systemic issue, not a single bad pod.

3. **Black Friday traffic amplifies risk.** If you continue to Cluster 2 and the same issue occurs worse (more pods fail, istiod becomes overwhelmed), you now have TWO clusters in a degraded state during peak revenue hours.

4. **Cluster 1 is already partially patched.** The services that successfully upgraded are now protected. Stopping doesn't leave you worse than before — it leaves you partially patched (which is better than unpatched).

**Immediate actions after stopping:**

```bash
# Investigate the failing pods
kubectl describe pod <failing-pod> -n <namespace> --context us-east-1-cluster
kubectl logs <failing-pod> -c istio-init -n <namespace> --context us-east-1-cluster

# Check istiod health
kubectl logs -n istio-system -l app=istiod --tail=100 --context us-east-1-cluster
kubectl top pod -n istio-system --context us-east-1-cluster

# Check if istiod is overloaded (pilot_xds_pushes, pilot_proxy_convergence_time)
curl -s "http://prometheus:9090/api/v1/query?query=pilot_proxy_convergence_time{quantile=\"0.99\"}" 

# Possible causes:
# 1. istiod resource limits too low for the new version's requirements
# 2. istiod webhook configuration conflict between old and new version
# 3. Network policy blocking init container → istiod on the new port
# 4. Certificate rotation issue during upgrade

# Fix and retry for Cluster 1 before touching Cluster 2
```

```text
Communication update:
"🟡 Istio upgrade paused. 3 pods in Cluster 1 showing init container 
timeouts connecting to istiod. Investigating. Clusters 2 and 3 remain 
on 1.20.0. Successfully upgraded services in Cluster 1 are healthy 
and protected. Will resume after root cause is identified and fixed.
Next update in 30 minutes."
```

---

### 4. Risk Analysis — Should We Have Waited Until Monday?

**Executive Summary: No. Patching Thursday was the correct decision.**

```text
RISK ANALYSIS: PATCH NOW vs WAIT UNTIL MONDAY
══════════════════════════════════════════════════════════════════

                        PATCH THURSDAY          WAIT UNTIL MONDAY
                        ──────────────          ─────────────────

VULNERABILITY WINDOW    6-8 hours               72+ hours
                        (during rollout)         (Fri-Mon)

EXPOSURE PERIOD         Thursday evening         Black Friday +
                        (declining traffic)      Saturday + Sunday
                                                 (PEAK traffic year)

EXPLOIT LIKELIHOOD      Low (patch in progress)  HIGH (9.8 CVSS, PoCs
                                                 likely within 24-48hr,
                                                 Black Friday attracts
                                                 attackers)

ATTACK SURFACE          ~400 pods, internet-     Same, but with 10-50x
                        facing during normal     more traffic volume and
                        Thursday evening traffic  more attacker attention

IF EXPLOITED —          RCE in Envoy sidecar     Same, but during peak
IMPACT                  Intercept all mesh       revenue period.
                        traffic, steal secrets,  Data breach during
                        pivot to any service     Black Friday = 
                                                 catastrophic PR

CUSTOMER IMPACT OF      Brief latency spike      ZERO (no change)
THE CHANGE ITSELF       during pod restarts      BUT: 72-hour window
                        (~50ms for 2-3 sec       where customers are
                        per pod). Within PDB     at risk of data theft
                        guarantees. No dropped   
                        requests.                

COST OF DISRUPTION      Minimal — rolling        N/A
FROM PATCHING           restart with PDBs.
                        Estimated: $0 revenue
                        impact if PDBs hold.

COST OF BREACH          If exploited during      If exploited during
IF NOT PATCHED          rollout window: limited  72-hour window:
                        (6-8hr window, most      maximum (peak traffic,
                        pods already patched)    all customer data at
                                                 risk, regulatory
                        Estimated: $50K-$200K    notification required)
                                                 
                                                 Estimated: $500K-$2M+

REGULATORY RISK         Low — demonstrates       HIGH — knowingly
                        prompt vulnerability      leaving a known
                        response                 critical vulnerability
                                                 unpatched for 72 hours
                                                 is hard to defend to
                                                 regulators or in
                                                 litigation

CHANGE FREEZE           Technical violation      Compliant with freeze
COMPLIANCE              (requires retroactive    
                        CAB review)              
                        Cost: 1 hour of CAB      Cost: 72 hours of
                        time on Monday           exposure
```

**The Math:**

```text
Expected cost of patching now:
  P(customer impact) × Impact = 0.05 × $10,000 = $500
  + P(rollout failure) × Recovery cost = 0.10 × $5,000 = $500
  + CAB review time = $500 (5 people × 1 hour)
  = $1,500 expected cost

Expected cost of waiting:
  P(exploit published within 72hr) × P(NovaMart targeted) × Impact
  = 0.60 × 0.15 × $1,500,000
  = $135,000 expected cost

  Plus: P(no exploit) × reputational risk of knowingly running 
  vulnerable during peak
  = 0.40 × $50,000 (regulatory risk of inaction)
  = $20,000

Total expected cost of waiting = $155,000
```

**$1,500 expected cost vs $155,000 expected cost. Patching now is 100:1 the better decision.**

**CVSS 9.8 Context:**

A CVSS 9.8 means:
- **Attack Vector:** Network (remotely exploitable)
- **Attack Complexity:** Low (no special conditions)
- **Privileges Required:** None (unauthenticated)
- **User Interaction:** None (no phishing needed)
- **Impact:** High on Confidentiality, Integrity, AND Availability

This is the most dangerous class of vulnerability. It's one step below a CVSS 10.0. Historical data shows that critical Istio/Envoy CVEs have public exploits within 24-72 hours of disclosure. Black Friday is the highest-traffic period of the year — the exact time when the most customer data flows through the vulnerable Envoy proxies.

**The change freeze exists to protect revenue. Leaving a CVSS 9.8 RCE unpatched during Black Friday is a greater threat to revenue than a controlled rolling restart.**

**Presentation to the VP:**

> "The change freeze policy exists to protect Black Friday revenue by preventing risky changes. I respect that purpose. But in this case, NOT changing is the risky option. 
>
> A CVSS 9.8 unauthenticated RCE in every Envoy sidecar means any attacker can execute arbitrary code inside our service mesh — intercepting every order, every payment token, every customer record — by sending a single crafted HTTP/2 request to any of our internet-facing services.
>
> During Black Friday, we process more transactions than any other day of the year. Our attack surface is at its maximum. Attacker attention on e-commerce platforms is at its peak. And the expected time to a public exploit for this class of vulnerability is 24-48 hours.
>
> The Istio patch upgrade is a minor version bump (1.20.0 → 1.20.2) with no breaking changes. The rolling restart with PDBs guarantees zero dropped requests. The worst case from patching is a brief latency spike. The worst case from NOT patching is a data breach during our highest-revenue day of the year.
>
> I patched because that was the decision that protected our customers and our revenue. I documented everything before, during, and after. I'm prepared for the retroactive CAB review on Monday and confident the review will validate this decision."




**Retroactive CAB Review Documentation (Monday 2 PM):**

```markdown
## Retroactive Change Advisory Board Review
## Emergency Change: EMG-2024-047

### Change Summary
- **What:** Istio upgrade 1.20.0 → 1.20.2 across 3 production clusters
- **When:** Thursday 5:00 PM — Friday 12:00 AM
- **Why:** CVSS 9.8 RCE vulnerability (CVE-2024-XXXX) in Envoy proxy
- **Who Approved:** [VP Engineering Name], [Security Lead Name]
- **Who Executed:** [Your Name],[On-call SRE Name]

### Change Freeze Override Justification
Per NovaMart Change Management Policy §4.3:
> "Emergency changes that address active security vulnerabilities 
> with CVSS ≥ 9.0, or that mitigate ongoing service degradation,
> may bypass the change freeze with VP Engineering approval and
> retroactive CAB review within 3 business days."

This change qualifies under the security vulnerability clause.

### Timeline of Events
| Time | Event |
|------|-------|
| Thu 4:00 PM | CVE-2024-XXXX published (CVSS 9.8, Istio Envoy RCE) |
| Thu 4:05 PM | Security team confirms NovaMart is affected |
| Thu 4:15 PM | Risk assessment document completed |
| Thu 4:30 PM | VP Engineering approves emergency change |
| Thu 4:45 PM | Pre-change documentation completed |
| Thu 5:00 PM | Canary rollout: notifications namespace, Cluster 1 |
| Thu 5:30 PM | Canary verified healthy — full Cluster 1 rollout begins |
| Thu 6:15 PM | 3 pods fail in Cluster 1 (istiod init timeout) |
| Thu 6:15 PM | Rollout PAUSED — investigation begins |
| Thu 6:45 PM | Root cause: istiod resource limits insufficient for new version |
| Thu 7:00 PM | istiod resource limits increased, failing pods recovered |
| Thu 7:15 PM | Rollout resumed for remaining Cluster 1 namespaces |
| Thu 8:00 PM | Cluster 1 complete — all namespaces on 1.20.2 |
| Thu 8:15 PM | Cluster 2 rollout begins (with istiod fix pre-applied) |
| Thu 10:00 PM | Cluster 2 complete |
| Thu 10:15 PM | Cluster 3 rollout begins |
| Fri 12:00 AM | All 3 clusters on Istio 1.20.2 — change complete |
| Fri 9:00 AM | Post-change verification report confirms all healthy |

### Customer Impact
- **Dropped requests:** 0 (PDBs maintained availability throughout)
- **Latency impact:** p99 latency increased by ~40ms during pod restarts
  in each namespace (duration: 3-5 minutes per namespace, within SLO)
- **Error rate impact:** No increase in 5xx error rate
- **Revenue impact:** $0 (no failed transactions attributed to the change)

### Issues Encountered
1. **3 pods failed in Cluster 1** (istiod init container timeout)
   - Root cause: Istio 1.20.2's istiod requires 15% more memory 
     during config push than 1.20.0
   - Fix: Increased istiod memory limit from 2Gi to 2.5Gi
   - Applied fix before proceeding to Clusters 2 and 3
   - No recurrence in Clusters 2 or 3

### Risk Assessment Outcomes
- **Risk of patching (actual):** Minor istiod resource issue, 
  resolved in 30 minutes. Zero customer impact.
- **Risk of NOT patching (avoided):** 72-hour exposure to CVSS 9.8 
  RCE during peak Black Friday traffic. Public PoC exploit was 
  published Saturday morning (within 36 hours of disclosure) — 
  NovaMart was already patched.

### Post-Change Metrics (Friday 9 AM)
| Metric | Pre-Change | Post-Change | Delta |
|--------|-----------|-------------|-------|
| p99 latency (order-svc) | 185ms | 182ms | -3ms ✅ |
| Error rate (all services) | 0.12% | 0.11% | -0.01% ✅ |
| Throughput (requests/sec) | 48,200 | 49,100 | +1.9% ✅ |
| Envoy proxy version | 1.20.0 | 1.20.2 | Patched ✅ |
| CVE-2024-XXXX status | VULNERABLE | REMEDIATED | Fixed ✅ |

### Lessons Learned
1. **Istio upgrade resource requirements should be tested in staging first.**
   Even though we were time-pressured, the staging test (which we did)
   didn't catch the istiod memory issue because staging has fewer pods
   and lower config push volume. 
   
   **Action:** Add a pre-upgrade check that compares istiod resource 
   usage at current scale and projects requirements for the new version.

2. **The change freeze policy should explicitly pre-authorize security patches
   above CVSS 9.0** to avoid the approval delay. In this case, the VP
   approved in 15 minutes — but what if they were unreachable?
   
   **Action:** Update change freeze policy to include standing authorization:
   "Security patches for CVSS ≥ 9.0 vulnerabilities are pre-approved during
   change freezes. The security team lead may authorize execution without
   VP approval, with notification to VP within 1 hour."

3. **The rollout plan worked.** Namespace-by-namespace ordering, canary
   verification, and the stop criteria caught the istiod issue before
   it affected critical services. The PDB configuration prevented any
   customer impact. This validates our rollout methodology.

### CAB Decision Requested
- [ ] **Ratify:** Approve the emergency change retroactively
- [ ] **Ratify with conditions:** Approve with required follow-up actions
-[ ] **Reject:** (Note: change is already complete — rejection triggers 
      a process review, not a rollback)

### Recommended Follow-Up Actions
1. Update change freeze policy per Lesson Learned #2
2. Add istiod resource projection to pre-upgrade checklist
3. Document this as a reference example for future emergency changes
4. Schedule Istio upgrade testing improvements for next sprint
```

---

**Process Improvement: Updated Change Freeze Policy**

```markdown
## NovaMart Change Management Policy — §4.3 Emergency Changes (REVISED)

### 4.3.1 Definition
An emergency change is a change that must be implemented immediately to:
  a) Resolve a service-impacting incident (SEV1 or SEV2), OR
  b) Remediate a critical security vulnerability (CVSS ≥ 9.0), OR
  c) Comply with a regulatory directive with an imminent deadline

### 4.3.2 Authorization During Change Freezes

#### Pre-Authorized Emergency Changes (no approval delay)
The following changes are PRE-AUTHORIZED during change freezes and 
may be executed by the on-call team with notification (not approval):

  - Security patches for vulnerabilities with CVSS ≥ 9.0
  - WAF rule changes to block active attacks
  - Certificate renewals for expiring certificates
  - DNS changes to failover during active outages
  - Scaling changes (HPA/node count) to handle traffic

**Requirements for pre-authorized changes:**
  1. Risk assessment document completed BEFORE execution
  2. Notification to VP Engineering within 1 hour of execution
  3. Retroactive CAB review within 3 business days
  4. Post-change verification report within 24 hours

#### Approval-Required Emergency Changes
All other emergency changes during change freezes require:
  1. VP Engineering verbal approval (followed by written confirmation)
  2. Security Lead concurrence (for security-related changes)
  3. Risk assessment document
  4. Retroactive CAB review within 3 business days

### 4.3.3 Execution Standards
Emergency changes follow the same technical standards as normal changes:
  - Canary/phased rollout where possible
  - PodDisruptionBudget compliance
  - Monitoring during rollout
  - Documented rollback plan
  - Post-change verification

The ONLY difference is the approval timeline — not the quality standard.

### 4.3.4 Post-Incident Review
Every emergency change during a change freeze triggers a review to assess:
  a) Was the emergency classification justified?
  b) Could the change have waited without unacceptable risk?
  c) Was the change executed with appropriate care?
  d) What would have prevented the emergency in the first place?
```

---

**Standing Authorization Matrix for Change Freezes:**

```text
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                              CHANGE FREEZE — AUTHORIZATION MATRIX                            │
├────────────────────────┬──────────────────┬──────────────────────────┬───────────────────────┤
│ CHANGE TYPE            │ AUTHORIZATION    │ REQUIRED                 │ CAB REVIEW            │
│                        │ LEVEL            │ BEFORE START             │                       │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ Security patch         │ Pre-approved     │ Risk doc +               │ Retroactive           │
│ (CVSS ≥ 9.0)           │ (notify VP)      │ notification             │ within 3 days         │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ Security patch         │ VP approval      │ Risk doc +               │ Retroactive           │
│ (CVSS 7.0-8.9)         │ required         │ VP sign-off              │ within 3 days         │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ Security patch         │ DENIED           │ Wait for                 │ N/A                   │
│ (CVSS < 7.0)           │                  │ freeze end               │                       │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ Active incident        │ Pre-approved     │ Incident                 │ Part of               │
│ mitigation (SEV1/2)    │ (IC decides)     │ record                   │ postmortem            │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ Scaling changes        │ Pre-approved     │ Notification             │ Not required          │
│ (HPA, node count)      │ (SRE on-call)    │ only                     │                       │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ WAF/firewall rules     │ Pre-approved     │ Risk doc +               │ Retroactive           │
│ (active attack)        │ (SecOps)         │ notification             │ within 3 days         │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ Certificate renewal    │ Pre-approved     │ Notification             │ Not required          │
│ (< 24hr to expiry)     │                  │ only                     │                       │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ Feature deployments    │ DENIED           │ Wait for                 │ N/A                   │
│                        │                  │ freeze end               │                       │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ Infrastructure         │ DENIED           │ Wait for                 │ N/A                   │
│ changes (non-          │ (unless VP       │ freeze end               │                       │
│ security)              │ override)        │                          │                       │
├────────────────────────┼──────────────────┼──────────────────────────┼───────────────────────┤
│ Database schema        │ DENIED           │ Wait for                 │ N/A                   │
│ changes                │                  │ freeze end               │                       │
└────────────────────────┴──────────────────┴──────────────────────────┴───────────────────────┘
```

---

**Appendix: Emergency Change Communication Templates**

```markdown
## Template 1: Initial Notification (post to #engineering-all)

🔒 EMERGENCY SECURITY PATCH —[SERVICE/COMPONENT]

**What:** [Brief description of the change]
**Why:**[CVE ID, CVSS score, 1-sentence impact]
**When:** Starting now, estimated completion [time]
**Customer impact:** [Expected impact or "None expected"]
**Your action needed:** None — this is informational

Updates will be posted in #incident-bridge.
Questions → DM [Your Name] (please don't post in #engineering-all)

---

## Template 2: Hourly Update (post to #incident-bridge)

🔄 ISTIO UPGRADE STATUS — [TIME]

**Progress:** Cluster [X] of 3 complete
**Namespaces upgraded:** [X] of [Y]
**Issues:** [None / Description]
**Customer impact observed:** [None / Description]
**Next milestone:** [What happens next and when]

---

## Template 3: Completion (post to #engineering-all)

✅ EMERGENCY SECURITY PATCH COMPLETE

**What:** Istio upgraded from 1.20.0 → 1.20.2 across all 3 clusters
**Duration:** [Start time] — [End time]
**Customer impact:** [None observed / Description]
**Issues encountered:** [None / Brief description and resolution]

CVE-2024-XXXX is now remediated across all production clusters.
Full post-change report: [link]
Retroactive CAB review scheduled: Monday 2 PM

Thank you to [names] for executing this safely under time pressure.
```

---

**Final Governance Integration — How This Connects to SOC 2:**

This entire emergency change process directly satisfies SOC 2 Trust Service Criteria:

```text
┌───────────────────────┬──────────────────────────────────────────────────────┐
│ SOC 2 CRITERIA        │ HOW THIS CHANGE SATISFIES IT                         │
├───────────────────────┼──────────────────────────────────────────────────────┤
│ CC6.8 (Change Mgmt)   │ - Documented change request with risk assessment     │
│                       │ - Approval trail (VP sign-off, timestamped)          │
│                       │ - Pre-change and post-change documentation           │
│                       │ - Retroactive CAB review                             │
├───────────────────────┼──────────────────────────────────────────────────────┤
│ CC7.1 (Monitoring)    │ - Continuous monitoring during rollout               │
│                       │ - Canary verification with specific metrics          │
│                       │ - Post-change verification report                    │
├───────────────────────┼──────────────────────────────────────────────────────┤
│ CC7.2 (Vulnerability) │ - Vulnerability identified same day as disclosure    │
│                       │ - Risk assessed within 15 minutes                    │
│                       │ - Remediated within 8 hours                          │
│                       │ - Faster than the 72-hour PoC publication            │
├───────────────────────┼──────────────────────────────────────────────────────┤
│ CC7.4 (Incident Mgmt) │ - Structured response process followed               │
│                       │ - Communication plan executed                        │
│                       │ - Escalation path documented                         │
│                       │ - Issue during rollout (istiod) handled per process  │
├───────────────────────┼──────────────────────────────────────────────────────┤
│ CC8.1 (Change Control)│ - Even under emergency conditions:                   │
│                       │   - Phased rollout (canary → full)                   │
│                       │   - PDB compliance verified                          │
│                       │   - Rollback plan documented                         │
│                       │   - Stop criteria defined AND triggered              │
│                       │   - Quality standards not compromised for speed      │
├───────────────────────┼──────────────────────────────────────────────────────┤
│ CC9.1 (Risk Mgmt)     │ - Formal risk assessment comparing patch vs wait     │
│                       │ - Quantified expected costs for both options         │
│                       │ - Decision documented with reasoning                 │
│                       │ - Lessons learned captured for future improvement    │
└───────────────────────┴──────────────────────────────────────────────────────┘
```

**The key audit message:** NovaMart's emergency change process demonstrates that even during the highest-pressure scenario imaginable — a CVSS 9.8 vulnerability during a Black Friday change freeze — the team followed a structured, documented, risk-assessed process. They didn't panic-deploy. They didn't skip steps. They didn't compromise quality for speed. They made a justified decision, documented it thoroughly, executed it carefully, caught and resolved an issue during rollout, and completed within 8 hours with zero customer impact.

This is exactly what an auditor wants to see: not that nothing ever goes wrong, but that when it does, there's a mature process that handles it.
