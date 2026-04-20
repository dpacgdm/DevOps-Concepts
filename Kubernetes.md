# PHASE 2 — Lesson 3: Kubernetes Architecture & Core Objects

We covered K8s networking in Phase 1. Now we go deeper into the control plane, how scheduling works, and the core objects you'll manage daily.

---

### The Control Plane — What Runs Kubernetes

```
┌─────────────────────── CONTROL PLANE ───────────────────────┐
│                                                             │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐  │
│  │  API Server   │  │   Scheduler   │  │ Controller      │  │
│  │  (kube-apiserver)│ (kube-scheduler)│ │ Manager         │  │
│  │               │  │               │  │ (kube-controller │  │
│  │ THE gatekeeper│  │ WHERE does    │  │  -manager)       │  │
│  │ ALL traffic   │  │ this pod go?  │  │                  │  │
│  │ goes through  │  │               │  │ Reconciliation   │  │
│  │ here. Period. │  │               │  │ loops. Desired   │  │
│  └──────┬───────┘  └───────────────┘  │ vs actual state.  │  │
│         │                              └──────────────────┘  │
│         │                                                    │
│  ┌──────▼───────┐  ┌───────────────────────────────────┐     │
│  │    etcd       │  │  Cloud Controller Manager         │     │
│  │               │  │  (cloud-specific: AWS, GCP, Azure)│     │
│  │ The database. │  │  Manages: LBs, nodes, routes      │     │
│  │ All cluster   │  │  EKS: runs as AWS-managed service │     │
│  │ state lives   │  └───────────────────────────────────┘     │
│  │ here.         │                                           │
│  └──────────────┘                                            │
└──────────────────────────────────────────────────────────────┘

┌─────────────────────── WORKER NODE ─────────────────────────┐
│                                                             │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐  │
│  │   kubelet     │  │  kube-proxy   │  │  Container       │  │
│  │               │  │               │  │  Runtime         │  │
│  │ Agent on each │  │ Network rules │  │  (containerd)    │  │
│  │ node. Talks   │  │ iptables/IPVS │  │                  │  │
│  │ to API server.│  │ for Services  │  │  Actually runs   │  │
│  │ Ensures pods  │  │               │  │  containers      │  │
│  │ are running.  │  │               │  │                  │  │
│  └──────────────┘  └───────────────┘  └──────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Pods    [ container ] [ container ] [ container ]   │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### What Each Component Actually Does

```bash
# API SERVER (kube-apiserver):
# - REST API endpoint for ALL cluster operations
# - ONLY component that talks to etcd directly
# - Authenticates requests (certs, tokens, OIDC)
# - Authorizes requests (RBAC)
# - Validates objects (admission controllers)
# - Serves watch streams (controllers subscribe to changes)
# kubectl → API server → etcd
# Scheduler → API server → etcd
# kubelet → API server → etcd
# EVERYTHING goes through the API server. No exceptions.

# ETCD:
# - Distributed key-value store (Raft consensus)
# - Stores ALL cluster state: pods, services, secrets, configmaps
# - NOT a general-purpose database — don't abuse it
# - Performance sensitive: SSD required, latency < 10ms
# - Backup etcd = backup your entire cluster state
# - EKS: managed by AWS (you don't touch it)
# - Self-managed: 3 or 5 node etcd cluster (odd number for Raft quorum)
#
# Key structure:
# /registry/pods/default/my-pod → pod spec JSON
# /registry/services/default/my-service → service spec JSON
# /registry/secrets/default/my-secret → encrypted secret

# SCHEDULER (kube-scheduler):
# Watches for unscheduled pods (pods with no node assigned)
# Decision process:
#   1. FILTERING: eliminate nodes that can't run the pod
#      - Insufficient CPU/memory
#      - Node taints that pod doesn't tolerate
#      - Node affinity/anti-affinity rules
#      - PV topology constraints
#   2. SCORING: rank remaining nodes
#      - Spread pods across nodes (LeastRequestedPriority)
#      - Pack pods tightly (MostRequestedPriority) — for cost
#      - Prefer nodes with image already cached
#      - Topology spread constraints
#   3. BINDING: assign pod to highest-scoring node
#      - Writes node name to pod spec via API server
#
# The scheduler ONLY decides WHERE. It doesn't start anything.
# kubelet on the chosen node sees the assignment and starts the pod.

# CONTROLLER MANAGER:
# Runs ~30+ control loops, each watching a resource type:
#
# Deployment controller:
#   Watches Deployments → creates/updates ReplicaSets
# ReplicaSet controller:
#   Watches ReplicaSets → creates/deletes Pods to match desired count
# Node controller:
#   Watches node heartbeats → marks nodes NotReady after timeout
# Job controller:
#   Watches Jobs → creates Pods, tracks completion
# Endpoint controller:
#   Watches Services + Pods → creates Endpoint objects
# ServiceAccount controller:
#   Creates default ServiceAccount for new namespaces
#
# Each controller follows the same pattern:
#   1. Watch desired state (spec)
#   2. Observe actual state (status)
#   3. Take action to reconcile (create/delete/update resources)
#   4. Repeat forever
# This is the "declarative model" — you declare WHAT, controllers figure out HOW

# KUBELET:
# Agent on every node. Responsibilities:
#   - Registers node with API server
#   - Watches for pods assigned to this node
#   - Pulls images, creates containers via container runtime
#   - Monitors pod health (liveness/readiness probes)
#   - Reports node status (capacity, conditions) to API server
#   - Mounts volumes, manages secrets/configmaps
#   - Evicts pods if node is under pressure (disk, memory, PIDs)
#
# kubelet does NOT manage pods not created by the API server
# (except static pods defined in /etc/kubernetes/manifests/)
```

---

### How Pod Creation Actually Works — The Full Chain

```
kubectl apply -f pod.yaml

Step 1: kubectl sends POST /api/v1/namespaces/default/pods to API server

Step 2: API server:
  a) Authenticates (is this user who they claim to be?)
  b) Authorizes via RBAC (can this user create pods in default namespace?)
  c) Runs Admission Controllers:
     - Mutating: inject sidecar (Istio), add labels, set defaults
     - Validating: check resource limits exist, enforce policies (OPA)
  d) Persists pod to etcd (status: Pending, no node assigned)

Step 3: Scheduler watches API server (via watch stream):
  - Sees new pod with no nodeName
  - Runs filtering → scoring → selects best node
  - PATCHes pod with nodeName via API server → etcd updated

Step 4: kubelet on selected node watches API server:
  - Sees pod assigned to this node
  - Pulls image (if not cached) via container runtime
  - Creates containers (containerd → runc)
  - Sets up networking (CNI plugin creates veth, assigns IP)
  - Mounts volumes
  - Starts containers
  - Reports pod status back to API server → etcd updated
  - Status: Running

Step 5: kube-proxy on ALL nodes:
  - If pod matches a Service selector
  - Endpoint controller adds pod IP to Endpoints
  - kube-proxy on every node updates iptables/IPVS rules
  - Pod is now reachable via Service ClusterIP

TOTAL TIME: typically 2-10 seconds for a simple pod
```

---

### Pods — The Atomic Unit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
    version: v1
spec:
  # INIT CONTAINERS — run BEFORE main containers, sequentially
  initContainers:
  - name: db-migration
    image: my-app:v1.2.3
    command: ["./migrate", "--up"]
    # Must complete successfully before main containers start
    # Use for: DB migrations, config generation, wait-for-dependency

  containers:
  - name: app
    image: my-app:v1.2.3
    ports:
    - containerPort: 8080
    
    # RESOURCE MANAGEMENT:
    resources:
      requests:
        cpu: 250m        # 0.25 CPU cores — used for SCHEDULING
        memory: 256Mi    # used for scheduling
      limits:
        cpu: 1000m       # 1 CPU core — hard ceiling (throttled beyond this)
        memory: 512Mi    # hard ceiling (OOM killed beyond this)
    
    # requests = what scheduler uses to find a node with enough capacity
    # limits = what cgroups enforce at runtime
    # requests < limits = "burstable" — can use more if available
    # requests = limits = "guaranteed" QoS class (highest priority)
    # no requests/limits = "best effort" QoS class (first to be evicted)

    # PROBES:
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15    # wait before first check
      periodSeconds: 10          # check every 10s
      failureThreshold: 3        # 3 failures → restart container
      # LIVENESS = "is the process alive?"
      # Failure action: RESTART the container
      # Use for: detecting deadlocks, infinite loops, hung processes
      # DON'T check dependencies here (DB, cache) — will cause restart loops

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
      # READINESS = "is the process ready to receive traffic?"
      # Failure action: REMOVE from Service endpoints (no traffic sent)
      # Use for: warming caches, waiting for dependencies, graceful drain
      # Check dependencies here — if DB is down, stop sending traffic

    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
      # STARTUP = "has the process finished starting?"
      # While startup probe is running, liveness/readiness are DISABLED
      # failureThreshold × periodSeconds = 300s = 5 min max startup time
      # Use for: slow-starting apps (Java, large ML models)
      # Without this: liveness probe kills the app before it finishes booting

    # LIFECYCLE HOOKS:
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo started > /tmp/started"]
        # Runs AFTER container starts, but NOT guaranteed before ENTRYPOINT
        # Runs in parallel with the main process
        
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10"]
        # Runs BEFORE container is sent SIGTERM
        # Use for: graceful shutdown, deregister from service discovery
        # The sleep 10 trick: gives kube-proxy time to update iptables
        # and stop routing traffic BEFORE the app shuts down
        # Without this: app receives SIGTERM, starts shutting down,
        # but kube-proxy hasn't updated yet → requests hit a dying pod → 502s

  # TERMINATION SEQUENCE:
  terminationGracePeriodSeconds: 30   # default: 30
  # 1. Pod marked for deletion
  # 2. Endpoints controller removes pod from Service endpoints
  # 3. preStop hook runs (if defined)
  # 4. SIGTERM sent to PID 1 in container
  # 5. Wait up to terminationGracePeriodSeconds
  # 6. SIGKILL sent (unconditional kill)
  # Steps 2 and 3 happen IN PARALLEL — this is why preStop sleep matters

  # VOLUMES:
  volumes:
  - name: config
    configMap:
      name: app-config
  - name: secrets
    secret:
      secretName: app-secrets
  - name: tmp
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
  - name: data
    persistentVolumeClaim:
      claimName: app-data
```

---

### Deployments and ReplicaSets

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  replicas: 3
  revisionHistoryLimit: 5      # keep 5 old ReplicaSets for rollback
  
  selector:
    matchLabels:
      app: api                  # MUST match template labels
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1              # can create 1 extra pod during update (4 total)
      maxUnavailable: 0        # never have fewer than 3 running
      # maxSurge=1 + maxUnavailable=0 = safest, slowest
      # maxSurge=25% + maxUnavailable=25% = default, balanced
      # maxSurge=0 + maxUnavailable=1 = resource-constrained (no extra pod)

  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: my-api:v1.2.3
        # ... probes, resources, etc.
```

### What Happens During a Rolling Update

```
# Current state: 3 pods running v1

# kubectl set image deployment/api api=my-api:v1.3.0
# (or kubectl apply with updated image tag)

# With maxSurge=1, maxUnavailable=0:

# Step 1: Create new ReplicaSet (v1.3.0) with 1 replica
# Total pods: 3 (v1) + 1 (v1.3.0) = 4

# Step 2: Wait for v1.3.0 pod to be Ready (readiness probe passes)

# Step 3: Scale down old ReplicaSet to 2
# Total pods: 2 (v1) + 1 (v1.3.0) = 3

# Step 4: Scale up new ReplicaSet to 2
# Total pods: 2 (v1) + 2 (v1.3.0) = 4

# Step 5: Wait for second v1.3.0 pod to be Ready

# Step 6: Scale down old to 1, scale up new to 3
# ... repeat until all pods are v1.3.0

# Old ReplicaSet kept (scaled to 0) for rollback

# ROLLBACK:
kubectl rollout undo deployment/api
# Scales old ReplicaSet back up, scales new one down
# Instant — no image pull needed (old pods were just scaled to 0)

kubectl rollout undo deployment/api --to-revision=3
# Roll back to specific revision

kubectl rollout history deployment/api
# See all revisions and their change causes

kubectl rollout status deployment/api
# Watch rollout progress in real time
```

### Deployment vs StatefulSet vs DaemonSet vs Job

```
DEPLOYMENT:
  - Stateless workloads
  - Pods are interchangeable (no identity)
  - Pod names: api-deployment-7b8f9c6d4-x2k9w (random suffix)
  - Rolling updates, rollback
  - Use for: APIs, web servers, workers

STATEFULSET:
  - Stateful workloads
  - Pods have STABLE identity: db-0, db-1, db-2
  - Ordered creation: db-0 must be running before db-1 starts
  - Ordered deletion: db-2 deleted before db-1
  - Stable DNS: db-0.db-headless.namespace.svc.cluster.local
  - Each pod gets its own PVC (not shared)
  - Use for: databases, Kafka, ZooKeeper, Redis Cluster, etcd
  
  # REQUIRES headless service:
  apiVersion: v1
  kind: Service
  metadata:
    name: db-headless
  spec:
    clusterIP: None            # headless
    selector:
      app: database
    ports:
    - port: 5432

DAEMONSET:
  - Runs ONE pod on EVERY node (or subset via nodeSelector)
  - New node joins → pod automatically scheduled
  - Node removed → pod removed
  - Use for: log collectors (Fluentd), monitoring agents (node-exporter),
    CNI plugins (Calico), storage daemons (EBS CSI), kube-proxy
  - Updates: rolling or OnDelete

JOB:
  - Run-to-completion workload
  - Pod runs, completes, exits
  - Retries on failure (backoffLimit)
  - parallelism: how many pods run simultaneously
  - completions: how many successful completions needed
  - Use for: database migrations, batch processing, backups

CRONJOB:
  - Job on a schedule (cron syntax)
  - Creates Job objects at scheduled times
  - concurrencyPolicy:
      Allow: multiple jobs can run simultaneously
      Forbid: skip if previous still running
      Replace: kill previous, start new
  - Use for: periodic backups, report generation, cleanup tasks
```

---

### ConfigMaps and Secrets

```yaml
# CONFIGMAP — non-sensitive configuration:
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "db.production.svc.cluster.local"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    features:
      new_checkout: true

# Mount as environment variables:
env:
- name: DATABASE_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: DATABASE_HOST

# Mount as file:
volumeMounts:
- name: config
  mountPath: /etc/app/config.yaml
  subPath: config.yaml          # mount single file, not entire dir
volumes:
- name: config
  configMap:
    name: app-config

# AUTO-RELOAD:
# Environment variables: NOT updated on ConfigMap change (requires pod restart)
# Mounted files: updated automatically (~60-90 seconds)
# BUT: app must watch the file for changes (not all apps do)

# SECRET — sensitive data:
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=     # base64 encoded (NOT encrypted!)
  API_KEY: c2VjcmV0a2V5MTIz

# base64 is NOT encryption. Anyone with cluster access can decode:
echo "cGFzc3dvcmQxMjM=" | base64 -d
# password123

# ENCRYPTING SECRETS AT REST:
# EKS: enable envelope encryption with KMS
# AWS encrypts the etcd data using a CMK
# Without this: secrets stored in etcd in plaintext

# EXTERNAL SECRETS (production approach):
# Don't store secrets in K8s at all
# Use: External Secrets Operator → pulls from:
#   - AWS Secrets Manager
#   - HashiCorp Vault
#   - Azure Key Vault
#   - GCP Secret Manager
# Syncs external secrets into K8s Secret objects automatically
# Rotation handled by the external provider
```

---

### Namespaces — Not Just Organization

```bash
# Namespaces are more than folders. They're:
# 1. RBAC boundary (who can do what WHERE)
# 2. Resource quota boundary (how much a team can consume)
# 3. Network policy boundary (which pods can talk to which)
# 4. Service DNS scope (svc-name.namespace.svc.cluster.local)

# Resource Quotas:
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "20"          # total CPU requests across all pods
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"                 # max 100 pods in this namespace
    services: "20"
    persistentvolumeclaims: "30"

# LimitRange — defaults and constraints per pod/container:
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-alpha
spec:
  limits:
  - type: Container
    default:                    # applied if no limits specified
      cpu: 500m
      memory: 256Mi
    defaultRequest:             # applied if no requests specified
      cpu: 100m
      memory: 128Mi
    max:                        # hard ceiling per container
      cpu: 4
      memory: 8Gi
    min:                        # minimum per container
      cpu: 50m
      memory: 64Mi

# WHY THIS MATTERS:
# Without LimitRange, a developer can deploy a pod with no resource limits
# That pod can consume ALL node resources → starves other pods
# LimitRange ensures every container has at least default limits
# ResourceQuota ensures a namespace can't consume the entire cluster
```

---

### Scheduling — Taints, Tolerations, Affinity

```yaml
# TAINTS — "this node repels pods unless they tolerate the taint"
# Applied to NODES:
kubectl taint nodes node1 dedicated=gpu:NoSchedule
# Only pods that tolerate "dedicated=gpu" can be scheduled here

