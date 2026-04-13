# Lab 02 — Your First Real Trace with the OTel SDK

> **Objective:** Set up a complete local OpenTelemetry environment with Jaeger, write a Python service that emits real traces, and see them visualized in the Jaeger UI. By the end you will have seen real trace data flowing end-to-end.

---

## 🎯 Lab Goals

1. Install the OpenTelemetry Python SDK
2. Run Jaeger locally with Docker
3. Configure the SDK to export traces via OTLP
4. Write a multi-function Python application with spans
5. View and interpret traces in the Jaeger UI

---

## ⏱️ Estimated Time: 60 minutes

---

## Prerequisites

- Docker Desktop running
- Python 3.9+ with pip
- Virtual environment from Module 01 activated

---

## Part 1 — Install OpenTelemetry SDK Packages

```bash
# Activate your virtual environment
.venv\Scripts\Activate.ps1

# Install OTel SDK packages
pip install \
  opentelemetry-api \
  opentelemetry-sdk \
  opentelemetry-exporter-otlp-proto-grpc \
  opentelemetry-semantic-conventions

# Verify
pip show opentelemetry-api
```

**Expected output:**
```
Name: opentelemetry-api
Version: 1.x.x
...
```

---

## Part 2 — Start Jaeger with Docker

Jaeger is our trace backend. We'll run it as a single "all-in-one" container.

### Step 2.1 — Create Docker Compose file

Create the file at:
```
Module-02-OTel-Architecture/labs/docker-compose.yml
```

```yaml
version: "3.8"

services:
  # Jaeger All-in-One: receives traces and provides UI
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    ports:
      - "16686:16686"   # Jaeger UI
      - "4317:4317"     # OTLP gRPC receiver
      - "4318:4318"     # OTLP HTTP receiver
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    restart: unless-stopped
```

### Step 2.2 — Start Jaeger

```bash
# From the labs/ directory
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry\Module-02-OTel-Architecture\labs"

docker compose up -d

# Verify it's running
docker compose ps
```

**Expected output:**
```
NAME      IMAGE                          STATUS
jaeger    jaegertracing/all-in-one:...   Up
```

### Step 2.3 — Open the Jaeger UI

Open your browser: **http://localhost:16686**

You should see the Jaeger UI with no traces yet. Keep this open!

---

## Part 3 — Write Your First Traced Application

Create `Module-02-OTel-Architecture/labs/traced_service.py`:

