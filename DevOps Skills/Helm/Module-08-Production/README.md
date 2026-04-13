# Module 08 — Production Best Practices, Security & CI/CD Integration

> **Professor's Note:** This is the graduation module. Everything you've learned becomes production-grade here. We cover the full checklist a senior DevOps/Platform engineer must follow before shipping a Helm chart to production.

---

## 📖 Lecture 8.1 — The Production Helm Chart Checklist

Before any chart goes to production, it must pass this checklist:

```
CHART QUALITY
  ✅ helm lint passes with zero errors and minimal warnings
  ✅ Chart has schema validation (values.schema.json)
  ✅ All values have comments explaining them
  ✅ Chart has a meaningful README.md
  ✅ NOTES.txt includes access instructions and useful commands

RESOURCE CONFIGURATION
  ✅ All containers have resource requests AND limits
  ✅ Liveness and readiness probes are configured
  ✅ Image tags are pinned (not "latest")
  ✅ imagePullPolicy is IfNotPresent (not Always)

SECURITY
  ✅ Containers run as non-root (runAsNonRoot: true)
  ✅ readOnlyRootFilesystem: true (where possible)
  ✅ Capabilities dropped (capabilities.drop: [ALL])
  ✅ No unnecessary privilege escalation
  ✅ ServiceAccount has minimal permissions (RBAC)
  ✅ Secrets never hardcoded in templates or values.yaml

HIGH AVAILABILITY
  ✅ replicaCount >= 2 in production
  ✅ PodDisruptionBudget configured
  ✅ Anti-affinity rules to spread across nodes
  ✅ HorizontalPodAutoscaler configured

NETWORKING
  ✅ NetworkPolicies defined
  ✅ Services have appropriate type (ClusterIP for internal)
  ✅ TLS configured if exposed externally

OBSERVABILITY
  ✅ Prometheus annotations on pods
  ✅ Structured logging configured
  ✅ Health endpoint available (/health or /healthz)
```

---

## 📖 Lecture 8.2 — Security Context Best Practices

```yaml
# values.yaml — production security settings
podSecurityContext:
  runAsNonRoot: true       # Container MUST NOT run as root
  runAsUser: 1000          # Specific non-root user
  runAsGroup: 1000         # Group
  fsGroup: 1000            # Files created in volumes use this group
  seccompProfile:
    type: RuntimeDefault   # Use default seccomp profile

securityContext:
  allowPrivilegeEscalation: false   # Prevent sudo/setuid escalation
  readOnlyRootFilesystem: true      # Filesystem is read-only
  capabilities:
    drop:
      - ALL                         # Drop ALL Linux capabilities
    add:
      - NET_BIND_SERVICE           # Only add what's needed (port < 1024)
  runAsNonRoot: true
  runAsUser: 1000
```

```yaml
# templates/deployment.yaml — apply security contexts
spec:
  securityContext:
    {{- toYaml .Values.podSecurityContext | nindent 8 }}
  containers:
    - name: app
      securityContext:
        {{- toYaml .Values.securityContext | nindent 12 }}
      # If readOnlyRootFilesystem: true, containers need writable directories:
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: varrun
          mountPath: /var/run
  volumes:
    - name: tmp
      emptyDir: {}
    - name: varrun
      emptyDir: {}
```

---

## 📖 Lecture 8.3 — High Availability Configuration

### Anti-Affinity Rules

```yaml
# values.yaml
affinity:
  # Spread pods across nodes — required for true HA
  podAntiAffinity:
    # requiredDuringScheduling = hard requirement (won't schedule on same node)
    # preferredDuringScheduling = soft requirement (tries to avoid same node)
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app.kubernetes.io/name: my-webapp
        topologyKey: kubernetes.io/hostname

    # Also try to spread across availability zones
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: my-webapp
          topologyKey: topology.kubernetes.io/zone
```

### Topology Spread Constraints (Modern Approach)

```yaml
# values.yaml
topologySpreadConstraints:
  - maxSkew: 1           # Allow at most 1 pod difference per zone
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: my-webapp
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: my-webapp
```

### PodDisruptionBudget

```yaml
# templates/pdb.yaml
{{- if and .Values.podDisruptionBudget.enabled (gt (int .Values.replicaCount) 1) }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "webapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  {{ if .Values.podDisruptionBudget.minAvailable -}}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- else if .Values.podDisruptionBudget.maxUnavailable -}}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable }}
  {{- end }}
{{- end }}
```