# Three taint effects:
# NoSchedule: don't schedule new pods (existing pods stay)
# PreferNoSchedule: try to avoid, but allow if necessary
# NoExecute: evict existing pods AND don't schedule new ones

# TOLERATIONS — "this pod can tolerate this taint"
# Applied to PODS:
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

# Common pattern — dedicated node pools:
# Taint GPU nodes: dedicated=gpu:NoSchedule
# ML pods tolerate the taint → scheduled on GPU nodes
# Regular pods don't tolerate → stay on regular nodes

# NODE AFFINITY — "this pod prefers/requires specific nodes"
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
            - us-east-1b
        # Pod MUST run in us-east-1a or us-east-1b
        
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: node.kubernetes.io/instance-type
            operator: In
            values:
            - m5.xlarge
        # PREFER m5.xlarge, but don't fail if unavailable

# POD ANTI-AFFINITY — "spread my replicas across nodes/zones"
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - api
        topologyKey: kubernetes.io/hostname
        # NEVER put two api pods on the same node
        
      # Or use topology spread constraints (more flexible):
  topologySpreadConstraints:
  - maxSkew: 1                    # max difference in pod count across zones
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: api
    # Spread api pods evenly across AZs
    # 3 pods, 3 AZs → 1 per AZ
    # 4 pods, 3 AZs → 2-1-1 (skew=1 allowed)
```

---

### RBAC — Who Can Do What

```yaml
# RBAC has 4 objects:
# Role/ClusterRole: WHAT actions are allowed
# RoleBinding/ClusterRoleBinding: WHO gets those permissions

# Role — namespace-scoped:
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]           # "" = core API group (pods, services, etc.)
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]    # subresource
  verbs: ["get"]

# ClusterRole — cluster-wide:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
# Nodes are cluster-scoped — can't be in a namespace Role

# RoleBinding — binds Role to users/groups/service accounts:
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: User
  name: jane@company.com
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: ci-pipeline
  namespace: ci
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

# TEST RBAC:
kubectl auth can-i create pods --namespace production --as jane@company.com
# yes / no

kubectl auth can-i '*' '*' --as system:serviceaccount:ci:ci-pipeline
# Check if SA has full access (should be no)

# PRINCIPLE OF LEAST PRIVILEGE:
# Don't give cluster-admin to CI pipelines
# Don't give * verbs unless absolutely necessary
# Audit with: kubectl auth can-i --list --as <user>
```

---

### Production Scenarios

#### Scenario 1: Pod Stuck in Pending

```bash
# kubectl get pods
# NAME         READY   STATUS    AGE
# api-xyz      0/1     Pending   15m

kubectl describe pod api-xyz
# Events:
# "0/10 nodes are available: 4 Insufficient cpu, 
#  3 node(s) had taint {dedicated=gpu:NoSchedule}, 
#  3 node(s) didn't match Pod's node affinity/selector"

# Translation:
# 10 nodes total
# 4 nodes: not enough CPU for the pod's requests
# 3 nodes: GPU-tainted, pod doesn't tolerate
# 3 nodes: wrong AZ (node affinity constraint)
# Result: 0 eligible nodes

# Debug checklist:
# 1. Check resource requests vs node capacity:
kubectl describe nodes | grep -A 5 "Allocated resources"
# 2. Check taints:
kubectl describe nodes | grep Taints
# 3. Check affinity rules in pod spec
# 4. Check ResourceQuota:
kubectl describe quota -n production
# Maybe namespace has hit its pod or CPU quota

# Fix depends on root cause:
# - Scale up node group (cluster autoscaler should handle this)
# - Reduce resource requests
# - Add tolerations
# - Relax affinity constraints
# - Increase ResourceQuota
```

#### Scenario 2: Deployment Rollout Stuck

```bash
kubectl rollout status deployment/api
# Waiting for deployment "api" rollout to finish: 
# 1 old replicas are pending termination...

# Why it sticks:
# maxUnavailable=0 means old pod can't terminate until new pod is Ready
# But new pod is never Ready because:

kubectl get pods
# api-new-xxx   0/1   CrashLoopBackOff   5

kubectl logs api-new-xxx
# Error: missing DATABASE_URL environment variable

# New image requires a new env var that wasn't added
# New pod crashes → never becomes Ready
# Old pod can't be terminated (maxUnavailable=0)
# Rollout is deadlocked

# Fix:
kubectl rollout undo deployment/api
# Rolls back to previous ReplicaSet immediately
# Then fix the ConfigMap/Secret, redeploy properly

# PREVENTION:
# progressDeadlineSeconds (default: 600 = 10 min)
# If rollout hasn't progressed in 10 minutes:
# Deployment condition: Progressing=False, reason=ProgressDeadlineExceeded
# Alert on this in monitoring
```

#### Scenario 3: Pods Evicted — Node Under Pressure

```bash
kubectl get pods
# api-abc   0/1   Evicted   0s

kubectl describe pod api-abc
# Status: Failed
# Reason: Evicted
# Message: The node was low on resource: memory.

# kubelet eviction order (by QoS class):
# 1. BestEffort (no requests/limits) — evicted FIRST
# 2. Burstable (requests < limits) — evicted SECOND
# 3. Guaranteed (requests = limits) — evicted LAST

# If your production pods are Burstable and a BestEffort
# monitoring agent isn't enough to free memory,
# YOUR production pods get evicted

# Fix:
# 1. Set requests = limits for critical pods (Guaranteed QoS)
# 2. Use PodDisruptionBudgets to limit concurrent evictions
# 3. Set proper resource requests so scheduler doesn't overcommit
# 4. Monitor node memory with alerts at 80% utilization
# 5. Use LimitRange to prevent pods with no limits
```

#### Scenario 4: 502 Errors During Deployment

```bash
# Symptoms:
# During rolling update, 1-2% of requests get 502
# App logs show graceful shutdown completing normally

# Root cause: RACE CONDITION between:
# - Endpoints controller removing pod from Service (async)
# - kubelet sending SIGTERM to pod (immediate)
# These happen IN PARALLEL
# Window: kube-proxy hasn't updated iptables yet, but pod is shutting down
# Traffic still routes to dying pod → 502

# Fix:
lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]
# preStop runs BEFORE SIGTERM
# During those 10 seconds:
# - Endpoints controller removes pod from Service
# - kube-proxy updates iptables on all nodes
# - After 10s, SIGTERM sent to already-deregistered pod
# No traffic hits the dying pod → no 502s

# Also ensure:
terminationGracePeriodSeconds: 45
# Must be > preStop sleep + app shutdown time
# 10s preStop + 30s app shutdown = 40s needed, 45s gives buffer
```
---

# 📋 QUICK REFERENCE — K8s Architecture & Core Objects

```
CONTROL PLANE:
  API Server: all traffic, auth, admission, etcd access
  etcd: all state, Raft consensus, backup = cluster backup
  Scheduler: filter → score → bind (WHERE not HOW)
  Controller Manager: 30+ reconciliation loops (desired vs actual)
  kubelet: node agent, runs pods, reports health
  kube-proxy: iptables/IPVS rules for Service routing

```
POD CREATION CHAIN:
  kubectl → API server (auth→RBAC→admission→etcd)
  → scheduler (filter→score→bind) → kubelet (pull→create→network→start)
  → kube-proxy (update endpoints/iptables on ALL nodes)

PROBES:
  Liveness:  is it alive?     Fail → restart container
  Readiness: is it ready?     Fail → remove from Service endpoints
  Startup:   has it started?  Disables liveness/readiness until pass
  DON'T check dependencies in liveness (restart loops)
  DO check dependencies in readiness (stop traffic)

QoS CLASSES:
  Guaranteed: requests = limits          (last to evict)
  Burstable:  requests < limits          (middle)
  BestEffort: no requests or limits      (first to evict)

RESOURCE MANAGEMENT:
  requests: scheduling decision (does node have capacity?)
  limits: runtime enforcement (cgroup hard ceiling)
  CPU over limit → throttled
  Memory over limit → OOM killed

TERMINATION SEQUENCE:
  1. Pod marked for deletion
  2. Endpoints controller removes from Service (async)
  3. preStop hook runs (parallel with step 2)
  4. SIGTERM sent to PID 1
  5. Wait terminationGracePeriodSeconds
  6. SIGKILL
  Steps 2+3 are parallel → preStop sleep(10) prevents 502s

WORKLOAD TYPES:
  Deployment:  stateless, rolling updates, rollback
  StatefulSet: stable identity (pod-0, pod-1), ordered, per-pod PVC
  DaemonSet:   one pod per node (logging, monitoring, CNI)
  Job:         run to completion, retries, parallelism
  CronJob:     Job on schedule, concurrencyPolicy

CONFIGMAP vs SECRET:
  ConfigMap: non-sensitive, plaintext
  Secret: base64 (NOT encrypted by default), enable KMS envelope encryption
  Env vars: NOT updated on change (requires restart)
  Mounted files: updated in ~60-90s (app must watch)
  Production: External Secrets Operator → Vault/AWS Secrets Manager

SCHEDULING:
  Taints (on nodes): repel pods unless tolerated
    NoSchedule / PreferNoSchedule / NoExecute
  Tolerations (on pods): allow scheduling on tainted nodes
  Node affinity: required vs preferred node selection
  Pod anti-affinity: spread replicas across nodes/zones
  TopologySpreadConstraints: maxSkew control across topology domains

RBAC:
  Role/ClusterRole: WHAT (verbs on resources)
  RoleBinding/ClusterRoleBinding: WHO gets WHAT
  Test: kubectl auth can-i <verb> <resource> --as <user>
  Principle of least privilege. Always.

NAMESPACES:
  ResourceQuota: total resource cap per namespace
  LimitRange: default/min/max per container
  Without LimitRange: pods can deploy with no limits → resource starvation

ROLLOUT:
  maxSurge + maxUnavailable control speed vs safety
  maxSurge=1, maxUnavailable=0: safest (never below desired count)
  Rollback: kubectl rollout undo deployment/<name>
  Stuck rollout: new pod CrashLoopBackOff + maxUnavailable=0 = deadlock
  progressDeadlineSeconds: alert when rollout stalls

DEBUGGING:
  Pending: describe pod → check resources, taints, affinity, quota
  CrashLoopBackOff: logs, describe → check image, env vars, probes
  Evicted: node memory pressure → check QoS class, set requests=limits
  502 during deploy: add preStop sleep(10), increase terminationGracePeriod
```

---

# 📝 Retention Questions — Lesson 3

**Q1:** A pod is stuck in `Pending` for 20 minutes. `kubectl describe pod` shows: `0/15 nodes are available: 8 Insufficient cpu, 4 had taint {team=data:NoSchedule}, 3 didn't match pod affinity rules.` The cluster autoscaler is enabled. Explain why the autoscaler hasn't added a node, and walk through your full debugging approach.

**Q2:** During a rolling deployment, your monitoring shows 502 errors affecting ~2% of requests. The application logs show clean graceful shutdowns. The pod spec has no `preStop` hook and `terminationGracePeriodSeconds: 30`. Explain the exact race condition causing this and the precise fix with correct values.

**Q3:** Your team deploys a pod with `requests.memory: 128Mi` and no memory limit. Another team's pod in the same namespace has `requests.memory: 512Mi` and `limits.memory: 512Mi`. The node runs out of memory. Which pod gets evicted first and why?

**Q4:** A developer creates a ConfigMap and mounts it as environment variables in their pod. They update the ConfigMap value and wait 10 minutes, but the pod still sees the old value. They ask you why. Explain the behavior and give them two options to pick up the new value.

**Go.** 🎯

### Q1: The Cluster Autoscaler (CA) Paradox

**Why the autoscaler hasn't added a node:**
The Cluster Autoscaler only triggers a scale-up if it believes that adding a new node from the Auto Scaling Group (ASG) **will actually allow the pod to be scheduled**.

In this scenario, the pod is unschedulable on 15 nodes for three different reasons. The CA analyzes the pod's constraints:
1.  **Insufficient CPU:** This is a "scale-up" trigger. If this were the *only* reason, CA would add a node.
2.  **Taints (`team=data:NoSchedule`):** If the pod does not have a corresponding **toleration** for this taint, it can never be scheduled on these nodes.
3.  **Affinity Rules:** If the pod has `nodeAffinity` or `podAffinity` that requires specific labels (e.g., `disk=ssd`), and the nodes in the ASG do not have those labels, the pod can never be scheduled there.

**The Verdict:** The CA has looked at the ASG configuration and realized that even if it adds a new node, that new node will either have the same taints or lack the required affinity labels. Therefore, adding a node would not solve the problem, and CA decides not to waste money/resources on a node that the pod still couldn't use.

**Debugging Approach:**
1.  **Inspect Pod Spec:** Run `kubectl get pod <pod> -o yaml` and look specifically at `tolerations` and `affinity`.
2.  **Inspect Node Templates:** Check the ASG/Node Group configuration. Do the new nodes have the labels required by the pod's affinity? Do they have the same taints that the pod doesn't tolerate?
3.  **Analyze CA Logs:** Check the Cluster Autoscaler logs. Look for messages like `"pod is unschedulable... but adding a node wouldn't help because..."` This provides the exact reason why the scale-up was skipped.

---

### Q2: The Endpoint Propagation Race

**The Exact Race Condition:**
This is a synchronization failure between the **Control Plane** and the **Data Plane**.

When a pod is terminated during a rolling update, two actions happen simultaneously:
1.  **The Signal:** The Kubelet sends a `SIGTERM` to the pod. Since the app is designed for "clean graceful shutdowns," it stops accepting new connections and exits almost immediately.
2.  **The Update:** The Endpoint Controller removes the pod's IP from the Service's Endpoint list. This change must then propagate to every single node in the cluster via `kube-proxy` (updating `iptables` or `ipvs` rules).

**The Gap:** The `SIGTERM` reaches the pod in milliseconds. However, the `iptables` update on the worker nodes can take several seconds to propagate. For a brief window, some nodes still believe the pod is healthy and route traffic to it. Because the pod has already shut down its listener, the request fails with a **502 Bad Gateway**.

**The Precise Fix:**
You must force the application to stay alive for a few seconds *after* it has been marked for deletion, allowing the network layer to "catch up."

1.  **Add a `preStop` hook:**
    ```yaml
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15"]
    ```
2.  **Adjust `terminationGracePeriodSeconds`:** Ensure this is higher than your sleep value (e.g., `30s`).

**Result:** The pod receives the `preStop` signal, sleeps for 15 seconds (during which time the endpoints are updated across the cluster), and only *then* receives the `SIGTERM` to shut down.

---

### Q3: QoS Classes and Eviction Priority

**Which pod gets evicted first?**
**Pod A** (requests: 128Mi, no limit) will be evicted first.

**The Reasoning (QoS Classes):**
Kubernetes assigns a Quality of Service (QoS) class to every pod based on its requests and limits:
1.  **Guaranteed:** `requests` == `limits` for all containers. (Pod B)
2.  **Burstable:** `requests` are specified, but they are less than `limits` (or limits are not specified). (Pod A)
3.  **BestEffort:** No `requests` or `limits` specified.

When a node runs out of memory (OOM), the `kubelet` must evict pods to save the node. It does this based on a strict priority hierarchy to protect the most "predictable" workloads:
**BestEffort $\rightarrow$ Burstable $\rightarrow$ Guaranteed.**

Since Pod B is **Guaranteed**, it is the last to be killed. Pod A is **Burstable**, making it a prime target for eviction as soon as the node feels pressure, especially if Pod A is using more memory than its requested 128Mi.

---

### Q4: The Static Nature of Environment Variables

**Why the pod still sees the old value:**
Environment variables are injected into a process **at startup**. 

When a ConfigMap is mounted as an environment variable (`envFrom` or `valueFrom`), the Kubelet reads the value of the ConfigMap *only* when the container is being created. Once the process (the app) starts, those variables are locked into the process's environment memory. 

Updating the ConfigMap in the Kubernetes API does not trigger an update to the environment variables of a running process. The "10-minute wait" is irrelevant because there is no mechanism in Linux to "hot-swap" the environment variables of a running PID.

**Two Options to pick up the new value:**

1.  **The "Hard Reset" (Restart):**
    Perform a rollout restart of the deployment:
    `kubectl rollout restart deployment <deployment-name>`
    This kills the pods and starts new ones, forcing the Kubelet to inject the updated ConfigMap values during startup.

2.  **The "Dynamic" Approach (Volume Mount):**
    Instead of environment variables, mount the ConfigMap as a **volume**.
    *   **How it works:** When you mount a ConfigMap as a volume, the Kubelet creates a file in the pod. When the ConfigMap is updated in the API, the Kubelet eventually updates the file on the disk (usually within a minute).
    *   **Requirement:** The application must be written to "watch" the file for changes or re-read the file periodically. This allows the app to update its configuration without a restart.


