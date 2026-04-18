# PHASE 2 — Lesson 2: Docker Internals

Docker is not a VM. If you think it's a VM, everything you build will be wrong. Docker is **a process running on the host kernel with resource isolation.** That's it. Understanding what creates that isolation is the difference between using Docker and understanding Docker.

---

### What Docker Actually Is — Linux Primitives

```
Docker containers are built on THREE Linux kernel features:

1. NAMESPACES  — Isolation (what a process can SEE)
2. CGROUPS     — Resource limits (what a process can USE)
3. UNION FS    — Layered filesystem (how images are built)

There is no "container" primitive in the Linux kernel.
A container is just a process with namespaces + cgroups applied.
```

### Namespaces — What the Process Can See

```bash
# Linux namespaces isolate different aspects of the system.
# Each container gets its own set of namespaces:

PID namespace:
  - Container sees its own process tree
  - PID 1 inside container ≠ PID 1 on host
  - Container process might be PID 3847 on host, PID 1 inside container
  - ps aux inside container only shows container processes

NET namespace:
  - Container gets its own network stack
  - Own interfaces, IP addresses, routing table, iptables rules
  - localhost inside container is NOT host's localhost
  - This is how each container gets its own IP

MNT namespace:
  - Container gets its own filesystem mount tree
  - Can't see host's /etc/passwd (unless you mount it in)
  - Container's / is the image's root filesystem

UTS namespace:
  - Container gets its own hostname
  - hostname inside container ≠ host's hostname

IPC namespace:
  - Isolates inter-process communication
  - Shared memory, semaphores, message queues
  - Container A can't access Container B's shared memory

USER namespace:
  - Maps container UIDs to host UIDs
  - Root inside container (UID 0) can map to non-root on host (UID 100000)
  - Rootless containers use this
  - Security: even if container is compromised, attacker isn't root on host

# See namespaces of a running container:
docker inspect --format '{{.State.Pid}}' <container>
# Returns host PID, e.g., 3847

ls -la /proc/3847/ns/
# lrwxrwxrwx cgroup -> cgroup:[4026532483]
# lrwxrwxrwx ipc -> ipc:[4026532411]
# lrwxrwxrwx mnt -> mnt:[4026532409]
# lrwxrwxrwx net -> net:[4026532414]
# lrwxrwxrwx pid -> pid:[4026532412]
# lrwxrwxrwx user -> user:[4026531837]
# lrwxrwxrwx uts -> uts:[4026532410]
# Each number is a namespace ID — different from host = isolated
```

### Cgroups — What the Process Can Use

```bash
# Control Groups limit, account, and isolate resource usage.

# CPU:
docker run --cpus=2 myapp
# Container can use at most 2 CPU cores worth of time
# Implemented via: cpu.cfs_quota_us / cpu.cfs_period_us
# 2 CPUs = quota 200000 / period 100000

docker run --cpu-shares=512 myapp
# Relative weight (default 1024)
# Only matters under contention — if CPU is idle, container uses all it wants
# If two containers compete: 512 vs 1024 → first gets 1/3, second gets 2/3

# MEMORY:
docker run --memory=512m myapp
# Hard limit: 512MB
# If container exceeds this → OOM killed
# This is why Java apps in containers are notorious:
#   JVM sees HOST memory (8GB), allocates 2GB heap
#   Container limit is 512MB → OOM killed
#   Fix: -XX:MaxRAMPercentage=75.0 (JVM reads cgroup limit)
#   Or: -Xmx384m (explicit, leave room for non-heap)

docker run --memory=512m --memory-swap=1g myapp
# 512MB RAM + 512MB swap = 1GB total
# --memory-swap = total (RAM + swap), not just swap

docker run --memory=512m --memory-swap=512m myapp
# Disable swap entirely for this container (swap = 0)

# OOM PRIORITY:
docker run --oom-kill-disable myapp
# DANGEROUS: if OOM, host might freeze because it can't free memory
# Almost never use this

docker run --oom-score-adj=-500 myapp
# Lower score = less likely to be OOM killed
# Range: -1000 (never kill) to 1000 (kill first)

# DISK I/O:
docker run --device-write-bps /dev/sda:10mb myapp
# Limit write speed to 10MB/s on /dev/sda

# PIDs:
docker run --pids-limit=100 myapp
# Container can create at most 100 processes
# Prevents fork bombs

# VIEW CGROUP LIMITS:
# cgroup v2 (modern):
cat /sys/fs/cgroup/docker/<container-id>/memory.max
cat /sys/fs/cgroup/docker/<container-id>/cpu.max

# cgroup v1 (legacy):
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_quota_us
```

---

### Container Networking — How Docker Connects Things

```bash
# Docker creates several network drivers:

# BRIDGE (default):
docker network ls
# NETWORK ID    NAME     DRIVER    SCOPE
# abc123        bridge   bridge    local

# When you run a container:
# 1. Docker creates a veth pair
# 2. One end in container's NET namespace (eth0 inside container)
# 3. Other end attached to docker0 bridge on host
# 4. Container gets IP from bridge subnet (172.17.0.0/16 default)

# Container-to-container on same bridge:
# Goes through docker0 bridge — direct L2 switching
# Same as Kubernetes pod-to-pod on same node

# Container-to-outside:
# iptables MASQUERADE rule on host NATs container IP to host IP
# This is why containers can reach the internet but need -p to be reached

# PORT MAPPING:
docker run -p 8080:80 nginx
# Host port 8080 → Container port 80
# Docker creates iptables DNAT rule:
# -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80

# HOST NETWORK:
docker run --network=host nginx
# Container shares the host's network namespace entirely
# No isolation, no port mapping needed
# Container's localhost IS host's localhost
# Use case: maximum network performance, tools that need host networking
# Trade-off: no network isolation, port conflicts with host

# NONE:
docker run --network=none myapp
# No networking at all. Completely isolated.
# Use case: security-sensitive batch processing

# CUSTOM BRIDGE:
docker network create --driver bridge my-net
docker run --network=my-net --name api api-image
docker run --network=my-net --name db postgres
# Containers on same custom network can reach each other BY NAME
# api can curl http://db:5432
# Docker's embedded DNS resolves container names
# DEFAULT bridge does NOT have DNS resolution — only custom bridges

# MACVLAN:
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 my-macvlan
# Container gets a real IP on the physical network
# Appears as a separate physical device to the network
# No NAT, no port mapping — direct L2 access
# Use case: legacy apps that need to be on the LAN
```

---

### Union Filesystem — Image Layers

