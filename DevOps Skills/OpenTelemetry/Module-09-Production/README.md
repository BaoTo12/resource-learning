# Module 09 — Production Patterns: Sampling & Scale

> **Professor's Note:** You don't realize how much data OpenTelemetry generates until you deploy it to production. A busy microservice architecture can generate terabytes of trace data per day. This module covers how to control costs and scale your observability infrastructure without losing critical insights.

---

## 📖 Lecture 9.1 — The Cost of Observability

When you instrument an application, you pay three costs:

1. **Application Overhead:** Generating Spans and Metrics takes CPU and memory in your application. (Usually negligible: < 2% CPU).
2. **Network Bandwidth:** Shipping all that telemetry out of your data center to a vendor or centralized cluster.
3. **Storage & Licensing Cost:** The backend (Datadog, Honeycomb, or your own Jaeger cluster) has to store and index all of it. **This is the expensive part.**

**The Problem:** Sending 100% of traces for a high-traffic service (e.g., 10,000 req/sec) is financially ruinous and technically unnecessary.

```
10,000 req/sec × 10 spans/req × 500 bytes/span = 50 MB/sec
= 3 GB/minute
= 4.3 TB/day OF JUST TRACES!
```

---

## 📖 Lecture 9.2 — What is Sampling?

**Sampling** is the practice of selectively recording or exporting only a subset of your traces, while throwing the rest away.

"But won't I miss important data?"

Think about a healthy `/health` check endpoint called 100 times a second. You don't need 100 identical traces of a successful health check every second. You just need to know it's working.

**The Goal of Sampling:**
Keep 100% of the *interesting* traces (errors, unusually slow requests, rare operations).
Keep only a small percentage (1-5%) of the *uninteresting* traces (fast, successful, common operations) to maintain a statistical baseline.

---

## 📖 Lecture 9.3 — Head-Based Sampling

Head-based sampling makes the decision to keep or drop a trace at the **very beginning of the request** (the "head").

**How it works:**
1. Request arrives at the API Gateway.
2. The OTel SDK rolls a dice (e.g., 10% chance to win).
3. If it wins, it creates a Trace ID and sets the `sampled` flag to `true`.
4. It passes that `sampled=true` flag downstream in the `traceparent` header.
5. All downstream services see `sampled=true` and record their spans.
6. If the Gateway rolled a loss, the flag is `sampled=false`. Downstream services don't bother creating spans.

**Pros:**
- extremely efficient. Dropped traces use exactly 0 CPU, 0 network, and 0 storage.
- Very easy to configure in the SDK.

**Cons:**
- You make the decision *before* you know if the request will fail!
- If an error happens deep in the stack on a non-sampled trace, you lose the trace.

```python
# Configuring Head-Based Sampling in the Python SDK
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased, ParentBased

# Sample 10% of root requests
sampler = TraceIdRatioBased(0.1)

# The ParentBased wrapper means:
# "If my parent was sampled, I will be sampled. If my parent was dropped, I will drop."
# (Crucial for distributed tracing!)
root_sampler = ParentBased(root=sampler)

provider = TracerProvider(sampler=root_sampler)
```

---

## 📖 Lecture 9.4 — Tail-Based Sampling

Tail-based sampling makes the decision to keep or drop a trace at the **end of the request** (the "tail").

**How it works:**
1. Every service records 100% of spans.
2. The Collector receives all spans and holds them in memory until the whole trace finishes.
3. The Collector evaluates the complete trace:
   - "Did any span have `status=ERROR`?" → KEEP IT
   - "Did the trace take longer than 2 seconds?" → KEEP IT
   - "Was it a fast, successful `/health` check?" → DROP 99% OF THEM
4. Only the kept traces are exported to the backend.

**Pros:**
- You **never** miss an error trace! The ultimate debugability.

**Cons:**
- Expensive. Your apps still spend CPU generating 100% of traces, and network sending them to the Collector.
- The Collector requires massive amounts of RAM to hold traces in memory while waiting for them to finish.
- Requires complex Collector deployments (Gateway pattern).

```yaml
# Tail-Based Sampling in the OTel Collector
processors:
  tail_sampling:
    decision_wait: 10s       # Wait 10s for the trace to finish
    num_traces: 50000        # Keep up to 50k traces in memory at once
    expected_new_traces_per_sec: 1000
    policies:
      # 1. ALWAYS keep errors
      - name: keep_errors
        type: status_code
        status_code: {status_codes: [ERROR]}
        
      # 2. ALWAYS keep slow traces (>1 second)
      - name: keep_slow
        type: latency
        latency: {threshold_ms: 1000}
        
      # 3. Randomly keep 5% of everything else
      - name: sample_baseline
        type: probabilistic
        probabilistic: {sampling_percentage: 5}
```

### Recommendation for Production:
Start with **Head-Based Sampling** (e.g., 5-10%) to get costs under control easily.
Move to **Tail-Based Sampling** when your team's observability maturity grows and you demand zero missed errors.

---

## 📖 Lecture 9.5 — High Availability Collectors

When you move OTel into production, the Collector becomes a critical piece of infrastructure. If it goes down, you are flying blind.

**Production Collector Rules:**
1. **Never run just one.** Always deploy a cluster (usually a Kubernetes Deployment) behind a Load Balancer.
2. **Use the Memory Limiter Processor.** Without it, a traffic spike will OOMKill your collector.
3. **Use the Batch Processor.** Send fewer, larger payloads to reduce network overhead.
4. **Monitor the Monitor.** The Collector exposes its own Prometheus metrics on port 8888. Alert on `otelcol_processor_dropped_spans`.

---

## ✅ Module 09 — Review Questions

1. Explain the difference between **Head-Based Sampling** and **Tail-Based Sampling**.
2. Why must a head-based sampler use the `ParentBased` wrapper? What happens if it doesn't?
3. Which sampling method ensures you capture 100% of error traces? What is the main drawback of that method?
4. What happens if a Tail-Based Sampling collector runs out of memory?
5. You have a background job that runs 1,000 times a minute. It almost never fails. Which sampling approach would you use to save costs?

---

## 🧪 Lab 09 — Configuring Tail-Based Sampling

**→ See [labs/lab-09-production.md](./labs/lab-09-production.md)**

---

*[← Module 08](../Module-08-Backends/README.md) | [Capstone Project →](../Capstone-Project/README.md)*
