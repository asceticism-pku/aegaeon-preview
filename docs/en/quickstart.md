# Quickstart: Send Your First Request

This example serves Qwen3-4B on one A100 PCIe 40GB GPU in Simple mode (TP=1, CUDA graphs off). Complete [installation](installation.md) using `--cuda cu129` for the tested route, activate the installed Conda environment, and run commands from the source checkout root.

## 1. Prepare the model and configuration

Download the complete checkpoint, or set `MODEL_DIR` to its existing absolute path and skip `hf download`:

```bash
MODEL_DIR="$HOME/aegaeon-models/Qwen3-4B"
hf download Qwen/Qwen3-4B --local-dir "$MODEL_DIR"
cp docs/en/examples/simple.yaml aegaeon.yaml
sed -i "s|/srv/models/Qwen3-4B|$MODEL_DIR|" aegaeon.yaml
```

The example reserves 20 GiB for model weights and 8 GiB for CPU KV cache in host RAM and `/dev/shm`; allow extra capacity for Ray and other processes. Matching profiles are bundled. For another model or GPU, adjust the [configuration](configuration.md) and collect matching [profiles](profiling.md). Download issues: [model preparation](models.md#download-and-proxy-troubleshooting).

## 2. Start the service

Select an available GPU; `--ray-address local` starts Ray automatically:

```bash
export CUDA_HOME="$CONDA_PREFIX"
CUDA_VISIBLE_DEVICES=0 aegaeon start --config aegaeon.yaml \
  --host 127.0.0.1 --port 8000 --ray-address local
```

Wait for `Application startup complete`. Run one API instance with the default actor/cache names per node set; other deployments are covered in [deployment](deployment.md).

## 3. Send a Chat request

In another terminal at the repository root:

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  --data-binary @docs/en/examples/chat-request.json
```

A successful response includes `choices[0].message.content` and token `usage`. The first request also loads weights onto the GPU. See the [API reference](api.md) for streaming or [troubleshooting](troubleshooting.md) for failures.
