# k3s-gitops — Copilot Instructions

## Project Overview

This is a **k3s GitOps repository** managed by ArgoCD. It defines all Kubernetes
applications, secrets, and infrastructure for a home k3s cluster using a
declarative, Git-driven workflow.

## Repository Structure

```
apps/
  root.yaml              # Root ArgoCD Application — auto-discovers all apps
  <app-name>/
    helm.yaml            # ArgoCD Application (Helm or Kustomize source)
    resources.yaml       # (optional) Separate ArgoCD Application for SOPS secrets
    resources/
      kustomization.yaml # Kustomize config with KSOPS generators
      ksops-generator.yaml
      secret.yaml        # SOPS-encrypted secrets (age-encrypted)
```

## Architecture

### ArgoCD App-of-Apps Pattern
- `apps/root.yaml` is the only manually applied resource (`kubectl apply -f apps/root.yaml`)
- Root app recursively discovers all YAML under `apps/`, excluding `**/resources/**`
- Each app directory contains an ArgoCD `Application` manifest

### Two Types of ArgoCD Applications

1. **Helm apps** (`helm.yaml`) — deploy charts from OCI registries or Git repos
2. **Resources apps** (`resources.yaml`) — deploy SOPS-encrypted secrets via Kustomize + KSOPS

### Secrets Management (SOPS + KSOPS)
- Secrets are encrypted with SOPS using `age` encryption
- KSOPS (Kustomize exec plugin) decrypts at ArgoCD sync time
- ArgoCD repo-server has `--enable-alpha-plugins --enable-exec --enable-helm`
- **Never** put real secret values in `helm.yaml` — use empty strings or `existingSecret`

### Secret Patterns (ordered by preference)

1. **`existingSecret` (best)** — Chart supports referencing an external Secret:
   ```yaml
   secrets:
     existingSecret: <secret-name>
   ```
   Chart skips Secret creation; SOPS resources app provides the real Secret.
   No `ignoreDifferences` needed. Used by: `teslasync`, `vitasync`

2. **Empty string placeholders** — Chart creates Secret but with empty values:
   ```yaml
   spotify:
     clientId: ""
     clientSecret: ""
   ```
   Requires `ignoreDifferences` + `RespectIgnoreDifferences` + separate resources app.
   Used by: `echostats`, `outline`

3. ~~**`MANAGED_VIA_SOPS`**~~ — **Deprecated. Do not use.** Literal placeholder strings
   that get written into K8s Secrets and break pod auth.

## Conventions

### Adding a New App

1. Create `apps/<name>/helm.yaml` with the ArgoCD Application
2. If the chart needs secrets:
   - Prefer charts with `existingSecret` support
   - Create `apps/<name>/resources/` with SOPS-encrypted secrets
   - Create `apps/<name>/resources.yaml` for the resources ArgoCD Application
3. Labels: always set `tier: apps` on Application metadata
4. Namespace: set `namespace: argocd` on Application metadata
5. Destination namespace: should match the app name

### Helm Values

- Keep values in `valuesObject` inline (not external files)
- **Never** put secrets in helm values — use `existingSecret` or empty strings
- Use `ServerSideApply=true` in syncOptions
- Set `CreateNamespace=true` in syncOptions

### SOPS Secrets

- Encrypted with `age` (recipient key in each secret's SOPS metadata)
- Secret names must match what the chart expects (e.g., same as `existingSecret` value)
- Files must end in `.yaml` and contain `sops:` metadata block
- KSOPS generator file references the encrypted secret files

### ArgoCD Sync Policies

- `root.yaml` uses automated sync with prune and selfHeal
- Individual apps do NOT use automated sync (manual trigger)
- Exception: `vitasync` uses automated sync

### ignoreDifferences (when needed)

Only needed when Helm chart creates a Secret that SOPS also manages:
```yaml
ignoreDifferences:
  - group: ""
    kind: Secret
    name: <secret-name>
    jsonPointers:
      - /data
      - /stringData
syncPolicy:
  syncOptions:
    - RespectIgnoreDifferences=true
```

## Infrastructure

- **Cluster**: k3s (lightweight Kubernetes)
- **GitOps**: ArgoCD v3.3.5
- **Ingress**: Traefik (external class)
- **TLS**: cert-manager with production Let's Encrypt
- **DNS**: Cloudflare tunnel
- **Auth**: Authentik (OIDC/OAuth2)
- **Databases**: PostgreSQL 17, Redis 7, MongoDB (external)
- **MQTT**: EMQX
- **Monitoring**: Prometheus, Grafana, Loki, Gatus
- **Backups**: Velero

## Commands

```bash
# Apply root app (one-time setup)
kubectl apply -f apps/root.yaml

# Check all app status
kubectl get applications -n argocd

# Sync a specific app
kubectl -n argocd patch application <name> --type merge \
  -p '{"operation":{"initiatedBy":{"username":"manual"},"sync":{"syncStrategy":{"hook":{}}}}}'

# Check SOPS secret decryption
kubectl get secret <name> -n <namespace> -o jsonpath='{.data.<KEY>}' | base64 -d
```

## Do NOT

- Commit unencrypted secrets or credentials
- Use `MANAGED_VIA_SOPS` as placeholder values
- Modify `root.yaml` without understanding the exclude pattern
- Add files under `apps/` that aren't ArgoCD Application manifests
  (except inside `resources/` directories which are excluded from root discovery)
- Use `kubectl apply` for anything other than `root.yaml` — let ArgoCD manage everything