---

## 📖 Lecture 8.4 — NetworkPolicy

```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "webapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  podSelector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow traffic from ingress controller only
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: {{ .Values.service.targetPort }}
    # Allow internal traffic from same namespace
    - from:
        - podSelector: {}
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    # Allow outbound to PostgreSQL
    {{- if .Values.postgresql.enabled }}
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: postgresql
      ports:
        - port: 5432
    {{- end }}
    # Block all other egress (allowlist approach)
{{- end }}
```

---

## 📖 Lecture 8.5 — RBAC for Service Accounts

```yaml
# templates/rbac.yaml
{{- if .Values.serviceAccount.create }}
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "webapp.serviceAccountName" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  # Disable auto-mounting of service account token (security best practice)
automountServiceAccountToken: {{ .Values.serviceAccount.automountToken | default false }}

{{- if .Values.serviceAccount.createRole }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "webapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
rules:
  {{- toYaml .Values.serviceAccount.rules | nindent 2 }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "webapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "webapp.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ include "webapp.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
{{- end }}
{{- end }}
```

---

## 📖 Lecture 8.6 — CI/CD Integration

### GitHub Actions — Complete Helm Deploy Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

env:
  RELEASE_NAME: my-webapp
  CHART_PATH: ./charts/webapp

jobs:
  lint-and-test:
    name: Lint & Validate Chart
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.13.0

      - name: Add required repos
        run: |
          helm repo add bitnami https://charts.bitnami.com/bitnami
          helm repo update

      - name: Update dependencies
        run: helm dependency update ${{ env.CHART_PATH }}

      - name: Lint chart
        run: |
          helm lint ${{ env.CHART_PATH }} \
            --values ${{ env.CHART_PATH }}/ci/lint-values.yaml

      - name: Template render check
        run: |
          helm template test-release ${{ env.CHART_PATH }} \
            --values ${{ env.CHART_PATH }}/ci/lint-values.yaml \
            > /dev/null

      - name: Validate with kubeval (optional)
        run: |
          helm template test-release ${{ env.CHART_PATH }} \
            --values ${{ env.CHART_PATH }}/ci/lint-values.yaml \
            | kubeval --strict

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: lint-and-test
    if: github.ref == 'refs/heads/staging'
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name staging-cluster

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.13.0

      - name: Add repos and update deps
        run: |
          helm repo add bitnami https://charts.bitnami.com/bitnami
          helm repo update
          helm dependency update ${{ env.CHART_PATH }}

      - name: Deploy to Staging
        run: |
          helm upgrade --install ${{ env.RELEASE_NAME }} ${{ env.CHART_PATH }} \
            --namespace staging \
            --create-namespace \
            --values ${{ env.CHART_PATH }}/values.yaml \
            --values ${{ env.CHART_PATH }}/values-staging.yaml \
            --set image.tag=${{ github.sha }} \
            --set global.environment=staging \
            --atomic \
            --timeout 10m \
            --cleanup-on-fail \
            --wait

      - name: Run Helm Tests
        run: |
          helm test ${{ env.RELEASE_NAME }} \
            --namespace staging \
            --timeout 5m

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production    # Requires manual approval in GitHub settings
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name production-cluster

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.13.0

      - name: Add repos and update deps
        run: |
          helm repo add bitnami https://charts.bitnami.com/bitnami
          helm repo update
          helm dependency update ${{ env.CHART_PATH }}

      - name: Deploy to Production
        run: |
          helm upgrade --install ${{ env.RELEASE_NAME }} ${{ env.CHART_PATH }} \
            --namespace production \
            --create-namespace \
            --values ${{ env.CHART_PATH }}/values.yaml \
            --values ${{ env.CHART_PATH }}/values-production.yaml \
            --set image.tag=${{ github.sha }} \
            --set global.environment=production \
            --atomic \
            --timeout 15m \
            --cleanup-on-fail \
            --wait \
            --history-max 10   # Keep only last 10 revisions

      - name: Run Helm Tests
        run: |
          helm test ${{ env.RELEASE_NAME }} \
            --namespace production \
            --timeout 5m

      - name: Notify on Success
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-type: application/json' \
            -d '{"text":"✅ ${{ env.RELEASE_NAME }} deployed to PRODUCTION successfully!\nCommit: ${{ github.sha }}\nAuthor: ${{ github.actor }}"}'

      - name: Notify on Failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-type: application/json' \
            -d '{"text":"❌ PRODUCTION DEPLOYMENT FAILED for ${{ env.RELEASE_NAME }}!\nCommit: ${{ github.sha }}\nAuthor: ${{ github.actor }}\nCheck: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}'
```

### CI Values Files Pattern

```
charts/webapp/
├── Chart.yaml
├── values.yaml              ← Base defaults
├── values-staging.yaml      ← Staging overrides
├── values-production.yaml   ← Production overrides
└── ci/
    └── lint-values.yaml     ← Minimal values for lint/test (CI use only)
```

```yaml
# charts/webapp/ci/lint-values.yaml
# Minimal values to make the chart lint successfully
replicaCount: 1
image:
  repository: nginx
  tag: "1.25.0"
service:
  type: ClusterIP
global:
  clusterName: ci-test
```

---

## 📖 Lecture 8.7 — Helm in ArgoCD / GitOps

```yaml
# ArgoCD Application using Helm chart
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-webapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/helm-charts
    targetRevision: HEAD
    path: charts/webapp
    helm:
      releaseName: my-webapp
      valueFiles:
        - values.yaml
        - values-production.yaml
      set:
        - name: image.tag
          value: v1.2.3
        - name: replicaCount
          value: "3"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

---

## 📖 Lecture 8.8 — Helm Chart Testing with ct (Chart Testing)

```bash
# Install the chart-testing (ct) tool
brew install chart-testing  # Mac
# or
pip3 install yamllint

# ct lint — validates chart structure and values
ct lint --chart-dirs charts --validate-maintainers=false

# ct install — installs chart and runs helm test
ct install --chart-dirs charts --namespace ct-lab

# Full CI with ct (recommended):
ct lint-and-install --chart-dirs charts
```

---

## 📖 Lecture 8.9 — Core Production Principles Summary

```
┌─────────────────────────────────────────────────────────────┐
│           HELM PRODUCTION PRINCIPLES                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. IMMUTABLE DEPLOYS                                       │
│     Always use specific image tags (never "latest")         │
│     Pin chart dependency versions in Chart.lock             │
│                                                             │
│  2. IDEMPOTENT CI/CD                                        │
│     Use `helm upgrade --install` always                     │
│     Never use `helm install` alone in pipelines             │
│                                                             │
│  3. FAIL FAST                                               │
│     Use --atomic --wait --timeout in production             │
│     Let the pipeline fail loudly and early                  │
│                                                             │
│  4. LEAST PRIVILEGE                                         │
│     ServiceAccounts with minimum RBAC permissions           │
│     Pods run as non-root with read-only filesystems         │
│                                                             │
│  5. HIGH AVAILABILITY BY DEFAULT                            │
│     replicaCount >= 2, PDB, AntiAffinity                    │
│                                                             │
│  6. SECRETS ARE EXTERNAL                                     │
│     Use External Secrets Operator, Vault, or SSM            │
│     Never commit secrets to Git                             │
│                                                             │
│  7. VALIDATION FIRST                                        │
│     helm lint + values.schema.json + ct lint                │
│     Fail in CI, not in production                           │
│                                                             │
│  8. OBSERVABILITY                                           │
│     Probes, Prometheus annotations, structured logs         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## ✅ Module 08 — Review Questions

1. Why should image tags never be `latest` in production?
2. What is the difference between `requiredDuringScheduling` and `preferredDuringScheduling` anti-affinity?
3. What does `readOnlyRootFilesystem: true` do, and what template change is required to support it?
4. What is `automountServiceAccountToken: false` and why is it a security best practice?
5. What does `--atomic` do in `helm upgrade`? When would it rollback?
6. What CI/CD pattern ensures staging is tested before production?
7. What is `ct` (chart-testing) and how does it differ from `helm lint`?
8. Why is `helm upgrade --install` preferred over `helm install` in CI/CD?

---

## 🧪 Lab 08 — Production Hardening

**→ See [labs/lab-08-production.md](./labs/lab-08-production.md)**

---

*[← Module 07](../Module-07-Repositories-Releases/README.md) | [Capstone Project →](../Capstone-Project/README.md)*
