# Lab 04 — Multi-Environment GitOps with Kustomize

> **Lab Objective:** Build a complete Kustomize-based multi-environment structure (dev/staging/production) and deploy each environment to separate namespaces via ArgoCD.
>
> **Estimated Time:** 75–90 minutes  
> **Difficulty:** ⭐⭐ Intermediate

---

## Part 1 — Build the Kustomize Directory Structure

In your `gitops-course-configs` repo:

```bash
cd gitops-course-configs

mkdir -p apps/myapp/base
mkdir -p apps/myapp/overlays/development
mkdir -p apps/myapp/overlays/staging
mkdir -p apps/myapp/overlays/production
```

---

## Part 2 — Create the Base Resources

### base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - serviceaccount.yaml
  - configmap.yaml
  - deployment.yaml
  - service.yaml

commonLabels:
  app.kubernetes.io/name: myapp
  app.kubernetes.io/managed-by: argocd
```

### base/namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp   # Will be overridden per overlay
```

### base/serviceaccount.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
automountServiceAccountToken: false
```

### base/configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  # These will be patched per environment
  APP_NAME: "My Application"
  LOG_LEVEL: "info"
  APP_ENV: "base"
```

### base/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1                # Base default — overridden per env
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
    spec:
      serviceAccountName: myapp
      containers:
        - name: app
          image: nginxdemos/hello:plain-text   # Updated per environment
          ports:
            - containerPort: 80
              name: http
          envFrom:
            - configMapRef:
                name: myapp-config
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 10
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

### base/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app.kubernetes.io/name: myapp
  ports:
    - name: http
      port: 80
      targetPort: http
```

---

## Part 3 — Development Overlay

### overlays/development/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Reference the base
bases:
  - ../../base

# Override namespace
namespace: myapp-dev

# Override image tag
images:
  - name: nginxdemos/hello
    newTag: "plain-text"    # Dev uses latest tag

# Override replica count
replicas:
  - name: myapp
    count: 1

# Add environment label to all resources
commonLabels:
  environment: development
  tier: dev

# Apply patches
patches:
  - path: patch-configmap.yaml
  - path: patch-deployment-env.yaml
```

### overlays/development/patch-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  LOG_LEVEL: "debug"
  APP_ENV: "development"
  FEATURE_DEBUG_UI: "true"
  DB_POOL_SIZE: "2"
```

### overlays/development/patch-deployment-env.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
```

---

## Part 4 — Staging Overlay

### overlays/staging/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: myapp-staging

images:
  - name: nginxdemos/hello
    newTag: "0.3"

replicas:
  - name: myapp
    count: 2

commonLabels:
  environment: staging

patches:
  - path: patch-configmap.yaml
  - path: patch-deployment-staging.yaml
```

### overlays/staging/patch-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  LOG_LEVEL: "info"
  APP_ENV: "staging"
  FEATURE_DEBUG_UI: "false"
  DB_POOL_SIZE: "5"
```

### overlays/staging/patch-deployment-staging.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

---

## Part 5 — Production Overlay

### overlays/production/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: myapp-production

images:
  - name: nginxdemos/hello
    newTag: "0.2"               # Pinned production version!

replicas:
  - name: myapp
    count: 3

commonLabels:
  environment: production

patches:
  - path: patch-configmap.yaml
  - path: patch-deployment-production.yaml

# Add security-related configs
resources:
  - hpa.yaml
  - pdb.yaml
```

### overlays/production/patch-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  LOG_LEVEL: "warn"
  APP_ENV: "production"
  FEATURE_DEBUG_UI: "false"
  DB_POOL_SIZE: "20"
```

### overlays/production/patch-deployment-production.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 512Mi
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app.kubernetes.io/name: myapp
                topologyKey: kubernetes.io/hostname
```

### overlays/production/hpa.yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
  namespace: myapp-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### overlays/production/pdb.yaml

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp
  namespace: myapp-production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
```

---

## Part 6 — Test Kustomize Locally

