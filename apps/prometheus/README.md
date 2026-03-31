# Prometheus (kube-prometheus-stack)

## Overview
Full-stack Kubernetes monitoring using the community kube-prometheus-stack chart. Includes Prometheus server, Alertmanager, node-exporter, kube-state-metrics, and the Prometheus Operator for CRD-based configuration.

Migrated from Bitnami `kube-prometheus` to community `kube-prometheus-stack` on 2026-03-31.

## Chart Details
| Field | Value |
|-------|-------|
| **Chart** | `prometheus-community/kube-prometheus-stack` |
| **Repo** | `https://prometheus-community.github.io/helm-charts` |
| **Version Pin** | `82.*` (auto-updates minor/patch) |
| **Namespace** | `prometheus` |

## Endpoints
| Service | Type | Address | Port |
|---------|------|---------|------|
| Prometheus UI | LoadBalancer | `192.168.68.220` | 9090 |
| Alertmanager | ClusterIP | `prometheus-kube-prometheus-alertmanager.prometheus.svc` | 9093 |
| Node Exporter | ClusterIP | Per-node DaemonSet | 9100 |
| Kube State Metrics | ClusterIP | `prometheus-kube-state-metrics.prometheus.svc` | 8080 |

## Components
| Component | Type | Replicas | Purpose |
|-----------|------|----------|---------|
| Prometheus | StatefulSet | 1 | Metrics collection, storage, and querying |
| Alertmanager | StatefulSet | 1 | Alert routing, deduplication, and notification |
| Prometheus Operator | Deployment | 1 | Manages Prometheus/Alertmanager CRDs |
| Kube State Metrics | Deployment | 1 | Exports Kubernetes object state as metrics |
| Node Exporter | DaemonSet | 1 per node | Exports host-level hardware/OS metrics |

## Persistence
| PVC | StorageClass | Size | Purpose |
|-----|-------------|------|---------|
| Prometheus TSDB | longhorn | 250Gi | Time-series metrics storage |
| Alertmanager | longhorn | 8Gi | Alert state and silences |

## ServiceMonitor Discovery
Prometheus is configured to discover **all** ServiceMonitors across all namespaces regardless of labels:
```yaml
serviceMonitorSelectorNilUsesHelmValues: false
podMonitorSelectorNilUsesHelmValues: false
ruleSelectorNilUsesHelmValues: false
```

### Active Custom ServiceMonitors
| Namespace | ServiceMonitor | Target |
|-----------|---------------|--------|
| `prometheus` | `authentik-server` | Authentik metrics |
| `prometheus` | `influxdb` | InfluxDB metrics |
| `prometheus` | `nextcloud` | Nextcloud metrics |
| `prometheus` | `postgres-v17-postgresql` | PostgreSQL v17 metrics |
| `prometheus` | `traefik-metrics` | Traefik ingress metrics |
| `authentik` | `ak-outpost-traefik` | Authentik outpost metrics |
| `longhorn` | `longhorn-prometheus-servicemonitor` | Longhorn storage metrics |
| `mqtt` | `mqtt-emqx` | EMQX MQTT broker metrics |

## Key Configuration
- **Retention**: 90 days
- **Grafana**: Disabled (separate Bitnami Grafana deployment in `grafana` namespace)
- **Storage**: Longhorn-backed PVCs

## Dependencies
- **Longhorn** — persistent volume backend
- **Grafana** (`grafana` namespace) — visualization (uses Prometheus as datasource)
- **MetalLB** — LoadBalancer IP assignment

## Adding New Scrape Targets

Prometheus discovers targets via **ServiceMonitor** and **PodMonitor** CRDs. No need to edit Prometheus config directly — this cluster is configured to discover all ServiceMonitors across all namespaces regardless of labels.

### Option 1: ServiceMonitor (for Services)
Create a ServiceMonitor in the app's namespace or in `prometheus` namespace:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: my-app          # can be any namespace — Prometheus discovers all
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app            # must match the Service's labels
  endpoints:
    - port: metrics          # must match the Service's port name
      interval: 30s          # scrape interval (default: 30s)
      path: /metrics         # metrics endpoint path (default: /metrics)
  namespaceSelector:
    matchNames:
      - my-app               # namespace where the Service lives
