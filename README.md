# Actions Runner GitOps

GitOps repository for deploying GitHub Actions self-hosted runners using Flux CD.

## Structure

```
actions-runner-gitops/
├── clusters/                    # Cluster-specific configurations
│   ├── aks-dev/                 # Dev cluster
│   │   ├── flux-system/         # Created by Flux bootstrap
│   │   ├── infrastructure/      # Infrastructure Kustomization
│   │   └── apps/                # Apps Kustomization
│   └── aks-prod/                # Prod cluster
│       ├── flux-system/
│       ├── infrastructure/
│       └── apps/
├── infrastructure/              # Shared infrastructure components
│   ├── sources/                 # HelmRepository & OCIRepository
│   └── controllers/             # ESO, cert-manager configs
└── apps/                        # Application deployments
    └── runners/                 # Runner configurations
        ├── base/                # Base HelmRelease
        └── overlays/            # Environment overlays
            ├── dev/
            └── prod/
```

## Bootstrap

Flux bootstrap is triggered via IssueOps on the InfraCreator repo:

```
/bootstrap-flux-dev    # Bootstrap dev cluster
/bootstrap-flux-prod   # Bootstrap prod cluster
```

This will:
1. Install Flux on the AKS cluster
2. Create the `flux-system` namespace
3. Setup GitOps sync from this repo
4. Push GitHub PAT to Azure Key Vault for ESO

## Deploying Runners

After bootstrap, deploy runners with:

```
/deploy-arc-dev    # Deploy runners on dev
/deploy-arc-prod   # Deploy runners on prod
```

## Manual Sync

To manually sync:

```bash
flux reconcile source git flux-system
flux reconcile kustomization infrastructure
flux reconcile kustomization apps
```

## Components

| Component | Purpose |
|-----------|---------|
| cert-manager | TLS certificate management |
| External Secrets | Sync secrets from Azure Key Vault |
| ARC Controller | GitHub Actions Runner Controller |
| Runner Scale Set | Auto-scaling runner pods |

## Runner Group

- **Name:** `straw-hat-runners-linux`
- **Labels:** `self-hosted`, `linux`, `x64`, `straw-hat-runners-linux`

### Usage

```yaml
jobs:
  build:
    runs-on: straw-hat-runners-linux
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running on AKS!"
```
