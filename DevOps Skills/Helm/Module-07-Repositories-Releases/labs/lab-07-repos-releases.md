# Lab 07 — Helm Repositories & Release Management

> **Lab Objective:** Package and publish a Helm chart to a local repository, simulate OCI registry workflow, and practice advanced release management (rollbacks, atomic upgrades, history inspection).
>
> **Estimated Time:** 60–90 minutes  
> **Difficulty:** ⭐⭐⭐ Advanced

---

## Part 1 — Package Your Chart

```bash
mkdir -p C:\Users\Admin\Desktop\DevOps-Skills\labs\module-07
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-07

# Copy the webapp chart from Module 02
Copy-Item -Recurse ..\module-02\webapp .

# Verify it lints cleanly
helm lint ./webapp

# Package it for version 0.1.0
helm package ./webapp
# Creates: webapp-0.1.0.tgz
```

---

## Part 2 — Create a Local Helm Repository

```bash
# Create a directory to serve as our chart repository
mkdir local-chart-repo

# Move the packaged chart into it
Move-Item webapp-0.1.0.tgz local-chart-repo/

# Generate the index.yaml for the repository
helm repo index local-chart-repo/ --url file:///C:/Users/Admin/Desktop/DevOps-Skills/labs/module-07/local-chart-repo

# Inspect the generated index
cat local-chart-repo/index.yaml
```

**Expected output:**
```yaml
apiVersion: v1
entries:
  webapp:
    - apiVersion: v2
      appVersion: 1.0.0
      created: "..."
      description: A sample web application Helm chart - ...
      digest: sha256:...
      name: webapp
      urls:
        - file:///C:/...local-chart-repo/webapp-0.1.0.tgz
      version: 0.1.0
generated: "..."
```

---

## Part 3 — Register the Local Repository

```bash
# Add your local repo to Helm
helm repo add local-charts file:///C:/Users/Admin/Desktop/DevOps-Skills/labs/module-07/local-chart-repo

# Verify it's registered
helm repo list

# Search for charts in your local repo  
helm search repo local-charts/

# Show chart details from the repo
helm show chart local-charts/webapp
helm show values local-charts/webapp
```

---

## Part 4 — Install from Local Repository

```bash
kubectl create namespace lab07

# Install from local repo (just like any other Helm repo!)
helm install my-webapp local-charts/webapp \
  -n lab07 \
  --set service.type=NodePort

# Verify installation
helm list -n lab07
kubectl get all -n lab07
```

---

## Part 5 — Version a Chart and Publish Updates

### 5.1 — Make Changes and Release v0.2.0

```bash
# Edit webapp/Chart.yaml — bump both version and appVersion
# Change:
#   version: 0.1.0 → 0.2.0
#   appVersion: "1.0.0" → "1.1.0"

# Edit webapp/values.yaml — add a new feature
# Add under resources:
#   autoscaling:
#     enabled: true
#     minReplicas: 1
#     maxReplicas: 5

# Package the new version
helm package ./webapp
# Creates: webapp-0.2.0.tgz

# Move to the repo
Move-Item webapp-0.2.0.tgz local-chart-repo/

# Regenerate the index (must include ALL chart versions)
helm repo index local-chart-repo/ \
  --url file:///C:/Users/Admin/Desktop/DevOps-Skills/labs/module-07/local-chart-repo \
  --merge local-chart-repo/index.yaml   # Merge with existing index!

# Update the local cache
helm repo update

# Verify both versions are now available
helm search repo local-charts/ --versions
```

### 5.2 — Upgrade to the New Version

```bash
# See what's currently installed
helm list -n lab07

# Upgrade to v0.2.0
helm upgrade my-webapp local-charts/webapp \
  -n lab07 \
  --version 0.2.0 \
  --set service.type=NodePort

# Check history — now shows 2 revisions
helm history my-webapp -n lab07
```

---

## Part 6 — Release Management: Full Upgrade Cycle

### 6.1 — Create a "bad" release

```bash
# Install a fresh release for testing
helm install release-test local-charts/webapp \
  -n lab07 \
  --set replicaCount=1 \
  --set service.type=NodePort

# REVISION 1 installed
helm history release-test -n lab07
```

### 6.2 — Upgrade (simulating a good release)

```bash
# Upgrade with more replicas
helm upgrade release-test local-charts/webapp \
  -n lab07 \
  --set replicaCount=2 \
  --set service.type=NodePort

# REVISION 2 installed
helm history release-test -n lab07
```

### 6.3 — Simulate a Failed Upgrade

