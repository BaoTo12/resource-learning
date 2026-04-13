# Lab 08 — Production Hardening

> **Lab Objective:** Take an existing Helm chart and harden it for production use: security contexts, PodDisruptionBudget, NetworkPolicy, RBAC, and production-grade values files.
>
> **Estimated Time:** 90–120 minutes  
> **Difficulty:** ⭐⭐⭐ Advanced

---

## Part 1 — Setup: Start with Module 02 webapp

```bash
mkdir -p C:\Users\Admin\Desktop\DevOps-Skills\labs\module-08
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-08

# Copy the webapp chart from module 02
Copy-Item -Recurse ..\module-02\webapp .
cd webapp
```

---

## Part 2 — Production values Files

### 2.1 — Create values-staging.yaml

```yaml
# values-staging.yaml — staging environment overrides
replicaCount: 2

image:
  tag: "1.25.3"

service:
  type: ClusterIP

app:
  environment: staging
  logLevel: debug

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 1

# Security hardening
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 101        # nginx default non-root user
  runAsGroup: 101
  fsGroup: 101

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false   # nginx needs to write to /tmp
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
```

### 2.2 — Create values-production.yaml

```yaml
# values-production.yaml — production environment overrides
replicaCount: 3

image:
  tag: "1.25.3"

service:
  type: ClusterIP

app:
  environment: production
  logLevel: warn

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 65

podDisruptionBudget:
  enabled: true
  minAvailable: 2       # At least 2 pods always available

# Anti-affinity: spread across nodes
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app.kubernetes.io/name
              operator: In
              values:
                - webapp
        topologyKey: kubernetes.io/hostname

# Topology spread
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: webapp

# Security hardening (stricter for prod)
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 101
  runAsGroup: 101
  fsGroup: 101
  seccompProfile:
    type: RuntimeDefault

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE

serviceAccount:
  create: true
  automountToken: false
  annotations: {}

networkPolicy:
  enabled: true

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "80"
  prometheus.io/path: "/"
```

### 2.3 — Create ci/lint-values.yaml

```yaml
# ci/lint-values.yaml — minimal values for CI lint only
replicaCount: 1
image:
  repository: nginx
  tag: "1.25.3"
service:
  type: ClusterIP
app:
  environment: development
```

---

## Part 3 — Add Security Context to Deployment

Update `webapp/templates/deployment.yaml` to include `topologySpreadConstraints`:

```yaml
# At the bottom of the spec.template.spec section, add:
      {{- with .Values.topologySpreadConstraints }}
      topologySpreadConstraints:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

Also add `automountServiceAccountToken` to serviceaccount.yaml:

```yaml
# webapp/templates/serviceaccount.yaml - add this line:
automountServiceAccountToken: {{ .Values.serviceAccount.automountToken | default false }}
```

---

## Part 4 — Create PodDisruptionBudget Template

Create `webapp/templates/pdb.yaml`:

```yaml
{{- if and .Values.podDisruptionBudget.enabled (gt (int .Values.replicaCount) 1) }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "webapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  {{- if .Values.podDisruptionBudget.minAvailable }}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- else if .Values.podDisruptionBudget.maxUnavailable }}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable }}
  {{- end }}
{{- end }}
```

Add to values.yaml:
```yaml
podDisruptionBudget:
  enabled: false
  minAvailable: 1
```

---

## Part 5 — Create NetworkPolicy Template

Create `webapp/templates/networkpolicy.yaml`:

```yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "webapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow HTTP traffic from any pod within the namespace
    - from:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: {{ .Values.service.targetPort | default 80 }}
    # Allow traffic from ingress-nginx namespace
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: {{ .Values.service.targetPort | default 80 }}
  egress:
    # Allow DNS resolution
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    # Allow outbound HTTPS (for external API calls if needed)
    - ports:
        - port: 443
          protocol: TCP
{{- end }}
```

Add to values.yaml:
```yaml
networkPolicy:
  enabled: false
```

---

## Part 6 — Add values.schema.json

Create `webapp/values.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["replicaCount", "image", "service"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": { "type": "string", "minLength": 1 },
        "tag": { "type": "string" },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"]
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer", "ExternalName"]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    },
    "app": {
      "type": "object",
      "properties": {
        "environment": {
          "type": "string",
          "enum": ["development", "staging", "production"]
        },
        "logLevel": {
          "type": "string",
          "enum": ["debug", "info", "warn", "error"]
        }
      }
    }
  }
}
```

---

## Part 7 — Lint and Validate

```bash
cd C:\Users\Admin\Desktop\DevOps-Skills\labs\module-08

