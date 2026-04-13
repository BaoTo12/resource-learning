# Module 02 — Your First Helm Chart

> **Professor's Note:** Now that you understand *what* Helm is, it's time to *build*. In this module you will create a complete Helm chart from scratch. We'll use `helm create` to generate the scaffold, then study every file in detail and modify it to deploy a real application.

---

## 📖 Lecture 2.1 — Chart Scaffold with `helm create`

Helm provides a command to generate a complete starter chart:

```bash
helm create my-first-chart
```

This creates:
```
my-first-chart/
├── Chart.yaml
├── values.yaml
├── charts/              (empty — no dependencies yet)
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── serviceaccount.yaml
    ├── _helpers.tpl
    ├── NOTES.txt
    └── tests/
        └── test-connection.yaml
```

### What `helm create` Gives You

The default chart deploys **nginx** as the container image. Every template is heavily commented. This scaffold is the de facto starting point for 90% of real-world Helm charts.

---

## 📖 Lecture 2.2 — Deep Dive: Chart.yaml

```yaml
apiVersion: v2
name: my-first-chart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

### Field Explanations

| Field | Required | Description |
|---|---|---|
| `apiVersion` | ✅ | Always `v2` for Helm 3 |
| `name` | ✅ | Chart name — must match directory name |
| `description` | ✅ | What does this chart deploy? |
| `type` | ❌ | `application` (default) or `library` |
| `version` | ✅ | **Chart** version — increment when chart changes (SemVer) |
| `appVersion` | ❌ | **Application** version (informational only) |
| `keywords` | ❌ | Tags for Artifact Hub searchability |
| `home` | ❌ | URL to the application's home page |
| `icon` | ❌ | URL to chart icon |
| `maintainers` | ❌ | List of chart maintainers |

> **KEY INSIGHT:** `version` and `appVersion` are separate!  
> - `version` is the **chart's** version — bump this when you change the chart structure  
> - `appVersion` is the **application's** version — informational, used in labels  
> These can change independently.

---

## 📖 Lecture 2.3 — Deep Dive: values.yaml

The `values.yaml` is the **contract** between your chart and its users.

```yaml
# Default values for my-first-chart.
# This is a YAML file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: ""

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  name: ""

podAnnotations: {}

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources: {}
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}
```

### Key Design Principles for values.yaml

1. **Every configurable thing goes in values.yaml** — not hardcoded in templates
2. **Use sensible defaults** — the chart should work with zero user configuration
3. **Comment everything** — users will read this file to understand the chart
4. **Use nested structure** — `image.repository`, `image.tag` rather than flat `imageRepository`
5. **Boolean flags** — `ingress.enabled: false` to disable features by default

---

## 📖 Lecture 2.4 — Deep Dive: templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-first-chart.fullname" . }}   ← Named template
  labels:
    {{- include "my-first-chart.labels" . | nindent 4 }}   ← Labels helper
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-first-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "my-first-chart.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-first-chart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### Template Syntax Breakdown

| Syntax | Meaning |
|---|---|
| `{{ .Values.replicaCount }}` | Access value from values.yaml |
| `{{ .Release.Name }}` | The release name (set at install time) |
| `{{ .Chart.Name }}` | Chart name from Chart.yaml |
| `{{ .Chart.AppVersion }}` | App version from Chart.yaml |
| `{{- if condition }}` | Conditional block (dash trims whitespace) |
| `{{- end }}` | End of conditional/loop/with |
| `{{- with .Values.x }}` | Execute block only if `.Values.x` is not empty |
| `{{ include "name" . }}` | Call a named template |
| `\| nindent 4` | Pipe to `nindent` function — add newline + 4 spaces indent |
| `\| toYaml` | Convert Go object to YAML string |
| `\| default "value"` | Use default if value is empty |

---

## 📖 Lecture 2.5 — Deep Dive: templates/_helpers.tpl

The `_helpers.tpl` file (note the underscore prefix — it's **NOT** rendered as a manifest) defines **named templates** reused across all other templates.

```gotemplate
{{/*
Expand the name of the chart.
*/}}
{{- define "my-first-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this.
*/}}
{{- define "my-first-chart.fullname" -}}
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
Create chart label
*/}}
{{- define "my-first-chart.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-first-chart.labels" -}}
helm.sh/chart: {{ include "my-first-chart.chart" . }}
{{ include "my-first-chart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-first-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-first-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-first-chart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-first-chart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

> **Why `_helpers.tpl`?**  
> 1. Files starting with `_` are NOT rendered as Kubernetes manifests  
> 2. They let you define **reusable snippets** so you DRY (Don't Repeat Yourself)  
> 3. The fullname logic is complex — define once, use everywhere

---

## 📖 Lecture 2.6 — NOTES.txt — Post-Install Help

After `helm install`, the content of `NOTES.txt` is displayed to the user.

```bash
{{- $svcType := .Values.service.type }}
1. Get the application URL by running these commands:
{{- if contains "NodePort" $svcType }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "my-first-chart.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" $svcType }}
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch its status by running 'kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "my-first-chart.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "my-first-chart.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" $svcType }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "my-first-chart.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
```

---

## 📖 Lecture 2.7 — The .Release, .Chart, .Values, and .Files Objects

Helm injects several objects into every template:

### Built-in Objects

```
.Release                 Information about the release
  .Release.Name          Release name ("my-app")
  .Release.Namespace     Namespace being deployed to
  .Release.IsInstall     True if this is an install operation
  .Release.IsUpgrade     True if this is an upgrade operation
  .Release.Revision      Release revision number (starts at 1)
  .Release.Service       "Helm"

.Chart                   Contents of Chart.yaml
  .Chart.Name            Name of the chart
  .Chart.Version         Chart version
  .Chart.AppVersion      App version
  .Chart.Description     Chart description

.Values                  Contents of values.yaml (merged with overrides)
  .Values.replicaCount   etc.

.Files                   Object to access non-special files in chart
  .Files.Get "config.txt"  Read file contents as string

.Capabilities            Info about Kubernetes cluster
  .Capabilities.KubeVersion.Major
  .Capabilities.KubeVersion.Minor
  .Capabilities.APIVersions.Has "apps/v1"

.Template                Info about current template file
  .Template.Name         Path of the current template
  .Template.BasePath     Base path of templates
```

---

## ✅ Module 02 — Review Questions

1. What is the difference between `chart version` and `appVersion`?
2. What does the underscore prefix in `_helpers.tpl` signify?
3. Why should you use named templates instead of repeating label definitions?
4. What object would you use to check the Kubernetes API version in a template?
5. What is the purpose of `NOTES.txt`?
6. What does `| nindent 4` do in a template, and why is it needed?
7. What does `| default .Chart.AppVersion` do in the image tag line?

---

## 🧪 Lab 02 — Create Your First Chart

**→ See [labs/lab-02-create-chart.md](./labs/lab-02-create-chart.md)**

---

*[← Module 01](../Module-01-Introduction/README.md) | [Module 03 →](../Module-03-Templates-Values/README.md)*
