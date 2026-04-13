# Lab 02 — Create Your First Helm Chart from Scratch

> **Lab Objective:** Build a complete, working Helm chart for a sample web application. You will understand every file in the chart structure and deploy it to your local cluster.
>
> **Estimated Time:** 60–90 minutes  
> **Difficulty:** ⭐ Beginner

---

## Part 1 — Generate the Chart Scaffold

### 1.1 — Navigate to Your Working Directory

```bash
# Create a dedicated workspace for this lab
mkdir -p C:\Users\Admin\Desktop\DevOps-Skills\labs\module-02
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-02
```

### 1.2 — Create the Chart

```bash
# Generate the starter chart
helm create webapp

# Explore the generated structure
tree webapp   # Windows: shows directory tree
# Or use:
Get-ChildItem -Recurse webapp  # PowerShell
```

You should see:
```
webapp/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── deployment.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── _helpers.tpl
    ├── NOTES.txt
    └── tests/
        └── test-connection.yaml
```

---

## Part 2 — Customize Chart.yaml

Open `webapp/Chart.yaml` and replace its content:

```yaml
apiVersion: v2
name: webapp
description: A sample web application Helm chart - built during Helm course Module 02
type: application

# Chart version - bump this when YOU change the chart structure
version: 0.1.0

# Application version - the version of the app being deployed
appVersion: "1.0.0"

keywords:
  - webapp
  - nginx
  - tutorial

maintainers:
  - name: Helm Student
    email: student@helmcourse.local

home: https://github.com/helm/helm
sources:
  - https://github.com/helm/helm
```

---

## Part 3 — Custom values.yaml

Open `webapp/values.yaml` and replace with this well-documented version:

```yaml
# ============================================================
# webapp Helm Chart - Default Values
# ============================================================
# These are the default configuration values for this chart.
# Override any of these by:
#   --values my-values.yaml
#   --set key=value
# ============================================================

# Number of pod replicas to run
replicaCount: 1

# Container image configuration
image:
  # The Docker image registry + repository
  repository: nginx
  # Image pull policy: Always, IfNotPresent, Never
  pullPolicy: IfNotPresent
  # Tag defaults to Chart.appVersion if left empty
  tag: ""

# Private registry credentials (list of Secret names)
imagePullSecrets: []

# Override the chart name (used in resource names)
nameOverride: ""
# Override the full resource name prefix
fullnameOverride: ""

# Application configuration (passed as environment variables)
app:
  # Application environment: development, staging, production
  environment: development
  # Log level for the application
  logLevel: info

# ServiceAccount configuration
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Annotations to add to each pod
podAnnotations: {}

# Pod-level security context
podSecurityContext:
  runAsNonRoot: false
  # fsGroup: 2000

# Container-level security context
securityContext: {}
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

# Service configuration
service:
  # ClusterIP: internal only | NodePort: expose via node | LoadBalancer: cloud LB
  type: ClusterIP
  # Port the service listens on
  port: 80
  # Port the container listens on
  targetPort: 80

# Ingress configuration
ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: webapp.local
      paths:
        - path: /
          pathType: Prefix
  tls: []
  # - secretName: webapp-tls
  #   hosts:
  #     - webapp.local

# Resource requests and limits
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi

# Horizontal Pod Autoscaler
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  # targetMemoryUtilizationPercentage: 80

# Liveness and readiness probe configuration
probes:
  liveness:
    enabled: true
    path: /
    initialDelaySeconds: 10
    periodSeconds: 10
  readiness:
    enabled: true
    path: /
    initialDelaySeconds: 5
    periodSeconds: 5

# Node selection, tolerations, and affinity
nodeSelector: {}
tolerations: []
affinity: {}
```

---

## Part 4 — Customize templates/deployment.yaml

Replace the content of `webapp/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "webapp.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          # ---- Environment Variables from values ----
          env:
            - name: APP_ENVIRONMENT
              value: {{ .Values.app.environment | quote }}
            - name: LOG_LEVEL
              value: {{ .Values.app.logLevel | quote }}
            - name: RELEASE_NAME
              value: {{ .Release.Name | quote }}
          # -------------------------------------------
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          # ---- Liveness Probe ----
          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          {{- end }}
          # ---- Readiness Probe ----
          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          {{- end }}
          # ---- Resource Limits ----
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

## Part 5 — Customize NOTES.txt

Replace `webapp/templates/NOTES.txt`:

```
🎉 webapp has been deployed successfully!