```

### Option 2: PodMonitor (for Pods without a Service)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app
  namespace: my-app
spec:
  selector:
    matchLabels:
      app: my-app            # must match the Pod's labels
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### Option 3: PrometheusRule (for alerting rules)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: prometheus
spec:
  groups:
    - name: my-app
      rules:
        - alert: MyAppDown
          expr: up{job="my-app"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "{{ $labels.instance }} is down"
```

### Verifying New Targets
After applying a ServiceMonitor/PodMonitor, verify it's being scraped:
```bash
# Check if target appears (may take up to 60s)
curl -s http://192.168.68.220:9090/api/v1/targets | grep "my-app"

# Or check the Prometheus UI at http://192.168.68.220:9090/targets
```

### Common Pitfalls
- **Target not appearing**: Ensure the ServiceMonitor's `selector.matchLabels` matches the **Service** labels (not Pod labels). Check with `kubectl get svc -n <ns> --show-labels`.
- **Target shows but is down**: The `/metrics` endpoint must be accessible from the Prometheus pod. Check network policies.
- **Duplicate targets**: Avoid creating ServiceMonitors in both the app namespace and `prometheus` namespace for the same service.
- **Wrong port name**: The `port` field in the endpoint must match the **port name** defined in the Service spec, not the port number.

## Upgrade
```bash
# Via ArgoCD (automatic with version pin)
# Manual:
helm upgrade prometheus prometheus-community/kube-prometheus-stack -n prometheus --reuse-values

# Check CRD updates (community chart doesn't auto-update CRDs)
kubectl apply --server-side -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_servicemonitors.yaml
```

### Upgrade Notes
- **CRDs are NOT automatically upgraded** by Helm. When upgrading across major versions, apply CRD manifests manually first.
- The community chart bundles Grafana by default — always set `grafana.enabled=false` to avoid conflicts.
- Check for deprecated APIs in PrometheusRule and ServiceMonitor specs when upgrading the operator.

## Backup & Restore

### Backup
```bash
# Prometheus TSDB snapshot via API
curl -X POST http://192.168.68.220:9090/api/v1/admin/tsdb/snapshot
# Snapshot saved to: /prometheus/snapshots/<snapshot-name>

# Copy snapshot out
kubectl cp prometheus/prometheus-prometheus-kube-prometheus-prometheus-0:/prometheus/snapshots/ ./prometheus-snapshots/ -c prometheus
```

### Restore
```bash
# Scale down
kubectl scale statefulset prometheus-prometheus-kube-prometheus-prometheus -n prometheus --replicas=0

# Copy snapshot data into PVC
kubectl cp ./prometheus-snapshots/<snapshot>/ prometheus/prometheus-prometheus-kube-prometheus-prometheus-0:/prometheus/data/ -c prometheus

# Scale back up
kubectl scale statefulset prometheus-prometheus-kube-prometheus-prometheus -n prometheus --replicas=1
```

## Troubleshooting
- **Targets not discovered**: Check ServiceMonitor labels. With `selectorNilUsesHelmValues: false`, all ServiceMonitors should be picked up. Verify with `curl http://192.168.68.220:9090/api/v1/targets`.
- **High memory usage**: Prometheus memory scales with active time series. Check cardinality: `curl http://192.168.68.220:9090/api/v1/status/tsdb`.
- **Stale targets showing down**: Look for orphaned EndpointSlices or Services from previous installs: `kubectl get endpointslices -n kube-system | grep prometheus`.
- **CRD version mismatch**: If ServiceMonitors fail to apply, update CRDs manually (see Upgrade section).
- **Node exporter pending**: Likely targeting a cordoned/removed node. Check with `kubectl get pods -n prometheus -o wide`.

## Alertmanager Configuration

Alertmanager routes alerts to notification channels. Configure receivers in the Helm values or via `alertmanager-config.yaml`.

### Receiver Configuration

Create an alertmanager config and apply via Helm. The example below has Discord active and all others commented out for easy enabling later.

