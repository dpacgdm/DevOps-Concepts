# PHASE 7, LESSON 3: CI/CD PIPELINE

## Building NovaMart's Deployment Machinery — Commit to Production

---

## ARCHITECTURE OVERVIEW

```
┌──────────────┐    webhook     ┌──────────────────┐
│  Bitbucket    │──────────────▶│    Jenkins        │
│  (Source)     │               │    Controller     │
│              │◀──────────────│    (EKS Pod)      │
│  2 repos:    │   status API   │                  │
│  - app-src   │               └────────┬─────────┘
│  - app-gitops│                        │ spawns
└──────────────┘                        │
                                ┌───────▼──────────┐
                                │  Jenkins Agent    │
                                │  (Ephemeral Pod)  │
                                │                   │
                                │ ┌───────────────┐ │
                                │ │ Build & Test  │ │
                                │ │ (Go/Java/Py)  │ │
                                │ └───────┬───────┘ │
                                │         │         │
                                │ ┌───────▼───────┐ │
                                │ │ SonarQube     │ │
                                │ │ (Quality Gate)│ │
                                │ └───────┬───────┘ │
                                │         │         │
                                │ ┌───────▼───────┐ │
                                │ │ Trivy + Black │ │
                                │ │ Duck (Security│ │
                                │ │ Scan)         │ │
                                │ └───────┬───────┘ │
                                │         │         │
                                │ ┌───────▼───────┐ │
                                │ │ Kaniko        │ │
                                │ │ (Build Image) │ │
                                │ └───────┬───────┘ │
                                │         │         │
                                │ ┌───────▼───────┐ │
                                │ │ Push to ECR   │ │
                                │ └───────┬───────┘ │
                                │         │         │
                                │ ┌───────▼───────┐ │
                                │ │ Update GitOps │ │
                                │ │ Repo (image   │ │
                                │ │ tag)          │ │
                                │ └───────────────┘ │
                                └───────────────────┘
                                        │
                                        │ git push (new image tag)
                                        ▼
                               ┌──────────────────┐
                               │  ArgoCD          │
                               │  (detects diff)  │
                               │                  │
                               │  Sync Strategy:  │
                               │  - dev: auto     │
                               │  - staging: auto │
                               │  - prod: manual  │
                               │    + Argo Rollout│
                               └────────┬─────────┘
                                        │
                          ┌─────────────▼──────────────┐
                          │     EKS Cluster             │
                          │                             │
                          │  ┌─────────────────────┐    │
                          │  │  Argo Rollouts      │    │
                          │  │  (Canary/Blue-Green)│    │
                          │  │                     │    │
                          │  │  10% → 30% → 60% → │    │
                          │  │  100% with analysis │    │
                          │  └─────────────────────┘    │
                          └─────────────────────────────┘
```

---

## DIRECTORY STRUCTURE

```
cicd/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── backend.tf
│   ├── jenkins-irsa.tf
│   ├── sonarqube-rds.tf
│   ├── s3-artifacts.tf
│   └── ecr-scanner.tf
│
├── jenkins/
│   ├── helm-values/
│   │   └── jenkins.yaml
│   ├── casc/
│   │   ├── jenkins.yaml           # Configuration as Code
│   │   ├── credentials.yaml
│   │   └── security.yaml
│   ├── agents/
│   │   ├── Dockerfile.golang
│   │   ├── Dockerfile.java
│   │   ├── Dockerfile.python
│   │   └── Dockerfile.kaniko
│   └── shared-library/
│       ├── vars/
│       │   ├── novamartPipeline.groovy    # Golden path template
│       │   ├── buildImage.groovy
│       │   ├── securityScan.groovy
│       │   ├── qualityGate.groovy
│       │   ├── deployToEnvironment.groovy
│       │   ├── notifySlack.groovy
│       │   └── promotionGate.groovy
│       ├── src/
│       │   └── com/novamart/
│       │       ├── PipelineConfig.groovy
│       │       └── GitOpsUpdater.groovy
│       └── resources/
│           └── com/novamart/
│               └── pipeline-defaults.yaml
│
├── sonarqube/
│   ├── helm-values/
│   │   └── sonarqube.yaml
│   └── quality-profiles/
│       ├── java-novamart.xml
│       ├── go-novamart.xml
│       └── python-novamart.xml
│
├── argo-rollouts/
│   ├── helm-values/
│   │   └── argo-rollouts.yaml
│   └── analysis-templates/
│       ├── success-rate.yaml
│       ├── latency.yaml
│       ├── error-budget.yaml
│       └── canary-analysis.yaml
│
├── gitops-repo-structure/
│   ├── base/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml        # → Rollout
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml
│   │   ├── service-monitor.yaml
│   │   └── external-secret.yaml
│   └── overlays/
│       ├── dev/
│       │   ├── kustomization.yaml
│       │   └── patches/
│       ├── staging/
│       │   ├── kustomization.yaml
│       │   └── patches/
│       └── production/
│           ├── kustomization.yaml
│           └── patches/
│
├── manifests/
│   ├── argocd-apps/
│   │   ├── payment-service-dev.yaml
│   │   ├── payment-service-staging.yaml
│   │   └── payment-service-production.yaml
│   └── jenkins-namespace.yaml
│
├── docs/
│   ├── design-decisions.md
│   ├── pipeline-developer-guide.md
│   ├── onboarding-checklist.md
│   └── troubleshooting.md
│
└── scripts/
    ├── bootstrap-jenkins.sh
    └── validate-cicd.sh
```

---

## 1. TERRAFORM — CI/CD INFRASTRUCTURE

### `terraform/variables.tf`
```hcl
variable "environment" {
  type    = string
  default = "production"
}

variable "cluster_name" {
  type    = string
  default = "novamart-production"
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "cluster_oidc_provider_arn" {
  type = string
}

variable "cluster_oidc_issuer_url" {
  type = string
}

variable "vpc_id" {
  type = string
}

variable "private_subnet_ids" {
  type = list(string)
}

variable "data_subnet_ids" {
  type = list(string)
}

variable "observability_kms_key_arn" {
  type = string
}
```

### `terraform/jenkins-irsa.tf`
```hcl
# ============================================================
# Jenkins Controller IRSA
# Needs: ECR auth, Secrets Manager (credentials), S3 (artifacts)
# ============================================================
module "irsa_jenkins_controller" {
  source = "../../modules/irsa"

  role_name         = "novamart-${var.environment}-jenkins-controller"
  namespace         = "jenkins"
  service_account   = "jenkins"
  oidc_provider_arn = var.cluster_oidc_provider_arn
  oidc_provider_url = var.cluster_oidc_issuer_url

  policy_json = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ECRAuth"
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken"
        ]
        Resource = ["*"]
      },
      {
        Sid    = "SecretsRead"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = [
          "arn:aws:secretsmanager:${var.aws_region}:*:secret:novamart/${var.environment}/jenkins/*",
          "arn:aws:secretsmanager:${var.aws_region}:*:secret:novamart/${var.environment}/sonarqube/*"
        ]
      },
      {
        Sid    = "S3Artifacts"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.jenkins_artifacts.arn,
          "${aws_s3_bucket.jenkins_artifacts.arn}/*"
        ]
      }
    ]
  })
}

# ============================================================
# Jenkins Agent IRSA
# Needs: ECR push/pull, S3 artifacts, STS for cross-account
# ============================================================
module "irsa_jenkins_agent" {
  source = "../../modules/irsa"

  role_name         = "novamart-${var.environment}-jenkins-agent"
  namespace         = "jenkins"
  service_account   = "jenkins-agent"
  oidc_provider_arn = var.cluster_oidc_provider_arn
  oidc_provider_url = var.cluster_oidc_issuer_url

  policy_json = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ECRFullAccess"
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:DescribeRepositories",
          "ecr:ListImages",
          "ecr:DescribeImages"
        ]
        Resource = ["*"]
      },
      {
        Sid    = "ECRAuth"
        Effect = "Allow"
        Action = ["ecr:GetAuthorizationToken"]
        Resource = ["*"]
      },
      {
        Sid    = "S3Artifacts"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket",
          "s3:DeleteObject"
        ]
        Resource = [
          aws_s3_bucket.jenkins_artifacts.arn,
          "${aws_s3_bucket.jenkins_artifacts.arn}/*"
        ]
      },
      {
        Sid    = "KMSDecrypt"
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = [var.observability_kms_key_arn]
      }
    ]
  })
}
```

### `terraform/sonarqube-rds.tf`
```hcl
# SonarQube needs PostgreSQL
resource "aws_db_instance" "sonarqube" {
  identifier = "novamart-${var.environment}-sonarqube"

  engine               = "postgres"
  engine_version       = "15.5"
  instance_class       = "db.t3.medium"
  allocated_storage    = 50
  max_allocated_storage = 100
  storage_encrypted    = true
  kms_key_id           = var.observability_kms_key_arn

  db_name  = "sonarqube"
  username = "sonarqube"
  manage_master_user_password = true

  multi_az            = false  # SonarQube is not user-facing — single AZ acceptable
  db_subnet_group_name = aws_db_subnet_group.sonarqube.name
  vpc_security_group_ids = [aws_security_group.sonarqube_db.id]

  backup_retention_period = 7
  skip_final_snapshot     = false
  final_snapshot_identifier = "novamart-sonarqube-final-${formatdate("YYYY-MM-DD", timestamp())}"

  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn

  parameter_group_name = aws_db_parameter_group.sonarqube.name

  tags = {
    Environment = var.environment
    Component   = "sonarqube"
    ManagedBy   = "terraform"
  }
}

resource "aws_db_parameter_group" "sonarqube" {
  family = "postgres15"
  name   = "novamart-${var.environment}-sonarqube"

  parameter {
    name  = "max_connections"
    value = "200"
  }
}

resource "aws_db_subnet_group" "sonarqube" {
  name       = "novamart-${var.environment}-sonarqube"
  subnet_ids = var.data_subnet_ids
}

resource "aws_security_group" "sonarqube_db" {
  name_prefix = "novamart-sonarqube-db-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    description     = "SonarQube pods"
    security_groups = [] # Populated by EKS node SG
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "novamart-${var.environment}-sonarqube-db"
  }
}
```

### `terraform/s3-artifacts.tf`
```hcl
resource "aws_s3_bucket" "jenkins_artifacts" {
  bucket = "novamart-${var.environment}-jenkins-artifacts"

  tags = {
    Environment = var.environment
    Component   = "jenkins"
  }
}

resource "aws_s3_bucket_versioning" "jenkins_artifacts" {
  bucket = aws_s3_bucket.jenkins_artifacts.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "jenkins_artifacts" {
  bucket = aws_s3_bucket.jenkins_artifacts.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.observability_kms_key_arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "jenkins_artifacts" {
  bucket = aws_s3_bucket.jenkins_artifacts.id

  rule {
    id     = "cleanup-old-artifacts"
    status = "Enabled"

    # Build artifacts older than 90 days → IA
    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }

    # Delete after 1 year
    expiration {
      days = 365
    }
  }

  rule {
    id     = "cleanup-pr-artifacts"
    status = "Enabled"
    filter {
      prefix = "pr-builds/"
    }
    # PR build artifacts expire fast
    expiration {
      days = 14
    }
  }
}

resource "aws_s3_bucket_public_access_block" "jenkins_artifacts" {
  bucket                  = aws_s3_bucket.jenkins_artifacts.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## 2. JENKINS ON KUBERNETES

### `helm-values/jenkins.yaml`
```yaml
# Jenkins LTS on EKS
# DD-CICD-001: Jenkins chosen as CI engine
#   - Existing team expertise
#   - Shared library ecosystem
#   - Kubernetes plugin for ephemeral agents
#   - Migration to Tekton/GitHub Actions planned for Phase 2

controller:
  image: jenkins/jenkins
  tag: "2.440.3-lts-jdk17"

  resources:
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      memory: 4Gi

  # Persistence for controller state
  persistence:
    enabled: true
    storageClass: gp3-encrypted
    size: 50Gi

  # Service account with IRSA
  serviceAccount:
    name: jenkins
    annotations:
      eks.amazonaws.com/role-arn: "arn:aws:iam::role/novamart-production-jenkins-controller"

  # JCasC — Configuration as Code
  JCasC:
    configScripts:
      welcome-message: |
        jenkins:
          systemMessage: "NovaMart CI/CD — Managed by Platform Team. Do NOT configure manually."

      security: |
        jenkins:
          securityRealm:
            oic:
              clientId: "${JENKINS_OIDC_CLIENT_ID}"
              clientSecret: "${JENKINS_OIDC_CLIENT_SECRET}"
              wellKnownOpenIDConfigurationUrl: "https://sso.novamart.com/.well-known/openid-configuration"
              userNameField: "preferred_username"
              fullNameFieldName: "name"
              emailFieldName: "email"
              groupsFieldName: "groups"
          authorizationStrategy:
            roleBased:
              roles:
                global:
                  - name: "admin"
                    permissions:
                      - "Overall/Administer"
                    entries:
                      - group: "platform-eng"
                  - name: "developer"
                    permissions:
                      - "Overall/Read"
                      - "Job/Read"
                      - "Job/Build"
                      - "Job/Cancel"
                      - "Job/Workspace"
                    entries:
                      - group: "developers"
                  - name: "viewer"
                    permissions:
                      - "Overall/Read"
                      - "Job/Read"
                    entries:
                      - group: "qa-team"
          crumbIssuer:
            standard:
              excludeClientIPFromCrumb: true

      shared-library: |
        unclassified:
          globalLibraries:
            libraries:
              - name: "novamart-pipeline-library"
                defaultVersion: "main"
                implicit: true
                retriever:
                  modernSCM:
                    scm:
                      git:
                        remote: "https://bitbucket.org/novamart/jenkins-shared-library.git"
                        credentialsId: "bitbucket-credentials"

      cloud: |
        jenkins:
          clouds:
            - kubernetes:
                name: "kubernetes"
                serverUrl: "https://kubernetes.default.svc"
                namespace: "jenkins"
                jenkinsUrl: "http://jenkins.jenkins.svc.cluster.local:8080"
                jenkinsTunnel: "jenkins-agent.jenkins.svc.cluster.local:50000"
                # Pod retention — always delete after build
                podRetention: "never"
                # Container cap — prevent runaway
                containerCap: 50
                # Timeout for pod startup
                waitForPodSec: 300
                # Default pod template
                templates:
                  - name: "default"
                    namespace: "jenkins"
                    label: "jenkins-agent"
                    serviceAccount: "jenkins-agent"
                    nodeSelector: "kubernetes.io/arch=amd64"
                    # Pod-level
                    yaml: |
                      spec:
                        securityContext:
                          runAsUser: 1000
                          runAsGroup: 1000
                          fsGroup: 1000
                        tolerations:
                          - key: "dedicated"
                            operator: "Equal"
                            value: "ci"
                            effect: "NoSchedule"
                        affinity:
                          nodeAffinity:
                            preferredDuringSchedulingIgnoredDuringExecution:
                              - weight: 100
                                preference:
                                  matchExpressions:
                                    - key: karpenter.sh/nodepool
                                      operator: In
                                      values:
                                        - ci-builds
                    containers:
                      - name: "jnlp"
                        image: "jenkins/inbound-agent:3206.vb_15dcf73f6a_9-1-jdk17"
                        resourceRequestCpu: "100m"
                        resourceRequestMemory: "256Mi"
                        resourceLimitMemory: "512Mi"

      credentials: |
        credentials:
          system:
            domainCredentials:
              - credentials:
                  - string:
                      scope: GLOBAL
                      id: "sonarqube-token"
                      description: "SonarQube analysis token"
                      secret: "${SONARQUBE_TOKEN}"
                  - string:
                      scope: GLOBAL
                      id: "slack-webhook"
                      description: "Slack webhook for notifications"
                      secret: "${SLACK_WEBHOOK_URL}"
                  - usernamePassword:
                      scope: GLOBAL
                      id: "bitbucket-credentials"
                      description: "Bitbucket app password"
                      username: "${BITBUCKET_USER}"
                      password: "${BITBUCKET_APP_PASSWORD}"

      tool: |
        tool:
          sonarRunnerInstallation:
            installations:
              - name: "SonarScanner"
                properties:
                  - installSource:
                      installers:
                        - sonarRunnerInstaller:
                            id: "5.0.1.3006"

  # Plugins
  installPlugins:
    - kubernetes:4203.v1dd44f5b_1cf9
    - workflow-aggregator:596.v8c21c963d92d
    - git:5.2.1
    - configuration-as-code:1775.v810dc950b_514
    - job-dsl:1.87
    - pipeline-utility-steps:2.16.2
    - sonar:2.17.1
    - slack:684.v833089650554
    - role-strategy:689.v731678c3e0eb_
    - oic-auth:4.220.v25a_4a_a_0fa_e65
    - pipeline-stage-view:2.34
    - blueocean:1.27.11
    - bitbucket:223.vd12f2bca5430
    - generic-webhook-trigger:2.2.1
    - pipeline-graph-view:264.v2532bb0e85de
    - prometheus:773.v3b_62d8178eec

  # Prometheus metrics
  prometheus:
    enabled: true
    serviceMonitorAdditionalLabels:
      release: kube-prometheus-stack

  # Ingress
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: alb
      alb.ingress.kubernetes.io/scheme: internal
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/certificate-arn: "${ACM_CERT_ARN}"
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
      alb.ingress.kubernetes.io/group.name: platform-internal
    hostName: jenkins.internal.novamart.com

  # Pod anti-affinity (controller is singleton but keep config for upgrade safety)
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: jenkins
            topologyKey: topology.kubernetes.io/zone

