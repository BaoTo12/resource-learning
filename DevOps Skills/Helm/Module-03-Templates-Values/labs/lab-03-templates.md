# Lab 03 — Advanced Templates & Values

> **Lab Objective:** Practice Helm templating by building a feature-rich chart with conditionals, loops, named templates, and schema validation.
>
> **Estimated Time:** 90–120 minutes  
> **Difficulty:** ⭐⭐ Intermediate

---

## Part 1 — Setting Up the Lab Chart

```bash
mkdir -p C:\Users\Admin\Desktop\DevOps-Skills\labs\module-03
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-03
helm create advanced-webapp
```

---

## Part 2 — Rich values.yaml

Replace `advanced-webapp/values.yaml` entirely:

```yaml
# ================================================================
# advanced-webapp — Rich values.yaml
# ================================================================

replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: ""

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

# ---- App Config ----
app:
  name: "Advanced Web App"
  environment: development   # development | staging | production
  version: "1.0.0"

# ---- Extra environment variables (free-form list) ----
extraEnvVars: []
# - name: MY_VAR
#   value: "my-value"
# - name: SECRET_VAR
#   valueFrom:
#     secretKeyRef:
#       name: my-secret
#       key: password

# ---- Extra environment variables from ConfigMaps/Secrets ----
extraEnvFrom: []
# - configMapRef:
#     name: my-config
# - secretRef:
#     name: my-secret

# ---- ConfigMap data (will be created and mounted as env vars) ----
configMap:
  enabled: true
  data:
    APP_TITLE: "My Helm App"
    CACHE_TTL: "300"
    FEATURE_FLAG_DARK_MODE: "true"

# ---- Annotations (added to pods) ----
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "80"
  prometheus.io/path: "/metrics"

# ---- Custom labels (added to all resources) ----
commonLabels:
  team: platform
  cost-center: engineering

# ---- ServiceAccount ----
serviceAccount:
  create: true
  annotations: {}
  name: ""

# ---- Security Contexts ----
podSecurityContext:
  runAsNonRoot: false

securityContext: {}

# ---- Service ----
service:
  type: ClusterIP
  port: 80
  targetPort: 80
  annotations: {}

# ---- Ingress ----
ingress:
  enabled: false
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: advanced-webapp.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

# ---- Resources ----
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi

# ---- Autoscaling ----
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# ---- Probes ----
livenessProbe:
  enabled: true
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3

readinessProbe:
  enabled: true
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

# ---- Init Containers ----
initContainers: []
# - name: init-wait
#   image: busybox:1.35
#   command: ['sh', '-c', 'echo "Waiting for services..."']

# ---- Extra sidecar containers ----
sidecarContainers: []
# - name: log-shipper
#   image: fluent/fluent-bit
#   ...

# ---- Extra volumes ----
extraVolumes: []
# - name: config-vol
#   configMap:
#     name: my-config

# ---- Extra volume mounts ----
extraVolumeMounts: []
# - name: config-vol
#   mountPath: /etc/config

# ---- Host aliases ----
hostAliases: []
# - ip: "127.0.0.1"
#   hostnames: ["myapp.local"]

# ---- Pod Disruption Budget ----
podDisruptionBudget:
  enabled: false
  minAvailable: 1
  # maxUnavailable: 1

# ---- Node placement ----
nodeSelector: {}
tolerations: []
affinity: {}
```

---

## Part 3 — Create templates/configmap.yaml

Create a new file `advanced-webapp/templates/configmap.yaml`:

```yaml
{{- if .Values.configMap.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "advanced-webapp.fullname" . }}-config
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "advanced-webapp.labels" . | nindent 4 }}
    {{- with .Values.commonLabels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
data:
  # Chart-provided static config
  RELEASE_NAME: {{ .Release.Name | quote }}
  RELEASE_NAMESPACE: {{ .Release.Namespace | quote }}
  APP_ENVIRONMENT: {{ .Values.app.environment | quote }}
  APP_VERSION: {{ .Values.app.version | quote }}
  {{- range $key, $value := .Values.configMap.data }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
```

---

## Part 4 — Enhance templates/deployment.yaml

