# 🎓 HELM MASTERY — University-Style Complete Course
### Professor's Edition | DevOps Engineering Program

---

> **"Helm is the package manager for Kubernetes — mastering it means you can deploy, manage, and scale applications on Kubernetes with confidence, repeatability, and elegance."**
> — *Your Professor*

---

## 📋 Course Information

| Field | Details |
|---|---|
| **Course Name** | Helm: Kubernetes Package Management (Complete) |
| **Level** | Beginner → Advanced |
| **Prerequisites** | Basic Linux CLI, Docker awareness, Kubernetes concepts helpful but not required |
| **Total Modules** | 8 Modules + Capstone Project |
| **Format** | Lecture Notes + Hands-On Labs |
| **Tools Required** | kubectl, Helm 3, Minikube or Kind (local K8s cluster) |

---

## 🗺️ Course Syllabus

| Module | Topic | Difficulty |
|---|---|---|
| [Module 01](./Module-01-Introduction/README.md) | Introduction to Helm & Kubernetes Packaging | ⭐ Beginner |
| [Module 02](./Module-02-First-Chart/README.md) | Your First Helm Chart | ⭐ Beginner |
| [Module 03](./Module-03-Templates-Values/README.md) | Helm Templates & Values Deep Dive | ⭐⭐ Intermediate |
| [Module 04](./Module-04-Chart-Lifecycle/README.md) | Chart Lifecycle, Hooks & Tests | ⭐⭐ Intermediate |
| [Module 05](./Module-05-Dependencies/README.md) | Chart Dependencies & Subchart Composition | ⭐⭐ Intermediate |
| [Module 06](./Module-06-Advanced-Templates/README.md) | Advanced Templating: Named Templates, Helpers & Flow Control | ⭐⭐⭐ Advanced |
| [Module 07](./Module-07-Repositories-Releases/README.md) | Helm Repositories, OCI Registries & Release Management | ⭐⭐⭐ Advanced |
| [Module 08](./Module-08-Production/README.md) | Production Best Practices, Security & CI/CD Integration | ⭐⭐⭐ Advanced |
| [Capstone](./Capstone-Project/README.md) | Build & Deploy a Full Microservices Stack with Helm | ⭐⭐⭐ Advanced |

---

## 🛠️ Environment Setup (Do This First!)

### Step 1 — Install Required Tools

**Install kubectl** (Kubernetes CLI):
```bash
# Windows (using Chocolatey)
choco install kubernetes-cli

# Or download directly from:
# https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
```

**Install Helm 3:**
```bash
# Windows (using Chocolatey)
choco install kubernetes-helm

# Or using winget
winget install Helm.Helm

# Verify installation
helm version
```

**Install Minikube (Local Kubernetes Cluster):**
```bash
# Windows (using Chocolatey)
choco install minikube

# Start your local cluster
minikube start --driver=docker --cpus=2 --memory=4096

# Verify cluster is running
kubectl get nodes
```

**Alternative: Install Kind (Kubernetes IN Docker):**
```bash
# Windows (using Chocolatey)
choco install kind

# Create a cluster
kind create cluster --name helm-course

# Verify
kubectl cluster-info --context kind-helm-course
```

### Step 2 — Verify Everything Works

```bash
# Check Helm
helm version
# Expected: version.BuildInfo{Version:"v3.x.x", ...}

# Check kubectl connected to cluster
kubectl get nodes
# Expected: NAME   STATUS   ROLES   AGE   VERSION

# Add the stable chart repository
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Confirm repos are added
helm repo list
```

### Step 3 — Recommended IDE Setup

- **VS Code** with extensions:
  - `ms-kubernetes-tools.vscode-kubernetes-tools` (Kubernetes extension)
  - `redhat.vscode-yaml` (YAML support)
  - `tim-koehler.helm-intellisense` (Helm IntelliSense)

---

## 📁 Course File Structure

```
DevOps Skills/
├── README.md                          ← You are here (Course Overview)
│
├── Module-01-Introduction/
│   ├── README.md                      ← Lecture notes
│   └── labs/
│       └── lab-01-explore-helm.md    ← Hands-on lab
│
├── Module-02-First-Chart/
│   ├── README.md
│   └── labs/
│       ├── lab-02-create-chart.md
│       └── my-first-chart/           ← Sample chart to build
│
├── Module-03-Templates-Values/
│   ├── README.md
│   └── labs/
│       ├── lab-03-templates.md
│       └── webapp-chart/
│
├── Module-04-Chart-Lifecycle/
│   ├── README.md
│   └── labs/
│       └── lab-04-hooks-tests.md
│
├── Module-05-Dependencies/
│   ├── README.md
│   └── labs/
│       └── lab-05-dependencies.md
│
├── Module-06-Advanced-Templates/
│   ├── README.md
│   └── labs/
│       └── lab-06-advanced-templates.md
│
├── Module-07-Repositories-Releases/
│   ├── README.md
│   └── labs/
│       └── lab-07-repos-releases.md
│
├── Module-08-Production/
│   ├── README.md
│   └── labs/
│       └── lab-08-production.md
│
└── Capstone-Project/
    ├── README.md
    └── charts/
        ├── frontend/
        ├── backend/
        └── platform/
```

---

## 📚 How to Take This Course

1. **Read the Module README.md** — This is your lecture. Read it thoroughly like you would read a textbook chapter.
2. **Complete the Lab** — Hands-on application of concepts. **Always do the labs.** Helm is learned by doing.
3. **Check Your Understanding** — Answer the review questions at the end of each module.
4. **Build the Capstone** — Demonstrates full mastery by applying everything together.

---

## 🎯 Learning Outcomes

By the end of this course you will:

- ✅ Understand what Helm is and why it exists
- ✅ Create Helm Charts from scratch with proper structure
- ✅ Write advanced Go-template-powered Helm templates
- ✅ Manage chart dependencies and umbrella charts
- ✅ Use Helm hooks for deployment lifecycle management
- ✅ Write Helm tests to validate deployments
- ✅ Publish charts to Helm repositories and OCI registries
- ✅ Apply Helm best practices for production environments
- ✅ Integrate Helm into CI/CD pipelines (GitHub Actions / GitLab CI)
- ✅ Deploy and manage a full microservices application with Helm

---

## 📖 Recommended Additional Reading

- [Official Helm Docs](https://helm.sh/docs/)
- [Helm Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Artifact Hub](https://artifacthub.io/) — Browse public Helm Charts
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

*Let's begin! Start with [Module 01 →](./Module-01-Introduction/README.md)*