# Agent configuration
agent:
  enabled: true
  # Service account for agents (separate from controller)
  serviceAccount:
    name: jenkins-agent
    annotations:
      eks.amazonaws.com/role-arn: "arn:aws:iam::role/novamart-production-jenkins-agent"

# Backup
backup:
  enabled: true
  schedule: "0 2 * * *"
  destination: "s3://novamart-production-jenkins-artifacts/backups/"
```

---

## 3. JENKINS AGENT IMAGES

### `jenkins/agents/Dockerfile.golang`
```dockerfile
# Go build agent
FROM golang:1.22-bookworm AS base

# Build tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    curl \
    jq \
    unzip \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# golangci-lint
RUN curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh \
    | sh -s -- -b /usr/local/bin v1.57.2

# Trivy
RUN curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
    | sh -s -- -b /usr/local/bin v0.50.4

# Cosign (image signing)
RUN curl -sSfL https://github.com/sigstore/cosign/releases/download/v2.2.3/cosign-linux-amd64 \
    -o /usr/local/bin/cosign && chmod +x /usr/local/bin/cosign

# Kustomize
RUN curl -sSfL https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh \
    | bash -s -- 5.3.0 /usr/local/bin

# yq (YAML processing)
RUN curl -sSfL https://github.com/mikefarah/yq/releases/download/v4.42.1/yq_linux_amd64 \
    -o /usr/local/bin/yq && chmod +x /usr/local/bin/yq

# AWS CLI v2
RUN curl -sSfL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip -q awscliv2.zip && ./aws/install && rm -rf aws awscliv2.zip

# Non-root user
RUN groupadd -g 1000 jenkins && useradd -u 1000 -g 1000 -m jenkins
USER 1000:1000
WORKDIR /home/jenkins

# Go module cache warm (optional — can mount as PVC)
ENV GOPATH=/home/jenkins/go
ENV GOCACHE=/home/jenkins/.cache/go-build
```

### `jenkins/agents/Dockerfile.java`
```dockerfile
# Java/Spring Boot build agent
FROM eclipse-temurin:17-jdk-jammy AS base

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    curl \
    jq \
    unzip \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Maven
ARG MAVEN_VERSION=3.9.6
RUN curl -sSfL "https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz" \
    | tar xz -C /opt && ln -s /opt/apache-maven-${MAVEN_VERSION}/bin/mvn /usr/local/bin/mvn

# Trivy
RUN curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
    | sh -s -- -b /usr/local/bin v0.50.4

# Cosign
RUN curl -sSfL https://github.com/sigstore/cosign/releases/download/v2.2.3/cosign-linux-amd64 \
    -o /usr/local/bin/cosign && chmod +x /usr/local/bin/cosign

# Kustomize + yq + AWS CLI (same as Go agent)
RUN curl -sSfL https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh \
    | bash -s -- 5.3.0 /usr/local/bin
RUN curl -sSfL https://github.com/mikefarah/yq/releases/download/v4.42.1/yq_linux_amd64 \
    -o /usr/local/bin/yq && chmod +x /usr/local/bin/yq
RUN curl -sSfL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip -q awscliv2.zip && ./aws/install && rm -rf aws awscliv2.zip

# Non-root
RUN groupadd -g 1000 jenkins && useradd -u 1000 -g 1000 -m jenkins
USER 1000:1000
WORKDIR /home/jenkins

# Maven settings for Artifactory/proxy
ENV MAVEN_OPTS="-Xmx1024m -XX:MaxMetaspaceSize=256m"
```

---

## 4. JENKINS SHARED LIBRARY — THE GOLDEN PATH

### `shared-library/vars/novamartPipeline.groovy`
```groovy
#!/usr/bin/env groovy

/**
 * NovaMart Golden Path Pipeline
 *
 * Usage in application Jenkinsfile:
 *
 * novamartPipeline(
 *   serviceName: 'payment-service',
 *   language: 'golang',           // golang | java | python
 *   team: 'payments',
 *   slackChannel: '#payments-eng',
 *   sonarqubeProject: 'novamart-payment-service',
 *   deployEnvironments: ['dev', 'staging', 'production'],
 *   productionStrategy: 'canary', // canary | blue-green | rolling
 *   canarySteps: [10, 30, 60, 100],
 *   runIntegrationTests: true,
 * )
 */
def call(Map config) {
    // Merge defaults
    def defaults = readYaml(text: libraryResource('com/novamart/pipeline-defaults.yaml'))
    config = defaults + config

    // Validate required fields
    assert config.serviceName : "serviceName is required"
    assert config.language : "language is required"
    assert config.team : "team is required"

    def imageTag = ""
    def imageDigest = ""
    def fullImageUri = ""
    def ecrRegistry = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
    def ecrRepo = "novamart/${config.serviceName}"
    def gitopsRepo = "https://bitbucket.org/novamart/app-gitops.git"

    // Determine agent image based on language
    def agentImage = getAgentImage(config.language)

    pipeline {
        agent none

        options {
            timeout(time: 60, unit: 'MINUTES')
            timestamps()
            buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
            disableConcurrentBuilds(abortPrevious: true)
        }

        environment {
            AWS_REGION      = 'us-east-1'
            AWS_ACCOUNT_ID  = credentials('aws-account-id')
            SONAR_HOST_URL  = 'https://sonarqube.internal.novamart.com'
        }

        stages {
            // ================================================
            // STAGE 1: CHECKOUT + METADATA
            // ================================================
            stage('Checkout') {
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'small')
                    }
                }
                steps {
                    checkout scm

                    script {
                        // Generate image tag: sha-<short_hash>
                        def gitCommit = sh(script: 'git rev-parse --short=8 HEAD', returnStdout: true).trim()
                        def gitBranch = env.BRANCH_NAME ?: env.GIT_BRANCH?.replaceAll('origin/', '')
                        imageTag = "sha-${gitCommit}"

                        // For release branches, also tag with version
                        if (gitBranch ==~ /^release\/.*/) {
                            def version = gitBranch.replace('release/', '')
                            imageTag = "v${version}-${gitCommit}"
                        }

                        env.IMAGE_TAG = imageTag
                        env.GIT_COMMIT_SHORT = gitCommit
                        env.GIT_AUTHOR = sh(script: 'git log -1 --format=%an', returnStdout: true).trim()

                        currentBuild.displayName = "#${BUILD_NUMBER} - ${imageTag}"
                        currentBuild.description = "Branch: ${gitBranch} | Author: ${env.GIT_AUTHOR}"
                    }
                }
            }

            // ================================================
            // STAGE 2: BUILD + UNIT TEST
            // ================================================
            stage('Build & Test') {
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'medium')
                    }
                }
                steps {
                    container('build') {
                        script {
                            switch (config.language) {
                                case 'golang':
                                    sh '''
                                        go mod download
                                        go build -ldflags="-s -w -X main.version=${IMAGE_TAG}" -o bin/${SERVICE_NAME} ./cmd/...
                                        go test -race -coverprofile=coverage.out -covermode=atomic ./...
                                        go test -bench=. -benchmem ./... > bench.txt || true
                                    '''
                                    break
                                case 'java':
                                    sh '''
                                        mvn clean verify \
                                            -Dmaven.test.failure.ignore=false \
                                            -Djacoco.destFile=target/jacoco.exec \
                                            -B -ntp
                                    '''
                                    break
                                case 'python':
                                    sh '''
                                        pip install -r requirements.txt -r requirements-dev.txt
                                        pytest --cov=src --cov-report=xml:coverage.xml \
                                               --junitxml=test-results.xml -v
                                    '''
                                    break
                            }
                        }
                    }
                }
                post {
                    always {
                        // Archive test results
                        junit allowEmptyResults: true, testResults: '**/test-results*.xml,**/surefire-reports/*.xml'
                        // Archive coverage
                        archiveArtifacts allowEmptyArchive: true, artifacts: '**/coverage.*,**/jacoco.exec'
                    }
                }
            }

            // ================================================
            // STAGE 3: SONARQUBE ANALYSIS
            // ================================================
            stage('SonarQube Analysis') {
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'medium')
                    }
                }
                steps {
                    container('build') {
                        withSonarQubeEnv('SonarQube') {
                            script {
                                qualityGate(
                                    language: config.language,
                                    projectKey: config.sonarqubeProject ?: "novamart-${config.serviceName}",
                                    serviceName: config.serviceName
                                )
                            }
                        }
                    }
                }
            }

            // ================================================
            // STAGE 4: QUALITY GATE
            // ================================================
            stage('Quality Gate') {
                steps {
                    timeout(time: 10, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }

            // ================================================
            // STAGE 5: BUILD CONTAINER IMAGE (Kaniko)
            // ================================================
            stage('Build Image') {
                when {
                    anyOf {
                        branch 'main'
                        branch 'release/*'
                        branch 'hotfix/*'
                    }
                }
                agent {
                    kubernetes {
                        yaml kanikoPodYaml()
                    }
                }
                steps {
                    container('kaniko') {
                        script {
                            fullImageUri = "${ecrRegistry}/${ecrRepo}:${imageTag}"

                            sh """
                                /kaniko/executor \
                                    --context=\${WORKSPACE} \
                                    --dockerfile=\${WORKSPACE}/Dockerfile \
                                    --destination=${fullImageUri} \
                                    --cache=true \
                                    --cache-repo=${ecrRegistry}/${ecrRepo}/cache \
                                    --cache-ttl=168h \
                                    --snapshot-mode=redo \
                                    --compressed-caching=false \
                                    --image-name-with-digest-file=/tmp/image-digest
                            """

                            imageDigest = readFile('/tmp/image-digest').trim().split('@')[1]
                            env.IMAGE_DIGEST = imageDigest
                            echo "Built image: ${fullImageUri}@${imageDigest}"
                        }
                    }
                }
            }

            // ================================================
            // STAGE 6: SECURITY SCAN (Image)
            // ================================================
            stage('Security Scan') {
                when {
                    anyOf {
                        branch 'main'
                        branch 'release/*'
                        branch 'hotfix/*'
                    }
                }
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'medium')
                    }
                }
                steps {
                    container('build') {
                        script {
                            securityScan(
                                imageUri: fullImageUri,
                                serviceName: config.serviceName,
                                failOnCritical: true,
                                failOnHigh: (env.BRANCH_NAME == 'main')
                            )
                        }
                    }
                }
            }

            // ================================================
            // STAGE 7: SIGN IMAGE (Cosign)
            // ================================================
            stage('Sign Image') {
                when {
                    anyOf {
                        branch 'main'
                        branch 'release/*'
                    }
                }
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'small')
                    }
                }
                steps {
                    container('build') {
                        sh """
                            # Keyless signing with IRSA (Sigstore + OIDC)
                            COSIGN_EXPERIMENTAL=1 cosign sign \
                                --yes \
                                --annotations "team=${config.team}" \
                                --annotations "pipeline=${env.BUILD_URL}" \
                                --annotations "git-commit=${env.GIT_COMMIT_SHORT}" \
                                ${fullImageUri}@${imageDigest}
                        """
                    }
                }
            }

            // ================================================
            // STAGE 8: DEPLOY TO DEV
            // ================================================
            stage('Deploy Dev') {
                when {
                    branch 'main'
                }
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'small')
                    }
                }
                steps {
                    container('build') {
                        script {
                            deployToEnvironment(
                                environment: 'dev',
                                serviceName: config.serviceName,
                                imageTag: imageTag,
                                gitopsRepo: gitopsRepo,
                                autoSync: true
                            )
                        }
                    }
                }
            }

            // ================================================
            // STAGE 9: INTEGRATION TESTS (Dev)
            // ================================================
            stage('Integration Tests') {
                when {
                    allOf {
                        branch 'main'
                        expression { config.runIntegrationTests }
                    }
                }
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'medium')
                    }
                }
                steps {
                    container('build') {
                        script {
                            // Wait for dev deployment to be healthy
                            sh """
                                echo "Waiting for deployment rollout in dev..."
                                sleep 30
                                # ArgoCD health check via API
                                curl -sf "https://argocd.internal.novamart.com/api/v1/applications/${config.serviceName}-dev" \
                                    -H "Authorization: Bearer \${ARGOCD_TOKEN}" \
                                    | jq -e '.status.health.status == "Healthy"'
                            """

                            switch (config.language) {
                                case 'golang':
                                    sh 'go test -tags=integration -v ./tests/integration/...'
                                    break
                                case 'java':
                                    sh 'mvn verify -P integration-test -B -ntp'
                                    break
                                case 'python':
                                    sh 'pytest tests/integration/ -v --junitxml=integration-results.xml'
                                    break
                            }
                        }
                    }
                }
                post {
                    always {
                        junit allowEmptyResults: true, testResults: '**/integration-results*.xml'
                    }
                }
            }

            // ================================================
            // STAGE 10: DEPLOY TO STAGING
            // ================================================
            stage('Deploy Staging') {
                when {
                    branch 'main'
                }
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'small')
                    }
                }
                steps {
                    container('build') {
                        script {
                            deployToEnvironment(
                                environment: 'staging',
                                serviceName: config.serviceName,
                                imageTag: imageTag,
                                gitopsRepo: gitopsRepo,
                                autoSync: true
                            )
                        }
                    }
                }
            }

            // ================================================
            // STAGE 11: STAGING SOAK + SMOKE TEST
            // ================================================
            stage('Staging Validation') {
                when {
                    branch 'main'
                }
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'small')
                    }
                }
                steps {
                    container('build') {
                        script {
                            echo "Waiting for staging soak period (5 minutes)..."
                            sleep(300)

                            // Check error rate in Prometheus
                            sh """
                                ERROR_RATE=\$(curl -sf 'http://thanos-query-frontend.observability.svc:9090/api/v1/query' \
                                    --data-urlencode 'query=sum(rate(response_total{service="${config.serviceName}",namespace="novamart-${config.team}",status_code=~"5.."}[5m])) / sum(rate(response_total{service="${config.serviceName}",namespace="novamart-${config.team}"}[5m]))' \
                                    | jq -r '.data.result[0].value[1] // "0"')

                                echo "Staging error rate: \${ERROR_RATE}"

                                if (( \$(echo "\${ERROR_RATE} > 0.01" | bc -l) )); then
                                    echo "ERROR: Staging error rate > 1%. Blocking production deployment."
                                    exit 1
                                fi
                            """
                        }
                    }
                }
            }

            // ================================================
            // STAGE 12: PRODUCTION APPROVAL
            // ================================================
            stage('Production Approval') {
                when {
                    branch 'main'
                }
                steps {
                    script {
                        notifySlack(
                            channel: config.slackChannel,
                            message: "🚀 *${config.serviceName}* `${imageTag}` ready for production. <${env.BUILD_URL}|Approve in Jenkins>",
                            color: 'warning'
                        )

                        timeout(time: 4, unit: 'HOURS') {
                            input(
                                message: "Deploy ${config.serviceName}:${imageTag} to PRODUCTION?",
                                ok: 'Deploy to Production',
                                submitter: 'platform-eng,${config.team}-leads',
                                parameters: [
                                    string(name: 'CHANGE_TICKET', defaultValue: '', description: 'Jira change ticket (required)'),
                                    booleanParam(name: 'SKIP_CANARY', defaultValue: false, description: 'Skip canary (emergency only)')
                                ]
                            )
                        }
                    }
                }
            }

            // ================================================
            // STAGE 13: DEPLOY TO PRODUCTION
            // ================================================
            stage('Deploy Production') {
                when {
                    branch 'main'
                }
                agent {
                    kubernetes {
                        yaml agentPodYaml(agentImage, 'small')
                    }
                }
                steps {
                    container('build') {
                        script {
                            notifySlack(
                                channel: '#deployments',
                                message: "🔵 *${config.serviceName}* `${imageTag}` deploying to production (${config.productionStrategy})",
                                color: 'info'
                            )

                            deployToEnvironment(
                                environment: 'production',
                                serviceName: config.serviceName,
                                imageTag: imageTag,
                                gitopsRepo: gitopsRepo,
                                autoSync: false,  // Manual sync in prod
                                strategy: config.productionStrategy,
                                canarySteps: config.canarySteps
                            )
                        }
                    }
                }
            }
        }

        post {
            success {
                script {
                    notifySlack(
                        channel: config.slackChannel,
                        message: "✅ *${config.serviceName}* `${imageTag}` pipeline succeeded",
                        color: 'good'
                    )
                }
            }
            failure {
                script {
                    notifySlack(
                        channel: config.slackChannel,
                        message: "❌ *${config.serviceName}* `${imageTag}` pipeline FAILED at stage: ${env.STAGE_NAME}. <${env.BUILD_URL}|View>",
                        color: 'danger'
                    )
                }
            }
            unstable {
                script {
                    notifySlack(
                        channel: config.slackChannel,
                        message: "⚠️ *${config.serviceName}* `${imageTag}` pipeline unstable (test failures). <${env.BUILD_URL}|View>",
                        color: 'warning'
                    )
                }
            }
        }
    }
}

