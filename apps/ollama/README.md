# ollama

Self-hosted LLM inference (Ollama) + ChatGPT-style frontend (Open WebUI),
running on the cluster's GPU (`helium`, RTX 3050 OEM 8GB).

## Components

| Resource | Purpose |
|----------|---------|
| `ollama-deployment.yaml` | Ollama server, pinned to `helium` with `nvidia.com/gpu: 1` and `runtimeClassName: nvidia` |
| `ollama-service.yaml` | ClusterIP service + 100Gi Longhorn PVC for `/root/.ollama` |
| `webui-deployment.yaml` | Open WebUI, runs anywhere, no GPU |
| `webui-service.yaml` | ClusterIP service + 10Gi Longhorn PVC for chat history |
| `ingressroute.yaml` | Two Traefik IngressRoutes, both behind Authentik |
| `model-pull-job.yaml` | ArgoCD PostSync hook — `ollama pull` each model after sync |

## URLs

| Host | Behind | Use |
|------|--------|-----|
| `https://ollama.cyphers.app` | Authentik | Raw Ollama API (port 11434). For IDE plugins (Continue, Cursor), curl, custom scripts. |
| `https://chat.cyphers.app` | Authentik | Open WebUI — ChatGPT-style chat |

## Pre-pulled models

The PostSync Job pulls `llama3.1:8b` (~5GB) on every sync. `ollama pull` is
idempotent so re-syncing is cheap. To add/remove models, edit
`model-pull-job.yaml` and re-sync.

8GB VRAM is comfortable for:

- `llama3.1:8b`, `qwen2.5:7b`, `mistral:7b` — fit easily
- `gemma2:9b` — fits with default quant
- `phi3.5:3.8b` — tiny, snappy
- Quantized 13B models (Q4) — tight, may swap to system RAM

`OLLAMA_MAX_LOADED_MODELS=1` keeps only one resident; switching models takes
a few seconds to reload from disk.

## Tuning knobs (in `ollama-deployment.yaml`)

| Env | Default here | What it does |
|-----|--------------|--------------|
| `OLLAMA_HOST` | `0.0.0.0:11434` | Listen on all interfaces |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | VRAM-friendly for 8GB cards |
| `OLLAMA_KEEP_ALIVE` | `30m` | How long model stays warm in VRAM after last request |
| `OLLAMA_ORIGINS` | `*` | Permissive CORS for in-cluster + browser clients |

## Verify after first sync

```bash
# Pod scheduled on helium, GPU mounted
kubectl -n ollama get pods -o wide

# Ollama can see the GPU
kubectl -n ollama exec deploy/ollama -- nvidia-smi

# Model pull job ran successfully
kubectl -n ollama logs job/ollama-model-pull

# Models present in the API
kubectl -n ollama exec deploy/ollama -- ollama list

# Test inference from inside the cluster
kubectl -n ollama exec deploy/ollama -- \
  ollama run llama3.1:8b "hello in three words"
```

## Use it

**Open WebUI**: visit `https://chat.cyphers.app`, log in via Authentik.

**API from anywhere** (with Authentik cookie or basic-auth header):

```bash
curl https://ollama.cyphers.app/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model":"llama3.1:8b","prompt":"why is the sky blue","stream":false}'
```

**IDE plugin (Continue.dev, Cursor, etc.)**: point at `https://ollama.cyphers.app`.
Note: IDE plugins typically don't handle Authentik's OIDC flow — for that use
case you may want to expose Ollama on a separate cluster-internal hostname
or add a token-based middleware.

## Resource limits

- Ollama: requests 500m CPU / 2Gi RAM / 1 GPU; limits 8 CPU / 12Gi RAM / 1 GPU
- WebUI: requests 100m / 256Mi; limits 2 CPU / 2Gi

## Troubleshooting

**Pod stuck Pending with `0/N nodes available: insufficient nvidia.com/gpu`**
→ The `nvidia-device-plugin` app hasn't synced or helium has no advertised
GPU. Run `kubectl describe node helium | grep nvidia.com/gpu` — expect `1`.

**Ollama crashloops with `CUDA error`**
→ Another pod is using the GPU. `kubectl -n ollama exec ... nvidia-smi`
to inspect. With `MAX_LOADED_MODELS=1` and a single GPU pod this shouldn't
happen, but a stuck old pod could be holding the device.

**Model pull job times out**
→ First pull of a fresh model can take minutes. Bump the loop in
`model-pull-job.yaml` or pull manually:
`kubectl -n ollama exec deploy/ollama -- ollama pull <model>`.
