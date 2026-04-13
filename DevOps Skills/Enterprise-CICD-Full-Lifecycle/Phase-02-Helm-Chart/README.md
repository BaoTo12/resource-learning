# 📦 PHASE 2: PACKAGING WITH HELM
### Infrastructure as Code | Templating Your Microservice

---

## 🎯 Objective
Instead of manual YAML manifests, we will create a **Helm Chart** for the Reservation Platform. This allows us to deploy the same app with different configurations across environments (dev, staging, prod).

---

## 🏗️ 1. Create Your Chart

From the root of your project (or in your `gitops-repo/charts/`), run:

```bash
helm create reservation-chart
```

This creates a skeleton. We'll simplify it for the Reservation Platform.

---

## 📑 2. Define the Blueprint (Chart.yaml)

Edit `reservation-chart/Chart.yaml`:

```yaml
apiVersion: v2
name: reservation-platform
description: A Helm chart for the Reservation Platform
type: application
version: 1.0.0
appVersion: "1.0.0"
```

---

## ⚙️ 3. The Central Config (values.yaml)

This is the most important file. It's where you store environment-specific variables.

```yaml
replicaCount: 2

image:
  repository: ghcr.io/your_user/reservation-app
  pullPolicy: IfNotPresent
  tag: "latest" # Will be overridden by CI/CD

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: reservation.local
      paths:
        - path: /
          pathType: ImplementationSpecific

# Environment Variables (Secrets & Configs)
env:
  DATABASE_URL: "postgres://user:pass@postgres-service:5432/db"
  LOG_LEVEL: "info"
```

---

## 🛠️ 4. Templating the Deployment

Update `templates/deployment.yaml` to use these values:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "reservation-chart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "reservation-chart.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "reservation-chart.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - name: http
              containerPort: 3000
          env:
            - name: DATABASE_URL
              value: {{ .Values.env.DATABASE_URL | quote }}
```

---

## 🧪 5. Testing Locally (Linting)

Ensure your chart is valid before pushing to Git:

```bash
helm lint reservation-chart
helm template reservation-chart # Check the rendered YAML
```

---

## ✅ Checklist for Phase 2
1. [ ] Helm chart created and linted.
2. [ ] `values.yaml` reflects your application's requirements.
3. [ ] Deployment template uses `{{ .Values.image.tag }}`.
4. [ ] Chart committed to your **GitOps repository**.

---

### [Next: Phase 3 — GitOps Continuous Delivery (ArgoCD) →](../Phase-03-GitOps-CD/README.md)
