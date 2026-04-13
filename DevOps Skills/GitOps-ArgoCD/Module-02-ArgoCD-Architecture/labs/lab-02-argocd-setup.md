# Lab 02 — ArgoCD Architecture Exploration

> **Lab Objective:** Explore every ArgoCD component hands-on. Examine logs, CRDs, configuration files, and understand what's happening inside the ArgoCD namespace.
>
> **Estimated Time:** 45–60 minutes  
> **Difficulty:** ⭐ Beginner

---

## Part 1 — Explore ArgoCD Components in Kubernetes

### 1.1 — View All ArgoCD Resources

```bash
# Everything in the argocd namespace
kubectl get all -n argocd

# Should show:
# pod/argocd-application-controller-0
# pod/argocd-applicationset-controller-xxx
# pod/argocd-dex-server-xxx
# pod/argocd-notifications-controller-xxx
# pod/argocd-redis-xxx
# pod/argocd-repo-server-xxx
# pod/argocd-server-xxx

# Deployments
kubectl get deployments -n argocd

# StatefulSets (application-controller runs as StatefulSet)
kubectl get statefulsets -n argocd

# Services
kubectl get svc -n argocd

# ConfigMaps (ArgoCD configuration)
kubectl get configmaps -n argocd

# Secrets (ArgoCD stores repo creds, cluster creds)
kubectl get secrets -n argocd
```

### 1.2 — Inspect Each Component

```bash
# Application Controller (the brain)
kubectl describe pod -n argocd \
  $(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-application-controller -o jsonpath='{.items[0].metadata.name}')

# Repo Server
kubectl describe pod -n argocd \
  $(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-repo-server -o jsonpath='{.items[0].metadata.name}')

# ArgoCD Server
kubectl describe pod -n argocd \
  $(kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o jsonpath='{.items[0].metadata.name}')
```

---

## Part 2 — Explore ArgoCD ConfigMaps

ArgoCD stores most of its configuration in ConfigMaps:

```bash
# Main ArgoCD configuration
kubectl get configmap argocd-cm -n argocd -o yaml
```

This ConfigMap controls:
- Git repositories
- OIDC/SSO configuration
- Custom health checks
- Resource exclusions

```bash
# RBAC configuration
kubectl get configmap argocd-rbac-cm -n argocd -o yaml
```

This controls access control:
- Role definitions
- User-to-role bindings

```bash
# SSH known hosts (for Git over SSH)
kubectl get configmap argocd-ssh-known-hosts-cm -n argocd -o yaml

# TLS certificates
kubectl get configmap argocd-tls-certs-cm -n argocd -o yaml
```

---

## Part 3 — Explore ArgoCD CRDs

```bash
# List all ArgoCD custom resource definitions
kubectl get crds | grep argoproj

# Get the Application CRD spec
kubectl get crd applications.argoproj.io -o yaml | head -100

# List all Application instances
kubectl get applications -n argocd

# Get details of an application (if you created one in Lab 01)
kubectl get application guestbook -n argocd -o yaml
```

**Study the output** — this is the raw YAML that ArgoCD stores:
- `spec.source` — where to get manifests
- `spec.destination` — where to deploy
- `spec.syncPolicy` — how to sync
- `status.sync` — current sync state
- `status.health` — current health
- `status.history` — sync history

---

## Part 4 — Watch Component Logs Live

Open 3 separate terminals for this exercise:

### Terminal 1: Application Controller Logs

```bash
kubectl logs -n argocd \
  -l app.kubernetes.io/name=argocd-application-controller \
  --follow \
  --tail=50
```

Watch for reconciliation messages:
```
time="..." level=info msg="Reconciliation completed" application=guestbook ...
time="..." level=info msg="Updating app status" application=guestbook sync=OutOfSync
```

### Terminal 2: Repo Server Logs

```bash
kubectl logs -n argocd \
  -l app.kubernetes.io/name=argocd-repo-server \
  --follow \
  --tail=50
```

Watch for Git clone/fetch messages:
```
time="..." level=info msg="Fetching git repository" repo=https://github.com/...
time="..." level=info msg="Generating manifests from git" revision=main
```

### Terminal 3: ArgoCD Server Logs

```bash
kubectl logs -n argocd \
  -l app.kubernetes.io/name=argocd-server \
  --follow \
  --tail=50
```

### Trigger Activity

```bash
# In a 4th terminal, force ArgoCD to reconcile:
argocd app get guestbook --refresh

# Watch Terminals 1 and 2 for activity!
```

---

## Part 5 — Explore ArgoCD Secrets

ArgoCD stores credentials as Kubernetes Secrets:

```bash
# List all secrets
kubectl get secrets -n argocd

# The initial admin password secret
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml

# Cluster connection secrets (one per registered cluster)
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=cluster

# Repository secrets (one per added repo with credentials)
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

---

## Part 6 — Add a Private Repository

If you want to use a private GitHub repository with ArgoCD:

### Method A: HTTPS with Token

```bash
# Generate a GitHub Personal Access Token:
# GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)
# Permissions: repo (full control)

