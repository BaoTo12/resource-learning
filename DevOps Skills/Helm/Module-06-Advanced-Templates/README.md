# Module 06 — Advanced Templating: Named Templates, Helpers & Flow Control

> **Professor's Note:** This is where Helm goes from useful tool to powerful platform. Named templates, `define`/`include`, and advanced flow control separate beginner charts from production-grade ones. Study this module deeply.

---

## 📖 Lecture 6.1 — Named Templates: `define` and `include`

### Defining a Named Template

Named templates are defined in `_helpers.tpl` (or any file starting with `_`):

```gotemplate
{{/*
Comment block — describes what the template does.
Always document your named templates!
*/}}
{{- define "mychrt.resourceName" -}}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if .Values.fullnameOverride }}
  {{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
  {{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
```

### Calling a Named Template

```gotemplate
# include — call template AND pipe its output
name: {{ include "mychrt.resourceName" . }}

# With piped processing
labels:
  {{- include "mychrt.labels" . | nindent 4 }}

# template — call template WITHOUT pipe capability (less common)
{{ template "mychrt.resourceName" . }}
```

> **Always use `include` not `template`** — `include` allows piping to functions like `nindent`, `trim`, etc.

### The Context Object `.`

The second argument to `include`/`template` is the **context** passed to the template:

```gotemplate
# Pass the entire root context
{{- include "myapp.labels" . }}

# Pass a specific sub-value
{{- include "myapp.renderEnvVars" .Values.app }}

# Pass a dict with custom data (build context manually)
{{- include "myapp.renderPorts" (dict "ports" .Values.service.ports "root" .) }}
```

---

## 📖 Lecture 6.2 — Building a Complete Helper Library

Here's a production-quality `_helpers.tpl`:

```gotemplate
{{/*
═══════════════════════════════════════════════════════
  NAMING HELPERS
═══════════════════════════════════════════════════════
*/}}

{{/*
Chart name — allows override via nameOverride
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Fully qualified name: <release>-<chart>
Truncated to 63 chars (Kubernetes limit)
*/}}
{{- define "myapp.fullname" -}}
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
Chart label value: <name>-<version>
*/}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
═══════════════════════════════════════════════════════
  LABEL HELPERS
═══════════════════════════════════════════════════════
*/}}

{{/*
Common labels — applied to ALL resources
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- if .Values.commonLabels }}
{{ toYaml .Values.commonLabels }}
{{- end }}
{{- end }}

{{/*
Selector labels — used in selector matchLabels (stable, do NOT add extra labels here!)
These CANNOT change after initial deployment (Pod selector is immutable!)
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
═══════════════════════════════════════════════════════
  SERVICE ACCOUNT HELPER
═══════════════════════════════════════════════════════
*/}}

{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
  {{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
  {{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
═══════════════════════════════════════════════════════
  IMAGE HELPER
═══════════════════════════════════════════════════════
*/}}

{{/*
Renders the full image reference: registry/repository:tag
Supports optional global registry override
*/}}
{{- define "myapp.image" -}}
{{- $registry := .Values.global.imageRegistry | default "" }}
{{- $repository := .Values.image.repository }}
{{- $tag := .Values.image.tag | default .Chart.AppVersion }}
{{- if $registry }}
  {{- printf "%s/%s:%s" $registry $repository $tag }}
{{- else }}
  {{- printf "%s:%s" $repository $tag }}
{{- end }}
{{- end }}

{{/*
═══════════════════════════════════════════════════════
  ENVIRONMENT VARIABLE HELPERS
═══════════════════════════════════════════════════════
*/}}

{{/*
Renders standard environment variables present in all containers
*/}}
{{- define "myapp.commonEnvVars" -}}
- name: APP_NAME
  value: {{ .Chart.Name | quote }}
- name: APP_VERSION
  value: {{ .Chart.AppVersion | quote }}
- name: RELEASE_NAME
  value: {{ .Release.Name | quote }}
- name: POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
- name: NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName
{{- end }}

{{/*
═══════════════════════════════════════════════════════
  PROBE HELPERS
═══════════════════════════════════════════════════════
*/}}

{{/*
Renders liveness probe block
Usage: {{ include "myapp.livenessProbe" .Values.livenessProbe }}
*/}}
{{- define "myapp.livenessProbe" -}}
{{- if .enabled }}
livenessProbe:
  {{- if .exec }}
  exec:
    command:
      {{- toYaml .exec.command | nindent 6 }}
  {{- else if .httpGet }}
  httpGet:
    path: {{ .httpGet.path | default "/" }}
    port: {{ .httpGet.port | default "http" }}
  {{- else if .tcpSocket }}
  tcpSocket:
    port: {{ .tcpSocket.port | default "http" }}
  {{- end }}
  initialDelaySeconds: {{ .initialDelaySeconds | default 15 }}
  periodSeconds: {{ .periodSeconds | default 20 }}
  failureThreshold: {{ .failureThreshold | default 3 }}
  successThreshold: {{ .successThreshold | default 1 }}
  timeoutSeconds: {{ .timeoutSeconds | default 5 }}
{{- end }}
{{- end }}

{{/*
═══════════════════════════════════════════════════════
  RESOURCE QUOTA HELPERS
═══════════════════════════════════════════════════════
*/}}

{{/*
Renders resource requests/limits.
If .Values.resources is empty, no resources block is created.
*/}}
{{- define "myapp.resources" -}}
{{- if . }}
resources:
  {{- toYaml . | nindent 2 }}
{{- end }}
{{- end }}

{{/*
═══════════════════════════════════════════════════════
  CHECKSUM ANNOTATION HELPER
═══════════════════════════════════════════════════════
*/}}

{{/*
Creates a checksum annotation to trigger rolling restart on ConfigMap/Secret change.
Usage: {{ include "myapp.checksumAnnotation" (list . "path/to/configmap.yaml") }}
*/}}
{{- define "myapp.checksumAnnotation" -}}
{{- $ctx := index . 0 }}
{{- $path := index . 1 }}
checksum/config: {{ include (print $ctx.Template.BasePath "/" $path) $ctx | sha256sum }}
{{- end }}
```

