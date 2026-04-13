# Lab 05 — Chart Dependencies & Subchart Composition

> **Lab Objective:** Build a full-stack application chart with PostgreSQL and Redis dependencies. Practice umbrella chart composition.
>
> **Estimated Time:** 90 minutes  
> **Difficulty:** ⭐⭐ Intermediate

---

## Part 1 — Create the Application Chart

```bash
mkdir -p C:\Users\Admin\Desktop\DevOps-Skills\labs\module-05
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-05
helm create full-stack-app
```

---

## Part 2 — Add Dependencies to Chart.yaml

Open `full-stack-app/Chart.yaml` and replace:

```yaml
apiVersion: v2
name: full-stack-app
description: A full-stack application with PostgreSQL and Redis
type: application
version: 0.1.0
appVersion: "1.0.0"

keywords:
  - webapp
  - postgresql
  - redis
  - full-stack

maintainers:
  - name: Helm Student
    email: student@helmcourse.local

dependencies:
  # PostgreSQL — primary database
  - name: postgresql
    version: "12.12.10"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled    # only if postgresql.enabled = true

  # Redis — cache layer
  - name: redis
    version: "18.1.2"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled          # only if redis.enabled = true
```

---

## Part 3 — Configure values.yaml for Subcharts

Replace `full-stack-app/values.yaml`:

```yaml
# ================================================================
# full-stack-app — values.yaml
# ================================================================

# ---- Application ----
replicaCount: 1

image:
  repository: nginxdemos/hello
  pullPolicy: IfNotPresent
  tag: ""

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: fullstack.local
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
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

nodeSelector: {}
tolerations: []
affinity: {}

# ================================================================
# Global values (shared across all charts)
# ================================================================
global:
  storageClass: ""
  environment: development

# ================================================================
# PostgreSQL Subchart Configuration
# Prefix: "postgresql" (matches dependency name in Chart.yaml)
# Full docs: helm show values bitnami/postgresql
# ================================================================
postgresql:
  enabled: true    # set to false to use external DB

  # Authentication settings
  auth:
    postgresPassword: "helmcourse123"
    username: "appuser"
    password: "apppassword"
    database: "appdb"

  # Primary database server
  primary:
    persistence:
      enabled: true
      size: 1Gi
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi

# ================================================================
# Redis Subchart Configuration
# Prefix: "redis" (matches dependency name in Chart.yaml)
# Full docs: helm show values bitnami/redis
# ================================================================
redis:
  enabled: true

  architecture: standalone   # "standalone" or "replication"

  auth:
    enabled: false            # disable password for dev simplicity

  master:
    persistence:
      enabled: false          # disable for dev

  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 128Mi

# ================================================================
# Application database connection config
# (used in deployment env vars to connect to the above services)
# ================================================================
appConfig:
  # If postgresql.enabled = true, these auto-reference the subchart
  database:
    host: ""          # Leave empty to auto-derive from release name
    port: 5432
    name: appdb       # Should match postgresql.auth.database
    user: appuser     # Should match postgresql.auth.username

  # If redis.enabled = true, these auto-reference the subchart
  cache:
    host: ""          # Leave empty to auto-derive from release name
    port: 6379
```

---

## Part 4 — Update the Deployment to Connect to Dependencies

Replace `full-stack-app/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "full-stack-app.fullname" . }}
  labels:
    {{- include "full-stack-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "full-stack-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "full-stack-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "full-stack-app.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          env:
            - name: APP_ENV
              value: {{ .Values.global.environment | quote }}

            # ---- PostgreSQL Connection (if enabled) ----
            {{- if .Values.postgresql.enabled }}
            - name: DB_HOST
              # Service name pattern: <release-name>-postgresql
              value: "{{ .Release.Name }}-postgresql"
            - name: DB_PORT
              value: {{ .Values.appConfig.database.port | quote }}
            - name: DB_NAME
              value: {{ .Values.appConfig.database.name | quote }}
            - name: DB_USER
              value: {{ .Values.appConfig.database.user | quote }}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: "{{ .Release.Name }}-postgresql"
                  key: password
            {{- end }}

            # ---- Redis Connection (if enabled) ----
            {{- if .Values.redis.enabled }}
            - name: REDIS_HOST
              # Service name pattern: <release-name>-redis-master
              value: "{{ .Release.Name }}-redis-master"
            - name: REDIS_PORT
              value: {{ .Values.appConfig.cache.port | quote }}
            {{- end }}

          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 10
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

---

## Part 5 — Download Dependencies

```bash
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-05