# Lint with base values
helm lint ./webapp

# Lint with staging values
helm lint ./webapp --values ./webapp/values-staging.yaml

# Lint with production values
helm lint ./webapp --values ./webapp/values-production.yaml

# Test schema validation works
helm lint ./webapp --set service.type=Invalid
# Should fail with schema error

# Template render check for all environments
helm template webapp ./webapp --values ./webapp/values-staging.yaml > /dev/null && echo "Staging: OK"
helm template webapp ./webapp --values ./webapp/values-production.yaml > /dev/null && echo "Production: OK"
```

---

## Part 8 — Deploy to Staging

```bash
kubectl create namespace staging

helm upgrade --install webapp ./webapp \
  -n staging \
  --values ./webapp/values-staging.yaml \
  --atomic \
  --timeout 5m \
  --wait \
  --debug

# Verify security contexts
kubectl get pods -n staging
POD=$(kubectl get pods -n staging -o jsonpath='{.items[0].metadata.name}')

# Check security context was applied
kubectl get pod $POD -n staging -o jsonpath='{.spec.securityContext}' | jq .

# Check PDB was created
kubectl get pdb -n staging

# Check HPA was created
kubectl get hpa -n staging
```

---

## Part 9 — Compare Environments

```bash
# Render both environments and compare
helm template webapp ./webapp --values ./webapp/values-staging.yaml > staging-rendered.yaml
helm template webapp ./webapp --values ./webapp/values-production.yaml > prod-rendered.yaml

# Show differences
diff staging-rendered.yaml prod-rendered.yaml | head -50
```

---

## Part 10 — Run Helm Tests

```bash
helm test webapp -n staging --logs

# Expected: all tests pass
```

---

## Part 11 — Simulate CI/CD Pipeline Manually

```bash
# Step 1: Simulated CI lint stage (what GitHub Actions would do)
echo "=== CI Stage: Lint ==="
helm lint ./webapp --values ./webapp/ci/lint-values.yaml
echo "Lint passed!"

# Step 2: Simulated template render check
echo "=== CI Stage: Template Render ==="
helm template ci-test ./webapp --values ./webapp/ci/lint-values.yaml > /dev/null
echo "Template render passed!"

# Step 3: Simulated deploy to staging
echo "=== CI Stage: Deploy Staging ==="
helm upgrade --install webapp ./webapp \
  -n staging \
  --values ./webapp/values-staging.yaml \
  --set image.tag="1.25.3" \
  --atomic \
  --timeout 5m

# Step 4: Run tests
echo "=== CI Stage: Integration Tests ==="
helm test webapp -n staging

echo "=== All stages passed! Ready for production deployment. ==="
```

---

## Part 12 — Cleanup

```bash
helm uninstall webapp -n staging
kubectl delete namespace staging
```

---

## 🏆 Final Challenges (Before the Capstone)

1. **Challenge 1:** Add a `CHANGELOG.md` to the chart and configure Artifact Hub annotations to reference it. Add `artifacthub.io/changes` annotation in `Chart.yaml`.

2. **Challenge 2:** Implement a production-ready `README.md` for the webapp chart that documents: parameters table, prerequisites, installation instructions, and upgrade notes.

3. **Challenge 3:** Write a shell script `deploy.sh` that:
   - Accepts `--env (staging|production)` and `--version` arguments
   - Runs `helm lint` first
   - Tags the image in the values
   - Deploys with `--atomic`
   - Runs `helm test` after deployment
   - Sends a notification (echo to console) on success/failure

---

*[← Module 08 Lecture](../README.md) | [Capstone Project →](../../Capstone-Project/README.md)*
