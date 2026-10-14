# CLI Reference

setup.py registers `aegaeon=aegaeon.cli:main`; `python -m aegaeon` is also available. Commands are doctor, start, deploy, and undeploy.

## doctor

```bash
aegaeon doctor
CUDA_VISIBLE_DEVICES=0 aegaeon doctor --config /srv/aegaeon/config.yaml --check-cuda
aegaeon doctor --config /srv/aegaeon/config.yaml --models Qwen/Qwen3-4B --strict
```

| Option | Parser default | Meaning |
|---|---|---|
| --config | None | YAML configuration for topology, local model/profile, and memory-budget checks |
| --models | None | Comma-separated startup models; overrides startup_models, as in start |
| --tensor-parallel-size | None | Explicit nonzero value overrides YAML; 0 falls back to YAML; only TP=1 is supported |
| --check-cuda | false | Run small PyTorch/CuPy CUDA operations, import vLLM's native extension, and probe one-page CUDA host registration on an allocated GPU |
| --strict | false | Exit with status 1 for warnings as well as failures |

The default checks cover runtime imports including `aegaeon.ops`, pinned backend
versions, the CUDA/C++ toolchain, Rust/Cargo availability, shared-memory
write/mmap and capacity, available host memory, and memlock limits. `--config` adds startup model
selection, readable local model assets and weight shards, declared decode
profile candidates, topology constraints, and the configured shared-memory budget. These
checks read local files without downloading models or starting Ray.

Results are `OK`, `WARN`, or `FAIL`. A failure exits with status 1. Warnings keep
the normal exit status at 0 unless `--strict` is set. Examples include a
memlock allowance below the budget, a finite allowance with an unknown budget,
and missing Rust/Cargo with installed extensions.
Plain `doctor` omits CUDA tensor operations; run `--check-cuda` on an allocated
GPU on each worker node. Checks apply to the current node; full-size host-memory
registration, Ray cluster readiness, and real inference remain separate
acceptance steps. See [runtime checks](installation.md#runtime-checks-and-wheel-installations).

`start` passes its configuration and model/TP overrides to static configuration
and local host checks before importing the API, with CUDA queries omitted.
Definite failures such as invalid local model files or insufficient cache memory
block startup. Static profile candidates that are missing or invalid produce
`WARN`; a static `OK` also leaves the actual worker GPU unverified.
Use `doctor --config PATH --check-cuda` for actual GPU/profile checks: missing or
invalid profiles for GPUs required by the engines then produce `FAIL`.
`--strict` also returns nonzero for the static uncertainties reported as warnings.

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

Read [configuration precedence](configuration.md) first. This guide uses the `aegaeon` CLI. `python -m aegaeon.api` has a separate parser and defaults TP to 1.

## deploy

```bash
aegaeon deploy Qwen/Qwen3-4B --path /srv/models/Qwen3-4B --replicas 2
aegaeon deploy Qwen/Qwen3-4B --nodes node_0 node_1
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

The client uses HTTPConnection with a 300-second timeout. For HTTP 200 with JSON `status=partial` or a nonempty `failed`, the CLI reports partial completion and exits with status 1; it prints success only for a complete result. When calling HTTP directly, inspect `status`, `failed`, and `ready_nodes`.

## undeploy

```bash
aegaeon undeploy Qwen/Qwen3-4B --nodes node_1
aegaeon undeploy Qwen/Qwen3-4B
```

| Option | Default | Meaning |
|---|---|---|
| model | Required positional argument | Service model name |
| --host | localhost | API host |
| --port | 8000 | API port |
| --nodes | All known replicas | Unload only the listed nodes |

Busy replicas enter DRAINING and appear as partial failures; the CLI exits with status 1. Run undeploy again after their requests finish. Use runtime replica and cache state to confirm release.

## Queries

```bash
aegaeon --help
aegaeon doctor --help
aegaeon start --help
aegaeon deploy --help
aegaeon undeploy --help
```

The CLI provides `doctor`, `start`, `deploy`, and `undeploy`. Query status and lists through HTTP, run benchmark/profile repository scripts, and stop the service through the process manager.
