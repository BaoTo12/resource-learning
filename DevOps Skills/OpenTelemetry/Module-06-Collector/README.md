# Module 06 — The OpenTelemetry Collector Architecture

> **Professor's Note:** The OTel Collector is the "swiss army knife" of observability. It sits between your applications and your backend vendors. Understanding its pipeline architecture (Receivers, Processors, Exporters) is mandatory for any production deployment.

---

## 📖 Lecture 6.1 — Why We Need the Collector

Without the Collector, your application SDK must do everything:
- Know the credentials for Datadog, Honeycomb, Prometheus...
- Format data perfectly for each backend
- Batch and retry failed exports (uses app memory!)
- Filter out PII or spammy traces

**With the Collector:**
Your app does **one** thing: Push OTLP to `localhost:4317` (the Collector).
The Collector handles all the heavy lifting.

```
       Your App (Python)               Your App (Java)
                │                               │
                └───► OTLP (port 4317) ◄────────┘
                            │
               ┌────────────▼──────────────┐
               │  OpenTelemetry Collector  │
               │                           │
               │  • Receivers              │
               │  • Processors             │
               │  • Exporters              │
               └────────────┬──────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
     Prometheus          Datadog           AWS X-Ray
```

---

## 📖 Lecture 6.2 — The Pipeline Architecture

Every Collector configuration is built around **Pipelines**. A pipeline has three stages:

### 1. Receivers (How data gets in)
- Accepts data in various formats.
- Ex: OTLP Receiver (gRPC/HTTP), Jaeger Receiver, Prometheus Scraper (pulls metrics!), Kafka Receiver.

### 2. Processors (What happens to data inside)
- Modifies, filters, or batches the data.
- Ex: Batch Processor (groups data), Memory Limiter (prevents crashes), Attributes Processor (adds/removes tags, masks PII).

### 3. Exporters (How data gets out)
- Sends data to backends in their required formats.
- Ex: OTLP Exporter, Prometheus Exporter, Datadog Exporter, Logging Exporter (prints to stdout for debugging).

---

## 📖 Lecture 6.3 — Deconstructing a `config.yaml`

A Collector configuration file defines the components, then chains them together in pipelines.

```yaml
# 1. DEFINE COMPONENTS FIRST

receivers:                     # Define how to receive data
  otlp:                        # Use the OTLP receiver plugin
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317 # Listen on port 4317 for gRPC

processors:                    # Define how to process data
  batch:                       # Use the Batch processor plugin
    send_batch_size: 10000     # Send chunks of 10k spans
    timeout: 10s               # Or every 10 seconds (whichever is first)
    
  memory_limiter:              # CRITICAL for production
    limit_mib: 1024            # Drop data if collector uses > 1GB RAM

exporters:                     # Define where to send data
  logging:                     # Just prints to stdout (debug)
    verbosity: detailed
    
  otlp/jaeger:                 # Send to Jaeger via OTLP (custom name 'otlp/jaeger')
    endpoint: jaeger:4317
    tls:
      insecure: true


# 2. CONNECT THEM IN PIPELINES

service:
  pipelines:
    traces:                    # Pipeline for trace data
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [logging, otlp/jaeger]
      
    metrics:                   # Pipeline for metric data
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [logging]     # Currently just logging metrics
```

---

## 📖 Lecture 6.4 — Essential Processors for Production

You should *always* run these processors in your Collector:

### 1. The memory_limiter processor
Prevents the Collector from running out of memory and crashing if the backend goes down (and data starts queuing).

```yaml
processors:
  memory_limiter:
    check_interval: 1s         # Check RAM usage every second
    limit_mib: 4000            # Hard limit: drop data if we hit 4GB
    spike_limit_mib: 800       # Start dropping early if RAM spikes fast
```

### 2. The batch processor
Batches telemetry before exporting. Greatly reduces network overhead and backend load.

```yaml
processors:
  batch:
    send_batch_size: 8192      # Send when we have 8k items
    timeout: 5s                # Force send every 5s even if not full
```

### 3. The attributes processor
Crucial for security (hiding PII) or enriching data.

```yaml
processors:
  attributes/masking:
    actions:
      # Hash email attributes for privacy
      - key: user.email
        action: hash
        
      # Add environment tag to everything
      - key: environment
        value: production
        action: insert
```

---

## 📖 Lecture 6.5 — Collector Deployment Patterns

### Pattern 1: Agent (Sidecar / DaemonSet)
The Collector runs on the *same machine/node* as the application.
- **Pros:** App sends to `localhost` (fast, no DNS needed). App credentials not needed on the app itself.
- **Cons:** Harder to manage stateful processing (like tail sampling).

```
[ App Container ] → localhost:4317 → [ Collector Agent Container ]
                                              │
                                             WAN
                                              ▼
                                         [ Datadog ]
```

### Pattern 2: Gateway (Standalone Cluster)
A dedicated cluster of Collectors behind a load balancer.
- **Pros:** Centralized credential management. Perfect for tail sampling (seeing the whole trace before deciding to keep it). Can scale independently.
- **Cons:** Apps must route traffic over network to the Gateway.

```
App 1 ─┐
App 2 ─┼─► Load Balancer ──► Collector Cluster ──► Backends
App 3 ─┘
```

### Pattern 3: Hybrid (Agent + Gateway)
The recommended enterprise architecture. Agent on localhost sends to Gateway, Gateway sends to vendors.

---

## ✅ Module 06 — Review Questions

1. Name the three stages of a Collector pipeline.
2. Why is it better to send telemetry to a local Collector instead of directly to Datadog/Jaeger?
3. What does the `batch` processor do, and why is it important?
4. What happens if the backend goes down and the Collector's memory gets full, assuming the `memory_limiter` processor is configured?
5. How do you distinguish between two exporters of the same type in the config (e.g., two OTLP exporters pointing to different vendors)?
6. Explain the difference between the 'Agent' deployment pattern and the 'Gateway' deployment pattern.

---

## 🧪 Lab 06 — Building a Collector Pipeline

**→ See [labs/lab-06-collector.md](./labs/lab-06-collector.md)**

---

*[← Module 05](../Module-05-Logs/README.md) | [Module 07 →](../Module-07-Auto-Instrumentation/README.md)*
