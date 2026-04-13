# Lab 04 — Helm Hooks & Tests

> **Lab Objective:** Implement pre-install, post-install hooks and comprehensive Helm tests for a chart.
>
> **Estimated Time:** 60–90 minutes  
> **Difficulty:** ⭐⭐ Intermediate

---

## Part 1 — Setup

```bash
mkdir -p C:\Users\Admin\Desktop\DevOps-Skills\labs\module-04
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-04
helm create hookdemo
```

---

## Part 2 — Create a Pre-Install Hook

Create `hookdemo/templates/hooks/pre-install-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "hookdemo.fullname" . }}-pre-install"
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "hookdemo.labels" . | nindent 4 }}
  annotations:
    # HOOK ANNOTATION — this runs before any resources are installed
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 2
  activeDeadlineSeconds: 120
  template:
    metadata:
      name: "{{ include "hookdemo.fullname" . }}-pre-install"
    spec:
      restartPolicy: Never
      containers:
        - name: pre-install
          image: busybox:1.35
          command:
            - /bin/sh
            - -c
            - |
              echo "============================================"
              echo "  PRE-INSTALL HOOK RUNNING"
              echo "  Release: {{ .Release.Name }}"
              echo "  Namespace: {{ .Release.Namespace }}"
              echo "  Revision: {{ .Release.Revision }}"
              echo "============================================"
              echo ""
              echo "Simulating database initialization..."
              sleep 5
              echo "✅ Database initialization complete!"
              echo ""
              echo "Simulating schema validation..."
              sleep 2
              echo "✅ Schema validation complete!"
              echo "============================================"
```

---

## Part 3 — Create a Post-Install Hook

Create `hookdemo/templates/hooks/post-install-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "hookdemo.fullname" . }}-post-install"
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "hookdemo.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 2
  activeDeadlineSeconds: 60
  template:
    metadata:
      name: "{{ include "hookdemo.fullname" . }}-post-install"
    spec:
      restartPolicy: Never
      containers:
        - name: post-install
          image: busybox:1.35
          command:
            - /bin/sh
            - -c
            - |
              echo "============================================"
              echo "  POST-INSTALL HOOK RUNNING"
              echo "  Release: {{ .Release.Name }}"
              echo "============================================"
              echo ""
              echo "Simulating cache warmup..."
              sleep 3
              echo "✅ Cache warmed up!"
              echo ""
              echo "Simulating notification..."
              echo "🔔 Deployment notification: {{ .Release.Name }} deployed successfully!"
              echo "============================================"
```

---

## Part 4 — Create a Pre-Delete Hook

Create `hookdemo/templates/hooks/pre-delete-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "hookdemo.fullname" . }}-pre-delete"
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "hookdemo.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-delete
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  backoffLimit: 1
  activeDeadlineSeconds: 60
  template:
    metadata:
      name: "{{ include "hookdemo.fullname" . }}-pre-delete"
    spec:
      restartPolicy: Never
      containers:
        - name: pre-delete
          image: busybox:1.35
          command:
            - /bin/sh
            - -c
            - |
              echo "============================================"
              echo "  PRE-DELETE HOOK RUNNING"
              echo "  Cleaning up before release deletion..."
              echo "============================================"
              sleep 2
              echo "✅ Cleanup complete! Proceeding with delete."
```

---

## Part 5 — Create Helm Tests

Replace `hookdemo/templates/tests/test-connection.yaml` with:

```yaml
---
# Test 1: Basic HTTP connectivity test
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "hookdemo.fullname" . }}-test-http"
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "hookdemo.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: test-http
      image: busybox:1.35
      command:
        - /bin/sh
        - -c
        - |
          echo "Test 1: HTTP connectivity check"
          RESULT=$(wget -qO- http://{{ include "hookdemo.fullname" . }}:{{ .Values.service.port }} 2>&1)
          if echo "$RESULT" | grep -i "nginx\|html\|welcome" > /dev/null; then
            echo "✅ Test 1 PASSED — Application is responding!"
          else
            echo "❌ Test 1 FAILED — No response from application"
            echo "Response: $RESULT"
            exit 1
          fi
---
# Test 2: DNS resolution test
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "hookdemo.fullname" . }}-test-dns"
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "hookdemo.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: test-dns
      image: busybox:1.35
      command:
        - /bin/sh
        - -c
        - |
          echo "Test 2: DNS resolution check"
          nslookup {{ include "hookdemo.fullname" . }}.{{ .Release.Namespace }}.svc.cluster.local
          if [ $? -eq 0 ]; then
            echo "✅ Test 2 PASSED — DNS resolving correctly!"
          else
            echo "❌ Test 2 FAILED — DNS resolution failed"
            exit 1
          fi
---
# Test 3: Service endpoint test
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "hookdemo.fullname" . }}-test-svc"
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "hookdemo.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: test-service
      image: busybox:1.35
      command:
        - /bin/sh
        - -c
        - |
          echo "Test 3: Service port connectivity check"
          nc -zv {{ include "hookdemo.fullname" . }} {{ .Values.service.port }}
          if [ $? -eq 0 ]; then
            echo "✅ Test 3 PASSED — Service port is open!"
          else
            echo "❌ Test 3 FAILED — Cannot connect to service port"
            exit 1
          fi
```

