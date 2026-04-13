# Module 03 — Distributed Tracing: Spans, Context & Propagation

> **Professor's Note:** Distributed tracing is the most powerful observability tool for microservices. In this module, we go deep on spans — their anatomy, attributes, semantic conventions, and most importantly, how trace context crosses service boundaries. When you finish this module, you'll be able to trace a request through 10 microservices and find the bottleneck in seconds.

---

## 📖 Lecture 3.1 — Anatomy of a Span

A **span** is the fundamental unit of a trace. Think of it as a timed operation with rich metadata attached.

```
┌─────────────────────────────────────────────────────────┐
│                         SPAN                            │
│                                                         │
│  Identity:                                              │
│    trace_id:  4bf92f3577b34da6a3ce929d0e0e4736         │
│    span_id:   00f067aa0ba902b7                          │
│    parent_id: ab1234cd5678ef90  (null if root)          │
│                                                         │
│  Timing:                                                │
│    start_time: 2026-04-12T08:15:22.000Z                 │
│    end_time:   2026-04-12T08:15:22.250Z                 │
│    duration:   250ms                                    │
│                                                         │
│  Metadata:                                              │
│    name:      "process-order"                           │
│    kind:      SERVER                                    │
│    status:    OK | ERROR                                │
│                                                         │
│  Attributes: (key-value pairs)                          │
│    "http.method": "POST"                                │
│    "http.url": "/api/orders"                            │
│    "order.id": "ORD-12345"                              │
│    "user.id": "user-789"                                │
│                                                         │
│  Events: (timestamped messages)                         │
│    08:15:22.010 - "cache.hit"                           │
│    08:15:22.100 - "payment.initiated"                   │
│    08:15:22.240 - "order.completed"                     │
│                                                         │
│  Links: (references to other spans/traces)              │
│    → async-job-span-id: abc456 (trace: def789)          │
└─────────────────────────────────────────────────────────┘
```

---

## 📖 Lecture 3.2 — Trace Structure: The Span Tree

A trace is a **tree** of spans. Each span has one parent (except the root).

```
Trace: abc123  (entire checkout request)
│
└── process-order  [0ms → 350ms]  ROOT SPAN
    ├── validate-input  [5ms → 20ms]
    │   └── check-fraud-score  [8ms → 18ms]  (calls fraud service)
    │
    ├── fetch-user  [20ms → 45ms]  (calls user service)
    │
    ├── check-inventory  [45ms → 180ms]  ← 135ms! BOTTLENECK
    │   ├── redis-get  [45ms → 55ms]  CACHE MISS
    │   └── postgres-query  [55ms → 178ms]  ← 123ms DB query!
    │
    ├── calculate-price  [180ms → 195ms]
    │
    └── process-payment  [195ms → 348ms]  (calls payment service)
        ├── stripe-api-call  [200ms → 340ms]  (external HTTP)
        └── save-transaction  [340ms → 347ms]
```

**Without this trace:** "The checkout is slow sometimes"  
**With this trace:** "The redis cache is missing for this product type, causing a 123ms PostgreSQL full-table-scan"

---

## 📖 Lecture 3.3 — Semantic Conventions: Standard Attribute Names

OpenTelemetry defines **Semantic Conventions** — standardized attribute names for common operations. Using them ensures your traces work correctly with backends and are consistent across services (even different languages).

**Never invent your own attribute names for standard things.**

### HTTP Server Spans

```python
# When your service RECEIVES an HTTP request (kind=SERVER)
span.set_attribute("http.method", "POST")          # GET, POST, PUT, DELETE
span.set_attribute("http.route", "/api/orders/{id}") # URL template, not specific URL
span.set_attribute("http.url", "https://api.example.com/api/orders/123")
span.set_attribute("http.status_code", 200)
span.set_attribute("http.request_content_length", 512)
span.set_attribute("server.address", "api.example.com")
span.set_attribute("server.port", 443)
```

### HTTP Client Spans

```python
# When your service MAKES an HTTP request (kind=CLIENT)
span.set_attribute("http.method", "GET")
span.set_attribute("http.url", "https://inventory-service/stock/PROD-001")
span.set_attribute("http.status_code", 200)
span.set_attribute("server.address", "inventory-service")
span.set_attribute("server.port", 8080)
```

### Database Spans

```python
# When making a database call (kind=CLIENT)
span.set_attribute("db.system", "postgresql")      # postgresql, mysql, redis, mongodb
span.set_attribute("db.name", "orders")            # Database name
span.set_attribute("db.operation", "SELECT")       # SELECT, INSERT, UPDATE, DELETE
span.set_attribute("db.statement",                 # The actual query (sanitize params!)
    "SELECT * FROM orders WHERE user_id = ?")
span.set_attribute("db.sql.table", "orders")
span.set_attribute("server.address", "db.internal")
span.set_attribute("server.port", 5432)
```