# Download all dependency charts
helm dependency update ./full-stack-app

# Verify charts/ directory now contains the downloaded charts
ls ./full-stack-app/charts/
# full-stack-app/charts/postgresql-12.12.10.tgz
# full-stack-app/charts/redis-18.1.2.tgz

# Check the generated Chart.lock
cat ./full-stack-app/Chart.lock
```

---

## Part 6 — Install the Full Stack

```bash
kubectl create namespace lab05

# IMPORTANT: Dry run first to see what gets created
helm install fullstack ./full-stack-app \
  -n lab05 \
  --dry-run \
  --debug | head -100

# Install everything!
helm install fullstack ./full-stack-app \
  -n lab05 \
  --create-namespace

# Watch everything come up (takes a minute)
kubectl get pods -n lab05 -w

# Full resource list
kubectl get all -n lab05
kubectl get pvc -n lab05   # PersistentVolumeClaims from PostgreSQL
kubectl get secret -n lab05
```

---

## Part 7 — Inspect the Subchart Behavior

```bash
# See what values are active for the release
helm get values fullstack -n lab05 --all

# Verify PostgreSQL is running
kubectl exec -n lab05 \
  $(kubectl get pods -n lab05 -l app.kubernetes.io/name=postgresql -o jsonpath='{.items[0].metadata.name}') \
  -- psql -U postgres -c '\l'

# Verify Redis is running  
kubectl exec -n lab05 \
  $(kubectl get pods -n lab05 -l app.kubernetes.io/name=redis -o jsonpath='{.items[0].metadata.name}') \
  -- redis-cli ping
# Expected: PONG
```

---

## Part 8 — Test Disabling a Subchart

```bash
# Upgrade: disable Redis
helm upgrade fullstack ./full-stack-app \
  -n lab05 \
  --set redis.enabled=false

# Verify Redis resources are gone
kubectl get pods -n lab05
# Redis pods should be terminating/gone

# Application still runs but Redis env vars are now omitted
kubectl exec -n lab05 \
  $(kubectl get pods -n lab05 -l app.kubernetes.io/name=full-stack-app -o jsonpath='{.items[0].metadata.name}') \
  -- env | grep REDIS
# No output — REDIS env vars were conditionally excluded
```

---

## Part 9 — Create an Umbrella Chart (Bonus)

```bash
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-05
mkdir platform-umbrella
cd platform-umbrella
```

Create `Chart.yaml`:
```yaml
apiVersion: v2
name: platform-umbrella
description: Umbrella chart composing the full company platform
type: application
version: 1.0.0

dependencies:
  - name: full-stack-app
    version: "0.1.0"
    repository: "file://../full-stack-app"
```

Create `values.yaml`:
```yaml
full-stack-app:
  replicaCount: 2
  postgresql:
    enabled: true
    auth:
      postgresPassword: "umbrella_prod_pass"
  redis:
    enabled: true
  global:
    environment: production
```

```bash
cd ..  # back to module-05

# Update dependencies for umbrella
helm dependency update ./platform-umbrella

# Install the entire platform with ONE command
helm install platform ./platform-umbrella -n lab05
```

---

## Part 10 — Cleanup

```bash
helm uninstall fullstack -n lab05 2>/dev/null || true
helm uninstall platform -n lab05 2>/dev/null || true
kubectl delete namespace lab05
```

---

## 🏆 Challenges

1. **Challenge 1:** Add `elasticsearch` from the Bitnami repo as an optional dependency (condition: `elasticsearch.enabled`). Deploy with and without it. Monitor resource usage.

2. **Challenge 2:** Use the `alias` feature to deploy **two separate PostgreSQL instances** from the same chart — one for the app database and one as an analytics read-replica.

3. **Challenge 3:** Create a simple library chart with a `common.labels` and `common.resourceName` template. Make `full-stack-app` depend on it and use the library templates instead of the generated ones.

---

*[← Module 05 Lecture](../README.md) | [Module 06 →](../../Module-06-Advanced-Templates/README.md)*
