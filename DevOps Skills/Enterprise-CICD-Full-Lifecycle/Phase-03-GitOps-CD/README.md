# 🔄 PHASE 3: GITOPS CONTINUOUS DELIVERY
### The Heart of Automation | ArgoCD + Automating Version Bumps

---

## 🎯 Objective
Automate the deployment process. We will configure **ArgoCD** to watch your GitOps repository and update the **Reservation Platform** automatically. We will also automate the "Handshake" between CI (Phase 1) and CD (Phase 3).

---

## 🏗️ 1. Create the ArgoCD Application

We want to define our application declaratively in Kubernetes. Create a file called `application.yaml` in your GitOps repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: reservation-platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your_username/reservation-platform-gitops.git
    targetRevision: HEAD
    path: charts/reservation-chart # Path to your Helm chart
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: reservation-ns # Target namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply it to your cluster:
```bash
kubectl apply -f application.yaml
```

---

## 🤝 2. The "Handshake": Automating Version Bumps

The missing link is: **How does ArgoCD know when a new Docker image is pushed in CI?**

We need to update the `values.yaml` file in the GitOps repository every time the CI pipeline finishes.

### Update Phase 1: CI Workflow (ci.yml)
Add a "Deployment Trigger" step to your GitHub Action:

```yaml
  update-gitops:
    needs: docker-build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout GitOps Repo
        uses: actions/checkout@v3
        with:
          repository: your_username/reservation-platform-gitops
          token: ${{ secrets.GITOPS_REPO_TOKEN }} # Use a Personal Access Token (PAT)

      - name: Update values.yaml with new image tag
        run: |
          sed -i "s/tag: .*/tag: \"${{ github.sha }}\"/" values.yaml
          git config user.name "CI Bot"
          git config user.email "ci@example.com"
          git add values.yaml
          git commit -m "chore: update image tag to ${{ github.sha }}"
          git push
```

---

## 🛠️ 3. Handling Secrets (Sealed Secrets - Free)

You shouldn't store raw passwords in Git. Use **Bitnami Sealed Secrets** (the industry-standard free tool):

1. **Install Sealed Secrets Controller** in your cluster.
2. **Seal your secret locally**:
   ```bash
   kubectl create secret generic db-secret --from-literal=password=mysecret --dry-run=client -o yaml | \
     kubeseal --format yaml > templates/sealed-secret.yaml
   ```
3. Commit `sealed-secret.yaml` to Git. **It is safe to public view!**

---

## ✅ Checklist for Phase 3
1. [ ] ArgoCD application is `Synced` and `Healthy`.
2. [ ] CI pipeline successfully updates the GitOps repo on every push.
3. [ ] No raw passwords exist in your repositories.
4. [ ] Deployment is fully automated from code commit to cluster sync.

---

### [Next: Phase 4 — Full Stack Observability (OpenTelemetry) →](../Phase-04-Observability/README.md)