### Messaging Spans (Queues)

```python
# Publishing to a queue (kind=PRODUCER)
span.set_attribute("messaging.system", "kafka")    # kafka, rabbitmq, sqs
span.set_attribute("messaging.destination", "orders.created")  # Topic/queue name
span.set_attribute("messaging.operation", "publish")
span.set_attribute("messaging.message.id", "msg-12345")

# Consuming from a queue (kind=CONSUMER)
span.set_attribute("messaging.system", "kafka")
span.set_attribute("messaging.destination", "orders.created")
span.set_attribute("messaging.operation", "process")
span.set_attribute("messaging.kafka.consumer.group", "order-processor")
```

### Exception Recording

```python
try:
    risky_operation()
except Exception as e:
    # This records the exception type, message, and stack trace as span attributes
    span.record_exception(e)
    span.set_status(Status(StatusCode.ERROR, str(e)))
    raise  # Re-raise unless you're handling it
```

---

## 📖 Lecture 3.4 — Context Propagation In Depth

This is the magic that makes distributed tracing work. Without propagation, each service would create isolated, disconnected traces.

### What Is Propagation?

When Service A calls Service B:
1. Service A **injects** the current trace context into the HTTP headers
2. Service B **extracts** the trace context from the incoming headers
3. Service B creates a **child span** using the extracted context as its parent

```
Service A                              Service B
┌─────────────────┐    HTTP Request    ┌─────────────────┐
│ Trace: abc123   │ ─────────────────► │                 │
│ Span: parent-1  │                    │ Extracts:       │
│                 │ Headers:           │   TraceID=abc123│
│ inject(headers) │ traceparent:       │   ParentID=1    │
│                 │  00-abc123-1-01    │                 │
│                 │                    │ Creates child:  │
│                 │                    │ TraceID=abc123  │
│                 │                    │ ParentID=parent-1│
└─────────────────┘                    └─────────────────┘
```

### Manual Propagation Example

```python
import requests
from opentelemetry import trace, propagate
from opentelemetry.trace import SpanKind

tracer = trace.get_tracer("order-service")

def call_inventory_service(product_id: str) -> dict:
    """Call inventory service with trace context propagation"""
    
    with tracer.start_as_current_span(
        "inventory.check",
        kind=SpanKind.CLIENT  # ← Client span: we're making an outgoing call
    ) as span:
        
        # Set HTTP client attributes
        span.set_attribute("http.method", "GET")
        span.set_attribute("http.url", f"http://inventory-service/stock/{product_id}")
        span.set_attribute("server.address", "inventory-service")
        
        # ⚡ INJECT TRACE CONTEXT INTO HEADERS
        headers = {}
        propagate.inject(headers)
        # headers is now: {"traceparent": "00-abc123...", "tracestate": ""}
        
        # Make the actual HTTP call with propagated headers
        response = requests.get(
            f"http://inventory-service/stock/{product_id}",
            headers=headers,    # ← Context travels with the request!
            timeout=5
        )
        
        span.set_attribute("http.status_code", response.status_code)
        return response.json()
```

```python
# In inventory-service (the receiving end):
from flask import Flask, request
from opentelemetry import trace, propagate
from opentelemetry.trace import SpanKind

app = Flask(__name__)
tracer = trace.get_tracer("inventory-service")

@app.route("/stock/<product_id>")
def check_stock(product_id):
    
    # ⚡ EXTRACT TRACE CONTEXT FROM INCOMING HEADERS
    context = propagate.extract(request.headers)
    
    with tracer.start_as_current_span(
        "inventory.check_stock",
        context=context,          # ← Child of the calling service's span!
        kind=SpanKind.SERVER,     # ← Server span: we're receiving a call
    ) as span:
        
        span.set_attribute("http.method", request.method)
        span.set_attribute("http.route", "/stock/{product_id}")
        span.set_attribute("inventory.product_id", product_id)
        
        # ... do the actual work ...
        
        return {"product_id": product_id, "available": 50}
```

---

## 📖 Lecture 3.5 — Baggage: Passing Data Across Services

**Baggage** is a key-value store that propagates alongside trace context. Unlike span attributes (which only exist on the span), baggage travels with every downstream call.

```python
from opentelemetry.baggage import set_baggage, get_baggage
from opentelemetry import context

# In the API Gateway: Set baggage on the incoming request
ctx = set_baggage("user.tier", "premium")
ctx = set_baggage("user.region", "APAC", context=ctx)

# This baggage now travels through ALL downstream service calls automatically

# In any downstream service: Read baggage
user_tier = get_baggage("user.tier")     # Returns "premium"
user_region = get_baggage("user.region") # Returns "APAC"
```

