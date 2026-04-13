# ✅ FINAL PROJECT CHECKLIST — Enterprise CI/CD Lifecycle
### Tracking Your Progress | Reservation Platform Graduation

---

## 🏆 Project Goal
To move from local code on your laptop to a fully automated, observable, and version-controlled production-style deployment on Kubernetes.

---

## 🏗️ Phase 1: CI (Continuous Integration)
- [ ] Create a `Dockerfile` for the Reservation Platform (Node.js/Python/Go).
- [ ] Create a `.github/workflows/ci.yml` in the project repo.
- [ ] Configure GitHub Actions to run unit tests and linter.
- [ ] Configure GitHub Actions to build a Docker image.
- [ ] Scan the image with **Trivy** for vulnerabilities.
- [ ] Successfully push the image to **GHCR** (ghcr.io).

## 📦 Phase 2: Helm Packaging
- [ ] Run `helm create reservation-chart`.
- [ ] Update `Chart.yaml` with version `1.0.0`.
- [ ] Update `values.yaml` to include your app's environment variables (DATABASE_URL, etc.).
- [ ] Parametrize the `image.tag` in `deployment.yaml`.
- [ ] Successfully run `helm lint` and `helm template` locally.
- [ ] Push the chart to your **GitOps repository**.

## 🔄 Phase 3: GitOps & Delivery
- [ ] Install **ArgoCD** in your Kubernetes cluster (Minikube/Kind).
- [ ] Create the ArgoCD `Application` manifest for the Reservation Platform.
- [ ] Create a **Personal Access Token (PAT)** on GitHub for the "Handshake."
- [ ] Add the PAT as a **GitHub Secret** in your source repo (`GITOPS_REPO_TOKEN`).
- [ ] Verify that CI automatically commits the new `image.tag` to the GitOps repo.
- [ ] Verify that ArgoCD detects the change and syncs the deployment.

## 👁️ Phase 4: Observability
- [ ] Add **OpenTelemetry SDK** to your application code.
- [ ] Configure the OTel exporter to send data to `otel-collector`.
- [ ] Deploy the **OTel Collector** using a ConfigMap in your Helm chart.
- [ ] Deploy **Jaeger** for distributed tracing.
- [ ] Verify that you can see a "Trace" in the Jaeger UI after making a request to your app.

---

## 🚀 The Graduation Test
1. Make a small code change (e.g., change a UI text or log message).
2. Commit and push to `main`.
3. **Wait 2 minutes.**
4. Check if the CI passed.
5. Check if the GitOps repo received a "chore: update image tag" commit.
6. Check if ArgoCD is "Healthy" and showing the new commit SHA.
7. Open the app in your browser and confirm the change is live.
8. Check Jaeger to see the trace for that specific request.

---

### **If all the above work, you have successfully built a full Enterprise DevOps Lifecycle!**