```bash
# Install kustomize
choco install kustomize
# OR: it's built into kubectl:
# kubectl kustomize ./path

# Preview development overlay
kubectl kustomize apps/myapp/overlays/development
# or:
kustomize build apps/myapp/overlays/development

# Count differences between environments
echo "=== DEVELOPMENT ===" && \
  kubectl kustomize apps/myapp/overlays/development | grep "replicas:" && \
  kubectl kustomize apps/myapp/overlays/development | grep "APP_ENV:"

echo "=== STAGING ===" && \
  kubectl kustomize apps/myapp/overlays/staging | grep "replicas:" && \
  kubectl kustomize apps/myapp/overlays/staging | grep "APP_ENV:"

echo "=== PRODUCTION ===" && \
  kubectl kustomize apps/myapp/overlays/production | grep "replicas:" && \
  kubectl kustomize apps/myapp/overlays/production | grep "APP_ENV:"
```

---

## Part 7 — Create ArgoCD Applications per Environment

Create `clusters/myapp-dev.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-development
  namespace: argocd
  labels:
    app: myapp
    environment: development
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-course-configs.git
    targetRevision: main
    path: apps/myapp/overlays/development
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Create `clusters/myapp-staging.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
  labels:
    app: myapp
    environment: staging
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-course-configs.git
    targetRevision: main
    path: apps/myapp/overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Create `clusters/myapp-production.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  labels:
    app: myapp
    environment: production
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-course-configs.git
    targetRevision: main
    path: apps/myapp/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false   # Safety: never auto-delete all prod resources
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true    # Delete old resources only after new ones are healthy
```

---

## Part 8 — Commit and Deploy All Environments

```bash
git add apps/myapp/ clusters/
git commit -m "feat: add myapp with dev/staging/production Kustomize overlays"
git push origin main

# Apply all Application definitions
kubectl apply -f clusters/myapp-dev.yaml
kubectl apply -f clusters/myapp-staging.yaml
kubectl apply -f clusters/myapp-production.yaml

# Watch all environments sync
argocd app list
```

---

## Part 9 — Simulate Environment Promotion

```bash
# "Promote" staging image to production
# Edit: overlays/production/kustomization.yaml
# Change: newTag: "0.2" → newTag: "0.3"  (same as staging)

git add apps/myapp/overlays/production/kustomization.yaml
git commit -m "promote: myapp v0.3 to production"
git push origin main

# ArgoCD auto-syncs production with newer image
argocd app get myapp-production --refresh
kubectl get pods -n myapp-production -w
```

---

## Part 10 — Observe Per-Environment Differences

```bash
# Compare configmap values across environments
for ns in myapp-dev myapp-staging myapp-production; do
  echo "=== $ns ==="
  kubectl get configmap myapp-config -n $ns -o jsonpath='{.data}' | jq .
done

# Compare resource limits across environments
for ns in myapp-dev myapp-staging myapp-production; do
  echo "=== $ns ==="
  kubectl get deployment myapp -n $ns \
    -o jsonpath='{.spec.template.spec.containers[0].resources}' | jq .
done

# Compare replica counts
for ns in myapp-dev myapp-staging myapp-production; do
  echo "=== $ns: $(kubectl get deployment myapp -n $ns -o jsonpath='{.spec.replicas}') replicas"
done
```

---

## Part 11 — Cleanup

```bash
argocd app delete myapp-development --cascade
argocd app delete myapp-staging --cascade
argocd app delete myapp-production --cascade
```

---

## 🏆 Challenges

1. **Challenge 1:** Add an `Ingress` resource to the production overlay only (not dev or staging) that uses `host: myapp.local`. Verify it only exists in the production namespace.

2. **Challenge 2:** Add a `staging-gate` PreSync hook that checks if all pods in development are Ready before allowing the staging sync to proceed (using kubectl in the hook container).

3. **Challenge 3:** Implement the ArgoCD Image Updater: install it and configure the development Application to automatically update the image tag when a new image is pushed to DockerHub.

---

*[← Module 04 Lecture](../README.md) | [Module 05 →](../../Module-05-Helm-Integration/README.md)*
