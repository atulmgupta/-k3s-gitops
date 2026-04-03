# Nexus Terminal — ArgoCD Application

## Structure

```
apps/nexus-terminal/
├── helm.yaml           # ArgoCD Application — deploys Helm chart from nexus-terminal repo
├── resources.yaml      # ArgoCD Application — deploys Kustomize resources (IngressRoute, secrets)
└── resources/
    ├── kustomization.yaml
    └── ingressroute.yaml   # Traefik IngressRoute for nexus.cyphers.app
```

## TODO before deploying

1. Fill in `helm.yaml` values:
   - `config.jwtSecret` — generate with `openssl rand -base64 32`
   - `postgresql.external.*` — host, port, username, password, database
   - `mongodb.external.url` — MongoDB Atlas connection string

2. Optionally add SOPS-encrypted secrets for DB credentials

## URLs

| Service | URL |
|---------|-----|
| Web | https://nexus.cyphers.app |
| API | https://nexus.cyphers.app/api |
| Docs | https://nexus-docs.cyphers.app |
