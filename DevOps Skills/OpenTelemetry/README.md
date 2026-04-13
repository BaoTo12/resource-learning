# 🎓 OPENTELEMETRY MASTERY — University-Style Complete Course
### Professor's Edition | DevOps & Observability Engineering Program

---

> **"You cannot fix what you cannot see. OpenTelemetry is the language your systems use to tell you what is happening inside them. Master it, and you will never fly blind in production again."**
> — *Your Professor*

---

## 📋 Course Information

| Field | Details |
|---|---|
| **Course Name** | OpenTelemetry: Observability for Modern Distributed Systems |
| **Level** | Beginner → Advanced |
| **Prerequisites** | Basic programming knowledge (any language), familiarity with HTTP APIs, Docker basics helpful |
| **Total Modules** | 9 Modules + Capstone Project |
| **Format** | Lecture Notes + Hands-On Labs + Code Examples |
| **Primary Language** | Python & Java (concepts apply to all languages) |
| **Tools Required** | Docker, Docker Compose, Python 3.9+ or Java 17+, VS Code |

---

## 🗺️ Course Syllabus

| Module | Topic | Difficulty |
|---|---|---|
| [Module 01](./Module-01-Observability-Foundations/README.md) | Observability Foundations: The Three Pillars | ⭐ Beginner |
| [Module 02](./Module-02-OTel-Architecture/README.md) | OpenTelemetry Architecture & Core Concepts | ⭐ Beginner |
| [Module 03](./Module-03-Traces/README.md) | Distributed Tracing: Spans, Context & Propagation | ⭐⭐ Intermediate |
| [Module 04](./Module-04-Metrics/README.md) | Metrics: Counters, Gauges, Histograms & Views | ⭐⭐ Intermediate |
| [Module 05](./Module-05-Logs/README.md) | Structured Logging & Log Correlation | ⭐⭐ Intermediate |
| [Module 06](./Module-06-Collector/README.md) | The OpenTelemetry Collector: Pipeline Architecture | ⭐⭐⭐ Advanced |
| [Module 07](./Module-07-Auto-Instrumentation/README.md) | Auto-Instrumentation & Zero-Code Observability | ⭐⭐ Intermediate |
| [Module 08](./Module-08-Backends/README.md) | Backends: Jaeger, Prometheus, Grafana & LGTM Stack | ⭐⭐⭐ Advanced |
| [Module 09](./Module-09-Production/README.md) | Production Patterns: Sampling, Security & Scale | ⭐⭐⭐ Advanced |
| [Capstone](./Capstone-Project/README.md) | Instrument a Full Microservices System End-to-End | ⭐⭐⭐ Advanced |

---

## 🛠️ Environment Setup (Do This First!)

### Step 1 — Install Docker Desktop

OpenTelemetry backends (Jaeger, Prometheus, Grafana) run in Docker. All labs use Docker Compose.

```bash
# Download Docker Desktop from:
# https://www.docker.com/products/docker-desktop/

# Verify installation
docker --version
docker compose version
```

### Step 2 — Install Python 3.9+ (Primary lab language)

```bash
# Windows — Download from https://python.org/downloads/
# Make sure to check "Add Python to PATH" during install

# Verify
python --version     # Should be 3.9 or higher
pip --version
```

### Step 3 — Install Java 17+ (Optional — used in Module 07)

```bash
# Windows — Download from https://adoptium.net/
# Or use winget:
winget install EclipseAdoptium.Temurin.17.JDK

# Verify
java -version
```

### Step 4 — Install VS Code Extensions

- `ms-python.python` — Python support
- `redhat.vscode-yaml` — YAML editing
- `ms-azuretools.vscode-docker` — Docker integration

### Step 5 — Verify Docker is Working

```bash
# Pull images needed for the course (do this now to save time)
docker pull jaegertracing/all-in-one:latest
docker pull prom/prometheus:latest
docker pull grafana/grafana:latest
docker pull otel/opentelemetry-collector-contrib:latest

# Test a quick container
docker run --rm hello-world
```

**Expected output:**
```
Hello from Docker!
...
```

### Step 6 — Create Course Python Virtual Environment

```bash
# Navigate to your working directory
cd "C:\Users\Admin\Desktop\DevOps Skills\OpenTelemetry"

# Create a Python virtual environment
python -m venv .venv

# Activate it (Windows PowerShell)
.venv\Scripts\Activate.ps1

# Install core OpenTelemetry packages
pip install opentelemetry-api opentelemetry-sdk

# Verify
python -c "import opentelemetry; print('OTel version:', opentelemetry.__version__)"
```

---

## 📁 Course File Structure

