# Profiling

Aegaeon uses model/device latency profiles to estimate Prefill and Decode time. Store profiles under `AEGAEON_PROFILE_ROOT`, which defaults to the repository's `profiles/` directory:

```text
profiles/Qwen/Qwen3-4B/A100-PCIE-40GB/i32b1.json
```

The model's `profile` and device's `profile` select these directories. Collect on the target GPU with the same runtime used for serving.

## Batch collection

Run from the repository root:

```bash
CUDA_VISIBLE_DEVICES=0 python profiles/do_profile.py --model-config /srv/aegaeon/config.yaml --tp 1
```

## Collect one point

For custom input lengths or context limits:

```bash
mkdir -p profiles/Qwen/Qwen3-4B/A100-PCIE-40GB
CUDA_VISIBLE_DEVICES=0 python benchmark/benchmark_latency.py \
  --model /srv/models/Qwen3-4B --dtype bfloat16 --enforce-eager \
  --tensor-parallel-size 1 --max-model-len 4096 --gpu-memory-utilization 0.90 \
  --input-len 32 --output-len 10 --batch-size 1 \
  --output-json profiles/Qwen/Qwen3-4B/A100-PCIE-40GB/i32b1.json
```

Cover the input lengths and batch sizes used by the service. Recollect after changing the model, hardware, or execution settings.