// ============================================================
// HELPER FUNCTIONS
// ============================================================

def getAgentImage(String language) {
    def registry = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
    switch (language) {
        case 'golang':  return "${registry}/novamart/jenkins-agent-golang:1.22"
        case 'java':    return "${registry}/novamart/jenkins-agent-java:17"
        case 'python':  return "${registry}/novamart/jenkins-agent-python:3.12"
        default:        error("Unsupported language: ${language}")
    }
}

def agentPodYaml(String image, String size) {
    def resources = [
        small:  [cpu: '500m',  memory: '512Mi',  limitMem: '1Gi'],
        medium: [cpu: '1',     memory: '1Gi',    limitMem: '2Gi'],
        large:  [cpu: '2',     memory: '2Gi',    limitMem: '4Gi'],
    ]
    def r = resources[size]

    return """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins-agent: 'true'
spec:
  serviceAccountName: jenkins-agent
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
    - name: build
      image: ${image}
      command: ['sleep']
      args: ['infinity']
      resources:
        requests:
          cpu: '${r.cpu}'
          memory: '${r.memory}'
        limits:
          memory: '${r.limitMem}'
      volumeMounts:
        - name: go-cache
          mountPath: /home/jenkins/go/pkg/mod
        - name: build-cache
          mountPath: /home/jenkins/.cache
  tolerations:
    - key: dedicated
      operator: Equal
      value: ci
      effect: NoSchedule
  volumes:
    - name: go-cache
      emptyDir: {}
    - name: build-cache
      emptyDir: {}
"""
}

def kanikoPodYaml() {
    return """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins-agent: 'true'
spec:
  serviceAccountName: jenkins-agent
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:v1.22.0-debug
      command: ['sleep']
      args: ['infinity']
      resources:
        requests:
          cpu: '1'
          memory: '1Gi'
        limits:
          memory: '2Gi'
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker
  tolerations:
    - key: dedicated
      operator: Equal
      value: ci
      effect: NoSchedule
  volumes:
    - name: kaniko-secret
      projected:
        sources:
          - secret:
              name: ecr-credentials
              items:
                - key: .dockerconfigjson
                  path: config.json
"""
}
```

### `shared-library/vars/securityScan.groovy`
```groovy
#!/usr/bin/env groovy

def call(Map config) {
    def imageUri = config.imageUri
    def serviceName = config.serviceName
    def failOnCritical = config.failOnCritical ?: true
    def failOnHigh = config.failOnHigh ?: false

    echo "=== Security Scan: ${imageUri} ==="

    // ------------------------------------------------
    // Trivy — Vulnerability scan
    // ------------------------------------------------
    def trivySeverity = failOnHigh ? 'HIGH,CRITICAL' : 'CRITICAL'

    sh """
        # Login to ECR for image pull
        aws ecr get-login-password --region \${AWS_REGION} | \
            docker login --username AWS --password-stdin \${AWS_ACCOUNT_ID}.dkr.ecr.\${AWS_REGION}.amazonaws.com 2>/dev/null || true

        # Scan for vulnerabilities
        trivy image \
            --severity ${trivySeverity} \
            --exit-code 0 \
            --format json \
            --output trivy-results.json \
            ${imageUri}

        # Human-readable table
        trivy image \
            --severity ${trivySeverity} \
            --format table \
            ${imageUri}

        # Fail on critical/high
        trivy image \
            --severity ${trivySeverity} \
            --exit-code 1 \
            --ignore-unfixed \
            ${imageUri}
    """

    // ------------------------------------------------
    // Trivy — Secret scan
    // ------------------------------------------------
    sh """
        trivy fs \
            --scanners secret \
            --exit-code 1 \
            --severity HIGH,CRITICAL \
            .
    """

    // ------------------------------------------------
    // Trivy — IaC misconfig scan (if Dockerfiles/K8s manifests present)
    // ------------------------------------------------
    sh """
        trivy config \
            --exit-code 0 \
            --severity HIGH,CRITICAL \
            --format table \
            . || true
    """

    // Archive results
    archiveArtifacts allowEmptyArchive: true, artifacts: 'trivy-results.json'

    echo "=== Security scan passed for ${serviceName} ==="
}
```

### `shared-library/vars/qualityGate.groovy`
```groovy
#!/usr/bin/env groovy

def call(Map config) {
    def language = config.language
    def projectKey = config.projectKey
    def serviceName = config.serviceName

    echo "=== SonarQube Analysis: ${projectKey} ==="

    switch (language) {
        case 'golang':
            sh """
                sonar-scanner \
                    -Dsonar.projectKey=${projectKey} \
                    -Dsonar.projectName=${serviceName} \
                    -Dsonar.sources=. \
                    -Dsonar.exclusions='**/*_test.go,**/vendor/**,**/mocks/**' \
                    -Dsonar.tests=. \
                    -Dsonar.test.inclusions='**/*_test.go' \
                    -Dsonar.go.coverage.reportPaths=coverage.out \
                    -Dsonar.go.tests.reportPaths=test-results.json \
                    -Dsonar.qualitygate.wait=false
            """
            break

        case 'java':
            sh """
                mvn sonar:sonar \
                    -Dsonar.projectKey=${projectKey} \
                    -Dsonar.projectName=${serviceName} \
                    -Dsonar.java.coveragePlugin=jacoco \
                    -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                    -Dsonar.qualitygate.wait=false \
                    -B -ntp
            """
            break

        case 'python':
            sh """
                sonar-scanner \
                    -Dsonar.projectKey=${projectKey} \
                    -Dsonar.projectName=${serviceName} \
                    -Dsonar.sources=src \
                    -Dsonar.tests=tests \
                    -Dsonar.python.coverage.reportPaths=coverage.xml \
                    -Dsonar.python.xunit.reportPath=test-results.xml \
                    -Dsonar.qualitygate.wait=false
            """
            break
    }
}
```

### `shared-library/vars/deployToEnvironment.groovy`
```groovy
#!/usr/bin/env groovy

def call(Map config) {
    def environment = config.environment
    def serviceName = config.serviceName
    def imageTag = config.imageTag
    def gitopsRepo = config.gitopsRepo
    def autoSync = config.autoSync ?: false

    echo "=== Deploying ${serviceName}:${imageTag} to ${environment} ==="

    // Clone GitOps repo
    dir('gitops') {
        git(
            url: gitopsRepo,
            credentialsId: 'bitbucket-credentials',
            branch: 'main'
        )

        // Update image tag in Kustomize overlay
        def overlayPath = "services/${serviceName}/overlays/${environment}"

        sh """
            cd ${overlayPath}

            # Update image tag in kustomization.yaml
            kustomize edit set image \
                novamart/${serviceName}=\${AWS_ACCOUNT_ID}.dkr.ecr.\${AWS_REGION}.amazonaws.com/novamart/${serviceName}:${imageTag}

            # Also update the image tag annotation for tracking
            yq eval '.commonAnnotations."novamart.com/image-tag" = "${imageTag}"' -i kustomization.yaml
            yq eval '.commonAnnotations."novamart.com/deployed-by" = "jenkins/${env.BUILD_URL}"' -i kustomization.yaml
            yq eval '.commonAnnotations."novamart.com/deployed-at" = "$(date -u +%Y-%m-%dT%H:%M:%SZ)"' -i kustomization.yaml

            # Verify kustomize build works
            kustomize build . > /dev/null

            # Commit and push
            git config user.email "jenkins@novamart.com"
            git config user.name "Jenkins CI"
            git add .
            git commit -m "deploy(${environment}): ${serviceName}:${imageTag}

            Pipeline: ${env.BUILD_URL}
            Author: ${env.GIT_AUTHOR}
            Commit: ${env.GIT_COMMIT_SHORT}"

            git push origin main
        """

        echo "GitOps repo updated. ArgoCD will detect change."

        if (autoSync) {
            // For dev/staging, wait for ArgoCD to sync
            timeout(time: 10, unit: 'MINUTES') {
                sh """
                    echo "Waiting for ArgoCD sync..."
                    for i in \$(seq 1 60); do
                        HEALTH=\$(curl -sf "https://argocd.internal.novamart.com/api/v1/applications/${serviceName}-${environment}" \
                            -H "Authorization: Bearer \${ARGOCD_TOKEN}" \
                            | jq -r '.status.health.status' 2>/dev/null || echo "Unknown")
                        SYNC=\$(curl -sf "https://argocd.internal.novamart.com/api/v1/applications/${serviceName}-${environment}" \
                            -H "Authorization: Bearer \${ARGOCD_TOKEN}" \
                            | jq -r '.status.sync.status' 2>/dev/null || echo "Unknown")

                        echo "Attempt \${i}/60: Health=\${HEALTH}, Sync=\${SYNC}"

                        if [ "\${HEALTH}" = "Healthy" ] && [ "\${SYNC}" = "Synced" ]; then
                            echo "Deployment successful!"
                            exit 0
                        fi

                        if [ "\${HEALTH}" = "Degraded" ]; then
                            echo "ERROR: Deployment degraded!"
                            exit 1
                        fi

                        sleep 10
                    done
                    echo "ERROR: Timeout waiting for deployment"
                    exit 1
                """
            }
        } else {
            echo "Production: Manual ArgoCD sync required."
            echo "ArgoCD will show OutOfSync. Team lead must approve sync."
        }
    }
}
```

### `shared-library/vars/notifySlack.groovy`
```groovy
#!/usr/bin/env groovy

def call(Map config) {
    def channel = config.channel ?: '#platform-alerts'
    def message = config.message
    def color = config.color ?: 'info'

    def colorMap = [
        good: '#36a64f',
        warning: '#f9a825',
        danger: '#d32f2f',
        info: '#1976d2'
    ]

    slackSend(
        channel: channel,
        color: colorMap[color] ?: color,
        message: message,
        tokenCredentialId: 'slack-webhook'
    )
}
```

### `shared-library/resources/com/novamart/pipeline-defaults.yaml`
```yaml
# Default pipeline configuration
# These can be overridden per-service in Jenkinsfile