---

## 📖 Lecture 6.3 — Advanced Flow Control

### The `fail` Function — Validation in Templates

Force a chart to fail with a helpful message if required values are not set:

```gotemplate
# Fail if a required value is missing
{{- if not .Values.global.clusterName }}
  {{- fail "ERROR: .Values.global.clusterName is required but not set!" }}
{{- end }}

# Fail if a value is an invalid option
{{- $validTypes := list "web" "worker" "scheduler" }}
{{- if not (has .Values.app.type $validTypes) }}
  {{- fail (printf "ERROR: app.type must be one of: %s" ($validTypes | join ", ")) }}
{{- end }}
```

### `required` Function

```gotemplate
# required: fails if the value is empty, with custom message
image: "{{ .Values.image.repository | required "image.repository is required" }}:latest"

# Combine with default:
db_host: {{ .Values.database.host | default (printf "%s-postgresql" .Release.Name) | required "DB host" }}
```

### `tpl` Function — Render Strings as Templates

Sometimes you want value strings to contain template expressions:

```yaml
# values.yaml
config:
  welcomeMessage: "Hello from {{ .Release.Name }} in {{ .Release.Namespace }}!"
  endpoint: "http://{{ .Release.Name }}.{{ .Release.Namespace }}.svc.cluster.local"
```

```gotemplate
# template
data:
  message: {{ tpl .Values.config.welcomeMessage . | quote }}
  endpoint: {{ tpl .Values.config.endpoint . | quote }}
```

> **Use sparingly** — `tpl` adds complexity. Only use it when values genuinely need to reference other Helm context.

### Advanced: `dict` and `merge` for Context Building

```gotemplate
{{/*
Build a context dict to pass extra data to sub-templates
*/}}
{{- define "myapp.podTemplate" -}}
{{- $context := merge (dict "currentComponent" "api-server") . }}
{{- include "myapp.container" $context }}
{{- end }}

{{- define "myapp.container" -}}
# In here, .currentComponent = "api-server"
name: {{ .currentComponent }}-container
{{- end }}
```

---

## 📖 Lecture 6.4 — Generating Multiple Resources with `range`

```gotemplate
# values.yaml
microservices:
  - name: user-api
    port: 8080
    replicas: 2
  - name: order-api
    port: 8081
    replicas: 3
  - name: payment-api
    port: 8082
    replicas: 2
```