# PHASE 2 — Lesson 3B: Kubernetes Production Deep Dive (The Missing Half)

---

### Storage — The Entire Layer I Skipped

```
STORAGE HIERARCHY:

  PersistentVolume (PV)
    ↑ bound to
  PersistentVolumeClaim (PVC)
    ↑ used by
  Pod (volumeMounts)
    
  StorageClass → defines HOW PVs are dynamically provisioned
  CSI Driver → interfaces between K8s and actual storage backends
```

```yaml
# STORAGECLASS — defines provisioning behavior:
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com         # CSI driver
parameters:
  type: gp3                           # EBS volume type
  iops: "5000"
  throughput: "250"                   # MB/s
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:..."
reclaimPolicy: Retain                 # what happens when PVC is deleted
  # Retain: PV and data preserved (manual cleanup needed)
  # Delete: PV and underlying storage deleted (DEFAULT — dangerous for DBs)
volumeBindingMode: WaitForFirstConsumer
  # WaitForFirstConsumer: don't provision until pod is scheduled
  #   WHY: EBS volumes are AZ-specific
  #   If PV created in us-east-1a but pod scheduled in us-east-1b → stuck
  #   WaitForFirstConsumer ensures PV created in same AZ as pod
  # Immediate: provision as soon as PVC is created
  #   Use only when storage is not topology-constrained
allowVolumeExpansion: true            # allow PVC resize without recreating

---
# PERSISTENTVOLUMECLAIM:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce          # RWO: mounted by ONE node
  # ReadWriteMany (RWX): mounted by MANY nodes (EFS, NFS — NOT EBS)
  # ReadOnlyMany (ROX): read-only by many nodes
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi

---
# POD using PVC:
spec:
  containers:
  - name: db
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: db-data
```

### CSI Drivers — What Actually Provides Storage

```bash
# CSI (Container Storage Interface) is the standard API
# between Kubernetes and storage providers

# AWS EBS CSI Driver:
# - Block storage (RWO only — one node at a time)
# - AZ-specific (can't move between AZs without snapshot)
# - gp3: 3000 IOPS baseline, up to 16000
# - io2: up to 64000 IOPS (databases)
# - Snapshot support (VolumeSnapshot CRD)
# Install: EKS add-on or Helm chart
# REQUIRES: IRSA (IAM Role for Service Account) with EBS permissions

# AWS EFS CSI Driver:
# - NFS-based (RWX — multiple pods across multiple nodes)
# - Regional (works across AZs)
# - Elastic (auto-grows, no pre-provisioning)
# - Higher latency than EBS
# - Use for: shared config, CMS uploads, ML training data
# - Access points for per-pod isolation

# AWS FSx CSI Driver:
# - High-performance filesystem (Lustre, NetApp ONTAP)
# - Use for: HPC, ML training at scale

# VOLUME SNAPSHOTS:
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-backup-2024-01-15
spec:
  volumeSnapshotClassName: ebs-snapclass
  source:
    persistentVolumeClaimName: db-data
# Creates EBS snapshot → can restore to new PVC
# Use for: backup before migrations, disaster recovery
```

### StatefulSet + Storage — The Full Picture

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless    # required for DNS
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  
  # VOLUMECLAIMTEMPLATES — each pod gets its OWN PVC:
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi

# Result:
# postgres-0 → PVC: data-postgres-0 → PV: pv-abc123 (100Gi EBS in us-east-1a)
# postgres-1 → PVC: data-postgres-1 → PV: pv-def456 (100Gi EBS in us-east-1b)
# postgres-2 → PVC: data-postgres-2 → PV: pv-ghi789 (100Gi EBS in us-east-1c)
#
# If postgres-1 pod dies and restarts:
# It reattaches to data-postgres-1 → same data, same identity
# This is WHY StatefulSets exist
#
# DANGER: if you delete a StatefulSet with --cascade=foreground
# PVCs are NOT deleted (by design — data protection)
# You must manually delete PVCs if you want to clean up
```

---

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60     # wait 60s before scaling up again
      policies:
      - type: Pods
        value: 4                         # add max 4 pods per 60s
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300    # wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10                        # remove max 10% of pods per 60s
        periodSeconds: 60
      # Scale down slowly to prevent thrashing
      # Aggressive scale-down + traffic spike = not enough pods = outage
  
  metrics:
  # CPU-based (requires metrics-server):
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70           # target 70% CPU utilization
  
  # Memory-based:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom metrics (from Prometheus via prometheus-adapter):
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 1000               # 1000 RPS per pod
  
  # External metrics (SQS queue depth):
  - type: External
    external:
      metric:
        name: sqs_queue_length
        selector:
          matchLabels:
            queue: order-processing
      target:
        type: AverageValue
        averageValue: 20                 # scale when > 20 messages per pod

# HOW IT WORKS:
# 1. HPA controller queries metrics every 15s (default)
# 2. Calculates: desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))
# 3. Example: 5 pods at 90% CPU, target 70%
#    desiredReplicas = ceil(5 × (90/70)) = ceil(6.43) = 7
# 4. HPA updates Deployment replicas from 5 to 7
# 5. Deployment controller creates 2 new pods

# REQUIREMENTS:
# metrics-server must be installed (EKS add-on)
# Pods MUST have resource requests defined
# Without requests, HPA can't calculate utilization percentage
# This is why LimitRange with defaultRequest is critical

# GOTCHA:
# HPA and manual replica count conflict
# If you set replicas: 3 in Deployment AND have HPA
# Every kubectl apply resets replicas to 3
# Fix: remove replicas field from Deployment when using HPA
# Or use kubectl apply --server-side with field management
```

### Vertical Pod Autoscaler (VPA)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Off"          # RECOMMENDATION ONLY — don't auto-apply
    # Off: just recommend (safest, use this first)
    # Auto: evict and recreate pods with new resources (disruptive!)
    # Initial: apply only to new pods (doesn't touch running ones)
  resourcePolicy:
    containerPolicies:
    - containerName: api
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources: ["cpu", "memory"]

# Check recommendations:
kubectl describe vpa api-vpa
# Recommendation:
#   Container: api
#     Lower Bound:  cpu: 250m,  memory: 256Mi
#     Target:       cpu: 500m,  memory: 512Mi    ← use this
#     Upper Bound:  cpu: 2,     memory: 2Gi
#     Uncapped:     cpu: 3.5,   memory: 4Gi

# CRITICAL RULE:
# DO NOT use HPA (CPU-based) and VPA together on the same Deployment
# HPA scales horizontally based on CPU %
# VPA changes CPU requests
# Changing requests changes the utilization percentage
# They fight each other in a feedback loop
#
# Exception: HPA on custom metrics (RPS, queue depth) + VPA on resources = OK
# Because HPA isn't using CPU utilization
```

### Cluster Autoscaler vs Karpenter

```bash
# CLUSTER AUTOSCALER (traditional):
# - Watches for pods that can't be scheduled (Pending)
# - Adds nodes to existing Auto Scaling Groups (ASGs)
# - Watches for underutilized nodes → drains and removes them
# - Configuration:
#   --scale-down-utilization-threshold=0.5   (50% util → consider removing)
#   --scale-down-delay-after-add=10m         (don't remove recently added nodes)
#   --skip-nodes-with-local-storage=true     (don't evict pods with emptyDir)
#   --skip-nodes-with-system-pods=true
#   --balance-similar-node-groups=true       (balance across AZs)
#
# LIMITATIONS:
# - Tied to ASG/node group (pre-defined instance types)
# - Slow: Pending pod → CA evaluates → ASG launches → node joins → pod scheduled
#   Total: 3-7 minutes
# - Can't mix instance types dynamically

# KARPENTER (next generation — AWS native):
# - NOT tied to ASGs
# - Watches for unschedulable pods directly
# - Launches RIGHT-SIZED instances from a wide pool
# - Provisions nodes in ~60 seconds (vs 3-7 min for CA)
# - Consolidation: automatically replaces underutilized nodes
# - Drift detection: replaces nodes with outdated AMIs
# - Spot instance support with automatic fallback to on-demand
# - THE standard for EKS autoscaling now

apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]     # prefer spot, fallback on-demand
      - key: karpenter.k8s.aws/instance-category
        operator: In
        values: ["m", "c", "r"]           # m5, c5, r5 families
      - key: karpenter.k8s.aws/instance-size
        operator: In
        values: ["large", "xlarge", "2xlarge"]
      nodeClassRef:
        name: default
  limits:
    cpu: "1000"                           # max 1000 vCPUs total
    memory: 2000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized  # auto-consolidate
    expireAfter: 720h                       # recycle nodes every 30 days
    # Forces AMI updates — prevents "snowflake" nodes running for months

---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-cluster
  instanceProfile: KarpenterNodeInstanceProfile
  blockDeviceMappings:
  - deviceName: /dev/xvda
    ebs:
      volumeSize: 100Gi
      volumeType: gp3
      encrypted: true
```

---

### PodDisruptionBudgets (PDBs)

```yaml
# PDBs protect against VOLUNTARY disruptions:
# - Node drain (kubectl drain)
# - Cluster autoscaler removing a node
# - Karpenter consolidation
# - Node upgrades (EKS managed node group updates)
# 
# PDBs do NOT protect against INVOLUNTARY disruptions:
# - Node crash, hardware failure
# - OOM kill
# - Kernel panic

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  # Option 1: minimum available
  minAvailable: 2              # always keep at least 2 pods running
  # OR
  # Option 2: maximum unavailable
  maxUnavailable: 1            # at most 1 pod can be down at a time
  
  selector:
    matchLabels:
      app: api

# HOW IT WORKS:
# kubectl drain node-1:
#   kubelet asks API server: "can I evict api-abc on this node?"
#   API server checks PDB: "api has 3 pods, minAvailable=2"
#   If evicting api-abc leaves 2 pods → allowed
#   If evicting api-abc leaves 1 pod → BLOCKED
#   drain command waits until another pod is healthy elsewhere
#
# Without PDB:
#   Node drain evicts ALL pods at once
#   If all 3 replicas on same node → 0 available → outage

# GOTCHA:
# PDB with minAvailable = replicas count (e.g., minAvailable: 3, replicas: 3)
# → NOTHING can be evicted → node drain hangs forever
# → Cluster autoscaler can't remove nodes
# → EKS node group updates stuck
# Always: minAvailable < replicas OR maxUnavailable >= 1

# COMMON PATTERN:
# 3 replicas → minAvailable: 2 (or maxUnavailable: 1)
# 5 replicas → maxUnavailable: "20%" (1 pod max)
# 1 replica → no PDB (can't protect a single pod from drain)
#             better: increase to 2+ replicas
```

---

### PriorityClasses and Preemption

```yaml
# PriorityClasses determine which pods survive when resources are scarce

apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 1000000              # higher = more important
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Critical production services"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard
value: 100000
globalDefault: true         # default for pods without explicit priority

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 10000
preemptionPolicy: Never     # can be preempted but won't preempt others
description: "Batch jobs, can be evicted"

# Usage in pod spec:
spec:
  priorityClassName: critical

# HOW PREEMPTION WORKS:
# 1. Critical pod can't be scheduled (no resources)
# 2. Scheduler looks for lower-priority pods to evict
# 3. Evicts enough lower-priority pods to make room
# 4. Critical pod gets scheduled
#
# ORDER OF EVICTION (combined with QoS):
# First: BestEffort + lowest priority
# Then: Burstable + lowest priority
# Last: Guaranteed + highest priority
#
# SYSTEM PRIORITY CLASSES (built-in):
# system-cluster-critical: 2000000000 (coredns, metrics-server)
# system-node-critical: 2000001000 (kube-proxy, CNI, CSI)
# Don't assign these to your workloads
```

---

### Node Maintenance — Drain, Cordon, Uncordon

```bash
# CORDON — mark node as unschedulable (no new pods):
kubectl cordon node-1
# Existing pods keep running
# New pods won't be scheduled here
# Use: before maintenance, investigation

# UNCORDON — mark node as schedulable again:
kubectl uncordon node-1

# DRAIN — cordon + evict all pods:
kubectl drain node-1 \
  --ignore-daemonsets \           # don't try to evict DaemonSet pods
  --delete-emptydir-data \        # allow eviction of pods with emptyDir
  --grace-period=60 \             # time for graceful shutdown
  --timeout=300s                  # give up after 5 min if pods won't evict

# What drain does:
# 1. Cordons the node (marks unschedulable)
# 2. Evicts pods one by one (respecting PDBs)
# 3. Waits for pods to terminate gracefully
# 4. DaemonSet pods are skipped (they MUST run on every node)
# 5. Pods with local storage block drain unless --delete-emptydir-data

# DRAIN STUCK? Common causes:
# - PDB preventing eviction (minAvailable too high)
# - Pod with no controller (standalone pod, no Deployment/RS)
#   Fix: --force (deletes standalone pods, they WON'T be recreated)
# - Finalizer on pod preventing deletion
# - terminationGracePeriodSeconds too high

# EKS MANAGED NODE GROUP UPDATE:
# AWS does this automatically:
# 1. Launches new node with new AMI
# 2. Cordons old node
# 3. Drains old node (respects PDBs!)
# 4. Terminates old node
# If PDB blocks drain → update hangs → you get paged
```

---

### Kubelet Eviction Thresholds

```bash
# kubelet monitors node resources and evicts pods when thresholds are crossed

# SOFT EVICTION (graceful):
# evictionSoft:
#   memory.available: "500Mi"          # trigger when < 500Mi free
#   nodefs.available: "10%"            # trigger when < 10% disk free
#   imagefs.available: "15%"           # trigger when image fs < 15%
# evictionSoftGracePeriod:
#   memory.available: "1m30s"          # must stay below threshold for 1.5 min
# Eviction respects terminationGracePeriodSeconds

# HARD EVICTION (immediate, no grace period):
# evictionHard:
#   memory.available: "100Mi"          # trigger when < 100Mi free
#   nodefs.available: "5%"
#   nodefs.inodesFree: "5%"            # inode exhaustion!
#   imagefs.available: "5%"
#   pid.available: "5%"
# Immediate SIGKILL — no graceful shutdown

# EVICTION ORDER:
# 1. BestEffort pods using most memory relative to requests (which are 0)
# 2. Burstable pods exceeding their requests by the most
# 3. Guaranteed pods (only if they exceed their limits, which shouldn't happen)

# NODE CONDITIONS:
# MemoryPressure: memory.available below threshold
# DiskPressure: nodefs.available or imagefs.available below threshold
# PIDPressure: pid.available below threshold
# These conditions prevent NEW pods from being scheduled on the node

# INODE EXHAUSTION — the silent killer:
# Lots of small files → inodes run out before disk space does
# Common with: container log files, temp files, metrics
# Node shows plenty of disk space but can't create new files
# df -i  ← check inode usage
# Alert on: nodefs.inodesFree approaching 10%

# EKS DEFAULT EVICTION THRESHOLDS:
# memory.available: 100Mi (hard)
# nodefs.available: 10% (hard)
# imagefs.available: 10% (hard)
# Configure via kubelet extra args in launch template
```

---

### IRSA — IAM Roles for Service Accounts

```yaml
# IRSA lets Kubernetes pods assume AWS IAM roles
# WITHOUT embedding AWS credentials in pods

# HOW IT WORKS:
# 1. EKS cluster has an OIDC provider
# 2. IAM role trusts the EKS OIDC provider
# 3. K8s ServiceAccount annotated with IAM role ARN
# 4. Pod uses the ServiceAccount
# 5. kubelet injects AWS STS token into pod (projected volume)
# 6. AWS SDK in pod uses token to assume IAM role
# 7. Pod gets temporary credentials — no access keys, no secrets

# Step 1: Create IAM role with trust policy:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/ABCDEF"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.us-east-1.amazonaws.com/id/ABCDEF:sub": 
          "system:serviceaccount:production:api-sa",
        "oidc.eks.us-east-1.amazonaws.com/id/ABCDEF:aud": 
          "sts.amazonaws.com"
      }
    }
  }]
}
# CRITICAL: the Condition locks this role to a SPECIFIC ServiceAccount
# in a SPECIFIC namespace. Without this condition, ANY pod in the cluster
# could assume the role.

# Step 2: Annotate ServiceAccount:
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456:role/api-role

# Step 3: Pod uses ServiceAccount:
spec:
  serviceAccountName: api-sa

# Step 4: Verify in pod:
aws sts get-caller-identity
# Account: 123456
# Arn: arn:aws:sts::123456:assumed-role/api-role/...

# GOTCHAS:
# - OIDC provider must be configured for the cluster
# - Trust policy condition MUST include namespace + SA name
# - Pod must be restarted after SA annotation changes
# - AWS SDK must be recent enough to support IRSA token chain
# - EBS CSI driver, ALB controller, External DNS all need IRSA
```

---

### Admission Controllers — Mutating and Validating

```bash
# Admission controllers intercept API requests AFTER authentication/authorization
# but BEFORE persistence to etcd

