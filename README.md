# k3s-gitops

GitOps for the **cyphers.app** k3s homelab using [ArgoCD](https://argo-cd.readthedocs.io/).

Push to `main` → ArgoCD syncs → cluster updated automatically.

## Architecture

```
k3s-gitops/
├── apps/
│   ├── root.yaml                # App-of-Apps — the only thing you kubectl apply
│   ├── <app-name>/
│   │   ├── helm.yaml            # ArgoCD App → Helm chart
│   │   ├── resources.yaml       # ArgoCD App → raw K8s manifests (IngressRoutes, secrets)
│   │   ├── resources/
│   │   │   ├── ingressroute.yaml
│   │   │   ├── secret.yaml      # SOPS-encrypted
│   │   │   └── ...
│   │   └── README.md
│   └── ...
├── .githooks/
│   └── pre-commit               # Blocks committing unencrypted secrets
├── .sops.yaml                   # SOPS age encryption config
└── .gitignore
```

### How It Works

1. **root.yaml** is the only Application you apply manually
2. ArgoCD scans `apps/` recursively, discovers `helm.yaml` and `resources.yaml` files
3. `helm.yaml` → deploys upstream Helm charts with inline values
4. `resources.yaml` → deploys raw manifests from `resources/` (IngressRoutes, SOPS-encrypted secrets)
5. Root app excludes `resources/` dirs — each `resources.yaml` manages its own namespace
6. ArgoCD auto-syncs on git push (prune + self-heal enabled)

### Per-App Structure

Each app has two ArgoCD Applications:

| File | Purpose | Destination |
|------|---------|-------------|
| `helm.yaml` | Helm chart deployment | App's namespace |
| `resources.yaml` | Raw K8s manifests (IngressRoutes, secrets) | App's namespace |
| `resources/` | Directory containing raw manifests | Managed by `resources.yaml` |
| `README.md` | Docs, upgrade notes, troubleshooting | — |

## Setup

### 1. Clone and configure

```bash
git clone https://github.com/atulmgupta/-k3s-gitops.git
cd -k3s-gitops

# Enable the pre-commit hook (required for secret safety)
git config core.hooksPath .githooks

# Set git identity
git config user.name "Atul Gupta"
git config user.email "iamatulmgupta@gmail.com"
```

### 2. Install prerequisites

```bash
# SOPS and age for secret encryption
winget install FiloSottile.age
winget install Mozilla.sops

# Generate age key (if not already done)
mkdir ~/.sops
age-keygen -o ~/.sops/age-key.txt

# Set SOPS env var
# PowerShell:
[Environment]::SetEnvironmentVariable("SOPS_AGE_KEY_FILE", "$env:USERPROFILE\.sops\age-key.txt", "User")
# Bash:
export SOPS_AGE_KEY_FILE=~/.sops/age-key.txt
```

### 3. Register the repo in ArgoCD

```bash
# If your repo is private, add credentials:
argocd repo add https://github.com/atulmgupta/-k3s-gitops.git \
  --username atulmgupta \
  --password <github-pat>
```

### 4. Apply the root application

```bash
kubectl apply -f apps/root.yaml
```

That's it. ArgoCD discovers and deploys everything.

## Adding a New Service

### Step 1: Create the directory structure

```bash
mkdir -p apps/my-app/resources
```

### Step 2: Create `helm.yaml` (Helm chart deployment)

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  labels:
    tier: apps
spec:
  project: default
  source:
    repoURL: https://charts.example.com
    chart: my-app
    targetRevision: "1.*"
    helm:
      valuesObject:
        # Your Helm values here
        replicaCount: 1
        persistence:
          enabled: true
          storageClass: longhorn
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

### Step 3: Create `resources.yaml` (raw manifests)

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-resources
  namespace: argocd
  labels:
    tier: apps
spec:
  project: default
  source:
    repoURL: https://github.com/atulmgupta/-k3s-gitops.git
    targetRevision: main
    path: apps/my-app/resources
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

### Step 4: Add IngressRoute (if exposing to internet)

```yaml
# apps/my-app/resources/ingressroute.yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
  namespace: my-app
  annotations:
    kubernetes.io/ingress.class: traefik-external
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`my-app.cyphers.app`)
      middlewares:
        - name: authentik-auth
          namespace: authentik
        - name: default-headers
          namespace: traefik
      services:
        - name: my-app
          port: 8080
  tls:
    secretName: production-cyphers-app-tls
```

### Step 5: Add secrets (SOPS-encrypted)

```bash
# Create the secret file
cat > apps/my-app/resources/secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secret
  namespace: my-app
type: Opaque
stringData:
  api-key: YOUR_ACTUAL_SECRET_HERE
  db-password: YOUR_DB_PASSWORD_HERE
EOF

# Encrypt with SOPS (must be in repo root)
cd /path/to/-k3s-gitops
sops --encrypt --in-place apps/my-app/resources/secret.yaml

# Verify it's encrypted
cat apps/my-app/resources/secret.yaml  # Should show ENC[AES256_GCM,...] values
```

### Step 6: Commit and push

```bash
git add apps/my-app/
git commit -m "Add my-app"
git push
```

ArgoCD picks it up automatically via the root app.

## SOPS Secret Management

### How It Works
- Secrets are encrypted with [SOPS](https://github.com/getsops/sops) using [age](https://github.com/FiloSottile/age) encryption
- Only `data` and `stringData` fields are encrypted — metadata stays readable for PR reviews
- ArgoCD repo-server has KSOPS plugin + age private key to decrypt during sync
- A pre-commit hook blocks committing unencrypted secret files

### Encrypting a Secret

```bash
# Must be in repo root (where .sops.yaml lives)
cd /path/to/-k3s-gitops

# Create your secret YAML with plain-text values
# Then encrypt:
sops --encrypt --in-place apps/<app>/resources/secret.yaml
```

### Decrypting a Secret (for editing)

```bash
sops apps/<app>/resources/secret.yaml
# Opens in $EDITOR with decrypted values, re-encrypts on save
```

### Rotating the Age Key

```bash
# Generate new key
age-keygen -o ~/.sops/age-key-new.txt

# Update .sops.yaml with new public key
# Re-encrypt all secrets:
find apps -name '*secret*.yaml' -exec sops updatekeys {} \;

# Update the ArgoCD secret on the cluster
kubectl create secret generic sops-age-key -n argocd \
  --from-file=keys.txt=$HOME/.sops/age-key-new.txt \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart repo-server
kubectl rollout restart deployment argocd-repo-server -n argocd
```

### Pre-commit Hook

The `.githooks/pre-commit` hook automatically blocks commits containing unencrypted secrets. After cloning, enable it:

```bash
git config core.hooksPath .githooks
```

If you see `Commit blocked: Unencrypted secrets detected`, encrypt the file first:
```bash
sops --encrypt --in-place <file>
```

## Cluster Details

### Nodes
| Node | Role | IP | Status |
|------|------|----|--------|
| lithium | control-plane | 192.168.68.133 | Ready (SchedulingDisabled) |
| carbon | worker | 192.168.68.112 | Ready |
| helium | worker | 192.168.68.102 | Ready |
| boron | worker (standby) | 192.168.68.103 | NotReady (powered off) |

### Current Services

| Service | Chart Source | Namespace | Tier |
|---------|-------------|-----------|------|
| grafana | grafana/grafana (official) | grafana | monitoring |
| prometheus | prometheus-community/kube-prometheus-stack | prometheus | monitoring |
| netdata | netdata/netdata (official) | netdata | monitoring |
| traefik | traefik/traefik | traefik | infra |
| cert-manager | jetstack/cert-manager | cert-manager | infra |
| longhorn | longhorn/longhorn | longhorn-system | infra |
| metallb | metallb/metallb | metallb-system | infra |

## Daily Commands

```bash
# Check all apps
argocd app list

# Sync a specific app
argocd app sync grafana

# Check app health
argocd app get grafana

# Diff before sync
argocd app diff grafana

# Disable auto-sync temporarily
argocd app set grafana --sync-policy none

# Re-enable auto-sync
argocd app set grafana --sync-policy automated --self-heal --auto-prune

# Open ArgoCD UI
# https://argocd.cyphers.app
```