```bash
# Attempt to upgrade with an invalid image tag (this will make pods fail)
helm upgrade release-test local-charts/webapp \
  -n lab07 \
  --set image.tag="this-tag-does-not-exist-abc123" \
  --set service.type=NodePort \
  --wait \
  --timeout 60s
# This should FAIL with timeout waiting for pods

# Check status — should show failed
helm list -n lab07 -a
helm history release-test -n lab07
```

### 6.4 — Rollback

```bash
# Rollback to revision 2 (last good version)
helm rollback release-test 2 -n lab07

# Check — rollback creates REVISION 4 (new revision, not reverting revision number)
helm history release-test -n lab07

# Verify pods are healthy
kubectl get pods -n lab07
```

---

## Part 7 — Simulate OCI Registry with Local Docker Registry

```bash
# First, ensure Docker is running
# Start a local OCI-compatible registry:
docker run -d -p 5000:5000 --name local-registry registry:2

# Verify registry is running
curl http://localhost:5000/v2/

# Package the webapp chart
helm package ./webapp
# Creates: webapp-0.2.0.tgz

# Login to local registry (no auth needed for local registry)
helm registry login localhost:5000 --insecure

# Push chart to local OCI registry
helm push webapp-0.2.0.tgz oci://localhost:5000/helm-charts

# Verify the chart was pushed
curl http://localhost:5000/v2/helm-charts/webapp/tags/list

# Pull and install from OCI registry
helm install oci-test oci://localhost:5000/helm-charts/webapp \
  -n lab07 \
  --version 0.2.0 \
  --set service.type=NodePort \
  --plain-http   # needed for insecure http (dev only)

# Verify
helm list -n lab07
```

---

## Part 8 — Atomic Upgrade Demo

```bash
# Install a fresh release
helm install atomic-test local-charts/webapp \
  -n lab07 \
  --set service.type=NodePort

# Perform a SAFE upgrade with --atomic
# (if it fails, it auto-rolls back)
helm upgrade atomic-test local-charts/webapp \
  -n lab07 \
  --set replicaCount=3 \
  --set service.type=NodePort \
  --atomic \
  --timeout 2m \
  --wait

# History shows revision was successfully applied
helm history atomic-test -n lab07

# Now try an ATOMIC upgrade that WILL FAIL
helm upgrade atomic-test local-charts/webapp \
  -n lab07 \
  --set image.tag="nonexistent-tag-xyz" \
  --set service.type=NodePort \
  --atomic \
  --timeout 60s

# Since --atomic was used, release AUTOMATICALLY rolled back!
helm history atomic-test -n lab07
# Current revision should be the pre-upgrade version

kubectl get pods -n lab07 -l app.kubernetes.io/name=webapp
# Pods should be running normally (rolled back)
```

---

## Part 9 — Inspect Helm Release Storage

```bash
# Helm stores releases as Kubernetes secrets
kubectl get secrets -n lab07 | grep helm

# Examine a specific release secret (base64 encoded data)
kubectl get secret sh.helm.release.v1.my-webapp.v1 -n lab07 -o yaml

# Cleanup old history (keep only last N revisions) — done via upgrade flag:
# --history-max=5 limits history to 5 revisions
helm upgrade my-webapp local-charts/webapp \
  -n lab07 \
  --set service.type=NodePort \
  --history-max=3

# Or set globally in Helm env
# HELM_MAX_HISTORY=10
```

---

## Part 10 — Cleanup

```bash
# Stop local Docker registry
docker stop local-registry
docker rm local-registry

# Uninstall all test releases
helm uninstall my-webapp -n lab07 2>/dev/null || true
helm uninstall release-test -n lab07 2>/dev/null || true
helm uninstall atomic-test -n lab07 2>/dev/null || true
helm uninstall oci-test -n lab07 2>/dev/null || true

# Remove local repo
helm repo remove local-charts

# Delete namespace
kubectl delete namespace lab07
```

---

## 🏆 Challenges

1. **Challenge 1:** Set up `chartmuseum` as a local Helm chart server (using Docker: `ghcr.io/helm/chartmuseum`). Push your chart using the ChartMuseum API. Register it as a Helm repo and install from it.

2. **Challenge 2:** Create a GitHub repository, set up GitHub Pages, and publish your webapp chart there. Add the GitHub Pages repo to your local Helm and install from it.

3. **Challenge 3:** Implement a full rollback script: write a shell script that checks if the current Helm release is healthy (all pods Running), and if not, automatically rolls back to the previous revision and sends an alert notification.

---

*[← Module 07 Lecture](../README.md) | [Module 08 →](../../Module-08-Production/README.md)*