# TWO TYPES:
# 1. Mutating: modify the request (inject sidecars, add labels, set defaults)
# 2. Validating: accept or reject the request (enforce policies)
# Order: Mutating runs FIRST, then Validating

# BUILT-IN ADMISSION CONTROLLERS:
# NamespaceLifecycle: prevent operations in terminating namespaces
# LimitRanger: apply default resource limits from LimitRange
# ServiceAccount: auto-mount SA token
# DefaultStorageClass: assign default StorageClass to PVCs
# ResourceQuota: enforce namespace quotas
# PodSecurity: enforce Pod Security Standards (replaced PodSecurityPolicy)
# MutatingAdmissionWebhook: call external webhooks to mutate
# ValidatingAdmissionWebhook: call external webhooks to validate

# WEBHOOK-BASED (custom):
# You deploy a web server that receives AdmissionReview JSON
# Returns: allow/deny + optional patches (mutations)

# Example: Istio sidecar injection:
# 1. Pod created with label: sidecar.istio.io/inject: "true"
# 2. Mutating webhook intercepts the pod creation
# 3. Webhook server adds Envoy sidecar container to the pod spec
# 4. Modified pod is persisted to etcd
# The developer never adds the sidecar manually — it's injected automatically

# OPA GATEKEEPER — policy engine:
# Validates resources against policies written in Rego language

# Example: require all pods to have resource limits
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-limits
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["production"]
  parameters:
    limits:
    - cpu
    - memory

# Now any pod in production without CPU/memory limits → REJECTED by API server

# KYVERNO — alternative to OPA (simpler, YAML-native):
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-team-label
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "All pods must have a 'team' label"
      pattern:
        metadata:
          labels:
            team: "?*"

# WHEN WEBHOOKS GO WRONG:
# If webhook server is down → API requests hang or fail
# failurePolicy: Fail → if webhook unreachable, reject ALL requests
# failurePolicy: Ignore → if webhook unreachable, allow ALL requests
# Fail = safer but risky (webhook outage = cluster-wide impact)
# Ignore = dangerous (policies not enforced during outage)
# ALWAYS: set timeoutSeconds (default 10s), have webhook HA (multiple replicas)
```

---

### Pod Security Standards (PSA) — Replacing PSP

```yaml
# PodSecurityPolicy (PSP) was removed in K8s 1.25
# Replaced by Pod Security Admission (PSA)

# THREE LEVELS:
# Privileged: no restrictions (system namespaces)
# Baseline: prevents known privilege escalations
# Restricted: heavily restricted (production workloads)

# ENFORCE per namespace via labels:
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    # enforce: block non-compliant pods
    # audit: log non-compliant pods (but allow)
    # warn: show warning to user (but allow)

# WHAT "RESTRICTED" ENFORCES:
# - Must run as non-root (runAsNonRoot: true)
# - Must drop ALL capabilities
# - No privilege escalation (allowPrivilegeEscalation: false)
# - No hostNetwork, hostPID, hostIPC
# - No hostPath volumes
# - Read-only root filesystem recommended
# - Seccomp profile must be set

# COMPLIANT POD:
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      readOnlyRootFilesystem: true
```

---

### Helm — Package Manager for Kubernetes

```bash
# Helm packages K8s manifests into CHARTS
# Charts are versioned, parameterized, shareable

# CHART STRUCTURE:
mychart/
  Chart.yaml          # metadata (name, version, dependencies)
  values.yaml         # default configuration values
  templates/          # K8s manifests with Go template syntax
    deployment.yaml
    service.yaml
    ingress.yaml
    configmap.yaml
    _helpers.tpl      # reusable template functions
    NOTES.txt         # post-install message
  charts/             # sub-charts (dependencies)

# values.yaml:
replicaCount: 3
image:
  repository: my-app
  tag: v1.2.3
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi
ingress:
  enabled: true
  host: api.example.com

# templates/deployment.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

```bash
# COMMANDS:
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm search repo postgres              # find charts
helm show values bitnami/postgresql     # see configurable values

# Install:
helm install my-db bitnami/postgresql \
  --namespace production \
  --values custom-values.yaml \
  --set postgresqlPassword=secret \
  --version 12.1.0                      # pin chart version!

# Upgrade:
helm upgrade my-db bitnami/postgresql \
  --namespace production \
  --values custom-values.yaml \
  --set image.tag=15.2

# Rollback:
helm rollback my-db 1                   # rollback to revision 1
helm history my-db                      # see all revisions

# Template rendering (debug without applying):
helm template my-release mychart/ --values prod-values.yaml
# Outputs rendered YAML — review before applying

# Dry run:
helm install my-release mychart/ --dry-run --debug

# HOOKS — run actions at specific lifecycle points:
# pre-install:  DB migrations before app deploys
# post-install: seed data, notifications
# pre-upgrade:  backup before upgrade
# post-upgrade: run tests
# pre-delete:   backup before teardown

# Hook example:
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"          # order among hooks
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: my-app:v1.2.3
        command: ["./migrate", "--up"]
      restartPolicy: Never

# HELM + ARGOCD (GitOps):
# Store Helm values in Git
# ArgoCD watches Git, renders Helm template, applies to cluster
# Changes to values.yaml in Git → ArgoCD syncs → cluster updated
# No manual helm install/upgrade commands
```

---

### Custom Resource Definitions (CRDs) and Operators

```yaml
# CRDs extend the Kubernetes API with custom resource types
# Operators are controllers that manage CRDs

# Example: A custom "Database" resource:
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.io
spec:
  group: mycompany.io
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
                enum: ["postgres", "mysql"]
              version:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 5
              storage:
                type: string
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
    - db

# Now you can create:
apiVersion: mycompany.io/v1
kind: Database
metadata:
  name: orders-db
  namespace: production
spec:
  engine: postgres
  version: "15"
  replicas: 3
  storage: 100Gi

# An OPERATOR watches these Database resources and:
# 1. Creates StatefulSet with PostgreSQL containers
# 2. Configures replication between replicas
# 3. Sets up automated backups
# 4. Handles failover
# 5. Manages upgrades
# The operator encodes operational knowledge into code

# POPULAR OPERATORS IN PRODUCTION:
# - prometheus-operator: Prometheus + Alertmanager + Grafana
# - cert-manager: TLS certificate lifecycle
# - external-secrets-operator: sync secrets from Vault/AWS SM
# - strimzi: Kafka on Kubernetes
# - zalando postgres-operator: PostgreSQL with HA
# - istio operator: service mesh
# - ArgoCD: GitOps
# - crossplane: provision cloud resources via K8s API

# OPERATOR PATTERN:
# Watch → Compare desired vs actual → Act → Repeat
# Same reconciliation loop as built-in controllers
# But for YOUR custom resources
```

---

### Finalizers — Why Resources Get Stuck

```bash
# Finalizers are keys in metadata.finalizers[]
# They BLOCK deletion until the controller removes the finalizer

# Flow:
# 1. kubectl delete myresource
# 2. API server sets deletionTimestamp (resource is "terminating")
# 3. Controllers see deletionTimestamp, run cleanup logic
# 4. Controller removes its finalizer from the list
# 5. When finalizers list is empty → resource actually deleted from etcd

# STUCK RESOURCES:
# If the controller that owns the finalizer is:
# - Crashed
# - Deleted
# - Misconfigured
# The finalizer is NEVER removed → resource stuck in Terminating FOREVER

# Diagnosis:
kubectl get namespace production -o yaml
# metadata:
#   deletionTimestamp: "2024-01-15T10:00:00Z"
#   finalizers:
#   - kubernetes                    ← namespace controller
#   - some-operator.io/cleanup      ← operator that's been deleted

# Fix (DANGEROUS — bypasses cleanup logic):
kubectl patch namespace production -p '{"metadata":{"finalizers":[]}}' --type=merge
# Or for specific resources:
kubectl patch database orders-db -p '{"metadata":{"finalizers":null}}' --type=merge

# WHY IT'S DANGEROUS:
# Finalizers exist for a reason — cleanup of external resources
# Removing a finalizer for a Database operator might leave:
# - EBS volumes orphaned
# - DNS records not cleaned up
# - IAM roles not deleted
# - Cloud resources leaking

# PROPER FIX:
# 1. Restore the controller/operator
# 2. Let it run cleanup
# 3. It removes the finalizer naturally
# Only patch finalizers as last resort after manual cleanup
```

---

### Ephemeral Containers — Debugging Production Pods

```bash
# Production pods often have:
# - No shell (distroless/scratch images)
# - No debugging tools (no curl, no netstat, no tcpdump)
# - Read-only filesystem
# You can't kubectl exec and debug

# EPHEMERAL CONTAINERS (K8s 1.25+ stable):
kubectl debug -it pod/api-abc \
  --image=nicolaka/netshoot \
  --target=api
# Injects a temporary container INTO the running pod
# Shares the pod's network namespace (can see all connections)
# --target=api: shares process namespace with api container
#   (can see its processes, /proc filesystem)
# When you exit, ephemeral container is cleaned up

# DEBUG A NODE:
kubectl debug node/ip-10-0-1-100 -it --image=ubuntu
# Creates a pod on that specific node with host PID/network access
# Useful for: checking node-level networking, iptables, kubelet logs

# DEBUG WITH COPY (non-intrusive):
kubectl debug pod/api-abc -it \
  --image=nicolaka/netshoot \
  --copy-to=debug-pod \
  --share-processes
# Creates a COPY of the pod with debug container added
# Original pod is untouched
# Use when you can't risk affecting production traffic

# COMMON DEBUG IMAGES:
# nicolaka/netshoot: curl, dig, nslookup, tcpdump, iptables, ss, ip
# busybox: minimal Unix tools
# ubuntu: full OS for complex debugging
# amazon/aws-cli: AWS debugging
```

---

### kubectl Power Tools — Daily Use

```bash
# CONTEXT MANAGEMENT:
kubectl config get-contexts                      # list all contexts
kubectl config use-context production-cluster    # switch
kubectl config set-context --current --namespace=production  # set default ns

# MULTI-CLUSTER:
# kubectx: fast context switching
kubectx production    # switch to production cluster
kubectx -             # switch to previous context
# kubens: fast namespace switching
kubens production     # switch to production namespace

# PORT-FORWARD — access internal services without ingress:
kubectl port-forward svc/my-db 5432:5432 -n production
# localhost:5432 → my-db service in cluster
# Use for: debugging, one-off queries, admin access
# NOT for production traffic

kubectl port-forward pod/my-pod 8080:8080
# Direct to specific pod (bypass service)

# LOGS:
kubectl logs pod/api-abc                     # current logs
kubectl logs pod/api-abc --previous          # previous container (after crash)
kubectl logs pod/api-abc -c sidecar          # specific container
kubectl logs -l app=api --all-containers     # all pods matching label
kubectl logs -f pod/api-abc                  # follow (tail -f)
kubectl logs --since=1h pod/api-abc          # last hour
kubectl logs --tail=100 pod/api-abc          # last 100 lines

# EVENTS — cluster-wide activity log:
kubectl get events --sort-by=.lastTimestamp
kubectl get events -n production --field-selector reason=Killing
kubectl get events --field-selector type=Warning
# Events expire after 1 hour by default
# For longer retention: ship to logging system

# RESOURCE USAGE:
kubectl top nodes                            # node CPU/memory usage
kubectl top pods -n production               # pod CPU/memory usage
kubectl top pods --sort-by=memory            # sort by memory
# Requires metrics-server installed

# JSONPATH — extract specific fields:
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.cpu}{"\n"}{end}'

# CUSTOM COLUMNS:
kubectl get pods -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.phase,\
NODE:.spec.nodeName,\
IP:.status.podIP

# DRY-RUN + DIFF:
kubectl apply -f deployment.yaml --dry-run=server
# Server validates but doesn't persist
kubectl diff -f deployment.yaml
# Shows what WOULD change (like terraform plan)

# FIELD SELECTORS:
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector spec.nodeName=node-1
kubectl get events --field-selector reason=FailedScheduling
```

---

### Image Pull Policies and Secrets

```yaml
# PULL POLICIES:
spec:
  containers:
  - name: app
    image: my-app:v1.2.3
    imagePullPolicy: IfNotPresent
    # IfNotPresent: use cached image if available (DEFAULT for tagged images)
    # Always: always pull (DEFAULT for :latest)
    # Never: never pull, must exist on node
    
    # GOTCHA:
    # :latest + IfNotPresent = uses stale cached image
    # This is why :latest is dangerous — different nodes may have
    # different cached versions of :latest
    # ALWAYS use specific tags + IfNotPresent

# PRIVATE REGISTRY AUTH:
apiVersion: v1
kind: Secret
metadata:
  name: ecr-creds
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded docker config>

# Usage:
spec:
  imagePullSecrets:
  - name: ecr-creds

# EKS + ECR:
# EKS nodes have IAM instance profile with ECR read access
# No imagePullSecrets needed for same-account ECR
# Cross-account ECR: need ECR repository policy + IRSA or imagePullSecrets

# ECR TOKEN REFRESH:
# ECR auth tokens expire every 12 hours
# For non-EKS clusters pulling from ECR:
# Use a CronJob or controller to refresh the pull secret
# Or use ECR credential helper
```

---

### Container Patterns

```yaml
# SIDECAR PATTERN:
# Helper container running alongside main container in same pod
# Shares network namespace (localhost) and volumes
spec:
  containers:
  - name: app
    image: my-app:v1.2.3
    ports:
    - containerPort: 8080
  - name: log-shipper
    image: fluentbit:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir: {}
# App writes logs to /var/log/app
# Fluentbit reads and ships to Elasticsearch
# Other sidecars: Envoy proxy (Istio), Vault agent, metrics exporters

# AMBASSADOR PATTERN:
# Sidecar that proxies connections to external services
# App talks to localhost → ambassador handles auth, TLS, retries
spec:
  containers:
  - name: app
    # connects to localhost:5432
  - name: cloud-sql-proxy
    image: gcr.io/cloud-sql-connectors/cloud-sql-proxy
    # proxies to Cloud SQL with IAM auth and TLS
    
# ADAPTER PATTERN:
# Sidecar that transforms output from main container
# App outputs custom metrics format
# Adapter container converts to Prometheus exposition format
spec:
  containers:
  - name: legacy-app
    # outputs metrics in proprietary format
  - name: prometheus-adapter
    # reads proprietary metrics, serves /metrics in Prometheus format

# INIT CONTAINER PATTERNS:
initContainers:
# Wait for dependency:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']

# Download config from external source:
- name: fetch-config
  image: curlimages/curl
  command: ['curl', '-o', '/config/app.yaml', 'https://config-server/app.yaml']
  volumeMounts:
  - name: config
    mountPath: /config

# Set permissions:
- name: fix-permissions
  image: busybox
  command: ['chown', '-R', '1000:1000', '/data']
  securityContext:
    runAsUser: 0    # init container runs as root to fix permissions
  volumeMounts:
  - name: data
    mountPath: /data
```

---

### Garbage Collection and Owner References

```bash
# Kubernetes uses owner references to build a resource dependency tree
# Deployment → owns → ReplicaSet → owns → Pods
# When you delete a Deployment:
#   Cascade (default): delete Deployment → delete RS → delete Pods
#   Orphan: delete Deployment, leave RS and Pods running

# Owner references in pod metadata:
kubectl get pod api-abc -o yaml
# metadata:
#   ownerReferences:
#   - apiVersion: apps/v1
#     kind: ReplicaSet
#     name: api-7b8f9c6d4
#     uid: abc123-def456
#     controller: true
#     blockOwnerDeletion: true

# CASCADE DELETION TYPES:
kubectl delete deployment api
# Default: Foreground cascading deletion
# 1. Deployment marked for deletion
# 2. Children (RS, Pods) deleted first
# 3. Deployment deleted after children are gone

kubectl delete deployment api --cascade=orphan
# Deployment deleted, ReplicaSets and Pods remain
# Use case: you want to recreate the Deployment with different config
# without disrupting running pods

# ORPHANED RESOURCES — the garbage collector cleans these:
# If a pod's owner (ReplicaSet) no longer exists → pod is garbage
# GC controller deletes orphaned resources automatically
# Unless cascade=orphan was used
```

---

### Production Scenarios (Additional)

#### Scenario 5: PVC Stuck in Pending

