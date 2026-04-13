# Module 03 — ArgoCD Applications: Defining, Syncing & Health

> **Professor's Note:** The `Application` CRD is the heart of ArgoCD. Everything you do in ArgoCD is through Applications. This module covers the full Application spec, sync strategies, health customization, resource hooks, and sync waves — the tools you'll use daily.

---

## 📖 Lecture 3.1 — The Full Application Spec

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-application
  namespace: argocd               # Always deploy Application to argocd namespace
  labels:
    environment: production
    team: platform
  annotations:
    argocd.argoproj.io/refresh: "normal"  # Force refresh on next reconcile

  # IMPORTANT: This finalizer ensures K8s resources are deleted when the
  # ArgoCD Application is deleted (cascade delete). Remove it to "orphan" resources.
  finalizers:
    - resources-finalizer.argocd.argoproj.io

spec:
  # Which ArgoCD Project this app belongs to (access control)
  project: default

  # ================================================================
  # SOURCE — where to get manifests
  # ================================================================
  source:
    repoURL: https://github.com/org/gitops-configs.git
    targetRevision: main          # Branch, tag, or full commit SHA
    path: apps/my-application    # Path within the repo

    # ---- If the path contains a Helm chart ----
    helm:
      # Override chart values
      valueFiles:
        - values.yaml
        - values-production.yaml
      values: |                  # Inline values (override valueFiles)
        replicaCount: 3
        image:
          tag: v1.2.3
      releaseName: my-app        # Override Helm release name (default: app name)
      parameters:                # Equivalent to helm --set
        - name: image.tag
          value: v1.2.3

    # ---- If the path contains Kustomize ----
    kustomize:
      version: v4.5.7            # Pin kustomize version
      namePrefix: prod-          # Prefix all resource names
      nameSuffix: -v2
      commonLabels:
        env: production
      images:                    # Override images
        - myapp=my-registry/myapp:v1.2.3

  # ================================================================
  # DESTINATION — where to deploy
  # ================================================================
  destination:
    # Either server OR name:
    server: https://kubernetes.default.svc   # K8s API URL
    # name: production-cluster               # ArgoCD cluster name

    namespace: my-application-ns

  # ================================================================
  # SYNC POLICY
  # ================================================================
  syncPolicy:

    # Automated sync — comment out for manual sync
    automated:
      prune: true           # Delete resources not in Git
      selfHeal: true        # Revert manual cluster changes
      allowEmpty: false     # Don't sync if rendered templates are empty (safety!)

    # Sync options
    syncOptions:
      - CreateNamespace=true              # Auto-create destination namespace
      - Validate=true                     # Validate manifests before applying
      - PruneLast=true                    # Delete old resources after new ones are healthy
      - PrunePropagationPolicy=foreground # Propagate deletion to dependents
      - Replace=false                     # Use 'replace' instead of 'apply' (use carefully)
      - ApplyOutOfSyncOnly=true           # Only apply changed resources (faster)
      - ServerSideApply=true              # Use server-side apply (better for CRDs)

    # Retry on failure
    retry:
      limit: 5               # Max retry attempts
      backoff:
        duration: 5s          # Initial retry delay
        factor: 2             # Backoff multiplier
        maxDuration: 3m       # Max retry delay

  # ================================================================
  # IGNORE DIFFERENCES — exclude some fields from diff calculation
  # ================================================================
  ignoreDifferences:
    # Ignore replicas set by HPA (otherwise ArgoCD fights HPA!)
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas

    # Ignore specific annotation on all resources
    - group: "*"
      kind: "*"
      managedFieldsManagers:
        - kube-controller-manager

    # Ignore a field by JSON Pointer
    - group: apps
      kind: StatefulSet
      name: my-statefulset
      jsonPointers:
        - /spec/volumeClaimTemplates/0/spec/storageClassName

    # Ignore using JQ expressions (more powerful)
    - group: apps
      kind: Deployment
      jqPathExpressions:
        - .spec.template.spec.initContainers[] | select(.name == "migration")
