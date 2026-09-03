# GitOps ArgoCD Demo

A full GitOps CI/CD demo using **GitHub Actions**, **ArgoCD**, and **ArgoCD Image Updater**. Every push to `main` automatically builds new Docker images, pushes them to Docker Hub, and triggers a live Kubernetes deployment — no manual `kubectl` needed.

---

## Architecture

```
GitHub Push
    │
    ▼
GitHub Actions CI
  ├─ Builds main-service Docker image
  └─ Builds aux-service Docker image
    │
    ▼
Docker Hub (aries5533/main-service, aries5533/aux-service)
    │
    ▼
ArgoCD Image Updater
  └─ Detects new image tag
  └─ Updates Kubernetes-manifest/kustomization.yaml via git
    │
    ▼
ArgoCD (watches repo, auto-syncs)
    │
    ▼
Kubernetes Cluster (argocd-demo-ns namespace)
  ├─ main-deployment  (LoadBalancer, public-facing)
  └─ aux-deployment   (ClusterIP, internal)
```

---

## Prerequisites

- A Kubernetes cluster (minikube, kind, EKS, GKE, etc.)
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/getting_started/) installed in the `argocd` namespace
- [ArgoCD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable/install/start/) installed
- A Docker Hub account
- GitHub repository secrets configured (see below)

---

## GitHub Secrets Required

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_PASSWORD` | Your Docker Hub access token |

---

## Setup

### 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Install ArgoCD Image Updater

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

### 3. Create the git credentials secret for Image Updater write-back

```bash
kubectl create secret generic git-creds \
  --namespace argocd \
  --from-literal=username=<your-github-username> \
  --from-literal=password=<your-github-pat>
```

### 4. Apply the ArgoCD Application

```bash
kubectl apply -f application.yaml
```

### 5. Apply the Image Updater config

```bash
kubectl apply -f image-updater.yaml
```

ArgoCD will create the `argocd-demo-ns` namespace and deploy both services automatically.

---

## How the GitOps Loop Works

1. You push a change to `main-api/` or `auxiliary-service/`
2. GitHub Actions builds only the affected service and pushes a new image tagged with the GitHub run number (e.g., `:42`)
3. ArgoCD Image Updater polls Docker Hub, detects the new tag, and commits an update to `Kubernetes-manifest/kustomization.yaml`
4. ArgoCD detects the new commit and syncs the cluster — rolling out the updated pods with zero downtime

---

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── main.yaml           # CI: build & push per-service on path change
├── Kubernetes-manifest/
│   ├── main-api.yaml           # Deployment + LoadBalancer Service for main-service
│   ├── aux-api.yaml            # Deployment + ClusterIP Service for aux-service
│   └── kustomization.yaml      # Image tags (auto-managed by Image Updater)
├── main-api/
│   └── Dockerfile
├── auxiliary-service/
│   └── Dockerfile
├── application.yaml            # ArgoCD Application manifest
└── image-updater.yaml          # ArgoCD Image Updater manifest
```

---

## Services

| Service | Type | Description |
|---|---|---|
| `main-service` | LoadBalancer | Public-facing entry point |
| `aux-service` | ClusterIP | Internal service, only reachable within the cluster |

---

## Improvements Made (vs. initial version)

- Added liveness and readiness probes to both deployments
- Added CPU and memory resource requests/limits
- Removed `:latest` tag from base manifests (Kustomize manages tags)
- Made `aux-service` type explicit (`ClusterIP`) with a comment
- Split CI workflow into independent per-service jobs (no cross-rebuilds)
- Added comment to `kustomization.yaml` explaining auto-managed tags