```bash
# Docker images are built in LAYERS. Each layer is read-only.
# A container adds a thin WRITABLE layer on top.

# Example Dockerfile:
FROM ubuntu:22.04           # Layer 1: base OS (~80MB)
RUN apt-get update          # Layer 2: package index (~40MB)
RUN apt-get install -y curl # Layer 3: curl binary (~5MB)
COPY app.py /app/           # Layer 4: your code (~1KB)
CMD ["python", "/app/app.py"]

# The image is a STACK of these layers:
# ┌─────────────────────────┐
# │ Layer 4: COPY app.py    │ (read-only)
# ├─────────────────────────┤
# │ Layer 3: install curl   │ (read-only)
# ├─────────────────────────┤
# │ Layer 2: apt-get update │ (read-only)
# ├─────────────────────────┤
# │ Layer 1: ubuntu:22.04   │ (read-only)
# └─────────────────────────┘

# When you run a container:
# ┌─────────────────────────┐
# │ CONTAINER LAYER         │ (read-write)  ← writes go here
# ├─────────────────────────┤
# │ Layer 4: COPY app.py    │ (read-only)
# ├─────────────────────────┤
# │ ...                     │
# └─────────────────────────┘

# COPY-ON-WRITE:
# When container modifies a file from a lower layer:
# 1. File is COPIED from read-only layer to writable layer
# 2. Modification happens on the COPY
# 3. Original layer is unchanged
# This is why container writes are slower than host writes

# VIEW LAYERS:
docker history myimage
# IMAGE          CREATED        SIZE    COMMENT
# abc123         2 min ago      1KB     COPY app.py
# def456         2 min ago      5MB     apt-get install curl
# ghi789         2 min ago      40MB    apt-get update
# jkl012         3 weeks ago    80MB    ubuntu:22.04

# LAYER SHARING:
# If 10 containers use ubuntu:22.04 base:
# ubuntu:22.04 layer stored ONCE on disk
# Each container has its own thin writable layer
# 10 containers ≠ 10× the disk space

# STORAGE DRIVERS:
# overlay2: default on modern Linux. Best performance.
# devicemapper: legacy, avoid
# btrfs/zfs: if host uses these filesystems
docker info | grep "Storage Driver"
# Storage Driver: overlay2
```

---

### Dockerfile — Production Best Practices

```dockerfile
# BAD — every mistake a junior makes:
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y python3 python3-pip
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 8080
CMD ["python3", "app.py"]

# PROBLEMS:
# 1. "latest" tag — build is non-reproducible
# 2. Separate RUN for update and install — cache invalidation issue
# 3. COPY . before pip install — any code change invalidates pip cache
# 4. Running as root (default)
# 5. No .dockerignore — copies .git, node_modules, etc.
# 6. Full OS base image (~80MB of unnecessary stuff)
# 7. No health check
# 8. pip installs dev dependencies too

# GOOD — production-grade:
FROM python:3.11-slim AS builder
# Pinned version, slim base (not alpine — glibc issues with Python)

WORKDIR /app

# Copy dependency files FIRST (cache optimization)
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt
# --no-cache-dir: don't store pip cache in image (saves space)
# --user: install to ~/.local (no root needed later)

# Copy application code AFTER dependencies
COPY . .

# --- MULTI-STAGE BUILD ---
FROM python:3.11-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy ONLY the installed packages and app code from builder
COPY --from=builder /root/.local /home/appuser/.local
COPY --from=builder /app .

# Set PATH for user-installed packages
ENV PATH=/home/appuser/.local/bin:$PATH

# Switch to non-root user
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["python3", "app.py"]
```

### Multi-Stage Builds — Why They Matter

```dockerfile
# WITHOUT multi-stage:
# Build tools (gcc, make, pip) stay in the final image
# Image size: 800MB+
# Attack surface: huge (compiler, package manager, headers)

# WITH multi-stage:
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server

FROM gcr.io/distroless/static-debian12
# Distroless: no shell, no package manager, no utilities
# Contains ONLY your binary and its dependencies
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]

# Result:
# Builder stage: ~1.2GB (Go toolchain + source)
# Final image: ~15MB (just the binary)
# No shell = attacker can't exec into container and poke around

# SCRATCH — the absolute minimum:
FROM scratch
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
# scratch = literally nothing. 0 bytes base.
# Only works for statically compiled binaries (Go, Rust)
# No libc, no /etc/passwd, no /tmp, nothing.
```

### .dockerignore — Mandatory

```bash
# .dockerignore (same syntax as .gitignore):
.git
.gitignore
Dockerfile
docker-compose*.yml
README.md
docs/
*.md
node_modules
__pycache__
*.pyc
.env
.env.*
*.pem
*.key
.terraform
*.tfstate*
coverage/
.nyc_output
dist/
build/

# WHY:
# COPY . /app copies EVERYTHING in the build context
# Without .dockerignore:
#   .git directory → 500MB of history in your image
#   node_modules → 300MB of deps you'll npm install anyway
#   .env → secrets baked into the image layer (FOREVER)
#   .terraform → state files with secrets
#
# Docker sends the ENTIRE build context to the daemon before building
# 1GB build context = 1GB transferred before first instruction runs
```

---

### Image Tagging and Registry

```bash
# IMAGE NAMING:
# registry/repository:tag
# docker.io/library/nginx:1.25    (Docker Hub, official)
# 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3
# gcr.io/my-project/my-app:latest

# TAGGING STRATEGY:
# NEVER use :latest in production
# latest doesn't mean "newest" — it means "no tag specified"
# Two deployments with :latest can run DIFFERENT images

# Good tagging:
docker build -t my-app:v1.2.3 .              # semver
docker build -t my-app:abc1234 .              # git SHA
docker build -t my-app:v1.2.3-abc1234 .       # both (best)
docker build -t my-app:20240115-abc1234 .      # date + SHA

# Immutable tags:
# ECR supports image tag immutability:
# Once v1.2.3 is pushed, it can NEVER be overwritten
# This prevents "I pushed a fix to the same tag" chaos

# Tag + push:
docker tag my-app:v1.2.3 123456.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3
docker push 123456.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3

# ECR login:
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456.dkr.ecr.us-east-1.amazonaws.com
```

---

### Docker Compose — Multi-Container Local Development

```yaml
# docker-compose.yml
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: development          # multi-stage: use dev stage
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
      - REDIS_URL=redis://cache:6379
    volumes:
      - ./src:/app/src             # live reload in development
    depends_on:
      db:
        condition: service_healthy  # wait for health check, not just "started"
    networks:
      - backend
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data    # named volume = persistent
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # seed data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - backend

  cache:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    networks:
      - backend

volumes:
  pgdata:                          # survives container restarts
  redisdata:

networks:
  backend:
    driver: bridge

# COMMANDS:
# docker compose up -d              start all services
# docker compose down               stop and remove containers
# docker compose down -v            stop, remove containers AND volumes
# docker compose logs -f api        tail logs for api service
# docker compose exec db psql -U user myapp   exec into db
# docker compose build --no-cache   rebuild without layer cache
# docker compose ps                 list running services
```

---

### Docker Security — What Actually Matters

```bash
# 1. NEVER RUN AS ROOT
# Default: container processes run as root (UID 0)
# If container is compromised → attacker is root inside container
# With kernel exploits → root on HOST
# Fix: USER directive in Dockerfile + runAsNonRoot in K8s

# 2. READ-ONLY FILESYSTEM
docker run --read-only myapp
# Container can't write to filesystem
# App needs /tmp? Mount a tmpfs:
docker run --read-only --tmpfs /tmp myapp

# 3. DROP CAPABILITIES
# Containers get a subset of Linux capabilities by default
# Drop everything unnecessary:
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
# Only capability: bind to ports < 1024
# No: CAP_SYS_ADMIN, CAP_NET_RAW, CAP_SYS_PTRACE, etc.

# 4. NO PRIVILEGE ESCALATION
docker run --security-opt=no-new-privileges myapp
# Prevents setuid binaries from escalating privileges

# 5. SCAN IMAGES
# Trivy (most popular):
trivy image my-app:v1.2.3
# Scans for: CVEs in OS packages, language dependencies, misconfigs
# Integrate into CI: fail pipeline if HIGH/CRITICAL CVEs found

# Grype (Anchore):
grype my-app:v1.2.3

# 6. DON'T MOUNT DOCKER SOCKET
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
# This gives the container FULL CONTROL over Docker on the host
# Can start privileged containers, mount host filesystem, etc.
# Equivalent to giving the container root on the host
# Only acceptable for: CI runners (with caution), monitoring tools
# Even then: prefer alternatives (Kaniko for builds, cri-dockerd)

# 7. IMAGE PROVENANCE
# Use digest-pinned images in production:
FROM python:3.11-slim@sha256:abc123def456...
# Tag can be overwritten. Digest cannot.
# Guarantees you're running exactly the image you tested.
```

