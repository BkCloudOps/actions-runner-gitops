# apps/runners — flavor + per-type layout

```
apps/runners/
├── controller.yaml                # ARC controller (shared by gh-supported)
├── base/
│   ├── gh-supported/              # actions/gha-runner-scale-set (official)
│   │   ├── kustomization.yaml
│   │   └── runner-scale-set.yaml
│   └── community/                 # actions-runner-controller (summerwind, legacy)
│       ├── kustomization.yaml
│       └── runner-deployment.yaml
├── shared/                        # ExternalSecrets (github-pat-secret, ghcr-pull-secret)
│   ├── kustomization.yaml
│   └── external-secrets.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml     # aggregator
    │   └── dind/                  # community ARC: straw-hat-runners-linux-dev-dind
    └── prod/
        ├── kustomization.yaml
        ├── dind/                  # straw-hat-runners-linux-prod-dind
        ├── kubernetes-pvc/        # straw-hat-runners-linux-prod-kubernetes-pvc
        └── kubernetes-novolume/   # straw-hat-runners-linux-prod-kubernetes-novolume
```

## Naming contract

- Runner scale-set name & GitHub runner group: **`straw-hat-runners-linux-<env>-<type>`**
- Runner image: **`ghcr.io/strawhat-enterprise/actions-runner:linux-<type>-latest`** (pulled with `ghcr-pull-secret`, synced from Key Vault by ESO)

## Flavors

| Flavor | Chart | CRDs | Notes |
|--------|-------|------|-------|
| `gh-supported` | `actions/gha-runner-scale-set` + `gha-runner-scale-set-controller` | `AutoscalingRunnerSet`, `EphemeralRunnerSet`, `EphemeralRunner`, `AutoscalingListener` | Current / recommended. See [`docs/13-crd-reference.md`](../../../actions-runner-helm/docs/13-crd-reference.md) in `actions-runner-helm`. |
| `community` | `actions-runner-controller` (summerwind) | `RunnerDeployment`, `Runner`, `HorizontalRunnerAutoscaler` | Legacy. Kept for migration purposes. |

## Bootstrap prerequisites (NOT in this layer)

These live in `infrastructure/controllers/` and must be reconciled first:

- `cert-manager`
- `external-secrets-operator` + a `ClusterSecretStore` pointing at Azure Key Vault
- The `ExternalSecret`s for `github-pat-secret` and `ghcr-pull-secret` are in `apps/runners/shared/` so they are scoped to the runner namespace but still managed alongside the runners.

## Cluster role mapping

- `dev` cluster: `community` flavor (legacy ARC) with `dind` runner type.
- `prod` cluster: `gh-supported` flavor with all requested runner types:
    - `dind`
    - `kubernetes-pvc` (container mode `kubernetes` + PVC)
    - `kubernetes-novolume` (container mode `kubernetes-novolume`)

## How types are toggled

Edit the env aggregator `apps/runners/overlays/<env>/kustomization.yaml` to add or remove a type entry. Flux will reconcile (and prune removed types). The IssueOps workflow `issueops-deploy-runners-v2.yml` in `InfraCreator` generates this file from a single comment:

```
/deploy-runners env=dev flavor=gh-supported types=dind,kubernetes

/deploy-runners env=prod flavor=gh-supported types=dind,kubernetes-pvc,kubernetes-novolume
```
