# GitOps Platform — ArgoCD App-of-Apps

A production-grade GitOps delivery platform built on ArgoCD using the App-of-Apps pattern. Manages multi-cluster Kubernetes deployments across environments using Kustomize overlays and Helm charts — all driven from a single Git repository.

## What this does

Instead of manually applying YAML files to your cluster, ArgoCD watches this repository. Any change you push is automatically reconciled onto the cluster. The "App-of-Apps" pattern means ArgoCD itself manages the lifecycle of all other ArgoCD Applications — so you just maintain YAML and Git does the rest.

## Structure

```
gitops-platform/
├── apps/
│   ├── production/          # Production cluster apps
│   │   ├── monitoring/      # Observability stack (Grafana, Loki, etc.)
│   │   ├── platform-tools/  # Platform addons (cert-manager, external-dns, etc.)
│   │   ├── kustomization.yaml
│   │   └── main-pipeline-app.yaml
│   └── staging/             # Staging cluster apps
│       ├── monitoring/
│       ├── platform-tools/
│       ├── kustomization.yaml
│       └── main-pipeline-app.yaml
├── bootstrap/               # One-time ArgoCD bootstrap
│   └── argocd-install.yaml
└── docs/
    └── architecture.md
```

## Prerequisites

- Kubernetes cluster (EKS, GKE, k3s — anything works)
- `kubectl` configured and pointing at your cluster
- `helm` v3+
- A GitHub/GitLab repository to host this code

## Quick Start

### 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Or use the included bootstrap manifest:

```bash
kubectl apply -f bootstrap/argocd-install.yaml
```

### 2. Update the repo URL

In `apps/production/main-pipeline-app.yaml` and `apps/staging/main-pipeline-app.yaml`, replace the `repoURL` with your own repository URL:

```yaml
source:
  repoURL: https://github.com/your-username/gitops-platform.git
```

Same change needed in each `application/application.yaml` file.

### 3. Bootstrap the App-of-Apps

```bash
kubectl apply -f apps/production/main-pipeline-app.yaml
```

ArgoCD will pick up this Application and then automatically discover and create all other Applications defined in `apps/production/kustomization.yaml`.

### 4. Access the ArgoCD UI

```bash
# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080` — username is `admin`.

## Adding a new application

1. Create a new folder under `apps/production/your-app/`
2. Add a Helm `Chart.yaml` and `values.yaml`
3. Add `application/application.yaml` pointing to that folder
4. Register it in `apps/production/kustomization.yaml` under `resources:`
5. Push — ArgoCD syncs it automatically

## Sync waves

Applications deploy in waves using the annotation:

```yaml
annotations:
  argocd.argoproj.io/sync-wave: "3"
```

- Wave `0` — infrastructure (namespaces, CRDs, network policies)
- Wave `1` — platform tools (cert-manager, external-dns, secrets operator)
- Wave `3+` — application workloads

## Key design decisions

- **Helm + ArgoCD**: each app is a Helm chart managed by ArgoCD, not `helm install`. This means ArgoCD owns drift detection and rollback.
- **Kustomize at the top**: `kustomization.yaml` composes ArgoCD Application objects, not app config — keeps separation clean.
- **Self-managing pipeline**: the `main-pipeline-app.yaml` itself is listed in `kustomization.yaml`, so the pipeline is self-healing.
- **ServerSideApply**: enabled to handle large CRDs without annotation size limits.

## Environment promotion

To promote a change from staging to production, just copy or update the relevant `values.yaml` in `apps/production/` and push. ArgoCD handles the rest.
