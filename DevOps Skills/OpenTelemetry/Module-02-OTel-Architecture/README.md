# Module 02 — OpenTelemetry Architecture & Core Concepts

> **Professor's Note:** This is arguably the most important module in the course. Understanding OTel's architecture — how the API, SDK, Collector, and OTLP protocol fit together — prevents 80% of the confusion students encounter. Read this carefully before touching any code.

---

## 📖 Lecture 2.1 — The OpenTelemetry Big Picture

```
┌───────────────────────────────────────────────────────────────────────┐
│                     OpenTelemetry Ecosystem                           │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    Your Application                             │  │
│  │   ┌────────────┐   ┌─────────────────┐   ┌─────────────────┐   │  │
│  │   │ OTel API   │   │   OTel SDK      │   │ Auto-           │   │  │
│  │   │            │   │                 │   │ Instrumentation  │   │  │
│  │   │ • Tracer   │   │ • TracerProvider│   │ Libraries        │   │  │
│  │   │ • Meter    │   │ • MeterProvider │   │ (frameworks)    │   │  │
│  │   │ • Logger   │   │ • LoggerProvider│   │                 │   │  │
│  │   └────────────┘   └────────┬────────┘   └─────────────────┘   │  │
│  └──────────────────────────────┼──────────────────────────────────┘  │
│                                 │ OTLP (gRPC / HTTP)                  │
│                                 ▼                                      │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                  OpenTelemetry Collector                          │  │
│  │                                                                   │  │
│  │   Receivers → Processors → Exporters                             │  │
│  └────────────────────┬────────────────────────────────────────────┘  │
│                        │                                               │
│          ┌─────────────┼──────────────┐                               │
│          ▼             ▼              ▼                               │
│       Jaeger       Prometheus    Vendor Backends                     │
│       (Traces)     (Metrics)     (Cloud, SaaS)                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 📖 Lecture 2.2 — The OTel API vs SDK Distinction

This confuses many beginners. Here is the critical distinction:

### The API (Specification / Interface)

The **OTel API** defines the *interface* — the contracts and types. It contains:
- Abstract classes / interfaces
- The `trace.get_tracer()`, `metrics.get_meter()` calls
- NO real implementation — it's a list of function signatures

**Analogy:** The API is like a *USB port specification*. It defines the shape of the plug, what pins do what. Any device that implements the USB spec will work.

```python
# This is API usage — calls the interface
from opentelemetry import trace

tracer = trace.get_tracer("my-service")           # API call
with tracer.start_as_current_span("do-work"):     # API call
    pass
```

### The SDK (Implementation)

The **OTel SDK** is the *concrete implementation* of the API. It:
- Actually creates spans with real IDs
- Collects metrics into memory
- Formats data
- Sends data to exporters

```python
# This is SDK configuration — sets up the real implementation
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)   # Registers SDK as the API implementation
```

### Why Does This Distinction Matter?

1. **Library authors** use only the **API** — so their library works with any SDK
2. **Application developers** configure the **SDK** — choosing exporters, samplers, etc.
3. Users can **swap implementations** without changing application code
4. If no SDK is configured, the API uses a **No-Op implementation** (does nothing) — zero overhead

```
Library Code:            Application Code:
  uses API only            configures SDK
  
  tracer.start_span()  →   SDK creates real span
                           SDK batches spans
                           SDK exports via OTLP
```

---

## 📖 Lecture 2.3 — Signals: Traces, Metrics, Logs

OpenTelemetry handles three **signals**. Each has its own API, SDK, and exporter:

### Signal Architecture Pattern

Every signal follows the same pattern:

```
Provider → (creates) → Instrument → (records to) → Exporter
```

### Traces

```
TracerProvider → Tracer → Span
                          ├── name
                          ├── start_time / end_time
                          ├── attributes (key-value pairs)
                          ├── events (timestamped messages)
                          ├── status (OK / ERROR)
                          └── parent span reference
```

### Metrics

```
MeterProvider → Meter → Instrument
                        ├── Counter      (only goes up: requests_total)
                        ├── UpDownCounter (can go down: active_connections)
                        ├── Histogram    (distribution: latency_ms)
                        └── Gauge        (current value: memory_used)
```

### Logs

```
LoggerProvider → Logger → LogRecord
                          ├── timestamp
                          ├── severity (INFO, WARN, ERROR)
                          ├── body (message text)
                          ├── attributes
                          └── trace_id / span_id (correlation!)
```

---

## 📖 Lecture 2.4 — The Provider Pattern (Critical Concept)

OpenTelemetry uses the **Provider Pattern**. This is how you configure the entire SDK globally.

```python
# The Provider is the "factory" for instrumentation objects
from opentelemetry.sdk.trace import TracerProvider