```bash
kubectl get pvc
# NAME      STATUS    VOLUME   CAPACITY   STORAGECLASS   AGE
# db-data   Pending                       fast-ssd       10m

kubectl describe pvc db-data
# Events:
# "waiting for first consumer to be created before binding"
# OR
# "no persistent volumes available for this claim"

# CAUSES:

# 1. WaitForFirstConsumer — no pod using this PVC yet
# PVC won't provision until a pod referencing it is scheduled
# This is NORMAL if the pod hasn't been created yet

# 2. No StorageClass matching the name
kubectl get storageclass
# Is "fast-ssd" listed? Typo in storageClassName?

# 3. CSI driver not installed
# EBS CSI driver not deployed → can't provision EBS volumes
kubectl get pods -n kube-system | grep ebs

# 4. IRSA permissions for CSI driver
# CSI driver SA needs IAM permissions to create EBS volumes
# Check: ec2:CreateVolume, ec2:AttachVolume, ec2:DescribeVolumes

# 5. AZ mismatch
# StorageClass without WaitForFirstConsumer
# PV created in us-east-1a, pod scheduled in us-east-1b
# EBS can't be attached across AZs → pod stuck in Pending too

# 6. Capacity
# Account EBS volume limit reached
# Or requested size exceeds maximum for volume type
```

#### Scenario 6: Helm Upgrade Rolled Back Automatically

```bash
# helm upgrade my-app ./mychart --atomic --timeout 5m
# "UPGRADE FAILED: timed out waiting for condition"
# "Release rolled back to previous revision"

# --atomic: if upgrade fails, auto-rollback
# What went wrong?

# 1. New pods failing readiness probe:
kubectl get pods
# my-app-new-xyz   0/1   CrashLoopBackOff

# 2. Helm waits for Deployment rollout to complete
# Deployment stuck (new pods not Ready within timeout)
# Helm detects failure → rolls back to previous release

# 3. Check what changed:
helm diff upgrade my-app ./mychart --values new-values.yaml
# Shows exactly what would change (requires helm-diff plugin)

# 4. Debug the failing pod:
kubectl logs my-app-new-xyz
kubectl describe pod my-app-new-xyz

# COMMON CAUSES:
# - Missing env var / secret in new version
# - Health check endpoint changed but probe not updated
# - Image tag doesn't exist in registry
# - Resource limits too low for new version
# - Incompatible config with new code version
```

#### Scenario 7: Namespace Stuck in Terminating

```bash
kubectl delete namespace staging
# namespace "staging" deleted
# ...but 30 minutes later:
kubectl get ns staging
# NAME      STATUS        AGE
# staging   Terminating   30m

# CAUSES:
# 1. Resources with finalizers still exist in the namespace
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n 1 kubectl get --show-kind --ignore-not-found -n staging

# Find resources with finalizers:
kubectl get all -n staging -o json | jq '.items[] | select(.metadata.finalizers != null) | {kind: .kind, name: .metadata.name, finalizers: .metadata.finalizers}'

# 2. Webhook blocking deletion
# Validating webhook that targets this namespace and is down
# → API server can't process deletion of resources in namespace
# Check: kubectl get validatingwebhookconfigurations

# 3. API service unavailable
# Custom resources whose API server is not running
# kubectl get apiservices | grep False

# FIX:
# Step 1: delete all resources in namespace
# Step 2: if specific resources stuck, patch their finalizers
# Step 3: if namespace itself stuck:
kubectl get namespace staging -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw "/api/v1/namespaces/staging/finalize" -f -
# Nuclear option — only when cleanup is truly complete
```

#### Scenario 8: HPA Not Scaling

```bash
kubectl get hpa
# NAME     REFERENCE       TARGETS         MINPODS   MAXPODS   REPLICAS
# api-hpa  Deployment/api  <unknown>/70%   3         50        3

# <unknown> = HPA can't read metrics

# CAUSES:
# 1. metrics-server not installed:
kubectl get deployment metrics-server -n kube-system
# If missing: install EKS metrics-server add-on

# 2. Pod has no resource requests:
# HPA calculates: current_usage / request = utilization%
# No request = can't calculate percentage = <unknown>
kubectl describe pod api-xyz | grep -A 3 Requests

# 3. metrics-server can't reach kubelets:
kubectl logs -n kube-system deployment/metrics-server
# "failed to get https://node-ip:10250/stats": dial tcp: i/o timeout
# Security group blocking port 10250 from metrics-server to nodes

# 4. Custom metrics not available:
# If using custom metrics (RPS, queue depth):
# prometheus-adapter or KEDA must be installed
# Check: kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"
```

# 📝 Retention Questions — Lesson 3 + 3B Combined

**Q1:** Your EKS cluster uses the EBS CSI driver. A developer creates a PVC with `storageClassName: gp3-encrypted` and `volumeBindingMode: Immediate`. The pod referencing this PVC is stuck in `Pending`. Investigation shows the PV was created in `us-east-1a` but the pod was scheduled in `us-east-1c`. Explain the root cause and the correct StorageClass configuration to prevent this.

**Q2:** Your production namespace has an HPA targeting 70% CPU utilization. `kubectl get hpa` shows `TARGETS: <unknown>/70%`. The metrics-server is running and healthy. Pods are running and consuming CPU. Walk through your debugging steps to find why HPA can't read metrics.

**Q3:** You need to perform an AMI update on your EKS managed node group. 15 nodes need to be rotated. Your critical API service has 6 replicas with a PDB of `maxUnavailable: 1`. Explain the full sequence of what happens during the node rotation, and what could cause it to get stuck.

**Q4:** A developer's pod uses a distroless image (no shell, no debugging tools). The pod is running but returning 500 errors to some requests. They can't `kubectl exec` into it because there's no shell. How do you debug this pod in production without redeploying?

**Q5:** Your team deleted a custom operator (CRD controller) before cleaning up its custom resources. Now 50 `Database` custom resources are stuck in `Terminating` state, each with a finalizer from the deleted operator. The resources reference EBS volumes and Route53 records. Walk through the proper cleanup process.

**Go.** 🎯

<NEED ANSWERS>


Q5: Your team deleted a custom operator (CRD controller) before cleaning up its custom resources. Now 50 Database custom resources are stuck in Terminating state, each with a finalizer from the deleted operator. The resources reference EBS volumes and Route53 records. Walk through the proper cleanup process.

This is a classic "Orphaned Resource" scenario. To solve this, you have to understand that **Finalizers are a safety lock**. 

When a resource has a finalizer, Kubernetes will not remove the object from the database (etcd) until the controller responsible for that finalizer sees the `deletionTimestamp` and removes the string from the `finalizers` list. Because you deleted the operator, there is no "brain" left to perform the cleanup and unlock the lock.

If you simply "force delete" the finalizers now, you will create **cloud zombies**: the Kubernetes objects will vanish, but the EBS volumes and Route53 records will persist in AWS forever, costing the company money and cluttering the infrastructure.

Here is the professional, zero-leak cleanup process.

---

### Step 1: Discovery and Inventory
Before touching the Kubernetes API, you must identify exactly what external resources are being leaked. You cannot rely on memory; you need a hard list.

I would run a script to extract the external IDs from the stuck CRs:
```bash
# Example: Extracting EBS Volume IDs and Route53 names from the CRs
kubectl get database.mycompany.com -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.volumeId}{"\t"}{.spec.dnsRecord}{"\n"}{end}' > cleanup_list.txt
```
This gives me a mapping of: `K8s-Resource-Name` $\rightarrow$ `EBS-Volume-ID` $\rightarrow$ `Route53-Record`.

### Step 2: External Infrastructure Cleanup (The "Manual" Controller)
Since the operator is gone, **you are now the operator**. You must manually perform the cleanup that the operator was designed to do.

