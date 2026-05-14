# dcgm-exporter

NVIDIA's [DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter) — exposes GPU
metrics to Prometheus.

## Scope

- DaemonSet pinned to **helium** only (the cluster's sole GPU node).
- ServiceMonitor auto-scraped by the cluster Prometheus (no labels needed —
  Prometheus has an empty `serviceMonitorSelector`).
- Companion Grafana dashboard ships from
  [`apps/grafana/resources/dashboards/dcgm/dcgm-gpu.yaml`](../grafana/resources/dashboards/dcgm/dcgm-gpu.yaml)
  → folder **Infrastructure** → "NVIDIA GPU (DCGM)".

## Key metrics

| Metric                              | Meaning                              |
| ----------------------------------- | ------------------------------------ |
| `DCGM_FI_DEV_GPU_UTIL`              | GPU utilization (%)                  |
| `DCGM_FI_DEV_FB_USED` / `FB_FREE`   | VRAM used / free (MiB)               |
| `DCGM_FI_DEV_GPU_TEMP`              | GPU temperature (°C)                 |
| `DCGM_FI_DEV_POWER_USAGE`           | Power draw (W)                       |
| `DCGM_FI_DEV_SM_CLOCK`              | Streaming-multiprocessor clock (MHz) |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`   | Tensor-core utilization (fraction)   |

## Why `runtimeClassName: nvidia`

Same reason as `apps/nvidia-device-plugin`: on k3s the nvidia container
runtime is registered as a **named** (non-default) runtime. Without
`runtimeClassName: nvidia` the exporter container starts without
`libdcgm.so` / `libnvidia-ml.so` from the host driver and crashes.

## What it doesn't tell you

- Per-prompt latency from Ollama — Ollama has no `/metrics` endpoint.
- Number of chat messages in Open WebUI — only stored in its SQLite DB.
- Container/pod CPU/RAM — that comes from cadvisor (already scraped by
  kube-prometheus-stack); use the standard kubernetes / node-exporter
  dashboards.
