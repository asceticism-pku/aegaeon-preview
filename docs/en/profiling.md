# Profiling and Latency Estimation

## Why profiles are required

PrefillEstimator and DecodeEstimator fit LinearRegression models from JSON collected on the target GPU. A missing usable sample set raises FileNotFoundError; collect the profile before starting a service that depends on it.

## Directory layout

```text
profiles/
  qwen3_4b/
    A100-PCIE-40GB/
      i32b1.json
      i64b1.json
      ...
```

Model keys prefer explicit profile aliases; otherwise names are normalized with compatibility checks for old directories. Device keys prefer DeviceSpec.extra.profile, then probe CUDA product names and logical device names.

JSON must provide the fields needed by the sampler and estimator, including prefill_latencies and decode_latencies. Latency extraction sorts samples, removes the smallest and largest, then averages the rest; one or two samples are insufficient.

## Batch collection at TP=1

```bash
CUDA_VISIBLE_DEVICES=0 python profiles/do_profile.py --model-config /srv/aegaeon/config.yaml --tp 1
```

The script enumerates input lengths from 16 to 4096, batch sizes, and filtering conditions. It invokes benchmark/benchmark_latency.py with bf16, enforce_eager, gpu-memory-utilization=0.90, and max-model-len=8192. It traverses all configured models rather than only startup_models.

The batch script fixes max-model-len at 8192 and gpu-memory-utilization at 0.90. Use the one-point command for models with a native context below 8192 or devices with insufficient memory. The script launches shell subprocesses, so check logs and JSON sample counts after collection.

## Collect one point

```bash
mkdir -p profiles/qwen3_4b/A100-PCIE-40GB
CUDA_VISIBLE_DEVICES=0 python benchmark/benchmark_latency.py \
  --model /srv/models/Qwen3-4B --dtype bfloat16 --enforce-eager \
  --tensor-parallel-size 1 --max-model-len 4096 \
  --input-len 32 --output-len 10 --batch-size 1 \
  --output-json profiles/qwen3_4b/A100-PCIE-40GB/i32b1.json
```

One point confirms the collection path. A scheduling profile needs the target input lengths, batch sizes, and context ranges, together with recorded regression error.

## TP and heterogeneous hardware

Set `--tp` to 1. The batch script exposes the option, but GPU allocation uses a single-device string and the estimator executes `assert tp == 1`; TP>1 is unsupported.

The script uses visible GPU0's product name for directory labels. Collect heterogeneous GPUs separately and check directory names to avoid mislabeled data.

## How estimates are used

Predictions drive scheduling; performance reports should use measured results. Recollect profiles after changing the backend, graph mode, model version, clocks, drivers, or kernels.