deployEnvironments:
  - dev
  - staging
  - production

productionStrategy: canary

canarySteps:
  - 10
  - 30
  - 60
  - 100

runIntegrationTests: true

sonarqubeEnabled: true

trivyEnabled: true
trivyFailOnCritical: true
trivyFailOnHigh: false  # Only on main branch

cosignEnabled: true

timeouts:
  build: 15
  test: 15
  scan: 10
  deploy: 10
  approval: 240  # 4 hours
```

---

## 5. APPLICATION JENKINSFILE (What developers write)

### `Jenkinsfile` (in payment-service repo)
```groovy
// That's it. This is the entire Jenkinsfile for payment-service.
// Everything else is in the shared library.

novamartPipeline(
    serviceName: 'payment-service',
    language: 'golang',
    team: 'payments',
    slackChannel: '#payments-eng',
    sonarqubeProject: 'novamart-payment-service',
    productionStrategy: 'canary',
    canarySteps: [5, 15, 30, 60, 100],  // Extra conservative for payments
    runIntegrationTests: true,
)
```

---

## 6. GITOPS REPO STRUCTURE (Kustomize)

### Repository: `app-gitops`

```
app-gitops/
├── services/
│   ├── payment-service/
│   │   ├── base/
│   │   │   ├── kustomization.yaml
│   │   │   ├── rollout.yaml
│   │   │   ├── service.yaml
│   │   │   ├── service-active.yaml
│   │   │   ├── service-canary.yaml
│   │   │   ├── hpa.yaml
│   │   │   ├── pdb.yaml
│   │   │   ├── service-monitor.yaml
│   │   │   ├── external-secret.yaml
│   │   │   └── configmap.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   ├── kustomization.yaml
│   │       │   └── patches/
│   │       │       ├── rollout-patch.yaml
│   │       │       └── hpa-patch.yaml
│   │       ├── staging/
│   │       │   ├── kustomization.yaml
│   │       │   └── patches/
│   │       │       └── rollout-patch.yaml
│   │       └── production/
│   │           ├── kustomization.yaml
│   │           └── patches/
│   │               ├── rollout-patch.yaml
│   │               └── hpa-patch.yaml
│   ├── order-service/
│   │   ├── base/ ...
│   │   └── overlays/ ...
│   ├── user-service/
│   │   ├── base/ ...
│   │   └── overlays/ ...
│   ├── cart-service/
│   │   ├── base/ ...
│   │   └── overlays/ ...
│   ├── api-gateway/
│   │   ├── base/ ...
│   │   └── overlays/ ...
│   ├── notification-service/
│   │   ├── base/ ...
│   │   └── overlays/ ...
│   └── inventory-service/
│       ├── base/ ...
│       └── overlays/ ...
│
├── common/
│   ├── labels.yaml           # Common labels applied to all
│   └── annotations.yaml
│
└── README.md
```

### `services/payment-service/base/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

metadata:
  name: payment-service

commonLabels:
  app.kubernetes.io/name: payment-service
  app.kubernetes.io/part-of: novamart
  app.kubernetes.io/managed-by: argocd
  novamart.com/team: payments

resources:
  - rollout.yaml
  - service.yaml
  - service-active.yaml
  - service-canary.yaml
  - hpa.yaml
  - pdb.yaml
  - service-monitor.yaml
  - external-secret.yaml
  - configmap.yaml

images:
  - name: novamart/payment-service
    newName: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/novamart/payment-service
    newTag: PLACEHOLDER  # Updated by Jenkins deployToEnvironment
```

### `services/payment-service/base/rollout.yaml`
```yaml
# Argo Rollout — NOT a Deployment
# DD-CICD-002: Argo Rollouts for progressive delivery
#   - Canary with automated analysis (Prometheus metrics)
#   - Automatic rollback on SLO violation
#   - Blue-green option for services that need instant cutover
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: payment-service
  template:
    metadata:
      labels:
        app.kubernetes.io/name: payment-service
        app.kubernetes.io/version: PLACEHOLDER
        novamart.com/team: payments
      annotations:
        linkerd.io/inject: enabled
        # OTel auto-instrumentation
        instrumentation.opentelemetry.io/inject-go: "true"
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: payment-service
      terminationGracePeriodSeconds: 60

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: payment-service
          image: novamart/payment-service  # Replaced by Kustomize
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP

          env:
            - name: SERVICE_NAME
              value: "payment-service"
            - name: ENVIRONMENT
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: payment-service-config
                  key: log-level
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: payment-service-db
                  key: host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: payment-service-db
                  key: password
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: payment-service-db
                  key: dbname
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: payment-service-redis
                  key: url
            - name: PAYMENT_GATEWAY_URL
              valueFrom:
                configMapKeyRef:
                  name: payment-service-config
                  key: payment-gateway-url
            - name: PAYMENT_GATEWAY_API_KEY
              valueFrom:
                secretKeyRef:
                  name: payment-service-gateway
                  key: api-key
            # OTel config
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-agent.observability.svc.cluster.local:4317"
            - name: OTEL_SERVICE_NAME
              value: "payment-service"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "service.namespace=$(ENVIRONMENT),service.version=PLACEHOLDER"

          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              memory: 512Mi
              # NO CPU limit — avoids CFS throttling (DD from Phase 2)

          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30  # 5×30=150s max startup time

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
                # 15s preStop to allow endpoint removal propagation
                # Prevents 502s during rolling update

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
            sizeLimit: 100Mi

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: payment-service

  # ============================================================
  # CANARY STRATEGY (default — overridden per environment)
  # ============================================================
  strategy:
    canary:
      canaryService: payment-service-canary
      stableService: payment-service-active
      trafficRouting:
        plugins:
          argoproj-labs/linkerd:
            # Linkerd traffic split via TrafficSplit CRD
            rootService: payment-service
            canaryWeight: 0
      steps:
        - setWeight: 5
        - pause: { duration: 2m }
        - analysis:
            templates:
              - templateName: canary-analysis
            args:
              - name: service-name
                value: payment-service
              - name: namespace
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
        - setWeight: 15
        - pause: { duration: 3m }
        - analysis:
            templates:
              - templateName: canary-analysis
        - setWeight: 30
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: canary-analysis
        - setWeight: 60
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: canary-analysis
        - setWeight: 100
      # Auto rollback on failure
      abortScaleDownDelaySeconds: 30
      # Anti-affinity between canary and stable
      antiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          weight: 100
```

### `services/payment-service/base/service.yaml`
```yaml
# Root service — Linkerd TrafficSplit targets this
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app.kubernetes.io/name: payment-service
  ports:
    - name: http
      port: 8080
      targetPort: http
      protocol: TCP
    - name: metrics
      port: 9090
      targetPort: metrics
      protocol: TCP
```

### `services/payment-service/base/service-active.yaml`
```yaml
# Stable (active) service — points to current stable ReplicaSet
apiVersion: v1
kind: Service
metadata:
  name: payment-service-active
spec:
  selector:
    app.kubernetes.io/name: payment-service
    # Argo Rollouts injects the rollouts-pod-template-hash selector
  ports:
    - name: http
      port: 8080
      targetPort: http
```

### `services/payment-service/base/service-canary.yaml`
```yaml
# Canary service — points to canary ReplicaSet during rollout
apiVersion: v1
kind: Service
metadata:
  name: payment-service-canary
spec:
  selector:
    app.kubernetes.io/name: payment-service
  ports:
    - name: http
      port: 8080
      targetPort: http
```

### `services/payment-service/base/hpa.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service
spec:
  scaleTargetRef:
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    name: payment-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 120
```

### `services/payment-service/base/pdb.yaml`
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: payment-service
```

### `services/payment-service/base/service-monitor.yaml`
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-service
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: payment-service
  namespaceSelector:
    any: true
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
      # Enable exemplars for trace correlation
      enableHttp2: true
      metricRelabelings:
        # Drop high-cardinality Go runtime metrics we don't need
        - sourceLabels: [__name__]
          regex: "go_gc_.*|go_memstats_.*"
          action: drop
```

### `services/payment-service/base/external-secret.yaml`
```yaml
# DB credentials from Secrets Manager
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-service-db
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-service-db
    creationPolicy: Owner
  data:
    - secretKey: host
      remoteRef:
        key: novamart/${ENVIRONMENT}/payments-db
        property: host
    - secretKey: password
      remoteRef:
        key: novamart/${ENVIRONMENT}/payments-db
        property: password
    - secretKey: dbname
      remoteRef:
        key: novamart/${ENVIRONMENT}/payments-db
        property: dbname
    - secretKey: username
      remoteRef:
        key: novamart/${ENVIRONMENT}/payments-db
        property: username
---
# Redis credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-service-redis
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-service-redis
    creationPolicy: Owner
  data:
    - secretKey: url
      remoteRef:
        key: novamart/${ENVIRONMENT}/redis
        property: url
---
# Payment gateway API key
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-service-gateway
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: payment-service-gateway
    creationPolicy: Owner
  data:
    - secretKey: api-key
      remoteRef:
        key: novamart/${ENVIRONMENT}/stripe
        property: api-key
```

### `services/payment-service/base/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-service-config
data:
  log-level: "info"
  payment-gateway-url: "https://api.stripe.com/v1"
  http-read-timeout: "30s"
  http-write-timeout: "30s"
  db-max-open-conns: "25"
  db-max-idle-conns: "10"
  db-conn-max-lifetime: "5m"
```

---

### OVERLAYS — Environment-Specific Patches

### `services/payment-service/overlays/dev/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: novamart-payments

resources:
  - ../../base

commonAnnotations:
  novamart.com/environment: dev
  novamart.com/image-tag: PLACEHOLDER  # Updated by Jenkins

patches:
  - path: patches/rollout-patch.yaml
  - path: patches/hpa-patch.yaml

# Dev-specific config overrides
configMapGenerator:
  - name: payment-service-config
    behavior: merge
    literals:
      - log-level=debug
      - payment-gateway-url=https://api.stripe.com/v1/test

images:
  - name: novamart/payment-service
    newName: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/novamart/payment-service
    newTag: PLACEHOLDER
```

### `services/payment-service/overlays/dev/patches/rollout-patch.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
spec:
  replicas: 1  # Single replica in dev
  strategy:
    canary:
      # No canary in dev — immediate rollout
      steps:
        - setWeight: 100
```

### `services/payment-service/overlays/dev/patches/hpa-patch.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service
spec:
  minReplicas: 1
  maxReplicas: 3
```

### `services/payment-service/overlays/staging/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: novamart-payments

resources:
  - ../../base

commonAnnotations:
  novamart.com/environment: staging

patches:
  - path: patches/rollout-patch.yaml

configMapGenerator:
  - name: payment-service-config
    behavior: merge
    literals:
      - log-level=info
      - payment-gateway-url=https://api.stripe.com/v1/test

images:
  - name: novamart/payment-service
    newName: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/novamart/payment-service
    newTag: PLACEHOLDER
```

### `services/payment-service/overlays/staging/patches/rollout-patch.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
spec:
  replicas: 2
  strategy:
    canary:
      # Simplified canary in staging — fewer steps
      steps:
        - setWeight: 25
        - pause: { duration: 2m }
        - analysis:
            templates:
              - templateName: canary-analysis
        - setWeight: 75
        - pause: { duration: 3m }
        - analysis:
            templates:
              - templateName: canary-analysis
        - setWeight: 100
```

### `services/payment-service/overlays/production/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: novamart-payments

resources:
  - ../../base

commonAnnotations:
  novamart.com/environment: production

patches:
  - path: patches/rollout-patch.yaml
  - path: patches/hpa-patch.yaml

configMapGenerator:
  - name: payment-service-config
    behavior: merge
    literals:
      - log-level=warn
      - payment-gateway-url=https://api.stripe.com/v1

images:
  - name: novamart/payment-service
    newName: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/novamart/payment-service
    newTag: PLACEHOLDER
```

### `services/payment-service/overlays/production/patches/rollout-patch.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
spec:
  replicas: 5  # Higher baseline in production
  strategy:
    canary:
      # Full canary — extra conservative for payments
      steps:
        - setWeight: 5
        - pause: { duration: 2m }
        - analysis:
            templates:
              - templateName: canary-analysis
            args:
              - name: service-name
                value: payment-service
              - name: namespace
                value: novamart-payments
        - setWeight: 15
        - pause: { duration: 3m }
        - analysis:
            templates:
              - templateName: canary-analysis
        - setWeight: 30
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: canary-analysis
        - setWeight: 60
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: canary-analysis
        - setWeight: 100
```

### `services/payment-service/overlays/production/patches/hpa-patch.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service
spec:
  minReplicas: 5
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65  # More conservative in prod
```

---

## 7. ARGO ROLLOUTS

### `argo-rollouts/helm-values/argo-rollouts.yaml`
```yaml
# Argo Rollouts v1.6.x
# DD-CICD-002: Progressive delivery with automated analysis
#   - Canary with Prometheus-based success/latency checks
#   - Automatic rollback on SLO violation
#   - Linkerd traffic splitting (not Istio — matches our mesh choice)

controller:
  replicas: 2
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      memory: 256Mi

  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      namespace: observability
      additionalLabels:
        release: kube-prometheus-stack

  # Linkerd traffic routing plugin
  trafficRouterPlugins:
    argoproj-labs/linkerd:
      url: "https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-linkerd/releases/download/v0.4.0/traffic-router-linkerd-plugin-linux-amd64"
      sha256: ""  # Pin the sha256 in production

dashboard:
  enabled: true
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      memory: 128Mi
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: alb
      alb.ingress.kubernetes.io/scheme: internal
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/group.name: platform-internal
    hosts:
      - rollouts.internal.novamart.com

podDisruptionBudget:
  enabled: true
  minAvailable: 1

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: argo-rollouts
          topologyKey: topology.kubernetes.io/zone
```

### `argo-rollouts/analysis-templates/canary-analysis.yaml`
```yaml
# Combined analysis template — runs during every canary pause
# Checks success rate, latency, and error budget impact
apiVersion: argoproj.io/v1alpha1
kind: ClusterAnalysisTemplate
metadata:
  name: canary-analysis
