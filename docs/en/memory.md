# Memory, Weight Caches, and KV Cache

## Budget breakdown

| Region | Main contents | Configuration |
|---|---|---|
| CPU weight cache | Model weight slices in QuickCache | model_cache_size_gb |
| CPU KV | Request KV slabs, shared mappings, pinned memory | cpu_num_slabs × cpu_slab_size_bytes |
| GPU | Active model, KV, prefetch, operator workspace, temporary tensors | memory_utilization contributes to the estimate |
| Other CPU memory | Python, Ray, tokenizer, metadata, loader pinned buffers | Not fully covered by the budgets above |

The gb suffix in the CPU weight cache setting actually means GiB, multiplying by 1024^3. QuickLoaderConfig defaults pinned_buffer_size to 4 GiB; it is not a general server YAML option. Include process and operating-system headroom in total CPU memory planning.

## Automatic weight cache sizing

model_cache_size_gb>0 uses the specified size. At 0 with startup models, sizing estimates total parameters ×2×1.2. At 0 without startup models, it uses 20 GiB. Automatic sizing does not provide unlimited dynamic growth.

A model enters the cache only when both total free bytes and the slice layout fit. Eviction covers removable inactive models. Deployment failures appear in allocator-fit logs and the response `failed` field.

## GPU estimates

ModelConfig estimates KV capacity from configured device capacity × memory_utilization minus the active model and optional prefetched model sizes. Weight estimates assume bf16, TP partitioning, and alignment. Nonpositive remaining capacity raises MemoryError.

Actual GPU peaks also include temporary loading, attention workspace, graph inputs, buffers, and runtime initialization. memory_utilization is neither a hard cap nor identical to vLLM's native parameter. Raising it blindly can worsen OOM risk.

## KV blocks

Aegaeon's logical BLOCK_SIZE is fixed at 16 tokens and cannot be changed through the current YAML interface. FlashAttention, MLA, sliding windows, and Mamba-style state have different descriptions and transfer requirements. Check cache_groups / cache_transfer compatibility for new models.

For conventional attention, a logical block estimate uses layer count, 16 tokens, per-rank KV heads, and K/V head sizes. This simplified formula does not cover all hybrid caches or backend physical page layouts.

## Shared memory

The default directory is `/dev/shm`; AEGAEON_SHM_DIR can override it. Directory capacity, physical host memory, and pinned memory support must all be sufficient. Small default container shm allocations are often inadequate.

```bash
df -h /dev/shm
free -h
nvidia-smi
```

Changing the directory only relocates shared files; pinned-memory capacity stays the same. Cache files retain their existing sizes. Give each service an isolated directory and remove only files created by that instance.

## Transfers and events

CPU→GPU and GPU→CPU transfers are asynchronous and coordinated by CUDA events. Logical CPU residency does not imply that all GPU→CPU events have completed. Work Stealing must preserve these dependencies when moving queues.

## Failure categories

- `not enough free blocks on CPU`: increase CPU KV capacity or reduce concurrency/output lengths; increasing the weight cache does not fix it.
- GPU OOM: inspect active/prefetched weights, temporary loading, and workspace; reduce context or concurrency and leave headroom.
- Cache fit failure: inspect CPU weight capacity, fragmentation, and active-model protection; this is distinct from GPU memory exhaustion.
- Graph VMM layout error: check archive addresses and tensor bindings; increasing GPU memory alone does not resolve it.
