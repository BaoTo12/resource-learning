# Module 01 — Observability Foundations: The Three Pillars

> **Professor's Note:** Before writing a single line of OpenTelemetry code, you must understand *why* it exists. This module answers the fundamental question every engineer should ask: "What is happening inside my system right now?" We explore the philosophical and practical difference between monitoring and observability, and why modern distributed systems demand a new approach.

---

## 📖 Lecture 1.1 — The Problem: Distributed Systems Are Dark

### The Old World (Monolith)

Imagine a single application running as one process on one server. When something breaks:

```
User → [Monolith App] → Database

"The app is slow"
→ ssh into the server
→ check CPU, memory, disk
→ tail the log file
→ Found it: database query taking 5 seconds
→ Fixed. Done.
```

Simple. One server, one log file, one place to look.

### The New World (Microservices)

Now imagine the same application, but modern:

```
User
  │
  ▼
[API Gateway]
  │         │
  ▼         ▼
[Order    [User
Service]  Service]
  │         │
  ▼         ▼
[Inventory] [Auth]
  │
  ▼
[Payment]
  │
  ▼
[Database]   [Queue]   [Cache]
```

"The checkout is slow for some users in Southeast Asia on Tuesdays."

Where do you look? There are now:
- 6+ services, each with their own logs
- Multiple instances of each service
- Network hops between every service
- Shared dependencies (cache, queue, database)
- Different teams owning different services

**This is the fundamental problem OpenTelemetry solves.**

---

## 📖 Lecture 1.2 — Monitoring vs. Observability

These terms are often used interchangeably, but they mean very different things.

### Monitoring

> "Monitoring tells you **when** something is broken."

Monitoring is about tracking predefined metrics and alerting when thresholds are crossed:

```
CPU > 80%  → ALERT
Error rate > 1% → ALERT
Response time > 2s → ALERT
```

**The limitation:** You only know about problems you *anticipated*. Monitoring answers questions you already knew to ask.

> "Is the server down?" → Monitoring can tell you.
> "Why is this specific user's checkout failing for orders over $500?" → Monitoring cannot help you.

### Observability

> "Observability tells you **why** something is broken — even for problems you never anticipated."

Observability is a property of a system. A system is **observable** if you can understand its internal state purely by examining its outputs — without needing to add new instrumentation after the fact.

**The key insight:** With true observability, you can explore your system interactively, ask arbitrary questions, and discover the root cause of unknown unknowns.

| | Monitoring | Observability |
|---|---|---|
| **Question type** | Predefined ("Is X above threshold?") | Ad-hoc ("Why did X happen?") |
| **Data type** | Metrics / Alerts | Traces + Metrics + Logs |
| **Best for** | Known failure modes | Unknown failure modes |
| **Granularity** | Aggregated | Per-request |
| **Debugging** | "Something is wrong" | "This is exactly what went wrong and why" |

---

## 📖 Lecture 1.3 — The Three Pillars of Observability

Observability is built on three types of telemetry data. Together, they give you complete visibility.

```
┌─────────────────────────────────────────────────────────┐
│              THE THREE PILLARS                          │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐   │
│  │  TRACES  │   │ METRICS  │   │      LOGS        │   │
│  │          │   │          │   │                  │   │
│  │ "What    │   │ "How     │   │ "What exactly    │   │
│  │ happened │   │ is the   │   │ happened at      │   │
│  │ to this  │   │ system   │   │ this moment?"    │   │
│  │ request?"│   │ doing?"  │   │                  │   │
│  └──────────┘   └──────────┘   └──────────────────┘   │
│                                                         │
│  Together they answer: "WHY is the system behaving     │
│  this way for this request at this time?"              │
└─────────────────────────────────────────────────────────┘
```

### Pillar 1: Traces 🔍

A **trace** is the complete journey of a single request as it travels through your distributed system.