**Warning:** Baggage is propagated in HTTP headers — every key-value you add **adds overhead to every HTTP call**. Keep it small — 3-5 items maximum.

**Good baggage uses:**
- User tier (for routing to different processing logic)
- A/B test assignment
- Feature flags
- Request source (mobile, web, batch)

**Bad baggage uses:**  
- Request body data
- Anything that changes during the request
- Sensitive PII

---

## 📖 Lecture 3.6 — Span Links

Sometimes operations are related but not in a parent-child relationship. **Span Links** model non-hierarchical causality.

**Example: Fan-out / Fan-in pattern**

```
Request comes in
→ Creates 10 parallel tasks (each is a separate trace)
→ Waits for all tasks
→ Aggregates results

The aggregator span LINKS to all 10 task traces
```

```python
from opentelemetry.trace import Link

def aggregate_results(task_span_contexts: list):
    """Aggregates results from multiple parallel tasks"""
    
    # Create links to all the parallel task spans
    links = [Link(context=ctx) for ctx in task_span_contexts]
    
    with tracer.start_as_current_span("aggregate-results", links=links) as span:
        span.set_attribute("aggregation.task_count", len(task_span_contexts))
        # ... do aggregation work ...
```

**Also used for:**
- Async processing: The HTTP response span links to the background job span
- Event-driven: The event consumer links to the event producer
- Batch processing: Each batch item trace links to the batch trigger trace

---

## 📖 Lecture 3.7 — The Context Object & Current Span

OpenTelemetry uses an implicit **Context** to track the current span. This is called **Context Propagation** within a single process.

```python
# When you create a span with 'with' statement:
with tracer.start_as_current_span("parent") as parent_span:
    
    # ← parent_span is the CURRENT span here
    # Any child span created inside will automatically have parent as their parent
    
    with tracer.start_as_current_span("child") as child_span:
        
        # ← child_span is the CURRENT span here
        # child_span.context.parent_id == parent_span.context.span_id  ✓
        
        # You can always get the current span:
        current = trace.get_current_span()
        assert current is child_span  # True!
```

### Async Context (Important!)

In async code, context must be explicitly managed:

```python
import asyncio
from opentelemetry import trace, context

tracer = trace.get_tracer("async-service")

async def async_child_task(ctx):
    """Async task that needs parent context"""
    
    # Attach the parent context in this coroutine
    token = context.attach(ctx)
    try:
        with tracer.start_as_current_span("async-child"):
            await asyncio.sleep(0.1)
            # This span correctly inherits the parent context
    finally:
        context.detach(token)


async def main():
    with tracer.start_as_current_span("async-parent") as span:
        # Capture current context to pass to coroutines
        current_ctx = context.get_current()
        
        # Run tasks with the parent context
        await asyncio.gather(
            async_child_task(current_ctx),
            async_child_task(current_ctx),
        )
```

---

## 📖 Lecture 3.8 — Interpreting Trace Waterfalls

In Jaeger or Zipkin, traces are shown as **waterfall diagrams**:

```
Trace: checkout (450ms total)
│
├─ [API Gateway] receive-request ████████████████████████████████████ 450ms
│
├─── [API Gateway] authenticate  ████                                  35ms
│
├─── [Order Svc] process-order   ████████████████████████████          380ms
│    │
│    ├── validate-input          ███                                    25ms
│    │
│    ├── [Inventory Svc] check   ████████████                          120ms
│    │   └── db-query            ████████████                          118ms ← SLOW
│    │
│    ├── [Price Svc] calculate   ████                                   35ms
│    │
│    └── [Payment Svc] charge    ████████████                          145ms
│        └── stripe-api          ████████████                          140ms
│
```

**How to read a waterfall:**
- Horizontal position = when it started (relative to trace start)
- Width = duration
- Nested indentation = parent-child relationship
- Color = status (green=OK, red=error)
- The widest bar IS the bottleneck

---

## ✅ Module 03 — Review Questions

1. List and explain the 5 fields that every span must have.
2. What is the difference between span **attributes** and span **events**?
3. Why should you use semantic conventions instead of custom attribute names?
4. What HTTP header carries trace context between services?
5. Decode this traceparent: `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`
6. In context propagation, what does **inject** do? What does **extract** do?
7. What is **Baggage** and how is it different from span attributes?
8. When should you use **Span Links** instead of parent-child relationships?
9. In async Python code, why must you explicitly pass context to coroutines?
10. Looking at a trace waterfall, how do you identify the bottleneck?

---

## 🧪 Lab 03 — Manual Tracing: Multi-Service Distributed Trace

**→ See [labs/lab-03-manual-tracing.md](./labs/lab-03-manual-tracing.md)**

---

*[← Module 02](../Module-02-OTel-Architecture/README.md) | [Module 04 →](../Module-04-Metrics/README.md)*
