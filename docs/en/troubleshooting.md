# Troubleshooting

## Investigation order

Preserve the full traceback, startup command, YAML, source revision, dependency versions, GPU/driver information, and logs. Check imports/extensions, Ray resources, cache budgets, models/profiles, and request schemas before investigating graph restoration or parallel execution.

## Installation and imports

**torch missing during setup**: setup.py imports torch at module scope; install a matching torch first. **Missing aegaeon.ops or undefined symbol**: check that the extension was built with the current torch/CUDA, whether torch changed, and whether nvcc matches the wheel. **Missing quick_model_loader._rlib**: check Rust tooling and the editable extension build. **vLLM internal import errors**: the project uses version-sensitive internal interfaces; check vllm==0.26.0 rather than upgrading indiscriminately.

**Conda reports database is locked**: libmamba's SQLite shard cache failed in the tested environment, even with a private package cache. Set `export CONDA_PKGS_DIRS="$HOME/.conda/pkgs-aegaeon"` and pass `--solver classic` to `conda create/install`; the installation guide already uses this combination.

### Optional installation checks

The installation script already checks CUDA execution, core imports, and dependency consistency. To diagnose a failed installation or an environment change, rerun these checks in the active environment:

```bash
python -c "import torch, vllm; print(torch.__version__, vllm.__version__, torch.version.cuda); print(torch.ones(1, device='cuda'))"
python -c "import torch, quick_model_loader._rlib, aegaeon.ops; from aegaeon import LLMService, NodeConfig, Request; print('core imports OK')"
aegaeon --help
aegaeon start --help
```

The installation guide uses PyTorch `2.11.0+cu129`, vLLM `0.26.0+cu129`, and CUDA `12.9`. The package uses lazy exports; import the modules above individually to check its CUDA extension and loader.

## Ray and GPUs

**Ray cannot find a cluster**: auto connects to an existing cluster; start Ray or set --ray-address. **Missing node_0/node_1 resources**: start Ray with the corresponding custom resources; a multi-GPU host still has nnodes=1. **Actors remain pending**: inspect CPU, GPU, and placement-group resources with ray status, especially worker_num_cpus×Worker count. **CUDA unavailable or incorrect IDs**: check nvidia-smi, torch.cuda.is_available, drivers, and actor visibility propagation.

```bash
ray status
nvidia-smi
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.device_count())"
```

## Caches and memory

**Insufficient CPU free blocks**: increase slabs or reduce concurrency/length; check host memory and the shared directory. **Deployment returns 507**: inspect failed, node cache fit, fragmentation, and active-model protection; 507 is not a generic GPU OOM status. **GPU MemoryError/OOM**: distinguish nonpositive estimated capacity from actual loading peaks; inspect current/prefetched weights and temporary workspace. **High CPU memory at startup**: defaults allocate 128 GiB of CPU KV plus the weight cache; set budgets explicitly for the machine.

### Host registration and memlock

QuickLoader uses shared host memory and CUDA host registration. For host registration or pinned-memory errors, inspect shared memory and the soft/hard memlock limits (`ulimit -l` uses KiB):

```bash
df -h /dev/shm
ulimit -Sl
ulimit -Hl
```

One A100 validation passed with a 64 MiB hard memlock limit; use actual startup results to determine the limits needed in other environments. Adjust service or container limits if needed, then restart Ray.

## Models and profiles

**No usable latency profiles**: inspect model.profile, device.profile, and JSON fields at the reported directory; recollect on the target device. **TP assertion**: profile loading currently requires tp==1; adding GPUs does not remove this restriction. **Parameter estimation fails**: check weight metadata and the 2^30 units for manual params. **Repeated special-token output**: check EOS IDs, generation_config, templates, and stops; preserve token IDs for diagnosis.

## HTTP interfaces

**400 for temperature/top_p**: only greedy values are supported. **422 extra_forbidden**: remove unsupported fields and inspect openapi.json and the request schema. **400 model not deployed**: match the registered name; configuration alone without startup/deploy does not make it available. **503**: inspect READY replicas and DRAINING state. **429**: inspect the 60-second window and environment settings; this is not a strict real-time token quota.

**Streaming output is delayed**: use curl -N and inspect proxy buffering, networking, and first-load costs. **Outstanding remains nonzero after disconnect**: backend execution continues to its stopping condition, and cleanup waits for completion. **YAML changes have no effect**: restart the service; global registries do not support hot reload. **CLI engine counts have no effect**: normal startup uses ServerConfig; change YAML.

## Foundry

**Hook path required**: AEGAEON_FOUNDRY_HOOK_PATH must point to the .so colocated with `foundry.ops` before Worker startup. **libboost_json/libboost_filesystem not found**: install matching Boost>=1.83 libraries or add a custom Boost `lib` directory to `LD_LIBRARY_PATH`. **undefined symbol: cuFuncGetName**: use the forward-compatibility library matching the CUDA wheel and set `AEGAEON_CUDA_COMPAT_PATH`; also add it to `LD_LIBRARY_PATH` for a direct import check. **expect one maximum sequence length**: use exactly one seq_len_buckets element. **archive incomplete**: verify every registry model, rank, and graph, not only startup models. **obsolete ABI / layout mismatch / binding mismatch**: restore a matching SAVE environment or capture again; strict=false cannot bypass every check. **Enabled without replay**: normal Simple mode has no integration, or batch/length/state does not match; use P/D and inspect replay logs.

## Information for an issue report

Include a short reproducible request, sanitized configuration, full traceback, expected and actual behavior, resources and dependency versions, and whether the failure reproduces with one model, TP=1, and graphs disabled. Performance issues also need actual token lengths, sample counts, warmup, and load conditions.
