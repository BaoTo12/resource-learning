# Lab 08 — Building the Full Local Stack

> **Objective:** We will run Jaeger, Prometheus, and Grafana simultaneously, using the OTel collector to route data. You'll run the metrics service from Lab 04 and manually add traces to it, seeing the full power of correlated telemetry in Grafana.

---

## ⏱️ Estimated Time: 30 minutes

---

## Part 1 — The Unified Docker Stack

Create `Module-08-Backends/labs/docker-compose-full.yml`:

```yaml
version: "3.8"

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel/config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel/config.yaml
    ports:
      - "4317:4317" # OTLP gRPC
      - "8889:8889" # Prometheus metrics endpoint

  jaeger:
    image: jaegertracing/all-in-one:latest
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "16686:16686" # Web UI
      - "4318:4318"   # Receives traces directly if needed

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
```

---

## Part 2 — The Unified Collector Config

Create `Module-08-Backends/labs/otel-collector-config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 1s

exporters:
  # TRACES go to Jaeger
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
      
  # METRICS go to Prometheus (via pull endpoint on port 8889)
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

---

## Part 3 — Configure Prometheus

Create `Module-08-Backends/labs/prometheus.yml`:

```yaml
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]
```

---

## Part 4 — Run Everything!

```bash
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry\Module-08-Backends\labs"

# Start the unified observability stack!
docker compose -f docker-compose-full.yml up -d
```

### Services Available:
- **Your App OTLP Endpoint:** `localhost:4317`
- **Jaeger UI (Traces):** `http://localhost:16686`
- **Prometheus UI (Metrics):** `http://localhost:9090`
- **Grafana UI (Dashboards):** `http://localhost:3000`

---

## Part 5 — The Unified Application

Create a quick script that emits BOTH traces and metrics to test the stack.

Create `Module-08-Backends/labs/unified_app.py`:

```python
"""
unified_app.py — Emits BOTH traces and metrics to the OTel Collector.
"""

import time
import random
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

# Shared resource
resource = Resource({"service.name": "unified-test-service"})

# Setup Traces
trace_provider = TracerProvider(resource=resource)
trace_exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
trace_provider.add_span_processor(BatchSpanProcessor(trace_exporter))
trace.set_tracer_provider(trace_provider)
tracer = trace.get_tracer("test")

# Setup Metrics
metric_exporter = OTLPMetricExporter(endpoint="http://localhost:4317", insecure=True)
reader = PeriodicExportingMetricReader(metric_exporter, export_interval_millis=5000)
meter_provider = MeterProvider(resource=resource, metric_readers=[reader])
metrics.set_meter_provider(meter_provider)
meter = metrics.get_meter("test")

# Metric instrument
request_counter = meter.create_counter("test_requests_total")

print("Sending unified telemetry... Press Ctrl+C to stop")
try:
    while True:
        with tracer.start_as_current_span("do-work") as span:
            time.sleep(random.uniform(0.01, 0.05))
            status = 200 если random.random() > 0.1 else 500
            
            span.set_attribute("http.status_code", status)
            request_counter.add(1, {"status": str(status)})
            
            print(f"Sent trace & metric point (status: {status})")
        time.sleep(1)
except KeyboardInterrupt:
    print("Done!")
```

Run it:
```bash
python unified_app.py
```

---

## Part 6 — Explore Grafana

Let's link Prometheus and Jaeger inside Grafana.

1. Open **http://localhost:3000**
2. **Add Prometheus Data Source:**
   - Configuration → Data Sources → Add
   - Select Prometheus
   - URL: `http://prometheus:9090` 
   - Save & Test
3. **Add Jaeger Data Source:**
   - Add another data source
   - Select Jaeger
   - URL: `http://jaeger:16686`
   - Save & Test

Now you have a single UI (Grafana) that can query metrics from Prometheus and traces from Jaeger!

---

## ✅ Lab 08 Completion Checklist

- [ ] Started the full docker stack
- [ ] Ran `unified_app.py`
- [ ] Found the traces in Jaeger UI
- [ ] Queried `test_requests_total` in Prometheus UI
- [ ] Logged into Grafana and successfully connected both data sources

---

## Cleanup

```bash
docker compose -f docker-compose-full.yml down
```

---

*[← Back to Module 08](../README.md) | [Module 09 →](../../Module-09-Production/README.md)*