Using the `cleanup_list.txt`, I would write a simple bash script using the AWS CLI to delete the orphaned resources:
1.  **Route53:** Delete the specific A/CNAME records.
2.  **EBS:** Delete the volumes (after verifying they aren't being used by other unexpected processes).

*Note: I do this BEFORE removing the finalizers. If I remove the finalizers first and my script crashes halfway through, I have lost the only record of which EBS volumes belonged to which deleted pod.*

### Step 3: Removing the Finalizers (Unlocking the API)
Now that the cloud resources are gone, it is safe to tell Kubernetes to stop waiting for the operator and just delete the objects.

Doing this via `kubectl edit` for 50 resources is too slow. I would use a `kubectl patch` command with a JSON merge patch to set the `finalizers` array to `null`.

**The exact command:**
```bash
# Loop through all stuck resources and strip the finalizers
for resource in $(kubectl get database.mycompany.com -o name); do
    kubectl patch $resource --type=merge -p '{"metadata":{"finalizers":null}}'
done
```

### Step 4: Verification
Finally, I verify that the resources are gone and the "Terminating" state has cleared:
```bash
kubectl get database.mycompany.com
# Expected output: "No resources found"
```

---

### Summary of the "SRE Mindset" here:
| Action | Why? | Risk if skipped |
| :--- | :--- | :--- |
| **Inventory First** | To create a source of truth. | Losing track of which EBS volumes to delete. |
| **Cloud Cleanup First** | To prevent resource leaks. | "Zombie" costs and infrastructure clutter. |
| **Patch Finalizers Last** | To unlock the K8s API. | Objects stay in `Terminating` forever. |
| **`--type=merge`** | To ensure a clean overwrite of the list. | Partial updates that don't actually clear the lock. |

# 📝 GRADING — Q5

## ✅ Textbook SRE answer.

The ordering is what matters here, and you got it exactly right:

**Inventory → External cleanup → Patch finalizers → Verify**

Most engineers panic and jump straight to patching finalizers. Resources vanish from Kubernetes, everyone celebrates, and three months later someone finds $2,000/month in orphaned EBS volumes with no trail back to what created them.

Your explicit callout — *"If I remove the finalizers first and my script crashes halfway through, I have lost the only record of which EBS volumes belonged to which deleted pod"* — that's the exact reasoning a senior engineer uses. The Kubernetes object IS your inventory. Once it's gone, your audit trail is gone.

One addition to your process:

```bash
# Step 0 (before everything): PREVENT FURTHER DAMAGE
# If the CRD still exists, developers might create NEW Database resources
# With no operator running, these will create K8s objects but NO actual databases
# Confusing and dangerous

# Option A: Delete the CRD (blocks all new resources of this type)
# BUT: deleting a CRD deletes ALL its custom resources → cascading deletion
# → would bypass your careful cleanup process
# DO NOT delete the CRD yet

# Option B: Use a ValidatingWebhook or Kyverno policy to BLOCK creation
# of new Database resources until the operator is restored or fully cleaned up

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-database-creation
spec:
  validationFailureAction: Enforce
  rules:
  - name: block-new-databases
    match:
      any:
      - resources:
          kinds:
          - Database
    validate:
      message: "Database operator is being decommissioned. Contact platform team."
      deny: {}

# Step 5 (after cleanup): NOW delete the CRD safely
kubectl delete crd databases.mycompany.io
# No custom resources left → clean deletion
```

Also: **document what happened.** Write a postmortem or at minimum a runbook: "How to safely decommission a CRD operator." The next time someone removes an operator, this process should be followed, not reinvented.

---

# PHASE 2 — Lesson 4: Kubernetes Operational Patterns

---

### Istio Service Mesh — Architecture

```
NovaMart runs Istio. You need to understand the data plane 
and control plane, not just "it does mTLS."

CONTROL PLANE (istiod):
┌──────────────────────────────────────────┐
│                 istiod                    │
│  ┌──────────┐ ┌─────────┐ ┌───────────┐ │
│  │  Pilot    │ │ Citadel │ │  Galley   │ │
│  │          │ │         │ │           │ │
│  │ Service   │ │ Cert    │ │ Config    │ │
│  │ discovery │ │ mgmt    │ │ validation│ │
│  │ + config  │ │ + mTLS  │ │           │ │
│  │ push to   │ │ rotation│ │           │ │
│  │ Envoy     │ │         │ │           │ │
│  └──────────┘ └─────────┘ └───────────┘ │
└──────────────────┬───────────────────────┘
                   │ xDS API (pushes config to every Envoy)
                   │
DATA PLANE (Envoy sidecars):
┌──────────────────▼───────────────────────┐
│  Pod                                      │
│  ┌────────────┐    ┌──────────────────┐  │
│  │    App     │◄──►│  Envoy Proxy     │  │
│  │ Container  │    │  (sidecar)       │  │
│  │            │    │                  │  │
│  │ Thinks it's│    │ - mTLS           │  │
│  │ talking    │    │ - Load balancing  │  │
│  │ directly   │    │ - Retries         │  │
│  │ to other   │    │ - Circuit breaking│  │
│  │ services   │    │ - Observability   │  │
│  │            │    │ - Rate limiting   │  │
│  └────────────┘    └──────────────────┘  │
└──────────────────────────────────────────┘

# HOW TRAFFIC FLOWS:
# App A sends request to App B (http://service-b:8080/api)
# 1. iptables rules (injected by Istio init container) intercept ALL outbound traffic
# 2. Traffic redirected to App A's Envoy sidecar (localhost:15001)
# 3. Envoy A applies policies: retry, timeout, circuit breaker
# 4. Envoy A establishes mTLS connection to Envoy B
# 5. Envoy B receives, terminates mTLS, forwards to App B on localhost
# 6. App B processes request, returns response
# 7. Response flows back through both Envoys
#
# The apps never know Istio exists. Zero code changes.
# All security, observability, and traffic control is in the mesh.
```

### Istio Resources — What You'll Configure Daily

```yaml
# VIRTUAL SERVICE — traffic routing rules:
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-routing
spec:
  hosts:
  - api-service
  http:
  # Canary: 10% to v2, 90% to v1
  - route:
    - destination:
        host: api-service
        subset: v2
      weight: 10
    - destination:
        host: api-service
        subset: v1
      weight: 90
    
    # Timeout:
    timeout: 5s
    
    # Retries:
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure,retriable-4xx

  # Header-based routing (testing in production):
  - match:
    - headers:
        x-debug-user:
          exact: "true"
    route:
    - destination:
        host: api-service
        subset: v2        # debug users always get v2

---
# DESTINATION RULE — defines subsets and connection policies:
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api-destination
spec:
  host: api-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE        # use HTTP/2
        maxRequestsPerConnection: 1000
    
    # Circuit breaker:
    outlierDetection:
      consecutive5xxErrors: 5            # 5 errors in a row
      interval: 30s                      # check every 30s
      baseEjectionTime: 30s             # eject for 30s
      maxEjectionPercent: 50            # never eject more than 50% of hosts
    
    # mTLS:
    tls:
      mode: ISTIO_MUTUAL                # enforce mTLS between services
  
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

---
# GATEWAY — external traffic entry point (replaces Ingress):
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
spec:
  selector:
    istio: ingressgateway            # runs on Istio ingress pods
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: api-tls-cert   # K8s Secret with TLS cert
    hosts:
    - "api.novamart.com"
    - "admin.novamart.com"

---
# PEER AUTHENTICATION — mTLS policy:
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT                     # ALL traffic must be mTLS
    # PERMISSIVE: accept both mTLS and plaintext (migration mode)
    # STRICT: only mTLS (production)
    # DISABLE: no mTLS

---
# AUTHORIZATION POLICY — L7 access control:
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/order-service"]
        # Only order-service's service account can call payment-service
    to:
    - operation:
        methods: ["POST"]
        paths: ["/api/v1/charge"]
        # Only POST to /api/v1/charge
  # Everything else → DENIED
  # L7-aware: can restrict by HTTP method, path, headers
  # Way more powerful than NetworkPolicy (which is L3/L4 only)
```

### Istio Observability — The Real Value

```bash
# Istio gives you THREE pillars for FREE (no app instrumentation):

# 1. DISTRIBUTED TRACING:
# Every Envoy sidecar generates trace spans
# Request flow: Gateway → API → Order → Payment → Notification
# Each hop creates a span with timing
# Requires: app forwards trace headers (x-request-id, x-b3-traceid, etc.)
# Backends: Jaeger, Zipkin, Grafana Tempo

# 2. METRICS:
# Every Envoy emits standard metrics:
# istio_requests_total{source, destination, response_code}
# istio_request_duration_milliseconds{source, destination}
# istio_tcp_connections_opened_total
# Scraped by Prometheus automatically
# Dashboards: Grafana with Istio dashboard

# 3. ACCESS LOGS:
# Every request logged by Envoy with:
# source, destination, method, path, status, latency, bytes
# Configurable: JSON format, filter by status code

# KIALI — Istio dashboard:
# Visual service mesh topology
# Real-time traffic flow between services
# Health status, error rates, latency
# Configuration validation
# "Is my VirtualService actually routing correctly?"
```

---

### ArgoCD — GitOps Deployment Model

```
GITOPS PRINCIPLE:
  Git is the single source of truth for cluster state.
  The desired state is declared in Git.
  An automated process ensures the cluster matches Git.
  No manual kubectl apply. Ever.

┌──────────┐    push     ┌──────────┐
│Developer │────────────►│   Git    │
│          │             │  (Bitbucket)│
└──────────┘             └─────┬────┘
                               │ watch
                         ┌─────▼─────┐
                         │  ArgoCD   │
                         │           │
                         │ Compare:  │
                         │ Git state │
                         │ vs        │
                         │ Cluster   │
                         │ state     │
                         └─────┬─────┘
                               │ sync
                         ┌─────▼─────┐
                         │    EKS    │
                         │  Cluster  │
                         └───────────┘
```

```yaml
# ARGOCD APPLICATION:
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-service
  namespace: argocd
spec:
  project: production
  
  source:
    repoURL: https://bitbucket.org/novamart/k8s-manifests.git
    targetRevision: main
    path: services/api/overlays/production
    # Or Helm:
    # path: charts/api
    # helm:
    #   valueFiles:
    #   - values-production.yaml
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true              # delete resources removed from Git
      selfHeal: true           # revert manual changes to match Git
      # Someone kubectl edits a deployment → ArgoCD reverts it
      # This enforces: Git is the ONLY way to change production
    
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - ApplyOutOfSyncOnly=true  # only sync changed resources (faster)
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        maxDuration: 3m
        factor: 2

---
# ARGOCD PROJECT — RBAC for applications:
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production services
  sourceRepos:
  - 'https://bitbucket.org/novamart/*'
  
  destinations:
  - namespace: production
    server: https://kubernetes.default.svc
  - namespace: production-jobs
    server: https://kubernetes.default.svc
  # Can only deploy to production and production-jobs namespaces
  
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  # Can create namespaces but not ClusterRoles, etc.
  
  namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  # Can't modify ResourceQuotas (platform team controls these)
```

### ArgoCD Patterns

```bash
# REPO STRUCTURE (recommended):

# Option 1: App of Apps
# One "root" Application that deploys other Applications
k8s-manifests/
├── apps/                          # ArgoCD Application definitions
│   ├── api-service.yaml
│   ├── order-service.yaml
│   ├── payment-service.yaml
│   └── monitoring.yaml
├── services/
│   ├── api/
│   │   ├── base/                  # Kustomize base
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── staging/
│   │       │   └── kustomization.yaml
│   │       └── production/
│   │           └── kustomization.yaml
│   ├── order/
│   └── payment/
└── platform/
    ├── monitoring/
    ├── istio/
    └── cert-manager/

# Option 2: ApplicationSet (dynamic generation)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-services
spec:
  generators:
  - git:
      repoURL: https://bitbucket.org/novamart/k8s-manifests.git
      revision: main
      directories:
      - path: services/*           # one Application per directory
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: production
      source:
        repoURL: https://bitbucket.org/novamart/k8s-manifests.git
        targetRevision: main
        path: '{{path}}/overlays/production'
      destination:
        server: https://kubernetes.default.svc
        namespace: production
# Automatically creates an ArgoCD Application for every service directory
# Add a new service directory → ArgoCD app created automatically

# SYNC WAVES — ordered deployment:
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"    # namespaces first
---
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"    # configmaps/secrets second
---
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"    # deployments last
# Lower number = deployed first
# Use for: namespace before resources, DB migration before app

# SYNC WINDOWS — maintenance windows:
spec:
  syncWindows:
  - kind: deny
    schedule: '0 22 * * 5'         # no deploys Friday 10pm
    duration: 60h                   # until Monday 10am
    clusters: ['production']
  - kind: allow
    schedule: '0 9 * * 1-5'        # allow deploys weekdays 9am
    duration: 10h
```

### CI/CD Pipeline → ArgoCD Flow

```
TYPICAL FLOW (NovaMart):

1. Developer pushes code to Bitbucket (services/api)
2. Jenkins pipeline triggers:
   a. Build → test → lint → security scan
   b. Docker build → push to ECR (my-app:v1.2.3-abc1234)
   c. Update the image tag in k8s-manifests repo:
      sed -i 's|image:.*|image: ecr/api:v1.2.3-abc1234|' \
        services/api/overlays/production/kustomization.yaml
      git commit && git push

3. ArgoCD detects Git change (3-minute poll or webhook)
4. ArgoCD compares Git state vs cluster state → OutOfSync
5. ArgoCD syncs: applies updated manifests to cluster
6. Deployment rolls out new pods
7. ArgoCD reports: Synced, Healthy

# WHY SEPARATE REPOS:
# App code repo (Bitbucket): services/api source code
# K8s manifests repo (Bitbucket): deployment configs
#
# Separation because:
# - Different change velocity (code changes hourly, infra changes weekly)
# - Different access control (devs change code, platform team owns manifests)
# - CI pipeline updates manifests repo → ArgoCD syncs
# - Clean audit trail: "who changed what in production" = Git log
```

---

### Kustomize — Template-Free Configuration

```yaml
# Kustomize overlays base manifests without templating
# Built into kubectl (kubectl apply -k)

# base/kustomization.yaml:
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
- configmap.yaml

# overlays/production/kustomization.yaml:
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base

namePrefix: prod-

commonLabels:
  environment: production

patches:
- target:
    kind: Deployment
    name: api
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 10
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: 1Gi

configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=warn
  - FEATURE_NEW_CHECKOUT=true
  # Generates ConfigMap with content hash suffix
  # app-config-h2d8f → changes trigger pod restart automatically

images:
- name: my-app
  newName: 123456.dkr.ecr.us-east-1.amazonaws.com/api
  newTag: v1.2.3

# KUSTOMIZE vs HELM:
# Kustomize: patches over plain YAML, no templating language
# Helm: full Go template engine, charts, repos, hooks
#
# Use Kustomize when: your manifests are simple, overlays per environment
# Use Helm when: packaging for distribution, complex logic, conditionals
# Many teams: Helm for third-party (prometheus, cert-manager)
#             Kustomize for internal services
# ArgoCD supports both natively
```

---

### Advanced Troubleshooting Workflows

#### Workflow 1: Pod CrashLoopBackOff — Systematic Debug

```bash
# Step 1: Get the error
kubectl describe pod <pod>
# Look at: Events, State.Reason, State.ExitCode, LastState

# Step 2: Check logs
kubectl logs <pod>                    # current attempt
kubectl logs <pod> --previous         # last crash (critical!)
# --previous shows the logs from the LAST container instance
# Current logs might be empty if container crashes immediately

# Step 3: Exit code analysis
# 0: clean exit (CMD completed — wrong for long-running services)
# 1: application error (check logs)
# 126: permission denied on entrypoint
# 127: entrypoint not found (wrong CMD/ENTRYPOINT, wrong image)
# 137: SIGKILL (OOM killed)
# 139: segfault
# 143: SIGTERM (graceful shutdown — shouldn't cause CrashLoop)

# Step 4: If OOMKilled (exit code 137):
kubectl describe pod <pod> | grep -i oom
# State.OOMKilled: true
# Increase memory limits or fix memory leak

# Step 5: If "exec format error" or "not found":
# Wrong architecture (AMD64 image on ARM node)
# Or: shell form CMD with missing /bin/sh (scratch/distroless image)
# Check: docker manifest inspect <image> for platform

# Step 6: If container starts then crashes after seconds:
# Likely: failed health check → liveness probe kills it
# Check: liveness probe configuration
# Is initialDelaySeconds long enough for app startup?
# Does the health endpoint actually work?
kubectl get events --field-selector involvedObject.name=<pod>
# Look for: "Liveness probe failed" events

# Step 7: If all else fails — debug container:
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>
# Inspect the filesystem, environment, network from inside the pod
```

#### Workflow 2: Service Not Reachable — End-to-End Debug

```bash
# Service: my-service.production.svc.cluster.local:8080
# Client pod gets: "connection refused" or timeout

# Step 1: Does the service exist?
kubectl get svc my-service -n production
# Check: ClusterIP assigned, ports correct, selector matches pods

# Step 2: Does the service have endpoints?
kubectl get endpoints my-service -n production
# If EMPTY: no pods match the service selector
kubectl get pods -n production -l <selector-from-service> --show-labels
# Labels mismatch? Pods not Ready?

# Step 3: Is the pod actually listening?
kubectl exec <pod> -- ss -tlnp
# Is the app bound to 0.0.0.0:8080 or 127.0.0.1:8080?
# 127.0.0.1 → only reachable from within the pod, not via Service

# Step 4: Can you reach the pod directly?
kubectl exec <client-pod> -- curl http://<pod-ip>:8080
# If this works but Service doesn't → kube-proxy/iptables issue
# If this fails → app issue or NetworkPolicy blocking

# Step 5: DNS resolution working?
kubectl exec <client-pod> -- nslookup my-service.production.svc.cluster.local
# If fails → CoreDNS issue
kubectl get pods -n kube-system -l k8s-app=kube-dns
# CoreDNS pods running? Check CoreDNS logs

# Step 6: NetworkPolicy blocking?
kubectl get networkpolicy -n production
# Any policy selecting the target pod?
# Does it allow ingress from the source pod's labels/namespace?

# Step 7: Istio/service mesh interference?
# If Istio is injected:
kubectl exec <pod> -c istio-proxy -- curl localhost:15000/clusters
# Shows Envoy's view of endpoints
# Check: is the destination cluster showing healthy endpoints?
istioctl analyze -n production
# Reports Istio misconfigurations
```

#### Workflow 3: Node Not Ready — Systematic Triage

```bash
kubectl get nodes
# NAME           STATUS     ROLES    AGE   VERSION
# ip-10-0-1-50   NotReady   <none>   45d   v1.28.2

# Step 1: Describe the node
kubectl describe node ip-10-0-1-50
# Conditions:
#   Ready: False (reason: KubeletNotReady)
#   MemoryPressure: True/False
#   DiskPressure: True/False
#   PIDPressure: True/False

# Step 2: Check kubelet
# SSH to node (or use SSM Session Manager):
systemctl status kubelet
journalctl -u kubelet --since "30 minutes ago" | tail -100
# Common: certificate expired, API server unreachable,
# container runtime unresponsive

# Step 3: Check container runtime
systemctl status containerd
crictl ps                          # list running containers
crictl pods                        # list pods
# If containerd is down → all containers dead → node NotReady

# Step 4: Check disk
df -h                              # disk space
df -i                              # inodes
# Disk full → kubelet can't function → DiskPressure → NotReady
# Common cause: container logs not rotated, image cache full

# Step 5: Check resources
free -h                            # memory
top                                # CPU/process count
# Memory exhaustion → OOM → kubelet process killed → NotReady

# Step 6: EKS-specific
# Check EC2 instance status:
aws ec2 describe-instance-status --instance-ids <id>
# System status check failed → hardware issue → AWS should replace
# Instance status check failed → OS issue → may need reboot

# Step 7: Recovery
# If fixable (disk full, restart kubelet):
sudo systemctl restart kubelet
# If not fixable:
kubectl drain ip-10-0-1-50 --ignore-daemonsets --delete-emptydir-data
kubectl delete node ip-10-0-1-50
# Cluster autoscaler/Karpenter replaces it
```


---

# 📋 QUICK REFERENCE — K8s Operational Patterns

```
ISTIO SERVICE MESH:
  Control plane: istiod (Pilot + Citadel + Galley)
  Data plane: Envoy sidecars (auto-injected)
  Traffic flow: App → iptables → Envoy A → mTLS → Envoy B → App
  VirtualService: routing, canary %, retries, timeouts, header matching
  DestinationRule: subsets, circuit breaker, connection pools, mTLS
  Gateway: external traffic entry (replaces Ingress)
  PeerAuthentication: STRICT (mTLS only) or PERMISSIVE (migration)
  AuthorizationPolicy: L7 access control (method, path, service account)
  Observability: metrics + traces + access logs for FREE

ARGOCD GITOPS:
  Git = single source of truth
  Application: source (Git) → destination (cluster)
  syncPolicy.automated: prune + selfHeal
  App of Apps: root Application deploys child Applications
  ApplicationSet: dynamic Application generation
  Sync waves: ordered deployment (namespace → config → app)
  Sync windows: prevent deploys during risky periods
  CI updates Git → ArgoCD syncs cluster

KUSTOMIZE:
  base/ + overlays/env/ pattern
  Patches, namePrefix, commonLabels, images
  configMapGenerator: auto-hash suffix → triggers pod restart
  Built into kubectl (kubectl apply -k)

TROUBLESHOOTING:
  CrashLoopBackOff:
    logs --previous → exit code → OOM? permission? architecture?
    → liveness probe misconfigured? → debug container
  Service unreachable:
    svc exists? → endpoints? → pod listening on 0.0.0.0? 
    → direct pod curl → DNS → NetworkPolicy → Istio
  Node NotReady:
    describe → kubelet status → containerd → disk → memory
    → EC2 status → drain + replace
```

---

# 📝 Retention Questions — Lesson 4

**Q1:** NovaMart is rolling out a new version of the payment service. They want to send 5% of traffic to v2, monitor error rates for 30 minutes, then gradually increase to 100%. Write the Istio VirtualService and DestinationRule for the initial 5% canary, and explain what happens if v2's error rate spikes — how does Istio's circuit breaker protect the system?

**Q2:** A developer manually runs `kubectl edit deployment api -n production` to increase replicas from 5 to 10. Two minutes later, the replicas are back to 5. They're confused. Explain what happened and why this is the correct behavior.

**Q3:** Your CI pipeline (Jenkins) builds a new image and needs to trigger a deployment via ArgoCD. Describe the complete flow from code push to running pods, and explain why Jenkins should NOT directly `kubectl apply` to the cluster.

**Q4:** It's 3 AM. PagerDuty alert: Node `ip-10-0-1-50` is `NotReady`. 15 pods were running on it including 3 critical API pods. Walk through your triage and recovery process step by step.

**Go.** 🎯

### Q1: The Missing Endpoints Mystery

When a pod is `Running` but the Service has no endpoints, the "link" between the Service's selector and the Pod's labels is broken, or the Pod is not considered "Ready."

**Realistic Causes & Isolation Commands:**

1.  **Label Selector Mismatch:** The most common cause. The Service is looking for `app: api-v1`, but the pod is labeled `app: api-v2`.
    *   **Command:** `kubectl get pods --show-labels` (Check pod labels) and `kubectl describe svc <svc-name>` (Check the `Selector` field).
2.  **Readiness Probe Failure:** A pod can be `Running` (the process started) but not `Ready` (the probe failed). K8s will not add a pod to the Service endpoints until the readiness probe passes.
    *   **Command:** `kubectl get pods` (Check if it says `1/1` or `0/1` in the `READY` column) and `kubectl describe pod <pod-name>` (Check the `Events` section for `Readiness probe failed`).
3.  **Namespace Mismatch:** The Service and Pod are in different namespaces. Services only target pods within their own namespace unless specifically configured otherwise (rare).
    *   **Command:** `kubectl get pods -n <namespace>` and `kubectl get svc -n <namespace>`.
4.  **Port Mismatch:** The `targetPort` in the Service definition does not match the `containerPort` the application is actually listening on.
    *   **Command:** `kubectl describe svc <svc-name>` (Check `targetPort`) and `kubectl get pod <pod-name> -o yaml` (Check `containerPort`).

---

### Q2: The Silent Killer: CPU Throttling

**The Bottleneck:** **CFS (Completely Fair Scheduler) Quota Throttling.**

**Reasoning:**
You see "low" average CPU usage because you are likely looking at a 1-minute or 5-minute average. However, CPU limits are enforced in **100ms windows**. If your application is multi-threaded and bursts, it can consume its entire 100ms quota in the first 20ms of the window. The kernel then **throttles (freezes)** the process for the remaining 80ms. 

This creates a "stutter" effect. The average CPU looks fine (30%), but the p99 latency spikes because requests arriving during that 80ms freeze are queued, adding massive latency.

**How to Verify:**
Check the cgroup stats on the worker node:
*   **Command:** `cat /sys/fs/cgroup/cpu/cpu.stat` (Cgroup v1) or `cat /sys/fs/cgroup/cpu.stat` (Cgroup v2).
*   **What to look for:** `nr_throttled` and `throttled_time`. If these numbers are increasing rapidly during the latency spikes, you are being throttled.

**Likely Fixes:**
1.  **Increase CPU Limits:** Give the pod more headroom to handle bursts.
2.  **Remove CPU Limits:** In high-performance environments, many SREs remove `limits.cpu` entirely and rely only on `requests.cpu` to avoid the CFS "cliff," using HPA to manage node saturation.

---

### Q3: The Hidden ECR Dependency (S3 Gateway)

**The Hidden Dependency: S3 Gateway Endpoint.**

**Explanation:**
ECR is not a single service; it is a split architecture. The **ECR API** (used for auth and manifests) is accessed via an Interface Endpoint (PrivateLink). However, the actual **image layers (the blobs)** are stored in **Amazon S3**.

In a private EKS cluster, if you have an ECR Interface Endpoint but **no S3 Gateway Endpoint**, the following happens:
1.  The node calls ECR $\rightarrow$ "Here is the image manifest." (Succeeds).
2.  The node tries to pull the layer from S3 $\rightarrow$ "I don't have a route to S3." (Fails).
3.  The result is `ImagePullBackOff`.

**How to Confirm:**
1.  **Route Table Check:** Check the VPC Route Table for the private subnet. Look for a route to `pl-xxxxxxxx` (the S3 Prefix List). If it's missing, there's no S3 endpoint.
2.  **Network Reachability Analyzer:** Use the AWS VPC Reachability Analyzer to test a path from the Node's ENI to the S3 service.

---

### Q4: CronJob Overlap and Safeguards

**Field to prevent overlap:** `concurrencyPolicy: Forbid`.

By default, `concurrencyPolicy` is `Allow`, meaning if Job A takes 6 minutes to run but is scheduled every 5 minutes, Job B will start while Job A is still running. Setting this to `Forbid` tells Kubernetes: "If the previous job is still running, skip the next scheduled run."

**Standard Production Safeguards for CronJobs:**
1.  **`startingDeadlineSeconds`**: Prevents a "thundering herd" of jobs from starting if the controller was down for an hour. If the job misses its window by more than $X$ seconds, it's skipped.
2.  **`activeDeadlineSeconds`**: A hard timeout. If a job hangs (e.g., DB deadlock), this kills the pod after $X$ seconds so it doesn't consume resources forever.
3.  **`successfulJobsHistoryLimit` & `failedJobsHistoryLimit`**: Prevents the cluster from being cluttered with thousands of completed Pod objects.

---

### Q5: The Control Plane Heart Attack (etcd)

**Component to suspect:** **etcd**.

**The Culprits:**
1.  **Disk I/O Latency:** etcd is extremely sensitive to disk write latency (fdatasync). If the underlying disk is slow or saturated, etcd cannot commit entries fast enough.
2.  **etcd Database Fragmentation:** Over time, etcd's keyspace becomes fragmented, slowing down reads/writes.
3.  **Resource Exhaustion:** Memory pressure on the etcd nodes leading to swapping.

**Why one component creates cluster-wide symptoms:**
etcd is the **single source of truth** for the entire cluster. Every time you run `kubectl`, create a pod, or update a service, the API server must write to etcd.
*   If etcd is slow, `kubectl` (which talks to the API server) feels sluggish.
*   If etcd cannot commit the "Pod Created" event, the Scheduler and Kubelet never receive the instruction to start the pod, leading to failures in creating new pods.
*   Existing pods are fine because they are already running and don't need to talk to etcd for their daily operations (unless they are updating status).

---

### Q6: Istio Canary and Circuit Breaking

**VirtualService (Traffic Split):**
```yaml
apiVersion, networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment.novamart.com
  http:
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 95
    - destination:
        host: payment-service
        subset: v2
      weight: 5
```

**DestinationRule (Subsets):**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Circuit Breaker Protection:**
If v2's error rate spikes, Istio's **Outlier Detection** (the circuit breaker) kicks in. You define a `outlierDetection` block in the `DestinationRule`. If a pod returns a 5xx error $X$ times in a row, Istio **ejects** that specific pod from the load balancing pool for a predetermined period. 

This protects the system by ensuring that the 5% of traffic sent to v2 doesn't just fail—it fails fast, and the faulty pods are removed entirely until they recover, preventing the "error storm" from affecting the overall system stability.

---

### Q7: The GitOps "Reconciliation Loop"

**What happened:**
The cluster is managed by a **GitOps Controller** (like ArgoCD or Flux).

In a GitOps model, the Git repository is the **Desired State**, and the cluster is the **Actual State**. 
1.  The developer changed the Actual State (Pod count = 10) via `kubectl edit`.
2.  The ArgoCD controller performed a "drift detection" scan and noticed: `Git (5) != Cluster (10)`.
3.  The controller automatically performed a **Sync/Reconciliation**, overwriting the manual change to bring the cluster back into alignment with the source of truth (Git).

**Why this is correct:**
This prevents "Configuration Drift." If you allow manual changes, you lose the ability to audit who changed what and you cannot reliably recreate the environment from Git in case of a disaster.

---

### Q8: The Jenkins $\rightarrow$ ArgoCD GitOps Flow

**The Complete Flow:**
1.  **Code Push:** Developer pushes code to Git.
2.  **Build (Jenkins):** Jenkins triggers a build, runs tests, and builds a Docker image.
3.  **Push Image:** Jenkins pushes the image to ECR with a unique tag (e.g., the Git SHA).
4.  **Manifest Update:** Instead of touching the cluster, Jenkins clones the **GitOps Manifest Repo** and updates the image tag in the YAML (e.g., `image: payment:v1` $\rightarrow$ `image: payment:sha-abc123`).
5.  **Commit & Push:** Jenkins commits this change back to the Manifest Repo.
6.  **Sync (ArgoCD):** ArgoCD detects the commit in the Manifest Repo and applies the change to the EKS cluster.

**Why Jenkins should NOT `kubectl apply`:**
1.  **Security:** Jenkins would need `cluster-admin` permissions, which is a huge security risk. If Jenkins is compromised, the whole cluster is gone.
2.  **Audit Trail:** `kubectl apply` is a "fire and forget" action. GitOps provides a permanent history of exactly when and why a version changed.
3.  **Drift Detection:** `kubectl apply` cannot detect if someone manually changed a setting. ArgoCD can.
4.  **Safe Rollbacks:** Rollbacks in GitOps are just `git revert` on the manifest repo, which is safer and faster than trying to remember which image tag was stable.

---

### Q9: 3 AM Node Failure Triage

**Step 1: Blast Radius Assessment (The "Panic" Phase)**
I immediately check if the 3 critical API pods are still healthy via the Load Balancer. If the pods were spread across nodes, the service might still be up. If they were all on `ip-10-0-1-50`, I have a P1 outage.

**Step 2: Node Health Diagnostics**
I attempt to determine if the node is "dead" or "unreachable":
1.  **AWS Console:** Check if the instance is `Running` or if it has failed a Status Check (System/Instance).
2.  **SSH Attempt:** Try to SSH in. If it times out, the network/kernel is dead.
3.  **Dmesg/Logs:** If I can get in, I check `dmesg` for "Kernel Panic" or "Out of Memory" (OOM) kills of the `kubelet`.

**Step 3: Recovery (Forcing Reschedule)**
Since the node is `NotReady`, Kubernetes will eventually evict the pods, but the default timeout is 5 minutes (too long for critical APIs).
1.  **Taint the Node:** If the node is unresponsive, I manually taint it to ensure no new pods land on it:
    `kubectl taint nodes ip-10-0-1-50 key=value:NoExecute`
    This forces the 15 pods to evacuate immediately and reschedule on healthy nodes.
2.  **Verify Pod Recovery:** I monitor `kubectl get pods -w` to ensure the 3 critical API pods are `Running` and `Ready` on other nodes.

**Step 4: Root Cause Analysis (Post-Recovery)**
Once the service is restored, I investigate the failed node:
*   Check CloudWatch metrics for CPU/Memory spikes.
*   Analyze the `kubelet` logs from the node's disk.
*   Check if there was a "noisy neighbor" pod that crashed the kernel.

---

# Lesson 3C: Everything I Owe — Operational Failure Modes

---

### CPU Throttling — The Silent Performance Killer

```bash
# CPU LIMITS USE CFS (Completely Fair Scheduler) BANDWIDTH CONTROL
# This is NOT "you get 1 CPU core worth of compute"
# It's: "you get X microseconds every 100ms window"

# Example:
# Container: limits.cpu = 1000m (1 core)
# CFS period: 100ms (100,000 µs)
# CFS quota: 100,000 µs per period
# Container can use 100ms of CPU time per 100ms window

# THE THROTTLING PROBLEM:
# Your app has 8 threads. All 8 burst simultaneously.
# In the first 12.5ms, all 8 threads consume 100ms of CPU time
# (8 threads × 12.5ms = 100ms quota used)
# For the remaining 87.5ms of the window → ALL THREADS FROZEN
# EVERY request arriving during those 87.5ms → queued → latency spike

# Monitoring shows: average CPU = 30%
# Reality: app is frozen 87.5% of each 100ms window during bursts
# p99 latency: through the roof
# Average latency: looks fine (because most requests hit unfrozen windows)

# HOW TO DETECT:
# On the node:
cat /sys/fs/cgroup/cpu,cpuacct/kubepods/pod<uid>/<container-id>/cpu.stat
# nr_periods: 125000        ← number of 100ms periods elapsed
# nr_throttled: 45000       ← number of periods where throttling occurred
# throttled_time: 28000000  ← total ns spent throttled

# Throttle ratio: 45000/125000 = 36% of periods had throttling
# That's BAD.

# Prometheus metrics (via cadvisor):
# container_cpu_cfs_throttled_periods_total
# container_cpu_cfs_periods_total
# Throttle ratio = throttled_periods / total_periods
# Alert when: ratio > 25% sustained for 5 minutes

# FIXES:

# Option 1: Increase CPU limits
resources:
  requests:
    cpu: 500m
  limits:
    cpu: 2000m    # 4x headroom for bursts

# Option 2: Remove CPU limits entirely
resources:
  requests:
    cpu: 500m
  # NO limits.cpu
# Container can burst to any available CPU on the node
# requests.cpu still guarantees scheduling and proportional sharing
# Under contention: CPU divided proportionally by requests
# No contention: container uses whatever is available
# Used by: Google (Borg doesn't enforce CPU limits), many SRE teams

# Option 3: Use GOMAXPROCS / thread tuning
# Reduce thread count to match CPU limit
# 1 CPU limit → GOMAXPROCS=1 (Go)
# 1 CPU limit → -XX:ActiveProcessorCount=1 (Java)
# Fewer threads = less simultaneous burst = less throttling

# THE DEBATE:
# Pro limits: prevents noisy neighbor, predictable performance
# Anti limits: CFS throttling causes more latency issues than it prevents
# Google's stance: use requests only, don't set limits
# Most companies: set limits conservatively high (2-4x requests)
# NovaMart approach: limits = 4x requests for API services,
#   no limits for batch jobs on dedicated node pools
```

---

### ContainerCreating — The Umbrella Failure State

```bash
# Pod status: ContainerCreating
# This can hide MANY different failures:

# 1. Image pull in progress (slow registry, large image)
kubectl describe pod <pod>
# Events: "Pulling image my-app:v1.2.3"
# Slow but not broken. Wait or check registry/network.

# 2. Image pull failure (about to become ImagePullBackOff)
# Events: "Failed to pull image: rpc error: code = Unknown desc = 
#   Error response from daemon: manifest not found"
# Wrong tag, wrong registry, auth failure

# 3. Volume mount failure
# Events: "Unable to attach or mount volumes: 
#   timed out waiting for the condition"
# EBS volume stuck attaching (previous node didn't detach cleanly)
# NFS server unreachable
# Secret/ConfigMap doesn't exist

# 4. CNI failure — no IP available
# Events: "Failed to create pod sandbox: 
#   rpc error: add cmd: failed to assign an IP address to container"
# VPC CNI: ENI IP pool exhausted
# Fix: prefix delegation, larger instance, or wait for IP recycling

# 5. Sandbox creation failure (container runtime issue)
# Events: "Failed to create pod sandbox: 
#   rpc error: context deadline exceeded"
# containerd/runc issue on the node
# Often fixed by: restarting containerd or replacing the node

# 6. Init container still running
# If pod has init containers, main containers stay in ContainerCreating
# until ALL init containers complete successfully
kubectl get pod <pod> -o jsonpath='{.status.initContainerStatuses}'
# Check if init container is stuck/failing

# KEY DEBUGGING COMMAND:
kubectl describe pod <pod>
# The Events section tells you EXACTLY which sub-step failed
# Always check Events, not just Status
```

---

### ImagePullBackOff — Full Debugging

```bash
kubectl describe pod <pod>
# Events:
# Failed to pull image "my-app:v1.2.3": 
#   rpc error: code = Unknown desc = Error response from daemon

# CAUSES:

# 1. Image doesn't exist:
# Wrong tag, wrong repository name, typo
# Verify: aws ecr describe-images --repository-name my-app --image-ids imageTag=v1.2.3

# 2. Authentication failure:
# ECR token expired (12-hour lifetime)
# imagePullSecrets not configured or wrong
# Cross-account ECR: repository policy missing
# Private registry: credentials wrong
# Verify:
kubectl get pod <pod> -o jsonpath='{.spec.imagePullSecrets}'
kubectl get secret <pull-secret> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# 3. Network unreachable (private cluster):
# No NAT Gateway and no VPC endpoints for ECR + S3
# Node can't reach ECR API or S3 (where layers live)
# Verify: SSH to node, curl https://api.ecr.us-east-1.amazonaws.com/

# 4. Rate limiting:
# Docker Hub: 100 pulls/6hr (anonymous), 200 pulls/6hr (authenticated)
# Events: "toomanyrequests: You have reached your pull rate limit"
# Fix: use ECR pull-through cache, or mirror images to private registry

# 5. Architecture mismatch:
# Image built for linux/amd64, node is ARM (Graviton)
# Events: "exec format error" after pull succeeds
# Verify: docker manifest inspect <image> | grep architecture

# 6. Image too large + slow network:
# 5GB image on a node with limited bandwidth
# Not technically an error, just slow
# Fix: smaller images, pre-pull DaemonSet for large images

# BACKOFF PROGRESSION:
# ImagePullBackOff follows exponential backoff:
# 10s → 20s → 40s → 80s → 160s → 300s (5 min cap)
# After fixing the issue, pod retries on the next backoff cycle
# To force immediate retry: delete and recreate the pod
```

---

### CoreDNS Operational Scaling

```bash
# DEFAULT: 2 CoreDNS pods in EKS
# At scale, this is NOT enough

# SYMPTOMS OF CoreDNS OVERLOAD:
# - DNS resolution timeouts from pods
# - dmesg: "nf_conntrack: table full" (DNS uses UDP → conntrack)
# - CoreDNS logs: "i/o timeout" when forwarding to upstream
# - Prometheus: coredns_dns_requests_total increasing, 
#   coredns_dns_responses_total not keeping up

# SCALING COREDNS:

# 1. Horizontal — more replicas:
kubectl -n kube-system edit deployment coredns
# Or better: use dns-autoscaler addon
# Scales CoreDNS pods based on node/core count in cluster
# Formula: replicas = max(ceil(cores / 256), ceil(nodes / 16))
# 100 nodes → max(ceil(400/256), ceil(100/16)) = max(2, 7) = 7 replicas

# 2. NodeLocal DNSCache (critical for production at scale):
# Runs a DNS cache daemon on EVERY node as a DaemonSet
# Pods resolve DNS against the LOCAL cache (avoid cross-node hops)
# Cache hit → instant response (no network)
# Cache miss → forwards to CoreDNS
# 
# Benefits:
# - Reduces CoreDNS load by 80-90%
# - Eliminates conntrack entries for DNS (uses TCP to CoreDNS)
# - Eliminates "DNS 5-second timeout" race condition
#   (the infamous glibc parallel A + AAAA query bug)
# - Sub-millisecond DNS for cached entries

# Enable on EKS:
# Deploy node-local-dns DaemonSet
# Configure kubelet to use node-local-dns IP (169.254.20.10)
# Pods automatically resolve against local cache

# 3. CoreDNS Corefile tuning:
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        forward . /etc/resolv.conf {
            max_concurrent 5000         # increase from default 1000
        }
        cache 30                        # cache TTL (default 30s)
        loop
        reload
        loadbalance
        # Add: autopath to reduce ndots:5 query amplification
        autopath @kubernetes
        # Without autopath: pod resolving "google.com" generates 5 queries
        # With autopath: CoreDNS short-circuits the search path
    }

# 4. Reduce ndots-related query amplification:
# Default ndots:5 means any name with < 5 dots goes through search path
# "redis-master" generates:
#   redis-master.default.svc.cluster.local
#   redis-master.svc.cluster.local
#   redis-master.cluster.local
#   redis-master.ec2.internal
#   redis-master              ← finally, the actual query
# That's 5 DNS queries for one resolution!
#
# Fix per pod:
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"         # reduce from 5 to 2
    # Names with < 2 dots go through search path
    # "redis-master" (0 dots) → search path (correct, finds service)
    # "google.com" (1 dot) → search path first, then absolute
    # "api.google.com" (2 dots) → absolute query directly (no wasted queries)
```

---

### EndpointSlice — The Scalable Endpoint

```bash
# ENDPOINTS (legacy):
# One Endpoints object per Service containing ALL pod IPs
# Service with 5000 pods → one Endpoints object with 5000 IPs
# Every kube-proxy watches ALL Endpoints
# One pod changes → entire Endpoints object updated → pushed to ALL kube-proxies
# At scale: massive watch traffic, slow updates

# ENDPOINTSLICE (default since K8s 1.21):
# Service with 5000 pods → 50 EndpointSlice objects (100 IPs each)
# When one pod changes → only ONE EndpointSlice updated
# Pushed only to kube-proxies that need it
# 100x reduction in API server and etcd load at scale

kubectl get endpointslice -n production
# NAME                   ADDRESSTYPE   PORTS   ENDPOINTS               AGE
# api-service-abc12      IPv4          8080    10.0.1.5,10.0.1.6...   5d
# api-service-def34      IPv4          8080    10.0.2.7,10.0.2.8...   5d

kubectl get endpointslice api-service-abc12 -o yaml
# Shows: individual endpoint conditions (ready, serving, terminating)
# More granular than legacy Endpoints

# WHY YOU SHOULD KNOW THIS:
# If you're debugging "Service has endpoints but traffic isn't reaching pods"
# Check EndpointSlices, not just Endpoints:
kubectl get endpointslice -l kubernetes.io/service-name=my-service
# Look for: conditions.ready = false on specific endpoints
# A pod might be in the EndpointSlice but marked not-ready
```

---

### Probe Anti-Patterns

```yaml
# ANTI-PATTERN 1: Checking dependencies in liveness probe
livenessProbe:
  httpGet:
    path: /health
    port: 8080
# /health checks: DB connection + Redis + external API
# If Redis goes down:
# → liveness fails → container restarted
# → new container starts → Redis still down → liveness fails again
# → CrashLoopBackOff for ALL pods
# → TOTAL OUTAGE of your service because Redis had a blip
# 
# FIX: liveness checks ONLY the process itself (is it responsive?)
# /healthz: return 200 if the HTTP server can respond. That's it.
# /ready: check dependencies. Readiness probe removes from traffic.

# ANTI-PATTERN 2: initialDelaySeconds too low
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5      # Java app takes 45s to start
  failureThreshold: 3
  periodSeconds: 10
# Sequence: pod starts → 5s delay → check → fail (app still booting)
# → 10s → check → fail → 10s → check → fail → RESTART
# → pod starts again → same thing → CrashLoopBackOff
# App never gets a chance to boot
#
# FIX: use startupProbe for slow starters
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30         # 30 × 10s = 300s = 5 min to start
  periodSeconds: 10
# Liveness/readiness disabled until startup probe passes

# ANTI-PATTERN 3: Probe hitting expensive endpoint
livenessProbe:
  httpGet:
    path: /api/v1/status       # runs DB query, aggregates metrics
    port: 8080
  periodSeconds: 5
# Every 5 seconds: full status check with DB query
# Under load: probe adds 20% overhead
# Probe timeout → liveness fails → restart → makes load worse
#
# FIX: dedicated lightweight health endpoint
# /healthz: return 200, no computation, < 1ms response

# ANTI-PATTERN 4: Same endpoint for liveness AND readiness
# They serve different purposes:
# Liveness: "restart me if I'm broken"
# Readiness: "stop sending traffic if I'm not ready"
# Same endpoint = same failure mode = wrong action
# Readiness failure on dependency → should remove from traffic
# If liveness uses same endpoint → restarts instead → wrong
#
# FIX: separate endpoints
# /healthz → liveness (process health only)
# /ready → readiness (process + dependencies)

# ANTI-PATTERN 5: Aggressive probe settings on high-latency apps
livenessProbe:
  timeoutSeconds: 1            # 1 second timeout
  periodSeconds: 5
  failureThreshold: 1          # ONE failure = restart
# Under GC pause or load spike, one slow response = restart
# Creates cascading restarts under load
#
# FIX: generous thresholds
livenessProbe:
  timeoutSeconds: 5
  periodSeconds: 15
  failureThreshold: 3          # 3 failures = 45s before restart
```

---

### OOMKilled vs Evicted vs Preempted vs Deleted — The Comparison

```bash
# Four ways a pod can die. Each has different cause, behavior, and response.

# 1. OOMKilled (container-level):
# CAUSE: Container exceeded its memory LIMIT
# WHO: Linux kernel OOM killer (cgroup enforcement)
# SCOPE: single container (other containers in pod may survive)
# RESTART: yes (if restartPolicy allows)
# DETECT:
kubectl describe pod <pod>
# State: Terminated
# Reason: OOMKilled
# Exit Code: 137
# FIX: increase memory limits, fix memory leak, tune JVM/runtime

# 2. Evicted (pod-level, node pressure):
# CAUSE: Node is running low on resources (memory, disk, PIDs)
# WHO: kubelet eviction manager
# SCOPE: entire pod removed from node
# RESTART: pod is NOT restarted on same node
#   Controller (Deployment/RS) creates NEW pod, scheduler places elsewhere
# DETECT:
kubectl describe pod <pod>
# Status: Failed
# Reason: Evicted
# Message: "The node was low on resource: memory"
# ORDER: BestEffort → Burstable (by overshoot) → Guaranteed
# FIX: right-size resource requests, use Guaranteed QoS for critical pods

# 3. Preempted (pod-level, scheduling priority):
# CAUSE: Higher-priority pod needs resources, no room on any node
# WHO: kube-scheduler
# SCOPE: lower-priority pods evicted to make room
# RESTART: preempted pod goes back to Pending, may schedule elsewhere
# DETECT:
kubectl get events --field-selector reason=Preempted
# FIX: assign appropriate PriorityClass to critical workloads

# 4. Deleted (pod-level, intentional):
# CAUSE: kubectl delete, scale down, rolling update, node drain
# WHO: API server / controllers
# SCOPE: specific pod
# RESTART: depends on controller
# Normal lifecycle — not an incident

# THE COMPARISON TABLE:
# ┌──────────────┬─────────────┬──────────┬─────────────────┐
# │ Type         │ Who decides  │ Why      │ Pod restarted?  │
# ├──────────────┼─────────────┼──────────┼─────────────────┤
# │ OOMKilled    │ Kernel      │ Mem limit│ Container yes   │
# │ Evicted      │ kubelet     │ Node     │ New pod, new    │
# │              │             │ pressure │ node            │
# │ Preempted    │ Scheduler   │ Priority │ Back to Pending │
# │ Deleted      │ User/       │ Intent   │ If controller   │
# │              │ Controller  │          │ exists          │
# └──────────────┴─────────────┴──────────┴─────────────────┘
```

---

### CronJob Production Safeguards

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: production
spec:
  schedule: "0 */6 * * *"              # every 6 hours
  
  concurrencyPolicy: Forbid
  # Forbid: skip if previous job still running
  # Allow: start new job even if previous running (DEFAULT — dangerous)
  # Replace: kill previous job, start new one
  
  startingDeadlineSeconds: 300
  # If CronJob controller was down and missed the schedule,
  # only start the job if it's within 300s of scheduled time
  # Without this: controller comes back up, fires ALL missed jobs at once
  # 100 missed jobs × db-backup = hammers the database
  
  successfulJobsHistoryLimit: 3        # keep 3 successful job pods
  failedJobsHistoryLimit: 5            # keep 5 failed job pods for debugging
  # Without limits: thousands of completed pods accumulate
  # Fills up etcd, clutters kubectl get pods output
  
  jobTemplate:
    spec:
      activeDeadlineSeconds: 3600      # kill job after 1 hour
      # Prevents hung jobs from running forever
      # DB backup usually takes 20 min → 1 hour is safe deadline
      # Without this: deadlocked job runs forever, blocks next schedule
      
      backoffLimit: 3                  # retry failed pods 3 times
      # Default is 6 — too many for something that runs every 6 hours
      
      ttlSecondsAfterFinished: 86400   # auto-delete job after 24 hours
      # Cleaner than relying on history limits alone
      
      template:
        spec:
          restartPolicy: OnFailure     # REQUIRED for Jobs (not Always)
          containers:
          - name: backup
            image: backup-tool:v1.0
            resources:
              requests:
                cpu: 500m
                memory: 1Gi
              limits:
                memory: 2Gi            # prevent OOM from backup spike
```

---

### Secret Rotation — The Reality

```bash
# KUBERNETES SECRETS ARE NOT AUTOMATICALLY ROTATED
# If you put a DB password in a Secret, it stays there forever
# Until someone manually updates it

# THE ROTATION CHAIN:

# 1. External secret store rotates the secret:
#    AWS Secrets Manager → automatic rotation via Lambda
#    HashiCorp Vault → dynamic secrets with TTL

# 2. External Secrets Operator syncs to K8s:
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h              # poll every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials           # K8s Secret to create/update
  data:
  - secretKey: password
    remoteRef:
      key: production/db/password  # path in AWS Secrets Manager

# 3. Pod picks up new secret:
# IF mounted as volume: kubelet updates file within ~60s
# IF mounted as env var: pod must be RESTARTED
# 
# For env vars: use Reloader or rolling restart after rotation
# For volumes: app must re-read the file (not all apps do this)

# THE GOTCHA NOBODY MENTIONS:
# Rotating a secret means there's a window where:
# - New pods get the NEW secret
# - Old pods still have the OLD secret
# - If the old secret is revoked → old pods break
#
# Solution: dual-write
# 1. Add new credential to the database (both old and new work)
# 2. Update the secret in K8s
# 3. Wait for all pods to pick up new secret
# 4. Remove old credential from database
# This is why Vault dynamic secrets are preferred — they handle this
```

---

### Resource Request Tuning — The Methodology

```bash
# THE PROBLEM:
# Developers guess resource requests
# cpu: 500m, memory: 512Mi ← where did these numbers come from?
# Usually: copied from another service, or "seemed reasonable"
# Result: 40-60% of cluster resources WASTED (reserved but unused)

# THE METHODOLOGY:

# Step 1: Deploy with VPA in recommendation mode
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Off"          # just recommend, don't touch pods

# Step 2: Wait 1-2 weeks (capture normal + peak traffic patterns)

# Step 3: Read recommendations
kubectl describe vpa api-vpa
# Target:       cpu: 125m,  memory: 256Mi    ← what the app actually uses
# Upper Bound:  cpu: 500m,  memory: 1Gi      ← peak usage estimate

# Step 4: Set requests and limits:
resources:
  requests:
    cpu: 150m          # slightly above VPA target (buffer)
    memory: 300Mi      # slightly above VPA target
  limits:
    cpu: 600m          # 4x requests (burst headroom)
    memory: 512Mi      # ~1.5-2x requests (memory spikes)

# Step 5: Monitor and iterate
# Prometheus queries:
# Actual CPU usage vs requests:
# container_cpu_usage_seconds_total / kube_pod_container_resource_requests{resource="cpu"}
# If consistently < 0.5 → over-provisioned, reduce requests
# If consistently > 0.9 → under-provisioned, increase requests

# Actual memory vs requests:
# container_memory_working_set_bytes / kube_pod_container_resource_requests{resource="memory"}

# TOOLS:
# Kubecost: shows cost per namespace/deployment, identifies waste
# Goldilocks: runs VPA in recommend mode for all deployments,
#   provides dashboard with suggested requests/limits
# kubectl-resource-recommender: CLI tool for recommendations

# THE RULE:
# Requests = what you actually need (based on data)
# Limits.memory = 1.5-2x requests (protect against leaks)
# Limits.cpu = 2-4x requests (allow bursts) or remove entirely
# Review quarterly as traffic patterns change
```

---

### etcd as a Failure Mode

```bash
# SYMPTOMS OF ETCD PROBLEMS:
# - kubectl commands hang or timeout
# - Pod scheduling delays (scheduler can't read/write etcd)
# - API server returning 500s or "etcdserver: request timed out"
# - Existing pods keep running (they don't need etcd)
# - But no NEW operations succeed

# ROOT CAUSES:

# 1. Disk I/O latency:
# etcd uses fsync for WAL writes — MUST be < 10ms
# If disk latency spikes (noisy neighbor, EBS throttling):
# etcd: "W | wal: sync duration exceeds 1s"
# FIX: use gp3 with provisioned IOPS, or io2
# EKS managed: AWS handles this (but you can hit limits at scale)

# 2. Database size / fragmentation:
# etcd has a default 2GB database size limit
# Lots of Events, Endpoints, ConfigMaps → DB grows
# Compaction removes old revisions but doesn't free disk space
# Defragmentation actually frees space but locks the member briefly
#
# Self-managed etcd:
etcdctl endpoint status --cluster
# Shows: DB size, Raft index, leader
etcdctl compact $(etcdctl endpoint status -w json | jq -r '.[] | .Status.header.revision')
etcdctl defrag --cluster
# EKS: AWS handles compaction/defrag automatically

# 3. Too many objects:
# 200 services × 50 endpoints each = 10,000 Endpoint objects
# Every change → etcd write → watch notification → all kube-proxies
# Fix: EndpointSlices (default now), reduce object churn
# Reduce CronJob history limits, Event TTL

# 4. Slow watchers:
# Controllers/operators that watch too broadly
# A badly-written operator watching ALL pods across ALL namespaces
# Generates massive watch traffic through API server to etcd
# Fix: use label selectors, namespace scopes in watches

# MONITORING:
# etcd_disk_wal_fsync_duration_seconds → must be < 10ms p99
# etcd_server_slow_apply_total → increasing = etcd can't keep up
# etcd_mvcc_db_total_size_in_bytes → approaching 2GB = danger
# apiserver_storage_objects → total objects in etcd
# apiserver_request_duration_seconds → API server latency (etcd is usually the cause)

# EKS:
# etcd is managed — you can't SSH to it
# But you CAN see symptoms: API server latency, scheduling delays
# If you suspect etcd: open AWS support ticket
# Monitor via: CloudWatch metrics for EKS control plane
```

---

### Runbook Structure — How to Write One

```markdown
# RUNBOOK: Service X Returns 502 Errors

## Severity: SEV2
## Owner: Platform Engineering
## Last Updated: 2024-01-15
## Last Tested: 2024-01-01

## Symptoms
- Grafana dashboard "Service X" shows 502 error rate > 1%
- PagerDuty alert: "service-x-502-rate-high"
- Customer reports: checkout failures

## Impact
- Checkout flow degraded
- Revenue impact: ~$5K/min during peak hours

## Quick Diagnosis (< 2 minutes)
1. Check pod health:
   ```
   kubectl get pods -n production -l app=service-x
   ```
   - All pods Running and Ready (1/1)?
   - Any CrashLoopBackOff or Pending?

2. Check endpoints exist:
   ```
   kubectl get endpoints service-x -n production
   ```
   - If empty → readiness probe failing

3. Check recent deployments:
   ```
   kubectl rollout history deployment/service-x -n production
   ```
   - Was there a deploy in the last 30 min?

## Resolution Steps

### If pods are crashing:
1. Check logs: `kubectl logs -l app=service-x --previous`
2. If OOMKilled: increase memory limits (runbook-link)
3. If config error: rollback: `kubectl rollout undo deployment/service-x`

### If pods are healthy but 502 persists:
1. Check upstream dependency:
   ```
   kubectl exec <pod> -- curl -v http://payment-service:8080/health
   ```
2. If payment-service down → escalate to Payment team
3. Check Istio circuit breaker: 
   ```
   istioctl proxy-config cluster <pod> | grep payment
   ```

### If recent deployment:
1. Rollback immediately: `kubectl rollout undo deployment/service-x`
2. Verify recovery
3. Investigate the failed deployment after service is stable

## Escalation
- If not resolved in 15 min → page on-call SRE lead
- If revenue impact > $50K → escalate to SEV1, page VP Engineering

## Post-Incident
- Write postmortem within 48 hours
- Link to template: [postmortem-template]
```

---

# 📝 GRADING — Lesson 4 Retention (Your Answers: Q6-Q9)

## Q6 (Istio Canary): ✅ Correct

VirtualService with 95/5 split and DestinationRule with subsets. The circuit breaker explanation via outlier detection is right — consecutive 5xx errors trigger pod ejection from the load balancing pool.

One gap: you described outlier detection conceptually but didn't include the actual configuration:

```yaml
outlierDetection:
  consecutive5xxErrors: 3
  interval: 10s
  baseEjectionTime: 30s
  maxEjectionPercent: 50
```

Without these numbers, the circuit breaker doesn't exist. Istio doesn't enable outlier detection by default — you must explicitly configure it in the DestinationRule.

---

## Q7 (GitOps Drift): ✅ Correct

ArgoCD `selfHeal: true` detected drift and reconciled. Git is source of truth. Manual changes get reverted. This is correct behavior, not a bug.

---

## Q8 (Jenkins → ArgoCD Flow): ✅ Complete

All 6 steps correct. The four reasons Jenkins shouldn't `kubectl apply` directly are solid: security (cluster-admin creds), audit trail, drift detection, safe rollbacks.

---

## Q9 (3 AM Node Failure): ✅ Good Triage

Blast radius assessment → node health diagnostics → forced reschedule → root cause analysis. Correct order of priorities: restore service first, investigate later.

One correction: you used `kubectl taint` with `NoExecute` to force pod evacuation. That works, but the standard approach is:

```bash
kubectl drain ip-10-0-1-50 --ignore-daemonsets --delete-emptydir-data --grace-period=30
```

Drain is preferred because:
- It respects PDBs (taint with NoExecute doesn't)
- It gives pods graceful shutdown time
- It's the documented procedure that other engineers will recognize

If the node is truly unresponsive and drain hangs:
```bash
kubectl delete node ip-10-0-1-50
# Cluster autoscaler/Karpenter replaces it
# Pods on the dead node: controller recreates them elsewhere
# after node-controller marks them for deletion (5 min default)
```

---
