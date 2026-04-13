# 🎓 Capstone Project — Build & Deploy a Full Microservices Platform with Helm

> **Professor's Note:** This is your final exam. You will apply EVERYTHING from the course to build a complete, production-quality Helm chart suite for a multi-service application. No handholding — just requirements. Reference the previous modules when you need them.

---

## 🎯 Project Brief

You are the Platform Engineer at **BookNow**, a hotel reservation startup. The DevOps team needs you to create a complete Helm chart infrastructure for their microservices application.

The system consists of:
- **Frontend** — React app served via nginx
- **API Gateway** — Routes requests to backend services
- **User Service** — Manages users and authentication
- **Booking Service** — Handles reservation logic
- **Notification Service** — Sends email/SMS notifications
- **PostgreSQL** — Primary database (via Bitnami subchart)
- **Redis** — Session cache and message queue (via Bitnami subchart)

---

## 📋 Project Requirements

### Requirement 1: Individual Service Charts

Create individual Helm charts for each service. Each chart must have:

- [ ] Proper `Chart.yaml` with semantic versioning
- [ ] Well-documented `values.yaml` with defaults
- [ ] `values.schema.json` with field validation
- [ ] `deployment.yaml` with resource limits, probes, security contexts
- [ ] `service.yaml` with configurable type
- [ ] `serviceaccount.yaml`
- [ ] `_helpers.tpl` with `fullname`, `labels`, `selectorLabels`
- [ ] `NOTES.txt` with access instructions
- [ ] At least one Helm test in `tests/`
- [ ] `README.md` with configuration table

### Requirement 2: Platform Umbrella Chart

Create a `platform` umbrella chart that:

- [ ] Declares all 5 microservices as local file dependencies
- [ ] Includes PostgreSQL and Redis as external dependencies
- [ ] Has a root `values.yaml` that configures all sub-charts
- [ ] Uses global values for shared configuration (registry, environment, clusterName)

### Requirement 3: Environment Values Files

Create environment-specific values for the umbrella chart:

- [ ] `values-development.yaml` — single replicas, no autoscaling, NodePort
- [ ] `values-staging.yaml` — 2 replicas, autoscaling enabled, PDB
- [ ] `values-production.yaml` — 3+ replicas, strict security, NetworkPolicy, anti-affinity

### Requirement 4: Hooks

Implement for the **Booking Service** chart:

- [ ] `pre-install` hook — database schema migration simulation
- [ ] `post-install` hook — deployment notification

### Requirement 5: Security Hardening

All charts must have:

- [ ] `runAsNonRoot: true` in pod security context
- [ ] `allowPrivilegeEscalation: false` in container security context
- [ ] `capabilities.drop: [ALL]` 
- [ ] `automountServiceAccountToken: false`
- [ ] PodDisruptionBudget template (enabled when replicas > 1)

### Requirement 6: CI/CD

Create a GitHub Actions workflow (`.github/workflows/deploy.yml`) that:

- [ ] Lints all charts in the `charts/` directory
- [ ] Renders templates and validates
- [ ] Deploys to staging on push to `staging` branch
- [ ] Deploys to production on push to `main` with environment protection

---

## 📁 Expected File Structure

```
Capstone-Project/
├── README.md                          ← This file
├── .github/
│   └── workflows/
│       └── deploy.yml                 ← CI/CD pipeline
├── charts/
│   ├── frontend/                     ← Frontend nginx chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── values.schema.json
│   │   ├── ci/
│   │   │   └── lint-values.yaml
│   │   └── templates/
│   │       ├── _helpers.tpl
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── serviceaccount.yaml
│   │       ├── hpa.yaml
│   │       ├── pdb.yaml
│   │       ├── NOTES.txt
│   │       └── tests/
│   │           └── test-connection.yaml
│   ├── api-gateway/                  ← API Gateway chart
│   ├── user-service/                 ← User Service chart
│   ├── booking-service/              ← Booking Service chart (+ hooks!)
│   │   └── templates/
│   │       └── hooks/
│   │           ├── pre-install-migrate.yaml
│   │           └── post-install-notify.yaml
│   ├── notification-service/         ← Notification Service chart
│   └── platform/                     ← UMBRELLA CHART
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-development.yaml
│       ├── values-staging.yaml
│       ├── values-production.yaml
│       └── templates/
│           └── NOTES.txt
```

