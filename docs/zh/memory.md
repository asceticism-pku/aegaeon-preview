# 内存、权重缓存与 KV Cache

## 预算分解

| 区域 | 主要内容 | 配置 |
|---|---|---|
| CPU 权重缓存 | QuickCache 缓存模型权重 slices | model_cache_size_gb |
| CPU KV | 请求 KV slab、共享映射、pinned memory | cpu_num_slabs × cpu_slab_size_bytes |
| GPU | active model、KV、prefetch、算子 workspace、临时张量 | memory_utilization 参与估算 |
| 其他 CPU | Python、Ray、tokenizer、metadata、loader pinned buffer 等 | 单独计入进程与系统余量 |

CPU 权重 cache 参数名的 gb 实际乘以 1024^3，即 GiB。QuickLoaderConfig 默认 pinned_buffer_size 为 4 GiB，当前 server YAML 保持该内部默认值。CPU 总预算必须加上进程与系统余量。

## 自动权重 cache

model_cache_size_gb>0 使用指定大小；为 0 且有启动模型时按参数总量 ×2×1.2 估算；为 0 且启动模型集合为空时使用 20 GiB。该预算在 Controller 初始化时确定。

模型进入 cache 需要同时满足总空闲 bytes 和连续 slice 布局。驱逐范围只包含可移除的非活动模型；部署失败信息位于 allocator fit 日志和响应的 `failed` 字段。

## GPU 估算

ModelConfig 以设备配置容量 × memory_utilization 减去当前模型与可选预取模型大小，估计可用 KV 字节；权重估算按 bf16、TP 划分并对齐。若剩余<=0 会抛 MemoryError。

实际 GPU 峰值还包括临时加载、attention workspace、图输入、buffers 和 runtime 初始化。memory_utilization 用于 Aegaeon 的预算估算，语义与 vLLM 同名参数有所区别。遇到 OOM 时应先拆分各项占用，再调整预算并复测。

## KV block

Aegaeon 逻辑 BLOCK_SIZE 当前固定为 16 tokens，YAML 省略该配置项。FlashAttention、MLA、sliding window 与 Mamba 类状态有不同描述与搬运需求；接入新模型时需要检查 cache_groups / cache_transfer 的适配。

常规 attention 的逻辑块估算按层数、16 tokens、每 rank KV heads 与 K/V head size 计算。hybrid cache 和其他 backend 的物理 page layout 需要使用各自的布局计算方式。

## 共享内存

默认共享目录为 `/dev/shm`，可通过 AEGAEON_SHM_DIR 指定。目录空间、主机物理内存与 pinned memory 条件都要满足；容器默认小 shm 经常不足。

```bash
df -h /dev/shm
free -h
nvidia-smi
```

更换目录会改变共享文件位置，pinned memory 容量仍由主机内存配置决定。缓存文件会保留原有大小；每个服务应使用独立目录，并只清理由该实例创建的文件。

## 搬运与事件

CPU→GPU、GPU→CPU 是异步路径，CUDA event 串联生命周期。CPU 逻辑 resident 状态先于 GPU→CPU 事件完成时，Work Stealing 会在移动队列时一并继承这些同步依赖。

## 故障分类

- `not enough free blocks on CPU`：增加 CPU KV 预算，或降低并发与输出长度；该错误对应 KV block，而非权重 cache。
- GPU OOM：检查 active/prefetch 权重、临时加载和 workspace，缩短上下文、减并发并留余量。
- cache fit 失败：检查 CPU 权重容量、碎片与活动模型保护；GPU 显存错误会走单独的 OOM 路径。
- 图 VMM layout 错误：检查存档地址和 tensor binding；显存预算与布局一致性需要分别排查。
