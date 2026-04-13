# Module 04 — Chart Lifecycle, Hooks & Tests

> **Professor's Note:** Deployment is rarely a single atomic action. You often need to run a database migration before pods start, or send a notification after deployment completes, or run integration tests after install. Helm Hooks give you lifecycle control. Helm Tests let you verify your chart works correctly.

---

## 📖 Lecture 4.1 — The Helm Release Lifecycle

When you run `helm install`, Helm does more than just apply YAML:

```
helm install my-app ./chart
     │
     ├─► 1. Load chart + values
     │
     ├─► 2. Execute pre-install hooks   ← Hooks!
     │
     ├─► 3. Apply all manifest templates to K8s
     │
     ├─► 4. Wait for resources to be ready
     │
     ├─► 5. Execute post-install hooks  ← Hooks!
     │
     └─► 6. Mark release as DEPLOYED
```

Full Lifecycle Events:
```
pre-install   → before manifests are applied
post-install  → after all resources are ready
pre-delete    → before resources are deleted
post-delete   → after all resources are deleted
pre-upgrade   → before upgrade manifests are applied
post-upgrade  → after upgrade resources are ready
pre-rollback  → before rollback manifests are applied
post-rollback → after rollback is done
test          → when helm test is run (special!)
```

---

## 📖 Lecture 4.2 — What Are Helm Hooks?

A Hook is a **regular Kubernetes manifest** (Job, Pod, etc.) annotated to run at a specific lifecycle point.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "webapp.fullname" . }}-db-migrate"
  annotations:
    # THIS is what makes it a hook:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["python", "manage.py", "migrate"]
```

### Hook Annotations Explained

**`helm.sh/hook`** — Which lifecycle event(s) to trigger on:
```yaml
"helm.sh/hook": pre-install           # single event
"helm.sh/hook": pre-install,pre-upgrade  # multiple events
```

**`helm.sh/hook-weight`** — Order when multiple hooks have the same event:
```yaml
"helm.sh/hook-weight": "-10"  # runs first (lower = earlier)
"helm.sh/hook-weight": "0"    # default
"helm.sh/hook-weight": "10"   # runs last
```

**`helm.sh/hook-delete-policy`** — When to delete the hook resource:
```yaml
# before-hook-creation: delete previous run before creating new one (default)
# hook-succeeded: delete after successful completion
# hook-failed: delete if hook fails
# Combine:
"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

---

## 📖 Lecture 4.3 — Common Hook Patterns

### Pattern 1: Database Migration (pre-install, pre-upgrade)

```yaml
# templates/hooks/pre-install-db-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "webapp.fullname" . }}-db-migrate-{{ .Release.Revision }}"
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 300   # fail if not done in 5 minutes
  template:
    metadata:
      name: "{{ include "webapp.fullname" . }}-db-migrate"
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}
    spec:
      restartPolicy: Never
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: db-migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          command:
            - /bin/sh
            - -c
            - |
              echo "Running database migrations..."
              # Your migration command here:
              # python manage.py migrate
              # ./bin/rails db:migrate
              # flyway migrate
              echo "Migrations complete!"
          env:
            {{- if .Values.extraEnvVars }}
            {{- toYaml .Values.extraEnvVars | nindent 12 }}
            {{- end }}
```

### Pattern 2: Slack/Webhook Notification (post-install, post-upgrade)

```yaml
# templates/hooks/post-install-notify.yaml
{{- if .Values.notifications.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "webapp.fullname" . }}-notify-{{ .Release.Revision }}"
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: notify
          image: curlimages/curl:latest
          command:
            - /bin/sh
            - -c
            - |
              curl -X POST \
                -H 'Content-type: application/json' \
                --data '{"text":"✅ {{ .Release.Name }} deployed successfully to {{ .Release.Namespace }}! Version: {{ .Chart.AppVersion }}"}' \
                {{ .Values.notifications.slackWebhook | quote }}
{{- end }}
```

### Pattern 3: Resource Cleanup (pre-delete)

```yaml
# templates/hooks/pre-delete-cleanup.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "webapp.fullname" . }}-cleanup"
  annotations:
    "helm.sh/hook": pre-delete
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: cleanup
          image: bitnami/kubectl:latest
          command:
            - /bin/sh
            - -c
            - |
              echo "Cleaning up application data..."
              kubectl delete pvc -l app.kubernetes.io/instance={{ .Release.Name }} \
                --namespace {{ .Release.Namespace }} \
                --ignore-not-found=true
              echo "Cleanup complete!"
          serviceAccountName: {{ include "webapp.serviceAccountName" . }}
```

---

## 📖 Lecture 4.4 — Helm Tests

Helm Tests validate that your deployed application is working correctly. They are Pods/Jobs annotated with `"helm.sh/hook": test`.

