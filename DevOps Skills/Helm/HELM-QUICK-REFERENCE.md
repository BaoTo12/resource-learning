# 📋 HELM QUICK REFERENCE — Professor's Cheat Sheet

> Keep this file open while you work. Every command and pattern you need, in one place.

---

## ⚡ Most Used Commands (Daily Drivers)

```bash
# === INSTALL ===
helm install <name> <chart>                        # Install chart
helm install <name> <chart> -n <namespace>         # In namespace
helm install <name> <chart> --create-namespace     # Create namespace if missing
helm install <name> <chart> -f values.yaml         # With values file
helm install <name> <chart> --set key=value        # With inline override
helm install <name> <chart> --dry-run --debug      # Dry run (preview)

# === UPGRADE ===
helm upgrade <name> <chart>                        # Upgrade release
helm upgrade --install <name> <chart>              # Install or Upgrade (CI/CD!)
helm upgrade --install <name> <chart> --atomic     # Auto-rollback on failure
helm upgrade --install <name> <chart> --wait       # Wait for readiness

# === INSPECT RELEASES ===
helm list                                          # List releases (current ns)
helm list -A                                       # All namespaces
helm list -n <namespace>                           # Specific namespace
helm status <name>                                 # Release status
helm get values <name>                             # User-supplied values
helm get values <name> --all                       # All values (merged)
helm get manifest <name>                           # Rendered manifests
helm history <name>                                # Release history

# === ROLLBACK ===
helm rollback <name>                               # Rollback to previous
helm rollback <name> <revision>                    # Rollback to specific rev

# === UNINSTALL ===
helm uninstall <name>                              # Remove release
helm uninstall <name> -n <namespace>               # From namespace

# === REPOS ===
helm repo add <name> <url>                         # Add repo
helm repo list                                     # List repos
helm repo update                                   # Refresh indexes
helm repo remove <name>                            # Remove repo
helm search repo <keyword>                         # Search repos
helm search repo <chart> --versions                # Show all versions

# === CHART INSPECTION ===
helm show chart <chart>                            # Chart metadata
helm show values <chart>                           # Default values
helm show readme <chart>                           # Chart README

# === DEVELOPMENT ===
helm create <name>                                 # Scaffold new chart
helm lint <path>                                   # Validate chart
helm template <name> <path>                        # Render locally (no K8s)
helm template <name> <path> -f values.yaml         # Render with values
helm package <path>                                # Package to .tgz

# === DEPENDENCIES ===
helm dependency update <path>                      # Download deps
helm dependency build <path>                       # Build from lock file
helm dependency list <path>                        # List deps

# === TESTING ===
helm test <name>                                   # Run Helm tests
helm test <name> --logs                            # Show test logs
```

---

## 📌 --set Cheat Sheet

```bash
--set key=value                          # String/int value
--set key="value with spaces"            # Quoted value
--set-string key=1.25                    # Force string type
--set nested.key=value                   # Nested value
--set list[0]=a --set list[1]=b         # List values
--set map.key1=v1 --set map.key2=v2     # Map values
--set "annotations.k8s\.io/ann=val"     # Key with dot (escape!)
```

---

## 🏗️ Chart File Reference

```
my-chart/
├── Chart.yaml           Required: metadata (name, version, apiVersion)
├── values.yaml          Required: default configuration values
├── values.schema.json   Optional: JSON Schema for value validation
├── charts/              Dependencies (auto-populated by dep update)
├── crds/                Custom Resource Definitions
├── templates/           Kubernetes manifest templates
│   ├── _helpers.tpl     NOT rendered — named template definitions
│   ├── NOTES.txt        Post-install user message
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   └── tests/
│       └── test-*.yaml  Helm test pods/jobs
└── README.md
```

---

## 🔧 Template Syntax Quick Ref

```gotemplate
{{ .Values.key }}                   # Access value
{{ .Release.Name }}                 # Release name
{{ .Release.Namespace }}            # Namespace
{{ .Chart.Name }}                   # Chart name
{{ .Chart.Version }}                # Chart version
{{ .Chart.AppVersion }}             # App version

{{ include "chart.helper" . }}      # Call named template
{{ define "chart.helper" }}         # Define named template
{{- end }}                          # End block

{{- if .Values.enabled }}           # Conditional (- trims whitespace)
{{- else if condition }}            # Else if
{{- else }}                         # Else
{{- end }}

{{- with .Values.block }}           # With (changes scope to .Values.block)
  value: {{ .field }}               # . is .Values.block here
{{- end }}

{{- range .Values.list }}           # Range over list
  item: {{ . }}                     # . is current item
{{- end }}

{{- range $k, $v := .Values.map }} # Range over map
  {{ $k }}: {{ $v }}
{{- end }}

{{- $var := "value" }}              # Variable
{{ $var }}                          # Use variable
{{ $ }}                             # Root context (use inside with/range)
```

---

## 🔩 Common Template Functions