# 1. Create and configure the provider
provider = TracerProvider(
    resource=Resource({"service.name": "my-service"}),
    sampler=TraceIdRatioBased(0.1),   # Sample 10%
)

# 2. Add processors/exporters to the provider
provider.add_span_processor(BatchSpanProcessor(
    OTLPSpanExporter(endpoint="http://otel-collector:4317")
))

# 3. Register the provider GLOBALLY (once at startup)
trace.set_tracer_provider(provider)

# --- Anywhere else in your code: ---
# 4. Get a Tracer FROM the provider (by name)
tracer = trace.get_tracer("my-component")

# 5. Create spans with the tracer
with tracer.start_as_current_span("my-operation"):
    pass
```

### Why a Global Provider?

In a running application:
- One provider → configured once at startup
- Every component asks the global API for its Tracer/Meter
- All telemetry goes through the same pipeline
- Changing where data goes = change the provider config, nothing else

---

## 📖 Lecture 2.5 — Resources: Tagging Your Telemetry

A **Resource** describes *what* is generating the telemetry. It's a set of key-value attributes that are attached to all telemetry from that SDK instance.

```python
from opentelemetry.sdk.resources import Resource

resource = Resource(attributes={
    "service.name": "order-service",         # Required: your service name
    "service.version": "2.1.0",              # Your app version
    "service.namespace": "ecommerce",        # Team/product namespace
    "deployment.environment": "production",  # dev / staging / prod
    "host.name": "pod-abc123",               # Where it's running
    "cloud.provider": "aws",                 # AWS, GCP, Azure
    "cloud.region": "ap-southeast-1",        # Cloud region
})
```

**All spans, metrics, and logs from this SDK will automatically carry these attributes.**

### OTel Semantic Conventions for Resources

OTel defines standard attribute names (semantic conventions) to ensure consistency:

| Attribute | Meaning |
|---|---|
| `service.name` | **Required** — Name of your service |
| `service.version` | Version of your service |
| `service.namespace` | Grouping for related services |
| `deployment.environment` | `production`, `staging`, `development` |
| `host.hostname` | Machine hostname |
| `process.runtime.name` | `python`, `java`, `nodejs` |
| `container.id` | Docker container ID |
| `k8s.pod.name` | Kubernetes pod name |

---

## 📖 Lecture 2.6 — OTLP: The Protocol

**OTLP** (OpenTelemetry Protocol) is the standard wire protocol for sending telemetry data.

```
Your App SDK  ────OTLP────►  OTel Collector  ────►  Backends
              (gRPC/HTTP)
```

### Two Transport Options

**OTLP/gRPC** (recommended for production):
- Port: `4317`
- Binary encoding (Protocol Buffers)
- Efficient, low-overhead
- Supports bidirectional streaming

**OTLP/HTTP** (easier for debugging/firewalls):
- Port: `4318`
- JSON or binary encoding
- Works through standard HTTP proxies
- Easier to debug with curl/browser tools

### Configuring OTLP Endpoint

```python
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Send to local Collector
exporter = OTLPSpanExporter(
    endpoint="http://localhost:4317",  # gRPC
)

# Or via environment variable (recommended):
# OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

---

## 📖 Lecture 2.7 — The OpenTelemetry Collector

The Collector is a standalone service that acts as a telemetry pipeline.

```
                    ┌────────────────────────────────┐
App 1 ──OTLP──►    │     OTEL COLLECTOR             │
App 2 ──OTLP──►    │                                │
App 3 ──OTLP──►    │  ┌──────────┐                  │
                    │  │Receivers │ ← accepts data   │
                    │  └────┬─────┘                  │
                    │       │                        │
                    │  ┌────▼─────┐                  │
                    │  │Processors│ ← transform data │
                    │  └────┬─────┘                  │
                    │       │                        │
                    │  ┌────▼─────┐                  │
                    │  │Exporters │ → send to backends│
                    │  └──────────┘                  │
                    └──────┬─────────────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
           Jaeger      Prometheus    Grafana
                                    Cloud
```

### Why Use a Collector?

**Without Collector** (each app exports directly):
- Each app needs credentials for every backend
- Changing backend = update every app and redeploy
- No centralized processing or filtering
- Apps do extra work (retry logic, buffering)

**With Collector** (apps export to Collector only):
- Apps only know about the Collector endpoint
- Add/change backends by changing Collector config only
- Central place to add attributes, filter data, sample
- The Collector handles retries, buffering, backpressure

---

## 📖 Lecture 2.8 — Context Propagation

**Context propagation** is how trace context travels from one service to another. This is what makes **distributed** tracing possible.

