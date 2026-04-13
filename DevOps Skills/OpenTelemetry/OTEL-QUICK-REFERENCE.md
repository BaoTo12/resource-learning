# 🚀 OPENTELEMETRY QUICK REFERENCE

Keep this sheet handy while instrumenting your applications.

## 🔭 Core API vs SDK

### The API (Used to code instrumentation)
```python
from opentelemetry import trace, metrics

tracer = trace.get_tracer("my.service.name")
meter = metrics.get_meter("my.service.name")
```

### The SDK (Used once at startup to configure pipelines)
```python
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider

resource = Resource({"service.name": "my-service", "deployment.environment": "prod"})
provider = TracerProvider(resource=resource)
# ... add processors/exporters ...
trace.set_tracer_provider(provider)
```

## ⏱️ Tracing Cheat Sheet

### Manual Span Creation
```python
# 1. Basic Span
with tracer.start_as_current_span("do_work") as span:
    span.set_attribute("item.id", 123)
    span.add_event("work.started")
    # ... do work ...
    span.set_status(Status(StatusCode.OK))

# 2. Span with Error
try:
    risky_operation()
except Exception as e:
    span.record_exception(e)
    span.set_status(Status(StatusCode.ERROR, str(e)))
    raise
```

### Context Propagation (HTTP)
```python
# INJECT (Client sending request)
from opentelemetry import propagate
headers = {}
propagate.inject(headers)
requests.get("http://api/", headers=headers)

# EXTRACT (Server receiving request)
context = propagate.extract(request.headers)
with tracer.start_as_current_span("route", context=context) as span:
    pass
```

## 📊 Metrics Cheat Sheet

```python
# 1. Counter (Add only, goes up)
req_counter = meter.create_counter("requests_total", description="Total requests")
req_counter.add(1, {"http.status": 200, "http.method": "GET"})

# 2. UpDownCounter (Goes up or down)
queue_depth = meter.create_up_down_counter("queue_depth")
queue_depth.add(1)   # Item enqueued
queue_depth.add(-1)  # Item dequeued

# 3. Histogram (Distribution/Percentiles)
latency_hist = meter.create_histogram("request_duration_seconds", unit="s")
latency_hist.record(0.145, {"http.route": "/users"})

# 4. Observable Gauge (Polled value)
def get_memory(options):
    return [metrics.Observation(1024, {"type": "heap"})]
meter.create_observable_gauge("memory_usage_bytes", callbacks=[get_memory])
```

## 🧩 Auto-Instrumentation CLI

```bash
# Install core + distro
pip install opentelemetry-api opentelemetry-sdk opentelemetry-distro

# Install framework-specific instrumentation automatically
opentelemetry-bootstrap -a install

# Run app with auto-instrumentation
export OTEL_SERVICE_NAME="my-api"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
opentelemetry-instrument python my_app.py
```

## 📦 Standard Semantic Conventions

Always use these standard attribute names instead of inventing your own:
- **Service Identity:** `service.name`, `service.version`, `service.instance.id`
- **Environment:** `deployment.environment`
- **HTTP/Web:** `http.method`, `http.status_code`, `http.url`, `http.route`
- **Database:** `db.system`, `db.name`, `db.statement`, `db.operation`
- **Networking:** `server.address`, `server.port`, `client.address`

## ⚙️ Standard Environment Variables

| Variable | Example Value | Description |
|---|---|---|
| `OTEL_SERVICE_NAME` | `order-service` | Defines the service name globally |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4317` | Where to route all OTLP data |
| `OTEL_TRACES_EXPORTER` | `otlp`, `console`, `none` | Which traces exporter to use |
| `OTEL_METRICS_EXPORTER` | `otlp`, `console`, `none` | Which metrics exporter to use |
| `OTEL_RESOURCE_ATTRIBUTES` | `env=prod,team=ops` | Global tags attached to all data |
| `OTEL_LOG_LEVEL` | `debug` | Verbosity of the internal SDK logger |
