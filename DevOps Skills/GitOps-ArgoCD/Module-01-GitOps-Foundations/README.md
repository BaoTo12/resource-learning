# Module 01 — GitOps Foundations: The Philosophy Behind the Practice

> **Professor's Note:** Before touching ArgoCD, you must understand *why* GitOps exists. Every design decision in ArgoCD makes perfect sense once you understand the GitOps model. This is the most conceptual module — read it carefully.

---

## 📖 Lecture 1.1 — The World Before GitOps

### How Did We Deploy Before?

The traditional way of deploying to Kubernetes:

```
Developer pushes code
      ↓
CI builds Docker image, pushes to registry
      ↓
Engineer SSHs into a server
      ↓
kubectl apply -f deployment.yaml  ← Manual, error-prone
      ↓
Prays it worked
```

**Or with a CI/CD pipeline:**
```
CI pipeline runs:
  kubectl apply -f ...
  helm upgrade ...
  terraform apply ...
```

### Problems with the Traditional Model

| Problem | Description |
|---|---|
| **No source of truth** | What's actually running in the cluster? Nobody knows for sure |
| **Cluster drift** | Over time, manual changes accumulate — cluster diverges from what's in Git |
| **Audit trail gaps** | Who deployed what, when, and why? Hard to answer |
| **No rollback story** | Reverting means re-running old pipeline or manual kubectl |
| **Pipeline access** | CI system needs kubectl credentials to your PRODUCTION cluster |
| **"Works on my machine"** | Manual kubectl commands from a developer's laptop create inconsistency |
| **No continuous enforcement** | Deploy once → someone manually changes pod count → drift is never detected |

---

## 📖 Lecture 1.2 — What Is GitOps?

**GitOps** is an operational framework where:

1. **Git is the single source of truth** for all infrastructure and application configuration
2. **Desired state** is the content of your Git repository
3. **Actual state** is what's running in your cluster
4. **A GitOps operator** (ArgoCD) continuously reconciles actual → desired

> **The Core Promise:** If it's in Git → it must be in the cluster. If it's NOT in Git → it must NOT be in the cluster.

### The GitOps Definition (OpenGitOps Principles)

The 4 official GitOps principles:

```
┌─────────────────────────────────────────────────────────────┐
│              THE 4 GITOPS PRINCIPLES                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. DECLARATIVE                                             │
│     The desired system state is expressed declaratively.    │
│     (YAML files describing "what", not shell scripts        │
│     describing "how")                                       │
│                                                             │
│  2. VERSIONED AND IMMUTABLE                                 │
│     Desired state is stored in Git — versioned,             │
│     auditable, and immutable (history can't be rewritten)   │
│                                                             │
│  3. PULLED AUTOMATICALLY                                    │
│     Approved changes are automatically applied to the       │
│     system. NO push from CI into the cluster.               │
│     The CLUSTER pulls from Git — not the other way around.  │
│                                                             │
│  4. CONTINUOUSLY RECONCILED                                 │
│     Software agents continuously observe actual system      │
│     state and attempt to achieve the desired state.         │
│     Drift is detected and corrected automatically.          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📖 Lecture 1.3 — Push vs. Pull Deployment (Critical Concept)

### Traditional Push-Based Deployment

```
  Git Repo                CI/CD Pipeline            K8s Cluster
  ─────────              ─────────────────          ──────────────
  developer  →  commit → CI runs pipeline  →PUSH→  kubectl apply
                          |
                          ↑ needs cluster credentials!
                          ↑ cluster is exposed to CI!
```

**Security problem:** Your CI system needs cluster credentials. If CI is compromised → cluster is compromised.

### GitOps Pull-Based Deployment

```
  Git Repo               ArgoCD (in cluster)        K8s Cluster
  ─────────             ──────────────────          ──────────────
  developer → commit →  ArgoCD polls/watches  ←PULL←  desired state
                          |                              in Git
                          ↓ applies difference
                          → reconciles cluster

                        CI only pushes images to
                        registry + updates manifest
                        in Git — never touches cluster!
```

**Security improvement:** 
- No cluster credentials outside the cluster
- CI only needs write access to the Git repo
- The cluster pulls its own config — nothing external pushes into it

---

## 📖 Lecture 1.4 — The GitOps Workflow in Action

```
Step 1: Developer makes a code change
        git commit -m "feat: increase replicas to 3"
        git push origin main

