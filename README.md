# k3s-gitops

GitOps for the **cyphers.app** k3s homelab using [ArgoCD](https://argo-cd.readthedocs.io/).

Push to `main` → ArgoCD syncs → cluster updated automatically.

## Architecture

```
k3s-gitops/
├── apps/
│   ├── root.yaml              # App-of-Apps — the only thing you kubectl apply
│   ├── traefik/
│   │   └── application.yaml
│   ├── cert-manager/
│   │   └── application.yaml
│   ├── longhorn/
│   │   └── application.yaml
│   ├── metallb/
│   │   └── application.yaml
│   ├── postgres/              # (example: app with secrets)
│   │   ├── application.yaml
│   │   └── secrets.yaml       # SOPS-encrypted
│   └── ...
├── manifests/                  # Standalone K8s manifests (ingresses, PVCs)
├── .sops.yaml                  # SOPS age encryption config
└── .gitignore
```

### How It Works

1. **root.yaml** is the only Application you apply manually
2. ArgoCD scans `apps/` recursively, discovers all child Application YAMLs
3. Each Application points directly at an upstream Helm chart repo + inline values
4. ArgoCD auto-syncs on git push (prune + self-heal enabled)

No Kustomize. No HelmRepository CRDs. No boilerplate. One YAML per service.

## Setup

### 1. Push to GitHub

```bash
cd F:\github\k3s-gitops
gh repo create k3s-gitops --private --source=. --push
```

### 2. Register the repo in ArgoCD

```bash
# If your repo is private, add credentials:
argocd repo add https://github.com/atulmgupta/k3s-gitops.git \
  --username atulmgupta \
  --password <github-pat>

# Or via the ArgoCD UI: Settings → Repositories → Connect Repo
```

### 3. Apply the root application

```bash
kubectl apply -f apps/root.yaml
```

That's it. ArgoCD discovers and deploys everything.

## Adding a New Service

Create one directory: `apps/<service>/application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: redis
    targetRevision: "*"
    helm:
      valuesObject:
        global:
          storageClass: longhorn
        architecture: replication
        replica:
          replicaCount: 3
  destination:
    server: https://kubernetes.default.svc
    namespace: redis
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

Then: `git add && git commit && git push`. ArgoCD picks it up automatically.

### For OCI charts

```yaml
source:
  repoURL: ghcr.io/biosync-io/charts
  chart: vitasync
  targetRevision: "2.22.0"
```

## Daily Commands

```bash
# Check all apps
argocd app list

# Sync a specific app
argocd app sync traefik

# Check app health
argocd app get traefik

# Diff before sync
argocd app diff traefik

# Disable auto-sync temporarily
argocd app set traefik --sync-policy none

# Re-enable auto-sync
argocd app set traefik --sync-policy automated --self-heal --auto-prune

# Open ArgoCD UI
# https://argocd.cyphers.app
```

## Current Services

| Service | Chart | Namespace | Tier |
|---------|-------|-----------|------|
| traefik | traefik/traefik | traefik | infra |
| cert-manager | jetstack/cert-manager | cert-manager | infra |
| longhorn | longhorn/longhorn | longhorn-system | infra |
| metallb | metallb/metallb | metallb-system | infra |
