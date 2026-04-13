# Module 03 — Helm Templates & Values Deep Dive

> **Professor's Note:** This is the most important module of the course. Everything in Helm ultimately comes down to templates and values. Master this module and the rest becomes intuitive.

---

## 📖 Lecture 3.1 — The Go Template Language

Helm uses Go's `text/template` package. All template actions are wrapped in double curly braces:

```
{{ action }}
```

### 3.1.1 — Whitespace Control

Whitespace in YAML matters. Helm provides whitespace control using dashes:

```gotemplate
{{- action -}}
  ^          ^
  |          └── Trim whitespace AFTER the action
  └────────────── Trim whitespace BEFORE the action
```

**Example: Why this matters**

```gotemplate
# WITHOUT whitespace control — produces extra blank lines
---
metadata:
  name: my-app

  labels:
    {{- include "labels" . | nindent 4 }}
```

```gotemplate
# WITH whitespace control — clean output
---
metadata:
  name: my-app
  labels:
    {{- include "labels" . | nindent 4 }}
```

**Rule of Thumb:**
- Use `{{-` on `if`, `range`, `with`, `end` statements to trim preceding newline
- This keeps your rendered YAML clean

---

## 📖 Lecture 3.2 — Variables in Templates

```gotemplate
# Assign to a variable
{{- $fullname := include "webapp.fullname" . -}}

# Use the variable
name: {{ $fullname }}

# Variables in range loops
{{- range $key, $value := .Values.annotations }}
  {{ $key }}: {{ $value }}
{{- end }}
```

### Scope and the Dollar Sign

```gotemplate
# The dot (.) represents current scope
{{ .Values.replicaCount }}  ← Works in top-level scope

# Inside a "with" block, the . scope changes!
{{- with .Values.service }}
  type: {{ .type }}   ← . here is .Values.service, NOT the root!
  port: {{ .port }}
{{- end }}

# To access root scope from within with/range, use $
{{- with .Values.service }}
  name: {{ $.Release.Name }}   ← $ always refers to root scope
  type: {{ .type }}
{{- end }}
```

---

## 📖 Lecture 3.3 — Conditionals: if / else if / else

```gotemplate
{{- if .Values.ingress.enabled }}
# Create ingress block...
{{- else if .Values.service.type "LoadBalancer" | eq }}
# Different configuration...
{{- else }}
# Default case
{{- end }}
```

### Truthiness in Helm Templates

| Value | Treated as |
|---|---|
| `false` | False |
| `0` | False |
| `""` (empty string) | False |
| `null` / `nil` | False |
| `[]` (empty list) | False |
| `{}` (empty map) | False |
| Everything else | True |

### Practical Examples

```gotemplate
# Only create HPA if autoscaling is enabled AND replicaCount field not nil
{{- if and .Values.autoscaling.enabled (not (eq .Values.replicaCount nil)) }}

# Check if a value is set
{{- if .Values.extraEnvVars }}
env:
  {{- toYaml .Values.extraEnvVars | nindent 10 }}
{{- end }}

# Conditional with comparison operators
{{- if gt .Values.replicaCount 1 }}
# replica count is greater than 1
{{- end }}

{{- if eq .Values.service.type "LoadBalancer" }}
# service type is LoadBalancer
{{- end }}
```

### Comparison and Logic Operators

```gotemplate
# Comparison (note: these are functions called in prefix notation)
{{- if eq .Values.env "production" }}   # equals
{{- if ne .Values.env "production" }}   # not equals
{{- if lt .Values.replicas 3 }}        # less than
{{- if le .Values.replicas 3 }}        # less than or equal
{{- if gt .Values.replicas 1 }}        # greater than
{{- if ge .Values.replicas 1 }}        # greater than or equal

# Logic operators
{{- if and condition1 condition2 }}
{{- if or condition1 condition2 }}
{{- if not condition }}

# Combining
{{- if and (eq .Values.env "production") (gt .Values.replicas 1) }}
```

---

## 📖 Lecture 3.4 — Iteration: range

### Ranging Over a List

```yaml
# values.yaml
hostAliases:
  - ip: "127.0.0.1"
    hostnames: ["localhost", "myapp.local"]
  - ip: "10.0.0.1"
    hostnames: ["db.internal"]
```

```gotemplate
# template
{{- if .Values.hostAliases }}
hostAliases:
  {{- range .Values.hostAliases }}
  - ip: {{ .ip | quote }}
    hostnames:
      {{- range .hostnames }}
      - {{ . | quote }}
      {{- end }}
  {{- end }}
{{- end }}
```

