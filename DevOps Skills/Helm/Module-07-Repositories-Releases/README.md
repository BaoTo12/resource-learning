# Module 07 — Helm Repositories, OCI Registries & Release Management

> **Professor's Note:** So far we've been consumers and local chart creators. Now it's time to become publishers. This module teaches you how to host charts, publish them, and manage releases professionally — including using OCI (Docker-compatible) registries, which is the modern Helm 3 approach.

---

## 📖 Lecture 7.1 — How Helm Repositories Work

A Helm repository is an HTTP server that:
1. Hosts `.tgz` packaged chart files
2. Provides an `index.yaml` file listing all charts and versions

```
https://charts.example.com/
├── index.yaml                     ← Chart index (auto-generated)
├── my-webapp-1.0.0.tgz
├── my-webapp-1.1.0.tgz
├── my-webapp-2.0.0.tgz
└── postgresql-12.5.6.tgz
```

**The index.yaml:**
```yaml
apiVersion: v1
entries:
  my-webapp:
    - apiVersion: v2
      appVersion: 1.0.0
      created: "2024-01-15T10:30:00Z"
      description: A simple web application
      digest: sha256:abc123...
      name: my-webapp
      urls:
        - https://charts.example.com/my-webapp-1.0.0.tgz
      version: 1.0.0
    - apiVersion: v2
      appVersion: 1.1.0
      created: "2024-02-01T10:30:00Z"
      name: my-webapp
      urls:
        - https://charts.example.com/my-webapp-1.1.0.tgz
      version: 1.1.0
generated: "2024-02-01T12:00:00Z"
```

---

## 📖 Lecture 7.2 — Publishing Charts via GitHub Pages

**GitHub Pages is the most common free Helm chart hosting.**

### Method 1: GitHub Pages Repository

**Step 1:** Create a GitHub repository. Enable GitHub Pages on `gh-pages` branch or `/docs` folder.

**Step 2:** Package your chart:
```bash
helm package ./my-webapp --destination ./charts-repo
```

**Step 3:** Generate the index:
```bash
# In your charts-repo directory
helm repo index ./charts-repo --url https://YOUR_USERNAME.github.io/YOUR_REPO
```

**Step 4:** Commit and push to GitHub:
```bash
git add charts-repo/
git commit -m "Add my-webapp v1.0.0"
git push
```

**Step 5:** Add the repo and use it:
```bash
helm repo add my-charts https://YOUR_USERNAME.github.io/YOUR_REPO
helm repo update
helm install my-app my-charts/my-webapp
```

### Automating with GitHub Actions

```yaml
# .github/workflows/release.yaml
name: Release Helm Charts

on:
  push:
    branches: [main]
    paths:
      - 'charts/**'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.5.0
        with:
          charts_dir: charts
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

---

## 📖 Lecture 7.3 — OCI Registries (Modern Approach)

Since Helm 3.8+, Helm supports **OCI (Open Container Initiative) compatible registries** for storing charts. This is the future of chart distribution.

```
OCI Registry
    ├── nginx:1.0.0        ← Docker image
    ├── nginx:1.1.0        ← Docker image
    ├── my-webapp:1.0.0    ← Helm chart (stored as OCI artifact!)
    └── my-webapp:1.1.0    ← Helm chart
```

**Why OCI?**
- Single registry for both Docker images AND Helm charts
- Better security (signing, attestations)
- Better access control
- Supported by: DockerHub, GitHub GHCR, AWS ECR, Google Artifact Registry, Azure ACR

### OCI Registry Commands

```bash
# The OCI:// prefix instead of https://
# NO `helm repo add` needed for OCI registries!

# Login to OCI registry
echo $GITHUB_TOKEN | helm registry login ghcr.io -u USERNAME --password-stdin
helm registry login registry-1.docker.io -u USERNAME --password-stdin

