# Lab 01 — The Three Pillars in Practice

> **Objective:** Experience the difference between a "dark" system and an "observable" system. You will run a broken service with no observability, experience the pain of debugging it, then add basic telemetry and feel the difference.

---

## 🎯 Lab Goals

1. Run a service with **zero observability** and try to find a bug
2. Add **logs** to gain some visibility
3. Add **metrics** to see trends
4. See how **traces** would complete the picture
5. Understand why all three pillars matter

---

## ⏱️ Estimated Time: 45 minutes

---

## 📋 Prerequisites

- Python 3.9+ installed
- pip working

---

## Part 1 — The Dark System (No Observability)

### Step 1.1 — Create the broken service

Create a directory and file:

```
OpenTelemetry/Module-01-Observability-Foundations/labs/code/
```

Create file `dark_service.py`:

```python
# dark_service.py — A service with NO observability
# This simulates what many legacy systems look like

import random
import time

def get_user(user_id: str) -> dict:
    """Fetch user from 'database'"""
    time.sleep(random.uniform(0.01, 0.05))  # Normal DB call
    if user_id == "user-404":
        return None
    return {"id": user_id, "name": "Alice", "tier": "premium"}


def check_inventory(product_id: str, quantity: int) -> bool:
    """Check if product is in stock"""
    # Bug: Random slowness for specific products
    if product_id.startswith("PROD-9"):
        time.sleep(2.0)  # ← HIDDEN BUG: slow for PROD-9xx products!
    
    time.sleep(random.uniform(0.01, 0.03))
    return random.random() > 0.1  # 90% in stock


def calculate_price(product_id: str, quantity: int, user_tier: str) -> float:
    """Calculate price with discounts"""
    base_price = 49.99
    if user_tier == "premium":
        return base_price * quantity * 0.9  # 10% discount
    return base_price * quantity


def process_order(user_id: str, product_id: str, quantity: int) -> dict:
    """Main business logic"""
    user = get_user(user_id)
    if not user:
        return {"success": False, "error": "User not found"}
    
    in_stock = check_inventory(product_id, quantity)
    if not in_stock:
        return {"success": False, "error": "Out of stock"}
    
    price = calculate_price(product_id, quantity, user["tier"])
    return {"success": True, "total": price, "order_id": f"ORD-{random.randint(1000,9999)}"}


# Simulate 20 orders
print("Processing orders...")
start = time.time()

results = {"success": 0, "failure": 0}
for i in range(20):
    user_id = random.choice(["user-1", "user-2", "user-404"])
    product_id = random.choice(["PROD-001", "PROD-500", "PROD-901", "PROD-999"])
    quantity = random.randint(1, 5)
    
    result = process_order(user_id, product_id, quantity)
    
    if result["success"]:
        results["success"] += 1
    else:
        results["failure"] += 1

elapsed = time.time() - start
print(f"Done. Success: {results['success']}, Failures: {results['failure']}")
print(f"Total time: {elapsed:.2f}s")
```

### Step 1.2 — Run it and try to debug

```bash
python dark_service.py
```

**You will see something like:**
```
Processing orders...
Done. Success: 14, Failures: 6
Total time: 8.43s
```

**Now answer these questions (without changing the code):**
- Why did it take 8 seconds? Which operation was slow?
- Which user_ids are causing failures?
- Which product_ids are causing slowness?
- Is the slowness consistent or random?

**Answer: You cannot tell. The system is completely dark.**

---

## Part 2 — Add Logs (First Pillar)

### Step 2.1 — Create the logged version

Create `logged_service.py`:

```python
# logged_service.py — Adding structured logging
import random
import time
import logging
import json
from datetime import datetime, timezone

# ============================================================
# STRUCTURED LOGGING SETUP
# ============================================================
class JSONFormatter(logging.Formatter):
    """Formats log records as JSON for machine-readability"""
    def format(self, record):
        log_entry = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "service": "order-service",
            "message": record.getMessage(),
        }
        # Add extra fields if provided
        for key, value in record.__dict__.items():
            if key not in ('name', 'msg', 'args', 'levelname', 'levelno', 
                          'pathname', 'filename', 'module', 'exc_info', 
                          'exc_text', 'stack_info', 'lineno', 'funcName',
                          'created', 'msecs', 'relativeCreated', 'thread',
                          'threadName', 'processName', 'process', 'message',
                          'taskName'):
                log_entry[key] = value
        return json.dumps(log_entry)

# Set up logger
logger = logging.getLogger("order-service")
logger.setLevel(logging.DEBUG)
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)
# ============================================================


def get_user(user_id: str) -> dict:
    logger.info("Fetching user", extra={"user_id": user_id, "operation": "get_user"})
    start = time.time()
    
    time.sleep(random.uniform(0.01, 0.05))
    
    if user_id == "user-404":
        logger.warning("User not found", extra={"user_id": user_id, "duration_ms": round((time.time()-start)*1000, 2)})
        return None
    
    duration = round((time.time() - start) * 1000, 2)
    logger.info("User fetched successfully", extra={"user_id": user_id, "duration_ms": duration})
    return {"id": user_id, "name": "Alice", "tier": "premium"}


def check_inventory(product_id: str, quantity: int) -> bool:
    logger.info("Checking inventory", extra={"product_id": product_id, "quantity": quantity})
    start = time.time()
    
    if product_id.startswith("PROD-9"):
        logger.warning("Slow inventory path detected", extra={"product_id": product_id, "reason": "PROD-9xx slow path"})
        time.sleep(2.0)
    
    time.sleep(random.uniform(0.01, 0.03))
    in_stock = random.random() > 0.1
    
    duration = round((time.time() - start) * 1000, 2)
    logger.info("Inventory check complete", extra={
        "product_id": product_id,
        "in_stock": in_stock,
        "duration_ms": duration
    })
    return in_stock


def calculate_price(product_id: str, quantity: int, user_tier: str) -> float:
    base_price = 49.99
    if user_tier == "premium":
        price = base_price * quantity * 0.9
        logger.debug("Premium discount applied", extra={"user_tier": user_tier, "discount": "10%", "final_price": price})
        return price
    return base_price * quantity


def process_order(user_id: str, product_id: str, quantity: int) -> dict:
    logger.info("=== Order processing started ===", extra={
        "user_id": user_id,
        "product_id": product_id,
        "quantity": quantity
    })
    order_start = time.time()

    user = get_user(user_id)
    if not user:
        logger.error("Order failed: user not found", extra={"user_id": user_id})
        return {"success": False, "error": "User not found"}
    
    in_stock = check_inventory(product_id, quantity)
    if not in_stock:
        logger.error("Order failed: out of stock", extra={"product_id": product_id})
        return {"success": False, "error": "Out of stock"}
    
    price = calculate_price(product_id, quantity, user["tier"])
    order_id = f"ORD-{random.randint(1000,9999)}"
    
    total_duration = round((time.time() - order_start) * 1000, 2)
    logger.info("Order completed successfully", extra={
        "order_id": order_id,
        "total": price,
        "duration_ms": total_duration,
        "user_id": user_id,
        "product_id": product_id
    })
    
    return {"success": True, "total": price, "order_id": order_id}


# Simulate 10 orders (fewer for readability)
print("=" * 70)
print("PROCESSING ORDERS WITH LOGGING")
print("=" * 70)

results = {"success": 0, "failure": 0}
for i in range(10):
    user_id = random.choice(["user-1", "user-2", "user-404"])
    product_id = random.choice(["PROD-001", "PROD-500", "PROD-901", "PROD-999"])
    quantity = random.randint(1, 5)
    
    result = process_order(user_id, product_id, quantity)
    if result["success"]:
        results["success"] += 1
    else:
        results["failure"] += 1

print("=" * 70)
print(f"SUMMARY: Success: {results['success']}, Failures: {results['failure']}")
```

### Step 2.2 — Run and observe

```bash
python logged_service.py 2>&1 | python -m json.tool --no-ensure-ascii
```

Or simply:
```bash
python logged_service.py
```

**Now you can see:**
- ✅ Which user_ids cause failures  
- ✅ Which product_ids trigger the slow path
- ✅ How long each operation takes
- ✅ Clear error messages

**What logs still can't tell you:**
- ❌ The relationship between the slowness and impact on the overall request
- ❌ Aggregate latency trends over time
- ❌ How many requests per second you're processing

---

## Part 3 — Understanding Metrics (Second Pillar)

### Step 3.1 — Conceptual understanding exercise

Logs are great for debugging individual requests, but consider this scenario:
- You have 1,000 requests per second
- That's 60,000 log lines per minute
- Reading logs to understand system health is impossible

This is where **metrics** shine. Instead of reading every log, you aggregate:

```
Logs say:
  08:15:00 - order success, 45ms
  08:15:00 - order success, 47ms
  08:15:00 - order FAIL (out of stock)
  08:15:00 - order success, 2100ms ← product PROD-901
  08:15:00 - order success, 52ms
  ... (thousands more)

Metrics say:
  order_requests_total{status="success"} = 8453
  order_requests_total{status="failure"} = 1205
  order_latency_p50 = 48ms
  order_latency_p99 = 2150ms  ← Alert! Something is slow at p99!
  order_latency_p99{product="PROD-9xx"} = 2100ms ← Found it!
```

### Step 3.2 — Run the metrics simulation