```gotemplate
# templates/micro-deployments.yaml
{{- range .Values.microservices }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ printf "%s-%s" $.Release.Name .name }}  ← $ for root, . for loop item
  labels:
    app: {{ .name }}
    release: {{ $.Release.Name }}
spec:
  replicas: {{ .replicas }}
  selector:
    matchLabels:
      app: {{ .name }}
  template:
    metadata:
      labels:
        app: {{ .name }}
    spec:
      containers:
        - name: {{ .name }}
          image: "{{ $.Values.image.repository }}/{{ .name }}:{{ $.Values.image.tag }}"
          ports:
            - containerPort: {{ .port }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ printf "%s-%s" $.Release.Name .name }}
spec:
  selector:
    app: {{ .name }}
  ports:
    - port: {{ .port }}
      targetPort: {{ .port }}
{{- end }}
```

---

## 📖 Lecture 6.5 — Template Composition Patterns

### Pattern 1: DRY Container Spec

```gotemplate
# _helpers.tpl
{{- define "myapp.containerSpec" -}}
image: {{ include "myapp.image" . }}
imagePullPolicy: {{ .Values.image.pullPolicy }}
{{- include "myapp.commonEnvVars" . | nindent 0 }}
{{- with .Values.extraEnvVars }}
{{- toYaml . | nindent 0 }}
{{- end }}
ports:
  - name: http
    containerPort: {{ .Values.service.targetPort }}
    protocol: TCP
{{- include "myapp.livenessProbe" .Values.livenessProbe | nindent 0 }}
resources:
  {{- toYaml .Values.resources | nindent 2 }}
{{- end }}
```

```yaml
# templates/deployment.yaml
containers:
  - name: app
    {{- include "myapp.containerSpec" . | nindent 4 }}
  - name: sidecar
    {{- include "myapp.containerSpec" . | nindent 4 }}
```

### Pattern 2: Conditional Image Pull Secrets

```gotemplate
{{- define "myapp.imagePullSecrets" -}}
{{- $secrets := list }}
{{- if .Values.imagePullSecrets }}
  {{- $secrets = .Values.imagePullSecrets }}
{{- end }}
{{- if .Values.global.imagePullSecrets }}
  {{- $secrets = concat $secrets .Values.global.imagePullSecrets }}
{{- end }}
{{- if $secrets }}
imagePullSecrets:
  {{- toYaml $secrets | nindent 2 }}
{{- end }}
{{- end }}
```

---

## 📖 Lecture 6.6 — Using `.Files` to Include Static Files

The `.Files` object lets you embed files from your chart into templates:

```
myapp/
├── Chart.yaml
├── values.yaml
├── config/
│   ├── nginx.conf
│   └── app.properties
└── templates/
    └── configmap.yaml
```

```gotemplate
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  # Inline file content directly
  nginx.conf: |
    {{- .Files.Get "config/nginx.conf" | nindent 4 }}

  # Or use Files.Glob to get multiple files
  {{- (.Files.Glob "config/*.properties").AsConfig | nindent 2 }}
```

```gotemplate
# Encode file as base64 for Secrets
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-certs
type: Opaque
data:
  tls.crt: {{ .Files.Get "certs/tls.crt" | b64enc }}
  tls.key: {{ .Files.Get "certs/tls.key" | b64enc }}
```

---

## ✅ Module 06 — Review Questions

1. What is the difference between `include` and `template`?
2. Why should the second argument to `include` always be `.` or `$`?
3. What is the `$` variable and when do you need it vs `.`?
4. How do you pass additional data (not in the main context) to a named template?
5. What does `fail` do and when should you use it?
6. How does `tpl` work? Give an example where it's useful.
7. When using `range` with a list, how do you access root-level values like `.Release.Name`?
8. What does `| nindent 4` do, and would you use it with `include` or `template`?
9. How do you include a file from your chart directory into a ConfigMap?

---

## 🧪 Lab 06 — Advanced Templates

**→ See [labs/lab-06-advanced-templates.md](./labs/lab-06-advanced-templates.md)**

---

*[← Module 05](../Module-05-Dependencies/README.md) | [Module 07 →](../Module-07-Repositories-Releases/README.md)*
