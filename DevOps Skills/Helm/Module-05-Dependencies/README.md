# Module 05 — Chart Dependencies & Subchart Composition

> **Professor's Note:** Real-world applications are not monoliths. A backend service needs a database. A monitoring stack needs Prometheus and Grafana. Helm handles this with **Dependencies** — composing multiple charts together into one deployable unit. This module teaches you chart composition the professional way.

---

## 📖 Lecture 5.1 — What Are Chart Dependencies?

A Chart Dependency is another Helm chart that yours depends on.

**Without Dependencies (the bad way):**
```bash
# You'd have to install each component separately:
helm install my-db bitnami/postgresql
helm install my-cache bitnami/redis
helm install my-app ./my-webapp --set db.host=my-db-postgresql
# Problems: multiple commands, ordering matters, values scattered
```

**With Dependencies (the good way):**
```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
```

```bash
# Download dependencies
helm dependency update ./my-webapp

# Install EVERYTHING in one command
helm install my-app ./my-webapp
# Deploys: postgres + redis + your webapp — in correct order!
```

---

## 📖 Lecture 5.2 — Declaring Dependencies in Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: my-platform
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  # Mandatory fields:
  - name: postgresql        # Chart name (from the repository)
    version: "12.5.6"       # Exact version OR range
    repository: "https://charts.bitnami.com/bitnami"

  # Alias — useful when using same chart multiple times
  - name: postgresql
    version: "12.5.6"
    repository: "https://charts.bitnami.com/bitnami"
    alias: read-replica-db  # deploy as a second separate instance

  # Condition — include/exclude based on a value
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled  # disabled if .Values.redis.enabled = false

  # Tags — group charts and enable/disable groups
  - name: prometheus
    version: "23.x.x"
    repository: https://prometheus-community.github.io/helm-charts
    tags:
      - monitoring

  - name: grafana
    version: "6.x.x"
    repository: https://grafana.github.io/helm-charts
    tags:
      - monitoring

  # Local chart as dependency (relative path, no repository)
  - name: my-shared-lib
    version: "0.1.0"
    repository: "file://../my-shared-lib"
```

### Version Ranges (Semantic Versioning)

```yaml
version: "12.5.6"    # Exact
version: "^12.5.6"   # >= 12.5.6, < 13.0.0 (compatible)
version: "~12.5.6"   # >= 12.5.6, < 12.6.0 (patch range)
version: "12.x.x"    # Any 12.x.x
version: ">= 12.0.0" # At least 12.0.0
version: "*"         # Any version (dangerous!)
```

---

## 📖 Lecture 5.3 — The Dependency Workflow

```bash
# Step 1: Declare dependencies in Chart.yaml
# (edit Chart.yaml)

# Step 2: Download dependencies to charts/ directory
helm dependency update ./my-platform
# Creates: Chart.lock + charts/*.tgz files

# Step 3: Verify dependencies were fetched
ls ./my-platform/charts/
# postgresql-12.5.6.tgz
# redis-17.x.x.tgz

# Step 4: Install (dependencies auto-installed first)
helm install my-platform ./my-platform

# To update to newer dependency versions:
helm dependency update ./my-platform
# Updates Chart.lock with new resolved versions

# To build from Chart.lock (reproducible builds):
helm dependency build ./my-platform
```

### Chart.lock — Pinned Versions

After `helm dependency update`, a `Chart.lock` is created:

```yaml
# Chart.lock (auto-generated — DO NOT manually edit)
dependencies:
- name: postgresql
  repository: https://charts.bitnami.com/bitnami
  version: 12.5.6  # Exact version resolved from range
- name: redis
  repository: https://charts.bitnami.com/bitnami
  version: 17.11.3
digest: sha256:abc123...
generated: "2024-01-15T10:30:00.000000+00:00"
```

> **Best Practice:** Commit BOTH `Chart.yaml` and `Chart.lock` to version control. This ensures reproducible builds — `helm dependency build` uses the lock file.

---

## 📖 Lecture 5.4 — Configuring Subchart Values

Subchart values are configured in the **parent chart's `values.yaml`** using the dependency's alias (or name) as a key:

```yaml
# my-platform/values.yaml

# ---- Your application ----
app:
  image:
    repository: my-company/my-app
    tag: "2.0.0"

# ---- PostgreSQL subchart config ----
# Key matches the dependency name/alias in Chart.yaml
postgresql:
  # These values are passed into the bitnami/postgresql chart
  enabled: true
  auth:
    postgresPassword: "changeme"
    username: myapp
    password: "myapppass"
    database: myappdb
  primary:
    persistence:
      enabled: true
      size: 10Gi

