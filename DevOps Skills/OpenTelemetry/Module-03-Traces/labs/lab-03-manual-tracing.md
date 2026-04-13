# Lab 03 — Multi-Service Distributed Tracing

> **Objective:** Build TWO Python HTTP services that communicate with each other, with full trace context propagation between them. You will see a single trace spanning both services in Jaeger — the defining experience of distributed tracing.

---

## 🎯 Lab Goals

1. Build a two-service system (API Gateway + Order Service)
2. Implement trace context propagation via HTTP headers
3. See end-to-end traces spanning multiple services in Jaeger
4. Practice semantic conventions for HTTP spans
5. Record exceptions with full stack trace in spans

---

## ⏱️ Estimated Time: 90 minutes

---

## Prerequisites

```bash
# Install additional packages
pip install flask requests opentelemetry-instrumentation-flask opentelemetry-instrumentation-requests
```

---

## Part 1 — Architecture Overview

```
Client (curl)
     │
     │ HTTP POST /checkout
     ▼
┌────────────────────────┐
│   api-gateway          │  Port 5000
│   (Flask)              │
│   Spans:               │
│   - receive-checkout   │
│   - call-order-svc     │
└────────────┬───────────┘
             │ HTTP POST /orders
             │ + traceparent header (propagation!)
             ▼
┌────────────────────────┐
│   order-service        │  Port 5001
│   (Flask)              │
│   Spans:               │
│   - process-order      │
│   - validate           │
│   - db-insert          │
└────────────────────────┘

Both services → OTLP → Jaeger
One trace, two services, many spans.
```

---

## Part 2 — Set Up Docker Compose

Create `Module-03-Traces/labs/docker-compose.yml`:

```yaml
version: "3.8"

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger-lab03
    ports:
      - "16686:16686"
      - "4317:4317"
      - "4318:4318"
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

```bash
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry\Module-03-Traces\labs"
docker compose up -d
```

---

## Part 3 — Shared OTel Setup Module

Create `Module-03-Traces/labs/code/otel_setup.py`:

```python
"""
otel_setup.py — Shared OpenTelemetry configuration
Each service imports this to configure its SDK.
"""

from opentelemetry import trace, propagate
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.propagators.b3 import B3Format
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator


def configure_otel(service_name: str, service_version: str = "1.0.0"):
    """
    Configure OpenTelemetry for a service.
    Call this ONCE at application startup.
    
    Args:
        service_name: Unique name for this service (shown in Jaeger)
        service_version: Version of this service
    """
    
    resource = Resource(attributes={
        "service.name": service_name,
        "service.version": service_version,
        "deployment.environment": "lab",
    })
    
    exporter = OTLPSpanExporter(
        endpoint="http://localhost:4317",
        insecure=True,
    )
    
    provider = TracerProvider(resource=resource)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)
    
    # Configure propagation: W3C Trace Context (standard)
    propagate.set_global_textmap(TraceContextTextMapPropagator())
    
    print(f"✅ [{service_name}] OpenTelemetry configured → Jaeger at localhost:4317")
    return trace.get_tracer(service_name)
```

---

## Part 4 — Build the Order Service

Create `Module-03-Traces/labs/code/order_service.py`:

```python
"""
order_service.py — The order processing backend service

Run this FIRST (it's the downstream service):
  python order_service.py

Listens on port 5001.
"""

import time
import random
import json
from flask import Flask, request, jsonify
from opentelemetry import trace, propagate
from opentelemetry.trace import SpanKind, Status, StatusCode
from opentelemetry.semconv.trace import SpanAttributes
import sys, os
sys.path.insert(0, os.path.dirname(__file__))
from otel_setup import configure_otel

# ── Configure OTel ──────────────────────────────────────────
tracer = configure_otel("order-service", "1.0.0")

app = Flask(__name__)


# ── Simulated Database ──────────────────────────────────────
ORDER_DB = {}


def db_insert_order(order: dict) -> str:
    """Simulates inserting an order into a database"""
    
    with tracer.start_as_current_span("db.insert_order") as span:
        # Database semantic conventions
        span.set_attribute(SpanAttributes.DB_SYSTEM, "postgresql")
        span.set_attribute(SpanAttributes.DB_NAME, "orders_db")
        span.set_attribute(SpanAttributes.DB_OPERATION, "INSERT")
        span.set_attribute(SpanAttributes.DB_STATEMENT, 
            "INSERT INTO orders (id, user_id, product_id, quantity, total) VALUES (?, ?, ?, ?, ?)")
        span.set_attribute("server.address", "postgres.internal")
        span.set_attribute("server.port", 5432)
        
        # Simulate DB write latency
        time.sleep(random.uniform(0.02, 0.08))
        
        # Occasionally simulate a slow query
        if order.get("quantity", 1) > 10:
            span.add_event("db.slow_query_detected", {
                "threshold_ms": 50,
                "actual_ms": 150,
            })
            time.sleep(0.1)  # Additional slow path
        
        order_id = f"ORD-{random.randint(10000, 99999)}"
        ORDER_DB[order_id] = order
        
        span.set_attribute("db.rows_affected", 1)
        span.add_event("db.transaction.committed")
        
        return order_id