```gotemplate
# String
{{ "val" | upper }}               # UPPERCASE
{{ "val" | lower }}               # lowercase
{{ "val" | title }}               # Title Case
{{ "val" | quote }}               # "quoted"
{{ "val" | replace "a" "b" }}    # Replace
{{ "hello" | trunc 3 }}           # Truncate
{{ printf "%s-%s" "a" "b" }}     # Format string
{{ "val" | trimSuffix "-end" }}   # Trim suffix

# Numbers
{{ add 1 .Values.count }}         # Addition
{{ gt .Values.replicas 1 }}       # greater than
{{ lt .Values.replicas 5 }}       # less than
{{ max 1 .Values.min }}           # maximum value

# Lists
{{ list "a" "b" | join "," }}     # Join list
{{ list "a" "b" | has "a" }}      # Contains
{{ list "a" "b" | first }}        # First element
{{ list "a" "b" | len }}          # Length

# YAML/JSON
{{ .Values.obj | toYaml }}        # To YAML string
{{ .Values.obj | toJson }}        # To JSON string

# Types
{{ "true" | toBool }}             # To bool
{{ 42 | toString }}               # To string

# Encoding
{{ "text" | b64enc }}             # Base64 encode
{{ "dGV4dA==" | b64dec }}        # Base64 decode
{{ "text" | sha256sum }}          # SHA256 hash

# Defaults  
{{ .Values.x | default "fallback" }}     # Default if empty
{{ .Values.x | required "msg" }}         # Fail if empty

# Indentation (for YAML)
{{ include "helper" . | nindent 4 }}     # Newline + 4-space indent
{{ include "helper" . | indent 4 }}      # 4-space indent only

# Randm
{{ randAlphaNum 16 }}             # Random 16-char alphanumeric
{{ uuidv4 }}                      # Random UUID

# Helm-specific
{{ tpl .Values.template . }}      # Render value as template
{{ fail "error message" }}        # Fail with message
{{ lookup "v1" "Secret" "ns" "name" }}  # K8s API lookup
```

---

## 📜 Hook Annotations Reference

```yaml
metadata:
  annotations:
    # When to run:
    helm.sh/hook: pre-install            # Before first install
    helm.sh/hook: post-install           # After first install
    helm.sh/hook: pre-upgrade            # Before upgrade
    helm.sh/hook: post-upgrade           # After upgrade
    helm.sh/hook: pre-delete             # Before uninstall
    helm.sh/hook: post-delete            # After uninstall
    helm.sh/hook: pre-rollback           # Before rollback
    helm.sh/hook: post-rollback          # After rollback
    helm.sh/hook: test                   # On helm test

    # Multiple events:
    helm.sh/hook: pre-install,pre-upgrade

    # Execution order (lower = first):
    helm.sh/hook-weight: "-5"

    # Cleanup policy:
    helm.sh/hook-delete-policy: before-hook-creation   # Default
    helm.sh/hook-delete-policy: hook-succeeded
    helm.sh/hook-delete-policy: hook-failed
    # Combine: (most common for jobs)
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
```

---

## 🌍 Global Values Pattern

```yaml
# Parent chart values.yaml
global:
  imageRegistry: ""
  imagePullSecrets: []
  environment: production
  clusterName: "my-cluster"

# Access in any chart/subchart:
{{ .Values.global.imageRegistry }}
{{ .Values.global.environment }}
```

---

## 📦 Dependency Config (Chart.yaml)

```yaml
dependencies:
  - name: postgresql              # Chart name in repo
    version: "12.x.x"            # Version or range
    repository: https://...       # Repo URL
    condition: postgresql.enabled # Enable/disable flag
    alias: postgres-main          # Override chart name
    tags:                         # Grouping tags
      - database
```

---

## 🔐 Security Context Template (Production)

```yaml
# In values.yaml (production):
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]

# In deployment template:
spec:
  securityContext:
    {{- toYaml .Values.podSecurityContext | nindent 8 }}
  containers:
    - securityContext:
        {{- toYaml .Values.securityContext | nindent 12 }}
```

---

## 🏗️ Production Upgrade Pattern

```bash
helm upgrade --install <name> <chart> \
  -n <namespace> \
  --create-namespace \
  --values values.yaml \
  --values values-production.yaml \
  --set image.tag=$IMAGE_TAG \
  --atomic \              # Auto-rollback on failure
  --timeout 10m \         # Fail if not ready in 10 min
  --cleanup-on-fail \     # Clean up new resources on failure
  --wait \                # Wait for all pods to be ready
  --history-max 10        # Keep only 10 revisions
```

---

## 🔍 Debugging Commands

```bash
# Render templates WITHOUT applying (no cluster needed)
helm template <name> <chart> --debug

# Dry run (with cluster connection, validating with K8s)
helm install <name> <chart> --dry-run --debug

# Show what values are actually in use
helm get values <name> --all

# Show what manifests are deployed
helm get manifest <name>

# Show hooks
helm get hooks <name>

# Show notes
helm get notes <name>

# Show full release info
helm get all <name>

# View hook job logs
kubectl logs -l helm.sh/chart=my-chart -n my-ns

# List releases in all states (including failed)
helm list -a -A
```

---

## 🌐 OCI Registry Commands

```bash
# Login
helm registry login <registry> -u <user> --password-stdin

# Push chart
helm push my-chart-1.0.0.tgz oci://ghcr.io/user/charts

# Install from OCI
helm install <name> oci://ghcr.io/user/charts/my-chart --version 1.0.0

# Show OCI chart info
helm show chart oci://ghcr.io/user/charts/my-chart

# Pull from OCI
helm pull oci://ghcr.io/user/charts/my-chart --version 1.0.0 --untar
```

---

## 🏷️ Helm Built-in Objects Summary

| Object | Key Fields |
|---|---|
| `.Release` | `.Name`, `.Namespace`, `.Revision`, `.Service`, `.IsInstall`, `.IsUpgrade` |
| `.Chart` | `.Name`, `.Version`, `.AppVersion`, `.Description` |
| `.Values` | All values from values.yaml + overrides |
| `.Files` | `.Get "path"`, `.Glob "*.conf"`, `.AsConfig`, `.AsSecrets` |
| `.Capabilities` | `.KubeVersion.Major`, `.KubeVersion.Minor`, `.APIVersions.Has "..."` |
| `.Template` | `.Name`, `.BasePath` |

---

*[← Back to Course](./README.md)*
