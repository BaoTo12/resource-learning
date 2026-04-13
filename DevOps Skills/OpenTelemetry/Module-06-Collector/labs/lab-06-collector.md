# Lab 06 — Building a Collector Pipeline

> **Objective:** Deploy the OpenTelemetry Collector using Docker Compose. Configure it to receive OTLP data, apply a processor to mask sensitive PII data, batch the telemetry, and export it to Jaeger. We'll use a Python script to send test data.

---

## ⏱️ Estimated Time: 30 minutes

---

## Part 1 — Create the Environment

Create the `Module-06-Collector/labs` directory and set up these files.

### 1. The Collector Configuration

Create `Module-06-Collector/labs/config/otel-collector-config.yaml`:

```yaml
# 1. RECEIVERS
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

# 2. PROCESSORS
processors:
  # Memory limiter - mandatory for production
  memory_limiter:
    check_interval: 1s
    limit_mib: 100
    spike_limit_mib: 20

  # Batcher - improves compression and reduces backend load
  batch:
    send_batch_size: 512
    timeout: 5s

  # Attributes Processor - Data Masking / Transformation
  attributes/masking:
    actions:
      # Action 1: Insert environment tag to all spans
      - key: environment
        value: lab-06-production
        action: insert
        
      # Action 2: HASH sensitive user emails
      - key: user.email
        action: hash
        
      # Action 3: DELETE credit card strings if someone accidentally logged them
      - key: payment.ccn
        action: delete

# 3. EXPORTERS
exporters:
  # Export to Jaeger
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
      
  # Logging exporter - great for debugging the collector itself
  logging:
    verbosity: detailed

# 4. PIPELINE DEFINITION
service:
  pipelines:
    traces:
      # Data flows from left to right through this array
      receivers: [otlp]
      processors: [memory_limiter, attributes/masking, batch]
      exporters: [logging, otlp/jaeger]
      
  # Collector exposes its own health and metrics
  telemetry:
    metrics:
      address: 0.0.0.0:8888
```

### 2. The Docker Compose Stack

Create `Module-06-Collector/labs/docker-compose.yml`:

```yaml
version: "3.8"

services:
  # 1. The OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: lab06-otel-collector
    command: ["--config=/etc/otel/config.yaml"]
    volumes:
      - ./config/otel-collector-config.yaml:/etc/otel/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC (Apps send here!)
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Collector's own metrics
    depends_on:
      - jaeger

  # 2. Jaeger (Backend)
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: lab06-jaeger
    ports:
      - "16686:16686" # Web UI
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

---

## Part 2 — Start the Infrastructure

```bash
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry\Module-06-Collector\labs"

# Start the stack
docker compose up -d

# Verify both are running
docker compose ps
```

---

## Part 3 — Send Test Data

Rather than building a full web app, we'll write a small script that generates traces to test our Collector pipeline.

Create `Module-06-Collector/labs/send_test_data.py`:

```python
"""
send_test_data.py — Sends manual traces to the Collector to test pipelines.
"""

import time
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Configure SDK to send to the COLLECTOR
resource = Resource({"service.name": "test-script"})
provider = TracerProvider(resource=resource)

# App sends to Collector on 4317
exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("test-tracer")

print("Sending 5 test traces with sensitive data...")

for i in range(5):
    with tracer.start_as_current_span(f"test-operation-{i}") as span:
        
        # Attributes we expect the collector to modify/hash/mask!
        span.set_attribute("user.email", f"user{i}@example.com")  # Should be hashed!
        span.set_attribute("payment.ccn", "4111-2222-3333-4444")  # Should be deleted!
        span.set_attribute("safe.data", "This is fine")           # Should remain
        
        print(f"  Sent trace {i+1}/5")
        time.sleep(0.5)

# Force flush
provider.force_flush()
print("Done! Check Jaeger.")
```

Run it:

```bash
# Ensure you are in your venv
pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp-proto-grpc
python send_test_data.py
```

---

## Part 4 — Verify the Pipeline Worked

1. Open Jaeger: **http://localhost:16686**
2. Search for the `test-script` service.
3. Open one of the traces.
4. Check the span attributes!

**Did the Collector processors do their job?**
- Do you see `environment: lab-06-production`? (Insert action)
- Is `user.email` a long unreadable hash string? (Hash action)
- Is `payment.ccn` completely missing? (Delete action)
- Is `safe.data` untouched?

If yes, congratulations! Your Collector successfully intercepted, processed, and modified telemetry in flight before Jaeger ever saw it.

---

## ✅ Lab 06 Completion Checklist

- [ ] Docker Compose stack with Collector and Jaeger running
- [ ] Ran `send_test_data.py`
- [ ] Confirmed in Jaeger that `user.email` was hashed
- [ ] Confirmed in Jaeger that `payment.ccn` was deleted
- [ ] Confirmed in Jaeger that `environment=lab-06-production` was added

---

## Cleanup

```bash
docker compose down
```

---

*[← Back to Module 06](../README.md) | [Module 07 →](../../Module-07-Auto-Instrumentation/README.md)*