```

---

## 📖 Lecture 3.2 — Sync Strategies

### Manual Sync

The safest approach: nothing changes without an explicit human trigger.

```yaml
syncPolicy: {}  # Empty = manual sync
```

Use cases:
- Production environments where changes need approval
- CI/CD pipelines that trigger sync explicitly after validation

```bash
# Trigger sync manually
argocd app sync my-app

# Sync with specific commit
argocd app sync my-app --revision abc123

# Sync with dry run first (show what would change)
argocd app diff my-app
argocd app sync my-app --dry-run

# Sync only changed/out-of-sync resources (optimization)
argocd app sync my-app --apply-out-of-sync-only
```

### Automated Sync with Self-Heal

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

- Git push → ArgoCD detects within 3 minutes (or instantly with webhook)
- Manual kubectl change → ArgoCD reverts within ~30 seconds

### Automated Sync WITHOUT Self-Heal

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: false      # Won't revert manual changes
```

Use case: Teams that still want some ability to make emergency hotfixes via kubectl but don't want to fully bypass GitOps. **Not recommended long-term.**

---

## 📖 Lecture 3.3 — Sync Waves: Ordering Resource Creation

Sync waves control the **order** in which resources are synced. Lower waves sync first.

```yaml
# This resource syncs in wave -1 (before everything else)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # ← Sync Wave annotation
data:
  DATABASE_URL: "postgresql://..."
---
# This resource syncs in wave 0 (default)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  # No annotation = wave 0
---
# This resource syncs in wave 5 (after deployment)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    argocd.argoproj.io/sync-wave: "5"
```

### Sync Wave Execution

```
Wave -5: CRDs (Custom Resource Definitions)
Wave -1: ConfigMaps, Secrets
Wave  0: Deployments, Services (default, no annotation)
Wave  5: Ingress rules
Wave 10: Tests/verification jobs
```

Within each wave:
1. ArgoCD applies all resources in the wave
2. Waits for all resources to become **Healthy**
3. Then proceeds to the next wave

---

## 📖 Lecture 3.4 — ArgoCD Resource Hooks

Hooks allow running operations at specific points in the sync lifecycle. Similar to Helm hooks but for any ArgoCD-managed resource.

```
PreSync  →  Sync  →  PostSync  →  SyncFail (if sync failed)
```

### Hook Annotations

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync          # When to run
    argocd.argoproj.io/hook-delete-policy: HookSucceeded  # When to delete
