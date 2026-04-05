# Authentik — ArgoCD Application

Identity Provider for all `*.cyphers.app` services via Traefik ForwardAuth.

## Architecture

```
helm.yaml           → ArgoCD Application (Helm chart: authentik/authentik)
resources.yaml       → ArgoCD Application (Kustomize: secrets + middleware)
resources/
├── kustomization.yaml
├── ksops-generator.yaml
├── secret.yaml          ← SOPS-encrypted credentials
└── middleware.yaml      ← Traefik ForwardAuth middleware
```

## Migration from Helm CLI

This app was migrated from a direct `helm install` to ArgoCD-managed GitOps.

### Steps to complete migration:

1. **Encrypt the secret** before committing:
   ```bash
   sops --encrypt --in-place apps/authentik/resources/secret.yaml
   ```

2. **Push to git** — ArgoCD root app auto-discovers the new Application

3. **Review in ArgoCD UI** — both Applications start with manual sync:
   - `authentik` (Helm) — review the diff, should match existing resources
   - `authentik-resources` (Kustomize) — creates Secret + Middleware

4. **Sync resources first** — sync `authentik-resources` so the Secret exists

5. **Sync helm app** — sync `authentik` to upgrade and adopt resources

6. **Remove old Helm release** after successful sync:
   ```bash
   kubectl delete secret -n authentik -l owner=helm,name=authentik
   ```

7. **Enable automated sync** — edit helm.yaml and resources.yaml:
   ```yaml
   syncPolicy:
     automated: { prune: true, selfHeal: true }
     syncOptions: [CreateNamespace=true]
   ```

## Protected Services

All 16+ IngressRoutes reference `authentik-auth` middleware from this namespace.
