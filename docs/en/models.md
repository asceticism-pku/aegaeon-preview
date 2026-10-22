# Model Preparation

Use a complete local checkpoint containing the model configuration, tokenizer, chat template, and all safetensors weight shards. Multi-node deployments use the same absolute path on every node.

## Download

```bash
hf download Qwen/Qwen3-4B --local-dir /srv/models/Qwen3-4B
```

The downloader uses standard `HTTP_PROXY` and `HTTPS_PROXY` settings. Configure proxy access before downloading large checkpoints.

## Register a model

```yaml
models:
- name: Qwen/Qwen3-4B
  path: /srv/models/Qwen3-4B
  profile: Qwen/Qwen3-4B
  max_model_len: 4096
```

- `name` identifies the model in client requests.
- `path` locates the checkpoint.
- `profile` selects performance data; see [Profiling](profiling.md).
- `max_model_len` includes both prompt and generated tokens.

Usually omit `params` so Aegaeon reads weight metadata. Explicit values use units of `2^30` parameters.

Chat uses the tokenizer's chat template and the checkpoint's EOS configuration. Set `tool_parser` to select a vLLM function-call parser when needed; see [Online Inference](serving.md).

The [configuration reference](configuration.md) lists all model fields. Models can also be [deployed dynamically](model-management.md).