Replace `advanced-webapp/templates/deployment.yaml` with this rich version:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "advanced-webapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "advanced-webapp.labels" . | nindent 4 }}
    {{- with .Values.commonLabels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "advanced-webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        # Causes rolling restart when ConfigMap changes
        {{- if .Values.configMap.enabled }}
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- end }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "advanced-webapp.selectorLabels" . | nindent 8 }}
        {{- with .Values.commonLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "advanced-webapp.serviceAccountName" . }}
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      # ---- Init Containers ----
      {{- if .Values.initContainers }}
      initContainers:
        {{- toYaml .Values.initContainers | nindent 8 }}
      {{- end }}
      # ---- Host Aliases ----
      {{- if .Values.hostAliases }}
      hostAliases:
        {{- toYaml .Values.hostAliases | nindent 8 }}
      {{- end }}
      containers:
        # ---- Main Application Container ----
        - name: {{ .Chart.Name }}
          {{- with .Values.securityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          # ---- Environment Variables ----
          env:
            - name: APP_NAME
              value: {{ .Values.app.name | quote }}
            {{- if .Values.extraEnvVars }}
            {{- toYaml .Values.extraEnvVars | nindent 12 }}
            {{- end }}
          # ---- Env From ConfigMap/Secret ----
          {{- if or .Values.configMap.enabled .Values.extraEnvFrom }}
          envFrom:
            {{- if .Values.configMap.enabled }}
            - configMapRef:
                name: {{ include "advanced-webapp.fullname" . }}-config
            {{- end }}
            {{- if .Values.extraEnvFrom }}
            {{- toYaml .Values.extraEnvFrom | nindent 12 }}
            {{- end }}
          {{- end }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          # ---- Liveness Probe ----
          {{- if .Values.livenessProbe.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.httpGet.path }}
              port: {{ .Values.livenessProbe.httpGet.port }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            failureThreshold: {{ .Values.livenessProbe.failureThreshold }}
          {{- end }}
          # ---- Readiness Probe ----
          {{- if .Values.readinessProbe.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.readinessProbe.httpGet.path }}
              port: {{ .Values.readinessProbe.httpGet.port }}
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
            failureThreshold: {{ .Values.readinessProbe.failureThreshold }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          # ---- Volume Mounts ----
          {{- if .Values.extraVolumeMounts }}
          volumeMounts:
            {{- toYaml .Values.extraVolumeMounts | nindent 12 }}
          {{- end }}
        # ---- Sidecar Containers ----
        {{- if .Values.sidecarContainers }}
        {{- toYaml .Values.sidecarContainers | nindent 8 }}
        {{- end }}
      # ---- Volumes ----
      {{- if .Values.extraVolumes }}
      volumes:
        {{- toYaml .Values.extraVolumes | nindent 8 }}
      {{- end }}
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

## Part 5 — Create templates/poddisruptionbudget.yaml

```yaml
{{- if and .Values.podDisruptionBudget.enabled (gt .Values.replicaCount 1) }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "advanced-webapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "advanced-webapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "advanced-webapp.selectorLabels" . | nindent 6 }}
  {{- if .Values.podDisruptionBudget.minAvailable }}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- else if .Values.podDisruptionBudget.maxUnavailable }}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable }}
  {{- end }}
{{- end }}
```

---

## Part 6 — Create values.schema.json

Create `advanced-webapp/values.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["replicaCount", "image", "service"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 50,
      "description": "Number of pod replicas"
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "minLength": 1
        },
        "tag": {
          "type": "string"
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer", "ExternalName"]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    },
    "app": {
      "type": "object",
      "properties": {
        "environment": {
          "type": "string",
          "enum": ["development", "staging", "production"]
        }
      }
    }
  }
}
```

---

## Part 7 — Test Schema Validation

```bash
# This should FAIL — wrong environment value
helm lint advanced-webapp \
  --set app.environment=prod  # "prod" is not in the enum!

# Expected error:
# [ERROR] values don't meet the specifications of the schema...
# - app.environment: app.environment must be one of the following: "development", "staging", "production"

# This should FAIL — too many replicas
helm lint advanced-webapp \
  --set replicaCount=100

# This should SUCCEED
helm lint advanced-webapp
```

---

## Part 8 — Template with Multiple Value Overrides

```bash
# Render templates with complex overrides
helm template test-app ./advanced-webapp \
  --set replicaCount=2 \
  --set app.environment=staging \
  --set configMap.enabled=true \
  --set "configMap.data.FEATURE_FLAG_DARK_MODE=false" \
  --set "podAnnotations.custom-annotation=my-value" \
  --set "commonLabels.team=backend" \
  --set podDisruptionBudget.enabled=true

# Save output and inspect
helm template test-app ./advanced-webapp > rendered-full.yaml
```

---

## Part 9 — Install, Verify ConfigMap Injection

```bash
kubectl create namespace lab03
helm install advanced-app ./advanced-webapp \
  -n lab03 \
  --create-namespace \
  --set service.type=NodePort \
  --set app.environment=development

# List all resources created
kubectl get all,cm,secret -n lab03

# Check if ConfigMap was created
kubectl get configmap -n lab03
kubectl describe configmap advanced-app-advanced-webapp-config -n lab03

# Check that env vars from ConfigMap were injected into pod
kubectl exec -n lab03 \
  $(kubectl get pods -n lab03 -o jsonpath='{.items[0].metadata.name}') \
  -- env | grep -E "APP_|CACHE_|FEATURE_|RELEASE_"
```

---

## Part 10 — Test ConfigMap Change Detection

```bash
# Update a config value
helm upgrade advanced-app ./advanced-webapp \
  -n lab03 \
  --set configMap.data.CACHE_TTL=600 \
  --set service.type=NodePort

# Watch for pod restart (the sha256 checksum annotation causes a rolling restart)
kubectl get pods -n lab03 -w
```

---

## Part 11 — Cleanup

```bash
helm uninstall advanced-app -n lab03
kubectl delete namespace lab03
```

---

## 🏆 Challenges

1. **Challenge 1:** Add functional `initContainers` that waits for a service using `busybox` and `nc -z <host> <port>`. Set the values and verify it runs.

2. **Challenge 2:** Add a `sidecarContainers` entry with a log shipper container (`busybox` printing to stdout). Verify it appears in the deployment.

3. **Challenge 3:** Modify the schema to require that if `autoscaling.enabled` is true, then `autoscaling.maxReplicas` must be greater than `autoscaling.minReplicas`. Research `if/then` in JSON Schema to solve this.

---

*[← Module 03 Lecture](../README.md) | [Module 04 →](../../Module-04-Chart-Lifecycle/README.md)*
