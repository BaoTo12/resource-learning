# Module 04 — Multi-Environment Management: Dev, Staging, Production

> **Professor's Note:** This is the module where theory becomes real engineering. Every company has multiple environments. How you structure them in GitOps determines whether your platform scales cleanly or becomes chaos. There are two dominant patterns — and you need to know both deeply.

---

## 📖 Lecture 4.1 — The Multi-Environment Challenge

A real application has at least 3 environments:

```
Development  →  Staging  →  Production

Each environment needs:
  - Different replica counts (1 vs 2 vs 5)
  - Different resource limits (small vs medium vs large)
  - Different image tags (latest vs rc vs stable)
  - Different secrets (dev db vs staging db vs prod db)
  - Different ingress hosts (dev.app.com vs staging.app.com vs app.com)
  - Different auto-scaling policies
  - Different monitoring/alerting thresholds
```

How do you manage this WITHOUT duplicating all your YAML?

---

## 📖 Lecture 4.2 — Pattern 1: Kustomize (Directory-Based Overlays)

**Kustomize** is a Kubernetes-native configuration management tool built into `kubectl`. It uses a `base` + `overlays` pattern.

```
gitops-configs/
└── apps/
    └── my-webapp/
        ├── base/                     ← Shared resources (no env-specific config)
        │   ├── kustomization.yaml    ← Kustomize entry point
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── serviceaccount.yaml
        │
        └── overlays/
            ├── development/
            │   ├── kustomization.yaml
            │   └── patch-replicas.yaml    ← dev-specific patches
            │
            ├── staging/
            │   ├── kustomization.yaml
            │   ├── patch-replicas.yaml
            │   └── patch-resources.yaml
            │
            └── production/
                ├── kustomization.yaml
                ├── patch-replicas.yaml
                ├── patch-resources.yaml
                └── patch-hpa.yaml
```

### Base (apps/my-webapp/base/)

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - serviceaccount.yaml

commonLabels:
  app.kubernetes.io/name: my-webapp
  app.kubernetes.io/managed-by: argocd
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-webapp
spec:
  replicas: 1             # Base default
  selector:
    matchLabels:
      app: my-webapp
  template:
    metadata:
      labels:
        app: my-webapp
    spec:
      containers:
        - name: app
          image: my-registry/my-webapp:latest   # Image tag managed per overlay
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

### Development Overlay

```yaml
# overlays/development/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Inherit from base
bases:
  - ../../base

# Environment-specific namespace
namespace: my-webapp-dev

# Override image tag
images:
  - name: my-registry/my-webapp
    newTag: "latest"

# Override replicas
replicas:
  - name: my-webapp
    count: 1

# Add environment-specific labels
commonLabels:
  environment: development

# Apply patches
patches:
  - path: patch-resources.yaml         # Override resource requests/limits
```

```yaml
# overlays/development/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-webapp
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          env:
            - name: APP_ENVIRONMENT
              value: development
            - name: LOG_LEVEL
              value: debug
```

### Production Overlay

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namespace: my-webapp-prod

images:
  - name: my-registry/my-webapp
    newTag: "v1.2.3"         # Pinned production version!

replicas:
  - name: my-webapp
    count: 3                  # 3 replicas in production

namePrefix: prod-            # Prefix all resource names

commonLabels:
  environment: production

patches:
  - path: patch-production.yaml
```

```yaml
# overlays/production/patch-production.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-webapp
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 1Gi
          env:
            - name: APP_ENVIRONMENT
              value: production
            - name: LOG_LEVEL
              value: warn
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: my-webapp
              topologyKey: kubernetes.io/hostname
```

### ArgoCD Applications per Environment

```yaml
# ArgoCD Application for development
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-webapp-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/gitops-configs.git
    targetRevision: main
    path: apps/my-webapp/overlays/development   # ← Kustomize overlay path
  destination:
    server: https://kubernetes.default.svc
    namespace: my-webapp-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
---
# ArgoCD Application for production
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-webapp-prod
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/company/gitops-configs.git
    targetRevision: main
    path: apps/my-webapp/overlays/production    # ← Different overlay
  destination:
    server: https://production-cluster-url
    namespace: my-webapp-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
```

---

## 📖 Lecture 4.3 — Pattern 2: Helm Values per Environment

When your apps are already Helm charts, use different `values.yaml` per environment.

**Option A: Values files in Git alongside Application**

```
gitops-configs/
└── environments/
    ├── development/
    │   ├── my-webapp.yaml          ← ArgoCD Application (dev)
    │   └── values/
    │       └── my-webapp-values.yaml
    ├── staging/
    │   ├── my-webapp.yaml
    │   └── values/
    │       └── my-webapp-values.yaml
    └── production/
        ├── my-webapp.yaml
        └── values/
            └── my-webapp-values.yaml
