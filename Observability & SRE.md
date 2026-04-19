# Phase 5: Observability & SRE
## Lesson 1: Observability Foundations — The Three Pillars + Prometheus Deep Dive

---

## Why This Phase Changes Everything

```
Phases 0-4 taught you to BUILD infrastructure.
Phase 5 teaches you to KNOW if it's working.

Without observability:
  "Is the system healthy?" → "I think so? Nobody's complained yet."
  "What caused the outage?" → "We're not sure. Let's check the logs... 
                                which logs? There are 200 services."
  "Are we meeting our reliability targets?" → "What targets?"

With observability:
  "Is the system healthy?" → "SLO compliance is 99.97%, error budget 
                               has 4.2 hours remaining this month."
  "What caused the outage?" → "Trace ID abc123 shows 2.3s latency at 
                                payment-service → db query, correlating 
                                with CPU throttling on pod payment-7f8b."
  "Are we meeting our reliability targets?" → Dashboard. One glance. Done.

At NovaMart:
  $50K/minute outage cost × average 15 minutes to detect without monitoring
  = $750K per incident in DETECTION DELAY ALONE
  
  With proper observability: detection in <2 minutes
  Savings: $650K per incident
```

---

## 1. The Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                                  │
│                                                                   │
│  "The ability to understand the internal state of a system        │
│   by examining its external outputs"                              │
│                                                                   │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐    │
│  │   METRICS   │   │    LOGS     │   │      TRACES         │    │
│  │             │   │             │   │                      │    │
│  │ "What is    │   │ "What       │   │ "What is the path   │    │
│  │  happening?"│   │  happened?" │   │  of a request?"     │    │
│  │             │   │             │   │                      │    │
│  │ Numeric     │   │ Text events │   │ Distributed call    │    │
│  │ time-series │   │ with        │   │ graphs across       │    │
│  │ data        │   │ context     │   │ services            │    │
│  │             │   │             │   │                      │    │
│  │ Examples:   │   │ Examples:   │   │ Examples:           │    │
│  │ CPU 78%     │   │ "User 123  │   │ API GW → Cart Svc   │    │
│  │ Req/s: 5200 │   │  failed    │   │ → Order Svc         │    │
│  │ Latency p99:│   │  login:    │   │ → Payment Svc       │    │
│  │  450ms      │   │  invalid   │   │ → DB (slow: 2.3s)   │    │
│  │ Errors: 12  │   │  password" │   │ → Notification Svc  │    │
│  │             │   │             │   │                      │    │
│  │ Tool:       │   │ Tool:      │   │ Tool:               │    │
│  │ Prometheus  │   │ Loki / ELK │   │ Jaeger / Tempo      │    │
│  └──────┬──────┘   └──────┬──────┘   └──────────┬──────────┘    │
│         │                 │                      │               │
│         └─────────────────┼──────────────────────┘               │
│                           │                                      │
│                    ┌──────▼──────┐                                │
│                    │  GRAFANA    │   Unified visualization        │
│                    │  Dashboard  │   Metrics + Logs + Traces      │
│                    │             │   correlated by time and       │
│                    │             │   trace ID                     │
│                    └─────────────┘                                │
└─────────────────────────────────────────────────────────────────┘
```

### Monitoring vs Observability

```
MONITORING (traditional):
  "I know what questions to ask in advance"
  → Set up checks for CPU > 80%, disk > 90%, HTTP 5xx > 10/min
  → Alerts fire when thresholds are crossed
  → Works for KNOWN failure modes
  → Breaks for UNKNOWN failure modes
  
  Example: You monitor CPU, memory, disk. But the outage is caused by 
  a DNS resolution timeout that makes service-to-service calls take 30s.
  None of your dashboards show this because you didn't think to monitor 
  DNS latency from inside pods.

OBSERVABILITY (modern):
  "I can ask NEW questions without deploying new code"
  → Rich telemetry: metrics, logs, traces with high cardinality
  → Can investigate issues you DIDN'T predict
  → Explore data interactively during incidents
  
  Example: Same DNS issue. With observability:
  1. Alert: p99 latency spiked on order-service
  2. Click trace → see 28s gap between order-service and payment-service
  3. Expand span → DNS resolution took 27.8s
  4. Correlate with metrics → CoreDNS pods are OOMKilled
  5. Root cause found in 3 minutes without any pre-built DNS dashboard

KEY INSIGHT:
  Monitoring tells you WHEN something is broken.
  Observability tells you WHY.
  You need BOTH.
```

### How the Three Pillars Correlate

```
INCIDENT TIMELINE:

1. ALERT fires (from METRICS)
   → "HTTP error rate on order-service exceeded 5% for 2 minutes"
   → p99 latency: 4200ms (normal: 200ms)

2. INVESTIGATE with TRACES
   → Find a slow trace: order-service → payment-service → RDS
   → The RDS span shows 3800ms (normal: 50ms)
   → payment-service is healthy; the database is the bottleneck

3. DEEP DIVE with LOGS
   → Filter logs by trace_id from step 2
   → RDS logs: "Lock wait timeout exceeded; try restarting transaction"
   → A long-running migration query is holding table locks

4. CORRELATE back to METRICS
   → RDS dashboard: active connections spiked to 190/200 max
   → CPU: 98% on the writer instance
   → The migration query started at 14:32 UTC (matches the alert time)

5. RESOLVE
   → Kill the migration query
   → Latency drops immediately
   → Error rate returns to baseline

Without all three pillars, you'd be:
  - Metrics only: "Something is slow" (WHERE?)
  - Logs only: "There's a lock timeout" (WHY? WHICH request? HOW widespread?)
  - Traces only: "This request hit the DB" (Is it an isolated case or systemic?)
```

---

## 2. Prometheus — Architecture Deep Dive

### Why Prometheus Won

```
Before Prometheus (2012-2015):
  Nagios:   Host-level checks, polling, terrible scaling, config is XML hell
  Zabbix:   Better than Nagios, still agent-based, rigid schema
  Graphite: Metrics storage + graphing, push-based, no alerting
  StatsD:   Metrics aggregation, no storage, no querying
  
  Problems:
  - No dimensional data model (can't filter by label)
  - Push-based (services push metrics → requires discovery + registration)
  - Poor K8s integration (designed for static servers, not ephemeral pods)
  - No native alerting with silencing/routing

Prometheus (2015+, CNCF graduated):
  ✓ Pull-based (Prometheus scrapes targets — works with K8s service discovery)
  ✓ Dimensional data model (labels: {service="order", region="us-east-1"})
  ✓ PromQL (powerful query language)
  ✓ Native alerting (Alertmanager with routing, silencing, grouping)
  ✓ K8s-native (ServiceMonitor CRDs, auto-discovery)
  ✓ Ecosystem (thousands of exporters, Grafana integration)
  ✓ Battle-tested at scale (SoundCloud, DigitalOcean, CERN, NovaMart)
```

### Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     PROMETHEUS SERVER                                │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │  Service      │  │  Scrape      │  │      TSDB               │   │
│  │  Discovery    │  │  Engine      │  │  (Time Series Database)  │   │
│  │              │  │              │  │                          │   │
│  │ Finds targets │  │ HTTP GET     │  │  ┌─────────────────┐    │   │
│  │ to scrape:    │  │ /metrics     │  │  │ Head Block      │    │   │
│  │              │  │ every 15-30s │  │  │ (last 2 hours,  │    │   │
│  │ - K8s API    │  │              │  │  │  in memory)     │    │   │
│  │ - DNS SRV    │  │ Parses       │  │  └────────┬────────┘    │   │
│  │ - Consul     │  │ Prometheus   │  │           │ compaction  │   │
│  │ - EC2 API    │  │ exposition   │  │  ┌────────▼────────┐    │   │
│  │ - File       │  │ format       │  │  │ Persistent      │    │   │
│  │ - Static     │  │              │  │  │ Blocks (2h      │    │   │
│  │              │  │              │  │  │ chunks on disk)  │    │   │
│  └──────┬───────┘  └──────┬───────┘  │  └─────────────────┘    │   │
│         │                 │          │                          │   │
│         └─────────────────┘          │  Retention: 15d default  │   │
│                                      │  WAL for crash recovery  │   │
│                                      └──────────────────────────┘   │
│                                                                      │
│  ┌──────────────┐  ┌──────────────────────────────────────────┐     │
│  │  PromQL      │  │  Rule Engine                              │     │
│  │  Engine      │  │                                            │     │
│  │              │  │  Recording Rules:                          │     │
│  │  Evaluates   │  │    Pre-compute expensive queries           │     │
│  │  queries     │  │    e.g., 5m error rate per service         │     │
│  │  from        │  │                                            │     │
│  │  Grafana,    │  │  Alerting Rules:                           │     │
│  │  API, rules  │  │    Evaluate conditions → fire alerts       │     │
│  │              │  │    e.g., error_rate > 5% for 5m            │     │
│  └──────────────┘  └────────────────────┬───────────────────────┘     │
│                                         │                            │
└─────────────────────────────────────────┼────────────────────────────┘
                                          │ alerts
                                          ▼
                              ┌──────────────────────┐
                              │   Alertmanager        │
                              │                       │
                              │  - Deduplication      │
                              │  - Grouping           │
                              │  - Silencing          │
                              │  - Inhibition         │
                              │  - Routing            │
                              │                       │
                              │  Routes to:           │
                              │  - PagerDuty (SEV1)   │
                              │  - Slack (SEV3-4)     │
                              │  - Email (reports)    │
                              └──────────────────────┘

EXTERNAL CONSUMERS:
  ┌──────────┐  ┌──────────────┐  ┌─────────────────┐
  │ Grafana  │  │ Thanos /     │  │  Custom tools   │
  │ (viz)    │  │ Cortex /     │  │  (Python scripts │
  │          │  │ Mimir        │  │   hitting API)   │
  │ Queries  │  │ (long-term   │  │                  │
  │ PromQL   │  │  storage,    │  │                  │
  │ via HTTP │  │  global      │  │                  │
  │ API      │  │  view)       │  │                  │
  └──────────┘  └──────────────┘  └─────────────────┘
```

### The Pull Model — Why It Matters

```
PUSH-BASED (StatsD, Graphite, CloudWatch):
  Service → pushes metrics → collector

  Problems:
  1. Service must know WHERE to push (hardcoded endpoint)
  2. If collector is down, metrics are LOST
  3. In K8s: pods are ephemeral — who pushes? where?
  4. Backpressure: if 200 services push simultaneously → overwhelm collector
  5. Can't tell the difference between "service is healthy with 0 errors" 
     and "service is dead and not pushing"

PULL-BASED (Prometheus):
  Prometheus → scrapes metrics ← from targets

  Advantages:
  1. Targets just expose /metrics — simple HTTP endpoint
  2. If Prometheus is down, targets are unaffected (metrics still generated)
  3. In K8s: Prometheus discovers pods via API — ephemeral is fine
  4. Prometheus controls scrape rate — no backpressure on targets
  5. "Up" detection is FREE: if scrape fails → target is down
     The "up" metric (0 or 1) comes automatically
  6. You can manually curl /metrics for debugging:
     curl http://pod-ip:8080/metrics | grep http_requests

EXCEPTION — Pushgateway:
  For SHORT-LIVED jobs (batch, cron, CI) that may finish before 
  Prometheus scrapes them:
  
  CronJob → pushes metrics → Pushgateway ← Prometheus scrapes
  
  USE SPARINGLY. Pushgateway is a single point of failure and 
  doesn't support "up" detection. Only for jobs, never for services.
```

### Metric Types — The Four Fundamental Types

```
┌────────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS METRIC TYPES                          │
│                                                                    │
│  ┌───────────┐  ┌───────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │  COUNTER  │  │   GAUGE   │  │  HISTOGRAM  │  │   SUMMARY    │ │
│  │           │  │           │  │             │  │              │ │
│  │ Only goes │  │ Goes up   │  │ Buckets of  │  │ Client-side  │ │
│  │ UP (or    │  │ AND down  │  │ observations│  │ quantiles    │ │
│  │ resets    │  │           │  │             │  │              │ │
│  │ to 0)     │  │           │  │ Server-side │  │ NOT          │ │
│  │           │  │           │  │ quantile    │  │ aggregatable │ │
│  │           │  │           │  │ calculation │  │              │ │
│  └───────────┘  └───────────┘  └─────────────┘  └──────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

#### Counter

```
# A counter ONLY increases (or resets to 0 on process restart).
# Use for: requests, errors, bytes transferred, tasks completed

# Example: HTTP request counter
http_requests_total{method="GET", path="/api/orders", status="200"} 1547892
http_requests_total{method="GET", path="/api/orders", status="500"} 342
http_requests_total{method="POST", path="/api/orders", status="201"} 89023

# NEVER use a counter's raw value. It's meaningless.
# "We've had 1,547,892 requests" — so what? Since when?

# ALWAYS use rate() or increase() with counters:

# Requests per second (averaged over 5 minutes):
rate(http_requests_total[5m])

# Errors per second:
rate(http_requests_total{status=~"5.."}[5m])

# Error percentage:
rate(http_requests_total{status=~"5.."}[5m]) 
/ rate(http_requests_total[5m]) * 100

# rate() handles counter resets automatically.
# If a pod restarts and counter goes from 50000 → 0 → 150,
# rate() sees the reset and calculates correctly.

# COMMON MISTAKE:
# increase(http_requests_total[1h]) — looks like "requests in the last hour"
# But increase() is syntactic sugar for rate() * seconds.
# It's an ESTIMATE, not an exact count. Use for dashboards, not billing.
```

#### Gauge

```
# A gauge can go UP or DOWN.
# Use for: temperature, memory usage, queue depth, active connections,
#          number of running pods

# Examples:
node_memory_MemAvailable_bytes 4294967296
node_filesystem_avail_bytes{mountpoint="/"} 53687091200
kube_deployment_spec_replicas{deployment="order-service"} 5
process_open_fds 127

# With gauges, the raw value IS meaningful:
# "Available memory is 4GB" — that's directly useful

# Useful functions for gauges:
# Current value (just the metric name):
node_memory_MemAvailable_bytes

# Min/max/avg over time:
min_over_time(node_memory_MemAvailable_bytes[1h])
max_over_time(node_memory_MemAvailable_bytes[1h])
avg_over_time(node_memory_MemAvailable_bytes[1h])

# Rate of change (useful for "is memory leaking?"):
deriv(process_resident_memory_bytes[1h])
# Positive = growing, negative = shrinking
# If consistently positive for hours → memory leak
```

#### Histogram

```
# A histogram samples observations and counts them in configurable BUCKETS.
# Use for: request duration, response size, queue wait time

# When you instrument:
#   histogram.Observe(0.25)  // Request took 250ms

# Prometheus stores THREE things:
http_request_duration_seconds_bucket{le="0.01"}  4892     # ≤ 10ms
http_request_duration_seconds_bucket{le="0.05"}  15823    # ≤ 50ms
http_request_duration_seconds_bucket{le="0.1"}   28456    # ≤ 100ms
http_request_duration_seconds_bucket{le="0.25"}  45012    # ≤ 250ms
http_request_duration_seconds_bucket{le="0.5"}   49823    # ≤ 500ms
http_request_duration_seconds_bucket{le="1.0"}   51203    # ≤ 1s
http_request_duration_seconds_bucket{le="2.5"}   51502    # ≤ 2.5s
http_request_duration_seconds_bucket{le="5.0"}   51520    # ≤ 5s
http_request_duration_seconds_bucket{le="10.0"}  51523    # ≤ 10s
http_request_duration_seconds_bucket{le="+Inf"}  51523    # all requests (= _count)

http_request_duration_seconds_sum    12043.29             # Total seconds spent
http_request_duration_seconds_count  51523                # Total request count

# BUCKET DESIGN IS CRITICAL:
# Too few buckets → poor resolution
# Too many buckets → high cardinality (memory + storage cost)
# Wrong boundaries → can't answer the questions you need

# NovaMart SLO: 99% of requests < 500ms
# You MUST have a bucket at 0.5 to answer this.
# If your closest buckets are 0.25 and 1.0, you're interpolating.

# Calculate percentiles SERVER-SIDE with histogram_quantile():

# p50 (median):
histogram_quantile(0.5, rate(http_request_duration_seconds_bucket[5m]))

# p95:
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# p99:
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# WHY HISTOGRAMS OVER SUMMARIES:
# 1. Histograms are AGGREGATABLE across instances:
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)
# This gives the TRUE p99 across all pods. Summaries can't do this.
#
# 2. Histograms let you change the quantile at QUERY TIME.
#    Summaries bake quantiles in at instrumentation time.
#
# 3. Histograms are cheaper to compute on the client side.
#
# USE SUMMARIES ONLY when you need exact client-calculated quantiles
# and don't need to aggregate across instances (rare).
```

#### Summary (Brief — Use Histograms Instead)

```
# A summary calculates quantiles ON THE CLIENT (in your app).
# Prometheus just stores the pre-calculated values.

http_request_duration_seconds{quantile="0.5"}  0.042
http_request_duration_seconds{quantile="0.9"}  0.098
http_request_duration_seconds{quantile="0.99"} 0.245
http_request_duration_seconds_sum   10043.29
http_request_duration_seconds_count 42523

# Problems:
# 1. Can't aggregate across instances (averaging percentiles is WRONG)
#    p99 of pod-1 = 100ms, p99 of pod-2 = 500ms
#    Average = 300ms? NO. The real p99 could be anything.
#
# 2. Quantiles are fixed at instrumentation time — can't change at query time
# 3. Expensive to compute on the client (streaming quantile estimation)
#
# VERDICT: Use histograms. Always. The only exception is when you 
# have a single instance and need exact quantiles (rare at NovaMart scale).
```

### Metric Naming Conventions

```
# FORMAT: <namespace>_<subsystem>_<name>_<unit>
# WHERE unit is a BASE UNIT (seconds, bytes, total — not milliseconds, megabytes)

# GOOD:
http_request_duration_seconds          # duration in seconds (use seconds, not ms)
http_requests_total                    # counter — _total suffix
node_memory_MemAvailable_bytes         # memory in bytes (use bytes, not MB)
process_open_fds                       # no unit needed (count of file descriptors)
http_response_size_bytes               # size in bytes

# BAD:
http_request_duration_milliseconds     # use seconds, convert in queries/dashboards
request_count                          # missing namespace, missing _total
memory_usage_mb                        # use bytes
requestLatency                         # camelCase (use snake_case)

# LABELS — The power of Prometheus:
http_requests_total{
  service="order-service",
  method="POST",
  path="/api/v1/orders",
  status="201",
  region="us-east-1",
  pod="order-service-7f8b9-xk2mv"
}

# Labels enable DIMENSIONAL queries:
# Error rate per service:
rate(http_requests_total{status=~"5.."}[5m]) by (service)

# Latency by region:
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, region)
)
```

### Cardinality — The Prometheus Killer

```
CARDINALITY = number of unique time series

Each unique combination of metric name + label values = one time series.

http_requests_total{service="order", method="GET", status="200"}   → 1 series
http_requests_total{service="order", method="GET", status="500"}   → 1 series
http_requests_total{service="order", method="POST", status="201"}  → 1 series
http_requests_total{service="cart", method="GET", status="200"}    → 1 series

4 services × 5 methods × 10 status codes × 3 regions = 600 series
That's fine.

NOW ADD A BAD LABEL:

http_requests_total{..., user_id="abc123"}

50M users × 600 combinations = 30 BILLION time series
Prometheus: dead. OOM killed. Disk full. Query timeouts.

THE RULES:
  ✓ Low-cardinality labels: service, method, status code, region, pod
  ✗ NEVER use high-cardinality values as labels:
    - user_id, email, IP address, request_id, trace_id
    - URL path with parameters: /users/12345 (infinite unique values)
    - Timestamps, UUIDs, session IDs
    
  If you need per-user data → that's LOGS, not metrics.
  Metrics are for AGGREGATES. Logs are for INDIVIDUAL EVENTS.

MONITORING CARDINALITY:
  # Total active time series:
  prometheus_tsdb_head_series
  
  # Series created per scrape (should be stable, not growing):
  rate(prometheus_tsdb_head_series_created_total[5m])
  
  # Top offenders (which metrics have the most series):
  topk(10, count by (__name__)({__name__=~".+"}))
  
  # NovaMart target: < 2 million active series per Prometheus instance
  # Above this: query performance degrades, memory usage spikes
```

---

## 3. Prometheus in Kubernetes — The Production Setup

### kube-prometheus-stack (The Standard)

```
At NovaMart, we don't deploy raw Prometheus.
We deploy kube-prometheus-stack (Helm chart) which includes:

┌────────────────────────────────────────────────────────────┐
│              kube-prometheus-stack                          │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ Prometheus   │  │ Alertmanager │  │ Grafana          │ │
│  │ Operator     │  │ (HA: 3      │  │ + pre-built      │ │
│  │              │  │  replicas)   │  │   dashboards     │ │
│  │ Manages      │  │              │  │                  │ │
│  │ Prometheus   │  │ Routes       │  │ K8s, node,       │ │
│  │ instances    │  │ alerts to    │  │ namespace,       │ │
│  │ via CRDs     │  │ PagerDuty/   │  │ workload         │ │
│  │              │  │ Slack        │  │ dashboards       │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ node-        │  │ kube-state-  │  │ Prometheus       │ │
│  │ exporter     │  │ metrics      │  │ (2 replicas      │ │
│  │              │  │              │  │  or Thanos        │ │
│  │ DaemonSet    │  │ Deployment   │  │  sidecar for HA) │ │
│  │ on every     │  │              │  │                  │ │
│  │ node         │  │ Reads K8s    │  │ TSDB + WAL       │ │
│  │              │  │ API, exposes │  │ persistent        │ │
│  │ Exposes:     │  │ as metrics   │  │ storage (EBS)    │ │
│  │ CPU, mem,    │  │              │  │                  │ │
│  │ disk, net    │  │ Exposes:     │  │                  │ │
│  │ per NODE     │  │ pod status,  │  │                  │ │
│  │              │  │ deployment   │  │                  │ │
│  │              │  │ replicas,    │  │                  │ │
│  │              │  │ job status,  │  │                  │ │
│  │              │  │ PVC, etc.    │  │                  │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
└────────────────────────────────────────────────────────────┘

CRITICAL DISTINCTION:
  node-exporter:     HARDWARE metrics (CPU, memory, disk, network — per NODE)
  kube-state-metrics: KUBERNETES metrics (pod state, deployment replicas — per K8s OBJECT)
  cAdvisor (kubelet): CONTAINER metrics (CPU, memory, network — per CONTAINER/POD)

  You need ALL THREE for complete visibility.
```

### Installation

```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install with custom values
helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f values-monitoring.yaml \
  --version 56.6.2
```

```yaml
# values-monitoring.yaml (NovaMart production config)
---
# Prometheus configuration
prometheus:
  prometheusSpec:
    # Retention
    retention: 15d
    retentionSize: "45GB"
    
    # Resources — Prometheus is memory-hungry
    resources:
      requests:
        cpu: "2"
        memory: 8Gi
      limits:
        cpu: "4"
        memory: 16Gi
    
    # Storage — MUST use persistent storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    
    # Discover ALL ServiceMonitors and PodMonitors in ALL namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    
    # Scrape interval
    scrapeInterval: "30s"
    evaluationInterval: "30s"
    
    # External labels (for Thanos/remote-write identification)
    externalLabels:
      cluster: novamart-prod-us-east-1
      environment: production
      region: us-east-1
    
    # Replicas for HA
    replicas: 2
    
    # Anti-affinity — don't schedule both replicas on same node
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app.kubernetes.io/name: prometheus
            topologyKey: kubernetes.io/hostname

# Alertmanager configuration
alertmanager:
  alertmanagerSpec:
    replicas: 3
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 10Gi

# Grafana
grafana:
  enabled: true
  adminPassword: "{{ vault_grafana_admin_password }}"
  persistence:
    enabled: true
    size: 10Gi
  
  # Ingress for Grafana UI
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - grafana.novamart.internal
    tls:
      - secretName: grafana-tls
        hosts:
          - grafana.novamart.internal

# Node Exporter (DaemonSet on every node)
nodeExporter:
  enabled: true

# kube-state-metrics
kubeStateMetrics:
  enabled: true

# Default alerting rules (200+ pre-built rules)
defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: true
    general: true
    k8s: true
    kubeApiserver: true
    kubeApiserverAvailability: true
    kubeApiserverSlos: true
    kubelet: true
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    node: true
    nodeExporterAlerting: true
    prometheus: true
```

### Service Discovery — How Prometheus Finds Targets

```yaml
# The Prometheus Operator introduces CRDs that tell Prometheus what to scrape:

# 1. ServiceMonitor — scrape pods behind a Service
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service
  namespace: monitoring
  labels:
    release: kube-prom-stack    # Must match Prometheus serviceMonitorSelector
spec:
  namespaceSelector:
    matchNames:
      - production
  selector:
    matchLabels:
      app: order-service        # Matches the Service's labels
  endpoints:
    - port: metrics             # Name of the port in the Service spec
      interval: 15s             # Override default scrape interval
      path: /metrics            # Default, but explicit is good
      scrapeTimeout: 10s
      metricRelabelings:
        # Drop high-cardinality metrics you don't need
        - sourceLabels: [__name__]
          regex: "go_gc_.*"
          action: drop

---
# 2. PodMonitor — scrape pods directly (no Service needed)
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: istio-envoy
  namespace: monitoring
spec:
  namespaceSelector:
    any: true                   # All namespaces
  selector:
    matchLabels:
      security.istio.io/tlsMode: istio
  podMetricsEndpoints:
    - port: http-envoy-prom
      path: /stats/prometheus
      interval: 30s

---
# 3. PrometheusRule — define alerting and recording rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-service-alerts
  namespace: monitoring
  labels:
    release: kube-prom-stack
spec:
  groups:
    - name: order-service.rules
      rules:
        # Recording rule — pre-compute expensive query
        - record: order_service:http_error_rate:5m
          expr: |
            sum(rate(http_requests_total{service="order-service", status=~"5.."}[5m]))
            / sum(rate(http_requests_total{service="order-service"}[5m]))
        
        # Alerting rule
        - alert: OrderServiceHighErrorRate
          expr: order_service:http_error_rate:5m > 0.05
          for: 5m
          labels:
            severity: critical
            service: order-service
            team: order-team
          annotations:
            summary: "Order service error rate is {{ $value | humanizePercentage }}"
            description: >
              Error rate for order-service has been above 5% for 5 minutes.
              Current value: {{ $value | humanizePercentage }}.
              This impacts customer checkout flow.
            runbook_url: https://runbooks.novamart.internal/order-service/high-error-rate
            dashboard_url: https://grafana.novamart.internal/d/order-service
```

---

## 4. PromQL — The Query Language

### Fundamentals

```
INSTANT VECTOR — a single value per time series at one point in time:
  http_requests_total{service="order-service"}
  → Returns current counter value for each label combination

RANGE VECTOR — a series of values over a time window:
  http_requests_total{service="order-service"}[5m]
  → Returns all samples from the last 5 minutes
  → CANNOT be graphed directly (use rate(), avg_over_time(), etc.)

SCALAR — a single numeric value:
  count(up{job="node-exporter"})
  → Returns: 47 (number of nodes being scraped)
```

### Essential PromQL for NovaMart

```promql
# ═══════════════════════════════════════════════════
# REQUEST RATE (The Golden Signal: Traffic)
# ═══════════════════════════════════════════════════

# Requests per second, per service
sum(rate(http_requests_total[5m])) by (service)

# Total cluster-wide RPS
sum(rate(http_requests_total[5m]))

# RPS by status code class
sum(rate(http_requests_total[5m])) by (status)
# More useful — group by 2xx, 4xx, 5xx:
sum(rate(http_requests_total{status=~"2.."}[5m])) # Success
sum(rate(http_requests_total{status=~"4.."}[5m])) # Client errors
sum(rate(http_requests_total{status=~"5.."}[5m])) # Server errors


# ═══════════════════════════════════════════════════
# ERROR RATE (The Golden Signal: Errors)
# ═══════════════════════════════════════════════════

# Error rate as a percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m])) * 100

# Error rate per service (for SLO tracking)
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/ sum(rate(http_requests_total[5m])) by (service)

# Only services with error rate > 1%
(
  sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
  / sum(rate(http_requests_total[5m])) by (service)
) > 0.01


# ═══════════════════════════════════════════════════
# LATENCY (The Golden Signal: Latency)
# ═══════════════════════════════════════════════════

# p50 (median) latency
histogram_quantile(0.50, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# p95 latency
histogram_quantile(0.95, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# p99 latency
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# Average latency (simpler but hides tail latency)
sum(rate(http_request_duration_seconds_sum[5m])) by (service)
/ sum(rate(http_request_duration_seconds_count[5m])) by (service)

# WHY P99 MATTERS MORE THAN AVERAGE:
# Average latency: 50ms (looks great!)
# p99 latency: 4200ms (1% of your users wait 4+ seconds)
# At 50M users, 1% = 500,000 users with terrible experience


# ═══════════════════════════════════════════════════
# SATURATION (The Golden Signal: Saturation)
# ═══════════════════════════════════════════════════

# CPU usage per pod (from cAdvisor via kubelet)
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)
/ sum(kube_pod_container_resource_requests{resource="cpu"}) by (pod, namespace)

# Memory usage vs limit
sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace)
/ sum(kube_pod_container_resource_limits{resource="memory"}) by (pod, namespace)

# Disk usage percentage
1 - (node_filesystem_avail_bytes{mountpoint="/"} 
     / node_filesystem_size_bytes{mountpoint="/"})

# CPU throttling (THE HIDDEN KILLER)
sum(rate(container_cpu_cfs_throttled_periods_total[5m])) by (pod)
/ sum(rate(container_cpu_cfs_periods_total[5m])) by (pod) > 0.25
# Pods with >25% of their CPU periods throttled


# ═══════════════════════════════════════════════════
# KUBERNETES-SPECIFIC QUERIES
# ═══════════════════════════════════════════════════

# Pods not ready
kube_pod_status_ready{condition="false"} == 1

# Pods in CrashLoopBackOff
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1

# Deployments with unavailable replicas
kube_deployment_status_replicas_unavailable > 0

# PVCs approaching capacity
kubelet_volume_stats_used_bytes 
/ kubelet_volume_stats_capacity_bytes > 0.85

# Nodes not ready
kube_node_status_condition{condition="Ready", status="true"} == 0

# Container restarts (high restart count = instability)
increase(kube_pod_container_status_restarts_total[1h]) > 5

# HPA at max replicas (scaling ceiling hit)
kube_horizontalpodautoscaler_status_current_replicas 
== kube_horizontalpodautoscaler_spec_max_replicas


# ═══════════════════════════════════════════════════
# NODE-LEVEL QUERIES
# ═══════════════════════════════════════════════════

# CPU usage per node
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)

# Memory usage per node
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Disk I/O utilization (% of time disk is busy)
rate(node_disk_io_time_seconds_total[5m])

# Network errors
rate(node_network_receive_errs_total[5m]) > 0
rate(node_network_transmit_errs_total[5m]) > 0
```

### PromQL Gotchas

```
GOTCHA 1: rate() requires a RANGE, not an instant
  WRONG: rate(http_requests_total)
  RIGHT: rate(http_requests_total[5m])
  
GOTCHA 2: rate() should only be used on COUNTERS
  WRONG: rate(node_memory_MemAvailable_bytes[5m])  ← gauge!
  RIGHT: deriv(node_memory_MemAvailable_bytes[5m])  ← rate of change for gauge

GOTCHA 3: Range window must be ≥ 2x scrape interval
  If scrape interval = 30s, minimum useful range = 1m
  rate(metric[30s]) with 30s scrape = only 1 sample = unreliable
  rate(metric[1m])  with 30s scrape = 2 samples = minimum viable
  rate(metric[5m])  with 30s scrape = 10 samples = smooth, recommended

GOTCHA 4: sum() by (label) vs sum() without (label)
  sum(rate(x[5m])) by (service)    → one value PER service
  sum(rate(x[5m])) without (pod)   → sum everything EXCEPT per-pod

GOTCHA 5: absent() for detecting missing metrics
  # Alert if order-service stops reporting metrics entirely:
  absent(up{job="order-service"} == 1)
  # Fires when NO time series matches — the job has disappeared

GOTCHA 6: histogram_quantile with sum by (le)
  WRONG: histogram_quantile(0.99, http_request_duration_seconds_bucket)
  # This gives p99 PER time series (per pod), not overall
  
  RIGHT: histogram_quantile(0.99, 
    sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
  )
  # This gives p99 ACROSS all pods — the true service-level p99
  # The "by (le)" is MANDATORY — histogram_quantile needs the le label

GOTCHA 7: The "stale" problem
  If a pod is terminated, its metrics become "stale" after 5 minutes.
  During those 5 minutes, the old data is still returned in queries.
  rate() handles this correctly (returns nothing for stale series).
  But gauge queries may show stale data.
  
  Fix: Use the "up" metric to filter:
  node_memory_MemAvailable_bytes and on(instance) up == 1
```

### Recording Rules — Pre-computed Queries

```yaml
# WHY:
# Complex PromQL queries that run on every dashboard load or every 
# alert evaluation are expensive. If 10 dashboards and 5 alerts
# all compute the same error rate, that's 15 evaluations of the same query.
#
# Recording rules pre-compute and store the result as a new metric.
# Dashboards and alerts read the pre-computed metric — instant.

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: novamart-recording-rules
  namespace: monitoring
spec:
  groups:
    - name: novamart.slo.rules
      interval: 30s    # Evaluate every 30 seconds
      rules:
        # Error rate per service (used by 12 dashboards + 3 alerts)
        - record: novamart:http_error_rate:5m
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
            / sum(rate(http_requests_total[5m])) by (service)

        # Request rate per service
        - record: novamart:http_request_rate:5m
          expr: |
            sum(rate(http_requests_total[5m])) by (service)

        # p99 latency per service
        - record: novamart:http_latency_p99:5m
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
            )

        # CPU saturation per namespace
        - record: novamart:namespace_cpu_utilization:5m
          expr: |
            sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (namespace)
            / sum(kube_pod_container_resource_requests{resource="cpu"}) by (namespace)

# NAMING CONVENTION:
#   level:metric_name:operations
#   novamart:http_error_rate:5m
#   │         │               │
#   │         │               └── aggregation window
#   │         └── what it measures
#   └── scope/org prefix
```

---

## 5. Alertmanager — Routing Alerts to Humans

### Configuration

```yaml
# alertmanager.yml (deployed via Helm values or AlertmanagerConfig CRD)
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/T00/B00/xxxx'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Inhibition: suppress child alerts when parent fires
inhibit_rules:
  # If the entire cluster is down, don't page for individual services
  - source_matchers:
      - severity = critical
      - alertname = KubeClusterDown
    target_matchers:
      - severity =~ "warning|critical"
    equal: ['cluster']
  
  # If a node is down, suppress pod alerts on that node
  - source_matchers:
      - alertname = NodeDown
    target_matchers:
      - alertname =~ "Pod.*"
    equal: ['node']

# Routing tree
route:
  # Default receiver
  receiver: 'slack-platform-alerts'
  
  # Group alerts by these labels
  group_by: ['alertname', 'service', 'namespace']
  
  # Wait before sending initial notification (batch alerts)
  group_wait: 30s
  
  # Wait before sending updates to an existing group
  group_interval: 5m
  
  # Wait before re-sending an already-sent alert
  repeat_interval: 4h

  # Child routes (first match wins)
  routes:
    # SEV1/Critical → PagerDuty (wake people up)
    - matchers:
        - severity = critical
      receiver: 'pagerduty-critical'
      group_wait: 10s              # Send critical alerts FAST
      repeat_interval: 30m         # Re-page every 30 min until resolved
      routes:
        # Payment service critical → payment on-call specifically
        - matchers:
            - service = payment-service
          receiver: 'pagerduty-payment-oncall'

    # SEV2/Warning → Slack with @channel
    - matchers:
        - severity = warning
      receiver: 'slack-platform-warnings'
      repeat_interval: 2h

    # Watchdog (dead man's switch) → OpsGenie heartbeat
    - matchers:
        - alertname = Watchdog
      receiver: 'opsgenie-heartbeat'
      repeat_interval: 1m

receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '<PAGERDUTY_SERVICE_KEY>'
        severity: critical
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        details:
          service: '{{ .GroupLabels.service }}'
          namespace: '{{ .GroupLabels.namespace }}'
          runbook: '{{ .CommonAnnotations.runbook_url }}'
          dashboard: '{{ .CommonAnnotations.dashboard_url }}'

  - name: 'pagerduty-payment-oncall'
    pagerduty_configs:
      - service_key: '<PAYMENT_TEAM_PD_KEY>'
        severity: critical

  - name: 'slack-platform-alerts'
    slack_configs:
      - channel: '#platform-alerts'
        send_resolved: true
        title: '{{ if eq .Status "firing" }}🔥{{ else }}✅{{ end }} {{ .GroupLabels.alertname }}'
        text: >-
          *Service:* {{ .GroupLabels.service }}
          *Severity:* {{ .GroupLabels.severity }}
          *Summary:* {{ .CommonAnnotations.summary }}
          *Description:* {{ .CommonAnnotations.description }}
          *Runbook:* {{ .CommonAnnotations.runbook_url }}

  - name: 'slack-platform-warnings'
    slack_configs:
      - channel: '#platform-warnings'
        send_resolved: true

  - name: 'opsgenie-heartbeat'
    webhook_configs:
      - url: 'https://api.opsgenie.com/v2/heartbeats/novamart-prometheus/ping'
        send_resolved: false
```

### The Watchdog Pattern (Dead Man's Switch)

```yaml
# A Watchdog alert ALWAYS fires. If it STOPS firing, 
# your monitoring is broken.

# PrometheusRule:
- alert: Watchdog
  expr: vector(1)    # Always true
  labels:
    severity: none
  annotations:
    summary: "Watchdog alert — monitoring is alive"

# Alertmanager routes this to an external heartbeat service.
# If the heartbeat service stops receiving pings → monitoring is dead
# → separate alerting system notifies humans.

# This solves: "Who monitors the monitor?"
# Answer: An external service that expects regular pings.
# If pings stop → your Prometheus/Alertmanager stack is down.
```

---

## 6. How Prometheus Breaks — Production Failure Modes

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| **OOM killed** | Prometheus pod restarts, gaps in metrics | Too many active series, expensive queries, insufficient memory | Reduce cardinality, add recording rules, increase memory, drop unused metrics via metric_relabel_configs |
| **Disk full** | Prometheus stops accepting new samples, WAL errors | Retention too long, high cardinality = more disk, compaction falling behind | Reduce retention, reduce cardinality, increase PVC size, move to remote write (Thanos/Mimir) |
| **Slow queries** | Grafana dashboards timeout, 504 errors | Querying millions of series without aggregation, missing recording rules, no `by (label)` clause | Add recording rules for dashboards, reduce query scope with label selectors, set query timeout |
| **Scrape failures** | `up == 0`, gaps in specific target metrics | Target pod crashed, network policy blocking, metrics endpoint wrong path/port, TLS mismatch | Check `up` metric per target, verify ServiceMonitor matches Service port name, check NetworkPolicy |
| **Stale data** | Dashboards show data for pods that no longer exist | Default 5-minute staleness period, gauge queries don't handle staleness | Filter with `up == 1`, use `absent()` for alerting, Prometheus handles this automatically for rate() |
| **High cardinality explosion** | Memory spike, TSDB head series growing rapidly | New deployment added user_id/request_id as metric label, unbounded label values | `topk(10, count by (__name__)({__name__=~".+"}))` to find offenders, metric_relabel_configs to drop |
| **WAL corruption** | Prometheus won't start after crash, `error opening WAL` | Node crash during write, EBS detach, disk failure | Delete WAL directory (lose last 2h of data), restart. Enable WAL compression. |
| **Alertmanager split-brain** | Duplicate pages, or missing pages | HA Alertmanager cluster lost gossip connectivity | Verify mesh networking between AM pods, check `alertmanager_cluster_members` metric |
| **Recording rule lag** | Dashboards show stale data, alert evaluation delayed | Rule group evaluation taking longer than interval | Check `prometheus_rule_group_last_duration_seconds`, reduce rules per group, optimize PromQL |
| **ServiceMonitor not picked up** | New service metrics not appearing | Missing labels for Prometheus to select it, wrong namespace | Check serviceMonitorSelector on Prometheus CR, verify labels match, check RBAC for cross-namespace |


---

## Quick Reference Card

```
THREE PILLARS:
  Metrics (Prometheus) → WHAT is happening (aggregates, time-series)
  Logs (Loki/ELK)     → WHAT happened (individual events with context)
  Traces (Jaeger/Tempo)→ HOW a request flows (distributed call graph)
  All three correlated in Grafana by time + trace_id

PROMETHEUS:
  Pull-based: scrapes /metrics from targets every 15-30s
  TSDB: head block (memory, 2h) → compressed blocks (disk)
  WAL: write-ahead log for crash recovery
  Retention: default 15d, tune for disk capacity

METRIC TYPES:
  Counter: only up (requests, errors) → USE rate()/increase()
  Gauge: up/down (memory, queue depth) → USE raw value or deriv()
  Histogram: bucketed observations → USE histogram_quantile()
  Summary: client-side quantiles → DON'T USE (not aggregatable)

FOUR GOLDEN SIGNALS (PromQL):
  Traffic:    sum(rate(http_requests_total[5m])) by (service)
  Errors:     rate(status=~"5..") / rate(total) * 100
  Latency:    histogram_quantile(0.99, sum(rate(bucket[5m])) by (le))
  Saturation: CPU throttling, memory vs limits, disk usage

CARDINALITY RULES:
  ✓ Labels: service, method, status, region, pod
  ✗ NEVER: user_id, IP, request_id, URL with params
  Monitor: prometheus_tsdb_head_series < 2M per instance

K8S METRICS SOURCES:
  node-exporter:      Node hardware (CPU, mem, disk, net)
  kube-state-metrics:  K8s objects (pods, deployments, PVCs)
  cAdvisor (kubelet):  Container resources (per-pod CPU, mem)

PROMQL GOTCHAS:
  rate() needs range [5m], only for counters
  Range ≥ 2x scrape interval
  histogram_quantile NEEDS "by (le)"
  absent() for "metric disappeared" alerts
  sum by (label) not sum without (label) for clarity

ALERTMANAGER:
  Route tree: first match wins
  group_by: [alertname, service] → batch related alerts
  group_wait: 30s (batch window)
  Inhibition: node down suppresses pod alerts
  Watchdog: always-firing alert → dead man's switch

RECORDING RULES:
  Pre-compute expensive queries → store as new metrics
  Naming: level:metric:operations
  Use for: dashboard queries used by >1 consumer, SLO metrics
```
# Phase 5, Lesson 1 — Retention Questions

## Prometheus + Metrics + PromQL + Alertmanager

---

### Q1: Cardinality Incident 🔥

**Scenario:** It's 2 AM. PagerDuty fires: Prometheus in `us-east-1` is OOM-killed and restarting in a loop. You check and see it was fine yesterday. A deploy went out at 11 PM adding a new custom metric to the `payment-svc`. The metric looks like this:

```
payment_transaction_duration_seconds{user_id="u-38291", endpoint="/pay", status="200", region="us-east-1", payment_method="credit_card", currency="USD"}
```

**Questions:**
1. What is the root cause? Be specific about WHY this kills Prometheus.
2. What is your **immediate** mitigation (Prometheus is down RIGHT NOW, alerts are dead)?
3. What is the **permanent fix** — show the exact Kubernetes resource/config you'd use to drop the problematic dimension before it hits Prometheus TSDB.
4. What metric would you monitor going forward to catch this before it becomes an outage?

---

### Q2: PromQL Under Pressure 🔥

**Scenario:** NovaMart's `order-svc` team reports that their p99 latency SLO is being breached. You need to investigate. The metric is:

```
http_request_duration_seconds_bucket{service="order-svc", le="..."}
```

**Write the exact PromQL queries for:**

1. P99 latency for `order-svc` over the last 5 minutes
2. P99 latency for `order-svc` **broken down by endpoint**, so you can find which endpoint is slow
3. The error rate (5xx responses as a percentage of total) for `order-svc` over the last 5 minutes
4. **Trap question:** A junior engineer writes this query and says "p99 looks fine to me":
   ```
   histogram_quantile(0.99, http_request_duration_seconds_bucket{service="order-svc"})
   ```
   This query will return a result, but it's **wrong**. Explain exactly what's wrong and what the result actually represents.

---

### Q3: Alertmanager Routing Logic 🔥

**Scenario:** You have this (simplified) Alertmanager config:

```yaml
route:
  receiver: 'slack-default'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-sev1'
      group_wait: 10s
      repeat_interval: 1h
    - match:
        severity: warning
        team: payments
      receiver: 'slack-payments'
    - match:
        severity: warning
      receiver: 'slack-warnings'
```

**Answer these:**

1. Alert fires: `severity=critical, team=payments, alertname=PaymentSvcDown`. Which receiver gets it?
2. Alert fires: `severity=warning, team=platform, alertname=HighMemory`. Which receiver gets it?
3. Alert fires: `severity=info, team=payments, alertname=DeployComplete`. Which receiver gets it?
4. Two critical alerts fire within 5 seconds of each other: `alertname=PaymentSvcDown, cluster=us-east-1` and `alertname=PaymentSvcDown, cluster=us-west-2`. How many PagerDuty notifications are sent, and when? Explain the grouping behavior.
5. Your Watchdog alert (always-firing) has `severity: none`. Is it routed to PagerDuty? Where does it go? Why does this matter?

---

### Q4: Prometheus Failure Mode Diagnosis 🔥

**Scenario:** Monday morning. You notice that over the weekend:
- Several alerts that **should have fired** did NOT fire
- Grafana dashboards show **gaps** in data (missing data points) for about 40% of your targets
- Prometheus pod did NOT restart (no OOM, no crash)
- `up` metric shows those targets were `up == 1` the whole time
- The Prometheus `/targets` page shows those targets as "UP" with `last scrape: 2s ago`

**This is contradictory — targets show as UP and being scraped, but data has gaps.**

1. What is the most likely root cause? (Hint: think about what happens when Prometheus can't keep up)
2. What specific Prometheus metrics would you query to confirm your theory?
3. How do you fix this — both immediate and long-term?

---

# Phase 5, Lesson 1 — Retention Answers

---

## Q1: Cardinality Incident

### 1. Root Cause

The `user_id` label is the killer. Every unique `user_id` × `endpoint` × `status` × `payment_method` × `currency` combination creates a **distinct time series** in the Prometheus TSDB head block, which lives **in RAM**.

NovaMart at scale: say 500K active users, 3 endpoints, 5 status codes, 4 payment methods, 3 currencies. That's:

```
500,000 × 3 × 5 × 4 × 3 = 90,000,000 series (from ONE metric)
```

Each series in the head block costs ~8KB of RAM. 90M series × 8KB = **~720 GB**. Prometheus typically has 16–64 GB. It's instant OOM.

**Why this kills Prometheus specifically:**
- Prometheus stores the **entire head block** (last 2 hours of all active series) in memory
- There is no spill-to-disk for the head block
- New series are indexed into an inverted index (also in memory)
- TSDB head chunk allocation is the dominant memory consumer
- Once memory exceeds cgroup limit → OOM-killed → restart → tries to replay WAL → WAL contains all the high-cardinality data → OOM again → **crash loop**

### 2. Immediate Mitigation (Prometheus is DOWN right now)

**Priority zero: restore alerting capability.** Without Prometheus, all alerts are dead. NovaMart is flying blind at $50K/min risk.

**Step 1 — Roll back the payment-svc deploy (run from your CI/CD or kubectl on the cluster):**

```bash
# On the cluster where payment-svc runs
kubectl -n payments rollout undo deployment/payment-svc
kubectl -n payments rollout status deployment/payment-svc --timeout=120s
```

This stops the bleeding — no new high-cardinality samples being generated.

**Step 2 — Delete the corrupted WAL so Prometheus can start without replaying poisoned data:**

```bash
# Exec into the Prometheus pod's persistent volume (or a debug pod mounting the PVC)
# If Prometheus is crash-looping, scale to 0 first:
kubectl -n monitoring scale statefulset/prometheus --replicas=0
kubectl -n monitoring wait --for=delete pod/prometheus-0 --timeout=60s

# Launch a debug pod mounting the PVC
kubectl -n monitoring run wal-cleanup --rm -it \
  --image=busybox \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "wal-cleanup",
        "image": "busybox",
        "command": ["sh"],
        "stdin": true,
        "tty": true,
        "volumeMounts": [{
          "name": "prometheus-data",
          "mountPath": "/prometheus"
        }]
      }],
      "volumes": [{
        "name": "prometheus-data",
        "persistentVolumeClaim": {
          "claimName": "prometheus-data-prometheus-0"
        }
      }]
    }
  }'

