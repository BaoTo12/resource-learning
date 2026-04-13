# 🎓 GITOPS & ARGOCD MASTERY — University-Style Complete Course
### Professor's Edition | DevOps Engineering Program

---

> **"GitOps is not a tool — it is an operating model. Git becomes the single source of truth for everything. ArgoCD is the engine that continuously reconciles your cluster's actual state toward that truth."**
> — *Your Professor*

---

## 📋 Course Information

| Field | Details |
|---|---|
| **Course Name** | GitOps & ArgoCD: Continuous Delivery for Kubernetes |
| **Level** | Beginner → Advanced |
| **Prerequisites** | Basic Kubernetes knowledge, Git familiarity, Helm (Module 05 integrates Helm) |
| **Total Modules** | 8 Modules + Capstone Project |
| **Format** | Lecture Notes + Hands-On Labs |
| **Tools Required** | kubectl, Helm 3, ArgoCD CLI, Minikube or Kind, Git (GitHub/GitLab account) |

---

## 🗺️ Course Syllabus

| Module | Topic | Difficulty |
|---|---|---|
| [Module 01](./Module-01-GitOps-Foundations/README.md) | GitOps Philosophy, Principles & the Problem It Solves | ⭐ Beginner |
| [Module 02](./Module-02-ArgoCD-Architecture/README.md) | ArgoCD Architecture, Installation & First Look | ⭐ Beginner |
| [Module 03](./Module-03-Applications-Sync/README.md) | ArgoCD Applications: Defining, Syncing & Health | ⭐⭐ Intermediate |
| [Module 04](./Module-04-Multi-Environment/README.md) | Multi-Environment Management: Dev, Staging, Prod | ⭐⭐ Intermediate |
| [Module 05](./Module-05-Helm-Integration/README.md) | ArgoCD + Helm: Charts, Values & Secrets | ⭐⭐ Intermediate |
| [Module 06](./Module-06-ApplicationSets/README.md) | ApplicationSets, Multi-Cluster & Generators | ⭐⭐⭐ Advanced |
| [Module 07](./Module-07-RBAC-Security/README.md) | RBAC, SSO, Projects & Security Hardening | ⭐⭐⭐ Advanced |
| [Module 08](./Module-08-Advanced-Patterns/README.md) | Advanced Patterns: Notifications, Hooks, CI/CD & Progressive Delivery | ⭐⭐⭐ Advanced |
| [Capstone](./Capstone-Project/README.md) | Build a Full GitOps Platform for a Microservices Company  | ⭐⭐⭐ Advanced |

---

## 🛠️ Environment Setup (Do This First!)

### Step 1 — Start a Local Kubernetes Cluster

**Option A: Minikube (Recommended for beginners)**
```bash
minikube start --cpus=4 --memory=6144 --driver=docker
minikube status
kubectl get nodes
```

**Option B: Kind (Lightweight)**
```bash
kind create cluster --name argocd-course
kubectl cluster-info --context kind-argocd-course
```

### Step 2 — Install ArgoCD

```bash
# Create the argocd namespace
kubectl create namespace argocd

# Install ArgoCD (latest stable)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready (takes 1-2 minutes)
kubectl wait --for=condition=available --timeout=300s \
  deployment/argocd-server -n argocd

# Verify all pods are running
kubectl get pods -n argocd
```

**Expected pods:**
```
NAME                                                READY   STATUS    RESTARTS
argocd-application-controller-0                     1/1     Running   0
argocd-applicationset-controller-xxx                1/1     Running   0
argocd-dex-server-xxx                               1/1     Running   0
argocd-notifications-controller-xxx                 1/1     Running   0
argocd-redis-xxx                                    1/1     Running   0
argocd-repo-server-xxx                              1/1     Running   0
argocd-server-xxx                                   1/1     Running   0
```

### Step 3 — Install the ArgoCD CLI

```bash
# Windows (Chocolatey)
choco install argocd-cli

# Windows (scoop)
scoop install argocd

# Or download manually from GitHub:
# https://github.com/argoproj/argo-cd/releases/latest
# Download: argocd-windows-amd64.exe → rename to argocd.exe → add to PATH

# Verify
argocd version --client
```

### Step 4 — Access the ArgoCD UI

