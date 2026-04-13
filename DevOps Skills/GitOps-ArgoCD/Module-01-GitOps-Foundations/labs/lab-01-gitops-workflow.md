# Lab 01 — GitOps Workflow Simulation

> **Lab Objective:** Set up your first GitOps repository, install ArgoCD, understand the sync cycle, and observe drift detection in action.
>
> **Estimated Time:** 45–60 minutes  
> **Difficulty:** ⭐ Beginner

---

## Prerequisites

```bash
# Cluster running
kubectl get nodes
# STATUS should be Ready

# ArgoCD installed (from README setup)
kubectl get pods -n argocd
# All pods should be Running
```

---

## Part 1 — Create Your GitOps Repository

### 1.1 — Create the Repository

1. Go to [github.com](https://github.com) → New repository
2. Name: `gitops-course-configs`
3. Visibility: **Public** (easiest for this course)
4. ✅ Initialize with README
5. Click "Create repository"

### 1.2 — Clone It Locally

```bash
git clone https://github.com/YOUR_USERNAME/gitops-course-configs.git
cd gitops-course-configs
```

### 1.3 — Create the Directory Structure

```bash
# Create the environments structure
mkdir -p apps/guestbook
mkdir -p infrastructure
mkdir -p clusters
```

### 1.4 — Create Your First Application Manifest

Create `apps/guestbook/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  namespace: guestbook
  labels:
    app: guestbook
    managed-by: argocd
spec:
  replicas: 1         # ← We will change this to test drift detection
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
        - name: guestbook
          image: gcr.io/heptio-images/ks-guestbook-demo:0.2
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

Create `apps/guestbook/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: guestbook
  namespace: guestbook
  labels:
    app: guestbook
spec:
  type: ClusterIP
  selector:
    app: guestbook
  ports:
    - port: 80
      targetPort: 80
      name: http
```

Create `apps/guestbook/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: guestbook
  labels:
    managed-by: argocd
```

### 1.5 — Commit and Push

```bash
git add .
git commit -m "feat: add guestbook application manifests"
git push origin main
```

---

## Part 2 — Install and Access ArgoCD

### 2.1 — Start Port-Forward (Keep This Open)

```bash
# Open a dedicated terminal for this
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 2.2 — Get Admin Password

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode && echo
```

### 2.3 — Login via CLI

```bash
argocd login localhost:8080 \
  --username admin \
  --password <PASTE_PASSWORD_HERE> \
  --insecure
```

### 2.4 — Open the UI

Visit: **https://localhost:8080**  
Login: `admin` / `<your-password>`

Explore the empty dashboard. It shows no applications yet.

---

## Part 3 — Create Your First ArgoCD Application

### Method A: ArgoCD CLI

```bash
argocd app create guestbook \
  --repo https://github.com/YOUR_USERNAME/gitops-course-configs.git \
  --path apps/guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace guestbook \
  --sync-policy manual

# Verify it was created
argocd app list
```

**Expected output:**
```
NAME        CLUSTER                         NAMESPACE  PROJECT  STATUS     HEALTH   SYNCPOLICY  CONDITIONS
guestbook   https://kubernetes.default.svc  guestbook  default  OutOfSync  Missing  Manual      <none>
```

- **OutOfSync** → Git has manifests, cluster does not (nothing deployed yet)
- **Missing** → The namespace doesn't even exist yet

### Method B: UI (Alternative)

1. Click "+ NEW APP" in the ArgoCD UI
2. Fill in:
   - Application Name: `guestbook`
   - Project: `default`
   - Sync Policy: `Manual`
   - Repository URL: `https://github.com/YOUR_USERNAME/gitops-course-configs`
   - Revision: `main`
   - Path: `apps/guestbook`
   - Cluster URL: `https://kubernetes.default.svc`
   - Namespace: `guestbook`
3. Click "CREATE"

### Method C: Declarative YAML (GitOps Best Practice)

Create `clusters/guestbook-app.yaml`:

```yaml
# This Application definition ITSELF is stored in Git!
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  labels:
    app.kubernetes.io/name: guestbook
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # Delete resources on uninstall
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-course-configs.git
    targetRevision: main
    path: apps/guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
      - CreateNamespace=true   # Auto-create namespace
```

```bash
# Apply the Application definition
kubectl apply -f clusters/guestbook-app.yaml -n argocd
```

---

## Part 4 — Sync the Application

### 4.1 — Manual Sync via CLI

```bash
# Trigger sync
argocd app sync guestbook

# Watch status
argocd app get guestbook
```

### 4.2 — Watch in the UI

1. Go to https://localhost:8080
2. Click on the `guestbook` application card
3. Watch the resources appear: Namespace → Deployment → Service
4. See the health turn green ✅

### 4.3 — Verify in kubectl

```bash
kubectl get all -n guestbook
# Should show Deployment, ReplicaSet, Pod, Service

kubectl get pods -n guestbook
# Pod should be Running

# Access the app
kubectl port-forward svc/guestbook -n guestbook 8888:80
# Open http://localhost:8888 → See the guestbook!
```

---

## Part 5 — The Sync Cycle in Detail

### 5.1 — Understand ArgoCD Status

```bash
# Full app status
argocd app get guestbook
```

Key status fields:
```
Name:               guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          guestbook
Repo:               https://github.com/...
Target:             main
Path:               apps/guestbook
Sync Policy:        Manual
Sync Status:        Synced to main (abc123)     ← Git commit hash
Health Status:      Healthy                     ← All resources healthy
```

### 5.2 — Check Sync Diff Before Syncing

This is like `git diff` but for your cluster:
```bash
# See what WOULD change if you synced
argocd app diff guestbook
```

---

## Part 6 — Observe Drift Detection ⭐ Core Concept

### 6.1 — Create Drift (Manual Kubernetes Change)

```bash
# Manually change replicas to 3 WITHOUT going through Git
kubectl scale deployment guestbook -n guestbook --replicas=3

# Verify the change took effect
kubectl get pods -n guestbook
# You should see 3 pods
```

### 6.2 — Watch ArgoCD Detect the Drift

```bash
# Watch the app status (refresh ArgoCD's cache)
argocd app get guestbook --refresh

# Status should now show: OutOfSync
# The diff will show:
argocd app diff guestbook
```

**Expected diff:**
```diff
===== apps/Deployment guestbook/guestbook ======
95c95
<   replicas: 1    ← Git says 1
---
>   replicas: 3    ← Cluster has 3
```

### 6.3 — Sync Back (Git Wins)

```bash
# Sync back to Git (replicas goes back to 1)
argocd app sync guestbook

# Verify cluster corrected
kubectl get pods -n guestbook
# Should be back to 1 pod
```

> 🎓 **This is the essence of GitOps:** Git always wins. Manual changes are drift. ArgoCD corrects them.

---

## Part 7 — Enable Auto-Sync (Self-Healing)

### 7.1 — Enable Auto-Sync via CLI

```bash
argocd app set guestbook \
  --sync-policy automated \
  --self-heal \
  --auto-prune
```

**Options explained:**
- `--sync-policy automated` → Sync automatically when Git changes
- `--self-heal` → Revert manual changes in cluster back to Git state
- `--auto-prune` → Delete resources removed from Git

### 7.2 — Test Self-Healing

```bash
# Manually scale to 5 replicas again
kubectl scale deployment guestbook -n guestbook --replicas=5

# Wait 10-30 seconds (ArgoCD checks fast)
sleep 30

# Check — should auto-correct back to 1
kubectl get pods -n guestbook
```

ArgoCD automatically reverted the manual change! This is self-healing.

### 7.3 — Test Git-Driven Scaling

```bash
# Now make a legitimate change via Git
# Edit apps/guestbook/deployment.yaml: replicas: 1 → replicas: 2

git add apps/guestbook/deployment.yaml
git commit -m "scale: increase guestbook replicas to 2"
git push origin main

# Wait for ArgoCD to detect (up to 3 minutes for polling, instant with webhook)
argocd app get guestbook --refresh

# Verify
kubectl get pods -n guestbook
# Should now have 2 pods — automatically deployed!
```

---

## Part 8 — Explore the ArgoCD UI

Navigate the UI and find:
1. **Application Graph** — Click into the guestbook app, see the resource tree
2. **Resource Details** — Click on the Deployment → see events and live manifest
3. **Sync History** — Click "History and Rollback" — see all previous syncs
4. **App Diff** — Before syncing, see exactly what will change

---

## Part 9 — Cleanup

```bash
# Delete the ArgoCD application
argocd app delete guestbook --cascade

# Verify resources were cleaned up
kubectl get all -n guestbook
kubectl get ns guestbook
```

The `--cascade` flag deletes all K8s resources managed by the app.

---

## 🏆 Lab Challenges

1. **Challenge 1:** Add a `ConfigMap` to your guestbook app manifests. Commit and push. Verify ArgoCD auto-applies it without any manual intervention.

2. **Challenge 2:** Delete the guestbook Deployment manually with `kubectl delete deployment guestbook -n guestbook`. With self-heal enabled, how long does it take for ArgoCD to recreate it?

3. **Challenge 3:** Create a second ArgoCD application pointing to a **different path** in the same Git repo. Push manifests for a second app (e.g., nginx) and observe both apps in ArgoCD.

---

*[← Module 01 Lecture](../README.md) | [Module 02 →](../../Module-02-ArgoCD-Architecture/README.md)*
