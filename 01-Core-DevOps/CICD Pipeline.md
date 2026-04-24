# Phase 3: CI/CD Pipelines
## Lesson 1: Jenkins Architecture & Pipeline Fundamentals

---

## Why This Matters at NovaMart

You have 200+ microservices. Each one needs to build, test, scan, package, and deploy — reliably, securely, and fast. Jenkins is your CI engine. ArgoCD handles CD (you've seen the basics). If Jenkins is slow, flaky, or insecure, **every team in the company is blocked**. A misconfigured Jenkins is one of the most common vectors for supply chain attacks. A poorly architected Jenkins is a bottleneck that makes 50 engineers hate you personally.

Let's make sure that doesn't happen.

---

## 1. Jenkins Architecture — How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                     JENKINS CONTROLLER                          │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │ Web UI   │  │ REST API │  │ Job Queue   │  │ Config Store │  │
│  │ (Stapler)│  │          │  │ (Build Q)   │  │ ($JENKINS_   │  │
│  └──────────┘  └──────────┘  └─────┬───────┘  │  HOME/jobs/) │  │
│                                    │          └──────────────┘  │
│  ┌──────────────┐  ┌──────────┐    │    ┌───────────────────┐   │
│  │ Credentials  │  │ Plugin   │    │    │ Executor Pool     │   │
│  │ Store        │  │ Manager  │    │    │ (0 on controller  │   │
│  │ (encrypted)  │  │ (200+)   │    │    │  in production!)  │   │
│  └──────────────┘  └──────────┘    │    └───────────────────┘   │
│                                    │                            │
└────────────────────────────────────┼────────────────────────────┘
                                     │ JNLP (WebSocket/TCP)
                                     │ or SSH
                    ┌────────────────┼────────────────┐
                    │                │                │
            ┌───────▼───────┐ ┌──────▼────────┐ ┌──────▼────────┐
            │  Agent (Node) │ │  Agent (Node) │ │  Agent (Node) │
            │  Label: java  │ │  Label: docker│ │  Label: gpu   │
            │               │ │               │ │               │
            │ Executors: 2  │ │ Executors: 4  │ │ Executors: 1  │
            │ Workspace:    │ │ Workspace:    │ │ Workspace:    │
            │ /var/jenkins  │ │ /var/jenkins  │ │ /var/jenkins  │
            └───────────────┘ └───────────────┘ └───────────────┘
```

### The Controller (formerly "Master")

The controller is the **brain** — it does NOT build things in production. It:

- **Schedules builds** — matches jobs to agents based on labels
- **Stores configuration** — all job definitions live under `$JENKINS_HOME/jobs/`
- **Manages plugins** — the plugin ecosystem is Jenkins's greatest strength and greatest liability
- **Serves the UI and API** — Stapler framework (Java, old, memory-hungry)
- **Holds credentials** — encrypted with a master key stored on disk
- **Manages the build queue** — FIFO by default, with priority plugins available

**Critical production setting:**
```groovy
// Set controller executors to ZERO
// Jenkins → Manage → Nodes → Built-In Node → # of executors = 0
// WHY: Builds on controller = security risk + stability risk
// A rogue build can OOM/crash the controller, killing ALL pipelines
```

`$JENKINS_HOME` structure:
```
$JENKINS_HOME/
├── config.xml                 # Global config (security, cloud config)
├── credentials.xml            # Encrypted credentials
├── secrets/                   # Master encryption key, agent secrets
│   ├── master.key             # THIS FILE = keys to the kingdom
│   ├── hudson.util.Secret     # Encryption key for credentials
│   └── initialAdminPassword   # First-time setup
├── jobs/
│   └── my-pipeline/
│       ├── config.xml         # Job definition
│       └── builds/
│           ├── 1/             # Build #1
│           │   ├── log        # Console output
│           │   ├── build.xml  # Build metadata
│           │   └── changelog.xml
│           └── lastSuccessfulBuild → 1  # Symlink
├── nodes/                     # Agent definitions
├── plugins/                   # Installed plugins (.jpi/.hpi)
├── users/                     # User configs
├── workspace/                 # Controller workspaces (should be EMPTY)
└── war/                       # Exploded WAR file
```

### Agents (formerly "Slaves")

Agents are **disposable compute** that run actual builds. Connection methods:

| Method | How | Use Case | Pros | Cons |
|--------|-----|----------|------|------|
| **JNLP/WebSocket** | Agent connects TO controller | Cloud agents, NAT'd agents | Outbound only, firewall-friendly | Agent needs controller URL |
| **SSH** | Controller connects TO agent | Static agents, VMs | Standard, well-understood | Controller needs SSH access |
| **Kubernetes Plugin** | Dynamic pod per build | EKS/GKE/AKS | Elastic, isolated, clean | Pod startup latency (~15-30s) |
| **EC2 Plugin** | Dynamic EC2 per build | AWS heavy shops | Full VM isolation | Longer startup (~2-5min) |
| **Docker Plugin** | Container per build | Single-host dev | Quick, lightweight | Single host limitation |

### Jenkins on Kubernetes (NovaMart Pattern)

```
┌─────────────────── EKS Cluster ──────────────────────────┐
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │           jenkins namespace                        │  │
│  │                                                    │  │
│  │  ┌─────────────────────┐   ┌───────────────────┐   │  │
│  │  │  Jenkins Controller │   │  EBS Volume (gp3) │   │  │
│  │  │  (StatefulSet,      │───│  $JENKINS_HOME    │   │  │
│  │  │   1 replica)        │   │  100Gi            │   │  │
│  │  │  CPU: 2, Mem: 4Gi   │   └───────────────────┘   │  │
│  │  └────────┬────────────┘                           │  │
│  │           │ K8s API (creates pods)                 │  │
│  └───────────┼────────────────────────────────────────┘  │
│              │                                           │
│  ┌───────────▼───────────────────────────────────────┐   │
│  │       jenkins-agents namespace                    │   │
│  │                                                   │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│  │  │ Build Pod│ │ Build Pod│ │ Build Pod│  (dynamic │   │
│  │  │ maven +  │ │ go +     │ │ node +   │  on-      │   │
│  │  │ docker   │ │ trivy    │ │ chrome   │  demand)  │   │
│  │  └──────────┘ └──────────┘ └──────────┘           │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

**Kubernetes plugin pod template:**
```groovy
// This defines WHAT the build agent pod looks like
podTemplate(
    label: 'java-builder',
    serviceAccount: 'jenkins-agent',     // IRSA for AWS access
    namespace: 'jenkins-agents',
    containers: [
        containerTemplate(
            name: 'maven',
            image: 'maven:3.9.6-eclipse-temurin-17',
            command: 'cat',              // Keep alive, don't run maven immediately
            ttyEnabled: true,
            resourceRequestCpu: '1',
            resourceRequestMemory: '2Gi',
            resourceLimitCpu: '2',
            resourceLimitMemory: '4Gi'
        ),
        containerTemplate(
            name: 'docker',
            image: 'docker:24-dind',     // Docker-in-Docker
            privileged: true,            // REQUIRED for DinD (security concern)
            resourceRequestCpu: '500m',
            resourceRequestMemory: '1Gi'
        ),
        containerTemplate(
            name: 'trivy',
            image: 'aquasec/trivy:0.50.0',
            command: 'cat',
            ttyEnabled: true
        )
    ],
    volumes: [
        // Maven cache — speeds up builds dramatically
        persistentVolumeClaim(
            mountPath: '/root/.m2/repository',
            claimName: 'maven-cache',
            readOnly: false
        ),
        // DinD needs this
        emptyDirVolume(mountPath: '/var/lib/docker', memory: false)
    ]
) {
    node('java-builder') {
        stage('Build') {
            container('maven') {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Scan') {
            container('trivy') {
                sh 'trivy image --severity HIGH,CRITICAL myimage:latest'
            }
        }
    }
}
```

### How It Breaks: Controller Failures

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **OOM crash** | Controller restarts, builds lost mid-flight | Too many plugins, builds on controller, big pipelines in memory | Set executors=0, increase JVM heap `-Xmx4g`, audit plugins |
| **Disk full** | UI unresponsive, builds fail to write logs | Build logs + artifacts + workspace accumulation | Discard old builds policy, `logRotator(daysToKeep:30, numToKeep:50)`, workspace cleanup |
| **Plugin hell** | Startup failures, ClassNotFound, UI 500s | Plugin version incompatibility, untested upgrade | Plugin compatibility checker, staging Jenkins, pin plugin versions |
| **Credential leak** | Secrets in console output, logs shipped to ELK | Pipeline `echo`s secrets, bad masking | `MaskPasswordsBuildWrapper`, never use `sh "echo $SECRET"`, credential binding plugin |
| **Split brain** | Two controllers think they're active | Someone ran two instances on same `$JENKINS_HOME` | NEVER share `$JENKINS_HOME`, use HA plugin properly |
| **Slow UI** | Pages take 30+ seconds | Too many jobs (>1000 in one view), too many builds retained | Folders, views, aggressive build discard, Performance plugin |
| **master.key lost** | All credentials unrecoverable | EBS snapshot missed `secrets/` dir, migration error | Backup `secrets/` dir separately, test restore regularly |
| **Agent disconnects** | Builds stuck "waiting for agent" | Network issues, JNLP port blocked, pod evicted | WebSocket (no extra port), agent health monitoring, pod priority class |

### How It Breaks: Agent/Build Failures

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **Workspace collision** | Flaky builds, wrong files | Concurrent builds sharing workspace | `ws()` step for custom workspace, or K8s plugin (unique pod per build) |
| **Docker socket sharing** | Container escape, cross-build contamination | DooD (Docker outside of Docker) | Kaniko for image builds, or DinD with `--userns-remap` |
| **DNS resolution fails** | `mvn` can't pull deps, npm install fails | K8s DNS issue, CoreDNS overwhelmed | NodeLocal DNSCache, check pod DNS config |
| **Pod pending** | Build queued forever | No capacity, resource requests too high, node taint | Karpenter, right-size agent pods, spot instances |
| **Maven/Gradle OOM** | Build killed mid-compilation | Default JVM heap too small in container | `MAVEN_OPTS=-Xmx1g`, match to container memory limit |
| **Stale cache** | Build uses old dependency version | PVC cache has stale artifacts | Cache TTL, periodic cache wipe CronJob |

---

## 2. Jenkinsfile — Declarative vs Scripted

### Declarative Pipeline (Use This)

```groovy
// Jenkinsfile (Declarative)
pipeline {
    agent none  // Don't allocate globally — each stage picks its own

    options {
        timeout(time: 30, unit: 'MINUTES')      // Kill runaway builds
        timestamps()                              // Timestamp every log line
        disableConcurrentBuilds()                 // One build at a time (per branch)
        buildDiscarder(logRotator(                // Disk management
            numToKeepStr: '20',
            daysToKeepStr: '30',
            artifactNumToKeepStr: '5'
        ))
        retry(0)                                  // Don't retry the whole pipeline
    }

    environment {
        // Available to all stages
        REGISTRY = '123456789.dkr.ecr.us-east-1.amazonaws.com'
        APP_NAME = 'order-service'
        // credentials() helper — binds from Jenkins credential store
        SONAR_TOKEN = credentials('sonarqube-token')     // Secret text
        DOCKER_CREDS = credentials('ecr-creds')          // Username/password
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target env')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip test stage')
        string(name: 'IMAGE_TAG', defaultValue: '', description: 'Override image tag')
    }

    stages {
        stage('Checkout') {
            agent { label 'lightweight' }
            steps {
                checkout scm
                script {
                    // Determine image tag
                    env.GIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = params.IMAGE_TAG ?: "${env.BRANCH_NAME}-${env.GIT_SHORT}-${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Build & Unit Test') {
            agent {
                kubernetes {
                    yaml '''
                    apiVersion: v1
                    kind: Pod
                    spec:
                      containers:
                      - name: maven
                        image: maven:3.9.6-eclipse-temurin-17
                        command: ["cat"]
                        tty: true
                        resources:
                          requests:
                            cpu: "1"
                            memory: "2Gi"
                          limits:
                            cpu: "2"
                            memory: "4Gi"
                        volumeMounts:
                        - name: maven-cache
                          mountPath: /root/.m2/repository
                      volumes:
                      - name: maven-cache
                        persistentVolumeClaim:
                          claimName: maven-cache-pvc
                    '''
                }
            }
            steps {
                container('maven') {
                    sh '''
                        mvn clean verify \
                            -Dmaven.test.failure.ignore=false \
                            -T 1C \
                            -B \
                            --no-transfer-progress
                    '''
                    // -T 1C = 1 thread per CPU core (parallel build)
                    // -B = batch mode (no interactive)
                    // --no-transfer-progress = clean logs
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'    // Test results
                    jacoco(execPattern: '**/target/jacoco.exec') // Coverage
                }
            }
        }

        stage('SonarQube Analysis') {
            agent { label 'lightweight' }
            steps {
                withSonarQubeEnv('sonarqube-server') {  // Configured in Jenkins global
                    sh '''
                        mvn sonar:sonar \
                            -Dsonar.projectKey=${APP_NAME} \
                            -Dsonar.branch.name=${BRANCH_NAME}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                    // Blocks until SonarQube webhook fires back
                    // REQUIRES: SonarQube → Administration → Webhooks → Jenkins URL
                }
            }
        }

        stage('Build & Push Image') {
            agent {
                kubernetes {
                    yaml '''
                    apiVersion: v1
                    kind: Pod
                    spec:
                      containers:
                      - name: kaniko
                        image: gcr.io/kaniko-project/executor:debug
                        command: ["cat"]
                        tty: true
                        volumeMounts:
                        - name: docker-config
                          mountPath: /kaniko/.docker
                      volumes:
                      - name: docker-config
                        secret:
                          secretName: ecr-docker-config
                    '''
                }
            }
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                            --context=dir://\${WORKSPACE} \
                            --dockerfile=Dockerfile \
                            --destination=${REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                            --destination=${REGISTRY}/${APP_NAME}:latest-${BRANCH_NAME} \
                            --cache=true \
                            --cache-repo=${REGISTRY}/${APP_NAME}/cache \
                            --snapshot-mode=redo \
                            --compressed-caching=false
                    """
                    // --cache=true: Layer caching in registry (HUGE speed boost)
                    // No Docker daemon needed. No privileged mode.
                }
            }
        }

        stage('Security Scan') {
            parallel {    // Run scans in parallel — saves 3-5 minutes
                stage('Trivy') {
                    agent { label 'scanner' }
                    steps {
                        sh """
                            trivy image \
                                --severity HIGH,CRITICAL \
                                --exit-code 1 \
                                --ignore-unfixed \
                                --format json \
                                --output trivy-report.json \
                                ${REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        """
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'trivy-report.json'
                        }
                    }
                }
                stage('BlackDuck') {
                    agent { label 'scanner' }
                    steps {
                        sh """
                            bash <(curl -s https://detect.synopsys.com/detect.sh) \
                                --blackduck.url=\${BLACKDUCK_URL} \
                                --blackduck.api.token=\${BLACKDUCK_TOKEN} \
                                --detect.project.name=${APP_NAME} \
                                --detect.policy.check.fail.on.severities=CRITICAL
                        """
                        // BlackDuck: license compliance + vulnerability (SCA)
                        // Trivy: container image scanning (CVE)
                        // Different tools, different purposes, both required
                    }
                }
            }
        }

        stage('Update Manifests') {
            when {
                branch 'main'    // Only update manifests for main branch
            }
            agent { label 'lightweight' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'bitbucket-pat',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PAT'
                )]) {
                    sh """
                        git clone https://\${GIT_USER}:\${GIT_PAT}@bitbucket.org/novamart/k8s-manifests.git
                        cd k8s-manifests/apps/${APP_NAME}/overlays/dev
                        kustomize edit set image ${REGISTRY}/${APP_NAME}=${REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        git add .
                        git commit -m "chore: update ${APP_NAME} to ${IMAGE_TAG}"
                        git push origin main
                    """
                    // ArgoCD watches this repo → detects diff → syncs to cluster
                    // This is the GitOps bridge: CI (Jenkins) → CD (ArgoCD)
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ ${APP_NAME} ${IMAGE_TAG} pipeline succeeded"
            )
        }
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ ${APP_NAME} pipeline FAILED: ${BUILD_URL}"
            )
        }
        always {
            // Clean up workspace on the agent
            cleanWs()
        }
    }
}
```

### Declarative vs Scripted — The Real Comparison

```groovy
// SCRIPTED (old style — full Groovy power, no guardrails)
node('java-builder') {
    try {
        stage('Build') {
            checkout scm
            sh 'mvn clean package'
        }
        stage('Test') {
            sh 'mvn test'
        }
    } catch (e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        cleanWs()
    }
}

// DECLARATIVE (modern — structured, validated before execution)
pipeline {
    agent { label 'java-builder' }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
    post {
        always { cleanWs() }
    }
}
```

| Aspect | Declarative | Scripted |
|--------|-------------|----------|
| Syntax validation | Before execution | At runtime (fails mid-pipeline) |
| `post` blocks | Built-in (`always`, `success`, `failure`, `unstable`, `changed`) | Manual `try/catch/finally` |
| `when` conditions | Built-in | Manual `if` statements |
| `options` | Structured block | Manual calls scattered |
| Parallel | `parallel` block inside stage | `parallel` as a step |
| Flexibility | Limited — `script {}` block for escape hatch | Full Groovy |
| Recommended | ✅ Yes | Only when declarative can't express what you need |

**Rule:** Start with declarative. Use `script {}` escape hatch when needed. If you find yourself using `script {}` in every stage, refactor — don't switch to scripted.

---

## 3. Shared Libraries — The Golden Path

At NovaMart, you do NOT want 200 teams writing 200 different Jenkinsfiles. You want **one standard pipeline** with escape hatches.

```
shared-library repo structure:
├── vars/                          # Global functions (the API)
│   ├── standardPipeline.groovy    # The golden path
│   ├── dockerBuild.groovy         # Reusable: build + push image
│   ├── trivyScan.groovy           # Reusable: security scan
│   ├── notifySlack.groovy         # Reusable: notification
│   └── updateManifests.groovy     # Reusable: GitOps manifest update
├── src/                           # Classes (complex logic)
│   └── com/novamart/pipeline/
│       ├── Config.groovy
│       └── Utils.groovy
├── resources/                     # Static files (templates, configs)
│   └── pod-templates/
│       ├── java-builder.yaml
│       └── node-builder.yaml
└── test/                          # Unit tests (yes, test your pipelines)
```

**The golden path template:**
```groovy
// vars/standardPipeline.groovy
def call(Map config) {
    // Validate required config
    assert config.appName : "appName is required"
    assert config.language : "language is required (java, go, node)"

    pipeline {
        agent none
        options {
            timeout(time: config.timeout ?: 30, unit: 'MINUTES')
            timestamps()
            buildDiscarder(logRotator(numToKeepStr: '20'))
        }

        stages {
            stage('Build') {
                agent {
                    kubernetes {
                        yaml libraryResource("pod-templates/${config.language}-builder.yaml")
                    }
                }
                steps {
                    script {
                        switch(config.language) {
                            case 'java':
                                container('maven') {
                                    sh "mvn clean verify -T 1C -B --no-transfer-progress"
                                }
                                break
                            case 'go':
                                container('golang') {
                                    sh "go build -o app ./cmd/..."
                                    sh "go test -race -coverprofile=coverage.out ./..."
                                }
                                break
                            case 'node':
                                container('node') {
                                    sh "npm ci --prefer-offline"
                                    sh "npm test -- --ci --coverage"
                                }
                                break
                        }
                    }
                }
            }

            stage('SonarQube') {
                when { expression { config.sonar != false } }
                steps { sonarAnalysis(config.appName) }
            }

            stage('Build Image') {
                steps { dockerBuild(config.appName) }
            }

            stage('Security Scan') {
                parallel {
                    stage('Trivy')    { steps { trivyScan(config.appName) } }
                    stage('BlackDuck') {
                        when { expression { config.blackduck != false } }
                        steps { blackduckScan(config.appName) }
                    }
                }
            }

            stage('Update Manifests') {
                when { branch 'main' }
                steps { updateManifests(config.appName) }
            }
        }

        post {
            success { notifySlack(config.appName, 'SUCCESS') }
            failure { notifySlack(config.appName, 'FAILURE') }
        }
    }
}
```

**Consuming team's Jenkinsfile — 5 lines:**
```groovy
// Jenkinsfile in the order-service repo
@Library('novamart-pipeline-library@v2.3.0') _    // Pin version!

standardPipeline(
    appName: 'order-service',
    language: 'java',
    timeout: 45
)
```

**How shared libraries break:**

| Failure | Impact | Fix |
|---------|--------|-----|
| Library `@main` (unpinned) | Breaking change affects ALL 200 services at once | Always pin: `@Library('lib@v2.3.0')` |
| No tests | Broken template discovered by team A at 2 AM | Unit test with `JenkinsPipelineUnit`, integration test with test Jenkins |
| Over-abstraction | Teams can't debug, can't customize, hate platform team | Provide escape hatches, good docs, `config.customStages` hook |
| CPS serialization | `java.io.NotSerializableException` at random | Mark non-serializable code with `@NonCPS`, avoid storing non-serializable objects |

**CPS (Continuation Passing Style) — Jenkins's Cursed Execution Model:**
```groovy
// Jenkins pipelines can pause/resume (survive controller restart)
// This means ALL variables must be serializable
// This WILL bite you:

// BROKEN — JsonSlurper returns non-serializable LazyMap
def data = new groovy.json.JsonSlurper().parseText(response)

// FIXED — Use @NonCPS helper or readJSON step
@NonCPS
def parseJson(String text) {
    return new groovy.json.JsonSlurper().parseText(text)
}
// OR just use the pipeline step:
def data = readJSON(text: response)
```

---

## 4. Pipeline Security — This Is Where People Get Owned

### Credential Management

```groovy
// GOOD — credentials() binding, auto-masked in logs
withCredentials([
    string(credentialsId: 'api-key', variable: 'API_KEY'),
    usernamePassword(credentialsId: 'db-creds', 
                     usernameVariable: 'DB_USER', 
                     passwordVariable: 'DB_PASS'),
    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
    sshUserPrivateKey(credentialsId: 'deploy-key', 
                      keyFileVariable: 'SSH_KEY',
                      usernameVariable: 'SSH_USER')
]) {
    sh 'deploy.sh'  // Variables available as env vars in this block only
}

// BAD — secret leaks into console log
sh "echo ${API_KEY}"                    // Printed in plaintext
sh "curl -u ${DB_USER}:${DB_PASS} ..."  // If curl errors, full URL with creds in logs

// WORSE — secret baked into image layer
sh "docker build --build-arg SECRET=${API_KEY} ."  // Visible in docker history

// RIGHT WAY for Docker secrets during build
sh """
    echo '${API_KEY}' > /tmp/secret.txt
    docker build --secret id=mysecret,src=/tmp/secret.txt .
    rm /tmp/secret.txt
"""
// In Dockerfile: RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret
```

### Script Security & Sandbox

Jenkins runs Groovy in a **sandbox**. Unapproved method calls are blocked.

```
Pipeline: Script Security Plugin
├── Sandbox enabled by default for all Jenkinsfiles
├── Approved signatures in: Manage Jenkins → In-process Script Approval
├── Each new method call needs admin approval
└── Admins see queue of pending approvals
```

**The danger:**
```groovy
// If someone gets "script approval" access, they can approve:
new File('/etc/passwd').text           // Read any file on controller
"rm -rf /".execute()                    // Execute arbitrary commands
System.getenv()                         // Dump all environment variables

// MITIGATION:
// 1. Restrict Script Approval to senior admins only
// 2. Use Role-Based Access (Role Strategy plugin)
// 3. Move complex logic to shared libraries (trusted, no sandbox)
// 4. Audit approved scripts regularly
```

### Folder-Level RBAC (NovaMart Pattern)

```
Jenkins
├── Platform/                    (Platform team — full access)
│   ├── infra-pipelines/
│   └── shared-library-tests/
├── Order-Team/                  (Order team — their folder only)
│   ├── order-service/
│   └── order-worker/
├── Payment-Team/                (Payment team — their folder only)
│   ├── payment-service/
│   └── payment-gateway/
└── Security-Scans/              (Security team — read only for others)
```

Key plugins: **Folder-level credentials** (credentials scoped to folder, not global), **Role-Based Authorization Strategy** (matrix-based RBAC), **Audit Trail** (who did what).

---

## 5. Pipeline Optimization

### The Build Time Budget

```
Typical unoptimized Java pipeline:    Optimized:
─────────────────────────────         ────────────────────────
Checkout:        30s                  Checkout:       10s (shallow clone)
Dependency DL:   3min                 Dependency DL:  15s (PVC cache)
Compile:         2min                 Compile:        1min (parallel -T 1C)
Unit Tests:      5min                 Unit Tests:     2min (parallel surefire)
SonarQube:       3min                 SonarQube:      2min (incremental)
Docker Build:    4min                 Docker Build:   45s (layer cache, Kaniko)
Docker Push:     2min                 Docker Push:    20s (only changed layers)
Trivy Scan:      2min ─┐             Trivy+BD:       2min (parallel!)
BlackDuck:       3min ─┘             Update Manifest: 10s
Update Manifest: 30s                  ────────────────────────
─────────────────────────────         Total:          ~8min
Total:           ~25min
```

### Optimization Techniques

**1. Shallow Clone**
```groovy
checkout([
    $class: 'GitSCM',
    branches: [[name: env.BRANCH_NAME]],
    extensions: [
        [$class: 'CloneOption', depth: 1, shallow: true, noTags: true],
        [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [
            [$class: 'SparseCheckoutPath', path: 'services/order-service/']
        ]]
    ],
    userRemoteConfigs: [[url: env.REPO_URL, credentialsId: 'bitbucket-pat']]
])
```

**2. Dependency Caching (PVC in K8s)**
```yaml
# maven-cache-pvc.yaml — shared across builds
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maven-cache-pvc
  namespace: jenkins-agents
spec:
  accessModes: [ReadWriteMany]    # EFS for multi-node access
  storageClassName: efs-sc
  resources:
    requests:
      storage: 50Gi
```

**3. Parallel Stages**
```groovy
stage('Quality & Security') {
    parallel {
        stage('SonarQube')  { steps { /* ... */ } }
        stage('Trivy')      { steps { /* ... */ } }
        stage('BlackDuck')  { steps { /* ... */ } }
        stage('Lint')       { steps { /* ... */ } }
    }
}
```

**4. Conditional Execution — Don't Run What You Don't Need**
```groovy
stage('Integration Tests') {
    when {
        anyOf {
            branch 'main'
            branch 'release/*'
            changeRequest target: 'main'    // PRs to main
        }
    }
    steps { /* expensive integration tests */ }
}

stage('Deploy to Prod') {
    when {
        allOf {
            branch 'main'
            not { changeRequest() }     // Not a PR
        }
    }
    steps { /* ... */ }
}
```

**5. Abort on Stale Builds**
```groovy
// If a new commit comes in, kill the running build for the same branch
options {
    disableConcurrentBuilds(abortPrevious: true)
}
```

---

## 6. Webhook Flow — End to End

```
Developer pushes to Bitbucket
         │
         ▼
Bitbucket Webhook fires ─────────► Jenkins /github-webhook/ endpoint
  (POST with payload:                        │
   repo, branch, commit,                     ▼
   changed files)              Jenkins Multibranch Pipeline
                               scans for Jenkinsfile in branch
                                             │
                                             ▼
                               Pipeline executes ──► Build, Test, Scan
                                             │
                                             ▼
                               Jenkins fires Bitbucket Build Status API
                               (INPROGRESS → SUCCESSFUL/FAILED)
                                             │
                                             ▼
                               PR in Bitbucket shows ✅ or ❌
                               (branch protection: must pass to merge)
```

**Webhook configuration gotchas:**

| Issue | Symptom | Fix |
|-------|---------|-----|
| Webhook URL wrong | Pushes don't trigger builds | Verify: `https://jenkins.novamart.com/bitbucket-hook/` |
| Jenkins behind VPN | Bitbucket cloud can't reach webhook | Webhook relay (smee.io) or allow Bitbucket IPs |
| Multibranch not scanning | New branch not discovered | Branch source: periodic scan as backup (every 5min) |
| Too many triggers | Every push triggers, even docs | Path-based filtering: `when { changeset "services/order-service/**" }` |
| Webhook secret missing | Anyone can trigger builds | Always configure webhook secret/token |



---

## Quick Reference Card

```
JENKINS ARCHITECTURE:
  Controller = brain (UI, queue, config, credentials) — NEVER build here
  Agent = muscle (runs builds) — disposable, labeled, dynamic on K8s
  $JENKINS_HOME/secrets/master.key = encrypt key for ALL credentials
  
PIPELINE:
  Declarative > Scripted (always start declarative)
  agent none → per-stage agents (K8s pods)
  options { timeout, timestamps, buildDiscarder, disableConcurrentBuilds }
  environment { REGISTRY = '...'; CREDS = credentials('id') }
  when { branch 'main'; changeset 'path/**' }
  parallel { stage('A') {...}; stage('B') {...} }
  post { always/success/failure/unstable/changed }

SHARED LIBRARIES:
  vars/ = global functions (the API surface)
  Pin version: @Library('lib@v2.3.0') _
  Golden path: standardPipeline(appName: 'x', language: 'java')

SECURITY:
  withCredentials([...]) { } — scoped, masked
  Never: echo $SECRET, --build-arg SECRET=, sh "curl -u $user:$pass"
  Script approval = privilege escalation vector — lock it down
  Folder-level credentials > global credentials

OPTIMIZATION:
  Shallow clone (--depth 1)
  PVC cache (maven, npm, go modules)
  Parallel stages (scans, tests)
  Conditional stages (when {})
  Kill stale builds (abortPrevious: true)
  Kaniko (no Docker daemon, cached layers in registry)

CPS GOTCHA:
  JsonSlurper → @NonCPS or readJSON step
  All pipeline variables must be Serializable

GITOPS BRIDGE:
  Jenkins (CI) → builds image → updates manifest repo → ArgoCD (CD) syncs
```

---

## Retention Questions

**Q1.** You're onboarding at NovaMart. The current Jenkins setup has the controller running builds (executors = 4), all 200 services use `@Library('pipeline-lib') _` without version pinning, and credentials are stored globally. The security team is flagging risk. You're asked to write a remediation plan. What are your top 5 changes, ordered by risk severity, and what's the blast radius of each if left unfixed?

**Q2.** A developer reports their pipeline takes 22 minutes. You look at the Jenkinsfile and see: sequential stages (build → test → sonar → trivy → blackduck → docker build → push → deploy), `checkout scm` doing a full clone of a 4GB monorepo, Maven downloading dependencies fresh every build, and Docker building from scratch each time (no layer cache). Walk me through your optimization plan — what changes would you make, what's the expected time savings for each, and what's the total target build time?

**Q3.** At 3 AM, PagerDuty alerts: "Jenkins controller is down." When it comes back up after restart, you discover 15 in-flight builds were lost, 3 teams can't deploy because their builds are stuck in "pending" (no agents available), and the SonarQube quality gate webhook is returning 404. Triage this incident: what do you investigate first, what's your recovery sequence, and what changes do you implement to prevent recurrence?

**Q4.** Explain the CPS (Continuation Passing Style) execution model in Jenkins pipelines. Why does it exist? What specific problems does it cause? Give a code example that would break due to CPS and show the fix. Then explain why shared library code in `src/` is trusted (no sandbox) while Jenkinsfile code is sandboxed — and what security implication this creates.

---



### Q1: The NovaMart Remediation Plan

The current setup is a "ticking time bomb." The combination of a controller-as-worker and unpinned shared libraries means a single bad commit in the library could crash the controller and block 200 services simultaneously.

**Remediation Plan (Ordered by Risk Severity):**

| Priority | Change | Risk if Left Unfixed | Blast Radius |
| :--- | :--- | :--- | :--- |
| **1. CRITICAL** | **Move Builds to Agents** (K8s Pods) | **Systemic Outage.** A single heavy build (e.g., a massive Java compile) consumes all Controller CPU/RAM, killing the UI and API. | **100%** of all pipelines. |
| **2. HIGH** | **Credential Scoping/Vault Integration** | **Lateral Movement.** Global creds mean any developer with "Job Configure" access can steal the production DB password or AWS keys. | All production secrets across all 200 services. |
| **3. HIGH** | **Version Pinning for Shared Libs** | **Mass-Failure.** A change to `pipeline-lib` is instantly applied to all 200 services. One syntax error = 200 broken pipelines. | All active deployments. |
| **4. MEDIUM** | **Implement JCasC (Config-as-Code)** | **Configuration Drift.** Manual UI changes are untracked. If the controller dies, you cannot reliably recreate the setup. | Recovery Time Objective (RTO) increases from mins to days. |
| **5. LOW** | **Set Global Timeouts & Concurrency Limits** | **Resource Starvation.** A "zombie" build or a loop in a pipeline occupies an executor forever, blocking other teams. | Specific teams/services (Queue congestion). |

---

### Q2: The Pipeline Optimization Audit

The current pipeline is a "sequential disaster." It treats a modern CI process like a 1990s batch job.

**Optimization Plan:**

1.  **Shallow Clone (`--depth 1`):**
    *   **Change:** Replace `checkout scm` with a shallow clone. We don't need 10 years of history to build a JAR.
    *   **Saving:** ~5–8 minutes (depending on network/repo size).
2.  **Parallelize "Quality" Stages:**
    *   **Change:** Run `Sonar`, `Trivy`, and `Blackduck` in a `parallel` block. They don't depend on each other.
    *   **Saving:** $\sum(\text{times}) \rightarrow \max(\text{times})$. Saving ~5–10 minutes.
3.  **Dependency Caching (Maven/Gradle):**
    *   **Change:** Mount a persistent volume (PV) to the agent at `/root/.m2` or use a local Nexus/Artifactory proxy. Stop downloading the internet on every build.
    *   **Saving:** ~5–10 minutes.
4.  **Docker Layer Caching & BuildKit:**
    *   **Change:** Use `--cache-from` pointing to the latest image in ECR and enable BuildKit (`DOCKER_BUILDKIT=1`). Order Dockerfile to put `COPY pom.xml` before `COPY src`.
    *   **Saving:** ~5 minutes (for incremental builds).

**Expected Total Build Time:**
$22\text{m} \rightarrow \mathbf{5\text{--}8\text{m}}$.

---

### Q3: 3 AM Triage: Jenkins Controller Outage

**Investigation Order:**
1.  **Host Health:** Check CPU/RAM/Disk of the controller node. If disk is 100%, the JVM likely crashed due to an inability to write logs or the `JENKINS_HOME` directory.
2.  **K8s Agent State:** Since builds are pending, check if the Agent pods are stuck in `Pending` or `ImagePullBackOff`. The controller might be up, but the "muscle" is missing.
3.  **Webhook Connectivity:** The 404 on the SonarQube webhook suggests a **Networking/DNS change** or a **misconfigured Ingress**. If the controller IP changed during the restart, the webhook is hitting a dead end.

**Recovery Sequence:**
`Controller Health` $\rightarrow$ `Agent Provisioning` $\rightarrow$ `External Webhook Verification` $\rightarrow$ `Priority-based Build Re-triggering`.

**Prevention:**
1.  **High Availability (HA):** Move to a multi-node Jenkins setup or a managed service.
2.  **Externalized State:** Store `JENKINS_HOME` on a high-performance EFS/SSD volume to prevent disk-related crashes.
3.  **Health Checks:** Implement a `/health` check in the controller that alerts PagerDuty *before* the crash (e.g., when heap usage hits 90%).

---

### Q4: The CPS Execution Model & Security

**What is CPS?**
Jenkins pipelines are not standard Groovy scripts; they are executed in **Continuation Passing Style (CPS)**. 

**Why it exists:**
Jenkins pipelines can take hours. If the Jenkins controller restarts, you can't lose the progress of a 3-hour build. CPS allows Jenkins to "pause" a method, save its entire state (the stack, variables, and program counter) to disk as a serialized object, and "resume" it after a restart.

**The Problem: `NotSerializableException`**
Since every variable must be serializable to be saved to disk, you cannot use non-serializable Java objects (like `Socket`, `InputStream`, or certain Third-Party Library objects) across a "pause point" (any step like `sh`, `sleep`, or `echo`).

**Example of a Break:**
```groovy
node {
    def scanner = new java.util.Scanner(System.in) // Non-serializable object
    sh 'echo Hello' // PAUSE POINT: Jenkins tries to save 'scanner' to disk
    println scanner.nextLine() // CRASH: NotSerializableException
}
```
**The Fix:** Use the `@NonCPS` annotation. This tells Jenkins: *"Run this method as a standard Java method; do not try to pause it or save its state."*

**The Sandbox vs. `src/` Security Implication:**
*   **Jenkinsfile (Sandboxed):** Runs in a restricted environment. It cannot call arbitrary Java methods because a developer could write `System.exit(0)` and kill the controller. It requires an admin to "approve" specific signatures.
*   **Shared Library `src/` (Trusted):** Code in `src/` is executed **outside the sandbox**. It has full access to the JVM and Jenkins internal API.

**Security Implication:** A Shared Library is a **super-user**. If an attacker can commit code to the `pipeline-lib` repository, they can execute arbitrary code on the Jenkins Controller, steal all global credentials, and potentially gain root access to the underlying node. **Access to the Shared Library repo must be as strictly guarded as the production environment itself.**

# Phase 3: CI/CD Pipelines
## Lesson 2: CI/CD Toolchain — SonarQube, Trivy, BlackDuck, Artifactory + Advanced Pipeline Patterns

---

## Why This Matters at NovaMart

You build code. But building code is the easy part. The hard part is answering these questions **before** code reaches production:

- Is this code maintainable or will it rot? → **SonarQube**
- Does this container have known vulnerabilities? → **Trivy**
- Are we using libraries with banned licenses or known CVEs? → **BlackDuck**
- Where do built artifacts live, who can access them, how long do we keep them? → **Artifactory**

At NovaMart, a single dependency with a critical CVE in the payment service could mean PCI-DSS audit failure, regulatory fines, and front-page news. These tools aren't optional — they're compliance gates.


Q1: Remediation Plan — 4.2/5
What you nailed:

Correct priority ordering. Controller-as-worker is the highest risk — systemic, not targeted.
Credential scoping and blast radius analysis is solid.
JCasC mention is excellent — most people forget configuration drift as a risk.
Blast radius descriptions are precise and business-aware.
What you missed:

No mention of Script Approval lockdown. You flagged credentials but ignored the fact that with builds running on the controller + global credentials + no script approval restrictions, any developer can write a Jenkinsfile that calls System.getenv() or reads master.key directly. This is worse than credential scoping — it's arbitrary code execution on the brain of your CI system. Should be Priority 2 or merged into your credential item.

No audit trail. You can't scope credentials retroactively without knowing who's using what. Step 0 of the credential remediation is: enable Audit Trail plugin, log all credential access for 2 weeks, THEN scope. Otherwise you'll break pipelines blindly.

Missing: RBAC / folder isolation. You mentioned JCasC but not folder-level RBAC. Without it, Team A can modify Team B's pipelines. Combined with global credentials, that's cross-team lateral movement.

Concurrency limits as "LOW" is debatable. A zombie build occupying the controller's 4 executors IS a systemic outage since builds run on the controller. With your Priority 1 fix in place (agents), then yes it drops to LOW. But in the current broken state, it's higher.

Production consequence of the gap: You'd implement credential scoping but leave script approval wide open — an attacker writes a Jenkinsfile that bypasses credential binding entirely by reading /var/jenkins_home/secrets/master.key directly. Game over.

Q2: Pipeline Optimization — 3.8/5
What you nailed:

Correct identification of all 4 major bottlenecks (clone, sequential, deps, docker cache).
Parallel block reasoning is correct (sum → max).
Target time range is realistic (5-8min).
Dockerfile layer ordering mention is good.
What you missed:

No mention of Kaniko. You said "BuildKit + --cache-from" which requires a Docker daemon (DinD, privileged mode). The lesson explicitly covered Kaniko as the production pattern — no daemon, no privileged, registry-based caching. This is the NovaMart standard.

Sparse checkout not mentioned. Shallow clone helps with history depth but this is a 4GB monorepo. You still download all 200 services' source code. Sparse checkout (sparseCheckoutPaths) limits to just the service directory — that's where the bulk of the clone savings come from.

No abortPrevious: true. If a developer pushes 3 commits in 5 minutes, you're running 3 full pipelines. The first two are waste. This is free optimization.

No conditional stage execution. Why run BlackDuck on a feature branch? Why run integration tests on a docs change? when { changeset } and when { branch } eliminate entire stages.

Savings math is hand-wavy. "5-10 minutes" for dependency caching is a range so wide it's not useful. Be specific: fresh Maven download of Spring Boot dependencies = ~3-4min. PVC cache hit = ~15s. Saving: ~3.5min. That's how you justify infra spend to management.

Production consequence: You optimize but still run privileged DinD containers and waste CI minutes on redundant builds.

Q3: 3 AM Triage — 3.5/5
What you nailed:

Correct instinct to check host health (disk/CPU/RAM) first.
Identifying the webhook 404 as a networking/DNS/ingress issue is correct.
Recovery sequence order is logical.
HA and externalized state for prevention are appropriate.
What you missed:

You skipped the 15 lost in-flight builds. This is the most urgent user-facing issue. Which 15 builds? Were any deploying to production mid-flight? A build that was pushing a Docker image and got killed might have pushed a partial/corrupt image tag. A build that was updating the manifest repo might have left a half-committed state. You need to audit the state of each lost build, not just "re-trigger" them.

"Pending" builds — wrong diagnosis direction. You jumped to "Agent pods stuck in Pending/ImagePullBackOff." More likely: the controller just restarted, the Kubernetes plugin hasn't re-registered, or the pod templates are misconfigured after the restart. First check: kubectl get pods -n jenkins-agents — are there ANY agent pods? If zero, the issue is controller → K8s API connectivity, not agent health.

Prevention is incomplete:

No mention of $JENKINS_HOME backup strategy (EBS snapshots, scheduled)
No mention of build distributed persistence (Durable Task plugin, which allows builds to survive controller restart — this is why CPS exists)
No mention of PDBs for the Jenkins controller StatefulSet
"/health check that alerts at 90% heap" — Jenkins has a built-in health check endpoint (/login returns 200) but for JVM heap, you'd use Prometheus JMX exporter + alerting, not a custom health check
"Multi-node Jenkins" for HA is vague. Jenkins doesn't natively support multi-controller. You either mean CloudBees CI (commercial, controller-per-team), or you mean active-passive with shared storage. Saying "multi-node" without specifics suggests you're guessing.

Production consequence: You restart Jenkins, re-trigger builds, but miss that build #4532 of payment-service pushed a corrupt image tag that ArgoCD is now trying to deploy to production.

Q4: CPS Explanation — 4/5
What you nailed:

Correct explanation of WHY CPS exists (survive controller restarts during long builds).
"Pause point" concept is accurate — any pipeline step is a potential serialization boundary.
The security implication of shared library src/ is perfectly stated: "A Shared Library is a super-user."
The fix (@NonCPS) is correct.
What you missed:

Your code example is wrong. new java.util.Scanner(System.in) isn't what causes CPS issues in practice — nobody reads stdin in a Jenkins pipeline. The classic real-world break is JsonSlurper (returns LazyMap, which is non-serializable) or HttpURLConnection. The lesson explicitly covered this. Use the real-world example.

You didn't mention readJSON/readYaml pipeline steps as the preferred alternative to @NonCPS. The @NonCPS annotation has a limitation: it cannot call CPS-transformed methods (pipeline steps like sh, echo) inside it. The better fix for JSON parsing is readJSON(text: response) — a proper pipeline step that's CPS-safe.

Missing: CPS performance impact. CPS transformation makes Groovy pipelines significantly slower than native Groovy. Tight loops, heavy string manipulation, large data structures — all suffer. This is why you push logic into shell scripts (sh) or external tools rather than doing it in Groovy.

Sandbox bypass via shared library — incomplete threat model. You correctly said an attacker with commit access to the library repo can execute arbitrary code. But you didn't mention: even without direct commit access, if the library is loaded from @Library('lib') _ without version pinning, a developer who can push to the library's default branch gets the same power. This connects directly to Q1 (version pinning).
---

## 1. SonarQube — Code Quality & Security Analysis

### Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                     SonarQube Server                          │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐  │
│  │  Web Server  │  │ Compute      │  │  Search (ES)        │  │
│  │  (UI + API)  │  │ Engine (CE)  │  │  (Elasticsearch     │  │
│  │  Port 9000   │  │  Processes   │  │   embedded)         │  │
│  │              │  │  analysis    │  │  Indexes results    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬──────────────┘  │
│         │                 │                  │                │
│         └────────┬────────┴──────────────────┘                │
│                  │                                            │
│         ┌────────▼─────────┐                                  │
│         │  PostgreSQL DB   │  (Issues, metrics, config,       │
│         │  (External)      │   quality profiles, rules)       │
│         └──────────────────┘                                  │
└───────────────────────────────────────────────────────────────┘
         ▲                              │
         │ Analysis report              │ Webhook (quality gate result)
         │ (POST /api/ce/submit)        │ (POST to Jenkins)
         │                              ▼
┌────────┴────────────┐          ┌────────────────────┐
│  SonarQube Scanner  │          │     Jenkins        │
│  (runs in CI/CD)    │          │  waitForQuality    │
│                     │          │  Gate()            │
│  - Parses source    │          └────────────────────┘
│  - Runs rules       │
│  - Sends report     │
│  (NOT the server    │
│   doing the scan!)  │
└─────────────────────┘
```

**Key distinction:** The scanner runs in your CI agent, collects data, and ships it to the server. The server's Compute Engine processes the analysis asynchronously. This is why `waitForQualityGate()` exists — there's a delay between scan completion and result availability.

### What SonarQube Actually Analyzes

```
ANALYSIS DIMENSIONS:
├── Bugs           → Code that is demonstrably wrong (null dereference, 
│                    resource leak, infinite loop)
├── Vulnerabilities → Security weaknesses (SQL injection, XSS, hardcoded 
│                     secrets, path traversal, weak crypto)
├── Code Smells    → Maintainability issues (deep nesting, long methods,
│                    duplicated code, cognitive complexity)
├── Coverage       → % of code exercised by unit tests (requires 
│                    JaCoCo/Istanbul/coverage.py report import)
├── Duplications   → Duplicate code blocks across files
├── Security Hotspots → Code that MIGHT be vulnerable (needs human review)
│                       vs Vulnerabilities which ARE vulnerable
└── Technical Debt  → Estimated time to fix all issues (measured in hours/days)
```

### Quality Profiles & Rules

```
Quality Profile = set of rules activated for a language

Built-in profiles:
├── Sonar Way (default)           → ~350 rules for Java, balanced
├── Sonar Way Recommended         → Superset, more rules
└── Custom (NovaMart Standard)    → Fork of Sonar Way + company rules

Rule severity:
├── BLOCKER    → Will crash or corrupt data (must fix before merge)
├── CRITICAL   → Security vulnerability or serious bug
├── MAJOR      → Significant code smell or moderate bug
├── MINOR      → Style issue, low-impact smell
└── INFO       → Informational, no fix required

Example custom rules for NovaMart:
- BLOCKER: No System.out.println (use structured logging)
- BLOCKER: No hardcoded passwords/API keys
- CRITICAL: Cognitive complexity > 15 per method
- MAJOR: Test coverage < 80% on new code
- MINOR: TODO comments must reference a Jira ticket
```

### Quality Gates — The Pipeline Gate

```
Quality Gate = pass/fail criteria applied to analysis results

NovaMart Quality Gate ("NovaMart Production"):
┌───────────────────────────────────────────────────────┐
│  Condition                    │ Operator │ Threshold  │
├───────────────────────────────────────────────────────┤
│  New Code Coverage            │    <     │   80%      │
│  New Code Duplications        │    >     │   3%       │
│  New Bugs                     │    >     │   0        │
│  New Vulnerabilities          │    >     │   0        │
│  New Security Hotspots Review │    <     │   100%     │
│  New Code Smells (BLOCKER)    │    >     │   0        │
│  New Code Smells (CRITICAL)   │    >     │   0        │
│  Overall Code Maintainability │    <     │   A        │
└───────────────────────────────────────────────────────┘

KEY CONCEPT: "New Code" vs "Overall Code"
├── New Code = code changed in this branch/PR (since branch point)
│   → This is what you gate on. "Clean as you code."
│   → You can't fix 10 years of tech debt overnight.
├── Overall Code = entire codebase metrics
│   → Track for trending, don't gate on it
│   → Exception: maintainability rating (A-E) on overall
└── New Code Period: defined per project (previous_version, 
    number_of_days, reference_branch)
```

### Branch Analysis & PR Decoration

```
BRANCH TYPES IN SONARQUBE:
├── Main Branch     → Full analysis, all metrics stored permanently
├── Long-lived      → release/*, develop — full analysis
├── Short-lived/PR  → Feature branches, PRs
│   ├── Compared against target branch (main)
│   ├── Only NEW code is analyzed
│   ├── Results posted as PR comments (Bitbucket/GitHub integration)
│   └── Quality Gate applies to new code only
└── Reference Branch → What "new code" is measured against

PR Decoration flow:
Developer opens PR → Jenkins runs → Scanner sends analysis →
SonarQube processes → Posts inline comments on PR →
Sets commit status (✅/❌) → Quality Gate result sent via webhook →
Jenkins waitForQualityGate() passes/fails pipeline
```

### Jenkins Integration — Complete

```groovy
// Step 1: Global config (Manage Jenkins → Configure System → SonarQube servers)
//   Name: sonarqube-server
//   URL: https://sonar.novamart.com
//   Auth token: (credential ID: sonar-token)
//   Webhook: SonarQube → Administration → Webhooks → 
//            https://jenkins.novamart.com/sonarqube-webhook/

// Step 2: In Jenkinsfile
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonarqube-server') {
            // For Maven projects:
            sh '''
                mvn sonar:sonar \
                    -Dsonar.projectKey=order-service \
                    -Dsonar.projectName="Order Service" \
                    -Dsonar.branch.name=${BRANCH_NAME} \
                    -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                    -Dsonar.java.binaries=target/classes \
                    -Dsonar.qualitygate.wait=false
            '''
            // For non-Maven (Go, Python, JS):
            sh '''
                sonar-scanner \
                    -Dsonar.projectKey=cart-service \
                    -Dsonar.sources=. \
                    -Dsonar.go.coverage.reportPaths=coverage.out \
                    -Dsonar.exclusions=**/vendor/**,**/*_test.go
            '''
        }
    }
}

stage('Quality Gate') {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
            // This BLOCKS until SonarQube fires the webhook back
            // If SonarQube is slow or webhook is misconfigured → timeout
        }
    }
}
```

### How SonarQube Breaks

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **Quality Gate timeout** | Pipeline hangs 5min then fails | Webhook not configured, wrong URL, SonarQube overloaded, CE queue backed up | Verify webhook URL, check CE queue in SonarQube admin, scale CE workers |
| **False positives flood** | Developers ignore all findings because 50% are noise | Default profile too aggressive for codebase, test code being scanned | Custom quality profile, `sonar.exclusions` for test/generated code, mark FPs in UI |
| **CE queue backed up** | Analysis results delayed 30+ minutes | Too many projects, underpowered server, large monorepo | Horizontal CE scaling (Data Center Edition), incremental analysis, split monorepo projects |
| **Coverage shows 0%** | Quality gate fails on coverage | JaCoCo/Istanbul report path wrong, report not generated, wrong format | Verify `jacoco.xml` exists, correct `xmlReportPaths`, run tests BEFORE sonar |
| **Branch analysis broken** | PR decoration missing, wrong diff | `sonar.branch.name` not set, community edition (no branch support) | Developer Edition minimum for branches, verify branch parameters |
| **Elasticsearch OOM** | SonarQube UI unresponsive, search broken | ES embedded instance needs more heap, too many projects/issues | `sonar.search.javaOpts=-Xmx2g`, external ES for large installs |
| **Upgrade breaks plugins** | 500 errors after upgrade | Custom plugins incompatible with new version | Test upgrades in staging, check plugin compatibility matrix BEFORE upgrade |
| **Stale results** | Dashboard shows old analysis | Webhook fires but Jenkins already moved on, `waitForQualityGate` was removed | Always use `waitForQualityGate`, verify CE processed the correct analysis |

### SonarQube Editions — What You Actually Get

```
Community (Free):        15+ languages, basic analysis, NO branch analysis
                         → Single main branch only. Useless for PR workflows.
                         
Developer ($):           Branch analysis, PR decoration, Bitbucket/GitHub/GitLab
                         integration, taint analysis (data flow for security)
                         → MINIMUM for any real team.
                         
Enterprise ($$):         Portfolio management, project transfer, regulatory 
                         reports (PDF), application-level view across services
                         → Multi-team orgs, compliance reporting.
                         
Data Center ($$$):       High availability (multiple CE nodes), component 
                         redundancy
                         → 200+ service shops like NovaMart.
```

---

## 2. Trivy — Container & Infrastructure Scanning

### What Trivy Scans

```
TRIVY SCAN TARGETS:
├── Container Images (image)     → CVEs in OS packages + language deps
├── Filesystem (fs)              → Scan local project directory
├── Git Repository (repo)        → Scan remote repo directly
├── Kubernetes (k8s)             → Scan running cluster for misconfigs
├── AWS Account (aws)            → Scan AWS resources for misconfigs
├── SBOM (sbom)                  → Scan Software Bill of Materials
└── Config files (config)        → IaC misconfigurations
    ├── Terraform (.tf)
    ├── Dockerfile
    ├── Kubernetes manifests
    ├── Helm charts
    └── CloudFormation
```

This is why Trivy is dominant — it's not just a container scanner. It's a **universal security scanner**.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Trivy CLI                          │
│                                                     │
│  ┌───────────────┐   ┌─────────────────────────┐    │
│  │  Scanner      │   │  Vulnerability DB       │    │
│  │  ├── OS pkg   │   │  (trivy-db)             │    │
│  │  ├── Language │   │  ├── NVD (NIST)         │    │
│  │  ├── License  │   │  ├── Red Hat OVAL       │    │
│  │  ├── Secret   │   │  ├── Debian Security    │    │
│  │  ├── Misconfig│   │  ├── Alpine SecDB       │    │
│  │  └── SBOM     │   │  ├── GitHub Advisory    │    │
│  └───────────────┘   │  └── And 10+ more       │    │
│                      └─────────────────────────┘    │
│                                                     │
│  DB download: ghcr.io/aquasecurity/trivy-db         │
│  Updated every 6 hours                              │
│  Cached locally: ~/.cache/trivy/                    │
└─────────────────────────────────────────────────────┘
```

### Trivy in CI Pipeline — Complete Reference

```bash
# ── IMAGE SCANNING (most common) ──

# Basic: fail on HIGH and CRITICAL
trivy image \
    --severity HIGH,CRITICAL \
    --exit-code 1 \
    123456789.dkr.ecr.us-east-1.amazonaws.com/order-service:v1.2.3

# Production pipeline scan:
trivy image \
    --severity HIGH,CRITICAL \
    --exit-code 1 \                    # Non-zero exit = pipeline fails
    --ignore-unfixed \                 # Skip CVEs with no patch available
    --format json \                    # Machine-readable output
    --output trivy-report.json \       # Save for archiving
    --timeout 10m \                    # Don't hang forever
    --skip-db-update \                 # Use pre-cached DB (CI optimization)
    --cache-dir /tmp/trivy-cache \     # Explicit cache location
    --ignorefile .trivyignore \        # Suppress accepted risks
    ${REGISTRY}/${APP_NAME}:${IMAGE_TAG}

# ── FILESYSTEM SCANNING (scan source code deps) ──

# Scans package-lock.json, pom.xml, go.sum, requirements.txt, etc.
trivy fs \
    --severity HIGH,CRITICAL \
    --exit-code 1 \
    --scanners vuln,secret \           # Also find hardcoded secrets!
    --format table \
    .

# ── CONFIG SCANNING (IaC misconfigurations) ──

# Scan Terraform files
trivy config \
    --severity HIGH,CRITICAL \
    --exit-code 1 \
    --format json \
    --output misconfig-report.json \
    ./terraform/

# Scan Dockerfiles
trivy config \
    --severity HIGH,CRITICAL \
    --exit-code 1 \
    ./Dockerfile

# Scan Kubernetes manifests
trivy config \
    --severity HIGH,CRITICAL \
    --exit-code 1 \
    ./k8s-manifests/

# ── KUBERNETES CLUSTER SCANNING ──

# Scan running cluster (needs kubeconfig)
trivy k8s \
    --report summary \
    --severity HIGH,CRITICAL \
    cluster

# ── SBOM GENERATION ──

# Generate SBOM (Software Bill of Materials)
trivy image \
    --format cyclonedx \               # Or spdx-json
    --output sbom.json \
    ${REGISTRY}/${APP_NAME}:${IMAGE_TAG}

# Why SBOM matters: when Log4Shell drops, you grep your SBOMs
# to find every service using log4j in 5 minutes, not 5 days
```

### .trivyignore — Accepted Risk

```bash
# .trivyignore — CVEs we've reviewed and accepted
# Format: one CVE ID per line, optional comment

# Accepted: low-risk, mitigated by WAF, no patch available
# Reviewed by: @security-team, Jira: SEC-1234, Expires: 2025-09-01
CVE-2023-44487

# Accepted: test dependency only, not in production image
CVE-2024-12345

# IMPORTANT: 
# 1. Every ignore MUST have a Jira ticket and reviewer
# 2. Every ignore MUST have an expiration date
# 3. Review .trivyignore monthly
# 4. Auditors WILL ask why CVEs are suppressed
```

### Trivy DB Caching for CI — Critical Optimization

```yaml
# Without caching: every build downloads ~40MB vulnerability DB
# With 200 builds/day = 8GB/day of redundant downloads
# Plus: if ghcr.io is down, ALL your builds fail

# Solution 1: Pre-cache in agent image
# Build a custom CI image with Trivy DB baked in
FROM aquasec/trivy:0.50.0
RUN trivy image --download-db-only
# Rebuild this image on a schedule (daily CronJob)

# Solution 2: Trivy server mode
# Run a persistent Trivy server that caches the DB
# Agents query the server instead of downloading DB
trivy server --listen 0.0.0.0:8080    # Server (DaemonSet or Deployment)
trivy image --server http://trivy-server:8080 myimage:tag  # Client

# Solution 3: Mirror DB to internal registry
# Copy trivy-db to your ECR/Artifactory
# Set TRIVY_DB_REPOSITORY=123456789.dkr.ecr.us-east-1.amazonaws.com/trivy-db
```

### How Trivy Breaks

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **DB download fails** | `FATAL: DB update error` | ghcr.io rate limit, network policy blocking, air-gapped env | Trivy server mode, mirror DB, pre-cache in image |
| **False positives** | Team ignores all findings | OS package CVE in base image that's not exploitable in context | `.trivyignore` with review process, use distroless/scratch images |
| **Scan timeout** | Pipeline hangs on large images | 2GB+ image with thousands of packages | `--timeout 15m`, use smaller base images, `--skip-dirs` |
| **Exit code confusion** | Pipeline passes with vulns | `--exit-code` not set (default is 0 regardless of findings) | Always set `--exit-code 1` for gated scans |
| **Stale DB** | New CVE not detected | `--skip-db-update` but cached DB is weeks old | Schedule DB refresh, alert if DB age > 24h |
| **SBOM drift** | SBOM doesn't match deployed image | SBOM generated at build time, image patched later | Regenerate SBOM on base image update, attach SBOM to image as OCI artifact |

---

## 3. BlackDuck (Synopsys) — Software Composition Analysis (SCA)

### What SCA Is and Why It's Different from Trivy

```
TRIVY vs BLACKDUCK — DIFFERENT TOOLS, DIFFERENT JOBS:

Trivy (Container/Vulnerability Scanner):
├── Scans container images for known CVEs in packages
├── Scans IaC configs for misconfigurations
├── Scans source for hardcoded secrets
├── Fast, free, CI-integrated
├── Finds: "this package has CVE-2024-XXXX"
└── Does NOT understand: licensing, transitive dependency trees,
    commercial obligation, export control

BlackDuck (Software Composition Analysis):
├── Deep dependency tree analysis (direct + transitive)
├── License compliance (GPL, AGPL, MIT, Apache, proprietary)
│   → "You're using a GPL library in your commercial product.
│      Your lawyers need to know about this NOW."
├── Operational risk (is the library maintained? last commit?)
├── Export control (some crypto libraries restricted by country)
├── Policy enforcement (auto-fail on banned licenses/versions)
├── Custom component identification (internal/forked libraries)
└── Regulatory reports for PCI-DSS, SOC 2, GDPR audits

WHY BOTH:
├── Trivy: "Is this container image safe to run?"
└── BlackDuck: "Are we legally allowed to ship this software?"
```

### How BlackDuck Works

```
┌───────────────────────────────────────────────────────────────┐
│                    BlackDuck Server                           │
│                                                               │
│  ┌───────────────┐  ┌─────────────────┐  ┌───────────────┐    │
│  │ KnowledgeBase │  │ Policy Engine   │  │ Reporting     │    │
│  │ (2.7M+ OSS    │  │ (License rules, │  │ (PDF, CSV,    │    │
│  │  components,  │  │  vuln rules,    │  │  API, SPDX,   │    │
│  │  licenses,    │  │  custom rules)  │  │  CycloneDX)   │    │
│  │  vulns)       │  │                 │  │               │    │
│  └───────────────┘  └─────────────────┘  └───────────────┘    │
└───────────────────────────▲───────────────────────────────────┘
                            │ Upload scan results
                            │
┌───────────────────────────┴──────────────────────────────────┐
│                   Synopsys Detect (CLI)                      │
│                                                              │
│  Runs in CI/CD agent. Detection methods:                     │
│  ├── Package manager inspection (pom.xml, go.mod, etc.)      │
│  │   → Builds full dependency tree including transitives     │
│  ├── Signature scanning (binary/JAR fingerprinting)          │
│  │   → Identifies components even without package manager    │
│  ├── Binary analysis (compiled code matching)                │
│  │   → Identifies vendored/copied code                       │
│  └── Snippet matching (code fragment identification)         │
│      → Finds copy-pasted OSS code (scary accurate)           │
└──────────────────────────────────────────────────────────────┘
```

### Jenkins Integration

```groovy
stage('BlackDuck SCA') {
    steps {
        // Synopsys Detect — the CLI tool
        sh """
            curl -sL https://detect.synopsys.com/detect9.sh -o detect.sh
            bash detect.sh \
                --blackduck.url=${BLACKDUCK_URL} \
                --blackduck.api.token=${BLACKDUCK_TOKEN} \
                --detect.project.name=${APP_NAME} \
                --detect.project.version.name=${BRANCH_NAME} \
                --detect.code.location.name=${APP_NAME}-${BRANCH_NAME} \
                --detect.policy.check.fail.on.severities=BLOCKER,CRITICAL \
                --detect.risk.report.pdf=true \
                --detect.timeout=600 \
                --detect.tools=DETECTOR \
                --logging.level.com.synopsys=INFO
        """
        // --detect.tools=DETECTOR: package manager analysis only (fast)
        // Add SIGNATURE for binary scanning (slow but thorough)
        // --detect.policy.check.fail.on.severities: pipeline gate
    }
}
```

### BlackDuck Policies — NovaMart Examples

```
POLICY: "No Copyleft in Commercial Code"
├── Trigger: Component license IN [GPL-2.0, GPL-3.0, AGPL-3.0]
├── Action: FAIL pipeline (BLOCKER)
├── Why: GPL requires releasing YOUR source code. AGPL applies 
│         even for SaaS. Legal liability is enormous.
└── Exception process: Legal review → Jira ticket → time-boxed waiver

POLICY: "No Critical Vulnerabilities"
├── Trigger: Component has vulnerability with CVSS ≥ 9.0
├── Action: FAIL pipeline (CRITICAL)
└── Why: PCI-DSS requires known critical vulns to be remediated

POLICY: "No Abandoned Libraries"
├── Trigger: Component last updated > 2 years ago
├── Action: WARN (don't fail, but flag for review)
└── Why: Unmaintained libraries won't get security patches

POLICY: "No Duplicate Components"
├── Trigger: Same library, different versions in dependency tree
├── Action: WARN
└── Why: Version conflicts cause runtime surprises, 
          increases attack surface
```

### How BlackDuck Breaks

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **Slow scans** | Pipeline takes 15+ minutes on SCA stage | Signature scanning on large codebase, full binary analysis | Use `--detect.tools=DETECTOR` for fast scans, SIGNATURE only on release |
| **False positive licenses** | Reports GPL for MIT library | KB mismatch, component misidentified, dual-licensed component | Manual review in BD UI, override component, refresh KB match |
| **Transitive dependency surprise** | Your code is clean but fails policy | Direct dep `A` pulls transitive dep `B` which has AGPL license | `mvn dependency:tree`, exclude transitive, find alternative |
| **Token expiry** | Pipeline auth fails | BD API token expired, no rotation | Rotate tokens on schedule, use service account with long-lived token |
| **KB out of date** | New component not recognized | BlackDuck KnowledgeBase hasn't indexed it yet | Submit component to BD, use manual component mapping |
| **Scan overwrites** | Branch scans overwrite main scan | Code location name not unique per branch | Include branch in `--detect.code.location.name` |

---

## 4. Artifactory — Artifact Management

### Why You Need an Artifact Repository

```
WITHOUT ARTIFACTORY:
├── Maven downloads from Maven Central every build (slow, fragile)
├── Docker images in ECR only (no npm, PyPI, Go modules, Helm charts)
├── Build artifacts (JARs, binaries) stored... where? Jenkins workspace? S3? 
├── No audit trail of what was deployed to production
├── No promotion workflow (dev → staging → prod)
├── Dependency on external registries (npmjs.com down = all builds fail)
└── No license/vulnerability metadata on artifacts

WITH ARTIFACTORY:
├── Universal proxy/cache for ALL package types
├── Single source of truth for all artifacts
├── Immutable releases (once published, never overwritten)
├── Promotion workflow (build-info tracks artifact through environments)
├── Full audit trail (who published what, when, from which build)
├── Replication across regions (DR)
└── Integrated security scanning (Xray)
```

### Architecture

```
┌───────────────────────────────────────────────────────────┐
│                    Artifactory                            │
│                                                           │
│  ┌──────────────────────────────────────────────────┐     │
│  │              Repository Types                    │     │
│  │                                                  │     │
│  │  LOCAL repos (you publish to):                   │     │
│  │  ├── libs-release-local    (immutable releases)  │     │
│  │  ├── libs-snapshot-local   (dev builds)          │     │
│  │  ├── docker-local          (Docker images)       │     │
│  │  ├── helm-local            (Helm charts)         │     │
│  │  ├── npm-local             (internal packages)   │     │
│  │  ├── pypi-local            (internal packages)   │     │
│  │  └── go-local              (Go modules)          │     │
│  │                                                  │     │
│  │  REMOTE repos (proxy/cache of external):         │     │
│  │  ├── maven-central-remote  → maven central       │     │
│  │  ├── npmjs-remote          → npmjs.com           │     │
│  │  ├── docker-hub-remote     → docker.io           │     │
│  │  ├── pypi-remote           → pypi.org            │     │
│  │  ├── go-remote             → proxy.golang.org    │     │
│  │  └── helm-remote           → various Helm repos  │     │
│  │                                                  │     │
│  │  VIRTUAL repos (aggregate local + remote):       │     │
│  │  ├── libs-release   = local + remote (reads)     │     │
│  │  ├── npm            = local + remote             │     │
│  │  └── docker         = local + remote             │     │
│  └──────────────────────────────────────────────────┘     │
│                                                           │
│  ┌───────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Build Info    │  │ Replication  │  │ Xray         │    │
│  │ (Tracks       │  │ (Cross-site  │  │ (Vuln scan   │    │
│  │  artifact     │  │  sync, DR)   │  │  + license)  │    │
│  │  provenance)  │  │              │  │              │    │
│  └───────────────┘  └──────────────┘  └──────────────┘    │
└───────────────────────────────────────────────────────────┘
```

### Repository Strategy — NovaMart Pattern

```
VIRTUAL REPO (what developers/CI see):
  libs-release  ←── resolves from ──→  libs-release-local
                                       maven-central-remote (cached)

Flow:
1. Developer/CI requests artifact from libs-release (virtual)
2. Artifactory checks libs-release-local first
3. If not found, checks maven-central-remote (proxy)
4. Downloads from Maven Central, caches locally
5. Next request → served from cache (fast, no external dependency)

Developer config (settings.xml / .npmrc / pip.conf):
  Points to virtual repo URL ONLY
  Never directly to Maven Central, npmjs.com, etc.
```

**Maven settings.xml for NovaMart:**
```xml
<!-- ~/.m2/settings.xml (or mounted in CI agent) -->
<settings>
  <servers>
    <server>
      <id>novamart-artifactory</id>
      <username>${env.ARTIFACTORY_USER}</username>
      <password>${env.ARTIFACTORY_TOKEN}</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>novamart-artifactory</id>
      <name>NovaMart Artifactory</name>
      <url>https://artifactory.novamart.com/libs-release</url>
      <mirrorOf>*</mirrorOf>  <!-- ALL requests go through Artifactory -->
    </mirror>
  </mirrors>
</settings>
```

### Docker Registry in Artifactory

```bash
# Artifactory as Docker registry
# Requires reverse proxy (Nginx) with subdomain routing:
#   docker-local.artifactory.novamart.com → local Docker repo
#   docker-remote.artifactory.novamart.com → Docker Hub proxy
#   docker.artifactory.novamart.com → virtual (both)

# Login
docker login docker.artifactory.novamart.com \
    -u ${ARTIFACTORY_USER} -p ${ARTIFACTORY_TOKEN}

# Push
docker tag order-service:v1.2.3 \
    docker.artifactory.novamart.com/order-service:v1.2.3
docker push docker.artifactory.novamart.com/order-service:v1.2.3

# Pull (checks local first, then Docker Hub via remote)
docker pull docker.artifactory.novamart.com/nginx:1.25

# WHY use Artifactory instead of ECR alone?
# 1. Unified auth (one token for Docker, Maven, npm, Helm, etc.)
# 2. Docker Hub rate limiting (100 pulls/6hr anonymous, 200 authenticated)
#    → Artifactory caches: pull once, serve from cache forever
# 3. Cross-region replication (Artifactory → Artifactory, not ECR → ECR)
# 4. Build info integration (link Docker image to source commit)
# 5. Xray scanning (scan on push, block vulnerable images from download)

# HOWEVER: Many NovaMart teams still use ECR for EKS-native integration
# Pattern: Build → push to Artifactory → replicate to ECR → EKS pulls from ECR
# This gives you Artifactory's features + ECR's native EKS integration
```

### Artifact Promotion Workflow

```
BUILD PIPELINE:
┌───────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Build    │────▶│   Dev    │────▶│ Staging  │────▶│   Prod  │
│  (CI)     │     │  Deploy  │     │  Deploy  │     │  Deploy  │
└───────────┘     └──────────┘     └──────────┘     └──────────┘
     │                │                │                │
     ▼                ▼                ▼                ▼
┌───────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ snapshot  │     │   dev    │     │ staging  │     │ release  │
│ repo      │────▶│  repo    │────▶│   repo   │────▶│  repo   │
│ (mutable) │     │(promote) │     │(promote) │     │(immutable│
└───────────┘     └──────────┘     └──────────┘     └──────────┘

PROMOTION = copy (not move) artifact from one repo to another
            with metadata update (promoted_by, promoted_at, build_id)

IMMUTABLE: Release repo has "Block Overwrite" enabled
           Once v1.2.3 is published, it can NEVER be replaced
           This is critical for audit trails and reproducibility
```

### Cleanup Policies — Disk Management

```
PROBLEM: Without cleanup, Artifactory grows forever.
NovaMart generates ~50GB of artifacts per week.

CLEANUP POLICIES:
├── Snapshots: Delete after 30 days
├── Release candidates (RC): Delete after 90 days
├── Releases: Keep forever (or last 20 versions)
├── Docker tags: 
│   ├── Keep all tags matching semver (v1.2.3)
│   ├── Delete untagged manifests after 7 days
│   └── Delete feature branch tags after 14 days
├── Remote cache: Evict unused cached artifacts after 30 days
└── Build info: Keep forever (tiny, invaluable for audits)

# Artifactory AQL (Artifactory Query Language) for cleanup:
# Find Docker images older than 30 days in snapshot repo
items.find({
    "repo": "docker-snapshot-local",
    "type": "folder",
    "created": {"$before": "30d"}
}).include("repo", "path", "name", "created")
```

### How Artifactory Breaks

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **Disk full** | Upload failures, 500 errors | No cleanup policy, too many snapshots, remote cache bloat | AQL-based cleanup, lifecycle policies, monitor disk with alerts at 80% |
| **Slow downloads** | Builds take 5min to resolve deps | Database query slow, storage backend (NFS/S3) latency, GC running | Tune DB, use direct S3 storage, schedule GC off-hours |
| **Remote repo down** | Can't download new deps | Maven Central / npmjs outage, DNS issue | Cached artifacts still serve. New deps fail. Mirror across multiple remotes. |
| **Auth failure** | 401 on pull/push | Token expired, LDAP/SSO integration broken, permission target misconfigured | Check access logs, verify LDAP connectivity, token rotation |
| **Replication lag** | Region B has stale artifacts | Network issue, replication queue backed up | Monitor replication status, alert on lag > 10min |
| **Docker manifest issues** | `manifest unknown` on pull | Tag was pushed but manifest not fully uploaded, concurrent pushes | Retry push, check for partial uploads, enable Docker V2 schema |
| **Xray blocks legitimate image** | Deployment blocked on pull | Xray policy too aggressive, false positive CVE match | Xray watch exceptions, policy tuning, separate watches per environment |

---

## 5. Advanced Pipeline Patterns

### Pattern 1: Matrix Builds

```groovy
// Build and test across multiple versions/platforms simultaneously
pipeline {
    agent none
    stages {
        stage('Matrix Build') {
            matrix {
                axes {
                    axis {
                        name 'JDK_VERSION'
                        values '17', '21'
                    }
                    axis {
                        name 'OS'
                        values 'linux', 'arm64'
                    }
                    axis {
                        name 'DB'
                        values 'postgres14', 'postgres16'
                    }
                }
                excludes {
                    // Don't test arm64 + postgres14 (not supported)
                    exclude {
                        axis { name 'OS'; values 'arm64' }
                        axis { name 'DB'; values 'postgres14' }
                    }
                }
                stages {
                    stage('Build & Test') {
                        agent { label "${OS}" }
                        steps {
                            sh """
                                export JAVA_HOME=/opt/jdk-${JDK_VERSION}
                                export DB_IMAGE=postgres:${DB.replace('postgres','')}
                                mvn clean verify -B
                            """
                        }
                    }
                }
            }
            // This generates: 2 JDK × 2 OS × 2 DB = 8 combinations
            // Minus 1 exclusion = 7 parallel builds
            // Without matrix: 7 copy-pasted stages
        }
    }
}
```

### Pattern 2: Monorepo Pipeline

```groovy
// 200 services in one repo — only build what changed
pipeline {
    agent { label 'lightweight' }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    // Get changed files compared to main
                    def changes = sh(
                        script: "git diff --name-only origin/main...HEAD",
                        returnStdout: true
                    ).trim().split('\n')

                    // Map changed files to services
                    env.CHANGED_SERVICES = changes
                        .findAll { it.startsWith('services/') }
                        .collect { it.split('/')[1] }
                        .unique()
                        .join(',')

                    // Map to shared libraries
                    env.SHARED_CHANGED = changes
                        .any { it.startsWith('libs/') }

                    echo "Changed services: ${env.CHANGED_SERVICES}"
                    echo "Shared libraries changed: ${env.SHARED_CHANGED}"
                }
            }
        }

        stage('Build Changed Services') {
            when {
                expression { env.CHANGED_SERVICES?.trim() }
            }
            steps {
                script {
                    def services = env.CHANGED_SERVICES.split(',')
                    def parallelStages = [:]

                    services.each { svc ->
                        parallelStages["Build ${svc}"] = {
                            // Each service gets its own agent pod
                            node('java-builder') {
                                dir("services/${svc}") {
                                    sh 'mvn clean verify -B'
                                    // + scan, build image, etc.
                                }
                            }
                        }
                    }

                    // If shared libs changed, rebuild ALL services
                    // that depend on them
                    if (env.SHARED_CHANGED == 'true') {
                        // Read dependency graph
                        def dependents = readJSON(
                            file: 'dependency-graph.json'
                        )
                        // Add all dependent services to build
                    }

                    parallel parallelStages
                }
            }
        }
    }
}
```

### Pattern 3: Input Gates (Manual Approval)

```groovy
stage('Deploy to Production') {
    when { branch 'main' }
    steps {
        // Automated checks first
        script {
            // Verify staging deploy was healthy for 30 minutes
            def stagingHealth = sh(
                script: '''
                    kubectl --context staging get deploy ${APP_NAME} \
                        -o jsonpath='{.status.conditions[?(@.type=="Available")].status}'
                ''',
                returnStdout: true
            ).trim()

            if (stagingHealth != 'True') {
                error "Staging deployment is not healthy. Aborting prod deploy."
            }
        }

        // Manual approval
        timeout(time: 4, unit: 'HOURS') {
            input(
                message: "Deploy ${APP_NAME}:${IMAGE_TAG} to production?",
                ok: 'Deploy',
                submitter: 'platform-leads,sre-team',  // RBAC: only these groups
                parameters: [
                    choice(
                        name: 'STRATEGY',
                        choices: ['canary-10', 'canary-25', 'canary-50', 'full'],
                        description: 'Rollout strategy'
                    ),
                    string(
                        name: 'REASON',
                        defaultValue: '',
                        description: 'Deployment reason (for audit log)'
                    )
                ]
            )
        }

        // If approved, update prod manifests
        sh """
            cd k8s-manifests/apps/${APP_NAME}/overlays/prod
            kustomize edit set image \
                ${REGISTRY}/${APP_NAME}=${REGISTRY}/${APP_NAME}:${IMAGE_TAG}
            git commit -am "prod: ${APP_NAME}:${IMAGE_TAG} - ${REASON}"
            git push
        """
    }
}
```

### Pattern 4: Retry with Exponential Backoff

```groovy
// Flaky integration tests or transient API failures
def retryWithBackoff(int maxRetries, Closure body) {
    int attempt = 0
    while (attempt < maxRetries) {
        try {
            body()
            return  // Success — exit
        } catch (e) {
            attempt++
            if (attempt >= maxRetries) {
                throw e  // Final failure
            }
            def waitTime = Math.pow(2, attempt) * 10  // 20s, 40s, 80s
            echo "Attempt ${attempt} failed. Retrying in ${waitTime}s..."
            sleep(time: waitTime.intValue(), unit: 'SECONDS')
        }
    }
}

// Usage:
stage('Integration Tests') {
    steps {
        script {
            retryWithBackoff(3) {
                sh 'mvn verify -Pintegration-tests'
            }
        }
    }
}
```

### Pattern 5: Stash/Unstash for Cross-Agent Artifacts

```groovy
// Problem: Stage 1 builds on agent A, Stage 2 needs the artifact on agent B
// Solution: stash (save) and unstash (restore)

stage('Build') {
    agent { label 'java-builder' }
    steps {
        sh 'mvn clean package -DskipTests'
        stash(
            name: 'build-artifacts',
            includes: 'target/*.jar',
            excludes: 'target/*-sources.jar'
        )
        // Stash uploads to controller (small files only!)
        // For large files: use Artifactory or S3
    }
}

stage('Docker Build') {
    agent { label 'docker-builder' }
    steps {
        unstash 'build-artifacts'
        // target/*.jar is now available on this different agent
        sh 'docker build -t myapp:latest .'
    }
}

// WARNING: stash goes through the controller
// If your JAR is 500MB, you're pushing 500MB through the controller
// for every build. Use Artifactory for anything > 10MB.
```

### Pattern 6: Pipeline Debugging

```groovy
// Debug techniques when pipelines fail mysteriously

// 1. Replay with edits (no commit needed)
//    Jenkins UI → Build → Replay → Edit Jenkinsfile → Run
//    Perfect for testing pipeline changes without polluting Git

// 2. Enable debug logging for a specific build
pipeline {
    options {
        // Shows all environment variables at start
        // Shows each step as it executes
    }
    stages {
        stage('Debug') {
            steps {
                // Print ALL environment variables
                sh 'env | sort'

                // Print Jenkins-specific vars
                echo "BUILD_URL: ${env.BUILD_URL}"
                echo "WORKSPACE: ${env.WORKSPACE}"
                echo "NODE_NAME: ${env.NODE_NAME}"

                // Check agent capabilities
                sh 'which docker && docker version || echo "No docker"'
                sh 'which mvn && mvn --version || echo "No maven"'
                sh 'cat /etc/os-release'
                sh 'df -h'      // Disk space
                sh 'free -m'    // Memory

                // Test connectivity from agent
                sh 'curl -s -o /dev/null -w "%{http_code}" https://sonar.novamart.com/api/system/status'
                sh 'curl -s -o /dev/null -w "%{http_code}" https://artifactory.novamart.com/api/system/ping'
            }
        }
    }
}

// 3. Jenkinsfile linting (validate BEFORE push)
//    curl -X POST -F "jenkinsfile=<Jenkinsfile" \
//        https://jenkins.novamart.com/pipeline-model-converter/validate

// 4. Declarative Directive Generator
//    Jenkins UI → Pipeline Syntax → Declarative Directive Generator
//    Generates correct syntax for steps you're unsure about

// 5. Blue Ocean Pipeline Editor
//    Visual pipeline editor — useful for non-DevOps contributors

// 6. Common debugging commands in failed K8s agent builds:
//    kubectl get pods -n jenkins-agents --sort-by=.metadata.creationTimestamp
//    kubectl describe pod <agent-pod> -n jenkins-agents
//    kubectl logs <agent-pod> -n jenkins-agents -c jnlp
//    kubectl logs <agent-pod> -n jenkins-agents -c maven
```

### Anti-Patterns — What NOT to Do

```
ANTI-PATTERN 1: "The God Pipeline"
  One 500-line Jenkinsfile with every possible stage
  → Use shared libraries + standardPipeline()

ANTI-PATTERN 2: "Works on My Jenkins"
  Pipeline depends on tools installed on specific agent
  → Container-based agents with everything declared in pod template

ANTI-PATTERN 3: "The Sleeper"
  sh 'sleep 60' to wait for service to be ready
  → Polling with timeout: retry + curl health check

ANTI-PATTERN 4: "The Secret Logger"
  sh "echo Deploying with token ${TOKEN}"
  → withCredentials() + never echo secrets

ANTI-PATTERN 5: "The Unguarded Main"
  No when{} conditions — every branch runs deploy-to-prod stages
  → when { branch 'main' } on every deployment stage

ANTI-PATTERN 6: "The Artifact Hoarder"
  archiveArtifacts '**/*' on every build, no discard policy
  → Archive specific files, buildDiscarder, external storage

ANTI-PATTERN 7: "The Retry Hammer"
  retry(5) around flaky test → hides real failures
  → Fix the flaky test. Retries mask bugs.
  → If you MUST retry, log each failure with context.

ANTI-PATTERN 8: "The Freestyle Dinosaur"
  Using Freestyle jobs with shell scripts instead of Pipeline
  → Pipeline as Code in Jenkinsfile, always
```

---

## Quick Reference Card

```
SONARQUBE:
  Scanner (CI agent) → Server (CE processes async) → Webhook → Jenkins
  Quality Gate on NEW CODE only (clean-as-you-code)
  waitForQualityGate() needs webhook configured in SonarQube admin
  Developer Edition minimum for branch analysis + PR decoration
  Key flags: -Dsonar.projectKey, -Dsonar.branch.name, 
             -Dsonar.coverage.jacoco.xmlReportPaths

TRIVY:
  Universal scanner: image, fs, repo, k8s, aws, sbom, config
  --exit-code 1 (ALWAYS — default doesn't fail pipeline)
  --ignore-unfixed (skip CVEs without patches)
  --severity HIGH,CRITICAL (don't gate on MEDIUM in CI)
  .trivyignore with Jira ticket + expiry date
  DB caching: server mode, mirror DB, pre-cache in image
  SBOM: --format cyclonedx for compliance/incident response

BLACKDUCK (SCA):
  Different from Trivy: licensing + transitive deps + operational risk
  Synopsys Detect CLI: --detect.tools=DETECTOR (fast)
  Policies: no copyleft, no critical CVEs, no abandoned libs
  --detect.policy.check.fail.on.severities=BLOCKER,CRITICAL
  Include branch in code.location.name to avoid overwrites

ARTIFACTORY:
  LOCAL (your artifacts) + REMOTE (proxy/cache) + VIRTUAL (aggregate)
  mirrorOf=* in settings.xml → all traffic through Artifactory
  Promotion: snapshot → dev → staging → release (immutable)
  Cleanup: AQL queries, lifecycle policies, monitor disk at 80%
  Docker: subdomain routing, cache Docker Hub (rate limits!)
  Stash: controller-bound, <10MB only. Use Artifactory for large files.

PIPELINE PATTERNS:
  Matrix: axes + excludes → parallel multi-version builds
  Monorepo: git diff → detect changes → build only affected services
  Input gates: submitter RBAC + timeout + audit reason
  Retry: exponential backoff function, not blind retry(5)
  Debug: Replay, env|sort, Jenkinsfile lint API, Blue Ocean
  
ANTI-PATTERNS:
  God pipeline, unguarded main, secret logging,
  retry hammer, freestyle dinosaurs, artifact hoarding
```
---

## Retention Questions

**Q1.** NovaMart's payment-service team reports their pipeline is failing at the SonarQube Quality Gate stage with: `WARN: Quality Gate check - FAILED. Status: ERROR. Conditions: Coverage on New Code < 80%`. The team lead says "we have 92% coverage in our JaCoCo report." Walk me through every possible reason the SonarQube coverage doesn't match the JaCoCo report, and how you'd diagnose each one.

**Q2.** Your security team runs an audit and discovers that 47 of NovaMart's 200 services have Docker images with CRITICAL CVEs in production. The CTO asks: "How did these get deployed?" and "How do we prevent this from ever happening again?" Design a comprehensive prevention strategy covering CI pipeline gates, registry-level enforcement, runtime detection, and the governance process for accepted risks. Be specific about which tools do what.

**Q3.** A developer opens a PR that adds a library called `json-schema-validator`. BlackDuck flags it with: `Policy Violation: LGPL-2.1 license detected`. The developer argues "LGPL is fine, it's not GPL." Who is right? What's the actual risk? What's your recommended process for handling this — from technical analysis through legal review to policy update?

**Q4.** NovaMart's Artifactory instance is at 94% disk usage. Builds are starting to fail with upload errors. You need to reclaim space immediately AND implement a long-term strategy. Walk me through: (a) emergency triage to free space in the next 30 minutes, (b) identifying what's consuming the most space, (c) designing cleanup policies for each repository type, and (d) monitoring/alerting to prevent recurrence.

---

### Q1: The SonarQube vs. JaCoCo Coverage Gap

The most important thing to understand here is that **JaCoCo generates the data, but SonarQube interprets it.** A mismatch usually means the "interpretation" is based on different assumptions than the "generation."

**Possible Reasons and Diagnosis:**

1.  **"New Code" Definition (The Most Likely Cause):**
    *   **Reason:** The team lead is looking at *Overall Coverage* (all code ever written), but the Quality Gate is failing on *Coverage on New Code*. If the new feature has 20% coverage, but the rest of the app has 99%, the overall average is 92%, but the "New Code" gate fails.
    *   **Diagnosis:** Check the "New Code" period in the SonarQube project settings (e.g., "previous version" or "last 30 days"). Compare the "New Code" tab vs. the "Overall" tab in the Sonar UI.

2.  **Incorrect Report Path (`sonar.coverage.jacoco.xmlReportPaths`):**
    *   **Reason:** SonarQube is looking at a stale JaCoCo report from a previous build or a different module. It is reporting coverage based on old data.
    *   **Diagnosis:** Check the Jenkins logs for the Sonar scanner step. Ensure the path to the `.xml` report is correct and that the report was actually generated in the current workspace *before* the scan started.

3.  **Binary/Source Mismatch (The "Compilation Gap"):**
    *   **Reason:** JaCoCo instrumented the code using one version of the compiler/JDK, but SonarQube is analyzing the source code. If the source code changed but the binaries weren't rebuilt, the coverage mapping shifts.
    *   **Diagnosis:** Verify that the `mvn clean verify` (or equivalent) is running in the same pipeline step immediately preceding the Sonar scan.

4.  **Excluded Files:**
    *   **Reason:** The JaCoCo report includes everything, but SonarQube has `sonar.exclusions` configured (e.g., ignoring DTOs or generated code). If the "uncovered" code is in the excluded files, the numbers will diverge.
    *   **Diagnosis:** Compare the list of files in the JaCoCo HTML report with the "Code" tab in SonarQube to see which files are missing.

---

### Q2: CVE Prevention Strategy (Supply Chain Security)

**How did they get deployed?**
They likely entered through **Dependency Drift** or **Base Image Decay**. An image that was "clean" six months ago is now "critical" because a new CVE was discovered in a library it contains. If there is no continuous scanning, you only find out during a manual audit.

**The Comprehensive Prevention Strategy:**

| Stage | Tool | Action | Enforcement |
| :--- | :--- | :--- | :--- |
| **CI Pipeline** | **Trivy / Snyk** | Scan image during the build stage. | **Hard Gate:** Fail the build if `CRITICAL` CVEs are found with a fix available. |
| **Registry** | **Harbor / Artifactory** | Continuous scan of all stored images. | **Pull-Blocking:** Prevent `kubelet` from pulling images that have been flagged as "Critical" since their last scan. |
| **Runtime** | **Trivy Operator / Aqua** | Scan pods currently running in EKS. | **Alerting:** Trigger PagerDuty if a new CVE is discovered in a running pod (since the image was clean at build time). |
| **Governance** | **CVE Exception List** | A signed-off YAML file of "Accepted Risks." | **Override:** If a CVE is a false positive or has no fix, SecOps adds the CVE-ID to the list to bypass the CI gate. |

---

### Q3: The LGPL-2.1 License Risk

**Who is right?**
The security team is right to flag it; the developer is oversimplifying.

**The Actual Risk:**
The **GPL (General Public License)** is "Strong Copyleft"—if you use it, your whole app must become open source. The **LGPL (Lesser GPL)** is "Weak Copyleft." 

The risk with LGPL-2.1 depends on **how the library is linked**:
1.  **Dynamic Linking (Safe):** If you link to the library as a separate `.so` or `.jar` file, you only have to open-source the changes you made *to the library itself*. Your main app remains proprietary.
2.  **Static Linking (Dangerous):** If the library is bundled/compiled directly into your binary, the LGPL can be interpreted as requiring the *entire* resulting binary to be open-sourced.
3.  **Modification (Dangerous):** If you modify the library's code, those modifications must be shared.

**Recommended Process:**
1.  **Technical Audit:** The SRE/Security team checks the build artifacts. Is the library statically or dynamically linked? Was the source code of the library modified?
2.  **Legal Review:** Present the findings to the Legal team. "We are using LGPL-2.1 via dynamic linking in a Java JAR; does this trigger our open-source disclosure obligations?"
3.  **Policy Decision:**
    *   **Approved:** Add the library to the "Allow-list."
    *   **Rejected:** Developer must find an MIT/Apache-2.0 alternative or rewrite the functionality.
4.  **Policy Update:** Update the BlackDuck policy to allow LGPL *only if* certain linking conditions are met.

---

### Q4: Artifactory Disk Crisis

** (a) Emergency Triage (Next 30 Minutes):**
I need to find the "low hanging fruit"—data that is useless but large.
1.  **Clean Snapshot Repositories:** Delete the oldest snapshot builds. Snapshots are for dev/test and are usually the biggest space hogs.
2.  **Clear Temporary Caches:** Empty the `remote-cache` directories for external proxies (e.g., cached Maven Central packages that are rarely used).
3.  **Empty the Trash:** If Artifactory has a "Trash" or "Recycle Bin" feature enabled, empty it immediately.

** (b) Identifying the Culprits:**
I would use the **Artifactory Storage Analysis** tool (built-in) to identify:
*   The top 10 largest repositories.
*   The largest individual artifacts.
*   The "unreferenced" blobs (blobs that exist on disk but are no longer linked to any metadata).

** (c) Designing Cleanup Policies:**
I would implement **Retention Policies** based on repository type:
*   **Local-Release Repos:** Keep all versions (immutable).
*   **Local-Snapshot Repos:** Keep only the last **10 builds per branch** or delete anything older than **30 days**.
*   **Remote-Cache Repos:** Delete any cached artifact that hasn't been requested in **90 days**.
*   **Build-Info Repos:** Delete build metadata for builds older than 6 months.

** (d) Monitoring and Alerting:**
1.  **Prometheus Exporter:** Install the Artifactory Prometheus exporter to track `artifactory_storage_used_bytes`.
2.  **Grafana Dashboard:** Create a "Storage Health" dashboard showing usage per repository.
3.  **Alerting:** Set a tiered alert in PagerDuty:
    *   **Warning (80%):** Ticket created for the SRE team to review retention policies.
    *   **Critical (90%):** Page the on-call SRE to perform manual triage.

## Q1: SonarQube vs JaCoCo Coverage Gap — 4.3/5

**What you nailed:**
- "New Code" vs "Overall Code" as the most likely cause — exactly right, and you correctly identified this as #1. This is the cause 70% of the time.
- Report path mismatch is a real and common issue.
- Binary/source mismatch — good catch, most people miss this.
- Exclusion divergence between JaCoCo and SonarQube — correct.

**What you missed:**

1. **Multi-module Maven projects.** This is the second most common cause at scale. A multi-module project (parent POM with 8 child modules) generates 8 separate `jacoco.xml` files. If `sonar.coverage.jacoco.xmlReportPaths` only points to one, you get partial coverage. The fix is a glob: `**/target/site/jacoco/jacoco.xml`. At NovaMart with Spring Boot microservices, multi-module POMs are extremely common.

2. **JaCoCo report format mismatch.** SonarQube requires the XML report, not the HTML or CSV report. If the Maven config generates only the HTML report (`jacoco:report` produces both, but some configs suppress XML), SonarQube sees zero coverage. Also: JaCoCo has two XML formats — the exec binary format and the XML report. SonarQube needs the XML report, not the `.exec` file (older SonarQube versions used `.exec` directly via a plugin, but modern versions require XML).

3. **Test execution order.** If `mvn sonar:sonar` is a separate Maven invocation from `mvn verify`, and the `clean` goal runs again, the JaCoCo report gets deleted before SonarQube reads it. This is why the lesson showed `mvn clean verify` as one step, then `mvn sonar:sonar` as a separate step WITHOUT `clean`. Alternatively, use `sonar.coverage.jacoco.xmlReportPaths` with an absolute path that survives across invocations.

4. **You didn't mention the diagnostic command.** When debugging, the fastest path is: go to the SonarQube project → Activity → click the specific analysis → look at the scanner context log. It shows exactly which coverage report files were found and parsed. If it says "No coverage report found," you have your answer in 10 seconds instead of guessing.

**Production consequence of the gap:** You'd correctly identify the New Code issue, but when the team says "we fixed that and it's still wrong," you'd be stuck debugging multi-module report paths without knowing to check the scanner context log first.

---

## Q2: CVE Prevention Strategy — 4/5

**What you nailed:**
- "Dependency Drift / Base Image Decay" as the root cause — exactly right. Images rot over time as new CVEs are discovered.
- Four-stage strategy (CI → Registry → Runtime → Governance) is the correct architecture.
- Pull-blocking at the registry level is advanced and correct.
- The table format is clean and actionable.

**What you missed:**

1. **Base image rebuild pipeline.** You described detection at every stage but didn't address the root cause: base images aren't being rebuilt. The fix is a scheduled pipeline (weekly CronJob or Jenkins timer trigger) that rebuilds ALL service images from their Dockerfiles, even with no code changes. This picks up base image patches. Without this, services that haven't had code changes in 6 months never get rescanned/rebuilt.

```
Weekly base image rebuild pipeline:
  For each service:
    Pull latest base image → rebuild → scan → 
    if clean: push new tag, update manifests
    if dirty: alert service team
```

2. **Admission controller enforcement.** You mentioned registry-level pull-blocking, but the stronger pattern in Kubernetes is an admission controller (Kyverno or OPA Gatekeeper) that **rejects pod creation** if the image isn't signed or hasn't been scanned within the last N days. This is the last line of defense — even if someone pushes directly to ECR bypassing Artifactory.

```yaml
# Kyverno policy: only allow signed, recently-scanned images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-verification
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-signature
    match:
      resources:
        kinds: [Pod]
    verifyImages:
    - imageReferences: ["*.dkr.ecr.*.amazonaws.com/*"]
      attestors:
      - entries:
        - keyless:
            issuer: "https://token.actions.githubusercontent.com"
```

3. **ECR native scanning.** You mentioned Trivy Operator for runtime but didn't mention that ECR has built-in scanning (Basic = Clair-based free, Enhanced = Inspector-based paid). Enabling "scan on push" in ECR gives you a free second opinion alongside Trivy. For NovaMart's 200 services, ECR Enhanced scanning with EventBridge → SNS → Slack gives continuous vulnerability notifications with zero pipeline changes.

4. **The "47 services" triage.** The CTO asked "how did these get deployed?" — you answered generically. A senior engineer also says: "I'll have a list of all 47 services with their specific CVEs, sorted by CVSS score, with business impact, within 2 hours." That's the Trivy Operator scan or ECR bulk scan output, cross-referenced with service criticality. Payment service with a CRITICAL CVE = SEV1. Internal reporting tool = queue it.

**Production consequence:** Your strategy prevents future CVEs but doesn't address the immediate crisis of 47 vulnerable services in production, and doesn't automate the base image refresh that prevents drift.

---

## Q3: LGPL License — 4.5/5

**This was your strongest answer.**

**What you nailed:**
- Correct that security team is right to flag, developer is oversimplifying.
- Static vs dynamic linking distinction is accurate and is the actual legal crux.
- Modification disclosure requirement — correct.
- The process (technical audit → legal review → policy decision → policy update) is mature and exactly what enterprises do.
- Java-specific context ("dynamically linked JAR") shows you understand the language-specific implications.

**What you missed:**

1. **Java-specific nuance is actually in your favor, and you should have stated it more forcefully.** In Java, virtually all library usage is dynamic linking (classpath loading, separate JARs). Static linking in the C/C++ sense doesn't apply. The Java community generally considers LGPL-2.1 safe for Java applications when the library is used as an unmodified dependency. This is a well-established interpretation (see: Hibernate was LGPL for years and used by every commercial Java app). You had the right instinct but were too cautious — a senior engineer should say: "For Java, LGPL via unmodified Maven dependency is almost certainly fine, but we still run it through legal for the paper trail."

2. **You didn't mention the allow-list mechanism in BlackDuck.** After legal approves, the technical implementation is: create a BlackDuck component-level override or a policy exception scoped to that specific library+version+project. Don't create a blanket "LGPL is allowed" policy — that opens the door for C/C++ services where LGPL IS dangerous with static linking.

3. **Alternative libraries.** A senior engineer also says: "Before we go through the legal process, are there Apache-2.0 or MIT alternatives that do the same thing?" For `json-schema-validator` specifically, there are multiple MIT-licensed alternatives. If a 5-minute search avoids a 2-week legal review, that's the pragmatic choice. Legal review is the fallback, not the first move.

---

## Q4: Artifactory Disk Crisis — 3.8/5

**What you nailed:**
- Emergency priority order is correct (snapshots → cache → trash).
- Identifying culprits via storage analysis — correct approach.
- Cleanup policies per repo type — well-structured.
- Monitoring with Prometheus exporter + tiered alerting — good.

**What you missed:**

1. **Your emergency triage is too slow for "30 minutes."** "Delete oldest snapshot builds" sounds simple, but if you have 200 services × 50 snapshots each, you need to know the commands:

```bash
# Artifactory REST API — delete all snapshots older than 7 days
# Step 1: Find them (AQL)
curl -X POST -H "Content-Type: text/plain" \
    -u admin:token \
    "https://artifactory.novamart.com/api/search/aql" \
    -d 'items.find({
        "repo":"libs-snapshot-local",
        "type":"folder",
        "created":{"$before":"7d"}
    }).include("repo","path","name","size")'

# Step 2: Calculate space to be freed (before deleting!)
# Step 3: Delete
curl -X DELETE -u admin:token \
    "https://artifactory.novamart.com/libs-snapshot-local/com/novamart/old-service/1.0-SNAPSHOT"

# Step 4: Run garbage collection to actually free disk
# Admin → Advanced → Maintenance → Garbage Collection
# OR: POST /api/system/storage/gc
```

Without the actual API calls and the GC step, you're describing a plan but can't execute it under pressure.

2. **Garbage collection is critical and you missed it.** Deleting artifacts in Artifactory removes the metadata, but the binary blobs remain on disk until garbage collection runs. If you delete 100GB of snapshots but don't trigger GC, disk usage doesn't change. This is the #1 reason people panic-delete and then say "it didn't free any space."

3. **Docker image layers are likely the biggest consumer and you didn't call them out specifically.** Docker images in Artifactory store layers individually. A 500MB image shared across 50 tags still stores unique layers. But untagged manifests (from CI builds that push intermediate tags) accumulate layers that are never cleaned. Docker-specific cleanup:

```bash
# Find untagged Docker manifests (orphaned layers)
# Artifactory → Admin → Docker → Cleanup Policies
# Or: use the built-in Docker Cleanup policy to remove 
# manifests with no tags older than N days
```

4. **Build-info retention of 6 months is too aggressive.** Build info is tiny (KB per build) and is your audit trail. Auditors will ask "what was deployed to production on March 15?" — if you deleted that build info, you can't answer. Keep build info for 2+ years. Delete the artifacts, keep the metadata.

5. **You mentioned "unreferenced blobs" which is good, but didn't explain how to find them.** The correct tool is: `POST /api/system/storage/gc` with dry-run first to see what would be collected, then run it for real.


---

# Phase 3: CI/CD Pipelines
## Lesson 3: ArgoCD Advanced + Argo Rollouts + Progressive Delivery

---

## Context Reset

You saw ArgoCD basics in Phase 2 Lesson 4: Application resource, syncPolicy, auto-sync, prune, selfHeal, App of Apps, sync waves. This lesson goes **deep** into production ArgoCD operations and introduces Argo Rollouts for progressive delivery — the mechanism that prevents bad deploys from killing NovaMart.

---

## 1. ArgoCD Architecture — Production Internals

```
┌────────────────────────── ArgoCD Namespace ────────────────────────────┐
│                                                                        │
│  ┌───────────────────────┐    ┌───────────────────────────────────┐    │
│  │  argocd-server        │    │  argocd-repo-server               │    │
│  │  (API + UI)           │    │  (Git clone, manifest generation) │    │
│  │                       │    │                                   │    │
│  │  - Serves Web UI      │    │  - Clones Git repos               │    │
│  │  - REST/gRPC API      │    │  - Runs Helm template             │    │
│  │  - Auth (OIDC/SSO)    │    │  - Runs Kustomize build           │    │
│  │  - RBAC enforcement   │    │  - Runs jsonnet, plain YAML       │    │
│  │  - Webhook receiver   │    │  - Caches repos (60s default)     │    │
│  │  2+ replicas (HA)     │    │  - CPU/memory intensive           │    │
│  └──────────┬────────────┘    │  2+ replicas (HA)                 │    │
│             │                  └───────────┬──────────────────────┘    │
│             │                              │                           │
│  ┌──────────▼──────────────────────────────▼──────────────────────┐    │
│  │              argocd-application-controller                     │    │
│  │              (THE BRAIN — reconciliation loop)                 │    │
│  │                                                                │    │
│  │  - Watches Application CRDs                                    │    │
│  │  - Compares desired state (Git) vs live state (cluster)        │    │
│  │  - Detects drift (OutOfSync)                                   │    │
│  │  - Applies manifests to target cluster (sync)                  │    │
│  │  - Runs health checks on resources                             │    │
│  │  - Manages sync waves and hooks                                │    │
│  │  - ONE controller OR sharded (by cluster/namespace)            │    │
│  └───────────────────────────────┬────────────────────────────────┘    │
│                                  │                                     │
│  ┌───────────────────────────────▼────────────────────────────────┐    │
│  │              argocd-applicationset-controller                  │    │
│  │              (Dynamic Application generation)                  │    │
│  │                                                                │    │
│  │  - Generates Application CRDs from templates                   │    │
│  │  - Sources: Git directory, Git file, list, cluster,            │    │
│  │    SCM provider, pull request, merge request                   │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                        │
│  ┌───────────────────────┐    ┌───────────────────────────────────┐    │
│  │  Redis                │    │  argocd-notifications-controller  │    │
│  │  (Cache layer)        │    │  (Slack, webhook, email triggers) │    │
│  │  - Manifest cache     │    │  - Sync succeeded/failed          │    │
│  │  - App state cache    │    │  - Health degraded                │    │
│  │  - CRITICAL for perf  │    │  - Custom triggers + templates    │    │
│  └───────────────────────┘    └───────────────────────────────────┘    │
│                                                                        │
│  Storage: Kubernetes etcd (Application CRDs + Secrets for repo creds)  │
│  NO external database. ArgoCD is fully Kubernetes-native.              │
└────────────────────────────────────────────────────────────────────────┘
```

### The Reconciliation Loop — How Sync Works

```
Every 3 minutes (default) OR on Git webhook:

  ┌──────────────┐
  │ repo-server   │──── git clone/fetch ────► Git Repo
  │ generates     │                           (manifests)
  │ desired state │
  └──────┬───────┘
         │ rendered manifests
         ▼
  ┌──────────────────┐     ┌──────────────────┐
  │ app-controller    │────►│ Kubernetes API   │
  │ compares:         │     │ (live state)     │
  │                   │     └──────────────────┘
  │ desired (Git)     │
  │   vs              │
  │ live (cluster)    │
  │                   │
  │ Result:           │
  │ ├── Synced ✅     │ (identical)
  │ ├── OutOfSync ⚠️  │ (drift detected)
  │ └── Unknown ❓    │ (can't determine)
  └──────┬────────────┘
         │
         │ If auto-sync enabled AND OutOfSync:
         ▼
  ┌──────────────────┐
  │ Apply manifests   │
  │ to cluster        │
  │ (kubectl apply    │
  │  equivalent)      │
  └──────────────────┘
```

**Tuning the reconciliation:**
```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # How often to re-render manifests from Git
  timeout.reconciliation: 180s    # Default 3min. Increase for large installs.

  # How often to check for Git changes
  # (webhook-driven is preferred — this is the fallback)
  repository.credentials: |
    ...

  # Resource exclusions (don't track these in ArgoCD)
  resource.exclusions: |
    - apiGroups: [""]
      kinds: ["Event"]
    - apiGroups: ["metrics.k8s.io"]
      kinds: ["*"]

  # Resource tracking method
  # label (default): adds app.kubernetes.io/instance label
  # annotation: uses annotation instead (avoids label conflicts)
  # annotation+label: both
  application.resourceTrackingMethod: annotation
```

### Multi-Cluster Management

```
┌─────────────────────────────────────────────────────────────────┐
│              ArgoCD Management Cluster                            │
│              (us-east-1, dedicated EKS)                          │
│                                                                   │
│  ArgoCD watches and deploys to:                                   │
│  ├── management cluster (itself — platform tools)                 │
│  ├── us-east-1 production cluster                                 │
│  ├── us-west-2 production cluster                                 │
│  ├── eu-west-1 production cluster                                 │
│  ├── staging cluster                                              │
│  └── dev cluster                                                  │
│                                                                   │
│  Connection: ArgoCD stores kubeconfig/bearer token as K8s Secret  │
│  Security: IRSA + cross-account IAM role for each target cluster  │
└─────────────────────────────────────────────────────────────────┘

# Register a cluster:
argocd cluster add arn:aws:eks:us-west-2:123456789:cluster/prod-west \
    --name prod-us-west-2 \
    --system-namespace argocd

# This creates a ServiceAccount in the target cluster
# with cluster-admin (restrict this! Use AppProject RBAC)
```

### AppProject — RBAC for Applications

```yaml
# "Which team can deploy what, where?"
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payment-team
  namespace: argocd
spec:
  description: "Payment team services"

  # Source repos this project can use
  sourceRepos:
    - 'https://bitbucket.org/novamart/k8s-manifests.git'
    - 'https://bitbucket.org/novamart/payment-*'    # Wildcards

  # Destination clusters and namespaces
  destinations:
    - server: https://kubernetes.default.svc    # In-cluster
      namespace: 'payment-*'                     # Only payment namespaces
    - server: https://prod-west.novamart.internal
      namespace: 'payment-*'

  # Allowed/denied Kubernetes resource types
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota    # Teams can't modify their own quotas
    - group: rbac.authorization.k8s.io
      kind: '*'              # Teams can't modify RBAC

  # Sync windows — when deployments are allowed
  syncWindows:
    - kind: allow
      schedule: '0 6-22 * * 1-5'    # Mon-Fri 6am-10pm only
      duration: 16h
      applications: ['*']
    - kind: deny
      schedule: '0 0 25 12 *'       # No deploys on Christmas
      duration: 24h
      applications: ['*']
    - kind: allow
      schedule: '* * * * *'          # Always allow
      applications: ['payment-hotfix-*']  # Except hotfixes
      manualSync: true               # But manual only

  # Role definitions (who can do what in this project)
  roles:
    - name: deployer
      description: "Can sync applications"
      policies:
        - p, proj:payment-team:deployer, applications, sync, payment-team/*, allow
        - p, proj:payment-team:deployer, applications, get, payment-team/*, allow
      groups:
        - payment-team-leads    # OIDC group mapping

    - name: viewer
      description: "Read-only access"
      policies:
        - p, proj:payment-team:viewer, applications, get, payment-team/*, allow
      groups:
        - payment-team-devs
```

### ApplicationSet — Dynamic Application Generation

```yaml
# Problem: 200 services × 3 environments = 600 Application manifests
# Solution: ApplicationSet generates them from templates

# Pattern 1: Git Directory Generator
# Each directory in the repo becomes an Application
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: novamart-services
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://bitbucket.org/novamart/k8s-manifests.git
        revision: main
        directories:
          - path: 'apps/*/overlays/dev'     # Each service has a dir
          - path: 'apps/payment-*'
            exclude: true                    # Except payment (separate project)

  template:
    metadata:
      name: '{{path.basename}}-dev'          # e.g., order-service-dev
      labels:
        team: '{{path[1]}}'                  # Extracted from path
    spec:
      project: default
      source:
        repoURL: https://bitbucket.org/novamart/k8s-manifests.git
        targetRevision: main
        path: '{{path}}'                     # The matched directory
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'       # Namespace = service name
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true

---
# Pattern 2: Matrix Generator (environments × clusters)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: novamart-multi-region
spec:
  generators:
    - matrix:
        generators:
          # First axis: services
          - git:
              repoURL: https://bitbucket.org/novamart/k8s-manifests.git
              revision: main
              directories:
                - path: 'apps/*'
          # Second axis: clusters
          - list:
              elements:
                - cluster: prod-east
                  url: https://eks-east.novamart.internal
                  region: us-east-1
                - cluster: prod-west
                  url: https://eks-west.novamart.internal
                  region: us-west-2
                - cluster: prod-eu
                  url: https://eks-eu.novamart.internal
                  region: eu-west-1

  template:
    metadata:
      name: '{{path.basename}}-{{cluster}}'
    spec:
      project: production
      source:
        repoURL: https://bitbucket.org/novamart/k8s-manifests.git
        path: '{{path}}/overlays/prod-{{region}}'
      destination:
        server: '{{url}}'
        namespace: '{{path.basename}}'

---
# Pattern 3: Pull Request Generator
# Auto-create preview environments for PRs
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: pr-previews
spec:
  generators:
    - pullRequest:
        bitbucketServer:
          api: https://bitbucket.novamart.com
          project: NOVA
          repo: order-service
        requeueAfterSeconds: 60
        filters:
          - branchMatch: 'feature/*'

  template:
    metadata:
      name: 'preview-order-{{number}}'
    spec:
      project: previews
      source:
        repoURL: https://bitbucket.org/novamart/order-service.git
        targetRevision: '{{branch}}'
        path: deploy/preview
      destination:
        server: https://kubernetes.default.svc
        namespace: 'preview-{{number}}'
      syncPolicy:
        automated:
          prune: true
        syncOptions:
          - CreateNamespace=true
  
  # CRITICAL: Clean up when PR is merged/closed
  # ApplicationSet deletes the Application when generator
  # no longer produces it (PR closed → no output → App deleted)
```

### Sync Waves & Hooks — Ordered Deployment

```yaml
# Problem: Deploy namespace, then configmap, then deployment, then service
# If they all apply simultaneously, deployment might start before configmap exists

# Sync Waves: lower number syncs first, waits for health before next wave
# Wave -1 → Wave 0 → Wave 1 → Wave 2

# Namespace (wave -2)
apiVersion: v1
kind: Namespace
metadata:
  name: order-service
  annotations:
    argocd.argoproj.io/sync-wave: "-2"

---
# Secrets from External Secrets Operator (wave -1)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"

---
# ConfigMap (wave 0 — default)
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"

---
# Deployment (wave 1)
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"

---
# Service + Ingress (wave 2)
apiVersion: v1
kind: Service
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"

---
# Post-deploy smoke test (hook, not wave)
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync         # Run after all waves complete
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
    # BeforeHookCreation: delete previous Job before creating new one
    # HookSucceeded: delete after success
    # HookFailed: delete after failure (usually don't want this)
spec:
  template:
    spec:
      containers:
        - name: smoke-test
          image: curlimages/curl:8.5.0
          command: ['sh', '-c']
          args:
            - |
              set -e
              # Wait for service to be ready
              for i in $(seq 1 30); do
                if curl -sf http://order-service:8080/health; then
                  echo "Service healthy!"
                  exit 0
                fi
                sleep 2
              done
              echo "Service not healthy after 60s"
              exit 1
      restartPolicy: Never
  backoffLimit: 2
```

### Notifications Controller

```yaml
# argocd-notifications-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Slack integration
  service.slack: |
    token: $slack-token          # From argocd-notifications-secret

  # Trigger definitions
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [slack-success]

  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Failed', 'Error']
      send: [slack-failure]

  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [slack-degraded]

  # Message templates
  template.slack-success: |
    message: |
      ✅ *{{.app.metadata.name}}* synced successfully
      Revision: {{.app.status.sync.revision | truncate 7 ""}}
      Environment: {{.app.spec.destination.namespace}}
      <{{.context.argocdUrl}}/applications/{{.app.metadata.name}}|View in ArgoCD>

  template.slack-failure: |
    message: |
      ❌ *{{.app.metadata.name}}* sync FAILED
      Error: {{.app.status.operationState.message}}
      Revision: {{.app.status.sync.revision | truncate 7 ""}}
      <{{.context.argocdUrl}}/applications/{{.app.metadata.name}}|View in ArgoCD>
      @oncall-sre

  template.slack-degraded: |
    message: |
      ⚠️ *{{.app.metadata.name}}* health degraded
      Status: {{.app.status.health.status}}
      <{{.context.argocdUrl}}/applications/{{.app.metadata.name}}|View in ArgoCD>

---
# Apply to specific applications via annotations:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: deployments
    notifications.argoproj.io/subscribe.on-sync-failed.slack: deployments-alerts
    notifications.argoproj.io/subscribe.on-health-degraded.slack: sre-alerts
```

### How ArgoCD Breaks

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **repo-server OOM** | Apps stuck in "Unknown," sync fails | Large repos, too many Helm templates, heavy Kustomize builds | Increase memory, use `server-side-apply`, shard repo-server |
| **Controller overwhelmed** | Sync takes 10+ minutes, UI laggy | 500+ Applications on single controller, constant reconciliation | Shard controller by cluster, increase `timeout.reconciliation`, use webhooks not polling |
| **Redis OOM/eviction** | Random sync failures, stale UI data | Cache too small for number of apps | Increase Redis memory, monitor eviction rate |
| **Git rate limiting** | `429 Too Many Requests` from Bitbucket | 200 apps polling every 3 min = 4000 git calls/hour | Use webhooks (Git → ArgoCD), increase poll interval, credential caching |
| **Drift false positives** | Apps constantly show "OutOfSync" | Mutating webhooks modify resources after apply (Istio sidecar injection, defaulting) | `resource.customizations.ignoreDifferences` in argocd-cm |
| **Sync window blocks deploy** | Emergency fix can't deploy | Overly restrictive sync windows | Emergency override via `argocd app sync --force`, or allow-window for hotfix apps |
| **Secret in Git** | Credentials visible in ArgoCD UI | Sealed/encrypted secrets decoded, shown in diff | Use External Secrets Operator (ArgoCD sees ExternalSecret, not Secret), or Vault agent |
| **Namespace stuck deleting** | App deleted but namespace won't terminate | Finalizers on resources in namespace, webhook blocking | Remove ArgoCD finalizer, check for stuck resources, investigate webhook |
| **ApplicationSet mass-delete** | Generator produces empty list → all apps deleted | Git repo access issue, broken directory structure | `preserveResourcesOnDeletion: true` policy, test generators in staging |

**The ApplicationSet mass-delete deserves special attention:**
```yaml
# DANGEROUS: If Git generator can't read the repo (auth failure, 
# network issue), it produces ZERO directories → ZERO applications →
# ArgoCD deletes ALL existing applications.

# PROTECTION:
spec:
  # Won't delete apps even if generator stops producing them
  syncPolicy:
    preserveResourcesOnDeletion: true
  
  # Also: ApplicationSet progressive syncs (rollout strategy)
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: env
              operator: In
              values: [staging]
        - matchExpressions:
            - key: env
              operator: In
              values: [prod]
          maxUpdate: 25%    # Only update 25% of prod apps at a time
```

---

## 2. Argo Rollouts — Progressive Delivery

### Why Standard Deployments Aren't Enough

```
Standard Kubernetes Deployment (rolling update):
├── Replaces pods gradually (maxSurge/maxUnavailable)
├── Health check: readiness probe passes
├── Problem: readiness probe passing ≠ application working correctly
│   ├── 0% error rate on health endpoint
│   ├── But 5% 500 errors on /api/orders
│   ├── Or P99 latency jumped from 200ms to 2s
│   └── Rolling update doesn't know, doesn't care, keeps rolling
├── Rollback: manual (kubectl rollout undo), slow, requires human
└── Blast radius: by the time you notice, 100% of pods are new version

Argo Rollouts:
├── Canary: send 5% of traffic → measure → 25% → measure → 100%
├── Blue-Green: new version gets 0% traffic until explicitly promoted
├── Analysis: automated metrics checking (Prometheus, Datadog, CloudWatch)
├── Automatic rollback: if metrics degrade, rollback WITHOUT human
└── Blast radius: limited to canary percentage at any given time
```

### Architecture

```
┌────────────────────────────────────────────────────────────┐
│                  Argo Rollouts Controller                   │
│                  (Deployment in argo-rollouts namespace)    │
│                                                             │
│  Watches: Rollout CRDs                                      │
│  Manages: ReplicaSets (canary + stable)                     │
│  Integrates with:                                           │
│  ├── Istio (VirtualService traffic splitting)               │
│  ├── AWS ALB (weighted target groups)                       │
│  ├── Nginx Ingress (canary annotations)                     │
│  ├── SMI (Service Mesh Interface)                           │
│  └── Traefik, Ambassador, etc.                              │
│                                                             │
│  Traffic management:                                        │
│  ├── Istio: Rollouts modifies VirtualService weights        │
│  ├── ALB: Rollouts modifies Ingress annotations             │
│  └── Nginx: Rollouts creates canary Ingress                 │
└────────────────────────────────────────────────────────────┘
```

### Canary Deployment — Full Production Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
  namespace: order
spec:
  replicas: 10
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: order-service
  
  # Pod template (same as Deployment)
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/order-service:v2.1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              memory: "1Gi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            periodSeconds: 5

  strategy:
    canary:
      # Traffic management via Istio
      trafficRouting:
        istio:
          virtualServices:
            - name: order-service-vsvc     # ArgoCD manages this VirtualService
              routes:
                - primary                   # Route name in VirtualService
          destinationRule:
            name: order-service-destrule
            canarySubsetName: canary
            stableSubsetName: stable

      # The canary steps — THIS IS THE ROLLOUT PLAN
      steps:
        # Step 1: Send 5% traffic to canary, run analysis
        - setWeight: 5
        - analysis:
            templates:
              - templateName: success-rate
              - templateName: latency-check
            args:
              - name: service-name
                value: order-service
        
        # Step 2: If analysis passed, bump to 25%
        - setWeight: 25
        - pause: { duration: 5m }    # Bake time — let metrics accumulate
        - analysis:
            templates:
              - templateName: success-rate
              - templateName: latency-check
            args:
              - name: service-name
                value: order-service

        # Step 3: 50% traffic
        - setWeight: 50
        - pause: { duration: 10m }   # Longer bake at higher traffic
        - analysis:
            templates:
              - templateName: success-rate
              - templateName: latency-check
              - templateName: error-budget   # Stricter at higher traffic
            args:
              - name: service-name
                value: order-service

        # Step 4: 100% — full promotion
        - setWeight: 100
        - pause: { duration: 5m }    # Final bake
        - analysis:
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: order-service

      # What happens if analysis fails at ANY step:
      # → Automatic rollback to stable version
      # → Canary ReplicaSet scaled to 0
      # → All traffic back to stable
      # → Notification sent to Slack

      # Anti-affinity — spread canary and stable across nodes
      antiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          weight: 100

      # Scale down delay — keep old ReplicaSet for fast rollback
      scaleDownDelaySeconds: 300     # 5 min before scaling down old

      # Abort if rollout takes too long overall
      progressDeadlineAbort: true
      progressDeadlineSeconds: 1800   # 30 min max for entire rollout
```

### AnalysisTemplate — Automated Metrics Checking

```yaml
# Success Rate Analysis (Prometheus)
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      # How often to check
      interval: 60s
      
      # How many checks before declaring success
      count: 5
      
      # How many failures allowed before declaring failure
      failureLimit: 1
      
      # The actual Prometheus query
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(
              istio_requests_total{
                destination_service=~"{{args.service-name}}.*",
                response_code!~"5.*",
                reporter="destination"
              }[5m]
            )) /
            sum(rate(
              istio_requests_total{
                destination_service=~"{{args.service-name}}.*",
                reporter="destination"
              }[5m]
            )) * 100
      
      # Success condition: success rate must be >= 99.5%
      successCondition: result[0] >= 99.5
      failureCondition: result[0] < 95.0
      # Between 95-99.5%: inconclusive (keep checking)

---
# Latency Analysis
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency-check
spec:
  args:
    - name: service-name
  metrics:
    - name: p99-latency
      interval: 60s
      count: 5
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            histogram_quantile(0.99,
              sum(rate(
                istio_request_duration_milliseconds_bucket{
                  destination_service=~"{{args.service-name}}.*",
                  reporter="destination"
                }[5m]
              )) by (le)
            )
      successCondition: result[0] <= 500    # P99 must be <= 500ms
      failureCondition: result[0] > 2000    # Fail if P99 > 2s

---
# Error Budget Analysis (SRE pattern)
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-budget
spec:
  args:
    - name: service-name
  metrics:
    - name: error-budget-remaining
      interval: 120s
      count: 3
      failureLimit: 0     # Zero tolerance — if error budget is spent, abort
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            # Remaining error budget for the month
            # SLO: 99.95% availability = 0.05% error budget
            1 - (
              sum(increase(
                istio_requests_total{
                  destination_service=~"{{args.service-name}}.*",
                  response_code=~"5.*"
                }[30d]
              )) /
              sum(increase(
                istio_requests_total{
                  destination_service=~"{{args.service-name}}.*"
                }[30d]
              ))
            ) - 0.9995
      successCondition: result[0] > 0     # Still have error budget
      failureCondition: result[0] <= 0    # Budget exhausted — no deploys!
```

### Blue-Green Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service      # Payment service — zero tolerance for errors
spec:
  replicas: 10
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
        - name: payment-service
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/payment-service:v3.0.0

  strategy:
    blueGreen:
      # Active service (receives production traffic)
      activeService: payment-service-active
      
      # Preview service (new version, internal access only)
      previewService: payment-service-preview
      
      # Auto-promote after analysis passes
      autoPromotionEnabled: true
      
      # How long preview runs before auto-promotion
      autoPromotionSeconds: 300    # 5 min preview + analysis
      
      # Scale down old version delay
      scaleDownDelaySeconds: 600   # Keep blue for 10 min (fast rollback)
      scaleDownDelayRevisionLimit: 1
      
      # Pre-promotion analysis
      prePromotionAnalysis:
        templates:
          - templateName: success-rate
          - templateName: latency-check
        args:
          - name: service-name
            value: payment-service

      # Post-promotion analysis (verify after traffic switch)
      postPromotionAnalysis:
        templates:
          - templateName: success-rate
        args:
          - name: service-name
            value: payment-service

      # Anti-affinity between blue and green pods
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}
```

```
BLUE-GREEN FLOW:

1. Current state: Blue (v2.9) serving 100% traffic
                  ┌────────────┐
   Users ───────► │ Active Svc │ ──► Blue pods (v2.9) ✅
                  └────────────┘

2. New version deployed: Green (v3.0) starts, gets preview service
                  ┌────────────┐
   Users ───────► │ Active Svc │ ──► Blue pods (v2.9) ✅
                  └────────────┘
                  ┌────────────┐
   QA/Tests ────► │ Preview Svc│ ──► Green pods (v3.0) 🔵
                  └────────────┘

3. Pre-promotion analysis runs against preview service
   Prometheus checks success-rate and latency...

4. Analysis passes → Active service switches to green
                  ┌────────────┐
   Users ───────► │ Active Svc │ ──► Green pods (v3.0) ✅
                  └────────────┘
                  Blue pods (v2.9) kept for 10 min (rollback safety)

5. Post-promotion analysis runs against active traffic
   If fails → automatic rollback to blue

6. After scaleDownDelay, blue pods terminated
```

### Canary vs Blue-Green — When to Use Which

| Aspect | Canary | Blue-Green |
|--------|--------|------------|
| Traffic shifting | Gradual (5% → 25% → 50% → 100%) | All-or-nothing switch |
| Resource cost | Low (canary = small % of replicas) | High (2x replicas during deploy) |
| Rollback speed | Instant (just route back to stable) | Instant (switch service back) |
| Risk exposure | Minimal (only canary % sees new code) | Preview has no real user traffic until switch |
| Testing with real traffic | ✅ Real users hit canary at low % | ❌ Preview only gets synthetic tests |
| Database migrations | Difficult (two versions reading same DB) | Same difficulty |
| Best for | Most services, gradual confidence | Critical services (payment), major version changes |
| NovaMart pattern | Default for all services | Payment, auth, checkout (zero-error-tolerance) |

---

## 3. Progressive Delivery — The Complete Picture

### Feature Flags + Canary = Full Control

```
WITHOUT feature flags:
  Deploy v2 with new checkout flow → 5% canary → if broken, rollback entire deploy

WITH feature flags:
  Deploy v2 with new checkout flow behind flag (disabled) → 100% deploy
  Enable flag for 1% of users → measure → 5% → 25% → 100%
  If broken: disable flag instantly (no redeploy needed)

COMBINED (NovaMart pattern):
  1. Deploy code with feature flag (disabled) via Argo Rollouts canary
  2. Canary validates infrastructure (no crashes, no OOM, no latency spike)
  3. After full rollout, enable feature flag gradually via LaunchDarkly/Flagsmith
  4. Feature flag validates business logic (conversion rate, error rate)
  5. After full flag rollout, remove flag in next deploy (tech debt cleanup)
```

### Header-Based Routing (Testing in Production)

```yaml
# Istio VirtualService — route specific users to canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    # QA team sees canary version (header-based)
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: canary

    # Internal testing by user ID
    - match:
        - headers:
            x-user-id:
              regex: "^(test-user-1|test-user-2|qa-.*)"
      route:
        - destination:
            host: order-service
            subset: canary

    # Everyone else → stable
    - route:
        - destination:
            host: order-service
            subset: stable
            weight: 95
        - destination:
            host: order-service
            subset: canary
            weight: 5
```

### The Complete CI/CD Flow at NovaMart

```
Developer pushes to feature branch
         │
         ▼
Bitbucket webhook → Jenkins
         │
         ▼
┌─────────────────────────────────────┐
│ Jenkins Pipeline                     │
│                                      │
│ 1. Shallow clone                     │
│ 2. Build + Unit test                 │
│ 3. SonarQube analysis                │
│ 4. Quality Gate (wait for webhook)   │
│ 5. Kaniko image build + push to ECR  │
│ 6. Trivy scan (fail on CRITICAL)     │
│ 7. BlackDuck SCA (fail on policy)    │
│ 8. Push to Artifactory               │
│ 9. Update manifests repo (dev)       │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│ ArgoCD detects manifest change       │
│                                      │
│ DEV: auto-sync, no approval          │
│ ┌─────────────────────────────┐      │
│ │ Argo Rollout (canary)       │      │
│ │ 5% → analysis → 100%        │      │
│ │ Analysis: dev Prometheus    │      │
│ └─────────────────────────────┘      │
│                                      │
│ STAGING: auto-sync after dev green   │
│ ┌─────────────────────────────┐      │
│ │ Argo Rollout (canary)       │      │
│ │ 10% → 50% → 100%            │      │
│ │ Analysis: staging Prometheus│      │
│ │ + integration test Job hook │      │
│ └─────────────────────────────┘      │
│                                      │
│ PROD: manual sync (input gate)       │
│ ┌─────────────────────────────┐      │
│ │ Sync window check           │      │
│ │ Argo Rollout (canary)       │      │
│ │ 5% → analysis → 25% →       │      │
│ │ analysis → 50% → analysis → │      │
│ │ 100% → post-promotion       │      │
│ │ Full analysis suite:        │      │
│ │ - success rate ≥ 99.5%      │      │
│ │ - P99 latency ≤ 500ms       │      │
│ │ - error budget remaining    │      │
│ │ - no new error log patterns │      │
│ └─────────────────────────────┘      │
│                                      │
│ Slack notifications at every stage   │
└─────────────────────────────────────┘
```

### How Argo Rollouts Breaks

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **Analysis flapping** | Rollout stuck, retrying analysis endlessly | Metrics query returns inconsistent results, low traffic | Increase `count`, use `failureLimit`, ensure sufficient traffic for statistical significance |
| **Stuck at 0% canary** | Canary pod not getting traffic | Istio VirtualService not updated, DestinationRule missing subsets | Check rollout status: `kubectl argo rollouts get rollout <name>`, verify VirtualService weights |
| **Canary passes but prod breaks** | 5% canary was healthy, 100% isn't | Canary hit only cache, not cold paths; resource exhaustion at scale | Add load testing to analysis, test at meaningful traffic percentages, include resource metrics |
| **Analysis timeout** | Rollout hangs at analysis step | Prometheus unreachable, query returns no data, `count` too high | Add `initialDelay`, verify Prometheus connectivity, use `failureCondition` to catch empty results |
| **Two Rollouts racing** | Both trying to update same VirtualService | Rapid successive deploys, first not finished | `progressDeadlineAbort: true`, or manual abort of first before starting second |
| **Database schema mismatch** | Canary reads old schema, errors | Migration ran for new version, old version can't read it | Schema changes must be backward-compatible, expand-then-contract migration pattern |
| **Rollback cascading** | Rollback triggers dependent service rollback | Service A canary fails, but service B depends on A's new API | Independent service versioning, API backward compatibility, contract testing |


---

## Quick Reference Card

```
ARGOCD ARCHITECTURE:
  server (UI/API) + repo-server (Git/Helm/Kustomize) + 
  controller (reconciliation) + Redis (cache) + notifications
  No external DB — all state in K8s etcd (Application CRDs)
  Reconciliation: every 3min OR webhook-triggered (prefer webhook)

APPPROJECT:
  sourceRepos + destinations + clusterResourceWhitelist + syncWindows + roles
  Payment team can only deploy to payment-* namespaces
  Sync windows: deny Christmas, allow hotfixes always

APPLICATIONSET:
  Generators: git-directory, matrix, list, PR, SCM provider
  DANGER: empty generator output = mass delete
  PROTECTION: preserveResourcesOnDeletion: true, progressive sync

SYNC WAVES:
  Lower number first: -2 (namespace) → -1 (secrets) → 0 (config) → 1 (deploy) → 2 (service)
  Hooks: PreSync, Sync, PostSync, SyncFail
  hook-delete-policy: BeforeHookCreation (most common)

ARGO ROLLOUTS — CANARY:
  setWeight → analysis → setWeight → analysis → 100%
  AnalysisTemplate: Prometheus query + successCondition
  Auto-rollback on failure (no human needed)
  trafficRouting: Istio VirtualService weight manipulation

ARGO ROLLOUTS — BLUE-GREEN:
  activeService + previewService
  prePromotionAnalysis → switch → postPromotionAnalysis
  scaleDownDelaySeconds for rollback safety
  Use for: payment, auth, critical services

ANALYSIS TEMPLATES:
  success-rate: 5xx / total < threshold
  latency: histogram_quantile P99 < threshold
  error-budget: remaining budget > 0
  interval + count + failureLimit = confidence

PROGRESSIVE DELIVERY PATTERN:
  CI (Jenkins) → Image → Manifest repo → ArgoCD → Argo Rollouts
  Dev: auto, fast canary | Staging: auto, thorough canary
  Prod: manual trigger, full analysis suite, sync windows
```

---

## Retention Questions

**Q1.** NovaMart has 200 services deployed via ArgoCD using an ApplicationSet with a Git directory generator. At 9 AM, you get a Slack alert: 47 Applications just went to "Missing" status and resources are being deleted from the production cluster. What happened, how do you stop the bleeding immediately, and what permanent safeguards do you implement?

**Q2.** You're designing the Argo Rollouts canary strategy for NovaMart's checkout-service (handles $50K/minute in transactions). Write the complete Rollout spec with: canary steps, analysis templates (success rate, latency, error budget), traffic routing via Istio, and explain your choices for each parameter. What's different about this compared to a low-criticality internal reporting service?

**Q3.** The order-service team complains that their ArgoCD Application always shows "OutOfSync" even though they haven't made any changes. Every time they click "Sync," it syncs successfully, but within 3 minutes it shows OutOfSync again. Diagnose this systematically — what are the possible causes and how do you fix each one?

**Q4.** Draw (in text) the complete flow from a developer's `git push` to traffic reaching the new version in production. Include every system touched (Bitbucket, Jenkins, ECR, Artifactory, ArgoCD, Argo Rollouts, Istio, Prometheus). For each transition point, identify what can go wrong and what the observable symptom would be.

Q1: The ApplicationSet "Death Spiral"

What happened:
You have a Git Directory Generator in your ApplicationSet. This generator automatically creates Application objects based on the files/folders it finds in a specific Git path.

Someone (likely a developer or a script) accidentally deleted or renamed folders in that Git directory. Because the ApplicationSet is designed to keep the cluster in sync with the Git directory, it immediately deleted the 47 Application objects from the cluster. Since these Applications likely had syncPolicy.automated.prune: true, ArgoCD then proceeded to delete all the actual resources (Deployments, Services, Ingresses) associated with those 47 services.

How to stop the bleeding immediately:

Disable Auto-Sync on the ApplicationSet: Immediately edit the ApplicationSet and set automated: null or remove the syncPolicy block. This prevents ArgoCD from performing any further deletions while you fix the source.

Revert the Git Commit: Perform a git revert on the commit that deleted the folders in the Git directory. Once the folders are back, the ApplicationSet will recreate the Application objects.

Manual Sync: Trigger a manual sync of the recreated Applications to restore the production resources.

Permanent Safeguards:

Disable Pruning on ApplicationSets: Set prune: false in the ApplicationSet. This means if a folder is deleted in Git, the Application object might vanish, but the actual resources in the cluster stay alive. You then perform a manual cleanup.

Protected Folders/CODEOWNERS: Implement a CODEOWNERS file in the Git repository. Require mandatory approval from the Platform Team for any changes to the directory used by the ApplicationSet generator.

Application-Level Finalizers: Implement a custom finalizer on critical resources to prevent them from being deleted without a manual "unlock" step.

Q2: High-Stakes Canary for checkout-service

For a service handling $50K/minute, we cannot risk a "big bang" release. We need a Tight Feedback Loop.

Rollout Spec:

code
Yaml
download
content_copy
expand_less
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: checkout-service
spec:
  strategy:
    canary:
      analysis:
        templates:
          - templateName: success-rate-check
            args:
              - name: service-name
                value: checkout-service
      steps:
        - setWeight: 1   # Start very small
        - pause: { duration: 5m }
        - setWeight: 5
        - pause: { duration: 10m }
        - setWeight: 20
        - pause: { duration: 30m } # Long soak time for payment edge cases
        - setWeight: 50
        - pause: { duration: 1h }
        - setWeight: 100

Analysis Template (The Guardrail):

code
Yaml
download
content_copy
expand_less
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate-check
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.999 # 99.9% Success rate
      failureLimit: 2 # Allow 2 failures before aborting
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(irate(istio_requests_total{reporter="destination",response_code!~"5.*",destination_service="checkout-service"}[1m])) 
            / 
            sum(irate(istio_requests_total{reporter="destination",destination_service="checkout-service"}[1m]))

Design Choices:

Istio Routing: I use Istio because it allows for precise weighted traffic shifting at the L7 layer. Unlike K8s services (which are random), Istio ensures exactly 1% of requests go to v2.

Strict Success Rate (99.9%): Because this is the checkout service, a 1% error rate is a financial disaster.

Long Soak Times: I use a 1-hour soak at 50% to catch "slow leak" issues (e.g., memory leaks or database connection saturation) that don't appear in a 5-minute test.

Comparison with Low-Criticality Service:
For an internal reporting service, I would use:

Aggressive Steps: 10% 
→
→
 50% 
→
→
 100% (No 1% start).

Loose Error Budget: 95% success rate is acceptable.

Shorter Pauses: 5-minute windows instead of 1 hour.

Q3: The "OutOfSync" Fight (Drift vs. Desired State)

Diagnosis:
This is a Conflict between two Controllers. ArgoCD is trying to enforce the state in Git, but something else in the cluster is modifying the resource in real-time.

Possible Causes & Fixes:

HPA (Horizontal Pod Autoscaler):

Cause: Git says replicas: 3. HPA sees high load and scales to replicas: 5. ArgoCD sees the difference 
→
→
 OutOfSync. You click Sync 
→
→
 ArgoCD sets it back to 3 
→
→
 HPA immediately sets it back to 5 
→
→
 OutOfSync.

Fix: Use ignoreDifferences in the ArgoCD Application spec to ignore the replicas field.

Admission Webhooks / Mutating Controllers:

Cause: An Istio sidecar injector or a security policy agent is adding labels or annotations to the pod at the moment of creation. These labels aren't in Git.

Fix: Use ignoreDifferences for the specific labels/annotations being added.

Service Mesh (Istio) Automatic Updates:

Cause: Istio might be updating the DestinationRule or VirtualService based on internal discovery.

Fix: Ensure the labels used by Istio are explicitly defined in Git or ignored by ArgoCD.

Q4: The Complete GitOps Data Flow
code
Ascii
download
content_copy
expand_less
[Dev Push] ──► [Bitbucket] ──► [Jenkins] ──► [Artifactory/ECR]
                                    │              │
                                    │      (Update Image Tag)
                                    │              ▼
                                    └────► [GitOps Manifest Repo]
                                                   │
                                                   ▼
                                            [ArgoCD Controller]
                                                   │
                                                   ▼
                                            [Argo Rollouts] ──► [Istio] ──► [Traffic]
                                                   ▲               │
                                                   └──── [Prometheus] ◄────┘

Transition Points & Failure Modes:

Transition	What can go wrong?	Observable Symptom
Bitbucket 
→
→
 Jenkins	Webhook failure / Network glitch	No build starts after a push.
Jenkins 
→
→
 ECR/Artifactory	Auth failure / Registry full	Pipeline fails at "Push Image" stage.
Jenkins 
→
→
 Manifest Repo	Git Merge Conflict / Permission denied	Pipeline fails at "Update Tag" stage.
Manifest Repo 
→
→
 ArgoCD	Sync Policy failure / App a broken	App shows OutOfSync or Degraded.
ArgoCD 
→
→
 Rollouts	Resource quota exceeded / ImagePullBackOff	Pods stuck in ContainerCreating.
Rollouts 
→
→
 Istio	VirtualService mismatch / Subset missing	503 Service Unavailable / Traffic not shifting.
Prometheus 
→
→
 Rollouts	Metric timeout / Query error	Canary "Aborted" or hangs indefinitely.
Q5: The 3 AM Node Failure Triage

Step 1: Impact Assessment (Triage)
I immediately check the Load Balancer and Core Service Health.

If the 3 critical API pods were distributed across the cluster, the service is likely still up.

If they were all on ip-10-0-1-50, we have a SEV1 Outage.

Step 2: Node Diagnosis (The "Why")
I need to know if the node is dead or just unreachable.

Cloud Console: Check AWS EC2 Status Checks. If "System Status Check" failed, it's an AWS hardware issue.

SSH Attempt: Try to SSH. If it times out, the kernel is likely panicked or the network interface is down.

Kubelet Logs: If I can get in, I check journalctl -u kubelet to see if it crashed due to memory pressure or disk failure.

Step 3: Recovery (The "Fix")
Since the node is NotReady, K8s will eventually evict the pods, but the default timeout (5m) is too slow for critical services.

Force Evacuation: I will manually taint the node with NoExecute:
kubectl taint nodes ip-10-0-1-50 outage=true:NoExecute
This forces the pods to move to healthy nodes immediately.

Verify Pod Health: I monitor kubectl get pods -w to ensure the critical APIs are Running and Ready on other nodes.

Step 4: Post-Mortem (Prevention)

Pod Topology Spread Constraints: I will implement topologySpreadConstraints for the critical API pods to ensure they are never all placed on the same node.

Node Problem Detector: Deploy the Node Problem Detector to catch "Kernel Panic" or "Disk Failure" patterns and alert us before the node goes NotReady.
---

## Q1: ApplicationSet Mass-Delete — 3.5/5

**What you nailed:**
- Correctly identified that the Git directory generator producing fewer results leads to Application deletion — right mechanism.
- Git revert as a recovery step — correct.
- CODEOWNERS as a prevention mechanism — good governance thinking.

**What you got wrong or missed:**

1. **Your root cause is the SECOND most likely cause, not the first.** You said "someone deleted or renamed folders." The MORE likely cause — and the one the lesson explicitly covered — is a **Git authentication failure or network issue**. If the repo-server can't clone the repo (expired token, Bitbucket outage, DNS failure), the generator produces **zero results** because it can't read the directory. Zero results = zero Applications = delete everything. This is more dangerous than human error because it's transient and automated.

2. **You missed the single most important safeguard: `preserveResourcesOnDeletion: true`.** This was explicitly covered in the lesson with a code example. This one setting prevents the ApplicationSet from deleting cluster resources even when Applications are removed. Your answer said "set `prune: false`" which is close but not the same thing — `prune` is on the Application's syncPolicy, `preserveResourcesOnDeletion` is on the ApplicationSet spec itself. The distinction matters: prune controls whether ArgoCD removes resources that are no longer in Git manifests. preserveResourcesOnDeletion controls whether deleting the Application CRD itself cascades to deleting cluster resources.

3. **Your immediate response is too slow.** "Edit the ApplicationSet and set automated: null" requires you to write YAML, apply it, and wait for the controller to reconcile. While you're editing, deletions are continuing. The faster response:

```bash
# IMMEDIATE (10 seconds): Scale down the applicationset-controller
kubectl scale deployment argocd-applicationset-controller \
    -n argocd --replicas=0

# THEN investigate, fix Git, add safeguards
# THEN scale back up
```

This is the "stop the bleeding" pattern — kill the controller that's causing damage, THEN fix the root cause.

4. **You didn't mention the progressive sync strategy** that was covered in the lesson:

```yaml
strategy:
  type: RollingSync
  rollingSync:
    steps:
      - matchExpressions:
          - key: env
            operator: In
            values: [staging]
      - matchExpressions:
          - key: env
            operator: In
            values: [prod]
        maxUpdate: 25%
```

This ensures that even if the generator does something unexpected, it can only affect 25% of prod apps at a time, giving you time to intervene.

5. **Finalizers on resources don't help here.** The issue isn't that Kubernetes resources are being deleted too fast — it's that ArgoCD Application CRDs are being deleted, which triggers the cascade. The correct protection is on the ApplicationSet and Application level, not on the downstream resources.

---

## Q2: Checkout-Service Canary — 3.8/5

**What you nailed:**
- Starting at 1% for a $50K/min service — correct instinct.
- Long soak times (30min, 1hr) — excellent, catches slow leaks.
- 99.9% success rate threshold — appropriate for checkout.
- Comparison with low-criticality service is directionally correct.

**What you missed:**

1. **No traffic routing configuration.** You mentioned "I use Istio" but didn't include the `trafficRouting` block in your Rollout spec. Without it, Argo Rollouts can't actually shift traffic — it just scales ReplicaSets. The critical piece:

```yaml
trafficRouting:
  istio:
    virtualServices:
      - name: checkout-service-vsvc
        routes:
          - primary
    destinationRule:
      name: checkout-service-destrule
      canarySubsetName: canary
      stableSubsetName: stable
```

Without this, your 1% weight is meaningless — Kubernetes Service does random round-robin across all pods, so traffic split is proportional to pod count, not your specified weight.

2. **Analysis is configured wrong.** You put `analysis` at the top level of the canary strategy, which makes it a **background analysis** that runs continuously. What you want is **step-level analysis** that runs at each weight checkpoint:

```yaml
steps:
  - setWeight: 1
  - pause: { duration: 5m }
  - analysis:            # ← Step-level: runs here, blocks until pass/fail
      templates:
        - templateName: success-rate-check
        - templateName: latency-check    # You're missing this entirely
        - templateName: error-budget     # And this
  - setWeight: 5
  # ...
```

3. **Missing latency analysis.** For checkout, P99 latency is as important as success rate. A checkout that takes 10 seconds but returns 200 OK still costs you conversions. The lesson provided an explicit latency AnalysisTemplate — your answer only has success rate.

4. **Missing error budget analysis.** The lesson showed an error budget template that prevents deploys when the monthly error budget is exhausted. For a $50K/min service, this is non-negotiable — if you've already burned your error budget this month, you should NOT be deploying to checkout.

5. **Missing operational safeguards:**
   - `progressDeadlineSeconds: 3600` (abort if entire rollout takes > 1hr)
   - `scaleDownDelaySeconds: 600` (keep old version for fast rollback)
   - `antiAffinity` (don't colocate canary and stable on same node)
   - `progressDeadlineAbort: true` (actually abort, not just mark degraded)

6. **Your Prometheus query uses `irate` instead of `rate`.** `irate` uses only the last two data points — it's noisy and inappropriate for analysis that needs stable measurements over time. Use `rate` with a 5m window for analysis decisions.

---

## Q3: Perpetual OutOfSync — 4.2/5

**What you nailed:**
- HPA as the #1 cause — exactly right, and this is the most common cause in production.
- Mutating admission webhooks (Istio sidecar injection) — correct, second most common.
- `ignoreDifferences` as the fix — correct mechanism.
- "Conflict between two controllers" framing — good mental model.

**What you missed:**

1. **You didn't show the actual ignoreDifferences config.** Knowing the concept isn't enough — you need to know the syntax:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  ignoreDifferences:
    # HPA manages replicas
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
    # Istio injects sidecar
    - group: apps
      kind: Deployment
      jqPathExpressions:
        - .spec.template.metadata.annotations."sidecar.istio.io/*"
    # Service gets defaulted fields
    - group: ""
      kind: Service
      jsonPointers:
        - /spec/clusterIP
        - /spec/clusterIPs
```

At 3 AM when this is happening, you need to write this from memory, not look it up.

2. **Missing cause: Kubernetes API server defaulting.** When you submit a resource, the API server adds default values that aren't in your Git manifest. For example:
   - `Service` gets `clusterIP` assigned
   - `Deployment` gets `strategy.rollingUpdate.maxSurge: 25%` if not specified
   - `PodSpec` gets `terminationGracePeriodSeconds: 30` defaulted

   ArgoCD compares your Git manifest (without defaults) against the live resource (with defaults) and sees drift. Fix: either explicitly set all defaulted fields in Git, or use `ignoreDifferences`.

3. **Missing cause: `kubectl edit` or `kubectl apply` by a human.** Someone manually edited a resource in the cluster (maybe during an incident). ArgoCD detects the drift. This is actually the **intended behavior** — ArgoCD SHOULD show OutOfSync when someone manually changes things. The fix here is `selfHeal: true` (auto-revert manual changes) and education (don't kubectl edit in GitOps environments).

4. **Missing the diagnostic approach.** Before jumping to causes, you should describe HOW to diagnose:

```bash
# Step 1: See what ArgoCD thinks is different
argocd app diff order-service

# Step 2: Compare rendered manifest vs live
argocd app manifests order-service --source live > live.yaml
argocd app manifests order-service --source git > git.yaml
diff live.yaml git.yaml

# Step 3: Check which specific fields are different
# This tells you exactly what's causing OutOfSync
```

---

## Q4: Complete GitOps Flow — 3.5/5

**What you nailed:**
- The overall flow direction is correct (push → build → image → manifests → ArgoCD → Rollouts → Istio).
- Prometheus feedback loop to Rollouts — correct.
- Failure mode table covers the major transition points.

**What you missed critically:**

1. **Your diagram is too simplified for what was asked.** The question said "include every system touched." You're missing: SonarQube (quality gate), Trivy (image scan), BlackDuck (SCA), the Quality Gate webhook back to Jenkins, Slack notifications at multiple points, and the separate manifests repo (you have it but don't clearly distinguish code repo from manifests repo).

2. **Missing transition points in your failure table:**

| Transition You Missed | Failure Mode | Observable Symptom |
|----------------------|--------------|-------------------|
| Jenkins → SonarQube | CE queue backed up, webhook misconfigured | Pipeline hangs at "Quality Gate" for 5+ min, then timeout |
| SonarQube → Jenkins (webhook) | SonarQube webhook URL wrong, Jenkins ingress changed | waitForQualityGate timeout — pipeline fails even though code is clean |
| Jenkins → Trivy | DB download fails, ghcr.io rate limit | "FATAL: DB update error" — pipeline fails at scan stage |
| ECR → EKS (kubelet pull) | ECR token expired, cross-account IRSA misconfigured | Pods in `ImagePullBackOff` — ArgoCD shows app "Degraded" |
| Argo Rollouts → Prometheus | Prometheus down, query returns empty | Analysis hangs as "Inconclusive" indefinitely, rollout stuck |
| ArgoCD → Git (manifest read) | Token expired, repo-server OOM | ALL apps show "Unknown" status, no syncs possible |

3. **Your failure symptoms are too vague.** "Pipeline fails at Push Image stage" — what does the engineer see? Is it a 401 from ECR? A timeout? A disk full error on the agent? The symptom should be specific enough that someone can grep logs for it.

4. **Missing: the TWO repos pattern.** This is fundamental to GitOps. You have a **code repo** (where developers push code) and a **manifests repo** (where image tags are updated). Jenkins bridges them. Your diagram doesn't clearly show this separation. Many failure modes exist at this boundary (Jenkins has permission to read code repo but not write to manifests repo, merge conflict in manifests repo from concurrent pipelines, etc.).

---

## Q5 (Unasked — Not Graded)

Brief acknowledgment: Your node failure triage is reasonable. The taint with `NoExecute` for immediate evacuation is correct. TopologySpreadConstraints for prevention is correct. Node Problem Detector is a good addition. The main gap is that you didn't check PodDisruptionBudgets before force-evicting, and you didn't mention that Karpenter (NovaMart's autoscaler) would automatically detect the unhealthy node and provision a replacement.

---


---

# Phase 3: CI/CD Pipelines
## Lesson 4: CI Platform Comparison — GitHub Actions, GitLab CI, AWS CodePipeline

---

## Why This Lesson Exists

NovaMart uses Jenkins. But you will encounter other CI platforms in interviews, in future roles, and potentially in a migration project at NovaMart itself. You need to:

1. **Read and understand** any CI platform's pipeline definition
2. **Compare tradeoffs** intelligently (not "GitHub Actions is better because it's newer")
3. **Design a migration path** if the business case justifies it
4. **Know what each platform does better and worse** than Jenkins

This lesson is breadth, not depth. You won't become an expert in GitHub Actions in one lesson. But you'll know enough to operate, evaluate, and interview confidently.

---

## 1. GitHub Actions — The Dominant Platform (2025)

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitHub.com / GitHub Enterprise               │
│                                                                 │
│  ┌─────────────┐    ┌───────────────────────────────────────┐   │
│  │ Repository  │    │         GitHub Actions Service        │   │
│  │             │    │                                       │   │
│  │ .github/    │    │  ┌──────────────┐  ┌──────────────┐   │   │
│  │  workflows/ │───►│  │ Orchestrator │  │ Runner Mgmt  │   │   │
│  │   ci.yml    │    │  │ (reads YAML, │  │ (assigns job │   │   │
│  │   deploy.yml│    │  │  creates jobs│  │  to runner)  │   │   │
│  │             │    │  └──────┬───────┘  └──────┬───────┘   │   │
│  └─────────────┘    │         │                 │           │   │
│                     └─────────┼─────────────────┼───────────┘   │
│                               │                 │               │
│              ┌────────────────┼─────────────────┼───────┐       │
│              │                ▼                 ▼       │       │
│              │  ┌──────────────────┐  ┌───────────────┐ │       │
│              │  │ GitHub-Hosted    │  │ Self-Hosted   │ │       │
│              │  │ Runners          │  │ Runners       │ │       │
│              │  │                  │  │               │ │       │
│              │  │ - Fresh VM every │  │ - Your infra  │ │       │
│              │  │   job (clean!)   │  │ - Persistent  │ │       │
│              │  │ - Ubuntu, macOS, │  │ - EKS pods,   │ │       │
│              │  │   Windows        │  │   EC2, on-prem│ │       │
│              │  │ - 2-core, 7GB    │  │ - Any size    │ │       │
│              │  │   (standard)     │  │ - Any OS      │ │       │
│              │  │ - Pre-installed  │  │               │ │       │
│              │  │   tools          │  │               │ │       │
│              │  └──────────────────┘  └───────────────┘ │       │
│              │        RUNNERS                           │       │
│              └──────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

**Key concept: GitHub-hosted runners are ephemeral VMs.** Every job gets a fresh VM. No workspace pollution, no stale cache (unless you explicitly cache). This is fundamentally different from Jenkins where agents can be persistent and accumulate state.

### Workflow Anatomy

```yaml
# .github/workflows/ci.yml
name: Order Service CI

# TRIGGERS — when does this workflow run?
on:
  push:
    branches: [main, 'release/**']
    paths:
      - 'services/order-service/**'    # Only if order-service changed
      - '.github/workflows/ci.yml'     # Or if this workflow changed
  pull_request:
    branches: [main]
    paths:
      - 'services/order-service/**'
  workflow_dispatch:                     # Manual trigger button in UI
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'dev'
        type: choice
        options: [dev, staging, prod]

# Permissions for OIDC (CRITICAL for AWS)
permissions:
  id-token: write      # Required for OIDC → AWS
  contents: read        # Read repo
  pull-requests: write  # Post PR comments (SonarQube, etc.)

# Environment variables (workflow-level)
env:
  REGISTRY: 123456789.dkr.ecr.us-east-1.amazonaws.com
  APP_NAME: order-service
  AWS_REGION: us-east-1

# Concurrency — prevent parallel runs for same branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true    # Kill old run if new push comes

jobs:
  # ─── JOB 1: Build & Test ───
  build:
    runs-on: ubuntu-latest    # GitHub-hosted runner
    # Or: runs-on: self-hosted  (your runner)
    # Or: runs-on: [self-hosted, linux, x64, gpu]  (labeled)

    outputs:
      image-tag: ${{ steps.meta.outputs.tag }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4      # Official action — pinned to v4
        with:
          fetch-depth: 0               # Full history for SonarQube

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'maven'               # Auto-caches ~/.m2/repository!

      - name: Build & Test
        working-directory: services/order-service
        run: |
          mvn clean verify -B -T 1C --no-transfer-progress
          # -B: batch mode, -T 1C: parallel

      - name: Generate Image Tag
        id: meta
        run: |
          TAG="${{ github.ref_name }}-$(git rev-parse --short HEAD)-${{ github.run_number }}"
          echo "tag=${TAG}" >> $GITHUB_OUTPUT

      - name: Upload Test Results
        if: always()    # Upload even if tests fail
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: services/order-service/target/surefire-reports/
          retention-days: 7

  # ─── JOB 2: SonarQube ───
  sonar:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v2
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}

      - name: Quality Gate
        uses: SonarSource/sonarqube-quality-gate-action@v1
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # ─── JOB 3: Security Scans (parallel) ───
  security:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        scan: [trivy, blackduck]
      fail-fast: false    # Don't cancel other scans if one fails
    steps:
      - uses: actions/checkout@v4

      - name: Trivy Scan
        if: matrix.scan == 'trivy'
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: 'fs'
          severity: 'HIGH,CRITICAL'
          exit-code: '1'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy to GitHub Security
        if: matrix.scan == 'trivy' && always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
          # Results appear in GitHub Security tab!

      - name: BlackDuck Scan
        if: matrix.scan == 'blackduck'
        uses: synopsys-sig/detect-action@v0.3.0
        with:
          scan-mode: RAPID    # Fast mode for PRs
          blackduck-url: ${{ vars.BLACKDUCK_URL }}
          blackduck-token: ${{ secrets.BLACKDUCK_TOKEN }}

  # ─── JOB 4: Build & Push Image ───
  image:
    needs: [build, sonar, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'    # Only on main branch

    steps:
      - uses: actions/checkout@v4

      # OIDC authentication to AWS — NO ACCESS KEYS!
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-ecr
          aws-region: ${{ env.AWS_REGION }}
          # OIDC: GitHub proves identity to AWS via JWT
          # No secrets stored! Role trust policy limits to this repo.

      - name: Login to ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          context: services/order-service
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.APP_NAME }}:${{ needs.build.outputs.image-tag }}
            ${{ env.REGISTRY }}/${{ env.APP_NAME }}:latest-main
          cache-from: type=gha        # GitHub Actions cache for layers!
          cache-to: type=gha,mode=max

  # ─── JOB 5: Update Manifests ───
  update-manifests:
    needs: image
    runs-on: ubuntu-latest
    steps:
      - name: Checkout manifests repo
        uses: actions/checkout@v4
        with:
          repository: novamart/k8s-manifests
          token: ${{ secrets.MANIFEST_REPO_PAT }}
          path: manifests

      - name: Update image tag
        run: |
          cd manifests/apps/order-service/overlays/dev
          kustomize edit set image \
            ${{ env.REGISTRY }}/${{ env.APP_NAME }}=${{ env.REGISTRY }}/${{ env.APP_NAME }}:${{ needs.build.outputs.image-tag }}
          git config user.name "github-actions"
          git config user.email "ci@novamart.com"
          git add .
          git commit -m "chore: update order-service to ${{ needs.build.outputs.image-tag }}"
          git push
```

### GitHub Actions OIDC → AWS (The Right Way)

```
┌────────────────┐     JWT Token      ┌──────────────┐
│ GitHub Actions│────────────────────►│  AWS STS     │
│ Runner        │  (signed by GitHub)│  AssumeRole   │
│               │◄────────────────────│  WithWebIdent│
│               │  Temporary creds    │              │
└───────────────┘  (15min-1hr)        └──────────────┘

IAM Trust Policy:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:novamart/order-service:ref:refs/heads/main"
        // ↑ CRITICAL: Lock to specific repo AND branch
        // Without this, ANY GitHub repo could assume this role
      }
    }
  }]
}
```

**Why OIDC matters:** No long-lived AWS access keys stored as GitHub Secrets. No key rotation. No risk of key leak. The JWT is short-lived and scoped. This is the **only acceptable pattern** for GitHub Actions → AWS in 2025.

### Reusable Workflows (GitHub's Shared Libraries)

```yaml
# .github/workflows/reusable-build.yml (in a shared repo)
name: Standard Build
on:
  workflow_call:    # This makes it callable from other repos
    inputs:
      app-name:
        required: true
        type: string
      language:
        required: true
        type: string
      java-version:
        required: false
        type: string
        default: '17'
    secrets:
      SONAR_TOKEN:
        required: true
    outputs:
      image-tag:
        description: "Built image tag"
        value: ${{ jobs.build.outputs.tag }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: |
          # ... build logic based on inputs.language
      - name: Output tag
        id: meta
        run: echo "tag=..." >> $GITHUB_OUTPUT

---
# Consuming workflow (in order-service repo)
# .github/workflows/ci.yml
name: CI
on: [push]
jobs:
  build:
    uses: novamart/shared-workflows/.github/workflows/reusable-build.yml@v2.3.0
    with:
      app-name: order-service
      language: java
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### Self-Hosted Runners on EKS (Actions Runner Controller)

```yaml
# Actions Runner Controller (ARC) — runs GitHub Actions jobs on K8s
# Similar concept to Jenkins Kubernetes plugin

# RunnerDeployment — pool of runners
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: novamart-runners
spec:
  replicas: 5
  template:
    spec:
      repository: novamart/order-service   # Or organization-level
      labels:
        - self-hosted
        - linux
        - novamart
      resources:
        requests:
          cpu: "2"
          memory: "4Gi"
        limits:
          cpu: "4"
          memory: "8Gi"
      # Ephemeral: runner pod deleted after each job
      ephemeral: true

---
# Horizontal Runner Autoscaler — scale based on queue
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: novamart-runners-autoscaler
spec:
  scaleTargetRef:
    name: novamart-runners
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
      scaleUpThreshold: '0.75'
      scaleDownThreshold: '0.25'
      scaleUpFactor: '2'
      scaleDownFactor: '0.5'
```

---

## 2. GitLab CI — The Integrated Platform

### Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    GitLab (SaaS or Self-Hosted)                │
│                                                                │
│  ┌───────────┐  ┌─────────────┐  ┌────────────────────────┐    │
│  │ Repository│  │ CI/CD       │  │ Built-in Features      │    │
│  │           │  │ Coordinator │  │                        │    │
│  │ Code      │  │             │  │ - Container Registry   │    │
│  │ MRs       │  │ Parses      │  │ - Package Registry     │    │
│  │ Issues    │  │ .gitlab-ci  │  │ - Security Dashboard   │    │
│  │           │  │ Assigns jobs│  │ - Environments/Deploy  │    │
│  │           │  │ to runners  │  │ - Feature Flags        │    │
│  └──────────┘  └──────┬──────┘  │ - Infrastructure (IaC)  │    │
│                       │          │ - Wiki, Pages          │    │
│                       │          └────────────────────────┘    │
│                       │                                        │
│              ┌────────▼────────┐                               │
│              │  GitLab Runners  │                              │
│              │                  │                              │
│              │  SaaS: shared    │                              │
│              │  Self-hosted:    │                              │
│              │  - Shell executor│                              │
│              │  - Docker exec.  │                              │
│              │  - K8s executor  │                              │
│              │  - Docker Machine│                              │
│              └──────────────────┘                              │
└────────────────────────────────────────────────────────────────┘
```

**GitLab's key differentiator:** Everything is built-in. Container registry, package registry, security scanning, feature flags, environments, monitoring — all in one product. No plugin ecosystem needed. The tradeoff: you're locked into GitLab's way of doing things.

### .gitlab-ci.yml Anatomy

```yaml
# .gitlab-ci.yml (always in repo root)

# Variables (like env in GitHub Actions)
variables:
  REGISTRY: 123456789.dkr.ecr.us-east-1.amazonaws.com
  APP_NAME: order-service
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

# Stages define the ORDER of execution
# Jobs in the same stage run in PARALLEL
stages:
  - build
  - test
  - scan
  - package
  - deploy

# Cache (persistent across pipelines — like PVC in Jenkins)
cache:
  key: ${CI_COMMIT_REF_SLUG}    # Cache per branch
  paths:
    - .m2/repository/
    - node_modules/

# ─── BUILD ───
build:
  stage: build
  image: maven:3.9.6-eclipse-temurin-17
  script:
    - mvn clean compile -B -T 1C --no-transfer-progress
  artifacts:
    paths:
      - target/
    expire_in: 1 hour    # Auto-cleanup!
  rules:                  # GitLab's "when" equivalent
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# ─── UNIT TEST ───
test:unit:
  stage: test
  image: maven:3.9.6-eclipse-temurin-17
  script:
    - mvn test -B
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml    # GitLab parses JUnit!
    paths:
      - target/site/jacoco/
  coverage: '/Total.*?([0-9]{1,3})%/'    # Regex to extract coverage %

# ─── SECURITY SCANS ───
trivy-scan:
  stage: scan
  image:
    name: aquasec/trivy:0.50.0
    entrypoint: [""]
  script:
    - trivy fs --severity HIGH,CRITICAL --exit-code 1 --format json --output trivy.json .
  artifacts:
    reports:
      # GitLab natively displays security findings in MR!
      container_scanning: trivy.json
  allow_failure: false

# GitLab built-in SAST (free!)
sast:
  stage: scan
# include:
#   - template: Security/SAST.gitlab-ci.yml
# GitLab auto-includes SAST scanners based on detected languages

# ─── DOCKER BUILD ───
docker-build:
  stage: package
  image: docker:24
  services:
    - docker:24-dind     # DinD as a "service" (sidecar)
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      docker build \
        --cache-from $CI_REGISTRY_IMAGE:latest \
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA \
        -t $CI_REGISTRY_IMAGE:latest .
      docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
      docker push $CI_REGISTRY_IMAGE:latest
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ─── DEPLOY ───
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:1.28
  environment:
    name: staging
    url: https://staging.novamart.com
  script:
    - kubectl set image deployment/order-service order-service=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-production:
  stage: deploy
  image: bitnami/kubectl:1.28
  environment:
    name: production
    url: https://novamart.com
  script:
    - kubectl set image deployment/order-service order-service=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual    # Manual gate for production
  needs:
    - deploy-staging  # Explicit dependency

# ─── PARENT-CHILD PIPELINES (GitLab's monorepo solution) ───
# trigger-order-service:
#   stage: build
#   trigger:
#     include: services/order-service/.gitlab-ci.yml
#     strategy: depend
#   rules:
#     - changes:
#         - services/order-service/**
```

### GitLab CI Unique Features

```
FEATURES GITLAB HAS THAT JENKINS DOESN'T (natively):
├── Merge Request pipelines (automatic, with MR widget showing results)
├── Environments with deployment tracking (who deployed what, when)
├── Review Apps (auto-deploy PR to unique URL, auto-cleanup)
├── Built-in container registry (per-project, free)
├── Built-in package registry (Maven, npm, PyPI, etc.)
├── Security dashboard (SAST, DAST, dependency scanning, container scanning)
├── Feature flags (built-in, no LaunchDarkly needed)
├── Auto DevOps (zero-config CI/CD for standard apps)
├── Directed Acyclic Graph (DAG) via "needs:" keyword
│   → Jobs can run as soon as their dependencies finish
│   → Not limited to stage ordering
├── Parent-child pipelines (monorepo trigger child pipelines)
├── Multi-project pipelines (trigger pipeline in another repo)
└── Pipeline analytics (duration trends, failure rates, wait times)
```

---

## 3. AWS CodePipeline — AWS-Native CI/CD

```
AWS CI/CD ecosystem:
├── CodeCommit     → Git repository (being deprecated 2024!)
├── CodeBuild      → Build service (like a single Jenkins agent)
├── CodeDeploy     → Deployment service (EC2, ECS, Lambda)
├── CodePipeline   → Orchestrator (connects the above)
└── CodeArtifact   → Package repository (like Artifactory)

┌─────────────────────────────────────────────────────────────┐
│                      CodePipeline                           │
│                                                             │
│  Source ──► Build ──► Test ──► Deploy                       │
│  (GitHub)   (Code    (Code    (Code                         │
│  (S3)       Build)    Build)   Deploy                       │
│  (ECR)                         ECS/EKS/Lambda)              │
│                                                             │
│  Each stage: 1+ action groups                               │
│  Each action: CodeBuild, Lambda, manual approval, etc.      │
└─────────────────────────────────────────────────────────────┘
```

```yaml
# buildspec.yml (CodeBuild's pipeline definition)
version: 0.2

env:
  variables:
    APP_NAME: order-service
  parameter-store:
    DB_PASSWORD: /novamart/prod/db-password    # SSM integration!
  secrets-manager:
    API_KEY: novamart/api-key                   # Secrets Manager!

phases:
  install:
    runtime-versions:
      java: corretto17
    commands:
      - echo Installing dependencies...

  pre_build:
    commands:
      - echo Logging in to ECR...
      - aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}

  build:
    commands:
      - mvn clean package -B
      - docker build -t $REGISTRY/$APP_NAME:$IMAGE_TAG .
      - docker push $REGISTRY/$APP_NAME:$IMAGE_TAG

  post_build:
    commands:
      - echo Build completed
      - printf '{"ImageURI":"%s"}' $REGISTRY/$APP_NAME:$IMAGE_TAG > imageDetail.json

artifacts:
  files:
    - imageDetail.json
    - appspec.yaml        # For CodeDeploy
    - taskdef.json        # For ECS deployment

cache:
  paths:
    - '/root/.m2/**/*'    # Maven cache

reports:
  junit-reports:
    files:
      - 'target/surefire-reports/*.xml'
    file-format: JUNITXML
```

### When to Use CodePipeline

```
USE CODEPIPELINE WHEN:
├── 100% AWS shop (ECS, Lambda, no K8s)
├── Simple pipeline (build → deploy, no complex logic)
├── Need native AWS IAM integration (no OIDC setup)
├── Small team, don't want to manage Jenkins
└── Deploying to CodeDeploy targets (EC2, ECS blue-green)

DON'T USE CODEPIPELINE WHEN:
├── Multi-cloud or hybrid (it's AWS-only)
├── Complex pipeline logic (limited branching/conditionals)
├── Kubernetes deployments (ArgoCD is better)
├── Need shared libraries / template reuse (very limited)
├── Large org with diverse needs (too rigid)
└── Cost sensitivity at scale (CodeBuild minutes add up)
```

---

## 4. The Comparison Table — Decision Framework

| Aspect | Jenkins | GitHub Actions | GitLab CI | CodePipeline |
|--------|---------|---------------|-----------|-------------|
| **Hosting** | Self-managed | SaaS + self-hosted runners | SaaS or self-managed | AWS-managed |
| **Config file** | Jenkinsfile (Groovy) | YAML (.github/workflows/) | YAML (.gitlab-ci.yml) | YAML (buildspec.yml) + Console |
| **Plugin ecosystem** | 1800+ plugins (strength AND weakness) | 20K+ marketplace actions | Built-in features | Limited — AWS services only |
| **Learning curve** | Steep (Groovy, CPS, plugin hell) | Low-Medium | Low-Medium | Low (if you know AWS) |
| **Runners/Agents** | Self-managed (K8s, EC2, SSH) | GitHub-hosted (free tier) + self-hosted | Shared (SaaS) + self-hosted | CodeBuild (managed compute) |
| **Secrets** | Credential store (on controller) | Encrypted secrets (per repo/org/env) | CI/CD variables (masked) | SSM + Secrets Manager (native) |
| **AWS integration** | Plugins, manual config | OIDC (excellent, no keys) | OIDC (good) | Native IAM (best) |
| **K8s integration** | K8s plugin (pods as agents) | ARC (Actions Runner Controller) | K8s executor | Weak (use CodeDeploy/ECS) |
| **Monorepo** | Manual (path-based scripts) | `paths` filter on triggers | `changes` + parent-child | Manual |
| **Branch pipelines** | Multibranch plugin | Native (every workflow file) | Native (rules/only/except) | Limited |
| **Security scanning** | Plugins (Trivy, SonarQube) | Actions (marketplace) + CodeQL | Built-in SAST, DAST, deps | Inspector integration |
| **Caching** | PVC (K8s), workspace | actions/cache (S3-backed) | Distributed cache (key-based) | S3 cache (manual) |
| **Artifacts** | archiveArtifacts (on controller!) | upload/download-artifact | Built-in with expiry | S3 (output artifacts) |
| **Manual approval** | input() step | environments with protection rules | `when: manual` | Manual approval action |
| **GitOps** | Updates manifest repo → ArgoCD | Same pattern | Same pattern | CodeDeploy (push-based) |
| **Cost** | Free (but infra costs) | Free tier (2000min/mo), then $/min | Free tier, then $/user | Pay per build minute |
| **HA** | DIY (hard) | Built-in (SaaS) | Built-in (SaaS) or DIY | Built-in (AWS) |
| **Maturity** | 20+ years | 5+ years | 10+ years | 7+ years |
| **Market trend (2025)** | Declining (legacy) | Dominant (growing) | Strong (enterprise) | Niche (AWS-only) |

### When to Choose What — NovaMart Decision Tree

```
                    "We need a CI/CD platform"
                            │
              ┌─────────────┼─────────────────┐
              ▼             ▼                  ▼
        Greenfield?    Already have        AWS-only, no K8s,
              │        Jenkins?            simple pipelines?
              │             │                  │
              ▼             ▼                  ▼
     ┌────────────┐   Keep Jenkins       CodePipeline
     │ GitHub     │   (migration is        (okay)
     │ Actions    │   expensive)
     │ (default)  │   │
     └────────────┘   │ UNLESS:
                      ├── Jenkins is unmaintained
     ┌────────────┐   ├── Security audit failing
     │ GitLab CI  │   ├── Plugin hell is unsustainable
     │ (if GitLab │   └── Team productivity suffering
     │  is SCM)   │       │
     └────────────┘       ▼
                      Migrate to GitHub Actions
                      (phased, service by service)
```

### Migration Strategy — Jenkins → GitHub Actions

```
Phase 1: Foundation (Week 1-2)
├── Set up OIDC for AWS access
├── Create shared reusable workflows (equivalent to shared library)
├── Set up self-hosted runners on EKS (ARC)
├── Configure branch protection rules
└── Set up Artifactory/ECR authentication

Phase 2: Pilot (Week 3-4)
├── Pick 3 low-risk services
├── Write GitHub Actions workflows (keep Jenkins running in parallel)
├── Compare: build time, reliability, developer experience
├── Document migration guide for teams
└── Get feedback from pilot teams

Phase 3: Rollout (Week 5-12)
├── Team-by-team migration (10-20 services/week)
├── Each team: pair with platform engineer for first migration
├── Maintain Jenkins for services not yet migrated
├── Monitor: build times, failure rates, developer satisfaction
└── Update shared workflows based on feedback

Phase 4: Decommission (Week 13-16)
├── Verify all services migrated
├── Keep Jenkins read-only for build history (30-60 days)
├── Decommission Jenkins infrastructure
├── Update runbooks, documentation, on-call guides
└── Post-migration review: what improved, what regressed
```

---

## 5. Cost Optimization in CI

```
COST LEVERS (applies to all platforms):

1. SPOT INSTANCES FOR RUNNERS
   ├── Jenkins: EC2 plugin with spot fleet
   ├── GitHub Actions: ARC on Karpenter with spot NodePool
   ├── GitLab: Docker Machine autoscale with spot
   └── Savings: 60-80% on compute

2. RIGHT-SIZE AGENTS
   ├── Most builds don't need 8 CPU / 16GB
   ├── Profile actual usage (Prometheus metrics on agent pods)
   ├── Different agent sizes for different workloads
   │   ├── "lightweight" (1 CPU, 2GB) — manifest updates, deploys
   │   ├── "standard" (2 CPU, 4GB) — most builds
   │   └── "heavy" (4 CPU, 8GB) — Java compile, Docker build
   └── Savings: 40-60% on oversized agents

3. CACHING
   ├── Maven: PVC or actions/cache → saves 3-5 min/build
   ├── Docker layers: BuildKit cache, Kaniko registry cache
   ├── npm: node_modules cache
   ├── Go: GOPATH/pkg/mod cache
   └── Savings: 30-50% on build time (= fewer agent minutes)

4. SKIP UNNECESSARY WORK
   ├── Path-based triggers (only build what changed)
   ├── Skip CI on docs-only changes ([skip ci] or paths-ignore)
   ├── Conditional stages (don't run integration tests on feature branches)
   ├── Cancel stale builds (concurrency: cancel-in-progress)
   └── Savings: 20-40% on total pipeline runs

5. BUILD TIME WINDOWS
   ├── Scale down runners during off-hours
   ├── Schedule heavy jobs (full integration tests) for off-peak
   └── Savings: 10-20% on compute (especially if paying for reserved)

NovaMart CI Cost Example (200 services):
  Before optimization: ~400 builds/day × 20min avg × $0.005/min = $40K/month
  After optimization:  ~300 builds/day × 8min avg × $0.002/min (spot) = $4.8K/month
  Savings: ~88%
```


## Quick Reference Card

```
GITHUB ACTIONS:
  .github/workflows/*.yml — multiple workflows per repo
  on: push/pull_request/workflow_dispatch/schedule
  jobs → steps → uses (action) or run (shell)
  OIDC → AWS: role-to-assume, no access keys ever
  Reusable workflows: workflow_call (like shared libraries)
  Cache: actions/cache or setup-java cache:'maven'
  Matrix: strategy.matrix (parallel combinations)
  ARC: self-hosted runners on K8s with autoscaling
  Concurrency: cancel-in-progress: true (kill stale)

GITLAB CI:
  .gitlab-ci.yml — single file in repo root
  stages define ORDER, jobs in same stage run PARALLEL
  rules: replaces only/except (use rules, not only/except)
  cache: key per branch, persistent across pipelines
  artifacts: with expire_in (auto-cleanup!)
  Built-in: container registry, SAST, DAST, feature flags
  Parent-child pipelines for monorepos
  needs: keyword for DAG (skip stage ordering)

AWS CODEPIPELINE:
  buildspec.yml for CodeBuild
  Native SSM + Secrets Manager integration
  Best for: AWS-only, ECS/Lambda, simple pipelines
  Worst for: K8s, multi-cloud, complex logic

DECISION FRAMEWORK:
  Greenfield → GitHub Actions (default)
  Already GitLab → GitLab CI
  AWS-only simple → CodePipeline
  Legacy Jenkins → Migrate if: unmaintained, security risk, productivity drain

CI COST OPTIMIZATION:
  Spot instances (60-80% savings)
  Right-size agents (40-60% savings)
  Caching (30-50% time savings)
  Path-based triggers (20-40% fewer runs)
  Cancel stale builds (free)
```


---

## Phase 3 Retention Questions (Final)

**Q1.** NovaMart's CTO asks you to evaluate migrating from Jenkins to GitHub Actions. The CFO wants a cost comparison. The security team wants to know about credential management. The engineering leads want to know if their custom shared library functions will work. Write the evaluation document covering: (a) technical feasibility, (b) cost analysis (current Jenkins infra vs GHA), (c) security posture comparison, (d) migration risk, and (e) your recommendation with justification.

**Q2.** You need to set up GitHub Actions OIDC for 15 different NovaMart repositories, each deploying to different AWS accounts (dev, staging, prod). Design the IAM trust policy strategy. How do you prevent repo A from accessing repo B's production account? How do you handle the case where a single service needs access to both the dev AND prod accounts in the same workflow?

**Q3.** A GitLab CI pipeline is taking 25 minutes. Looking at the `.gitlab-ci.yml`, you see: 5 sequential stages, no caching, `image: maven:latest` (downloads fresh every time), DinD for Docker builds, and all jobs run on shared runners. Optimize this pipeline — rewrite the key sections of the `.gitlab-ci.yml` with your improvements and estimate the new total time.

**Q4.** Compare how you would implement the following NovaMart requirement across all four platforms (Jenkins, GitHub Actions, GitLab CI, CodePipeline): "On every PR to main, run unit tests, SonarQube analysis, and Trivy scan in parallel. Block the PR merge if any check fails. Post results as PR comments." Show the relevant config snippet for each platform and note the limitations of each approach.
---

### Q1: Jenkins $\rightarrow$ GitHub Actions (GHA) Evaluation

**To: CTO, CFO, CISO, Engineering Leads**
**Subject: Evaluation of CI/CD Migration (Jenkins $\rightarrow$ GitHub Actions)**

#### (a) Technical Feasibility: Moderate/Low
The move is feasible but requires a **complete rewrite** of the pipeline logic.
*   **The Blockers:** NovaMart's custom shared libraries are written in Groovy. GHA uses YAML for orchestration and JavaScript/TypeScript/Docker for custom "Actions." There is no automated converter for Groovy $\rightarrow$ GHA.
*   **The Path:** We must map our `standardJavaPipeline` Groovy functions to a set of **Composite Actions** or **Reusable Workflows**.

#### (b) Cost Analysis
| Component | Jenkins (Current) | GitHub Actions (Proposed) |
| :--- | :--- | :--- |
| **Infrastructure** | AWS EC2s for Controller + Agents (~$1.2k/mo) | Managed (SaaS) - Pay per minute |
| **Maintenance** | ~15 SRE hours/week (Patching, Plugin Hell) | $\approx 0$ (Managed by GitHub) |
| **Licensing** | Open Source ($0) | Enterprise License (per seat) |
| **TCO** | **High OpEx** (Engineering time is the cost) | **Predictable OpEx** (License + Usage) |

#### (c) Security Posture
*   **Jenkins:** Current state is "Secret-heavy." Credentials stored in Jenkins' internal XML/Database; requires heavy plugin management to rotate.
*   **GHA:** **OIDC-native.** We can eliminate long-lived AWS Access Keys entirely by using GitHub's OIDC provider to assume IAM roles dynamically.

#### (d) Migration Risk
*   **Risk:** "Pipeline Parity." If a complex deployment logic is missed in the rewrite, we risk production outages.
*   **Mitigation:** Dual-run strategy. Run GHA in "shadow mode" (non-blocking) alongside Jenkins for 2 weeks to verify the outcome of every stage.

#### (e) Recommendation: MIGRATE
**Justification:** The cost of maintaining a legacy Jenkins controller for 200 services is an "Innovation Tax." By moving to GHA, we shift the maintenance burden to GitHub, enable OIDC security, and integrate the CI directly into the PR flow, increasing developer velocity.

---

### Q2: OIDC Trust Policy Strategy

To prevent Repo A from accessing Account B, we must leverage the **`sub` (subject)** claim in the OIDC token.

**The Trust Policy (applied to the IAM Role in AWS):**
Instead of trusting the entire GitHub Organization, the role only trusts a specific repository and branch.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:NovaMart/payment-service:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Handling Dev vs. Prod for a Single Service:**
The workflow handles this by attempting to assume two different roles based on the environment context.

**The GitHub Action Workflow snippet:**
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Assume Role
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ env.AWS_ROLE_ARN }} # Set via env: DEV_ROLE or PROD_ROLE
          aws-region: us-east-1
```
**The Logic:**
1.  If `github.ref == 'refs/heads/develop'`, the workflow sets `AWS_ROLE_ARN` to the **Dev Account Role**.
2.  If `github.ref == 'refs/heads/main'`, it sets it to the **Prod Account Role**.
3.  The IAM Role in the Prod account has a trust policy that *only* allows the `main` branch subject. This ensures a developer cannot push to a "feature" branch and deploy it to production.

---

### Q3: GitLab CI Optimization

**Current State:** 25 mins (Sequential, no cache, fresh images).
**Optimized State:** ~8-10 mins.

**The Optimized `.gitlab-ci.yml`:**
```yaml
image: maven:3.8.6-openjdk-17 # Pinning version to avoid "latest" drift

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

cache:
  paths:
    - .m2/repository

stages:
  - build
  - test-and-scan # Combined parallel stage
  - deploy

build_job:
  stage: build
  script:
    - mvn clean compile # Build once, use artifacts for next stages
  artifacts:
    paths:
      - target/

test_sonar_trivy:
  stage: test-and-scan
  parallel:
    matrix:
      - TASK: [unit-tests, sonar, trivy]
  script:
    - if [ "$TASK" == "unit-tests" ]; then mvn test; fi
    - if [ "$TASK" == "sonar" ]; then mvn sonar:sonar; fi
    - if [ "$TASK" == "trivy" ]; then trivy fs . ; fi

docker_build:
  stage: deploy
  image:
    name: gcr.io/kaniko-project/executor:debug # No DinD!
  script:
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile Dockerfile --destination $REGISTRY_IMAGE
```

**Optimization Breakdown:**
1.  **Cache (`.m2/repository`):** Saves ~7m by not re-downloading dependencies.
2.  **Parallel Matrix:** Moves `unit-tests`, `sonar`, and `trivy` from sequential $\rightarrow$ parallel. Saving $\sim (\text{sum} - \max) \approx 8\text{m}$.
3.  **Kaniko instead of DinD:** Removes the need for privileged mode and the overhead of starting a Docker daemon. Saving ~2m.
4.  **Pinned Image:** Prevents unexpected build failures when `maven:latest` updates.

---

### Q4: Cross-Platform PR Guardrails

**The Requirement:** Parallel Tests/Sonar/Trivy $\rightarrow$ Block Merge $\rightarrow$ Post Comments.

#### 1. Jenkins (The "Plugin" Way)
Requires a `parallel` block and the "GitHub Branch Branch Protection" plugin.
```groovy
stage('Checks') {
    parallel {
        stage('Tests') { steps { sh './mvnw test' } }
        stage('Sonar') { steps { sh './mvnw sonar:sonar' } }
        stage('Trivy') { steps { sh 'trivy fs .' } }
    }
}
```
*   **Limitation:** Jenkins doesn't natively "block" the PR. You must configure GitHub Branch Protection to require the "Jenkins" status check to pass.

#### 2. GitHub Actions (The "Native" Way)
Parallelism is the default for `jobs`.
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps: [run tests]
  sonar:
    runs-on: ubuntu-latest
    steps: [run sonar]
  trivy:
    runs-on: ubuntu-latest
    steps: [run trivy]
```
*   **Limitation:** PR comments require a specific Action (e.g., `marocchino/sticky-pull-request-comment`) because the default output is just a log.

#### 3. GitLab CI (The "Integrated" Way)
Uses `stages` and `parallel` jobs.
```yaml
test_stage:
  stage: test
  parallel: 3 # or separate jobs
  script: [ ... ]
  allow_failure: false # Blocks the pipeline
```
*   **Limitation:** GitLab's "Merge Request" blocking is handled via "Pipeline must succeed" setting in Project Settings.

#### 4. AWS CodePipeline (The "Step-Function" Way)
Uses **CodeBuild** projects in a single stage.
*   **Config:** Create one Stage containing 3 CodeBuild projects.
*   **Limitation:** CodePipeline is the most rigid. It has no native "PR Comment" system; you must write a custom Lambda function to call the GitHub API and post the results.

**Summary Table:**

| Platform | Parallelism | Block Merge | PR Comments |
| :--- | :--- | :--- | :--- |
| **Jenkins** | `parallel { }` | Via GitHub API | Plugin-based |
| **GHA** | Job-level (native) | Branch Protection | Action-based |
| **GitLab** | Stage-level | "Pipeline must pass" | Integrated |
| **AWS** | CodeBuild projects | Stage gate | Lambda required |
