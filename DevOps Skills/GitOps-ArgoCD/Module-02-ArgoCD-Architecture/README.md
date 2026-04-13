# Module 02 — ArgoCD Architecture: How It Really Works

> **Professor's Note:** Understanding ArgoCD's internal architecture makes you a better operator. When something breaks — and it will — knowing which component failed tells you exactly where to look. This module is your ArgoCD internals deep-dive.

---

## 📖 Lecture 2.1 — ArgoCD Components Map

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          ARGOCD CLUSTER                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     argocd Namespace                            │   │
│  │                                                                 │   │
│  │   ┌──────────────┐    API/gRPC    ┌──────────────────────────┐ │   │
│  │   │  argocd-      │◄─────────────►│  argocd-server           │ │   │
│  │   │  cli / UI     │               │  (HTTP/gRPC gateway)     │ │   │
│  │   └──────────────┘               └────────────┬─────────────┘ │   │
│  │                                               │               │   │
│  │   ┌──────────────────────────────────────────▼─────────────┐  │   │
│  │   │                  argocd-repo-server                     │  │   │
│  │   │  • Clones Git repos (cached)                           │  │   │
│  │   │  • Renders manifests (Helm, Kustomize, plain YAML)     │  │   │
│  │   │  • Returns rendered YAML to controller                 │  │   │
│  │   └──────────────────────────────────────────┬─────────────┘  │   │
│  │                                               │               │   │
│  │   ┌─────────────────────────────────────────▼──────────────┐  │   │
│  │   │            argocd-application-controller               │  │   │
│  │   │  • Core reconciliation loop                            │  │   │
│  │   │  • Compares Git state vs Live state                    │  │   │
│  │   │  • Applies sync operations                             │  │   │
│  │   │  • Updates Application status                          │  │   │
│  │   └──────────────────────────────────────────┬─────────────┘  │   │
│  │                                               │               │   │
│  │   ┌─────────────────┐  ┌─────────────────────▼──────────────┐ │   │
│  │   │  argocd-redis   │  │     argocd-dex-server              │ │   │
│  │   │  (state cache)  │  │     (SSO / OIDC provider)          │ │   │
│  │   └─────────────────┘  └────────────────────────────────────┘ │   │
│  │                                                                 │   │
│  │   ┌─────────────────────────────────────────────────────────┐  │   │
│  │   │           argocd-applicationset-controller              │  │   │
│  │   │  • Generates Applications from templates                │  │   │
│  │   │  • Handles generators (Git, List, Cluster, Matrix...)   │  │   │
│  │   └─────────────────────────────────────────────────────────┘  │   │
│  │                                                                 │   │
│  │   ┌─────────────────────────────────────────────────────────┐  │   │
│  │   │           argocd-notifications-controller               │  │   │
│  │   │  • Sends Slack/email/webhook on ArgoCD events           │  │   │
│  │   └─────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 📖 Lecture 2.2 — Component Deep Dives

### 1. argocd-server (The API Gateway)

**What it does:**
- Serves the Web UI (`https://argocd-server:443`)
- Exposes the gRPC API used by the ArgoCD CLI
- Handles authentication (local users + SSO via Dex)
- Proxies Kubernetes API calls for the UI

**Key facts:**
- This is the only component exposed to external users
- It is stateless — horizontally scalable
- Communicates with ArgoCD's state stored in CRDs (Custom Resource Definitions) in Kubernetes

### 2. argocd-repo-server (The Git Worker)

**What it does:**
- Clones and caches Git repositories
- Renders manifests from source types: plain YAML, Helm, Kustomize, Jsonnet
- Returns the final rendered YAML manifests to the application controller

**Key facts:**
- Heavy component — responsible for all repo I/O
- Each repo is cloned locally and cached in `/tmp/argocd-repo`
- Supports plugins for custom manifest formats
- Scale horizontally for many repos

### 3. argocd-application-controller (The Brain)

**What it does:**
- The core reconciliation engine
- Continuously watches Applications (Kubernetes CRDs)
- Calls repo-server to get desired state from Git
- Queries live Kubernetes resources
- Computes the diff (what needs to change)
- Applies changes during sync
- Updates Application status (Synced, OutOfSync, Degraded, etc.)

**Key facts:**
- Runs as a StatefulSet (single instance with pod index 0)
- One instance handles all Applications by default (sharding available for scale)
- This is the component that implements the reconciliation loop

### 4. argocd-redis (The Cache)

- Caches application state, resource data, and repo metadata
- Standard Redis — stateful
- If Redis goes down, ArgoCD still works (degrades to slower re-fetching)