# Inside the debug pod:
rm -rf /prometheus/wal/*
rm -rf /prometheus/chunks_head/*
exit
```

```bash
# Bring Prometheus back
kubectl -n monitoring scale statefulset/prometheus --replicas=1
kubectl -n monitoring wait --for=condition=Ready pod/prometheus-0 --timeout=120s
```

**⚠️ Trade-off:** You lose ~2 hours of recent metrics data (the head block). This is acceptable — the alternative is zero monitoring.

**Step 3 — Verify Prometheus is healthy:**

```bash
kubectl -n monitoring port-forward svc/prometheus 9090:9090 &
curl -s http://localhost:9090/-/healthy
# Expected: "Prometheus Server is Healthy."

curl -s http://localhost:9090/api/v1/status/tsdb | jq '.data.headStats'
# Verify seriesCountByMetricName doesn't show the payment metric
```

### 3. Permanent Fix — Metric Relabeling to Drop user_id

**Use a `metric_relabel_configs` rule in the Prometheus scrape config.** This drops the label **after** scrape, **before** TSDB ingestion.

If using **Prometheus Operator** (kube-prometheus-stack), apply a `ServiceMonitor` or `PodMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-svc
  namespace: payments
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: payment-svc
  endpoints:
    - port: metrics
      interval: 15s
      metricRelabelings:
        # DROP the user_id label entirely from all metrics
        - sourceLabels: [__name__]
          regex: 'payment_transaction_duration_seconds.*'
          action: keep
        - action: labeldrop
          regex: 'user_id'
        # SAFETY NET: drop any metric with >1000 series per scrape
        # (requires Prometheus 2.45+)
```

Apply it:

```bash
kubectl apply -f servicemonitor-payment-svc.yaml

# Verify the ServiceMonitor is picked up
kubectl -n monitoring get prometheus -o jsonpath='{.items[0].status.conditions}' | jq .
```

**If using raw `prometheus.yml`** (e.g., in a ConfigMap):

```yaml
scrape_configs:
  - job_name: 'payment-svc'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names: ['payments']
    metric_relabel_configs:
      # Drop user_id label from all payment metrics
      - action: labeldrop
        regex: 'user_id'
      # Nuclear option: drop the entire metric if it still has too many series
      # - source_labels: [__name__]
      #   regex: 'payment_transaction_duration_seconds'
      #   action: drop
    sample_limit: 50000  # Per-scrape safety net
```

```bash
# Reload Prometheus config (no restart needed)
curl -X POST http://localhost:9090/-/reload

# Verify config was accepted
curl -s http://localhost:9090/api/v1/status/config | jq '.data.yaml' | head -50
```

**But the real permanent fix is also in the application code.** The `payment-svc` team must remove `user_id` from the metric labels:

```go
// WRONG — unbounded cardinality
paymentDuration.WithLabelValues(userID, endpoint, status, region, paymentMethod, currency).Observe(duration)

// CORRECT — bounded cardinality
paymentDuration.WithLabelValues(endpoint, status, region, paymentMethod, currency).Observe(duration)
```

User-level latency tracking belongs in **traces** (Jaeger/Tempo), not **metrics**.

### 4. Forward-Looking Monitoring

Monitor these metrics on Prometheus itself to catch cardinality bombs before OOM:

```yaml
# Alert rule: cardinality growth rate
groups:
  - name: prometheus-cardinality
    rules:
      - alert: HighCardinalityGrowth
        expr: |
          delta(prometheus_tsdb_head_series[1h]) > 100000
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "TSDB head series grew by {{ $value }} in the last hour"
          runbook: "https://wiki.novamart.internal/runbooks/cardinality-bomb"

      - alert: PrometheusNearMemoryLimit
        expr: |
          process_resident_memory_bytes{job="prometheus"}
            / on() kube_pod_container_resource_limits{resource="memory", container="prometheus"}
          > 0.80
        for: 10m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Prometheus using {{ $value | humanizePercentage }} of memory limit"

      - alert: ScrapeSampleLimitHit
        expr: |
          rate(prometheus_target_scrapes_exceeded_sample_limit_total[5m]) > 0
        for: 1m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Scrape targets hitting sample_limit — possible cardinality issue"
```

**The single most important metric:**

```promql
prometheus_tsdb_head_series
```

Track this on a separate monitoring system (Grafana Cloud free tier, Datadog agent, or a second lightweight Prometheus) — because if your primary Prometheus dies, you need to still see this.

---

## Q2: PromQL Under Pressure

### 1. P99 Latency — order-svc — Last 5 Minutes

```promql
histogram_quantile(
  0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{service="order-svc"}[5m])
  )
)
```

**Why `sum by (le)`:** Collapses all other label dimensions (instance, pod, endpoint) so you get a single p99 across all order-svc instances. The `le` label **must** be preserved — it defines the bucket boundaries that `histogram_quantile` needs.

**Why `rate()` inside:** Histogram buckets are counters. `rate()` converts cumulative counts into per-second rates over the 5-minute window.

### 2. P99 Latency — Broken Down by Endpoint

```promql
histogram_quantile(
  0.99,
  sum by (le, endpoint) (
    rate(http_request_duration_seconds_bucket{service="order-svc"}[5m])
  )
)
```

**Only change:** added `endpoint` to the `sum by` clause. This preserves the endpoint dimension, giving you one p99 value per endpoint. You'll immediately see which endpoint (e.g., `/checkout` vs `/search`) is breaching the SLO.

### 3. Error Rate (5xx / Total) — Last 5 Minutes

```promql
sum by (service) (
  rate(http_requests_total{service="order-svc", status=~"5.."}[5m])
)
/
sum by (service) (
  rate(http_requests_total{service="order-svc"}[5m])
)
```

**Note:** This assumes a separate counter `http_requests_total` with a `status` label. If the team only exposes the histogram, you'd use the `+Inf` bucket as a total count:

```promql
sum(rate(http_request_duration_seconds_bucket{service="order-svc", le="+Inf", status=~"5.."}[5m]))
/
sum(rate(http_request_duration_seconds_bucket{service="order-svc", le="+Inf"}[5m]))
```

### 4. Trap Question — What's Wrong

The junior's query:

```promql
histogram_quantile(0.99, http_request_duration_seconds_bucket{service="order-svc"})
```

**Two critical bugs:**

**Bug 1 — No `rate()`.** Histogram buckets are **cumulative counters** that increase monotonically since process start. Without `rate()`, `histogram_quantile` is computing the quantile over the **total cumulative count of requests since the process started** — not over any recent time window. The result:

- Heavily dominated by historical data — recent latency spikes are diluted
- Appears artificially stable/low because millions of old requests drown out recent ones
- After a pod restart, counters reset to 0, and the calculation becomes completely nonsensical (negative rates implicitly, or the quantile jumps wildly)

**Bug 2 — No `sum by (le)`.** Without aggregation, `histogram_quantile` is computed **per individual time series** (per pod, per instance, per endpoint). The result is multiple p99 values — one per unique label combination — not a single aggregated p99. If the junior is looking at this in Grafana, they might see one of many lines and think it's "the" p99, when it's actually the p99 for a single pod.

**What the junior actually sees:** A deceptively smooth, slowly-changing value that looks "fine" — because the cumulative nature of raw counters acts as a massive moving average stretching back to process start time. It hides real-time latency spikes entirely.

**Correct query for reference:**

```promql
histogram_quantile(
  0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{service="order-svc"}[5m])
  )
)
```

---

## Q3: Alertmanager Routing Logic

Alertmanager routes are **first-match, depth-first**. Once a route matches, child routes of siblings are not evaluated (unless `continue: true` is set, which it isn't here).

```
route (default: slack-default)
├── [1] severity=critical        → pagerduty-sev1
├── [2] severity=warning AND team=payments → slack-payments
└── [3] severity=warning         → slack-warnings
```

### 1. `severity=critical, team=payments, alertname=PaymentSvcDown`

**→ `pagerduty-sev1`**

Route [1] matches on `severity=critical`. First match wins. The `team=payments` label is irrelevant — route [1] doesn't check it, and Alertmanager stops traversing after the first match.

### 2. `severity=warning, team=platform, alertname=HighMemory`

**→ `slack-warnings`**

- Route [1]: No — `severity` is `warning`, not `critical`
- Route [2]: No — `severity=warning` matches, but `team=platform` ≠ `team=payments`. **Both** `match` conditions must be satisfied (AND logic)
- Route [3]: Yes — `severity=warning` matches

### 3. `severity=info, team=payments, alertname=DeployComplete`

**→ `slack-default`**

- Route [1]: No — not `critical`
- Route [2]: No — not `warning`
- Route [3]: No — not `warning`
- No child route matches → **falls through to the root route's receiver**: `slack-default`

### 4. Two Critical Alerts, Grouping Behavior

```
t=0s   Alert: alertname=PaymentSvcDown, cluster=us-east-1, severity=critical
t=5s   Alert: alertname=PaymentSvcDown, cluster=us-west-2, severity=critical
```

Root route has `group_by: ['alertname', 'cluster']`. This is **inherited** by child routes unless overridden (it's not overridden in route [1]).

**Two different groups** are created because the `cluster` label differs:

```
Group A: {alertname="PaymentSvcDown", cluster="us-east-1"}
Group B: {alertname="PaymentSvcDown", cluster="us-west-2"}
```

The critical route has `group_wait: 10s`.

**Timeline:**

```
t=0s    Alert A arrives → Group A created → 10s group_wait timer starts
t=5s    Alert B arrives → Group B created → 10s group_wait timer starts
t=10s   Group A timer fires → PagerDuty notification #1 (us-east-1)
t=15s   Group B timer fires → PagerDuty notification #2 (us-west-2)
```

**Answer: TWO PagerDuty notifications**, at t=10s and t=15s.

**If the cluster label were the same**, both alerts would be in the same group, and only ONE notification would be sent at t=10s, containing both alerts bundled together.

**This is a real production design decision:** grouping by `cluster` means your on-call gets per-region pages. If both regions are down, they get two pages — which is actually correct because it might require two separate remediation actions in two different regions.

### 5. Watchdog Alert (`severity: none`)

**→ `slack-default`**

- Route [1]: No — not `critical`
- Route [2]: No — not `warning`
- Route [3]: No — not `warning`
- Falls to root → `slack-default`

**It is NOT routed to PagerDuty.** This is correct behavior.

**Why Watchdog matters:** It's a **dead man's switch**. The Watchdog alert is always-firing by design. If your `#slack-default` channel **stops** receiving Watchdog notifications, it means one of:

1. Alertmanager is down
2. Prometheus is down
3. The Prometheus → Alertmanager → Slack pipeline is broken

**This is your "alerting on alerting" mechanism.** Without it, your entire monitoring stack can be dead and nobody knows until a customer calls.

**Production hardening:** Watchdog should ideally be routed to a **dead man's switch service** (e.g., PagerDuty's heartbeat monitoring, Healthchecks.io, or Cronitor) that pages you if it **stops receiving** the signal:

```yaml
routes:
  - match:
      alertname: Watchdog
    receiver: 'deadmans-switch'
    repeat_interval: 1m
    group_wait: 0s
  - match:
      severity: critical
    # ... rest of routes
```

This route must be **first** so it matches before anything else.

---

## Q4: Prometheus Failure Mode Diagnosis

### 1. Root Cause: TSDB Head Compaction / Ingestion Backpressure

The contradiction — targets show UP and being scraped, but data has gaps — points to one thing: **Prometheus is successfully scraping targets but failing to append samples to the TSDB.**

Here's the mechanism:

```
Scrape Loop                          TSDB
──────────                          ────
HTTP GET /metrics  ──→  Parse       ──→  Append to Head Block
       ↑                                       ↑
  (this succeeds,                    (THIS FAILS — samples
   so up=1 and                        are silently dropped)
   /targets = UP)
```

**Most likely specific cause:** TSDB **head compaction** running over the weekend. When the head block compacts (writes to persistent blocks on disk), the compaction can:

1. **Block new appends** if disk I/O is saturated (especially on network-attached EBS volumes)
2. **Cause the sample appender to hit the WAL sync deadline** and drop samples
3. **Trigger chunk memory-mapping** that slows down the append path

**Why 40% and not 100%:** High-cardinality targets (more series per scrape) are disproportionately affected because they require more append operations per scrape cycle. Low-cardinality targets (few series) sneak through between compaction I/O bursts.

**Why over the weekend:** Compaction is triggered by time-based block boundaries (default 2h blocks). If a large number of blocks needed compaction (e.g., retention policy causing block merges), and I/O was already constrained, the compaction storm compounds.

### 2. Confirming Metrics

Query these **on Prometheus itself** (or from a federated/remote Prometheus if available):

```promql
# 1. Sample append failures — THE smoking gun
rate(prometheus_tsdb_head_samples_appended_total[1h])
```
If this shows dips or flatlines during the gap periods, confirmed.

```promql
# 2. Out-of-order sample drops
rate(prometheus_tsdb_out_of_order_samples_total[1h])
```
Spikes here mean Prometheus received samples but couldn't place them in the TSDB timeline.

```promql
# 3. Compaction duration — look for long-running compactions
prometheus_tsdb_compaction_duration_seconds
```

```promql
# 4. WAL corruption or truncation issues
rate(prometheus_tsdb_wal_truncate_duration_seconds_sum[1h])
rate(prometheus_tsdb_wal_corruptions_total[1h])
```

```promql
# 5. Scrape duration vs interval — are scrapes backing up?
prometheus_target_interval_length_seconds{quantile="0.99"}
```
If this exceeds your `scrape_interval` (e.g., 15s), scrapes are falling behind.

```promql
# 6. Head chunk write queue
prometheus_tsdb_head_chunks_created_total
prometheus_tsdb_head_gc_duration_seconds
```

```promql
# 7. Process-level resource pressure
rate(process_cpu_seconds_total{job="prometheus"}[5m])
process_resident_memory_bytes{job="prometheus"}
```

**Quick diagnosis script (run where you can reach Prometheus API):**

```bash
#!/usr/bin/env bash
PROM="http://localhost:9090"

echo "=== Sample Append Rate (should be steady) ==="
curl -s "$PROM/api/v1/query?query=rate(prometheus_tsdb_head_samples_appended_total[1h])" | jq '.data.result[0].value[1]'

echo "=== Out-of-Order Samples (should be 0) ==="
curl -s "$PROM/api/v1/query?query=rate(prometheus_tsdb_out_of_order_samples_total[1h])" | jq '.data.result[0].value[1]'

echo "=== Head Series Count ==="
curl -s "$PROM/api/v1/query?query=prometheus_tsdb_head_series" | jq '.data.result[0].value[1]'

echo "=== Compaction Duration (last) ==="
curl -s "$PROM/api/v1/query?query=prometheus_tsdb_compaction_duration_seconds" | jq '.data.result'

echo "=== WAL Corruptions ==="
curl -s "$PROM/api/v1/query?query=prometheus_tsdb_wal_corruptions_total" | jq '.data.result[0].value[1]'

echo "=== TSDB Status ==="
curl -s "$PROM/api/v1/status/tsdb" | jq '.data.headStats'
```

### 3. Fix — Immediate and Long-Term

**Immediate (stop the data gaps RIGHT NOW):**

```bash
# 1. Check current compaction state
curl -s http://localhost:9090/api/v1/status/tsdb | jq .

# 2. If disk I/O is the bottleneck (EBS), resize the volume for more IOPS
# For gp3 volumes, you can increase IOPS without downtime:
VOLUME_ID=$(aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/created-for/pvc/name,Values=prometheus-data-prometheus-0" \
  --query 'Volumes[0].VolumeId' --output text)

aws ec2 modify-volume \
  --volume-id "$VOLUME_ID" \
  --volume-type gp3 \
  --iops 6000 \
  --throughput 400

# Verify modification
aws ec2 describe-volumes-modifications \
  --volume-ids "$VOLUME_ID" \
  --query 'VolumesModifications[0].ModificationState'
# Wait for "completed" or "optimizing"

# 3. If memory-pressure: bump Prometheus memory limit
kubectl -n monitoring patch statefulset prometheus \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"32Gi"}]'
```

**Long-Term:**

```
┌─────────────────────────────────────────────────────────────┐
│                  LAYERED SOLUTION                            │
├─────────────────────────────────────────────────────────────┤
│ Layer 1: STORAGE                                            │
│  - Use gp3 EBS with provisioned IOPS (3000+)               │
│  - Or use local NVMe (i3/i4 instances) for TSDB             │
│  - Set --storage.tsdb.min-block-duration=2h                 │
│  - Set --storage.tsdb.max-block-duration=2h (prevents       │
│    mega-compactions that block ingestion)                    │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: SHARDING                                           │
│  - Split scrape targets across multiple Prometheus          │
│    instances using hashmod relabeling                       │
│  - Use Thanos or Cortex for query federation                │
│  - Each shard handles fewer series → less compaction load   │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: ALERTING ON PROMETHEUS ITSELF                      │
│  - Alert on sample_appended rate drops                      │
│  - Alert on compaction duration spikes                      │
│  - Alert on WAL size growth                                 │
│  - These alerts must fire from a SEPARATE Prometheus        │
│    (meta-monitoring)                                        │
├─────────────────────────────────────────────────────────────┤
│ Layer 4: META-MONITORING                                    │
│  - Second Prometheus instance scraping the first            │
│  - OR Grafana Cloud free tier as external watchdog          │
│  - Catches exactly this scenario: primary is "up" but       │
│    silently degraded                                        │
└─────────────────────────────────────────────────────────────┘
```

**Prometheus config flags for production (in StatefulSet or Prometheus CR):**

```yaml
# If using Prometheus Operator:
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  retention: 15d
  resources:
    requests:
      memory: 16Gi
      cpu: "4"
    limits:
      memory: 32Gi
      cpu: "8"
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: gp3-high-iops
        resources:
          requests:
            storage: 500Gi
  additionalArgs:
    - name: storage.tsdb.max-block-duration
      value: "2h"
    - name: storage.tsdb.min-block-duration
      value: "2h"
    - name: storage.tsdb.wal-compression
      value: "true"
  # Meta-monitoring: scrape yourself
  additionalScrapeConfigs:
    name: prometheus-self-scrape
    key: config.yaml
```

**Alert rules to prevent recurrence:**

```yaml
groups:
  - name: prometheus-health
    rules:
      - alert: PrometheusIngestionDropping
        expr: |
          deriv(prometheus_tsdb_head_samples_appended_total[30m]) < 0
        for: 10m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Prometheus sample ingestion rate is DECREASING — data gaps imminent"

      - alert: PrometheusCompactionTooSlow
        expr: |
          prometheus_tsdb_compaction_duration_seconds{quantile="0.99"} > 600
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "TSDB compaction taking >10 minutes — ingestion backpressure risk"

      - alert: PrometheusWALTooLarge
        expr: |
          prometheus_tsdb_wal_storage_size_bytes > 8e9
        for: 15m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "WAL is {{ $value | humanize1024 }} — replay on restart will be slow"
```

---

## Summary: Key Takeaways

```
Q1: user_id in labels = cardinality bomb = OOM crash loop
    Fix: metric_relabel_configs to drop, sample_limit as safety net
    Monitor: prometheus_tsdb_head_series

Q2: histogram_quantile ALWAYS needs rate() + sum by (le)
    Without rate() = meaningless cumulative average
    Without sum by (le) = per-pod instead of aggregate

Q3: Alertmanager = first-match, depth-first, AND within match{}
    Unmatched alerts fall to root receiver
    Watchdog → dead man's switch → must route to heartbeat service

Q4: "UP but data missing" = TSDB ingestion backpressure
    Scrape succeeds ≠ append succeeds
    Fix: better disk I/O, sharding, meta-monitoring
    NEVER trust a single Prometheus to monitor itself
```

# Phase 5, Lesson 1 — Grading

---

## Q1: Cardinality Incident — **4.8/5**

**What you nailed:**
- Root cause identification is textbook perfect. The math, the RAM explanation, the WAL replay crash loop — all correct.
- Immediate mitigation sequence is excellent: rollback first (stop the bleeding), then WAL cleanup with a debug pod, then verify. The priority ordering is exactly right.
- Permanent fix correctly uses `metricRelabelings` in a ServiceMonitor AND correctly identifies that the app code is the real fix. `user_id` belongs in traces, not metrics — that's senior-level thinking.
- Forward monitoring correctly identifies `prometheus_tsdb_head_series` and correctly calls out that meta-monitoring must be on a **separate system**.

**One issue:**

Your ServiceMonitor has a logic bug:

```yaml
metricRelabelings:
  # This KEEP action only keeps payment_transaction_duration_seconds
  # and DROPS every other metric from payment-svc
  - sourceLabels: [__name__]
    regex: 'payment_transaction_duration_seconds.*'
    action: keep
  - action: labeldrop
    regex: 'user_id'
```

That `keep` action drops **every other metric** that `payment-svc` exposes — `go_*`, `process_*`, any other business metrics. You only needed the `labeldrop`:

```yaml
metricRelabelings:
  - action: labeldrop
    regex: 'user_id'
```

Or if you wanted to target only the specific metric:

```yaml
metricRelabelings:
  - sourceLabels: [__name__, user_id]
    regex: 'payment_transaction_duration_seconds.*;.+'
    action: drop
```

Minor, but in production this silently kills monitoring for that service.

---

## Q2: PromQL Under Pressure — **5/5**

Flawless. Every query is correct. The trap question answer is comprehensive — you identified **both** bugs (no `rate()` AND no `sum by (le)`) and correctly explained why the result is deceptively stable. The bonus of showing the `+Inf` bucket alternative for error rate when only histograms are available — that's practical production knowledge.

Nothing to add.

---

## Q3: Alertmanager Routing — **5/5**

Perfect across all five parts. You correctly traced every route, understood AND logic within `match`, identified the root fallthrough, calculated the grouping behavior with correct timeline math, and — the cherry on top — correctly identified that the Watchdog route should be **first** in the routing tree and pointed to a heartbeat service.

The production hardening note on `group_wait: 0s` and `repeat_interval: 1m` for Watchdog is exactly right. Many teams put Watchdog last and wonder why their dead man's switch has a 4-hour repeat interval.

---

## Q4: Prometheus Failure Mode Diagnosis — **4.5/5**

**What you nailed:**
- The core insight is correct: **scrape succeeding ≠ append succeeding**. That's the key mental model.
- Diagnostic metrics are solid — `prometheus_tsdb_head_samples_appended_total`, out-of-order samples, compaction duration, WAL metrics.
- The layered long-term fix (storage IOPS, sharding, meta-monitoring) is production-grade.
- The alert rules for prevention are well-designed.

**What was slightly off:**

Your `--storage.tsdb.max-block-duration=2h` and `--storage.tsdb.min-block-duration=2h` suggestion effectively **disables all compaction** in Prometheus. This is the correct approach when running **Thanos** (because Thanos Compactor handles compaction externally), but for standalone Prometheus this means blocks never get merged, queries across long time ranges get progressively slower, and disk usage grows faster. You should have flagged this distinction.

Also, one additional root cause you didn't mention: **slow PromQL queries** can compete with the ingestion path for TSDB locks. If someone left a heavy dashboard open over the weekend with auto-refresh, expensive queries can starve the append pipeline. The metric to check:

```promql
prometheus_engine_query_duration_seconds{quantile="0.99"}
rate(prometheus_engine_queries_concurrent_max[5m])
```

And the flag `--query.max-concurrency` (default 20) can limit this.

Not a big miss — your answer was thorough and the reasoning chain was sound.



Proceeding directly to Lesson 2.

---

# Phase 5 — Lesson 2: Grafana + Loki + Long-Term Storage (Thanos/Mimir)

---

## Part 1: Grafana — The Visualization Layer

### What Grafana IS and ISN'T

```
Grafana IS:                           Grafana is NOT:
─────────                             ────────────────
✅ Visualization / dashboarding        ❌ A data store
✅ Query interface to data sources     ❌ A metrics collector
✅ Alerting engine (since v8+)         ❌ A replacement for Alertmanager
✅ Correlation layer (metrics↔logs)    ❌ A log aggregation system
✅ RBAC / multi-tenancy               ❌ The source of truth for anything
```

Grafana **queries data sources at render time**. Every time you load a dashboard, it sends PromQL/LogQL/SQL queries to backends. It stores **nothing** except dashboard definitions, user configs, and alert rules.

### Architecture in NovaMart

```
                    ┌─────────────────────────────┐
                    │         GRAFANA              │
                    │                              │
                    │  ┌────────┐ ┌────────────┐   │
                    │  │ Dash-  │ │ Alert      │   │
                    │  │ boards │ │ Engine     │   │
                    │  └───┬────┘ └──────┬─────┘   │
                    │      │             │         │
                    │  ┌───▼─────────────▼─────┐   │
                    │  │   Data Source Layer    │   │
                    │  └───┬──────┬──────┬─────┘   │
                    └──────┼──────┼──────┼─────────┘
                           │      │      │
              ┌────────────▼┐ ┌───▼───┐ ┌▼──────────┐
              │ Prometheus  │ │ Loki  │ │ Tempo/    │
              │ (metrics)   │ │ (logs)│ │ Jaeger    │
              │             │ │       │ │ (traces)  │
              └──────┬──────┘ └───┬───┘ └─────┬─────┘
                     │            │            │
                     └────────────┴────────────┘
                     Correlated via labels/traceID
```

### Grafana Deployment in Kubernetes

**kube-prometheus-stack already includes Grafana.** But production requires tuning:

```yaml
# values-grafana.yaml (within kube-prometheus-stack Helm values)
grafana:
  replicas: 2  # HA — requires shared database
  
  persistence:
    enabled: true
    type: pvc
    storageClassName: gp3
    size: 10Gi
  
  # In HA mode, use external database instead of SQLite
  grafana.ini:
    server:
      root_url: https://grafana.novamart.internal
    database:
      type: postgres
      host: grafana-db.novamart.internal:5432
      name: grafana
      user: grafana
      password: $__vault{grafana/db#password}
    auth:
      disable_login_form: false
    auth.generic_oauth:
      enabled: true
      name: Okta
      client_id: ${OAUTH_CLIENT_ID}
      client_secret: ${OAUTH_CLIENT_SECRET}
      scopes: openid profile email groups
      auth_url: https://novamart.okta.com/oauth2/v1/authorize
      token_url: https://novamart.okta.com/oauth2/v1/token
      api_url: https://novamart.okta.com/oauth2/v1/userinfo
      role_attribute_path: contains(groups[*], 'platform-eng') && 'Admin' || contains(groups[*], 'dev') && 'Editor' || 'Viewer'
    security:
      admin_password: $__vault{grafana/admin#password}
      cookie_secure: true
      strict_transport_security: true
    unified_alerting:
      enabled: true
    alerting:
      enabled: false  # Disable legacy alerting
  
  # Data sources provisioned as code (GitOps)
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://prometheus-operated.monitoring:9090
          access: proxy
          isDefault: true
          jsonData:
            timeInterval: 15s  # Must match scrape_interval
            exemplarTraceIdDestinations:
              - name: traceID
                datasourceUid: tempo
        - name: Loki
          type: loki
          url: http://loki-gateway.monitoring:3100
          access: proxy
          jsonData:
            derivedFields:
              - name: TraceID
                matcherRegex: '"traceID":"(\w+)"'
                url: '$${__value.raw}'
                datasourceUid: tempo
        - name: Tempo
          type: tempo
          url: http://tempo-query-frontend.monitoring:3100
          access: proxy
          uid: tempo
          jsonData:
            tracesToMetrics:
              datasourceUid: prometheus
              tags: [{key: 'service.name', value: 'service'}]
            tracesToLogs:
              datasourceUid: loki
              tags: ['service.name']
            nodeGraph:
              enabled: true
            serviceMap:
              datasourceUid: prometheus
  
  # Dashboard provisioning
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: 'default'
          orgId: 1
          folder: 'NovaMart'
          type: file
          disableDeletion: true
          editable: false  # Prevent drift from GitOps
          options:
            path: /var/lib/grafana/dashboards/default
  
  dashboardsConfigMaps:
    default: grafana-dashboards

  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: "2"
      memory: 1Gi
  
  # Sidecar for dashboard auto-discovery from ConfigMaps
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
      searchNamespace: ALL
      folderAnnotation: grafana_folder
    datasources:
      enabled: true
      label: grafana_datasource
```

### Dashboard Design Principles — The Four Golden Signals Dashboard

**Every service at NovaMart gets a standardized dashboard following this layout:**

```
┌─────────────────────────────────────────────────────────────┐
│  SERVICE: ${service}  CLUSTER: ${cluster}  NS: ${namespace} │
│  [variables dropdown row]                                    │
├──────────────────────┬──────────────────────────────────────┤
│  REQUEST RATE        │  ERROR RATE                          │
│  (Traffic)           │  (Errors)                            │
│  ┌──────────────┐    │  ┌──────────────┐                    │
│  │  ▅▆▇█▇▆▅▆▇  │    │  │  ▁▁▂▁▁▅▁▁▁  │                    │
│  └──────────────┘    │  └──────────────┘                    │
├──────────────────────┼──────────────────────────────────────┤
│  P50/P90/P99 LATENCY│  SATURATION                          │
│  (Duration)          │  (CPU/Mem/Disk)                      │
│  ┌──────────────┐    │  ┌──────────────┐                    │
│  │  ───P99      │    │  │  CPU ▇▇▆     │                    │
│  │  ───P90      │    │  │  Mem ▅▅▅     │                    │
│  │  ───P50      │    │  │  Disk▃▃▃     │                    │
│  └──────────────┘    │  └──────────────┘                    │
├─────────────────────────────────────────────────────────────┤
│  K8s STATUS: Pods Ready/Desired, Restarts, HPA %, PDB      │
├─────────────────────────────────────────────────────────────┤
│  LOGS PANEL: Loki query filtered to ${service}              │
│  (click any spike above → auto-correlates to logs here)     │
└─────────────────────────────────────────────────────────────┘
```

### Template Variables

Variables make dashboards reusable across 200+ microservices:

```json
{
  "templating": {
    "list": [
      {
        "name": "cluster",
        "type": "query",
        "query": "label_values(up, cluster)",
        "refresh": 2,
        "sort": 1
      },
      {
        "name": "namespace",
        "type": "query",
        "query": "label_values(kube_namespace_labels{cluster=\"$cluster\"}, namespace)",
        "refresh": 2
      },
      {
        "name": "service",
        "type": "query",
        "query": "label_values(http_requests_total{cluster=\"$cluster\", namespace=\"$namespace\"}, service)",
        "refresh": 2
      },
      {
        "name": "interval",
        "type": "interval",
        "query": "1m,5m,10m,30m,1h",
        "auto": true,
        "auto_min": "1m"
      }
    ]
  }
}
```

**Variable chaining:** `cluster` → filters `namespace` → filters `service`. User selects cluster first, namespace dropdown updates, then service dropdown updates. This prevents showing services from wrong clusters.

**The `$__rate_interval` magic variable:**

```promql
# DON'T hardcode rate windows in dashboard queries:
rate(http_requests_total{service="$service"}[5m])  # BAD

# DO use $__rate_interval — auto-adjusts to scrape interval + dashboard resolution:
rate(http_requests_total{service="$service"}[$__rate_interval])  # GOOD
```

`$__rate_interval` = max(4 × scrape_interval, dashboard_step). This guarantees at least 4 data points in the range window, preventing `rate()` from returning empty results when zoomed out.

### Annotations

Annotations overlay events on time-series graphs — deployments, incidents, config changes:

```yaml
# Annotation query from Prometheus (deployment events from kube-state-metrics):
{
  "datasource": "Prometheus",
  "expr": "changes(kube_deployment_status_observed_generation{namespace=\"$namespace\", deployment=\"$service\"}[2m]) > 0",
  "tagKeys": "deployment",
  "titleFormat": "Deploy: {{deployment}}",
  "textFormat": "Generation changed"
}
```

**Why annotations matter in production:** When you see a latency spike at 14:32, and there's a deployment annotation at 14:30, you've found your suspect in 2 seconds instead of 20 minutes.

### Dashboard as Code (GitOps)

**Never** create dashboards manually in production. They drift, they get deleted, they're unreproducible.

```
Git repo: novamart-observability/
├── dashboards/
│   ├── golden-signals.json      # Exported from Grafana
│   ├── k8s-cluster.json
│   ├── node-overview.json
│   └── payment-svc-detail.json
├── provisioning/
│   ├── datasources.yaml
│   └── dashboards.yaml
└── alerts/
    └── grafana-alerts.yaml
```

**Workflow:**
1. Engineer creates/edits dashboard in Grafana (dev instance)
2. Export as JSON → commit to Git → PR review
3. ArgoCD syncs ConfigMap to cluster
4. Grafana sidecar detects new ConfigMap, loads dashboard
5. Production Grafana dashboards are **read-only** (`editable: false`)

```yaml
# ConfigMap for a dashboard
apiVersion: v1
kind: ConfigMap
metadata:
  name: golden-signals-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
  annotations:
    grafana_folder: "NovaMart Services"
data:
  golden-signals.json: |
    { ... dashboard JSON ... }
```

### Grafana Alerting (Unified Alerting — v8+)

Grafana can evaluate alert rules directly. **But at NovaMart, we use Prometheus Alertmanager as the primary alert engine**, and Grafana Alerting only for:
- Loki-based log alerts (Prometheus can't query logs)
- Multi-datasource alerts (correlating metrics + logs)
- Business-metric alerts from non-Prometheus sources

```yaml
# Grafana alert rule for log-based alerting
# (e.g., alert when payment-svc logs 50+ "transaction_failed" in 5 minutes)
apiVersion: 1
groups:
  - orgId: 1
    name: log-based-alerts
    folder: NovaMart Alerts
    interval: 1m
    rules:
      - uid: payment-log-errors
        title: Payment Transaction Failures Spike
        condition: C
        data:
          - refId: A
            datasourceUid: loki
            model:
              expr: 'sum(count_over_time({app="payment-svc"} |= "transaction_failed" [5m]))'
          - refId: B
            datasourceUid: __expr__
            model:
              type: reduce
              expression: A
              reducer: last
          - refId: C
            datasourceUid: __expr__
            model:
              type: threshold
              expression: B
              conditions:
                - evaluator:
                    type: gt
                    params: [50]
        for: 2m
        labels:
          severity: warning
          team: payments
        annotations:
          summary: "payment-svc logging >50 transaction failures in 5m"
          runbook_url: "https://wiki.novamart.internal/runbooks/payment-failures"
```

### Grafana Failure Modes

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| Dashboard loads empty | Panels show "No data" | Datasource misconfigured, Prometheus down, or query time range wrong | Check datasource health (`Settings → Data Sources → Test`), verify Prometheus is reachable from Grafana pod |
| Dashboard loads slowly (>10s) | Spinner on panels | Heavy PromQL queries, too many series, time range too wide | Use recording rules, add `$__rate_interval`, reduce cardinality in queries, add variable filters |
| "Too many data points" error | Panel rendering fails | Query returns >11,000 points per series | Increase min step, use `$__rate_interval`, or set max data points on panel |
| SQLite database locked | Grafana errors on save | HA mode with SQLite (single-writer) | Migrate to PostgreSQL — SQLite doesn't support concurrent writes |
| Dashboard drift | Prod dashboards differ from Git | Someone edited directly in prod | Set `editable: false`, use provisioning, RBAC to restrict Editor role |
| LDAP/OAuth login fails | Users can't log in | Token expiry, group mapping wrong, Okta outage | Check `grafana.log`, verify OAuth callback URL, test token endpoint manually |
| Grafana OOM | Pod killed | Too many concurrent users, heavy queries, or dashboard with 50+ panels | Set `--max-concurrent-datasource-requests`, reduce panel count, use pagination |
| Alert evaluation lag | Alerts delayed or missing | Too many alert rules, datasource slow | Increase eval interval for non-critical rules, optimize underlying queries |

---

## Part 2: Loki — Log Aggregation

### Why Loki Exists

```
                    ELK Stack                          Loki
                    ─────────                          ────
Architecture:       Full-text index of ALL content     Index ONLY labels/metadata
                    (Elasticsearch inverted index)     (content stored as compressed chunks)
                    
Storage cost:       HIGH (index = same size as data)   LOW (index is tiny fraction)
                    
Query speed:        Fast for arbitrary text search     Fast for label-filtered queries
                    (pre-indexed)                      Slower for grep across all streams
                    
Operations:         HEAVY (JVM tuning, shard mgmt,     LIGHT (stateless components,
                    cluster rebalancing, split-brain)   object storage backend)
                    
Scaling:            Vertical + horizontal (complex)    Horizontal (microservice mode)

Cost at NovaMart    ~$15K/month                        ~$3K/month
scale:              (500GB/day ingestion)               (same 500GB/day)
```

**Loki's philosophy:** "Like Prometheus, but for logs." Index on metadata (labels), not content. Store logs cheaply in object storage.

### Loki Architecture

```
                           ┌─────────────────┐
                           │   Grafana        │
                           │   (LogQL)        │
                           └────────┬─────────┘
                                    │ Query
                                    ▼
┌──────────┐            ┌──────────────────────┐
│ Promtail │──Push──▶   │      LOKI            │
│ (Agent)  │            │                      │
└──────────┘            │  ┌─────────────────┐ │
                        │  │   Distributor    │◄── Receives log streams
┌──────────┐            │  │   (validation,   │    validates labels,
│ Fluent   │──Push──▶   │  │    rate limit)   │    distributes to ingesters
│ Bit      │            │  └────────┬─────────┘ │
└──────────┘            │           │            │
                        │  ┌────────▼─────────┐  │
                        │  │    Ingester       │  │ Builds chunks in memory
                        │  │   (in-memory      │  │ Flushes to storage
                        │  │    chunks, WAL)   │  │ (similar to Prometheus head block)
                        │  └────────┬─────────┘  │
                        │           │ Flush       │
                        │  ┌────────▼─────────┐  │
                        │  │  Object Storage   │  │ S3, GCS, or MinIO
                        │  │  (chunks + index) │  │
                        │  └──────────────────┘  │
                        │                        │
                        │  ┌──────────────────┐  │
                        │  │  Query Frontend   │  │ Splits/caches queries
                        │  │     + Querier     │  │ Reads from ingesters + storage
                        │  └──────────────────┘  │
                        └────────────────────────┘
```

### Deployment Modes

```
┌─────────────┬──────────────────────────────────────────────────────────┐
│ Mode        │ When to Use                                             │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Monolithic  │ Dev/test, <100GB/day. Single binary, all components.    │
│             │ helm: deploymentMode: SingleBinary                      │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Simple      │ Small-medium prod, 100-500GB/day. Read + Write + Backend│
│ Scalable    │ helm: deploymentMode: SimpleScalable (3 components)     │
│ (SSD)       │ NovaMart uses this. Best balance of simplicity/scale.   │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Microservice│ Large scale, >500GB/day. Each component separate.       │
│             │ Maximum flexibility, maximum operational complexity.     │
│             │ helm: deploymentMode: Distributed                       │
└─────────────┴──────────────────────────────────────────────────────────┘
```

### Loki Helm Deployment for NovaMart

```yaml
# values-loki.yaml
loki:
  deploymentMode: SimpleScalable
  
  auth_enabled: true  # Multi-tenancy (X-Scope-OrgID header)
  
  limits_config:
    ingestion_rate_mb: 20           # Per-tenant MB/s
    ingestion_burst_size_mb: 30     # Burst allowance
    max_entries_limit_per_query: 5000
    max_query_series: 500
    retention_period: 30d           # Auto-delete after 30 days
    max_streams_per_user: 10000     # Prevents label explosion
    max_label_name_length: 1024
    max_label_value_length: 2048
    max_label_names_per_series: 15
    per_stream_rate_limit: 3MB      # Per-stream rate limit
    per_stream_rate_limit_burst: 15MB
  
  schema_config:
    configs:
      - from: "2024-01-01"
        store: tsdb              # New TSDB index (not BoltDB)
        object_store: s3
        schema: v13              # Latest schema version
        index:
          prefix: loki_index_
          period: 24h
  
  storage:
    type: s3
    s3:
      region: us-east-1
      bucketnames: novamart-loki-chunks
      # Use IRSA — no static credentials
    bucketNames:
      chunks: novamart-loki-chunks
      ruler: novamart-loki-ruler
  
  # Structured metadata — attach traceID, spanID without creating streams
  # (Loki 3.0+)
  limits_config:
    allow_structured_metadata: true
  
  compactor:
    retention_enabled: true
    delete_request_store: s3

# Write path
write:
  replicas: 3
  resources:
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      cpu: "3"
      memory: 4Gi
  persistence:
    size: 50Gi
    storageClass: gp3

# Read path
read:
  replicas: 3
  resources:
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      cpu: "4"
      memory: 8Gi  # Reads are memory-hungry (decompressing chunks)

# Backend (compactor, ruler, index gateway)
backend:
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
  persistence:
    size: 50Gi
    storageClass: gp3

# Gateway (nginx — routing, auth, rate limiting)
gateway:
  replicas: 2
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
```

### Log Collection — Promtail vs Fluent Bit vs OTel Collector

```
┌───────────────┬─────────────────────┬──────────────────┬───────────────────┐
│               │ Promtail            │ Fluent Bit       │ OTel Collector    │
├───────────────┼─────────────────────┼──────────────────┼───────────────────┤
│ Made for      │ Loki specifically   │ General purpose  │ General purpose   │
│ Protocol      │ Loki push API       │ Many outputs     │ OTLP + many      │
│ Resource use  │ Low                 │ Very low (C)     │ Medium (Go)       │
│ K8s metadata  │ Auto (K8s SD)       │ kubernetes filter│ k8sattributes     │
│ Log parsing   │ Pipeline stages     │ Parsers/filters  │ Processors        │
│ Multi-output  │ Loki only           │ Yes              │ Yes               │
│ NovaMart pick │ ✅ Primary (Loki)   │ Legacy ELK path  │ Future direction  │
└───────────────┴─────────────────────┴──────────────────┴───────────────────┘
```

At NovaMart: **Promtail** as DaemonSet for Loki ingestion. Fluent Bit still running for legacy ELK migration path. Future: OTel Collector unifying everything.

### Promtail Configuration

```yaml
# Promtail DaemonSet config
config:
  server:
    http_listen_port: 3101
  
  clients:
    - url: http://loki-gateway.monitoring:3100/loki/api/v1/push
      tenant_id: novamart
      batchwait: 1s
      batchsize: 1048576  # 1MB
      backoff_config:
        min_period: 500ms
        max_period: 5m
        max_retries: 10
  
  positions:
    filename: /run/promtail/positions.yaml  # Track read position (like consumer offset)
  
  scrape_configs:
    # Auto-discover all pod logs
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
        - role: pod
      
      relabel_configs:
        # Use pod labels as Loki labels
        - source_labels: [__meta_kubernetes_pod_label_app]
          target_label: app
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace
        - source_labels: [__meta_kubernetes_pod_name]
          target_label: pod
        - source_labels: [__meta_kubernetes_pod_node_name]
          target_label: node
        - source_labels: [__meta_kubernetes_container_name]
          target_label: container
        # Drop pods with annotation promtail.io/ignore: "true"
        - source_labels: [__meta_kubernetes_pod_annotation_promtail_io_ignore]
          action: drop
          regex: "true"
      
      pipeline_stages:
        # Parse Docker/CRI log format
        - cri: {}
        
        # If JSON logs, extract structured fields
        - json:
            expressions:
              level: level
              msg: message
              traceID: traceID
              duration: duration
        
        # Set log level as label (BOUNDED — only debug/info/warn/error/fatal)
        - labels:
            level:
        
        # Extract traceID as structured metadata (Loki 3.0+)
        # NOT as a label (unbounded cardinality!)
        - structured_metadata:
            traceID:
        
        # Drop debug logs in production (reduce volume by ~40%)
        - match:
            selector: '{namespace!="dev"}'
            stages:
              - drop:
                  source: level
                  value: debug
        
        # Metrics from logs (counter of errors)
        - metrics:
            log_errors_total:
              type: Counter
              description: "Total error log lines"
              source: level
              config:
                value: error
                action: inc
    
    # System logs (journal)
    - job_name: journal
      journal:
        max_age: 12h
        labels:
          job: systemd-journal
      relabel_configs:
        - source_labels: [__journal__systemd_unit]
          target_label: unit
```

### Label Design — THE Critical Decision

**Labels in Loki create streams.** Every unique label combination = one stream. This is identical to Prometheus cardinality, but even more dangerous because log volume is 10-1000x metrics volume.

```
GOOD LABELS (bounded, known at deploy time):
─────────────────────────────────────────────
✅ namespace     (10-50 values)
✅ app / service (200 at NovaMart)
✅ container     (1-5 per pod)
✅ level         (5 values: debug/info/warn/error/fatal)
✅ cluster       (3 at NovaMart)
✅ node          (50-100 nodes)
✅ env           (dev/staging/prod)

BAD LABELS (unbounded, data-dependent):
─────────────────────────────────────────────
❌ user_id       (millions)
❌ request_id    (billions)
❌ trace_id      (billions)
❌ IP address    (millions)
❌ URL path      (could be infinite with path params)
❌ pod name      (changes every deploy)
❌ timestamp     (infinite)

Where do unbounded values go?
──────────────────────────────
→ Log content (grep it with |= or |~)
→ Structured metadata (Loki 3.0+ — indexed but doesn't create streams)
```

**Pod name as a label — the gotcha:** Promtail auto-discovers pod names. Every rolling deployment creates new pod names. If `pod` is a label, every deploy creates new streams, old streams go stale. This inflates the index. At NovaMart's scale (200+ services, deploys daily), this causes:
- Index bloat
- Ingester memory pressure
- Slow queries

**Mitigation:** Keep `pod` as a label for debuggability, but set aggressive retention and use `max_streams_per_user` limit. Some teams drop `pod` and use `{app="payment-svc"} |= "payment-svc-7d9f8b6c4-x2k9p"` to filter by pod name in the content instead.

### LogQL — Querying Logs

LogQL has two types of queries:

```
1. Log queries    → Return log lines     (like grep)
2. Metric queries → Return numeric values (like PromQL, derived from logs)
```

#### Log Queries (Stream Selection + Pipeline)

```
┌───────────────────┐     ┌──────────────────────────────────────┐
│ Stream Selector   │  +  │ Pipeline (filter, parse, format)     │
│ {app="order-svc"} │     │ |= "error" | json | line_format ... │
└───────────────────┘     └──────────────────────────────────────┘
```

```promql
# Basic: all logs from order-svc
{app="order-svc"}

# Filter: only error lines
{app="order-svc"} |= "error"

# Negative filter: exclude health checks
{app="order-svc"} != "/healthz"

# Regex filter: match pattern
{app="order-svc"} |~ "timeout|connection refused"

# Case-insensitive regex
{app="order-svc"} |~ "(?i)error"

# Chain multiple filters (AND logic — all must match)
{app="order-svc", namespace="production"} 
  |= "error" 
  != "health" 
  |~ "payment|checkout"

# Parse JSON logs and filter on parsed fields
{app="order-svc"} 
  | json 
  | level="error" 
  | duration > 5s

# Parse with logfmt (key=value format)
{app="order-svc"} 
  | logfmt 
  | status >= 500

# Regex parsing with named capture groups
{app="nginx"} 
  | regexp `(?P<ip>\S+) \S+ \S+ \[(?P<timestamp>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+).*" (?P<status>\d+) (?P<bytes>\d+)`
  | status >= 500

# Pattern parsing (simpler than regex)
{app="nginx"} 
  | pattern `<ip> - - [<timestamp>] "<method> <path> <_>" <status> <bytes>`
  | status >= 500

# Format output (like printf)
{app="order-svc"} 
  | json 
  | line_format "{{.level}} | {{.msg}} | duration={{.duration}}"

# Label filter after parsing (extracted fields become labels within the query)
{app="payment-svc"} 
  | json 
  | method="POST" 
  | path="/api/v1/charge" 
  | duration > 2s 
  | line_format "SLOW: {{.duration}} {{.path}} user={{.user_id}}"
```

**Pipeline stage order matters:**

```
{selector} 
  | line filter (|=, !=, |~, !~)     ← FASTEST: applied before parsing
  | parser (json, logfmt, regexp)     ← Extracts fields from content
  | label filter (field="value")      ← Filters on extracted fields
  | line_format (reformat output)     ← Reshapes the output line
  | label_format (rename labels)      ← Modifies label names
  | unwrap (for metric queries)       ← Extracts numeric value for aggregation
```

**Put the most selective filter FIRST.** `|= "error"` before `| json` avoids parsing 99% of lines that don't match.

#### Metric Queries (Aggregation Over Logs)

```promql
# Count of error logs per service in last 5 minutes
sum by (app) (
  count_over_time({namespace="production"} |= "error" [5m])
)

# Rate of errors per second
sum by (app) (
  rate({namespace="production"} |= "error" [5m])
)

# Throughput — log lines per second per service
sum by (app) (
  rate({namespace="production"} [1m])
)

# P99 request duration FROM logs (unwrap a numeric field)
quantile_over_time(0.99, 
  {app="order-svc"} 
  | json 
  | unwrap duration 
  | __error__="" [5m]
) by (app)
# __error__="" drops lines where JSON parsing failed

# Bytes rate (ingestion volume monitoring)
sum by (app) (
  bytes_rate({namespace="production"} [5m])
)

# Count unique values (approximate)
count(
  count by (status) (
    count_over_time({app="order-svc"} | json [5m])
  )
)
```

### Production LogQL Queries for NovaMart

```promql
# 1. Find all errors in payment-svc with trace context
{app="payment-svc", level="error"} 
  | json 
  | line_format "traceID={{.traceID}} {{.msg}}"

# 2. Slow requests (>2s) across ALL services
{namespace="production"} 
  | json 
  | duration > 2s 
  | line_format "{{.app}} | {{.duration}} | {{.path}} | traceID={{.traceID}}"

# 3. OOM events in pod logs
{namespace="production"} |~ "OOMKilled|Out of memory|oom-kill"

# 4. Connection refused (downstream dependency failures)
{namespace="production"} |~ "connection refused|ECONNREFUSED|dial tcp.*refused"

# 5. Count 5xx errors per service per minute (for alerting)
sum by (app) (
  rate({namespace="production"} | json | status >= 500 [1m])
)

# 6. Log volume by service (who's logging too much?)
sum by (app) (
  bytes_rate({namespace="production"} [5m])
) / 1024 / 1024  # MB/s

# 7. Detect log storms (sudden spike in volume)
sum(rate({namespace="production"} [1m])) by (app) 
  > 1000  # More than 1000 lines/sec = investigate

# 8. Find specific transaction
{app="payment-svc"} |= "txn-ABC123XYZ"

# 9. Compare error rates before/after deployment
# (Use Grafana's "Compare" feature or offset)
sum(count_over_time({app="order-svc", level="error"} [1h]))
# vs
sum(count_over_time({app="order-svc", level="error"} [1h] offset 1d))
```

### Loki Failure Modes

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| `429 Too Many Requests` | Logs being dropped, Promtail backlog | Ingestion rate limit hit (`ingestion_rate_mb`) | Increase limit OR reduce log volume (drop debug) |
| `stream limit exceeded` | New streams rejected | `max_streams_per_user` hit — label cardinality explosion | Audit labels, remove unbounded labels (pod name with high churn) |
| Queries timeout | "context deadline exceeded" | Querying too many streams or too wide time range | Add label filters, narrow time range, use `max_entries_limit_per_query` |
| Missing logs | Logs not appearing in Grafana | Promtail position file lost (pod rescheduled), log file rotated before read | Use PVC for positions file, increase `sync_period` |
| Duplicate logs | Same line appears multiple times | Promtail restarted and replayed from last position | Idempotent processing; Loki deduplicates by timestamp+labels+content |
| Ingester OOM | Ingester pods crash-looping | Too many active streams in memory, flush config wrong | Increase memory, reduce `chunk_idle_period`, tune `max_chunk_age` |
| Chunk flush failures | Data loss risk, WAL growing | S3 permissions wrong, bucket doesn't exist, network issue | Check IRSA, verify bucket, check S3 endpoints in VPC |
| Query reads all chunks | Queries are slow and expensive | Missing or wrong label filters — Loki must scan everything | Educate users: ALWAYS start with `{app="..."}`, never `{} |= "error"` |
| Index corruption | Queries return incomplete results | Compactor crashed mid-compaction | Rebuild index from chunks (loki-canary helps detect) |
| Clock skew | "entry out of order" errors | Nodes have different times, logs arrive out of timestamp order | NTP sync across all nodes, `unordered_writes: true` in Loki config |

**The cardinal Loki performance rule:**

```
{} |= "error"                    ← NEVER DO THIS. Scans ALL streams.
{app="payment-svc"} |= "error"  ← ALWAYS DO THIS. Scans one stream set.
```

The stream selector `{}` with no labels is the Loki equivalent of `SELECT * FROM logs`. It WILL time out at scale.

---

## Part 3: Correlation — Connecting Metrics ↔ Logs ↔ Traces

This is what separates observability from "three separate tools":

```
Incident: order-svc p99 latency spike
───────────────────────────────────────

Step 1: METRICS (Prometheus/Grafana)
  → See p99 spike on Golden Signals dashboard
  → histogram_quantile(0.99, ...) jumped from 200ms to 5s at 14:32
  → Annotation shows deployment at 14:30 — suspect!
  
Step 2: LOGS (Loki via Grafana)
  → Click "Split" view, switch to Loki
  → {app="order-svc", level="error"} | json | duration > 2s
  → See: "database connection timeout" with traceID=abc123
  
Step 3: TRACES (Tempo/Jaeger via Grafana)
  → Click traceID link → opens trace view
  → Trace shows: order-svc → db-proxy → RDS
  → db-proxy span: 4.8s (normally 50ms)
  → Root cause: RDS failover happened at 14:31, connection pool stale
  
Step 4: FIX
  → Restart connection pool / configure connection validation
  → Latency recovers at 14:45
```

**This correlation flow is configured in the datasource config** (shown above in the Grafana section):
- Prometheus → Loki: via shared labels (`app`, `namespace`)
- Loki → Tempo: via `derivedFields` (regex extracting traceID from log lines)
- Tempo → Prometheus: via `tracesToMetrics` (service name mapping)

---

## Part 4: Long-Term Storage — Thanos vs Mimir vs Cortex

### The Problem

Prometheus is **not designed for long-term storage:**

```
┌─────────────────────────────────────────────────────────────┐
│ Prometheus Limitations for Large-Scale / Long-Term          │
├─────────────────────────────────────────────────────────────┤
│ 1. Single-node TSDB — no horizontal scaling                 │
│ 2. Retention limited by local disk (15d-30d typical)        │
│ 3. No global query view across multiple Prometheus          │
│ 4. HA = two independent instances (no dedup at query time)  │
│ 5. Vertical scaling hits ceiling (~20M active series)       │
│ 6. Disk fills up, compaction slows, data gaps (Lesson 1 Q4)│
└─────────────────────────────────────────────────────────────┘
```

**NovaMart's needs:**
- 3 clusters × 2 Prometheus instances (HA) = 6 Prometheus servers
- 90-day retention for capacity planning, SLO reporting
- Global query: "show me p99 across ALL clusters"
- Cost-effective: can't afford 90 days on SSD/EBS

### The Three Solutions

```
┌──────────────────┬─────────────────────┬─────────────────────┬──────────────────┐
│                  │ Thanos              │ Grafana Mimir       │ Cortex           │
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ Architecture     │ Sidecar model       │ Fully distributed   │ Fully distributed│
│                  │ (extends Prometheus)│ microservices       │ microservices    │
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ How data enters  │ Sidecar uploads     │ Remote write from   │ Remote write from│
│                  │ blocks from local   │ Prometheus           │ Prometheus       │
│                  │ Prometheus TSDB     │                     │                  │
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ Query            │ Thanos Query        │ Query frontend +    │ Query frontend + │
│                  │ (fan-out to sidecars│ Querier (reads from │ Querier          │
│                  │ + object storage)   │ ingesters + storage)│                  │
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ Storage          │ Object storage (S3) │ Object storage (S3) │ Object storage   │
│                  │ Prometheus format   │ Own block format    │ Same as Thanos   │
│                  │ blocks              │ (optimized)         │ (forked origin)  │
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ Multi-tenancy    │ Basic (external     │ Native (built-in,   │ Native           │
│                  │ labels)             │ per-tenant limits)  │                  │
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ Deduplication    │ Query-time dedup    │ Write-time dedup    │ Write-time dedup │
│                  │ (from HA pairs)     │ (replication factor)│                  │
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ Downsampling     │ Yes (5m, 1h auto)   │ No (not needed —   │ No               │
│                  │                     │ query optimization)│                  │
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ Operational      │ Medium (sidecar +   │ Higher (more        │ Highest          │
│ complexity       │ compactor + store)  │ components)         │ (Mimir supercedes│
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ Maturity         │ CNCF Incubating     │ Grafana Labs (OSS)  │ Deprecated in    │
│                  │ Very mature         │ Rapidly maturing    │ favor of Mimir   │
├──────────────────┼─────────────────────┼─────────────────────┼──────────────────┤
│ Best for         │ Existing Prometheus │ Greenfield, high    │ Legacy (migrate  │
│                  │ extending with      │ scale, multi-tenant │ to Mimir)        │
│                  │ minimal changes     │ platforms           │                  │
└──────────────────┴─────────────────────┴─────────────────────┴──────────────────┘
```

**NovaMart choice: Thanos** — because we already have Prometheus deployed, and Thanos extends it with minimal disruption.

### Thanos Architecture (Deep Dive)

```
Cluster: us-east-1                     Cluster: us-west-2
┌────────────────────────┐             ┌────────────────────────┐
│ Prometheus             │             │ Prometheus             │
│ ┌────────┐ ┌────────┐ │             │ ┌────────┐ ┌────────┐ │
│ │  TSDB  │ │ Thanos │ │             │ │  TSDB  │ │ Thanos │ │
│ │  Head  │ │Sidecar │─┼─upload──┐   │ │  Head  │ │Sidecar │─┼─upload──┐
│ │  Block │ │        │ │   │     │   │ │  Block │ │        │ │   │     │
│ └────────┘ └───┬────┘ │   │     │   │ └────────┘ └───┬────┘ │   │     │
│                │       │   │     │   │                │       │   │     │
└────────────────┼───────┘   │     │   └────────────────┼───────┘   │     │
                 │StoreAPI   │     │                    │StoreAPI   │     │
                 │           │     │                    │           │     │
                 ▼           ▼     │                    ▼           ▼     │
           ┌──────────────────┐    │              ┌──────────────────┐   │
           │   S3 Bucket      │◄───┘              │   S3 Bucket      │◄──┘
           │ (long-term       │                   │ (long-term       │
           │  metrics)        │                   │  metrics)        │
           └────────┬─────────┘                   └────────┬─────────┘
                    │                                      │
                    └──────────────┬────────────────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │       Thanos Query           │ (Global query endpoint)
                    │  (fan-out to all StoreAPIs)  │
                    │  ┌─────────────────────────┐ │
                    │  │ Deduplication            │ │ (removes HA duplicates)
                    │  │ Partial response         │ │ (query succeeds even if
                    │  │                          │ │  one sidecar is down)
                    │  └─────────────────────────┘ │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │    Thanos Query Frontend      │
                    │  - Query splitting            │
                    │  - Results caching (memcached) │
                    │  - Retry / limits             │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │       Grafana                 │
                    │  (datasource: Thanos Query)   │
                    └──────────────────────────────┘

BACKGROUND COMPONENTS:
┌──────────────────────┐  ┌──────────────────────┐
│   Thanos Compactor   │  │   Thanos Store       │
│ - Merges blocks in S3│  │   Gateway            │
│ - Downsamples (5m,1h)│  │ - Serves queries     │
│ - Retention policy   │  │   from object storage│
│ - SINGLETON (no HA!) │  │ - Caches index/chunks│
└──────────────────────┘  └──────────────────────┘
```

### Thanos Components Explained

**1. Sidecar** — runs alongside each Prometheus

```yaml
# In Prometheus Operator (kube-prometheus-stack values):
prometheus:
  prometheusSpec:
    thanos:
      image: quay.io/thanos/thanos:v0.35.0
      version: v0.35.0
      objectStorageConfig:
        existingSecret:
          name: thanos-objstore-config
          key: objstore.yml
      # Sidecar resources
      resources:
        requests:
          cpu: 200m
          memory: 256Mi
        limits:
          cpu: "1"
          memory: 1Gi
    # CRITICAL: disable compaction in Prometheus (Thanos Compactor handles it)
    disableCompaction: true
    # External labels for global deduplication
    externalLabels:
      cluster: us-east-1
      replica: $(POD_NAME)  # For HA dedup
```

The sidecar does two things:
- **Uploads** completed TSDB blocks (every 2h) to object storage
- **Serves StoreAPI** for recent data (still in Prometheus head block)

**2. Store Gateway** — serves queries from object storage

```yaml
# thanos-store.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: thanos-store
spec:
  replicas: 2  # HA
  template:
    spec:
      containers:
        - name: thanos-store
          image: quay.io/thanos/thanos:v0.35.0
          args:
            - store
            - --data-dir=/var/thanos/store
            - --objstore.config-file=/etc/thanos/objstore.yml
            - --index-cache-size=1GB
            - --chunk-pool-size=4GB
            # Time-based partitioning for scale
            - --min-time=-90d
            - --max-time=-2h  # Recent data from sidecar, not store
          resources:
            requests:
              memory: 4Gi
              cpu: "1"
            limits:
              memory: 8Gi
          volumeMounts:
            - name: data
              mountPath: /var/thanos/store
            - name: objstore-config
              mountPath: /etc/thanos
```

**3. Query** — the global query endpoint

```yaml
# thanos-query deployment
args:
  - query
  - --http-address=0.0.0.0:9090
  - --grpc-address=0.0.0.0:10901
  # Connect to all StoreAPI endpoints
  - --endpoint=dnssrv+_grpc._tcp.thanos-sidecar-us-east-1.monitoring.svc
  - --endpoint=dnssrv+_grpc._tcp.thanos-sidecar-us-west-2.monitoring.svc
  - --endpoint=dnssrv+_grpc._tcp.thanos-sidecar-eu-west-1.monitoring.svc
  - --endpoint=dnssrv+_grpc._tcp.thanos-store.monitoring.svc
  # HA deduplication
  - --query.replica-label=replica
  - --query.replica-label=prometheus_replica
  # Allow partial responses (don't fail if one cluster is unreachable)
  - --query.partial-response
  # Auto-downsampling for long ranges
  - --query.auto-downsampling
```

**4. Compactor** — background maintenance on object storage

```yaml
# thanos-compactor deployment — MUST be a singleton
args:
  - compact
  - --data-dir=/var/thanos/compact
  - --objstore.config-file=/etc/thanos/objstore.yml
  # Downsampling (automatic)
  - --downsampling.disable=false
  # Retention policies
  - --retention.resolution-raw=30d      # Full resolution: 30 days
  - --retention.resolution-5m=90d       # 5-minute downsampled: 90 days
  - --retention.resolution-1h=365d      # 1-hour downsampled: 1 year
  # Compaction
  - --compact.concurrency=4
  - --compact.cleanup-interval=5m
  # Wait instead of exit (run as long-lived process)
  - --wait
  - --wait-interval=5m
resources:
  requests:
    memory: 2Gi
    cpu: "1"
  limits:
    memory: 4Gi
```

**⚠️ CRITICAL: Compactor is a SINGLETON.** Never run more than one compactor against the same bucket. Two compactors will corrupt blocks by competing on compaction/deletion operations. This is the #1 Thanos operational mistake.

```
If compactor dies → blocks accumulate in S3 → queries get slower
                  → but NO data loss. It's safe to be down temporarily.
                  → Set an alert for compactor being down > 1 hour.
```

**5. Query Frontend** — query optimization layer

```yaml
args:
  - query-frontend
  - --http-address=0.0.0.0:9090
  - --query-frontend.downstream-url=http://thanos-query:9090
  # Split long-range queries into day-sized chunks
  - --query-range.split-interval=24h
  # Cache results in memcached
  - --query-range.response-cache-config-file=/etc/thanos/cache.yml
  # Retry failed queries
  - --query-range.max-retries-per-request=3
  # Limit query concurrency
  - --query-frontend.max-body-size=10MB
```

Cache config:

```yaml
# cache.yml
type: MEMCACHED
config:
  addresses:
    - memcached.monitoring.svc:11211
  timeout: 500ms
  max_idle_connections: 100
  max_async_concurrency: 50
  max_async_buffer_size: 25000
  max_get_multi_concurrency: 100
  max_item_size: 1MiB
  dns_provider_update_interval: 10s
```

### Object Storage Configuration

```yaml
# objstore.yml (referenced by sidecar, store, compactor)
type: S3
config:
  bucket: novamart-thanos-metrics
  endpoint: s3.us-east-1.amazonaws.com
  region: us-east-1
  # Use IRSA — no static credentials
  # AWS SDK auto-discovers from projected service account token
  sse_config:
    type: SSE-S3       # Server-side encryption
  # Cost optimization: set lifecycle rules on the bucket
  # (Thanos compactor handles retention, but bucket lifecycle
  #  is a safety net for orphaned blocks)
```

**S3 bucket policy for cost optimization:**

```json
{
  "Rules": [
    {
      "ID": "Transition to IA after 30 days",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        }
      ]
    },
    {
      "ID": "Transition to Glacier after 180 days",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 180,
          "StorageClass": "GLACIER_IR"
        }
      ],
      "Filter": {
        "Prefix": "debug/"
      }
    },
    {
      "ID": "Abort incomplete multipart uploads",
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

**⚠️ DO NOT use Glacier for Thanos blocks.** Thanos Store Gateway needs to read blocks for queries. Glacier retrieval times (minutes to hours) will cause query timeouts. Use `STANDARD_IA` at most. The Glacier rule above is only for a hypothetical debug prefix you'd never query.

### Thanos vs Mimir — When to Choose What

```
Choose THANOS when:
────────────────────
✅ You already have Prometheus deployed
✅ You want minimal changes to existing setup (just add sidecar)
✅ You have < 50M active series across all clusters
✅ Your team is small and wants simpler operations
✅ You need downsampling for very long retention (>6 months)
✅ NovaMart's current situation

Choose MIMIR when:
────────────────────
✅ Greenfield deployment (no existing Prometheus to preserve)
✅ Multi-tenant platform (many teams sharing one metrics system)
✅ > 50M active series (Mimir scales better horizontally)
✅ You need ingest-time dedup (not query-time like Thanos)
✅ You want a single system instead of Prometheus + addon
✅ NovaMart in 2 years when scale grows
```

### Thanos Failure Modes

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| Sidecar upload failing | Gaps in long-term data, Store Gateway returns incomplete results | S3 permissions (IRSA misconfigured), bucket doesn't exist, network to S3 blocked | Check sidecar logs, verify IRSA trust policy, check VPC endpoint for S3 |
| Compactor halted | `thanos_compact_halted == 1`, block count growing in S3 | Overlapping blocks from manual operations, corrupted block metadata | Check compactor logs, use `thanos tools bucket verify`, delete corrupted block |
| Query returns partial data | Missing metrics from one cluster, Thanos Query logs "partial response" | Sidecar unreachable (network, pod down), StoreAPI timeout | Check all `--endpoint` targets, verify gRPC connectivity, increase timeout |
| Slow long-range queries | Queries over 7+ days take >30s | No query frontend, no caching, downsampled data not being used | Deploy Query Frontend with memcached, enable `--query.auto-downsampling` |
| Two compactors running | Block corruption, duplicate blocks, retention not applied | Deployment misconfiguration (replicas > 1) | **Immediately** scale to 1 replica. Run `thanos tools bucket verify --repair` |
| Store Gateway OOM | Crashes when loading block index | Too many blocks, index cache too small | Increase memory, increase `--index-cache-size`, time-partition store gateways |
| HA dedup not working | Duplicate series in query results | `external_labels` missing `replica` label, or `--query.replica-label` not set | Add `replica` external label to each Prometheus, configure Query dedup |
| Sidecar blocks Prometheus | Prometheus can't compact, disk fills | `disableCompaction: true` not set, Prometheus and sidecar both compacting | Set `disableCompaction: true` in Prometheus spec |
| Stale data in query | Old data appears, recent data missing | Store Gateway cache stale, sidecar hasn't uploaded yet (2h delay) | Sidecar serves recent data via StoreAPI; verify sidecar is in Query endpoints |
| S3 costs exploding | Unexpected AWS bill spike | No retention policy in compactor, no S3 lifecycle, too many replicas uploading uncompacted blocks | Set retention flags on compactor, add S3 lifecycle rules, verify compactor is running |

### The Full NovaMart Observability Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA COLLECTION                              │
│                                                                     │
│  Microservices ──metrics──▶ Prometheus (scraped every 15s)         │
│  Microservices ──logs────▶ Promtail ──push──▶ Loki                 │
│  Microservices ──traces──▶ OTel Collector ──push──▶ Tempo          │
│                                                                     │
│  node-exporter ──metrics──▶ Prometheus (node hardware)             │
│  kube-state-metrics ─────▶ Prometheus (K8s object state)           │
│  kubelet/cAdvisor ───────▶ Prometheus (container resources)        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                        SHORT-TERM STORAGE                           │
│                                                                     │
│  Prometheus TSDB ──── 15 days local (EBS gp3)                      │
│  Loki Ingesters ───── few hours in-memory + WAL                    │
│  Tempo Ingesters ──── few hours in-memory + WAL                    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                        LONG-TERM STORAGE                            │
│                                                                     │
│  Thanos Sidecar ──upload──▶ S3 (novamart-thanos-metrics)           │
│  │  Compactor ────── compaction + downsampling + retention          │
│  │  Store GW ─────── serves old data for queries                   │
│  │                                                                  │
│  Loki ────flush───▶ S3 (novamart-loki-chunks)                     │
│  │  Compactor ────── index compaction + retention                  │
│  │                                                                  │
│  Tempo ───flush───▶ S3 (novamart-tempo-traces)                    │
│  │  Compactor ────── block compaction + retention                  │
│  │                                                                  │
│  Retention: Metrics=90d, Logs=30d, Traces=15d                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                        QUERY / VISUALIZATION                        │
│                                                                     │
│  Thanos Query ───▶ Grafana (metrics)                               │
│  Loki Query ─────▶ Grafana (logs)                                  │
│  Tempo Query ────▶ Grafana (traces)                                │
│                                                                     │
│  Correlation: metrics ←→ logs ←→ traces (via labels + traceID)    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                        ALERTING                                     │
│                                                                     │
│  Prometheus ──rules──▶ Alertmanager ──route──▶ PagerDuty (SEV1)   │
│                                       ──route──▶ Slack (SEV2-4)    │
│  Grafana ──log alerts──▶ Alertmanager (contact point)              │
│  Watchdog ──heartbeat──▶ Dead Man's Switch ──▶ PagerDuty           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Card

```
GRAFANA
───────
Deploy:          kube-prometheus-stack includes it
HA:              replicas: 2 + PostgreSQL (NOT SQLite)
Datasources:     Provision as code (YAML), never manual in prod
Dashboards:      GitOps via ConfigMap + sidecar, editable: false in prod
Variables:        Chain: cluster → namespace → service
                 Use $__rate_interval (NOT hardcoded [5m])
Annotations:     Overlay deploys/incidents on graphs
Alerting:        Use for Loki-based alerts only; Prometheus rules for metrics

LOKI
────
Architecture:    Index labels only, store content in object storage
Deploy modes:    Monolithic → SimpleScalable → Distributed
Label rules:     BOUNDED only (app, namespace, level, cluster)
                 NEVER: user_id, request_id, trace_id, IP
                 Use structured_metadata for unbounded (Loki 3.0+)
LogQL order:     {selector} | line_filter | parser | label_filter | format
Performance:     ALWAYS start with label selector, NEVER bare {}
Promtail:        DaemonSet, auto-discovers pods, pipeline_stages for parsing
Metric queries:  count_over_time(), rate(), quantile_over_time(unwrap)

THANOS
──────
Components:      Sidecar, Store GW, Query, Query Frontend, Compactor
Sidecar:         Uploads blocks to S3, serves StoreAPI for recent data
Compactor:       SINGLETON — never >1 replica. Handles compaction + downsample + retention
Store GW:        Serves queries from S3, needs memory for index cache
Query:           Fan-out to all StoreAPIs, dedup via --query.replica-label
Query Frontend:  Split queries, cache results (memcached)
Key config:      disableCompaction: true on Prometheus
                 external_labels: cluster + replica
Retention:       Set on Compactor (raw=30d, 5m=90d, 1h=365d)
Cost:            S3 Standard-IA after 30d, NEVER Glacier for queried data

CORRELATION
───────────
Metrics → Logs:   Shared labels (app, namespace)
Logs → Traces:    derivedFields regex on traceID in Loki datasource config
Traces → Metrics: tracesToMetrics mapping in Tempo datasource config
```

---

## Retention Questions — Phase 5 Lesson 2

### Q1: Loki Label Cardinality Incident 🔥

**Scenario:** NovaMart's Loki cluster starts returning `429 Too Many Requests` errors. Promtail agents across all nodes are reporting `server returned HTTP status 429 Too Many Requests`. Log ingestion has stopped. You check Loki's distributor metrics and see `loki_distributor_lines_received_total` is flat.

Investigation reveals that the `cart-svc` team deployed a new version 2 hours ago that switched to structured JSON logging. Their new log format includes this in every line:

```json
{"level":"info","msg":"item added","userID":"u-948271","sessionID":"sess-abc123","cartID":"cart-xyz789","timestamp":"2024-01-15T10:32:01Z"}
```

Their Promtail pipeline was updated to extract labels:

```yaml
pipeline_stages:
  - json:
      expressions:
        level: level
        userID: userID
        sessionID: sessionID
        cartID: cartID
  - labels:
      level:
      userID:
      sessionID:
      cartID:
```

**Questions:**
1. Explain exactly why this causes `429` errors and why it affects **all** services, not just `cart-svc`.
2. What is your immediate fix (logs are being dropped RIGHT NOW for the entire cluster)?
3. Write the corrected Promtail pipeline that preserves debuggability without killing Loki.
4. What Loki metric and alert would you set up to catch this before it becomes an outage next time?

---

### Q2: Thanos Query Returns Incomplete Data 🔥

**Scenario:** It's Wednesday morning. The VP of Engineering asks for a report: "Show me the p99 latency trend for `payment-svc` across all three regions for the past 60 days."

You query Thanos:

```promql
histogram_quantile(0.99, 
  sum by (le, cluster) (
    rate(http_request_duration_seconds_bucket{service="payment-svc"}[1h])
  )
)
```

**Results show:**
- `us-east-1`: Full 60-day data ✅
- `us-west-2`: Data only for the last 14 days ❌
- `eu-west-1`: No data at all ❌

All three clusters have Prometheus running and currently healthy.

**Questions:**
1. List the **three most likely causes** in order of probability, and for each, give the exact command or metric you'd check to confirm.
2. For the `eu-west-1` case (no data at all), walk through the full Thanos data path and identify every point where the chain could break.
3. The VP needs the report TODAY. What is your short-term workaround to get the data while you fix the underlying issue?

---

### Q3: LogQL Under Pressure 🔥

**Scenario:** 3 AM. PagerDuty fires: `PaymentSvcErrorRateHigh`. You open Grafana.

**Write the exact LogQL queries for each investigation step:**

1. Find all error logs from `payment-svc` in the last 15 minutes
2. Parse the JSON logs and find only errors related to the `/charge` endpoint with duration > 2 seconds
3. Extract the `traceID` from those slow errors so you can pivot to distributed tracing
4. Count the error rate per minute for `payment-svc` over the last hour (for a Grafana graph)
5. Compare the current error count (last 1 hour) against yesterday's error count (same hour) to confirm this is abnormal
6. **Trap question:** A junior team member runs `{} |= "payment" |= "error"` and says "I'm checking payment errors." Explain why this is dangerous and what will actually happen at NovaMart's scale.

---

### Q4: Grafana Dashboard Debugging 🔥

**Scenario:** The `order-svc` team reports their Grafana dashboard is broken:
- Two panels show "No data"
- One panel shows data but the values seem "impossibly low" (order rate showing 0.002 req/s when they know they're handling thousands of requests)
- The dashboard was working fine yesterday
- No deployments happened to Grafana itself

**Questions:**
1. For the "No data" panels — give a systematic debugging checklist (at least 5 steps) with the exact actions you'd take in Grafana.
2. For the "impossibly low values" panel — what are the three most likely causes? For one of them, explain the exact mechanism of how it produces misleadingly low numbers.
3. You discover that the issue started when someone changed a template variable default. The dashboard uses chained variables (`cluster` → `namespace` → `service`). How can variable misconfiguration cause "No data" and how do you prevent this from happening again?

---

# Phase 5, Lesson 2 — Retention Answers

---

## Q1: Loki Label Cardinality Incident

### 1. Why 429s, and Why ALL Services Are Affected

**The root cause is identical to the Prometheus cardinality bomb from Lesson 1, but the mechanism is different.**

Loki's fundamental design principle: **index labels, not log content.** Loki stores logs in **streams**, where each unique label combination = one stream. The distributor and ingester enforce a **per-tenant stream limit** to protect the cluster.

The Promtail pipeline is promoting `userID`, `sessionID`, and `cartID` into **labels**. The cardinality math:

```
500K users × N sessions/user × M carts/session = millions of unique streams
```

Each stream in Loki requires:
- An entry in the **inverted index** (stored in the index store — DynamoDB/BoltDB/TSDB)
- A **chunk** in the ingester's memory (each stream gets its own chunk buffer)
- An entry in the **distributor's ring hash** for routing

Loki has hard limits (configurable but finite):

```yaml
limits_config:
  max_streams_per_user: 10000          # default
  max_global_streams_per_user: 50000   # default
  per_stream_rate_limit: 3MB           # default
  ingestion_rate_mb: 4                 # default per-tenant
  ingestion_burst_size_mb: 6           # default per-tenant
```

When `cart-svc` blows past `max_streams_per_user`, the **ingester returns 429 to the distributor, which returns 429 to ALL Promtail agents** — not just `cart-svc`'s. Here's why:

```
                    Distributor (shared, per-tenant rate limiting)
                   ╱            |              ╲
            Ingester-1    Ingester-2    Ingester-3
               ↑               ↑              ↑
          Stream limit is GLOBAL per tenant, not per service

Promtail (cart-svc)  ──→ ┐
Promtail (order-svc) ──→ ├──→ Distributor ──→ 429 (tenant limit hit)
Promtail (payment-svc)──→ ┘
```

**The tenant-level rate limit doesn't distinguish between services.** Once `cart-svc` exhausts the stream budget, the distributor rejects ALL pushes for that tenant, including `order-svc` and `payment-svc` logs. This is the **noisy neighbor problem** — one service's cardinality explosion causes a cluster-wide log blackout.

**Additional damage:** Even after the stream creation stops, the ingester is now holding millions of tiny chunks in memory. This causes:
- Ingester OOM risk
- Flush storms to object storage (S3) when chunks get flushed
- Index write amplification (thousands of index entries per second)

### 2. Immediate Fix — Logs Are Being Dropped RIGHT NOW

**Priority order: restore log ingestion for all services, then fix cart-svc.**

**Step 1 — Temporarily raise the stream limit to unblock all services (run against Loki's runtime config or API):**

```bash
# If Loki uses runtime_config file (most production setups):
# Edit the runtime overrides config
cat <<'EOF' > /tmp/runtime-overrides.yaml
overrides:
  "novamart":  # tenant ID
    max_streams_per_user: 100000
    max_global_streams_per_user: 500000
    ingestion_rate_mb: 20
    ingestion_burst_size_mb: 40
EOF

# Apply — if runtime_config is a ConfigMap:
kubectl -n logging create configmap loki-runtime-config \
  --from-file=overrides.yaml=/tmp/runtime-overrides.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# Loki picks up runtime config changes WITHOUT restart (polls periodically)
# Default poll interval: 10s
```

**⚠️ This is a bandaid.** It unblocks ingestion but doesn't fix the cardinality bomb.

**Step 2 — Stop the bleeding: scale cart-svc Promtail's label extraction immediately:**

```bash
# Option A: If Promtail runs as DaemonSet with per-service pipeline configs in a ConfigMap
# Patch the cart-svc pipeline to stop extracting high-cardinality labels
kubectl -n logging edit configmap promtail-config
# Remove userID, sessionID, cartID from the labels stage (fix shown in Q1.3)

# Then restart Promtail pods to pick up the config:
kubectl -n logging rollout restart daemonset/promtail
kubectl -n logging rollout status daemonset/promtail --timeout=120s
```

```bash
# Option B: If you can't change Promtail config fast enough,
# use Loki's per-stream rate limiting to drop cart-svc's toxic streams:
# Add to runtime overrides:
cat <<'EOF' >> /tmp/runtime-overrides.yaml
overrides:
  "novamart":
    max_streams_per_user: 100000
    per_stream_rate_limit: 3MB
    per_stream_rate_limit_burst: 10MB
    # This won't fix the stream count but prevents individual stream flooding
EOF
```

**Step 3 — Verify ingestion is flowing again:**

```bash
# Check Loki distributor is accepting logs
kubectl -n logging port-forward svc/loki-distributor 3100:3100 &

# Check the distributor is receiving lines
curl -s http://localhost:3100/metrics | grep loki_distributor_lines_received_total
# Should be increasing

# Check for 429s — should drop to 0
curl -s http://localhost:3100/metrics | grep loki_request_duration_seconds_count | grep 429
# Or from Promtail side:
kubectl -n logging logs daemonset/promtail --tail=20 | grep "429"
# Should stop appearing

# Verify you can query recent logs
curl -s -G http://localhost:3100/loki/api/v1/query \
  --data-urlencode 'query={namespace="payments"} |= "payment"' \
  --data-urlencode 'limit=5' | jq '.data.result | length'
# Should return > 0
```

**Step 4 — Revert the temporary limit increase after cart-svc is fixed:**

```bash
# Reset to safe defaults
cat <<'EOF' > /tmp/runtime-overrides.yaml
overrides:
  "novamart":
    max_streams_per_user: 10000
    max_global_streams_per_user: 50000
    ingestion_rate_mb: 10
    ingestion_burst_size_mb: 20
EOF

kubectl -n logging create configmap loki-runtime-config \
  --from-file=overrides.yaml=/tmp/runtime-overrides.yaml \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 3. Corrected Promtail Pipeline

**Principle: labels are for FILTERING streams, not for storing data fields.** High-cardinality values go into the **log line itself** (they're already there) and get extracted at **query time** using LogQL parsers.

```yaml
scrape_configs:
  - job_name: cart-svc
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
    pipeline_stages:
      # Parse JSON to validate structure and extract ONLY low-cardinality fields as labels
      - json:
          expressions:
            level: level
            # userID, sessionID, cartID are NOT extracted as labels
            # They remain in the log line for query-time extraction

      # Only promote bounded-cardinality fields to labels
      - labels:
          level:
          # That's it. level has ~5 values: debug, info, warn, error, fatal

      # Optional: drop debug logs in production to save storage
      - match:
          selector: '{app="cart-svc"} |= "level" | json | level="debug"'
          action: drop
          drop_counter_reason: debug_logs_dropped

      # Optional: add a structured_metadata field for traceID (Loki 3.0+)
      # This is indexed but doesn't create new streams
      - structured_metadata:
          userID:
          sessionID:
```

**At query time, debuggability is preserved:**

```logql
# Find all logs for a specific user — no label needed, parse at query time
{app="cart-svc"} | json | userID="u-948271"

# Find slow cart operations for a specific session
{app="cart-svc"} | json | sessionID="sess-abc123" | duration > 2s
```

**Cardinality comparison:**

```
BEFORE (broken):
  Streams = users × sessions × carts × levels = millions

AFTER (fixed):
  Streams = namespaces × apps × pods × levels
         = 1 × 1 × 10 pods × 5 levels = 50 streams
```

### 4. Monitoring and Alerting to Catch This Early

**Key Loki metrics to monitor (scraped from Loki's `/metrics` endpoint by Prometheus):**

```yaml
groups:
  - name: loki-cardinality
    rules:
      # Stream creation rate — the earliest warning signal
      - alert: LokiHighStreamCreationRate
        expr: |
          sum(rate(loki_ingester_streams_created_total[5m])) > 500
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Loki stream creation rate is {{ $value }}/s — cardinality explosion risk"
          runbook: "https://wiki.novamart.internal/runbooks/loki-cardinality"

      # Active streams approaching limit
      - alert: LokiStreamLimitApproaching
        expr: |
          sum(loki_ingester_memory_streams) 
            / 
          loki_limits_overrides{limit_name="max_global_streams_per_user"}
          > 0.80
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Loki at {{ $value | humanizePercentage }} of stream limit"

      # 429s being returned — the actual outage signal
      - alert: LokiIngestionRejections
        expr: |
          sum(rate(loki_request_duration_seconds_count{status_code="429"}[5m])) > 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Loki returning 429s — log ingestion is being rejected"

      # Ingestion rate drop — correlated signal
      - alert: LokiIngestionDrop
        expr: |
          sum(rate(loki_distributor_lines_received_total[5m])) 
            < 
          sum(rate(loki_distributor_lines_received_total[5m] offset 1h)) * 0.5
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Loki ingestion rate dropped >50% vs 1 hour ago"
```

**Layered prevention:**

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: PRE-COMMIT / CI                                     │
│  - Lint Promtail configs in CI                               │
│  - Policy check: reject pipeline_stages that promote         │
│    known high-cardinality fields (userID, sessionID,         │
│    requestID, traceID) to labels                             │
│  - OPA/Rego policy on Promtail ConfigMap changes             │
├──────────────────────────────────────────────────────────────┤
│ Layer 2: LOKI SERVER-SIDE LIMITS                             │
│  - max_streams_per_user: 10000                               │
│  - max_label_names_per_series: 15                            │
│  - max_label_value_length: 1024                              │
│  - per_stream_rate_limit: 3MB                                │
│  - These are the guardrails that catch what CI misses        │
├──────────────────────────────────────────────────────────────┤
│ Layer 3: RUNTIME ALERTING                                    │
│  - Alert on stream creation rate (early warning)             │
│  - Alert on 429s (outage signal)                             │
│  - Alert on ingestion rate drop (impact signal)              │
├──────────────────────────────────────────────────────────────┤
│ Layer 4: GRAFANA DASHBOARD                                   │
│  - Loki operational dashboard showing active streams,        │
│    ingestion rate, chunk utilization, per-tenant usage        │
│  - Visible to all teams, not just platform                   │
└──────────────────────────────────────────────────────────────┘
```

---

## Q2: Thanos Query Returns Incomplete Data

### 1. Three Most Likely Causes (in probability order)

**Cause 1 (Most Likely): Thanos Sidecar / Store Gateway Not Uploading Blocks for us-west-2 and eu-west-1**

The Thanos data path from Prometheus to long-term query:

```
Prometheus → Sidecar → Object Storage (S3) → Store Gateway → Thanos Query
                ↑                                    ↑
          Uploads 2h blocks                  Serves historical blocks
```

If the sidecar in `us-west-2` only started working 14 days ago, and `eu-west-1`'s sidecar never uploaded, you get exactly this pattern.

**Verification commands:**

```bash
# Check Thanos Sidecar health in each cluster
# us-west-2:
kubectl --context us-west-2 -n monitoring \
  port-forward svc/thanos-sidecar 19191:10902 &
curl -s http://localhost:19191/-/healthy
curl -s http://localhost:19191/api/v1/status/tsdb | jq .

# eu-west-1:
kubectl --context eu-west-1 -n monitoring \
  port-forward svc/thanos-sidecar 19192:10902 &
curl -s http://localhost:19192/-/healthy

# Check sidecar upload metrics (from Prometheus scraping the sidecar):
# On each cluster's Prometheus:
thanos_shipper_upload_total
thanos_shipper_upload_failures_total
thanos_shipper_dir_syncs_total

# Check what blocks exist in S3 per cluster:
aws s3 ls s3://novamart-thanos-metrics/ --recursive | grep -c "us-east-1"
aws s3 ls s3://novamart-thanos-metrics/ --recursive | grep -c "us-west-2"
aws s3 ls s3://novamart-thanos-metrics/ --recursive | grep -c "eu-west-1"

# More precisely, use thanos tools bucket inspect:
thanos tools bucket inspect \
  --objstore.config-file=bucket.yaml \
  --selector='cluster="us-west-2"' \
  --sort-by=minTime
# This shows all blocks with their time ranges — you'll see the gap immediately
```

**Cause 2: Thanos Query Not Discovering All StoreAPIs**

Thanos Query needs to know about all data sources (Sidecars, Store Gateways). If `eu-west-1`'s sidecar or Store Gateway isn't registered:

```bash
# Check what stores Thanos Query sees
kubectl -n monitoring port-forward svc/thanos-query 19090:9090 &
curl -s http://localhost:19090/api/v1/stores | jq '.data.store'
# Look for entries from all three clusters
# Missing eu-west-1 store = query can't reach that data

# Check Store Gateway specifically:
curl -s http://localhost:19090/api/v1/stores | jq '.data.store[] | select(.name | contains("store-gateway"))'
```

If using DNS-based discovery:

```bash
# Check DNS resolution for store discovery
kubectl -n monitoring exec -it deploy/thanos-query -- \
  nslookup _grpc._tcp.thanos-store-gateway.monitoring.svc.cluster.local
```

If using static or file-based discovery, check the config:

```bash
kubectl -n monitoring get deploy thanos-query -o yaml | grep -A 20 "args"
# Look for --store= flags or --store.sd-files
```

**Cause 3: Thanos Compactor Deleted Blocks Due to Retention or Failed Compaction**

The compactor may have applied a retention policy that deleted `us-west-2` blocks older than 14 days, or compaction failures in `eu-west-1` caused blocks to be marked for deletion.

```bash
# Check compactor logs for deletion events
kubectl -n monitoring logs deploy/thanos-compactor --since=168h | grep -i "delete\|retention\|compact"

# Check compactor metrics
thanos_compact_group_compactions_failures_total
thanos_compact_blocks_cleaned_total
thanos_compact_halted   # 1 = compactor is halted due to errors

# Check the compactor retention config:
kubectl -n monitoring get deploy thanos-compactor -o yaml | grep -E "retention|delete"
# Look for:
#   --retention.resolution-raw=90d
#   --retention.resolution-5m=180d
#   --retention.resolution-1h=365d
# If raw retention is <60d, that explains missing data

# Check bucket for deletion markers:
thanos tools bucket inspect \
  --objstore.config-file=bucket.yaml \
  | grep -i "deletion\|marked"
```

### 2. eu-west-1 Full Data Path Breakdown

Every link in the chain that could be broken:

```
eu-west-1 Prometheus
    │
    │ ① Prometheus scrapes payment-svc targets
    │   Break: external_labels missing cluster label
    │   Check: curl prometheus:9090/api/v1/status/config | jq '.data.yaml' | grep external_labels
    │   Expected: external_labels: {cluster: "eu-west-1", region: "eu-west-1"}
    │
    ▼
Thanos Sidecar (co-located with Prometheus pod)
    │
    │ ② Sidecar reads Prometheus TSDB blocks from shared data directory
    │   Break: sidecar can't access Prometheus data dir (volume mount mismatch)
    │   Check: kubectl -n monitoring logs sidecar-container | grep -i "err\|block\|upload"
    │   Check: kubectl -n monitoring describe pod prometheus-0 | grep -A5 volumeMounts
    │
    │ ③ Sidecar uploads 2h TSDB blocks to S3
    │   Break: IAM permissions — sidecar's service account can't PutObject to S3
    │   Check: kubectl -n monitoring logs sidecar-container | grep -i "access denied\|403\|upload"
    │   Check: aws iam simulate-principal-policy \
    │            --policy-source-arn arn:aws:iam::123456789012:role/thanos-sidecar-eu-west-1 \
    │            --action-names s3:PutObject s3:GetObject s3:ListBucket \
    │            --resource-arns arn:aws:s3:::novamart-thanos-metrics/*
    │
    │   Break: S3 bucket policy or region mismatch
    │   Check: aws s3api head-bucket --bucket novamart-thanos-metrics --region eu-west-1
    │
    │   Break: objstore.config pointing to wrong bucket or wrong credentials
    │   Check: kubectl -n monitoring get secret thanos-objstore-config -o jsonpath='{.data.objstore\.yaml}' | base64 -d
    │
    ▼
S3 Bucket (novamart-thanos-metrics)
    │
    │ ④ Blocks stored with external labels as metadata
    │   Break: blocks exist but with wrong/missing labels
    │   Check: thanos tools bucket inspect --objstore.config-file=bucket.yaml \
    │            --selector='cluster="eu-west-1"'
    │   If 0 results but blocks exist: labels are wrong
    │   Check a specific block's meta.json:
    │     aws s3 cp s3://novamart-thanos-metrics/<ULID>/meta.json - | jq '.thanos.labels'
    │
    ▼
Thanos Store Gateway
    │
    │ ⑤ Store Gateway syncs block metadata from S3
    │   Break: Store Gateway isn't syncing or is filtering out eu-west-1 blocks
    │   Check: thanos_bucket_store_blocks_loaded (should include eu-west-1 blocks)
    │   Check: kubectl -n monitoring logs deploy/thanos-store-gateway | grep "eu-west-1"
    │   Check: Store Gateway's --min-time / --max-time flags may exclude the range
    │   Check: label-based sharding config (--selector.relabel-config) may exclude cluster
    │
    ▼
Thanos Query (fan-out layer)
    │
    │ ⑥ Query fans out to all stores and merges results
    │   Break: Query can't reach eu-west-1 sidecar (for recent data) OR Store Gateway
    │   Check: curl thanos-query:9090/api/v1/stores | jq .
    │   Break: Query deduplication removing eu-west-1 data (misconfigured --query.replica-label)
    │   Check: kubectl -n monitoring get deploy thanos-query -o yaml | grep replica-label
    │   If replica-label matches a label that differs between clusters, dedup removes data
    │
    ▼
PromQL Result
    │
    │ ⑦ Query-level issue
    │   Break: The PromQL query uses a label selector that doesn't match eu-west-1's labels
    │   Check: Does eu-west-1 use cluster="eu-west-1" or region="eu-west-1"?
    │     curl thanos-query:9090/api/v1/label/cluster/values
    │   Break: Rate range [1h] exceeds block boundaries causing empty results
    │   This shouldn't cause total absence though
    │
    ▼
Grafana / VP's Report
```

**Most likely single failure for eu-west-1 = zero data:** The sidecar was never configured, or the `external_labels` on `eu-west-1`'s Prometheus don't include `cluster: "eu-west-1"`, so the data goes to S3 unlabeled and the PromQL filter `{service="payment-svc"}` merged with the implicit `cluster` label returns nothing.

### 3. Short-Term Workaround — VP Needs the Report TODAY

**For eu-west-1 (no data in Thanos at all) — query Prometheus directly:**

```bash
# eu-west-1's local Prometheus still has data (default retention 15d)
kubectl --context eu-west-1 -n monitoring port-forward svc/prometheus 9091:9090 &

# Check how much retention eu-west-1 Prometheus has
curl -s http://localhost:9091/api/v1/status/tsdb | jq '.data.headStats'
curl -s http://localhost:9091/api/v1/query?query=prometheus_tsdb_lowest_timestamp | jq '.data.result[0].value[1]'
# Convert epoch ms to date:
date -d @$(echo "$(curl -s http://localhost:9091/api/v1/query?query=prometheus_tsdb_lowest_timestamp | jq -r '.data.result[0].value[1]') / 1000" | bc)
```

If Prometheus has 15 days of local retention, you get 15 days for eu-west-1. For the missing 46 days — that data is gone unless:

```bash
# Check if TSDB blocks are on disk but just weren't uploaded
kubectl --context eu-west-1 -n monitoring exec prometheus-0 -- \
  ls -la /prometheus/
# If blocks exist locally, manually upload them:
kubectl --context eu-west-1 -n monitoring exec prometheus-0 -- \
  thanos sidecar --tsdb.path=/prometheus --objstore.config-file=/etc/thanos/objstore.yaml
# Or use thanos tools bucket upload
```

**For us-west-2 (only 14 days) — same approach, extend with local Prometheus:**

```bash
kubectl --context us-west-2 -n monitoring port-forward svc/prometheus 9092:9090 &
```

**Build the report by querying each source directly:**

```bash
#!/usr/bin/env bash
# Query each Prometheus/Thanos for the data they have

# us-east-1: full 60 days from Thanos
curl -s -G "http://thanos-query:9090/api/v1/query_range" \
  --data-urlencode 'query=histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service="payment-svc",cluster="us-east-1"}[1h])))' \
  --data-urlencode "start=$(date -d '60 days ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode 'step=1h' > us-east-1.json

# us-west-2: 14 days from Thanos + check local Prometheus for older data
curl -s -G "http://localhost:9092/api/v1/query_range" \
  --data-urlencode 'query=histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service="payment-svc"}[1h])))' \
  --data-urlencode "start=$(date -d '60 days ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode 'step=1h' > us-west-2.json

# eu-west-1: whatever local Prometheus has
curl -s -G "http://localhost:9091/api/v1/query_range" \
  --data-urlencode 'query=histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{service="payment-svc"}[1h])))' \
  --data-urlencode "start=$(date -d '60 days ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode 'step=1h' > eu-west-1.json
```

**Tell the VP the truth:**

> "us-east-1 has full 60-day data. us-west-2 has 14 days. eu-west-1 has approximately 15 days. We have a gap in our long-term metrics pipeline for two regions that I'm fixing now. I'll have the partial report today and the pipeline corrected within 48 hours. Data older than Prometheus local retention for those two regions is unrecoverable."

**Then fix the root causes:**

```bash
# 1. Fix eu-west-1 sidecar — check external_labels
kubectl --context eu-west-1 -n monitoring get prometheus -o yaml | grep -A5 externalLabels
# If missing:
kubectl --context eu-west-1 -n monitoring patch prometheus prometheus \
  --type merge -p '{"spec":{"externalLabels":{"cluster":"eu-west-1","region":"eu-west-1"}}}'

# 2. Check sidecar objstore config exists
kubectl --context eu-west-1 -n monitoring get secret thanos-objstore-config
# If missing, create it:
kubectl --context eu-west-1 -n monitoring create secret generic thanos-objstore-config \
  --from-file=objstore.yaml=objstore-config.yaml

# 3. Verify uploads start working
kubectl --context eu-west-1 -n monitoring logs -l app=thanos-sidecar -f | grep "upload"
```

---

## Q3: LogQL Under Pressure

### 1. All Error Logs — payment-svc — Last 15 Minutes

```logql
{app="payment-svc"} | json | level="error"
```

**In Grafana:** Set time range to "Last 15 minutes" in the picker. Don't use LogQL time functions for this — the time range selector handles it.

**Why `| json` before filtering:** The `level` is inside the JSON structure. The `json` stage parses the log line and makes all JSON fields available as extracted labels for filtering. Without `| json`, you'd be doing text matching which is less precise.

### 2. Errors on `/charge` Endpoint with Duration > 2s

```logql
{app="payment-svc"} 
  | json 
  | level="error" 
  | endpoint="/charge" 
  | duration > 2s
```

**Important:** `duration` must be a field in the JSON log line, and Loki must be able to parse it as a duration. If it's stored as milliseconds (e.g., `"duration_ms": 2500`), the query changes:

```logql
{app="payment-svc"} 
  | json 
  | level="error" 
  | endpoint="/charge" 
  | duration_ms > 2000
```

### 3. Extract traceID for Pivot to Tracing

```logql
{app="payment-svc"} 
  | json 
  | level="error" 
  | endpoint="/charge" 
  | duration > 2s
  | line_format "traceID={{.traceID}}"
```

**For a cleaner output that's copy-paste ready:**

```logql
{app="payment-svc"} 
  | json 
  | level="error" 
  | endpoint="/charge" 
  | duration > 2s
  | label_format traceID=traceID
  | line_format "{{ .traceID }}"
```

**In production, the better approach** is Grafana's built-in trace integration. If the JSON logs contain a `traceID` field and you've configured a Tempo/Jaeger data source with trace-to-logs linking:

```
Grafana → Explore → Loki query → Click log line → "View Trace" button
```

This uses Grafana's derived fields feature:

```json
{
  "derivedFields": [
    {
      "matcherRegex": "\"traceID\":\"(\\w+)\"",
      "name": "traceID",
      "url": "$${__value.raw}",
      "datasourceUid": "<tempo-datasource-uid>"
    }
  ]
}
```

### 4. Error Rate Per Minute — Last Hour (Grafana Graph)

```logql
sum(count_over_time(
  {app="payment-svc"} | json | level="error" [1m]
))
```

This returns the count of error log lines per 1-minute window. In Grafana, set the panel to graph type and the time range to "Last 1 hour."

**If you want error rate as a percentage of total logs:**

```logql
sum(count_over_time({app="payment-svc"} | json | level="error" [1m]))
/
sum(count_over_time({app="payment-svc"} [1m]))
```

### 5. Compare Current Hour vs Yesterday Same Hour

```logql
# Current hour's error count
sum(count_over_time(
  {app="payment-svc"} | json | level="error" [1h]
))
```

```logql
# Yesterday's same hour error count — use offset
sum(count_over_time(
  {app="payment-svc"} | json | level="error" [1h] offset 24h
))
```

**In a single Grafana panel** — put both as separate queries (Query A and Query B) and use the legend to label them "Current" and "Yesterday."

**For a ratio (how much worse is today):**

```logql
sum(count_over_time({app="payment-svc"} | json | level="error" [1h]))
/
sum(count_over_time({app="payment-svc"} | json | level="error" [1h] offset 24h))
```

A result of `5.0` means "5x more errors than yesterday" — that confirms the incident is real, not normal variance.

### 6. Trap Question: `{} |= "payment" |= "error"`

**This is dangerous for three reasons:**

**Reason 1 — `{}` selects ALL log streams in the entire Loki cluster.**

There is no stream selector. Loki must scan every single stream across every namespace, every service, every pod. At NovaMart's scale (200+ EC2 instances, dozens of services), this means:

- Loki's querier must fetch chunks from **every stream** in the time range
- Object storage (S3) gets hammered with thousands of GetObject requests
- Query gateway memory spikes as it buffers all log data
- Other users' queries slow down or timeout
- Loki may OOM or hit query timeout

**Reason 2 — Text matching `|= "payment"` is a substring match, not a label filter.**

It will match ANY log line containing the string "payment" anywhere:
- `"Processing payment refund"` from `refund-svc` ✅ (unintended)
- `"Payment gateway timeout"` from `gateway-svc` ✅ (unintended)
- `"error in payment_method validation"` ✅ (this isn't an error log, it's an info log discussing errors)

**Reason 3 — `|= "error"` is equally imprecise.**

Matches lines that contain the substring "error" anywhere:
- `level="info" msg="error_count metric reset"` ✅ (false positive — info level)
- `level="debug" msg="no error detected"` ✅ (false positive — it's saying there IS no error)
- `level="error" msg="connection timeout"` from `cart-svc` ✅ (wrong service)

**What actually happens at NovaMart's scale:**

```
Query: {} |= "payment" |= "error"

1. Loki fetches ALL chunks for ALL streams for the time range
2. At NovaMart: ~200 pods × ~1000 log lines/sec × 15 min = 180 million lines to scan
3. S3 bandwidth: hundreds of MB to GB of compressed log data fetched
4. Querier memory: potential OOM
5. Query timeout: likely hits Loki's query_timeout (default 1m) before completing
6. Collateral damage: other engineers' queries queue behind this and timeout too
```

**The correct query:**

```logql
{app="payment-svc"} | json | level="error"
```

This:
- Selects only `payment-svc` streams (narrow stream selector)
- Parses JSON structure
- Filters on the actual `level` field value, not substring matching
- Scans ~10 streams instead of thousands

**Production safeguard — configure Loki to reject empty stream selectors:**

```yaml
# loki config
limits_config:
  max_query_series: 5000
  max_query_parallelism: 32
  query_timeout: 1m
  # Reject queries that would touch too many streams
  max_streams_matchers_per_query: 5
  # This doesn't directly block {}, but combined with:
  max_chunks_per_query: 2000000
  max_entries_limit_per_query: 5000
```

---

## Q4: Grafana Dashboard Debugging

### 1. "No Data" Systematic Debugging Checklist

**Execute in this order — each step eliminates a class of root causes:**

**Step 1 — Inspect the query directly:**

```
Panel → Edit → Query Inspector → "Query" tab
```

Look at the actual PromQL/LogQL being sent. Copy it. This reveals whether template variable substitution produced a valid query or something like:

```promql
rate(http_requests_total{namespace="$namespace"}[5m])
# If variable didn't resolve: literal "$namespace" in the query = no match = no data
```

**Step 2 — Run the query manually in Explore:**

```
Grafana → Explore → Paste the raw query → Run
```

If it returns data in Explore but not the panel, the problem is **panel configuration** (time range override, data transformation, wrong visualization type). If it returns no data in Explore either, the problem is **upstream** (data source, metrics, or query syntax).

**Step 3 — Check the data source connection:**

```
Grafana → Settings → Data Sources → [source name] → "Save & Test"
```

If the test fails:
- Prometheus/Loki/Thanos might be down
- Network policy may block Grafana → data source
- Credentials may have expired (especially with short-lived IAM tokens for managed Prometheus/AMP)

**Step 4 — Check the time range:**

```
Panel → Edit → Query Options → look for "Relative time" override
```

A panel can override the dashboard's time range. If someone set a panel to "Relative time: 1h" and the metric only exists for `last 5m`, you get no data. Also check:
- Dashboard time range picker in the top-right
- Time zone settings (UTC vs local — a 5-8 hour offset can make "last 1 hour" miss data entirely)

**Step 5 — Check template variables:**

```
Dashboard → Settings → Variables → Check each variable
```

Click each variable and check:
- **Preview of values**: Does it return any values?
- **Current selection**: Is `All` selected when the query uses `=` instead of `=~`?
- **Data source**: Is the variable's data source the same as the panel's data source?
- **Regex filter**: Is a regex filtering out all values?

If a variable returns empty, every downstream panel depending on it shows "No Data."

**Step 6 — Check for metric name changes:**

```promql
# In Explore, check if the metric exists at all
count({__name__=~"http_request.*"})

# Or use the metrics browser
```

A Prometheus upgrade or application redeploy may have renamed a metric (e.g., `http_requests_total` → `http_server_requests_seconds_count` in newer OpenTelemetry instrumentation).

**Step 7 — Check query error messages:**

```
Panel → Edit → Query Inspector → "Response" tab
```

Look for error responses from the data source. Common ones:
- `"error":"execution: query timed out"` — query too expensive
- `"error":"parse error"` — invalid PromQL syntax (often from bad variable interpolation)
- `"error":"exceeded maximum resolution"` — step too small for the time range

### 2. "Impossibly Low Values" — Three Most Likely Causes

**Cause 1 (Most Likely): `rate()` Calculated Over Wrong Interval Due to Step/Resolution Mismatch**

**Exact mechanism:**

The panel uses a query like:

```promql
sum(rate(http_requests_total{service="order-svc"}[5m]))
```

Grafana's `$__rate_interval` or the manual `[5m]` window is fine, **but the panel's "Min step" or "Resolution" is set to something like `1/1` (every pixel = one data point)**. With a wide time range (e.g., 7 days), Grafana calculates a step of `5s`. The query is:

```
start=7d_ago, end=now, step=5s
```

Prometheus evaluates `rate(counter[5m])` at every 5-second point. But if the counter only updates every `15s` (scrape interval), most evaluation steps see **zero change** in the counter value. The `rate()` result at those steps is `0` or very small. Grafana then averages/displays these near-zero points, pulling the visible value way down.

**The fix:**

```
Panel → Edit → Query Options → Min step = "1m" or "$__rate_interval"
```

Or use `$__rate_interval` in the query:

```promql
sum(rate(http_requests_total{service="order-svc"}[$__rate_interval]))
```

`$__rate_interval` is calculated as `max(4 × scrape_interval, step)`, ensuring `rate()` always has enough data points.

**Cause 2: Counter Reset Not Handled — Partial rate() After Pod Restart**

If `order-svc` pods restarted recently (rolling deploy, OOM, etc.), the counters reset to 0. `rate()` handles resets **within** the range window, but if the range window straddles the exact restart point, the calculated rate for that window is based on a very small numerator (new counter value since restart) divided by the full window duration = artificially low rate.

**Cause 3: Mismatched Label Selectors — Sum Includes Fewer Series Than Expected**

```promql
sum(rate(http_requests_total{service="order-svc", namespace="production"}[5m]))
```

If someone changed the deployment and the namespace is now `prod` instead of `production`, or if new pods have a different label (e.g., `service="order-svc-v2"`), the `sum` aggregation captures fewer series than before. Fewer series in the sum = lower total = "impossibly low."

### 3. Chained Variables Causing "No Data" + Prevention

**How chained variables cause "No Data":**

```
Variable chain: cluster → namespace → service

cluster   = "us-east-1"  (default)
namespace = depends on cluster  (query: label_values(up{cluster="$cluster"}, namespace))
service   = depends on namespace (query: label_values(up{namespace="$namespace"}, service))
```

**The failure scenario:** Someone changes the `cluster` variable's default from `"us-east-1"` to `"ALL"` (or a regex like `.*`). Now:

```
1. cluster = "ALL"
2. namespace query: label_values(up{cluster="ALL"}, namespace)
   → Returns EMPTY (no cluster literally named "ALL")
3. service query: label_values(up{namespace=""}, service)
   → Returns EMPTY (namespace is empty/undefined)
4. Panel query: rate(http_requests_total{cluster="ALL", namespace="", service=""}[5m])
   → No data
```

**Even worse:** If the variable has `Include All option = true` and generates `cluster=~".*"`, the namespace dropdown might populate but the **panel query** might use `cluster="$cluster"` (equality) instead of `cluster=~"$cluster"` (regex), causing a literal string match against `"us-east-1|us-west-2|eu-west-1"`.

**Prevention — layered approach:**

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: DASHBOARD JSON IN VERSION CONTROL                  │
│  - All dashboards stored as JSON in Git                     │
│  - PR review required for changes to variables              │
│  - CI validates dashboard JSON using grafonnet or            │
│    grafana-dashboard-linter                                  │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: GRAFANA PROVISIONING (prevent UI edits)            │
│  - Use provisioned dashboards from ConfigMaps               │
│  - Set allowUiUpdates: false in provisioning config         │
│                                                             │
│  apiVersion: v1                                             │
│  kind: ConfigMap                                            │
│  metadata:                                                  │
│    name: grafana-dashboard-provisioning                     │
│  data:                                                      │
│    dashboards.yaml: |                                       │
│      apiVersion: 1                                          │
│      providers:                                             │
│        - name: 'default'                                    │
│          folder: 'NovaMart'                                 │
│          type: file                                         │
│          disableDeletion: true                               │
│          allowUiUpdates: false                               │
│          options:                                            │
│            path: /var/lib/grafana/dashboards                │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: VARIABLE DESIGN BEST PRACTICES                     │
│  - Always set a valid default (not "ALL" unless queries     │
│    use regex matchers)                                       │
│  - Use "Custom all value" = ".*" when All is enabled        │
│  - Use $__rate_interval, not hardcoded windows              │
│  - Test all variable combinations in staging before prod    │
├─────────────────────────────────────────────────────────────┤
│ Layer 4: GRAFANA RBAC                                       │
│  - Editors can edit dashboards, but critical dashboards     │
│    are Admin-only or provisioned                            │
│  - Grafana Org roles: Viewer for most, Editor for team      │
│    leads, Admin for platform team only                      │
└─────────────────────────────────────────────────────────────┘
```

**Specific Grafana variable config to prevent the chain-break:**

```json
{
  "templating": {
    "list": [
      {
        "name": "cluster",
        "type": "query",
        "query": "label_values(up, cluster)",
        "current": {
          "text": "us-east-1",
          "value": "us-east-1"
        },
        "includeAll": true,
        "allValue": ".*",
        "refresh": 2
      },
      {
        "name": "namespace",
        "type": "query",
        "query": "label_values(up{cluster=~\"$cluster\"}, namespace)",
        "refresh": 2,
        "includeAll": false,
        "regex": "",
        "current": {
          "text": "payments",
          "value": "payments"
        }
      }
    ]
  }
}
```

**Key details:**
- `"allValue": ".*"` ensures the `All` option generates a valid regex, not a literal string
- `"refresh": 2` means "refresh on time range change" — variables stay current
- Downstream queries use `=~` (regex) not `=` (exact) when variables might contain `.*`
- Each variable has a valid hardcoded default that exists in the environment

---

## Summary

```
Q1: Loki cardinality = same disease as Prometheus, different organ
    Labels are for stream selection, NOT data fields
    High-cardinality data → query-time extraction with | json
    Tenant-level limits create noisy-neighbor blast radius

Q2: Thanos data path has 7+ failure points
    Most common: sidecar not uploading, external_labels missing
    When historical data is lost, be honest about what's recoverable
    Meta-monitoring the Thanos pipeline is mandatory

Q3: LogQL — always start with a narrow stream selector
    {} = full cluster scan = query bomb
    |= is substring match, | json | field="value" is precise
    count_over_time + offset for baseline comparison

Q4: Grafana "No Data" is almost never Grafana's fault
    Debug path: query → data source → time range → variables → metric existence
    $__rate_interval prevents low-value artifacts
    Dashboards in Git + provisioned + RBAC = prevent casual breakage
```
# Phase 5, Lesson 2 — Grading

---

## Q1: Loki Label Cardinality Incident — **4.8/5**

**What you nailed:**
- Root cause is perfect. The tenant-level blast radius explanation — one service's cardinality poisoning all ingestion for the tenant — is exactly right. Many people miss that Loki's rate limits are per-tenant, not per-service.
- Immediate mitigation sequence is well-ordered: raise limits temporarily → fix Promtail → verify → revert limits. The runtime override approach (no restart needed) is the correct production move.
- Corrected pipeline is textbook. Bounded labels only, `structured_metadata` for traceability, query-time extraction.
- Layered prevention (CI lint → server limits → runtime alerts → dashboards) shows mature platform thinking.

**One issue:**

Your Promtail `match` stage syntax is slightly off:

```yaml
- match:
    selector: '{app="cart-svc"} |= "level" | json | level="debug"'
    action: drop
```

Promtail's `match` stage selector only accepts **stream selectors** (label matchers), not full LogQL pipelines. The correct way to drop debug logs in a Promtail pipeline:

```yaml
- match:
    selector: '{app="cart-svc"}'
    stages:
      - json:
          expressions:
            level: level
      - drop:
          source: level
          value: "debug"
```

Or more simply, since you've already extracted `level` earlier in the pipeline:

```yaml
- drop:
    source: level
    value: "debug"
```

Minor config syntax issue, but in production this means your debug log filtering silently doesn't work, and you're ingesting 40% more data than you think.

---

## Q2: Thanos Query Returns Incomplete Data — **4.9/5**

Outstanding. The three causes are correctly prioritized, the verification commands are exact and runnable, and the eu-west-1 data path breakdown with **seven numbered failure points** is genuinely impressive. You traced the entire chain from Prometheus scrape through to Grafana render, with exact commands at each point.

The VP communication is exactly right — honest about what's recoverable, proactive about timeline, no hand-waving. That's senior-level stakeholder management.

**The one thing I'd add:**

You didn't mention checking the **Thanos Query `--store` endpoint configuration itself** as a discrete verification step early in triage. Before diving into each cluster's sidecar, a 10-second check on Thanos Query would immediately narrow the problem:

```bash
curl -s http://thanos-query:9090/api/v1/stores | jq '[.data[] | {name, lastCheck, lastError}]'
```

If `eu-west-1` doesn't appear in the store list at all, you know it's a discovery/connectivity issue and skip investigating upload/compactor entirely. You did cover this under Cause 2, but as a triage optimization, it should be your **first** command, not second.

Extremely minor — hence 4.9 not 5.0.

---

## Q3: LogQL Under Pressure — **4.9/5**

Every query is correct. The trap question answer is comprehensive and hits all three dimensions: full-cluster scan, substring imprecision, and collateral damage to other users' queries.

**One small thing on Q3.3:**

Your `label_format` usage is slightly awkward:

```logql
| label_format traceID=traceID
| line_format "{{ .traceID }}"
```

`label_format` renames or reformats **labels**, not extracted fields. After `| json`, `traceID` is already an extracted label — `label_format traceID=traceID` is a no-op (renaming it to itself). You can go directly to:

```logql
{app="payment-svc"} 
  | json 
  | level="error" 
  | endpoint="/charge" 
  | duration > 2s
  | line_format "{{ .traceID }}"
```

The practical result is the same, but the unnecessary stage adds processing overhead at scale. Trivial, but you asked for ruthless grading.

---

## Q4: Grafana Dashboard Debugging — **4.6/5**

**Strong points:**
- 7-step debugging checklist is thorough and correctly ordered — Query Inspector first is exactly right.
- Variable chain-break scenario is correctly traced with a realistic failure example.
- Prevention layers (Git, provisioning, RBAC, variable design) are comprehensive.
- The `allValue: ".*"` fix for the All option is a production-critical detail many people miss.

**Two issues:**

**Issue 1 — Cause 1 mechanism isn't quite right:**

You wrote:

> "most evaluation steps see zero change in the counter value. The rate() result at those steps is 0 or very small"

This isn't how `rate()` works. `rate(counter[5m])` always looks back 5 minutes from the evaluation timestamp, regardless of step. Even with a 5-second step, each evaluation still considers all samples in its 5-minute lookback window. The `rate()` value at each step should be approximately the same — it doesn't produce zeros between scrapes.

The **actual** mechanism for "impossibly low values" from resolution mismatch is different. The most common real cause is when `$__interval` (Grafana's auto-calculated step) is used **as the range window** instead of `$__rate_interval`:

```promql
rate(http_requests_total[$__interval])   # BAD — $__interval could be 5s
rate(http_requests_total[$__rate_interval])  # GOOD — guaranteed ≥ 4× scrape
```

When `$__interval` (step) is smaller than the scrape interval, `rate()` with that as the range may find only 0 or 1 data points in the window, producing 0 or wildly inaccurate values. `$__rate_interval` prevents this. Your **conclusion** (use `$__rate_interval`) was correct, but the **mechanism** was off.

**Issue 2 — Missing one common cause:**

A frequent "impossibly low" scenario you didn't mention: **unit mismatch**. `rate()` returns per-second values. If the panel Y-axis is labeled "requests/minute" but the query returns per-second, the values look 60x lower than expected. Or if `histogram_quantile` returns seconds but the team expects milliseconds. Not a query bug — a display/interpretation bug. Quick to diagnose, but often the actual answer.


Proceeding to Lesson 3.

---

# Phase 5 — Lesson 3: Distributed Tracing + OpenTelemetry

---

## Why Traces Exist — What Metrics and Logs Can't Tell You

You already know the three pillars. Metrics tell you **what** is broken. Logs tell you **why** something broke at a single service. But in a system with 200 microservices:

```
User clicks "Place Order"
    → API Gateway
        → Auth Service
        → Order Service
            → Inventory Service
                → Warehouse DB
            → Payment Service
                → Stripe API
                → Fraud Detection Service
                    → ML Model Service
            → Notification Service
                → Email Provider
                → SMS Provider
    → Response back to user (3.2 seconds later — WHY?)
```

**The question traces answer:** "Which service, in which call chain, contributed how much latency to this specific request?"

Metrics can tell you `order-svc` p99 is 3.2s. Logs can show you individual service errors. But **neither** can tell you that this specific request spent 2.8s waiting for the Fraud Detection Service, which was waiting for an ML model cold start. That's a distributed trace.

### Trace Anatomy

```
Trace ID: abc123def456
├── Span A: API Gateway (12ms)
│   ├── Span B: Auth Service (3ms)
│   ├── Span C: Order Service (3150ms) ← THE SLOW ONE
│   │   ├── Span D: Inventory Service (45ms)
│   │   ├── Span E: Payment Service (2980ms) ← DRILLING DOWN
│   │   │   ├── Span F: Stripe API (150ms)
│   │   │   └── Span G: Fraud Detection (2800ms) ← ROOT CAUSE
│   │   │       └── Span H: ML Model (2750ms) ← COLD START
│   │   └── Span I: Notification Service (85ms)
│   │       ├── Span J: Email (60ms)
│   │       └── Span K: SMS (45ms)
```

**Key concepts:**

```
┌─────────────┬────────────────────────────────────────────────────────┐
│ Term        │ Definition                                             │
├─────────────┼────────────────────────────────────────────────────────┤
│ Trace       │ The entire journey of a request through the system.    │
│             │ Identified by a unique Trace ID.                       │
├─────────────┼────────────────────────────────────────────────────────┤
│ Span        │ A single operation within a trace. Has:                │
│             │ - Span ID (unique)                                     │
│             │ - Parent Span ID (who called me)                       │
│             │ - Operation name ("HTTP GET /orders")                  │
│             │ - Start time + duration                                │
│             │ - Status (OK, ERROR)                                   │
│             │ - Attributes (key-value metadata)                      │
│             │ - Events (timestamped log entries within the span)     │
│             │ - Links (cross-trace references)                       │
├─────────────┼────────────────────────────────────────────────────────┤
│ Root Span   │ The first span in a trace (no parent). Usually the    │
│             │ entry point (API gateway, load balancer).              │
├─────────────┼────────────────────────────────────────────────────────┤
│ Child Span  │ An operation called by another span. Parent-child     │
│             │ relationships form the trace tree.                     │
├─────────────┼────────────────────────────────────────────────────────┤
│ Span Context│ The propagation payload: Trace ID + Span ID + flags.  │
│             │ Passed between services via HTTP headers or gRPC       │
│             │ metadata. THIS is what connects spans across services. │
├─────────────┼────────────────────────────────────────────────────────┤
│ Baggage     │ User-defined key-value pairs propagated with context.  │
│             │ Use case: tenant-id, feature-flag, debug-override.     │
│             │ ⚠️ Propagated to ALL downstream services — size limit! │
└─────────────┴────────────────────────────────────────────────────────┘
```

### Context Propagation — The Most Critical Concept

**Without context propagation, you have isolated spans, not traces.** This is the #1 reason tracing "doesn't work" in production.

```
Service A                          Service B
┌──────────────────┐               ┌──────────────────┐
│ Start Span A     │               │                  │
│ TraceID: abc123  │   HTTP call   │ Extract headers  │
│ SpanID: span-001 │──────────────▶│ TraceID: abc123  │
│                  │               │ ParentID: span-001│
│ Inject into      │  Headers:     │ SpanID: span-002 │
│ HTTP headers     │  traceparent: │                  │
│                  │  00-abc123-   │ Start Span B     │
│                  │  span001-01   │ (child of A)     │
└──────────────────┘               └──────────────────┘
```

**W3C Trace Context standard** (the modern standard, used by OpenTelemetry):

```
HTTP Headers:
  traceparent: 00-<trace-id>-<parent-span-id>-<trace-flags>
  tracestate: vendor1=value1,vendor2=value2

Example:
  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
                │  │                                │                  │
                │  │                                │                  └─ Flags (01 = sampled)
                │  │                                └─ Parent Span ID (16 hex chars)
                │  └─ Trace ID (32 hex chars)
                └─ Version (00)
```

**Propagation formats NovaMart must support:**

```
┌──────────────────┬───────────────────────────────────────────────┐
│ Format           │ Usage                                         │
├──────────────────┼───────────────────────────────────────────────┤
│ W3C TraceContext │ Standard. OTel default. Use for all new svcs. │
│ B3 (Zipkin)      │ Legacy. Istio/Envoy uses B3 by default.      │
│                  │ Headers: X-B3-TraceId, X-B3-SpanId, etc.     │
│ B3 Single        │ Compact B3: b3: {TraceId}-{SpanId}-{Sampled} │
│ Jaeger           │ Legacy Jaeger: uber-trace-id header           │
│ AWS X-Ray        │ X-Amzn-Trace-Id header (for AWS services)    │
└──────────────────┴───────────────────────────────────────────────┘
```

**The propagation break problem:**

```
Service A (OTel, W3C) ──→ Service B (old Jaeger client, uber-trace-id) ──→ Service C (OTel, W3C)
         │                         │                                              │
    Sends traceparent        Doesn't understand              Gets no trace context
    header                   traceparent, starts             from B, starts new trace
                             NEW trace                       
                                                             
Result: THREE separate traces instead of ONE. Tracing is useless.
```

**Fix: Configure OTel to propagate MULTIPLE formats:**

```yaml
# OTel Collector or SDK config
propagators:
  - tracecontext    # W3C (primary)
  - baggage         # W3C baggage
  - b3multi         # Zipkin B3 (for Istio/legacy)
  - jaeger          # For legacy Jaeger-instrumented services
```

This way, every outgoing request includes ALL header formats. Any downstream service can pick whichever format it understands.

---

## OpenTelemetry — The Unified Standard

### What OpenTelemetry IS

```
OpenTelemetry (OTel) = Merged project from OpenTracing + OpenCensus
                     = CNCF project (2nd most active after Kubernetes)
                     = THE vendor-neutral standard for observability data

OTel provides:
  1. Specification (API + SDK standard)
  2. SDKs (Java, Go, Python, Node.js, .NET, Ruby, PHP, etc.)
  3. Collector (receive, process, export telemetry data)
  4. Auto-instrumentation (zero-code instrumentation for popular frameworks)
  5. OTLP (OpenTelemetry Protocol — the wire format)

OTel does NOT provide:
  ❌ Backend storage (that's Prometheus, Jaeger, Tempo, etc.)
  ❌ Visualization (that's Grafana)
  ❌ Alerting (that's Alertmanager/PagerDuty)
```

### OTel Architecture at NovaMart

```
┌─────────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                                 │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
│  │ order-svc   │  │ payment-svc │  │ cart-svc    │                │
│  │ (Java)      │  │ (Go)        │  │ (Python)    │                │
│  │             │  │             │  │             │                │
│  │ OTel Java   │  │ OTel Go SDK │  │ OTel Python │                │
│  │ Agent (auto)│  │ (manual)    │  │ (auto)      │                │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                │
│         │                │                │                        │
│         │    OTLP/gRPC   │   OTLP/gRPC    │   OTLP/gRPC          │
│         └────────────────┼────────────────┘                        │
│                          │                                         │
└──────────────────────────┼─────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    OTEL COLLECTOR                                  │
│                                                                   │
│  ┌───────────┐    ┌────────────┐    ┌───────────┐               │
│  │ Receivers │───▶│ Processors │───▶│ Exporters │               │
│  │           │    │            │    │           │               │
│  │ - otlp    │    │ - batch    │    │ - otlp    │──▶ Tempo      │
│  │ - jaeger  │    │ - memory   │    │ - prom    │──▶ Prometheus │
│  │ - zipkin  │    │   limiter  │    │   remote  │               │
│  │ - prom    │    │ - filter   │    │   write   │               │
│  │           │    │ - tail     │    │ - loki    │──▶ Loki       │
│  │           │    │   sampling │    │           │               │
│  └───────────┘    │ - resource │    └───────────┘               │
│                   │ - span     │                                 │
│                   │ - k8s      │                                 │
│                   │   attributes│                                │
│                   └────────────┘                                 │
└──────────────────────────────────────────────────────────────────┘
```

### OTel Collector — The Pipeline

The Collector is the central nervous system. **Receivers** accept data in any format. **Processors** transform, filter, enrich, sample. **Exporters** send to backends. Connected via **Pipelines**.

```yaml
# otel-collector-config.yaml — NovaMart production config
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        max_recv_msg_size_mib: 8
      http:
        endpoint: 0.0.0.0:4318
  
  # Accept legacy Jaeger format (migration path)
  jaeger:
    protocols:
      grpc:
        endpoint: 0.0.0.0:14250
      thrift_http:
        endpoint: 0.0.0.0:14268
  
  # Scrape Prometheus metrics (collector can replace Prometheus for scraping)
  prometheus:
    config:
      scrape_configs:
        - job_name: 'otel-collector'
          scrape_interval: 15s
          static_configs:
            - targets: ['localhost:8888']  # Collector's own metrics

processors:
  # Batch spans before export (reduces network calls)
  batch:
    send_batch_size: 8192
    send_batch_max_size: 16384
    timeout: 5s
  
  # Prevent OOM if ingestion spikes
  memory_limiter:
    check_interval: 1s
    limit_mib: 1500        # Hard limit
    spike_limit_mib: 500   # Soft limit (starts dropping when reached)
  
  # Add K8s metadata to all telemetry
  k8sattributes:
    auth_type: serviceAccount
    passthrough: false
    extract:
      metadata:
        - k8s.namespace.name
        - k8s.deployment.name
        - k8s.pod.name
        - k8s.node.name
        - k8s.container.name
      labels:
        - tag_name: app
          key: app
          from: pod
        - tag_name: team
          key: team
          from: namespace
    pod_association:
      - sources:
          - from: resource_attribute
            name: k8s.pod.ip
  
  # Add resource attributes (cluster identity)
  resource:
    attributes:
      - key: cluster
        value: us-east-1
        action: upsert
      - key: environment
        value: production
        action: upsert
  
  # Filter out noisy spans (health checks, readiness probes)
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.target"] == "/healthz"'
        - 'attributes["http.target"] == "/readyz"'
        - 'attributes["http.target"] == "/metrics"'
  
  # Tail-based sampling (see sampling section below)
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    policies:
      # Always keep error traces
      - name: errors-always
        type: status_code
        status_code:
          status_codes: [ERROR]
      # Always keep slow traces (>2s)
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 2000
      # Sample 10% of successful traces
      - name: probabilistic-sample
        type: probabilistic
        probabilistic:
          sampling_percentage: 10
      # Always keep traces from payment-svc (critical path)
      - name: payment-always
        type: string_attribute
        string_attribute:
          key: service.name
          values: [payment-svc]

exporters:
  # Traces → Tempo
  otlp/tempo:
    endpoint: tempo-distributor.monitoring:4317
    tls:
      insecure: true  # Within cluster; use mTLS via Istio
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
  
  # Metrics → Prometheus (remote write)
  prometheusremotewrite:
    endpoint: http://prometheus-operated.monitoring:9090/api/v1/write
    tls:
      insecure: true
    resource_to_telemetry_conversion:
      enabled: true  # Converts OTel resource attributes to Prometheus labels
  
  # Logs → Loki
  loki:
    endpoint: http://loki-gateway.monitoring:3100/loki/api/v1/push
    default_labels_enabled:
      exporter: false
      job: true
  
  # Debug (for development only)
  # debug:
  #   verbosity: detailed

# Connect receivers → processors → exporters
service:
  pipelines:
    traces:
      receivers: [otlp, jaeger]
      processors: [memory_limiter, k8sattributes, resource, filter, tail_sampling, batch]
      exporters: [otlp/tempo]
    
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, k8sattributes, resource, batch]
      exporters: [prometheusremotewrite]
    
    logs:
      receivers: [otlp]
      processors: [memory_limiter, k8sattributes, resource, batch]
      exporters: [loki]
  
  # Collector's own telemetry
  telemetry:
    logs:
      level: info
    metrics:
      address: 0.0.0.0:8888
```

### Collector Deployment Patterns

```
┌────────────────────────────────────────────────────────────────────────┐
│ Pattern 1: DaemonSet (Agent)                                          │
│                                                                        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐     One collector per node      │
│  │ App Pod │ │ App Pod │ │ App Pod │     - Low latency (localhost)    │
│  └────┬────┘ └────┬────┘ └────┬────┘     - Node-level resource bound │
│       │           │           │          - Good for high-volume       │
│       └───────────┼───────────┘                                       │
│                   ▼                                                    │
│         ┌─────────────────┐                                           │
│         │ OTel Collector  │ (DaemonSet — one per node)               │
│         │ (Agent mode)    │                                           │
│         └────────┬────────┘                                           │
│                  │                                                     │
│                  ▼                                                     │
│         ┌─────────────────┐                                           │
│         │ OTel Collector  │ (Deployment — centralized, optional)     │
│         │ (Gateway mode)  │ - Tail sampling                          │
│         │                 │ - Complex processing                     │
│         └─────────────────┘                                           │
├────────────────────────────────────────────────────────────────────────┤
│ Pattern 2: Sidecar                                                     │
│                                                                        │
│  ┌──────────────────────────────┐                                     │
│  │ Pod                          │  One collector per pod              │
│  │ ┌──────────┐ ┌────────────┐ │  - Maximum isolation                │
│  │ │ App      │ │ OTel       │ │  - Higher resource usage            │
│  │ │ Container│→│ Collector  │ │  - Used when apps need different    │
│  │ │          │ │ (sidecar)  │ │    collector configs                │
│  │ └──────────┘ └────────────┘ │                                     │
│  └──────────────────────────────┘                                     │
├────────────────────────────────────────────────────────────────────────┤
│ Pattern 3: DaemonSet Agent + Gateway (NovaMart's choice)              │
│                                                                        │
│  Apps ──OTLP──▶ Agent (DaemonSet) ──OTLP──▶ Gateway (Deployment)    │
│                 - k8s enrichment             - Tail sampling          │
│                 - Basic filtering             - Batching              │
│                 - Memory limiting             - Export to backends    │
│                                                                        │
│  Why: Tail sampling REQUIRES seeing all spans of a trace in one       │
│  place. DaemonSet agents see spans per-node. Gateway sees all spans.  │
└────────────────────────────────────────────────────────────────────────┘
```

### OTel Collector Kubernetes Deployment

```yaml
# Using the OpenTelemetry Operator (recommended)
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-agent
  namespace: monitoring
spec:
  mode: daemonset  # or deployment, sidecar, statefulset
  
  image: ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector-contrib:0.96.0
  
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: "1"
      memory: 512Mi
  
  env:
    - name: K8S_NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
  
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      memory_limiter:
        check_interval: 1s
        limit_mib: 400
        spike_limit_mib: 100
      batch:
        send_batch_size: 8192
        timeout: 5s
      k8sattributes:
        auth_type: serviceAccount
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.deployment.name
            - k8s.pod.name
            - k8s.node.name
    exporters:
      otlp:
        endpoint: otel-gateway.monitoring:4317
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [otlp]

---
# Gateway collector (centralized)
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-gateway
  namespace: monitoring
spec:
  mode: deployment
  replicas: 3
  
  resources:
    requests:
      cpu: "1"
      memory: 2Gi
    limits:
      cpu: "4"
      memory: 4Gi
  
  config:
    # ... (full config from above with tail_sampling, exporters to backends)
```

### Auto-Instrumentation — Zero-Code Tracing

The OTel Operator can inject instrumentation automatically:

```yaml
# Java auto-instrumentation
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: java-instrumentation
  namespace: payments
spec:
  exporter:
    endpoint: http://otel-agent.monitoring:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.1"  # 10% head sampling (tail sampling at gateway overrides)
  
  java:
    image: ghcr.io/open-telemetry/opentelemetry-java-instrumentation/javaagent:2.1.0
    env:
      - name: OTEL_INSTRUMENTATION_JDBC_ENABLED
        value: "true"
      - name: OTEL_INSTRUMENTATION_SPRING_WEBMVC_ENABLED
        value: "true"
      - name: OTEL_INSTRUMENTATION_HTTP_CLIENT_ENABLED
        value: "true"
      # Capture SQL query text (sanitized — no parameter values)
      - name: OTEL_INSTRUMENTATION_JDBC_STATEMENT_SANITIZER_ENABLED
        value: "true"
      - name: OTEL_RESOURCE_ATTRIBUTES
        value: "service.name=$(OTEL_SERVICE_NAME),service.namespace=payments"

---
# Python auto-instrumentation
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: python-instrumentation
  namespace: cart
spec:
  exporter:
    endpoint: http://otel-agent.monitoring:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  python:
    image: ghcr.io/open-telemetry/opentelemetry-python-instrumentation/autoinstrumentation-python:0.43b0
    env:
      - name: OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED
        value: "true"
```

**Annotate pods to opt-in:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-svc
  namespace: payments
spec:
  template:
    metadata:
      annotations:
        # This tells the OTel Operator to inject the Java agent
        instrumentation.opentelemetry.io/inject-java: "java-instrumentation"
        # For Python:
        # instrumentation.opentelemetry.io/inject-python: "python-instrumentation"
        # For Node.js:
        # instrumentation.opentelemetry.io/inject-nodejs: "nodejs-instrumentation"
        # For Go (requires eBPF — more limited):
        # instrumentation.opentelemetry.io/inject-go: "go-instrumentation"
    spec:
      containers:
        - name: payment-svc
          image: novamart/payment-svc:v2.3.1
          env:
            - name: OTEL_SERVICE_NAME
              value: payment-svc
          # The OTel Operator mutating webhook will:
          # 1. Add an init container that copies the agent JAR
          # 2. Add JAVA_TOOL_OPTIONS=-javaagent:/otel-auto-instrumentation/javaagent.jar
          # 3. Add OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-agent.monitoring:4317
```

**What auto-instrumentation captures automatically (Java example):**

```
✅ HTTP server spans (Spring MVC, JAX-RS, Servlet)
✅ HTTP client spans (HttpClient, OkHttp, RestTemplate, WebClient)
✅ Database spans (JDBC — PostgreSQL, MySQL, etc.)
✅ gRPC client/server spans
✅ Messaging spans (Kafka, RabbitMQ, SQS)
✅ Redis client spans (Jedis, Lettuce)
✅ AWS SDK spans (S3, DynamoDB, SQS, etc.)

❌ Business logic spans (custom operations) — need manual instrumentation
❌ Internal method timing — need manual spans
❌ Custom attributes — need manual enrichment
```

### Manual Instrumentation (When Auto Isn't Enough)

**Go example** (order-svc — Go requires more manual work):

```go
package main

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("order-svc")

func PlaceOrder(ctx context.Context, order Order) error {
    // Start a span for this business operation
    ctx, span := tracer.Start(ctx, "PlaceOrder",
        trace.WithAttributes(
            attribute.String("order.id", order.ID),
            attribute.Float64("order.total", order.Total),
            attribute.String("order.currency", order.Currency),
            // ⚠️ NEVER put PII (email, name, credit card) in span attributes
            // Traces are stored and searchable — treat like logs
        ),
    )
    defer span.End()
    
    // Validate inventory
    if err := checkInventory(ctx, order); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "inventory check failed")
        return err
    }
    
    // Process payment (this creates a child span)
    if err := processPayment(ctx, order); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "payment failed")
        return err
    }
    
    // Add event (like a log line within the span)
    span.AddEvent("order.confirmed", trace.WithAttributes(
        attribute.String("confirmation.id", "conf-xyz789"),
    ))
    
    span.SetStatus(codes.Ok, "")
    return nil
}

func processPayment(ctx context.Context, order Order) error {
    // ctx carries the trace context — this span becomes a child of PlaceOrder
    ctx, span := tracer.Start(ctx, "ProcessPayment",
        trace.WithSpanKind(trace.SpanKindClient), // This is an outgoing call
    )
    defer span.End()
    
    // ... payment logic ...
    // The HTTP client (instrumented by OTel) will automatically
    // propagate trace context to the downstream payment gateway
    
    return nil
}
```

**Key rule: ALWAYS pass `ctx` through the call chain.** The context carries the span. If you create a new `context.Background()` mid-chain, you break the trace.

```go
// WRONG — breaks the trace chain
func handleRequest(ctx context.Context) {
    go processAsync(context.Background())  // NEW context = new trace = broken
}

// CORRECT — preserves the trace chain
func handleRequest(ctx context.Context) {
    go processAsync(ctx)  // Same context = same trace = connected
}
```

---

## Sampling — You Can't Store Everything

### The Math

```
NovaMart: 50M monthly active users
         ~2,000 requests/second average, 10,000 peak
         Average trace: 12 spans
         Average span size: 500 bytes

Daily trace volume: 2,000 × 86,400 × 12 × 500 bytes
                  = ~1 TB/day of raw trace data

At $0.023/GB S3 storage: ~$23/day → $700/month just storage
Plus query costs, ingestion compute, etc.

With 10% sampling: ~100 GB/day → $70/month
                   Still statistically representative for aggregation
```

### Head Sampling vs Tail Sampling

```
┌─────────────────────────────────────────────────────────────────┐
│                    HEAD SAMPLING                                  │
│                                                                   │
│  Decision made AT THE START of the trace (before any spans)      │
│                                                                   │
│  ┌──────┐                                                        │
│  │ Dice │──▶ 10% → Trace sampled (all spans collected)          │
│  │ Roll │──▶ 90% → Trace NOT sampled (no spans collected)       │
│  └──────┘                                                        │
│                                                                   │
│  Pros: Simple, low resource usage, no state needed               │
│  Cons: Might miss all error/slow traces (they're in the 90%)    │
│                                                                   │
│  Implementation: SDK-side (sampler config)                       │
│    sampler:                                                      │
│      type: parentbased_traceidratio                              │
│      argument: "0.1"   # 10%                                    │
│                                                                   │
│  parentbased = if parent was sampled, child is sampled too       │
│  (prevents partial traces)                                       │
├─────────────────────────────────────────────────────────────────┤
│                    TAIL SAMPLING                                  │
│                                                                   │
│  Decision made AFTER the trace is complete (all spans available) │
│                                                                   │
│  ┌──────────┐                                                    │
│  │ Collect  │──▶ Analyze complete trace:                        │
│  │ ALL      │    - Has errors? → KEEP (100%)                    │
│  │ spans    │    - Duration > 2s? → KEEP (100%)                 │
│  │ first    │    - payment-svc? → KEEP (100%)                   │
│  │          │    - Normal trace? → KEEP (10%)                   │
│  └──────────┘                                                    │
│                                                                   │
│  Pros: Intelligent — keeps ALL interesting traces                │
│  Cons: MUST see all spans of a trace in one place               │
│        Higher memory (buffering traces until decision)           │
│        decision_wait latency before export                       │
│                                                                   │
│  Implementation: Collector-side (tail_sampling processor)        │
│  ⚠️ Requires Gateway pattern — one collector sees all spans     │
└─────────────────────────────────────────────────────────────────┘
```

**NovaMart strategy: Head sampling at 100% (keep all) + Tail sampling at Gateway:**

```
App SDK: sample 100% (send everything to agent)
Agent (DaemonSet): forward all to gateway
Gateway (Deployment): tail_sampling — keep errors, slow, critical; sample 10% of rest
Gateway → Tempo: only sampled traces stored
```

Why not head-sample at 10% in the SDK? Because then you lose 90% of error traces, and those are the ones you actually need for debugging.

---

## Tempo — The Trace Backend

### Why Tempo

```
┌──────────────┬──────────────────────┬──────────────────────┐
│              │ Jaeger               │ Tempo                │
├──────────────┼──────────────────────┼──────────────────────┤
│ Storage      │ Elasticsearch/       │ Object storage (S3)  │
│              │ Cassandra/Kafka      │ Only. Nothing else.  │
├──────────────┼──────────────────────┼──────────────────────┤
│ Index        │ Full index of all    │ NO index (Trace ID   │
│              │ span attributes      │ lookup only)         │
│              │                      │ + TraceQL for search │
├──────────────┼──────────────────────┼──────────────────────┤
│ Cost         │ HIGH (index storage) │ LOW (S3 only)        │
├──────────────┼──────────────────────┼──────────────────────┤
│ Operations   │ Heavy (ES cluster    │ Light (stateless +   │
│              │ management)          │ object storage)      │
├──────────────┼──────────────────────┼──────────────────────┤
│ Query model  │ Search by any attr   │ Trace ID lookup is   │
│              │ (fast)               │ fast. Search via     │
│              │                      │ TraceQL or metrics   │
│              │                      │ (service graph)      │
├──────────────┼──────────────────────┼──────────────────────┤
│ Integration  │ Standalone UI        │ Native Grafana       │
│              │                      │ integration          │
├──────────────┼──────────────────────┼──────────────────────┤
│ NovaMart     │ Legacy (migrating    │ Target state         │
│              │ away)                │                      │
└──────────────┴──────────────────────┴──────────────────────┘
```

**Tempo's philosophy:** Same as Loki — index almost nothing, store cheaply in object storage. Find traces via:
1. **Trace ID** (direct lookup — from logs or metrics exemplars)
2. **TraceQL** (query language for searching traces)
3. **Service graph** (generated from span metrics)

### Tempo Deployment

```yaml
# values-tempo.yaml (Helm)
tempo:
  storage:
    trace:
      backend: s3
      s3:
        bucket: novamart-tempo-traces
        endpoint: s3.us-east-1.amazonaws.com
        region: us-east-1
  
  # Metrics generator — creates metrics FROM traces
  # (RED metrics: Rate, Error, Duration — per service, per operation)
  metricsGenerator:
    enabled: true
    remoteWriteUrl: http://prometheus-operated.monitoring:9090/api/v1/write
    processor:
      service_graphs:
        enabled: true
        dimensions:
          - service.namespace
          - http.method
      span_metrics:
        enabled: true
        dimensions:
          - service.namespace
          - http.method
          - http.status_code
  
  retention: 15d
  
  overrides:
    defaults:
      search:
        duration_limit: 720h    # Allow search up to 30 days
      ingestion:
        rate_strategy: local
        rate_limit_bytes: 15000000  # 15MB/s per distributor
        burst_size_bytes: 20000000
        max_traces_per_user: 10000

# Tempo components (SimpleScalable mode — same as Loki)
distributor:
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 512Mi

ingester:
  replicas: 3
  resources:
    requests:
      cpu: "1"
      memory: 2Gi
  persistence:
    enabled: true
    size: 20Gi

querier:
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 1Gi

queryFrontend:
  replicas: 2

compactor:
  replicas: 1  # Singleton, like Thanos
```

### Metrics Generated FROM Traces (Tempo Metrics Generator)

This is powerful — Tempo automatically generates RED metrics from trace data:

```
traces_service_graph_request_total{client="order-svc", server="payment-svc", status="ok"}
traces_service_graph_request_failed_total{client="order-svc", server="payment-svc"}
traces_service_graph_request_server_seconds_bucket{client="order-svc", server="payment-svc", le="0.5"}

traces_spanmetrics_latency_bucket{service="payment-svc", span_name="ProcessPayment", le="0.1"}
traces_spanmetrics_calls_total{service="payment-svc", span_name="ProcessPayment", status_code="OK"}
```

**These flow into Prometheus** via remote write, so you get:
- Service dependency graphs in Grafana (who calls whom)
- Per-operation latency histograms (without manual instrumentation)
- Error rates per span operation

**This bridges the metrics↔traces gap without requiring separate metric instrumentation.**

### TraceQL — Querying Traces

```
# Find traces where payment-svc had errors
{ resource.service.name = "payment-svc" && status = error }

# Find slow database spans
{ span.db.system = "postgresql" && duration > 500ms }

# Find traces that touched both order-svc and payment-svc
{ resource.service.name = "order-svc" } && { resource.service.name = "payment-svc" }

# Find traces where HTTP status was 500
{ span.http.status_code = 500 }

# Find traces by specific attribute
{ span.order.id = "ord-12345" }

# Aggregate: average duration of payment spans
{ resource.service.name = "payment-svc" && name = "ProcessPayment" } | avg(duration)

# Find traces with specific error message
{ resource.service.name = "payment-svc" && status = error } >> { span.error.message =~ ".*timeout.*" }
```

---

## The Complete Correlation Model

### How Metrics, Logs, and Traces Connect

```
┌──────────────────────────────────────────────────────────────────┐
│                    EXEMPLARS (Metrics → Traces)                   │
│                                                                   │
│  Prometheus metric sample carries a trace ID as exemplar:        │
│                                                                   │
│  http_request_duration_seconds_bucket{le="1.0"} 1532             │
│  # Exemplar: {traceID="abc123"} 0.95 1704067200                 │
│                                                                   │
│  In Grafana: hover over a histogram → see exemplar dots →        │
│  click → jump to trace in Tempo                                  │
│                                                                   │
│  Requirement: app must emit exemplars (OTel SDK does this        │
│  automatically when histogram metrics are linked to active spans)│
│                                                                   │
│  Prometheus config:                                               │
│    --enable-feature=exemplar-storage                             │
│    (enabled by default in recent versions)                       │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    LOG → TRACE CORRELATION                        │
│                                                                   │
│  Application logs include traceID field:                         │
│  {"level":"error","msg":"payment failed","traceID":"abc123"}     │
│                                                                   │
│  Grafana Loki datasource config (derivedFields):                 │
│  - matcherRegex: '"traceID":"(\w+)"'                            │
│    datasourceUid: tempo                                          │
│                                                                   │
│  In Grafana: view log line → click traceID → jump to Tempo      │
│                                                                   │
│  Requirement: Application must inject traceID into log context   │
│  (OTel SDK + logging bridge does this automatically)             │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    TRACE → LOGS CORRELATION                       │
│                                                                   │
│  Tempo datasource config (tracesToLogs):                         │
│  - datasourceUid: loki                                           │
│    tags: ['service.name']                                        │
│    mappedTags: [{key: 'service.name', value: 'app'}]            │
│    filterByTraceID: true                                         │
│                                                                   │
│  In Grafana: view trace → click "Logs for this span" →          │
│  jump to Loki filtered by traceID and time range                │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    TRACE → METRICS CORRELATION                    │
│                                                                   │
│  Tempo datasource config (tracesToMetrics):                      │
│  - datasourceUid: prometheus                                     │
│    tags: [{key: 'service.name', value: 'service'}]              │
│                                                                   │
│  In Grafana: view trace → click "Metrics for this service" →    │
│  jump to Prometheus dashboard for that service + time range     │
└──────────────────────────────────────────────────────────────────┘
```

### The Complete Incident Investigation Flow

```
INCIDENT: order-svc p99 > 5s SLO breach at 14:32

STEP 1: METRICS (Grafana → Prometheus/Thanos)
┌──────────────────────────────────────────────────────────┐
│ Dashboard: Golden Signals - order-svc                     │
│                                                           │
│ PromQL: histogram_quantile(0.99,                         │
│   sum by (le)(rate(http_request_duration_seconds_bucket   │
│   {service="order-svc"}[$__rate_interval])))              │
│                                                           │
│ Result: p99 jumped from 200ms to 5.2s at 14:32           │
│ Notice: Exemplar dot at 5.1s with traceID=abc123         │
│                                                           │
│ Action: Click exemplar → opens trace in Tempo             │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
STEP 2: TRACE (Grafana → Tempo)
┌──────────────────────────────────────────────────────────┐
│ Trace ID: abc123                                          │
│                                                           │
│ ├── API Gateway (8ms)                                    │
│ ├── order-svc: PlaceOrder (5100ms)                       │
│ │   ├── Inventory check (42ms) ✅                        │
│ │   ├── payment-svc: ProcessPayment (4950ms) ← SLOW!    │
│ │   │   ├── Stripe API call (120ms) ✅                   │
│ │   │   └── fraud-svc: CheckFraud (4800ms) ← ROOT CAUSE │
│ │   │       └── ml-model-svc: Predict (4750ms) ← HERE   │
│ │   │           span.attributes:                         │
│ │   │             model.version = "v3.2.1"               │
│ │   │             model.cold_start = true ← SMOKING GUN  │
│ │   └── notification-svc (80ms) ✅                       │
│                                                           │
│ Action: Click "Logs for this span" on ml-model-svc       │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
STEP 3: LOGS (Grafana → Loki)
┌──────────────────────────────────────────────────────────┐
│ Auto-generated query:                                     │
│ {app="ml-model-svc"} |= "abc123"                        │
│                                                           │
│ Results:                                                  │
│ 14:32:01 INFO  Loading model v3.2.1 from S3...          │
│ 14:32:03 WARN  Model cache miss, downloading 2.1GB      │
│ 14:32:05 INFO  Model loaded, inference starting          │
│ 14:32:05 INFO  Prediction complete: fraud_score=0.02     │
│                                                           │
│ ROOT CAUSE: ML model cold start (cache eviction)         │
│ Model download from S3 took 4.7 seconds                  │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
STEP 4: FIX
┌──────────────────────────────────────────────────────────┐
│ Immediate: Pre-warm model cache, increase cache TTL      │
│ Long-term: Model sidecar pattern with startup probe      │
│            PodReadinessGate to prevent traffic before     │
│            model is loaded                               │
│                                                           │
│ Time to root cause: ~3 minutes                           │
│ (Without tracing: hours of guessing between teams)       │
└──────────────────────────────────────────────────────────┘
```

---

## Tracing Failure Modes

| Failure | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| Broken traces (disconnected spans) | Trace shows single span, no children | Context propagation broken — service doesn't extract/inject headers | Check propagator config, ensure all services use same format (W3C), verify HTTP client is instrumented |
| Missing spans | Trace has gaps (parent → grandchild, missing child) | Intermediate service not instrumented, or sampling dropped the span | Add instrumentation to missing service, check sampling decisions |
| Duplicate spans | Same operation appears twice | Both auto-instrumentation AND manual span for same operation | Remove manual span when auto-instrumentation covers it |
| Span explosion | Thousands of spans per trace, Tempo OOM | Loop calling instrumented function, N+1 query generating N spans | Use `otel.library.name` filter in Collector, batch DB calls, suppress inner spans |
| Traces not appearing in Tempo | Apps send spans but Tempo shows nothing | Collector exporter misconfigured, Tempo ingester rejecting, wrong OTLP endpoint | Check Collector exporter logs, verify Tempo distributor metrics, check `tempo_distributor_spans_received_total` |
| High latency from tracing | App latency increased after adding OTel | Synchronous exporter (blocking on send), too many attributes, no batching | Use async exporter (default in OTel SDK), enable batch processor, reduce attribute count |
| Trace ID not in logs | Can't correlate logs to traces | Logging bridge not configured, MDC/context not propagated | Configure OTel logging bridge (Java: MDC auto-injection, Python: LoggingInstrumentor) |
| Sampling confusion | "We never see error traces" | Head sampling at 10% drops 90% of errors | Switch to tail sampling at Collector, or use `parentbased_always_on` with tail sampling at gateway |
| Collector OOM | OTel Collector pods restarting | Tail sampling buffering too many traces, `decision_wait` too long | Increase memory, reduce `decision_wait`, reduce `num_traces`, add `memory_limiter` processor |
| Clock skew in traces | Span timelines overlap impossibly | Node clocks not synchronized | Ensure NTP is running on all nodes, use `chrony` not `ntpd`, verify with `chronyc tracking` |

---

## Quick Reference Card

```
DISTRIBUTED TRACING
────────────────────
Trace = full request journey across services
Span  = single operation (has ID, parent ID, duration, attributes, events)
Root Span = entry point (no parent)

CONTEXT PROPAGATION (the #1 failure point)
──────────────────────────────────────────
W3C TraceContext: traceparent header (standard — use this)
B3/Jaeger/X-Ray: legacy formats (support for migration)
Configure MULTIPLE propagators during migration
ALWAYS pass ctx through call chains (Go: context.Context)
New context.Background() = broken trace

OPENTELEMETRY
─────────────
SDK:        In-app instrumentation (auto or manual)
Collector:  Receive → Process → Export pipeline
OTLP:       Wire protocol (gRPC :4317, HTTP :4318)
Auto-inst:  OTel Operator + pod annotation = zero-code tracing
Manual:     tracer.Start(ctx, "name") for business logic spans

COLLECTOR DEPLOYMENT (NovaMart)
──────────────────────────────
Agent (DaemonSet): k8s enrichment, filtering, forwarding
Gateway (Deployment): tail sampling, export to backends
Agent+Gateway required for tail sampling (all spans in one place)

SAMPLING
────────
Head: Decision at trace start. Simple, but misses errors.
Tail: Decision after trace complete. Intelligent, but needs memory.
NovaMart: 100% head → tail at gateway (keep errors+slow, 10% rest)

TEMPO
─────
Storage: S3 only (like Loki — cheap)
Index: Minimal (trace ID lookup + TraceQL)
Metrics Generator: Creates RED metrics FROM traces → Prometheus
TraceQL: { resource.service.name = "x" && status = error }
vs Jaeger: Lower cost, less flexible search, better Grafana integration

CORRELATION
───────────
Metrics → Traces: Exemplars (histogram sample carries traceID)
Logs → Traces:    derivedFields regex on traceID in log lines
Traces → Logs:    tracesToLogs config (filter Loki by traceID + time)
Traces → Metrics: tracesToMetrics config (jump to service dashboard)
All three: shared labels (service.name ↔ app label)

FAILURE MODES TOP 5
───────────────────
1. Broken traces:     propagation format mismatch
2. Missing spans:     service not instrumented
3. No error traces:   head sampling dropping them → use tail sampling
4. Collector OOM:     tail sampling buffer too large
5. TraceID not in logs: logging bridge not configured
```
---

## Retention Questions — Phase 5 Lesson 3

### Q1: Broken Traces Investigation 🔥

**Scenario:** NovaMart just onboarded `fraud-svc` (Python) to distributed tracing. The team instrumented it with the OTel Python SDK and auto-instrumentation. They report that traces show `order-svc` → `payment-svc` → ??? → `ml-model-svc`, but the `fraud-svc` spans are **completely missing** from all traces. `fraud-svc` IS generating spans (you can see them in the Collector's debug exporter), but they show up as **independent root spans** — not connected to the parent trace.

Meanwhile, all other services' traces are correctly connected.

**Questions:**
1. What is the most likely root cause? Be specific about the exact mechanism.
2. What are the **three things** you would check, in order, to diagnose this? Give exact commands or configs for each.
3. Write the fix — show the exact configuration change needed.
4. How would you verify the fix is working without waiting for a real user request?

---

### Q2: Sampling Decision Gone Wrong 🔥

**Scenario:** NovaMart configured tail sampling at the OTel Gateway with these policies:

```yaml
tail_sampling:
  decision_wait: 10s
  num_traces: 100000
  policies:
    - name: errors-always
      type: status_code
      status_code:
        status_codes: [ERROR]
    - name: slow-traces
      type: latency
      latency:
        threshold_ms: 2000
    - name: keep-10-percent
      type: probabilistic
      probabilistic:
        sampling_percentage: 10
```

**Three problems are reported:**

**Problem A:** The on-call engineer says "I can see the error in our logs with traceID `xyz789`, but when I search for it in Tempo, the trace doesn't exist."

**Problem B:** The Collector Gateway pods are getting OOM-killed during peak traffic (10,000 req/s).

**Problem C:** A developer complains: "My service has 100% error rate right now, but I only see a few error traces in Tempo."

**For each problem, explain the root cause and provide the fix.**

---

### Q3: OTel Collector Pipeline Design 🔥

**Scenario:** NovaMart's platform team receives these requirements:

1. All microservices send traces, metrics, and logs via OTLP to the Collector
2. Health check endpoints (`/healthz`, `/readyz`) must NOT generate trace spans
3. All telemetry must be enriched with Kubernetes metadata (namespace, pod, node, deployment)
4. Traces from `payment-svc` and `fraud-svc` must ALWAYS be kept (100%)
5. All other traces: keep errors and slow (>1s), sample 5% of the rest
6. Metrics must flow to Prometheus, traces to Tempo, logs to Loki
7. The Collector must not OOM under burst traffic (limit to 2GB memory)
8. Legacy services still send Jaeger format (thrift over HTTP) — must accept both

**Write the complete OTel Collector configuration** that satisfies ALL eight requirements. Use the Agent + Gateway pattern. Show both configs.

---

### Q4: End-to-End Correlation Under Pressure 🔥

**Scenario:** 2 AM. PagerDuty fires: `PaymentSvcP99LatencyHigh` — payment-svc p99 is 8 seconds (SLO: 2 seconds).

**Walk through your exact investigation steps using the three pillars.** For each step:
- State which tool you use (Grafana → Prometheus/Loki/Tempo)
- Write the **exact query** (PromQL, LogQL, or TraceQL)
- State what you expect to find and what your next action is
- When you pivot from one pillar to another, explain **exactly how** (which link you click, which ID you carry over)

**The twist:** You discover that the trace shows `payment-svc` → `stripe-api` taking 7.5 seconds, but the Stripe status page shows no issues. What do you investigate next, and what PromQL query would reveal the actual bottleneck?

---

# Phase 5, Lesson 3 — Retention Answers

---

## Q1: Broken Traces Investigation

### 1. Root Cause: Trace Context Propagation Mismatch

The spans exist. The Collector receives them. They appear as **independent root spans** — not missing, but **disconnected**. This means one thing: **`fraud-svc` is not reading the incoming trace context from the parent request, so it starts a new trace instead of joining the existing one.**

The exact mechanism:

```
order-svc (Go)                payment-svc (Java)           fraud-svc (Python)          ml-model-svc (Go)
     │                              │                            │                           │
     │──HTTP──→ traceparent: 00-TRACEID-SPANID-01               │                           │
     │          b3: TRACEID-SPANID-1                             │                           │
     │                              │──gRPC──→  ???              │                           │
     │                              │           fraud-svc does   │                           │
     │                              │           NOT extract      │                           │
     │                              │           traceparent      │                           │
     │                              │           from incoming    │                           │
     │                              │           request headers  │                           │
     │                              │                            │──HTTP──→ traceparent: 00-NEW_TRACEID-...
     │                              │                            │          (new trace!)
```

**Three specific causes in order of probability:**

**Cause A (most likely): Propagator mismatch.** The upstream services (`payment-svc`, Java) are sending context using W3C `traceparent` headers. The Python `fraud-svc` is configured (or defaulting) to only read **B3 propagation format** (Zipkin-style) — or vice versa. The OTel Python SDK's default propagator is `tracecontext,baggage` (W3C), but if someone manually configured it to use only `b3` or `b3multi`, it won't read `traceparent` headers.

**Cause B: The call between payment-svc and fraud-svc uses a transport that auto-instrumentation doesn't cover.** If `payment-svc` calls `fraud-svc` via a message queue (SQS, Kafka, RabbitMQ) rather than HTTP/gRPC, the auto-instrumentation won't automatically inject/extract trace context from message attributes. The context is lost at the queue boundary.

**Cause C: Framework not instrumented.** `fraud-svc` might use a Python web framework that OTel auto-instrumentation doesn't cover (e.g., a raw `asyncio` TCP server, or a custom framework). The auto-instrumentation hooks into known frameworks (`flask`, `django`, `fastapi`, `grpc`). If the entry point isn't instrumented, the incoming context is never extracted.

### 2. Three Diagnostic Checks (In Order)

**Check 1 — Verify what headers `payment-svc` sends to `fraud-svc`:**

```bash
# Exec into fraud-svc pod and inspect incoming request headers
# Temporarily add debug logging to see headers, or use tcpdump:
kubectl -n payments exec -it deploy/fraud-svc -- python3 -c "
import os
# Check what propagators are configured
print('OTEL_PROPAGATORS:', os.environ.get('OTEL_PROPAGATORS', 'NOT SET (defaults to tracecontext,baggage)'))
print('OTEL_TRACES_EXPORTER:', os.environ.get('OTEL_TRACES_EXPORTER', 'NOT SET'))
"

# Check the actual HTTP headers arriving at fraud-svc
# Use ephemeral debug container or tcpdump:
kubectl -n payments debug -it deploy/fraud-svc --image=nicolaka/netshoot -- \
  tcpdump -A -s 0 -i any port 8080 -c 20 2>/dev/null | grep -iE "traceparent|b3|tracestate"
```

**What you expect:** If you see `traceparent: 00-<traceid>-<spanid>-01` in incoming headers but `fraud-svc` spans have a **different** traceID → propagator extraction is failing.

If you see **no trace headers at all** in the incoming request → the upstream (`payment-svc`) isn't injecting context for this particular call path (maybe it's a message queue or a non-instrumented HTTP client).

**Check 2 — Verify fraud-svc's OTel SDK propagator configuration:**

```bash
# Check the deployment's environment variables
kubectl -n payments get deploy fraud-svc -o yaml | grep -A2 -E "OTEL_|PROPAGATOR|TRACE"

# Expected correct config:
# - name: OTEL_PROPAGATORS
#   value: "tracecontext,baggage"
# If it says "b3" or "b3multi" and upstream sends W3C → MISMATCH

# Check if auto-instrumentation is configured via OTel Operator injection:
kubectl -n payments get deploy fraud-svc -o yaml | grep -A5 "instrumentation.opentelemetry.io"
# Look for the annotation:
#   instrumentation.opentelemetry.io/inject-python: "true"

# Check the Instrumentation CR if using OTel Operator:
kubectl -n payments get instrumentation -o yaml
# Look for the propagators field:
#   spec:
#     propagators:
#       - tracecontext
#       - baggage
```

**Check 3 — Compare traceIDs between connected services and fraud-svc:**

```bash
# In Grafana → Explore → Tempo
# Search for a known trace from order-svc → payment-svc
# Note the traceID: e.g., abc123def456

# Now search fraud-svc spans in the same time window:
# TraceQL:
{ resource.service.name = "fraud-svc" } | status = ok

# Look at any fraud-svc span — check its traceID
# If it's a DIFFERENT traceID than abc123def456 → context not propagated
# If it has no parentSpanId → it's a root span (started new trace)

# Cross-reference with Collector debug output:
kubectl -n monitoring logs deploy/otel-collector -c collector | grep "fraud-svc" | head -5
# Look at the span's traceID and parentSpanID fields
```

### 3. The Fix

**If Cause A (propagator mismatch) — which is most likely:**

```yaml
# Option 1: Fix via environment variables on the fraud-svc Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fraud-svc
  namespace: payments
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-python: "true"
    spec:
      containers:
        - name: fraud-svc
          env:
            # Explicitly set propagators to match upstream services
            - name: OTEL_PROPAGATORS
              value: "tracecontext,baggage"
            # Set the service name correctly
            - name: OTEL_SERVICE_NAME
              value: "fraud-svc"
            # Point to the Collector
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.monitoring:4317"
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: "grpc"
```

```bash
kubectl apply -f fraud-svc-deployment.yaml
kubectl -n payments rollout status deploy/fraud-svc --timeout=120s
```

**If the upstream payment-svc sends both W3C AND B3 (common in mixed environments), configure fraud-svc to accept both:**

```yaml
            - name: OTEL_PROPAGATORS
              value: "tracecontext,baggage,b3multi"
```

**If using the OTel Operator Instrumentation CR:**

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: python-instrumentation
  namespace: payments
spec:
  propagators:
    - tracecontext
    - baggage
    - b3multi
  python:
    env:
      - name: OTEL_PYTHON_EXCLUDED_URLS
        value: "healthz,readyz"
  exporter:
    endpoint: "http://otel-collector.monitoring:4317"
```

```bash
kubectl apply -f instrumentation.yaml
# Restart fraud-svc to pick up injection
kubectl -n payments rollout restart deploy/fraud-svc
kubectl -n payments rollout status deploy/fraud-svc --timeout=120s
```

**If Cause B (message queue transport):** Add explicit context injection/extraction in code:

```python
# In payment-svc (producer side) — inject context into message attributes
from opentelemetry import context, propagate

def call_fraud_check(order_data):
    carrier = {}
    propagate.inject(carrier)  # Injects traceparent into carrier dict

    sqs_client.send_message(
        QueueUrl=FRAUD_CHECK_QUEUE,
        MessageBody=json.dumps(order_data),
        MessageAttributes={
            'traceparent': {
                'DataType': 'String',
                'StringValue': carrier.get('traceparent', '')
            },
            'tracestate': {
                'DataType': 'String',
                'StringValue': carrier.get('tracestate', '')
            }
        }
    )
```

```python
# In fraud-svc (consumer side) — extract context from message attributes
from opentelemetry import context, propagate, trace

def process_message(message):
    carrier = {
        'traceparent': message['MessageAttributes'].get('traceparent', {}).get('StringValue', ''),
        'tracestate': message['MessageAttributes'].get('tracestate', {}).get('StringValue', ''),
    }
    ctx = propagate.extract(carrier)

    tracer = trace.get_tracer("fraud-svc")
    with tracer.start_as_current_span("process_fraud_check", context=ctx):
        # Now this span is correctly parented to the upstream trace
        do_fraud_check(message)
```

### 4. Verification Without Waiting for Real Traffic

**Generate a synthetic trace that crosses the full chain:**

```bash
# Use otel-cli to send a test trace through the full path
# Install: https://github.com/equinixmetal/otel-cli

# Step 1: Start a parent span simulating order-svc
TRACE_ID=$(openssl rand -hex 16)
PARENT_SPAN_ID=$(openssl rand -hex 8)

otel-cli exec \
  --service "order-svc" \
  --name "test-trace-validation" \
  --endpoint "http://otel-collector.monitoring:4317" \
  --tp-print \
  --force-trace-id "$TRACE_ID" \
  --force-span-id "$PARENT_SPAN_ID" \
  -- echo "parent span created"
# Output: traceparent: 00-$TRACE_ID-$PARENT_SPAN_ID-01

# Step 2: Call fraud-svc's endpoint with that traceparent header
TRACEPARENT="00-${TRACE_ID}-${PARENT_SPAN_ID}-01"

kubectl -n payments run trace-test --rm -it --image=curlimages/curl -- \
  curl -v -H "traceparent: ${TRACEPARENT}" \
  http://fraud-svc.payments:8080/check \
  -d '{"order_id":"test-123","amount":99.99}'

# Step 3: Wait 10s for Collector processing + Tempo ingestion, then query
sleep 15

# Step 4: Query Tempo for the trace
curl -s "http://tempo.monitoring:3200/api/traces/${TRACE_ID}" | jq '.batches[].resource.attributes[] | select(.key=="service.name") | .value.stringValue'
# Expected output should include BOTH "order-svc" AND "fraud-svc"
# If fraud-svc is missing → fix didn't work
# If fraud-svc is present → context propagation is working
```

**In Grafana:**

```
Explore → Tempo → TraceQL:
{ trace:id = "<TRACE_ID>" }
```

You should see a waterfall with `order-svc` as parent and `fraud-svc` as a child span. If `fraud-svc` appears as a **separate root trace**, the propagation is still broken.

---

## Q2: Sampling Decision Gone Wrong

### Problem A: Error Trace Exists in Logs but Not in Tempo

**Root Cause: The trace with ID `xyz789` never reached the Gateway Collector where tail sampling happens.**

Tail sampling requires **ALL spans of a trace to arrive at the SAME Collector instance** before the `decision_wait` window expires. Here's what happens:

```
                     fraud-svc pod-1 ──→ Agent-1 ──→ Gateway-1 (has spans A, B)
user request ──→                                                    │
                     payment-svc pod-3 ──→ Agent-2 ──→ Gateway-2 (has spans C, D)
                                                            │
                                    These are the SAME trace but split
                                    across two Gateway instances!
                                    
Gateway-1 sees: 2 spans, no ERROR status → probabilistic 10% → DROPPED
Gateway-2 sees: 2 spans, one with ERROR → keeps it → partial trace
```

**The fundamental issue:** Without a **load balancing exporter** at the Agent layer that routes by traceID, spans from the same trace land on different Gateway pods. Each Gateway makes an independent sampling decision with incomplete information. Gateway-1 doesn't see the error span, so it drops its portion.

**The fix — use the `loadbalancing` exporter in the Agent config:**

```yaml
# Agent config — routes all spans from the same trace to the same Gateway
exporters:
  loadbalancing:
    routing_key: "traceID"
    protocol:
      otlp:
        timeout: 5s
        tls:
          insecure: true
    resolver:
      dns:
        hostname: otel-gateway-headless.monitoring.svc.cluster.local
        port: 4317
```

**This ensures all spans with the same traceID hit the same Gateway instance**, so the tail sampler has the complete picture.

**Additional issue:** `decision_wait: 10s` may be too short. If `fraud-svc` or `ml-model-svc` takes longer than 10 seconds, their spans arrive after the Gateway already made the sampling decision and evicted the trace from memory. The late-arriving spans are **orphaned** — they have no sampling decision, and the default behavior is to drop them.

```yaml
tail_sampling:
  decision_wait: 30s  # Increase to accommodate slow services
  # But be aware: longer wait = more memory usage
```

### Problem B: Gateway Pods OOM During Peak Traffic

**Root Cause: Tail sampling buffers ALL incoming spans in memory for `decision_wait` seconds before deciding.**

The math at 10,000 req/s:

```
10,000 req/s × ~5 spans/trace × 10s decision_wait = 500,000 spans in memory
Each span ≈ 1-5 KB → 500MB - 2.5GB in the tail sampler buffer alone
Plus: num_traces: 100000 trace ID index overhead
Plus: normal Collector pipeline buffers
Total: easily >2GB → OOM
```

**Fix — multi-layered:**

```yaml
# 1. Reduce what reaches the Gateway by head-sampling at the Agent level
# Agent config:
processors:
  probabilistic_sampler:
    sampling_percentage: 50  # Drop 50% at the edge — BEFORE Gateway
    # This is crude but halves Gateway memory pressure

# 2. On the Gateway, add memory limiter
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 1800      # Hard limit — starts dropping at this point
    spike_limit_mib: 400  # Allow 400MB bursts above soft limit
    # Soft limit = limit_mib - spike_limit_mib = 1400 MiB
    # At 1400 MiB, Collector starts refusing data
    # At 1800 MiB, Collector force-drops data

# 3. Reduce num_traces to match actual memory budget
tail_sampling:
  decision_wait: 15s     # Reduced from 30s to save memory
  num_traces: 50000      # Down from 100000
  # 50,000 traces × ~5 spans × ~3KB = ~750MB — fits in budget
```

**But the real fix is architecture:** Don't do ALL sampling at the Gateway. Use a two-tier approach:

```
Agent (head sampling)              Gateway (tail sampling)
─────────────────────              ──────────────────────
- Always pass errors (check        - Final tail sampling decision
  span status before sending)      - Has complete trace context
- Always pass payment-svc          - Much less data to buffer
- Probabilistic 50% for rest       - Fits in 2GB memory
```

### Problem C: 100% Error Rate But Few Error Traces in Tempo

**Root Cause: Policy evaluation order. Tail sampling policies are evaluated as a logical OR — the FIRST matching policy's decision is used.**

Look at the policy order:

```yaml
policies:
  - name: errors-always        # Policy 1: keep if any span has ERROR
    type: status_code
    status_code:
      status_codes: [ERROR]
  - name: slow-traces          # Policy 2: keep if latency > 2s
    type: latency
    latency:
      threshold_ms: 2000
  - name: keep-10-percent      # Policy 3: probabilistic 10%
    type: probabilistic
    probabilistic:
      sampling_percentage: 10
```

**Wait — that's actually wrong.** In the OTel Collector's tail sampling processor, policies are evaluated as **OR by default** — if ANY policy says "sample," the trace is kept. So errors-always SHOULD keep all error traces.

**The actual root cause for Problem C is the same as Problem A:** spans from error traces are split across Gateway instances. Gateway-1 gets the non-error spans, Gateway-2 gets the error span.

- Gateway-1: sees no ERROR status → `errors-always` doesn't trigger → `keep-10-percent` kicks in → 90% chance of DROP
- Gateway-2: sees ERROR → keeps its spans → but only a **partial trace** stored in Tempo

When the developer searches Tempo, they find some error traces, but many are incomplete (missing spans) or entirely absent (the Gateway that had the error span happened to be the minority of spans).

**The fix is the same as Problem A: `loadbalancing` exporter at the Agent layer.**

**Additionally, to guarantee payment-svc and fraud-svc errors are always captured regardless of routing, add a composite policy:**

```yaml
tail_sampling:
  decision_wait: 15s
  num_traces: 50000
  policies:
    - name: always-keep-critical-services
      type: and
      and:
        and_sub_policy:
          - name: critical-service-filter
            type: string_attribute
            string_attribute:
              key: service.name
              values: ["payment-svc", "fraud-svc"]
          - name: always-sample
            type: always_sample

    - name: errors-always
      type: status_code
      status_code:
        status_codes: [ERROR]

    - name: slow-traces
      type: latency
      latency:
        threshold_ms: 2000

    - name: baseline-sampling
      type: probabilistic
      probabilistic:
        sampling_percentage: 5
```

---

## Q3: OTel Collector Pipeline Design

### Architecture Overview

```
                                ┌──────────────────────────────────────┐
  ┌──────────┐   OTLP/gRPC     │         OTel Agent (DaemonSet)       │
  │ app pods │ ──────────────→  │  receivers → processors → exporters  │
  └──────────┘                  │            (per node)                │
  ┌──────────┐   Jaeger/thrift  │                                      │
  │ legacy   │ ──────────────→  │  Accepts OTLP + Jaeger              │
  └──────────┘                  └──────────────┬───────────────────────┘
                                               │ loadbalancing exporter
                                               │ (route by traceID)
                                               ▼
                                ┌──────────────────────────────────────┐
                                │       OTel Gateway (Deployment)      │
                                │  receivers → processors → exporters  │
                                │    (tail sampling, enrichment)       │
                                └───┬──────────┬──────────┬────────────┘
                                    │          │          │
                                    ▼          ▼          ▼
                                 Tempo    Prometheus    Loki
                                (traces)  (metrics)    (logs)
```

### Agent Config (DaemonSet — runs on every node)

```yaml
# otel-agent-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  # Requirement 8: Accept Jaeger format from legacy services
  jaeger:
    protocols:
      thrift_http:
        endpoint: 0.0.0.0:14268

processors:
  # Requirement 7: Memory safety — first processor in every pipeline
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

  # Requirement 3: Enrich with Kubernetes metadata
  k8sattributes:
    auth_type: "serviceAccount"
    passthrough: false
    extract:
      metadata:
        - k8s.namespace.name
        - k8s.pod.name
        - k8s.pod.uid
        - k8s.node.name
        - k8s.deployment.name
      labels:
        - tag_name: app
          key: app
          from: pod
    pod_association:
      - sources:
          - from: resource_attribute
            name: k8s.pod.ip
      - sources:
          - from: connection

  # Requirement 2: Drop health check spans
  filter/healthcheck:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.target"] == "/healthz"'
        - 'attributes["http.target"] == "/readyz"'
        - 'attributes["url.path"] == "/healthz"'
        - 'attributes["url.path"] == "/readyz"'

  # Batch for efficiency
  batch:
    timeout: 5s
    send_batch_size: 8192
    send_batch_max_size: 16384

  # Add resource detection
  resourcedetection:
    detectors: [env, eks]
    timeout: 5s
    override: false

exporters:
  # Traces → Gateway via load balancing (route by traceID)
  loadbalancing:
    routing_key: "traceID"
    protocol:
      otlp:
        timeout: 10s
        tls:
          insecure: true
    resolver:
      dns:
        hostname: otel-gateway-headless.monitoring.svc.cluster.local
        port: 4317

  # Metrics → Gateway via standard OTLP
  otlp/metrics:
    endpoint: otel-gateway.monitoring.svc.cluster.local:4317
    tls:
      insecure: true

  # Logs → Gateway via standard OTLP
  otlp/logs:
    endpoint: otel-gateway.monitoring.svc.cluster.local:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp, jaeger]
      processors: [memory_limiter, resourcedetection, k8sattributes, filter/healthcheck, batch]
      exporters: [loadbalancing]

    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, k8sattributes, batch]
      exporters: [otlp/metrics]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, k8sattributes, batch]
      exporters: [otlp/logs]

  telemetry:
    logs:
      level: info
    metrics:
      address: 0.0.0.0:8888
```

### Gateway Config (Deployment — 3+ replicas)

```yaml
# otel-gateway-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Requirement 7: Memory safety — FIRST processor
  memory_limiter:
    check_interval: 1s
    limit_mib: 1800
    spike_limit_mib: 400

  # Requirement 4 & 5: Tail sampling for traces
  tail_sampling:
    decision_wait: 15s
    num_traces: 50000
    expected_new_traces_per_sec: 5000
    policies:
      # Requirement 4: payment-svc and fraud-svc always kept at 100%
      - name: critical-services-always
        type: and
        and:
          and_sub_policy:
            - name: critical-svc-filter
              type: string_attribute
              string_attribute:
                key: service.name
                values:
                  - payment-svc
                  - fraud-svc
            - name: keep-all
              type: always_sample

      # Requirement 5: All errors always kept
      - name: errors-always
        type: status_code
        status_code:
          status_codes: [ERROR]

      # Requirement 5: Slow traces (>1s) always kept
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 1000

      # Requirement 5: 5% baseline for everything else
      - name: baseline-5-percent
        type: probabilistic
        probabilistic:
          sampling_percentage: 5

  # Batch before export
  batch/traces:
    timeout: 5s
    send_batch_size: 4096

  batch/metrics:
    timeout: 10s
    send_batch_size: 8192

  batch/logs:
    timeout: 5s
    send_batch_size: 4096

exporters:
  # Requirement 6: Traces → Tempo
  otlp/tempo:
    endpoint: tempo-distributor.monitoring.svc.cluster.local:4317
    tls:
      insecure: true
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s

  # Requirement 6: Metrics → Prometheus (via remote write)
  prometheusremotewrite:
    endpoint: http://prometheus.monitoring.svc.cluster.local:9090/api/v1/write
    tls:
      insecure: true
    resource_to_telemetry_conversion:
      enabled: true
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s

  # Requirement 6: Logs → Loki
  loki:
    endpoint: http://loki-distributor.monitoring.svc.cluster.local:3100/loki/api/v1/push
    default_labels_enabled:
      exporter: false
      job: true
    labels:
      attributes:
        level: ""
        service.name: "service_name"
        k8s.namespace.name: "namespace"
    retry_on_failure:
      enabled: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, batch/traces]
      exporters: [otlp/tempo]

    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch/metrics]
      exporters: [prometheusremotewrite]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch/logs]
      exporters: [loki]

  telemetry:
    logs:
      level: info
    metrics:
      address: 0.0.0.0:8888
```

### Kubernetes Manifests

**Agent DaemonSet:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-agent
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: otel-agent
  template:
    metadata:
      labels:
        app: otel-agent
    spec:
      serviceAccountName: otel-agent
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:0.96.0
          args: ["--config=/conf/otel-agent-config.yaml"]
          ports:
            - containerPort: 4317    # OTLP gRPC
            - containerPort: 4318    # OTLP HTTP
            - containerPort: 14268   # Jaeger thrift HTTP
            - containerPort: 8888    # Self metrics
          resources:
            requests:
              memory: 256Mi
              cpu: 200m
            limits:
              memory: 512Mi
              cpu: 500m
          env:
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: KUBE_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - name: config
              mountPath: /conf
      volumes:
        - name: config
          configMap:
            name: otel-agent-config
```

**Gateway Deployment + Headless Service (required for loadbalancing exporter DNS resolution):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: otel-gateway-headless
  namespace: monitoring
spec:
  clusterIP: None  # Headless — required for DNS-based load balancing
  selector:
    app: otel-gateway
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
---
apiVersion: v1
kind: Service
metadata:
  name: otel-gateway
  namespace: monitoring
spec:
  selector:
    app: otel-gateway
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-gateway
  namespace: monitoring
spec:
  replicas: 3
  selector:
    matchLabels:
      app: otel-gateway
  template:
    metadata:
      labels:
        app: otel-gateway
    spec:
      serviceAccountName: otel-gateway
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:0.96.0
          args: ["--config=/conf/otel-gateway-config.yaml"]
          ports:
            - containerPort: 4317
            - containerPort: 4318
            - containerPort: 8888
          resources:
            requests:
              memory: 1Gi
              cpu: "1"
            limits:
              memory: 2Gi    # Requirement 7: 2GB max
              cpu: "2"
          volumeMounts:
            - name: config
              mountPath: /conf
      volumes:
        - name: config
          configMap:
            name: otel-gateway-config
```

**RBAC for k8sattributes processor (Agent needs to read pod metadata):**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-agent
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-agent
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces", "nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-agent
subjects:
  - kind: ServiceAccount
    name: otel-agent
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: otel-agent
  apiGroup: rbac.authorization.k8s.io
```

### Verification

```bash
# 1. Verify Agent is running on all nodes
kubectl -n monitoring get pods -l app=otel-agent -o wide
# Should show one pod per node

# 2. Verify Gateway replicas
kubectl -n monitoring get pods -l app=otel-gateway
# Should show 3 pods

# 3. Verify headless service resolves to all Gateway pods
kubectl -n monitoring run dns-test --rm -it --image=busybox -- \
  nslookup otel-gateway-headless.monitoring.svc.cluster.local
# Should list all 3 pod IPs

# 4. Verify Jaeger receiver works (legacy services)
kubectl -n monitoring port-forward ds/otel-agent 14268:14268 &
curl -X POST http://localhost:14268/api/traces \
  -H "Content-Type: application/x-thrift" \
  -d ''
# Should get 200 or 415 (not connection refused)

# 5. Check Collector self-metrics
kubectl -n monitoring port-forward ds/otel-agent 8888:8888 &
curl -s http://localhost:8888/metrics | grep otelcol_receiver_accepted_spans
# Should be increasing

# 6. Verify health check spans are being dropped
curl -s http://localhost:8888/metrics | grep otelcol_processor_filter_spans_filtered
# Should be > 0
```

---

## Q4: End-to-End Correlation Under Pressure

### Step 1: Confirm the Alert — Metrics (Prometheus via Grafana)

**Tool:** Grafana → Explore → Prometheus data source

```promql
histogram_quantile(0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{service="payment-svc"}[5m])
  )
)
```

**Expected:** Value around 8s, confirming the alert is real and not a flapping threshold.

**Follow-up — is it all endpoints or specific?**

```promql
histogram_quantile(0.99,
  sum by (le, endpoint) (
    rate(http_request_duration_seconds_bucket{service="payment-svc"}[5m])
  )
)
```

**Expected:** One endpoint (likely `/charge`) showing 8s while others are normal. This narrows the blast radius.

**Check if it's one pod or all pods:**

```promql
histogram_quantile(0.99,
  sum by (le, pod) (
    rate(http_request_duration_seconds_bucket{service="payment-svc", endpoint="/charge"}[5m])
  )
)
```

**Expected:** All pods showing similar latency → not a single-pod issue, it's systemic.

**When did it start?**

```promql
histogram_quantile(0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{service="payment-svc", endpoint="/charge"}[5m])
  )
)
```

Set Grafana time range to last 6 hours, look for the inflection point. Note the exact timestamp.

### Step 2: Pivot to Logs — Find Error Context

**Pivot mechanism:** From the Grafana metrics panel, I know the affected service (`payment-svc`), endpoint (`/charge`), and time window. I switch to Loki data source in Explore.

**Tool:** Grafana → Explore → Loki data source

```logql
{app="payment-svc"}
  | json
  | level="error"
  | endpoint="/charge"
  | line_format "{{.timestamp}} {{.msg}} duration={{.duration}} traceID={{.traceID}}"
```

**Expected:** Error messages with high durations. Looking for patterns — timeout messages, connection errors, upstream failures.

**Find the slowest requests:**

```logql
{app="payment-svc"}
  | json
  | endpoint="/charge"
  | duration > 5s
  | line_format "{{.timestamp}} traceID={{.traceID}} duration={{.duration}} msg={{.msg}}"
```

**Action:** Copy a `traceID` from one of the slow requests for the next step.

### Step 3: Pivot to Traces — See the Full Request Path

**Pivot mechanism:** From the log line, I have a specific `traceID`. If Grafana has trace-to-logs correlation configured, I click the `traceID` link directly. Otherwise, I switch to Tempo data source.

**Tool:** Grafana → Explore → Tempo data source

```
TraceQL: { trace:id = "<traceID-from-logs>" }
```

Or simply paste the traceID into Tempo's search field.

**Expected:** A waterfall view showing:

```
order-svc         ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  (8.2s total)
  payment-svc     ░░████████████████████████████████  (8.0s)
    fraud-svc     ░░████░░░░░░░░░░░░░░░░░░░░░░░░░░░  (0.3s — fast)
    stripe-api    ░░░░░░██████████████████████████░░  (7.5s — THE BOTTLENECK)
```

**Action:** The trace reveals `payment-svc` → `stripe-api` external call taking 7.5 seconds out of the 8-second total.

### Step 4: Investigate the Stripe Bottleneck

**Stripe status page shows no issues.** This means the problem is likely:

1. **Network path between NovaMart and Stripe** — not Stripe itself
2. **Connection pool exhaustion** in payment-svc's HTTP client
3. **DNS resolution delays** for Stripe's API endpoint
4. **TLS handshake overhead** if connections aren't being reused
5. **NAT Gateway throughput limit** if payment-svc routes through a NAT GW

**Back to Metrics — targeted queries:**

```promql
# Check if it's ALL Stripe calls or just bursts
histogram_quantile(0.99,
  sum by (le) (
    rate(http_client_request_duration_seconds_bucket{
      service="payment-svc",
      http_url=~".*stripe.*"
    }[5m])
  )
)
```

**Connection pool exhaustion — the most likely culprit at NovaMart's scale:**

```promql
# Active connections to Stripe — are we maxing out the pool?
http_client_active_connections{service="payment-svc", remote="api.stripe.com"}

# Or if using Go's net/http pool metrics:
go_connections_active{service="payment-svc"}

# Connection wait time — how long are requests queuing for a connection?
histogram_quantile(0.99,
  sum by (le) (
    rate(http_client_connection_wait_seconds_bucket{
      service="payment-svc",
      http_url=~".*stripe.*"
    }[5m])
  )
)
```

If `connection_wait` is high → connection pool is exhausted → requests queue → latency spikes.

**DNS resolution:**

```promql
# DNS lookup duration for Stripe's API
histogram_quantile(0.99,
  sum by (le) (
    rate(dns_lookup_duration_seconds_bucket{service="payment-svc", host="api.stripe.com"}[5m])
  )
)
```

**NAT Gateway — check from AWS:**

```bash
# Check NAT Gateway metrics — CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name PacketsDropCount \
  --dimensions Name=NatGatewayId,Value=nat-0abc123def456 \
  --start-time "$(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --period 300 \
  --statistics Sum

# Check ErrorPortAllocation — NAT GW ran out of ports
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name ErrorPortAllocation \
  --dimensions Name=NatGatewayId,Value=nat-0abc123def456 \
  --start-time "$(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --period 300 \
  --statistics Sum

# Check ConnectionAttemptCount and ActiveConnectionCount
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name ActiveConnectionCount \
  --dimensions Name=NatGatewayId,Value=nat-0abc123def456 \
  --start-time "$(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --period 300 \
  --statistics Maximum
```

**If `ErrorPortAllocation > 0`:** The NAT Gateway has exhausted its 55,000 concurrent connections per destination. This is a **hard AWS limit** per NAT GW. At NovaMart's scale with thousands of requests to `api.stripe.com:443`, each connection occupies a source port. If connections aren't being reused (no keep-alive, or connection pool misconfigured), you burn through ports.

**Fix for connection pool exhaustion (the most likely root cause):**

```yaml
# payment-svc configuration — increase HTTP client pool
# If Go service:
# http.Client{
#   Transport: &http.Transport{
#     MaxIdleConns:        200,
#     MaxIdleConnsPerHost: 50,     # Default is 2! This is usually the culprit
#     IdleConnTimeout:     90 * time.Second,
#   },
# }

# If Java/Spring:
# spring.codec.max-in-memory-size=10MB
# In WebClient or RestTemplate config:
# ConnectionProvider.builder("stripe")
#   .maxConnections(200)
#   .maxIdleTime(Duration.ofSeconds(60))
#   .pendingAcquireTimeout(Duration.ofSeconds(10))
#   .build()
```

**Fix for NAT Gateway port exhaustion:**

```bash
# Add additional NAT Gateways across AZs and distribute routes
# Or use VPC endpoints / PrivateLink if Stripe supports it

# Check current NAT GW quota
aws service-quotas get-service-quota \
  --service-code vpc \
  --quota-code L-FE5A380F
# Default: 5 NAT GWs per AZ
```

### Investigation Flow Summary

```
                    ┌──────────────────────┐
                    │  PagerDuty Alert      │
                    │  P99 Latency > SLO    │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  METRICS (Prometheus)  │
              ┌─────│  Which endpoint?       │
              │     │  Which pods?           │
              │     │  When did it start?    │
              │     └──────────┬───────────┘
              │                │ service=payment-svc
              │                │ endpoint=/charge
              │                │ time window identified
              │     ┌──────────▼───────────┐
              │     │  LOGS (Loki)          │
              │     │  Error messages?       │
              │     │  Slow request logs?    │
              │     │  Extract traceID       │
              │     └──────────┬───────────┘
              │                │ traceID=abc123
              │     ┌──────────▼───────────┐
              │     │  TRACES (Tempo)       │
              │     │  Waterfall view        │
              │     │  Identify bottleneck   │
              │     │  span: stripe-api 7.5s │
              │     └──────────┬───────────┘
              │                │ Stripe says fine
              │     ┌──────────▼───────────┐
              └────→│  METRICS (Prometheus)  │
                    │  Connection pool?      │
                    │  DNS latency?          │
                    │  NAT GW port alloc?    │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  AWS CloudWatch        │
                    │  NAT GW metrics        │
                    │  ErrorPortAllocation   │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  ROOT CAUSE:           │
                    │  Connection pool       │
                    │  MaxIdleConnsPerHost=2 │
                    │  + NAT GW port exhaust │
                    └──────────────────────┘
```

# Phase 5, Lesson 3 — Grading

---

## Q1: Broken Traces Investigation — **4.9/5**

Excellent. The three-cause prioritization is exactly right — propagator mismatch first, then transport gap (message queue), then framework coverage. The diagnostic sequence is well-ordered: check what headers arrive → check SDK config → compare traceIDs.

The message queue fix with explicit `propagate.inject()` / `propagate.extract()` is production-critical knowledge. Many teams add tracing, forget about async boundaries, and wonder why 30% of their traces are broken.

The verification approach using `otel-cli` to generate synthetic traces is creative and practical.

**One minor issue:**

Your `tcpdump` approach in Check 1 is slightly impractical for gRPC traffic (which is binary/HTTP2, not plaintext HTTP1.1). If `payment-svc` calls `fraud-svc` via gRPC, `grep` on the tcpdump output won't find `traceparent` as readable text. You'd need:

```bash
# For gRPC, check via OTel debug exporter or environment variable logging:
OTEL_LOG_LEVEL=debug  # On fraud-svc — logs all incoming span context
# Or temporarily add the debug exporter to the Collector pipeline
```

For HTTP/1.1 calls your approach works perfectly. Just flag the transport assumption.

---

## Q2: Sampling Decision Gone Wrong — **4.8/5**

**What you nailed:**

Problem A — the `loadbalancing` exporter with `routing_key: traceID` is exactly the right answer. This is the #1 tail-sampling operational mistake in production, and you identified both the mechanism (spans split across Gateways) and the architectural fix. The `decision_wait` timeout issue for slow services is also a valid secondary cause.

Problem B — the math is correct and the multi-layered fix (memory_limiter + reduced num_traces + architecture change) is sound.

Problem C — correctly identifying that this is the same root cause as Problem A (split spans) rather than a policy ordering issue shows you understand the actual data flow, not just the config syntax.

**One correction:**

You wrote about tail sampling policies:

> "policies are evaluated as OR by default — if ANY policy says 'sample,' the trace is kept"

This is correct for the default behavior, but you should know there's a nuance. The tail sampling processor actually evaluates **all** policies and uses a **composite decision**. The default decision logic is:

- If ANY policy says **Sample** → trace is sampled
- If ALL policies say **NotSampled** → trace is dropped
- If ANY policy says **InvertNotSampled** → different behavior

The important production gotcha you didn't mention: if a trace arrives but **no policy matches at all** (none of the policy conditions are met AND the probabilistic sampler's dice roll says no), the trace is dropped silently. There's no "default keep" safety net. For critical production systems, some teams add a final `always_sample` policy with a very low rate (0.1%) as a catch-all to ensure you always have *some* baseline visibility:

```yaml
- name: absolute-baseline
  type: probabilistic
  probabilistic:
    sampling_percentage: 0.1
```

Minor, but at NovaMart's scale, silently dropping 100% of traces for a low-traffic service that doesn't match any policy is a real risk.

---

## Q3: OTel Collector Pipeline Design — **4.9/5**

This is outstanding work. Both configs are complete, correct, and deployable. Every requirement is mapped to a specific config element. The architecture diagram, the headless Service for DNS-based load balancing, the RBAC manifests, and the verification commands — this is production-ready.

**Specific things done right:**
- `memory_limiter` as FIRST processor in every pipeline (non-negotiable)
- `loadbalancing` exporter at Agent → Gateway for tail sampling correctness
- Headless Service for DNS resolution (many people forget this)
- `k8sattributes` at Agent level (enrichment at the edge, not the gateway)
- Filter for health checks in the traces pipeline only (not metrics/logs)
- Separate batch processors per pipeline with different settings
- RBAC for the k8sattributes processor

**One thing I'd add:**

Your Gateway's `memory_limiter` is set to `limit_mib: 1800` but the container limit is `memory: 2Gi` (2048 MiB). That gives only 248 MiB headroom between the soft drop threshold and the OOM kill. The Go runtime, batch buffers, and gRPC overhead can easily consume 200+ MiB. Safer:

```yaml
# Container limit: 2Gi (2048 MiB)
memory_limiter:
  limit_mib: 1600       # Hard limit at ~78% of container limit
  spike_limit_mib: 400  # Soft limit at 1200 MiB
```

Rule of thumb: set `limit_mib` to ~75-80% of container memory limit. Your 88% is cutting it close.

---

## Q4: End-to-End Correlation Under Pressure — **4.7/5**

The investigation flow is excellent — metrics → logs → traces → back to metrics → AWS CloudWatch. The progression is logical, each step carries specific data to the next, and the pivot mechanisms are clearly stated.

The twist investigation (Stripe shows no issues) is handled well — connection pool exhaustion and NAT Gateway port allocation are both realistic root causes at NovaMart's scale.

**Two gaps:**

**Gap 1: You didn't use exemplars.**

The lesson specifically covered exemplars as the Metrics → Traces bridge. In your Step 1 (metrics), you should have mentioned:

> "On the latency histogram panel, I look for **exemplar dots** on the high-latency data points. Each exemplar carries a traceID. I click one of the 8-second exemplars to jump directly to the trace in Tempo — no need to go through logs first."

This skips Step 2 entirely for the initial investigation. The flow becomes:

```
Metrics (see spike) → Exemplar click → Trace (see bottleneck) → Logs (for detail)
```

Instead of:

```
Metrics → Logs (find traceID) → Trace
```

Both work, but exemplars are faster and that's literally why they exist. You covered them in the lesson material but didn't use them in the incident response.

**Gap 2: The PromQL queries for HTTP client connection pool metrics assume those metrics exist.**

Metrics like `http_client_active_connections` and `http_client_connection_wait_seconds_bucket` only exist if the application explicitly instruments them, or if OTel auto-instrumentation includes them (which varies by language and client library). In a real incident, you might not have these metrics.

A more reliable approach when specific client metrics are unavailable:

```promql
# Check for TCP connection state from node-exporter (works regardless of app instrumentation)
# Count ESTABLISHED connections from payment-svc pods to Stripe's IP range
node_netstat_Tcp_CurrEstab{instance=~"payment-svc-node.*"}

# Or from the service mesh (Istio) if available:
istio_tcp_connections_opened_total{
  destination_service="api.stripe.com",
  source_workload="payment-svc"
}

# Connection duration from Istio:
istio_request_duration_milliseconds_bucket{
  destination_service_name="api.stripe.com",
  source_workload="payment-svc"
}
```

Not a major miss — your investigation was thorough and would have found the root cause. But relying on metrics that may not exist is a trap in real incidents.



Proceeding to Lesson 4.

---

# Phase 5 — Lesson 4: SLOs, SLIs, Error Budgets, and SLO-Based Alerting

---

## Why SLOs Exist — The Fundamental Problem They Solve

Without SLOs, your team lives in one of two failure modes:

```
FAILURE MODE 1: "Everything is an emergency"
──────────────────────────────────────────────
Every alert fires → on-call pages → team scrambles
p99 went from 180ms to 220ms? PAGE!
Error rate went from 0.01% to 0.03%? PAGE!
Team burns out. Alert fatigue. Real incidents get ignored.

FAILURE MODE 2: "Ship and pray"
──────────────────────────────────────────────
No reliability targets → developers push features nonstop
No one knows when quality is "good enough"
Users complain → reactive firefighting → trust erodes
Management asks "are we reliable?" and nobody has a number
```

**SLOs solve both problems by answering one question: "How reliable do we NEED to be?"**

Not how reliable we CAN be (100% is impossible and infinitely expensive), but how reliable we NEED to be for our users and business to succeed.

---

## The SLO Framework — Definitions

```
┌──────────┬────────────────────────────────────────────────────────────┐
│ Term     │ Definition                                                 │
├──────────┼────────────────────────────────────────────────────────────┤
│ SLI      │ Service Level INDICATOR                                    │
│          │ A quantitative MEASUREMENT of a service's behavior.        │
│          │ "What are we measuring?"                                   │
│          │                                                            │
│          │ Format: ratio of good events / total events               │
│          │ Example: 99.2% of HTTP requests returned in < 300ms       │
│          │                                                            │
│          │ SLIs are ALWAYS expressed as proportions (0-100%)          │
│          │ NEVER as raw numbers (not "200ms p99")                    │
├──────────┼────────────────────────────────────────────────────────────┤
│ SLO      │ Service Level OBJECTIVE                                    │
│          │ A TARGET value for an SLI over a time window.              │
│          │ "What's our reliability target?"                           │
│          │                                                            │
│          │ Format: SLI ≥ target% over rolling window                 │
│          │ Example: 99.9% of requests succeed within 300ms           │
│          │          measured over a 30-day rolling window             │
│          │                                                            │
│          │ SLOs are INTERNAL targets — chosen by engineering          │
│          │ They should be TIGHTER than SLAs                          │
├──────────┼────────────────────────────────────────────────────────────┤
│ SLA      │ Service Level AGREEMENT                                    │
│          │ A CONTRACT with customers. Legal/financial consequences    │
│          │ if breached. Typically looser than SLOs.                   │
│          │                                                            │
│          │ Example: "99.9% uptime or customer gets credits"          │
│          │                                                            │
│          │ SLA ≤ SLO (always) — SLO is the internal safety margin   │
│          │ If SLO = 99.95% and SLA = 99.9%, you have a 0.05%       │
│          │ buffer before you owe customers money                     │
├──────────┼────────────────────────────────────────────────────────────┤
│ Error    │ The amount of unreliability you're ALLOWED to have.       │
│ Budget   │                                                            │
│          │ Error Budget = 1 - SLO target                             │
│          │                                                            │
│          │ If SLO = 99.9%, Error Budget = 0.1%                      │
│          │ Over 30 days: 0.1% × 30 × 24 × 60 = 43.2 minutes       │
│          │                                                            │
│          │ You can "spend" this budget on:                           │
│          │   - Planned maintenance                                   │
│          │   - Risky deployments                                     │
│          │   - Incidents/outages                                     │
│          │                                                            │
│          │ When budget exhausted → STOP deploying features           │
│          │                        → Focus on reliability work         │
└──────────┴────────────────────────────────────────────────────────────┘
```

### The Error Budget Math

```
SLO: 99.9% availability over 30-day window

Error Budget = 1 - 0.999 = 0.001 = 0.1%

In time:
  30 days × 24 hours × 60 minutes = 43,200 minutes
  43,200 × 0.001 = 43.2 minutes of allowed downtime

In requests (at 2,000 req/s):
  30 days × 86,400 s/day × 2,000 req/s = 5,184,000,000 total requests
  5,184,000,000 × 0.001 = 5,184,000 allowed failed requests

Common SLO targets and their budgets (30-day window):
┌──────────┬──────────────┬─────────────────┬───────────────────────┐
│ SLO      │ Error Budget │ Downtime/month  │ Typical use           │
├──────────┼──────────────┼─────────────────┼───────────────────────┤
│ 99%      │ 1%           │ 7.2 hours       │ Internal tools        │
│ 99.5%    │ 0.5%         │ 3.6 hours       │ Non-critical services │
│ 99.9%    │ 0.1%         │ 43.2 minutes    │ Most production APIs  │
│ 99.95%   │ 0.05%        │ 21.6 minutes    │ Critical user-facing  │
│ 99.99%   │ 0.01%        │ 4.32 minutes    │ Payment, auth         │
│ 99.999%  │ 0.001%       │ 26 seconds      │ Almost never — don't  │
│          │              │                 │ set this unless you're│
│          │              │                 │ operating a database  │
└──────────┴──────────────┴─────────────────┴───────────────────────┘
```

**Why 100% is wrong:**

```
100% SLO means:
  - Zero error budget
  - NEVER deploy (any change risks an error)
  - NEVER do maintenance
  - NEVER scale down for cost savings
  - Your reliability target blocks ALL engineering velocity
  - And it's STILL impossible (hardware fails, networks partition, S3 has outages)

The question is NOT "how do we achieve 100%?"
The question is "what's the RIGHT level of unreliability?"
```

---

## Choosing SLIs — What to Measure

### The SLI Menu

Not every metric is an SLI. SLIs must **directly reflect user experience:**

```
GOOD SLIs (user-facing):                    BAD SLIs (internal/indirect):
────────────────────────                     ────────────────────────────
✅ Request latency (user waits)              ❌ CPU utilization (not user-facing)
✅ Error rate (user sees failure)            ❌ Memory usage (infra metric)
✅ Availability (can user reach us?)         ❌ Queue depth (internal plumbing)
✅ Throughput (are we processing?)           ❌ Pod restart count (K8s detail)
✅ Data freshness (is data current?)         ❌ Deployment frequency (process)
✅ Correctness (is data right?)              ❌ Alert count (meta-metric)
```

### SLI Types and When to Use Each

```
┌──────────────────┬────────────────────────────────────────────────────┐
│ SLI Type         │ Formula & Use Case                                 │
├──────────────────┼────────────────────────────────────────────────────┤
│ Availability     │ successful requests / total requests               │
│                  │ Use for: API endpoints, web pages                  │
│                  │ "Successful" = non-5xx (exclude 4xx — those are    │
│                  │ client errors, not OUR failures)                   │
│                  │                                                    │
│                  │ ⚠️ Include timeouts as failures                    │
│                  │ ⚠️ Include circuit-breaker rejections as failures  │
├──────────────────┼────────────────────────────────────────────────────┤
│ Latency          │ requests faster than threshold / total requests    │
│                  │ Use for: APIs where speed matters (search, page    │
│                  │ load, checkout)                                    │
│                  │                                                    │
│                  │ NOT "p99 < 300ms" — instead:                      │
│                  │ "99% of requests complete within 300ms"           │
│                  │                                                    │
│                  │ Can have multiple thresholds:                      │
│                  │   - 90% of requests < 100ms (fast)                │
│                  │   - 99% of requests < 500ms (acceptable)          │
│                  │   - 99.9% of requests < 2s (max tolerable)        │
├──────────────────┼────────────────────────────────────────────────────┤
│ Freshness        │ data updated within threshold / total data points  │
│                  │ Use for: Data pipelines, caches, search indexes    │
│                  │                                                    │
│                  │ Example: "99.9% of product prices reflect changes │
│                  │ within 5 minutes of update"                       │
├──────────────────┼────────────────────────────────────────────────────┤
│ Correctness      │ correct results / total results                    │
│                  │ Use for: Search results, recommendations, billing  │
│                  │                                                    │
│                  │ Example: "99.99% of charges match the displayed   │
│                  │ price at time of checkout"                        │
│                  │                                                    │
│                  │ ⚠️ Hardest to measure — often requires synthetic  │
│                  │ probes or end-to-end validation                   │
├──────────────────┼────────────────────────────────────────────────────┤
│ Throughput       │ actual throughput / expected throughput             │
│                  │ Use for: Batch processing, streaming systems       │
│                  │                                                    │
│                  │ Example: "99% of order events are processed       │
│                  │ within 30 seconds of creation"                    │
├──────────────────┼────────────────────────────────────────────────────┤
│ Durability       │ data retrievable / data stored                     │
│                  │ Use for: Storage systems, databases                │
│                  │                                                    │
│                  │ Example: "99.999999999% of objects stored in S3   │
│                  │ are retrievable" (AWS's own SLA: 11 nines)       │
└──────────────────┴────────────────────────────────────────────────────┘
```

---

## NovaMart SLOs — Real Implementation

### SLO Definitions for Key Services

```
┌────────────────────┬──────────────────────────────────────────────────────────┐
│ Service            │ SLOs                                                      │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ API Gateway        │ Availability: 99.95% of requests return non-5xx          │
│                    │ Latency: 99% of requests complete within 200ms           │
│                    │ Window: 30-day rolling                                   │
│                    │                                                          │
│                    │ Error budget: 0.05% = 21.6 min/month                    │
│                    │ At 2000 req/s = 2,592,000 failed requests/month         │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Payment Service    │ Availability: 99.99% of charge requests succeed          │
│                    │ Latency: 99.9% of requests complete within 2s           │
│                    │ Correctness: 99.999% of charges are for correct amount  │
│                    │ Window: 30-day rolling                                   │
│                    │                                                          │
│                    │ Error budget: 0.01% = 4.32 min/month                    │
│                    │ ⚠️ Tightest SLO — money is on the line                  │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Search Service     │ Availability: 99.9% of queries return results            │
│                    │ Latency: 95% < 200ms, 99% < 1s                          │
│                    │ Freshness: 99.9% of products searchable within 5 min    │
│                    │            of catalog update                             │
│                    │ Window: 30-day rolling                                   │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Order Service      │ Availability: 99.95% of order submissions succeed        │
│                    │ Latency: 99% complete within 3s (complex workflow)       │
│                    │ Correctness: 99.99% of orders match cart contents        │
│                    │ Window: 30-day rolling                                   │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Notification Svc   │ Availability: 99.9% of notifications sent               │
│                    │ Freshness: 99% of notifications delivered within 60s     │
│                    │ Window: 30-day rolling                                   │
│                    │                                                          │
│                    │ ⚠️ Looser SLO — notifications are important but not     │
│                    │ transactional. Users tolerate email 60s late.            │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Data Pipeline      │ Freshness: 99.9% of analytics data < 15 min stale      │
│ (Airflow)          │ Completeness: 99.9% of scheduled jobs complete          │
│                    │ Window: 7-day rolling (shorter — batch workload)         │
└────────────────────┴──────────────────────────────────────────────────────────┘
```

### Implementing SLIs in Prometheus

**Availability SLI — API Gateway:**

```yaml
# PrometheusRule for recording the SLI
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-api-gateway
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: slo-api-gateway
      interval: 30s
      rules:
        # SLI: ratio of successful requests (non-5xx) to total requests
        # Recording rule — pre-compute every 30s
        - record: sli:api_gateway:availability:rate5m
          expr: |
            sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="api-gateway"}[5m]))

        # Error ratio (inverse of availability — useful for alerting)
        - record: sli:api_gateway:error_ratio:rate5m
          expr: |
            1 - sli:api_gateway:availability:rate5m

        # SLI over different windows (for burn rate calculation)
        - record: sli:api_gateway:error_ratio:rate1h
          expr: |
            1 - (
              sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[1h]))
              /
              sum(rate(http_requests_total{service="api-gateway"}[1h]))
            )

        - record: sli:api_gateway:error_ratio:rate6h
          expr: |
            1 - (
              sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[6h]))
              /
              sum(rate(http_requests_total{service="api-gateway"}[6h]))
            )

        # Error budget remaining (percentage)
        # SLO target: 99.95% → error budget = 0.05%
        - record: slo:api_gateway:error_budget_remaining:ratio
          expr: |
            1 - (
              sli:api_gateway:error_ratio:rate30d
              /
              0.0005
            )
          # Result: 1.0 = full budget remaining
          #         0.5 = half budget consumed
          #         0.0 = budget exhausted
          #        -0.5 = 50% over budget (SLO breached)
```

**Latency SLI — Payment Service:**

```yaml
        # Latency SLI: proportion of requests faster than 2s
        # Uses histogram_quantile indirectly — count requests below threshold
        - record: sli:payment_svc:latency:good_ratio:rate5m
          expr: |
            sum(rate(http_request_duration_seconds_bucket{
              service="payment-svc",
              le="2.0"
            }[5m]))
            /
            sum(rate(http_request_duration_seconds_bucket{
              service="payment-svc",
              le="+Inf"
            }[5m]))

        # Error ratio for latency SLI
        - record: sli:payment_svc:latency:error_ratio:rate5m
          expr: |
            1 - sli:payment_svc:latency:good_ratio:rate5m
```

**Why use the histogram bucket at `le="2.0"` instead of `histogram_quantile()`?**

```
histogram_quantile(0.999, ...) answers: "What latency value does the 99.9th percentile hit?"
  → Gives you a DURATION (e.g., 1.8s)
  → Useful for understanding latency distribution
  → NOT directly useful for SLO calculation (you need a ratio, not a value)

http_request_duration_seconds_bucket{le="2.0"} answers: "How many requests completed in ≤ 2s?"
  → Gives you a COUNT of "good" events
  → Divide by total = ratio of good events = SLI
  → THIS is what SLOs need

The SLI is "99.9% of requests complete within 2s"
  → We measure: (requests ≤ 2s) / (total requests)
  → If result ≥ 0.999, we're meeting the SLO
```

**⚠️ This means your histogram bucket boundaries MUST include the SLO threshold.** If your SLO says "2 seconds" and your histogram has buckets `[0.1, 0.5, 1.0, 5.0, 10.0]`, there's no `2.0` bucket. Prometheus interpolates between 1.0 and 5.0, which is inaccurate. **Design your histogram buckets around your SLO thresholds.**

```go
// Go — design buckets with SLO thresholds
prometheus.NewHistogram(prometheus.HistogramOpts{
    Name:    "http_request_duration_seconds",
    Help:    "HTTP request duration",
    Buckets: []float64{
        0.01, 0.025, 0.05, 0.1, 0.25, 0.5,
        1.0, 2.0,    // ← SLO threshold bucket
        5.0, 10.0,
    },
})
```

---

## Error Budget Policies — What Happens When the Budget Runs Out

### The Error Budget Policy Document

This is the **social contract** between SRE/Platform and product development teams:

```
NovaMart Error Budget Policy
═══════════════════════════════

1. WHEN ERROR BUDGET > 50% REMAINING:
   ────────────────────────────────────
   - Normal development velocity
   - Standard change management
   - Feature work prioritized
   - On-call follows normal rotation

2. WHEN ERROR BUDGET 25-50% REMAINING:
   ────────────────────────────────────
   - Team reviews recent incidents
   - Risky deployments require additional review
   - Reliability work gets 20% of sprint capacity
   - On-call team checks SLO dashboard daily

3. WHEN ERROR BUDGET 0-25% REMAINING:
   ────────────────────────────────────
   - All non-critical deployments PAUSED
   - Only bug fixes, security patches, reliability work deployed
   - Team dedicates 50% capacity to reliability
   - Incident review for all budget-consuming events
   - Engineering manager and SRE lead review daily

4. WHEN ERROR BUDGET EXHAUSTED (≤ 0):
   ─────────────────────────────────────
   - ALL feature deployments FROZEN
   - 100% team focus on reliability
   - VP-level approval required for ANY deployment
   - Postmortem required for the budget-exhausting incident
   - Freeze continues until budget recovers (new window rolls in)
   
5. EXCEPTIONS:
   ────────────
   - Security-critical patches: always allowed
   - Revenue-critical fixes: VP approval, must include reliability improvements
   - Rollbacks: always allowed (they improve reliability)

6. ESCALATION:
   ────────────
   - Service owner + SRE lead: daily check at 50%
   - + Engineering Director: when budget < 25%
   - + VP Engineering: when budget exhausted
   - Monthly SLO review meeting (all stakeholders)
```

**Why this policy works at NovaMart:**

```
Without policy:                        With policy:
─────────────────                      ──────────────
Dev: "Let's ship feature X"           Dev: "Let's ship feature X"
SRE: "That's risky"                   SRE: "Error budget is at 15%"
Dev: "Business wants it"               Dev: "So we need VP approval?"
SRE: "Fine... (gets paged at 3AM)"    SRE: "Yes, and include rollback plan"
                                       Dev: "OK, let's also fix that flaky test"
→ Adversarial, political               → Data-driven, collaborative
→ SRE = gatekeeper (hated)            → SRE = enabler with guardrails
→ Reliability is someone else's job    → Reliability is everyone's job
```

---

## SLO-Based Alerting — Burn Rate Alerts

### Why Traditional Alerting Fails for SLOs

```
Traditional alert:
  "Alert if error rate > 1% for 5 minutes"

Problems:
  1. 1% for 5 minutes = 0.003% of monthly budget consumed
     → Way too sensitive. False positives.
  
  2. If error rate is 0.08% for 3 days:
     0.08% × 3 days / 30 days = 0.8% of budget consumed per day
     Over 30 days: 24% of budget consumed → significant!
     But 0.08% never triggers the 1% threshold → MISSED.
  
  3. A 5-minute burst of 50% error rate:
     50% × 5min / 43,200min = 0.06% of budget → trivial
     But the alert fires → unnecessary page.
```

**The solution: alert on how fast you're BURNING your error budget.**

### Burn Rate Explained

```
Burn rate = actual error rate / allowed error rate (per SLO)

If SLO = 99.9% over 30 days:
  Allowed error rate = 0.1%
  
If actual error rate = 0.1%:
  Burn rate = 0.1% / 0.1% = 1.0
  → Consuming budget at exactly the expected rate
  → At this rate, budget exhausted at exactly day 30 = OK

If actual error rate = 1%:
  Burn rate = 1% / 0.1% = 10
  → Consuming budget 10x faster than allowed
  → Budget exhausted in 3 days instead of 30 → ALERT!

If actual error rate = 10%:
  Burn rate = 10% / 0.1% = 100
  → Budget exhausted in 7.2 hours → PAGE NOW!
```

### Multi-Window, Multi-Burn-Rate Alerting

Google's SRE book recommends a multi-window approach for comprehensive coverage:

```
┌────────────────┬─────────────┬─────────────────┬────────────────────────────┐
│ Severity       │ Burn Rate   │ Long Window     │ Short Window (reset check) │
│                │             │ (detection)     │ (is it still happening?)   │
├────────────────┼─────────────┼─────────────────┼────────────────────────────┤
│ PAGE (SEV1)    │ 14.4x       │ 1 hour          │ 5 minutes                  │
│ (2% budget     │             │                 │                            │
│  consumed in   │             │ If 14.4x for    │ AND 14.4x in last 5min    │
│  1 hour)       │             │ 1 hour → 2%     │ (still burning = not       │
│                │             │ budget gone      │  a brief spike that ended) │
├────────────────┼─────────────┼─────────────────┼────────────────────────────┤
│ PAGE (SEV1)    │ 6x          │ 6 hours         │ 30 minutes                 │
│ (5% budget     │             │                 │                            │
│  consumed in   │             │                 │                            │
│  6 hours)      │             │                 │                            │
├────────────────┼─────────────┼─────────────────┼────────────────────────────┤
│ TICKET (SEV3)  │ 3x          │ 1 day           │ 2 hours                    │
│ (10% budget    │             │                 │                            │
│  consumed in   │             │                 │                            │
│  1 day)        │             │                 │                            │
├────────────────┼─────────────┼─────────────────┼────────────────────────────┤
│ TICKET (SEV4)  │ 1x          │ 3 days          │ 6 hours                    │
│ (10% budget    │             │                 │                            │
│  consumed in   │             │ Slow burn — not │                            │
│  3 days)       │             │ urgent but will │                            │
│                │             │ exhaust budget   │                            │
└────────────────┴─────────────┴─────────────────┴────────────────────────────┘

Why two windows?
─────────────────
Long window: detects that significant budget is being consumed
Short window: confirms the problem is ONGOING (not a past spike that's already resolved)

Without short window: you get paged for a 5-minute spike that ended 55 minutes ago
Without long window: you miss slow-burn reliability degradation
```

### Implementing Burn Rate Alerts in Prometheus

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-burn-rate-alerts
  namespace: monitoring
spec:
  groups:
    - name: slo-api-gateway-burn-rate
      rules:
        # === RECORDING RULES for error ratios at different windows ===
        
        # 5-minute window
        - record: sli:api_gateway:error_ratio:rate5m
          expr: |
            1 - (
              sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[5m]))
              /
              sum(rate(http_requests_total{service="api-gateway"}[5m]))
            )
        
        # 30-minute window
        - record: sli:api_gateway:error_ratio:rate30m
          expr: |
            1 - (
              sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[30m]))
              /
              sum(rate(http_requests_total{service="api-gateway"}[30m]))
            )
        
        # 1-hour window
        - record: sli:api_gateway:error_ratio:rate1h
          expr: |
            1 - (
              sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[1h]))
              /
              sum(rate(http_requests_total{service="api-gateway"}[1h]))
            )
        
        # 2-hour window
        - record: sli:api_gateway:error_ratio:rate2h
          expr: |
            1 - (
              sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[2h]))
              /
              sum(rate(http_requests_total{service="api-gateway"}[2h]))
            )
        
        # 6-hour window
        - record: sli:api_gateway:error_ratio:rate6h
          expr: |
            1 - (
              sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[6h]))
              /
              sum(rate(http_requests_total{service="api-gateway"}[6h]))
            )
        
        # 1-day window
        - record: sli:api_gateway:error_ratio:rate1d
          expr: |
            1 - (
              sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[1d]))
              /
              sum(rate(http_requests_total{service="api-gateway"}[1d]))
            )
        
        # 3-day window
        - record: sli:api_gateway:error_ratio:rate3d
          expr: |
            1 - (
              sum(rate(http_requests_total{service="api-gateway", status!~"5.."}[3d]))
              /
              sum(rate(http_requests_total{service="api-gateway"}[3d]))
            )

        # === BURN RATE ALERTS ===
        # SLO: 99.95% → error budget = 0.0005

        # PAGE: 14.4x burn rate for 1 hour (2% budget consumed)
        # AND still burning in last 5 minutes
        - alert: APIGatewayHighBurnRate_Page
          expr: |
            sli:api_gateway:error_ratio:rate1h > (14.4 * 0.0005)
            and
            sli:api_gateway:error_ratio:rate5m > (14.4 * 0.0005)
          for: 2m
          labels:
            severity: critical
            team: platform
            slo: api-gateway-availability
          annotations:
            summary: "API Gateway SLO burn rate critical — 2% budget consumed in 1 hour"
            description: |
              Error ratio 1h: {{ $value | humanizePercentage }}
              Burn rate: {{ printf "%.1f" (divf $value 0.0005) }}x
              At this rate, error budget exhausted in {{ printf "%.1f" (divf 1 (divf $value 0.0005) | mulf 30) }} days
            runbook_url: "https://wiki.novamart.internal/runbooks/slo-burn-api-gateway"
            dashboard_url: "https://grafana.novamart.internal/d/slo-api-gateway"

        # PAGE: 6x burn rate for 6 hours (5% budget consumed)
        - alert: APIGatewayHighBurnRate_Page_Slow
          expr: |
            sli:api_gateway:error_ratio:rate6h > (6 * 0.0005)
            and
            sli:api_gateway:error_ratio:rate30m > (6 * 0.0005)
          for: 5m
          labels:
            severity: critical
            team: platform
            slo: api-gateway-availability
          annotations:
            summary: "API Gateway SLO sustained burn — 5% budget consumed in 6 hours"
            runbook_url: "https://wiki.novamart.internal/runbooks/slo-burn-api-gateway"

        # TICKET: 3x burn rate for 1 day
        - alert: APIGatewayModerateBurnRate_Ticket
          expr: |
            sli:api_gateway:error_ratio:rate1d > (3 * 0.0005)
            and
            sli:api_gateway:error_ratio:rate2h > (3 * 0.0005)
          for: 15m
          labels:
            severity: warning
            team: platform
            slo: api-gateway-availability
          annotations:
            summary: "API Gateway SLO moderate burn — 10% budget consumed in 1 day"

        # TICKET: 1x burn rate for 3 days (slow burn — will exhaust budget)
        - alert: APIGatewaySlowBurnRate_Ticket
          expr: |
            sli:api_gateway:error_ratio:rate3d > (1 * 0.0005)
            and
            sli:api_gateway:error_ratio:rate6h > (1 * 0.0005)
          for: 30m
          labels:
            severity: warning
            team: platform
            slo: api-gateway-availability
          annotations:
            summary: "API Gateway SLO slow burn — on pace to exhaust budget in 30 days"
```

### Grafana SLO Dashboard

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    API GATEWAY SLO DASHBOARD                             │
├──────────────────────┬───────────────────────────────────────────────────┤
│   ERROR BUDGET       │   BURN RATE (current)                             │
│                      │                                                   │
│   ██████████░░░░░░   │   ┌─────────────────────────────────┐            │
│   68% remaining      │   │  ▂▃▂▂▃▂▂▂▂▂▅▇█▇▅▃▃▂▂▂         │            │
│                      │   │  ─── 14.4x page threshold       │            │
│   Budget: 30d window │   │  ─── 6x page threshold          │            │
│   Consumed: 32%      │   │  ─── 1x expected rate           │            │
│                      │   └─────────────────────────────────┘            │
├──────────────────────┼───────────────────────────────────────────────────┤
│   AVAILABILITY SLI   │   LATENCY SLI                                    │
│   (over 30d window)  │   (over 30d window)                              │
│                      │                                                   │
│   Current: 99.967%   │   Current: 99.4% < 200ms                        │
│   Target:  99.950%   │   Target:  99.0% < 200ms                        │
│   Status:  ✅ MET    │   Status:  ✅ MET                                │
│                      │                                                   │
│   ┌──────────────┐   │   ┌──────────────┐                              │
│   │ ─────────    │   │   │ ─────────    │                              │
│   │   99.95 ─ ─ ─│   │   │   99.0 ─ ─ ─│                              │
│   │ ─────────    │   │   │ ─────────    │                              │
│   └──────────────┘   │   └──────────────┘                              │
├──────────────────────┴───────────────────────────────────────────────────┤
│   ERROR BUDGET CONSUMPTION TIMELINE (30-day)                             │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │         ╱                                                        │   │
│   │        ╱  Incident on day 12                                     │   │
│   │ ──────╱───── consumed 15% in 2 hours                            │   │
│   │      ╱                                                           │   │
│   │     ╱                                                            │   │
│   │────╱──── steady consumption from background errors               │   │
│   │   ╱                                                              │   │
│   │──╱                                                               │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│   Day 1                                               Day 30           │
├─────────────────────────────────────────────────────────────────────────┤
│   RECENT INCIDENTS CONSUMING BUDGET                                     │
│   ┌─────────────────────────────────────────────────────────────┐       │
│   │ Day 12: Payment gateway timeout — consumed 15% budget       │       │
│   │ Day 18: Deploy regression — consumed 3% budget              │       │
│   │ Day 22: DNS resolution delay — consumed 1% budget           │       │
│   └─────────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key PromQL queries for this dashboard:**

```promql
# Error budget remaining (percentage)
(1 - (
  sli:api_gateway:error_ratio:rate30d / 0.0005
)) * 100

# Current SLI value (availability over 30 days)
(1 - sli:api_gateway:error_ratio:rate30d) * 100

# Current burn rate
sli:api_gateway:error_ratio:rate1h / 0.0005

# Error budget consumption over time (cumulative errors)
sum(increase(http_requests_total{service="api-gateway", status=~"5.."}[30d]))
/
(sum(increase(http_requests_total{service="api-gateway"}[30d])) * 0.0005)
* 100
```

---

## SLO Anti-Patterns and Gotchas

### 1. Setting SLOs Too High

```
Team says: "We want 99.99% for everything!"

Reality:
  99.99% = 4.32 minutes/month error budget
  A single 5-minute deploy rollback consumes your ENTIRE monthly budget
  You can NEVER deploy during business hours
  One slow DNS resolution eats half your budget

Fix: Start with 99.9%, observe actual reliability, tighten IF users need it
     AND you can afford the engineering investment to maintain it
```

### 2. SLOs Without Error Budget Policies

```
Having SLOs without error budget policies is like having speed limits
without police. Nobody cares.

"We're at 99.85% against a 99.9% target."
"So what? Ship the feature anyway."

Fix: Error budget policies must be AGREED upon and ENFORCED
     With executive buy-in, in writing, reviewed quarterly
```

### 3. SLIs That Don't Match User Experience

```
SLI: "API returns 200 OK within 500ms"

But the user's actual experience:
  Browser → CDN → Load Balancer → API → DB → API → Load Balancer → CDN → Browser
  Total: 2.5 seconds even though the API said 200 in 400ms

The API SLI is green while the user is frustrated.

Fix: Measure SLIs AS CLOSE TO THE USER AS POSSIBLE
  - Browser Real User Monitoring (RUM) for web
  - Synthetic probes from external locations
  - Load balancer metrics (capture full round-trip)
  - NOT just server-side metrics
```

### 4. Excluding Things That Users Notice

```
"Our availability is 99.99%! (excluding maintenance windows, 
 partial outages, slow degradation, and errors that auto-recover)"

If the user experienced it, it counts. Period.

Fix: SLI measurement must include ALL user-facing degradation
  - Planned maintenance? Users still can't use the service → counts
  - Partial outage (10% of users)? 10% failure rate → counts
  - Auto-recovered after 30 seconds? 30 seconds of failures → counts
```

### 5. The 30-Day Rolling Window Trap

```
Day 1-29: Perfect reliability (100%)
Day 30: 45-minute outage (consumes 100%+ of monthly budget)

With calendar month: Budget resets on day 1 → no consequences
With rolling window: That outage stays in the window for 30 days

NovaMart uses ROLLING windows — this ensures every outage
has sustained impact on the SLO, preventing "burn it all at month end"
```

### 6. Denominator Problems

```
At 3 AM, NovaMart gets 10 req/s instead of 2000 req/s.
One error = 10% error rate!
SLO alert fires. On-call gets paged.

But 1 error out of 10 requests is meaningless noise.

Fix: Minimum request threshold before SLI is evaluated
  expr: |
    (
      sli:api_gateway:error_ratio:rate1h > (14.4 * 0.0005)
      and
      sli:api_gateway:error_ratio:rate5m > (14.4 * 0.0005)
    )
    and
    sum(rate(http_requests_total{service="api-gateway"}[5m])) > 1
    # Only alert if traffic > 1 req/s (significant volume)
```

---

## SLO Tooling — Sloth

Manually writing burn rate recording rules and alerts is tedious and error-prone. **Sloth** generates them from a simple SLO spec:

```yaml
# sloth-slos/api-gateway.yaml
version: "prometheus/v1"
service: "api-gateway"
labels:
  team: platform
  tier: "1"
slos:
  - name: "requests-availability"
    objective: 99.95  # SLO target
    description: "API Gateway availability"
    sli:
      events:
        error_query: sum(rate(http_requests_total{service="api-gateway", status=~"5.."}[{{.window}}]))
        total_query: sum(rate(http_requests_total{service="api-gateway"}[{{.window}}]))
    alerting:
      name: APIGatewayAvailability
      labels:
        team: platform
      annotations:
        runbook_url: "https://wiki.novamart.internal/runbooks/slo-api-gateway"
      page_alert:
        labels:
          severity: critical
      ticket_alert:
        labels:
          severity: warning

  - name: "requests-latency"
    objective: 99.0
    description: "API Gateway latency under 200ms"
    sli:
      events:
        error_query: |
          sum(rate(http_request_duration_seconds_count{service="api-gateway"}[{{.window}}]))
          -
          sum(rate(http_request_duration_seconds_bucket{service="api-gateway", le="0.2"}[{{.window}}]))
        total_query: sum(rate(http_request_duration_seconds_count{service="api-gateway"}[{{.window}}]))
    alerting:
      name: APIGatewayLatency
      labels:
        team: platform
      annotations:
        runbook_url: "https://wiki.novamart.internal/runbooks/slo-latency-api-gateway"
      page_alert:
        labels:
          severity: critical
      ticket_alert:
        labels:
          severity: warning
```

```bash
# Generate Prometheus recording rules + alert rules
sloth generate -i sloth-slos/api-gateway.yaml -o prometheus-rules/api-gateway.yaml

# The output is a PrometheusRule YAML with:
# - All recording rules for error ratios at every window (5m, 30m, 1h, 2h, 6h, 1d, 3d)
# - All burn rate alerts (page + ticket) with proper multi-window logic
# - Error budget recording rules
# - 50+ lines of rules generated from 30 lines of spec
```

**Sloth in CI/CD:**

```yaml
# Jenkinsfile
stage('Generate SLO Rules') {
  steps {
    sh 'sloth generate -i sloth-slos/ -o generated-rules/'
    sh 'promtool check rules generated-rules/*.yaml'
    sh 'kubectl apply -f generated-rules/ --namespace monitoring'
  }
}
```

---

## Quick Reference Card

```
SLO FRAMEWORK
─────────────
SLI = measurement (ratio: good events / total events, always 0-100%)
SLO = target (SLI ≥ X% over Y-day rolling window)
SLA = contract (SLO with legal/financial consequences, always ≤ SLO)
Error Budget = 1 - SLO target (the allowed unreliability)

ERROR BUDGET MATH (30-day window)
─────────────────────────────────
99%    → 7.2 hours    │  99.95% → 21.6 minutes
99.5%  → 3.6 hours    │  99.99% → 4.32 minutes
99.9%  → 43.2 minutes │  99.999% → 26 seconds

SLI IMPLEMENTATION
──────────────────
Availability: non-5xx / total  (use http_requests_total)
Latency: bucket{le="threshold"} / bucket{le="+Inf"}  (NOT histogram_quantile)
⚠️ Histogram buckets MUST include SLO threshold value

BURN RATE ALERTING
──────────────────
Burn rate = actual error rate / allowed error rate
1x  = consuming at expected pace (exhausted at day 30)
14.4x = consuming 14.4x faster (exhausted in ~2 days) → PAGE
6x  = exhausted in ~5 days → PAGE
3x  = exhausted in ~10 days → TICKET
1x  = on pace to exhaust → TICKET

Multi-window: long window (detection) AND short window (still happening?)
Page:   14.4x/1h AND 14.4x/5m,  6x/6h AND 6x/30m
Ticket: 3x/1d AND 3x/2h,        1x/3d AND 1x/6h

ERROR BUDGET POLICY
───────────────────
>50% remaining:  Normal velocity
25-50%:          Increased review, 20% reliability work
0-25%:           Non-critical deploys paused, 50% reliability
Exhausted:       Feature freeze, VP approval for any deploy

TOOLING
───────
Sloth:  SLO spec → Prometheus recording rules + alerts (auto-generated)
Design: histogram buckets around SLO thresholds
Dashboard: budget remaining, burn rate, SLI over time, budget timeline
```

---

## Retention Questions — Phase 5 Lesson 4

### Q1: SLO Design Under Constraints 🔥

**Scenario:** NovaMart's VP of Product comes to you with a demand: "Payment service needs to be 99.999% available. Our competitors advertise five nines. Make it happen."

The payment service currently runs at 99.97% availability over the last 90 days.

**Questions:**
1. Calculate the error budget for 99.999% over a 30-day window. How many minutes of downtime is this?
2. Explain to the VP (non-technical audience) why 99.999% is almost certainly the wrong target. Give **three specific technical constraints** that make this impractical for NovaMart's payment service.
3. What SLO would you recommend instead? Justify with the current 99.97% data and NovaMart's architecture.
4. Write the **exact Prometheus recording rules** for both an availability SLI and a latency SLI for `payment-svc`. The latency SLO is "99.9% of charge requests complete within 2 seconds."

---

### Q2: Burn Rate Alert Investigation 🔥

**Scenario:** Monday 9 AM. You receive a **ticket** (not a page) from the SLO alerting system:

```
Alert: OrderServiceSlowBurnRate_Ticket
Severity: warning
Description: "Order service SLO slow burn — 1x burn rate for 3 days, on pace to exhaust budget in 30 days"
Error budget remaining: 38%
```

This is not an acute outage — the service is working, but it's slowly degrading.

**Questions:**
1. Is this alert actionable? Should you investigate or ignore it? Justify your answer.
2. What PromQL queries would you run to understand WHERE the budget is being consumed? (Which endpoints? Which error codes? Which time periods?)
3. You discover that the error rate is 0.12% (against a 0.1% budget). The errors are all `503 Service Unavailable` from a single downstream dependency (`inventory-svc`). The errors happen in bursts of 2-3 minutes, roughly every 4 hours. What's the most likely root cause?
4. What's your fix — both immediate and permanent? Include the specific change that stops the budget bleed.

