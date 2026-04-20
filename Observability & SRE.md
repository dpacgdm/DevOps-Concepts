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
┌───────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                                  │
│                                                                   │
│  "The ability to understand the internal state of a system        │
│   by examining its external outputs"                              │
│                                                                   │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐      │
│  │   METRICS   │   │    LOGS     │   │      TRACES         │      │
│  │             │   │             │   │                     │      │
│  │ "What is    │   │ "What       │   │ "What is the path   │      │
│  │  happening?"│   │  happened?" │   │  of a request?"     │      │
│  │             │   │             │   │                     │      │
│  │ Numeric     │   │ Text events │   │ Distributed call    │      │
│  │ time-series │   │ with        │   │ graphs across       │      │
│  │ data        │   │ context     │   │ services            │      │
│  │             │   │             │   │                     │      │
│  │ Examples:   │   │ Examples:   │   │ Examples:           │      │
│  │ CPU 78%     │   │ "User 123   │   │ API GW → Cart Svc   │      │
│  │ Req/s: 5200 │   │  failed     │   │ → Order Svc         │      │
│  │ Latency p99:│   │  login:     │   │ → Payment Svc       │      │
│  │  450ms      │   │  invalid    │   │ → DB (slow: 2.3s)   │      │
│  │ Errors: 12  │   │  password"  │   │ → Notification Svc  │      │
│  │             │   │             │   │                     │      │
│  │ Tool:       │   │ Tool:       │   │ Tool:               │      │
│  │ Prometheus  │   │ Loki / ELK  │   │ Jaeger / Tempo      │      │
│  └──────┬──────┘   └──────┬──────┘   └──────────┬──────────┘      │
│         │                 │                      │                │
│         └─────────────────┼──────────────────────┘                │
│                           │                                       │
│                    ┌──────▼──────┐                                │
│                    │  GRAFANA    │   Unified visualization        │
│                    │  Dashboard  │   Metrics + Logs + Traces      │
│                    │             │   correlated by time and       │
│                    │             │   trace ID                     │
│                    └─────────────┘                                │
└───────────────────────────────────────────────────────────────────┘
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
│                    PROMETHEUS METRIC TYPES                         │
│                                                                    │
│  ┌───────────┐  ┌───────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │  COUNTER  │  │   GAUGE   │  │  HISTOGRAM  │  │   SUMMARY    │   │
│  │           │  │           │  │             │  │              │   │
│  │ Only goes │  │ Goes up   │  │ Buckets of  │  │ Client-side  │   │
│  │ UP (or    │  │ AND down  │  │ observations│  │ quantiles    │   │
│  │ resets    │  │           │  │             │  │              │   │
│  │ to 0)     │  │           │  │ Server-side │  │ NOT          │   │
│  │           │  │           │  │ quantile    │  │ aggregatable │   │
│  │           │  │           │  │ calculation │  │              │   │
│  └───────────┘  └───────────┘  └─────────────┘  └──────────────┘   │
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

┌─────────────────────────────────────────────────────────────┐
│              kube-prometheus-stack                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │ Prometheus   │  │ Alertmanager │  │ Grafana            │ │
│  │ Operator     │  │ (HA: 3       │  │ + pre-built        │ │
│  │              │  │  replicas)   │  │   dashboards       │ │
│  │ Manages      │  │              │  │                    │ │
│  │ Prometheus   │  │ Routes       │  │ K8s, node,         │ │
│  │ instances    │  │ alerts to    │  │ namespace,         │ │
│  │ via CRDs     │  │ PagerDuty/   │  │ workload           │ │
│  │              │  │ Slack        │  │ dashboards         │ │
│  └──────────────┘  └──────────────┘  └────────────────────┘ │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ node-        │  │ kube-state-  │  │ Prometheus       │   │
│  │ exporter     │  │ metrics      │  │ (2 replicas      │   │
│  │              │  │              │  │  or Thanos       │   │
│  │ DaemonSet    │  │ Deployment   │  │  sidecar for HA) │   │
│  │ on every     │  │              │  │                  │   │
│  │ node         │  │ Reads K8s    │  │ TSDB + WAL       │   │
│  │              │  │ API, exposes │  │ persistent       │   │
│  │ Exposes:     │  │ as metrics   │  │ storage (EBS)    │   │ 
│  │ CPU, mem,    │  │              │  │                  │   │
│  │ disk, net    │  │ Exposes:     │  │                  │   │
│  │ per NODE     │  │ pod status,  │  │                  │   │
│  │              │  │ deployment   │  │                  │   │
│  │              │  │ replicas,    │  │                  │   │
│  │              │  │ job status,  │  │                  │   │
│  │              │  │ PVC, etc.    │  │                  │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘

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
│                  LAYERED SOLUTION                           │
├─────────────────────────────────────────────────────────────┤
│ Layer 1: STORAGE                                            │
│  - Use gp3 EBS with provisioned IOPS (3000+)                │
│  - Or use local NVMe (i3/i4 instances) for TSDB             │
│  - Set --storage.tsdb.min-block-duration=2h                 │
│  - Set --storage.tsdb.max-block-duration=2h (prevents       │
│    mega-compactions that block ingestion)                   │
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
                    │  │   Data Source Layer   │   │
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
│  [variables dropdown row]                                   │
├──────────────────────┬──────────────────────────────────────┤
│  REQUEST RATE        │  ERROR RATE                          │
│  (Traffic)           │  (Errors)                            │
│  ┌──────────────┐    │  ┌────────────────────┐              │
│  │  ▅▆▇█▇▆▅▆▇ │  │    ▁▁▂▁▁▅▁▁▁     │              │
│  └──────────────┘    │  └────────────────────┘              │
├──────────────────────┼──────────────────────────────────────┤
│  P50/P90/P99 LATENCY│  SATURATION                           │
│  (Duration)          │  (CPU/Mem/Disk)                      │
│  ┌──────────────┐    │  ┌──────────────┐                    │
│  │  ───P99      │    │  │  CPU ▇▇▆   │                    │
│  │  ───P90      │    │  │  Mem ▅▅▅   │                    │
│  │  ───P50      │    │  │  Disk▃▃▃   │                    │
│  └──────────────┘    │  └──────────────┘                    │
├─────────────────────────────────────────────────────────────┤
│  K8s STATUS: Pods Ready/Desired, Restarts, HPA %, PDB       │
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
                           ┌──────────────────┐
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
┌──────────┐            │  │   (validation,    │    validates labels,
│ Fluent   │──Push──▶   │  │    rate limit)    │    distributes to ingesters
│ Bit      │            │  └────────┬───────────┘ │
└──────────┘            │           │             │
                        │  ┌────────▼──────────┐  │
                        │  │    Ingester       │  │ Builds chunks in memory
                        │  │   (in-memory      │  │ Flushes to storage
                        │  │    chunks, WAL)   │  │ (similar to Prometheus head block)
                        │  └────────┬──────────┘  │
                        │           │ Flush       │
                        │  ┌────────▼──────────┐  │
                        │  │  Object Storage   │  │ S3, GCS, or MinIO
                        │  │  (chunks + index) │  │
                        │  └──────────────────┘   │
                        │                         │
                        │  ┌──────────────────┐   │
                        │  │  Query Frontend   │  │ Splits/caches queries
                        │  │     + Querier     │  │ Reads from ingesters + storage
                        │  └──────────────────┘   │
                        └─────────────────────────┘