# Add repo to ArgoCD
argocd repo add https://github.com/YOUR_USERNAME/PRIVATE_REPO.git \
  --username YOUR_USERNAME \
  --password YOUR_GITHUB_TOKEN

# Verify
argocd repo list
```

### Method B: SSH Key

```bash
# Generate an SSH key pair
ssh-keygen -t ed25519 -C "argocd@gitops-course" -f argocd-key -N ""

# Add the PUBLIC key to GitHub:
# GitHub repo → Settings → Deploy Keys → Add Deploy Key
# Paste contents of: argocd-key.pub

# Add the PRIVATE key to ArgoCD
argocd repo add git@github.com:YOUR_USERNAME/PRIVATE_REPO.git \
  --ssh-private-key-path argocd-key

# Verify
argocd repo list
```

### Method C: Declarative Repository Secret

```yaml
# Create a Kubernetes Secret that ArgoCD uses
apiVersion: v1
kind: Secret
metadata:
  name: my-private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository   # This label is required!
type: Opaque
stringData:
  type: git
  url: https://github.com/YOUR_USERNAME/PRIVATE_REPO.git
  username: YOUR_USERNAME
  password: YOUR_GITHUB_TOKEN  # Use a real secret manager in production!
```

```bash
kubectl apply -f repo-secret.yaml
argocd repo list  # Should show the new repo
```

---

## Part 7 — Add a Second Cluster

In production you'll have multiple clusters. Let's simulate adding one.

### 7.1 — Using kind to create a second cluster

```bash
# Create a second cluster
kind create cluster --name cluster-2

# The new cluster's context is: kind-cluster-2
kubectl config get-contexts

# Switch back to your ArgoCD cluster
kubectl config use-context kind-argocd-course

# Login to ArgoCD (if session expired)
argocd login localhost:8080 --username admin --password <password> --insecure

# Add the second cluster to ArgoCD
argocd cluster add kind-cluster-2 \
  --name cluster-2 \
  --kubeconfig ~/.kube/config

# Verify
argocd cluster list
# Should now show 2 clusters:
# SERVER                                  NAME             STATUS   MESSAGE
# https://kubernetes.default.svc          in-cluster       Successful
# https://...                             cluster-2        Successful
```

### 7.2 — View Cluster in the UI

1. Go to: Settings → Clusters
2. You'll see both clusters listed
3. Click on a cluster to see its namespaces and resources

```bash
# Clean up second cluster when done
kind delete cluster --name cluster-2
```

---

## Part 8 — ArgoCD Server Configuration

Study the main configuration:

```bash
kubectl edit configmap argocd-cm -n argocd
```

Key settings you'll configure in later modules:

```yaml
# argocd-cm ConfigMap data section
data:
  # Enable insecure mode (skip TLS for UI — dev only)
  server.insecure: "false"

  # Default sync timeout
  timeout.reconciliation: 180s

  # Repositories
  repositories: |
    - url: https://github.com/org/repo

  # Resource tracking method
  application.resourceTrackingMethod: label

  # Disable admin account (use SSO only)
  admin.enabled: "true"

  # Custom health checks
  resource.customizations.health.apps_Deployment: |
    hs = {}
    if obj.status ~= nil then
      ...
    end
```

---

## Part 9 — ArgoCD Metrics

ArgoCD exposes Prometheus metrics:

```bash
# Port-forward to metrics endpoints
kubectl port-forward svc/argocd-metrics -n argocd 8082:8082 &
kubectl port-forward svc/argocd-server-metrics -n argocd 8083:8083 &
kubectl port-forward svc/argocd-applicationset-controller-metrics -n argocd 8084:8080 &

# View metrics
curl http://localhost:8082/metrics | head -50    # Application metrics
curl http://localhost:8083/metrics | head -50    # Server metrics
```

**Key metrics to watch:**
```
argocd_app_info                  # Basic app info (labels: name, namespace, health, sync)
argocd_app_sync_total            # Total sync count
argocd_app_sync_duration_seconds # Sync duration
argocd_cluster_api_request_total # Kubernetes API calls
argocd_git_request_total         # Git operations
argocd_git_request_duration_seconds # Git fetch duration
```

---

## Part 10 — Upgrade ArgoCD

```bash
# Check current version
argocd version

# To upgrade ArgoCD (re-apply the install manifest with newer version)
# Find latest at: https://github.com/argoproj/argo-cd/releases

# Example: upgrade to specific version
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/v2.10.0/manifests/install.yaml

# Watch components restart
kubectl get pods -n argocd -w
```

---

## 🏆 Challenges

1. **Challenge 1:** Configure ArgoCD to use a custom resource health check. Add a custom health check for a `CronJob` resource in `argocd-cm` that marks it as `Healthy` when its last schedule was successful.

2. **Challenge 2:** Find and decode the Kubernetes secret that ArgoCD uses to store the admin password. What hashing algorithm is used?

3. **Challenge 3:** Using `kubectl`, watch for changes to the `Application` CRD in real-time while triggering a sync. What fields update in real-time?
```bash
kubectl get application guestbook -n argocd -w
```

---

*[← Module 02 Lecture](../README.md) | [Module 03 →](../../Module-03-Applications-Sync/README.md)*
