# Lab 06 — Advanced Helm Templating

> **Lab Objective:** Build a multi-component chart using advanced named templates, the `range` directive to generate multiple resources, file embedding, and validation helpers.
>
> **Estimated Time:** 90–120 minutes  
> **Difficulty:** ⭐⭐⭐ Advanced

---

## Part 1 — Setup: Multi-Microservice Chart

This lab builds a chart that can deploy **multiple microservices** from a single chart using `range` to generate resources.

```bash
mkdir -p C:\Users\Admin\Desktop\DevOps-Skills\labs\module-06
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-06
helm create microservices-platform
```

---

## Part 2 — Rich values.yaml (Multi-Service)

Replace `microservices-platform/values.yaml`:

```yaml
# ================================================================
# microservices-platform values.yaml
# ================================================================

# ---- Global configuration ----
global:
  imageRegistry: ""
  imagePullSecrets: []
  environment: development
  clusterName: ""       # Required! Must be set by user

# ---- Common image defaults ----
defaultImage:
  pullPolicy: IfNotPresent
  tag: "latest"

# ----------------------------------------------------------------
# microservices — list of microservices to deploy
# Each entry generates a Deployment + Service
# ----------------------------------------------------------------
microservices:
  - name: api-gateway
    image:
      repository: nginx
      tag: "1.25.3"
    replicas: 2
    port: 80
    env:
      - name: SERVICE_TYPE
        value: "gateway"
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi
    labels:
      tier: frontend

  - name: user-service
    image:
      repository: nginxdemos/hello
      tag: "latest"
    replicas: 1
    port: 80
    env:
      - name: SERVICE_TYPE
        value: "user"
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
    labels:
      tier: backend

  - name: order-service
    image:
      repository: nginxdemos/hello
      tag: "latest"
    replicas: 1
    port: 80
    env:
      - name: SERVICE_TYPE
        value: "orders"
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
    labels:
      tier: backend

# ---- Common service settings ----
serviceDefaults:
  type: ClusterIP

# ---- ServiceAccount ----
serviceAccount:
  create: true
  annotations: {}
  name: ""

# ---- Pod security ----
podSecurityContext:
  runAsNonRoot: false

# ---- Pod annotations applied to all pods ----
podAnnotations:
  prometheus.io/scrape: "true"

# ---- Common labels on all resources ----
commonLabels:
  managed-by: helm
  project: microservices-platform

# ---- Affinity / node placement ----
nodeSelector: {}
tolerations: []
affinity: {}
```

---

## Part 3 — Build Rich _helpers.tpl

Replace `microservices-platform/templates/_helpers.tpl`:

```gotemplate
{{/*
═══════════════════════════════════════════════════════════════════════
  microservices-platform Helm Helpers
═══════════════════════════════════════════════════════════════════════
*/}}

{{/*
Expand the name of the chart.
*/}}
{{- define "microservices-platform.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Fully qualified release name.
Truncated to 63 chars for Kubernetes name limits.
*/}}
{{- define "microservices-platform.fullname" -}}
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

{{/*
Component-specific resource name.
Usage: {{ include "microservices-platform.componentName" (dict "root" . "component" "user-service") }}
*/}}
{{- define "microservices-platform.componentName" -}}
{{- printf "%s-%s" .root.Release.Name .component | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Chart label value.
*/}}
{{- define "microservices-platform.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels for ALL resources.
*/}}
{{- define "microservices-platform.labels" -}}
helm.sh/chart: {{ include "microservices-platform.chart" . }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
{{- if .Values.commonLabels }}
{{ toYaml .Values.commonLabels }}
{{- end }}
{{- end }}

{{/*
Selector labels for a specific microservice component.
Usage: {{ include "microservices-platform.componentSelectorLabels" (dict "root" . "svcName" "user-service") }}
*/}}
{{- define "microservices-platform.componentSelectorLabels" -}}
app.kubernetes.io/name: {{ .svcName | quote }}
app.kubernetes.io/instance: {{ .root.Release.Name | quote }}
app.kubernetes.io/component: {{ .svcName | quote }}
{{- end }}

{{/*
Image reference builder.
Supports global image registry override.
Usage: {{ include "microservices-platform.image" (dict "root" . "image" .image) }}
*/}}
{{- define "microservices-platform.image" -}}
{{- $registry := .root.Values.global.imageRegistry | default "" }}
{{- $repository := .image.repository }}
{{- $tag := .image.tag | default .root.Values.defaultImage.tag }}
{{- if $registry }}
  {{- printf "%s/%s:%s" $registry $repository $tag }}
{{- else }}
  {{- printf "%s:%s" $repository $tag }}
{{- end }}
{{- end }}

{{/*
ServiceAccount name.
*/}}
{{- define "microservices-platform.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
  {{- default (include "microservices-platform.fullname" .) .Values.serviceAccount.name }}
{{- else }}
  {{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
Validate required values — call this from NOTES.txt or any template.
Fails early if critical config is missing.
*/}}
{{- define "microservices-platform.validate" -}}
{{- if not .Values.global.clusterName }}
  {{- fail "\n\nERROR ❌: global.clusterName is REQUIRED!\n\nPlease set it:\n  helm install ... --set global.clusterName=my-cluster\n\n" }}
{{- end }}
{{- if not .Values.microservices }}
  {{- fail "\n\nERROR ❌: At least one microservice must be defined in .Values.microservices\n\n" }}
{{- end }}
{{- end }}
```

