# Troubleshooting

## Investigation order

Preserve the full traceback, startup command, YAML, source revision, dependency versions, GPU/driver information, and logs. Check imports/extensions, Ray resources, cache budgets, models/profiles, and request schemas before investigating graph restoration or parallel execution.

## Installation and imports

**torch missing during setup**: setup.py imports torch at module scope; install a matching torch first. **Missing aegaeon.ops or undefined symbol**: check that the extension was built with the current torch/CUDA, whether torch changed, and whether nvcc matches the wheel. **Missing quick_model_loader._rlib**: check Rust tooling and the editable extension build. **vLLM internal import errors**: the project uses version-sensitive internal interfaces; check vllm==0.26.0 rather than upgrading indiscriminately.

**Conda reports database is locked**: A lock conflict in libmamba's SQLite shard cache can cause this error. Use `--solver classic` when creating the environment; the installer supplies a separate cache path when `CONDA_PKGS_DIRS` is unset and uses the classic solver for toolchain installation. If Foundry dependency installation still fails in `shards_cache`, prefix that `conda install` command with `CONDA_PLUGINS_USE_SHARDED_REPODATA=false` to disable sharded repodata caching for the command. See the command in [CUDA Graph installation](cuda-graphs.md#install-foundry).

**The installer reports an incompatible driver**: default auto mode checks the cu130 driver and GPU prerequisites before installing packages. It chooses CUDA 13.0 when they pass and otherwise falls back to cu129. Driver `535.247.01` selects cu129. The cu129 / CUDA 12.9 route requires a driver branch number of at least 525 before installing packages and verifies real CUDA execution afterward. This branch check is only an initial filter; the GPU, exact driver version, and CUDA features used by the application must meet the [NVIDIA CUDA 12.x compatibility requirements](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html). The experimental CUDA 13.0 install route uses a conservative prerequisite of Linux driver `580.95.05` or newer to match the Update 2 toolchain; see the [CUDA 13.0 Update 2 release notes](https://docs.nvidia.com/cuda/archive/13.0.2/cuda-toolkit-release-notes/index.html). Every detected GPU must have compute capability of at least `7.5`. With explicit `--cuda cu130`, failing prerequisites terminate the installer. That route has not completed project end-to-end testing. Automatic fallback happens before installing packages; download, installation, or compilation failures terminate the selected route. Use `--cuda cu129` to force the tested route. PyTorch 2.11.0 is required by vLLM 0.26.0; use the fixed version combination provided by the installer.

Auto checks the driver and GPU compute capability. cu130 also needs a [Linux distribution supported by CUDA 13.0.2](https://docs.nvidia.com/cuda/archive/13.0.2/cuda-installation-guide-linux/index.html), such as Ubuntu 22.04 or 24.04. Ubuntu 20.04 was removed from this version's support table. If the distribution requirement is not met, select `--cuda cu129` explicitly in a new environment. The script installs the matching Toolkit for the selected route; an existing Toolkit does not determine automatic selection.

**CUDA Toolkit/compiler mismatch**: local extensions still require a CUDA Toolkit matching the PyTorch CUDA build. Rerun the installer in the intended environment to prepare matching CUDA, GCC/G++, and Rust tooling, then rebuild the extensions.

### Optional installation checks

The installation script includes checks for CUDA execution, core imports, and dependency consistency. To diagnose a failed installation or an environment change, rerun these checks in the active environment:

```bash
python -c "from importlib.metadata import version; import torch, vllm; print(torch.__version__, version('vllm'), torch.version.cuda); print(torch.ones(1, device='cuda'))"
python -c "import torch, quick_model_loader._rlib, aegaeon.ops; from aegaeon import LLMService, NodeConfig, Request; print('core imports OK')"
aegaeon --help
aegaeon start --help
```

The tested cu129 route should report PyTorch `2.11.0+cu129`, vLLM `0.26.0+cu129`, and CUDA `12.9`. For the cu130 route, which has not completed end-to-end testing, verify PyTorch base version `2.11.0` (the displayed version can include a CUDA suffix), vLLM `0.26.0`, and CUDA `13.0`. The package uses lazy exports; import the modules above individually to check its CUDA extension and loader. These are installation checks only; see the [QuickStart](quickstart.md) and [graph archive guide](cuda-graphs.md) for inference and CUDA Graph verification.

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

Preserve the specific host-registration error and inspect both shared-memory capacity and service or container resource limits. Use the traceback to identify the failing constraint and restart Ray after changes.

## Models and profiles

**No usable latency profiles**: inspect model.profile, device.profile, and JSON fields at the reported directory; recollect on the target device. **TP assertion**: profile loading currently requires tp==1; adding GPUs does not remove this restriction. **Parameter estimation fails**: check weight metadata and the 2^30 units for manual params. **Repeated special-token output**: check EOS IDs, generation_config, templates, and stops; preserve token IDs for diagnosis.

## HTTP interfaces

**400 for temperature/top_p**: only greedy values are supported. **422 extra_forbidden**: remove unsupported fields and inspect openapi.json and the request schema. **400 model not deployed**: match the registered name; configuration alone without startup/deploy does not make it available. **503**: inspect READY replicas and DRAINING state. **429**: inspect the 60-second window and environment settings; this is not a strict real-time token quota.

**Streaming output is delayed**: use curl -N and inspect proxy buffering, networking, and first-load costs. **Outstanding remains nonzero after disconnect**: backend execution continues to its stopping condition, and cleanup waits for completion. **YAML changes have no effect**: restart the service; global registries do not support hot reload. **CLI engine counts have no effect**: normal startup uses ServerConfig; change YAML.

## Foundry

The tested Foundry and CUDA Graph route uses cu129 / CUDA 12.9. Foundry builds and SAVE/LOAD with CUDA 13.0 have not completed end-to-end testing. The CUDA 12.9 compatibility package linked below applies only to CUDA 12.9 environments. Use a separate environment after switching versions, rebuild the extensions and Foundry, and use a new archive directory.

**NumPy import failure / NP_SUPPORTED_MODULES**: loading the system `libstdc++` can fail when NumPy's dependencies require a newer version. For an `NP_SUPPORTED_MODULES` error, first inspect earlier NumPy or `GLIBCXX_*` import errors in the full traceback. Add `$CONDA_PREFIX/lib` to `LD_LIBRARY_PATH` before starting Python. If an older driver also needs a compatibility library, keep that library first and Conda `lib` second, then start a fresh process:

```bash
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
# Run the next line only when a CUDA forward-compatibility library is configured
export LD_LIBRARY_PATH="$AEGAEON_CUDA_COMPAT_PATH:$LD_LIBRARY_PATH"
```

**Hook path required**: AEGAEON_FOUNDRY_HOOK_PATH must point to the .so colocated with `foundry.ops` before Worker startup. **libboost_json/libboost_filesystem not found**: install matching Boost>=1.83 libraries or add a custom Boost `lib` directory to `LD_LIBRARY_PATH`. **undefined symbol: cuFuncGetName**: follow the [CUDA 12.9 compatibility-library steps](cuda-graphs.md#cuda-129-compatibility-library) to download, verify, and extract the matching NVIDIA forward-compatibility library without root; set `AEGAEON_CUDA_COMPAT_PATH` and prepend it to `LD_LIBRARY_PATH` before importing Foundry. That download example is limited to Ubuntu 20.04 x86_64, A100, driver `535.247.01`, and CUDA 12.9. **expect one maximum sequence length**: use exactly one seq_len_buckets element. **archive incomplete**: verify every registry model, rank, and graph, not only startup models. **obsolete ABI / layout mismatch / binding mismatch**: restore a matching SAVE environment or capture again; strict=false cannot bypass every check. **Enabled without replay**: normal Simple mode has no integration, or batch/length/state does not match; use P/D and inspect replay logs.

## Information for an issue report

Include a short reproducible request, sanitized configuration, full traceback, expected and actual behavior, resources and dependency versions, and whether the failure reproduces with one model, TP=1, and graphs disabled. Performance issues also need actual token lengths, sample counts, warmup, and load conditions.
