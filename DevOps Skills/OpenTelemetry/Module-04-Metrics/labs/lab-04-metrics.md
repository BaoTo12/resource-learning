# Lab 04 — Metrics in Action: Prometheus + Grafana Stack

> **Objective:** Build a Python service instrumented with all four metric types, push metrics to Prometheus via the OTel Collector, and build a Grafana dashboard. By the end, you will have a real RED method dashboard for your service.

---

## ⏱️ Estimated Time: 90 minutes

---

## Part 1 — Install Packages

```bash
pip install \
  opentelemetry-api \
  opentelemetry-sdk \
  opentelemetry-exporter-otlp-proto-grpc \
  opentelemetry-exporter-prometheus \
  prometheus-client \
  flask
```

---

## Part 2 — Docker Compose Stack

Create `Module-04-Metrics/labs/docker-compose.yml`:

```yaml
version: "3.8"

services:
  # OpenTelemetry Collector — receives OTLP, exports to Prometheus format
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    command: ["--config=/etc/otel/config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus metrics exporter (collector exposes metrics here)

  # Prometheus — scrapes from OTel Collector
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=15d"

  # Grafana — visualizes Prometheus data
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

---

## Part 3 — OTel Collector Config

Create `Module-04-Metrics/labs/otel-collector-config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s

exporters:
  # Expose metrics in Prometheus format for Prometheus to scrape
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: otel
    const_labels:
      source: opentelemetry-collector

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

---

## Part 4 — Prometheus Config

Create `Module-04-Metrics/labs/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]
    # Collector exposes metrics at /metrics on port 8889
```

---

## Part 5 — The Metrics-Instrumented Service

Create `Module-04-Metrics/labs/code/metrics_service.py`:

```python
"""
metrics_service.py — A Flask service with comprehensive RED method metrics.

Run: python metrics_service.py
Then hit the endpoints with the test script.
"""

import time
import random
import threading
from flask import Flask, request, jsonify
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

# ============================================================
# CONFIGURE METRICS SDK
# ============================================================
resource = Resource({"service.name": "order-service", "service.version": "1.0.0"})

exporter = OTLPMetricExporter(
    endpoint="http://localhost:4317",
    insecure=True,
)

# Export every 15 seconds (good for demo; use 60s in production)
reader = PeriodicExportingMetricReader(exporter, export_interval_millis=15_000)
provider = MeterProvider(resource=resource, metric_readers=[reader])
metrics.set_meter_provider(provider)

meter = metrics.get_meter("order-service")

# ============================================================
# CREATE INSTRUMENTS — RED Method
# ============================================================

# R — Rate: Count all requests
http_requests_total = meter.create_counter(
    name="http_requests_total",
    description="Total HTTP requests",
    unit="1",
)

# E — Errors: Count error responses
http_errors_total = meter.create_counter(
    name="http_errors_total",
    description="Total HTTP error responses",
    unit="1",
)

# D — Duration: Request latency distribution
http_request_duration_seconds = meter.create_histogram(
    name="http_request_duration_seconds",
    description="HTTP request duration in seconds",
    unit="s",
)

# ============================================================
# BUSINESS METRICS
# ============================================================

orders_created_total = meter.create_counter(
    name="orders_created_total",
    description="Total orders successfully created",
    unit="1",
)

order_value_dollars = meter.create_histogram(
    name="order_value_dollars",
    description="Distribution of order values",
    unit="$",
)

inventory_checks_total = meter.create_counter(
    name="inventory_checks_total",
    description="Total inventory checks",
    unit="1",
)

# ============================================================
# RESOURCE METRICS (USE Method)
# ============================================================

# Active database connections (up-down counter)
active_db_connections = meter.create_up_down_counter(
    name="db_connections_active",
    description="Currently active database connections",
    unit="1",
)

# Queue depth
orders_queue_depth = meter.create_up_down_counter(
    name="orders_queue_depth",
    description="Orders waiting to be processed",
    unit="1",
)

# ============================================================
# OBSERVABLE GAUGE — polled on collection
# ============================================================

# We'll track this externally for the observable gauge
_current_memory_mb = 0.0

def get_memory_usage(_options):
    """Called by SDK when it's time to collect this metric"""
    import psutil
    mem = psutil.virtual_memory()
    return [
        metrics.Observation(mem.used / 1024 / 1024, {"type": "used"}),
        metrics.Observation(mem.available / 1024 / 1024, {"type": "available"}),
    ]

try:
    import psutil
    memory_gauge = meter.create_observable_gauge(
        name="process_memory_mb",
        description="Process memory usage in MB",
        unit="MB",
        callbacks=[get_memory_usage],
    )
    print("✅ psutil available — memory gauge active")
except ImportError:
    print("⚠️  psutil not installed. Run: pip install psutil")
    print("   Memory gauge will not be created")

# ============================================================
# MIDDLEWARE — Auto-instrument all routes
# ============================================================

app = Flask(__name__)

@app.before_request
def before_request():
    request._start_time = time.time()

@app.after_request
def after_request(response):
    duration = time.time() - request._start_time
    
    labels = {
        "http_method": request.method,
        "http_route": request.url_rule.rule if request.url_rule else "unknown",
        "http_status_code": str(response.status_code),
    }
    
    # R — Increment request counter
    http_requests_total.add(1, labels)
    
    # D — Record duration
    http_request_duration_seconds.record(duration, labels)
    
    # E — Count errors (4xx and 5xx)
    if response.status_code >= 400:
        error_labels = {**labels, "error_type": "4xx" if response.status_code < 500 else "5xx"}
        http_errors_total.add(1, error_labels)
    
    return response

# ============================================================
# ROUTES
# ============================================================

@app.route("/orders", methods=["POST"])
def create_order():
    body = request.get_json() or {}
    product_id = body.get("product_id", "PROD-001")
    quantity = body.get("quantity", 1)
    
    # Validate
    if not isinstance(quantity, int) or quantity <= 0:
        return jsonify({"error": "Invalid quantity"}), 400
    
    # Simulate inventory check
    inventory_checks_total.add(1, {"product_family": product_id[:4]})
    
    # Simulate processing time
    processing_time = random.uniform(0.02, 0.15)
    if product_id.startswith("PROD-9"):
        processing_time += 0.3  # Slow product family
    time.sleep(processing_time)
    
    # Simulate 5% failure
    if random.random() < 0.05:
        return jsonify({"error": "Service temporarily unavailable"}), 503
    
    # Calculate order value
    value = round(49.99 * quantity, 2)
    order_id = f"ORD-{random.randint(10000,99999)}"
    
    # Record business metrics
    orders_created_total.add(1, {
        "product_family": product_id[:4],
        "quantity_range": "small" if quantity <= 5 else "large",
    })
    order_value_dollars.record(value, {"product_family": product_id[:4]})
    
    return jsonify({"order_id": order_id, "total": value}), 201


@app.route("/inventory/<product_id>")
def get_inventory(product_id):
    time.sleep(random.uniform(0.005, 0.03))
    inventory_checks_total.add(1, {"product_family": product_id[:4]})
    return jsonify({"product_id": product_id, "available": random.randint(0, 100)})


@app.route("/health")
def health():
    return jsonify({"status": "ok"})


@app.route("/metrics-test")
def metrics_info():
    """Shows what metrics are being collected"""
    return jsonify({
        "metrics": [
            "http_requests_total (Counter)",
            "http_errors_total (Counter)",
            "http_request_duration_seconds (Histogram)",
            "orders_created_total (Counter)",
            "order_value_dollars (Histogram)",
            "inventory_checks_total (Counter)",
            "db_connections_active (UpDownCounter)",
            "orders_queue_depth (UpDownCounter)",
            "process_memory_mb (Gauge)",
        ],
        "endpoint": "Metrics exported via OTLP to collector:4317"
    })


if __name__ == "__main__":
    print("🚀 Metrics Service starting on port 5000...")
    print("📊 Exporting metrics to OTel Collector at localhost:4317")
    print("   → Collector exposes Prometheus at localhost:8889")
    print("   → Prometheus at localhost:9090")
    print("   → Grafana at localhost:3000 (admin/admin)")
    app.run(host="0.0.0.0", port=5000, debug=False)
```

---

## Part 6 — Load Generator

Create `Module-04-Metrics/labs/code/load_generator.py`:

```python
"""
load_generator.py — Generates realistic traffic to create interesting metrics

Run alongside metrics_service.py:
  python load_generator.py
"""

import time
import random
import requests

BASE_URL = "http://localhost:5000"
PRODUCTS = ["PROD-001", "PROD-002", "PROD-500", "PROD-901", "PROD-999"]

def send_order():
    """Send a random order"""
    payload = {
        "product_id": random.choice(PRODUCTS),
        "quantity": random.randint(1, 20),
    }
    try:
        r = requests.post(f"{BASE_URL}/orders", json=payload, timeout=5)
        return r.status_code
    except Exception as e:
        return None

def send_inventory_check():
    """Check inventory for a product"""
    product = random.choice(PRODUCTS)
    try:
        r = requests.get(f"{BASE_URL}/inventory/{product}", timeout=5)
        return r.status_code
    except Exception:
        return None

print("🔄 Load Generator starting...")
print("   Sending ~5 RPS to the metrics service")
print("   Press Ctrl+C to stop")
print()

request_count = 0
start_time = time.time()

try:
    while True:
        # Mix of request types
        if random.random() < 0.6:
            status = send_order()
            request_type = "order"
        else:
            status = send_inventory_check()
            request_type = "inventory"
        
        request_count += 1
        elapsed = time.time() - start_time
        rps = request_count / elapsed
        
        if request_count % 20 == 0:
            print(f"  Sent {request_count} requests | {rps:.1f} RPS | Last: {request_type} → {status}")
        
        # 5 RPS average
        time.sleep(random.uniform(0.1, 0.3))

except KeyboardInterrupt:
    print(f"\n✅ Stopped. Sent {request_count} total requests in {elapsed:.0f}s")
```