---

## Part 4 — Generate Multiple Deployments with range

Delete the default deployment template and create `templates/microservice-deployments.yaml`:

```yaml
{{/*
VALIDATION — Run at template time to catch missing required values
*/}}
{{- include "microservices-platform.validate" . }}

{{/*
Generate a Deployment for EACH microservice defined in values.yaml
*/}}
{{- range .Values.microservices }}
{{/* 
  Within this range, the dot (.) = the current microservice item
  Use $ to access root-level context (Release, Values, Chart, etc.)
*/}}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "microservices-platform.componentName" (dict "root" $ "component" .name) }}
  namespace: {{ $.Release.Namespace }}
  labels:
    {{- include "microservices-platform.labels" $ | nindent 4 }}
    app.kubernetes.io/component: {{ .name | quote }}
    {{- if .labels }}
    {{- toYaml .labels | nindent 4 }}
    {{- end }}
spec:
  replicas: {{ .replicas | default 1 }}
  selector:
    matchLabels:
      {{- include "microservices-platform.componentSelectorLabels" (dict "root" $ "svcName" .name) | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "microservices-platform.componentSelectorLabels" (dict "root" $ "svcName" .name) | nindent 8 }}
        {{- if $.Values.commonLabels }}
        {{- toYaml $.Values.commonLabels | nindent 8 }}
        {{- end }}
      {{- if $.Values.podAnnotations }}
      annotations:
        {{- toYaml $.Values.podAnnotations | nindent 8 }}
      {{- end }}
    spec:
      serviceAccountName: {{ include "microservices-platform.serviceAccountName" $ }}
      {{- with $.Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .name }}
          image: {{ include "microservices-platform.image" (dict "root" $ "image" .image) | quote }}
          imagePullPolicy: {{ $.Values.defaultImage.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .port | default 80 }}
              protocol: TCP
          # ---- Standard env vars ----
          env:
            - name: APP_NAME
              value: {{ .name | quote }}
            - name: APP_ENV
              value: {{ $.Values.global.environment | quote }}
            - name: CLUSTER_NAME
              value: {{ $.Values.global.clusterName | quote }}
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            {{- if .env }}
            # ---- Service-specific env vars ----
            {{- toYaml .env | nindent 12 }}
            {{- end }}
          # ---- Probes ----
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          # ---- Resources ----
          {{- if .resources }}
          resources:
            {{- toYaml .resources | nindent 12 }}
          {{- end }}
      {{- with $.Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with $.Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with $.Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
{{- end }}
```

---

## Part 5 — Generate Multiple Services with range

Delete the default service template and create `templates/microservice-services.yaml`:

```yaml
{{/*
Generate a Service for EACH microservice
*/}}
{{- range .Values.microservices }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ include "microservices-platform.componentName" (dict "root" $ "component" .name) }}
  namespace: {{ $.Release.Namespace }}
  labels:
    {{- include "microservices-platform.labels" $ | nindent 4 }}
    app.kubernetes.io/component: {{ .name | quote }}
spec:
  type: {{ $.Values.serviceDefaults.type }}
  selector:
    {{- include "microservices-platform.componentSelectorLabels" (dict "root" $ "svcName" .name) | nindent 4 }}
  ports:
    - name: http
      protocol: TCP
      port: {{ .port | default 80 }}
      targetPort: http
{{- end }}
```

---

## Part 6 — Delete Unused Default Templates

```bash
# Remove templates that helm create generated (we replaced them)
Remove-Item microservices-platform/templates/deployment.yaml
Remove-Item microservices-platform/templates/service.yaml
Remove-Item microservices-platform/templates/hpa.yaml        # we're not using this
Remove-Item microservices-platform/templates/ingress.yaml    # skip for now
```

---

## Part 7 — Update NOTES.txt

Replace `microservices-platform/templates/NOTES.txt`:

```
🚀 microservices-platform deployed successfully!

Release:     {{ .Release.Name }}
Namespace:   {{ .Release.Namespace }}
Environment: {{ .Values.global.environment }}
Cluster:     {{ .Values.global.clusterName }}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DEPLOYED MICROSERVICES:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{{- range .Values.microservices }}
  • {{ .name }} ({{ .replicas }} replica{{ if gt .replicas 1 }}s{{ end }})
    Service: {{ include "microservices-platform.componentName" (dict "root" $ "component" .name) }}:{{ .port }}
{{- end }}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PORT-FORWARD COMMANDS:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{{- range $i, $svc := .Values.microservices }}
  kubectl port-forward svc/{{ include "microservices-platform.componentName" (dict "root" $ "component" $svc.name) }} {{ add 8080 $i }}:{{ $svc.port }} -n {{ $.Release.Namespace }}
{{- end }}
```

---

## Part 8 — Preview and Validate

```bash
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-06

# Lint the chart
helm lint microservices-platform --set global.clusterName=my-cluster

# Expected error: fails without clusterName!
helm lint microservices-platform
# ERROR: global.clusterName is REQUIRED!

# Render and inspect the output
helm template my-platform ./microservices-platform \
  --set global.clusterName=my-local-cluster \
  --set global.environment=staging

# Count the Deploymentsand Services generated
helm template my-platform ./microservices-platform \
  --set global.clusterName=test \
  | grep "^kind:" | sort | uniq -c
```

---

## Part 9 — Install and Verify

```bash
kubectl create namespace lab06

helm install my-platform ./microservices-platform \
  -n lab06 \
  --set global.clusterName=minikube-local \
  --set global.environment=development

# Watch all pods come up
kubectl get pods -n lab06 -w

# List all resources created
kubectl get deployment,svc,sa -n lab06

# Should show 3 deployments + 3 services
# api-gateway, user-service, order-service
```

---

## Part 10 — Add a New Microservice Dynamically

```bash
# Add a 4th service via --set WITHOUT changing any code
helm upgrade my-platform ./microservices-platform \
  -n lab06 \
  --set global.clusterName=minikube-local \
  --set "microservices[3].name=payment-service" \
  --set "microservices[3].image.repository=nginxdemos/hello" \
  --set "microservices[3].image.tag=latest" \
  --set "microservices[3].replicas=1" \
  --set "microservices[3].port=80"

# Verify the 4th service was created
kubectl get deployment,svc -n lab06
```

---

## Part 11 — Cleanup

```bash
helm uninstall my-platform -n lab06
kubectl delete namespace lab06
```

---

## 🏆 Challenges

1. **Challenge 1:** Add a named template `microservices-platform.topologySpreadConstraints` that spreads pods across nodes when `replicas > 1`. Include it conditionally in each Deployment.

2. **Challenge 2:** Add file-based config support: create `microservices-platform/config/app.properties` with key=value pairs. Use `.Files.Get` to embed this file into a ConfigMap template, then mount it into all service containers.

3. **Challenge 3:** Use `tpl` to allow users to specify image repositories as templates: `repository: "{{ .Values.global.imageRegistry }}/myapp"`. Test that the registry prefix is correctly applied.

---

*[← Module 06 Lecture](../README.md) | [Module 07 →](../../Module-07-Repositories-Releases/README.md)*