```
OpenTelemetry/
├── README.md                                   ← You are here (Course Overview)
├── OTEL-QUICK-REFERENCE.md                    ← Daily cheat sheet
│
├── Module-01-Observability-Foundations/
│   ├── README.md                              ← Lecture: What is Observability?
│   └── labs/
│       └── lab-01-three-pillars.md
│
├── Module-02-OTel-Architecture/
│   ├── README.md                              ← Lecture: OTel Components
│   └── labs/
│       └── lab-02-first-trace.md
│
├── Module-03-Traces/
│   ├── README.md                              ← Lecture: Distributed Tracing
│   └── labs/
│       ├── lab-03-manual-tracing.md
│       └── code/
│           ├── app.py
│           └── docker-compose.yml
│
├── Module-04-Metrics/
│   ├── README.md                              ← Lecture: Metrics Instrumentation
│   └── labs/
│       ├── lab-04-metrics.md
│       └── code/
│           ├── metrics_app.py
│           └── docker-compose.yml
│
├── Module-05-Logs/
│   ├── README.md                              ← Lecture: Structured Logging
│   └── labs/
│       ├── lab-05-logs.md
│       └── code/
│           └── logging_app.py
│
├── Module-06-Collector/
│   ├── README.md                              ← Lecture: OTel Collector Architecture
│   └── labs/
│       ├── lab-06-collector.md
│       └── config/
│           ├── otel-collector-config.yaml
│           └── docker-compose.yml
│
├── Module-07-Auto-Instrumentation/
│   ├── README.md                              ← Lecture: Zero-Code Observability
│   └── labs/
│       ├── lab-07-auto-instrumentation.md
│       └── code/
│           ├── flask_app.py
│           └── spring_app/
│
├── Module-08-Backends/
│   ├── README.md                              ← Lecture: Jaeger, Prometheus, Grafana
│   └── labs/
│       ├── lab-08-full-stack.md
│       └── docker-compose-full.yml
│
├── Module-09-Production/
│   ├── README.md                              ← Lecture: Sampling, Security & Scale
│   └── labs/
│       └── lab-09-production.md
│
└── Capstone-Project/
    ├── README.md
    └── microservices/
        ├── api-gateway/
        ├── order-service/
        ├── inventory-service/
        ├── payment-service/
        └── docker-compose.yml
```

---

## 📚 How to Take This Course

1. **Read the Module README.md** — This is your lecture. Read it like a textbook chapter. Understanding concepts *before* the lab is critical.
2. **Complete the Lab** — Observability is learned by *doing*. Type the code yourself — don't copy-paste blindly.
3. **Run the Docker stacks** — Every lab provides a complete Docker Compose setup. Seeing real traces/metrics makes concepts click.
4. **Answer Review Questions** — At the end of each module. If you can't answer them, re-read the lecture.
5. **Build the Capstone** — Apply everything learned to a realistic microservices system.

---

## 🎯 Learning Outcomes

By the end of this course you will:

- ✅ Understand the difference between monitoring and observability
- ✅ Explain the three pillars of observability: Traces, Metrics, Logs
- ✅ Understand the OpenTelemetry project: SDK, API, Collector, Protocol (OTLP)
- ✅ Instrument Python and Java applications with manual OpenTelemetry SDK
- ✅ Create distributed traces with spans, attributes, events, and status
- ✅ Propagate trace context across service boundaries (HTTP, gRPC, queues)
- ✅ Instrument custom metrics: counters, gauges, histograms
- ✅ Emit structured, correlated logs (linked to traces)
- ✅ Deploy and configure the OpenTelemetry Collector with pipelines
- ✅ Visualize traces in Jaeger and metrics in Prometheus + Grafana
- ✅ Implement auto-instrumentation for frameworks (Flask, FastAPI, Spring Boot)
- ✅ Apply production patterns: sampling, batching, security, and high-volume scale
- ✅ Build a complete observability stack for a microservices application

---

## 🧭 The Big Picture: What You're Building Toward

```
Your Application (Python/Java)
         │
         │  OTLP (gRPC or HTTP)
         ▼
┌─────────────────────────┐
│  OpenTelemetry          │
│  Collector              │
│  ┌─────┐ ┌────────┐    │
│  │Recv │→│Process │    │
│  └─────┘ └──┬─────┘    │
│             ▼           │
│         ┌────────┐      │
│         │Export  │      │
│         └───┬────┘      │
└─────────────┼───────────┘
              │
    ┌─────────┼──────────┐
    ▼         ▼          ▼
 Jaeger   Prometheus  Cloud
(Traces)  (Metrics)  Vendors
              │
              ▼
           Grafana
         (Dashboards)
```

---

## 📖 Recommended Additional Reading

- [Official OpenTelemetry Docs](https://opentelemetry.io/docs/)
- [OTel Python SDK](https://opentelemetry-python.readthedocs.io/)
- [OTLP Protocol Specification](https://opentelemetry.io/docs/specs/otlp/)
- [OpenTelemetry Demo App](https://github.com/open-telemetry/opentelemetry-demo)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)

---

## ⚡ Quick-Start: Your First Trace in 5 Minutes

Want a taste before starting Module 01? Run this right now:

```bash
# Activate virtual environment
.venv\Scripts\Activate.ps1

# Install
pip install opentelemetry-api opentelemetry-sdk

# Run
python -c "
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export.in_memory_span_exporter import InMemorySpanExporter
from opentelemetry.sdk.trace.export import SimpleSpanProcessor

# Set up a tracer (in-memory for now)
exporter = InMemorySpanExporter()
provider = TracerProvider()
provider.add_span_processor(SimpleSpanProcessor(exporter))
trace.set_tracer_provider(provider)

# Create your first trace!
tracer = trace.get_tracer('my-first-tracer')

with tracer.start_as_current_span('my-first-span') as span:
    span.set_attribute('greeting', 'Hello, OpenTelemetry!')
    print('🔭 Span is running...')

# See the span!
spans = exporter.get_finished_spans()
print(f'✅ Recorded {len(spans)} span(s)')
print(f'   Span name: {spans[0].name}')
print(f'   Trace ID:  {format(spans[0].context.trace_id, \"032x\")}')
print(f'   Span ID:   {format(spans[0].context.span_id, \"016x\")}')
"
```

You just created your first trace! Now let's understand what that means. **Start with [Module 01 →](./Module-01-Observability-Foundations/README.md)**

---

*Start with [Module 01 →](./Module-01-Observability-Foundations/README.md)*