### Ranging Over a Map (key-value)

```yaml
# values.yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  custom-annotation: my-value
```

```gotemplate
# template
annotations:
  {{- range $key, $val := .Values.annotations }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

### Range with Index

```yaml
# values.yaml
initContainers:
  - name: init-db
    image: busybox
    command: ["sh", "-c", "until nc -z db 5432; do sleep 2; done"]
  - name: init-config
    image: busybox
    command: ["sh", "-c", "cp /src/* /dest/"]
```

```gotemplate
{{- if .Values.initContainers }}
initContainers:
  {{- range $index, $container := .Values.initContainers }}
  - name: {{ $container.name }}
    image: {{ $container.image }}
    command:
      {{- toYaml $container.command | nindent 6 }}
  {{- end }}
{{- end }}
```

---

## 📖 Lecture 3.5 — The `with` Block

`with` is like `if` but also **changes the scope** to the value being tested:

```gotemplate
# Without with — verbose
{{- if .Values.podSecurityContext }}
securityContext:
  runAsNonRoot: {{ .Values.podSecurityContext.runAsNonRoot }}
  fsGroup: {{ .Values.podSecurityContext.fsGroup }}
{{- end }}

# With with — cleaner, more idiomatic
{{- with .Values.podSecurityContext }}
securityContext:
  runAsNonRoot: {{ .runAsNonRoot }}   ← . is .Values.podSecurityContext here
  fsGroup: {{ .fsGroup }}
{{- end }}

# Most idiomatic for YAML blocks:
{{- with .Values.podSecurityContext }}
securityContext:
  {{- toYaml . | nindent 2 }}   ← dump entire block as YAML
{{- end }}
```

---

## 📖 Lecture 3.6 — Template Functions

Helm includes 60+ template functions (from Sprig library + Helm-specific ones).

### String Functions

```gotemplate
{{ "hello world" | upper }}         → "HELLO WORLD"
{{ "hello world" | lower }}         → "hello world"
{{ "hello world" | title }}         → "Hello World"
{{ "hello world" | replace " " "-" }} → "hello-world"
{{ "  hello  " | trim }}            → "hello"
{{ "hello-world" | trimSuffix "-world" }} → "hello"
{{ "hello" | repeat 3 }}            → "hellohellohello"
{{ "hello world" | contains "world" }} → true
{{ "my-release" | trunc 63 }}       → "my-release" (safe for K8s names)
{{ printf "%s-%s" "foo" "bar" }}    → "foo-bar"
{{ "my string" | quote }}           → "\"my string\""
{{ "my string" | squote }}          → "'my string'"
```

### Number Functions

```gotemplate
{{ .Values.port | int }}             → integer cast
{{ .Values.port | int64 }}          → int64 cast
{{ add 1 .Values.replicaCount }}    → arithmetic
{{ max 1 .Values.minReplicas }}     → maximum value
{{ min 100 .Values.maxReplicas }}   → minimum value
{{ ceil .Values.percentage }}       → ceiling
{{ floor .Values.percentage }}      → floor
```

### List Functions

```gotemplate
{{ list "a" "b" "c" }}              → [a b c]
{{ list "a" "b" | join ", " }}      → "a, b"
{{ list "a" "b" | first }}          → "a"
{{ list "a" "b" | last }}           → "b"
{{ list "a" "b" "c" | len }}        → 3
{{ list "a" "b" | has "a" }}        → true
{{ list "a" "b" | without "a" }}    → [b]
{{ list "a" "b" | uniq }}           → [a b] (remove duplicates)
```

### Dictionary Functions

```gotemplate
{{ dict "key1" "val1" "key2" "val2" }}   → map
{{ .Values.labels | keys }}             → list of keys
{{ .Values.labels | values }}           → list of values
{{ hasKey .Values "replicas" }}         → true/false
{{ .Values | merge $defaults }}         → merge two dicts
```

### Type Conversion

```gotemplate
{{ "true" | toBool }}               → true
{{ 42 | toString }}                 → "42"
{{ .Values.anything | toYaml }}     → YAML string
{{ .Values.anything | toJson }}     → JSON string
{{ .Values.anything | toPrettyJson }} → pretty JSON
```

### Date Functions

```gotemplate
{{ now | date "2006-01-02" }}       → current date
{{ now | date "2006-01-02T15:04:05Z07:00" }} → ISO 8601
{{ now | unixEpoch }}               → Unix timestamp
{{ now | dateModify "-1h" }}        → subtract 1 hour
```

### Encoding Functions

```gotemplate
{{ "hello" | b64enc }}              → "aGVsbG8="  (Base64 encode)
{{ "aGVsbG8=" | b64dec }}          → "hello"     (Base64 decode)
{{ "hello" | sha256sum }}           → SHA256 hash
{{ "hello" | sha1sum }}             → SHA1 hash
```

### Kubernetes Specific

```gotemplate
{{ .Values.port | atoi }}           → string to int
{{ randAlphaNum 16 }}               → 16 random alphanumeric chars
{{ uuidv4 }}                        → random UUID
{{ lookup "v1" "Secret" "default" "my-secret" }}  → K8s API lookup
```

---

## 📖 Lecture 3.7 — Values Deep Dive

### Values Hierarchy (Precedence Order — High to Low)

```
1. --set flags (highest precedence)
     ↓