```
Trace ID: abc123 (one checkout request)
│
├── [API Gateway]          0ms → 250ms    (250ms total)
│   ├── auth-check         0ms → 15ms     (15ms)
│   └── route-to-order     15ms → 250ms
│       │
│       ├── [Order Service]  20ms → 200ms  (180ms total)
│       │   ├── validate-cart  20ms → 35ms  (15ms)
│       │   ├── check-inventory  35ms → 100ms ← ⚠️ 65ms! SLOW
│       │   │   └── [Inventory DB query]  36ms → 99ms
│       │   └── create-order   100ms → 200ms
│       │
│       └── [Payment Service] 200ms → 250ms  (50ms)
│           └── charge-card   201ms → 249ms
```

**What traces tell you:**
- Which service is the bottleneck
- The exact sequence of operations
- How much time was spent in each step
- Which downstream calls happened

### Pillar 2: Metrics 📊

**Metrics** are numerical measurements sampled over time. They tell you the *health and performance* of your system at the aggregate level.

```
Examples:
- http_requests_total{service="order", status="200"} = 15432
- http_request_duration_seconds{p99} = 0.85
- active_database_connections = 47
- queue_depth = 312
- memory_used_bytes = 2147483648
```

**What metrics tell you:**
- System-wide trends (are things getting worse?)
- Resource utilization (do I need to scale?)
- Business KPIs (how many orders per second?)
- Alerting thresholds (SLOs and SLAs)

### Pillar 3: Logs 📝

**Logs** are time-stamped, human-readable records of discrete events.

```json
{
  "timestamp": "2026-04-12T08:15:22.341Z",
  "level": "ERROR",
  "service": "order-service",
  "trace_id": "abc123",
  "span_id": "def456",
  "message": "Inventory check timeout",
  "user_id": "user-789",
  "order_id": "ord-555",
  "timeout_ms": 5000,
  "inventory_service_url": "inventory:8080"
}
```

**What logs tell you:**
- Precise details about specific events
- Error messages and stack traces
- Business events ("Order created, OrderID=555")
- Context that enriches traces

---

## 📖 Lecture 1.4 — How the Three Pillars Work Together

The real power emerges when you *connect* them:

### The Investigative Flow

```
1. ALERT fires:
   "p99 checkout latency > 5s for 10 minutes"
          ↓ (Metrics told you something is wrong)

2. Open Grafana dashboard:
   "The spike started at 08:15 UTC. It's isolated
    to the order-service."
          ↓ (Metrics narrow down where)

3. Open Jaeger (trace explorer):
   Filter: service=order-service, duration>4s
   Find: All slow traces show slow inventory spans
          ↓ (Traces show you the bottleneck)

4. Click on a specific slow trace:
   Span: "check-inventory" took 4.8s
   Span has attribute: db.statement = "SELECT * FROM inventory WHERE..."
          ↓ (Traces show you the exact operation)

5. Open logs correlated to that trace_id:
   "ERROR: Full table scan detected. Missing index on product_id"
          ↓ (Logs give you the exact root cause)

ROOT CAUSE FOUND in < 5 minutes:
Missing database index on inventory.product_id
```

### The Critical Link: Trace Context in Logs

This investigative flow only works if all three pillars share a **common identifier** — the Trace ID:

```python
# Logs that are NOT correlated (hard to debug):
2026-04-12 08:15:22 ERROR "Inventory timeout"   ← We don't know which request!

# Logs WITH trace context (observable):
2026-04-12 08:15:22 ERROR "Inventory timeout" trace_id=abc123 span_id=def456
#                                              ↑ This links to the trace!
```

---

## 📖 Lecture 1.5 — The Observability Problem Before OpenTelemetry

Before OpenTelemetry, every vendor had their own SDK:

```
Your App
├── DataDog SDK     → DataDog only
├── New Relic SDK   → New Relic only
├── Dynatrace SDK   → Dynatrace only
└── Jaeger Client   → Jaeger only
```