def validate_and_process(order_data: dict) -> dict:
    """Validates order and processes it"""
    
    with tracer.start_as_current_span("order.validate_and_process") as span:
        span.set_attribute("order.user_id", order_data.get("user_id", "unknown"))
        span.set_attribute("order.product_id", order_data.get("product_id", "unknown"))
        span.set_attribute("order.quantity", order_data.get("quantity", 0))
        
        # Validation
        time.sleep(0.01)
        
        if not order_data.get("user_id"):
            raise ValueError("user_id is required")
        if not order_data.get("product_id"):
            raise ValueError("product_id is required")
        if order_data.get("quantity", 0) <= 0:
            raise ValueError(f"Invalid quantity: {order_data.get('quantity')}")
        
        span.add_event("validation.passed")
        
        # Calculate total
        base_price = 49.99
        quantity = order_data["quantity"]
        total = round(base_price * quantity, 2)
        span.set_attribute("order.total", total)
        
        # Insert into DB
        order_id = db_insert_order({**order_data, "total": total})
        
        span.set_attribute("order.id", order_id)
        span.add_event("order.created", {"order_id": order_id, "total": total})
        
        return {"order_id": order_id, "total": total, "status": "created"}


@app.route("/orders", methods=["POST"])
def create_order():
    """POST /orders — Create a new order"""
    
    # ⚡ EXTRACT trace context from incoming headers
    # This links this service's spans to the caller's spans
    context = propagate.extract(request.headers)
    
    with tracer.start_as_current_span(
        "order-service.create_order",
        context=context,           # ← Parent context from API Gateway!
        kind=SpanKind.SERVER,      # ← We are receiving a call
    ) as span:
        
        # HTTP server semantic conventions
        span.set_attribute(SpanAttributes.HTTP_METHOD, "POST")
        span.set_attribute(SpanAttributes.HTTP_ROUTE, "/orders")
        span.set_attribute(SpanAttributes.HTTP_URL, request.url)
        span.set_attribute("server.address", "order-service")
        span.set_attribute("server.port", 5001)
        
        try:
            order_data = request.get_json()
            
            if not order_data:
                span.set_status(Status(StatusCode.ERROR, "No request body"))
                span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, 400)
                return jsonify({"error": "Request body required"}), 400
            
            result = validate_and_process(order_data)
            
            span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, 201)
            span.set_attribute("order.id", result["order_id"])
            span.set_status(Status(StatusCode.OK))
            
            return jsonify(result), 201
            
        except ValueError as e:
            span.record_exception(e)  # ← Records exception with stack trace!
            span.set_status(Status(StatusCode.ERROR, str(e)))
            span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, 400)
            return jsonify({"error": str(e)}), 400
            
        except Exception as e:
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR, "Internal error"))
            span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, 500)
            return jsonify({"error": "Internal server error"}), 500


@app.route("/health")
def health():
    return jsonify({"status": "ok", "service": "order-service"})


if __name__ == "__main__":
    print("🚀 Order Service starting on port 5001...")
    app.run(host="0.0.0.0", port=5001, debug=False)
```

---

## Part 5 — Build the API Gateway

Create `Module-03-Traces/labs/code/api_gateway.py`:

```python
"""
api_gateway.py — The API Gateway (entry point for clients)

Run this SECOND (after order_service.py):
  python api_gateway.py

Listens on port 5000.
"""

import time
import requests as http_requests
from flask import Flask, request, jsonify
from opentelemetry import trace, propagate
from opentelemetry.trace import SpanKind, Status, StatusCode
from opentelemetry.semconv.trace import SpanAttributes
import sys, os
sys.path.insert(0, os.path.dirname(__file__))
from otel_setup import configure_otel

# ── Configure OTel ──────────────────────────────────────────
tracer = configure_otel("api-gateway", "1.0.0")

app = Flask(__name__)

ORDER_SERVICE_URL = "http://localhost:5001"


def authenticate_request(auth_header: str) -> dict:
    """Validates the authorization header"""
    
    with tracer.start_as_current_span("auth.validate_token") as span:
        span.set_attribute("auth.header_present", bool(auth_header))
        
        time.sleep(0.005)  # Fast auth check
        
        if not auth_header:
            span.set_attribute("auth.result", "missing")
            return None
        
        # Simulate token validation
        if auth_header.startswith("Bearer "):
            token = auth_header.split(" ")[1]
            span.set_attribute("auth.token_prefix", token[:4] + "...")
            span.set_attribute("auth.result", "valid")
            
            # Decode tier from token (simplified)
            user_tier = "premium" if "premium" in token else "standard"
            span.set_attribute("auth.user_tier", user_tier)
            
            return {"user_id": f"user-{hash(token) % 1000}", "tier": user_tier}
        
        span.set_attribute("auth.result", "invalid_format")
        return None