```

### Hook Types

| Hook | Runs When |
|---|---|
| `PreSync` | Before sync begins (DB migrations!) |
| `Sync` | During sync (same as normal, but treated as hook) |
| `PostSync` | After all resources are Healthy (notifications, smoke tests) |
| `SyncFail` | Only when sync fails (alert, rollback) |
| `PostDelete` | After ArgoCD Application is deleted |

### Hook Delete Policies

| Policy | Meaning |
|---|---|
| `HookSucceeded` | Delete after hook completes successfully |
| `HookFailed` | Delete after hook fails |
| `BeforeHookCreation` | Delete previous hook run before new one starts |

### Real Pre-Sync Hook Example (DB Migration)

```yaml
# preSync-db-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  namespace: my-app
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
    argocd.argoproj.io/sync-wave: "-1"     # Run before DB migrations are pre-sync
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-migrate
          image: my-company/my-app:latest
          command: ["sh", "-c", "python manage.py migrate --no-input"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
```

### PostSync Smoke Test Hook

```yaml
# postSync-smoke-test.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  namespace: my-app
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: smoke-test
          image: curlimages/curl:latest
          command:
            - /bin/sh
            - -c
            - |
              echo "Running smoke tests..."
              curl -f http://my-app/health || exit 1
              curl -f http://my-app/api/status || exit 1
              echo "✅ All smoke tests passed!"
```

---

## 📖 Lecture 3.5 — ignoreDifferences: Managing External Controllers

The most common production pain point: ArgoCD reports `OutOfSync` on fields managed by other controllers.

### Problem 1: HPA Fighting ArgoCD

```
HPA sets deployment replicas to 5 (based on load)
ArgoCD sees: Git says 2, cluster has 5 → OutOfSync!
ArgoCD syncs: sets replicas back to 2
HPA sets replicas to 5 again...
```

**Solution:**

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas   # ← Ignore replicas in Deployment diff
```

### Problem 2: Webhook Injectors Adding Annotations

Admission webhooks (Istio, Vault, etc.) inject annotations/sidecars that ArgoCD sees as drift.

```yaml
spec:
  ignoreDifferences:
    # Ignore Istio-injected sidecar
    - group: apps
      kind: Deployment
      jqPathExpressions:
        - '.spec.template.spec.containers[] | select(.name == "istio-proxy")'

    # Ignore all managed fields (catches most webhook mutations)
    - group: "*"
      kind: "*"
      managedFieldsManagers:
        - kube-controller-manager
        - kube-scheduler
```

### Problem 3: Default Values Added by Kubernetes

```yaml
spec:
  ignoreDifferences:
    # Kubernetes adds default storage class to PVCs
    - group: ""
      kind: PersistentVolumeClaim
      jsonPointers:
        - /spec/storageClassName
        - /spec/volumeMode
```

---

## 📖 Lecture 3.6 — Health Checks

ArgoCD has built-in health checks for common Kubernetes resources:

| Resource | Health Logic |
|---|---|
| `Deployment` | Healthy when all replicas are available |
| `StatefulSet` | Healthy when all replicas are ready |
| `DaemonSet` | Healthy when all desired pods are ready |
| `Job` | Healthy when job completes successfully |
| `Pod` | Healthy when Running and all containers Ready |
| `Service` | Healthy when endpoints exist |
| `Ingress` | Healthy when load balancer IP is assigned |
| `PVC` | Healthy when bound |

### Custom Health Checks (Lua)

For custom resources (CRDs), you define health in Lua scripts:

```yaml
# In argocd-cm ConfigMap
data:
  resource.customizations.health.certmanager.io_Certificate: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.conditions ~= nil then
        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "Ready" and condition.status == "False" then
            hs.status = "Degraded"
            hs.message = condition.message
            return hs
          end
          if condition.type == "Ready" and condition.status == "True" then
            hs.status = "Healthy"
            hs.message = condition.message
            return hs
          end
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for certificate"
    return hs
```

---

## 📖 Lecture 3.7 — The App-of-Apps Pattern

As teams grow, manually managing dozens of Applications becomes unworkable. The **App-of-Apps** pattern solves this.

```
root-app (ArgoCD Application)
└── Watches: clusters/production/
    └── Contains Application manifests:
        ├── frontend-app.yaml         → Creates "frontend" Application
        ├── user-service-app.yaml     → Creates "user-service" Application
        ├── payment-service-app.yaml  → Creates "payment-service" Application
        └── infra-app.yaml            → Creates "infrastructure" Application
```

**The root app watches a directory of Application YAML files and creates them in ArgoCD.**

```yaml
# clusters/production/frontend-app.yaml
# This file LIVES in Git and creates an Application in ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/company/gitops-configs.git
    targetRevision: main
    path: apps/frontend/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: frontend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```yaml
# Root App that watches clusters/production/
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/gitops-configs.git
    targetRevision: HEAD
    path: clusters/production           # ← Watches this directory
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd                   # ← Creates Application resources in argocd ns
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## ✅ Module 03 — Review Questions

1. What is the purpose of `resources-finalizer.argocd.argoproj.io`?
2. What does `syncPolicy.automated.prune: true` risk if not used carefully?
3. What is a sync wave? Give an example of when wave ordering matters.
4. What is the difference between a sync wave and a sync hook?
5. When does a `PostSync` hook run?
6. Why would you configure `ignoreDifferences` for `/spec/replicas` on a Deployment?
7. What is the App-of-Apps pattern and what problem does it solve?
8. What Lua scripting is used for in ArgoCD?
9. What does `allowEmpty: false` protect against?

---

## 🧪 Lab 03 — Applications, Sync Waves & Hooks

**→ See [labs/lab-03-first-application.md](./labs/lab-03-first-application.md)**

---

*[← Module 02](../Module-02-ArgoCD-Architecture/README.md) | [Module 04 →](../Module-04-Multi-Environment/README.md)*