---

## Part 7 — Run the Stack

### Step 7.1 — Start Docker services

```bash
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry\Module-04-Metrics\labs"
docker compose up -d
docker compose ps  # Verify all 3 containers are running
```

### Step 7.2 — Start the service and load generator

**Terminal 1:**
```bash
cd code
python metrics_service.py
```

**Terminal 2:**
```bash
cd code
python load_generator.py
```

Wait **2-3 minutes** for metrics to accumulate.

---

## Part 8 — Explore in Prometheus

Open **http://localhost:9090**

### Try These Queries

```promql
# Total request rate (per second over 1 minute)
rate(otel_http_requests_total[1m])

# Error rate percentage
sum(rate(otel_http_errors_total[1m])) / sum(rate(otel_http_requests_total[1m])) * 100

# p95 request latency
histogram_quantile(0.95, sum(rate(otel_http_request_duration_seconds_bucket[1m])) by (le))

# p50 request latency by route
histogram_quantile(0.50,
  sum(rate(otel_http_request_duration_seconds_bucket[1m])) by (le, http_route)
)

# Orders per minute
sum(increase(otel_orders_created_total[1m]))

# Average order value
sum(increase(otel_order_value_dollars_sum[5m])) / sum(increase(otel_order_value_dollars_count[5m]))
```

---

## Part 9 — Build a Grafana Dashboard

### Step 9.1 — Connect Prometheus to Grafana

1. Open **http://localhost:3000** (admin / admin)
2. Click **Configuration → Data Sources → Add data source**
3. Select **Prometheus**
4. URL: `http://prometheus:9090`
5. Click **Save & Test** — should say "Data source is working"

### Step 9.2 — Create RED Method Dashboard

1. Click **+ → Dashboard → Add panel** (repeat for each panel)

**Panel 1 — Request Rate (RPS)**
```
Title: Request Rate (RPS)
Query: sum(rate(otel_http_requests_total[1m]))
Visualization: Time series
Unit: requests/sec
```

**Panel 2 — Error Rate (%)**
```
Title: Error Rate
Query: sum(rate(otel_http_errors_total[1m])) / sum(rate(otel_http_requests_total[1m])) * 100
Visualization: Gauge (0-100%)
Thresholds: 0=green, 1=yellow, 5=red
```

**Panel 3 — Latency by Percentile**
```
Title: Request Latency (p50/p95/p99)
Query A (p50): histogram_quantile(0.50, sum(rate(otel_http_request_duration_seconds_bucket[1m])) by (le))
Query B (p95): histogram_quantile(0.95, sum(rate(otel_http_request_duration_seconds_bucket[1m])) by (le))
Query C (p99): histogram_quantile(0.99, sum(rate(otel_http_request_duration_seconds_bucket[1m])) by (le))
Visualization: Time series
Unit: seconds
```

**Panel 4 — Orders per Minute**
```
Title: Order Throughput
Query: sum(increase(otel_orders_created_total[1m]))
Visualization: Stat
Unit: orders/min
```

3. Save the dashboard as "Order Service - RED Method Dashboard"

---

## ✅ Lab 04 Completion Checklist

- [ ] Docker Compose stack running (collector, Prometheus, Grafana)
- [ ] `metrics_service.py` running and receiving requests
- [ ] `load_generator.py` generating traffic
- [ ] Prometheus shows scraped metrics (check Status → Targets)
- [ ] Ran all 5 PromQL queries in Prometheus UI
- [ ] Created Grafana dashboard with at least 3 panels
- [ ] Can see request rate, error rate, and latency in real-time

---

## 🧠 Lab Questions

1. In Prometheus, query `otel_http_request_duration_seconds_bucket`. What do the `le` labels represent?
2. What PromQL function converts a counter into a rate?
3. What latency percentile is most useful for SLO monitoring? Why?
4. Which product family (`PROD-9xx` vs others) has higher latency in your dashboard?
5. What is the difference in what the `collector` does vs what `prometheus` does?
6. Why is `export_interval_millis=15_000` set to 15 seconds in the lab but you'd use 60 seconds in production?

---

## Cleanup

```bash
docker compose down -v
```

---

*[← Back to Module 04](../README.md) | [Module 05 →](../../Module-05-Logs/README.md)*
