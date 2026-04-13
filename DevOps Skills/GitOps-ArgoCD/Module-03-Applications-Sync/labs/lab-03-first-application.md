# Lab 03 — Applications, Sync Waves & Resource Hooks

> **Lab Objective:** Build a multi-resource application with sync wave ordering and resource hooks. Observe the precise order of operations controlled by ArgoCD.
>
> **Estimated Time:** 60–90 minutes  
> **Difficulty:** ⭐⭐ Intermediate

---

## Part 1 — Git Repo Structure for This Lab

In your `gitops-course-configs` repository, create:

```bash
cd gitops-course-configs

mkdir -p apps/wave-demo
mkdir -p apps/hook-demo
```

---

## Part 2 — Sync Wave Demo Application

### 2.1 — Create Manifests with Sync Waves

Create `apps/wave-demo/namespace.yaml` (Wave -10 — first!):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: wave-demo
  labels:
    app: wave-demo
  annotations:
    argocd.argoproj.io/sync-wave: "-10"
```

Create `apps/wave-demo/configmap.yaml` (Wave -5):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: wave-demo
  annotations:
    argocd.argoproj.io/sync-wave: "-5"
data:
  APP_MODE: "production"
  API_TIMEOUT: "30s"
  WAVE_ORDER: "This ConfigMap was created in Wave -5"
```

Create `apps/wave-demo/secret.yaml` (Wave -5):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: wave-demo
  annotations:
    argocd.argoproj.io/sync-wave: "-5"
type: Opaque
stringData:
  API_KEY: "lab03-demo-key-not-real"
  DB_PASSWORD: "demo-password-not-real"
```

Create `apps/wave-demo/deployment.yaml` (Wave 0 — default):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wave-demo-app
  namespace: wave-demo
  labels:
    app: wave-demo-app
  # No sync-wave annotation = wave 0 (default)
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wave-demo-app
  template:
    metadata:
      labels:
        app: wave-demo-app
    spec:
      containers:
        - name: app
          image: nginxdemos/hello:latest
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secret
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

Create `apps/wave-demo/service.yaml` (Wave 0):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wave-demo-app
  namespace: wave-demo
spec:
  selector:
    app: wave-demo-app
  ports:
    - port: 80
      targetPort: 80
```

Create `apps/wave-demo/job-verify.yaml` (Wave 10 — last!):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: post-deploy-verify
  namespace: wave-demo
  annotations:
    argocd.argoproj.io/sync-wave: "10"   # Run LAST after everything is healthy
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: verify
          image: busybox:1.35
          command:
            - /bin/sh
            - -c
            - |
              echo "===== Wave 10: Post-Deploy Verification ====="
              echo "Testing connectivity to wave-demo-app service..."
              # Test if service is reachable
              wget -qO- http://wave-demo-app.wave-demo.svc.cluster.local
              if [ $? -eq 0 ]; then
                echo "✅ Connectivity check PASSED!"
              else
                echo "❌ Connectivity check FAILED!"
                exit 1
              fi
              echo "=============================================="
```

### 2.2 — Commit and Push

```bash
git add apps/wave-demo/
git commit -m "feat: add wave-demo application with sync waves"
git push origin main
```

### 2.3 — Create ArgoCD Application

Create `clusters/wave-demo-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: wave-demo
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-course-configs.git
    targetRevision: main
    path: apps/wave-demo
  destination:
    server: https://kubernetes.default.svc
    namespace: wave-demo
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    # Manual sync so we can observe wave behavior
```

```bash
git add clusters/wave-demo-app.yaml
git commit -m "feat: add wave-demo ArgoCD Application"
git push origin main

kubectl apply -f clusters/wave-demo-app.yaml
```

### 2.4 — Sync and Observe Waves

```bash
# Open ArgoCD UI: https://localhost:8080
# Find the wave-demo application

# Trigger sync and WATCH carefully
argocd app sync wave-demo

# In another terminal, watch resources appear one by one:
kubectl get all -n wave-demo -w
```

**What you should observe:**
1. Namespace created first (Wave -10)
2. ConfigMap + Secret created (Wave -5) → waits for them to be ready
3. Deployment + Service created (Wave 0) → waits for pods to be Ready
4. Job verify created (Wave 10) → runs after everything else is healthy

---

## Part 3 — Resource Hooks Demo

Create `apps/hook-demo/` directory:

```bash
mkdir -p apps/hook-demo
```

Create `apps/hook-demo/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hook-demo
```

Create `apps/hook-demo/pre-sync-job.yaml` (PreSync Hook):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-sync-setup
  namespace: hook-demo
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
spec:
  backoffLimit: 2
  activeDeadlineSeconds: 120
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pre-sync
          image: busybox:1.35
          command:
            - /bin/sh
            - -c
            - |
              echo "======================================"
              echo "   PRE-SYNC HOOK RUNNING"
              echo "   This runs BEFORE any other resources"
              echo "   Use for: DB migrations, validations"
              echo "======================================"
              sleep 5
              echo "✅ Pre-sync setup complete!"
```

Create `apps/hook-demo/deployment.yaml` (Regular resource):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hook-demo-app
  namespace: hook-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hook-demo-app
  template:
    metadata:
      labels:
        app: hook-demo-app
    spec:
      containers:
        - name: app
          image: nginxdemos/hello:latest
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

Create `apps/hook-demo/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hook-demo-app
  namespace: hook-demo
spec:
  selector:
    app: hook-demo-app
  ports:
    - port: 80
      targetPort: 80
```

