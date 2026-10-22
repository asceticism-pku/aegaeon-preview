# 系统架构

```mermaid
flowchart TD
    Client[应用客户端] --> API[FastAPI 兼容层]
    API --> Service[LLMService 路由与副本表]
    Service --> Ctrl[选中的物理节点 Controller]
    Ctrl --> PD[PrefillDispatcher]
    PD --> PE[PrefillEngine 与 Workers]
    PE -->|首个 token| Out[StepOutput 与响应]
    PE --> DD[DecodeDispatcher]
    DD --> DE[DecodeEngine 与 Workers]
    DE --> Out
    Ctrl --> Simple[可选 SimpleDispatcher / Engine]
    Simple --> Out
    Ctrl --- BM[节点级 BlockManager]
    BM --- CPU[节点共享 CPU KV Cache]
    PE --- PKV[Prefill：各 Worker 的 GPU KV 缓存]
    DE --- DKV[Decode：各 Worker 的 GPU KV 缓存]
    Simple --- SKV[Simple：各 Worker 的 GPU KV 缓存]
    BM -. 统一协调 .-> PKV
    BM -. 统一协调 .-> DKV
    BM -. 统一协调 .-> SKV
    Ctrl --- QC[QuickCache CPU 权重]
    QC --> PE
    QC --> DE
    QC --> Simple
```

Simple 与 P/D 是两种可选执行模式。每个物理节点有一个 Controller 和共享 CPU 缓存；各 Worker 分别持有自己的 GPU KV 缓存，由节点的 BlockManager 统一协调。

## 组件

- **API / LLMService：** 请求格式处理、副本放置与路由。
- **Controller / Scheduler：** 节点内分发、batch 和模型配额调度。
- **StageEngine / Worker：** 使用 vLLM 模型与内核执行 GPU 计算。
- **QuickCache：** CPU 模型权重缓存与快速加载。
- **BlockManager：** 请求 KV 状态及 CPU/GPU 搬运。

请求选定一个 READY 节点后，由 Prefill 产生首个 token，未结束的请求再进入 Decode。模型切换通过节点的 CPU 缓存保留 KV 状态。详见[调度](scheduling.md)和[内存管理](memory.md)。

## 源码入口

| 层 | 主要文件 |
|---|---|
| CLI / HTTP | cli.py、api.py |
| 控制面与集群 | llm.py |
| 引擎 / GPU 执行 | stage_engine.py、worker.py |
| 调度 | simple/prefill/decode dispatcher 与 scheduler |
| 权重加载 | loader/、quick_model_loader/ |
| KV 管理 | block_manager.py、cache_groups.py、cache_transfer.py、vllm_cache.py、仓库根目录 ops/ |
| 注册与预算 | config.py、model_registry.py、device_registry.py、estimator.py |
| 图恢复 | cuda_graph.py、graph_config.py、graph_bindings.py、foundry_runtime.py、foundry/ |
