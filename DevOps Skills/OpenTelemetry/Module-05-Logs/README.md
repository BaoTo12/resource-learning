# Module 05 — Structured Logging & Log Correlation

> **Professor's Note:** Logs are the oldest observability tool — but raw logs are nearly useless at scale. In this module you learn to write *structured* logs, correlate them to traces using Trace IDs, and ship them via OTel. The key insight: a log without a Trace ID is a dead end. A log *with* a Trace ID is a direct link to the full request journey.

---

## 📖 Lecture 5.1 — Unstructured vs. Structured Logs

### Unstructured (the old way)

```
2026-04-12 08:15:22 ERROR Failed to process order for user user-789, product PROD-901 qty 3: timeout after 5000ms
2026-04-12 08:15:23 INFO  Order ORD-12345 created successfully for user user-1
2026-04-12 08:15:23 WARN  Inventory low: PROD-001 only 5 remaining
```

**Problems:**
- You can only search by text (grep)
- You can't filter by `user_id=user-789` efficiently
- Parsing is brittle (regex breaks on format change)
- No connection to traces at all

### Structured (the right way)

```json
{"timestamp":"2026-04-12T08:15:22Z","level":"ERROR","service":"order-service","message":"Order processing failed","user_id":"user-789","product_id":"PROD-901","quantity":3,"error":"timeout","timeout_ms":5000,"trace_id":"4bf92f357...","span_id":"00f067aa..."}
{"timestamp":"2026-04-12T08:15:23Z","level":"INFO","service":"order-service","message":"Order created","order_id":"ORD-12345","user_id":"user-1","trace_id":"abc123def...","span_id":"xyz456..."}
{"timestamp":"2026-04-12T08:15:23Z","level":"WARN","service":"order-service","message":"Inventory low","product_id":"PROD-001","remaining":5,"trace_id":"def789...","span_id":"uvw012..."}
```

**Benefits:**
- Machine-readable — filter by any field instantly
- `trace_id` links directly to the trace in Jaeger
- `span_id` links to the exact span within the trace

---

## 📖 Lecture 5.2 — The Trace ID Correlation: The Golden Thread

The most powerful feature of OTel logs is **trace correlation**. Every log emitted while a span is active automatically gets the current `trace_id` and `span_id` injected.

```python
# This flow:
with tracer.start_as_current_span("process-order") as span:
    logger.info("Processing order", extra={"order_id": "ORD-123"})
    #             ↑ automatically gets trace_id and span_id from the active span!

# The log becomes:
{
  "message": "Processing order",
  "order_id": "ORD-123",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",  ← from active span!
  "span_id": "00f067aa0ba902b7",                    ← from active span!
}
```

**In practice:**
- User reports: "My order ORD-123 is broken"
- You search logs for `order_id=ORD-123`
- Find the trace_id in that log entry
- Open Jaeger with that trace_id
- See the full request journey across all services
- Find the exact failed span

---

## 📖 Lecture 5.3 — OTel Logging Architecture

OTel logging is the newest signal (added after traces and metrics). There are two approaches:

### Approach A: Bridge from standard logging library (Recommended)

Your app uses Python's standard `logging` module. A bridge injects trace context automatically.

```
Python logging.Logger.info("message")
         ↓
  OTel Log Bridge     ← intercepts and adds trace_id/span_id
         ↓
  OTel LoggerProvider
         ↓
  OTLP Exporter      → Collector → Backend
```

### Approach B: OTel Logger directly

Uses OTel's own Logger API (less common, more explicit).

For production use, **Approach A** is standard — it's non-invasive and works with all existing logging code.

---

## 📖 Lecture 5.4 — Log Severity Levels

OTel maps standard severity levels (use these correctly!):

| Python Level | OTel Severity | When to Use |
|---|---|---|
| `DEBUG` | TRACE / DEBUG | Detailed internal state, only for debugging |
| `INFO` | INFO | Normal business events ("Order created") |
| `WARNING` | WARN | Something unexpected but handled ("Retry #2") |
| `ERROR` | ERROR | Operation failed but app is still running |
| `CRITICAL` | FATAL | App cannot continue, about to crash |

---

## 📖 Lecture 5.5 — What to Log (and What NOT to)

### ✅ Always Log

```python
# Business events with IDs
logger.info("Order created", extra={"order_id": "ORD-123", "user_id": "user-789", "total": 49.99})

# Errors with context
logger.error("Payment failed", extra={"order_id": "ORD-123", "gateway": "stripe", "error_code": "card_declined"})

# State transitions
logger.info("Order status changed", extra={"order_id": "ORD-123", "from": "pending", "to": "confirmed"})

# External service calls (success and failure)
logger.info("Inventory checked", extra={"product_id": "PROD-001", "available": 50, "duration_ms": 23})
```

### ❌ Never Log

```python
# PII / sensitive data
logger.info("User logged in", extra={"email": "user@example.com", "password": "hunter2"})  # ← NEVER!

# High-frequency low-value events (creates log spam)
for item in items:
    logger.debug(f"Processing item {item}")  # ← at 10k/s this is useless noise

# Raw request/response bodies (usually too large, often contains PII)
logger.debug("Response body: " + json.dumps(huge_response_body))  # ← NEVER!
```

---

## 📖 Lecture 5.6 — Log Levels in Production

```python
import os

# Configure level from environment variable
log_level = os.environ.get("LOG_LEVEL", "INFO").upper()

logging.basicConfig(level=getattr(logging, log_level))
```

Production level rules:
- **Production:** `INFO` — only business events and errors
- **Staging:** `DEBUG` — more details for troubleshooting
- **Local dev:** `DEBUG` — everything

---

## ✅ Module 05 — Review Questions

1. What is the key difference between unstructured and structured logging?
2. Why is including `trace_id` in every log entry so powerful?
3. Which OTel logging approach is recommended for existing Python apps?
4. Name 3 things you should **always** log and 3 things you should **never** log.
5. What log level should you use in production? Why not DEBUG?
6. If a user reports "my order ORD-555 failed", what is your step-by-step debugging process using correlated logs and traces?
7. What Python package provides the OTel logging bridge?
8. How do you make a log record automatically inherit the current span's trace_id?

---

## 🧪 Lab 05 — Correlated Logging End-to-End

**→ See [labs/lab-05-logs.md](./labs/lab-05-logs.md)**

---

*[← Module 04](../Module-04-Metrics/README.md) | [Module 06 →](../Module-06-Collector/README.md)*