```

```yaml
# ArgoCD Application using Helm (dev environment)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-webapp-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/helm-charts.git  # Chart repo
    targetRevision: main
    path: charts/my-webapp                               # Chart path
    helm:
      valueFiles:
        - values.yaml                                    # Base values in chart
        - $gitops/environments/development/values/my-webapp-values.yaml  # Env override
  sources:    # Multi-source: chart + values from different repos
    - repoURL: https://github.com/company/helm-charts.git
      targetRevision: main
      path: charts/my-webapp
      helm:
        valueFiles:
          - $gitops/environments/development/values/my-webapp-values.yaml
    - repoURL: https://github.com/company/gitops-configs.git  # Values repo
      targetRevision: main
      ref: gitops    # Reference name for the $gitops variable above
  destination:
    server: https://kubernetes.default.svc
    namespace: my-webapp-dev
```

---

## 📖 Lecture 4.4 — Promotion Between Environments

**Promotion** = moving a change from one environment to the next.

### Manual Promotion Pattern (Most Controlled)

```
Step 1: Code change merged to main
        ↓
Step 2: CI builds image, pushes to registry
        Image: my-registry/my-webapp:sha-abc123
        ↓
Step 3: CI opens PR to gitops-configs repo
        Updates environments/development/values.yaml:
          image.tag: sha-abc123
        ↓
Step 4: ArgoCD auto-deploys to development
        ↓
Step 5: QA team approves staging promotion
        → Merge PR/commit to staging values
        image.tag: sha-abc123
        ↓
Step 6: ArgoCD auto-deploys to staging
        ↓
Step 7: Production release approved
        → Merge PR to production values
        image.tag: sha-abc123
        ↓
Step 8: ArgoCD auto-deploys to production
        (or manual sync for extra safety)
```

### Image Updater — Automatic Promotion

[ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/) auto-updates image tags in Git when new images are pushed to a registry.

```yaml
# Annotation on Application — auto-update image from registry
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=my-registry/my-webapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver  # Track semver tags
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git   # Update Git automatically
    argocd-image-updater.argoproj.io/git-branch: main
```

---

## 📖 Lecture 4.5 — Cluster-Per-Environment vs Namespace-Per-Environment

### Option A: Namespace-per-Environment (Simpler)

```
Single Cluster:
  namespace: my-webapp-dev        ← dev
  namespace: my-webapp-staging    ← staging
  namespace: my-webapp-production ← prod
```

✅ Simpler — one cluster to manage  
✅ Lower cost  
❌ No blast radius isolation — prod broken by noisy dev  
❌ Not compliant for many regulatory requirements  

### Option B: Cluster-per-Environment (Enterprise)

```
dev-cluster.company.com:
  namespace: my-webapp

staging-cluster.company.com:
  namespace: my-webapp

prod-cluster.company.com:
  namespace: my-webapp
```

✅ Full isolation — dev can't affect prod  
✅ Different scaling policies per cluster  
✅ Compliant with SOC2, PCI, HIPAA  
❌ More infrastructure to manage  
❌ Higher cost  

---

## 📖 Lecture 4.6 — Environment Promotion Gates

Best practice: enforce quality gates before promotion.

```yaml
# .github/workflows/promote.yaml
name: Promote to Production

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to promote'
        required: true

jobs:
  gate-checks:
    runs-on: ubuntu-latest
    steps:
      - name: Check staging is healthy
        run: |
          STATUS=$(argocd app get my-webapp-staging \
            --server argocd.company.com \
            --auth-token ${{ secrets.ARGOCD_TOKEN }} \
            -o json | jq -r '.status.health.status')

          if [ "$STATUS" != "Healthy" ]; then
            echo "Staging is not Healthy! Cannot promote to production."
            exit 1
          fi

      - name: Run integration tests
        run: |
          # Run your integration test suite against staging
          npm test --env=staging

      - name: Promote to production
        run: |
          # Update production values.yaml in GitOps repo
          git clone https://github.com/company/gitops-configs
          cd gitops-configs
          sed -i "s/tag: .*/tag: ${{ github.event.inputs.image_tag }}/" \
            environments/production/values/my-webapp-values.yaml
          git commit -am "promote: my-webapp ${{ github.event.inputs.image_tag }} to prod"
          git push
```

---

## ✅ Module 04 — Review Questions

1. What is the difference between Kustomize `base` and `overlays`?
2. How does Kustomize `patches` work? What file formats does it support?
3. How do you override an image tag in Kustomize without modifying the base deployment?
4. What is "promotion" in GitOps? Why should it require a Git commit?
5. What is ArgoCD Image Updater and what problem does it solve?
6. What are the tradeoffs between namespace-per-environment and cluster-per-environment?
7. Why is `allowEmpty: false` especially important for production?

---

## 🧪 Lab 04 — Multi-Environment Setup

**→ See [labs/lab-04-environments.md](./labs/lab-04-environments.md)**

---

*[← Module 03](../Module-03-Applications-Sync/README.md) | [Module 05 →](../Module-05-Helm-Integration/README.md)*