Release: {{ .Release.Name }}
Namespace: {{ .Release.Namespace }}
Chart: {{ .Chart.Name }}-{{ .Chart.Version }}
App Version: {{ .Chart.AppVersion }}
Environment: {{ .Values.app.environment }}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HOW TO ACCESS YOUR APP:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{{- if .Values.ingress.enabled }}
  Your app is available via Ingress:
  {{- range .Values.ingress.hosts }}
  http://{{ .host }}
  {{- end }}
{{- else if eq .Values.service.type "LoadBalancer" }}
  Watch for an external IP:
  kubectl get svc --namespace {{ .Release.Namespace }} {{ include "webapp.fullname" . }} -w

  Once available:
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "webapp.fullname" . }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo "http://$SERVICE_IP:{{ .Values.service.port }}"
{{- else if eq .Values.service.type "NodePort" }}
  export NODE_PORT=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "webapp.fullname" . }} -o jsonpath='{.spec.ports[0].nodePort}')
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath='{.items[0].status.addresses[0].address}')
  echo "http://$NODE_IP:$NODE_PORT"
{{- else }}
  Use port-forwarding to access your app:
  kubectl --namespace {{ .Release.Namespace }} port-forward svc/{{ include "webapp.fullname" . }} 8080:{{ .Values.service.port }}

  Then open: http://127.0.0.1:8080
{{- end }}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
USEFUL COMMANDS:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  # Check pods status
  kubectl get pods -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }}

  # View logs
  kubectl logs -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }}

  # Check release status
  helm status {{ .Release.Name }} -n {{ .Release.Namespace }}
```

---

## Part 6 — Validate and Install

### 6.1 — Lint the Chart

```bash
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-02

# Lint catches YAML errors and bad practices
helm lint webapp

# Expected: ==> Linting webapp
#           [INFO] Chart.yaml: icon is recommended
#           1 chart(s) linted, 0 chart(s) failed
```

### 6.2 — Render Templates Locally

```bash
# Render templates without connecting to K8s
helm template my-webapp ./webapp

# Render with custom values
helm template my-webapp ./webapp \
  --set replicaCount=2 \
  --set service.type=NodePort \
  --set app.environment=staging

# Save rendered output to a file for inspection
helm template my-webapp ./webapp > rendered.yaml
cat rendered.yaml
```

### 6.3 — Dry Run

```bash
# Dry run: connects to cluster but doesn't apply
helm install my-webapp ./webapp \
  --namespace helm-lab \
  --create-namespace \
  --dry-run \
  --debug
```

### 6.4 — Install!

```bash
# Actually install the chart
helm install my-webapp ./webapp \
  --namespace helm-lab \
  --create-namespace \
  --set service.type=NodePort \
  --set app.environment=development

# Verify
kubectl get all -n helm-lab
kubectl get pods -n helm-lab -w
```

### 6.5 — Verify the App is Running

```bash
# Port-forward to access locally
kubectl port-forward svc/my-webapp -n helm-lab 8080:80

# Open browser: http://localhost:8080
# You should see the nginx welcome page!
```

---

## Part 7 — Make Changes and Upgrade

### 7.1 — Create a values override file for production

Create `prod-values.yaml`:
```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.25.3"

app:
  environment: production
  logLevel: warn

service:
  type: NodePort

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

```bash
# Upgrade with production values
helm upgrade my-webapp ./webapp \
  --namespace helm-lab \
  --values prod-values.yaml

# Verify changes
helm get values my-webapp -n helm-lab
kubectl get pods -n helm-lab
kubectl get hpa -n helm-lab  # Check HPA was created
```

---

## Part 8 — Package the Chart

```bash
# Package into a distributable .tgz archive
helm package ./webapp

# You should see: Successfully packaged chart and saved it to: webapp-0.1.0.tgz

# Inspect the package
tar -tzf webapp-0.1.0.tgz  # Windows: might need 7-zip or similar
```

---

## Part 9 — Cleanup

```bash
helm uninstall my-webapp -n helm-lab
kubectl delete namespace helm-lab
```

---

## 🏆 Challenges

1. **Challenge 1:** Add a `ConfigMap` template that holds the `app.environment` and `app.logLevel` values, and mount it as environment variables in the deployment (using `envFrom`).

2. **Challenge 2:** Modify the `webapp` chart to support deploying a custom Docker image: `nginxdemos/hello`. The image can be set via `values.yaml`. Install and verify.

3. **Challenge 3:** Add a `podDisruptionBudget.yaml` template that creates a `PodDisruptionBudget` when `replicaCount > 1`, using `if` conditional logic.

---

*[← Module 02 Lecture](../README.md) | [Module 03 →](../../Module-03-Templates-Values/README.md)*