spec:
  args:
    - name: service-name
    - name: namespace
      value: ""  # Injected by Rollout

  metrics:
    # ============================================================
    # CHECK 1: Success rate > 99.5%
    # ============================================================
    - name: success-rate
      interval: 60s
      count: 3             # Run 3 times
      failureLimit: 1      # Fail if any single check fails
      successCondition: "result[0] >= 0.995"
      failureCondition: "result[0] < 0.99"  # Hard fail below 99%
      provider:
        prometheus:
          address: http://thanos-query-frontend.observability.svc.cluster.local:9090
          query: |
            sum(rate(
              response_total{
                service="{{args.service-name}}",
                namespace="{{args.namespace}}",
                status_code!~"5..",
                rollouts_pod_template_hash="{{templates.podTemplateHash}}"
              }[2m]
            ))
            /
            sum(rate(
              response_total{
                service="{{args.service-name}}",
                namespace="{{args.namespace}}",
                rollouts_pod_template_hash="{{templates.podTemplateHash}}"
              }[2m]
            ))

    # ============================================================
    # CHECK 2: P99 latency < 500ms
    # ============================================================
    - name: latency-p99
      interval: 60s
      count: 3
      failureLimit: 1
      successCondition: "result[0] <= 0.5"
      failureCondition: "result[0] > 1.0"  # Hard fail above 1s
      provider:
        prometheus:
          address: http://thanos-query-frontend.observability.svc.cluster.local:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(
                http_request_duration_seconds_bucket{
                  service="{{args.service-name}}",
                  namespace="{{args.namespace}}",
                  rollouts_pod_template_hash="{{templates.podTemplateHash}}"
                }[2m]
              )) by (le)
            )

    # ============================================================
    # CHECK 3: No canary pods restarting (CrashLoopBackOff)
    # ============================================================
    - name: pod-restarts
      interval: 60s
      count: 3
      failureLimit: 0      # Zero tolerance for restarts
      successCondition: "result[0] == 0"
      provider:
        prometheus:
          address: http://thanos-query-frontend.observability.svc.cluster.local:9090
          query: |
            sum(increase(
              kube_pod_container_status_restarts_total{
                namespace="{{args.namespace}}",
                pod=~"{{args.service-name}}-.*",
                container="{{args.service-name}}"
              }[5m]
            )) or vector(0)

    # ============================================================
    # CHECK 4: Error budget not being consumed faster than 1x burn rate
    # ============================================================
    - name: error-budget-burn
      interval: 60s
      count: 3
      failureLimit: 1
      successCondition: "result[0] <= 1.0"  # Burn rate at or below 1x is acceptable
      provider:
        prometheus:
          address: http://thanos-query-frontend.observability.svc.cluster.local:9090
          query: |
            slo:burn_rate:1h{
              service="{{args.service-name}}",
              slo_name="availability"
            } or vector(0)
```

### `argo-rollouts/analysis-templates/success-rate.yaml`
```yaml
# Standalone success rate analysis — used for simpler services
apiVersion: argoproj.io/v1alpha1
kind: ClusterAnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
    - name: namespace
    - name: threshold
      value: "0.995"
  metrics:
    - name: success-rate
      interval: 60s
      count: 5
      failureLimit: 2
      successCondition: "result[0] >= asFloat(args.threshold)"
      provider:
        prometheus:
          address: http://thanos-query-frontend.observability.svc.cluster.local:9090
          query: |
            sum(rate(
              response_total{
                service="{{args.service-name}}",
                namespace="{{args.namespace}}",
                status_code!~"5.."
              }[5m]
            ))
            /
            sum(rate(
              response_total{
                service="{{args.service-name}}",
                namespace="{{args.namespace}}"
              }[5m]
            ))
```

### `argo-rollouts/analysis-templates/latency.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ClusterAnalysisTemplate
metadata:
  name: latency-check
spec:
  args:
    - name: service-name
    - name: namespace
    - name: threshold-ms
      value: "500"
  metrics:
    - name: p99-latency
      interval: 60s
      count: 5
      failureLimit: 2
      successCondition: "result[0] <= asFloat(args.threshold-ms) / 1000"
      provider:
        prometheus:
          address: http://thanos-query-frontend.observability.svc.cluster.local:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(
                http_request_duration_seconds_bucket{
                  service="{{args.service-name}}",
                  namespace="{{args.namespace}}"
                }[5m]
              )) by (le)
            )
```

---

## 8. SONARQUBE

### `sonarqube/helm-values/sonarqube.yaml`
```yaml
# SonarQube Community Edition 10.x
# DD-CICD-003: SonarQube for static analysis + quality gates
#   - Integrated with Jenkins via shared library
#   - Quality gate blocks merge on: new bugs, security hotspots, <80% coverage
#   - Branch analysis (PR decoration on Bitbucket)

edition: community

replicaCount: 1  # SonarQube Community does not support HA

image:
  tag: "10.4.1-community"

resources:
  requests:
    cpu: "1"
    memory: 2Gi
  limits:
    memory: 4Gi

# JVM tuning
jvmOpts: "-Xmx2g -Xms1g -XX:MaxMetaspaceSize=512m -XX:+UseG1GC"

persistence:
  enabled: true
  storageClass: gp3-encrypted
  size: 20Gi

# External PostgreSQL (Terraform-managed RDS)
postgresql:
  enabled: false  # Disable embedded PostgreSQL
jdbcOverwrite:
  enable: true
  jdbcUrl: "jdbc:postgresql://${SONARQUBE_DB_HOST}:5432/sonarqube"
  jdbcUsername: "sonarqube"
  jdbcSecretName: "sonarqube-db-credentials"
  jdbcSecretPasswordKey: "password"

# Ingress
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: platform-internal
    alb.ingress.kubernetes.io/healthcheck-path: /api/system/status
  hosts:
    - name: sonarqube.internal.novamart.com

# Prometheus monitoring
prometheusMonitoring:
  podMonitor:
    enabled: true
    namespace: observability

# Init sysctl — SonarQube requires vm.max_map_count=524288
initSysctl:
  enabled: true
  securityContext:
    privileged: true

# Install plugins
plugins:
  install:
    - "https://github.com/mc1arke/sonarqube-community-branch-plugin/releases/download/1.19.0/sonarqube-community-branch-plugin-1.19.0.jar"
    # Community branch plugin — enables PR analysis in Community edition

# Security context
securityContext:
  fsGroup: 1000
  runAsUser: 1000

# Quality gates and profiles bootstrapped via API after install
# See: scripts/bootstrap-sonarqube.sh
```

---

## 9. ARGOCD APPLICATION MANIFESTS (Per Service Per Environment)

### `manifests/argocd-apps/payment-service-dev.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-dev
  namespace: argocd
  labels:
    novamart.com/service: payment-service
    novamart.com/environment: dev
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: applications
  source:
    repoURL: https://bitbucket.org/novamart/app-gitops.git
    targetRevision: main
    path: services/payment-service/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: novamart-payments
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
      - PruneLast=true
      - RespectIgnoreDifferences=true
    retry:
      limit: 3
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 3m
  # Ignore Argo Rollouts-managed fields (weight changes during canary)
  ignoreDifferences:
    - group: argoproj.io
      kind: Rollout
      jsonPointers:
        - /status
    - group: ""
      kind: Service
      jsonPointers:
        - /spec/selector
```

### `manifests/argocd-apps/payment-service-staging.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-staging
  namespace: argocd
  labels:
    novamart.com/service: payment-service
    novamart.com/environment: staging
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: applications
  source:
    repoURL: https://bitbucket.org/novamart/app-gitops.git
    targetRevision: main
    path: services/payment-service/overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: novamart-payments
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
      - PruneLast=true
      - RespectIgnoreDifferences=true
    retry:
      limit: 3
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 3m
  ignoreDifferences:
    - group: argoproj.io
      kind: Rollout
      jsonPointers:
        - /status
```

### `manifests/argocd-apps/payment-service-production.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service-production
  namespace: argocd
  labels:
    novamart.com/service: payment-service
    novamart.com/environment: production
  annotations:
    # Notifications
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: "#deployments"
    notifications.argoproj.io/subscribe.on-sync-failed.slack: "#incidents"
    notifications.argoproj.io/subscribe.on-health-degraded.slack: "#payments-eng"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: applications
  source:
    repoURL: https://bitbucket.org/novamart/app-gitops.git
    targetRevision: main
    path: services/payment-service/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: novamart-payments
  # PRODUCTION: NO auto-sync. Manual sync required.
  syncPolicy:
    syncOptions:
      - CreateNamespace=false
      - PruneLast=true
      - RespectIgnoreDifferences=true
    retry:
      limit: 3
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 3m
  ignoreDifferences:
    - group: argoproj.io
      kind: Rollout
      jsonPointers:
        - /status
```

---

## 10. KARPENTER NODEPOOL FOR CI BUILDS

### `manifests/karpenter-ci-nodepool.yaml`
```yaml
# DD-CICD-004: Dedicated CI node pool
#   - Spot instances (CI builds are fault-tolerant, 60-70% savings)
#   - Taint to prevent non-CI workloads from scheduling
#   - Right-sized: c-family for build, m-family for test
#   - Max 20 nodes to cap CI cost
---
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: ci-builds
spec:
  template:
    metadata:
      labels:
        karpenter.sh/nodepool: ci-builds
        novamart.com/purpose: ci
    spec:
      requirements:
        - key: "kubernetes.io/arch"
          operator: In
          values: ["amd64"]
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["spot"]  # Spot only — CI is fault-tolerant
        - key: "node.kubernetes.io/instance-type"
          operator: In
          values:
            - "c6i.xlarge"     # 4 vCPU, 8Gi — build-optimized
            - "c6i.2xlarge"    # 8 vCPU, 16Gi
            - "c6a.xlarge"     # AMD alternative (cheaper spot)
            - "c6a.2xlarge"
            - "m6i.xlarge"     # 4 vCPU, 16Gi — for memory-heavy tests
            - "m6i.2xlarge"    # 8 vCPU, 32Gi
        - key: "topology.kubernetes.io/zone"
          operator: In
          values: ["us-east-1a", "us-east-1b", "us-east-1c"]
      taints:
        - key: dedicated
          value: ci
          effect: NoSchedule
      nodeClassRef:
        name: ci-builds
  limits:
    cpu: "80"        # Max 80 vCPU (≈20 × 4vCPU nodes)
    memory: "160Gi"
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 60s  # Aggressively reclaim — CI is bursty
  # Weight — lower priority than application nodes
  weight: 10
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: ci-builds
spec:
  amiFamily: AL2
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: novamart-production
        kubernetes.io/role/internal-elb: "1"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: novamart-production
  instanceProfile: "KarpenterNodeInstanceProfile-novamart-production"
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi  # CI needs disk for builds/caches
        volumeType: gp3
        encrypted: true
        deleteOnTermination: true
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 2
    httpTokens: required  # IMDSv2
  tags:
    Environment: production
    Purpose: ci-builds
    ManagedBy: karpenter
```

---

## 11. JENKINS NAMESPACE NETWORK POLICIES

### `manifests/jenkins-network-policies.yaml`
```yaml
# Jenkins namespace — controlled egress
---
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    novamart.com/team: platform
    # Kyverno exclusion — Jenkins agents need flexible permissions
    kyverno.io/exclude-resource-limits: "true"
---
# Default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: jenkins
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: jenkins
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
# Jenkins controller → agents (JNLP)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: controller-to-agents
  namespace: jenkins
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: jenkins
  policyTypes:
    - Egress
    - Ingress
  ingress:
    # Agents connect to controller on 50000 (JNLP) and 8080 (HTTP)
    - from:
        - podSelector:
            matchLabels:
              jenkins-agent: "true"
      ports:
        - protocol: TCP
          port: 50000
        - protocol: TCP
          port: 8080
    # ALB health checks
    - from:
        - ipBlock:
            cidr: 10.0.0.0/16
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Controller → K8s API (pod management)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
---
# Agents → various targets
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-egress
  namespace: jenkins
spec:
  podSelector:
    matchLabels:
      jenkins-agent: "true"
  policyTypes:
    - Egress
  egress:
    # → Jenkins controller (JNLP + HTTP)
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: jenkins
      ports:
        - protocol: TCP
          port: 50000
        - protocol: TCP
          port: 8080

    # → SonarQube
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: sonarqube
      ports:
        - protocol: TCP
          port: 9000

    # → ECR (HTTPS)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443

    # → Bitbucket (HTTPS — git clone)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
      ports:
        - protocol: TCP
          port: 443
        - protocol: TCP
          port: 22  # SSH git

    # → ArgoCD API (deployment health checks)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: argocd
      ports:
        - protocol: TCP
          port: 443
        - protocol: TCP
          port: 8080

    # → Thanos (staging validation — Prometheus queries)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: observability
      ports:
        - protocol: TCP
          port: 9090