---

### Production Scenarios

#### Scenario 1: Container OOM Killed Repeatedly

```bash
# Symptoms:
# Container restarts every 10-15 minutes
# docker inspect: "OOMKilled": true
# Application: Java Spring Boot

# Investigation:
docker stats <container>
# CONTAINER    CPU%    MEM USAGE / LIMIT
# abc123       12%     490MiB / 512MiB     ← approaching limit

# Root cause:
# Container has --memory=512m
# JVM doesn't know about cgroup limits (older JVM)
# JVM default heap = 1/4 of PHYSICAL memory = 2GB on an 8GB host
# JVM allocates 2GB → cgroup kills it at 512MB

# Fix:
# Java 11+: JVM reads cgroup limits automatically IF:
# -XX:+UseContainerSupport (default: enabled since Java 10)
#
# Set heap as percentage of container memory:
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"
CMD ["java", "$JAVA_OPTS", "-jar", "app.jar"]
# 75% of 512MB = 384MB heap
# Remaining 128MB for: metaspace, threads, native memory, OS

# Or explicit:
ENV JAVA_OPTS="-Xmx384m -Xms256m"

# ALSO CHECK:
# - Is native memory leaking? (JNI, NIO direct buffers)
# - Thread count × stack size (default 1MB per thread)
#   200 threads = 200MB just for stacks
# - Metaspace unbounded? Add -XX:MaxMetaspaceSize=128m
```

#### Scenario 2: Build Takes 20 Minutes — Layer Cache Busted Every Time

```bash
# Symptoms:
# Docker build takes 20 min
# pip install runs from scratch every build
# Even when only app code changed

# Root cause — BAD Dockerfile:
FROM python:3.11-slim
COPY . /app                     # ← THIS LINE
WORKDIR /app
RUN pip install -r requirements.txt
# ANY file change in the project invalidates COPY . /app
# Which invalidates ALL subsequent layers
# pip install runs from scratch every time

# Fix — order by change frequency:
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .         # rarely changes
RUN pip install -r requirements.txt  # cached unless requirements.txt changed
COPY . .                        # frequently changes, but pip is already cached

# ADDITIONAL OPTIMIZATIONS:
# 1. Use BuildKit (default in Docker 23+):
DOCKER_BUILDKIT=1 docker build .
# Parallel layer building, better caching, secret mounts

# 2. Cache mounts (BuildKit):
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
# pip cache persists ACROSS builds
# Even if requirements.txt changes, cached packages reused

# 3. .dockerignore to reduce build context transfer time

# Result: 20 min → 30 seconds for code-only changes
```

#### Scenario 3: Image Size Exploded to 2GB

```bash
# Investigation:
docker history my-app:latest --no-trunc
# Look for the largest layers

# Common causes:

# 1. Build tools left in final image
# Fix: multi-stage build (copy only artifacts)

# 2. apt-get cache not cleaned:
# BAD:
RUN apt-get update
RUN apt-get install -y curl
# Two layers. apt cache in layer 1 can't be removed in layer 2.
# Layers are immutable — deleting in a later layer just adds a whiteout file

# GOOD:
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
# Single layer. Cache cleaned in same layer = actually removed.

# 3. Unnecessary files copied (no .dockerignore)
# .git alone can be 500MB+

# 4. Wrong base image:
# ubuntu:22.04 → 80MB
# python:3.11 → 900MB (includes build tools, docs)
# python:3.11-slim → 150MB
# python:3.11-alpine → 50MB (but: musl libc issues, slow pip compiles)
# distroless → ~20MB
# scratch → 0MB (static binaries only)

# Check image size breakdown:
docker image inspect my-app:latest --format='{{.Size}}'
# Or use dive tool:
dive my-app:latest
# Interactive layer-by-layer analysis showing wasted space
```

#### Scenario 4: "Works on My Machine" — Container Runs Locally, Crashes in Production

```bash
# Symptoms:
# App works perfectly in docker-compose locally
# Crashes immediately in EKS with "Permission denied" or "read-only filesystem"

# Common causes:

# 1. Running as root locally, non-root in K8s:
# K8s pod security:
# runAsNonRoot: true
# runAsUser: 1000
# Dockerfile uses root → K8s forces UID 1000 → can't read files owned by root
# Fix: USER directive in Dockerfile, chown files appropriately

# 2. K8s sets readOnlyRootFilesystem: true
# App tries to write to /tmp or /var/log → permission denied
# Fix: mount emptyDir volumes for writable paths

# 3. Resource limits in K8s but not in docker-compose
# docker-compose: no memory limit → app uses 2GB happily
# K8s: resources.limits.memory: 512Mi → OOM killed

# 4. Different image
# Local: :latest (built yesterday)
# K8s: :latest (pulled from registry last week — cached on node)
# Fix: use immutable tags, set imagePullPolicy: Always or use digests

# 5. Network differences
# docker-compose: containers on same bridge, DNS by container name
# K8s: services, DNS is svc.namespace.svc.cluster.local
# Hardcoded localhost:5432 in compose → needs service DNS in K8s
```

---

### Container Runtime — Beyond Docker

```bash
# Docker is ONE container runtime. Kubernetes doesn't require Docker.

# THE STACK:
# High-level runtime:  Docker Engine, Podman, nerdctl
# Low-level runtime:   containerd, CRI-O
# Execution runtime:   runc, crun, gVisor, Kata Containers

# CONTAINERD:
# What Docker Engine uses under the hood
# Kubernetes talks to containerd directly (via CRI)
# Docker was removed from Kubernetes in v1.24 ("dockershim removal")
# Your images still work — they're OCI standard
# docker build → produces OCI image → runs on containerd, CRI-O, anything

# PODMAN:
# Docker-compatible CLI (podman build, podman run)
# Daemonless — no docker daemon running as root
# Rootless by default
# Can generate Kubernetes YAML from running containers:
podman generate kube my-container > pod.yaml

# IMAGE BUILDING ALTERNATIVES (for CI without Docker daemon):
# Kaniko: builds images inside Kubernetes pods (no Docker socket needed)
# Buildah: builds OCI images without a daemon
# BuildKit: Docker's next-gen builder (standalone mode available)

# WHY THIS MATTERS:
# CI pipelines that mount /var/run/docker.sock are a security risk
# Kaniko builds inside a container without Docker socket
# Standard approach for building images in Kubernetes-based CI
```

---

# 🔍 SELF-EVALUATION — Lesson 2