2. --values / -f files (processed left to right, last wins)
     ↓
3. Parent chart's values.yaml (for subcharts)
     ↓
4. Chart's values.yaml (lowest precedence — defaults)
```

### --set Syntax

```bash
# Simple value
--set replicaCount=3

# Nested value (uses dot notation)
--set image.tag=1.25.0

# List value (use curly braces and comma)
--set imagePullSecrets={secret1,secret2}

# List of objects (use index)
--set ingress.hosts[0].host=myapp.com
--set ingress.hosts[0].paths[0].path=/

# Values with special characters (escape with backslash)
--set annotation."kubernetes\.io/ingress\.class"=nginx

# Comma-separated multiple values
--set replicaCount=3,image.tag=1.25.0

# Force string type (even if value looks like a number)
--set-string image.tag=1.25   # prevents "1.25" → 1 (float) conversion
```

### --set vs --values

| Scenario | Use |
|---|---|
| Quick one-off override | `--set` |
| Multiple values / complex config | `--values file.yaml` |
| CI/CD with environment-specific configs | `--values base.yaml --values env.yaml` |
| Sensitive values (passwords) | `--set` or separate secrets management |

### Values File Override Pattern (Best Practice)

```bash
# Layer values files — later files override earlier ones
helm install my-app ./webapp \
  --values values.yaml \           # base defaults
  --values values-staging.yaml \   # staging overrides
  --values secrets.yaml            # sensitive overrides (never commit!)
```

---

## 📖 Lecture 3.8 — Values Schema Validation (values.schema.json)

You can add a JSON Schema file to validate user-supplied values:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["replicaCount", "image"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 20,
      "description": "Number of replicas to deploy"
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "description": "Docker image repository"
        },
        "tag": {
          "type": "string",
          "description": "Docker image tag"
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
    }
  }
}
```

If a user provides an invalid value, Helm will reject it with a clear error:
```
Error: values don't meet the specifications of the schema(s) in the following chart(s):
webapp:
- replicaCount: Must be of type integer
- image.pullPolicy: image.pullPolicy must be one of the following: "Always", "IfNotPresent", "Never"
```

---

## 📖 Lecture 3.9 — The `lookup` Function

The `lookup` function queries the live Kubernetes cluster for existing resources:

```gotemplate
{{- $secret := lookup "v1" "Secret" .Release.Namespace "my-db-password" -}}
{{- if $secret }}
  # Secret exists — use its value
  password: {{ $secret.data.password }}
{{- else }}
  # Secret doesn't exist — generate a random one
  password: {{ randAlphaNum 32 | b64enc }}
{{- end }}
```

> **Warning:** `lookup` returns empty dict during `helm template` and `--dry-run` since there's no live cluster. Always handle the "not found" case.

---

## ✅ Module 03 — Review Questions

1. What does `{{-` (with leading dash) do in a template?
2. What is the `$` variable and when do you need to use it?
3. What's the difference between `with` and `if`?
4. How do you iterate over a YAML map in a template? Write an example.
5. What does `| nindent 4` do and why is it needed?
6. What is the values precedence order in Helm?
7. What's the difference between `--set` and `--set-string`?
8. What files does Helm process to build the final values object?
9. How would you validate that `service.type` is one of ClusterIP, NodePort, or LoadBalancer?

---

## 🧪 Lab 03 — Advanced Templates & Values

**→ See [labs/lab-03-templates.md](./labs/lab-03-templates.md)**

---

*[← Module 02](../Module-02-First-Chart/README.md) | [Module 04 →](../Module-04-Chart-Lifecycle/README.md)*