```

---

## 12. PIPELINE FAILURE MODES & TROUBLESHOOTING

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CI/CD FAILURE MODE CATALOG                        │
├────────┬───────────────────────────┬─────────────────────────────────┤
│ STAGE  │ FAILURE                   │ ROOT CAUSE → FIX                │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Check  │ "Permission denied" on    │ Bitbucket app password expired  │
│ out    │ git clone                 │ → Rotate in Secrets Manager,    │
│        │                           │   ESO will sync to Jenkins cred │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Check  │ "Shallow clone" errors    │ Jenkins git plugin defaults to  │
│ out    │ during git log/blame      │ shallow. Add: depth(0) in       │
│        │                           │ checkout step for full history  │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Build  │ Agent pod stuck Pending   │ 1. No spot capacity → Karpenter │
│        │                           │    add more instance types      │
│        │                           │ 2. Taint not tolerated → check  │
│        │                           │    pod template tolerations     │
│        │                           │ 3. ResourceQuota hit in jenkins │
│        │                           │    ns → increase or clean up    │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Build  │ OOM during compilation    │ Agent resource limit too low    │
│        │ (Java mvn, Go CGO)        │ → Use 'large' agent size or    │
│        │                           │   increase limit in pod YAML   │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Build  │ Go module download slow   │ No module cache → add PVC or   │
│        │ / timeout                 │ Go proxy: GOPROXY=https://      │
│        │                           │ proxy.golang.org,direct         │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Test   │ Flaky tests fail randomly │ NOT a CI/CD problem. Do NOT    │
│        │                           │ retry. Fix the test or quarant- │
│        │                           │ ine it. Retrying masks bugs.    │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Sonar  │ Quality gate timeout      │ SonarQube processing backlog   │
│        │                           │ → Check SonarQube pod resources │
│        │                           │   and DB connection pool        │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Sonar  │ "Not enough memory" in    │ JVM heap too small → increase  │
│        │ SonarQube analysis        │ jvmOpts or agent memory limit  │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Kaniko │ "Unauthorized" push to    │ ECR token expired (12h lifetime)│
│        │ ECR                       │ → IRSA should auto-refresh.    │
│        │                           │   Check SA annotation matches  │
│        │                           │   the IAM role ARN             │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Kaniko │ Build slow / no cache     │ 1. --cache=true missing        │
│        │                           │ 2. Cache repo not created in   │
│        │                           │    ECR → create /cache repo    │
│        │                           │ 3. Dockerfile layer order wrong│
│        │                           │    → COPY deps before code     │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Kaniko │ "Cannot run as root"      │ Kaniko REQUIRES root. Pod spec │
│        │ policy violation          │ needs Kyverno PolicyException  │
│        │                           │ for kaniko container in jenkins│
│        │                           │ ns. Or use Buildah rootless.   │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Trivy  │ "CRITICAL vulnerability   │ 1. Check if fixable (--ignore- │
│        │ found" blocks pipeline    │    unfixed filters unfixed)    │
│        │                           │ 2. If in base image, update    │
│        │                           │    base image                  │
│        │                           │ 3. If false positive, add to   │
│        │                           │    .trivyignore with ticket #  │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Cosign │ Signing fails             │ 1. IRSA not configured for     │
│        │                           │    Sigstore OIDC federation    │
│        │                           │ 2. Cosign version mismatch     │
│        │                           │ 3. ECR image not found (race)  │
│        │                           │    → Add retry after push      │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ GitOps │ "Merge conflict" on git   │ Two pipelines updating same    │
│        │ push to gitops repo       │ overlay simultaneously.        │
│        │                           │ → Add retry with rebase:       │
│        │                           │   git pull --rebase && push    │
│        │                           │ → Or use per-service branches  │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ GitOps │ ArgoCD shows OutOfSync    │ 1. Image tag updated but       │
│        │ but won't sync            │    kustomize build fails       │
│        │                           │    → Validate: kustomize build │
│        │                           │ 2. Sync window blocking        │
│        │                           │    → Manual sync (always ok)   │
│        │                           │ 3. Resource diff on Rollout    │
│        │                           │    → Check ignoreDifferences   │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Canary │ Analysis fails → rollback │ WORKING AS INTENDED. Check:    │
│        │                           │ 1. Which metric failed (Argo   │
│        │                           │    Rollouts dashboard)         │
│        │                           │ 2. Is it the new code or       │
│        │                           │    environmental noise?        │
│        │                           │ 3. Adjust thresholds if too    │
│        │                           │    sensitive (but carefully)    │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Canary │ Analysis passes but users │ Canary traffic % too low to    │
│        │ report errors post-100%   │ catch issue. OR: analysis      │
│        │                           │ queries wrong metric. Verify   │
│        │                           │ rollouts_pod_template_hash     │
│        │                           │ label is present on metrics.   │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Canary │ Canary stuck (not         │ 1. Analysis never completes    │
│        │ progressing)              │    → Prometheus unreachable    │
│        │                           │    from analysis controller    │
│        │                           │ 2. No traffic to canary pods   │
│        │                           │    → Linkerd TrafficSplit not  │
│        │                           │    applied (check CRD)         │
│        │                           │ 3. progressDeadlineSeconds     │
│        │                           │    reached → Rollout aborts    │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Full   │ Pipeline succeeds but app │ 1. ExternalSecret not synced   │
│        │ CrashLoops after deploy   │    → Secret missing in env     │
│        │                           │ 2. ConfigMap wrong for env     │
│        │                           │ 3. DB migration not run        │
│        │                           │ 4. Resource limits too tight   │
│        │                           │ 5. Network policy blocks new   │
│        │                           │    dependency                  │
├────────┼───────────────────────────┼─────────────────────────────────┤
│ Full   │ "concurrent build         │ disableConcurrentBuilds        │
│        │ aborted"                  │ (abortPrevious:true) killed    │
│        │                           │ older build. Expected behavior │
│        │                           │ for rapid pushes. No fix.      │
└────────┴───────────────────────────┴─────────────────────────────────┘
```

### Emergency: Production Rollback

```bash
# Option 1: Argo Rollouts rollback (preferred — instant)
kubectl argo rollouts undo payment-service -n novamart-payments

# Option 2: Abort in-progress canary
kubectl argo rollouts abort payment-service -n novamart-payments

# Option 3: Force revert in GitOps repo (triggers ArgoCD sync)
cd app-gitops
git revert HEAD  # Reverts the image tag change
git push origin main
# ArgoCD picks up revert, syncs old image

# Option 4: Nuclear — kubectl set image directly
kubectl set image rollout/payment-service \
  payment-service=ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/novamart/payment-service:KNOWN_GOOD_TAG \
  -n novamart-payments
# WARNING: ArgoCD will show OutOfSync. Commit the fix to Git afterward.
```

---

## 13. DESIGN DECISIONS

### `docs/design-decisions.md`
```markdown
# CI/CD Design Decisions

## DD-CICD-001: CI Engine — Jenkins on EKS
**Decision:** Jenkins LTS with Kubernetes plugin, ephemeral agents
**Alternatives:** Tekton, GitHub Actions, GitLab CI, CircleCI
**Rationale:**
- Existing team expertise with Jenkins (6 engineers)
- Shared library pattern enables golden path with 10-line Jenkinsfiles
- Kubernetes plugin provides on-demand agents (no idle cost)
- JCasC enables GitOps for Jenkins config
- Migration to Tekton planned for Phase 2 (reduce controller overhead)
**Tradeoffs:**
- Jenkins controller is stateful singleton (SPOF) — mitigated by PVC + daily backups
- JVM resource overhead (~2Gi for controller)
- Plugin ecosystem is powerful but fragile (version conflicts)

## DD-CICD-002: Progressive Delivery — Argo Rollouts
**Decision:** Argo Rollouts with canary strategy (Linkerd traffic splitting)
**Alternatives:** Flagger, Istio canary, manual canary, blue-green
**Rationale:**
- Native ArgoCD integration (same GitOps workflow)
- Linkerd traffic routing plugin (matches our mesh choice from DD-PS-001)
- Prometheus-based AnalysisTemplates for automated verification
- Supports both canary and blue-green per service
- Automatic rollback on analysis failure
**Tradeoffs:**
- Requires Rollout CRD instead of Deployment (migration cost for existing services)
- Analysis templates assume app exposes standard metrics (training needed)

## DD-CICD-003: Static Analysis — SonarQube Community
**Decision:** Self-hosted SonarQube Community Edition
**Alternatives:** SonarCloud, CodeClimate, Codacy
**Rationale:**
- Data stays in our network (PCI compliance — no code leaving AWS)
- No per-user SaaS costs at scale
- Community branch plugin enables PR analysis
- Quality profiles customizable per language/team
**Tradeoffs:**
- Community Edition lacks HA (single instance)
- We operate it (patching, DB, backups)
- No advanced security analysis (SAST is Developer Edition+)

## DD-CICD-004: CI Node Pool — Spot Instances via Karpenter
**Decision:** Dedicated Karpenter NodePool for CI with spot instances
**Alternatives:** On-demand nodes, shared node pool, external CI runners
**Rationale:**
- 60-70% cost savings over on-demand
- CI builds are inherently fault-tolerant (retry on spot interruption)
- Dedicated taint prevents CI from competing with production workloads
- Karpenter consolidation reclaims nodes quickly when CI is idle
- c6i/c6a for build-heavy, m6i for memory-heavy tests
**Cost Impact:** ~$3,000/month vs ~$9,000/month on-demand (estimated 20-node avg)

## DD-CICD-005: Container Build — Kaniko (not Docker)
**Decision:** Kaniko for container image builds
**Alternatives:** Docker-in-Docker, Docker-out-of-Docker, Buildah
**Rationale:**
- No Docker daemon required (security — no privileged container)
- Runs in standard K8s pod
- Layer caching via ECR cache repo
- Well-supported by Jenkins K8s plugin
**Tradeoffs:**
- Kaniko requires root (needs Kyverno PolicyException in jenkins ns)
- Slightly slower than local Docker daemon (no local cache)
- BuildKit features (secret mounts, SSH mounts) not available

## DD-CICD-006: Image Signing — Cosign Keyless
**Decision:** Cosign with keyless signing (Sigstore OIDC)
**Alternatives:** Cosign with KMS key, Docker Content Trust, Notary v2
**Rationale:**
- Keyless = no key rotation, no key management overhead
- IRSA OIDC token used as signing identity (tied to service account)
- Kyverno can verify signatures before allowing pod creation
- Sigstore transparency log provides audit trail
**Tradeoffs:**
- Depends on Sigstore infrastructure (public Rekor + Fulcio)
- If Sigstore is down, signing fails (non-blocking for now, enforce later)

## DD-CICD-007: GitOps Structure — Kustomize (not Helm for apps)
**Decision:** Kustomize overlays for application manifests in GitOps repo
**Alternatives:** Helm charts per service, raw YAML, Jsonnet
**Rationale:**
- Kustomize is native to kubectl (no additional tooling)
- Base + overlay pattern naturally models dev/staging/production
- No template language learning curve (pure YAML)
- ArgoCD supports Kustomize natively
- Platform components use Helm (complex charts), apps use Kustomize (simpler)
**Tradeoffs:**
- Kustomize patches can be unintuitive for complex changes
- No conditional logic (unlike Helm templates)
- ConfigMapGenerator hash suffix can confuse — mitigated with annotationGenerator

## DD-CICD-008: Two-Repo Strategy (source + GitOps)
**Decision:** Separate repos: app-src (code) and app-gitops (manifests)
**Alternatives:** Single repo (code + manifests), branch-per-env
**Rationale:**
- Clear separation of concerns (code vs config)
- Jenkins updates GitOps repo → ArgoCD detects → deploys
- Avoids infinite loop: ArgoCD webhook → Jenkins → commit → ArgoCD webhook
- Different access controls (devs write code, CI writes manifests)
- Git history in GitOps repo is clean deployment log
**Tradeoffs:**
- Two repos to maintain
- GitOps repo push can race (two services deploy simultaneously)
  → Mitigated by retry with rebase in shared library
```

---

## 14. DEVELOPER ONBOARDING GUIDE

### `docs/pipeline-developer-guide.md`
```markdown
# NovaMart CI/CD Developer Guide

## Quick Start: Add CI/CD to Your Service

### Step 1: Create Jenkinsfile in your repo root
```groovy
novamartPipeline(
    serviceName: 'your-service-name',   // Must match ECR repo name
    language: 'golang',                  // golang | java | python
    team: 'your-team',                   // Matches novamart.com/team label
    slackChannel: '#your-team-eng',
)
```

### Step 2: Create Dockerfile in your repo root
Follow the template: https://wiki.novamart.com/docker-templates

Requirements:
- Multi-stage build
- Non-root user (UID 1000)
- Health check endpoints: /healthz (liveness), /readyz (readiness)
- Metrics endpoint: /metrics (Prometheus format)
- MUST NOT use :latest tag as base

### Step 3: Request platform team to:
1. Create ECR repository: `novamart/your-service-name`
2. Create GitOps manifests: `app-gitops/services/your-service-name/`
3. Create ArgoCD Applications (dev/staging/production)
4. Create Secrets Manager entries for your service
5. Create SonarQube project
6. Add Bitbucket webhook to Jenkins

### Step 4: Push to main
Pipeline runs automatically on push to main.
PR builds run on every PR (build + test + scan, no deploy).

## Pipeline Stages

| Stage | PR Build | Main Branch | What It Does |
|-------|----------|-------------|--------------|
| Checkout | ✅ | ✅ | Clone repo, generate image tag |
| Build & Test | ✅ | ✅ | Compile + unit tests |
| SonarQube | ✅ | ✅ | Static analysis + coverage |
| Quality Gate | ✅ | ✅ | Blocks if quality gate fails
| Security Scan | ✅ | ✅ | Trivy vuln + secret + IaC scan |
| Build Image | ❌ | ✅ | Kaniko → ECR |
| Sign Image | ❌ | ✅ | Cosign keyless |
| Deploy Dev | ❌ | ✅ | GitOps update, auto-sync |
| Integration Tests | ❌ | ✅ | Per-language integration suite |
| Deploy Staging | ❌ | ✅ | GitOps update, auto-sync |
| Staging Validation | ❌ | ✅ | 5min soak + Prometheus check |
| Production Approval | ❌ | ✅ | Slack notify + manual gate |
| Deploy Production | ❌ | ✅ | GitOps update, manual ArgoCD sync, canary rollout |

## Customization Options

```groovy
novamartPipeline(
    // Required
    serviceName: 'payment-service',
    language: 'golang',
    team: 'payments',

    // Optional — all have sensible defaults
    slackChannel: '#payments-eng',              // Default: #platform-alerts
    sonarqubeProject: 'custom-project-key',     // Default: novamart-<serviceName>
    productionStrategy: 'canary',               // canary | blue-green | rolling
    canarySteps: [5, 15, 30, 60, 100],          // Default: [10, 30, 60, 100]
    runIntegrationTests: true,                   // Default: true
    deployEnvironments: ['dev', 'staging', 'production'],  // Default: all three
    trivyFailOnHigh: true,                       // Default: false (only CRITICAL blocks)
)
```

## Application Requirements

### Metrics (REQUIRED)
Your service MUST expose Prometheus metrics on `/metrics`:
- `http_request_duration_seconds` (histogram) — request latency
- `response_total` (counter) with labels: `status_code`, `method`, `path`

These metrics are used by:
- Canary analysis (auto-rollback if error rate spikes)
- SLO burn rate alerting
- Grafana dashboards

Without these metrics, canary analysis will fail with "no data" and block your deployment.

### Health Endpoints (REQUIRED)
- `GET /healthz` — liveness (is the process alive?)
  - Return 200 if alive, any error if wedged
  - Do NOT check dependencies (DB, Redis) here
- `GET /readyz` — readiness (can I receive traffic?)
  - Return 200 if ready to serve
  - Check: DB connection pool healthy, warm caches loaded
  - Return 503 during startup, drain, or dependency failure

### Logging (REQUIRED)
- JSON structured logs to stdout
- Include `trace_id` field when available (for log-to-trace correlation)
- Include `service`, `level`, `msg`, `timestamp` fields minimum

Example:
```json
{"timestamp":"2024-03-15T10:30:00Z","level":"info","service":"payment-service","trace_id":"abc123def456","msg":"payment processed","amount":99.99,"currency":"USD"}
```

### Graceful Shutdown (REQUIRED)
- Handle SIGTERM
- Stop accepting new requests
- Finish in-flight requests (within 45s — pod has 60s grace period, 15s preStop)
- Close DB connections, flush buffers
- Exit cleanly

### Environment Variables
Your service receives these automatically:
| Variable | Source | Example |
|----------|--------|---------|
| SERVICE_NAME | Rollout spec | payment-service |
| ENVIRONMENT | Namespace (fieldRef) | novamart-payments |
| OTEL_EXPORTER_OTLP_ENDPOINT | Rollout spec | http://otel-agent... |
| OTEL_SERVICE_NAME | Rollout spec | payment-service |
| DB_HOST, DB_PASSWORD | ExternalSecret | (from Secrets Manager) |
| REDIS_URL | ExternalSecret | (from Secrets Manager) |

## Troubleshooting

### "My pipeline is stuck at Agent Pod"
1. Check: `kubectl get pods -n jenkins -l jenkins-agent=true`
2. If Pending: `kubectl describe pod <pod> -n jenkins` — look at Events
3. Common: no spot capacity → wait, or ask platform team to add instance types

### "Quality gate failed"
1. Go to SonarQube: https://sonarqube.internal.novamart.com
2. Find your project → Issues tab
3. Fix the issues. Common: new code coverage < 80%, new bugs, security hotspots

### "Trivy found CRITICAL vulnerability"
1. Check the Trivy report in Jenkins artifacts
2. If in base image: update your base image (e.g., golang:1.22.1 → 1.22.2)
3. If in dependency: update the dependency
4. If false positive: add CVE to `.trivyignore` with comment explaining why + Jira ticket

### "Canary rolled back automatically"
1. Go to Argo Rollouts dashboard: https://rollouts.internal.novamart.com
2. Click your rollout → Analysis tab
3. See which metric failed (success-rate, latency, pod-restarts, error-budget)
4. Check Grafana dashboard for the canary pods
5. Fix the issue and re-deploy

### "ArgoCD shows OutOfSync but won't sync"
1. Check sync window: platform changes blocked outside business hours
2. Manual sync is ALWAYS allowed: `argocd app sync <app-name>`
3. If sync fails: `argocd app get <app-name>` → check sync status/message
```

