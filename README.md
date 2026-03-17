# GitOps with ArgoCD — Drift Detection & Auto-Healing

A production-style GitOps lab built on a local Kubernetes cluster. Any manual change made directly to the cluster is **automatically detected and reverted within 30 seconds** by ArgoCD, which continuously reconciles live cluster state against this repository.

---

## What this project demonstrates

- **Drift detection** — ArgoCD watches this repo every 30s and flags any deviation from desired state
- **Auto-healing** — `selfHeal: true` automatically reverts unauthorized `kubectl` changes
- **Multi-environment management** — a single `ApplicationSet` manages `dev` and `staging` from one template
- **Closed-loop CI** — GitHub Actions builds images, commits the new tag back here, and ArgoCD rolls it out automatically

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     GitHub Repo                         │
│                                                         │
│   apps/guestbook/        argocd/                        │
│   ├── deployment.yaml    ├── guestbook-app.yaml         │
│   ├── service.yaml       ├── appset.yaml                │
│   └── namespace.yaml     └── argocd-cm-patch.yaml       │
└───────────────────┬─────────────────────────────────────┘
                    │ watches every 30s
                    ▼
┌─────────────────────────────────────────────────────────┐
│               Local kind Cluster                        │
│                                                         │
│   ┌─────────────┐    ┌──────────────┐                  │
│   │   ArgoCD    │───▶│  guestbook   │  namespace        │
│   │  namespace  │    │  (dev)       │                   │
│   └─────────────┘    ├──────────────┤                   │
│                      │  guestbook   │  namespace        │
│   selfHeal: true     │  (staging)   │                   │
│   prune: true        └──────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

---

## Repository structure

```
gitops-argocd-drift-detection/
│
├── .github/
│   └── workflows/
│       └── ci.yaml                  # CI pipeline — build image → commit tag → ArgoCD deploys
│
├── apps/
│   └── guestbook/
│       ├── namespace.yaml           # Creates the guestbook namespace
│       ├── deployment.yaml          # Runs the app (replicas: 2)
│       └── service.yaml             # Exposes the app on port 80
│
├── argocd/
│   ├── guestbook-app.yaml           # ArgoCD Application — points to apps/guestbook/
│   ├── appset.yaml                  # ApplicationSet — generates dev + staging apps
│   └── argocd-cm-patch.yaml         # Sets reconciliation interval to 30s
│
├── cluster/
│   └── kind-cluster.yaml            # Local 3-node kind cluster definition
│
└── README.md
```

---

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| Docker Desktop | Container runtime (WSL2 backend) | [docker.com](https://docker.com) |
| kind | Kubernetes in Docker | `curl -Lo kind ... && sudo mv kind /usr/local/bin/` |
| kubectl | Kubernetes CLI | `curl -LO ... && sudo mv kubectl /usr/local/bin/` |
| helm | Package manager | `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \| bash` |
| argocd CLI | ArgoCD CLI | `curl -sSL -o argocd ... && sudo mv argocd /usr/local/bin/` |

> All commands run inside **WSL2** (Ubuntu). Docker Desktop must have WSL2 integration enabled for your distro.

---

## Quick start

### 1. Create the cluster

```bash
cd "/mnt/d/Study/New Projects/gitops-argocd-drift-detection"

kind create cluster --name gitops-lab --config cluster/kind-cluster.yaml
kubectl cluster-info --context kind-gitops-lab
kubectl get nodes
# Expect: 3 nodes in Ready state
```

### 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=Available deployment/argocd-server \
  -n argocd --timeout=300s
```

### 3. Access the ArgoCD UI

In a dedicated terminal tab (keep this running):
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the admin password:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Open `https://localhost:8080` → login: `admin` / `<password above>`

CLI login:
```bash
argocd login localhost:8080 --username admin --insecure
```

### 4. Deploy the application

```bash
kubectl apply -f argocd/argocd-cm-patch.yaml    # set 30s sync interval
kubectl apply -f argocd/guestbook-app.yaml       # register app with ArgoCD

argocd app sync guestbook
kubectl get pods -n guestbook
# Expect: 2 pods Running
```

### 5. Deploy to multiple environments (dev + staging)

```bash
kubectl apply -f argocd/appset.yaml
argocd app list
# guestbook-dev      Synced  Healthy
# guestbook-staging  Synced  Healthy
```

---

## Drift detection — how to test it

### Test 1 — scale a deployment manually

```bash
kubectl scale deployment guestbook -n guestbook --replicas=5
kubectl get pods -n guestbook -w
```

ArgoCD detects the drift, marks the app `OutOfSync`, and reverts to 2 replicas within 30 seconds.

### Test 2 — delete a resource

```bash
kubectl delete service guestbook -n guestbook
kubectl get svc -n guestbook -w
```

ArgoCD re-creates the service automatically because of `selfHeal: true`.

### Test 3 — force an immediate sync

```bash
argocd app sync guestbook --force
```

---

## CI/CD loop (Phase 7)

Push any change to `src/` on `main` and the following happens automatically:

```
Code push → GitHub Actions builds Docker image
          → Pushes image to ghcr.io
          → Commits updated image tag to apps/guestbook/deployment.yaml
          → ArgoCD detects the new commit
          → ArgoCD rolls out the new image to the cluster
```

No manual `kubectl apply` needed at any step.

---

## Key ArgoCD concepts used

| Concept | File | What it does |
|---------|------|-------------|
| `selfHeal: true` | `guestbook-app.yaml` | Reverts any manual cluster change back to git state |
| `prune: true` | `guestbook-app.yaml` | Deletes resources that are removed from git |
| `timeout.reconciliation: 30s` | `argocd-cm-patch.yaml` | Checks for drift every 30s instead of the default 3 minutes |
| `ApplicationSet` | `appset.yaml` | Generates one ArgoCD Application per environment from a single template |

---

## Useful commands

```bash
# Check app status
argocd app get guestbook
argocd app list

# Watch pods live
kubectl get pods -n guestbook -w
kubectl get pods -n guestbook-dev -w

# Force sync immediately
argocd app sync guestbook --force

# Trigger drift (for testing)
kubectl scale deployment guestbook -n guestbook --replicas=5
kubectl delete service guestbook -n guestbook

# View ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server -f
```

---

## Recreating the cluster after a restart

`kind` clusters do not persist across Docker Desktop restarts. After a reboot:

```bash
kind create cluster --name gitops-lab --config cluster/kind-cluster.yaml
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=300s
kubectl apply -f argocd/argocd-cm-patch.yaml
kubectl apply -f argocd/guestbook-app.yaml
kubectl apply -f argocd/appset.yaml
```

---

## Tech stack

- [kind](https://kind.sigs.k8s.io/) — local multi-node Kubernetes cluster
- [ArgoCD](https://argo-cd.readthedocs.io/) — GitOps continuous delivery for Kubernetes
- [GitHub Actions](https://docs.github.com/en/actions) — CI pipeline
- [GitHub Container Registry](https://docs.github.com/en/packages) — Docker image hosting
- WSL2 + Docker Desktop — Windows-native Kubernetes development environment