---

## 🚀 Getting Started

### Step 1 — Create the Charts Skeleton

```bash
mkdir -p C:\Users\Admin\Desktop\DevOps-Skills\capstone
cd C:\Users\Admin\Desktop\DevOps-Skills\capstone
mkdir -p charts

# Create each service chart
helm create charts/frontend
helm create charts/api-gateway
helm create charts/user-service
helm create charts/booking-service
helm create charts/notification-service

# Create the umbrella chart
helm create charts/platform
# Then REMOVE templates/ from platform (umbrella has none)
# Or keep only NOTES.txt
```

### Step 2 — The _helpers.tpl Pattern (Use for ALL charts)

Use this standard pattern in every chart's `_helpers.tpl`:

```gotemplate
{{- define "CHARTNAME.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "CHARTNAME.fullname" -}}
{{- if .Values.fullnameOverride }}
  {{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
  {{- $name := default .Chart.Name .Values.nameOverride }}
  {{- if contains $name .Release.Name }}
    {{- .Release.Name | trunc 63 | trimSuffix "-" }}
  {{- else }}
    {{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
  {{- end }}
{{- end }}
{{- end }}

{{- define "CHARTNAME.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "CHARTNAME.labels" -}}
helm.sh/chart: {{ include "CHARTNAME.chart" . }}
{{ include "CHARTNAME.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "CHARTNAME.selectorLabels" -}}
app.kubernetes.io/name: {{ include "CHARTNAME.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "CHARTNAME.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
  {{- default (include "CHARTNAME.fullname" .) .Values.serviceAccount.name }}
{{- else }}
  {{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

Replace `CHARTNAME` with the actual chart name (e.g., `frontend`, `booking-service`).

### Step 3 — Standard values.yaml Template

Use this as the base `values.yaml` for all service charts:

```yaml
replicaCount: 1

image:
  repository: nginx          # Replace with actual image
  pullPolicy: IfNotPresent
  tag: ""                    # Defaults to Chart.appVersion

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

app:
  environment: development
  logLevel: info

serviceAccount:
  create: true
  automountToken: false
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "80"

podSecurityContext:
  runAsNonRoot: false     # Set to true for production

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL

service:
  type: ClusterIP
  port: 80
  targetPort: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: false
  minAvailable: 1

networkPolicy:
  enabled: false

livenessProbe:
  enabled: true
  path: /
  initialDelaySeconds: 15
  periodSeconds: 20

readinessProbe:
  enabled: true
  path: /
  initialDelaySeconds: 5
  periodSeconds: 10

nodeSelector: {}
tolerations: []
affinity: {}
topologySpreadConstraints: []
```

### Step 4 — Platform Umbrella Chart.yaml

```yaml
# charts/platform/Chart.yaml
apiVersion: v2
name: platform
description: BookNow complete application platform
type: application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  # Internal microservices
  - name: frontend
    version: "0.1.0"
    repository: "file://../frontend"
    condition: frontend.enabled

  - name: api-gateway
    version: "0.1.0"
    repository: "file://../api-gateway"
    condition: api-gateway.enabled

  - name: user-service
    version: "0.1.0"
    repository: "file://../user-service"
    condition: user-service.enabled

  - name: booking-service
    version: "0.1.0"
    repository: "file://../booking-service"
    condition: booking-service.enabled

  - name: notification-service
    version: "0.1.0"
    repository: "file://../notification-service"
    condition: notification-service.enabled

  # External infrastructure
  - name: postgresql
    version: "12.12.10"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled

  - name: redis
    version: "18.1.2"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