```python
"""
traced_service.py — First complete OTel traced application

This simulates an e-commerce order processing service.
Watch the traces appear in Jaeger as you run this!
"""

import time
import random
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.trace import Status, StatusCode


# ============================================================
# STEP 1: CONFIGURE THE SDK (do this once at startup)
# ============================================================

def configure_tracing():
    """Set up the OpenTelemetry SDK"""
    
    # Resource: describes this service
    resource = Resource(attributes={
        "service.name": "order-service",
        "service.version": "1.0.0",
        "deployment.environment": "local-lab",
    })
    
    # Exporter: sends traces to Jaeger via OTLP
    exporter = OTLPSpanExporter(
        endpoint="http://localhost:4317",  # Jaeger OTLP gRPC port
        insecure=True,
    )
    
    # Provider: the central SDK configuration
    provider = TracerProvider(resource=resource)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    
    # Register globally
    trace.set_tracer_provider(provider)
    
    print("✅ OpenTelemetry configured → sending to Jaeger at localhost:4317")


# ============================================================
# STEP 2: GET A TRACER (one per component/module)
# ============================================================
configure_tracing()
tracer = trace.get_tracer("order-service.main")


# ============================================================
# STEP 3: INSTRUMENT YOUR FUNCTIONS
# ============================================================

def validate_order(user_id: str, product_id: str, quantity: int):
    """Validates the incoming order request"""
    
    with tracer.start_as_current_span("validate-order") as span:
        # Set attributes — this is metadata about the operation
        span.set_attribute("order.user_id", user_id)
        span.set_attribute("order.product_id", product_id)
        span.set_attribute("order.quantity", quantity)
        span.set_attribute("validation.rules_checked", 3)
        
        time.sleep(0.01)  # Simulate validation work
        
        # Validate quantity
        if quantity <= 0 or quantity > 100:
            span.set_status(Status(StatusCode.ERROR, "Invalid quantity"))
            span.set_attribute("validation.error", "quantity_out_of_range")
            raise ValueError(f"Invalid quantity: {quantity}")
        
        # Add an event — a point-in-time occurrence
        span.add_event("validation.passed", {
            "rules": "quantity_check,product_exists,user_active"
        })
        
        span.set_status(Status(StatusCode.OK))
        return True


def check_inventory(product_id: str, quantity: int) -> bool:
    """Checks if the product is available in inventory"""
    
    with tracer.start_as_current_span("check-inventory") as span:
        span.set_attribute("inventory.product_id", product_id)
        span.set_attribute("inventory.requested_qty", quantity)
        
        # Simulate: some products are slower (database call)
        if product_id.startswith("PROD-9"):
            span.add_event("cache.miss", {"reason": "product_9xx_no_cache"})
            time.sleep(0.2)  # Slower: cache miss, hit DB
            span.set_attribute("inventory.cache_hit", False)
        else:
            span.add_event("cache.hit")
            time.sleep(0.02)  # Fast: from cache
            span.set_attribute("inventory.cache_hit", True)
        
        # Simulate 85% chance of being in stock
        in_stock = random.random() > 0.15
        available_qty = random.randint(quantity, quantity + 50) if in_stock else 0
        
        span.set_attribute("inventory.available_qty", available_qty)
        span.set_attribute("inventory.in_stock", in_stock)
        
        if not in_stock:
            span.set_status(Status(StatusCode.ERROR))
            span.add_event("inventory.out_of_stock")
        
        return in_stock


def calculate_price(product_id: str, quantity: int, user_tier: str) -> float:
    """Calculates the final price including discounts"""
    
    with tracer.start_as_current_span("calculate-price") as span:
        span.set_attribute("pricing.product_id", product_id)
        span.set_attribute("pricing.quantity", quantity)
        span.set_attribute("pricing.user_tier", user_tier)
        
        time.sleep(0.005)  # Very fast operation
        
        base_price = 49.99
        discount = 0.0
        
        if user_tier == "premium":
            discount = 0.10
            span.add_event("pricing.discount_applied", {
                "type": "premium_tier",
                "discount_pct": 10
            })
        elif user_tier == "vip":
            discount = 0.20
            span.add_event("pricing.discount_applied", {
                "type": "vip_tier",
                "discount_pct": 20
            })
        
        final_price = base_price * quantity * (1 - discount)
        
        span.set_attribute("pricing.base_price", base_price)
        span.set_attribute("pricing.discount_pct", discount * 100)
        span.set_attribute("pricing.final_price", round(final_price, 2))
        
        return round(final_price, 2)


def process_payment(order_id: str, amount: float) -> bool:
    """Processes payment (simulated)"""
    
    with tracer.start_as_current_span("process-payment") as span:
        span.set_attribute("payment.order_id", order_id)
        span.set_attribute("payment.amount", amount)
        span.set_attribute("payment.currency", "USD")
        span.set_attribute("payment.method", "credit_card")
        
        span.add_event("payment.gateway.called", {
            "gateway": "stripe",
            "endpoint": "charges/create"
        })
        
        time.sleep(0.05)  # Payment gateway call
        
        # Simulate 95% success rate
        success = random.random() > 0.05
        
        if success:
            span.add_event("payment.approved", {"authorization_code": "AUTH-XYZ789"})
            span.set_attribute("payment.status", "approved")
            span.set_status(Status(StatusCode.OK))
        else:
            span.add_event("payment.declined", {"reason": "insufficient_funds"})
            span.set_attribute("payment.status", "declined")
            span.set_status(Status(StatusCode.ERROR, "Payment declined"))
        
        return success


def process_order(user_id: str, product_id: str, quantity: int, user_tier: str = "standard"):
    """
    Main function: processes a complete e-commerce order.
    This creates the ROOT SPAN — all other spans are children of this one.
    """
    
    # The ROOT SPAN — represents the entire operation
    with tracer.start_as_current_span("process-order") as span:
        
        # Set attributes on the root span
        span.set_attribute("order.user_id", user_id)
        span.set_attribute("order.product_id", product_id)
        span.set_attribute("order.quantity", quantity)
        span.set_attribute("order.user_tier", user_tier)
        
        order_id = f"ORD-{random.randint(10000, 99999)}"
        span.set_attribute("order.id", order_id)
        
        span.add_event("order.started")
        
        try:
            # Step 1: Validate
            validate_order(user_id, product_id, quantity)
            
            # Step 2: Check inventory
            if not check_inventory(product_id, quantity):
                span.set_status(Status(StatusCode.ERROR, "Out of stock"))
                span.set_attribute("order.outcome", "failed_inventory")
                return {"success": False, "error": "Out of stock", "order_id": order_id}
            
            # Step 3: Calculate price
            price = calculate_price(product_id, quantity, user_tier)
            span.set_attribute("order.total", price)
            
            # Step 4: Process payment
            if not process_payment(order_id, price):
                span.set_status(Status(StatusCode.ERROR, "Payment failed"))
                span.set_attribute("order.outcome", "failed_payment")
                return {"success": False, "error": "Payment declined", "order_id": order_id}
            
            # Success!
            span.add_event("order.completed", {"total": price})
            span.set_status(Status(StatusCode.OK))
            span.set_attribute("order.outcome", "success")
            
            return {"success": True, "order_id": order_id, "total": price}
            
        except ValueError as e:
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR, str(e)))
            span.set_attribute("order.outcome", "failed_validation")
            return {"success": False, "error": str(e), "order_id": order_id}


# ============================================================
# STEP 4: RUN THE SERVICE
# ============================================================

if __name__ == "__main__":
    print("\n🚀 Order Service Starting...")
    print("📡 Sending traces to Jaeger: http://localhost:16686\n")
    
    test_orders = [
        {"user_id": "user-1", "product_id": "PROD-001", "quantity": 2, "user_tier": "premium"},
        {"user_id": "user-2", "product_id": "PROD-500", "quantity": 1, "user_tier": "standard"},
        {"user_id": "user-3", "product_id": "PROD-901", "quantity": 3, "user_tier": "vip"},   # ← Slow!
        {"user_id": "user-4", "product_id": "PROD-999", "quantity": 1, "user_tier": "premium"},  # ← Slow!
        {"user_id": "user-5", "product_id": "PROD-100", "quantity": 0, "user_tier": "standard"},  # ← Invalid!
    ]
    
    results = []
    for order in test_orders:
        print(f"Processing: user={order['user_id']}, product={order['product_id']}")
        result = process_order(**order)
        results.append(result)
        print(f"  Result: {result}")
        time.sleep(0.1)  # Small pause between orders
    
    print(f"\n✅ Processed {len(results)} orders")
    print(f"   Success: {sum(1 for r in results if r['success'])}")
    print(f"   Failed:  {sum(1 for r in results if not r['success'])}")
    
    # Give the BatchSpanProcessor time to flush
    print("\n⏳ Flushing traces to Jaeger...")
    time.sleep(2)
    print("✅ Done! Check Jaeger UI at http://localhost:16686")
    print("   → Select service 'order-service' and click 'Find Traces'")
```

