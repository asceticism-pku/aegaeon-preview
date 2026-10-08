# Model Preparation and Registration

## Local checkpoints

Use the same absolute checkpoint path on every node and configure `path` explicitly. Include the full configuration, tokenizer files, required chat template and generation configuration, and all safetensors weight shards; the main service's QuickCache/QuickLoader path reads safetensors only.

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

## Download and proxy troubleshooting

A complete local checkpoint needs no download. If a Hugging Face download fails, check connectivity first; if a proxy is needed, configure a working address in the shell running the download:

```bash
export HTTPS_PROXY=http://127.0.0.1:7890
export HTTP_PROXY="$HTTPS_PROXY"
curl -fsS --max-time 20 https://huggingface.co/api/models/Qwen/Qwen3-4B >/dev/null
```

Replace the example proxy address for your environment, or omit the proxy settings for direct access. If Xet weight downloads time out through the proxy, set `export HF_HUB_DISABLE_XET=1` before starting the download process to disable the hf-xet path; see the [upstream environment reference](https://huggingface.co/docs/huggingface_hub/en/package_reference/environment_variables#hfhubdisablexet). After a network interruption, rerun the same `hf download` command with the same `--local-dir` to complete the download. Keeping `.cache/huggingface` metadata allows reuse of complete, up-to-date files; see the [upstream download guide](https://huggingface.co/docs/huggingface_hub/en/guides/cli#download-to-a-local-folder). Finish downloading before starting the service.

## Model configuration

ModelConfig uses bf16, estimates 2 bytes per parameter, and sets `trust_remote_code=True`, `model_impl='auto'`, `quantization=None`, `revision=None`, and vLLM `enforce_eager=True`. YAML has no dtype, quantization, or revision options.

When `max_model_len` is omitted, the service takes the smallest known candidate context length from the model configuration. If no candidate is present, it reads the tokenizer's `model_max_length`. An infinite length or placeholder of at least `10**30` falls back to 2048. The override sets a service-side limit, while RoPE and trained context capability remain properties of the model and weights.

## EOS and chat templates

EOS IDs are merged from hf_config, text_config, and generation_config to support models with multiple end tokens. Use short questions to check stopping, especially with additional markers such as `<end_of_turn>`.

Chat uses the tokenizer's apply_chat_template. For normalized model names containing qwen3, the API always passes `enable_thinking=False` to produce direct answers. The current request schema omits general chat_template_kwargs and enable_thinking fields.

## Profiles and parameter estimates

Use a stable, safe relative profile alias. Without an explicit alias, `Qwen/Qwen3.5-9B` can normalize to `qwen3_5_9b`. Absolute paths and `..` path segments are rejected.

Explicit `params` takes precedence over automatic counting and uses the [2^30 conversion](configuration.md) for estimates. Otherwise, counting sums tensor elements from the shapes in all `.safetensors` files at the checkpoint root. Supply `params` explicitly if those files are absent or reading fails. Automatic CPU weight cache sizing estimates parameters × 2 × 1.2. Actual GPU peak memory also includes loading and runtime allocations.

## Accept a new text model

1. Generate short text with one model, TP=1, and graphs disabled.
2. Verify the tokenizer, chat template, EOS, stop parameters, and context limits.
3. Collect target-device profiles and check scheduling estimates.
4. Alternate two models serially and compare token outputs and cache release before and after switching.
5. Check concurrency, P/D, replicas, sliding windows, MLA, or Mamba state used by the target deployment.
6. For Foundry, complete SAVE/LOAD, model-switching, and token-ID comparisons.

Complete the checks applicable to the target deployment before adding the model to that environment's usable set. This acceptance procedure covers text models; other input types need corresponding processing and inference checks.
