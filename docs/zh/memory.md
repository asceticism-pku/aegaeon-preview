# 内存与 KV Cache

Aegaeon 分别管理模型权重缓存和请求的 KV 状态。

| 区域 | 内容 | 主要配置 |
|---|---|---|
| CPU 权重缓存 | 节点内 Worker 共享的模型权重 | `model_cache_size_gb` |
| CPU KV Cache | 执行步骤之间保存的请求状态 | `cpu_num_slabs` × `cpu_slab_size_bytes` |
| GPU KV 预算 | 从容量估算中扣除当前和预取权重后的空间 | `memory_utilization` |
| QuickLoader GPU arena | 为权重和 KV 分配预留的空间 | `AEGAEON_ARENA_FRACTION` |

GB/GiB 配置以 `1024^3` 字节计。总预算还需留出 Ray、tokenizer、映射文件及临时张量的空间。

## 权重缓存

`model_cache_size_gb` 大于 0 时使用固定预算；为 0 时根据启动模型估算，空服务则使用 20 GiB。缓存可驱逐非活跃模型以加载其他模型。

## KV Cache

BlockManager 分配请求的 KV block，并协调 CPU/GPU 搬运。P/D 模式下，Prefill 与 Decode 共享节点的 CPU KV Cache。Decode Work Stealing 在同一节点内转移 batch 的归属。

## 共享内存

CPU 缓存使用共享的 pinned host memory。为 `/dev/shm` 分配足够容量，并配置服务或容器的 memlock 限制。独立服务使用不同的共享内存目录，通过 `AEGAEON_SHM_DIR` 设置。

预算字段见[配置参考](configuration.md)，分配错误见[故障排查](troubleshooting.md)。
