# Quickstart: Send Your First Request

This guide uses one Linux host, one available NVIDIA GPU, a real local model, Simple mode, and TP=1. Complete [installation](installation.md) first and provide sufficient CPU memory and shared memory.

## 1. Prepare the model and configuration

Download the complete `Qwen/Qwen3-4B` checkpoint to `/srv/models/Qwen3-4B`. Replace this example with your actual absolute path. The directory must include the configuration, tokenizer, chat template, and all weight shards.

Copy the single-node example to a working configuration, then update the model path, GPU name, capacity, and profile names:

```bash
cp docs/en/examples/simple.yaml aegaeon.yaml
```

The key configuration sections are:

```yaml
server:
  nnodes: 1
  num_engines: 1
  num_prefill_engines: 0
  num_decode_engines: 0
  tensor_parallel_size: 1
  cpu_num_slabs: 8
  cpu_slab_size_bytes: 1073741824
  model_cache_size_gb: 20
  memory_utilization: 0.775
  startup_models: [Qwen/Qwen3-4B]
  cuda_graph:
    mode: 'off'
models:
  - name: Qwen/Qwen3-4B
    path: /srv/models/Qwen3-4B
    profile: Qwen/Qwen3-4B
    max_model_len: 4096
devices:
  - name: A100_40GB
    memory: 39
    bandwidth: 32
    profile: A100-PCIE-40GB
```

The model and device profile names map to `profiles/Qwen/Qwen3-4B/A100-PCIE-40GB/`. Use the quoted string `mode: 'off'` for compatibility with YAML 1.1 parsers that interpret bare `off` as a boolean. Capacity, bandwidth, and memory budgets demonstrate the format; measure and set them for the target hardware.

## 2. Check or collect profiles

Scheduling estimates depend on `profiles/<model-profile>/<device-profile>/i<input>b<batch>.json`. Follow the [profiling guide](profiling.md) to check that the data is complete and matches the model and GPU. Collect missing profiles on the target GPU first.

```bash
CUDA_VISIBLE_DEVICES=0 python profiles/do_profile.py --model-config aegaeon.yaml --tp 1
```

Batch collection takes time and occupies the selected GPU. Check the logs and generated JSON files when it finishes.

## 3. Start Ray and the service

This example pins up to 20 GiB of model cache and 8 GiB of CPU KV cache in each worker. Before starting Ray, confirm that `/dev/shm` has enough capacity and that the inherited soft memlock limit covers at least 28 GiB (`ulimit -l` reports KiB). See the [installation guide](installation.md) if the hard limit is lower.

If no Ray cluster is running, execute on this host:

```bash
df -h /dev/shm
ulimit -Sl
ray start --head --port=6789 --num-cpus="$(nproc)" --resources='{"node_0": 1}'
mkdir -p logs
AEGAEON_LOG_FILE="$PWD/logs/server.log" \
  aegaeon start --config aegaeon.yaml \
  --host 127.0.0.1 --port 8000 --ray-address 127.0.0.1:6789
```

Startup initializes shared CPU caches and actors; model size, cache configuration, and machine state determine its duration. Run one API instance with the default actor and cache names for each node set.

For a single node, `--ray-address local` creates a local Ray runtime with the `node_0` resource. For an existing multi-node cluster, pass the head-node address. Ray workers inherit resource limits from the Ray process, so changing `ulimit` after the cluster has started does not fix an insufficient memlock limit.

## 4. Check the service and model

```bash
curl -sS http://127.0.0.1:8000/health
curl -sS http://127.0.0.1:8000/v1/models
```

`/health` returns `{"status":"ok"}`, but this is a fixed process response rather than a check of every engine and GPU. The model list should include the configured model name.

## 5. Send a Chat request

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"Explain KV Cache in one sentence."}],"temperature":0,"max_tokens":64}'
```

The response includes `choices[0].message.content`, `finish_reason`, and `usage`. The model and input determine the text. The first request also includes GPU weight loading and initialization time.

## 6. Stream output and inspect runtime state

```bash
curl -N http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"Explain the difference between Prefill and Decode."}],"temperature":0,"max_tokens":128,"stream":true,"stream_options":{"include_usage":true}}'
curl -sS http://127.0.0.1:8000/v1/aegaeon/runtime
```


