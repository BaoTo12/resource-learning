# Module 05 — ArgoCD + Helm: Charts, Values & Secrets

> **Professor's Note:** In the real world, most teams use Helm charts for their applications. This module covers everything about combining ArgoCD with Helm — from the basics of deploying Helm charts through ArgoCD, to the critical problem of secrets management in a GitOps world.

---

## 📖 Lecture 5.1 — How ArgoCD Renders Helm Charts

When you point an ArgoCD Application at a Helm chart, it works like this:

```
ArgoCD Application
  spec.source.helm.valueFiles → ["values.yaml", "values-production.yaml"]
                                         ↓
                            argocd-repo-server
                            runs: helm template <chart> -f values.yaml -f values-prod.yaml
                                         ↓
                            Rendered YAML manifests
                                         ↓
                            application-controller applies to cluster
```

**Important:** ArgoCD uses `helm template` — it does NOT use `helm install`. This means:
- Helm release history is **not** stored in the cluster
- Rollback is done via Git, not `helm rollback`
- `helm list` will show **nothing** (ArgoCD manages the lifecycle, not Helm)

---

## 📖 Lecture 5.2 — Helm Source Configuration Options

### Option A: Helm Chart from a Git Repository

```yaml
source:
  repoURL: https://github.com/company/helm-charts.git
  targetRevision: main
  path: charts/my-webapp        # Path to chart directory
  helm:
    valueFiles:
      - values.yaml             # Relative to chart root
      - values-production.yaml  # Another values file
    values: |                   # Inline values (highest precedence)
      replicaCount: 3
      image:
        tag: v1.2.3
    parameters:                 # Equivalent to --set
      - name: image.tag
        value: v1.2.3
    releaseName: my-webapp-prod  # Override release name
    version: v3                  # Helm version to use
    skipCrds: false              # Whether to skip CRD installation
```

### Option B: Helm Chart from a Repository (OCI or HTTP)

```yaml
source:
  repoURL: https://charts.bitnami.com/bitnami    # Chart repo URL
  chart: postgresql                               # Chart name
  targetRevision: "12.5.6"                       # Chart version
  helm:
    values: |
      auth:
        postgresPassword: "changeme"
      primary:
        persistence:
          enabled: true
          size: 10Gi
```

### Option C: Multiple Sources (Values from Different Repo)

This is the most powerful pattern — separate the chart code from the config values:

```yaml
sources:
  # Source 1: The Helm chart (from chart repo or OCI)
  - repoURL: https://charts.bitnami.com/bitnami
    chart: postgresql
    targetRevision: "12.5.6"
    helm:
      valueFiles:
        - $values/clusters/production/postgresql-values.yaml  # ← From source 2!

  # Source 2: The values files (from GitOps config repo)
  - repoURL: https://github.com/company/gitops-configs.git
    targetRevision: main
    ref: values    # ← This "ref" name is referenced as $values above
```

---

## 📖 Lecture 5.3 — Managing Values Across Environments

### Layered Values Strategy

```
environments/
├── base/
│   └── my-webapp-values.yaml              ← Shared defaults
├── development/
│   └── my-webapp-values.yaml              ← Dev overrides
├── staging/
│   └── my-webapp-values.yaml              ← Staging overrides
└── production/
    └── my-webapp-values.yaml              ← Prod overrides
```

Each ArgoCD Application stacks the values:

```yaml
# Development Application
spec:
  source:
    helm:
      valueFiles:
        - environments/base/my-webapp-values.yaml
        - environments/development/my-webapp-values.yaml    # overrides base

# Production Application
spec:
  source:
    helm:
      valueFiles:
        - environments/base/my-webapp-values.yaml
        - environments/production/my-webapp-values.yaml    # overrides base
```

---

## 📖 Lecture 5.4 — The Secrets Problem in GitOps ⚠️

> **This is the most important section in the entire course. Read this TWICE.**

### The Problem

GitOps says: **Git is the source of truth for everything.**  
Security says: **Never commit secrets to Git.**

These two requirements seem to conflict. How do you solve it?

### Anti-Pattern (NEVER DO THIS)

```yaml
# DON'T: Secret stored in Git as plain text
apiVersion: v1
kind: Secret
metadata:
  name: db-password
data:
  password: cGFzc3dvcmQ=    # Base64 is NOT encryption! echo "password" | base64
```

Base64 is trivially reversible. Anyone with Git access can decode your secrets.

---

## 📖 Lecture 5.5 — Secrets Solution 1: Sealed Secrets

[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) uses asymmetric cryptography to encrypt secrets that can ONLY be decrypted by the SealedSecrets controller in YOUR cluster.

```
You encrypt → Store encrypted SealedSecret in Git ✅
Controller decrypts → Creates K8s Secret in cluster
ArgoCD syncs → SealedSecret manifest to cluster
Controller watches → Creates real Secret from SealedSecret
```

### Installing Sealed Secrets

```bash
# Install via Helm
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system

# Install the CLI (kubeseal)
choco install kubeseal               # Windows
brew install kubeseal                # Mac
```

### Sealing a Secret

```bash
# Create the raw secret (NEVER COMMIT THIS)
kubectl create secret generic db-credentials \
  --from-literal=password=SuperSecretProd123 \
  --from-literal=username=admin \
  --dry-run=client \
  -o yaml > secret-raw.yaml

# Seal it with the cluster's public key
kubeseal --format yaml < secret-raw.yaml > db-credentials-sealed.yaml

# The sealed secret is safe to commit!
cat db-credentials-sealed.yaml
# apiVersion: bitnami.com/v1alpha1
# kind: SealedSecret
# metadata:
#   name: db-credentials
# spec:
#   encryptedData:
#     password: AgB3Rq...very long encrypted string...
#     username: AgC9Kp...
```