# Push a chart to OCI registry
helm package ./my-webapp
helm push my-webapp-1.0.0.tgz oci://ghcr.io/YOUR_USERNAME/helm-charts

# Pull and install from OCI
helm install my-app oci://ghcr.io/YOUR_USERNAME/helm-charts/my-webapp
helm install my-app oci://ghcr.io/YOUR_USERNAME/helm-charts/my-webapp --version 1.0.0

# Show chart info from OCI
helm show chart oci://ghcr.io/YOUR_USERNAME/helm-charts/my-webapp

# Pull without installing
helm pull oci://ghcr.io/YOUR_USERNAME/helm-charts/my-webapp
helm pull oci://ghcr.io/YOUR_USERNAME/helm-charts/my-webapp --version 1.0.0 --untar

# List versions (requires registry CLI, not Helm)
# For GHCR: use GitHub API
# For AWS ECR: use aws ecr list-images
```

### Popular OCI Registries

| Registry | URL Pattern | Notes |
|---|---|---|
| GitHub GHCR | `oci://ghcr.io/` | Free for public, paid private |
| DockerHub | `oci://registry-1.docker.io/` | Standard Docker Hub |
| AWS ECR | `oci://<account>.dkr.ecr.<region>.amazonaws.com/` | Use `aws ecr get-login-password` |
| Google Artifact Registry | `oci://<region>-docker.pkg.dev/<project>/` | Use `gcloud auth` |
| Azure ACR | `oci://<registry>.azurecr.io/` | Use `az acr login` |

---

## 📖 Lecture 7.4 — Artifact Hub — The Helm Chart Registry

