# Lab 07 — Zero-Code Python Observability

> **Objective:** Take a plain, uninstrumented Flask application that makes HTTP requests and queries a SQLite database. Without modifying a single line of business code, run it with the OpenTelemetry CLI and see full distributed traces appear in Jaeger.

---

## ⏱️ Estimated Time: 30 minutes

---

## Part 1 — The Plain Application

We'll create a completely clean API. Notice there are **no OpenTelemetry imports** in this file.

Create `Module-07-Auto-Instrumentation/labs/code/plain_app.py`:

```python
"""
plain_app.py — A standard Flask app with SQLite and requests.
Notice there is ZERO OpenTelemetry code here. None.
"""

import sqlite3
import requests
import time
from flask import Flask, jsonify

app = Flask(__name__)

# --- Setup Database ---
def init_db():
    conn = sqlite3.connect("users.db")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY,
            name TEXT
        )
    """)
    cursor.execute("INSERT OR IGNORE INTO users (id, name) VALUES (1, 'Alice')")
    cursor.execute("INSERT OR IGNORE INTO users (id, name) VALUES (2, 'Bob')")
    conn.commit()
    conn.close()

init_db()


# --- Routes ---
@app.route("/")
def index():
    return "Hello! Try /users or /external"


@app.route("/users")
def get_users():
    """Fetches users from the database"""
    time.sleep(0.1)  # Simulate logic
    
    conn = sqlite3.connect("users.db")
    cursor = conn.cursor()
    
    # This query will be automatically traced!
    cursor.execute("SELECT id, name FROM users")
    users = [{"id": row[0], "name": row[1]} for row in cursor.fetchall()]
    conn.close()
    
    return jsonify(users)


@app.route("/external")
def call_external():
    """Makes an HTTP call to an external API"""
    
    # This outgoing request will be automatically traced!
    response = requests.get("https://httpbin.org/delay/1")
    data = response.json()
    
    return jsonify({
        "status": "success",
        "external_response_origin": data.get("origin")
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=False)
```

---

## Part 2 — Setup Jaeger

Just like previous labs, start a Jaeger container:

```bash
docker run -d --name jaeger-lab07 -p 16686:16686 -p 4317:4317 -e COLLECTOR_OTLP_ENABLED=true jaegertracing/all-in-one:latest
```

---

## Part 3 — The Auto-Instrumentation Magic

### Step 3.1 — Install the libraries

First, install the app's standard dependencies:
```bash
pip install flask requests
```

Next, install the core OpenTelemetry auto-instrumentation tools:
```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
```

### Step 3.2 — Bootstrap

This command inspects your installed packages (Flask, requests, etc.) and automatically installs the specific OpenTelemetry instrumentation libraries for them (e.g., `opentelemetry-instrumentation-flask`, `opentelemetry-instrumentation-requests`, `opentelemetry-instrumentation-sqlite3`).

```bash
opentelemetry-bootstrap -a install
```

### Step 3.3 — Run the App

Instead of `python plain_app.py`, use the wrapper:

**Windows PowerShell:**
```powershell
$env:OTEL_SERVICE_NAME="auto-flask-api"
$env:OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
$env:OTEL_TRACES_EXPORTER="otlp"
$env:OTEL_METRICS_EXPORTER="none"  # Just doing traces for now

opentelemetry-instrument python Module-07-Auto-Instrumentation/labs/code/plain_app.py
```

**Git Bash / WSL / Linux:**
```bash
OTEL_SERVICE_NAME="auto-flask-api" \
OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317" \
OTEL_TRACES_EXPORTER="otlp" \
OTEL_METRICS_EXPORTER="none" \
opentelemetry-instrument python plain_app.py
```

---

## Part 4 — Generate Traffic and View Traces

While the app is running, send some requests:

```bash
curl http://localhost:5000/users
curl http://localhost:5000/external
```

Now open Jaeger at **http://localhost:16686**. Search for the `auto-flask-api` service.

### What to Notice in Jaeger:

1. **The `/users` trace:**
   - Look at the child span representing the database query.
   - Look at the attributes on that span — it captured the exact SQL string (`db.statement` = `SELECT id, name FROM users`), the database system name (`db.system` = `sqlite`), and the latency.
   - Notice that the parent span representing the Flask route has all the standard HTTP semantic attributes (`http.method`, `http.url`).

2. **The `/external` trace:**
   - Look at the child span representing the outgoing request.
   - It automatically extracted the HTTP method (`GET`), URL (`https://httpbin.org/..`), and the resulting status code from the `requests` library.

**All of this happened without modifying `plain_app.py`!**

---

## ✅ Lab 07 Completion Checklist

- [ ] Created `plain_app.py` with no OTel code
- [ ] Ran `opentelemetry-bootstrap` to install wrappers
- [ ] Started the app via `opentelemetry-instrument`
- [ ] Saw traces with SQL statements in Jaeger
- [ ] Saw traces with external HTTP calls in Jaeger

---

## Cleanup

```bash
docker stop jaeger-lab07 && docker rm jaeger-lab07
```

---

*[← Back to Module 07](../README.md) | [Module 08 →](../../Module-08-Backends/README.md)*
