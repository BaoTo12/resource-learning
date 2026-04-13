# Module 04 — Metrics: Counters, Gauges, Histograms & Views

> **Professor's Note:** Metrics are the heartbeat of your system. While traces help you debug individual requests, metrics tell you how the *entire system* is performing at scale. In this module, you will learn all four metric instrument types, semantic conventions for metrics, and how to expose them for Prometheus scraping.

---

## 📖 Lecture 4.1 — Why Metrics?

Consider this: you have 10,000 requests per second. You cannot look at a trace for every one of them. Metrics aggregate the behavior of *all* those requests into queryable numbers.

```
10,000 individual traces                  3 metrics
─────────────────────────────    →       ─────────────────────────────
req-1: 45ms OK                           requests_total = 10,000
req-2: 52ms OK                           error_rate = 0.2%
req-3: ERROR 500                         p99_latency = 87ms
req-4: 41ms OK
... (9,996 more)
```

**Metrics excel at:**
- Alerting (SLOs: "99.9% of requests must succeed")
- Dashboards (system health at a glance)
- Capacity planning (correlate load with resource usage)
- Trend analysis (is latency getting worse over 30 days?)
- Business KPIs (orders per second, revenue rate)

---

## 📖 Lecture 4.2 — The Four Metric Instruments

OpenTelemetry provides 4 fundamental instrument types. Choosing the right one is critical.

### 1. Counter — Monotonically Increasing

A value that **only goes up** (or resets to zero on restart). You always `add()` to it.

```
Use for: "How many times did X happen?"
Examples:
  - Total HTTP requests
  - Total errors
  - Total bytes transferred
  - Total orders placed

DO NOT use Counter for:
  - Values that can decrease (use UpDownCounter)
  - Current state (use Gauge)
```

```python
from opentelemetry import metrics

meter = metrics.get_meter("order-service")

# Create a counter
requests_counter = meter.create_counter(
    name="http.server.requests",
    description="Total HTTP requests received",
    unit="1",   # '1' = dimensionless count
)

# Use it:
requests_counter.add(1, {"http.method": "POST", "http.route": "/orders", "status": "200"})
```

### 2. UpDownCounter — Can Go Up or Down

Like Counter but supports negative additions (the value can decrease).

```
Use for: "What is the current count of active X?"
Examples:
  - Active database connections (comes and goes)
  - Items in a queue (added and removed)
  - Active WebSocket connections
  - Number of background jobs running

DO NOT use:
  - When value can never decrease (use Counter)
```

```python
active_connections = meter.create_up_down_counter(
    name="db.connection.pool.active",
    description="Active database connections",
    unit="1",
)

# When connection is acquired:
active_connections.add(1, {"pool": "primary"})

# When connection is released:
active_connections.add(-1, {"pool": "primary"})
```

### 3. Histogram — Value Distribution

Records the distribution of a value over time. This is what gives you p50, p95, p99.

```
Use for: "How long did X take?" or "How large was X?"
Examples:
  - Request duration (latency)
  - Request/response body size
  - Database query time
  - Queue wait time
  - Batch processing size

Recording a value adds it to buckets automatically.
```

```python
request_duration = meter.create_histogram(
    name="http.server.request.duration",
    description="HTTP request duration",
    unit="s",   # Seconds — always use seconds for duration in OTel
)

import time
start = time.time()
# ... process request ...
duration = time.time() - start

request_duration.record(
    duration,  # in seconds
    {"http.method": "POST", "http.route": "/orders", "http.status_code": 200}
)
```

### 4. Gauge — Current Value (Observed)

Represents a reading taken at a specific point in time. The value can go up and down freely. Typically used for values you *observe* (like memory) rather than values you *increment*.

```
Use for: "What is the current value of X right now?"
Examples:
  - Memory usage
  - CPU utilization
  - Temperature
  - Cache size
  - Thread count
```

```python
# Gauge uses an observable callback — called by SDK on collection
def get_memory_usage(_options):
    import psutil
    return [
        metrics.Observation(psutil.virtual_memory().used, {"type": "rss"}),
        metrics.Observation(psutil.virtual_memory().percent, {"type": "percent"}),
    ]

memory_gauge = meter.create_observable_gauge(
    name="process.memory.usage",
    description="Process memory usage",
    unit="By",   # Bytes
    callbacks=[get_memory_usage],
)
```

---

## 📖 Lecture 4.3 — Attributes (Labels/Dimensions)

Attributes on metrics are like "labels" in Prometheus or "dimensions" in CloudWatch. They let you break down metrics by category.

```python
# Without attributes — only tells you total requests
requests_counter.add(1)

# With attributes — tells you which endpoint, method, and status
requests_counter.add(1, {
    "http.method": "POST",
    "http.route": "/orders",
    "http.status_code": 200,
    "service.name": "order-service",
})
```

### The Cardinality Warning ⚠️

**Cardinality** = the number of unique label combinations.

```
# LOW cardinality (good) — finite, predictable values
{"http.method": "GET|POST|PUT|DELETE"}      # 4 values
{"http.status_code": "200|400|500|..."}     # ~10 values
→ 4 × 10 = 40 unique series

# HIGH cardinality (dangerous!) — unbounded values
{"user.id": "user-1, user-2, ..."}         # millions of values
{"order.id": "ORD-1, ORD-2, ..."}          # millions of values
→ Millions of unique series → destroys Prometheus!
```

**Rule:** Never put high-cardinality values as metric attributes. Use trace attributes instead.

