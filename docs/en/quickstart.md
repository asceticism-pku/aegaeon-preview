# Quickstart: Send Your First Request

This example uses one Linux host, one available A100 40 GB GPU, Simple mode, TP=1, and CUDA Graphs disabled. Complete [installation](installation.md), then run the following commands from the repository root.

The repository includes the example configuration and matching profiles. For other models or GPUs, adjust the [configuration](configuration.md) and collect matching [profiles](profiling.md).

## 1. Prepare the model and configuration

Download the complete `Qwen/Qwen3-4B` checkpoint. If your network requires a proxy, follow the [model download guide](models.md#download-and-proxy-troubleshooting) first. For an existing complete local checkpoint, set `MODEL_DIR` to its absolute path and skip the download command.

```bash
MODEL_DIR="$HOME/aegaeon-models/Qwen3-4B"
hf download Qwen/Qwen3-4B --local-dir "$MODEL_DIR"
cp docs/en/examples/simple.yaml aegaeon.yaml
sed -i "s|/srv/models/Qwen3-4B|$MODEL_DIR|" aegaeon.yaml
```

## 2. Start the service

Select an available GPU; the example below uses GPU 0. `local` creates a local Ray runtime with the `node_0` resource.

```bash
export CUDA_VISIBLE_DEVICES=0
aegaeon start --config aegaeon.yaml \
  --host 127.0.0.1 --port 8000 --ray-address local
```

Startup initializes CPU caches and actors. Wait for `Application startup complete` in the terminal. Run one API instance with the default actor and cache names per node set. See [deployment](deployment.md) for an existing Ray cluster or multiple nodes.

## 3. Send a Chat request

In another terminal, run:

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"Explain KV Cache in one sentence."}],"temperature":0,"max_tokens":64}'
```

A successful response includes `choices[0].message.content`, `finish_reason`, and `usage`. The first request also includes GPU weight-loading time.

See the [API reference](api.md) for streaming and runtime queries, or [troubleshooting](troubleshooting.md) if startup or a request fails.