def call_order_service(order_payload: dict) -> dict:
    """Makes an HTTP call to the order service WITH trace propagation"""
    
    with tracer.start_as_current_span(
        "http.call_order_service",
        kind=SpanKind.CLIENT,   # ← We are making an outgoing call
    ) as span:
        
        url = f"{ORDER_SERVICE_URL}/orders"
        
        # HTTP client semantic conventions
        span.set_attribute(SpanAttributes.HTTP_METHOD, "POST")
        span.set_attribute(SpanAttributes.HTTP_URL, url)
        span.set_attribute("server.address", "order-service")
        span.set_attribute("server.port", 5001)
        
        # ⚡ INJECT trace context into outgoing headers
        # This is what links api-gateway spans to order-service spans!
        headers = {"Content-Type": "application/json"}
        propagate.inject(headers)
        
        # After inject, headers contains:
        # {"Content-Type": "...", "traceparent": "00-abc123...-01"}
        span.add_event("propagation.injected", {
            "traceparent": headers.get("traceparent", "not-set")[:30] + "..."
        })
        
        try:
            response = http_requests.post(
                url,
                json=order_payload,
                headers=headers,      # ← Propagated headers!
                timeout=10,
            )
            
            span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, response.status_code)
            
            if response.status_code >= 400:
                span.set_status(Status(StatusCode.ERROR, f"HTTP {response.status_code}"))
            else:
                span.set_status(Status(StatusCode.OK))
            
            return response.json(), response.status_code
            
        except http_requests.exceptions.Timeout:
            span.record_exception(Exception("Order service timeout"))
            span.set_status(Status(StatusCode.ERROR, "Timeout"))
            raise
        except http_requests.exceptions.ConnectionError as e:
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR, "Connection refused"))
            raise


@app.route("/checkout", methods=["POST"])
def checkout():
    """POST /checkout — Process a checkout request"""
    
    # This is the ENTRY POINT — creates the ROOT span
    with tracer.start_as_current_span(
        "api-gateway.checkout",
        kind=SpanKind.SERVER,   # ← We receive from the client
    ) as span:
        
        # HTTP server attributes (what the client sees)
        span.set_attribute(SpanAttributes.HTTP_METHOD, "POST")
        span.set_attribute(SpanAttributes.HTTP_ROUTE, "/checkout")
        span.set_attribute(SpanAttributes.HTTP_URL, request.url)
        span.set_attribute("client.address", request.remote_addr)
        
        # Step 1: Authenticate
        user = authenticate_request(request.headers.get("Authorization"))
        
        if not user:
            span.set_status(Status(StatusCode.ERROR, "Unauthorized"))
            span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, 401)
            return jsonify({"error": "Unauthorized"}), 401
        
        span.set_attribute("user.id", user["user_id"])
        span.set_attribute("user.tier", user["tier"])
        span.add_event("auth.successful", {"user_id": user["user_id"]})
        
        # Step 2: Parse request
        body = request.get_json()
        if not body:
            span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, 400)
            return jsonify({"error": "Request body required"}), 400
        
        # Build order payload
        order_payload = {
            "user_id": user["user_id"],
            "product_id": body.get("product_id"),
            "quantity": body.get("quantity", 1),
            "user_tier": user["tier"],
        }
        
        span.set_attribute("order.product_id", order_payload["product_id"])
        span.set_attribute("order.quantity", order_payload["quantity"])
        
        # Step 3: Call Order Service (with propagation)
        try:
            result, status_code = call_order_service(order_payload)
            
            if status_code == 201:
                span.set_attribute("order.id", result.get("order_id", ""))
                span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, 200)
                span.set_status(Status(StatusCode.OK))
                span.add_event("checkout.completed", {"order_id": result.get("order_id")})
                
                return jsonify({
                    "success": True,
                    "order_id": result.get("order_id"),
                    "total": result.get("total"),
                    "message": "Order placed successfully!"
                }), 200
            else:
                span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, status_code)
                span.set_status(Status(StatusCode.ERROR))
                return jsonify(result), status_code
                
        except Exception as e:
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR, str(e)))
            span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, 503)
            return jsonify({"error": "Service unavailable"}), 503


@app.route("/health")
def health():
    return jsonify({"status": "ok", "service": "api-gateway"})