### Semantic Conventions for Metrics

```python
# HTTP Server metrics (standard names and attributes)
http_server_requests = meter.create_counter("http.server.request.count")
http_server_duration = meter.create_histogram("http.server.request.duration", unit="s")

# Record with semantic attributes:
http_server_requests.add(1, {
    "http.request.method": "POST",
    "http.route": "/orders",
    "http.response.status_code": 200,
    "url.scheme": "https",
    "server.address": "api.example.com",
})
```

---

## 📖 Lecture 4.4 — MeterProvider & SDK Configuration

Just like traces, metrics need a Provider configured at startup:

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.resources import Resource

# Resource (same as traces — identifies this service)
resource = Resource({"service.name": "order-service"})

# Exporter: sends metrics via OTLP
exporter = OTLPMetricExporter(endpoint="http://localhost:4317", insecure=True)

# Reader: collects metrics every 60 seconds and exports them
reader = PeriodicExportingMetricReader(exporter, export_interval_millis=60_000)

# Provider: the central configuration
provider = MeterProvider(resource=resource, metric_readers=[reader])

# Register globally
metrics.set_meter_provider(provider)

# ── Now use metrics anywhere ──
meter = metrics.get_meter("my-component")
```

---

## 📖 Lecture 4.5 — Prometheus Exposition vs. OTLP

There are two ways to get metrics out of your application:

### Option A: OTLP Push (Recommended with OTel)

Your app **pushes** metrics via OTLP to the Collector, which exports to Prometheus.

```
App → OTLP → OTel Collector → Prometheus format → Prometheus scrapes
```

```python
# Periodic reader pushes on schedule
reader = PeriodicExportingMetricReader(
    OTLPMetricExporter(endpoint="http://collector:4317"),
    export_interval_millis=30_000,  # every 30 seconds
)
```

### Option B: Prometheus Pull (Exposition)

Prometheus **scrapes** your app's `/metrics` endpoint directly.

```python
from opentelemetry.exporter.prometheus import PrometheusExporter
from prometheus_client import start_http_server

# Start Prometheus metrics endpoint
start_http_server(port=8000, addr="0.0.0.0")

exporter = PrometheusExporter()
reader = metrics.sdk.metrics.export.MetricReader(exporter)
```

```
Prometheus ─────────── scrapes ──────────► /metrics endpoint (port 8000)
```

---

## 📖 Lecture 4.6 — Views: Customizing Metrics

**Views** let you customize how metrics are collected — change bucket boundaries, rename metrics, or drop attributes.

```python
from opentelemetry.sdk.metrics.view import View
from opentelemetry.sdk.metrics import Aggregation

# Custom histogram buckets (defaults are too coarse for your use case)
custom_latency_view = View(
    instrument_name="http.server.request.duration",
    aggregation=Aggregation.explicit_bucket_histogram(
        boundaries=[0.005, 0.01, 0.025, 0.05, 0.075, 0.1, 0.25, 0.5, 1.0, 2.5]
        # 5ms, 10ms, 25ms, 50ms, 75ms, 100ms, 250ms, 500ms, 1s, 2.5s
    ),
)

# Drop a high-cardinality attribute before it reaches Prometheus
filtered_view = View(
    instrument_name="http.server.requests",
    attribute_keys={"http.method", "http.route", "http.status_code"},
    # Drops any other attributes (like user.id, order.id) from this metric
)

# Register views with the provider
provider = MeterProvider(
    resource=resource,
    metric_readers=[reader],
    views=[custom_latency_view, filtered_view],
)
```

---

## 📖 Lecture 4.7 — The RED and USE Methods

These are industry frameworks for what metrics to create:

### RED Method (For Services)

Every service should have:
- **R** — Rate: requests per second
- **E** — Error rate: % of failed requests  
- **D** — Duration: latency distribution

```python
# Rate
request_counter = meter.create_counter("http.server.requests.total")

# Errors (or use status_code attribute on request_counter)
error_counter = meter.create_counter("http.server.errors.total")

# Duration
request_duration = meter.create_histogram("http.server.request.duration", unit="s")
```

### USE Method (For Resources)

For every resource (CPU, memory, disk, connections):
- **U** — Utilization: % of time the resource is busy
- **S** — Saturation: how much extra work is waiting
- **E** — Errors: error events for the resource

```python
# Database connection pool (USE method example)
pool_utilization = meter.create_observable_gauge("db.pool.utilization")
pool_connections_waiting = meter.create_up_down_counter("db.pool.queue_depth")
pool_errors = meter.create_counter("db.pool.errors.total")
```

---

## ✅ Module 04 — Review Questions

1. Name the 4 metric instrument types and when to use each.
2. What is the difference between a `Counter` and an `UpDownCounter`?
3. Why should you never put `user_id` or `order_id` as a metric attribute?
4. What does `PeriodicExportingMetricReader` do?
5. What is **cardinality** in the context of metrics?
6. What does the RED method stand for? Apply it to an order-service.
7. What is a **View** in OTel metrics? Give two examples of when you'd use one.
8. What unit should you use for latency histograms?
9. What is the difference between OTLP Push and Prometheus Pull?
10. What does a histogram store that a counter does not?

---

## 🧪 Lab 04 — Metrics in Action with Prometheus

**→ See [labs/lab-04-metrics.md](./labs/lab-04-metrics.md)**

---

*[← Module 03](../Module-03-Traces/README.md) | [Module 05 →](../Module-05-Logs/README.md)*