```
Service A                         Service B
┌────────────┐  HTTP Request      ┌────────────┐
│ Span A     │ ─────────────────► │ Span B     │
│            │ Headers:           │            │
│ TraceID=x  │ traceparent: ...   │ TraceID=x  │ ← Same trace!
│ SpanID=1   │                    │ SpanID=2   │
│            │                    │ ParentID=1 │ ← Linked to A!
└────────────┘                    └────────────┘
```

### The W3C traceparent Header

OpenTelemetry uses the W3C Trace Context standard:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │               │
             │  └── Trace ID (128-bit hex)       │               │
             │       (identifies the whole       └── Span ID     │
             │        distributed trace)          (64-bit hex)   │
             │                                                    │
             └── Version (always 00)        ─── Flags (01=sampled)
```

### Propagators

OTel provides **propagators** that inject/extract context:

```python
# Injector: Adds context to outgoing HTTP headers
headers = {}
propagator.inject(headers)
# headers now = {"traceparent": "00-abc..."}

# Extractor: Reads context from incoming HTTP headers
context = propagator.extract(incoming_headers)
# context now has the trace + span from the caller
```

Popular propagator formats:
- **W3C TraceContext** (default, recommended)
- **Baggage** (W3C, for passing user-defined values)
- **B3** (Zipkin-style, for legacy systems)
- **Jaeger** (Jaeger-style, for legacy systems)

---

## 📖 Lecture 2.9 — Span Lifecycle & Key Concepts

```python
tracer = trace.get_tracer("my-tracer")

# Start a span — this is the unit of work
with tracer.start_as_current_span("process-order") as span:
    
    # Attributes: Key-value metadata about the operation
    span.set_attribute("order.id", "ORD-12345")
    span.set_attribute("order.total", 49.99)
    span.set_attribute("user.id", "user-789")
    
    # Events: Point-in-time occurrences within the span
    span.add_event("payment.initiated", {"payment_method": "credit_card"})
    
    try:
        result = do_work()
        
        span.add_event("payment.completed")
        span.set_status(Status(StatusCode.OK))
        
    except Exception as e:
        # Record exception details
        span.record_exception(e)
        span.set_status(Status(StatusCode.ERROR, str(e)))
        raise

# Span ends when the 'with' block exits
```

### Span Kinds

| Kind | Use When |
|---|---|
| `INTERNAL` | Default. Operations within the same service |
| `SERVER` | Span is handling an incoming RPC/HTTP request |
| `CLIENT` | Span is making an outgoing RPC/HTTP call |
| `PRODUCER` | Span is publishing a message to a queue |
| `CONSUMER` | Span is consuming a message from a queue |

---

## 📖 Lecture 2.10 — Complete Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenTelemetry in Your App                        │
│                                                                     │
│  CONFIGURE ONCE (at startup):                                       │
│  Resource → Who am I? (service.name, version, environment)         │
│  TracerProvider → How do I handle traces?                           │
│  MeterProvider  → How do I handle metrics?                          │
│  LoggerProvider → How do I handle logs?                             │
│  Propagator     → How do I pass context between services?           │
│                                                                     │
│  USE ANYWHERE (in your code):                                       │
│  tracer = trace.get_tracer("component-name")                        │
│  meter  = metrics.get_meter("component-name")                       │
│  logger = logging.getLogger("component-name")   ← standard logging │
│                                                                     │
│  INSTRUMENT:                                                        │
│  with tracer.start_as_current_span("operation") as span:           │
│      span.set_attribute("key", "value")                             │
│      counter.add(1, {"label": "value"})                             │
│      logger.info("Something happened")                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## ✅ Module 02 — Review Questions

1. What is the difference between the **OTel API** and the **OTel SDK**? Why does this matter for library authors?
2. What are the three **signals** in OpenTelemetry?
3. What is a **Provider** and why is it configured globally?
4. What is a **Resource** and what attributes should every service set?
5. What is **OTLP**? What are the two transport protocols and their ports?
6. What does the **OpenTelemetry Collector** do? Name 3 benefits of using it.
7. What is **context propagation** and which HTTP header does it use?
8. Decode this traceparent: `00-abc123def456abc123def456abc12345-00f067aa0ba902b7-01`
9. Name the 5 **Span Kinds** and when you'd use each.
10. What happens in your application if you use the OTel API but never configure an SDK?

---

## 🧪 Lab 02 — Your First Real Trace with SDK

**→ See [labs/lab-02-first-trace.md](./labs/lab-02-first-trace.md)**

---

*[← Module 01](../Module-01-Observability-Foundations/README.md) | [Module 03 →](../Module-03-Traces/README.md)*
