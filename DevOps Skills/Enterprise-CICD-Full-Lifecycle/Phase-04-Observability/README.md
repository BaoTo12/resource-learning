# 👁️ PHASE 4: FULL STACK OBSERVABILITY
### Monitoring Your Platform | OpenTelemetry + Metrics + Traces

---

## 🎯 Objective
A platform is "flying blind" without observability. We will instrument the **Reservation Platform** with **OpenTelemetry (OTel)** to gain visibility into every request, error, and database query.

---

## 🏗️ 1. App Instrumentation (Auto-Instrumentation)

For most languages (Node.js, Java, Python), you don't need to change much code. Use the **OpenTelemetry SDK**.

### Node.js Example:
Create `tracing.js` in your source code root:

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector:4318/v1/traces', # Send to Collector
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

---

## 📦 2. Deploying the OTel Collector (Free)

The Collector receives data from your app and "exports" it to your chosen backend (Jaeger, Prometheus, etc.).

Add this to your Helm Chart's `templates/otel-collector.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-conf
data:
  otel-collector-config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    exporters:
      logging:
        loglevel: debug
      jaeger:
        endpoint: "jaeger-all-in-one:14250"
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [jaeger, logging]
```

---

## 📊 3. Visualizing (Jaeger & Grafana - Free)

1. **Jaeger**: Deploy the Jaeger "all-in-one" image for free distributed tracing.
2. **Prometheus**: For metrics (RAM, CPU, HTTP Error rates).
3. **Grafana**: The dashboard to see it all.

### Key Metrics to Track for Reservations:
- **P99 Latency**: How long do users wait for a reservation?
- **Error Rate**: Are we losing customers due to 500 errors?
- **Throughput**: How many reservations per second can we handle?

---

## 🏁 Final Step: The Complete Circle

Now your lifecycle is complete:
1. **Commit Code** → CI Builds & Scans (Phase 1).
2. **Push Image** → Version Bump in GitOps (Phase 2).
3. **Sync to Cluster** → ArgoCD Deploys (Phase 3).
4. **Monitor Health** → OpenTelemetry Observes (Phase 4).

---

## ✅ Checklist for Phase 4
1. [ ] App is sending traces to the OTel Collector.
2. [ ] Collector is exporting to Jaeger/Prometheus.
3. [ ] You can see a full "Trace" from a user request down to the database query.

---

### Congratulations! You have built a Modern Enterprise CI/CD Lifecycle.