---

## Part 6 — Install and Watch Hooks Execute

```bash
kubectl create namespace lab04

# Install — watch hooks execute in order
helm install hook-demo ./hookdemo \
  -n lab04 \
  --set service.type=NodePort

# In another terminal, watch all jobs and pods:
kubectl get pods -n lab04 -w
```

**Expected sequence:**
1. `hook-demo-hookdemo-pre-install` job appears first
2. After it succeeds, the main Deployment/Service/etc. are created  
3. `hook-demo-hookdemo-post-install` job appears
4. Release marked as DEPLOYED

```bash
# Verify release status
helm status hook-demo -n lab04

# Check history
helm history hook-demo -n lab04
```

---

## Part 7 — Run Helm Tests

```bash
# Wait for the app to be fully running first
kubectl get pods -n lab04 -w

# Run the Helm tests
helm test hook-demo -n lab04

# View test results
helm test hook-demo -n lab04 --logs
```

**Expected Output:**
```
NAME: hook-demo
LAST DEPLOYED: ...
NAMESPACE: lab04
STATUS: deployed
REVISION: 1
TEST SUITE:     hook-demo-hookdemo-test-http
Last Started:   ...
Last Completed: ...
Phase:          Succeeded

TEST SUITE:     hook-demo-hookdemo-test-dns
Phase:          Succeeded

TEST SUITE:     hook-demo-hookdemo-test-svc
Phase:          Succeeded
```

---

## Part 8 — Test a Failing Hook

Let's see what happens when a hook fails:

```bash
# Temporarily modify the pre-install hook to fail
# Edit hookdemo/templates/hooks/pre-install-job.yaml
# Change:  sleep 5
# To:      exit 1     ← Force failure!
```

```bash
# Try to install — it should fail!
helm install failing-demo ./hookdemo -n lab04

# Expected: Error: INSTALLATION FAILED: failed pre-install: ...

# Check the failed job
kubectl get jobs -n lab04
kubectl describe job failing-demo-hookdemo-pre-install -n lab04
kubectl logs -n lab04 \
  $(kubectl get pods -n lab04 | grep "pre-install" | awk '{print $1}')

# Release should show as failed
helm list -n lab04 -a   # -a shows all states including failed

# Clean up the failed release
helm uninstall failing-demo -n lab04
```

---

## Part 9 — Upgrade and Observe Hook Re-Execution

```bash
# Restore the hook (undo the exit 1 change)
# Upgrade the release
helm upgrade hook-demo ./hookdemo -n lab04 --set replicaCount=2

# Watch hooks run again on upgrade
kubectl get pods -n lab04 -w
helm history hook-demo -n lab04
```

---

## Part 10 — Cleanup with Pre-Delete Hook

```bash
# Watch what happens during uninstall
kubectl get pods -n lab04 -w &

# Uninstall — pre-delete hook should run first
helm uninstall hook-demo -n lab04

# You should see the pre-delete job pod appear before resources are deleted
```

---

## Part 11 — Full Cleanup

```bash
kubectl delete namespace lab04
```

---

## 🏆 Challenges

1. **Challenge 1:** Create a `post-install` hook that uses `kubectl` to verify that the deployment's pods are all Ready before marking the hook as successful. Use `bitnami/kubectl` as the image.

2. **Challenge 2:** Implement a hook weight ordering system: create 3 pre-install hooks with weights `-10`, `0`, and `10`, each printing their weight number. Verify they execute in the correct order by reading the pod logs.

3. **Challenge 3:** Create a Helm test that calls a specific API endpoint path (e.g. `/api/health`) and parses a JSON response. Use `jq` (available in `alpine/curl` or similar images).

---

*[← Module 04 Lecture](../README.md) | [Module 05 →](../../Module-05-Dependencies/README.md)*