**Problems:**
- **Vendor lock-in**: Switching costs were enormous
- **Multiple SDKs**: Your code was cluttered with vendor-specific code
- **Inconsistency**: Each SDK worked differently
- **Duplicate overhead**: Multiple agents running in your process
- **Format chaos**: Zipkin format, Jaeger format, W3C format, B3 format...

---

## 📖 Lecture 1.6 — Enter OpenTelemetry

**OpenTelemetry (OTel)** is a CNCF (Cloud Native Computing Foundation) project. It is:

> A **vendor-neutral, open-source** collection of APIs, SDKs, and tools to generate, collect, and export telemetry data (traces, metrics, logs).

```
                    ┌─────────────────────────────────┐
Your App            │       OpenTelemetry              │
                    │                                  │
├── OTel API ───────►  Standardized Instrumentation   │
├── OTel SDK ───────►  Consistent Collection          │
│                   │          │                       │
│                   │          ▼                       │
│                   │   OTel Collector                 │
│                   │   (Route to any backend)         │
│                   └──────────┬────────────────────── ┘
│                              │
│                   ┌──────────┼───────────┐
│                   ▼          ▼           ▼
│               DataDog    Grafana     Honeycomb
│               New Relic  Jaeger      Dynatrace
│               (any!)     (any!)      (any!)
```

**What OpenTelemetry gives you:**
- ✅ One SDK for all signals (traces, metrics, logs)
- ✅ Instrument once, export anywhere
- ✅ Supported in 11+ programming languages
- ✅ Auto-instrumentation for popular frameworks
- ✅ Industry-standard wire protocol (OTLP)
- ✅ The OpenTelemetry Collector for routing/processing telemetry

### OTel is Industry Standard (Not Vendor)

OpenTelemetry is now the **second most active CNCF project** (after Kubernetes itself). Every major vendor now supports it:
- AWS CloudWatch natively accepts OTLP
- Google Cloud Operations supports OTel
- Azure Monitor supports OTel
- Datadog, New Relic, Dynatrace all have OTel ingestion

---

## 📖 Lecture 1.7 — Key Terminology You Must Know

| Term | Definition |
|---|---|
| **Telemetry** | Data emitted by a system describing its behavior |
| **Instrumentation** | The code you add to emit telemetry |
| **Signal** | A category of telemetry: Traces, Metrics, or Logs |
| **Span** | A single unit of work within a trace (has start/end time) |
| **Trace** | A collection of spans representing one request's journey |
| **Trace ID** | A unique 128-bit identifier shared by all spans in a trace |
| **Span ID** | A unique 64-bit identifier for a single span |
| **Exporter** | Component that sends telemetry to a backend |
| **Collector** | A proxy/pipeline that receives, processes, and exports telemetry |
| **OTLP** | OpenTelemetry Protocol — the standard wire format |
| **SDK** | The OpenTelemetry implementation for your language |
| **API** | The language-agnostic interfaces (what SDK implements) |
| **Backend** | The system storing and visualizing telemetry (Jaeger, Prometheus, etc.) |

---

## ✅ Module 01 — Review Questions

1. What is the key difference between **monitoring** and **observability**?
2. Name the **three pillars** of observability and what each one is best for.
3. What problem existed *before* OpenTelemetry regarding vendor SDKs?
4. What does **CNCF** stand for, and why does it matter that OTel is a CNCF project?
5. What is a **Trace ID** and why is it important for connecting the three pillars?
6. If your p99 latency alert fires, describe the **investigative flow** using all three pillars.
7. What's the difference between a **Trace** and a **Span**?
8. What does "instrument once, export anywhere" mean in the context of OpenTelemetry?
9. A system has good metrics dashboards but no traces. What types of questions can it *not* answer?
10. What is **OTLP**?

---

## 🧪 Lab 01 — The Three Pillars in Practice

**→ See [labs/lab-01-three-pillars.md](./labs/lab-01-three-pillars.md)**

---

*[Module 02 →](../Module-02-OTel-Architecture/README.md)*