```

### Deployment Modes

```
┌─────────────┬──────────────────────────────────────────────────────────┐
│ Mode        │ When to Use                                              │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Monolithic  │ Dev/test, <100GB/day. Single binary, all components.     │
│             │ helm: deploymentMode: SingleBinary                       │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Simple      │ Small-medium prod, 100-500GB/day. Read + Write + Backend │
│ Scalable    │ helm: deploymentMode: SimpleScalable (3 components)      │
│ (SSD)       │ NovaMart uses this. Best balance of simplicity/scale.    │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Microservice│ Large scale, >500GB/day. Each component separate.        │
│             │ Maximum flexibility, maximum operational complexity.     │
│             │ helm: deploymentMode: Distributed                        │
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
│ Protocol      │ Loki push API       │ Many outputs     │ OTLP + many       │
│ Resource use  │ Low                 │ Very low (C)     │ Medium (Go)       │
│ K8s metadata  │ Auto (K8s SD)       │ kubernetes filter│ k8sattributes     │
│ Log parsing   │ Pipeline stages     │ Parsers/filters  │ Processors        │
│ Multi-output  │ Loki only           │ Yes              │ Yes               │
│ NovaMart pick │ ✅ Primary (Loki)   │ Legacy ELK path  │ Future direction │
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
│ {app="order-svc"} │     │ |= "error" | json | line_format ...  │
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
│ Downsampling     │ Yes (5m, 1h auto)   │ No (not needed —   │ No                │
│                  │                     │ query optimization)│                   │
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
│ ┌────────┐ ┌────────┐  │             │ ┌────────┐ ┌────────┐  │
│ │  TSDB  │ │ Thanos │  │             │ │  TSDB  │ │ Thanos │  │
│ │  Head  │ │Sidecar │──┼─upload──┐   │ │  Head  │ │Sidecar │──┼─upload──┐
│ │  Block │ │        │  │   │     │   │ │  Block │ │        │  │   │     │
│ └────────┘ └───┬────┘  │   │     │   │ └────────┘ └───┬────┘  │   │     │
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
                    └──────────────┬───────────────────────┘
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
                    ┌──────────────▼────────────────┐
                    │    Thanos Query Frontend      │
                    │  - Query splitting            │
                    │  - Results caching (memcached)│
                    │  - Retry / limits             │
                    └──────────────┬────────────────┘
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
│  Prometheus TSDB ──── 15 days local (EBS gp3)                       │
│  Loki Ingesters ───── few hours in-memory + WAL                     │
│  Tempo Ingesters ──── few hours in-memory + WAL                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                        LONG-TERM STORAGE                            │
│                                                                     │
│  Thanos Sidecar ──upload──▶ S3 (novamart-thanos-metrics)           │
│  │  Compactor ────── compaction + downsampling + retention          │
│  │  Store GW ─────── serves old data for queries                    │
│  │                                                                  │
│  Loki ────flush───▶ S3 (novamart-loki-chunks)                      │
│  │  Compactor ────── index compaction + retention                   │
│  │                                                                  │
│  Tempo ───flush───▶ S3 (novamart-tempo-traces)                     │
│  │  Compactor ────── block compaction + retention                   │
│  │                                                                  │
│  Retention: Metrics=90d, Logs=30d, Traces=15d                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                        QUERY / VISUALIZATION                        │
│                                                                     │
│  Thanos Query ───▶ Grafana (metrics)                               │
│  Loki Query ─────▶ Grafana (logs)                                  │
│  Tempo Query ────▶ Grafana (traces)                                │
│                                                                     │
│  Correlation: metrics ←→ logs ←→ traces (via labels + traceID)      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                        ALERTING                                     │
│                                                                     │
│  Prometheus ──rules──▶ Alertmanager ──route──▶ PagerDuty (SEV1)    │
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
│    ingestion rate, chunk utilization, per-tenant usage       │
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
│  - CI validates dashboard JSON using grafonnet or           │
│    grafana-dashboard-linter                                 │
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
│          disableDeletion: true                              │
│          allowUiUpdates: false                              │
│          options:                                           │
│            path: /var/lib/grafana/dashboards                │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: VARIABLE DESIGN BEST PRACTICES                     │
│  - Always set a valid default (not "ALL" unless queries     │
│    use regex matchers)                                      │
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
│ Root Span   │ The first span in a trace (no parent). Usually the     │
│             │ entry point (API gateway, load balancer).              │
├─────────────┼────────────────────────────────────────────────────────┤
│ Child Span  │ An operation called by another span. Parent-child      │
│             │ relationships form the trace tree.                     │
├─────────────┼────────────────────────────────────────────────────────┤
│ Span Context│ The propagation payload: Trace ID + Span ID + flags.   │
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
┌───────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                              │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
│  │ order-svc   │  │ payment-svc │  │ cart-svc    │                │
│  │ (Java)      │  │ (Go)        │  │ (Python)    │                │
│  │             │  │             │  │             │                │
│  │ OTel Java   │  │ OTel Go SDK │  │ OTel Python │                │
│  │ Agent (auto)│  │ (manual)    │  │ (auto)      │                │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                │
│         │                │                │                       │
│         │    OTLP/gRPC   │   OTLP/gRPC    │   OTLP/gRPC           │
│         └────────────────┼────────────────┘                       │
│                          │                                        │
└──────────────────────────┼────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    OTEL COLLECTOR                                │
│                                                                  │
│  ┌───────────┐    ┌────────────┐    ┌───────────┐                │
│  │ Receivers │───▶│ Processors │───▶│ Exporters │               │
│  │           │    │            │    │           │                │
│  │ - otlp    │    │ - batch    │    │ - otlp    │──▶ Tempo       │
│  │ - jaeger  │    │ - memory   │    │ - prom    │──▶ Prometheus  │
│  │ - zipkin  │    │   limiter  │    │   remote  │                │
│  │ - prom    │    │ - filter   │    │   write   │                │
│  │           │    │ - tail     │    │ - loki    │──▶ Loki       │
│  │           │    │   sampling │    │           │                │
│  └───────────┘    │ - resource │    └───────────┘                │
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
│ Pattern 1: DaemonSet (Agent)                                           │
│                                                                        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐     One collector per node        │
│  │ App Pod │ │ App Pod │ │ App Pod │     - Low latency (localhost)     │
│  └────┬────┘ └────┬────┘ └────┬────┘     - Node-level resource bound   │
│       │           │           │          - Good for high-volume        │
│       └───────────┼───────────┘                                        │
│                   ▼                                                    │
│         ┌─────────────────┐                                            │
│         │ OTel Collector  │ (DaemonSet — one per node)                 │
│         │ (Agent mode)    │                                            │
│         └────────┬────────┘                                            │
│                  │                                                     │
│                  ▼                                                     │
│         ┌─────────────────┐                                            │
│         │ OTel Collector  │ (Deployment — centralized, optional)       │
│         │ (Gateway mode)  │ - Tail sampling                            │ 
│         │                 │ - Complex processing                       │ 
│         └─────────────────┘                                            │
├────────────────────────────────────────────────────────────────────────┤
│ Pattern 2: Sidecar                                                     │
│                                                                        │
│  ┌──────────────────────────────┐                                      │
│  │ Pod                          │  One collector per pod               │
│  │ ┌──────────┐ ┌────────────┐ │  - Maximum isolation                  │
│  │ │ App      │ │ OTel       │ │  - Higher resource usage              │
│  │ │ Container│→│ Collector  │ │  - Used when apps need different      │
│  │ │          │ │ (sidecar)  │ │    collector configs                  │
│  │ └──────────┘ └────────────┘ │                                       │
│  └──────────────────────────────┘                                      │
├────────────────────────────────────────────────────────────────────────┤
│ Pattern 3: DaemonSet Agent + Gateway (NovaMart's choice)               │
│                                                                        │
│  Apps ──OTLP──▶ Agent (DaemonSet) ──OTLP──▶ Gateway (Deployment)      │
│                 - k8s enrichment             - Tail sampling           │
│                 - Basic filtering             - Batching               │
│                 - Memory limiting             - Export to backends     │
│                                                                        │
│  Why: Tail sampling REQUIRES seeing all spans of a trace in one        │
│  place. DaemonSet agents see spans per-node. Gateway sees all spans.   │
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
┌───────────────────────────────────────────────────────────────────┐
│                    HEAD SAMPLING                                  │
│                                                                   │
│  Decision made AT THE START of the trace (before any spans)       │
│                                                                   │
│  ┌──────┐                                                         │
│  │ Dice │──▶ 10% → Trace sampled (all spans collected)           │
│  │ Roll │──▶ 90% → Trace NOT sampled (no spans collected)        │
│  └──────┘                                                         │
│                                                                   │
│  Pros: Simple, low resource usage, no state needed                │
│  Cons: Might miss all error/slow traces (they're in the 90%)      │
│                                                                   │
│  Implementation: SDK-side (sampler config)                        │
│    sampler:                                                       │
│      type: parentbased_traceidratio                               │
│      argument: "0.1"   # 10%                                      │
│                                                                   │
│  parentbased = if parent was sampled, child is sampled too        │
│  (prevents partial traces)                                        │
├───────────────────────────────────────────────────────────────────┤
│                    TAIL SAMPLING                                  │
│                                                                   │
│  Decision made AFTER the trace is complete (all spans available)  │
│                                                                   │
│  ┌──────────┐                                                     │
│  │ Collect  │──▶ Analyze complete trace:                         │
│  │ ALL      │    - Has errors? → KEEP (100%)                      │
│  │ spans    │    - Duration > 2s? → KEEP (100%)                   │
│  │ first    │    - payment-svc? → KEEP (100%)                     │
│  │          │    - Normal trace? → KEEP (10%)                     │
│  └──────────┘                                                     │
│                                                                   │
│  Pros: Intelligent — keeps ALL interesting traces                 │
│  Cons: MUST see all spans of a trace in one place                 │
│        Higher memory (buffering traces until decision)            │
│        decision_wait latency before export                        │
│                                                                   │
│  Implementation: Collector-side (tail_sampling processor)         │
│  ⚠️ Requires Gateway pattern — one collector sees all spans      │
└───────────────────────────────────────────────────────────────────┘
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
┌───────────────────────────────────────────────────────────────────┐
│                    EXEMPLARS (Metrics → Traces)                   │
│                                                                   │
│  Prometheus metric sample carries a trace ID as exemplar:         │
│                                                                   │
│  http_request_duration_seconds_bucket{le="1.0"} 1532              │
│  # Exemplar: {traceID="abc123"} 0.95 1704067200                   │
│                                                                   │
│  In Grafana: hover over a histogram → see exemplar dots →         │
│  click → jump to trace in Tempo                                   │
│                                                                   │
│  Requirement: app must emit exemplars (OTel SDK does this         │
│  automatically when histogram metrics are linked to active spans) │
│                                                                   │
│  Prometheus config:                                               │
│    --enable-feature=exemplar-storage                              │
│    (enabled by default in recent versions)                        │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│                    LOG → TRACE CORRELATION                        │
│                                                                   │
│  Application logs include traceID field:                          │
│  {"level":"error","msg":"payment failed","traceID":"abc123"}      │
│                                                                   │
│  Grafana Loki datasource config (derivedFields):                  │
│  - matcherRegex: '"traceID":"(\w+)"'                              │
│    datasourceUid: tempo                                           │
│                                                                   │
│  In Grafana: view log line → click traceID → jump to Tempo        │
│                                                                   │
│  Requirement: Application must inject traceID into log context    │
│  (OTel SDK + logging bridge does this automatically)              │
└───────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    TRACE → LOGS CORRELATION                      │
│                                                                  │
│  Tempo datasource config (tracesToLogs):                         │
│  - datasourceUid: loki                                           │
│    tags: ['service.name']                                        │
│    mappedTags: [{key: 'service.name', value: 'app'}]             │
│    filterByTraceID: true                                         │
│                                                                  │
│  In Grafana: view trace → click "Logs for this span" →           │
│  jump to Loki filtered by traceID and time range                 │
└──────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│                    TRACE → METRICS CORRELATION                    │
│                                                                   │
│  Tempo datasource config (tracesToMetrics):                       │
│  - datasourceUid: prometheus                                      │
│    tags: [{key: 'service.name', value: 'service'}]                │
│                                                                   │
│  In Grafana: view trace → click "Metrics for this service" →      │
│  jump to Prometheus dashboard for that service + time range       │
└───────────────────────────────────────────────────────────────────┘
```

### The Complete Incident Investigation Flow

```
INCIDENT: order-svc p99 > 5s SLO breach at 14:32

STEP 1: METRICS (Grafana → Prometheus/Thanos)
┌───────────────────────────────────────────────────────────┐
│ Dashboard: Golden Signals - order-svc                     │
│                                                           │
│ PromQL: histogram_quantile(0.99,                          │
│   sum by (le)(rate(http_request_duration_seconds_bucket   │
│   {service="order-svc"}[$__rate_interval])))              │
│                                                           │
│ Result: p99 jumped from 200ms to 5.2s at 14:32            │
│ Notice: Exemplar dot at 5.1s with traceID=abc123          │
│                                                           │
│ Action: Click exemplar → opens trace in Tempo             │
└───────────────────────────────────────────────────────────┘
                              │
                              ▼
STEP 2: TRACE (Grafana → Tempo)
┌───────────────────────────────────────────────────────────┐
│ Trace ID: abc123                                          │
│                                                           │
│ ├── API Gateway (8ms)                                     │
│ ├── order-svc: PlaceOrder (5100ms)                        │
│ │   ├── Inventory check (42ms) ✅                        │
│ │   ├── payment-svc: ProcessPayment (4950ms) ← SLOW!      │
│ │   │   ├── Stripe API call (120ms) ✅                   │
│ │   │   └── fraud-svc: CheckFraud (4800ms) ← ROOT CAUSE  │
│ │   │       └── ml-model-svc: Predict (4750ms) ← HERE    │
│ │   │           span.attributes:                         │
│ │   │             model.version = "v3.2.1"               │
│ │   │             model.cold_start = true ← SMOKING GUN  │
│ │   └── notification-svc (80ms) ✅                       │
│                                                          │
│ Action: Click "Logs for this span" on ml-model-svc       │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
STEP 3: LOGS (Grafana → Loki)
┌──────────────────────────────────────────────────────────┐
│ Auto-generated query:                                    │
│ {app="ml-model-svc"} |= "abc123"                         │
│                                                          │
│ Results:                                                 │
│ 14:32:01 INFO  Loading model v3.2.1 from S3...           │
│ 14:32:03 WARN  Model cache miss, downloading 2.1GB       │
│ 14:32:05 INFO  Model loaded, inference starting          │
│ 14:32:05 INFO  Prediction complete: fraud_score=0.02     │
│                                                          │
│ ROOT CAUSE: ML model cold start (cache eviction)         │
│ Model download from S3 took 4.7 seconds                  │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
STEP 4: FIX
┌──────────────────────────────────────────────────────────┐
│ Immediate: Pre-warm model cache, increase cache TTL      │
│ Long-term: Model sidecar pattern with startup probe      │
│            PodReadinessGate to prevent traffic before    │
│            model is loaded                               │
│                                                          │
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
  ┌──────────┐   OTLP/gRPC      │         OTel Agent (DaemonSet)       │
  │ app pods │ ──────────────→  │  receivers → processors → exporters  │
  └──────────┘                  │            (per node)                │
  ┌──────────┐   Jaeger/thrift  │                                      │
  │ legacy   │ ──────────────→  │  Accepts OTLP + Jaeger               │
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
                    ┌───────────────────────┐
                    │  PagerDuty Alert      │
                    │  P99 Latency > SLO    │
                    └──────────┬────────────┘
                               │
                    ┌──────────▼─────────────┐
                    │  METRICS (Prometheus)  │
              ┌─────│  Which endpoint?       │
              │     │  Which pods?           │
              │     │  When did it start?    │
              │     └──────────┬─────────────┘
              │                │ service=payment-svc
              │                │ endpoint=/charge
              │                │ time window identified
              │     ┌──────────▼─────────────┐
              │     │  LOGS (Loki)           │
              │     │  Error messages?       │
              │     │  Slow request logs?    │
              │     │  Extract traceID       │
              │     └──────────┬─────────────┘
              │                │ traceID=abc123
              │     ┌──────────▼─────────────┐
              │     │  TRACES (Tempo)        │
              │     │  Waterfall view        │
              │     │  Identify bottleneck   │
              │     │  span: stripe-api 7.5s │
              │     └──────────┬─────────────┘
              │                │ Stripe says fine
              │     ┌──────────▼─────────────┐
              └────→│  METRICS (Prometheus)  │
                    │  Connection pool?      │
                    │  DNS latency?          │
                    │  NAT GW port alloc?    │
                    └──────────┬─────────────┘
                               │
                    ┌──────────▼─────────────┐
                    │  AWS CloudWatch        │
                    │  NAT GW metrics        │
                    │  ErrorPortAllocation   │
                    └──────────┬─────────────┘
                               │
                    ┌──────────▼─────────────┐
                    │  ROOT CAUSE:           │
                    │  Connection pool       │
                    │  MaxIdleConnsPerHost=2 │
                    │  + NAT GW port exhaust │
                    └────────────────────────┘
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
│          │ Format: ratio of good events / total events                │
│          │ Example: 99.2% of HTTP requests returned in < 300ms        │
│          │                                                            │
│          │ SLIs are ALWAYS expressed as proportions (0-100%)          │
│          │ NEVER as raw numbers (not "200ms p99")                     │
├──────────┼────────────────────────────────────────────────────────────┤
│ SLO      │ Service Level OBJECTIVE                                    │
│          │ A TARGET value for an SLI over a time window.              │
│          │ "What's our reliability target?"                           │
│          │                                                            │
│          │ Format: SLI ≥ target% over rolling window                  │
│          │ Example: 99.9% of requests succeed within 300ms            │
│          │          measured over a 30-day rolling window             │
│          │                                                            │
│          │ SLOs are INTERNAL targets — chosen by engineering          │
│          │ They should be TIGHTER than SLAs                          │
├──────────┼────────────────────────────────────────────────────────────┤
│ SLA      │ Service Level AGREEMENT                                    │
│          │ A CONTRACT with customers. Legal/financial consequences    │
│          │ if breached. Typically looser than SLOs.                   │
│          │                                                            │
│          │ Example: "99.9% uptime or customer gets credits"           │
│          │                                                            │
│          │ SLA ≤ SLO (always) — SLO is the internal safety margin     │
│          │ If SLO = 99.95% and SLA = 99.9%, you have a 0.05%          │
│          │ buffer before you owe customers money                      │
├──────────┼────────────────────────────────────────────────────────────┤
│ Error    │ The amount of unreliability you're ALLOWED to have.        │
│ Budget   │                                                            │
│          │ Error Budget = 1 - SLO target                              │
│          │                                                            │
│          │ If SLO = 99.9%, Error Budget = 0.1%                        │
│          │ Over 30 days: 0.1% × 30 × 24 × 60 = 43.2 minutes           │
│          │                                                            │
│          │ You can "spend" this budget on:                            │
│          │   - Planned maintenance                                    │
│          │   - Risky deployments                                      │
│          │   - Incidents/outages                                      │
│          │                                                            │
│          │ When budget exhausted → STOP deploying features            │
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
│                  │ NOT "p99 < 300ms" — instead:                       │
│                  │ "99% of requests complete within 300ms"            │
│                  │                                                    │
│                  │ Can have multiple thresholds:                      │
│                  │   - 90% of requests < 100ms (fast)                 │
│                  │   - 99% of requests < 500ms (acceptable)           │
│                  │   - 99.9% of requests < 2s (max tolerable)         │
├──────────────────┼────────────────────────────────────────────────────┤
│ Freshness        │ data updated within threshold / total data points  │
│                  │ Use for: Data pipelines, caches, search indexes    │
│                  │                                                    │
│                  │ Example: "99.9% of product prices reflect changes  │
│                  │ within 5 minutes of update"                        │
├──────────────────┼────────────────────────────────────────────────────┤
│ Correctness      │ correct results / total results                    │
│                  │ Use for: Search results, recommendations, billing  │
│                  │                                                    │
│                  │ Example: "99.99% of charges match the displayed    │
│                  │ price at time of checkout"                         │
│                  │                                                    │
│                  │ ⚠️ Hardest to measure — often requires synthetic   │
│                  │ probes or end-to-end validation                    │
├──────────────────┼────────────────────────────────────────────────────┤
│ Throughput       │ actual throughput / expected throughput            │
│                  │ Use for: Batch processing, streaming systems       │
│                  │                                                    │
│                  │ Example: "99% of order events are processed        │
│                  │ within 30 seconds of creation"                     │
├──────────────────┼────────────────────────────────────────────────────┤
│ Durability       │ data retrievable / data stored                     │
│                  │ Use for: Storage systems, databases                │
│                  │                                                    │
│                  │ Example: "99.999999999% of objects stored in S3    │
│                  │ are retrievable" (AWS's own SLA: 11 nines)         │
└──────────────────┴────────────────────────────────────────────────────┘
```

---

## NovaMart SLOs — Real Implementation

### SLO Definitions for Key Services

```
┌────────────────────┬──────────────────────────────────────────────────────────┐
│ Service            │ SLOs                                                     │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ API Gateway        │ Availability: 99.95% of requests return non-5xx          │
│                    │ Latency: 99% of requests complete within 200ms           │
│                    │ Window: 30-day rolling                                   │
│                    │                                                          │
│                    │ Error budget: 0.05% = 21.6 min/month                     │
│                    │ At 2000 req/s = 2,592,000 failed requests/month          │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Payment Service    │ Availability: 99.99% of charge requests succeed          │
│                    │ Latency: 99.9% of requests complete within 2s            │
│                    │ Correctness: 99.999% of charges are for correct amount   │
│                    │ Window: 30-day rolling                                   │
│                    │                                                          │
│                    │ Error budget: 0.01% = 4.32 min/month                     │
│                    │ ⚠️ Tightest SLO — money is on the line                  │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Search Service     │ Availability: 99.9% of queries return results            │
│                    │ Latency: 95% < 200ms, 99% < 1s                           │
│                    │ Freshness: 99.9% of products searchable within 5 min     │
│                    │            of catalog update                             │
│                    │ Window: 30-day rolling                                   │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Order Service      │ Availability: 99.95% of order submissions succeed        │
│                    │ Latency: 99% complete within 3s (complex workflow)       │
│                    │ Correctness: 99.99% of orders match cart contents        │
│                    │ Window: 30-day rolling                                   │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Notification Svc   │ Availability: 99.9% of notifications sent                │
│                    │ Freshness: 99% of notifications delivered within 60s     │
│                    │ Window: 30-day rolling                                   │
│                    │                                                          │
│                    │ ⚠️ Looser SLO — notifications are important but not      │
│                    │ transactional. Users tolerate email 60s late.            │
├────────────────────┼──────────────────────────────────────────────────────────┤
│ Data Pipeline      │ Freshness: 99.9% of analytics data < 15 min stale        │
│ (Airflow)          │ Completeness: 99.9% of scheduled jobs complete           │
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
│  consumed in   │             │ If 14.4x for    │ AND 14.4x in last 5min     │
│  1 hour)       │             │ 1 hour → 2%     │ (still burning = not       │
│                │             │ budget gone      │  a brief spike that ended)│
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
│                │             │ exhaust budget  │                            │
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
│   ██████████░░░░░░   │   ┌───────────────────────────────────────┐       │
│   68% remaining      │   │  ▂▃▂▂▃▂▂▂▂▂▅▇█▇▅▃▃▂▂▂ │        │
│                      │   │  ─── 14.4x page threshold            │        │
│   Budget: 30d window │   │  ─── 6x page threshold               │        │
│   Consumed: 32%      │   │  ─── 1x expected rate                │        │
│                      │   └──────────────────────────────────────┘        │
├──────────────────────┼───────────────────────────────────────────────────┤
│   AVAILABILITY SLI   │   LATENCY SLI                                     │
│   (over 30d window)  │   (over 30d window)                               │
│                      │                                                   │
│   Current: 99.967%   │   Current: 99.4% < 200ms                          │
│   Target:  99.950%   │   Target:  99.0% < 200ms                          │
│   Status:  ✅ MET    │   Status:  ✅ MET                                │
│                      │                                                   │
│   ┌──────────────┐   │   ┌──────────────┐                                │
│   │ ─────────    │   │   │ ─────────    │                                │
│   │   99.95 ─ ─ ─│   │   │   99.0 ─ ─ ─│                                 │
│   │ ─────────    │   │   │ ─────────    │                                │
│   └──────────────┘   │   └──────────────┘                                │
├──────────────────────┴───────────────────────────────────────────────────┤
│   ERROR BUDGET CONSUMPTION TIMELINE (30-day)                             │
│   ┌─────────────────────────────────────────────────────────────────┐    │
│   │         ╱                                                        │   │
│   │        ╱  Incident on day 12                                     │   │
│   │ ──────╱───── consumed 15% in 2 hours                             │   │
│   │      ╱                                                           │   │
│   │     ╱                                                            │   │
│   │────╱──── steady consumption from background errors               │   │
│   │   ╱                                                              │   │
│   │──╱                                                               │   │
│   └──────────────────────────────────────────────────────────────────┘   │
│   Day 1                                               Day 30             │
├──────────────────────────────────────────────────────────────────────────┤
│   RECENT INCIDENTS CONSUMING BUDGET                                      │
│   ┌─────────────────────────────────────────────────────────────┐        │
│   │ Day 12: Payment gateway timeout — consumed 15% budget       │        │
│   │ Day 18: Deploy regression — consumed 3% budget              │        │
│   │ Day 22: DNS resolution delay — consumed 1% budget           │        │
│   └─────────────────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────────────────┘
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


You investigate and find:
- The SLI is calculated as: `non-5xx responses / total responses`
- Search-svc is indeed returning `200 OK` for almost every request
- But many of those `200 OK` responses contain `{"results": [], "count": 0}` for queries that should return products

**Questions:**
1. Explain exactly why the current SLI is lying. What fundamental SLI design principle is being violated?
2. Design a **corrected set of SLIs** for search-svc that would have caught this. You need at least three SLIs covering different dimensions of search quality. For each, give the metric name, what it measures, and how you'd implement it in Prometheus.
3. The search team says "we can't add a correctness SLI because we don't know what the 'correct' result is for every query." How do you solve this? Give two practical approaches.
4. Write the PromQL for an SLI that measures "proportion of search requests that return at least one result, excluding queries for non-existent products." Assume the app exposes `search_requests_total{status="200", result_count_bucket="..."}`.

---

### Q4: Error Budget Policy Enforcement 🔥

**Scenario:** It's day 18 of the 30-day SLO window. Payment-svc's error budget situation:

```
SLO: 99.99% availability
Error budget (30 days): 4.32 minutes
Budget consumed so far: 3.8 minutes (from a SEV1 incident on day 5)
Budget remaining: 0.52 minutes (12% remaining)
```

The payments team has a **critical feature launch** scheduled for tomorrow — a new payment provider integration that touches the core charge path. The feature was promised to the CEO for a partnership announcement.

The payments tech lead comes to you: "We need to deploy this tomorrow. The CEO committed to the partner. Can you approve it?"

**Questions:**
1. According to NovaMart's error budget policy, what should happen? Be specific about which policy tier applies.
2. The tech lead argues: "The feature has been tested thoroughly in staging. It's low risk." What are the **three specific counter-arguments** you'd make, grounded in SRE principles?
3. What would you propose as a compromise that respects both the error budget policy AND the business commitment? Be specific about the deployment strategy and safeguards.
4. After the conversation, the VP of Engineering overrides the error budget policy and orders the deploy. What do you do? (This tests organizational maturity, not just technical knowledge.)

---
# Phase 5, Lesson 4 — SLO/SLI/Error Budget Answers

---

## Q1: SLO Design Under Constraints

### 1. Error Budget Calculation: 99.999% Over 30 Days

```
30-day window = 30 × 24 × 60 = 43,200 minutes

Error budget = (1 - SLO target) × window
             = (1 - 0.99999) × 43,200
             = 0.00001 × 43,200
             = 0.432 minutes
             = 25.92 seconds

Per day:  25.92 / 30 = 0.864 seconds/day
Per week: 25.92 / 4.29 = 6.04 seconds/week
```

**25.92 seconds of total downtime per month.** A single slow deployment rollout, a DNS TTL refresh, or one pod OOM-kill-and-restart cycle would consume the entire budget.

For context, here's the tier comparison:

```
┌───────────┬──────────────────┬───────────────────────────────────┐
│ SLO       │ Budget (30 days) │ Real-world meaning                │
├───────────┼──────────────────┼───────────────────────────────────┤
│ 99.9%     │ 43.2 minutes     │ One moderate incident per month   │
│ 99.95%    │ 21.6 minutes     │ One short incident per month      │
│ 99.99%    │ 4.32 minutes     │ Almost zero tolerance             │
│ 99.999%   │ 25.92 seconds    │ Theoretical — requires redundancy │
│           │                  │ at EVERY layer including physics  │
└───────────┴──────────────────┴───────────────────────────────────┘
```

### 2. Why 99.999% Is the Wrong Target — Three Technical Constraints

**Framed for the VP (non-technical but concrete):**

**Constraint 1: Our cloud provider doesn't guarantee it, so we mathematically can't either.**

AWS's own SLAs for the services `payment-svc` depends on:

```
┌──────────────────────┬────────────┬──────────────────────────┐
│ AWS Service          │ SLA        │ Max budget (30 days)     │
├──────────────────────┼────────────┼──────────────────────────┤
│ EC2 / EKS            │ 99.99%     │ 4.32 minutes             │
│ ALB                  │ 99.99%     │ 4.32 minutes             │
│ RDS Multi-AZ         │ 99.95%     │ 21.6 minutes             │
│ DynamoDB             │ 99.999%    │ 25.92 seconds            │
│ Route 53             │ 100%       │ 0 (aspirational)         │
│ NAT Gateway          │ 99.95%     │ 21.6 minutes             │
└──────────────────────┴────────────┴──────────────────────────┘
```

The payment request path touches ALL of these. Availability of a **chain** of dependencies is multiplicative, not additive:

```
Combined availability ≤ EKS × ALB × RDS × NAT GW
                     ≤ 0.9999 × 0.9999 × 0.9995 × 0.9995
                     = 0.9988 (99.88%)
```

**Our infrastructure's theoretical ceiling is ~99.88% before we write a single line of application code.** Promising 99.999% on top of a foundation that can't deliver 99.9% is like guaranteeing a car will go 300mph with a 150mph engine.

**Constraint 2: Every deployment is a risk event, and we deploy frequently.**

NovaMart deploys `payment-svc` approximately 2-4 times per week. Each deployment involves:

- Rolling update: old pods terminate, new pods start (~10-30 seconds of partial capacity)
- Health check grace period: new pods pass readiness probes (~5-15 seconds)
- Connection draining: in-flight requests to old pods complete (~5-10 seconds)

Even with zero-downtime deployment techniques, there are edge-case request failures during rollouts. A conservative estimate: **2-5 failed requests per deployment** from connection races, readiness probe timing, and load balancer target registration delays.

At 4 deploys/week × 4 weeks × 3 failed requests × 200ms average latency = not much in absolute terms, but against a 25.92-second budget, **each deployment consumes a measurable fraction of the budget.** We'd need to either stop deploying (unacceptable for velocity) or guarantee literally zero failed requests during rollouts (impossible).

**Constraint 3: We depend on Stripe, which doesn't offer 99.999%.**

The payment charge path makes a synchronous HTTP call to Stripe's API. Stripe's published availability is approximately 99.99% (their status page history shows multiple incidents per quarter). We cannot be more available than our slowest synchronous dependency.

```
payment-svc availability ≤ min(our_infra, stripe_api)
                         ≤ min(99.88%, 99.99%)
                         = 99.88%
```

**To actually achieve 99.999%, we would need:**
- Multi-provider payment routing (Stripe + Adyen + Braintree) with automatic failover — 6-12 months of engineering
- Multi-region active-active with global load balancing — we have multi-region but not active-active for payments
- Zero-downtime deployment with proven zero-error rollouts — requires extensive canary infrastructure
- Chaos engineering validation of every failure mode — months of game days

**Estimated cost: $2-5M in engineering and infrastructure over 12-18 months.** And even then, 99.999% is aspirational, not guaranteed.

### 3. Recommended SLO

**Recommendation: 99.95% availability SLO for payment-svc charge endpoint.**

**Justification:**

```
Current measured availability (90 days): 99.97%
                                         ├── Means we're running 0.02% above 99.95%
                                         └── 0.07% below 99.99%

99.99% (4.32 min budget):
  - Current performance: 99.97% → we'd start ALREADY in violation
  - 90-day data shows ~12.96 min of downtime/month
  - 12.96 min > 4.32 min budget → we'd be breaching SLO every month
  - Setting a target we consistently fail is worse than having no target
  - Teams stop trusting the system, alerts become noise

99.95% (21.6 min budget):
  - Current performance: 99.97% → we have ~8.64 min of headroom
  - Enough budget for 1-2 moderate incidents per month
  - Tight enough that SEV1s are felt and drive improvement
  - Loose enough that normal deployments don't trigger budget alerts
  - Room for experimentation and velocity
  - Achievable with current architecture, doesn't require multi-provider
```

**The SRE principle:** An SLO should be set at the level where **customers start to notice and complain.** If NovaMart's payment success rate drops below 99.95%, users will see checkout failures and call support. Above 99.95%, the failure rate is low enough that most users succeed on retry.

**I'd structure it as two SLOs:**

```
SLO 1 — Availability:
  "99.95% of /charge requests return non-5xx responses over a 30-day window"
  Budget: 21.6 minutes

SLO 2 — Latency:
  "99.9% of /charge requests complete within 2 seconds over a 30-day window"
  Budget: 0.1% of requests can exceed 2s
```

**Growth path:** Start at 99.95%, measure for two quarters. If we consistently hit 99.98%+, tighten to 99.97%. Ratchet toward 99.99% only when the architecture supports it (multi-provider payments, active-active).

### 4. Prometheus Recording Rules

**Why recording rules:** SLI calculations run over long windows (30 days). Evaluating a `rate()` over 30 days at query time is prohibitively expensive. Recording rules pre-compute the components every evaluation interval (typically 30s or 1m) and store them as new time series.

```yaml
# File: slo-recording-rules.yaml
# Applied to Prometheus via PrometheusRule CR or rules file

groups:
  # ============================================================
  # SLI Component Recording Rules
  # These pre-compute the per-second rates that the SLO rules consume
  # ============================================================
  - name: payment-svc-sli-components
    interval: 30s
    rules:
      # ----- Availability SLI Components -----

      # Total requests per second to /charge endpoint
      - record: sli:payment_charge_requests:rate5m
        expr: |
          sum(rate(http_requests_total{
            service="payment-svc",
            endpoint="/charge"
          }[5m]))

      # Failed requests per second (5xx only) to /charge endpoint
      - record: sli:payment_charge_errors:rate5m
        expr: |
          sum(rate(http_requests_total{
            service="payment-svc",
            endpoint="/charge",
            status=~"5.."
          }[5m]))

      # ----- Latency SLI Components -----

      # Requests completing within 2 seconds (using histogram bucket)
      # The le="2" bucket counts all requests with duration ≤ 2s
      - record: sli:payment_charge_duration_le2s:rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{
            service="payment-svc",
            endpoint="/charge",
            le="2"
          }[5m]))

      # Total requests (using +Inf bucket = total count)
      - record: sli:payment_charge_duration_total:rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{
            service="payment-svc",
            endpoint="/charge",
            le="+Inf"
          }[5m]))

  # ============================================================
  # SLI Ratio Rules (the actual SLI values)
  # ============================================================
  - name: payment-svc-sli-ratios
    interval: 30s
    rules:
      # Availability SLI: proportion of successful requests
      # Value: 0.0 to 1.0 (1.0 = 100% available)
      - record: sli:payment_charge_availability:ratio_rate5m
        expr: |
          1 - (
            sli:payment_charge_errors:rate5m
            /
            sli:payment_charge_requests:rate5m
          )

      # Latency SLI: proportion of requests within 2s threshold
      # Value: 0.0 to 1.0 (1.0 = all requests under 2s)
      - record: sli:payment_charge_latency_2s:ratio_rate5m
        expr: |
          sli:payment_charge_duration_le2s:rate5m
          /
          sli:payment_charge_duration_total:rate5m

  # ============================================================
  # Error Budget Consumption Rules
  # ============================================================
  - name: payment-svc-error-budget
    interval: 1m
    rules:
      # Availability: What fraction of 30-day error budget is consumed?
      # Uses 30-day rolling window
      # Budget = 1 - SLO_target = 1 - 0.9995 = 0.0005
      - record: sli:payment_charge_availability:error_budget_remaining
        expr: |
          1 - (
            (
              1 - (
                sum(sum_over_time(sli:payment_charge_availability:ratio_rate5m[30d]))
                /
                count_over_time(sli:payment_charge_availability:ratio_rate5m[30d])
              )
            )
            /
            0.0005
          )
        # Result: 1.0 = full budget, 0.0 = budget exhausted, negative = SLO breached

      # Latency: What fraction of 30-day error budget is consumed?
      # Budget = 1 - SLO_target = 1 - 0.999 = 0.001
      - record: sli:payment_charge_latency_2s:error_budget_remaining
        expr: |
          1 - (
            (
              1 - (
                sum(sum_over_time(sli:payment_charge_latency_2s:ratio_rate5m[30d]))
                /
                count_over_time(sli:payment_charge_latency_2s:ratio_rate5m[30d])
              )
            )
            /
            0.001
          )

  # ============================================================
  # Burn Rate Rules (for multi-window alerting)
  # ============================================================
  - name: payment-svc-burn-rates
    interval: 30s
    rules:
      # --- Availability Burn Rates ---

      # 1-hour burn rate
      - record: sli:payment_charge_availability:burnrate1h
        expr: |
          1 - (
            sum(rate(http_requests_total{
              service="payment-svc",
              endpoint="/charge",
              status!~"5.."
            }[1h]))
            /
            sum(rate(http_requests_total{
              service="payment-svc",
              endpoint="/charge"
            }[1h]))
          )
          /
          0.0005
        # Result: 1.0 = consuming budget at exactly the sustainable rate
        #         14.4 = will exhaust 30-day budget in 2 days at this rate
        #         0.0 = no errors (not consuming budget)

      # 6-hour burn rate
      - record: sli:payment_charge_availability:burnrate6h
        expr: |
          1 - (
            sum(rate(http_requests_total{
              service="payment-svc",
              endpoint="/charge",
              status!~"5.."
            }[6h]))
            /
            sum(rate(http_requests_total{
              service="payment-svc",
              endpoint="/charge"
            }[6h]))
          )
          /
          0.0005

      # 1-day burn rate
      - record: sli:payment_charge_availability:burnrate1d
        expr: |
          1 - (
            sum(rate(http_requests_total{
              service="payment-svc",
              endpoint="/charge",
              status!~"5.."
            }[1d]))
            /
            sum(rate(http_requests_total{
              service="payment-svc",
              endpoint="/charge"
            }[1d]))
          )
          /
          0.0005

      # 3-day burn rate
      - record: sli:payment_charge_availability:burnrate3d
        expr: |
          1 - (
            sum(rate(http_requests_total{
              service="payment-svc",
              endpoint="/charge",
              status!~"5.."
            }[3d]))
            /
            sum(rate(http_requests_total{
              service="payment-svc",
              endpoint="/charge"
            }[3d]))
          )
          /
          0.0005
```

**Multi-window burn rate alerts (Google SRE book methodology):**

```yaml
groups:
  - name: payment-svc-slo-alerts
    rules:
      # ============================================================
      # FAST BURN — Pages on-call (severe, acute)
      # 14.4x burn rate = exhausts 30-day budget in 2 days
      # Short window (1h) catches onset, long window (1h) confirms persistence
      # ============================================================
      - alert: PaymentSvcHighBurnRate_Page
        expr: |
          sli:payment_charge_availability:burnrate1h > 14.4
          and
          sli:payment_charge_availability:burnrate6h > 6
        for: 2m
        labels:
          severity: critical
          team: payments
          slo: payment-availability
        annotations:
          summary: "payment-svc /charge burning error budget at {{ $value }}x rate"
          description: |
            1h burn rate: {{ with query "sli:payment_charge_availability:burnrate1h" }}{{ . | first | value }}{{ end }}x
            6h burn rate: {{ with query "sli:payment_charge_availability:burnrate6h" }}{{ . | first | value }}{{ end }}x
            Budget remaining: {{ with query "sli:payment_charge_availability:error_budget_remaining" }}{{ . | first | value | humanizePercentage }}{{ end }}
          runbook: "https://wiki.novamart.internal/runbooks/payment-svc-slo-breach"

      # ============================================================
      # SLOW BURN — Creates ticket (degraded but not acute)
      # 1x burn rate sustained = exhausts budget exactly at window end
      # 3-day window avoids noise from transient spikes
      # ============================================================
      - alert: PaymentSvcSlowBurnRate_Ticket
        expr: |
          sli:payment_charge_availability:burnrate1d > 1
          and
          sli:payment_charge_availability:burnrate3d > 1
        for: 1h
        labels:
          severity: warning
          team: payments
          slo: payment-availability
        annotations:
          summary: "payment-svc /charge slow burn — budget will exhaust before window ends"
          description: |
            1d burn rate: {{ with query "sli:payment_charge_availability:burnrate1d" }}{{ . | first | value }}{{ end }}x
            3d burn rate: {{ with query "sli:payment_charge_availability:burnrate3d" }}{{ . | first | value }}{{ end }}x
            Budget remaining: {{ with query "sli:payment_charge_availability:error_budget_remaining" }}{{ . | first | value | humanizePercentage }}{{ end }}
          runbook: "https://wiki.novamart.internal/runbooks/payment-svc-slow-burn"

      # ============================================================
      # BUDGET EXHAUSTED — Triggers deployment freeze
      # ============================================================
      - alert: PaymentSvcErrorBudgetExhausted
        expr: |
          sli:payment_charge_availability:error_budget_remaining < 0
        for: 5m
        labels:
          severity: critical
          team: payments
          slo: payment-availability
          policy: deployment-freeze
        annotations:
          summary: "payment-svc /charge SLO BREACHED — error budget exhausted"
          description: "Error budget is negative. Deployment freeze in effect per SLO policy."
```

**Verification after applying:**

```bash
# Apply the rules
kubectl apply -f slo-recording-rules.yaml

# Wait for Prometheus to load the rules
sleep 30

# Verify rules are loaded and evaluating
curl -s http://prometheus:9090/api/v1/rules | jq '.data.groups[] | select(.name | contains("payment-svc")) | {name: .name, rules: [.rules[] | {name: (.name // .alert), health: .health, lastError: .lastError}]}'

# Verify recording rules are producing data
curl -s 'http://prometheus:9090/api/v1/query?query=sli:payment_charge_availability:ratio_rate5m' | jq '.data.result'
# Should return a value between 0.0 and 1.0

# Verify burn rate calculation
curl -s 'http://prometheus:9090/api/v1/query?query=sli:payment_charge_availability:burnrate1h' | jq '.data.result[0].value[1]'
# Should return a number (1.0 = exactly on budget pace)
```

---

## Q2: Burn Rate Alert Investigation

### 1. Is This Alert Actionable?

**Yes, this is actionable. Do NOT ignore it.**

Here's why, with math:

```
Error budget: 100% at start of 30-day window
Currently consumed: 62% (remaining: 38%)
Current day: assumed mid-window based on "3 days at 1x burn"
Burn rate: 1x (consuming budget at exactly the rate that exhausts it by day 30)

If we do nothing:
  - Budget hits 0% before the window ends
  - SLO is breached for the period
  - Per NovaMart's error budget policy, this triggers a deployment freeze
  - The payments team loses deploy velocity for the remainder of the window
```

**The slow burn alert exists specifically for this scenario.** It's not an outage — it's a *trend*. The whole point of SLO-based alerting is catching degradation before it becomes an outage or a breach.

**The correct response:**

```
Severity: WARNING (ticket, not page)
Timeline: Investigate within 1 business day
Goal:     Identify the source and stop the bleed within 48 hours
Escalate: If fix requires >48 hours, escalate to team lead for prioritization
```

The alternative — ignoring it — leads to budget exhaustion around day 28-30, triggering a deployment freeze at the worst possible time (month-end, when business pressure is highest).

### 2. PromQL Queries to Find WHERE Budget Is Being Consumed

**Query 1 — Error rate broken down by endpoint (find the hot path):**

```promql
sum by (endpoint) (
  rate(http_requests_total{
    service="order-svc",
    status=~"5.."
  }[6h])
)
/
sum by (endpoint) (
  rate(http_requests_total{
    service="order-svc"
  }[6h])
)
```

**What this tells you:** Which endpoint is contributing errors. If `/checkout` shows 0.5% error rate while `/search` shows 0.01%, you know where to focus.

**Query 2 — Error rate broken down by status code (5xx subtypes):**

```promql
sum by (status) (
  rate(http_requests_total{
    service="order-svc",
    status=~"5.."
  }[6h])
)
```

**What this tells you:** Are these `500 Internal Server Error` (application bugs), `502 Bad Gateway` (upstream crash), `503 Service Unavailable` (upstream overload/circuit breaker), or `504 Gateway Timeout` (upstream slow)?

**Query 3 — Error rate broken down by downstream dependency:**

```promql
sum by (upstream_service) (
  rate(http_requests_total{
    service="order-svc",
    status=~"5..",
    upstream_service!=""
  }[6h])
)
```

Or if using service mesh (Istio):

```promql
sum by (destination_service) (
  rate(istio_requests_total{
    source_workload="order-svc",
    response_code=~"5.."
  }[6h])
)
```

**Query 4 — Error pattern over time (find periodicity):**

```promql
sum(rate(http_requests_total{
  service="order-svc",
  status=~"5.."
}[5m]))
```

View this in Grafana with a 7-day window. Look for:
- Periodic spikes (cron jobs, scaling events)
- Step changes (deploy introduced a bug)
- Gradual increase (resource exhaustion)

**Query 5 — Errors correlated with specific pods (one bad pod?):**

```promql
sum by (pod) (
  rate(http_requests_total{
    service="order-svc",
    status=~"5.."
  }[1h])
)
```

### 3. Root Cause: 503s from inventory-svc, Bursts Every 4 Hours

**Pattern: 2-3 minutes of 503s every 4 hours from a single downstream dependency.**

The regularity (every 4 hours) eliminates random failures. This is a **scheduled event**. The three most likely causes, in order:

**Most likely: `inventory-svc` is performing a scheduled task that causes temporary unavailability.**

Specific candidates:

```
┌──────────────────────────────────────────────────────────────────┐
│ Candidate 1 (MOST LIKELY): CronJob / Scheduled Task              │
│                                                                  │
│ inventory-svc runs a stock reconciliation or cache rebuild       │
│ every 4 hours. During this operation:                            │
│ - Database connections are saturated by the batch query          │
│ - OR the service restarts as part of the job                     │
│ - OR the service enters a "maintenance mode" returning 503       │
│                                                                  │
│ Evidence to check:                                               │
│   kubectl -n inventory get cronjobs                              │
│   kubectl -n inventory get jobs --sort-by=.status.startTime      │
│   # Look for jobs running on a 4-hour schedule                   │
├──────────────────────────────────────────────────────────────────┤
│ Candidate 2: Kubernetes HPA scaling oscillation                  │
│                                                                  │
│ inventory-svc HPA scales down during low traffic, then a         │
│ periodic traffic spike (order processing batch) triggers         │
│ scale-up. During scale-up, new pods aren't ready yet → 503.      │
│                                                                  │
│ Evidence to check:                                               │
│   kubectl -n inventory get hpa inventory-svc -o yaml             │
│   kubectl -n inventory describe hpa inventory-svc                │
│   # Look at scaling events and cooldown periods                  │
├──────────────────────────────────────────────────────────────────┤
│ Candidate 3: Connection pool exhaustion                          │
│                                                                  │
│ inventory-svc's database connection pool fills up on a           │
│ 4-hour cycle (e.g., batch reporting queries from analytics).     │
│ While the pool is exhausted, health checks pass (separate        │
│ connection) but requests get 503 (no connections available).     │
│                                                                  │
│ Evidence to check:                                               │
│   # Query connection pool metrics                                │
│   inventory_db_connections_active                                │
│   inventory_db_connections_max                                   │
│   inventory_db_connections_wait_total                            │
└──────────────────────────────────────────────────────────────────┘
```

```bash
# Check for CronJobs on a 4h schedule
kubectl -n inventory get cronjobs -o custom-columns=\
NAME:.metadata.name,\
SCHEDULE:.spec.schedule,\
LAST:.status.lastScheduleTime

# Check recent job history
kubectl -n inventory get jobs --sort-by=.status.startTime | tail -10

# Correlate job times with error bursts
# In Grafana, overlay CronJob completions with error rate:
```

```promql
# Check if inventory-svc pod restarts correlate with error bursts
changes(kube_pod_container_status_restarts_total{
  namespace="inventory",
  container="inventory-svc"
}[5m])
```

### 4. Fix — Immediate and Permanent

**Immediate fix (stop the budget bleed within hours):**

**Add retry logic with backoff at the `order-svc` → `inventory-svc` call boundary:**

If `order-svc` doesn't already have a circuit breaker and retry:

```yaml
# If using Istio, apply a DestinationRule + retry policy
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: inventory-svc-resilience
  namespace: inventory
spec:
  host: inventory-svc.inventory.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        h2UpgradePolicy: DEFAULT
        maxRequestsPerConnection: 100
      tcp:
        maxConnections: 200
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-svc-retry
  namespace: inventory
spec:
  hosts:
    - inventory-svc.inventory.svc.cluster.local
  http:
    - route:
        - destination:
            host: inventory-svc.inventory.svc.cluster.local
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: "5xx,reset,connect-failure,retriable-4xx"
      timeout: 8s
```

```bash
kubectl apply -f inventory-svc-resilience.yaml
# Verify the policy is applied
kubectl -n inventory get destinationrule inventory-svc-resilience -o yaml
kubectl -n inventory get virtualservice inventory-svc-retry -o yaml
```

**Why this helps immediately:** The 2-3 minute `inventory-svc` unavailability windows cause ~0.02% error rate. With 3 retries and 30s outlier ejection, `order-svc` will:
1. Retry failed requests (catches transient errors)
2. Eject unhealthy `inventory-svc` pods from the load balancing pool
3. Route to healthy pods (if multiple replicas exist)

This should reduce the error contribution from 503s to near-zero within the first retry.

**If not using Istio, implement at the application level:**

```go
// In order-svc's HTTP client for inventory-svc calls
retryClient := retryablehttp.NewClient()
retryClient.RetryMax = 3
retryClient.RetryWaitMin = 100 * time.Millisecond
retryClient.RetryWaitMax = 2 * time.Second
retryClient.CheckRetry = func(ctx context.Context, resp *http.Response, err error) (bool, error) {
    if err != nil {
        return true, nil
    }
    if resp.StatusCode == 503 || resp.StatusCode == 502 {
        return true, nil
    }
    return false, nil
}
```

**Permanent fix (eliminate the root cause):**

**If CronJob is the cause:**

```yaml
# Option A: Move the batch job to a separate deployment that doesn't share
# the serving pods. inventory-svc should separate its "serve API requests"
# path from its "run batch reconciliation" path.

# Option B: If the job must run on the same service, implement graceful
# degradation — return stale cached data during reconciliation instead of 503
# This is an application-level change the inventory team must make.

# Option C: Adjust the job schedule to run during low-traffic windows
apiVersion: batch/v1
kind: CronJob
metadata:
  name: inventory-reconciliation
  namespace: inventory
spec:
  schedule: "0 3 * * *"  # Once daily at 3 AM instead of every 4 hours
  # Or if 4h is required:
  schedule: "0 */4 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: reconciler
              # Use a SEPARATE container/deployment, not the serving pods
              image: novamart/inventory-reconciler:latest
              resources:
                requests:
                  cpu: 500m
                  memory: 1Gi
```

**If connection pool exhaustion is the cause:**

```yaml
# Increase the connection pool and add connection timeout
# Application config for inventory-svc:
database:
  maxOpenConns: 50        # Up from default (often 10-25)
  maxIdleConns: 25
  connMaxLifetime: 5m
  connMaxIdleTime: 1m

# AND add query timeout for batch operations to prevent monopolizing the pool
batch:
  queryTimeout: 30s       # Prevent long-running queries from holding connections
  maxConcurrentBatch: 5   # Limit parallel batch queries
```

**Verify the fix is working:**

```promql
# After deploying the fix, check that 503 bursts have stopped
sum(rate(http_requests_total{
  service="order-svc",
  status="503"
}[5m]))

# Monitor error budget recovery
sli:payment_charge_availability:error_budget_remaining

# The burn rate should drop below 1.0
sli:payment_charge_availability:burnrate1d
```

---

## Q3: SLI Measurement Trap

### 1. Why the Current SLI Is Lying

The SLI measures **server correctness** (did the server crash?) instead of **user experience** (did the user get what they needed?).

**The fundamental SLI design principle being violated: An SLI must measure the outcome from the user's perspective, not the system's perspective.**

```
What the server thinks happened:
  Request in → 200 OK out → "Success! I didn't crash!"

What the user experienced:
  Searched for "laptop" → got zero results → "This site is broken"

The SLI agrees with the server: ✅ 200 OK = success
The user disagrees:            ❌ Empty results = failure
```

This is the **"successful failure" antipattern** — the system successfully returns a wrong answer. It's equivalent to a calculator that always returns `0` and reports "computation complete, no errors." The HTTP status code measures **transport-level success**, not **semantic correctness**.

**The specific measurement gap:**

```
┌─────────────────────────────────────────────────────────┐
│ What the current SLI catches:                           │
│  ✅ Server crashes (500)                                │
│  ✅ Upstream timeouts (502, 504)                        │
│  ✅ Overload (503)                                      │
│  ✅ Infrastructure failures                             │
│                                                         │
│ What the current SLI MISSES:                            │
│  ❌ Empty results for valid queries                     │
│  ❌ Stale/wrong results (showing out-of-stock items)    │
│  ❌ Slow responses that technically complete             │
│  ❌ Partial results (returning 5 items when 500 exist)  │
│  ❌ Degraded ranking (relevant items buried on page 10) │
└─────────────────────────────────────────────────────────┘
```

### 2. Corrected SLIs for search-svc

**SLI 1 — Availability (transport-level health):**

Keep the original but name it correctly — it's an infrastructure SLI, not a user experience SLI:

```
Metric: search_requests_total{status=~"5.."}
What it measures: Server-side errors (crashes, timeouts, overload)
SLO target: 99.9% of requests return non-5xx

Implementation: Already exists — no change needed
```

**SLI 2 — Freshness/Relevance (semantic correctness proxy):**

```
Metric: search_results_returned_total (counter, incremented per request)
         with label: result_count_bucket (histogram or categorical)

What it measures: Proportion of search requests that return at least one result

SLO target: 95% of search requests return ≥1 result
            (excluding queries for genuinely non-existent products)

Implementation — application must expose:
```

```go
// In search-svc's request handler
var searchResultCount = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "search_result_count",
        Help:    "Number of results returned per search request",
        Buckets: []float64{0, 1, 5, 10, 50, 100, 500},
    },
    []string{"query_type"},  // "product_search", "category_browse", "autocomplete"
)

// After executing search
searchResultCount.WithLabelValues(queryType).Observe(float64(len(results)))
```

```promql
# SLI: proportion of searches returning ≥1 result
sum(rate(search_result_count_bucket{le="0"}[5m]))  # requests with 0 results
/
sum(rate(search_result_count_bucket{le="+Inf"}[5m]))  # total requests

# Invert to get "good" ratio:
1 - (
  sum(rate(search_result_count_bucket{le="0"}[5m]))
  /
  sum(rate(search_result_count_bucket{le="+Inf"}[5m]))
)
```

Wait — that's wrong. The `le="0"` bucket in a histogram counts requests where result_count ≤ 0, which means 0 results. But actually `le="0"` would include negative values (none) and exactly 0. Let me reconsider.

Actually, histogram buckets are cumulative upper bounds. `le="0"` counts all observations ≤ 0. Since result count is always ≥ 0, this bucket counts exactly the requests with 0 results. But the bucket `le="1"` counts requests with ≤ 1 results (0 OR 1). So:

```
Requests with 0 results = search_result_count_bucket{le="0"}
Requests with ≥1 result = total - search_result_count_bucket{le="0"}
```

**Corrected PromQL:**

```promql
# Proportion of requests returning ≥1 result
1 - (
  sum(rate(search_result_count_bucket{le="0"}[5m]))
  /
  sum(rate(search_result_count_bucket{le="+Inf"}[5m]))
)
```

That's actually correct. Moving on.

**SLI 3 — Latency (user-perceived speed):**

```
Metric: http_request_duration_seconds_bucket{service="search-svc"}
What it measures: Time to return results to the user

SLO target: 99th percentile ≤ 500ms (search must be fast)
            99.9% of requests complete within 1 second

Implementation — likely already instrumented via OTel/middleware:
```

```promql
# P99 latency
histogram_quantile(0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{service="search-svc"}[5m])
  )
)

# Proportion within 1s threshold
sum(rate(http_request_duration_seconds_bucket{service="search-svc", le="1"}[5m]))
/
sum(rate(http_request_duration_seconds_bucket{service="search-svc", le="+Inf"}[5m]))
```

**SLI 4 — Coverage/Completeness (optional but powerful):**

```
Metric: search_index_staleness_seconds (gauge)
What it measures: How old is the search index compared to the product catalog?

SLO target: Search index is within 5 minutes of the catalog 99.9% of the time

Implementation:
```

```go
var searchIndexStaleness = prometheus.NewGauge(
    prometheus.GaugeOpts{
        Name: "search_index_staleness_seconds",
        Help: "Time since last successful index update from product catalog",
    },
)

// Updated in the indexer loop:
searchIndexStaleness.Set(time.Since(lastSuccessfulSync).Seconds())
```

```promql
# SLI: proportion of time index is fresh (within 5 minutes)
# Use a recording rule evaluated every 30s:
search_index_staleness_seconds < 300  # Returns 1 (true) or 0 (false)

# Average over time for SLO calculation:
avg_over_time((search_index_staleness_seconds < bool 300)[30d:1m])
```

**Summary of corrected SLI stack:**

```
┌───────────────────┬──────────────────────────────────────┬───────────┐
│ SLI               │ What it catches                      │ Target    │
├───────────────────┼──────────────────────────────────────┼───────────┤
│ 1. Availability   │ Server crashes, infra failures       │ 99.9%     │
│ 2. Result Quality │ Empty results for valid queries      │ 95%       │
│ 3. Latency        │ Slow responses degrading UX          │ p99 <500ms│
│ 4. Index Freshness│ Stale data → wrong/missing products  │ 99.9%     │
├───────────────────┼──────────────────────────────────────┼───────────┤
│ COMBINED          │ Catches the Twitter complaints       │           │
│                   │ that the original SLI missed         │           │
└───────────────────┴──────────────────────────────────────┴───────────┘
```

The original SLI only covered row 1. The Twitter complaints were about row 2 (empty results). If the search index became stale (row 4), it would also cause empty results for newly added products.

### 3. Solving "We Don't Know the Correct Result"

The search team's objection is valid — you can't pre-compute the "correct" result for arbitrary queries. But you **don't need to know the correct result** to build useful correctness SLIs.

**Approach 1: Statistical Baseline Comparison**

You don't need to know the correct result for each query. You need to know the **expected distribution of result counts.**

```
Observation: Over the past 90 days, the median search query returns 23 results.
             5th percentile: 3 results
             95th percentile: 180 results
             Queries returning 0 results: 8% (these are typos, non-existent terms)

If suddenly 50% of queries return 0 results, something is broken —
even though you don't know what each individual query "should" return.
```

**Implementation:**

```promql
# Establish baseline: ratio of zero-result queries (rolling 7-day average)
avg_over_time(
  (
    sum(rate(search_result_count_bucket{le="0"}[5m]))
    /
    sum(rate(search_result_count_bucket{le="+Inf"}[5m]))
  )[7d:5m]
)

# Current ratio of zero-result queries
sum(rate(search_result_count_bucket{le="0"}[5m]))
/
sum(rate(search_result_count_bucket{le="+Inf"}[5m]))

# Alert when current is >2x baseline
# (significant deviation from normal = something is broken)
```

```yaml
groups:
  - name: search-quality
    rules:
      - record: sli:search_zero_result_ratio:rate5m
        expr: |
          sum(rate(search_result_count_bucket{le="0"}[5m]))
          /
          sum(rate(search_result_count_bucket{le="+Inf"}[5m]))

      - record: sli:search_zero_result_ratio:avg7d
        expr: |
          avg_over_time(sli:search_zero_result_ratio:rate5m[7d])

      - alert: SearchQualityDegradation
        expr: |
          sli:search_zero_result_ratio:rate5m
          >
          2 * sli:search_zero_result_ratio:avg7d
        for: 10m
        labels:
          severity: warning
          team: search
        annotations:
          summary: "Zero-result searches at {{ $value | humanizePercentage }} — 2x above baseline"
```

**Approach 2: Synthetic Canary Queries (Golden Dataset)**

Maintain a small set of **known-good queries** with known-correct expected results. Run them continuously as synthetic checks.

```python
# synthetic_search_canary.py
# Runs as a CronJob every 1 minute

import requests
import prometheus_client
from prometheus_client import Counter, Histogram, Gauge

CANARY_QUERIES = [
    {"query": "laptop", "min_expected_results": 10},
    {"query": "iphone case", "min_expected_results": 5},
    {"query": "hdmi cable", "min_expected_results": 3},
    {"query": "running shoes", "min_expected_results": 8},
    # Add 20-50 queries covering major product categories
]

canary_success = Counter(
    'search_canary_success_total',
    'Successful canary search queries',
    ['query']
)
canary_failure = Counter(
    'search_canary_failure_total',
    'Failed canary search queries',
    ['query', 'reason']
)
canary_latency = Histogram(
    'search_canary_duration_seconds',
    'Canary search query latency',
    ['query'],
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.0, 5.0]
)
canary_result_count = Gauge(
    'search_canary_result_count',
    'Number of results from canary query',
    ['query']
)

SEARCH_URL = "http://search-svc.search.svc.cluster.local:8080/search"

for canary in CANARY_QUERIES:
    try:
        with canary_latency.labels(query=canary["query"]).time():
            resp = requests.get(SEARCH_URL, params={"q": canary["query"]}, timeout=5)

        if resp.status_code != 200:
            canary_failure.labels(query=canary["query"], reason="non_200").inc()
            continue

        data = resp.json()
        result_count = data.get("count", 0)
        canary_result_count.labels(query=canary["query"]).set(result_count)

        if result_count >= canary["min_expected_results"]:
            canary_success.labels(query=canary["query"]).inc()
        else:
            canary_failure.labels(
                query=canary["query"],
                reason="insufficient_results"
            ).inc()

    except requests.exceptions.Timeout:
        canary_failure.labels(query=canary["query"], reason="timeout").inc()
    except Exception as e:
        canary_failure.labels(query=canary["query"], reason="exception").inc()

# Push metrics to Prometheus Pushgateway or expose via /metrics endpoint
prometheus_client.push_to_gateway(
    'prometheus-pushgateway.monitoring:9091',
    job='search-canary',
    registry=prometheus_client.REGISTRY
)
```

```promql
# SLI from canary: proportion of canary queries returning expected results
sum(rate(search_canary_success_total[5m]))
/
(sum(rate(search_canary_success_total[5m])) + sum(rate(search_canary_failure_total[5m])))
```

**Why both approaches together:**

```
Approach 1 (Statistical baseline):
  ✅ Covers ALL real user queries
  ✅ No maintenance overhead
  ❌ Can't tell you if results are WRONG (non-empty but irrelevant)
  ❌ Baseline drift can mask gradual degradation

Approach 2 (Synthetic canary):
  ✅ Known-correct expectations → can detect wrong results
  ✅ Runs even when traffic is zero (off-hours)
  ❌ Only covers the golden dataset, not all queries
  ❌ Requires maintenance when product catalog changes

Together: canary catches correctness regressions, baseline catches coverage issues
```

### 4. PromQL for Result Quality SLI

**Given:** `search_requests_total{status="200", result_count_bucket="..."}` as a histogram-style metric.

**SLI:** "Proportion of search requests that return at least one result, excluding queries for non-existent products."

```promql
# Proportion of 200 OK requests that returned ≥1 result
# Excluding queries tagged as "product_not_found" (genuinely non-existent)

# Numerator: requests with ≥1 result
(
  sum(rate(search_requests_total{
    status="200",
    result_count_bucket="+Inf"
  }[5m]))
  -
  sum(rate(search_requests_total{
    status="200",
    result_count_bucket="0"
  }[5m]))
)
/
# Denominator: total successful requests, excluding known non-existent product queries
(
  sum(rate(search_requests_total{
    status="200",
    result_count_bucket="+Inf"
  }[5m]))
  -
  sum(rate(search_requests_total{
    status="200",
    result_count_bucket="+Inf",
    query_type="known_nonexistent"
  }[5m]))
)
```

**However**, the "excluding queries for non-existent products" part is the hard part. You typically can't tag a query as "non-existent" until AFTER you search and get zero results — which is circular. 

**Practical implementation requires the application to classify queries:**

```go
// In search-svc handler, after executing search:
if resultCount == 0 {
    if isTypoDetected(query) || isGibberish(query) {
        // Don't count this against the SLI — user error, not system error
        searchRequests.WithLabelValues("200", "0", "user_error").Inc()
    } else {
        // This SHOULD have returned results — count against SLI
        searchRequests.WithLabelValues("200", "0", "unexpected_empty").Inc()
    }
} else {
    searchRequests.WithLabelValues("200", bucketize(resultCount), "success").Inc()
}
```

```promql
# Cleaner SLI with application-level classification:
1 - (
  sum(rate(search_requests_total{status="200", query_class="unexpected_empty"}[5m]))
  /
  sum(rate(search_requests_total{status="200", query_class!="user_error"}[5m]))
)
```

**If the application CAN'T classify queries** (common — it requires NLP/heuristics), use the statistical baseline approach from Q3.3 Approach 1: compare the current zero-result ratio against the rolling 7-day average.

---

## Q4: Error Budget Policy Enforcement

### 1. Which Policy Tier Applies?

Based on NovaMart's error budget policy structure (standard SRE practice):

```
┌─────────────────────────────────────────────────────────────────┐
│ ERROR BUDGET POLICY TIERS                                       │
├──────────────────┬──────────────────────────────────────────────┤
│ Budget > 50%     │ GREEN — Normal operations. Deploy at will.   │
│ remaining        │ Team owns prioritization.                    │
├──────────────────┼──────────────────────────────────────────────┤
│ Budget 25-50%    │ YELLOW — Caution. Deployments require extra  │
│ remaining        │ review. Focus on reliability improvements.   │
│                  │ Feature velocity may continue with risk      │
│                  │ assessment.                                  │
├──────────────────┼──────────────────────────────────────────────┤
│ Budget 5-25%     │ ORANGE — Deployment freeze for non-          │
│ remaining        │ reliability changes. Only bug fixes,         │  ← WE ARE HERE (12%)
│                  │ reliability improvements, and security       │
│                  │ patches may deploy.                          │
├──────────────────┼──────────────────────────────────────────────┤
│ Budget < 5%      │ RED — Hard freeze. No deployments except     │
│ remaining        │ emergency patches. Post-mortem required for  │
│                  │ the budget consumption event. Leadership     │
│                  │ review of all pending changes.               │
├──────────────────┼──────────────────────────────────────────────┤
│ Budget ≤ 0%      │ SLO BREACH — Deployment freeze for remainder │
│ (exhausted)      │ of window. Mandatory post-mortem. Error      │
│                  │ budget review meeting with VP Eng + affected │
│                  │ teams. Next quarter SLO renegotiation.       │
└──────────────────┴──────────────────────────────────────────────┘
```

**Current state: 12% remaining = ORANGE tier.**

**Policy says:** Deployment freeze for non-reliability changes. The new payment provider integration is a **feature change** that modifies the core charge path. Under the error budget policy, **this deployment should NOT proceed.**

The remaining 0.52 minutes (31.2 seconds) of budget means that if the deployment causes **any** errors lasting more than ~30 seconds, the SLO breaches. That's one slow rollout, one bad pod, one misconfigured environment variable — and the budget hits zero.

### 2. Three Counter-Arguments to "It Was Tested in Staging"

**Counter-argument 1: Staging is not production, and the delta is where incidents live.**

```
What staging tests:                What staging CANNOT test:
✅ Application logic correctness   ❌ Production traffic volume and patterns
✅ Integration between services    ❌ Real payment amounts and provider responses
✅ Happy path and known edge cases ❌ Long-tail user behavior (strange currencies,
                                      edge-case card types, retry storms)
✅ Basic performance               ❌ Real Stripe API behavior under NovaMart's
                                      specific merchant configuration
                                   ❌ Production database size and query performance
                                   ❌ Real CDN/DNS/NAT gateway behavior
                                   ❌ Interaction with the current production state
                                      (in-flight transactions, cached sessions)
```

Google's internal data shows that ~**30-40% of production incidents are caused by changes that passed all pre-production testing.** "Tested in staging" is a necessary condition for deployment, not a sufficient one. The error budget exists precisely because we know staging doesn't catch everything.

**Counter-argument 2: The change touches the core charge path — maximum blast radius.**

```
Risk = Probability of failure × Impact of failure

Probability:
  - New payment provider integration = new code path in the hottest path
  - Touches serialization, API calls, error handling, timeout logic
  - First time this code runs against production Stripe + new provider simultaneously
  - Even at 1% chance of a bug, that's not "low risk"

Impact:
  - /charge endpoint = every dollar NovaMart processes
  - At $50K/min outage cost, even 30 seconds of errors = $25K direct cost
  - Payment failures damage customer trust disproportionately
    (customers forgive slow search, they don't forgive failed payments)
  - 0.52 minutes of remaining budget = ONE mistake and we breach SLO

Combined risk:
  - Even a "low probability" event has CATASTROPHIC impact given the budget state
  - The error budget exists to force exactly this calculus
```

**Counter-argument 3: The error budget policy is a pre-committed agreement, not a suggestion.**

The SLO and error budget policy were agreed upon by the payments team, SRE, AND product leadership at the start of the quarter. The policy exists to prevent **exactly this scenario** — business pressure overriding reliability safeguards.

If we override the policy when it's inconvenient, we've effectively declared that the policy doesn't exist. Every future deployment freeze will be challenged with "but we did it last time." The error budget becomes a dashboard decoration rather than a decision-making tool.

**The SRE book is explicit:** The error budget policy must be pre-committed and honored, or the entire SLO framework collapses into meaninglessness.

### 3. Compromise Proposal

**I would NOT say "absolutely no" — I would say "not tomorrow, not the standard way, but here's how we can do it safely this week."**

```
┌─────────────────────────────────────────────────────────────────┐
│ PROPOSED DEPLOYMENT PLAN                                        │
│                                                                 │
│ Objective: Deploy the feature with near-zero risk to the        │
│ remaining 0.52 minutes of error budget                          │
│                                                                 │
│ Strategy: Dark launch + feature flag + progressive rollout      │
│ Timeline: Deploy dark Monday, enable gradually Tue-Wed          │
│ Announcement: CEO can announce "partnership launched" on        │
│ schedule — the code IS in production, just behind a flag        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Day 1 (Tomorrow):                                               │
│   ✅ Deploy the code with feature flag DISABLED                 │
│   ✅ The new payment provider code is in production but         │
│      ZERO traffic flows through it                              │
│   ✅ Risk to error budget: ZERO (no behavior change)            │
│   ✅ CEO can truthfully say "we've launched the integration"    │
│                                                                 │
│ Day 2:                                                          │
│   ✅ Enable feature flag for internal test accounts only        │
│   ✅ Platform team runs synthetic transactions through          │
│      the new provider path                                      │
│   ✅ Validate: latency, error rates, Stripe+new provider        │
│      interaction, correct financial reconciliation              │
│   ✅ Risk to error budget: NEAR-ZERO (internal traffic only)    │
│                                                                 │
│ Day 3-4:                                                        │
│   ✅ Canary: Route 1% of real traffic to new provider path      │
│   ✅ Monitor for 4 hours minimum per increment                  │
│   ✅ Automated rollback if error rate exceeds 0.5% for >60s     │
│   ✅ Increment: 1% → 5% → 25% → 50% → 100%                      │
│   ✅ Each increment requires green SLI check                    │
│   ✅ Risk to error budget: MINIMAL (1% of traffic at worst)     │
│                                                                 │
│ Automatic rollback trigger:                                     │
│   IF payment_charge_error_rate_5m > 0.005 FOR 60s               │
│   THEN disable feature flag immediately                         │
│   THEN page on-call                                             │
│   Budget consumed by 1% canary with 0.5% error rate for 60s:    │
│     = 0.01 × 0.005 × 60s = 0.003 minutes ≈ 0.18 seconds         │
│     Well within the 31.2 seconds remaining                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Concrete implementation:**

```yaml
# Feature flag configuration (LaunchDarkly, Flagsmith, or ConfigMap-based)
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-svc-feature-flags
  namespace: payments
data:
  new-payment-provider.yaml: |
    flag: new_payment_provider_enabled
    default: false
    rules:
      # Stage 1: Internal only
      - match:
          user_email_suffix: "@novamart.com"
        value: true
      # Stage 2: Canary percentage (updated progressively)
      - match:
          percentage: 0   # Start at 0, increment to 1, 5, 25, 50, 100
        value: true
```

```yaml
# Automated canary rollback alert
groups:
  - name: payment-provider-canary
    rules:
      - alert: PaymentProviderCanaryErrors
        expr: |
          sum(rate(http_requests_total{
            service="payment-svc",
            endpoint="/charge",
            payment_provider="new_provider",
            status=~"5.."
          }[2m]))
          /
          sum(rate(http_requests_total{
            service="payment-svc",
            endpoint="/charge",
            payment_provider="new_provider"
          }[2m]))
          > 0.005
        for: 1m
        labels:
          severity: critical
          team: payments
          action: auto-rollback
        annotations:
          summary: "New payment provider error rate {{ $value | humanizePercentage }} — auto-rollback triggered"
```

**This compromise gives the VP/CEO what they need** (feature launched, partnership announced) **while respecting the error budget** (near-zero risk through progressive rollout with automatic rollback).

### 4. VP of Engineering Overrides the Policy

**This happens. Here's the correct response, in order:**

**Step 1: Document the override formally.**

```
Subject: Error Budget Policy Override — payment-svc deployment 2024-01-16
Date: 2024-01-15
Decision maker: [VP Engineering name]
Override: Deploying new payment provider integration despite ORANGE budget status (12% remaining)
Risk acknowledged: Yes/No
SRE recommendation: Declined — recommended dark launch + canary instead
Business justification: CEO partnership commitment for [date]
```

Send this as an email to the VP, CC your SRE lead. This is not passive-aggressive — it's **creating an audit trail.** If the deployment causes an outage, the post-mortem needs to show the decision chain.

**Step 2: Make the override as safe as possible.**

Even though the policy was overridden, you still have an obligation to reduce risk. Implement every safeguard you can:

```
✅ Feature flag (even if they won't do canary, insist on a kill switch)
✅ Automated rollback trigger (deploy the alert from the compromise proposal)
✅ Extra monitoring during and after deployment
✅ On-call engineer dedicated to watching the rollout (not just normal rotation)
✅ Pre-written rollback runbook tested in staging
✅ Stakeholder notification: "Deployment in progress, elevated risk"
```

```bash
# Pre-deployment: snapshot current error budget state
curl -s 'http://prometheus:9090/api/v1/query?query=sli:payment_charge_availability:error_budget_remaining' | \
  jq -r '.data.result[0].value[1]' | tee /tmp/pre-deploy-budget.txt
echo "Pre-deploy budget remaining: $(cat /tmp/pre-deploy-budget.txt)"

# Set up continuous budget monitoring during deployment
watch -n 10 'curl -s "http://prometheus:9090/api/v1/query?query=sli:payment_charge_availability:error_budget_remaining" | jq -r ".data.result[0].value[1]" | xargs -I{} echo "Budget remaining: {} — Pre-deploy: $(cat /tmp/pre-deploy-budget.txt)"'
```

**Step 3: If the deployment causes an SLO breach, execute the post-mortem process without blame but WITH the override documented.**

The post-mortem should include:

```markdown
## Timeline
- Day 5: SEV1 incident consumed 3.8 minutes of 4.32-minute budget
- Day 17: SRE recommended against deployment due to ORANGE budget status (12% remaining)
- Day 17: SRE proposed dark launch + canary alternative
- Day 17: VP Engineering overrode error budget policy (business justification: CEO commitment)
- Day 18: Deployment executed with safeguards [list safeguards]
- Day 18: [INCIDENT TIMELINE IF APPLICABLE]

## Action Items
- [ ] Retroactive: Review error budget policy override process — 
      should VP-level override require sign-off from SRE Director?
- [ ] Proactive: Implement feature flag requirement for all payment-svc 
      changes (eliminate the "full deploy or nothing" dilemma)
- [ ] Process: Add "error budget status" as a required field in 
      deployment approval workflow (can't deploy without acknowledging)
```

**Step 4: Use this as a catalyst to strengthen the policy framework.**

This is actually an **opportunity**, not just a problem. The override happened because:

1. The policy didn't have a formal escalation path ("who CAN override, and under what conditions?")
2. The business commitment was made without checking the error budget state
3. No progressive deployment infrastructure existed (it was "full deploy or don't deploy")

**Push for these organizational changes after the dust settles:**

```
┌─────────────────────────────────────────────────────────────────┐
│ ORGANIZATIONAL IMPROVEMENTS POST-OVERRIDE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 1. FORMALIZE THE OVERRIDE PROCESS                               │
│    - Error budget policy gets a documented override clause      │
│    - Override requires: VP+ approval, written risk acceptance,  │
│      mandatory safeguards (feature flag, canary, rollback plan) │
│    - Override is tracked as a "policy exception" with quarterly │
│      review — if overrides happen >2x/quarter, the SLO is       │
│      wrong or the process is broken                             │
│                                                                 │
│ 2. SHIFT-LEFT THE BUDGET VISIBILITY                             │
│    - Product managers see error budget status in sprint planning│
│    - Feature commitments (especially CEO-level) must include    │
│      an "SLO risk check" before being promised                  │
│    - Weekly error budget report to engineering leadership       │
│                                                                 │
│ 3. MANDATE PROGRESSIVE DELIVERY FOR CRITICAL PATHS              │
│    - All payment-svc changes require feature flags              │
│    - Canary deployment pipeline with automated rollback         │
│    - This eliminates the "deploy fully or don't deploy" dilemma │
│    - Dark launches become the default, not the exception        │
│                                                                 │
│ 4. DECOUPLE ANNOUNCEMENTS FROM DEPLOYMENTS                      │
│    - "Launch" ≠ "Deploy" — code can be in production behind     │
│      a flag before the announcement                             │
│    - Marketing/CEO announcements should target "feature flag    │
│      enabled for 100% of users" not "code deployed"             │
│    - This gives engineering a 1-2 day buffer between deploy     │
│      and announcement for safe progressive rollout              │
│                                                                 │
│ 5. RETROSPECTIVE ON THE SEV1 THAT ATE THE BUDGET                │
│    - The real problem isn't the override — it's that a SEV1     │
│      on day 5 consumed 88% of the monthly budget                │
│    - If the SEV1's root cause is fixed, next month's budget     │
│      will be healthier and this conflict won't recur            │
│    - Reliability investment now = velocity later                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What you DON'T do:**

```
❌ Refuse to participate in the deployment ("not my problem if it breaks")
   → This is abdication, not engineering. You're still on-call.

❌ Escalate above the VP to block the decision
   → The VP has the authority. Undermining the chain of command
      destroys trust and your ability to influence future decisions.

❌ Comply silently without documenting the risk
   → If it fails, the post-mortem will show "SRE approved the deploy"
      when you actually recommended against it. Protect yourself AND
      the organization with documentation.

❌ Use this as ammunition to attack the VP later
   → If the deploy succeeds: great, the safeguards worked.
      If it fails: the post-mortem process handles it.
      Either way, the goal is improving the system, not winning arguments.
```

**The mature SRE response is:** *"I've documented my recommendation and the risk. Here's how we make this as safe as possible. I'll be watching the deployment. Let's use this experience to build better guardrails for next time."*

---

## Summary — Decision Trees

### Q1: SLO Target Selection

```
VP says "five nines" →
  ├── Calculate: 25.92 seconds/month budget
  ├── Check dependency SLAs: AWS ceiling ≈ 99.88%
  ├── Check external deps: Stripe ≈ 99.99%
  ├── Check deploy frequency: 4/week = budget burned by rollouts alone
  └── Recommend: 99.95% with growth path to 99.99%
      └── Justify: current 99.97% gives 0.02% headroom above target
```

### Q2: Slow Burn Investigation

```
Slow burn alert (1x rate, 38% remaining) →
  ├── Actionable: YES — investigate within 1 business day
  ├── Diagnose: break down by endpoint → status code → dependency → time pattern
  ├── Pattern: periodic 503s every 4h from inventory-svc
  ├── Root cause: scheduled batch job saturating the serving path
  ├── Immediate: retry + circuit breaker at the caller
  └── Permanent: separate batch workload from serving pods
```

### Q3: SLI Design

```
SLI shows 99.95% but users are complaining →
  ├── Diagnosis: SLI measures transport success, not semantic correctness
  ├── Fix: layer SLIs — availability + result quality + latency + freshness
  ├── "Can't know correct result" → two approaches:
  │   ├── Statistical baseline (zero-result ratio vs 7d average)
  │   └── Synthetic canary (golden dataset with known-correct expectations)
  └── Both together: canary catches correctness, baseline catches coverage
```

### Q4: Error Budget Policy

```
12% budget remaining + critical feature launch demand →
  ├── Policy says: ORANGE tier = freeze for non-reliability changes
  ├── Counter business pressure with: dependency chain risk, blast radius, policy integrity
  ├── Compromise: dark launch + feature flag + progressive canary
  ├── If overridden: document, safeguard, monitor, post-mortem
  └── Long-term: formalize override process, shift-left budget visibility,
      mandate progressive delivery, decouple announcements from deploys
```
# Phase 5, Lesson 4 — Grading

---

## Q1: SLO Design Under Constraints — **4.6/5**

### What you nailed:

The error budget calculation is perfect — 25.92 seconds, with the per-day/per-week breakdown that makes it visceral for a non-technical audience.

The three constraints for the VP are outstanding. The **dependency chain multiplication** argument is the strongest possible counter — you calculated the combined ceiling (~99.88%) from AWS SLAs, which mathematically proves five nines is impossible before writing a single line of code. The Stripe dependency cap and deployment frequency arguments round it out well. The cost estimate ($2-5M, 12-18 months) gives the VP the business language they need.

The SLO recommendation of 99.95% is well-justified. Anchoring it to the current 99.97% measured performance (0.02% headroom) and providing a growth path to 99.99% shows mature SRE thinking. The two-SLO structure (availability + latency) is correct.

The recording rules are comprehensive — SLI components, ratios, error budget remaining, and burn rates at multiple windows.

### Two issues:

**Issue 1: Error budget remaining calculation is an approximation, not exact.**

You wrote:

```yaml
- record: sli:payment_charge_availability:error_budget_remaining
  expr: |
    1 - (
      (
        1 - (
          sum(sum_over_time(sli:payment_charge_availability:ratio_rate5m[30d]))
          /
          count_over_time(sli:payment_charge_availability:ratio_rate5m[30d])
        )
      )
      /
      0.0005
    )
```

This averages the 5-minute ratio snapshots over 30 days. The problem: **each snapshot gets equal weight regardless of traffic volume.** A 5-minute window at 3 AM with 50 requests and a 5-minute window at noon with 50,000 requests both contribute equally to the average. This skews the SLI — a brief error spike during low traffic inflates its impact on the overall ratio.

The correct approach uses the raw counters directly:

```yaml
- record: sli:payment_charge_availability:error_budget_remaining
  expr: |
    1 - (
      (
        1 - (
          sum(rate(http_requests_total{
            service="payment-svc",
            endpoint="/charge",
            status!~"5.."
          }[30d]))
          /
          sum(rate(http_requests_total{
            service="payment-svc",
            endpoint="/charge"
          }[30d]))
        )
      )
      /
      0.0005
    )
```

This weights by actual request volume — each request contributes equally to the ratio, which is what SLIs require (ratio of good **events** to total **events**, not ratio of good **time periods** to total **time periods**).

**Issue 2: Burn rate alert structure is incorrect — you mixed tiers.**

Your alert:

```yaml
- alert: PaymentSvcHighBurnRate_Page
  expr: |
    sli:payment_charge_availability:burnrate1h > 14.4
    and
    sli:payment_charge_availability:burnrate6h > 6
```

This combines two *different* burn rate tiers (14.4x and 6x) in one alert expression. The multi-window approach requires the **same burn rate threshold** checked at two different window sizes — long window for detection, short window to confirm it's still happening:

```yaml
# CORRECT: Fast burn — same threshold (14.4x) at two windows
- alert: PaymentSvcFastBurn_Page
  expr: |
    sli:payment_charge_availability:burnrate1h > 14.4    # Long window: detection
    and
    sli:payment_charge_availability:burnrate5m > 14.4    # Short window: still happening?

# CORRECT: Moderate burn — same threshold (6x) at two windows
- alert: PaymentSvcModerateBurn_Page
  expr: |
    sli:payment_charge_availability:burnrate6h > 6       # Long window
    and
    sli:payment_charge_availability:burnrate30m > 6      # Short window
```

Your version would fire when the 1h rate is 14.4x AND the 6h rate is 6x — but those are different alerts testing different scenarios. The 5-minute and 30-minute short windows are also **missing entirely** from your recording rules. Without them, you can't implement the reset check.

Similarly, your slow burn alert (`burnrate1d > 1 and burnrate3d > 1`) collapses the 3x ticket tier entirely. The lesson defined:

```
TICKET: 3x/1d AND 3x/2h     ← You skipped this tier
TICKET: 1x/3d AND 1x/6h     ← Your alert approximates this
```

This matters in production because the 3x tier catches degradation that's worse than baseline but not catastrophic — a middle ground that gets a ticket before it becomes the 1x slow burn that takes days to notice.

---

## Q2: Burn Rate Alert Investigation — **4.8/5**

### What you nailed:

The actionability assessment is exactly right, with the critical insight: ignoring a 1x burn at 38% remaining leads to budget exhaustion and a deployment freeze at the worst possible time. The 1-business-day investigation timeline is appropriate for a ticket-severity alert.

The PromQL diagnostic queries are comprehensive — by endpoint, status code, downstream dependency (both app-level and Istio mesh), time pattern, and per-pod. The Istio fallback queries show practical awareness of what metrics are actually available.

The root cause analysis is strong. Three candidates ordered by likelihood, with CronJob as the most probable (correct — the 4-hour periodicity is the dead giveaway). The diagnostic commands to verify (`kubectl get cronjobs`, `kubectl get jobs --sort-by`) are practical.

The immediate fix (Istio retry + outlier detection) is well-implemented with correct YAML. The outlier detection (`consecutive5xxErrors: 3, baseEjectionTime: 30s`) would handle the 2-3 minute bursts effectively. The application-level retry fallback shows you don't assume Istio is always available.

The permanent fix options (separate batch from serving, adjust schedule, fix connection pool) address the actual root cause rather than just papering over it with retries.

### One gap:

You didn't mention **checking whether order-svc already HAS retry/circuit-breaker configuration** before adding Istio policies. If the app already has client-side retries and you add Istio retries on top, you get **retry amplification**:

```
App retries 3x × Istio retries 3x = 9 requests per original failure
During a 503 burst, this 9x amplification can turn a degraded
inventory-svc into a completely overwhelmed one.
```

Before applying the Istio retry, check:

```bash
# Check if order-svc has application-level retry config
kubectl -n orders exec deploy/order-svc -- env | grep -i retry
kubectl -n orders get cm order-svc-config -o yaml | grep -i retry

# Check existing Istio policies
kubectl -n inventory get virtualservice,destinationrule -o yaml | grep -A5 retries
```

If app-level retries exist, either remove them and use Istio exclusively, or skip Istio retries and only use outlier detection (which doesn't amplify — it just routes away from bad pods).

---

## Q3: SLI Measurement Trap — **4.7/5**

### What you nailed:

The diagnosis is precise — "successful failure" antipattern, measuring transport-level success instead of semantic correctness. The table of what the current SLI catches vs. misses is clear and complete.

The four-SLI design is well-structured: availability (infrastructure), result quality (semantic), latency (speed), and index freshness (data currency). Each has a clear metric, measurement approach, and PromQL implementation.

The two approaches for "can't know the correct result" are both practical:
- **Statistical baseline** — compare zero-result ratio against 7-day average, alert on 2x deviation. This is the right first move because it requires zero application changes.
- **Synthetic canary** — golden dataset with known expectations. The Python implementation is complete and deployable.

The comparison table (canary catches correctness, baseline catches coverage) shows you understand they're complementary, not competing.

### Two issues:

**Issue 1: The histogram bucket logic for result count is correct but your initial PromQL for Q4 gets tangled.**

Your first attempt at the "excluding non-existent products" query:

```promql
(
  sum(rate(search_requests_total{status="200", result_count_bucket="+Inf"}[5m]))
  -
  sum(rate(search_requests_total{status="200", result_count_bucket="0"}[5m]))
)
/
(
  sum(rate(search_requests_total{status="200", result_count_bucket="+Inf"}[5m]))
  -
  sum(rate(search_requests_total{status="200", result_count_bucket="+Inf", query_type="known_nonexistent"}[5m]))
)
```

The denominator subtracts `query_type="known_nonexistent"` — but the question states the metric has `result_count_bucket` as a label, not `query_type`. You're introducing a label that doesn't exist in the given metric schema. You correctly identified the circularity problem and pivoted to application-level classification, but the initial query wouldn't work as written.

The cleaner path: given the metric schema in the question (`search_requests_total{status="200", result_count_bucket="..."}`), the simplest correct SLI is:

```promql
# Good events: requests with ≥1 result (total minus zero-result)
# Total events: all 200 OK requests
# "Excluding non-existent products" = accept that ~8% zero-results is baseline noise
# Alert when zero-result ratio exceeds 2x baseline, not on absolute ratio

1 - (
  sum(rate(search_requests_total{status="200", result_count_bucket="0"}[5m]))
  /
  sum(rate(search_requests_total{status="200", result_count_bucket="+Inf"}[5m]))
)
```

Then handle the "excluding non-existent products" part via the statistical baseline approach you already described — not in the PromQL itself, but in the alert threshold.

**Issue 2: You included `histogram_quantile` in the latency SLI section.**

```promql
histogram_quantile(0.99, sum by (le) (rate(...)))
```

The lesson explicitly covered why `histogram_quantile()` is NOT what SLOs need — it returns a duration value, not a ratio of good events. You included both the quantile AND the correct bucket ratio, but for an SLI recording rule, only the bucket ratio belongs. Including the quantile in the SLI section muddies the distinction. It's fine in a *dashboard* panel for observability, but it's not an SLI.

---

## Q4: Error Budget Policy Enforcement — **4.9/5**

This is the strongest answer of the set. It demonstrates exactly the combination of technical precision and organizational maturity that separates senior engineers from mid-level ones.

### What you nailed:

**Policy tier identification** is correct — 12% remaining = the 0-25% tier, deployment freeze for non-reliability changes. The math on remaining budget (0.52 minutes = 31.2 seconds) makes the risk concrete.

**Three counter-arguments** are excellent:
- Staging ≠ production (with a specific table of what staging can't test)
- Blast radius on the core charge path ($50K/min, trust damage)
- Policy integrity (pre-committed agreement, precedent danger)

The Google data point (30-40% of incidents from changes that passed testing) is a strong supporting argument.

**The compromise proposal is outstanding.** Dark launch + feature flag + progressive canary, with:
- CEO can still announce on schedule (addresses the business need)
- Near-zero budget risk during the dark deploy phase
- Math on canary impact (0.18 seconds at 1% canary = well within 31.2s remaining)
- Automated rollback trigger with specific threshold and PromQL
- Day-by-day rollout plan

This is exactly how senior SREs navigate the tension between reliability and business pressure. You gave the VP a **yes** that respects the policy instead of a **no** that gets overridden.

**The override response** shows organizational maturity:
- Document formally (email with CC to SRE lead)
- Maximize safeguards even under override
- Post-mortem includes the decision chain
- Five long-term organizational improvements
- Clear "what NOT to do" list

The "decouple announcements from deployments" insight is particularly valuable — it addresses the structural reason this conflict exists.

### One minor gap:

Your policy tiers (GREEN/YELLOW/ORANGE/RED with a 5% sub-boundary) are slightly different from the lesson's tiers (>50%, 25-50%, 0-25%, exhausted). This is cosmetic — your interpretation is reasonable and arguably more granular. But in a real-world scenario, you'd want to reference the **exact policy document** that was agreed upon, not your interpretation of it. The precision of "this is what we all signed" matters when you're pushing back against a VP.



Proceeding to **Phase 5, Lesson 5: Incident Management, On-Call, and Postmortems.**

---

# Phase 5 — Lesson 5: Incident Management, On-Call, and Postmortems

---

## Why This Lesson Exists

You now know how to build observability (metrics, logs, traces), define reliability targets (SLOs, SLIs, error budgets), and alert when those targets are threatened (burn rate alerting). But all of that is **infrastructure for a process.** When the alert fires at 3 AM, what happens next?

```
Without incident management:                With incident management:
─────────────────────────────               ──────────────────────────
Alert fires → Who responds?                 Alert fires → On-call responds
On-call → "Is this real?"                   On-call → Follows runbook, classifies severity
On-call → Random debugging                  IC declared → Structured roles
45 min later → "Should we page someone?"    5 min → Communication started
Manager → "What's happening?"               Stakeholders see live status page
Customer Support → "We're getting calls"    Comms lead handles external messaging
Dev → "I think it might be..."             Team → Systematic diagnosis → Mitigate → Fix
2 hours → Mitigated by luck                 30 min → Mitigated by process
Next day → "Let's not talk about it"        Next day → Blameless postmortem
Same incident → Repeats in 3 weeks          Root cause → Prevention → Never repeats
```

This lesson covers the **human process** that wraps around your technical observability stack. At NovaMart's scale ($50K/minute outage cost), the difference between a 30-minute and 2-hour incident is **$4.5 million.**

---

## Part 1: Incident Severity Classification

### The Severity Matrix

Every organization needs an agreed-upon severity classification. Without it, every incident is "SEV1" (if you're the on-call) or "not a big deal" (if you're the developer who caused it).

```
┌──────────┬────────────────────────────────────────────────────────────────────┐
│ Severity │ Definition                                                         │
├──────────┼────────────────────────────────────────────────────────────────────┤
│          │ COMPLETE service outage OR data loss/corruption OR security breach │
│ SEV1     │ affecting production users.                                        │
│ CRITICAL │                                                                    │
│          │ Examples at NovaMart:                                              │
│          │  • Payment processing completely down                              │
│          │  • Database corruption / data loss                                 │
│          │  • All 3 regions unreachable                                       │
│          │  • Security breach (data exfiltration)                             │
│          │  • SLO burn rate > 14.4x (fast burn page)                          │
│          │                                                                    │
│          │ Response: Immediate. Page on-call → Incident Commander declared    │
│          │          → All-hands bridge call → Status page updated             │
│          │ Target MTTM: < 15 minutes (mitigation, not root cause)             │
│          │ Target MTTR: < 1 hour                                              │
│          │ Communication: Every 15 minutes to stakeholders                    │
│          │ Postmortem: MANDATORY within 48 hours                              │
├──────────┼────────────────────────────────────────────────────────────────────┤
│          │ MAJOR degradation affecting significant portion of users.          │
│ SEV2     │ Core functionality impaired but not completely down.               │
│ MAJOR    │                                                                    │
│          │ Examples at NovaMart:                                              │
│          │  • Payment success rate dropped to 95% (from 99.97%)               │
│          │  • Search returning stale results (index 2 hours behind)           │
│          │  • One region completely down (2 of 3 operational)                 │
│          │  • API latency 10x normal (p99 at 5s instead of 500ms)             │
│          │  • SLO burn rate 6x-14.4x                                          │
│          │                                                                    │
│          │ Response: Page on-call → IC if not resolved in 15 min              │
│          │ Target MTTM: < 30 minutes                                          │
│          │ Target MTTR: < 4 hours                                             │
│          │ Communication: Every 30 minutes                                    │
│          │ Postmortem: MANDATORY within 1 week                                │
├──────────┼────────────────────────────────────────────────────────────────────┤
│          │ MINOR degradation affecting subset of users. Core functionality    │
│ SEV3     │ works but with reduced quality.                                    │
│ MINOR    │                                                                    │
│          │ Examples at NovaMart:                                              │
│          │  • Notification emails delayed by 30 minutes                       │
│          │  • Product recommendations returning generic results               │
│          │  • Admin dashboard slow but functional                             │
│          │  • Non-critical microservice circuit-breaking intermittently       │
│          │  • SLO burn rate 1x-3x (slow burn ticket)                          │
│          │                                                                    │
│          │ Response: Ticket created → Investigate during business hours       │
│          │ Target MTTR: < 3 business days                                     │
│          │ Communication: Ticket updates, daily standup mention               │
│          │ Postmortem: Optional (team decides)                                │
├──────────┼────────────────────────────────────────────────────────────────────┤
│          │ COSMETIC or minimal impact. Workaround exists.                     │
│ SEV4     │                                                                    │
│ LOW      │ Examples at NovaMart:                                              │
│          │  • Grafana dashboard rendering slowly                              │
│          │  • Log format changed, some log parsers need update                │
│          │  • Non-critical CI pipeline flaky                                  │
│          │  • Internal tool UI bug                                            │
│          │                                                                    │
│          │ Response: Backlog ticket → Sprint planning                         │
│          │ Target MTTR: Next sprint or best-effort                            │
│          │ Postmortem: No                                                     │
└──────────┴────────────────────────────────────────────────────────────────────┘
```

### Severity Decision Flowchart

```
                    Alert fires
                        │
                        ▼
              ┌─────────────────────┐
              │ Is data being LOST  │──── YES ──→ SEV1
              │ or CORRUPTED?       │
              └─────────┬───────────┘
                        │ NO
                        ▼
              ┌─────────────────────┐
              │ Is a core revenue   │──── YES ──→ Is it TOTAL failure
              │ path affected?      │             or DEGRADED?
              │ (payments, checkout,│               │          │
              │  auth, search)      │             TOTAL     DEGRADED
              └─────────┬───────────┘               │          │
                        │ NO                       SEV1      SEV2
                        ▼
              ┌─────────────────────┐
              │ What % of users     │
              │ are affected?       │
              └─────────┬───────────┘
                        │
              ┌─────────┼──────────┐
              │         │          │
           > 50%     10-50%     < 10%
              │         │          │
            SEV2      SEV3       SEV4
                                (unless
                                 revenue
                                 impact)
```

### The Severity Escalation Anti-Pattern

```
Common failure: Everything starts as SEV3 and gets escalated to SEV1

Why this is bad:
  1. Delayed response — 30 minutes lost treating a SEV1 as SEV3
  2. Incomplete communication — stakeholders not notified
  3. Under-staffed response — only on-call working, no IC

Fix: START HIGH, DOWNGRADE IF WARRANTED.
  If you're unsure between SEV1 and SEV2, declare SEV1.
  Downgrading a SEV1 to SEV2 costs nothing.
  Upgrading a SEV3 to SEV1 after 45 minutes costs $2.25M at NovaMart.

"When in doubt, escalate." — This is a cultural value, not optional.
```

---

## Part 2: Incident Response Roles and Structure

### The Incident Command System (ICS)

Borrowed from emergency services. Adapted for tech:

```
┌──────────────────────────────────────────────────────────────────┐
│                    INCIDENT COMMAND STRUCTURE                    │
│                                                                  │
│                   ┌──────────────────┐                           │
│                   │    INCIDENT       │                          │
│                   │    COMMANDER (IC) │                          │
│                   │                  │                           │
│                   │  Owns the PROCESS│                           │
│                   │  NOT the fix     │                           │
│                   └────────┬─────────┘                           │
│                            │                                     │
│           ┌────────────────┼────────────────┐                    │
│           │                │                │                    │
│   ┌───────▼──────┐  ┌─────▼──────┐  ┌──────▼───────┐             │
│   │  TECHNICAL   │  │   COMMS    │  │  OPERATIONS  │             │
│   │    LEAD      │  │    LEAD    │  │    LEAD      │             │
│   │              │  │            │  │              │             │
│   │ Owns the FIX │  │ Owns the   │  │ Owns the     │             │
│   │ Drives debug │  │ MESSAGE    │  │ EXECUTION    │             │
│   │ Decides what │  │ Status pg  │  │ Runs commands│             │
│   │ to try next  │  │ Slack      │  │ Deploys fixes│             │
│   └───────┬──────┘  │ Exec brief │  │ Monitors     │             │
│           │         └────────────┘  └──────────────┘             │
│           │                                                      │
│    ┌──────▼───────┐                                              │
│    │  SUBJECT     │                                              │
│    │  MATTER      │                                              │
│    │  EXPERTS     │                                              │
│    │              │                                              │
│    │ Database,    │                                              │
│    │ Network,     │                                              │
│    │ App team,    │                                              │
│    │ Cloud team   │                                              │
│    └──────────────┘                                              │
└──────────────────────────────────────────────────────────────────┘
```

### Role Details

**Incident Commander (IC):**

```
DOES:                                    DOES NOT:
─────                                    ─────────
✅ Declares severity level               ❌ Debug the problem themselves
✅ Assigns roles (Tech Lead, Comms)      ❌ SSH into servers
✅ Sets priorities and timeframes        ❌ Write code fixes
✅ Makes escalation decisions            ❌ Get into the technical weeds
✅ Calls for additional resources         ❌ Talk to customers directly
✅ Decides on mitigation vs root cause   ❌ Second-guess the Tech Lead
✅ Keeps the bridge call organized       ❌ Allow side conversations
✅ Decides when to communicate           ❌ Let status updates lag
✅ Calls "mitigated" and "resolved"      ❌ Declare resolution prematurely
✅ Ensures postmortem is scheduled       ❌ Skip the postmortem

KEY SKILL: "What's the current status? What are we trying next?
           What's the ETA? Who needs to know?"

The IC is a CONDUCTOR, not a MUSICIAN.
They don't play the instruments — they ensure the orchestra plays together.
```

**Technical Lead (TL):**

```
DOES:                                    DOES NOT:
─────                                    ─────────
✅ Drives the technical investigation    ❌ Write status updates
✅ Proposes hypotheses and tests them    ❌ Talk to executives
✅ Delegates tasks to SMEs              ❌ Make business decisions
✅ Decides mitigation approach           ❌ Worry about communications
✅ Assesses risk of proposed fixes       ❌ Run commands directly (usually)
✅ Reports status to IC                  ❌ Work in silence for 20 minutes
✅ Identifies when more expertise needed

KEY SKILL: "Based on the metrics, I believe the root cause is X.
           Let's verify by checking Y. If confirmed, mitigation is Z.
           Risk of mitigation: [LOW/MED/HIGH]."
```

**Communications Lead:**

```
DOES:                                    DOES NOT:
─────                                    ─────────
✅ Updates status page (every 15 min     ❌ Debug the problem
   for SEV1, 30 min for SEV2)            ❌ Make technical decisions
✅ Posts to #incident Slack channel      ❌ Promise resolution times
✅ Briefs customer support team          ❌ Speculate on root cause externally
✅ Briefs executives (separate from      ❌ Share technical details with customers
   technical discussion)                 ❌ Use jargon in external comms
✅ Manages customer communication
✅ Drafts post-incident customer notice

STATUS PAGE TEMPLATE:
  "We are aware of [impact description].
   Our team is actively investigating.
   [Service X] may experience [specific symptom].
   We will provide an update in [15/30] minutes.
   Last updated: [timestamp]"

NEVER say:                               INSTEAD say:
  "We don't know what's happening"        "We are actively investigating"
  "It's a database problem"               "We've identified the affected area"
  "Should be fixed soon"                  "We expect to provide an update by [time]"
  "This is minor"                         "Some users may experience [symptom]"
```

**Operations Lead:**

```
DOES:                                    DOES NOT:
─────                                    ─────────
✅ Executes commands/deploys as          ❌ Decide what to try next
   directed by Tech Lead                   (that's the Tech Lead's job)
✅ Monitors dashboards during changes    ❌ Freelance debugging
✅ Confirms success/failure of actions   ❌ Make changes without announcing
✅ Documents every action taken (with    ❌ Skip the timestamp log
   timestamps)
✅ Prepares rollback before executing
✅ Announces in bridge: "Executing [X]
   in 30 seconds. Any objections?"

CRITICAL RULE: Every action is announced BEFORE execution.
  "I'm going to restart payment-svc pods. 3 replicas will restart
   with rolling strategy. Expected impact: 10-15 seconds of reduced
   capacity. Proceeding in 30 seconds unless objections."
```

### Small Team Adaptation (NovaMart Platform: 6 engineers)

```
At NovaMart's team size, one person often fills multiple roles:

SEV1 (all-hands):
  IC: Engineering Manager or Senior SRE
  Tech Lead: Most relevant engineer (database → DBA, networking → infra)
  Comms: EM or designated person
  Ops: On-call engineer
  SMEs: Paged as needed

SEV2 (2-3 people):
  IC + Comms: On-call engineer (combined)
  Tech Lead: On-call or secondary
  Ops: On-call

SEV3-4 (1 person):
  On-call handles solo, escalates if needed

Role assignment at NovaMart happens in Slack:
  IC: "I'm IC for this incident. @alice you're Tech Lead.
       @bob handle comms. @carol stand by for Ops."
```

---

## Part 3: The Incident Lifecycle

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        INCIDENT LIFECYCLE                                  │
│                                                                            │
│  DETECT → TRIAGE → MOBILIZE → INVESTIGATE → MITIGATE → RESOLVE → LEARN     │
│                                                            │               │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌────▼─┐  ┌──────┐      │
│  │Alert │→ │Sev?  │→ │Page  │→ │Debug │→ │Stop  │→ │Root  │→ │Post- │      │
│  │fires │  │Who?  │  │Team  │  │Find  │  │bleed │  │cause │  │mortem│      │
│  │      │  │      │  │Roles │  │cause │  │ing   │  │fix   │  │      │      │
│  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘      │
│                                                                            │
│  ◄── MTTD ──►◄── MTTE ──►◄───── MTTM ────►◄── MTTR ──►                     │
│  (detect)     (engage)     (mitigate)        (resolve)                     │
│                                                                            │
│  Key metrics:                                                              │
│    MTTD = Time from failure start to alert firing                          │
│    MTTE = Time from alert to first human response                          │
│    MTTM = Time from alert to user impact stopped                           │
│    MTTR = Time from alert to root cause fixed (permanent)                  │
│                                                                            │
│  ⚠️ MITIGATE FIRST, ROOT CAUSE LATER.                                      │
│     If restarting the service stops the bleeding, RESTART.                 │
│     Don't spend 30 minutes debugging while users suffer.                   │
│     Collect evidence (logs, metrics, heap dumps) THEN mitigate.            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Phase 1: Detection

```
AUTOMATED DETECTION (preferred):
  - SLO burn rate alerts (Lesson 4)
  - Prometheus alerting rules
  - Synthetic probes (health checks from external locations)
  - Log-based alerts (Loki/Grafana alerting)
  - AWS CloudWatch alarms

HUMAN DETECTION (inevitable, but try to minimize):
  - Customer support tickets
  - Social media complaints
  - Executive "the site feels slow"
  - Developer "my deploy seems weird"
  - Partner/vendor notification

DETECTION ANTI-PATTERNS:
  ❌ Customer detects before monitoring = monitoring gap
  ❌ Alert fatigue = too many low-quality alerts, real ones get ignored
  ❌ Alert on symptoms (CPU high) instead of impact (requests failing)

DETECTION METRIC: MTTD
  NovaMart targets:
    SEV1: < 2 minutes (fast burn alerts)
    SEV2: < 10 minutes (moderate burn alerts)
    SEV3: < 1 hour (slow burn alerts or manual detection)
```

### Phase 2: Triage and Mobilization

```
On-call receives alert:
  1. ACKNOWLEDGE the alert in PagerDuty (stops re-escalation)
     └── PagerDuty escalation policy:
         0 min:  Primary on-call
         5 min:  Secondary on-call (if not ACK'd)
         10 min: Engineering manager (if not ACK'd)
         15 min: VP Engineering (if not ACK'd)

  2. ASSESS severity using the flowchart
     └── Check: what's the user impact RIGHT NOW?
     └── Check: is this getting worse?
     └── Check: is this a known issue (runbook exists)?

  3. DECLARE the incident
     Slack:
       /incident declare
       Title: "Payment processing failures"
       Severity: SEV1
       IC: @oncall-primary
       
     This triggers automation:
       → Creates #inc-20240118-payments channel
       → Posts to #incidents (company-wide visibility)
       → Creates PagerDuty incident
       → Creates Jira ticket
       → Updates status page to "Investigating"
       → Starts incident timer

  4. MOBILIZE
     For SEV1: Page relevant teams immediately
       "I need a database SME and someone from the payments team
        in #inc-20240118-payments NOW."
     
     For SEV2: Ask in channel, page if no response in 5 minutes
```

### Phase 3: Investigation and Mitigation

This is where your observability stack earns its keep:

```
INVESTIGATION FRAMEWORK (USE THIS ORDER):
──────────────────────────────────────────

Step 1: SCOPE THE IMPACT (2 minutes)
  ├── What services are affected?
  ├── What percentage of users?
  ├── What's the error rate and trend? (Getting worse?)
  ├── Which regions/availability zones?
  └── When did it start? (Look at metrics, not guesses)
  
  Key queries:
    sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
    sum by (region) (rate(http_requests_total{status=~"5.."}[5m]))

Step 2: CHECK WHAT CHANGED (3 minutes)
  ├── Recent deployments (ArgoCD: any syncs in last 2 hours?)
  ├── Config changes (ConfigMap/Secret updates?)
  ├── Infrastructure changes (Terraform applies?)
  ├── Traffic pattern changes (DDoS? Traffic spike?)
  ├── Dependency status (AWS status page? Stripe? MongoDB Atlas?)
  └── Certificate expirations?
  
  Key commands:
    kubectl -n <ns> rollout history deployment/<svc>
    kubectl get events --sort-by='.lastTimestamp' -A | tail -30
    argocd app list --output wide
    # Check #deployments Slack channel

Step 3: FOLLOW THE DATA PATH (5-10 minutes)
  ├── Start at the user-facing edge (ALB metrics, Cloudflare analytics)
  ├── Move inward: API Gateway → downstream services → databases
  ├── At each hop: check error rates, latency, saturation
  ├── Use traces to find the bottleneck span
  └── Use logs for specific error messages

  Key queries:
    # Latency at each service
    histogram_quantile(0.99, sum by (le, service) (
      rate(http_request_duration_seconds_bucket[5m])
    ))
    
    # Saturation signals
    container_memory_working_set_bytes / container_spec_memory_limit_bytes
    rate(container_cpu_usage_seconds_total[5m]) / container_spec_cpu_quota

Step 4: MITIGATE (as soon as you have enough to act)
  ├── Can you rollback? → DO IT. Don't wait for root cause.
  ├── Can you restart? → DO IT (if safe — check for data loss risk).
  ├── Can you failover? → Switch traffic to healthy region/replica.
  ├── Can you scale? → Add capacity if it's a load problem.
  ├── Can you feature-flag? → Disable the problematic feature.
  └── Can you redirect? → Send traffic around the broken component.
  
  ⚠️ COLLECT EVIDENCE BEFORE MITIGATING:
    # Save state before restart
    kubectl -n payments logs deploy/payment-svc --tail=1000 > /tmp/pre-restart-logs.txt
    kubectl -n payments describe pods > /tmp/pre-restart-describe.txt
    kubectl -n payments top pods > /tmp/pre-restart-resources.txt
    # If heap dump needed:
    kubectl -n payments exec deploy/payment-svc -- jcmd 1 GC.heap_dump /tmp/heap.hprof
    kubectl -n payments cp payment-svc-xxx:/tmp/heap.hprof ./heap.hprof
    # NOW restart
    kubectl -n payments rollout restart deployment/payment-svc

Step 5: VERIFY MITIGATION
  ├── Is the error rate dropping?
  ├── Is the SLI recovering?
  ├── Are users confirming improvement?
  └── Is the mitigation stable (not oscillating)?
  
  Watch for 15 minutes minimum before declaring "mitigated."
```

### The MITIGATE FIRST Principle

```
THIS IS THE HARDEST LESSON FOR ENGINEERS:

  ❌ "I want to understand WHY before I fix it"
  ❌ "If I restart, I'll lose the debug state"
  ❌ "Let me just check one more thing..."
  ❌ "I think I know the root cause, let me verify"
  
  WHILE YOU'RE DEBUGGING, USERS ARE SUFFERING.
  AT $50K/MINUTE, EVERY MINUTE OF "LET ME JUST CHECK" COSTS $50K.

  ✅ Collect evidence (logs, dumps, metrics screenshots) → 2 minutes
  ✅ Mitigate (restart, rollback, failover) → immediately
  ✅ Verify mitigation is working → 15 minutes
  ✅ THEN investigate root cause (with the evidence you saved)

  The root cause investigation can happen AFTER user impact is resolved.
  The postmortem exists for exactly this purpose.
```

---

## Part 4: Incident Communication

### The Communication Timeline

```
MINUTE 0:  Alert fires
MINUTE 2:  On-call acknowledges in PagerDuty
MINUTE 5:  Incident declared in Slack
           Status page: "Investigating"
MINUTE 10: First Slack update with initial assessment
           "Impact: payment failures affecting ~30% of checkouts
            Scope: us-east-1 only
            Investigation: checking recent deployments"
MINUTE 15: Status page update
           "We are aware of payment processing issues affecting
            some customers. Our team is actively investigating."
MINUTE 20: Internal Slack update
           "Root cause identified: connection pool exhaustion to
            Stripe after deploy at 02:47. Rolling back now."
MINUTE 25: Status page update
           "We have identified the cause and are implementing a fix.
            Some payment attempts may still fail during recovery."
MINUTE 30: Mitigation applied
MINUTE 35: Verification — error rates returning to normal
MINUTE 45: Status page: "Monitoring"
           "The fix has been applied and payment processing is
            returning to normal. We are monitoring closely."
MINUTE 60: Confidence in stability
           Status page: "Resolved"
           "Payment processing has fully recovered. A full
            postmortem will be published within 48 hours."
```

### Slack Incident Channel Format

```
#inc-20240118-payments

📢 INCIDENT DECLARED — SEV1
━━━━━━━━━━━━━━━━━━━━━━━━━━
Title: Payment processing failures
Severity: SEV1
Start time: 2024-01-18 02:45 UTC
IC: @alice
Tech Lead: @bob  
Comms: @carol
Status page: https://status.novamart.com
Dashboard: https://grafana.novamart.internal/d/slo-payment

─── UPDATE 02:55 UTC (IC) ───
Impact: ~30% payment failures in us-east-1
Trend: Stable (not getting worse)
Current action: Checking recent deploys

─── UPDATE 03:05 UTC (Tech Lead) ───
Root cause identified: payment-svc v2.34.1 deployed at 02:47
has a connection pool misconfiguration.
Action: Rolling back to v2.34.0
ETA: 5 minutes for rollback, 10 minutes for full recovery

─── UPDATE 03:12 UTC (Ops) ───
Rollback executed: payment-svc now on v2.34.0
Error rate dropping: 30% → 12% → 5%
Monitoring for full recovery

─── UPDATE 03:25 UTC (IC) ───
✅ MITIGATED
Error rate at 0.02% (normal baseline)
SLI: 99.97% (recovering)
Monitoring for stability. Will declare resolved in 30 minutes
if stable.

─── UPDATE 03:55 UTC (IC) ───
✅ RESOLVED
Payment processing fully recovered.
Duration: 70 minutes (02:45 — 03:55)
Error budget consumed: ~4.2 minutes
Postmortem scheduled: 2024-01-19 14:00 UTC
Jira: PLAT-4521
```

### Executive Communication

```
During SEV1, executives need a DIFFERENT format than engineers:

TO: VP Engineering, CTO
FROM: Incident Commander
SUBJECT: SEV1 — Payment Processing — UPDATE 3

Current Status: MITIGATED (recovering)
User Impact: Payment failures affected ~30% of checkouts for 40 minutes
Revenue Impact: Estimated $2M in failed transactions (retries expected to recover most)
Root Cause: Configuration error in deployment (rolled back)
Current State: Error rates returning to normal, monitoring
Next Update: 30 minutes or on status change
Postmortem: Scheduled for tomorrow 2 PM

───────────────────────────────────────────────────────────
DO NOT include in executive comms:
  ❌ Technical details (connection pool settings)
  ❌ Blame ("Bob's commit caused it")
  ❌ Uncertainty ("we think maybe...")
  ❌ Jargon ("k8s pod OOMKilled the sidecar")

DO include:
  ✅ Business impact (revenue, user %, duration)
  ✅ Current status (one word: Investigating/Mitigating/Resolved)
  ✅ When next update will come
  ✅ When postmortem will happen
```

---

## Part 5: On-Call Best Practices

### On-Call Structure at NovaMart

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ON-CALL ROTATION                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Primary on-call: 1-week rotation, 6 engineers                      │
│  Secondary on-call: backup if primary unavailable                   │
│                                                                     │
│  Schedule:                                                          │
│    Week 1: Alice (primary) / Bob (secondary)                        │
│    Week 2: Bob / Carol                                              │
│    Week 3: Carol / Dave                                             │
│    Week 4: Dave / Eve                                               │
│    Week 5: Eve / Frank                                              │
│    Week 6: Frank / Alice                                            │
│    ─── repeat ───                                                   │
│                                                                     │
│  Managed in: PagerDuty                                              │
│                                                                     │
│  Escalation policy:                                                 │
│    0 min  → Primary on-call                                         │
│    5 min  → Secondary on-call                                       │
│    10 min → Engineering Manager                                     │
│    15 min → VP Engineering                                          │
│    20 min → CTO (nuclear option)                                    │
│                                                                     │
│  Handoff: Friday 10 AM local time                                   │
│    Outgoing: writes handoff doc (open incidents, known issues,      │
│              upcoming risky changes)                                │
│    Incoming: reads handoff, verifies PagerDuty access, checks       │
│              laptop/VPN/MFA working                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### On-Call Health Metrics

```
Track these to prevent burnout and improve process:

┌──────────────────────────┬────────────────────┬──────────────────────┐
│ Metric                   │ Target             │ Action if exceeded   │
├──────────────────────────┼────────────────────┼──────────────────────┤
│ Pages per week           │ < 2 per on-call    │ Tune alerts, fix     │
│ (interrupt, not ticket)  │ shift              │ noisy sources        │
├──────────────────────────┼────────────────────┼──────────────────────┤
│ After-hours pages        │ < 1 per week       │ Review: can this     │
│                          │                    │ wait until morning?  │
├──────────────────────────┼────────────────────┼──────────────────────┤
│ Mean time to ACK         │ < 5 minutes        │ Check: phone on      │
│                          │                    │ silent? Bad process? │
├──────────────────────────┼────────────────────┼──────────────────────┤
│ Alert-to-mitigate time   │ < 30 min (SEV1)    │ Improve runbooks     │
│                          │ < 2 hrs (SEV2)     │                      │
├──────────────────────────┼────────────────────┼──────────────────────┤
│ False positive rate      │ < 10% of pages     │ Alert tuning sprint  │
│ (pages that need no      │                    │                      │
│  action)                 │                    │                      │
├──────────────────────────┼────────────────────┼──────────────────────┤
│ On-call satisfaction     │ > 3.5/5 (survey)   │ Process review,      │
│ (post-rotation survey)   │                    │ automation invest    │
├──────────────────────────┼────────────────────┼──────────────────────┤
│ Toil ratio               │ < 50% of on-call   │ Automate repetitive  │
│ (manual repetitive work  │ time               │ responses            │
│  vs engineering work)    │                    │                      │
└──────────────────────────┴────────────────────┴──────────────────────┘
```

### On-Call Anti-Patterns

```
ANTI-PATTERN 1: "Hero culture"
  One engineer takes all the pages because they're "the expert"
  → They burn out. The team never learns.
  Fix: Rotate strictly. No exceptions. Document knowledge in runbooks.

ANTI-PATTERN 2: "The team doesn't own their service"
  Service team ships code → Platform team gets paged
  → Perverse incentive: no consequences for shipping broken code
  Fix: "You build it, you run it." Service teams share on-call for
       their services. Platform supports infrastructure, not app bugs.

ANTI-PATTERN 3: "Alert noise"
  50 alerts per shift, 45 are false positives or need no action
  → On-call ignores alerts → misses the real ones
  Fix: Every alert must be actionable. If you get paged and the answer
       is "do nothing," that alert is BROKEN. Fix it or delete it.
       Track false positive rate. Target: < 10%.

ANTI-PATTERN 4: "No runbook"
  Alert fires → on-call has no idea what it means or what to do
  → Scrambles for 30 minutes before figuring out the context
  Fix: Every alert MUST have a runbook link. If there's no runbook,
       don't create the alert (or create the runbook first).

ANTI-PATTERN 5: "On-call as punishment"
  On-call is seen as a burden, assigned to junior engineers
  → Juniors are overwhelmed, incidents are mishandled
  Fix: On-call is a professional responsibility. Compensate fairly
       (on-call bonus or comp time). Senior engineers must participate.

ANTI-PATTERN 6: "Eternal on-call"
  Engineer is never really off-call (checks Slack at dinner, answers
  pages on weekends even when not on rotation)
  → Burnout. No work-life boundary.
  Fix: When you're off-call, you're OFF-CALL. Don't check alerts.
       The secondary and escalation chain exists for a reason.
```

### On-Call Compensation at NovaMart

```
On-call expectations:
  - Respond to pages within 5 minutes (ACK in PagerDuty)
  - Have laptop, VPN, and MFA available at all times during shift
  - Stay within reliable internet coverage
  - No alcohol or substances that impair response capability

Compensation:
  - $500/week on-call bonus (whether or not paged)
  - After-hours page: 2 hours comp time per page (minimum)
  - SEV1 response after hours: half-day comp time next day
  - If paged > 3 times in one night: next day off (no questions)
```

---

## Part 6: Runbooks

### Runbook Structure

```yaml
# runbook-payment-svc-high-error-rate.md

# Payment Service

Continuing exactly from where I stopped — Part 6: Runbooks, Runbook Structure.

---

```yaml
# runbook-payment-svc-high-error-rate.md

# Payment Service — High Error Rate

## Metadata
- **Alert:** PaymentSvcHighBurnRate_Page
- **Severity:** SEV1/SEV2 (depends on burn rate)
- **Service:** payment-svc
- **Team:** Payments (#team-payments)
- **Escalation:** @payments-oncall → @payments-lead → @vp-engineering
- **Last updated:** 2024-01-15
- **Author:** Platform Engineering
- **Dashboard:** https://grafana.novamart.internal/d/slo-payment
- **SLO:** 99.95% availability, 99.9% < 2s latency

## Symptoms
- PagerDuty alert: burn rate > 14.4x (fast burn) or > 6x (moderate burn)
- Grafana dashboard shows error rate spike on /charge endpoint
- Customer support receiving "payment failed" complaints
- Error budget consumption accelerating

## Impact
- Users cannot complete purchases
- Revenue loss: ~$50K/minute at peak traffic
- Customer trust degradation (payment failures are high-impact)

## Quick Assessment (2 minutes)

### 1. Confirm the impact is real
```bash
# Current error rate
curl -s 'http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total{service="payment-svc",endpoint="/charge",status=~"5.."}[5m]))/sum(rate(http_requests_total{service="payment-svc",endpoint="/charge"}[5m]))' | jq '.data.result[0].value[1]'

# Current burn rate
curl -s 'http://prometheus:9090/api/v1/query?query=sli:payment_charge_availability:burnrate1h' | jq '.data.result[0].value[1]'

# Error budget remaining
curl -s 'http://prometheus:9090/api/v1/query?query=sli:payment_charge_availability:error_budget_remaining' | jq '.data.result[0].value[1]'
```

### 2. Check what changed recently
```bash
# Recent deployments
kubectl -n payments rollout history deployment/payment-svc | tail -5

# Recent config changes
kubectl -n payments get events --sort-by='.lastTimestamp' | grep -E 'ConfigMap|Secret|Deployment' | tail -10

# ArgoCD sync status
argocd app get payment-svc --output json | jq '{sync: .status.sync.status, health: .status.health.status, lastSync: .status.operationState.finishedAt}'
```

### 3. Check error breakdown
```bash
# Errors by status code
curl -s 'http://prometheus:9090/api/v1/query?query=sum+by+(status)(rate(http_requests_total{service="payment-svc",endpoint="/charge",status=~"5.."}[5m]))' | jq '.data.result[] | {status: .metric.status, rate: .value[1]}'

# Errors by downstream dependency
curl -s 'http://prometheus:9090/api/v1/query?query=sum+by+(upstream_service)(rate(http_requests_total{service="payment-svc",status=~"5.."}[5m]))' | jq '.data.result[]'
```

## Diagnosis Decision Tree

```
Error code is 500 (Internal Server Error)?
├── YES → Application bug or unhandled exception
│   ├── Check logs: kubectl -n payments logs deploy/payment-svc --tail=200 | grep -i error
│   ├── Look for stack traces, NPE, OOM
│   ├── If recent deploy → ROLLBACK (see Mitigation Option 1)
│   └── If no recent deploy → Check resource exhaustion (memory, connections)
│
Error code is 502 (Bad Gateway)?
├── YES → payment-svc pods are crashing or not responding
│   ├── Check pod status: kubectl -n payments get pods
│   ├── Check for OOMKilled: kubectl -n payments describe pods | grep -A3 "Last State"
│   ├── Check readiness probe: kubectl -n payments describe pods | grep -A5 Readiness
│   └── If pods are CrashLooping → check logs for startup errors
│
Error code is 503 (Service Unavailable)?
├── YES → Upstream dependency is down or circuit breaker is open
│   ├── Check Stripe status: https://status.stripe.com
│   ├── Check internal dependencies:
│   │   kubectl -n orders get pods (order-svc)
│   │   kubectl -n inventory get pods (inventory-svc)
│   │   kubectl -n users get pods (user-svc)
│   ├── Check circuit breaker state:
│   │   curl http://payment-svc:8080/actuator/health (Spring Boot)
│   │   Check Istio circuit breaker:
│   │   istioctl proxy-config cluster deploy/payment-svc -n payments | grep circuit
│   └── If Stripe is down → Mitigation Option 3 (queue or failover)
│
Error code is 504 (Gateway Timeout)?
├── YES → Downstream call is too slow
│   ├── Check latency per dependency:
│   │   Trace search in Tempo for slow spans
│   ├── Check database: RDS Performance Insights
│   ├── Check connection pools:
│   │   payment_db_connections_active / payment_db_connections_max
│   └── If database slow → Check for long-running queries, locks
```

## Mitigation Options

### Option 1: Rollback (if recent deploy caused it)
```bash
# Check last deployment
kubectl -n payments rollout history deployment/payment-svc

# Rollback to previous version
kubectl -n payments rollout undo deployment/payment-svc

# OR via ArgoCD (preferred — maintains GitOps)
# Revert the commit in the manifests repo, ArgoCD will sync

# Verify rollback
kubectl -n payments rollout status deployment/payment-svc
kubectl -n payments get pods -o wide
# Wait 2-3 minutes, then check error rate
```

### Option 2: Restart (if not deployment-related)
```bash
# FIRST: Collect evidence
kubectl -n payments logs deploy/payment-svc --tail=2000 > /tmp/pre-restart-payment-logs.txt
kubectl -n payments describe pods > /tmp/pre-restart-payment-describe.txt
kubectl -n payments top pods > /tmp/pre-restart-payment-resources.txt

# Restart with rolling strategy (no downtime)
kubectl -n payments rollout restart deployment/payment-svc

# Monitor the restart
kubectl -n payments rollout status deployment/payment-svc --watch
```

### Option 3: Failover (if regional/AZ issue)
```bash
# Check if issue is regional
# If us-east-1 is affected:
# Update Route53 health check to failover traffic
aws route53 update-health-check \
  --health-check-id <id> \
  --disabled

# Or adjust weighted routing to shift traffic
aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch file://failover-to-west.json
```

### Option 4: Scale (if capacity issue)
```bash
# Scale up payment-svc
kubectl -n payments scale deployment/payment-svc --replicas=10

# If node capacity is insufficient
# Karpenter will auto-provision, but verify:
kubectl get nodes --sort-by='.metadata.creationTimestamp' | tail -5
kubectl get provisioners -o yaml  # Check Karpenter limits
```

## Verification After Mitigation
```bash
# Error rate should be dropping
watch -n 10 'curl -s "http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total{service=\"payment-svc\",endpoint=\"/charge\",status=~\"5..\"}[2m]))/sum(rate(http_requests_total{service=\"payment-svc\",endpoint=\"/charge\"}[2m]))" | jq -r ".data.result[0].value[1]" | xargs printf "Error rate: %s\n"'

# Burn rate should be dropping below threshold
watch -n 10 'curl -s "http://prometheus:9090/api/v1/query?query=sli:payment_charge_availability:burnrate1h" | jq -r ".data.result[0].value[1]" | xargs printf "Burn rate: %sx\n"'

# Monitor for 15 minutes minimum before declaring mitigated
```

## Escalation
- If mitigation doesn't work within 15 minutes → page payments tech lead
- If dependency (Stripe) is confirmed down → page VP Engineering for business decision
- If data corruption suspected → page database team AND VP Engineering immediately
```

### Runbook Quality Checklist

```
Every runbook MUST have:
  ✅ Alert name it corresponds to (1:1 mapping)
  ✅ Dashboard link (one click to see the problem)
  ✅ Copy-paste commands (not "check the pods" — give the exact kubectl)
  ✅ Decision tree (not just "investigate" — give the logical flow)
  ✅ Multiple mitigation options (not just "rollback")
  ✅ Verification steps (how do you KNOW the fix worked?)
  ✅ Escalation path (who to call if you can't fix it)
  ✅ Last updated date (stale runbooks are dangerous)

Every runbook MUST NOT have:
  ❌ Vague instructions ("check if things look normal")
  ❌ Missing context (assumes tribal knowledge)
  ❌ Only one mitigation path (what if rollback doesn't work?)
  ❌ No verification (how do you know you didn't make it worse?)

TESTING RUNBOOKS:
  A runbook is only as good as its last test.
  - New engineer test: Can someone who's never seen this service
    follow the runbook and mitigate the incident?
  - Game day test: Inject the failure in staging, follow the runbook.
  - Post-incident update: After every incident, update the runbook
    with what was actually done vs. what the runbook said.
```

---

## Part 7: Blameless Postmortems

### Why Blameless?

```
BLAMEFUL POSTMORTEM:                     BLAMELESS POSTMORTEM:
────────────────────                     ──────────────────────
"Bob pushed a bad config"                "A config change was pushed without
                                          validation because our pipeline
                                          lacked config linting"

"The DBA should have caught it"          "The review process didn't include
                                          database impact assessment"

"If Alice had been faster, we'd          "Our detection took 15 minutes
 have caught it sooner"                   because we lacked an SLI for
                                          this failure mode"

Result: People hide mistakes             Result: People report freely
        Information is suppressed                 Real causes are found
        Same incident repeats                     Systemic fixes are made
        Culture of fear                           Culture of learning
```

**The fundamental insight:**

```
Humans make mistakes. That's a CONSTANT, not a variable.

If a human error caused an outage, the question is NOT "who did it?"
The question is: "Why did our SYSTEM allow a human error to reach production?"

Every human error in production represents a SYSTEM failure:
  - Missing validation (linting, tests, policy checks)
  - Missing guardrails (canary, rollback automation)
  - Missing review (4-eyes principle, approval gates)
  - Missing detection (monitoring gap)
  - Missing training (documentation, runbooks)

Fix the SYSTEM. The human was the trigger, not the cause.
```

### Postmortem Structure

```markdown
# Postmortem: Payment Processing Failures — 2024-01-18

## Metadata
- **Incident ID:** INC-2024-0118-001
- **Date:** 2024-01-18
- **Duration:** 70 minutes (02:45 — 03:55 UTC)
- **Severity:** SEV1
- **IC:** Alice Chen
- **Tech Lead:** Bob Kim
- **Author:** Alice Chen
- **Reviewers:** Platform team + Payments team
- **Status:** Action items in progress

## Executive Summary
On January 18 at 02:45 UTC, payment processing experienced a 70-minute 
degradation. ~30% of checkout attempts failed during the incident.
Root cause: a connection pool misconfiguration in payment-svc v2.34.1 
deployed at 02:47 UTC. Mitigation: rollback to v2.34.0 at 03:12 UTC.
Estimated revenue impact: $2.1M in failed transactions (most recovered 
via customer retry).

## Impact
- **User impact:** 30% of payment attempts returned errors for 70 minutes
- **Error budget consumed:** 4.2 minutes (of 21.6 min monthly budget)
- **Revenue impact:** ~$2.1M in initially failed transactions
- **Recovery:** 85% of affected users retried successfully within 2 hours
- **Support tickets:** 342 payment-related tickets (vs ~15 baseline)
- **SLO status:** 99.91% over 30d window (still above 99.95% target due 
  to remaining budget from clean first 18 days)

## Timeline (ALL TIMES UTC)

| Time  | Event |
|-------|-------|
| 02:47 | payment-svc v2.34.1 deployed via ArgoCD auto-sync |
| 02:49 | Connection pool begins exhausting (new config: max 5 connections, previous: 50) |
| 02:51 | First 503 errors appear in logs |
| 02:53 | Error rate exceeds 14.4x burn rate threshold |
| 02:55 | **ALERT FIRES:** PaymentSvcHighBurnRate_Page |
| 02:57 | On-call (Alice) acknowledges, opens incident channel |
| 02:58 | SEV1 declared, Bob paged as Tech Lead |
| 03:02 | Bob identifies v2.34.1 deployed 15 minutes ago |
| 03:05 | Bob reviews diff — finds connection pool change (50 → 5) |
| 03:07 | Decision: rollback to v2.34.0 |
| 03:08 | Evidence collected (logs, pod describe, connection metrics) |
| 03:12 | Rollback executed via ArgoCD |
| 03:15 | New pods passing readiness checks, error rate dropping |
| 03:25 | Error rate at baseline (0.02%). IC declares MITIGATED |
| 03:55 | 30 minutes stable. IC declares RESOLVED |

## Root Cause Analysis

### Direct cause:
payment-svc v2.34.1 included a configuration change that reduced the 
database connection pool from 50 to 5 connections. At NovaMart's traffic 
level (~800 req/s to payment-svc), 5 connections are insufficient, 
causing connection wait timeouts that manifested as 503 errors to callers.

### The configuration change:
```yaml
# v2.34.0 (working)
database:
  maxOpenConns: 50
  maxIdleConns: 25

# v2.34.1 (broken)
database:
  maxOpenConns: 5    # Changed for "dev environment optimization"
  maxIdleConns: 2    # Corresponding reduction
```

### Why the bad config reached production:
1. The config change was intended for the dev environment only, but was 
   committed to the shared `values.yaml` instead of `values-dev.yaml`
2. The PR review did not catch the environment-specific nature of the change
3. No automated validation exists for connection pool settings
4. ArgoCD auto-sync deployed it without human approval for production

### Contributing factors:
- No config validation in CI pipeline (no check for "is maxOpenConns 
  reasonable for production traffic?")
- PR review focused on code logic, not configuration values
- No canary deployment for config-only changes
- Auto-sync enabled for production (bypasses manual approval)

## Detection Analysis
- **MTTD:** 8 minutes (02:47 deploy → 02:55 alert)
  - Could this be faster? The burn rate alert requires 5m of data. 
    A raw error rate alert would have fired at 02:53 (~6 min)
  - Connection pool metrics would have shown immediate exhaustion 
    but we don't alert on pool utilization directly
- **MTTE:** 2 minutes (02:55 alert → 02:57 acknowledge)
  - Within SLA. Good.
- **MTTM:** 30 minutes (02:55 alert → 03:25 mitigated)
  - Acceptable for SEV1. Could improve with automated rollback.

## What Went Well
- On-call responded within 2 minutes
- Root cause identified quickly (deployment correlation)
- Decision to rollback was made promptly (no "let me debug more" delay)
- Evidence was collected BEFORE rollback
- Communication to stakeholders was timely
- SLO framework correctly identified the severity

## What Went Wrong
- Dev-targeted config change reached production
- No config validation in CI
- Auto-sync bypassed human approval for production changes
- 8-minute detection time (users experienced errors for 8 minutes 
  before we knew)

## Where We Got Lucky
- The on-call engineer had recently worked on payment-svc and immediately 
  suspected the deploy. A less familiar engineer might have spent 15+ 
  minutes before checking deployment history.
- This happened at 3 AM (low traffic). At peak (2 PM), the impact 
  would have been 5x worse.

## Action Items

| ID | Action | Owner | Priority | Due | Status |
|----|--------|-------|----------|-----|--------|
| 1 | Add config linting to CI: validate maxOpenConns >= 20 for production | Bob | P1 | 2024-01-25 | In Progress |
| 2 | Separate per-environment values files (values-dev.yaml, values-staging.yaml, values-prod.yaml) | Carol | P1 | 2024-01-25 | Not Started |
| 3 | Disable ArgoCD auto-sync for production; require manual sync or PR-triggered sync | Alice | P1 | 2024-01-22 | Done |
| 4 | Add connection pool utilization alert (>80% for >2min → warning) | Dave | P2 | 2024-02-01 | Not Started |
| 5 | Implement automated rollback on error rate spike (Argo Rollouts analysis) | Platform | P2 | 2024-02-15 | Not Started |
| 6 | Add "Where We Got Lucky" section awareness: create runbook entry for "check recent deploys" as step 1 | Alice | P3 | 2024-01-30 | Not Started |

## Lessons Learned
1. Configuration changes are as dangerous as code changes — they need 
   the same level of review and validation
2. Environment-specific configs must be physically separated, not just 
   logically separated within a single file
3. Auto-sync to production without human approval is a loaded gun
4. "Where We Got Lucky" analysis reveals hidden risks — if luck is 
   required for success, the system is fragile
```

### Postmortem Anti-Patterns

```
ANTI-PATTERN 1: "5 Whys" without stopping
  Why did payments fail? → Bad config
  Why was config bad? → Developer mistake
  Why did developer make a mistake? → Tired
  Why were they tired? → Overworked
  Why overworked? → Understaffed
  
  This leads to "hire more people" for every postmortem.
  
  FIX: Stop at the ACTIONABLE level.
    "Config validation was missing from CI"
    → Action: Add config linting
    This is specific, actionable, and preventive.

ANTI-PATTERN 2: No action items
  "We discussed the incident and learned from it."
  → Nothing changes. Same incident repeats.
  
  FIX: Every postmortem MUST produce at least one P1 action item
       with an owner and due date. Track to completion.

ANTI-PATTERN 3: Action items that never get done
  "Add better monitoring" — assigned to nobody, no due date
  → Still undone 6 months later
  
  FIX: Action items in Jira, tracked in sprint planning, reviewed
       in monthly SLO meeting. P1 items due within 1 week.

ANTI-PATTERN 4: Blaming the human
  "Action item: Retrain Bob on configuration management"
  → Bob feels terrible. Nobody reports future mistakes.
  → The system that allowed the mistake is unchanged.
  
  FIX: "Action item: Add automated config validation so that
        incorrect connection pool values cannot pass CI."
  → Fixes the system. Bob is relieved. Everyone reports freely.

ANTI-PATTERN 5: Skipping the postmortem
  "It was a quick fix, we know what happened, no need to write it up."
  → The fix addresses the symptom, not the system.
  → No institutional memory. New team members repeat the same mistakes.
  
  FIX: SEV1 and SEV2 postmortems are MANDATORY. No exceptions.
       SEV3 postmortems are optional but encouraged.
       Postmortem completion is tracked as a team metric.

ANTI-PATTERN 6: "Where We Got Lucky" is missing
  Most postmortems cover what went wrong and what went well.
  Few cover what could have been MUCH worse but wasn't.
  
  "Where We Got Lucky" reveals hidden fragilities:
    - "Lucky it was 3 AM" → What's our plan for peak-hour incidents?
    - "Lucky the expert was on-call" → Is the runbook good enough 
       for any engineer?
    - "Lucky only one service was affected" → What if the blast radius 
       were larger?
  
  These lead to PROACTIVE improvements, not just reactive fixes.
```

### The Postmortem Meeting

```
FORMAT: 60 minutes, within 48 hours (SEV1) or 1 week (SEV2)

ATTENDEES:
  - All incident responders
  - Service owner/tech lead
  - Engineering manager
  - Optional: VP Engineering (for SEV1), affected team leads

AGENDA:
  00-05: IC presents timeline (facts only, no interpretation)
  05-15: Walk through the root cause analysis
  15-25: Open discussion: "What surprised you?"
  25-35: Review proposed action items
  35-45: Prioritize and assign action items
  45-55: "Where We Got Lucky" discussion
  55-60: Next steps, publication timeline

GROUND RULES:
  1. No blame. If you hear "Bob should have..." redirect to
     "What system change would have prevented this?"
  2. Facts first, opinions second.
  3. Everyone's perspective is valuable — the person who caused
     the incident often has the best insight into prevention.
  4. Action items must be SMART (Specific, Measurable, Assignable,
     Relevant, Time-bound)
  5. The postmortem is published team-wide (or company-wide for SEV1)
     → Transparency builds trust and shares learning

FACILITATION TIPS:
  - If discussion goes to blame: "I hear a system improvement
    opportunity. How would we make this impossible to repeat?"
  - If discussion goes to rabbit holes: "Let's parking-lot this
    and focus on the action items."
  - If action items are vague: "Who owns this? By when?
    How will we know it's done?"
```

---

## Part 8: Chaos Engineering

### Why Break Things on Purpose?

```
"Everything fails, all the time." — Werner Vogels, CTO of AWS

You can either:
  A) Discover failure modes during a 3 AM SEV1 with users screaming
  B) Discover failure modes during a controlled 2 PM experiment

Chaos engineering is option B.

                    ┌─────────────────────────────┐
                    │     CONFIDENCE MODEL        │
                    │                             │
                    │  Without chaos:             │
                    │  "We THINK the system is    │
                    │   resilient"                │
                    │                             │
                    │  With chaos:                │
                    │  "We KNOW the system        │
                    │   survives [specific        │
                    │   failure mode]"            │
                    │  OR                         │
                    │  "We KNOW the system        │
                    │   DOESN'T survive [failure] │
                    │   and here's the fix"       │
                    └─────────────────────────────┘
```

### Chaos Engineering Process

```
Step 1: DEFINE STEADY STATE
  What does "normal" look like? (SLIs!)
    - Payment success rate: 99.97%
    - p99 latency: 400ms
    - Order throughput: 200 orders/min

Step 2: HYPOTHESIZE
  "We believe that if [failure occurs], [system behavior]"
  
  Example:
  "We believe that if one payment-svc pod is killed,
   the remaining pods will absorb the traffic with
   < 1% increase in error rate and < 200ms increase in p99 latency"

Step 3: INTRODUCE THE FAILURE
  Kill the pod, partition the network, exhaust the CPU,
  fail the database, corrupt the cache

Step 4: OBSERVE
  Did the system behave as hypothesized?
  Check SLIs, dashboards, alerts, logs

Step 5: LEARN
  If hypothesis confirmed → increase scope (kill 2 pods, 
  fail entire AZ, etc.)
  If hypothesis rejected → FIX THE WEAKNESS, then re-test

CRITICAL RULE: Start small, in staging, during business hours,
with everyone watching. Graduate to production only after
staging experiments pass consistently.
```

### Chaos Tools

```
┌─────────────────┬──────────────────────────────────────────────────┐
│ Tool            │ What it does                                     │
├─────────────────┼──────────────────────────────────────────────────┤
│ Litmus Chaos    │ K8s-native chaos engineering platform            │
│                 │ ChaosEngine CRD → injects failures               │
│                 │ Pod kill, network partition, CPU stress,         │
│                 │ disk fill, DNS failure                           │
│                 │ Integrates with Argo Workflows                   │
│                 │ NovaMart's choice (K8s native, open source)      │
├─────────────────┼──────────────────────────────────────────────────┤
│ Chaos Mesh      │ Similar to Litmus, K8s-native                    │
│                 │ Strong network chaos (partition, delay, loss)    │
│                 │ Time skew injection (clock manipulation)         │
│                 │ Good dashboard/UI                                │
├─────────────────┼──────────────────────────────────────────────────┤
│ Gremlin         │ Commercial chaos platform                        │
│                 │ Enterprise features (RBAC, audit, scheduling)    │
│                 │ Supports K8s, VMs, containers, cloud services    │
│                 │ "Failure as a Service"                           │
├─────────────────┼──────────────────────────────────────────────────┤
│ AWS Fault       │ AWS-native chaos (FIS)                           │
│ Injection       │ EC2, ECS, EKS, RDS, network                      │
│ Simulator       │ Integrates with CloudWatch for stop conditions   │
│                 │ Good for AWS-specific failures (AZ outage sim)   │
├─────────────────┼──────────────────────────────────────────────────┤
│ Netflix Chaos   │ The OG — random instance termination             │
│ Monkey          │ "If your service can't survive a random pod kill │
│                 │  it's not production-ready"                      │
│                 │ Concept more important than the specific tool    │
└─────────────────┴──────────────────────────────────────────────────┘
```

### Litmus Chaos Example at NovaMart

```yaml
# Experiment: Kill a payment-svc pod, verify resilience
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: payment-svc-pod-kill
  namespace: payments
spec:
  appinfo:
    appns: payments
    applabel: app=payment-svc
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"           # Kill pods for 60 seconds
            - name: CHAOS_INTERVAL
              value: "10"           # Kill one every 10 seconds
            - name: FORCE
              value: "false"        # Graceful termination (SIGTERM first)
            - name: PODS_AFFECTED_PERC
              value: "50"           # Kill up to 50% of pods
        probe:
          - name: payment-svc-availability
            type: promProbe
            mode: Continuous
            runProperties:
              probeTimeout: 5
              retry: 2
              interval: 5
              probePollingInterval: 2
            promProbe/inputs:
              endpoint: http://prometheus.monitoring:9090
              query: |
                sum(rate(http_requests_total{service="payment-svc",
                  endpoint="/charge", status=~"5.."}[1m]))
                /
                sum(rate(http_requests_total{service="payment-svc",
                  endpoint="/charge"}[1m]))
              comparator:
                type: float
                criteria: "<="
                value: "0.01"       # Error rate must stay under 1%
```

```yaml
# Experiment: Network latency to database
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: payment-svc-network-latency
  namespace: payments
spec:
  appinfo:
    appns: payments
    applabel: app=payment-svc
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: NETWORK_INTERFACE
              value: "eth0"
            - name: TARGET_CONTAINER
              value: "payment-svc"
            - name: NETWORK_LATENCY
              value: "500"          # Add 500ms latency
            - name: JITTER
              value: "100"          # ±100ms jitter
            - name: TOTAL_CHAOS_DURATION
              value: "120"          # 2 minutes
            - name: DESTINATION_IPS
              value: "10.0.3.0/24"  # Database subnet only
        probe:
          - name: payment-latency-sli
            type: promProbe
            mode: Continuous
            promProbe/inputs:
              endpoint: http://prometheus.monitoring:9090
              query: |
                histogram_quantile(0.99,
                  sum by (le) (rate(http_request_duration_seconds_bucket{
                    service="payment-svc", endpoint="/charge"
                  }[1m]))
                )
              comparator:
                type: float
                criteria: "<="
                value: "3.0"        # p99 must stay under 3s
```

### Game Day Format at NovaMart

```
QUARTERLY GAME DAY
══════════════════

Schedule: First Wednesday of every quarter, 10 AM - 4 PM

Preparation (1 week before):
  ├── Select 3-5 experiments (escalating severity)
  ├── Review hypotheses with service owners
  ├── Verify staging experiments pass
  ├── Notify customer support (just in case)
  ├── Prepare abort procedures
  └── Ensure all participants have dashboard access

Game Day Schedule:
  10:00  Kickoff — review experiments, assign observers
  10:30  Experiment 1: Kill 1 pod in payment-svc (warm-up)
  11:00  Experiment 2: Network latency to database (50ms → 500ms)
  11:30  Debrief experiments 1-2
  12:00  Lunch
  13:00  Experiment 3: Kill entire AZ (drain all nodes in us-east-1a)
  14:00  Experiment 4: Redis cluster failover simulation
  14:30  Experiment 5: DNS failure for inventory-svc
  15:00  Debrief all experiments
  15:30  Document findings, create action items
  16:00  Wrap-up, schedule follow-up

Rules:
  ✅ Start in staging (experiments 1-2), graduate to production (3-5)
  ✅ IC assigned for each experiment
  ✅ Abort button ready — halt immediately if unexpected production impact
  ✅ SLO dashboard visible to everyone at all times
  ✅ Record everything — screen recordings of debugging sessions
  ❌ No "gotcha" experiments — everyone knows what's being tested
  ❌ No experiments during peak traffic (schedule around business hours)
  ❌ Never target data-layer chaos in production without extensive staging
     (database kill, storage corruption = potential data loss)

Output:
  - Experiment results document (hypothesis confirmed/rejected)
  - New action items for discovered weaknesses
  - Updated runbooks with newly discovered failure modes
  - Confidence score per service (% of experiments passed)
```

---

## Part 9: Putting It All Together — NovaMart Incident Management Maturity

```
┌─────────────────────────────────────────────────────────────────────────┐
│              INCIDENT MANAGEMENT MATURITY MODEL                         │
├──────────┬──────────────────────────────────────────────────────────────┤
│ Level 1  │ REACTIVE                                                     │
│ (Bad)    │ Customers find issues. No severity levels. No roles.         │
│          │ No postmortems. Same incidents repeat.                       │
├──────────┼──────────────────────────────────────────────────────────────┤
│ Level 2  │ DEFINED                                                      │
│ (OK)     │ Severity levels defined. On-call rotation exists.            │
│          │ Alerts fire for major issues. Postmortems sometimes done.    │
│          │ Communication is ad-hoc.                                     │
├──────────┼──────────────────────────────────────────────────────────────┤
│ Level 3  │ MANAGED                                                      │
│ (Good)   │ SLOs drive alerting. IC process is followed. Postmortems     │
│          │ are mandatory. Runbooks exist for common alerts.             │
│          │ Communication templates. On-call health tracked.             │
│          │                                                              │
│          │ ← NovaMart target (current state: between 2 and 3)           │
├──────────┼──────────────────────────────────────────────────────────────┤
│ Level 4  │ PROACTIVE                                                    │
│ (Great)  │ Chaos engineering regular. Game days quarterly.              │
│          │ Automated remediation for known failures. Runbooks tested.   │
│          │ On-call is sustainable. Postmortem action items tracked      │
│          │ to completion. Cross-team learning.                          │
├──────────┼──────────────────────────────────────────────────────────────┤
│ Level 5  │ OPTIMIZED                                                    │
│ (Elite)  │ Incident prediction (ML on metrics). Self-healing systems.   │
│          │ Chaos engineering in production daily. Near-zero toil.       │
│          │ Incidents are rare AND well-handled when they occur.         │
│          │ Google/Netflix/Meta level.                                   │
└──────────┴──────────────────────────────────────────────────────────────┘
```

### NovaMart's Incident Management Stack

```
┌───────────────────────────────────────────────────────────────────┐
│                NOVAMART INCIDENT MANAGEMENT FLOW                  │
│                                                                   │
│  DETECTION                                                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐   │
│  │ Prometheus/     │  │ Synthetic       │  │ Customer         │   │
│  │ Grafana Alerts  │  │ Probes          │  │ Reports          │   │
│  │ (SLO burn rate) │  │ (external check)│  │ (support/social) │   │
│  └───────┬─────────┘  └───────┬─────────┘  └───────┬──────────┘   │
│          │                   │                     │              │
│          └───────────┬───────┘─────────────────────┘              │
│                      ▼                                            │
│  ROUTING                                                          │
│  ┌──────────────────────────────┐                                 │
│  │       PagerDuty              │                                 │
│  │  ┌──────────────────────┐    │                                 │
│  │  │ Alertmanager webhook │    │                                 │
│  │  │ → PagerDuty API      │    │                                 │
│  │  │ → Escalation policy  │    │                                 │
│  │  │ → Phone/SMS/Push     │    │                                 │
│  │  └──────────────────────┘    │                                 │
│  └──────────────┬───────────────┘                                 │
│                 ▼                                                 │
│  RESPONSE                                                         │
│  ┌───────────────────────────────────────────┐                    │
│  │              Slack                        │                    │
│  │  /incident declare                        │                    │
│  │    → #inc-YYYYMMDD-<name> channel created │                    │
│  │    → Roles assigned (IC, TL, Comms)       │                    │
│  │    → Status page updated automatically    │                    │
│  │    → Jira ticket created                  │                    │
│  │    → Timeline bot starts recording        │                    │
│  └──────────────────┬────────────────────────┘                    │
│                     ▼                                             │
│  TOOLING DURING INCIDENT                                          │
│  ┌────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐    │
│  │ Grafana    │ │ Tempo    │ │ Loki     │ │ kubectl/argocd   │    │
│  │ Dashboards │ │ Traces   │ │ Logs     │ │ CLI tools        │    │
│  └────────────┘ └──────────┘ └──────────┘ └──────────────────┘    │
│                     ▼                                             │
│  POST-INCIDENT                                                    │
│  ┌──────────────────────────────────────────┐                     │
│  │  Postmortem (Confluence/Google Docs)     │                     │
│  │    → Template auto-generated             │                     │
│  │    → Timeline pre-populated from Slack   │                     │
│  │    → Action items → Jira                 │                     │
│  │    → Published to #postmortems channel   │                     │
│  │    → Reviewed in monthly SLO meeting     │                     │
│  └──────────────────────────────────────────┘                     │
└───────────────────────────────────────────────────────────────────┘
```

### PagerDuty Configuration at NovaMart

```yaml
# PagerDuty service configuration (conceptual — PagerDuty uses UI/API)

service: "Payment Service - Production"
escalation_policy:
  rules:
    - targets:
        - type: schedule
          id: "platform-primary-oncall"
      escalation_delay_in_minutes: 5
    - targets:
        - type: schedule
          id: "platform-secondary-oncall"
      escalation_delay_in_minutes: 5
    - targets:
        - type: user
          id: "engineering-manager"
      escalation_delay_in_minutes: 5
    - targets:
        - type: user
          id: "vp-engineering"
      escalation_delay_in_minutes: 5

# Alertmanager → PagerDuty integration
# In alertmanager.yml:
receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: '<pagerduty-integration-key>'
        severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
        description: '{{ .CommonAnnotations.summary }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          dashboard: '{{ .CommonAnnotations.dashboard_url }}'
          runbook: '{{ .CommonAnnotations.runbook_url }}'
          slo: '{{ .CommonLabels.slo }}'
        links:
          - href: '{{ .CommonAnnotations.dashboard_url }}'
            text: 'Grafana Dashboard'
          - href: '{{ .CommonAnnotations.runbook_url }}'
            text: 'Runbook'

  - name: 'slack-warnings'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/xxx'
        channel: '#alerts-platform'
        title: '{{ .CommonAnnotations.summary }}'
        text: >-
          *Alert:* {{ .CommonLabels.alertname }}
          *Severity:* {{ .CommonLabels.severity }}
          *Service:* {{ .CommonLabels.service }}
          {{ .CommonAnnotations.description }}
          <{{ .CommonAnnotations.runbook_url }}|Runbook> |
          <{{ .CommonAnnotations.dashboard_url }}|Dashboard>
```

### Incident Metrics Dashboard

```
┌──────────────────────────────────────────────────────────────────────┐
│              INCIDENT MANAGEMENT METRICS (Monthly)                   │
├─────────────────────┬─────────────┬──────────────────────────────────┤
│ Metric              │ This Month  │ Trend                            │
├─────────────────────┼─────────────┼──────────────────────────────────┤
│ Total incidents     │ 12          │ ████████░░ (↓ from 18 last mo)   │
│  - SEV1             │ 1           │ ██░░░░░░░░                       │
│  - SEV2             │ 3           │ ████░░░░░░                       │
│  - SEV3             │ 8           │ ████████░░                       │
├─────────────────────┼─────────────┼──────────────────────────────────┤
│ MTTD (median)       │ 4 min       │ ████░░░░░░ (target: < 5 min)     │
│ MTTE (median)       │ 3 min       │ ███░░░░░░░ (target: < 5 min)     │
│ MTTM (median)       │ 22 min      │ ████████░░ (target: < 30 min)    │
│ MTTR (median)       │ 4.2 hrs     │ ████████░░ (target: < 8 hrs)     │
├─────────────────────┼─────────────┼──────────────────────────────────┤
│ Pages per on-call   │ 1.8/week    │ ██░░░░░░░░ (target: < 2)         │
│ After-hours pages   │ 3 total     │ ███░░░░░░░ (target: < 4)         │
│ False positive rate │ 8%          │ █░░░░░░░░░ (target: < 10%)       │
├─────────────────────┼─────────────┼──────────────────────────────────┤
│ Postmortems written │ 4/4 (100%)  │ ██████████ (target: 100%)        │
│ Action items done   │ 11/15 (73%) │ ███████░░░ (target: > 80%)       │
│ Repeat incidents    │ 1 (8%)      │ █░░░░░░░░░ (target: 0%)          │
├─────────────────────┼─────────────┼──────────────────────────────────┤
│ On-call satisfaction│ 3.8/5       │ ████████░░ (target: > 3.5)       │ 
│ Game days completed │ 1 (quarterly│ ██████████                       │
│                     │  on track)  │                                  │
└─────────────────────┴─────────────┴──────────────────────────────────┘
```

---

## Quick Reference Card

```
SEVERITY CLASSIFICATION
───────────────────────
SEV1: Total outage, data loss, security breach → PAGE → IC → 15min comms → Mandatory postmortem 48h
SEV2: Major degradation, significant users affected → PAGE → 30min comms → Mandatory postmortem 1wk
SEV3: Minor degradation, subset of users → TICKET → Business hours → Optional postmortem
SEV4: Cosmetic, workaround exists → BACKLOG → Sprint planning

WHEN IN DOUBT → ESCALATE UP. Downgrading is free. Late escalation is expensive.

INCIDENT ROLES
──────────────
IC: Owns process. Assigns roles. Makes decisions. NOT debugging.
Tech Lead: Owns investigation. Proposes hypotheses. Directs SMEs.
Comms: Status page, Slack, executives. Every 15min (SEV1) / 30min (SEV2).
Ops: Executes commands. Announces before acting. Documents timestamps.

INVESTIGATION FRAMEWORK
───────────────────────
1. Scope impact (2 min): What services? What %? Getting worse?
2. What changed? (3 min): Deploys, configs, infra, traffic, deps
3. Follow data path (5-10 min): Edge → services → databases. Metrics → traces → logs.
4. MITIGATE (ASAP): Collect evidence THEN rollback/restart/failover/scale
5. Verify (15 min): Error rate dropping? SLI recovering? Stable?

⚠️ MITIGATE FIRST. Root cause LATER. $50K/minute.

ON-CALL HEALTH
──────────────
< 2 pages/week | < 1 after-hours/week | < 10% false positive
< 5 min ACK | < 30 min MTTM (SEV1) | > 3.5/5 satisfaction

POSTMORTEM MUST-HAVES
─────────────────────
Timeline | Root cause (system, not human) | Impact | Detection analysis
What went well | What went wrong | WHERE WE GOT LUCKY
Action items (owner + due date + Jira ticket)
NO BLAME. Fix the system, not the person.

CHAOS ENGINEERING
─────────────────
Steady state → Hypothesize → Inject failure → Observe → Learn
Start small (1 pod) → Graduate (AZ failure) → Production (quarterly game day)
Tools: Litmus Chaos (NovaMart), Chaos Mesh, Gremlin, AWS FIS
```
---

## Retention Questions — Phase 5 Lesson 5

### Q1: SEV1 Incident Response 🔥

**Scenario:** 2:47 AM. PagerDuty wakes you up:

```
Alert: PaymentSvcHighBurnRate_Page
Severity: critical
Description: "payment-svc /charge burning error budget at 22x rate"
Burn rate 1h: 22x
Error budget remaining: 71%
```

You acknowledge the alert. You're the primary on-call. The secondary on-call is in a different timezone (it's their evening).

**Questions:**
1. Walk through your **first 10 minutes** step by step. What do you do, in what order, with what exact commands/queries? Include Slack messages you'd post and role assignments.
2. At minute 8, you discover: error rate is 40% on `/charge`, ALL errors are `502 Bad Gateway`, and payment-svc pods are all `Running` with no restarts. What does this pattern tell you? List your top 3 hypotheses in order of likelihood.
3. At minute 12, you confirm the root cause: ArgoCD auto-synced a new Istio VirtualService that routes `/charge` traffic to a canary deployment `payment-svc-canary` — but the canary has 0 replicas (it was a skeleton deploy for next week's test). How do you mitigate in under 2 minutes? Give the exact commands.
4. Write the **"Where We Got Lucky"** section for this incident's postmortem.

---

### Q2: On-Call Process Design 🔥

**Scenario:** You've been at NovaMart for 3 months. The current on-call situation is a mess:
- 15-25 pages per week (most are noise)
- No runbooks for 80% of alerts
- Two engineers refuse to participate in on-call ("I'm a developer, not ops")
- The same Redis connection timeout alert fires 5x/day and everyone ignores it
- Last month's SEV1 took 2 hours to mitigate because the on-call couldn't find the right dashboard

**Questions:**
1. Design a 90-day plan to fix the on-call process. Be specific about what happens in weeks 1-4, 5-8, and 9-12. Include metrics you'll track to prove improvement.
2. How do you handle the two engineers who refuse on-call? Give the technical argument AND the organizational/cultural argument.
3. The Redis alert that fires 5x/day — what's your approach? You can't just delete it. Walk through your analysis and resolution.
4. Propose a specific alert quality scoring system that NovaMart could use to systematically improve alert signal-to-noise ratio.

---

### Q3: Postmortem Under Pressure 🔥

**Scenario:** NovaMart had a SEV1 last week. During the postmortem meeting:
- The VP of Engineering says: "We need to name who was responsible. The board is asking."
- The developer who deployed the bad change is visibly anxious and hasn't spoken.
- The payments tech lead says: "If we had just reviewed the PR properly, none of this would have happened."
- Two action items from the LAST postmortem (2 months ago) are still not done.

**Questions:**
1. You're facilitating this postmortem. How do you handle the VP's demand for individual accountability? Give the **exact words** you would say.
2. How do you redirect the tech lead's comment from blame to system improvement? Give the **exact words** you would say.
3. The developer who deployed the change — how do you create safety for them to contribute? What specific thing would you say or do?
4. The undone action items from 2 months ago — how do you address this pattern? What systemic change prevents it from recurring?

---

### Q4: Chaos Engineering Game Day Design 🔥

**Scenario:** You're planning NovaMart's first-ever game day. The VP is nervous: "I don't want you breaking production on purpose." The payments team is resistant: "Our service is fine, we don't need chaos testing." You have one day to convince them and one week to prepare.

**Questions:**
1. Write the **executive summary** (5 sentences max) you'd send to the VP to get approval. Focus on business value, not technical excitement.
2. Design 4 experiments for the game day, escalating in severity. For each: state the hypothesis, the injection method, the success criteria (in SLI terms), and the abort condition.
3. The payments team says "our service is fine." What data would you pull from existing monitoring to prove they have untested assumptions? Give specific queries.
4. During the game day, Experiment 3 (AZ failure simulation) causes an unexpected cascade: order-svc loses connection to RDS because the RDS primary was in the failed AZ, and RDS failover takes 45 seconds during which ALL orders fail. This wasn't predicted. What do you do in the moment AND what action items come out of this?

# Phase 5, Lesson 5 — SRE Practices Answers

---

## Q1: SEV1 Incident Response

### 1. First 10 Minutes — Step by Step

**Minute 0-1: Acknowledge and Assess**

```
# Acknowledge the PagerDuty alert (phone or app)
# This stops escalation to secondary and starts the incident clock
```

**First action — open laptop, verify the alert is real (not a false positive):**

```promql
# From phone (Grafana mobile) or laptop — verify burn rate
sli:payment_charge_availability:burnrate1h
# Expected: ~22 (confirms alert)

# Quick error rate check
sum(rate(http_requests_total{service="payment-svc", endpoint="/charge", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="payment-svc", endpoint="/charge"}[5m]))
# If this returns > 0.01 (1%), this is real. At 22x burn rate, expect ~1.1% error rate minimum
```

**Minute 1-2: Declare the incident and open communications**

Post to `#incidents` Slack channel:

```
🔴 SEV1 DECLARED — Payment /charge errors

Alert: PaymentSvcHighBurnRate_Page — 22x burn rate
Impact: payment-svc /charge returning errors — customers cannot complete purchases
Error budget: 71% remaining, burning fast
Incident Commander: [your name]
War room: #inc-20240116-payment-charge
Status page: Investigating

@oncall-secondary I need you online — joining war room
@payments-team-lead FYI — payment-svc incident, may need your team's context
```

**Create dedicated incident channel:**

```
/create-incident Payment /charge errors — 22x burn rate
# Or manually:
Slack: Create channel #inc-20240116-payment-charge
```

**Minute 2-3: Update status page (before deep investigation)**

```bash
# If using Statuspage.io/Atlassian Statuspage:
# Set payment processing to "Degraded Performance"
# This is critical — customers and support need to know IMMEDIATELY

# If using API:
curl -X POST "https://api.statuspage.io/v1/pages/${PAGE_ID}/incidents" \
  -H "Authorization: OAuth ${STATUSPAGE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "incident": {
      "name": "Payment processing errors",
      "status": "investigating",
      "impact_override": "major",
      "body": "We are investigating elevated error rates on payment processing. Some customers may experience failed checkout attempts. We are actively working on a fix.",
      "component_ids": ["'${PAYMENT_COMPONENT_ID}'"],
      "components": {"'${PAYMENT_COMPONENT_ID}'": "degraded_performance"}
    }
  }'
```

**Minute 3-5: Systematic diagnosis — the "what, where, when" triage**

```promql
# WHAT: What errors exactly?
sum by (status) (
  rate(http_requests_total{service="payment-svc", endpoint="/charge", status=~"5.."}[5m])
)
# Tells us: are these 500, 502, 503, 504?

# WHERE: All pods or specific pods?
sum by (pod) (
  rate(http_requests_total{service="payment-svc", endpoint="/charge", status=~"5.."}[5m])
)
# Tells us: localized to one pod or cluster-wide?

# WHEN: When did it start? (look at the step change)
# Set Grafana to 2-hour view:
sum(rate(http_requests_total{service="payment-svc", endpoint="/charge", status=~"5.."}[1m]))
# Look for the inflection point — note exact timestamp
```

```promql
# Are pods healthy?
kube_pod_status_phase{namespace="payments", pod=~"payment-svc.*"}
kube_pod_container_status_restarts_total{namespace="payments", container="payment-svc"}
# Check: are pods Running? Any recent restarts?

# Is it a downstream dependency?
# Check exemplars or trace for a failing request
# Click an exemplar on the error spike → Tempo trace → see which span fails
```

```bash
# Check recent deployments/changes (most incidents are caused by changes)
kubectl -n payments get events --sort-by='.lastTimestamp' | tail -20

# Check ArgoCD for recent syncs
argocd app list --output json | jq '.[] | select(.metadata.name | contains("payment")) | {name: .metadata.name, syncStatus: .status.sync.status, health: .status.health.status, lastSync: .status.operationState.finishedAt}'

# Check Istio config changes
kubectl -n payments get virtualservice,destinationrule --sort-by='.metadata.creationTimestamp' | tail -10
```

**Minute 5-7: Post initial findings to war room**

```
📊 STATUS UPDATE — Minute 5

Errors: [X]% of /charge requests failing
Error type: [status code]
Scope: [all pods / specific pods / all regions / specific region]
Start time: ~[HH:MM] UTC (approximately [X] minutes ago)
Pod health: [Running/CrashLooping/OOM]
Recent changes: [list any deploys, config changes, ArgoCD syncs]

Current hypothesis: [based on what we see so far]
Next action: [what I'm checking now]

Budget impact: at 22x burn rate, we consume ~0.73% of budget per minute
               71% remaining → hits 0% in approximately 97 minutes if unmitigated
```

**The budget math:**

```
22x burn rate means consuming budget 22x faster than sustainable
Sustainable rate exhausts 100% in 30 days
22x → exhausts 100% in 30/22 = 1.36 days = 32.7 hours
From 71% remaining: 0.71 × 32.7 hours = 23.2 hours until budget exhaustion

BUT more importantly: at 22x burn rate with 40% error rate,
NovaMart is losing 40% of payment transactions right now.
At $50K/min revenue, that's $20K/min in lost transactions.
Every minute of diagnosis costs $20K.
```

**Minute 7-10: Deep dive based on findings**

This branches based on what minute 3-5 revealed. Proceeding to the scenario where we discover 502s...

### 2. Minute 8 Diagnosis: 40% Error Rate, All 502s, Pods Running

**Pattern analysis:**

```
502 Bad Gateway = the request reached the proxy/mesh layer but the
                  upstream service returned an invalid response or
                  was unreachable

Pods Running + No restarts = the payment-svc containers are alive
                             but something between the client and
                             the pods is broken

40% (not 100%) = some requests succeed, some fail
                 → partial routing issue, not total outage
```

**This pattern is CLASSIC for a routing/mesh misconfiguration.** The pods are healthy (they can serve traffic), but some percentage of traffic is being routed to a destination that can't handle it.

**Top 3 hypotheses in order of likelihood:**

**Hypothesis 1 (Most Likely): Traffic routing misconfiguration — traffic split to a bad destination**

```
Evidence: 
  - 502 = upstream unreachable (not 500 = app crash)
  - Pods healthy = the real pods work fine
  - 40% not 100% = traffic is SPLIT between working and broken destinations
  - This ratio often maps to a weighted route or canary percentage

Mechanism:
  Istio VirtualService or Ingress rule changed → some % of /charge traffic
  routed to a destination with 0 replicas, wrong port, or non-existent service
  → envoy proxy gets connection refused → returns 502

Check:
```

```bash
kubectl -n payments get virtualservice -o yaml | grep -A 20 "/charge"
kubectl -n payments get destinationrule -o yaml
kubectl -n payments get deploy -l app=payment-svc
# Look for: canary deployments, weight-based routing, new destinations
```

**Hypothesis 2: Downstream dependency failure (Stripe API, database)**

```
Evidence supporting:
  - 502 can also mean payment-svc made a call to a downstream that
    failed, and the error propagated back as 502 through the mesh

Evidence against:
  - If Stripe was down, we'd expect 504 (timeout) or 503, not 502
  - 40% failure is unusual for Stripe (usually 100% or 0%)
  - Pods not restarting means the app isn't crashing from the errors

Check:
```

```promql
# Check Stripe call success rate
sum by (status) (
  rate(http_client_request_duration_seconds_count{
    service="payment-svc",
    remote=~".*stripe.*"
  }[5m])
)
```

**Hypothesis 3: Resource exhaustion — envoy sidecar or connection pool**

```
Evidence supporting:
  - 502 from envoy when sidecar can't proxy to the local container
  - Partial failure (40%) could indicate thread/connection pool saturation

Evidence against:
  - Usually causes gradual degradation, not step-function failure
  - Would expect to see increasing latency before errors

Check:
```

```promql
# Envoy sidecar metrics
envoy_cluster_upstream_cx_active{cluster_name="inbound|8080||"}
envoy_cluster_upstream_cx_overflow{cluster_name="inbound|8080||"}
```

### 3. Minute 12 Mitigation: VirtualService Routing to 0-Replica Canary

**Root cause confirmed:** ArgoCD auto-synced a VirtualService that routes some percentage of `/charge` traffic to `payment-svc-canary` which has 0 replicas. Envoy tries to route to it, gets no healthy endpoints, returns 502.

**Fix in under 2 minutes — TWO options, do the FASTEST one first:**

**Option A (fastest — ~30 seconds): Delete or patch the VirtualService to remove the canary route:**

```bash
# FIRST: Check what the VirtualService looks like
kubectl -n payments get virtualservice payment-svc-vs -o yaml

# You'll see something like:
# http:
#   - match:
#       - uri:
#           prefix: /charge
#     route:
#       - destination:
#           host: payment-svc
#           port:
#             number: 8080
#         weight: 60
#       - destination:
#           host: payment-svc-canary    ← THE PROBLEM
#           port:
#             number: 8080
#         weight: 40                    ← 40% of traffic going here = 40% errors

# FIX: Patch to send 100% to the working service
kubectl -n payments patch virtualservice payment-svc-vs --type='json' \
  -p='[
    {"op":"replace","path":"/spec/http/0/route","value":[
      {"destination":{"host":"payment-svc","port":{"number":8080}},"weight":100}
    ]}
  ]'

# Verify immediately
kubectl -n payments get virtualservice payment-svc-vs -o jsonpath='{.spec.http[0].route[*].destination.host}'
# Expected: payment-svc (only one destination, no canary)
```

**Option B (if ArgoCD will immediately re-sync and revert your fix):**

```bash
# Disable ArgoCD auto-sync on this app FIRST
argocd app set payment-svc --sync-policy none

# Verify auto-sync is disabled
argocd app get payment-svc | grep "Sync Policy"
# Expected: Sync Policy: <none>

# THEN apply the VirtualService fix
kubectl -n payments patch virtualservice payment-svc-vs --type='json' \
  -p='[
    {"op":"replace","path":"/spec/http/0/route","value":[
      {"destination":{"host":"payment-svc","port":{"number":8080}},"weight":100}
    ]}
  ]'
```

**⚠️ CRITICAL: If you don't disable ArgoCD auto-sync first, ArgoCD will detect the drift within 3 minutes (default sync interval) and re-apply the broken VirtualService. You'll fix it, it'll break again, you'll fix it, it'll break again — while $20K/min is burning.**

```bash
# Verify the fix is working (within 30 seconds of applying)
# Error rate should drop to near-zero almost immediately
# because Istio applies VirtualService changes within seconds

# Quick check from command line:
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    https://api.novamart.com/charge \
    -H "Content-Type: application/json" \
    -d '{"test": true}'
  sleep 1
done
# Should see 200s, not 502s

# Verify in Prometheus (may take 30-60s for rate() to reflect)
# In Grafana or curl:
curl -s "http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total{service=\"payment-svc\",endpoint=\"/charge\",status=~\"5..\"}[1m]))" | jq '.data.result[0].value[1]'
# Should be dropping toward 0
```

**Post-mitigation Slack update:**

```
✅ MITIGATED — Minute 14

Root cause: ArgoCD auto-synced a VirtualService that routed 40% of 
/charge traffic to payment-svc-canary (0 replicas). Envoy returned 
502 for all requests to the canary destination.

Fix applied: Patched VirtualService to route 100% to payment-svc.
ArgoCD auto-sync disabled on payment-svc app to prevent revert.

Error rate: dropping to baseline (confirm in 2 minutes)
Duration: ~[X] minutes
Budget impact: approximately [X]% consumed during incident

Status page: updating to "Monitoring"
Next: confirm recovery, then postmortem scheduling

⚠️ DO NOT re-enable ArgoCD auto-sync until the broken VirtualService 
is fixed in Git.
```

```bash
# Update status page to monitoring
curl -X PATCH "https://api.statuspage.io/v1/pages/${PAGE_ID}/incidents/${INCIDENT_ID}" \
  -H "Authorization: OAuth ${STATUSPAGE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "incident": {
      "status": "monitoring",
      "body": "We have identified and resolved the issue causing payment processing errors. We are monitoring the fix. Some customers may have experienced failed checkout attempts between approximately 02:47 and 03:01 UTC."
    }
  }'
```

### 4. "Where We Got Lucky" — Postmortem Section

```markdown
## Where We Got Lucky

1. **The canary had 0 replicas, not 1 replica with a bug.**
   If the canary had been running with a partially-functional version 
   (e.g., accepting requests but charging wrong amounts), we would 
   have had 40% of payments processed incorrectly instead of 40% 
   failing. Silent data corruption is infinitely worse than loud 
   failures. The 502s were immediately visible in metrics and 
   triggered the burn rate alert within minutes. A canary silently 
   charging wrong amounts could have run for hours before anyone 
   noticed — via customer complaints, not monitoring.

2. **The error budget was at 71%, not 15%.**
   At 22x burn rate, we consumed approximately 0.73% of budget per 
   minute. The incident lasted ~14 minutes = ~10.2% budget consumed. 
   If we had been at 15% remaining (which we WERE two months ago 
   after the last SEV1), this incident would have breached the SLO 
   and triggered a deployment freeze — during a week with three 
   planned feature launches. The current budget headroom was 
   coincidental, not by design.

3. **The on-call engineer knew to check ArgoCD sync history.**
   Our runbook for 502 errors focuses on pod health and downstream 
   dependencies. It does NOT include "check for recent Istio config 
   changes via GitOps." The on-call happened to check ArgoCD based 
   on personal experience, not because the runbook directed them 
   there. A less experienced on-call would have spent 20+ additional 
   minutes investigating pods and Stripe before thinking to check 
   service mesh routing — consuming another ~15% of error budget.

4. **This happened at 2:47 AM, not 2:47 PM.**
   NovaMart's peak traffic is 11 AM - 3 PM. At peak, /charge handles 
   approximately 500 req/s. At 2:47 AM, traffic is approximately 
   50 req/s. The 40% error rate at 50 req/s = ~20 failed payments/s 
   = ~$600/min in lost transactions. At peak, the same error rate = 
   ~200 failed payments/s = ~$6,000/min. We lost approximately 
   $8,400 in failed transactions. At peak, the same duration would 
   have cost ~$84,000.

5. **ArgoCD didn't re-sync during mitigation.**
   ArgoCD's default sync interval is 3 minutes. We patched the 
   VirtualService AND disabled auto-sync within ~2 minutes. If 
   ArgoCD had re-synced before we disabled auto-sync, the broken 
   config would have been re-applied and the incident would have 
   extended by another investigation cycle. The timing was fortunate.
```

```markdown
## Action Items Derived from "Where We Got Lucky"

| # | Action | Owner | Priority | Addresses |
|---|--------|-------|----------|-----------|
| 1 | Add "Check recent ArgoCD syncs and Istio config changes" to 502 runbook | SRE | P1 | Lucky #3 |
| 2 | ArgoCD sync policy for payment-svc: require manual sync for VirtualService/DestinationRule changes | Platform | P1 | Lucky #5 |
| 3 | Add canary validation webhook: block VirtualService routes pointing to 0-replica deployments | Platform | P1 | Lucky #1 |
| 4 | SLI for payment correctness (charge amount validation), not just availability | Payments | P2 | Lucky #1 |
| 5 | Alert on ArgoCD sync events for production namespace with Slack notification | Platform | P2 | Lucky #3, #5 |
```

---

## Q2: On-Call Process Design

### 1. 90-Day On-Call Improvement Plan

**Weeks 1-4: STOP THE BLEEDING (Reduce noise, build foundation)**

```
┌─────────────────────────────────────────────────────────────────┐
│ WEEK 1: Audit and Classify Every Alert                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Action 1: Export the last 30 days of PagerDuty alerts           │
│                                                                 │
│   curl -s "https://api.pagerduty.com/incidents?since=30d_ago    │
│     &until=now&statuses[]=resolved&statuses[]=acknowledged"     │
│     -H "Authorization: Token token=${PD_TOKEN}" | jq . > alerts.json │
│                                                                 │
│ Action 2: Categorize every alert into one of four buckets:      │
│                                                                 │
│   A. ACTIONABLE — Required human intervention, correct to page  │
│   B. NOISY — Fired but required no action (auto-resolved,       │
│      false positive, or below customer impact threshold)        │
│   C. MISSING RUNBOOK — Was actionable but responder didn't      │
│      know what to do                                            │
│   D. DUPLICATE — Same root cause triggered multiple alerts      │
│                                                                 │
│ Metric: Calculate current Action Rate                           │
│   Action Rate = (Bucket A) / (Total Alerts)                     │
│   Target: Currently likely ~20%, goal is >80%                   │
│                                                                 │
│ Action 3: Identify the top 5 noisiest alerts by frequency       │
│   jq '[.incidents[] | .service.summary] | group_by(.) |         │
│     map({alert: .[0], count: length}) |                         │
│     sort_by(-.count) | .[0:5]' alerts.json                      │
│                                                                 │
│   The Redis alert will be #1 or #2. Address it (see Q2.3)       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│ WEEK 2: Kill the Top 5 Noisy Alerts                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ For each of the top 5:                                          │
│   - If never actionable → DELETE or downgrade to warning/ticket │
│   - If sometimes actionable → tighten thresholds, add for:      │
│     duration, or add inhibition rules                           │
│   - If always actionable but missing runbook → write runbook    │
│                                                                 │
│ Specifically for the Redis alert (see Q2.3 for full analysis):  │
│   - Fix the underlying issue OR adjust the threshold            │
│   - This single fix eliminates ~5 pages/day = 35/week           │
│   - That alone drops weekly pages from 15-25 to 10-20           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│ WEEK 3: Runbook Sprint                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Remaining actionable alerts need runbooks. For each alert:      │
│                                                                 │
│ Template:                                                       │
│   1. What does this alert mean? (plain English)                 │
│   2. What is the customer impact?                               │
│   3. What dashboard do I open FIRST?                            │
│   4. Decision tree: if X → do Y, if Z → do W                    │
│   5. Escalation: when do I page the secondary/team lead?        │
│   6. Mitigation commands (copy-pasteable)                       │
│   7. Rollback procedure                                         │
│                                                                 │
│ Assign: Each service team writes runbooks for their alerts      │
│ Deadline: End of week 3                                         │
│ Enforce: Alert without runbook link = alert gets downgraded     │
│          to ticket until runbook exists                         │
│                                                                 │
│ Add runbook_url annotation to every alert:                      │
│                                                                 │
│   annotations:                                                  │
│     runbook: "https://wiki.novamart.internal/runbooks/{{ $labels.alertname }}" │
│                                                                 │
│ PagerDuty custom action: "Open Runbook" button in every page    │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│ WEEK 4: On-Call Onboarding Kit + Dashboard                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Build the "On-Call Home Dashboard" in Grafana:                  │
│   Row 1: SLO status for all services (error budget remaining)   │
│   Row 2: Active alerts (current firing alerts)                  │
│   Row 3: Recent deployments (last 24h, all services)            │
│   Row 4: Key business metrics (orders/min, payments/min)        │
│                                                                 │
│ Create on-call onboarding document:                             │
│   - How to access PagerDuty, Grafana, kubectl, ArgoCD           │
│   - First 5 minutes of any incident (checklist)                 │
│   - Escalation matrix (who to call for what)                    │
│   - VPN/access/credential setup                                 │
│   - Link to the Home Dashboard                                  │
│                                                                 │
│ Shadow rotation: New on-call engineers shadow for 1 full        │
│ rotation before going primary                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Weeks 5-8: BUILD MUSCLE (Process, training, accountability)**

```
┌─────────────────────────────────────────────────────────────────┐
│ WEEK 5-6: On-Call Training Program                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Wheel of Misfortune sessions (simulated incidents):             │
│   - Weekly 1-hour session                                       │
│   - Facilitator injects a scenario, on-call engineer responds   │
│   - Practice using runbooks, dashboards, escalation             │
│   - Record common mistakes → improve runbooks                   │
│                                                                 │
│ Topics:                                                         │
│   Week 5: "Payment-svc 502 errors" (use real Q1 scenario)       │
│   Week 6: "Database failover during peak traffic"               │
│   Week 7: "Cardinality bomb killing Prometheus"                 │
│   Week 8: "Certificate expiry causing cascading TLS failures"   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│ WEEK 7-8: Alert Tuning Sprint + On-Call Feedback Loop           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ On-call handoff meeting (30 min, every rotation change):        │
│   1. Outgoing on-call: "Here's what fired, here's what I did"   │
│   2. Flag any alerts that were noisy or missing runbooks         │
│   3. Incoming on-call: "Any ongoing issues I should know about?" │
│   4. Action items from the rotation get filed as tickets         │
│                                                                 │
│ Alert review meeting (monthly, 1 hour):                         │
│   - Review all alerts from the past month                       │
│   - Score each alert (see Q2.4 scoring system)                  │
│   - Delete/tune/improve based on scores                         │
│   - Track Action Rate trend month-over-month                    │
│                                                                 │
│ Implement alert-level feedback in PagerDuty:                    │
│   - After resolving, engineer rates: "Actionable? Y/N"          │
│   - "Runbook helpful? Y/N"                                      │
│   - Free-text: "What would have helped?"                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Weeks 9-12: MATURE (Measure, automate, sustain)**

```
┌─────────────────────────────────────────────────────────────────┐
│ WEEK 9-10: Automation of Common Remediations                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Identify the top 3 most frequent ACTIONABLE alerts              │
│ For each, evaluate: can the mitigation be automated?            │
│                                                                 │
│ Example: "Pod OOM-killed" alert                                 │
│   Manual response: kubectl rollout restart                      │
│   Automated: Kubernetes handles this natively with restartPolicy│
│   Fix: Ensure resource limits are set correctly, add VPA        │
│                                                                 │
│ Example: "Disk >90%" alert                                      │
│   Manual response: SSH in, clean up logs/tmp                    │
│   Automated: Log rotation policy + PVC auto-expansion           │
│                                                                 │
│ Goal: Reduce actionable alerts that require HUMAN action        │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│ WEEK 11-12: Metrics Review + Process Certification              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Present to engineering leadership:                              │
│                                                                 │
│ Metrics to track and present:                                   │
│                                                                 │
│ 1. Pages per week (target: <5 actionable pages)                 │
│    Week 1 baseline: 15-25                                       │
│    Week 12 target: 3-7                                          │
│                                                                 │
│ 2. Action Rate (target: >80%)                                   │
│    Week 1 baseline: ~20%                                        │
│    Week 12 target: >80%                                         │
│                                                                 │
│ 3. MTTA — Mean Time to Acknowledge (target: <5 min)             │
│    Measured from PagerDuty                                      │
│                                                                 │
│ 4. MTTR — Mean Time to Resolve (target: <30 min for SEV2)       │
│    Measured from incident creation to resolution                │
│                                                                 │
│ 5. Runbook coverage (target: 100% of paging alerts)             │
│    Week 1 baseline: ~20%                                        │
│    Week 12 target: 100%                                         │
│                                                                 │
│ 6. On-call satisfaction (quarterly survey, 1-5 scale)           │
│    "Was on-call manageable this rotation?"                      │
│    "Did you have the tools/runbooks you needed?"                │
│    "Did you lose significant sleep?"                            │
│                                                                 │
│ 7. Alert quality score (see Q2.4)                               │
│    Average across all alerts, trending up                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Handling Engineers Who Refuse On-Call

**The technical argument:**

"You write the code, you understand it best. When `payment-svc` pages at 2 AM, the person who wrote the retry logic and knows the Stripe integration intimately will resolve the incident in 5 minutes. An SRE who's never seen the code will take 30 minutes — that's 25 minutes of additional customer impact at $50K/min.

On-call isn't 'ops work.' It's the **feedback loop** that makes you a better engineer. Every on-call rotation teaches you what breaks in production, which directly improves the code you write. Engineers who never do on-call write code that's harder to operate, debug, and monitor — because they've never experienced the consequences of their design decisions at 2 AM.

The alternative — a separate ops team that handles all production issues — is the pre-DevOps model. It creates a wall between 'people who write code' and 'people who deal with the consequences.' That wall produces worse software, slower incident response, and higher outage costs."

**The organizational/cultural argument:**

"On-call is shared responsibility across the engineering organization. This is a NovaMart engineering policy, not a request.

That said — I hear you. If on-call feels unreasonable, let's look at why:

1. **If the concern is noise:** We're actively fixing alert quality. By week 4, noisy pages drop by 60%+. On-call should mean 0-2 real pages per week, not 15-25.

2. **If the concern is knowledge:** We have runbooks, shadowing rotations, and training. Nobody goes on primary on-call without a shadow rotation first and passing a Wheel of Misfortune exercise.

3. **If the concern is compensation:** On-call should come with compensation — stipend per rotation, time-off-in-lieu for overnight pages, or equivalent. I'll advocate for this with leadership.

4. **If the concern is fairness:** On-call rotation is shared equally across the team. No one person carries more than others. The schedule is published 6 weeks in advance. Swaps are allowed and encouraged.

What I can't do is exempt individuals. If two engineers don't participate, the remaining team members carry a disproportionate burden — that's unfair AND it burns out the people who ARE participating, which means we lose them.

Let's address your specific concerns. What would make on-call workable for you?"

**If they still refuse after this conversation:**

This becomes a management issue. Escalate to their engineering manager with:

```
"[Names] have declined to participate in the on-call rotation. 
I've addressed their concerns about alert noise (we're reducing it), 
training (we have a shadow program), and compensation (I've proposed 
a stipend). The remaining concern is that they view production 
operations as outside their role. 

This is a team policy decision for you to enforce. The impact of 
their non-participation is that [N] other engineers on the team 
carry [X]% more on-call burden, which creates burnout and retention 
risk for the engineers who ARE participating."
```

### 3. The Redis Alert That Fires 5x/Day

**Step 1: Understand the alert**

```bash
# Pull the alert definition
curl -s "http://prometheus:9090/api/v1/rules" | jq '.data.groups[].rules[] | select(.name == "RedisConnectionTimeout") '

# Likely something like:
# alert: RedisConnectionTimeout
# expr: redis_connection_errors_total > 0
# for: 1m
# labels:
#   severity: critical
```

**Step 2: Analyze the pattern**

```promql
# When does it fire? Look at the last 7 days
redis_connection_errors_total
# Or if it's a counter:
rate(redis_connection_errors_total[5m])

# Correlate with time of day
# In Grafana: 7-day view, look for pattern
```

```promql
# Is it always the same Redis instance?
sum by (instance) (rate(redis_connection_errors_total[5m]))

# Is it always the same client service?
sum by (service) (rate(redis_connection_errors_total[5m]))

# What's the actual impact?
# Connection errors vs total connections:
rate(redis_connection_errors_total[5m])
/
rate(redis_connections_total[5m])
# If this is 0.001 (0.1%) → the alert threshold is too sensitive
```

**Step 3: Determine the root cause of the connection timeouts**

```bash
# Check Redis itself
kubectl -n data exec -it redis-0 -- redis-cli info clients
# Look at: connected_clients, blocked_clients, rejected_connections

kubectl -n data exec -it redis-0 -- redis-cli info stats
# Look at: total_connections_received, rejected_connections

kubectl -n data exec -it redis-0 -- redis-cli info memory
# Look at: used_memory vs maxmemory — is Redis under memory pressure?

# Check client-side connection pool config
kubectl -n payments get configmap payment-svc-config -o yaml | grep -i redis
# Look at: pool size, timeout settings, retry config
```

**Most likely scenario:** Redis is healthy overall, but brief connection blips occur during:
- Redis Sentinel failover tests
- Kubernetes pod scheduling on the Redis node
- Connection pool recycling
- Network micro-partitions

The errors are **transient**, the application **retries and succeeds**, and there's **zero customer impact** — but the alert fires on ANY error, so 5 timeouts out of 500,000 connections triggers 5 pages.

**Step 4: Fix — restructure the alert, don't delete it**

```yaml
# BEFORE (broken): alerts on any error, no rate context, no impact assessment
- alert: RedisConnectionTimeout
  expr: redis_connection_errors_total > 0
  for: 1m
  labels:
    severity: critical

# AFTER (correct): alerts on error RATE that indicates real impact
groups:
  - name: redis-alerts
    rules:
      # Alert only when error rate is meaningful relative to total traffic
      - alert: RedisConnectionErrorRateHigh
        expr: |
          (
            sum(rate(redis_connection_errors_total[5m]))
            /
            sum(rate(redis_connections_total[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Redis connection error rate {{ $value | humanizePercentage }}"
          runbook: "https://wiki.novamart.internal/runbooks/redis-connection-errors"

      # Critical only if sustained high error rate (actual Redis outage)
      - alert: RedisConnectionErrorRateCritical
        expr: |
          (
            sum(rate(redis_connection_errors_total[5m]))
            /
            sum(rate(redis_connections_total[5m]))
          ) > 0.25
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Redis connection error rate {{ $value | humanizePercentage }} — likely Redis outage"
          runbook: "https://wiki.novamart.internal/runbooks/redis-outage"

      # Also alert if Redis is completely unreachable (up metric)
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Redis instance {{ $labels.instance }} is DOWN"
```

**This preserves the safety net** (we still alert if Redis has a real problem) while eliminating the noise (5 transient timeouts/day no longer page anyone).

**Step 5: Verify the fix**

```promql
# After deploying, confirm the error rate stays below 5%
sum(rate(redis_connection_errors_total[5m]))
/
sum(rate(redis_connections_total[5m]))
# Should be << 0.05, meaning the warning won't fire

# Set up a test: manually trigger a Redis connection error
# and verify it only fires if the RATE threshold is breached
```

### 4. Alert Quality Scoring System

```
┌─────────────────────────────────────────────────────────────────┐
│ NOVAMART ALERT QUALITY SCORECARD                                │
│                                                                 │
│ Every alert is scored monthly on 5 dimensions (1-5 each)        │
│ Maximum score: 25/25                                            │
│ Minimum to remain as a paging alert: 15/25                      │
│ Below 10/25: Delete or fundamentally redesign                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ DIMENSION 1: ACTIONABILITY (Does it require human action?)      │
│                                                                 │
│   5: Every firing requires specific human intervention          │
│   4: >80% of firings require action                             │
│   3: 50-80% require action                                      │
│   2: 20-50% require action (noisy)                              │
│   1: <20% require action (mostly noise)                         │
│                                                                 │
│   Measured by: On-call feedback ("Was this actionable? Y/N")    │
│                                                                 │
│ DIMENSION 2: URGENCY (Does it need immediate response?)         │
│                                                                 │
│   5: Customer-facing impact if not addressed in <15 minutes     │
│   4: Customer impact within 1 hour                              │
│   3: Internal impact within 4 hours                             │
│   2: Can wait until next business day                           │
│   1: Can wait until next sprint                                 │
│                                                                 │
│   If score ≤ 2: Should be a ticket, not a page                  │
│                                                                 │
│ DIMENSION 3: COMPLETENESS (Does responder know what to do?)     │
│                                                                 │
│   5: Runbook exists, tested, includes exact commands,           │
│      escalation path, and rollback procedure                    │
│   4: Runbook exists but missing some details                    │
│   3: Runbook exists but is outdated or vague                    │
│   2: No runbook, but alert description gives some guidance      │
│   1: No runbook, no description, responder has to guess         │
│                                                                 │
│ DIMENSION 4: ACCURACY (Signal-to-noise ratio)                   │
│                                                                 │
│   5: Zero false positives in the past month                     │
│   4: 1-2 false positives per month                              │
│   3: 1 false positive per week                                  │
│   2: Multiple false positives per week                          │
│   1: Fires daily with false positives                           │
│                                                                 │
│   Measured by: (True positives) / (Total firings)               │
│                                                                 │
│ DIMENSION 5: CUSTOMER CORRELATION (Does it predict impact?)     │
│                                                                 │
│   5: Fires if and only if customers are impacted                │
│   4: Strongly correlated with customer impact                   │
│   3: Sometimes fires without customer impact (leading indicator)│
│   2: Weak correlation with customer experience                  │
│   1: No measurable correlation with customer experience         │
│                                                                 │
│   Measured by: correlation between alert and SLI degradation    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation — automated scoring where possible:**

```yaml
# Recording rules for automated alert quality metrics
groups:
  - name: alert-quality-metrics
    interval: 1m
    rules:
      # Track how often each alert fires
      - record: alert:firing_count:monthly
        expr: |
          count_over_time(ALERTS{alertstate="firing"}[30d])

      # Track auto-resolution (likely false positives)
      - record: alert:auto_resolved_ratio:monthly
        expr: |
          count_over_time(ALERTS{alertstate="firing"}[30d] @ end() unless ALERTS{alertstate="firing"}[30d] @ start())
          /
          count_over_time(ALERTS{alertstate="firing"}[30d])
```

**Monthly review process:**

```markdown
## Alert Quality Review — [Month]

| Alert | Fires/mo | Action Rate | Has Runbook | Avg Score | Action |
|-------|----------|-------------|-------------|-----------|--------|
| RedisTimeout | 150 | 5% | No | 6/25 | DELETE, replace with rate-based |
| PaymentSvcHighBurn | 3 | 100% | Yes | 23/25 | No change |
| HighMemory | 45 | 30% | Partial | 11/25 | Tune threshold, complete runbook |
| CertExpiry30d | 2 | 100% | Yes | 20/25 | No change (ticket, not page) |

### Decisions
- Alerts scoring <10/25: Must be fixed or deleted by next review
- Alerts scoring 10-14/25: Must have improvement plan filed as ticket
- Alerts scoring ≥15/25: No action required
- New alerts must score ≥15/25 at first review or get demoted to ticket
```

---

## Q3: Postmortem Under Pressure

### 1. Handling the VP's Demand for Individual Accountability

**Exact words:**

"Thank you for raising that, [VP name]. I understand the board wants accountability, and they'll get it — but the most effective form of accountability is fixing the systems that allowed this to happen, not identifying an individual to blame.

Here's why I say that specifically: If we name an individual as responsible, we get two outcomes — both bad. First, that engineer and everyone watching learns to hide mistakes instead of reporting them quickly. Our mean-time-to-detect goes up because people cover their tracks instead of raising their hand. Second, we fix nothing systemic — the same failure mode will recur with a different engineer next quarter.

What I can give the board is something much more valuable: 'Here are the three systemic gaps that allowed this incident, here are the specific fixes with owners and deadlines, and here is the verification plan that proves the fixes work.' That's accountability that actually prevents the next outage.

Can we proceed with identifying those systemic gaps? I think we'll find that the root cause isn't a person — it's a process that let a risky change reach production without the right safeguards."

**If the VP pushes back ("The board specifically asked WHO"):**

"I'd suggest framing it to the board as: 'The change was deployed by a team member following our standard process — the process itself lacked adequate safeguards.' The accountability sits with the engineering organization for not having those safeguards, and here's the plan to fix it. I can help you draft that summary if that would be useful.

Naming an individual to the board creates legal and HR risk as well. [VP name], perhaps we can discuss the board communication separately after this meeting — I want to make sure we get the most out of this postmortem with the full team present."

### 2. Redirecting the Tech Lead's Blame Comment

The tech lead said: *"If we had just reviewed the PR properly, none of this would have happened."*

**Exact words:**

"That's an interesting thread to pull on, [tech lead name]. Let's dig into it — but let's reframe slightly. Instead of 'we should have reviewed the PR properly,' let's ask: **what about our review process made it possible for this to slip through?**

Was the PR review checklist missing a specific check for this type of change? Was the reviewer under time pressure because of sprint commitments? Was the risk not visible in the diff because the dangerous behavior was in a downstream interaction that doesn't show up in a code review?

I'm asking because 'review PRs better' isn't an action item — it's a wish. 'Add automated check X to the CI pipeline that catches this specific pattern' IS an action item. Let's find the specific, implementable fix.

[Developer name who made the change], you have the most context here — can you walk us through what the PR looked like? What did the diff show, and what would have made the risk more visible?"

**What this does:**
1. Validates the tech lead's concern (doesn't dismiss them)
2. Redirects from "who failed" to "what systemic gap exists"
3. Converts a blame statement into an investigation question
4. Explicitly brings in the developer as a contributor, not a defendant

### 3. Creating Safety for the Developer

**Before the meeting — private Slack DM to the developer:**

```
Hey [name] — I'm facilitating the postmortem for last week's incident.
I want you to know: this is a blameless process. You are not in trouble. 
The code change was the trigger, but the root cause is always systemic — 
our systems should have caught this before it reached production.

Your perspective is the most valuable in the room because you have the 
deepest context on what happened. I'll be asking you to walk through the 
timeline from your point of view. Just describe what happened factually — 
what you saw, what you did, what you expected to happen.

If at any point the conversation feels like it's becoming personal, I'll 
redirect it. That's my job as facilitator. You're safe.

See you at 2 PM.
```

**During the meeting — after the tech lead's comment has been redirected:**

"[Developer name], you're the person with the most context on this change. Can you walk us through the timeline? Start from when you picked up the ticket — what did you build, what was the review process, and what happened when it deployed?

I want to be explicit: we're doing this to understand the sequence of events, not to assign blame. Every person in this room has shipped a change that caused an incident or will in the future — that's normal in complex systems. What matters is what we learn."

**If the developer is still visibly anxious and not contributing:**

"Let me make this easier. [Developer name], I've got the timeline from the logs and metrics. Let me share what I see, and you can correct or add context where I'm wrong."

Then present the factual timeline yourself, pausing at key decision points:

"At this point, the PR was approved and merged. [Developer name], what did the CI pipeline show at this stage? Did it run any checks specific to Istio configurations?"

**This technique:**
- Removes the burden of self-narration from a stressed person
- Gives them the role of "fact-checker" instead of "defendant"
- Uses specific questions that have factual (not judgmental) answers
- Demonstrates that you've already done the investigation and aren't on a witch hunt

### 4. Undone Action Items from 2 Months Ago

**Address it directly in the postmortem:**

"Before we close, I want to flag something. We have two action items from the October postmortem that are still open. [Read them aloud.] I'm not calling anyone out — but I want us to look at this as a system problem. If action items routinely don't get done, our postmortem process is generating paperwork, not improvement. Every undone action item is a bet that the failure mode won't recur. Sometimes that bet pays off. Sometimes you get today's incident.

Let me ask the group: why didn't these get done? Were they too vague? Too large? Not prioritized by product? Assigned to someone who was pulled to other work?"

**Systemic fix — postmortem action item lifecycle:**

```
┌─────────────────────────────────────────────────────────────────┐
│ POSTMORTEM ACTION ITEM LIFECYCLE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 1. CREATION (during postmortem):                                │
│    - Every action item gets: Owner, Priority, Due Date          │
│    - P1 action items: due within 1 week                         │
│    - P2 action items: due within 1 sprint (2 weeks)             │
│    - P3 action items: due within 1 month                        │
│    - Action items without due dates are not action items        │
│                                                                 │
│ 2. TRACKING:                                                    │
│    - Action items are filed as Jira tickets (not just in the    │
│      postmortem doc — docs get forgotten, tickets get tracked)  │
│    - Tagged with label: "postmortem-action"                     │
│    - Linked to the postmortem document                          │
│                                                                 │
│ 3. REVIEW:                                                      │
│    - Weekly SRE standup: review open postmortem action items     │
│    - Overdue items escalated to team lead, then to eng manager  │
│    - Monthly leadership report includes:                        │
│      "X of Y postmortem actions completed on time"              │
│                                                                 │
│ 4. CLOSURE:                                                     │
│    - Action item isn't "done" until it's VERIFIED               │
│    - "Add alert for X" → done when alert exists AND has fired   │
│      in a test scenario                                         │
│    - "Fix deployment pipeline" → done when the same failure     │
│      mode is tested and confirmed blocked                       │
│                                                                 │
│ 5. ESCALATION LADDER:                                           │
│    - 1 week overdue: Slack reminder to owner                    │
│    - 2 weeks overdue: Discussed in team standup                 │
│    - 3 weeks overdue: Escalated to engineering manager          │
│    - 4+ weeks overdue: Flagged in monthly leadership review     │
│                                                                 │
│ 6. MEASUREMENT:                                                 │
│    - Postmortem Action Completion Rate (target: >90% on time)   │
│    - Median Time to Complete postmortem actions                  │
│    - Recurrence Rate: % of incidents where a previous           │
│      postmortem action would have prevented/mitigated           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Automated tracking:**

```bash
# Jira JQL to find overdue postmortem actions
jql = 'labels = "postmortem-action" AND status != Done AND due < now()'

# Weekly Slack report (cron job or Jira automation)
# Posts to #sre-postmortem-actions every Monday:
```

```python
# postmortem_action_tracker.py
import jira
import slack_sdk

client = jira.JIRA(server=JIRA_URL, token_auth=JIRA_TOKEN)

overdue = client.search_issues(
    'labels = "postmortem-action" AND status != Done AND due < now()',
    maxResults=50
)

if overdue:
    blocks = [{
        "type": "header",
        "text": {"type": "plain_text", "text": f"⚠️ {len(overdue)} Overdue Postmortem Action Items"}
    }]
    for issue in overdue:
        days_overdue = (datetime.now() - issue.fields.duedate).days
        blocks.append({
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*<{JIRA_URL}/browse/{issue.key}|{issue.key}>*: {issue.fields.summary}\n"
                        f"Owner: {issue.fields.assignee.displayName} | "
                        f"Due: {issue.fields.duedate} | "
                        f"*{days_overdue} days overdue*"
            }
        })

    slack = slack_sdk.WebClient(token=SLACK_TOKEN)
    slack.chat_postMessage(channel="#sre-postmortem-actions", blocks=blocks)
```

---

## Q4: Chaos Engineering Game Day Design

### 1. Executive Summary for VP Approval

"NovaMart's payment system generates $72M/month. Our monitoring tells us when things ARE broken — but we have untested assumptions about what happens when infrastructure fails. A controlled game day lets us discover those blind spots on our terms (Wednesday 10 AM with the whole team watching) instead of discovering them on the customer's terms (Saturday 2 AM with one on-call engineer).

We will run four experiments with escalating severity, each with a pre-defined abort condition that rolls back within 60 seconds. The engineering team will be on standby, and we'll start with non-production, moving to production only for experiments where we have high confidence. The business risk is a maximum of 2 minutes of degraded service for a subset of users — versus the 14-minute uncontrolled outage we had last month that cost $84K.

The deliverable is a concrete list of gaps in our resilience — the things we think work but haven't proven. Every FAANG company runs game days; the question isn't whether we'll have failures, it's whether we discover them on our schedule or the customer's."

### 2. Four Experiments — Escalating Severity

**Experiment 1: Single Pod Kill (Low Risk)**

```
┌─────────────────────────────────────────────────────────────────┐
│ EXPERIMENT 1: Pod Termination — payment-svc                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Hypothesis: "If a single payment-svc pod is killed, Kubernetes  │
│ will reschedule it within 30 seconds and the service will       │
│ maintain its 99.95% availability SLO with zero customer impact."│
│                                                                 │
│ Injection method:                                               │
│   kubectl -n payments delete pod payment-svc-<hash> --grace-period=0 │
│                                                                 │
│ Pre-conditions:                                                 │
│   - payment-svc has ≥3 replicas running                         │
│   - PodDisruptionBudget is configured (minAvailable: 2)         │
│   - Verify current SLI is healthy before starting               │
│                                                                 │
│ Success criteria (in SLI terms):                                │
│   ✅ sli:payment_charge_availability:ratio_rate5m > 0.999       │
│      (no measurable drop in availability)                       │
│   ✅ histogram_quantile(0.99, ...) < 2s                         │
│      (p99 latency stays within SLO)                             │
│   ✅ New pod reaches Ready within 30 seconds                    │
│   ✅ Zero 5xx errors during the experiment                      │
│                                                                 │
│ Abort condition:                                                │
│   IF error_rate > 1% for > 30 seconds                           │
│   OR p99_latency > 5s for > 30 seconds                          │
│   THEN: Abort (pod is already being replaced by K8s)            │
│   Recovery: kubectl -n payments scale deploy/payment-svc --replicas=5 │
│                                                                 │
│ Blast radius: 1 pod out of 3+ → ~33% capacity reduction         │
│ Expected duration: 2 minutes observation window                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Experiment 2: Dependency Latency Injection (Medium Risk)**

```
┌─────────────────────────────────────────────────────────────────┐
│ EXPERIMENT 2: Inject 3-second latency on payment-svc → Stripe   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Hypothesis: "If Stripe API responses are delayed by 3 seconds,  │
│ payment-svc's circuit breaker will open after 5 consecutive     │
│ slow responses, returning a fast-fail to clients within 500ms   │
│ instead of letting the latency cascade to order-svc."           │
│                                                                 │
│ Injection method (Istio fault injection):                       │
│                                                                 │
│   apiVersion: networking.istio.io/v1beta1                       │
│   kind: VirtualService                                          │
│   metadata:                                                     │
│     name: stripe-fault-injection                                │
│     namespace: payments                                         │
│   spec:                                                         │
│     hosts:                                                      │
│       - api.stripe.com                                          │
│     http:                                                       │
│       - fault:                                                  │
│           delay:                                                │
│             percentage:                                         │
│               value: 100                                        │
│             fixedDelay: 3s                                      │
│         route:                                                  │
│           - destination:                                        │
│               host: api.stripe.com                              │
│                                                                 │
│ Success criteria:                                               │
│   ✅ Circuit breaker opens within 30 seconds                    │
│   ✅ payment-svc returns 503 with retry-after header            │
│      (fast-fail, not slow-fail)                                 │
│   ✅ order-svc handles the 503 gracefully (queues the order     │
│      for retry, doesn't crash)                                  │
│   ✅ p99 latency of order-svc stays < 5s (doesn't inherit       │
│      the 3s injection)                                          │
│                                                                 │
│ Abort condition:                                                │
│   IF order-svc error rate > 5% for > 60 seconds                 │
│   OR payment-svc p99 > 10s for > 60 seconds                     │
│   OR any service starts crash-looping                           │
│   THEN:                                                         │
│     kubectl -n payments delete virtualservice stripe-fault-injection │
│                                                                 │
│ Blast radius: All Stripe-bound requests from payment-svc         │
│ Expected duration: 5 minutes                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Experiment 3: Availability Zone Failure (High Risk)**

```
┌─────────────────────────────────────────────────────────────────┐
│ EXPERIMENT 3: Simulate us-east-1a AZ failure                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Hypothesis: "If AZ us-east-1a becomes unavailable, NovaMart's   │
│ services will automatically failover to us-east-1b and 1c with  │
│ <30 seconds of impact. The ALB health checks will drain 1a      │
│ targets within 15 seconds, RDS Multi-AZ will failover within    │
│ 30 seconds, and ElastiCache will promote a replica within 20    │
│ seconds."                                                       │
│                                                                 │
│ Injection method:                                               │
│   # Block all traffic from AZ us-east-1a nodes                  │
│   # Using AWS FIS (Fault Injection Simulator):                  │
│                                                                 │
│   aws fis create-experiment-template \                          │
│     --description "AZ failure simulation" \                     │
│     --targets '{                                                │
│       "az-nodes": {                                             │
│         "resourceType": "aws:ec2:instance",                     │
│         "selectionMode": "ALL",                                 │
│         "resourceTags": {"topology.kubernetes.io/zone": "us-east-1a"}, │
│         "filters": [{"path":"State.Name","values":["running"]}] │
│       }                                                         │
│     }' \                                                        │
│     --actions '{                                                │
│       "az-failure": {                                           │
│         "actionId": "aws:ec2:send-spot-instance-interruptions", │
│         "parameters": {},                                       │
│         "targets": {"Instances": "az-nodes"}                    │
│       }                                                         │
│     }' \                                                        │
│     --stop-conditions '[{                                       │
│       "source": "aws:cloudwatch:alarm",                         │
│       "value": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:GameDayAbort" │
│     }]' \                                                       │
│     --role-arn arn:aws:iam::123456789012:role/FISRole            │
│                                                                 │
│   # Or simpler — cordon and drain AZ nodes:                     │
│   for node in $(kubectl get nodes -l topology.kubernetes.io/zone=us-east-1a -o name); do │
│     kubectl cordon $node                                        │
│     kubectl drain $node --grace-period=0 --force --ignore-daemonsets │
│   done                                                          │
│                                                                 │
│ Success criteria:                                               │
│   ✅ Overall availability > 99.9% during the failover window    │
│   ✅ RDS failover completes within 30 seconds                   │
│   ✅ ALB shifts traffic to healthy AZs within 15 seconds        │
│   ✅ No data loss (all committed transactions preserved)        │
│   ✅ Pods rescheduled to other AZs within 60 seconds            │
│                                                                 │
│ Abort condition:                                                │
│   IF overall_error_rate > 10% for > 2 minutes                   │
│   OR any data store reports data loss/corruption                │
│   OR payment processing completely halted for > 60 seconds      │
│   THEN:                                                         │
│     # Uncordon all nodes immediately                            │
│     for node in $(kubectl get nodes -l topology.kubernetes.io/zone=us-east-1a -o name); do │
│       kubectl uncordon $node                                    │
│     done                                                        │
│     #                                                           |
│     # Uncordon all nodes immediately                            │
│     for node in $(kubectl get nodes -l topology.kubernetes.io/zone=us-east-1a -o name); do │
│       kubectl uncordon $node                                    │
│     done                                                        │
│     # Or if using FIS: stop the experiment                      │
│     aws fis stop-experiment --id <experiment-id>                │
│                                                                 │
│ Blast radius: ~33% of compute capacity (1 of 3 AZs)            │
│ Expected duration: 10 minutes observation after injection       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Experiment 4: Cascading Failure — Payment Provider Outage During Peak (Highest Risk)**

```
┌─────────────────────────────────────────────────────────────────┐
│ EXPERIMENT 4: Complete Stripe API outage + traffic surge        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Hypothesis: "If Stripe becomes completely unreachable during    │
│ 2x normal traffic, payment-svc's circuit breaker opens within   │
│ 10 seconds, order-svc queues affected orders for retry,         │
│ customer-facing error rate stays below 5%, and the system       │
│ recovers automatically within 60 seconds of Stripe returning."  │
│                                                                 │
│ Injection method (two simultaneous injections):                 │
│                                                                 │
│   # Injection A: Block all Stripe traffic via NetworkPolicy     │
│   apiVersion: networking.k8s.io/v1                              │
│   kind: NetworkPolicy                                           │
│   metadata:                                                     │
│     name: block-stripe-gameday                                  │
│     namespace: payments                                         │
│   spec:                                                         │
│     podSelector:                                                │
│       matchLabels:                                              │
│         app: payment-svc                                        │
│     policyTypes:                                                │
│       - Egress                                                  │
│     egress:                                                     │
│       - to:                                                     │
│         # Allow everything EXCEPT Stripe's IP ranges            │
│         - ipBlock:                                              │
│             cidr: 0.0.0.0/0                                     │
│             except:                                             │
│               - 3.18.12.63/32    # Stripe API IPs               │
│               - 3.130.192.163/32 # (example — verify current)   │
│               - 13.110.215.0/24                                 │
│                                                                 │
│   # Injection B: Generate 2x traffic via load generator         │
│   # Using k6 or Locust pointed at order-svc /checkout           │
│   k6 run --vus 200 --duration 5m loadtest-checkout.js           │
│                                                                 │
│ Success criteria:                                               │
│   ✅ Circuit breaker opens within 10 seconds of Stripe block    │
│   ✅ payment-svc returns fast-fail (503) within 500ms           │
│      (not hanging for timeout duration)                         │
│   ✅ order-svc enqueues failed orders to DLQ/retry queue        │
│   ✅ Customer-facing error message is graceful ("Payment is     │
│      temporarily unavailable, your order is saved")             │
│   ✅ No cascading failure to other services (cart-svc,          │
│      search-svc unaffected)                                     │
│   ✅ When Stripe is unblocked, system recovers within 60s       │
│   ✅ Queued orders are retried and processed successfully       │
│                                                                 │
│ Abort condition:                                                │
│   IF any non-payment service error rate > 5% for > 30 seconds   │
│   OR payment-svc pods start OOM/crash-looping                   │
│   OR order-svc DLQ depth > 10,000 (runaway queue growth)        │
│   OR any database connection pool exhaustion detected           │
│   THEN:                                                         │
│     kubectl -n payments delete networkpolicy block-stripe-gameday │
│     # Stop the load generator                                   │
│     k6 cloud abort <test-run-id>                                │
│                                                                 │
│ Blast radius: All payment processing                            │
│ Expected duration: 5 minutes blocked, 5 minutes recovery        │
│                                                                 │
│ ⚠️ RUN THIS IN STAGING FIRST. Only run in production if         │
│    Experiments 1-3 all passed and team is confident.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Proving the Payments Team Has Untested Assumptions

The payments team says "our service is fine." Pull these queries to show them what they **assume** works but have **never tested:**

**Assumption 1: "Our circuit breaker works"**

```promql
# Has the circuit breaker EVER opened in production?
sum(increase(circuit_breaker_state_transitions_total{
  service="payment-svc",
  to_state="open"
}[90d]))

# If this returns 0: the circuit breaker has NEVER been tested
# in production. You're trusting untested code for your most
# critical failure mode.
```

```
If result = 0:
  "Your circuit breaker has never opened in 90 days. That means 
  either you've never had a downstream failure (unlikely) or your 
  circuit breaker thresholds are misconfigured and it will never 
  open. Do you want to find out which during a real Stripe outage 
  at 2 AM, or during a controlled test at 10 AM with the team 
  watching?"
```

**Assumption 2: "RDS failover is seamless"**

```promql
# When was the last RDS failover?
aws rds describe-events \
  --source-type db-instance \
  --source-identifier novamart-payment-db \
  --event-categories failover \
  --duration 10080  # Last 7 days in minutes

# Check if application handles failover gracefully
# Look for connection errors during any past failover:
sum(rate(db_connection_errors_total{service="payment-svc"}[5m]))
# Overlay with RDS failover events — were there error spikes?
```

```
If no recent failovers:
  "Your RDS hasn't failed over in months. RDS Multi-AZ failover 
  takes 30-60 seconds. During that time, does payment-svc retry 
  connections? Does it return 503 or does it hang? Does the 
  connection pool recover automatically or does it need a pod 
  restart? You don't know. Let's find out."
```

**Assumption 3: "We handle pod failures gracefully"**

```promql
# How are pods distributed across AZs?
count by (topology_kubernetes_io_zone) (
  kube_pod_info{namespace="payments", pod=~"payment-svc.*"}
)

# If all pods are in one AZ:
# AZ failure = 100% payment outage, not 33%
```

```promql
# Is there a PodDisruptionBudget?
kube_poddisruptionbudget_status_current_healthy{namespace="payments"}

# If no PDB exists:
# A node drain during maintenance can kill ALL pods simultaneously
```

```bash
kubectl -n payments get pdb
# If empty: "You have no PodDisruptionBudget. During a node 
# upgrade, Kubernetes can terminate all your pods simultaneously. 
# That's a payment outage."
```

**Assumption 4: "Our retry logic works"**

```promql
# Has retry logic ever triggered successfully?
sum(increase(http_client_retries_total{service="payment-svc"}[30d]))

# What's the retry success rate when it does trigger?
sum(rate(http_client_retries_total{service="payment-svc", result="success"}[30d]))
/
sum(rate(http_client_retries_total{service="payment-svc"}[30d]))

# If retry count = 0 or retry success rate is unknown:
# "You've never actually retried in production. Is the retry 
# code even working? Is it retrying on the right error codes? 
# Is it using exponential backoff or hammering the downstream?"
```

**Assumption 5: "Our timeout settings are correct"**

```promql
# What's the actual p99 latency to Stripe?
histogram_quantile(0.99,
  sum by (le) (
    rate(http_client_request_duration_seconds_bucket{
      service="payment-svc",
      remote=~".*stripe.*"
    }[7d])
  )
)

# What's the configured timeout?
# If p99 = 1.2s and timeout = 30s:
# "Your timeout is 25x your p99 latency. If Stripe slows down,
# your threads will be blocked for 30 seconds each before timing
# out. At 500 req/s, that's 15,000 blocked threads before the
# first timeout fires. Your pod will OOM before the timeout helps."
```

**Present all of this as a one-page summary:**

```markdown
## Payment-svc: Untested Assumptions

| Assumption | Evidence | Risk |
|---|---|---|
| Circuit breaker works | Never opened in 90 days | Unknown failure mode during Stripe outage |
| RDS failover is seamless | No failover in 6 months | 30-60s of unknown behavior |
| Pod failure is handled | No PDB, pods clustered in 1 AZ | AZ failure = total outage |
| Retry logic works | Zero retries recorded | Dead code? Wrong error codes? |
| Timeouts are correct | 30s timeout vs 1.2s p99 | Thread exhaustion before timeout helps |

**Question for the team:** Are you comfortable betting $50K/minute that all five of these work correctly, having never tested any of them?
```

### 4. Experiment 3 Unexpected Cascade — RDS Failover During AZ Failure

**What happened:**

```
t=0s     AZ us-east-1a nodes cordoned and drained
t=2s     Pods in 1a terminating, ALB health checks failing for 1a targets
t=5s     ALB starts routing traffic only to 1b and 1c targets ✅
t=5s     RDS PRIMARY was in us-east-1a — connection refused for ALL writes
t=5-45s  ALL order-svc database writes FAIL (inserts, updates)
         order-svc returns 500 for every /checkout request
         payment-svc can't record transactions
         45 SECONDS OF COMPLETE ORDER PROCESSING FAILURE
t=45s    RDS Multi-AZ failover completes — new primary in us-east-1b
t=46s    Application connection pools... DON'T RECONNECT
         Still pointing at old primary endpoint
t=46-???s  STILL FAILING — stale connections in pool
```

**In the moment — what you do:**

**Second 0-30: Observe. This is expected behavior (AZ drain). Monitor.**

```
War room message:
"Experiment 3 in progress. AZ us-east-1a draining. 
Monitoring error rates across all services."
```

**Second 30-60: Error rate spiking — this is the unexpected cascade.**

```
War room message:
"⚠️ Unexpected: order-svc error rate spiking to 100% on /checkout.
Errors are 500 Internal Server Error, not 502.
Investigating — this looks like a database issue, not routing."
```

```promql
# Confirm database errors
sum(rate(db_connection_errors_total{service="order-svc"}[1m]))

# Check which database operation is failing
sum by (operation) (
  rate(db_query_errors_total{service="order-svc"}[1m])
)
```

**Second 60-90: RDS failover completed but connections stale.**

```bash
# Check RDS failover status
aws rds describe-events \
  --source-type db-instance \
  --source-identifier novamart-order-db \
  --duration 5

# Expected: "Multi-AZ instance failover started"
#           "Multi-AZ instance failover completed"

# Check current primary endpoint
aws rds describe-db-instances \
  --db-instance-identifier novamart-order-db \
  --query 'DBInstances[0].Endpoint.Address'
```

**Second 90: Error rate still elevated. Connection pool stale. Force recovery:**

```bash
# Option A: Restart order-svc pods to force new connections
kubectl -n orders rollout restart deploy/order-svc
kubectl -n orders rollout status deploy/order-svc --timeout=120s

# Option B: If the app supports connection pool refresh without restart
# (e.g., HikariCP has a refresh endpoint, or send SIGHUP)
kubectl -n orders exec deploy/order-svc -- curl -X POST localhost:8081/admin/db/reconnect
```

**Second 120: Evaluate whether to continue or abort.**

```
War room message:
"Order-svc recovering after pod restart. Error rate dropping.
RDS failover took ~45 seconds — during that window, all orders failed.
Application connection pools did NOT automatically reconnect after failover.

DECISION: Aborting experiment. Uncordoning AZ us-east-1a nodes.
Reason: Found TWO critical issues that need fixing before we proceed."
```

```bash
# Abort — uncordon all nodes
for node in $(kubectl get nodes -l topology.kubernetes.io/zone=us-east-1a -o name); do
  kubectl uncordon $node
done

# Verify nodes are schedulable
kubectl get nodes -l topology.kubernetes.io/zone=us-east-1a
# STATUS should show "Ready" (not "Ready,SchedulingDisabled")

# Verify pods are rebalancing
kubectl -n orders get pods -o wide
kubectl -n payments get pods -o wide
```

**Update status page:**

```bash
curl -X POST "https://api.statuspage.io/v1/pages/${PAGE_ID}/incidents" \
  -H "Authorization: OAuth ${STATUSPAGE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "incident": {
      "name": "Brief order processing interruption — planned testing",
      "status": "resolved",
      "impact_override": "minor",
      "body": "During a planned resilience test, some customers may have experienced brief order processing failures for approximately 90 seconds. All systems are now fully operational. No orders were lost — any failed orders can be retried."
    }
  }'
```

**Action items from this discovery:**

```markdown
## Game Day Findings — Experiment 3

### Finding 1: RDS failover causes ~45 seconds of complete write unavailability
**Expected:** 30 seconds (AWS documentation)
**Actual:** 45 seconds (real-world, under load)
**Impact:** ALL services depending on this RDS instance lost write capability

Action Items:
| # | Action | Owner | Priority | Due |
|---|--------|-------|----------|-----|
| 1 | Implement RDS Proxy for all production databases — absorbs failover transparently, maintains connection pool across failover | Platform | P1 | 2 weeks |
| 2 | Add database write failure handling in order-svc: queue to SQS/DLQ instead of returning 500 when DB is unreachable | Orders team | P1 | 2 weeks |
| 3 | Move to Aurora with read replicas + failover time <15s (vs RDS Multi-AZ ~45s) — evaluate cost/benefit | Platform | P2 | 1 month |

### Finding 2: Application connection pools don't recover after RDS failover
**Expected:** Connection pool detects dead connections and creates new ones
**Actual:** Stale connections remain in pool, all queries fail until pod restart
**Root cause:** Connection validation not configured (no `testOnBorrow` or `validationQuery`)

Action Items:
| # | Action | Owner | Priority | Due |
|---|--------|-------|----------|-----|
| 4 | Configure connection pool validation for ALL services using RDS | All teams | P1 | 1 week |
```

**Specific connection pool fix:**

```yaml
# For Spring/HikariCP (Java services):
spring:
  datasource:
    hikari:
      connection-timeout: 5000
      validation-timeout: 3000
      connection-test-query: "SELECT 1"
      max-lifetime: 600000        # 10 minutes — forces pool refresh
      idle-timeout: 300000         # 5 minutes
      minimum-idle: 5
      maximum-pool-size: 20
      # CRITICAL: enable leak detection
      leak-detection-threshold: 30000
```

```go
// For Go services using database/sql:
db.SetMaxOpenConns(20)
db.SetMaxIdleConns(5)
db.SetConnMaxLifetime(10 * time.Minute)  // Force connection refresh
db.SetConnMaxIdleTime(5 * time.Minute)

// Add health check loop
go func() {
    ticker := time.NewTicker(30 * time.Second)
    for range ticker.C {
        if err := db.Ping(); err != nil {
            log.Warn("Database ping failed, connections may be stale", "error", err)
            // The pool will automatically create new connections
            // on next query, but logging gives us visibility
        }
    }
}()
```

```yaml
# RDS Proxy — the architectural fix
# Terraform:
resource "aws_db_proxy" "orders" {
  name                   = "novamart-orders-proxy"
  debug_logging          = false
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800
  require_tls            = true
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  vpc_subnet_ids         = var.private_subnet_ids

  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }
}

resource "aws_db_proxy_default_target_group" "orders" {
  db_proxy_name = aws_db_proxy.orders.name

  connection_pool_config {
    connection_borrow_timeout    = 120
    max_connections_percent      = 100
    max_idle_connections_percent = 50
  }
}

resource "aws_db_proxy_target" "orders" {
  db_instance_identifier = aws_db_instance.orders.identifier
  db_proxy_name          = aws_db_proxy.orders.name
  target_group_name      = aws_db_proxy_default_target_group.orders.name
}

# Applications connect to the proxy endpoint instead of RDS directly
# Proxy handles failover transparently — connections are maintained
```

**Finding 3: We didn't know RDS primary was in us-east-1a**

```markdown
### Finding 3: No visibility into which AZ hosts RDS primary
**Expected:** Team knows which AZ hosts the primary before AZ-level experiments
**Actual:** Nobody checked; the primary happened to be in the failure AZ
**Impact:** What should have been a compute-only test became a data-tier test

Action Items:
| # | Action | Owner | Priority | Due |
|---|--------|-------|----------|-----|
| 5 | Add RDS primary AZ to the on-call home dashboard | Platform | P2 | 1 week |
| 6 | Pre-flight checklist for game days must include "identify all stateful resources in target AZ" | SRE | P1 | Before next game day |
```

```bash
# Command to check RDS primary AZ (should have been in pre-flight):
aws rds describe-db-instances \
  --db-instance-identifier novamart-order-db \
  --query 'DBInstances[0].AvailabilityZone'
# Output: "us-east-1a" ← this should have been a red flag before Experiment 3

# Also check ElastiCache primary AZ:
aws elasticache describe-replication-groups \
  --replication-group-id novamart-sessions \
  --query 'ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint'
```

---

## Summary

```
Q1: SEV1 Incident Response
  First 10 min: Acknowledge → Declare → Status page → Diagnose → Communicate
  502 + pods healthy = routing/mesh misconfiguration (not app crash)
  ArgoCD auto-sync + 0-replica canary = the exact failure mode
  Mitigate: patch VirtualService + disable auto-sync (ORDER MATTERS)
  "Where We Got Lucky" = the section that drives the best action items

Q2: On-Call Process Design
  Weeks 1-4: Stop the bleeding (audit, kill top 5 noisy alerts, runbooks)
  Weeks 5-8: Build muscle (training, handoff meetings, feedback loops)
  Weeks 9-12: Mature (automate, measure, present to leadership)
  Refusing on-call: technical argument + organizational argument + escalate
  Redis alert: don't delete — restructure from absolute threshold to error rate
  Alert quality: 5-dimension scorecard, minimum 15/25 to remain as page

Q3: Postmortem Under Pressure
  VP wants names → redirect to systemic accountability
  Tech lead blames reviewer → redirect to "what about the process"
  Anxious developer → private safety DM + fact-checker role in meeting
  Undone action items → Jira tickets + weekly review + escalation ladder
  
Q4: Chaos Engineering Game Day
  Executive pitch: "our schedule or the customer's"
  4 experiments: pod kill → dependency latency → AZ failure → cascading
  Prove untested assumptions with existing metrics (circuit breaker, PDB, timeouts)
  Unexpected cascade: RDS failover + stale connection pools
  Fix: RDS Proxy + connection pool validation + pre-flight AZ checklist
```