Create `apps/hook-demo/post-sync-job.yaml` (PostSync Hook):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: post-sync-notify
  namespace: hook-demo
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: notify
          image: busybox:1.35
          command:
            - /bin/sh
            - -c
            - |
              echo "======================================"
              echo "   POST-SYNC HOOK RUNNING"
              echo "   This runs AFTER all resources are"
              echo "   Healthy. Use for: notifications,"
              echo "   smoke tests, cache warming"
              echo "======================================"
              sleep 3
              echo "🔔 Deployment complete! Notifying team..."
              echo "✅ Post-sync notification sent!"
```

Create `apps/hook-demo/sync-fail-job.yaml` (SyncFail Hook):

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sync-fail-alert
  namespace: hook-demo
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: alert
          image: busybox:1.35
          command:
            - /bin/sh
            - -c
            - |
              echo "======================================"
              echo "   SYNC FAIL HOOK RUNNING"
              echo "   Something went wrong during sync!"
              echo "   Use for: alerts, rollback triggers"
              echo "======================================"
              echo "🚨 ALERT: Sync failed! Sending alert..."
              echo "⚠️  Team notified via webhook"
```

### Commit and Create Application

```bash
git add apps/hook-demo/ clusters/
git commit -m "feat: add hook-demo application with all hook types"
git push origin main
```

Create `clusters/hook-demo-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hook-demo
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-course-configs.git
    targetRevision: main
    path: apps/hook-demo
  destination:
    server: https://kubernetes.default.svc
    namespace: hook-demo
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f clusters/hook-demo-app.yaml

# Sync and observe hooks
argocd app sync hook-demo

# Watch pods appear in sequence
kubectl get pods -n hook-demo -w

# View pre-sync hook logs
kubectl logs -n hook-demo \
  $(kubectl get pods -n hook-demo | grep pre-sync | awk '{print $1}')

# View post-sync hook logs (after deployment is healthy)
kubectl logs -n hook-demo \
  $(kubectl get pods -n hook-demo | grep post-sync | awk '{print $1}')
```

---

## Part 4 — Test ignoreDifferences with HPA

### 4.1 — Create an HPA for wave-demo

```bash
# Create an HPA manually
kubectl autoscale deployment wave-demo-app \
  -n wave-demo \
  --min=2 \
  --max=10 \
  --cpu-percent=50

# Check HPA
kubectl get hpa -n wave-demo
```

### 4.2 — Observe the Conflict

```bash
# HPA scales the deployment
# Wait for HPA to set replicas (might be a minute)
kubectl get deployment wave-demo-app -n wave-demo

# Now refresh ArgoCD
argocd app get wave-demo --refresh

# Check diff — ArgoCD wants to set replicas back to 2!
argocd app diff wave-demo
```

### 4.3 — Fix with ignoreDifferences

Update `clusters/wave-demo-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: wave-demo
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-course-configs.git
    targetRevision: main
    path: apps/wave-demo
  destination:
    server: https://kubernetes.default.svc
    namespace: wave-demo
  # ADD THIS:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas    # ← Let HPA manage replicas
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

```bash
git add clusters/wave-demo-app.yaml
git commit -m "fix: ignore replicas diff managed by HPA"
git push origin main

kubectl apply -f clusters/wave-demo-app.yaml

# Now refresh — should show as Synced even with HPA-managed replicas
argocd app get wave-demo --refresh
```

---

## Part 5 — App-of-Apps Implementation

### 5.1 — Create Parent "root-app" Application

Create `clusters/root-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-course-configs.git
    targetRevision: main
    path: clusters          # ← Watches the clusters/ directory!
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd       # ← Deploys Application resources into argocd ns
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
# Apply the root-app (this one Application manages all others!)
kubectl apply -f clusters/root-app.yaml

# Check in ArgoCD UI — you'll see:
# root-app → manages wave-demo + hook-demo Applications
argocd app list
```

### 5.2 — Add a New App via Git Only

Create `clusters/nginx-test-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-test
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx-test
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
git add clusters/nginx-test-app.yaml
git commit -m "feat: add nginx-test application via app-of-apps"
git push origin main

# root-app detects the new file and automatically creates the nginx-test Application!
# Wait 3 minutes or force refresh:
argocd app get root-app --refresh

# The nginx-test app should appear automatically
argocd app list
```

---

## Part 6 — Cleanup

```bash
# Delete root-app (cascade deletes all child apps and their resources)
argocd app delete root-app --cascade

# Or delete individually:
argocd app delete wave-demo --cascade
argocd app delete hook-demo --cascade
argocd app delete nginx-test --cascade
```

---

## 🏆 Challenges

1. **Challenge 1:** Add a sync wave to the `hook-demo` application so that the `pre-sync-job.yaml` runs in wave `-5` and the `deployment.yaml` runs in wave `0`. Observe that when BOTH hooks and waves exist, hooks run first, then waves.

2. **Challenge 2:** Deliberately break the `PostSync` hook (add `exit 1` to the command). Observe what happens to the ArgoCD sync status. Does the sync show as Failed?

3. **Challenge 3:** Implement a real `SyncFail` hook that stores the failure timestamp in a ConfigMap using kubectl within the container (mount ServiceAccount token).

---

*[← Module 03 Lecture](../README.md) | [Module 04 →](../../Module-04-Multi-Environment/README.md)*