---

## 15. VALIDATION SCRIPT

### `scripts/validate-cicd.sh`
```bash
#!/usr/bin/env bash
set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

PASS=0
FAIL=0
WARN=0

check() {
  local description="$1"
  local command="$2"
  if eval "$command" &>/dev/null; then
    echo -e "${GREEN}✅ PASS${NC}: $description"
    ((PASS++))
  else
    echo -e "${RED}❌ FAIL${NC}: $description"
    ((FAIL++))
  fi
}

warn_check() {
  local description="$1"
  local command="$2"
  if eval "$command" &>/dev/null; then
    echo -e "${GREEN}✅ PASS${NC}: $description"
    ((PASS++))
  else
    echo -e "${YELLOW}⚠️  WARN${NC}: $description"
    ((WARN++))
  fi
}

echo "============================================"
echo "  NovaMart CI/CD Validation"
echo "  $(date)"
echo "============================================"
echo ""

# --- JENKINS ---
echo "--- Jenkins ---"
check "Jenkins namespace exists" \
  "kubectl get namespace jenkins"
check "Jenkins controller pod running" \
  "kubectl get pods -n jenkins -l app.kubernetes.io/name=jenkins --field-selector=status.phase=Running | grep -c Running"
check "Jenkins PVC bound" \
  "kubectl get pvc -n jenkins | grep Bound"
check "Jenkins service account has IRSA" \
  "kubectl get sa jenkins -n jenkins -o jsonpath='{.metadata.annotations.eks\\.amazonaws\\.com/role-arn}' | grep -q 'role'"
check "Jenkins agent service account has IRSA" \
  "kubectl get sa jenkins-agent -n jenkins -o jsonpath='{.metadata.annotations.eks\\.amazonaws\\.com/role-arn}' | grep -q 'role'"
check "Jenkins accessible via internal ALB" \
  "kubectl get ingress -n jenkins | grep jenkins"
warn_check "Jenkins has recent successful build" \
  "curl -sf https://jenkins.internal.novamart.com/api/json 2>/dev/null | jq -e '.jobs | length > 0'"
echo ""

# --- SONARQUBE ---
echo "--- SonarQube ---"
check "SonarQube pod running" \
  "kubectl get pods -n jenkins -l app.kubernetes.io/name=sonarqube --field-selector=status.phase=Running | grep -c Running"
check "SonarQube accessible" \
  "kubectl exec -n jenkins deployment/sonarqube -- curl -sf http://localhost:9000/api/system/status | grep -q UP"
echo ""

# --- ARGO ROLLOUTS ---
echo "--- Argo Rollouts ---"
check "Argo Rollouts controller running (2 replicas)" \
  "kubectl get pods -n argo-rollouts -l app.kubernetes.io/name=argo-rollouts --field-selector=status.phase=Running -o name | wc -l | grep -q 2"
check "Argo Rollouts dashboard running" \
  "kubectl get pods -n argo-rollouts -l app.kubernetes.io/component=rollouts-dashboard --field-selector=status.phase=Running | grep -c Running"
check "ClusterAnalysisTemplate canary-analysis exists" \
  "kubectl get clusteranalysistemplate canary-analysis"
check "ClusterAnalysisTemplate success-rate exists" \
  "kubectl get clusteranalysistemplate success-rate"
check "ClusterAnalysisTemplate latency-check exists" \
  "kubectl get clusteranalysistemplate latency-check"
echo ""

# --- KARPENTER CI NODEPOOL ---
echo "--- CI Node Pool ---"
check "Karpenter NodePool ci-builds exists" \
  "kubectl get nodepool ci-builds"
check "EC2NodeClass ci-builds exists" \
  "kubectl get ec2nodeclass ci-builds"
warn_check "CI nodes exist (may be 0 if no builds running)" \
  "kubectl get nodes -l karpenter.sh/nodepool=ci-builds 2>/dev/null || true"
echo ""

# --- NETWORK POLICIES ---
echo "--- Jenkins Network Policies ---"
check "Default deny in jenkins namespace" \
  "kubectl get networkpolicy default-deny -n jenkins"
check "DNS egress policy" \
  "kubectl get networkpolicy allow-dns -n jenkins"
check "Agent egress policy" \
  "kubectl get networkpolicy agent-egress -n jenkins"
echo ""

# --- ECR ---
echo "--- ECR Repositories ---"
for repo in payment-service order-service user-service cart-service \
            api-gateway notification-service inventory-service; do
  check "ECR repo novamart/${repo} exists" \
    "aws ecr describe-repositories --repository-names novamart/${repo} --region us-east-1"
done
echo ""

# --- GITOPS REPO ---
echo "--- GitOps Structure ---"
warn_check "GitOps repo reachable" \
  "git ls-remote https://bitbucket.org/novamart/app-gitops.git HEAD"
echo ""

# --- ARGOCD APPS ---
echo "--- ArgoCD Application Manifests ---"
for env in dev staging production; do
  check "payment-service-${env} ArgoCD app exists" \
    "kubectl get application payment-service-${env} -n argocd"
done
echo ""

# --- S3 ---
echo "--- S3 Buckets ---"
check "Jenkins artifacts bucket exists" \
  "aws s3api head-bucket --bucket novamart-production-jenkins-artifacts --region us-east-1"
echo ""

# --- SUMMARY ---
echo ""
echo "============================================"
echo "  CI/CD VALIDATION SUMMARY"
echo "============================================"
echo -e "  ${GREEN}PASSED${NC}: $PASS"
echo -e "  ${RED}FAILED${NC}: $FAIL"
echo -e "  ${YELLOW}WARNINGS${NC}: $WARN"
echo "============================================"

if [ $FAIL -gt 0 ]; then
  echo -e "${RED}CI/CD validation FAILED. Fix $FAIL issues before proceeding.${NC}"
  exit 1
elif [ $WARN -gt 0 ]; then
  echo -e "${YELLOW}CI/CD validation PASSED with $WARN warnings.${NC}"
  exit 0
else
  echo -e "${GREEN}CI/CD validation PASSED. All checks OK.${NC}"
  exit 0
fi
```

---

## 16. COMPLETE DATA FLOW — COMMIT TO PRODUCTION

```
 Developer pushes to main branch
           │
           ▼
 ┌──────────────────┐     webhook (POST)      ┌─────────────────────┐
 │   Bitbucket      │ ───────────────────────▶ │  Jenkins Controller │
 │   (app-src repo) │                          │  (EKS pod)          │
 └──────────────────┘                          └──────────┬──────────┘
                                                          │
                                               Spawns ephemeral agent pod
                                               on ci-builds NodePool (spot)
                                                          │
                                                          ▼
                                               ┌──────────────────────┐
                                               │   Jenkins Agent Pod   │
                                               │                      │
                                               │  1. git clone        │
                                               │  2. go build / mvn   │
                                               │  3. go test / mvn    │
                                               │     verify           │
                                               │  4. sonar-scanner →  │──────▶ SonarQube
                                               │  5. waitForQuality   │        (quality gate)
                                               │     Gate             │◀──────
                                               │  6. kaniko build →   │──────▶ ECR
                                               │     push to ECR      │        (immutable tag:
                                               │  7. trivy scan       │         sha-abc12345)
                                               │  8. cosign sign      │
                                               │                      │
                                               │  9. git clone        │
                                               │     app-gitops repo  │
                                               │  10. kustomize edit  │
                                               │      set image       │
                                               │      (dev overlay)   │
                                               │  11. git push        │──────▶ app-gitops repo
                                               │                      │        (new commit:
                                               │  [DEV DEPLOYED]      │         image tag updated)
                                               │                      │
                                               │  12. Wait for ArgoCD │
                                               │      health=Healthy  │
                                               │  13. Run integration │
                                               │      tests vs dev    │
                                               │                      │
                                               │  14. kustomize edit  │
                                               │      (staging overlay)│
                                               │  15. git push        │──────▶ app-gitops repo
                                               │                      │
                                               │  [STAGING DEPLOYED]  │
                                               │                      │
                                               │  16. Wait 5min soak  │
                                               │  17. Query Prometheus │
                                               │      error_rate < 1% │
                                               │                      │
                                               │  18. Slack: "Ready   │──────▶ Slack
                                               │      for prod"       │        #payments-eng
                                               │                      │
                                               │  19. input() gate    │◀────── Human approval
                                               │      (Jira ticket    │        (team lead)
                                               │       required)      │
                                               │                      │
                                               │  20. kustomize edit  │
                                               │      (prod overlay)  │
                                               │  21. git push        │──────▶ app-gitops repo
                                               │                      │
                                               │  [PROD: GITOPS       │
                                               │   UPDATED]           │
                                               │                      │
                                               │  Agent pod dies      │
                                               └──────────────────────┘
                                                          │
                                                          │ ArgoCD detects diff
                                                          │ (auto-sync OFF in prod)
                                                          ▼
                                               ┌──────────────────────┐
                                               │   ArgoCD             │
                                               │   Shows: OutOfSync   │
                                               │                      │
                                               │   Team lead clicks   │──────▶ Slack
                                               │   "Sync" in ArgoCD   │        #deployments
                                               │   UI/CLI             │        "🔵 Deploying..."
                                               └──────────┬───────────┘
                                                          │
                                                          ▼
                                               ┌──────────────────────┐
                                               │   Argo Rollouts      │
                                               │   (Canary Strategy)  │
                                               │                      │
                                               │   Step 1: 5% traffic │
                                               │   ├─ pause 2m        │
                                               │   ├─ analysis ──────▶│──▶ Prometheus
                                               │   │  (success rate,  │     (query metrics
                                               │   │   latency, burns)│      for canary pods)
                                               │   │                  │
                                               │   │  ✅ Pass         │
                                               │   │                  │
                                               │   Step 2: 15% traffic│
                                               │   ├─ pause 3m        │
                                               │   ├─ analysis ✅     │
                                               │   │                  │
                                               │   Step 3: 30% traffic│
                                               │   ├─ pause 5m        │
                                               │   ├─ analysis ✅     │
                                               │   │                  │
                                               │   Step 4: 60% traffic│
                                               │   ├─ pause 5m        │
                                               │   ├─ analysis ✅     │
                                               │   │                  │
                                               │   Step 5: 100%       │
                                               │   └─ PROMOTED ✅     │──────▶ Slack
                                               │                      │        #deployments
                                               │   Old ReplicaSet     │        "✅ Deployed!"
                                               │   scaled to 0        │
                                               └──────────────────────┘
                                               
                            IF ANALYSIS FAILS AT ANY STEP:
                                               
                                               │   ❌ analysis fails  │
                                               │   (error rate > 0.5%)│
                                               │                      │
                                               │   AUTO ROLLBACK      │──────▶ Slack
                                               │   Canary → 0%        │        #incidents
                                               │   Stable → 100%      │        "❌ Rollback!"
                                               │   (instant, no       │
                                               │    user impact)      │──────▶ PagerDuty
                                               └──────────────────────┘        (if critical)
```



## 📋 QUICK REFERENCE CARD

```
┌─────────────────────────────────────────────────────────────────┐
│                 NOVAMART CI/CD QUICK REFERENCE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  URLS:                                                           │
│    Jenkins:    https://jenkins.internal.novamart.com             │
│    SonarQube:  https://sonarqube.internal.novamart.com           │
│    ArgoCD:     https://argocd.internal.novamart.com              │
│    Rollouts:   https://rollouts.internal.novamart.com            │
│                                                                  │
│  REPOS:                                                          │
│    Source:     bitbucket.org/novamart/<service-name>             │
│    GitOps:     bitbucket.org/novamart/app-gitops                │
│    Library:    bitbucket.org/novamart/jenkins-shared-library     │
│                                                                  │
│  IMAGE TAGS:                                                     │
│    Format:     sha-<8-char-commit-hash>                         │
│    Release:    v<version>-<8-char-hash>                         │
│    ECR:        <account>.dkr.ecr.us-east-1.amazonaws.com/       │
│                novamart/<service>:<tag>                          │
│                                                                  │
│  ROLLOUT COMMANDS:                                               │
│    Status:     kubectl argo rollouts status <name> -n <ns>      │
│    Promote:    kubectl argo rollouts promote <name> -n <ns>     │
│    Abort:      kubectl argo rollouts abort <name> -n <ns>       │
│    Rollback:   kubectl argo rollouts undo <name> -n <ns>        │
│    Dashboard:  kubectl argo rollouts dashboard                   │
│                                                                  │
│  EMERGENCY ROLLBACK (fastest to slowest):                        │
│    1. kubectl argo rollouts undo <name> -n <ns>     (instant)   │
│    2. kubectl argo rollouts abort <name> -n <ns>    (in-flight) │
│    3. git revert in app-gitops + ArgoCD sync        (30-60s)    │
│    4. kubectl set image rollout/<name> ...           (manual)    │
│                                                                  │
│  PIPELINE TROUBLESHOOTING:                                       │
│    Agent stuck:    kubectl get pods -n jenkins -l jenkins-agent  │
│    Build logs:     Jenkins UI → Build → Console Output          │
│    Canary status:  kubectl argo rollouts get <name> -n <ns>     │
│    Analysis:       kubectl get analysisrun -n <ns>              │
│    ArgoCD sync:    argocd app get <app-name>                    │
│                                                                  │
│  CONTACT:                                                        │
│    Platform team:  #platform-eng (Slack)                        │
│    CI/CD issues:   JIRA: PLAT project, component: ci-cd        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```


---

## 🧪 RETENTION QUESTIONS

**Q1: The payment-service canary is at 30% traffic. The `canary-analysis` AnalysisRun reports `success-rate` metric returned `NaN`. The canary is stuck — not progressing, not rolling back. What happened and how do you fix it?**