```bash
# Run tests after installing a release
helm test my-release
helm test my-release -n my-namespace

# Output:
# NAME: my-release
# LAST DEPLOYED: ...
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# TEST SUITE:     my-release-webapp-test-connection
# Last Started:   ...
# Last Completed: ...
# Phase:          Succeeded
```

### Default Test Template (generated by `helm create`)

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "webapp.fullname" . }}-test-connection"
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "webapp.fullname" . }}:{{ .Values.service.port }}']
```

### Writing Comprehensive Helm Tests

```yaml
# templates/tests/test-webapp.yaml
---
# Test 1: Basic HTTP connectivity
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "webapp.fullname" . }}-test-http"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: http-test
      image: curlimages/curl:latest
      command:
        - /bin/sh
        - -c
        - |
          echo "Testing HTTP endpoint..."
          RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
            http://{{ include "webapp.fullname" . }}:{{ .Values.service.port }})
          
          if [ "$RESPONSE" = "200" ]; then
            echo "✅ HTTP test PASSED — got 200 OK"
          else
            echo "❌ HTTP test FAILED — got $RESPONSE"
            exit 1
          fi
---
# Test 2: Check health endpoint
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "webapp.fullname" . }}-test-health"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: health-test
      image: curlimages/curl:latest
      command:
        - /bin/sh
        - -c
        - |
          echo "Testing /health endpoint..."
          curl -f http://{{ include "webapp.fullname" . }}:{{ .Values.service.port }}/health \
            && echo "✅ Health check PASSED" \
            || (echo "❌ Health check FAILED" && exit 1)
---
# Test 3: Check that the deployment has the right replicas
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "webapp.fullname" . }}-test-replicas"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  serviceAccountName: {{ include "webapp.serviceAccountName" . }}
  containers:
    - name: replica-test
      image: bitnami/kubectl:latest
      command:
        - /bin/sh
        - -c
        - |
          READY=$(kubectl get deployment {{ include "webapp.fullname" . }} \
            -n {{ .Release.Namespace }} \
            -o jsonpath='{.status.readyReplicas}')
          DESIRED={{ .Values.replicaCount }}
          
          if [ "$READY" = "$DESIRED" ]; then
            echo "✅ Replica test PASSED — $READY/$DESIRED ready"
          else
            echo "❌ Replica test FAILED — $READY/$DESIRED ready"
            exit 1
          fi
```

---

## 📖 Lecture 4.5 — Hook Flow Example: True-to-Life Scenario

Let's trace a full upgrade with hooks:

```bash
helm upgrade my-webapp ./webapp --values prod-values.yaml
```

1. **pre-upgrade hooks run:**
   - Job: `db-migrate` — runs database migrations
   - Helm waits for job completion
   - If job fails → upgrade fails, rollback happens

2. **Upgrade manifests applied:**
   - `Deployment` updated → new pods slowly replacing old ones
   - `Service` updated

3. **post-upgrade hooks run:**
   - Job: `notify` — sends Slack message
   - Job: `cache-warm` — warms up application cache

4. **Release marked as DEPLOYED**

---

## 📖 Lecture 4.6 — Hook vs Regular Resource

| Aspect | Regular Resource | Hook Resource |
|---|---|---|
| Applied | On every install/upgrade | Only at specific lifecycle event |
| Included in `helm get manifest` | ✅ Yes | ❌ No |
| Tracked in release | ✅ Yes | ❌ No (unless using delete policy) |
| Blocks release | No | Yes — if hook fails, release fails |
| Lifetime | Managed by Helm | Managed by delete-policy annotation |

---

## 📖 Lecture 4.7 — Troubleshooting Hooks

```bash
# Hooks that failed leave their pods/jobs around
# List all pods including completed/failed
kubectl get pods -n my-namespace -a  # older kubectl
kubectl get pods -n my-namespace     # shows all states

# View hook job logs
kubectl logs -n my-namespace \
  $(kubectl get pods -n my-namespace | grep "db-migrate" | awk '{print $1}')

# If a hook fails, Helm stores the release as FAILED
helm list -n my-namespace
# STATUS shows: failed

# You must fix the issue and try again
# Force a new install (delete stuck release first):
helm uninstall my-webapp -n my-namespace
helm install my-webapp ...
```

---

## ✅ Module 04 — Review Questions

1. What is a Helm Hook and how is it different from a regular K8s resource?
2. Name 4 Helm hook events and describe when each fires.
3. What does `helm.sh/hook-weight` control?
4. What happens to a release if a pre-install hook **fails**?
5. What is the purpose of `hook-delete-policy: before-hook-creation`?
6. What annotation makes a Pod/Job a Helm test?
7. How do you run Helm tests after installation?
8. Should Helm tests be included in the rendered `kubectl apply` manifests? Why?

---

## 🧪 Lab 04 — Hooks & Tests

**→ See [labs/lab-04-hooks-tests.md](./labs/lab-04-hooks-tests.md)**

---

*[← Module 03](../Module-03-Templates-Values/README.md) | [Module 05 →](../Module-05-Dependencies/README.md)*