```yaml
# alertmanager-config.yaml (pass via --set-file or inline in Helm values)
alertmanager:
  config:
    global:
      resolve_timeout: 5m

    route:
      receiver: discord
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      routes:
        # Route critical alerts to all channels
        # - match:
        #     severity: critical
        #   receiver: all-channels

        # Route specific namespaces
        # - match:
        #     namespace: teslasync
        #   receiver: discord

    receivers:
      # ── Discord (ACTIVE) ────────────────────────────────
      - name: discord
        webhook_configs:
          - url: 'https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN'
            send_resolved: true

      # ── Slack ───────────────────────────────────────────
      # - name: slack
      #   slack_configs:
      #     - channel: '#k8s-alerts'
      #       api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
      #       send_resolved: true
      #       title: '{{ .GroupLabels.alertname }}'
      #       text: >-
      #         {{ range .Alerts }}
      #         *Alert:* {{ .Annotations.summary }}
      #         *Severity:* {{ .Labels.severity }}
      #         *Namespace:* {{ .Labels.namespace }}
      #         {{ end }}

      # ── Email (SendGrid) ───────────────────────────────
      # - name: email
      #   email_configs:
      #     - to: 'alerts@cyphers.app'
      #       from: 'alertmanager@cyphers.app'
      #       smarthost: 'smtp.sendgrid.net:587'
      #       auth_username: 'apikey'
      #       auth_password: '<sendgrid-api-key>'
      #       send_resolved: true
      #       headers:
      #         Subject: '[K8s Alert] {{ .GroupLabels.alertname }}'

      # ── PagerDuty ──────────────────────────────────────
      # - name: pagerduty
      #   pagerduty_configs:
      #     - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
      #       send_resolved: true

      # ── Microsoft Teams (via webhook) ──────────────────
      # - name: teams
      #   webhook_configs:
      #     - url: 'https://outlook.office.com/webhook/YOUR_TEAMS_WEBHOOK'
      #       send_resolved: true

      # ── Multi-channel (for critical alerts) ────────────
      # - name: all-channels
      #   webhook_configs:
      #     - url: 'https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN'
      #       send_resolved: true
      #   email_configs:
      #     - to: 'alerts@cyphers.app'
      #       from: 'alertmanager@cyphers.app'
      #       smarthost: 'smtp.sendgrid.net:587'
      #       auth_username: 'apikey'
      #       auth_password: '<sendgrid-api-key>'
      #       send_resolved: true

    # Inhibit rules — suppress lower-severity alerts when critical fires
    inhibit_rules:
      - source_matchers: [severity="critical"]
        target_matchers: [severity="warning"]
        equal: [alertname, namespace]
```

### Applying the Config
```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  -n prometheus --reuse-values \
  --set alertmanager.config.global.resolve_timeout=5m \
  --set alertmanager.config.route.receiver=discord \
  --set 'alertmanager.config.route.group_by[0]=alertname' \
  --set 'alertmanager.config.route.group_by[1]=namespace' \
  --set alertmanager.config.route.group_wait=30s \
  --set alertmanager.config.route.group_interval=5m \
  --set alertmanager.config.route.repeat_interval=4h \
  --set 'alertmanager.config.receivers[0].name=discord' \
  --set 'alertmanager.config.receivers[0].webhook_configs[0].url=https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN' \
  --set 'alertmanager.config.receivers[0].webhook_configs[0].send_resolved=true'
```

### Verifying
```bash
# Check Alertmanager config was applied
kubectl get secret -n prometheus alertmanager-prometheus-kube-prometheus-alertmanager-generated \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# Check Alertmanager UI
# http://192.168.68.220:9093 (if exposed via LoadBalancer)

# Send a test alert
curl -X POST http://192.168.68.220:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TestAlert","severity":"warning"},"annotations":{"summary":"This is a test alert"}}]'
```

## Migration History
- **2024**: Initially deployed with Bitnami `kube-prometheus` chart
- **2026-03-31**: Migrated to community `kube-prometheus-stack`. Cleaned up stale Bitnami services (`prometheus-adv-kube-promet-*`) and endpointslices from kube-system namespace.