[Artifact Hub](https://artifacthub.io) is the central discovery platform for Helm charts.

### Publishing to Artifact Hub

1. Host your chart repository (GitHub Pages, OCI, etc.)
2. Add an `artifacthub-repo.yml` to your repo root:

```yaml
# artifacthub-repo.yml
repositoryID: YOUR_REPO_UUID
owners:
  - name: Your Name
    email: you@example.com
```

3. Register your repository at [artifacthub.io/control-panel/repositories](https://artifacthub.io/control-panel/repositories)
4. Charts are indexed automatically

### Chart Annotations for Artifact Hub

Add these to `Chart.yaml` for better Artifact Hub presentation:

```yaml
annotations:
  artifacthub.io/license: Apache-2.0
  artifacthub.io/readme: "https://raw.githubusercontent.com/YOUR/REPO/main/README.md"
  artifacthub.io/changes: |
    - kind: added
      description: Added PostgreSQL support
    - kind: fixed
      description: Fixed health check path
  artifacthub.io/screenshots: |
    - title: Application Dashboard
      url: https://example.com/screenshot.png
  artifacthub.io/links: |
    - name: Documentation
      url: https://docs.example.com
    - name: Source
      url: https://github.com/example/chart
  artifacthub.io/maintainers: |
    - name: Your Name
      email: you@example.com
```

---

## 📖 Lecture 7.5 — Release Management Deep Dive

### Understanding Revisions

Every Helm operation creates a new **revision**:

```bash
helm install my-app ./chart   # Revision 1
helm upgrade my-app ./chart   # Revision 2
helm rollback my-app 1        # Revision 3 (rollback creates a NEW revision!)
helm upgrade my-app ./chart   # Revision 4
```

```bash
# Full revision history
helm history my-app -n production

# REVISION  UPDATED                  STATUS     CHART       APP VERSION  DESCRIPTION
# 1         Sat Jan 15 10:00:00 2024 superseded webapp-1.0  1.0.0        Install complete
# 2         Sat Jan 15 11:00:00 2024 superseded webapp-1.1  1.1.0        Upgrade complete
# 3         Sat Jan 15 12:00:00 2024 superseded webapp-1.0  1.0.0        Rollback to 1
# 4         Sat Jan 15 13:00:00 2024 deployed   webapp-1.2  1.2.0        Upgrade complete
```

### Rollback Options

```bash
# Rollback to previous revision
helm rollback my-app

# Rollback to specific revision
helm rollback my-app 2

# Rollback in a namespace
helm rollback my-app 2 -n production

# Dry run rollback (see what would change)
helm rollback my-app 2 --dry-run

# Rollback timeout
helm rollback my-app 2 --timeout 5m

# Rollback and wait for resources to be ready
helm rollback my-app 2 --wait
```

### Release State Machine

```
PENDING_INSTALL → DEPLOYED         (successful install)
PENDING_INSTALL → FAILED           (install failed)
PENDING_UPGRADE → DEPLOYED         (successful upgrade)
PENDING_UPGRADE → FAILED           (upgrade failed — auto-rollback)
DEPLOYED        → SUPERSEDED       (when upgraded, old revision becomes superseded)
DEPLOYED        → UNINSTALLED      (when uninstalled with --keep-history)
```

---

## 📖 Lecture 7.6 — Upgrade Strategies & Atomic Deploys

### `--atomic` Flag (Recommended for Production)

```bash
helm upgrade my-app ./chart \
  --atomic \              # Auto-rollback on failure
  --timeout 5m \         # Wait up to 5 minutes
  --wait                 # Wait for all pods to be ready
```

With `--atomic`:
- Helm waits for all resources to be Ready
- If ANY resource fails to become ready within timeout → auto-rollback to previous revision
- The failed revision is deleted

### `--cleanup-on-fail`

```bash
helm upgrade my-app ./chart \
  --cleanup-on-fail   # Delete resources created in this upgrade if it fails
```

### Smart Upgrade with Status Check

```bash
# Check current status before upgrading
helm status my-app -n production

# Get current revision
helm history my-app -n production | tail -1

# Upgrade with all safety options
helm upgrade my-app ./chart \
  -n production \
  --values prod-values.yaml \
  --atomic \
  --timeout 10m \
  --cleanup-on-fail \
  --wait \
  --debug
```

---

## 📖 Lecture 7.7 — Helm Secrets Management

**Never put secrets in `values.yaml` committed to Git!**

### Approach 1: External Secrets (Best Practice)

Use [External Secrets Operator](https://external-secrets.io/):

```yaml
# ExternalSecret — pulls from AWS Secrets Manager / Vault / etc.
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: my-app-secrets
  data:
    - secretKey: db-password
      remoteRef:
        key: production/my-app/database
        property: password
```

### Approach 2: Helm-Secrets Plugin

```bash
# Install the plugin
helm plugin install https://github.com/jkroepke/helm-secrets

# Encrypt secrets file with age or GPG
helm secrets enc secrets.yaml

# Use encrypted secrets file
helm upgrade my-app ./chart \
  --values values.yaml \
  --values secrets://secrets.yaml   # transparent decryption!
```

### Approach 3: `--set` from CI/CD Secrets

```bash
# In GitHub Actions:
helm upgrade my-app ./chart \
  --set database.password="${{ secrets.DB_PASSWORD }}" \
  --set api.key="${{ secrets.API_KEY }}"
```

---

## ✅ Module 07 — Review Questions

1. What files make up a Helm repository?
2. What does `helm repo index` do?
3. What is the difference between a traditional Helm HTTP repository and an OCI registry?
4. Why don't you need `helm repo add` for OCI charts?
5. What is Artifact Hub and what problem does it solve?
6. When you do `helm rollback`, does it go BACK in revision number? What actually happens?
7. What does `--atomic` do for `helm upgrade`?
8. Why should secrets never be stored in `values.yaml` committed to Git?

---

## 🧪 Lab 07 — Repositories & Releases

**→ See [labs/lab-07-repos-releases.md](./labs/lab-07-repos-releases.md)**

---

*[← Module 06](../Module-06-Advanced-Templates/README.md) | [Module 08 →](../Module-08-Production/README.md)*
