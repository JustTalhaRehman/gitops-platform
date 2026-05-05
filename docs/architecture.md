# Architecture

## Overview

This platform uses the ArgoCD **App-of-Apps** pattern. One parent Application (`main-pipeline`) manages the lifecycle of all other Applications. The parent itself is self-referencing, so it keeps itself alive.

```
Git Repository
      │
      ▼
main-pipeline (ArgoCD App)
      │
      ├── platform-tools (Wave 1)
      │     └── cert-manager, external-dns, cluster-autoscaler
      │
      └── monitoring (Wave 3)
            └── grafana, loki, tempo, mimir
```

## Sync Waves

ArgoCD processes Applications in wave order within a single sync:

| Wave | What deploys |
|------|-------------|
| 0    | The pipeline itself (self-reference) |
| 1    | Platform infrastructure (CRDs, operators, DNS) |
| 3+   | Application workloads |

## Adding a new environment

1. Copy `apps/production/` to `apps/new-env/`
2. Update `repoURL` and `path` fields in all `application.yaml` files
3. Apply `apps/new-env/main-pipeline-app.yaml` once to the target cluster
4. ArgoCD takes it from there

## Cluster targeting

Each Application's `destination.server` field determines which cluster it deploys to:

- `https://kubernetes.default.svc` — the cluster ArgoCD is running on
- External cluster URL — register the cluster with `argocd cluster add`