```bash
# Commit the SealedSecret (it's safe!)
git add db-credentials-sealed.yaml
git commit -m "feat: add sealed database credentials"
git push

# ArgoCD syncs the SealedSecret to the cluster
# Sealed Secrets controller decrypts and creates the real K8s Secret
kubectl get secret db-credentials   # ✅ Now exists!
```

---

## 📖 Lecture 5.6 — Secrets Solution 2: External Secrets Operator

[External Secrets Operator (ESO)](https://external-secrets.io/) connects to external secret stores (AWS SSM, Vault, GCP Secret Manager, Azure KeyVault) and creates Kubernetes Secrets from them.

```
External Secret Manager                ESO                K8s Cluster
(AWS SSM / Vault / GCP)    →     ExternalSecret CRD →  K8s Secret
                                 (synced via ArgoCD!)
```

### Installing ESO

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets-operator \
  --create-namespace \
  --set installCRDs=true
```

### ClusterSecretStore (Connect to AWS SSM)

```yaml
# This manifest IS committed to Git — it's just config, not a secret
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-ssm-store
spec:
  provider:
    aws:
      service: ParameterStore
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-creds           # K8s Secret with AWS credentials
            key: access-key-id
            namespace: external-secrets-operator
          secretAccessKeySecretRef:
            name: aws-creds
            key: secret-access-key
            namespace: external-secrets-operator
```

### ExternalSecret (Also Committed to Git)

```yaml
# THIS is stored in Git — defines what secret to pull:
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: my-app
spec:
  refreshInterval: 1h             # How often to sync from external store
  secretStoreRef:
    name: aws-ssm-store
    kind: ClusterSecretStore
  target:
    name: db-secret              # Name of the K8s Secret to create
    creationPolicy: Owner
  data:
    - secretKey: password         # Key in K8s Secret
      remoteRef:
        key: /production/database/password    # Path in AWS SSM
    - secretKey: username
      remoteRef:
        key: /production/database/username
```

ArgoCD syncs the `ExternalSecret` manifest to the cluster. ESO operator reads it and creates the actual Kubernetes Secret by pulling from AWS SSM.

---

## 📖 Lecture 5.7 — Secrets Solution 3: ArgoCD Vault Plugin

The [ArgoCD Vault Plugin (AVP)](https://argocd-vault-plugin.readthedocs.io/) allows you to store secret placeholders in Git and have AVP replace them at sync time.

```yaml
# In Git (safe to commit):
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  annotations:
    avp.kubernetes.io/path: "secret/data/production/database"  # Vault path
type: Opaque
stringData:
  password: <password>           # ← Placeholder — AVP replaces this!
  username: <username>           # ← Placeholder
```

When ArgoCD syncs, AVP:
1. Connects to HashiCorp Vault
2. Reads `secret/data/production/database`
3. Replaces `<password>` with the actual value
4. Returns the real YAML to ArgoCD

---

## 📖 Lecture 5.8 — Helm + ArgoCD Best Practices

### 1. Use `helm.releaseName` to Avoid Name Conflicts

```yaml
spec:
  source:
    helm:
      releaseName: my-webapp-production  # Explicit release name
```

### 2. Pin Chart Versions

```yaml
spec:
  source:
    chart: my-webapp
    targetRevision: "1.2.3"    # Never use "latest" or "*" in production!
```

### 3. Never use `--set` for Secrets

```yaml
# BAD: Secret in ArgoCD Application spec (ends up in ArgoCD's etcd)
helm:
  parameters:
    - name: database.password
      value: "plaintext-password"   # ← This is stored in ArgoCD's etcd!

# GOOD: Use ExternalSecret or SealedSecret
```

### 4. Use valueFiles for Complex Configs

```yaml
# BAD: Giant inline values block
helm:
  values: |
    replicaCount: 3
    image:
      repository: my-app
      tag: v1.2.3
    # ... 100 more lines ...

# GOOD: Reference files
helm:
  valueFiles:
    - values.yaml
    - values-production.yaml
```

### 5. Avoid `force: true` in Production

```yaml
# This replaces resources (kubectl replace) instead of applying
# Causes brief downtime — avoid in production
syncOptions:
  - Replace=true    # Avoid unless specifically needed
```

---

## ✅ Module 05 — Review Questions

1. When ArgoCD deploys a Helm chart, does `helm list` show the release? Why or why not?
2. What is the difference between `helm.values` (inline) and `helm.valueFiles`?
3. Why should secrets NEVER be committed to Git as plain text or base64?
4. How does Sealed Secrets ensure the SealedSecret can only be decrypted by YOUR cluster?
5. What is the role of ESO's `ClusterSecretStore`? Is it committed to Git?
6. What does AVP's `<placeholder>` syntax do at sync time?
7. What ArgoCD feature allows you to use Helm chart values from a DIFFERENT Git repository?
8. Why is pinning chart versions important in production?

---

## 🧪 Lab 05 — Helm + ArgoCD + Secrets

**→ See [labs/lab-05-helm-argocd.md](./labs/lab-05-helm-argocd.md)**

---

*[← Module 04](../Module-04-Multi-Environment/README.md) | [Module 06 →](../Module-06-ApplicationSets/README.md)*
