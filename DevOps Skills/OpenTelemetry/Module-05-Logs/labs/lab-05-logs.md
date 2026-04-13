# Lab 05 — Correlated Logging End-to-End

> **Objective:** Configure Python's standard logging library to bridge with OpenTelemetry. You will write code that emits logs, creates spans, and see how the logs automatically pick up the Trace ID and Span ID of the currently active span.

---

## ⏱️ Estimated Time: 30 minutes

---

## Part 1 — Install Packages

```bash
pip install opentelemetry-sdk opentelemetry-exporter-otlp-proto-grpc opentelemetry-instrumentation-logging
```

---

## Part 2 — The Python Logging Bridge

Create `Module-05-Logs/labs/code/logging_app.py`:

```python
"""
logging_app.py — Demonstrates standard Python logging bridged to OpenTelemetry
so that logs automatically carry trace_id and span_id.
"""

import logging
import time
import json
import random
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.instrumentation.logging import LoggingInstrumentor

# ============================================================
# 1. SETUP STRUCTURED LOGGING FORMAT
# ============================================================

class JSONFormatter(logging.Formatter):
    """
    Formats logs as JSON.
    CRITICAL: It includes 'trace_id' and 'span_id' added by the OTel bridge!
    """
    def format(self, record):
        log_entry = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            
            # These are added automatically when inside an OTel span!
            "trace_id": getattr(record, "otelTraceID", "0"),
            "span_id": getattr(record, "otelSpanID", "0"),
        }
        
        # Add any extra kwargs passed to logger
        for key, value in record.__dict__.items():
            if key not in ['args', 'asctime', 'created', 'exc_info', 'exc_text', 
                           'filename', 'funcName', 'id', 'levelname', 'levelno', 
                           'lineno', 'message', 'module', 'msecs', 'msg', 'name', 
                           'pathname', 'process', 'processName', 'relativeCreated', 
                           'stack_info', 'thread', 'threadName', 'taskName',
                           'otelTraceID', 'otelSpanID', 'otelServiceName', 'otelTraceSampled']:
                log_entry[key] = value
                
        return json.dumps(log_entry)


# Set up standard Python logger
logger = logging.getLogger("order-processor")
logger.setLevel(logging.INFO)
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)


# ============================================================
# 2. SETUP OPENTELEMETRY TRACING
# ============================================================

resource = Resource({"service.name": "order-processor", "service.version": "1.0.0"})
provider = TracerProvider(resource=resource)

# We'll just print spans to console for this demo instead of Jaeger
# so you can see the correlation in the terminal output
from opentelemetry.sdk.trace.export import ConsoleSpanExporter
provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("order-processor")

# 🔥 CRITICAL STEP: Tells Python logging to inject OTel context
LoggingInstrumentor().instrument(set_logging_format=False)


# ============================================================
# 3. APPLICATION CODE
# ============================================================

def process_payment(order_id: str, amount: float):
    # This log is inside the child span
    with tracer.start_as_current_span("process-payment") as span:
        span.set_attribute("payment.amount", amount)
        
        logger.info("Initiating payment to Stripe", extra={
            "order_id": order_id, 
            "gateway": "stripe"
        })
        time.sleep(0.1)
        
        if random.random() > 0.8:
            logger.error("Payment declined due to insufficient funds", extra={
                "order_id": order_id,
                "error_code": "insufficient_funds"
            })
            raise ValueError("Payment declined")
            
        logger.info("Payment successful", extra={"order_id": order_id})
        return True

def handle_order(user_id: str, quantity: int):
    # This log is NOT inside a span - watch its trace_id!
    logger.info("Received new raw request on port 8080", extra={"raw_qty": quantity})
    
    order_id = f"ORD-{random.randint(1000, 9999)}"
    
    # NOW we start the trace
    with tracer.start_as_current_span("handle-order") as span:
        span.set_attribute("order.id", order_id)
        
        # This log is INSIDE the root span!
        logger.info("Starting order processing", extra={
            "order_id": order_id,
            "user_id": user_id
        })
        
        try:
            total = 49.99 * quantity
            process_payment(order_id, total)
            
            logger.info("Order completed successfully", extra={
                "order_id": order_id,
                "total": total
            })
        except ValueError:
            logger.warning("Order failed, notifying user", extra={
                "order_id": order_id
            })

if __name__ == "__main__":
    print("\n🚀 Starting Logging Test\n")
    
    handle_order("user-123", 2)
    print("\n" + "="*80 + "\n")
    handle_order("user-456", 500) # Might fail
    
    print("\n⏳ Flushing spans...")
```

---

## Part 3 — Run and Observe

```bash
python Module-05-Logs/labs/code/logging_app.py
```

### Inspect the Output!

Look closely at the JSON lines printed to your terminal.

1. **The first log (outside span):**
   ```json
   {"level": "INFO", "message": "Received new raw request...", "trace_id": "0", "span_id": "0", ...}
   ```
   Notice that `trace_id` and `span_id` are `0`. There is no active trace context.

2. **The second log (inside root span):**
   ```json
   {"level": "INFO", "message": "Starting order processing", "trace_id": "3b29c...", "span_id": "8f1a...", ...}
   ```
   Now it has a real `trace_id` and `span_id`!

3. **The third log (inside child span - payment):**
   ```json
   {"level": "INFO", "message": "Initiating payment...", "trace_id": "3b29c...", "span_id": "92cd...", ...}
   ```
   Notice that the `trace_id` is the **EXACT SAME** as the second log! It's part of the same trace. But the `span_id` is different (it's the payment span's ID).

---

## ✅ Lab 05 Completion Checklist

- [ ] Ran the script
- [ ] Verified logs outside spans have `trace_id: "0"`
- [ ] Verified logs inside spans have real `trace_ids`
- [ ] Verified that logs in different nested spans share the same `trace_id` but have different `span_ids`

---

*[← Back to Module 05](../README.md) | [Module 06 →](../../Module-06-Collector/README.md)*
