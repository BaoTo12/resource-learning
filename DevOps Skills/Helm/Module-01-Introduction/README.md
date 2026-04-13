# Module 01 — Introduction to Helm & Kubernetes Packaging

> **Professor's Note:** Before you write a single line of Helm code, you must understand *why* Helm exists. Understanding the problem it solves will make every concept you learn feel intuitive rather than arbitrary.

---

## 📖 Lecture 1.1 — The Problem Helm Solves

### Life Without Helm — Raw Kubernetes YAML

Imagine you want to deploy a simple web application to Kubernetes. You need:

1. A `Deployment` — defines your application pods
2. A `Service` — exposes your application  
3. A `ConfigMap` — stores configuration
4. An `Ingress` — routes external traffic
5. A `Secret` — stores sensitive values
6. `HorizontalPodAutoscaler` — auto-scales your pods
7. `ServiceAccount` + `RBAC` — security permissions

That's **7+ YAML files** for just one application.

**Problems with raw YAML:**

| Problem | Description |
|---|---|
| **Duplication** | Copy-paste the same YAML for dev, staging, prod — just with different values |
| **No Versioning** | Hard to track which version of your config is deployed |
| **No Rollback** | Reverting a deployment means manually re-applying old YAML |
| **No Dependency Mgmt** | If your app needs a database, you manage that separately |
| **Environment Config** | Changing values for different environments requires editing files |
| **No Templating** | Same value repeated in 10 places — must change 10 times |

**Example of the pain:** You have 5 microservices × 3 environments = 15 sets of YAML files. Each service has 7 YAML files = **105 YAML files** to manage. This is chaos.

---

## 📖 Lecture 1.2 — What is Helm?

**Helm is the package manager for Kubernetes.**

Think of it like:
- **apt** / **yum** → Linux package managers
- **npm** / **pip** → Language package managers  
- **Helm** → Kubernetes package manager

### The Three Key Concepts of Helm

```
┌─────────────────────────────────────────────────────────────┐
│                        HELM UNIVERSE                        │
├──────────────┬──────────────────────┬───────────────────────┤
│   CHART      │      REPOSITORY      │       RELEASE         │
│              │                      │                       │
│ A package    │ A collection of      │ An installed          │
│ containing   │ charts (like npm     │ instance of a         │
│ K8s resource │ registry / Docker    │ chart in your         │
│ templates    │ Hub)                 │ cluster               │
│              │                      │                       │
│ = Source     │ = Registry           │ = Running App         │
└──────────────┴──────────────────────┴───────────────────────┘
```

### Core Helm Concepts Explained

**1. Chart**
```
A Helm Chart is a collection of files that describe a 
related set of Kubernetes resources.

Like a .deb or .rpm package but for Kubernetes.
```

**2. Release**
```
A Release is a specific instance of a chart that has 
been installed to your cluster.

You can install the same chart multiple times 
(each with a different release name and config).

Example:
  helm install my-nginx bitnami/nginx   ← Release: "my-nginx"
  helm install prod-nginx bitnami/nginx  ← Release: "prod-nginx"
  
Both use the SAME chart but are SEPARATE releases.
```

**3. Repository**
```
A Chart Repository is a place where packaged charts 
are stored and shared.

Like Docker Hub but for Helm Charts.
Notable repos:
  - Artifact Hub (https://artifacthub.io)
  - Bitnami Charts
  - Prometheus Community
```

---

## 📖 Lecture 1.3 — Helm Architecture

### How Helm Works (Helm 3)

```
┌──────────────────────────────────────────────────────────────┐
│                    YOUR MACHINE                              │
│                                                              │
│   ┌─────────┐     helm install     ┌────────────────────┐   │
│   │  Helm   │──────────────────────│  Chart Repository  │   │
│   │  CLI    │◄─ fetches chart ─────│  (bitnami, etc.)   │   │
│   └────┬────┘                      └────────────────────┘   │
│        │                                                     │
│        │ renders templates + applies to cluster              │
│        ▼                                                     │
│   ┌──────────────────────────────────────────────────────┐  │
│   │              KUBERNETES CLUSTER                      │  │
│   │                                                      │  │
│   │  ┌──────────┐  ┌──────────┐  ┌────────────────────┐ │  │
│   │  │Deployment│  │ Service  │  │ ConfigMap / Secret │ │  │
│   │  └──────────┘  └──────────┘  └────────────────────┘ │  │
│   │                                                      │  │
│   │  Release metadata stored as Secrets in K8s namespace │  │
│   └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

> **Important:** Helm 3 has NO server-side component (Tiller is gone since Helm 2). All state is stored as Kubernetes Secrets in the cluster.

### Helm 2 vs Helm 3 — Why This Matters

| Feature | Helm 2 | Helm 3 |
|---|---|---|
| Server component | Required Tiller in cluster | No Tiller — client only |
| Security | Tiller had broad permissions | Uses your kube context permissions |
| Release storage | ConfigMaps | Kubernetes Secrets |
| Chart API | apiVersion: v1 | apiVersion: v2 |
| Status | **Deprecated** | **Current (Use this!)** |

---

## 📖 Lecture 1.4 — Helm Anatomy (Chart Structure)

Every Helm Chart follows this directory structure:

```
my-chart/
├── Chart.yaml           ← Chart metadata (name, version, description)
├── values.yaml          ← Default configuration values
├── values.schema.json   ← (Optional) JSON Schema to validate values
├── charts/              ← Dependency charts (subcharts)
├── crds/                ← Custom Resource Definitions
├── templates/           ← Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl     ← Named template helpers (NOT rendered directly)
│   ├── NOTES.txt        ← Post-install notes shown to users
│   └── tests/           ← Helm test definitions
│       └── test-connection.yaml
└── README.md            ← Chart documentation
```

### Chart.yaml Explained

```yaml
apiVersion: v2                    # Helm 3 uses v2
name: my-webapp                   # Chart name
description: A simple web app     # Human-readable description
type: application                 # 'application' or 'library'
version: 0.1.0                    # Chart version (SemVer)
appVersion: "1.0.0"               # App version (metadata only)
keywords:
  - webapp
  - nginx