---

## Part 4 — Run the Application

```bash
# From the labs directory
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry\Module-02-OTel-Architecture\labs"

# Run
python traced_service.py
```

**Expected output:**
```
🚀 Order Service Starting...
📡 Sending traces to Jaeger: http://localhost:16686

Processing: user=user-1, product=PROD-001
  Result: {'success': True, 'order_id': 'ORD-54321', 'total': 89.98}
Processing: user=user-2, product=PROD-500
  Result: {'success': True, 'order_id': 'ORD-12345', 'total': 49.99}
...

✅ Done! Check Jaeger UI at http://localhost:16686
```

---

## Part 5 — Explore Traces in Jaeger UI

1. **Open** http://localhost:16686

2. **Select Service:** In the left panel, select `order-service`

3. **Click "Find Traces"**

4. **You should see 5 traces** (one per order)

### What to Look For

**Click on a successful trace:**
- You see `process-order` as the root span
- Under it: `validate-order`, `check-inventory`, `calculate-price`, `process-payment`
- The timeline shows how long each step took
- Click any span to see its attributes (user_id, product_id, etc.)
- Look for the Events tab to see the events you added

**Find the slow trace:**
- Look for traces with `PROD-901` or `PROD-999`
- The `check-inventory` span should be noticeably slower (200ms vs 20ms)
- This is the bug you couldn't see without traces!

**Find the failed trace:**
- The trace for `quantity=0` should show as failed (red)
- The root span `process-order` is red
- Clicking into `validate-order` shows the error status
- Look at the exception details

### Screenshots to Note

Capture what you see for your own reference:
- The trace list (all 5 traces with their durations)
- One expanded trace (the span waterfall)
- One span's attributes panel
- One error span with exception details

---

## Part 6 — Cleanup

```bash
# Stop Jaeger
docker compose down
```

---

## ✅ Lab 02 Completion Checklist

- [ ] Installed OpenTelemetry Python SDK packages
- [ ] Started Jaeger with Docker Compose
- [ ] Successfully ran `traced_service.py` without errors
- [ ] Found the 5 traces in Jaeger UI
- [ ] Identified the slow `PROD-9xx` trace by looking at span durations
- [ ] Found the error trace for the invalid quantity order
- [ ] Examined attributes on individual spans
- [ ] Saw the events attached to spans

---

## 🧠 Lab Questions

1. Open the trace for the `PROD-901` order. Which span is the slowest? How long did it take?
2. What is the Trace ID of one of your traces? (find it in Jaeger)
3. Look at the `process-order` span attributes. Which attributes are present?
4. What is the difference between a span **attribute** and a span **event**?
5. Why did we call `time.sleep(2)` at the end of the script?
6. What would happen to your traces if you stopped Jaeger but kept your app running?

---

*[← Back to Module 02](../README.md) | [Module 03 →](../../Module-03-Traces/README.md)*