| Sub-Topic | Covered |
|-----------|---------|
| Namespaces (PID, NET, MNT, UTS, IPC, USER) | ✅ |
| Cgroups (CPU, memory, I/O, PIDs) | ✅ |
| JVM + cgroup OOM issue | ✅ |
| Container networking (bridge, host, none, macvlan, custom) | ✅ |
| Docker DNS (custom bridge only) | ✅ |
| Union filesystem / layers / copy-on-write | ✅ |
| Dockerfile best practices (layer ordering, multi-stage) | ✅ |
| Multi-stage builds (builder pattern, distroless, scratch) | ✅ |
| .dockerignore | ✅ |
| Image tagging strategy (never :latest, immutable tags, digests) | ✅ |
| Docker Compose (health checks, depends_on condition, volumes) | ✅ |
| Docker security (non-root, read-only, capabilities, no-new-privileges) | ✅ |
| Image scanning (Trivy, Grype) | ✅ |
| Docker socket danger | ✅ |
| Production: OOM killed (JVM cgroup) | ✅ |
| Production: slow builds (cache busting) | ✅ |
| Production: image size explosion | ✅ |
| Production: works locally crashes in K8s | ✅ |
| Container runtimes (containerd, CRI-O, Podman, Kaniko) | ✅ |
| Dockershim removal from K8s | ✅ |

**Not covered:** Docker volume drivers (local, NFS, EBS), ENTRYPOINT vs CMD edge cases (exec form vs shell form), `docker system prune` and disk management, buildx for multi-architecture images (ARM vs AMD64), image signing (Cosign/Notary), Docker content trust, init processes (tini/dumb-init for zombie reaping). Some of these will come up in K8s lessons or security phase.

**Self-Score: 8.5/10** — Solid coverage of what a Senior DevOps engineer needs daily. The gaps are either niche (volume drivers, content trust) or better covered in later phases (image signing in Security).

---

# 📋 QUICK REFERENCE — Docker Internals

```
CONTAINERS ARE:
  Process + namespaces (isolation) + cgroups (limits) + union FS (layers)
  NOT VMs. Share host kernel. No hypervisor.

NAMESPACES:
  PID: own process tree       NET: own network stack
  MNT: own filesystem         UTS: own hostname
  IPC: own shared memory      USER: UID remapping

CGROUPS:
  --cpus=2            hard CPU limit
  --cpu-shares=512    relative weight (contention only)
  --memory=512m       hard memory limit (OOM kill on exceed)
  --pids-limit=100    fork bomb protection
  JVM FIX: -XX:MaxRAMPercentage=75.0

NETWORKING:
  bridge (default): NAT, port mapping, veth pairs
  host: no isolation, max performance
  custom bridge: DNS resolution by container name
  macvlan: real LAN IP, no NAT

LAYERS:
  Each Dockerfile instruction = one layer (read-only)
  Container adds writable layer on top (copy-on-write)
  Layer order matters: least-changing first, most-changing last
  Clean up in SAME RUN instruction (layers are immutable)

DOCKERFILE RULES:
  Pin versions (no :latest)
  Multi-stage builds (separate build from runtime)
  COPY dependency files before code (cache optimization)
  USER nonroot (never run as root)
  --no-install-recommends + rm cache in same RUN
  .dockerignore (mandatory)
  HEALTHCHECK directive

SECURITY:
  Non-root user (USER directive)
  --read-only + tmpfs for writable paths
  --cap-drop=ALL + add only what's needed
  --security-opt=no-new-privileges
  Scan with Trivy/Grype in CI
  Never mount Docker socket unless absolutely necessary
  Use digest-pinned images in production

IMAGE SIZE:
  scratch < distroless < alpine < slim < full
  Multi-stage: build in fat image, copy to thin image
  dive tool for layer analysis

RUNTIMES:
  Docker Engine → containerd → runc (the chain)
  K8s 1.24+: no Docker, uses containerd/CRI-O directly
  CI builds: Kaniko (no Docker socket), Buildah (daemonless)
```

---

# 📝 Retention Questions — Lesson 2

**Q1:** A Java Spring Boot container with `--memory=512m` keeps getting OOM killed. `docker stats` shows memory usage at 490MB just before the kill. The developer says "my heap is only 256MB." Explain what's consuming the remaining ~234MB and how to fix it.

**Q2:** A developer's Dockerfile runs `apt-get update` in one `RUN` instruction and `apt-get install` in another. They then add `RUN rm -rf /var/lib/apt/lists/*` as a third instruction. The image is still 200MB larger than expected. Explain why the cleanup doesn't work and how to fix it.

**Q3:** Your team runs 50 containers from the same base image on one host. A new engineer says "that must use 50x the disk space." Correct them and explain the actual disk usage model, including what happens when a container writes a file.

**Q4:** In production, your EKS pod has `readOnlyRootFilesystem: true` and `runAsNonRoot: true`. The application needs to write temporary files and log to `/var/log/app/`. It works locally in Docker but crashes in EKS. Explain the full chain of failures and the fixes.

**Go.** 🎯

### Q1: The JVM Non-Heap Memory Gap

**The Explanation:**
The developer is confusing **JVM Heap Memory** with the **Process Resident Set Size (RSS)**. 

