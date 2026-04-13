# 🎓 Capstone Project: End-to-End Observable Microservices

> **Professor's Note:** You've reached the final test. This project requires no code writing from you—rather, you will architect the observability strategy for a complex system. You will apply everything you've learned about correlation, auto-instrumentation, metrics, and collector pipelines.

---

## 🎯 Project Scenario

You have been hired as the Lead Observability Engineer for "ShopO11y", an e-commerce platform. The company is migrating from a monolith to microservices. 

The current state is a mess. When customers check out, the request goes through an API Gateway, an Order Service, an Inventory Service, and a Payment Service. The CEO says:
> *"The site is slow during flash sales, and we have no idea why. Engineering blames the database, DevOps blames the network. Fix it."*

---

## 🏗️ The Architecture

```
                                  [LGTM Stack]
                                    ▲
[Frontend (Next.js)]                │ (OTLP)
        │                           │
        ▼                     ┌─────┴───────┐
[API Gateway (Python)] ───────► OTel Gateway │
        │                     └─────────────┘
        │                             ▲
        ▼                             │ (OTLP Push)
[Order Service (Java)]  ──────────────┤
        │                             │
    ┌───┴───┐                         │
    ▼       ▼                         │
[Inventory] [Payment]   ──────────────┘
(Python)    (Node.js)
```

---

## 📝 The Requirements

You must define the observability architecture by writing three key components:

### 1. The Strategy Document (`strategy.md`)
Define the semantic conventions your engineers must use, which metric instruments apply to which components, and the overall context propagation strategy.

### 2. The Collector Configuration (`collector-production.yaml`)
Write a complete, production-ready OTel Collector configuration. It must include:
- Both gRPC and HTTP receivers
- A Memory Limiter processor
- A Batch processor
- An Attributes processor that hashes the `user_email` field to meet GDPR compliance
- A Tail Sampling processor that keeps 100% of errors and 5% of successes
- Exporters for Jaeger (traces) and Prometheus (metrics)

### 3. The Implementation Plan
Outline precisely *how* you will get observability into the Java and Python services without stopping production (hint: auto-instrumentation).

---

## 🚀 How to Complete This Capstone

Create a folder named `Capstone-Solution`. In it, create your `strategy.md` and `collector-production.yaml`.

*When you are ready, review the hint below to check your work!*

<details>
<summary>💡 Professor's Solution Checklist (Don't peek until you're done!)</summary>

**For the Collector Config:**
- Did you put `tail_sampling` BEFORE `batch` in the pipeline? (Crucial: tail sampler needs entire traces, batching them first breaks it).
- Did you include `limit_mib` in `memory_limiter`?
- Did you use the `attributes/masking` processor with `action: hash`?

**For the Strategy:**
- Did you specify `W3C TraceContext` for propagation?
- Did you recommend Java Agent for the Order Service and `opentelemetry-instrument` wrapper for the Python services?
- Did you define metrics like `Counter` for request rate and `Histogram` for latency (RED method)?

</details>

---

# 🎉 Congratulations!

If you successfully completed this capstone, you are no longer a beginner. You possess the skills to architect and deploy world-class observability systems.

You understand that observability isn't just "more dashboards"—it's the ability to ask arbitrary questions of a running system and get answers via traces, metrics, and logs.

**Now, go make the dark systems visible!**
