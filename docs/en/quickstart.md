# Quickstart

Complete [installation](installation.md), then run these commands from the repository root. This example uses one GPU in Simple mode.

## Prepare the model

```bash
MODEL_DIR="$HOME/aegaeon-models/Qwen3-4B"
hf download Qwen/Qwen3-4B --local-dir "$MODEL_DIR"
cp docs/en/examples/simple.yaml aegaeon.yaml
sed -i "s|/srv/models/Qwen3-4B|$MODEL_DIR|" aegaeon.yaml
```

For another model or GPU, update the paths, memory budgets, and performance profiles in the [configuration](configuration.md).

## Start the service

```bash
export CUDA_VISIBLE_DEVICES=0
aegaeon start --config aegaeon.yaml --host 127.0.0.1 --port 8000 --ray-address local
```

Wait for `Application startup complete`.

## Send a request

In another terminal:

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"Explain KV Cache in one sentence."}],"temperature":0,"max_tokens":64}'
```

The response includes `choices[0].message.content`, `finish_reason`, and `usage`. See [Online Inference](serving.md) for client integration and streaming.
