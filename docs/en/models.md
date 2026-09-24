# Model Preparation and Registration

## Local checkpoints

Use the same absolute checkpoint path on every node and configure `path` explicitly. Include the full configuration, tokenizer files, required chat template and generation configuration, and every safetensors or other loader-supported weight shard.

The model name identifies client requests, the profile alias identifies performance data, and path locates the actual weights:

```yaml
models:
  - name: Qwen/Qwen3-4B
    path: /srv/models/Qwen3-4B
    profile: qwen3_4b
    max_model_len: 4096
```

## Download sources

`model_downloader.py` uses ModelScope by default; Hugging Face is also available:

```bash
export AEGAEON_DEFAULT_MODEL_REPO=modelscope
# Or
export AEGAEON_DEFAULT_MODEL_REPO=huggingface
```

Download implementations cover these two values. The code checks `~/.cache/modelscope/hub/<name>` or `~/.cache/huggingface/hub/<name>` before calling the SDK's snapshot_download. Use an explicit path for offline and multi-node deployments to avoid depending on SDK cache layouts.

The service forwards several MODEL_ROOT/PROXY variables to actors, while the downloader uses settings recognized by its SDK. Configure `HTTP_PROXY` and `HTTPS_PROXY` for network access, or set a local `path` for offline deployment.

## Model configuration

ModelConfig uses bf16, estimates 2 bytes per parameter, and sets `trust_remote_code=True`, `model_impl='auto'`, `quantization=None`, `revision=None`, and vLLM `enforce_eager=True`. YAML has no dtype, quantization, or revision options.

When `max_model_len` is omitted, the service takes the smallest known candidate context length from the model configuration; an empty candidate set raises an exception. The override sets a service-side limit, while RoPE and trained context capability remain properties of the model and weights.

## EOS and chat templates

EOS IDs are merged from hf_config, text_config, and generation_config to support models with multiple end tokens. Use short questions to check stopping, especially with additional markers such as `<end_of_turn>`.

Chat uses the tokenizer's apply_chat_template. For normalized model names containing qwen3, the API always passes `enable_thinking=False` to produce direct answers. The current request schema omits general chat_template_kwargs and enable_thinking fields.

## Profiles and parameter estimates

Use a stable, safe relative profile alias. Without an explicit alias, `Qwen/Qwen3.5-9B` can normalize to `qwen3_5_9b`. Absolute paths and `..` path segments are rejected.

Parameter counts are computed from weight metadata. Manual params is considered only if that fails; see the [2^30 conversion](configuration.md). Automatic CPU weight cache sizing estimates parameters × 2 × 1.2. This is a capacity approximation rather than actual GPU peak memory.

## Accept a new text model

1. Generate short text with one model, TP=1, and graphs disabled.
2. Verify the tokenizer, chat template, EOS, stop parameters, and context limits.
3. Collect target-device profiles and check scheduling estimates.
4. Alternate two models serially and compare token outputs and cache release before and after switching.
5. Check concurrency, P/D, replicas, sliding windows, MLA, or Mamba state used by the target deployment.
6. For Foundry, complete SAVE/LOAD, model-switching, and token-ID comparisons.

The model enters that deployment's usable set after all six checks pass. The current support scope covers text models only; multimodal models are excluded from this procedure.
