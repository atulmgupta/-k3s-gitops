# nvidia-device-plugin

Exposes `nvidia.com/gpu` as a schedulable resource on the cluster's GPU node
(`helium` — NVIDIA RTX 3050 OEM, 8 GB VRAM).

## Components

| File | Purpose |
|------|---------|
| `helm.yaml` | NVIDIA k8s-device-plugin Helm chart, pinned to `helium` |
| `resources.yaml` | ArgoCD app for the raw manifests below |
| `resources/runtimeclass.yaml` | Cluster `RuntimeClass nvidia` that GPU pods reference |

## Node prerequisites (one-time, on `helium`)

```bash
# 1. NVIDIA driver (open kernel modules, server variant)
sudo apt install -y nvidia-driver-595-server-open
sudo reboot

# 2. NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit

# 3. Restart k3s-agent so containerd auto-registers the `nvidia` runtime
sudo systemctl restart k3s-agent

# 4. Verify
sudo grep -A3 'nvidia' /var/lib/rancher/k3s/agent/etc/containerd/config.toml
nvidia-smi
```

## How GPU pods opt in

```yaml
spec:
  runtimeClassName: nvidia
  nodeSelector:
    kubernetes.io/hostname: helium
  containers:
    - name: my-gpu-app
      resources:
        limits:
          nvidia.com/gpu: 1
```

## Verify after sync

```bash
# DaemonSet pod is running on helium
kubectl get pods -n nvidia-device-plugin -o wide

# Node now advertises nvidia.com/gpu
kubectl describe node helium | grep -A2 -E 'Capacity|Allocatable' | grep nvidia

# RuntimeClass exists
kubectl get runtimeclass nvidia
```

Expected: `nvidia.com/gpu: 1` under both Capacity and Allocatable on helium.

## Why no NFD / GFD?

The chart can deploy Node Feature Discovery + GPU Feature Discovery for
auto-labeling. Skipped here — single GPU node, hostname-pin is simpler.
Re-enable `gfd.enabled: true` if more GPU nodes are added later.
