# Lab 09 — Configuring Tail-Based Sampling

> **Objective:** Configure an OpenTelemetry Collector to use Tail-Based Sampling. You'll generate a mix of successful and failed traces. The Collector will drop the vast majority of successful traces but will keep **100% of the failed ones**.

---

## ⏱️ Estimated Time: 30 minutes

---

## Part 1 — The Collector Configuration

Create `Module-09-Production/labs/config.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  # Wait for full traces before evaluating
  tail_sampling:
    decision_wait: 5s 
    num_traces: 10000 
    expected_new_traces_per_sec: 100
    policies:
      # 1. ALWAYSKEEP errors
      - name: keep_errors
        type: status_code
        status_code: {status_codes: [ERROR]}
        
      # 2. DROP 90% of successful traces (keep exactly 10%)
      - name: sample_baseline
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

  batch:
    timeout: 1s

exporters:
  logging:
    verbosity: detailed
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      # IMPORTANT ORDER: sampling decision MUST happen before batching!
      processors: [tail_sampling, batch]
      exporters: [logging, otlp/jaeger]
```

---

## Part 2 — Start the Stack

Create `Module-09-Production/labs/docker-compose.yml`:

```yaml
version: "3.8"

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel/config.yaml"]
    volumes:
      - ./config.yaml:/etc/otel/config.yaml
    ports:
      - "4317:4317"
    depends_on:
      - jaeger

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

```bash
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry\Module-09-Production\labs"
docker compose up -d
```

---

## Part 3 — The Traffic Generator

Create `Module-09-Production/labs/generator.py`:

```python
"""
generator.py — Sends 100 traces. 
95 will be successful. 5 will fail.
With 10% sampling on successes, Jaeger should show ~14 traces total 
(5 failures + ~9 sampled successes).
"""

import time
import random
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.trace import Status, StatusCode

provider = TracerProvider(resource=Resource({"service.name": "sampling-demo"}))
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("demo")

SUCCESS_COUNT = 95
ERROR_COUNT = 5

print(f"Generating {SUCCESS_COUNT + ERROR_COUNT} traces...")

# Generate successes
for i in range(SUCCESS_COUNT):
    with tracer.start_as_current_span(f"successful-operation-{i}") as span:
        span.set_status(Status(StatusCode.OK))
        time.sleep(0.01)

# Generate errors
for i in range(ERROR_COUNT):
    with tracer.start_as_current_span(f"failed-operation-{i}") as span:
        # This is what the tail sampler is looking for!
        span.set_status(Status(StatusCode.ERROR, "Simulated failure"))
        time.sleep(0.01)

provider.force_flush()
print("Done! Check Jaeger in about 5 seconds (give the tail sampler time to decide).")
```

Run it:
```bash
python generator.py
```

---

## Part 4 — Verify the Magic

1. Open **http://localhost:16686**.
2. Select service `sampling-demo` and Find Traces.
3. Look at the count at the top right of the search results.
   - You sent 100 traces...
   - You should only see roughly **10-15** traces!
4. **Crucial Verification:** Look for the error traces. You should see **exactly 5 red traces**. The tail sampler kept 100% of the failures!

This is how you save 90% of your observability bill while keeping 100% of your critical debugging data.

---

## Cleanup

```bash
docker compose down
```

---

*[← Back to Module 09](../README.md) | [Capstone Project →](../../Capstone-Project/README.md)*