### 5. argocd-dex-server (The SSO Hub)

- Implements OpenID Connect (OIDC) for single sign-on
- Integrates with: GitHub, GitLab, Google, Okta, LDAP/AD, SAML
- Proxies authentication requests to external identity providers
- Not required if using only local users

### 6. argocd-applicationset-controller

- Manages `ApplicationSet` resources (covered in Module 06)
- Generates multiple Applications from templates and generators
- Key for multi-cluster and multi-tenant deployments

### 7. argocd-notifications-controller

- Sends notifications when application state changes
- Supports: Slack, Email, Teams, PagerDuty, Webhook, GitHub PR comments
- Covered in Module 08

---

## 📖 Lecture 2.3 — The Reconciliation Loop

Understanding exactly what happens inside the application-controller:

```
┌────────────────────────────────────────────────────────────┐
│         APPLICATION CONTROLLER RECONCILIATION LOOP         │
│                                                            │
│  Every 3 minutes (configurable) OR on webhook trigger:    │
│                                                            │
│  1. Read Application CRD from Kubernetes API              │
│     (gets: repoURL, path, targetRevision, destination)    │
│                                                            │
│  2. Ask repo-server: "What is the desired state?"         │
│     repo-server → clones/caches Git → renders manifest    │
│     → returns manifest list                               │
│                                                            │
│  3. Get live state from Kubernetes API                    │
│     kubectl get ... (for all resources in the app)        │
│                                                            │
│  4. Compute diff: desired vs live                         │
│     → Compare resource by resource                        │
│     → Track: Added / Modified / Deleted / Unchanged       │
│                                                            │
│  5. Update Application.status:                            │
│     Synced: true/false                                    │
│     Health: Healthy/Progressing/Degraded/Missing/Unknown  │
│                                                            │
│  6. If sync triggered (manual or auto): Apply diff        │
│     → kubectl apply for added/modified resources          │
│     → kubectl delete for removed resources (if prune on) │
│                                                            │
│  7. Wait for resources to reach desired state             │
│     → Checks pod readiness, rollout status                │
│                                                            │
│  8. Update status to Synced + Health                     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## 📖 Lecture 2.4 — ArgoCD Custom Resources (CRDs)

ArgoCD extends Kubernetes with its own Custom Resource Definitions:

```bash
# List all ArgoCD CRDs
kubectl get crds | grep argoproj.io

# You'll see:
# applications.argoproj.io          ← Application (the main one)
# applicationsets.argoproj.io       ← ApplicationSet (Module 06)
# appprojects.argoproj.io           ← AppProject (Module 07)
```

### The Application CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application                        # ArgoCD's core resource
metadata:
  name: my-app
  namespace: argocd                      # Must be in argocd namespace
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # Delete K8s resources on uninstall
spec:
  project: default                       # ArgoCD project (access control)

  # ---- SOURCE: where to get manifests ----
  source:
    repoURL: https://github.com/org/repo
    targetRevision: main                 # Branch, tag, or commit SHA
    path: apps/my-app                   # Directory in the repo

  # ---- DESTINATION: where to deploy ----
  destination:
    server: https://kubernetes.default.svc   # Target cluster
    namespace: my-app                        # Target namespace

  # ---- SYNC POLICY ----
  syncPolicy:
    automated:                           # Enable auto-sync
      prune: true                        # Delete resources removed from Git
      selfHeal: true                     # Revert manual cluster changes

    syncOptions:
      - CreateNamespace=true             # Auto-create destination namespace
      - PrunePropagationPolicy=foreground # How to delete child resources
      - PruneLast=true                   # Prune after all other resources sync

    retry:                               # Retry failed syncs
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

status:                                  # Populated by ArgoCD (don't edit)
  sync:
    status: Synced                       # Synced | OutOfSync
    revision: abc123                     # Current Git commit
  health:
    status: Healthy                      # Healthy | Progressing | Degraded | Suspended | Missing | Unknown
  conditions:                            # Errors/warnings
  history:                               # Sync history
```

---

## 📖 Lecture 2.5 — Sync Status vs Health Status

These are two separate concepts that students often confuse:

### Sync Status

```
Synced       → Cluster matches Git exactly
OutOfSync    → Cluster differs from Git (drift or new change in Git)
Unknown      → Cannot determine (repo unreachable, etc.)
```

### Health Status

```
Healthy      → All resources are ready and functioning
Progressing  → Resources are being created/updated (transitioning)
Degraded     → Resources failed (CrashLoopBackOff, ImagePullError, etc.)
Suspended    → Rollout is paused
Missing      → Resource doesn't exist in cluster at all
Unknown      → Health cannot be determined
```