The Heap (`-Xmx`) is only one part of a Java process's memory footprint. The remaining ~234MB is consumed by **Non-Heap Memory**, which includes:
1.  **Metaspace:** Where the JVM stores class definitions and metadata.
2.  **Code Cache:** Where the JIT (Just-In-Time) compiler stores compiled native code for performance.
3.  **Thread Stacks:** Every single thread created by the app (including Spring's worker threads and GC threads) has its own stack (usually 1MB per thread). 64 threads = 64MB.
4.  **Direct Buffer Memory:** Used by NIO (Non-blocking I/O) for high-performance network/disk access, which bypasses the heap.
5.  **Native Memory:** Memory used by the JVM itself (C++ code), shared libraries, and the OS overhead.

When the container reaches its 512MB limit, the Linux kernel doesn't care that the Java heap is only at 256MB; it sees the total process memory (Heap + Non-Heap) has hit the limit and triggers the OOM Killer.

**The Fix:**
Stop using hard-coded heap limits (like `-Xmx256m`) and switch to **percentage-based limits**.
Use the flag: `-XX:MaxRAMPercentage=75.0`

This tells the JVM to calculate the heap size based on the container's actual memory limit (512MB), leaving 25% (128MB) for the non-heap overhead, which is a safer and more flexible approach for containers.

---

### Q2: The Docker Layering Trap

**The Explanation:**
The cleanup doesn't work because of how **Docker Layers** and the **Union File System** function.

Each `RUN` instruction in a Dockerfile creates a new, immutable read-only layer.
1.  **Layer 1 (`apt-get update`):** Downloads the package indices. These are written to `/var/lib/apt/lists/`. This layer is now permanently saved as part of the image.
2.  **Layer 2 (`apt-get install`):** Installs the software. This creates a new layer.
3.  **Layer 3 (`rm -rf`):** Deletes the indices. However, because Layer 1 is immutable, Docker cannot actually "delete" the data from it. Instead, it adds a **"whiteout" marker** in Layer 3 that tells the container: *"Ignore the files in Layer 1 at this path."*

The files are hidden from the running container, but they still exist in the underlying image layers, which is why the image remains 200MB larger.

**The Fix:**
You must perform the update, the installation, and the cleanup in **one single `RUN` instruction**. This ensures the indices are created and deleted within the same layer, so they are never committed to the final image.

```dockerfile
RUN apt-get update && \
    apt-get install -y my-package && \
    rm -rf /var/lib/apt/lists/*
```

---

### Q3: Layer Sharing and Copy-on-Write (CoW)

**The Correction:**
The engineer is wrong. 50 containers from the same image do **not** use 50x the disk space. They use:
$\text{(Size of Base Image)} + \text{(Sum of all unique changes made by each container)}$

**The Actual Disk Usage Model:**
Docker uses a **Union File System**. The base image consists of a stack of **read-only layers**. When you start 50 containers, they all share the *exact same* read-only layers on the disk. No matter how many containers you run, the base image is stored only once.

**What happens when a container writes a file?**
Docker uses a mechanism called **Copy-on-Write (CoW)**:
1.  If a container needs to **read** a file from the base image, it reads it directly from the shared read-only layer.
2.  If a container needs to **modify** a file from the base image, Docker first **copies** that file from the read-only layer up into the container's own thin, writable "Container Layer."
3.  The modification is performed on the copy in the writable layer.

The original file in the base image remains untouched, and other containers still see the original version. Only the specific change is stored on the disk.

---

### Q4: The Read-Only Root Filesystem Crash

**The Full Chain of Failures:**
1.  **`readOnlyRootFilesystem: true`**: This tells the container runtime to mount the entire root partition (`/`) as read-only. Since `/var/log/app/` is a directory within the root partition, any attempt by the app to write a log file results in an immediate `Read-only file system` error, causing the app to crash.
2.  **`runAsNonRoot: true`**: Even if the filesystem were writable, the process is running as a non-root user. In most base images, `/var/log` is owned by `root:root` with permissions that prevent non-root users from creating new directories or files.
3.  **Local vs. EKS:** Locally, the developer likely didn't enable `readOnlyRootFilesystem` or `runAsNonRoot`, so the app had root access to a writable disk.

**The Fixes:**
You cannot write to a read-only root filesystem, so you must provide a **writable volume** specifically for those paths.

1.  **Use an `emptyDir` Volume:** Create a temporary volume that exists only for the lifetime of the pod.
    ```yaml
    volumes:
      - name: app-logs
        emptyDir: {}
    ```
2.  **Mount the Volume:** Map that volume specifically to the path the app needs.
    ```yaml
    volumeMounts:
      - name: app-logs
        mountPath: /var/log/app/
    ```
3.  **Set Security Context:** Ensure the volume is owned by the non-root user using `fsGroup`.
    ```yaml
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      fsGroup: 1000 # Ensures the pod has permission to write to the volume
    ```


# PHASE 2 — Lesson 2B: Docker Production Deep Dive (The Missing Half)

---

### ENTRYPOINT vs CMD — The PID 1 Problem

```dockerfile
# TWO FORMS:

# SHELL FORM (string):
CMD echo "hello"
# Actually runs: /bin/sh -c 'echo "hello"'
# PID 1 = /bin/sh (NOT your process)
# Your process is a CHILD of sh
# SIGTERM sent to PID 1 (sh) → sh does NOT forward it to your process
# Container doesn't gracefully shut down → waits 10s → SIGKILL
# YOUR APP NEVER RECEIVES SIGTERM

# EXEC FORM (array):
CMD ["echo", "hello"]
# Runs: echo "hello" directly
# PID 1 = echo (YOUR process)
# SIGTERM goes directly to your process
# Graceful shutdown works

# ALWAYS USE EXEC FORM IN PRODUCTION

# ENTRYPOINT vs CMD:
# ENTRYPOINT = the executable (fixed)
# CMD = default arguments (overridable)

ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
# docker run myapp → python app.py --port 8080
# docker run myapp --port 9090 → python app.py --port 9090
# CMD is replaced, ENTRYPOINT stays

# Only CMD:
CMD ["python", "app.py"]
# docker run myapp → python app.py
# docker run myapp /bin/sh → /bin/sh (CMD completely replaced)

# Only ENTRYPOINT:
ENTRYPOINT ["python", "app.py"]
# docker run myapp → python app.py
# docker run myapp --debug → python app.py --debug (args appended)
# Can't override without --entrypoint flag

# PRODUCTION PATTERN:
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD []
# Or with a wrapper script:
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["start"]
# docker run myapp start → /docker-entrypoint.sh start
# docker run myapp migrate → /docker-entrypoint.sh migrate
```

### The PID 1 Zombie Reaping Problem

```bash
# In Linux, PID 1 (init) has a special responsibility:
# It must REAP orphaned child processes (zombies)
# A zombie = process that exited but parent hasn't called wait()
# Normal OS: systemd (PID 1) reaps zombies
# Container: YOUR process is PID 1

# Problem scenario:
# Your app spawns child processes (workers, scripts)
# Child exits → becomes zombie
# Your app doesn't call wait() → zombie accumulates
# 100s of zombies → PID namespace exhaustion → container can't fork

# SOLUTION 1: tini (lightweight init)
FROM python:3.11-slim
RUN apt-get update && apt-get install -y tini && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["tini", "--"]
CMD ["python", "app.py"]
# tini is PID 1 → reaps zombies, forwards signals to your app
# tini receives SIGTERM → forwards to python → graceful shutdown
# Only 30KB binary

# SOLUTION 2: dumb-init (Yelp)
ENTRYPOINT ["dumb-init", "--"]
CMD ["python", "app.py"]
# Same concept, slightly different signal handling

# SOLUTION 3: Docker's built-in init
docker run --init myapp
# Docker injects tini automatically
# Or in compose:
services:
  app:
    init: true
# In Kubernetes:
# No built-in equivalent — must include tini/dumb-init in image
# OR use shareProcessNamespace: true (but that has other implications)

# WHEN YOU NEED IT:
# - Shell scripts that spawn background processes
# - Applications using process pools (Gunicorn prefork, Celery workers)
# - Anything that forks child processes
# 
# WHEN YOU DON'T:
# - Single-process containers (Go binary, Node.js single-thread)
# - JVM (handles its own threads internally, not separate processes)
```

---

### Signal Handling in Containers

```bash
# SHUTDOWN SEQUENCE:
# docker stop <container>:
# 1. Sends SIGTERM to PID 1
# 2. Waits --stop-timeout (default 10s)
# 3. Sends SIGKILL

# docker kill <container>:
# 1. Sends SIGKILL immediately (no grace period)

# SHELL FORM TRAP:
# Dockerfile: CMD python app.py
# Actually runs: /bin/sh -c "python app.py"
# PID 1 = sh
# sh does NOT forward SIGTERM to python
# Python never knows it should shut down
# After 10s → SIGKILL → dirty shutdown (connections dropped, data loss)

# FIX 1: exec form (already covered)
CMD ["python", "app.py"]

# FIX 2: exec in shell script
#!/bin/sh
# docker-entrypoint.sh
echo "Starting app..."
exec python app.py
# exec REPLACES the shell process with python
# python becomes PID 1 → receives SIGTERM directly

# FIX 3: trap in shell script
#!/bin/sh
cleanup() {
    echo "Caught signal, shutting down..."
    kill -TERM "$child"
    wait "$child"
}
trap cleanup SIGTERM SIGINT

python app.py &
child=$!
wait "$child"
# Shell traps signal, forwards to child, waits for clean exit

# STOPSIGNAL directive:
# Some apps listen for different signals:
STOPSIGNAL SIGQUIT    # Nginx uses SIGQUIT for graceful shutdown
STOPSIGNAL SIGINT     # Some apps prefer SIGINT

# Kubernetes:
# preStop hook runs BEFORE SIGTERM
# terminationGracePeriodSeconds = total time for preStop + shutdown
# SIGKILL after grace period — unconditional
```

---

### Volume Types — Bind Mounts vs Volumes vs tmpfs

```bash
# THREE TYPES:

# 1. NAMED VOLUMES (Docker-managed):
docker volume create my-data
docker run -v my-data:/app/data myapp
# Stored at: /var/lib/docker/volumes/my-data/_data
# Docker manages lifecycle
# Survives container removal
# Can be backed by drivers (local, NFS, EBS, etc.)
# PREFERRED for persistent data

# 2. BIND MOUNTS (host path):
docker run -v /host/path:/container/path myapp
# Direct mount of host directory into container
# Changes visible immediately on both sides
# Use for: development (live reload), host config files
# DANGER: container can modify host files
# NOT portable (depends on host path existing)

# 3. TMPFS (memory-backed):
docker run --tmpfs /tmp:rw,noexec,nosuid,size=64m myapp
# RAM-backed filesystem
# Fast, auto-cleaned on container stop
# Use for: sensitive temp files (secrets processing), scratch space
# Not persisted, not shared between containers

# VOLUME DRIVERS:
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=nfs-server.example.com,rw \
  --opt device=:/shared/data \
  nfs-data
# local driver with NFS options
# Other drivers: REX-Ray (EBS/EFS), Portworx, Flocker

# COMPOSE VOLUME SYNTAX:
services:
  app:
    volumes:
      - my-data:/app/data              # named volume
      - ./config:/app/config:ro        # bind mount, read-only
      - type: tmpfs                    # tmpfs
        target: /tmp
        tmpfs:
          size: 67108864               # 64MB

# READ-ONLY VOLUMES:
docker run -v my-data:/app/data:ro myapp
# Container can read but not write
# Use for: shared config, certificates

# VOLUME INSPECTION:
docker volume ls
docker volume inspect my-data
# Shows: mount point, driver, labels, creation date

# ORPHANED VOLUMES:
docker volume ls -f dangling=true
# Volumes not referenced by any container
# Common source of disk space leak
docker volume prune    # remove all dangling volumes
```

---

### Docker Disk Management

```bash
# DISK USAGE:
docker system df
# TYPE            TOTAL   ACTIVE   SIZE      RECLAIMABLE
# Images          45      12       12.5GB    8.3GB (66%)
# Containers      15      8        2.1GB     1.8GB (85%)
# Local Volumes   23      8        5.6GB     3.2GB (57%)
# Build Cache     -       -        4.2GB     4.2GB

docker system df -v    # verbose — per-image, per-container breakdown

# CLEANUP:
# Remove stopped containers:
docker container prune

# Remove unused images (not referenced by any container):
docker image prune
# Remove ALL images not used by running containers:
docker image prune -a

# Remove unused volumes:
docker volume prune

# Remove unused networks:
docker network prune

# NUCLEAR: remove everything unused:
docker system prune -a --volumes
# Removes: stopped containers, unused networks, unused images,
#           build cache, dangling volumes
# DANGEROUS in production — removes cached images = slow next pull

# BUILD CACHE:
docker builder prune
# Remove build cache (can be massive after many builds)
docker builder prune --keep-storage=5GB
# Keep 5GB of most recently used cache

# AUTOMATED CLEANUP IN PRODUCTION:
# K8s kubelet handles image GC:
# --image-gc-high-threshold=85 (start GC when disk is 85% full)
# --image-gc-low-threshold=80 (GC until disk is 80% full)
# Removes least recently used images first
# On EKS: configured via kubelet extra args in launch template

# LOG ROTATION (often missed source of disk fill):
# Docker logs can grow unbounded by default!
# /var/lib/docker/containers/<id>/<id>-json.log

# Configure in /etc/docker/daemon.json:
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",        
    "max-file": "3"           
  }
}
# Each container: max 3 log files × 50MB = 150MB max
# Without this: a verbose app can fill a 100GB disk in hours

# PER-CONTAINER override:
docker run --log-opt max-size=10m --log-opt max-file=3 myapp
```

---

### Docker Logging Drivers

```bash
# Docker supports multiple log drivers:

# json-file (DEFAULT):
# Writes JSON-formatted logs to disk
# docker logs command works
# Needs rotation configured (see above)

# journald:
# Sends to systemd journal
# docker logs still works
# journalctl CONTAINER_NAME=my-container

# syslog:
# Sends to syslog daemon
# docker logs does NOT work (logs go to syslog, not local)

# fluentd:
# Sends to Fluentd collector
# docker logs does NOT work
# Use for: centralized logging pipelines

# awslogs:
# Sends directly to AWS CloudWatch Logs
# docker logs does NOT work
# Use for: ECS tasks, standalone Docker on EC2
{
  "log-driver": "awslogs",
  "log-opts": {
    "awslogs-region": "us-east-1",
    "awslogs-group": "/docker/my-app",
    "awslogs-stream": "my-container",
    "awslogs-create-group": "true"
  }
}

# IMPORTANT:
# In Kubernetes: DON'T use Docker log drivers
# kubelet reads container logs from the default json-file/local driver
# kubectl logs depends on this
# If you change the driver → kubectl logs stops working
# K8s logging: use DaemonSet (Fluentbit/Fluentd) to ship log files
# from /var/log/containers/*.log to your logging backend
```

---

### Docker Restart Policies

```bash
docker run --restart=no myapp          # DEFAULT: never restart
docker run --restart=on-failure:5 myapp  # restart on non-zero exit, max 5 times
docker run --restart=always myapp      # always restart (even on clean exit)
docker run --restart=unless-stopped myapp  # like always, but not after docker stop

# BACKOFF:
# Docker uses exponential backoff for restarts:
# 1st: immediate
# 2nd: 1 second
# 3rd: 2 seconds
# 4th: 4 seconds
# ...up to 1 minute cap
# Reset to 0 after container runs successfully for 10+ seconds

# COMPOSE:
services:
  app:
    restart: unless-stopped
    # or deploy.restart_policy for Swarm mode

# KUBERNETES EQUIVALENT:
# restartPolicy: Always (default for Deployments)
# restartPolicy: OnFailure (for Jobs)
# restartPolicy: Never (for one-shot pods)
# K8s kubelet manages restarts with its own backoff:
# CrashLoopBackOff: 10s → 20s → 40s → 80s → 160s → 300s (5 min cap)
```

---

### Multi-Architecture Builds (buildx)

```bash
# WHY: ARM instances (Graviton on AWS) are 20-40% cheaper
# Your image built on AMD64 laptop won't run on ARM nodes
# You need images for BOTH architectures

# SETUP:
docker buildx create --name multiarch --driver docker-container --use
docker buildx inspect --bootstrap

# BUILD FOR MULTIPLE PLATFORMS:
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag my-app:v1.2.3 \
  --push \
  .
# Builds TWO images, creates a manifest list
# When pulled: Docker automatically selects the right architecture
# docker pull my-app:v1.2.3 on AMD64 → gets AMD64 image
# docker pull my-app:v1.2.3 on ARM64 → gets ARM64 image

# Dockerfile considerations for multi-arch:
# Most base images support multi-arch (python:3.11-slim, node:20-slim)
# But if you compile native code:
FROM --platform=$BUILDPLATFORM golang:1.21 AS builder
ARG TARGETARCH
RUN GOARCH=$TARGETARCH go build -o /app/server
# $BUILDPLATFORM = where you're building (your laptop)
# $TARGETARCH = target architecture (amd64 or arm64)
# Cross-compilation happens on your machine → fast

# CHECK IMAGE ARCHITECTURE:
docker manifest inspect my-app:v1.2.3
# Shows all platforms available in the manifest list

# CI PIPELINE:
# Build multi-arch in CI (Jenkins/GitHub Actions):
# Use QEMU emulation for cross-platform builds
docker buildx create --name ci-builder \
  --driver docker-container \
  --driver-opt image=moby/buildkit:latest
# Or: use native ARM runners (GitHub has Graviton runners)

# EKS WITH GRAVITON:
# m6g.xlarge, c6g.large, r6g.2xlarge = ARM (Graviton)
# Karpenter can select these automatically:
# requirements:
# - key: kubernetes.io/arch
#   operator: In
#   values: ["amd64", "arm64"]   # allow both
# Karpenter picks cheapest → usually Graviton
# Your image MUST support arm64 or pods will crash
```

---

### Image Signing and Verification

```bash
# PROBLEM: how do you know the image you're pulling
# hasn't been tampered with?

# COSIGN (Sigstore — industry standard):
# Sign:
cosign sign --key cosign.key my-registry/my-app:v1.2.3
# Signature stored alongside image in registry

# Verify:
cosign verify --key cosign.pub my-registry/my-app:v1.2.3
# If signature doesn't match → verification fails

# Keyless signing (using OIDC identity — CI pipeline identity):
cosign sign my-registry/my-app:v1.2.3
# Uses GitHub Actions OIDC token / Fulcio CA
# No key management needed
# Signature tied to: "this image was built by THIS CI pipeline in THIS repo"

# ENFORCE IN KUBERNETES:
# Kyverno policy to require signed images:
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-cosign
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "my-registry/*"
      attestors:
      - entries:
        - keyless:
            issuer: "https://token.actions.githubusercontent.com"
            subject: "https://github.com/myorg/*"

# DOCKER CONTENT TRUST (DCT):
export DOCKER_CONTENT_TRUST=1
docker push my-app:v1.2.3
# Push signs the image with Notary
docker pull my-app:v1.2.3
# Pull verifies signature
# Any unsigned image → pull rejected

# DCT is older, less flexible than Cosign
# Industry is moving toward Cosign/Sigstore
```

---

### Docker-in-Docker vs Docker-out-of-Docker

```bash
# CI pipelines need to build Docker images.
# How do you build images inside a container?

# DOCKER-IN-DOCKER (DinD):
# Run a full Docker daemon INSIDE a container
docker run --privileged -d docker:dind
# The inner Docker daemon is completely separate
# Builds happen inside the inner daemon
#
# PROBLEMS:
# --privileged = full host capabilities = SECURITY NIGHTMARE
# File system layers get nested (slow, complex)
# Build cache is lost when container stops
# NOT recommended for production CI

# DOCKER-OUT-OF-DOCKER (DooD):
# Mount the HOST's Docker socket into the container
docker run -v /var/run/docker.sock:/var/run/docker.sock docker
# Container uses the HOST's Docker daemon
# Builds appear on the host
# Cache is shared with host
#
# PROBLEMS:
# Container has FULL CONTROL of host Docker
# Can start privileged containers, mount host filesystem
# Effectively root on the host
# Containers built are siblings, not children (confusing networking)

# KANIKO (the right answer for Kubernetes):
# Builds OCI images inside a container WITHOUT Docker daemon
# No privileged mode, no Docker socket
apiVersion: v1
kind: Pod
metadata:
  name: kaniko-build
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args:
    - "--dockerfile=Dockerfile"
    - "--context=git://github.com/myorg/myrepo"
    - "--destination=my-registry/my-app:v1.2.3"
    - "--cache=true"                 # layer caching via registry
    - "--cache-repo=my-registry/my-app/cache"
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker/
  volumes:
  - name: docker-config
    secret:
      secretName: docker-registry-creds
# No privileged mode, no Docker socket
# Builds happen in userspace using snapshotting
# Standard for Kubernetes-based CI (Jenkins on K8s, Tekton, etc.)

# BUILDAH (daemonless alternative):
buildah bud -t my-app:v1.2.3 .
buildah push my-app:v1.2.3 docker://my-registry/my-app:v1.2.3
# No daemon required
# Can run rootless
# OCI-compliant images
# Good for: non-Kubernetes CI environments
```

---

### Docker Inspect and Filesystem Debugging

```bash
# INSPECT — full container metadata:
docker inspect <container>
# Returns JSON with EVERYTHING:
# - State (running, paused, dead, exit code, OOMKilled, pid)
# - NetworkSettings (IP, ports, networks)
# - Mounts (volumes, bind mounts)
# - Config (env vars, cmd, entrypoint, labels)
# - HostConfig (resources, restart policy, capabilities)

# Useful filters:
docker inspect --format='{{.State.ExitCode}}' <container>
docker inspect --format='{{.State.OOMKilled}}' <container>
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container>
docker inspect --format='{{json .Config.Env}}' <container> | jq
docker inspect --format='{{.HostConfig.Memory}}' <container>

# DIFF — see what changed in the container filesystem:
docker diff <container>
# A /app/data/new-file.txt          (Added)
# C /var/log                         (Changed)
# D /tmp/old-file.txt               (Deleted)
# Useful for: debugging unexpected writes, auditing changes

# CP — copy files in/out:
docker cp <container>:/app/config.yaml ./config.yaml    # out
docker cp ./fix.py <container>:/app/fix.py              # in
# Use for: extracting logs, injecting hotfixes (NOT for production)

# EXPORT — full container filesystem as tar:
docker export <container> > container-fs.tar
# For deep forensic analysis

# HISTORY — see how image was built:
docker history my-app:v1.2.3 --no-trunc
# Shows every layer, command that created it, size
# Useful for: understanding third-party images, finding bloat

# EVENTS — real-time Docker daemon events:
docker events
# container create, start, die, kill, oom, health_status
# image pull, push, tag, delete
# volume create, mount, unmount
# network connect, disconnect
# Useful for: debugging startup issues, OOM events
docker events --filter event=oom --filter event=die
# Monitor only death and OOM events
```

---

### Docker Compose Production Patterns

```yaml
# DEVELOPMENT vs PRODUCTION compose:

# docker-compose.yml (base):
services:
  api:
    image: my-app:${APP_VERSION}
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 40s     # don't check health during startup

# docker-compose.override.yml (auto-loaded, development):
services:
  api:
    build:
      context: .
      target: development
    volumes:
      - ./src:/app/src     # live reload
    ports:
      - "8080:8080"        # expose to host
    environment:
      - DEBUG=true

# docker-compose.prod.yml (explicit, production):
services:
  api:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"

# Usage:
# Dev (auto-loads override):
docker compose up

# Prod (explicit file):
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# PROFILES (selective service startup):
services:
  api:
    profiles: ["app"]
  worker:
    profiles: ["app"]
  debug-tools:
    profiles: ["debug"]       # only started when explicitly requested
  db:
    # no profile = always started

docker compose --profile app up        # api + worker + db
docker compose --profile debug up      # debug-tools + db

# DEPENDS_ON with conditions:
services:
  api:
    depends_on:
      db:
        condition: service_healthy      # wait for health check
      redis:
        condition: service_started      # just wait for start
      migrate:
        condition: service_completed_successfully  # wait for completion
```

---

### Container Resource Visibility — /proc and Cgroup Awareness

```bash
# CRITICAL ISSUE:
# Inside a container, /proc/meminfo and /proc/cpuinfo show HOST values
# Not the container's cgroup limits

cat /proc/meminfo
# MemTotal: 65536000 kB    ← HOST has 64GB
# But container is limited to 512MB via cgroup

cat /proc/cpuinfo | grep processor | wc -l
# 16                        ← HOST has 16 CPUs
# But container is limited to 2 CPUs

# WHY THIS MATTERS:
# Applications that auto-configure based on /proc:
# - JVM: calculates heap from /proc/meminfo (old JVMs)
# - Node.js: UV_THREADPOOL_SIZE defaults to 4, could set based on CPUs
# - Go: GOMAXPROCS defaults to /proc/cpuinfo CPU count
# - Python multiprocessing: os.cpu_count() reads /proc/cpuinfo

# FIXES:
# Java 10+: reads cgroup limits automatically (UseContainerSupport)
# Go: use automaxprocs library (uber-go/automaxprocs)
#   import _ "go.uber.org/automaxprocs"
#   Reads cgroup CPU quota, sets GOMAXPROCS correctly
# Node.js: UV_THREADPOOL_SIZE=<your CPU limit>
# Python: read cgroup directly or use container-aware libraries

# LXCFS (advanced):
# Mounts cgroup-aware /proc into container
# /proc/meminfo shows container's memory limit, not host's
# /proc/cpuinfo shows container's CPU limit
# Used in: multi-tenant environments where apps MUST see correct resources
# Not common in K8s (most apps handle this natively now)
```

---

### Container Networking — Advanced Patterns

```bash
# DNS RESOLUTION IN CONTAINERS:

# Default bridge network:
docker run --name web nginx
docker run --name app myapp
# app CANNOT reach web by name "web"
# Must use IP address (which changes on restart)
# Default bridge: no DNS resolution between containers

# Custom bridge network:
docker network create mynet
docker run --network mynet --name web nginx
docker run --network mynet --name app myapp
# app CAN reach web by name "web"
# Docker's embedded DNS server resolves container names
# DNS server at 127.0.0.11 inside the container

# MULTIPLE NETWORKS (isolation):
docker network create frontend
docker network create backend
docker run --network frontend --name web nginx
docker run --network frontend --network backend --name api myapp
docker run --network backend --name db postgres
# web can reach api (both on frontend)
# api can reach db (both on backend)
# web CANNOT reach db (different networks, no route)
# This is network segmentation without firewall rules

# ALIAS:
docker run --network mynet --network-alias search elasticsearch
docker run --network mynet --network-alias search elasticsearch
# Both containers respond to DNS name "search"
# DNS round-robin load balancing
# Primitive but functional for development

# HOST NETWORKING WITH SPECIFIC PORTS:
# --network=host bypasses ALL Docker networking
# Container uses host's network stack directly
# Performance: eliminates NAT overhead (~5-10% improvement in high-throughput)
# Use case: performance-critical network applications, monitoring tools
# Trade-off: no network isolation, port conflicts

# CONTAINER NETWORKING:
docker run --network container:<other-container-name> myapp
# Shares ANOTHER container's network namespace
# Same IP, same ports, same interfaces
# Use case: sidecar pattern without Kubernetes
# Debug container sharing network with target container
```

---

### Health Checks — Docker vs Kubernetes

```bash
# DOCKER HEALTHCHECK:
# In Dockerfile:
HEALTHCHECK --interval=30s --timeout=5s --retries=3 --start-period=60s \
  CMD curl -f http://localhost:8080/health || exit 1

# Or at runtime:
docker run \
  --health-cmd="curl -f http://localhost:8080/health || exit 1" \
  --health-interval=30s \
  --health-timeout=5s \
  --health-retries=3 \
  --health-start-period=60s \
  myapp

# States:
# starting: within start-period, health not yet checked
# healthy: health check passing
# unhealthy: failed retries threshold

# Docker health check vs K8s probes:
# Docker HEALTHCHECK: only affects container STATUS display
#   Docker restart policy can use it (restart unhealthy containers)
#   docker compose depends_on: service_healthy uses it
#   But it does NOT control traffic routing
#
# K8s probes: control EVERYTHING
#   Liveness: restart on failure
#   Readiness: remove from Service endpoints on failure
#   Startup: gate liveness/readiness until ready
#
# In K8s: Docker HEALTHCHECK is IGNORED
# K8s uses its own probes defined in pod spec
# Don't rely on Dockerfile HEALTHCHECK for K8s workloads
# But keep it for: docker-compose, standalone Docker, ECS
```

---

### Docker Build Optimization — Advanced

```bash
# BUILDKIT CACHE MOUNTS:
# Persist package manager caches across builds

# Go:
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /app/server

# Python:
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Node.js:
RUN --mount=type=cache,target=/root/.cache/npm \
    npm ci

# apt:
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt/lists \
    apt-get update && apt-get install -y curl

# RESULT: package downloads cached across ALL builds
# Even if requirements.txt changes, only new packages downloaded

# SECRET MOUNTS (BuildKit):
# Pass secrets without embedding in image layers:
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm install
# Build:
docker build --secret id=npm_token,src=.npmrc .
# Secret available only during that RUN instruction
# NOT stored in any image layer
# NOT visible in docker history

# SSH MOUNTS (BuildKit):
# Forward SSH agent for private repo access:
RUN --mount=type=ssh git clone git@github.com:private/repo.git
# Build:
docker build --ssh default .
# SSH keys never touch the image

# REGISTRY CACHE:
docker buildx build \
  --cache-from type=registry,ref=my-registry/my-app:cache \
  --cache-to type=registry,ref=my-registry/my-app:cache,mode=max \
  --push \
  -t my-registry/my-app:v1.2.3 .
# Pushes build cache to registry
# Next build pulls cache from registry
# Works across different CI runners (no local cache dependency)
# mode=max: cache ALL layers (not just final stage)

# BUILD ARGS vs ENV:
ARG BUILD_VERSION=unknown    # available ONLY during build
ENV APP_VERSION=${BUILD_VERSION}  # persisted in image
# ARG: use for build-time configuration (versions, flags)
# ENV: use for runtime configuration
# SECURITY: ARG values visible in docker history!
# Never pass secrets via ARG
```

---

### Docker Troubleshooting Playbook

```bash
# CONTAINER WON'T START:
docker logs <container>                    # check stdout/stderr
docker inspect <container> | jq '.State'   # check exit code, OOMKilled
# Exit code 0: clean exit (CMD finished)
# Exit code 1: application error
# Exit code 126: permission denied (can't execute CMD)
# Exit code 127: command not found (CMD doesn't exist in image)
# Exit code 137: SIGKILL (OOM killed or docker kill)
# Exit code 139: SIGSEGV (segfault — native code crash)
# Exit code 143: SIGTERM (docker stop, graceful)

# HIGH DISK USAGE:
docker system df                           # overview
docker system df -v                        # detailed
# Check: dangling images, stopped containers, unused volumes
# Check: container log sizes
du -sh /var/lib/docker/containers/*/      # log file sizes per container

# NETWORKING ISSUES:
docker exec <container> cat /etc/resolv.conf  # DNS config
docker exec <container> ping <other-container>
docker network inspect <network>              # see connected containers
iptables -t nat -L -n | grep DOCKER           # see port mappings

# IMAGE WON'T PULL:
# 1. Auth: docker login / ECR token expired
# 2. Network: DNS resolution, proxy settings
# 3. Rate limit: Docker Hub limits (100 pulls/6h anonymous)
#    Fix: use ECR pull-through cache or mirror
# 4. Image doesn't exist for your architecture (arm64 vs amd64)

# CONTAINER RUNNING BUT APP NOT RESPONDING:
docker exec <container> ss -tlnp           # is app listening?
docker exec <container> curl localhost:8080 # can app reach itself?
docker port <container>                     # port mapping correct?
# If app binds to 127.0.0.1 inside container → not reachable from outside
# Must bind to 0.0.0.0 inside container for port mapping to work
```
