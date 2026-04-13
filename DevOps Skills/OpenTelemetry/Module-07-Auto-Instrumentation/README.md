# Module 07 — Auto-Instrumentation & Zero-Code Observability

> **Professor's Note:** Up until now, we've been writing manual `tracer.start_as_current_span()` code. While manual instrumentation gives you the most control, telling developers to manually trace every HTTP endpoint and database query is a non-starter. This module covers **Auto-Instrumentation** — how to get 80% of your observability with 0 lines of code.

---

## 📖 Lecture 7.1 — What is Auto-Instrumentation?

**Auto-instrumentation** is the process of modifying an application at runtime (or load time) to automatically inject telemetry code into standard libraries and frameworks.

**What you get for free:**
- **HTTP Servers** (e.g., Flask, Django, Spring Boot): A semantic span for every incoming request.
- **HTTP Clients** (e.g., `requests`, `urllib`, `HttpClient`): A span for every outgoing API call, with automatic context injection.
- **Databases** (e.g., SQLAlchemy, psycopg2, JDBC): A span for every DB query, capturing the SQL statement and latency.
- **Message Queues** (e.g., Kafka, RabbitMQ): Spans for producing and consuming messages.

---

## 📖 Lecture 7.2 — Python Auto-Instrumentation

Python uses **monkey-patching** for auto-instrumentation. The OTel tool wraps standard library functions (like `requests.get`) with tracing logic.

### Option 1: The Zero-Code Wrapper (Recommended)

You don't change your code at all. You run your app using the `opentelemetry-instrument` CLI command.

```bash
# 1. Install the CLI
pip install opentelemetry-distro

# 2. Automatically install packages for frameworks detected in your project
opentelemetry-bootstrap -a install

# 3. Run your app via the wrapper
export OTEL_SERVICE_NAME="my-api"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"

opentelemetry-instrument \
    --traces_exporter otlp \
    --metrics_exporter otlp \
    python my_flask_app.py
```

**How it works:** Before `my_flask_app.py` runs, the CLI loads the OTel SDK, configures the provider using environment variables, and patches libraries like Flask and requests.

### Option 2: Programmatic Auto-Instrumentation

You call the instrumentation libraries explicitly in your `main.py`. This is useful if you need to pass specific config flags to the instrumentors.

```python
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
import flask

app = flask.Flask(__name__)

# Must call these at startup
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

# Now all Flask routes and 'requests.get()' are automatically traced
```

---

## 📖 Lecture 7.3 — Java Auto-Instrumentation

Java has the most powerful auto-instrumentation via the **Java Agent**. It modifies bytecode as classes are loaded into the JVM. Literally zero code changes are required.

```bash
# 1. Download the agent jar
wget https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

# 2. Run your app, attaching the agent
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.service.name=my-spring-app \
     -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
     -jar my-app.jar
```

The Java agent will automatically trace Spring Boot, Hibernate/JDBC, Apache HttpClient, Kafka, gRPC, and dozens of others.

---

## 📖 Lecture 7.4 — Hybrid Instrumentation (Combining Auto and Manual)

Auto-instrumentation is fantastic, but it only knows about infrastructure boundaries (HTTP in, DB out, HTTP out). It **doesn't know your business logic**.

The best approach is **Hybrid**:
1. Use auto-instrumentation for the "edges" (HTTP, DB, Queues).
2. Use manual instrumentation for key business operations inside the route.

```python
# The Flask route is traced automatically by auto-instrumentation
@app.route("/checkout")
def checkout():
    # Retrieve the auto-created span to add business context
    current_span = trace.get_current_span()
    current_span.set_attribute("user.tier", "premium")
    
    # Add a manual child span for specific business logic
    with tracer.start_as_current_span("calculate-complex-discount") as span:
        # ... logic ...
    
    # The database call is traced automatically by SQLAlchemy auto-instrumentor
    db.session.add(order)
    db.session.commit()
```

---

## 📖 Lecture 7.5 — Environment Variable Configuration

Notice that auto-instrumentation relies heavily on environment variables standard to OpenTelemetry.

**Key OTel Environment Variables:**
| Variable | Description |
|---|---|
| `OTEL_SERVICE_NAME` | The name of your service |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Where to send data (e.g., `http://localhost:4317`) |
| `OTEL_TRACES_EXPORTER` | Which exporter to use (`otlp`, `console`, `none`) |
| `OTEL_METRICS_EXPORTER` | Which exporter to use (`otlp`, `console`, `none`) |
| `OTEL_LOGS_EXPORTER` | Which exporter to use (`otlp`, `console`, `none`) |
| `OTEL_RESOURCE_ATTRIBUTES` | Comma-separated tags e.g. `env=prod,team=backend` |

---

## ✅ Module 07 — Review Questions

1. What does the `opentelemetry-instrument` CLI command do in Python?
2. Why is auto-instrumentation considered "zero code"?
3. Name 3 types of libraries that auto-instrumentation typically targets.
4. Auto-instrumentation gives you a span for `POST /users` and a span for `INSERT INTO users`. What does it *not* give you?
5. How do you configure the endpoint when using the Java agent or Python CLI?

---

## 🧪 Lab 07 — Zero-Code Python Observability

**→ See [labs/lab-07-auto-instrumentation.md](./labs/lab-07-auto-instrumentation.md)**

---

*[← Module 06](../Module-06-Collector/README.md) | [Module 08 →](../Module-08-Backends/README.md)*