Step 2: ArgoCD detects the Git change
        (polling every 3 minutes, or webhook for immediate)

Step 3: ArgoCD compares desired state (Git) vs actual state (cluster)
        Git says: replicas: 3
        Cluster has: replicas: 1
        → DIFF DETECTED

Step 4: ArgoCD syncs the cluster
        kubectl apply ... (done by ArgoCD internally)
        Cluster now has: replicas: 3

Step 5: ArgoCD marks the app as Synced ✅
        ArgoCD continues watching for future drifts
```

### Drift Detection

```
Scenario: Someone manually runs `kubectl scale deployment app --replicas=5`

Step 1: Cluster now has replicas: 5
        Git still says: replicas: 3
        → DRIFT DETECTED → App shows as "OutOfSync" ⚠️

Step 2: ArgoCD (if auto-sync enabled) corrects the drift:
        Syncs back to replicas: 3

Step 3: The manual change is reverted
        Git is always the winner
```

> **This is the entire power of GitOps:** Self-healing infrastructure. Any change that isn't in Git is automatically corrected back.

---

## 📖 Lecture 1.5 — GitOps Repository Structures

A key decision in GitOps is **how to organize your Git repository**.

### Structure 1: Monorepo

```
gitops-repo/
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── user-service/
│   └── payment-service/
└── infrastructure/
    ├── namespaces.yaml
    └── rbac.yaml
```

✅ Single PR touches everything  
✅ Easy to see all changes together  
❌ Can get large with many services  

### Structure 2: App-of-Apps per Service

```
gitops-repo/
├── clusters/
│   ├── production/
│   │   ├── apps.yaml          ← ArgoCD "App of Apps"
│   │   └── values-prod.yaml
│   └── staging/
│       ├── apps.yaml
│       └── values-staging.yaml
└── charts/
    ├── frontend/
    ├── user-service/
    └── payment-service/
```

✅ Environment-specific config clearly separated  
✅ Common chart, different values per environment  

### Structure 3: Environment Branches

```
main branch          → production config
staging branch       → staging config
development branch   → dev config
```

❌ **Don't use this!** Branch-per-environment is considered an anti-pattern. Branches diverge, merges become nightmares. Use directories or repos instead.

### Structure 4: Separate Repos (Most Common in Enterprise)

```
app-source-repo/         ← Application code + Dockerfile
  └── (developers work here)

gitops-config-repo/      ← Kubernetes manifests / Helm values
  ├── apps/
  ├── clusters/           ← ArgoCD Application definitions
  └── infrastructure/
  └── (DevOps/Platform team manages this)
```

✅ Clear separation of code and config  
✅ Different access controls per repo  
✅ Scale: multiple app teams, one platform team  

---

## 📖 Lecture 1.6 — What ArgoCD Is

**ArgoCD** is a GitOps **continuous delivery tool** for Kubernetes.

It is a Kubernetes-native application that:
- Runs **inside** your cluster
- **Watches** your Git repository
- **Compares** Git (desired) vs cluster (actual)
- **Syncs** the cluster to match Git
- Provides a **UI and CLI** to visualize and control the process

### ArgoCD vs Other CD Tools

| Tool | Push/Pull | GitOps Native | K8s UI | Helm Support | Multi-Cluster |
|---|---|---|---|---|---|
| **ArgoCD** | Pull | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Flux** | Pull | ✅ Yes | ❌ No (separate) | ✅ Yes | ✅ Yes |
| Jenkins | Push | ❌ No | ❌ No | Plugin | Plugin |
| GitHub Actions | Push | ❌ No | ❌ No | Plugin | ❌ No |
| Spinnaker | Push | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |

---

## ✅ Module 01 — Review Questions

1. What is the fundamental difference between push-based and pull-based deployment?
2. Name the 4 GitOps principles. Which one ensures the cluster self-heals?
3. What does "drift" mean in the context of GitOps?
4. Why is `branch-per-environment` considered an anti-pattern?
5. Why is the pull model more secure than CI pushing directly to the cluster?
6. What is the role of Git in GitOps vs traditional deployment?
7. If a developer manually runs `kubectl delete pod`, what happens in a GitOps-managed cluster?

---

## 🧪 Lab 01 — GitOps Workflow Simulation

**→ See [labs/lab-01-gitops-workflow.md](./labs/lab-01-gitops-workflow.md)**

---

*[← Course Overview](../README.md) | [Module 02 →](../Module-02-ArgoCD-Architecture/README.md)*