### The 4 States of an Application

| Sync | Health | Meaning |
|---|---|---|
| Synced | Healthy | ✅ Everything is perfect |
| OutOfSync | Healthy | ⚠️ Git changed, but cluster is old version (still running OK) |
| Synced | Degraded | ❌ Cluster matches Git, but the app is broken |
| OutOfSync | Degraded | 🔥 Both Git changed AND app is broken |

---

## 📖 Lecture 2.6 — Source Types

ArgoCD can deploy from multiple source types:

### 1. Plain Kubernetes YAML/JSON

```yaml
source:
  repoURL: https://github.com/org/repo
  path: kubernetes/manifests
  # No special config needed
```

### 2. Helm Chart (in repo)

```yaml
source:
  repoURL: https://github.com/org/repo
  path: charts/my-app
  helm:
    valueFiles:
      - values.yaml
      - values-production.yaml
    values: |
      replicaCount: 3
      image:
        tag: v1.2.3
```

### 3. Helm Chart (from repository)

```yaml
source:
  repoURL: https://charts.bitnami.com/bitnami
  chart: postgresql
  targetRevision: "12.5.6"
  helm:
    values: |
      auth:
        password: changeme
```

### 4. Kustomize

```yaml
source:
  repoURL: https://github.com/org/repo
  path: overlays/production
  kustomize:
    namePrefix: prod-
    commonLabels:
      env: production
    images:
      - myapp=my-registry/myapp:v1.2.3
```

### 5. Multiple Sources (ArgoCD 2.6+)

```yaml
sources:
  - repoURL: https://github.com/org/helm-charts
    targetRevision: HEAD
    path: charts/my-app
    helm:
      valueFiles:
        - $values/environments/production/values.yaml  # ← From another repo!
  - repoURL: https://github.com/org/gitops-config      # ← Second source!
    targetRevision: HEAD
    ref: values                                         # ← Named reference
```

---

## 📖 Lecture 2.7 — Key ArgoCD CLI Commands

```bash
# ---- APPLICATION MANAGEMENT ----
argocd app list                              # List all apps
argocd app get <name>                        # Full app status
argocd app get <name> --refresh              # Force status refresh from cluster
argocd app create ...                        # Create application
argocd app delete <name>                     # Delete app (keeps K8s resources)
argocd app delete <name> --cascade           # Delete app + all K8s resources

# ---- SYNC ----
argocd app sync <name>                       # Trigger sync
argocd app sync <name> --dry-run             # Simulate sync (show what would change)
argocd app sync <name> --force               # Force sync (replace resources)
argocd app sync <name> --prune              # Allow pruning resources not in Git

# ---- DIFF ----
argocd app diff <name>                       # Show diff between Git and cluster

# ---- HISTORY & ROLLBACK ----
argocd app history <name>                    # Show sync history
argocd app rollback <name> <revision>        # Rollback to previous sync

# ---- LOGS / RESOURCES ----
argocd app resources <name>                  # List managed resources
argocd app manifests <name>                  # Show live cluster manifests
argocd app logs <name>                       # Stream application logs

# ---- CLUSTER / REPO ----
argocd cluster list                          # List connected clusters
argocd cluster add <context>                 # Add cluster from kubeconfig
argocd repo list                             # List connected repos
argocd repo add <url>                        # Add repo credentials

# ---- ACCOUNT ----
argocd account list                          # List users
argocd account update-password              # Change password
argocd account get-user-info                 # Current user

# ---- SERVER ----
argocd version                               # ArgoCD version (client + server)
argocd login <server>                        # Login
argocd logout                                # Logout
```

---

## ✅ Module 02 — Review Questions

1. Name all 7 ArgoCD components and describe what each one does.
2. Which component is the "brain" of ArgoCD — the one doing reconciliation?
3. What is the role of `argocd-repo-server`?
4. What is the difference between **Sync Status** and **Health Status**?
5. What does `OutOfSync` + `Healthy` mean in practice?
6. What CRD is the core resource in ArgoCD? What namespace must it be in?
7. What does the `resources-finalizer.argocd.argoproj.io` finalizer do?
8. Name 4 source types ArgoCD supports.
9. What does `syncPolicy.automated.selfHeal` do?
10. What does `syncPolicy.automated.prune` do? When is this important?

---

## 🧪 Lab 02 — ArgoCD Architecture Exploration

**→ See [labs/lab-02-argocd-setup.md](./labs/lab-02-argocd-setup.md)**

---

*[← Module 01](../Module-01-GitOps-Foundations/README.md) | [Module 03 →](../Module-03-Applications-Sync/README.md)*
