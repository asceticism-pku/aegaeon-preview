# Profiling and Latency Estimation

## Why profiles are required

PrefillEstimator and DecodeEstimator fit LinearRegression models from JSON collected on the target GPU. When no profile data is found for fitting, initialization raises FileNotFoundError; malformed JSON or insufficient samples at an individual point fail during reading or calculation; collect the profile before starting a service that depends on it.

## Directory layout

```text
profiles/
  Qwen/
    Qwen3-4B/
      A100-PCIE-40GB/
        i32b1.json
        i64b1.json
        ...
```

The layout matches `profile: Qwen/Qwen3-4B` in the [QuickStart](quickstart.md) and the bundled A100 profile data. Model keys prefer explicit profile aliases; otherwise names are normalized with compatibility checks for old directories. Device keys prefer DeviceSpec.extra.profile, then probe CUDA product names and logical device names.

An explicit model `profile` is used exactly as configured. The bundled directory is `Qwen/Qwen3-4B`, so set that alias when reusing it; `profile: qwen3_4b` requires separately collected data under `profiles/qwen3_4b/`. When `profile` is omitted, lookup first tries the normalized model name and then the original model-name directory for compatibility. This fallback is only applied when the model has no explicit alias.

`--model` in the collection command below is the checkpoint path, while `--output-json` selects the profile location. The batch collector derives the model directory from `models[].profile` and the device directory from the assigned GPU's product name. Match the serving configuration's aliases to the resulting directories; these keys are independent of the public API model name and local checkpoint path.

`AEGAEON_PROFILE_ROOT` selects the root used for serving and batch collection; the default is the checkout's `profiles/` directory. Place one-point `--output-json` results under the same root.

JSON must provide the fields needed by the sampler and estimator, including prefill_latencies and decode_latencies. Latency extraction sorts samples, removes the smallest and largest, then averages the rest; one or two samples are insufficient.

The collector runs vLLM's `LLM` directly, rather than the Aegaeon HTTP service. Each prefill sample is the first request's `first_token_ts - scheduled_ts`; each decode sample is that request's first-to-last token interval divided by generated tokens minus one. Both use seconds. Defaults are 4 warmup iterations and 12 measured iterations. Prompts use random token IDs in `[0, 10000)`, so the model must have valid IDs throughout that range.

Prefix caching is disabled so repeated warmup and measurement inputs execute prefill instead of reusing cached prefixes.

## Batch collection at TP=1

```bash
CUDA_VISIBLE_DEVICES=0 python profiles/do_profile.py --model-config /srv/aegaeon/config.yaml --tp 1
```

The script enumerates input lengths from 16 to 4096, batch sizes, and filtering conditions. It invokes benchmark/benchmark_latency.py with bf16, enforce_eager, gpu-memory-utilization=0.90, and max-model-len=8192. It traverses all configured models rather than only startup_models.

The batch script fixes max-model-len at 8192 and gpu-memory-utilization at 0.90. Use the one-point command for models with a native context below 8192 or devices with insufficient memory. Each assigned GPU runs its benchmark in a separate subprocess. A failed command, missing output, or invalid sample array causes a nonzero exit; a validated new JSON file replaces the previous profile only after collection succeeds.

## Collect one point

```bash
mkdir -p profiles/Qwen/Qwen3-4B/A100-PCIE-40GB
CUDA_VISIBLE_DEVICES=0 python benchmark/benchmark_latency.py \
  --model /srv/models/Qwen3-4B --dtype bfloat16 --enforce-eager \
  --tensor-parallel-size 1 --max-model-len 4096 --gpu-memory-utilization 0.90 \
  --input-len 32 --output-len 10 --batch-size 1 \
  --output-json profiles/Qwen/Qwen3-4B/A100-PCIE-40GB/i32b1.json
```

One point confirms the collection path. A scheduling profile needs the target input lengths, batch sizes, and context ranges, together with recorded regression error.

## TP and heterogeneous hardware

Set `--tp` to 1. The batch script exposes the option, but GPU allocation uses a single-device string and the estimator executes `assert tp == 1`; TP>1 is unsupported.

The script labels each model's output with its assigned GPU's product name. Match the serving device's explicit profile alias to that output directory. Collect models on each target GPU type separately when heterogeneous devices need profiles for the same models.

## How estimates are used

Predictions drive scheduling; performance reports should use measured results. Recollect profiles after changing the backend, graph mode, model version, clocks, drivers, or kernels.