---

## 📊 Grading Rubric

| Category | Points | Criteria |
|---|---|---|
| **Chart Structure** | 20 | All required files present, correct YAML syntax |
| **Template Quality** | 20 | No hardcoded values, uses helpers, proper indentation |
| **Values Design** | 15 | Well-documented, sensible defaults, schema validation |
| **Hooks** | 10 | Pre-install and post-install hooks work correctly |
| **Security** | 15 | Security contexts, RBAC, PDB, NetworkPolicy |
| **Umbrella Chart** | 10 | Platform chart composes all services correctly |
| **Environment Values** | 10 | Dev/staging/prod files with appropriate config |
| **CI/CD** | 10 | Valid GitHub Actions workflow with lint and deploy |
| **BONUS** | +10 | Helm tests pass on all charts |
| **BONUS** | +10 | helmfile or ArgoCD Application manifests included |

---

## 🔧 Deployment Commands

### Development
```bash
cd charts/platform
helm dependency update .
helm upgrade --install booknow . \
  -n development \
  --create-namespace \
  --values values.yaml \
  --values values-development.yaml
```

### Staging
```bash
helm upgrade --install booknow . \
  -n staging \
  --create-namespace \
  --values values.yaml \
  --values values-staging.yaml \
  --atomic \
  --timeout 10m
```

### Production
```bash
helm upgrade --install booknow . \
  -n production \
  --create-namespace \
  --values values.yaml \
  --values values-production.yaml \
  --set image.tag=$IMAGE_TAG \
  --atomic \
  --timeout 15m \
  --cleanup-on-fail
```

---

## 📚 Reference Guide — Module Mapping

| Feature Needed | Module Reference |
|---|---|
| Chart structure | Module 02 |
| Template syntax, conditions, loops | Module 03 |
| Hooks (migration, notify) | Module 04 |
| PostgreSQL + Redis dependencies | Module 05 |
| Named templates, _helpers.tpl, range | Module 06 |
| Release management, --atomic | Module 07 |
| Security contexts, NetworkPolicy, PDB | Module 08 |

---

## ✅ Self-Assessment Checklist

Before submitting your capstone:

```
Core Functionality
  [ ] All 5 microservice charts deploy without errors
  [ ] Platform umbrella chart installs all components together
  [ ] helm lint passes for ALL charts with zero errors

Templating
  [ ] resource names use fullname helper (no hardcoded names)
  [ ] All labels use the labels helper
  [ ] Image tag defaults to .Chart.AppVersion
  [ ] No secrets hardcoded anywhere

Values & Config
  [ ] values.schema.json validates input for all charts
  [ ] Three environment-specific values files exist
  [ ] Global values work across umbrella and subcharts

Security
  [ ] runAsNonRoot: true in production values
  [ ] PDB enables for production
  [ ] NetworkPolicy defined (even if disabled by default)

Operations
  [ ] Helm tests present and pass
  [ ] Hooks execute (pre-install, post-install on booking-service)
  [ ] NOTES.txt shows useful post-install instructions

CI/CD
  [ ] GitHub Actions workflow lints all charts
  [ ] Deploy workflow uses helm upgrade --install --atomic
```

---

## 🎓 Congratulations!

If you've completed this capstone, you have demonstrated mastery of:

- ✅ Helm chart creation and structure
- ✅ Advanced Go templating
- ✅ Chart lifecycle management (hooks, tests)
- ✅ Chart composition and dependencies
- ✅ Repository management
- ✅ Production security and HA patterns
- ✅ CI/CD pipeline integration

**You are now a Helm practitioner ready for real-world Kubernetes platform engineering.**

---

*[← Module 08](../Module-08-Production/README.md) | [Course Overview](../README.md)*
