# 术语与FAQ

| 术语 | 本项目含义 |
|---|---|
| Prefill | 消费prompt，建立状态，通常产生首token |
| Decode | 自回归后续token迭代 |
| P/D | Prefill/Decode分离模式 |
| Simple | 一个Engine承担两阶段 |
| Controller | 一个物理Ray节点上的控制actor |
| Engine | 调度/执行单位，管理一个或多个Workers |
| Worker | GPU执行actor |
| QuickCache | CPU权重缓存 |
| KV Cache | 请求级attention K/V或相关模型状态 |
| READY replica | 可接收新请求的节点缓存副本；GPU 权重按执行生命周期加载 |
| Reservation | API侧已分配未完成请求记账 |
| Round / Turn / Quota | Scheduler批次循环、某次执行段与配额 |
| TTFT | Time To First Token，必须注明观测层 |
| TPOT | Time Per Output Token，注意排除首token与计时口径 |
| HOL blocking | 前面的工作阻塞后续工作 |
| CV | 总体标准差/平均值；空集合或均值为 0 时，当前实现返回 0 |
| Foundry SAVE/LOAD | 录制/恢复图、设备代码及确定性布局 |
| GiB | 2^30 bytes，代码中若干gb字段实际上用此单位 |

## Aegaeon能直接当vLLM server替换吗？

接入范围以当前接口表和贪心参数为准；函数 tools 仅覆盖文档所列、且具备匹配 vLLM parser 的模型。上游的完整采样、Responses 等能力需要分别适配。

## 为什么一台多GPU机器仍nnodes=1？

node代表物理Ray节点，Engine数表达单机GPU执行实例；节点共享Controller和QuickCache。

## READY后第一次请求为什么仍慢？

CPU 权重缓存部署完成后，GPU 加载、初始化和图恢复仍在后续阶段执行。

## CUDA Graph是否默认开启？

默认 off。SAVE/LOAD 使用 P/D Decode、TP=1、Foundry hook 和单长度桶；LOAD 还需要匹配的 archive。

## 能原生在Windows或CPU运行吗？

主实现面向 Ubuntu/CUDA，依赖 `.so`、`/dev/shm` 和原生 CUDA/Rust 扩展。当前运行平台范围为 Linux GPU。

## 文档中的性能数字是否保证能复现？

性能取决于模型、硬件、负载和执行配置。复现实验时应使用相同条件，并区分首次加载、模型切换和稳态推理。
