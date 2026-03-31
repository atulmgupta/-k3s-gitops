# Netdata

## Overview
Netdata provides real-time, per-second monitoring of system and application metrics across all cluster nodes. Runs as a parent-child architecture — child DaemonSets on each node stream metrics to a central parent.

## Chart Details
| Field | Value |
|-------|-------|
| **Chart** | `netdata/netdata` (official) |
| **Repo** | `https://netdata.github.io/helmchart` |
| **Version Pin** | `3.*` |
| **Namespace** | `netdata` |

## Architecture
| Component | Type | Purpose |
|-----------|------|---------|
| **Parent** | Deployment (1 replica) | Central aggregator, dashboard UI, long-term storage |
| **Child** | DaemonSet (1 per node) | Per-node metrics collector, streams to parent |
| **k8sState** | Deployment | Kubernetes state monitoring (pods, deployments, etc.) |

## Endpoints
| Service | Type | Address | Port |
|---------|------|---------|------|
| Netdata UI | LoadBalancer | Assigned by MetalLB | 19999 |
| Internal | ClusterIP | `netdata.netdata.svc.cluster.local` | 19999 |

## Persistence
| PVC | StorageClass | Size | Purpose |
|-----|-------------|------|---------|
| Parent DB | longhorn | 10Gi | Metrics database (dbengine) |

## Configuration
- **Cloud claiming**: Disabled (standalone mode)
- **Tolerations**: Child pods tolerate all taints (runs on every node including cordoned ones)
- **Streaming**: Children auto-stream to parent via internal service discovery

## Upgrade
```bash
# Via ArgoCD (automatic with 3.* pin)
# Manual:
helm upgrade netdata netdata/netdata -n netdata --reuse-values
```

### Upgrade Notes
- The official Netdata Helm chart follows Netdata agent versioning.
- Child DaemonSet updates cause a brief gap in per-node metrics during rollout.
- Parent PVC retains historical data across upgrades.

## Backup & Restore

### Backup
```bash
# Netdata stores metrics in its dbengine format
kubectl cp netdata/<parent-pod>:/var/cache/netdata ./netdata-backup/
```

### Restore
```bash
kubectl scale deployment netdata -n netdata --replicas=0
kubectl cp ./netdata-backup/ netdata/<parent-pod>:/var/cache/netdata
kubectl scale deployment netdata -n netdata --replicas=1
```

## Troubleshooting
- **Child pod Pending/CrashLoop**: Check node resources. Netdata children need ~256Mi memory per node.
- **No data in parent UI**: Verify child→parent streaming. Check child logs: `kubectl logs -n netdata <child-pod> | grep stream`.
- **High CPU on child**: Reduce collection frequency in `netdata.conf` via Helm values or limit collectors.
- **Stale child pods after node removal**: Force delete Terminating pods from removed nodes: `kubectl delete pod -n netdata <pod> --force --grace-period=0`.