# ---- Redis subchart config ----
redis:
  enabled: true
  auth:
    enabled: false  # disable password for dev
  master:
    persistence:
      enabled: false  # disable for dev

# ---- Monitoring (disabled by default) ----
prometheus:
  enabled: false
tags:
  monitoring: false  # disables ALL monitoring-tagged charts
```

### Accessing Subchart Services from Your App

When PostgreSQL is installed as a subchart, its service name follows the pattern:
```
<release-name>-postgresql        # default service
<release-name>-postgresql-hl     # headless service
```

Reference this in your app's deployment:
```yaml
# templates/deployment.yaml - in your env vars
env:
  - name: DB_HOST
    value: "{{ .Release.Name }}-postgresql"
  - name: DB_PORT
    value: "5432"
  - name: DB_NAME
    value: {{ .Values.postgresql.auth.database | quote }}
  - name: DB_USER
    value: {{ .Values.postgresql.auth.username | quote }}
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ .Release.Name }}-postgresql
        key: password
```

---

## 📖 Lecture 5.5 — Global Values

**Global values** are shared across the parent chart AND all subcharts.

```yaml
# values.yaml (parent)
global:
  # These are accessible in ALL charts as .Values.global.*
  imageRegistry: "my-registry.company.com"
  imagePullSecrets:
    - name: registry-credentials
  storageClass: "fast-ssd"
  environment: production
  # Custom fields you define
  postgresql:
    # All charts can use this to connect to shared DB
    host: "shared-postgres.company.com"
```

In any subchart template:
```yaml
{{- if .Values.global.imageRegistry }}
image: "{{ .Values.global.imageRegistry }}/{{ .Values.image.repository }}"
{{- end }}
```

> **Rule:** You can always override global values from the parent. Subcharts can consume them but never set them.

---

## 📖 Lecture 5.6 — Library Charts

A **Library Chart** is a chart that provides reusable template definitions but **cannot be installed alone** (it produces no Kubernetes resources).

```yaml
# Chart.yaml for a library chart
apiVersion: v2
name: common-helpers
description: Shared Helm template helpers
type: library   # ← This makes it a library
version: 0.5.0
```

```gotemplate
# common-helpers/templates/_labels.tpl
{{/*
Standard labels — shared across all company charts
*/}}
{{- define "common.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
company.com/team: {{ .Values.global.team | default "platform" | quote }}
company.com/environment: {{ .Values.global.environment | default "dev" | quote }}
{{- end }}

{{/*
Resource name generator
*/}}
{{- define "common.fullname" -}}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
```

**Use in a dependent chart:**
```yaml
# my-service/Chart.yaml
dependencies:
  - name: common-helpers
    version: "0.5.0"
    repository: "file://../common-helpers"
```

```yaml
# my-service/templates/deployment.yaml
metadata:
  name: {{ include "common.fullname" . }}
  labels:
    {{- include "common.labels" . | nindent 4 }}
```

---

## 📖 Lecture 5.7 — Umbrella Charts

An **Umbrella Chart** is a parent chart whose entire purpose is to compose and configure multiple subcharts. It may have NO templates of its own.

```
platform/           ← Umbrella chart
├── Chart.yaml
├── values.yaml     ← All config for all subcharts
├── charts/         ← All subcharts go here (after dep update)
│   ├── postgresql-12.5.6.tgz
│   ├── redis-17.x.x.tgz
│   └── my-webapp-1.0.0.tgz
└── templates/
    └── (potentially empty, or just NOTES.txt)
```

```yaml
# platform/Chart.yaml
apiVersion: v2
name: platform
description: Complete application platform
version: 2.0.0
dependencies:
  - name: my-webapp
    version: "1.0.0"
    repository: "file://../my-webapp"
  - name: postgresql
    version: "12.5.6"
    repository: https://charts.bitnami.com/bitnami
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
```

```bash
# Install the entire platform in one command!
helm install my-platform ./platform
```

---

## ✅ Module 05 — Review Questions

1. What is the benefit of declaring dependencies in `Chart.yaml` over installing charts separately?
2. What does `helm dependency update` do?
3. What is `Chart.lock` and why should it be committed to version control?
4. How do you configure a subchart's values from the parent chart?
5. What are Global Values and how are they different from regular values?
6. What is a Library Chart? What makes it different from an Application Chart?
7. What is an Umbrella Chart? What problem does it solve?
8. How does Helm determine what service name a subchart creates?

---

## 🧪 Lab 05 — Chart Dependencies

**→ See [labs/lab-05-dependencies.md](./labs/lab-05-dependencies.md)**

---

*[← Module 04](../Module-04-Chart-Lifecycle/README.md) | [Module 06 →](../Module-06-Advanced-Templates/README.md)*
