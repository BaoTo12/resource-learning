# Module 08 — Backends: Jaeger, Prometheus, Grafana & LGTM Stack

> **Professor's Note:** OpenTelemetry generates data, but backends make it useful. In this module, I explain how different observability backends fit together. You need to know the landscape: open-source stacks vs. managed SaaS vendors.

---

## 📖 Lecture 8.1 — The Telemetry Pipeline Again

```
[ App ] ---OTLP---> [ Collector ] ----> Backend A (Traces)
                                  ----+ Backend B (Metrics)
                                  ----+ Backend C (Logs)
```

OpenTelemetry aims to standardize the left side (instrumentation and collection) so you can easily swap components on the right side (backends).

---

## 📖 Lecture 8.2 — The Leading Open-Source Backends

If you are running observability on-premises or want to avoid vendor lock-in, these are the standard tools:

### For Traces:
1. **Jaeger (CNCF):** Very popular, excellent UI for visualizing single traces. Harder to do complex trace analytics over long periods.
2. **Tempo (Grafana Labs):** High scale, stores traces in object storage (S3). Integrates deeply with Grafana dashboards.
3. **Zipkin:** The older predecessor, still widely used but Jaeger/Tempo are taking over.

### For Metrics:
1. **Prometheus (CNCF):** The undisputed king of metrics in Kubernetes environments. Uses a pull model (scraping). Understands PromQL.
2. **Mimir (Grafana Labs):** Highly scalable, multi-tenant Prometheus alternative. Solves the long-term storage problems of raw Prometheus.
3. **VictoriaMetrics:** Another extremely fast, scalable Prometheus alternative.

### For Logs:
1. **Loki (Grafana Labs):** "Like Prometheus, but for logs." Highly efficient because it only indexes labels, not the full text.
2. **Elasticsearch / ELK Stack:** Very powerful text searching, but resource-heavy to maintain at scale.

### For Dashboards:
1. **Grafana:** The absolute standard for viewing metrics, searching logs, and finding traces all in one unified UI.

---

## 📖 Lecture 8.3 — The LGTM Stack

The **LGTM** stack (Loki, Grafana, Tempo, Mimir) provided by Grafana Labs is currently the most cohesive open-source observability stack.

Why is LGTM so popular? Because it provides **flawless correlation**.
- You can put a Grafana dashboard showing Mimir metrics next to a Loki log stream.
- From a log line in Loki, you can click a button to jump immediately to the Tempo trace.
- From a trace span, you can jump directly to the metrics for the pod that executed that span.

When you configure the OTel Collector to push to LGTM, you export:
- Metrics to Mimir (via Prometheus remote-write exporter)
- Traces to Tempo (via OTLP exporter)
- Logs to Loki (via Loki exporter)

---

## 📖 Lecture 8.4 — Commercial / SaaS Backends

Managing Prometheus, Jaeger, and Elasticsearch at scale is a full-time job for a platform team. Many companies prefer to pay SaaS vendors to host the backends.

The major players (Datadog, New Relic, Dynatrace, Honeycomb) all now natively ingest OTLP!

**How to migrate from Jaeger to Datadog with OTel:**

Change *nothing* in your application. Just update the Collector configuration:

```yaml
# old-config.yaml
exporters:
  otlp/jaeger:
    endpoint: "jaeger:4317"
```

```yaml
# new-config.yaml
exporters:
  datadog:
    api:
      key: ${DD_API_KEY}
```

This is the promise of OpenTelemetry fulfilled: preventing vendor lock-in.

---

## 📖 Lecture 8.5 — Exemplars (Connecting Metrics to Traces)

**Exemplars** are specific trace IDs attached to metric data points.

Imagine a Grafana chart showing a spike in `p99 latency`. An exemplar appears as a tiny dot ON that spike in the chart. When you hover over the dot, it shows a specific Trace ID that contributed to that spike. You click the dot, and it jumps straight to the Jaeger/Tempo trace.

This is the "holy grail" debugging flow: Metrics show the problem → trace shows the cause.

OpenTelemetry supports exemplars natively when exporting to Prometheus!

---

## ✅ Module 08 — Review Questions

1. Name the dominant open-source backends for Traces, Metrics, and Logs.
2. What does the acronym LGTM stand for in observability?
3. If you switch from Jaeger (local) to Honeycomb (SaaS), what changes in your application code?
4. What is an **exemplar** and why is it useful?
5. Why might a large company choose to use a SaaS vendor like Datadog instead of hosting the LGTM stack themselves?

---

## 🧪 Lab 08 — Building the Full Local Stack

**→ See [labs/lab-08-full-stack.md](./labs/lab-08-full-stack.md)**

---

*[← Module 07](../Module-07-Auto-Instrumentation/README.md) | [Module 09 →](../Module-09-Production/README.md)*