**Q2: A developer complains their pipeline failed at "Security Scan" with: `CRITICAL: CVE-2024-1234 in golang:1.21-bookworm (fixed in 1.22)`. But their Dockerfile uses `FROM golang:1.22-bookworm`. Explain what's actually happening and the correct fix.**

**Q3: Two developers push to main within 30 seconds of each other (payment-service and order-service). The second pipeline fails at "Deploy Staging" with `git push rejected (non-fast-forward)` on the GitOps repo. Explain the root cause, the immediate fix, and the architectural fix to prevent this permanently.**

**Q4: After a production canary rollback, the team wants to re-deploy the same commit with a config change (increased DB connection pool). Walk through the exact steps — what changes, what doesn't change, and what the pipeline does differently.**


## LESSON 3 RETENTION ANSWERS

---

### Q1: Canary at 30%, `success-rate` returns `NaN`, stuck — not progressing, not rolling back.

**What happened:**

`NaN` from a Prometheus query means **division by zero** — the denominator returned 0. Looking at the AnalysisTemplate:

```promql
sum(rate(response_total{service="payment-service", ..., rollouts_pod_template_hash="<canary_hash>"}[2m]))
```

The denominator is `rate(response_total{...}[2m])` for the **canary pods specifically** (filtered by `rollouts_pod_template_hash`). If this returns 0, the division produces `NaN`.

**Three possible root causes:**

1. **The canary pods aren't receiving traffic.** Linkerd TrafficSplit CRD exists but isn't routing traffic to the canary Service. This happens when:
   - The Argo Rollouts Linkerd plugin failed to create/update the TrafficSplit resource
   - The canary Service selector doesn't match the canary pods (Rollouts injects `rollouts-pod-template-hash` into the selector — if the Service was manually defined without this, it won't match)
   - Linkerd proxy injection failed on canary pods (check `linkerd.io/inject` annotation)

2. **The `rollouts_pod_template_hash` label doesn't exist on the metrics.** The app's Prometheus metrics must include this label for canary-specific filtering. If the app doesn't propagate the pod template hash as a metric label, the query matches zero series. This is the **most common cause** — the AnalysisTemplate assumes the app or the ServiceMonitor relabels this label onto the metrics, but it wasn't configured.

3. **The metric name is wrong.** The app exposes `http_requests_total` but the template queries `response_total`. Zero series matched → 0/0 → NaN.

**Why it's stuck (not progressing AND not rolling back):**

The AnalysisTemplate has:
```yaml
successCondition: "result[0] >= 0.995"
failureCondition: "result[0] < 0.99"
```

`NaN` satisfies **neither** condition. It's not ≥ 0.995 and it's not < 0.99. NaN comparisons return false in both directions. So the AnalysisRun is **Inconclusive** — neither pass nor fail. Argo Rollouts doesn't know what to do, so it waits.

**Immediate fix:**

```bash
# 1. Check if canary pods are receiving traffic
kubectl get trafficsplit -n novamart-payments
linkerd viz stat deploy/payment-service-canary -n novamart-payments

# 2. If no traffic, check TrafficSplit
kubectl describe trafficsplit payment-service -n novamart-payments

# 3. If traffic exists but metrics empty, check metric name
kubectl exec -n novamart-payments <canary-pod> -- curl localhost:9090/metrics | grep -E "response_total|http_request"

# 4. Abort the stuck rollout
kubectl argo rollouts abort payment-service -n novamart-payments
# This rolls back to stable immediately
```

**Permanent fix (prevent recurrence):**

1. Add `inconclusive` handling to the AnalysisTemplate:
```yaml
- name: success-rate
  failureCondition: "result[0] < 0.99 || isNaN(result[0])"
  #                                       ^^^^^^^^^^^^^^^^
  # NaN now triggers failure → automatic rollback
  inconclusiveLimit: 3  # After 3 inconclusive results, fail
```

2. Ensure the metric label exists. Either:
   - App includes `rollouts_pod_template_hash` in its metrics (passed via env var from Downward API)
   - OR remove `rollouts_pod_template_hash` from the query and compare canary vs stable using separate AnalysisTemplate queries against the canary Service endpoint directly
   - OR use Linkerd metrics instead of app metrics: `response_total{deployment="payment-service-canary"}` (Linkerd proxy metrics include deployment label natively)

3. Add a "canary receiving traffic" pre-check as the first analysis metric:
```yaml
- name: canary-has-traffic
  successCondition: "result[0] > 0"
  failureCondition: "result[0] == 0"
  provider:
    prometheus:
      query: |
        sum(rate(response_total{service="payment-service",
          namespace="novamart-payments"}[2m])) > 0
```

---

### Q2: Trivy reports CVE in `golang:1.21-bookworm` but Dockerfile uses `golang:1.22-bookworm`.

**What's actually happening:**

Trivy is scanning the **final image**, not the Dockerfile. This is a **multi-stage build**. The developer's Dockerfile looks like:

```dockerfile
FROM golang:1.22-bookworm AS builder
RUN go build ...

FROM debian:bookworm-slim
COPY --from=builder /app/binary /app/binary
```

The final image is `debian:bookworm-slim`, not `golang:1.22`. But Trivy reports `golang:1.21-bookworm` — that's not even in the Dockerfile. So what's happening?

**Root cause: Kaniko layer cache is stale.** 

Our Kaniko config uses `--cache=true --cache-repo=.../cache`. The cache contains layers from a **previous build** when the Dockerfile still used `golang:1.21`. Kaniko's cache matched the layer digest for the `FROM` instruction (because the instruction text or preceding layers matched a cached entry) and reused the old layer. The final image contains residual metadata or libraries from the old `golang:1.21` base cached layer.

**OR (more likely):** The developer updated the Dockerfile but the **runtime base image** (`debian:bookworm-slim`) contains `libgcc` or `libc` packages that are the same vulnerable versions Trivy attributes to `golang:1.21-bookworm` because they share the same Debian package repository. Trivy is detecting the OS-level vulnerability in the base image packages, and the CPE (Common Platform Enumeration) matching is attributing it to `golang:1.21` because the vulnerable package (`libgcc-s1`, `libstdc++6`, etc.) has the same version string.

**Actually, the most precise answer:** Trivy detects vulnerabilities by **OS package version**, not by image name. The `golang:1.22-bookworm` image is the **build stage** — it's discarded in the final image. The runtime image (`debian:bookworm-slim` or `distroless`) contains the vulnerable package at the OS level. Trivy's output says "golang:1.21-bookworm" because that's the **CPE match** for the vulnerable library version, not because that Docker image is being used.

**Correct fix:**

```bash
# 1. Check which layer/package is actually vulnerable
trivy image --severity CRITICAL --format json <image> | \
  jq '.Results[] | select(.Vulnerabilities) | .Vulnerabilities[] | select(.VulnerabilityID=="CVE-2024-1234") | {PkgName, InstalledVersion, FixedVersion, Target}'

# This shows the actual package name and which layer it's in
```

2. If it's in the runtime base image:
```dockerfile
# Update the runtime base image
FROM debian:bookworm-slim
RUN apt-get update && apt-get upgrade -y && rm -rf /var/lib/apt/lists/*
# OR better: use distroless (no package manager, minimal attack surface)
FROM gcr.io/distroless/static-debian12:nonroot
```

3. If it's a Go stdlib vulnerability (compiled into the binary):
```dockerfile
# The Go binary itself contains the vulnerable Go runtime
# Fix: update Go version in builder stage
FROM golang:1.22.2-bookworm AS builder  # Patch version matters
```

4. If it's genuinely a false positive (Trivy CPE mismatch):
```
# .trivyignore
# CVE-2024-1234: False positive — Trivy matches golang:1.21 CPE but we use 1.22
# Runtime image is distroless, package not present. Verified manually.
# Jira: PLAT-4567
CVE-2024-1234
```

**Key lesson:** Never trust the image name in Trivy output. Always drill into the **package name and layer**. The image name is a CPE heuristic, not a guarantee.

---

### Q3: Two simultaneous pushes, second pipeline fails with `git push rejected (non-fast-forward)` on GitOps repo.

**Root cause:**

Both pipelines clone the GitOps repo at roughly the same time. Both get the same `HEAD` commit. 

```
Timeline:
T=0s   Pipeline A: git clone app-gitops (HEAD = commit X)
T=2s   Pipeline B: git clone app-gitops (HEAD = commit X)
T=15s  Pipeline A: kustomize edit set image (payment-service overlay)
T=17s  Pipeline B: kustomize edit set image (order-service overlay)  
T=20s  Pipeline A: git commit + git push → SUCCESS (HEAD now = commit Y)
T=22s  Pipeline B: git commit + git push → REJECTED
        (B's parent is X, but remote HEAD is Y — non-fast-forward)
```

Pipeline B's commit was based on commit X, but the remote has moved to commit Y. Git refuses the push because it would lose Pipeline A's commit.

**They're editing different files** (different service overlays), so there's no actual content conflict — just a git history conflict.

**Immediate fix:**

Retry the deployToEnvironment step. The shared library should already handle this:

```groovy
// In deployToEnvironment.groovy — add retry with rebase
retry(3) {
    sh """
        git pull --rebase origin main
        git push origin main
    """
}
```

`git pull --rebase` replays Pipeline B's commit on top of Pipeline A's commit Y. Since they touch different files, the rebase succeeds cleanly. Then `git push` succeeds.

If this isn't already in the shared library, manually:

```bash
# In the failed Jenkins build, re-run or:
cd gitops
git pull --rebase origin main
git push origin main
```

**Architectural fix (prevent permanently):**

**Option A: Retry with rebase in shared library (simplest, recommended)**

Already shown above. Add retry(3) with `git pull --rebase` before every push. This handles 99% of cases because different services edit different files.

```groovy
// deployToEnvironment.groovy — wrap push in retry
def maxRetries = 3
for (int i = 0; i < maxRetries; i++) {
    def exitCode = sh(script: """
        git pull --rebase origin main 2>&1
        git push origin main 2>&1
    """, returnStatus: true)
    
    if (exitCode == 0) break
    if (i == maxRetries - 1) error("Failed to push to GitOps repo after ${maxRetries} retries")
    sleep(5 * (i + 1))  // Back off: 5s, 10s, 15s
}
```

**Option B: Per-service branches with auto-merge bot**

Each pipeline pushes to `deploy/<service-name>` branch. A bot (or ArgoCD ApplicationSet) watches all `deploy/*` branches. No contention because each service has its own branch.

Downside: more complex ArgoCD config, harder to see "what's deployed" in one place.

**Option C: Atomic GitOps API (e.g., Flux Image Automation or ArgoCD Image Updater)**

Instead of pipelines pushing to Git, use ArgoCD Image Updater which watches ECR for new tags and updates the GitOps repo itself. Single writer = no contention.

```yaml
# ArgoCD Image Updater annotation
annotations:
  argocd-image-updater.argoproj.io/image-list: payment=ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/novamart/payment-service
  argocd-image-updater.argoproj.io/payment.update-strategy: digest
```

Downside: loses the explicit "Jenkins decides when to deploy" control. Better for dev/staging (auto), not production.

**Recommendation for NovaMart:** Option A (retry with rebase). It's simple, handles the race condition, and maintains the explicit CI-driven deployment model. The race is rare enough that 3 retries with backoff eliminates it practically.

---

### Q4: Re-deploy same commit with config change (increased DB connection pool) after canary rollback.

**What changes:**

The config change is in the **ConfigMap** (`payment-service-config`), not in the application code. Specifically, the `db-max-open-conns` value in the overlay's configMapGenerator.

**What doesn't change:**

- The application code (same Git commit)
- The container image (same `sha-<hash>` tag in ECR)
- The image tag in the Kustomize overlay (still points to same image)

**Exact steps:**

**Step 1: Update the config in the GitOps repo**

```bash
cd app-gitops/services/payment-service/overlays/production

# Edit the configMapGenerator to increase DB pool
# In kustomization.yaml:
```

```yaml
configMapGenerator:
  - name: payment-service-config
    behavior: merge
    literals:
      - db-max-open-conns=50     # Was 25, now 50
      - db-max-idle-conns=20     # Was 10, now 20
```

```bash
git add .
git commit -m "config(production): payment-service increase DB pool to 50

After canary rollback of sha-abc12345, root cause identified as 
connection pool exhaustion under canary traffic split.
Increasing max_open_conns from 25 to 50.

Jira: PAY-1234"

git push origin main
```

**Step 2: What happens in the pipeline**

**The pipeline does NOT run.** The change was made directly to the GitOps repo, not to the application source repo. Jenkins is triggered by webhooks from the `app-src` repo, not `app-gitops`. This is intentional — config changes skip the build/test/scan cycle because there's no new code.

**Step 3: ArgoCD detects the change**

ArgoCD polls the GitOps repo (or receives webhook), sees the ConfigMap has changed, shows the application as **OutOfSync**.

Important: Kustomize's `configMapGenerator` appends a **hash suffix** to the ConfigMap name (e.g., `payment-service-config-7f8d9c2`). When the content changes, the hash changes, which means the Rollout's pod template references a **new** ConfigMap name → Rollout detects a pod spec change → triggers a **new rollout**.

**Step 4: Manual sync in production**

```bash
# Team lead approves sync in ArgoCD
argocd app sync payment-service-production
```

**Step 5: Argo Rollouts runs canary again**

Even though the image is the same, the pod template changed (new ConfigMap hash). Argo Rollouts treats this as a new revision and runs the **full canary strategy** again:

```
5% → analysis → 15% → analysis → 30% → analysis → 60% → analysis → 100%
```

This is correct behavior — we WANT canary analysis on the config change because the increased connection pool could cause different behavior (e.g., overwhelming the database with too many connections).

**Step 6: Monitor**

The canary analysis checks the same metrics (success rate, latency, restarts, burn rate). If the higher connection pool fixes the original issue, the canary passes and promotes to 100%.

**What if we need to skip canary (emergency)?**

```bash
# Option 1: Promote immediately (skip remaining steps)
kubectl argo rollouts promote payment-service -n novamart-payments --full

# Option 2: Modify the Rollout to use rolling strategy temporarily
# (Not recommended — defeats the safety mechanism)
```

**Key distinction:** If someone wanted to re-deploy the **exact same image AND exact same config** (maybe the rollback was a false positive), they would need to trigger a restart without any manifest change:

```bash
# Force restart without GitOps change
kubectl argo rollouts restart payment-service -n novamart-payments
# This does a rolling restart but NOT a canary — it restarts existing pods
# To force a new canary, add a dummy annotation:
```

```yaml
# In overlays/production/kustomization.yaml
commonAnnotations:
  novamart.com/redeploy: "2024-03-15T14:30:00Z"  # Timestamp forces new hash
```

This changes the pod template → Rollouts runs canary → same image, same config, but verified through canary.
