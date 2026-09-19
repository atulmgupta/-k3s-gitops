# Grafana

## Overview
Grafana is the primary dashboarding and visualization platform for the cluster. It connects to Prometheus, Loki, and InfluxDB for metrics, logs, and time-series data.

## Chart Details
| Field | Value |
|-------|-------|
| **Chart** | `bitnami/grafana` |
| **Repo** | `https://charts.bitnami.com/bitnami` |
| **Version Pin** | `12.*` (auto-updates minor/patch) |
| **App** | Grafana OSS |
| **Namespace** | `grafana` |

## Endpoints
| Service | Type | Address |
|---------|------|---------|
| Grafana UI | ClusterIP | `grafana.grafana.svc.cluster.local:3000` |
| External URL | Traefik/Cloudflare | `https://grafana.cyphers.app` |

## Authentication
- **Admin**: Credentials stored in `grafana-admin-secret` (key: `password`)
- **SSO**: Authentik Generic OAuth via `http://authentik-server.authentik.svc.cluster.local`
- **SMTP**: SendGrid for email notifications, credentials in `grafana-smtp-secret`

## Persistence
| PVC | StorageClass | Size | Mount Path |
|-----|-------------|------|------------|
| `grafana-pvc` | longhorn | 10Gi | `/opt/bitnami/grafana/data` |

The SQLite database (`grafana.db`) stores all dashboards, datasources, users, and alert rules.

## Dependencies
- **Prometheus** (`prometheus` namespace) — primary metrics datasource
- **Loki** (`monitoring` namespace) — log aggregation datasource
- **Authentik** (`authentik` namespace) — SSO/OAuth provider
- **Longhorn** — persistent volume backend

## TeslaSync Dashboards

The 77 dashboards from TeslaSync's `grafana/dashboards/` are provisioned by
`grafana-resources` as labeled ConfigMaps:

| Source directory | GitOps directory under `resources/dashboards/` | Grafana folder | Count |
|------------------|-----------------------------------------------|----------------|-------|
| `system` | `teslasync` | TeslaSync | 59 |
| `infra` | `teslasync-infra` | TeslaSync Infra | 17 |
| `science` | `teslasync-science` | TeslaSync Science | 1 |

When updating, preserve dashboard UIDs and wrap each source JSON in a ConfigMap
with the `grafana_dashboard: "1"` label and the corresponding `grafana_folder`
annotation. Map source datasource UIDs `DS_TESLASYNC_PROMETHEUS` and
`DS_TESLASYNC_TEMPO` to this cluster's `prometheus` and `tempo`; retain
`DS_TESLASYNC_POSTGRESQL`. Register every ConfigMap in
`resources/kustomization.yaml`. Keep Helm-provisioned dashboards and the separate
`teslasync-tracing` dashboard out of this import to avoid duplicate UIDs.

Preserve the GitOps Previous/Next links and TeslaSync dashboard dropdown when
refreshing source JSON. System dashboards and Science Lab Inputs form one
circular sequence; Infra dashboards form a separate sequence. New dashboards
must be linked in both directions from their neighbors, keeping the selected
time range and variables. Retain any additional "Open in app" links.

Push changes to `main` to deploy through the automated `grafana-resources` ArgoCD
sync. The Grafana sidecar loads the ConfigMaps without a Grafana restart.

## Secrets Required
| Secret Name | Namespace | Keys | Purpose |
|-------------|-----------|------|---------|
| `grafana-admin-secret` | grafana | `password` | Admin login password |
| `grafana-smtp-secret` | grafana | `GF_SMTP_USER`, `GF_SMTP_PASSWORD` | SendGrid SMTP credentials |

## Upgrade
```bash
# Via ArgoCD (automatic with 12.* pin)
# Manual override:
helm upgrade grafana bitnami/grafana -n grafana --reuse-values

# To pin a specific version:
helm upgrade grafana bitnami/grafana -n grafana --reuse-values --version 12.1.8
```

### Upgrade Notes
- Bitnami major chart versions may introduce breaking changes — check release notes before upgrading across majors.
- The `grafana.kind` field must be set (defaults to `Deployment`). If upgrading from chart <12.x, you may need `--set grafana.kind=Deployment`.
- Image tag is managed by the chart. Do not pin `image.tag` unless intentionally freezing the version.

## Backup & Restore

### Backup
```bash
# Export the SQLite database (contains all dashboards, datasources, users)
kubectl cp grafana/<pod-name>:/opt/bitnami/grafana/data/grafana.db ./grafana-backup.db

# Export individual dashboards via API
curl -s -H "Authorization: Bearer <api-key>" https://grafana.cyphers.app/api/search | jq '.[].uid' | while read uid; do
  curl -s -H "Authorization: Bearer <api-key>" "https://grafana.cyphers.app/api/dashboards/uid/$uid" > "dashboard-$uid.json"
done
```

### Restore
```bash
# Scale down first
kubectl scale deployment grafana -n grafana --replicas=0

# Copy backup into PVC
kubectl cp ./grafana-backup.db grafana/<pod-name>:/opt/bitnami/grafana/data/grafana.db

# Scale back up
kubectl scale deployment grafana -n grafana --replicas=1
```

## Troubleshooting
- **Pod won't start (Multi-Attach error)**: RWO volume stuck on another node. Scale to 0, wait, scale to 1.
- **Dashboards missing after restart**: Verify PVC is mounted at `/opt/bitnami/grafana/data` — run `kubectl exec -n grafana <pod> -- df -h /opt/bitnami/grafana/data/`.
- **OAuth login fails**: Check Authentik server is running and OAuth endpoints are reachable from the Grafana pod.
- **SMTP not sending**: Verify `grafana-smtp-secret` exists and contains valid SendGrid credentials.