```bash
# Method 1: Port-forward (Recommended for local dev)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Visit: https://localhost:8080
# Note: Self-signed cert — click "Advanced > Proceed anyway"

# Method 2: NodePort change (Minikube)
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'
minikube service argocd-server -n argocd
```

### Step 5 — Get Initial Admin Password

```bash
# Retrieve the auto-generated admin password
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode && echo

# Save this password!
# Username: admin
# Password: <output above>
```

### Step 6 — Login via CLI

```bash
# Login (while port-forward is active)
argocd login localhost:8080 \
  --username admin \
  --password <your-password> \
  --insecure

# Verify you're logged in
argocd account get-user-info

# Change the admin password (do this now!)
argocd account update-password
```

### Step 7 — Setup Your Git Repository

You need a Git repository to store your Kubernetes manifests/Helm charts.

**Option A: GitHub (Recommended)**
1. Create a free account at github.com
2. Create a new repository: `gitops-course-configs`
3. Make it **public** (for this course — simpler)
4. Note the HTTPS URL: `https://github.com/YOUR_USERNAME/gitops-course-configs`

**Option B: GitLab, Bitbucket, or local Gitea**
- Any Git host works — ArgoCD supports all

---

## 📁 Course File Structure

```
GitOps-ArgoCD/
├── README.md                               ← You are here (Course Overview)
├── ARGOCD-QUICK-REFERENCE.md              ← Daily cheat sheet
│
├── Module-01-GitOps-Foundations/
│   ├── README.md                          ← Lecture: What is GitOps?
│   └── labs/
│       └── lab-01-gitops-workflow.md
│
├── Module-02-ArgoCD-Architecture/
│   ├── README.md
│   └── labs/
│       └── lab-02-argocd-setup.md
│
├── Module-03-Applications-Sync/
│   ├── README.md
│   └── labs/
│       └── lab-03-first-application.md
│
├── Module-04-Multi-Environment/
│   ├── README.md
│   └── labs/
│       └── lab-04-environments.md
│
├── Module-05-Helm-Integration/
│   ├── README.md
│   └── labs/
│       └── lab-05-helm-argocd.md
│
├── Module-06-ApplicationSets/
│   ├── README.md
│   └── labs/
│       └── lab-06-applicationsets.md
│
├── Module-07-RBAC-Security/
│   ├── README.md
│   └── labs/
│       └── lab-07-rbac-security.md
│
├── Module-08-Advanced-Patterns/
│   ├── README.md
│   └── labs/
│       └── lab-08-advanced.md
│
└── Capstone-Project/
    ├── README.md
    └── gitops-repo/               ← Sample GitOps repo structure
        ├── apps/
        ├── clusters/
        └── infrastructure/
```

---

## 📚 How to Take This Course

1. **Read the Module README.md** — Your lecture. Study this before coding.
2. **Complete the Lab** — The lab is the real learning. Never skip it.
3. **Create a real Git repo** — Every module builds on it. You'll end up with a real GitOps repo.
4. **Answer Review Questions** — Confirms understanding before moving on.
5. **Complete the Capstone** — Your portfolio project demonstrating production GitOps.

---

## 🎯 Learning Outcomes

By the end of this course you will:

- ✅ Deeply understand the GitOps operating model and its 4 principles
- ✅ Install and configure ArgoCD from scratch
- ✅ Create and manage ArgoCD Applications declaratively
- ✅ Implement multi-environment GitOps (dev, staging, production)
- ✅ Integrate ArgoCD with Helm charts and manage secrets securely
- ✅ Use ApplicationSets to manage dozens of applications and clusters
- ✅ Implement RBAC, SSO (GitHub/Okta), and ArgoCD Projects
- ✅ Configure notifications (Slack, webhook, email)
- ✅ Implement progressive delivery patterns (sync waves, hooks)
- ✅ Build a complete GitOps platform for a microservices company

---

## 📖 Recommended Reading

- [Argo CD Official Docs](https://argo-cd.readthedocs.io/)
- [GitOps Principles — OpenGitOps](https://opengitops.dev/)
- [Argo CD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [The GitOps Book (Weaveworks)](https://www.gitops.tech/)

---

*Start with [Module 01 →](./Module-01-GitOps-Foundations/README.md)*