Create `metrics_simulation.py`:

```python
# metrics_simulation.py — Simulating what metrics show you
import random
import time
from collections import defaultdict

# Simulate a "metrics store" (in production, this would be Prometheus)
class SimpleMetrics:
    def __init__(self):
        self.counters = defaultdict(int)
        self.durations = defaultdict(list)
    
    def increment(self, name, labels=None):
        key = name + str(sorted((labels or {}).items()))
        self.counters[key] += 1
    
    def record_duration(self, name, duration_ms, labels=None):
        key = name + str(sorted((labels or {}).items()))
        self.durations[key].append(duration_ms)
    
    def report(self):
        print("\n" + "="*60)
        print("📊 METRICS REPORT")
        print("="*60)
        
        print("\n📈 COUNTERS:")
        for key, value in sorted(self.counters.items()):
            print(f"  {key}: {value}")
        
        print("\n⏱️  LATENCY DISTRIBUTIONS:")
        for key, values in sorted(self.durations.items()):
            if values:
                sorted_vals = sorted(values)
                p50 = sorted_vals[int(len(sorted_vals) * 0.50)]
                p95 = sorted_vals[int(len(sorted_vals) * 0.95)]
                p99 = sorted_vals[int(len(sorted_vals) * 0.99)] if len(sorted_vals) >= 100 else sorted_vals[-1]
                avg = sum(sorted_vals) / len(sorted_vals)
                print(f"  {key}:")
                print(f"    count={len(values)}, avg={avg:.1f}ms, p50={p50:.1f}ms, p95={p95:.1f}ms, p99={p99:.1f}ms")

metrics = SimpleMetrics()

def simulate_request():
    product_id = random.choice(["PROD-001", "PROD-500", "PROD-901", "PROD-999"])
    user_id = random.choice(["user-1", "user-2", "user-404"])
    
    start = time.time()
    
    # Simulate the slow bug for PROD-9xx
    if product_id.startswith("PROD-9"):
        time.sleep(0.05)  # Simulating scaled-down 2s → 50ms for the demo
        success = True
    elif user_id == "user-404":
        success = False
    else:
        time.sleep(random.uniform(0.001, 0.005))
        success = random.random() > 0.05
    
    duration_ms = (time.time() - start) * 1000
    
    # Record metrics
    product_family = "PROD-9xx" if product_id.startswith("PROD-9") else "PROD-other"
    
    metrics.increment("order_requests_total", {"status": "success" if success else "failure"})
    metrics.record_duration("order_latency_ms", duration_ms)
    metrics.record_duration("order_latency_ms_by_product", duration_ms, {"product_family": product_family})

# Run simulation
print("Running 500 simulated requests...")
for _ in range(500):
    simulate_request()

metrics.report()

print("\n💡 INSIGHT: The PROD-9xx product family has dramatically higher latency!")
print("   This pattern would trigger a p99 alert — pointing you directly to the problem.")
```

```bash
python metrics_simulation.py
```

---

## Part 4 — Reflection: What Would Traces Add?

Even with logs and metrics, there's still a gap. Create `reflection.md` in the same directory:

```markdown
## What Traces Add (That Logs and Metrics Cannot)

### Scenario: Complex Microservice Request

Imagine the order service calls 4 other services:
OrderService → InventoryService → CacheService → DatabaseService
OrderService → PricingService → TaxService

Logs tell you: "InventoryService call took 800ms"
Metrics tell you: "p99 for inventory checks is 750ms"

BUT traces tell you:
- The InventoryService spent only 50ms in its own code
- It spent 750ms waiting for CacheService
- The CacheService was waiting for DatabaseService (missing Redis key)
- This happens for orders over $100 (attribute on the span)

Without traces, you would blame InventoryService.
With traces, you find the root cause: Cache miss for high-value orders hitting DB directly.

Key: Traces show you the CAUSAL CHAIN across service boundaries.
```

---

## ✅ Lab 01 Completion Checklist

- [ ] Ran `dark_service.py` and experienced debugging blindness
- [ ] Added structured JSON logging and could identify the bug
- [ ] Ran `metrics_simulation.py` and saw how metrics aggregate behavior
- [ ] Understood conceptually what traces add on top of logs + metrics
- [ ] Can explain in your own words why all three pillars are needed

---

## 🧠 Lab Questions

1. What specific bug did logs reveal that was invisible in the dark service?
2. Look at the metric p99 values. Which product family is causing high latency?
3. If you had to explain to a manager why the system was slow, what would you say using only metrics? What could you add using logs?
4. What would a trace across `order-service → inventory-service → database` look like as a diagram?

---

*[← Back to Module 01](../README.md) | [Module 02 →](../../Module-02-OTel-Architecture/README.md)*
