# CLI Reference

Use `aegaeon` or `python -m aegaeon`.

## doctor

```bash
aegaeon doctor
CUDA_VISIBLE_DEVICES=0 aegaeon doctor --check-cuda
```

| Option | Parser default | Meaning |
|---|---|---|
| --check-cuda | false | Run small PyTorch/CuPy CUDA operations and import vLLM's native extension on an allocated GPU |

Check dependencies; `--check-cuda` also runs CUDA checks.

## start

```bash
aegaeon start --config /srv/aegaeon/config.yaml --host 127.0.0.1 --port 8000 --ray-address 127.0.0.1:6789
```

| Option | Parser default | Meaning |
|---|---|---|
| --config | Required | YAML file |
| --host | 0.0.0.0 | Bind address |
| --port | 8000 | Port |
| --models | None | Comma-separated startup models; overrides startup_models |
| --num-prefill-engines | 1 | Normal startup uses YAML ServerConfig |
| --num-decode-engines | 1 | Same |
| --num-engines | 0 | Same |
| --tensor-parallel-size | None | Explicit nonzero value overrides YAML; 0 falls back to YAML |
| --model-cache-size-gb | 20 | Fallback CPU weight capacity in GiB |
| --cpu-num-slabs | 15 | Fallback slab count |
| --cpu-slab-size-gb | 1 | Fallback size in GiB |
| --ray-address | None | Cluster address, ahead of environment variables |

Set topology and memory budgets in [YAML configuration](configuration.md).

## deploy

```bash
aegaeon deploy Qwen/Qwen3-4B --path /srv/models/Qwen3-4B --replicas 2
```

| Option | Default | Meaning |
|---|---|---|
| model | Required positional argument | Service model name |
| --path | None | Local weights; otherwise registered path, then download |
| --host | localhost | API host |
| --port | 8000 | API port |
| --max-model-len | None | Context override |
| --replicas | 1 | Minimum READY replica target for automatic placement |
| --nodes | None | Explicit node IDs; bypass automatic placement |
| --tool-parser | None | vLLM tool-parser override for this model alias |

Partial operations exit with a nonzero status; see [Model Management](model-management.md).

## undeploy

```bash
aegaeon undeploy Qwen/Qwen3-4B --nodes node_1
```

| Option | Default | Meaning |
|---|---|---|
| model | Required positional argument | Service model name |
| --host | localhost | API host |
| --port | 8000 | API port |
| --nodes | All known replicas | Unload only the listed nodes |

Retry draining replicas after their active requests finish.

## Queries

The main help lists commands; each command's help lists its options and parser defaults.

```bash
aegaeon --help
aegaeon doctor --help
aegaeon start --help
aegaeon deploy --help
aegaeon undeploy --help
```

For startup resources, YAML values and defaults take precedence over the legacy resource fallback options. `--models` and a nonzero `--tensor-parallel-size` override their YAML fields; `--ray-address` precedes `AEGAEON_RAY_ADDRESS`, `RAY_ADDRESS`, then `auto`.
