# Memory and KV Cache

Aegaeon manages weight caches separately from request KV state in order for fine-grained migration manangement.

| Region | Contents | Main setting |
|---|---|---|
| CPU weight cache | Model weights shared by node Workers | `model_cache_size_gb` |
| CPU KV cache | Request state between execution steps | `cpu_num_slabs` × `cpu_slab_size_bytes` |
| GPU KV budget | Capacity estimate after subtracting active and prefetched weights | `memory_utilization` |
| QuickLoader GPU arena | Reserved space for weights and KV allocations | `AEGAEON_ARENA_FRACTION` |

Aegeaeon also leaves capacity for Ray, tokenizers, mapped files, and temporary tensors.

## Weight cache

A positive `model_cache_size_gb` sets a fixed budget. Zero sizes the cache from startup models, or uses 20 GiB for an empty startup. Inactive models can be evicted to admit another model.

## KV cache

BlockManager allocates request KV blocks and coordinates CPU/GPU transfers. In P/D mode, Prefill and Decode share the node's CPU KV cache. Decode Work Stealing transfers batch ownership within the same node.

## Shared memory

CPU caches use shared, pinned host memory. Give `/dev/shm` enough capacity and set service/container memlock limits accordingly. Independent services need isolated shared-memory directories; configure `AEGAEON_SHM_DIR` before startup.

Choose budgets in the [configuration reference](configuration.md). For allocation errors, see [Troubleshooting](troubleshooting.md).