maintainers:
  - name: Your Name
    email: you@example.com
dependencies:                     # External chart dependencies
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
```

---

## 📖 Lecture 1.5 — Essential Helm CLI Commands

These are the commands you will use every single day:

### Installation & Management

```bash
# Install a chart (creates a new release)
helm install <release-name> <chart>
helm install my-app bitnami/nginx

# Install with custom values
helm install my-app bitnami/nginx --values my-values.yaml
helm install my-app bitnami/nginx --set service.type=NodePort

# Install in a specific namespace (creates namespace if --create-namespace)
helm install my-app bitnami/nginx -n production --create-namespace

# Upgrade an existing release
helm upgrade my-app bitnami/nginx --values my-values.yaml

# Install OR Upgrade (idempotent — use this in CI/CD!)
helm upgrade --install my-app bitnami/nginx --values my-values.yaml

# Uninstall a release
helm uninstall my-app
helm uninstall my-app -n production  # with namespace
```

### Inspection & Debugging

```bash
# List all releases
helm list
helm list -A           # all namespaces
helm list -n staging   # specific namespace

# Get release status
helm status my-app

# Get the values used in a release
helm get values my-app                # user-supplied values only
helm get values my-app --all          # all values including defaults

# Get rendered manifests of an installed release
helm get manifest my-app

# Show release history
helm history my-app

# Rollback to previous version
helm rollback my-app 1    # rollback to revision 1

# Dry run — see what would be deployed WITHOUT applying
helm install my-app bitnami/nginx --dry-run

# Template rendering — render templates locally without connecting to K8s
helm template my-app bitnami/nginx --values my-values.yaml

# Lint — validate chart syntax
helm lint ./my-chart
```

### Repository Management

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# List repositories
helm repo list

# Update repository index (like apt-get update)
helm repo update

# Remove a repository
helm repo remove bitnami

# Search for charts in repositories
helm search repo nginx
helm search repo nginx --versions    # show all versions
helm search hub wordpress            # search Artifact Hub
```

### Chart Development

```bash
# Create a new chart scaffold
helm create my-chart

# Package a chart into a .tgz tarball
helm package ./my-chart

# Inspect a chart's values, readme, or chart info
helm show values bitnami/nginx
helm show chart bitnami/nginx
helm show readme bitnami/nginx
helm show all bitnami/nginx

# Pull a chart locally without installing
helm pull bitnami/nginx
helm pull bitnami/nginx --untar     # extract it
```

---

## 📖 Lecture 1.6 — The Helm Template Engine

Helm uses **Go's `text/template`** package with additional Helm-specific functions.

When you run `helm install`, Helm:
1. Reads your `values.yaml` (default values)
2. Merges with any `--values` files or `--set` overrides
3. Passes the merged values through each template file
4. Sends the rendered YAML to Kubernetes API

```
values.yaml  ──►  templates/deployment.yaml  ──►  rendered YAML  ──►  K8s API
(data layer)      (template layer)                 (output)
```

### Quick Template Preview

```yaml
# templates/deployment.yaml (template)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app       # ← Helm built-in
  labels:
    app: {{ .Chart.Name }}            # ← from Chart.yaml
    version: {{ .Values.image.tag }}  # ← from values.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

```yaml
# values.yaml (data)
replicaCount: 3
image:
  repository: nginx
  tag: "1.25.0"
```

```yaml
# Rendered output (what K8s gets)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-release-app
  labels:
    app: my-chart
    version: 1.25.0
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: "nginx:1.25.0"
```

---

## ✅ Module 01 — Review Questions

Answer these to confirm your understanding before moving to Module 02:

1. What is the difference between a **Chart**, a **Release**, and a **Repository**?
2. Why was **Tiller** removed in Helm 3, and what replaced it?
3. Where does Helm 3 store release state in Kubernetes?
4. What is the purpose of `values.yaml` in a Helm Chart?
5. What does `helm upgrade --install` do, and why is it preferred in CI/CD?
6. What is the difference between `helm install --dry-run` and `helm template`?
7. Name 3 files found in every Helm chart and describe their purpose.

---

## 🧪 Lab 01 — Explore Helm

**→ See [labs/lab-01-explore-helm.md](./labs/lab-01-explore-helm.md)**

---

*[← Back to Course Overview](../README.md) | [Module 02 →](../Module-02-First-Chart/README.md)*
