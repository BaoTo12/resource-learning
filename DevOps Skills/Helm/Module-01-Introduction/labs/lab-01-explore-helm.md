# Lab 01 — Explore Helm: Your First Commands

> **Lab Objective:** Get comfortable with the Helm CLI. Explore repositories, install your first chart, inspect a release, and clean up.
>
> **Estimated Time:** 30–45 minutes  
> **Difficulty:** ⭐ Beginner

---

## Prerequisites

Before starting, verify your environment:

```bash
# 1. Helm is installed
helm version

# 2. Kubernetes cluster is running
kubectl get nodes

# 3. Cluster is ready (STATUS should be Ready)
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   5m    v1.28.x
```

If `kubectl get nodes` shows no Ready nodes, start Minikube:
```bash
minikube start
```

---

## Exercise 1 — Explore Chart Repositories

### 1.1 — Add Popular Repositories

```bash
# Add Bitnami (most popular chart collection)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Add Prometheus Community charts  
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Add Ingress NGINX
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update all repo indexes
helm repo update

# List all repos you've added
helm repo list
```

**Expected Output:**
```
NAME                    URL
bitnami                 https://charts.bitnami.com/bitnami
prometheus-community    https://prometheus-community.github.io/helm-charts
ingress-nginx           https://kubernetes.github.io/ingress-nginx
```

### 1.2 — Search for Charts

```bash
# Search for nginx charts in repositories
helm search repo nginx

# Search for postgresql
helm search repo postgresql

# Show ALL available versions of a chart
helm search repo bitnami/nginx --versions | head -20

# Search Artifact Hub (the public Helm registry)
helm search hub wordpress
```

### 1.3 — Inspect a Chart Before Installing

Always inspect a chart before you install it. Never install unknown charts blindly.

```bash
# See the chart's metadata
helm show chart bitnami/nginx

# See ALL default values (this is crucial for understanding configuration)
helm show values bitnami/nginx

# Save values to a file so you can study them
helm show values bitnami/nginx > nginx-default-values.yaml

# Open and read the file
# Look at: image.registry, image.repository, service.type, replicaCount
cat nginx-default-values.yaml | head -60
```

---

## Exercise 2 — Install Your First Chart

### 2.1 — Create a Namespace

```bash
# Create a dedicated namespace for our lab
kubectl create namespace helm-lab

# Verify it was created
kubectl get namespaces
```

### 2.2 — Install Nginx (with Dry Run First!)

**Always do a dry run first:**
```bash
# Dry run — see what would be created WITHOUT actually creating anything
helm install my-nginx bitnami/nginx \
  --namespace helm-lab \
  --dry-run \
  --debug
```

Study the output. You'll see:
- The rendered YAML that would be sent to Kubernetes
- Template rendering errors (if any)

**Now actually install it:**
```bash
helm install my-nginx bitnami/nginx \
  --namespace helm-lab \
  --set service.type=NodePort
```

**Expected Output:**
```
NAME: my-nginx
LAST DEPLOYED: <timestamp>
NAMESPACE: helm-lab
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **
...
```

### 2.3 — Verify the Installation

```bash
# List Helm releases
helm list -n helm-lab

# Check Kubernetes resources created
kubectl get all -n helm-lab

# Check specifically
kubectl get deployment -n helm-lab
kubectl get service -n helm-lab
kubectl get pods -n helm-lab
```

**Wait for pods to be Running:**
```bash
# Watch pods come up (Ctrl+C to stop watching)
kubectl get pods -n helm-lab -w
```

### 2.4 — Access the Application

```bash
# If using Minikube with NodePort:
minikube service my-nginx -n helm-lab

# Or get the URL manually
kubectl get svc my-nginx -n helm-lab
# Note the NodePort number, then access:
# http://<minikube-ip>:<nodeport>

# Get minikube IP
minikube ip
```

---

## Exercise 3 — Inspect the Release

### 3.1 — Get Release Details