if __name__ == "__main__":
    print("🚀 API Gateway starting on port 5000...")
    print("📡 Forwarding to Order Service at:", ORDER_SERVICE_URL)
    app.run(host="0.0.0.0", port=5000, debug=False)
```

---

## Part 6 — Run and Test

### Step 6.1 — Start services (in separate terminals)

**Terminal 1 — Order Service:**
```bash
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry\Module-03-Traces\labs\code"
.\.venv\Scripts\Activate.ps1  # or activate your venv
python order_service.py
```

Expected:
```
✅ [order-service] OpenTelemetry configured → Jaeger at localhost:4317
🚀 Order Service starting on port 5001...
```

**Terminal 2 — API Gateway:**
```bash
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry\Module-03-Traces\labs\code"
.\.venv\Scripts\Activate.ps1
python api_gateway.py
```

Expected:
```
✅ [api-gateway] OpenTelemetry configured → Jaeger at localhost:4317
🚀 API Gateway starting on port 5000...
```

### Step 6.2 — Send Test Requests

**Terminal 3 — Test requests:**

```bash
# Successful order (premium user)
curl -X POST http://localhost:5000/checkout \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer premium-token-abc123" \
  -d "{\"product_id\": \"PROD-001\", \"quantity\": 2}"

# Successful order (standard user)
curl -X POST http://localhost:5000/checkout \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer standard-token-xyz789" \
  -d "{\"product_id\": \"PROD-500\", \"quantity\": 5}"

# Large order (will trigger slow DB path)
curl -X POST http://localhost:5000/checkout \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer premium-token-abc123" \
  -d "{\"product_id\": \"PROD-100\", \"quantity\": 15}"

# Invalid order (no quantity)
curl -X POST http://localhost:5000/checkout \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer standard-token-xyz789" \
  -d "{\"product_id\": \"PROD-200\", \"quantity\": -1}"

# Unauthorized (no token)
curl -X POST http://localhost:5000/checkout \
  -H "Content-Type: application/json" \
  -d "{\"product_id\": \"PROD-001\", \"quantity\": 1}"
```

**Windows PowerShell equivalent:**
```powershell
# Successful order
Invoke-WebRequest -Uri "http://localhost:5000/checkout" `
  -Method POST `
  -Headers @{"Authorization"="Bearer premium-token-abc123"; "Content-Type"="application/json"} `
  -Body '{"product_id": "PROD-001", "quantity": 2}'
```

---

## Part 7 — Explore the Distributed Traces

Open **http://localhost:16686**

### What to Find

**Multi-Service Trace:**
1. Select service `api-gateway` → Find Traces
2. Click on a trace
3. You should see spans from **BOTH services** in ONE trace!
4. The `api-gateway.checkout` span is the root
5. Under it: `auth.validate_token`, `http.call_order_service`
6. Under the call: spans from `order-service`!

**The Propagation Proof:**
- Add column: Look at the "Process" column for each span
- Some spans say `api-gateway`, some say `order-service`
- But they all share the SAME Trace ID
- This proves context propagation worked!

**Find the Slow Trace:**
- Sort by duration
- The `quantity=15` order should be slowest (DB slow query)
- Expand it — the `db.insert_order` span should show the slow query event

**Find the Error Trace:**
- Look for red traces (the `quantity=-1` order)
- Click it — see the ERROR status on `order.validate_and_process`
- Click that span — see the recorded exception with stack trace

### Screenshot Checklist

Take note of (or screenshot):
- [ ] A trace showing **both** `api-gateway` and `order-service` spans
- [ ] The `traceparent` in the propagation injected event
- [ ] The span attributes for the DB span
- [ ] An error span with exception details

---

## ✅ Lab 03 Completion Checklist

- [ ] Both services running without errors
- [ ] Successfully sent 5 test requests
- [ ] Found multi-service traces in Jaeger (both services in one trace)
- [ ] Identified `api-gateway` and `order-service` spans in same trace
- [ ] Found and understood the slow trace (quantity=15)
- [ ] Found the error trace and examined the exception details
- [ ] Understood how `propagate.inject()` and `propagate.extract()` work together

---

## 🧠 Lab Questions

1. Copy the `traceparent` header value from one of your traces (hint: look at the propagation.injected event). What is the Trace ID portion?
2. How many spans does a successful checkout request produce? List them.
3. What method did you call in `api_gateway.py` to add trace context to outgoing headers?
4. What method did you call in `order_service.py` to read trace context from incoming headers?
5. What does `kind=SpanKind.SERVER` vs `kind=SpanKind.CLIENT` tell Jaeger?
6. If you ran the order service without `from otel_setup import configure_otel` (no SDK configured), what would happen to the spans in the order service?

---

## Cleanup

```bash
docker compose down
```

---

*[← Back to Module 03](../README.md) | [Module 04 →](../../Module-04-Metrics/README.md)*
