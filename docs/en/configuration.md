# Configuration Reference

## YAML structure and precedence

Configuration has three main sections: `server`, `models`, and `devices`. `aegaeon start --config` is required. Use absolute model paths and explicit memory budgets. Prepare identical model files and profiles on every Ray node before startup.

`create_service()` applies these per-field precedence rules:

| Setting | Effective rule |
|---|---|
| host / port | CLI passes directly to uvicorn; server YAML does not read these fields |
| Ray address | `--ray-address` → `AEGAEON_RAY_ADDRESS` → `RAY_ADDRESS` → `auto` |
| Startup models | `--models` → `server.startup_models` → all configured models |
| tensor_parallel_size | Must be 1; a nonzero CLI value wins; unset or 0 reads YAML / ServerConfig |
| Engine counts, CPU slabs, weight cache size | Normal startup uses ServerConfig; corresponding CLI values mainly serve as fallbacks |
| SLO | Environment variables read at utils import, not server YAML |

For omitted server fields, `getattr` reads ServerConfig defaults. Put Engine topology directly in YAML; matching CLI options such as `--num-engines` serve only the fallback path.

Omitted or null startup_models starts all configured models. `[]` starts an empty service for later deployment; a nonempty list selects those models. CLI `--models ''` expresses an empty set. Unknown models fail startup. Duplicate CLI names are deduplicated; YAML lists require unique nonempty strings.

## ServerConfig defaults

These are class defaults. Size production budgets from host memory, model count, and concurrency.

| Field | Default | Meaning / constraints |
|---|---:|---|
| cpu_num_slabs | 128 | CPU KV slabs per node |
| cpu_slab_size_bytes | 1073741824 | 1 GiB per slab, in bytes |
| memory_utilization | 0.775 | GPU capacity estimate; the CUDA allocator manages actual allocations separately |
| worker_num_cpus | 16 | Ray CPUs requested per Worker |
| nnodes | 1 | Physical Ray node count |
| num_prefill_engines | 1 | Prefill Engines per node |
| num_decode_engines | 1 | Decode Engines per node |
| num_engines | 0 | Simple Engines per node; mutually exclusive with P/D |
| tensor_parallel_size | 1 | One Worker per Engine; 1 is the only supported value |
| model_cache_size_gb | 0 | CPU weight cache in GiB; 0 selects automatic sizing |
| dummy_weights | false | Placeholder weights for control-path tests only |
| decode_work_stealing | false | Node-local Decode batch transfers |
| decode_work_stealing_min_batch_gap | 2 | Minimum donor/receiver batch-count gap, at least 2 |
| model_placement_policy | least-loaded | Also accepts round-robin |
| request_routing_policy | least-outstanding | Also accepts round-robin |
| startup_models | null | null=all models; []=empty service |
| cuda_graph | CudaGraphConfig() | Disabled by default; see below |

Default CPU KV capacity is 128 GiB per node, in addition to the CPU weight cache, mapped weight files, and process memory. Hosts below these budgets need smaller slab and cache settings.

A P/D topology needs at least one Prefill Engine and one Decode Engine. Simple mode instead sets `num_engines>0` and both P/D Engine counts to 0.

## Model entries

| Field | Default | Meaning |
|---|---|---|
| name | Required | Client-facing model name; need not match the profile folder |
| path | Empty string | Local weights; when unset, ModelSpec.path() triggers download |
| params | null | **Input multiplied by 2^30 to obtain parameter count** |
| profile | Empty string | Performance directory alias; otherwise normalized from the model name's last segment |
| max_model_len | null | Total context length override, still subject to model capability |
| id | Unset | Generated sequentially when all are omitted; when explicit, supply an ID for every model |
| tool_parser | null | Nonempty vLLM tool parser name; explicitly selects generated function-call parsing for a model alias |

`params: 4` means `4 × 2^30` parameters. Usually omit params so the registry reads weight metadata. If that fails, divide the actual total parameter count by `2^30` before setting `params`. Model entries parse only the fields above; GPU clock fields currently never reach ModelSpec.

## Device entries

| Field | Meaning |
|---|---|
| name | Required logical device name matching detection/aliases |
| memory | Required; multiplied by 2^30 to obtain bytes |
| bandwidth | Optional; multiplied by 2^30 to obtain bytes/s; omission attempts NVML detection |
| profile | Retained in DeviceSpec.extra as a performance directory alias |
| Other fields | Retained in DeviceSpec.extra; only fields read by an implementation have an effect |

PCIe generation and width provide a bandwidth estimate. Use profiling data for measured H2D bandwidth. Omitting devices leaves the registry empty. Runtime detection uses PyTorch CUDA for GPU names and capacity and NVML for a PCIe bandwidth estimate. Configure the device explicitly in YAML if device queries or bandwidth detection fail.

## CUDA graph defaults and constraints

| Field | Class default | Constraints |
|---|---|---|
| mode | 'off' | 'off' / save / load; quote each value |
| archive_dir | '' | Required for save/load |
| batch_sizes | [1,2,4,8] | Nonempty positive integers, sorted and deduplicated |
| seq_len_buckets | [128,256,512,1024,2048] | Class accepts multiple; **Worker save/load currently requires exactly one maximum-length bucket** |
| strict | true | Controls strict checks such as initialization scratch bounds; ABI/layout/binding checks always run |
| base_address | 0x400000000000 | Positive and 2 MiB aligned; accepts a quoted hexadecimal string |
| region_size_bytes | 274877906944 | 256 GiB virtual address range; positive and 2 MiB aligned |
| scratch_size_bytes | 4294967296 | 4 GiB; positive, 2 MiB aligned, and smaller than region_size_bytes |

Normal Simple execution does not enable the Foundry hook. Use P/D Decode to test this integration. See [CUDA Graphs](cuda-graphs.md).

## After editing configuration

Registries live in process-global state, and `set_model_registry()` returns immediately after initialization. Restart the service after YAML changes. Deployment APIs change runtime state while leaving the YAML file unchanged.