```bash
# Full status of the release
helm status my-nginx -n helm-lab

# Get the values that were used (user-supplied only)
helm get values my-nginx -n helm-lab

# Get ALL values (including chart defaults)
helm get values my-nginx -n helm-lab --all

# Get the actual manifests (rendered YAML) that are deployed
helm get manifest my-nginx -n helm-lab
```

### 3.2 — Look at How Helm Stores State

```bash
# Helm stores release data as Kubernetes Secrets!
# Let's see them:
kubectl get secrets -n helm-lab

# You should see something like:
# sh.helm.release.v1.my-nginx.v1   helm.sh/release.v1   1      5m

# Look at the release history
helm history my-nginx -n helm-lab
```

---

## Exercise 4 — Upgrade a Release

### 4.1 — Upgrade with New Values

Let's change the number of replicas:

```bash
# Check current replicas
kubectl get deployment -n helm-lab

# Upgrade: change replica count
helm upgrade my-nginx bitnami/nginx \
  --namespace helm-lab \
  --set replicaCount=2 \
  --set service.type=NodePort

# Check history — now shows 2 revisions
helm history my-nginx -n helm-lab
```

### 4.2 — Use a Values File for Upgrade

Create a file `my-nginx-values.yaml`:

```yaml
# my-nginx-values.yaml
replicaCount: 3

service:
  type: NodePort

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"
```

```bash
# Apply the values file
helm upgrade my-nginx bitnami/nginx \
  --namespace helm-lab \
  --values my-nginx-values.yaml

# Verify
helm get values my-nginx -n helm-lab
kubectl get pods -n helm-lab
```

---

## Exercise 5 — Rollback

### 5.1 — Rollback to Previous Version

```bash
# Show history
helm history my-nginx -n helm-lab

# Rollback to revision 1
helm rollback my-nginx 1 -n helm-lab

# Verify rollback
helm history my-nginx -n helm-lab
kubectl get pods -n helm-lab
```

---

## Exercise 6 — Install Multiple Releases from Same Chart

This demonstrates the power of Helm — same chart, different configurations:

```bash
# Install a "development" nginx
helm install nginx-dev bitnami/nginx \
  --namespace helm-lab \
  --set replicaCount=1 \
  --set service.type=NodePort

# Install a "production" nginx
helm install nginx-prod bitnami/nginx \
  --namespace helm-lab \
  --set replicaCount=3 \
  --set service.type=NodePort

# List all releases — you see both!
helm list -n helm-lab

# Each has its own pods:
kubectl get pods -n helm-lab
```

---

## Exercise 7 — Cleanup

```bash
# Uninstall specific releases
helm uninstall my-nginx -n helm-lab
helm uninstall nginx-dev -n helm-lab
helm uninstall nginx-prod -n helm-lab

# Verify releases are gone
helm list -n helm-lab

# Verify Kubernetes resources are gone
kubectl get all -n helm-lab

# Delete the namespace (optional)
kubectl delete namespace helm-lab
```

---

## 🏆 Lab Challenges (Optional — Push Yourself!)

1. **Challenge 1:** Install `bitnami/wordpress` with a custom admin username of `helm-student`. Find the correct `--set` flag by examining `helm show values bitnami/wordpress`.

2. **Challenge 2:** Install nginx in 3 different namespaces (`dev`, `staging`, `prod`) each with different replica counts (1, 2, 3). Use `helm list -A` to list all releases across all namespaces.

3. **Challenge 3:** Install a chart, corrupt the release (simulate a bad upgrade with an invalid value), and practice rollback using `helm rollback`.

---

## 📝 Lab Reflection

After completing this lab, you should be able to answer:

1. What command shows you all values a chart accepts before you install it?
2. What is the difference between a **Release Name** and a **Chart Name**?
3. Where is Helm release state stored in Kubernetes?
4. Which command would you use in a CI/CD pipeline and why? `helm install` or `helm upgrade --install`?

---

*[← Module 01 Lecture](../README.md) | [Module 02 →](../../Module-02-First-Chart/README.md)*
