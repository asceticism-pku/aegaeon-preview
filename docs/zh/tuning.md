# 性能调优流程

## 建立基线

固定TP=1、图关闭、单模型和明确预算，测实际token长度、TTFT、E2E、吞吐与CPU/GPU占用。将首个加载与后续稳态分开，保留原始响应及失败数量。

## 按瓶颈调整

| 现象 | 首先核查 | 可评估的调整 |
|---|---|---|
| TTFT高 | 排队、GPU首次加载、切换、Prefill长度 | 增Prefill或Simple资源、降低输入长度、调整到达率 |
| Decode间隔大 | 模型复用配额、切换、KV搬运 | 增加 Decode 资源、减少 active 模型竞争、重新采集对应 profile |
| 节点路由不均 | READY集合、outstanding和累计分配 | least-outstanding与round-robin成对对比 |
| CPU KV不足 | slab数量、实际并发和长度 | 增CPU KV或降低并发/长度 |
| 权重cache不足 | 模型参数、slice碎片、活动保护 | 增CPU权重cache或显式控制副本 |
| GPU OOM | current/prefetch权重、临时workspace、KV | 留预算余量、减少batch/上下文、检查模型适配 |
| Decode引擎空闲不均 | 是否有安全候选和空receiver | 在满足条件的P/D拓扑评估Work Stealing |
| 小 batch Decode 控制成本明显 | eager 基线与图实际命中 | token ID 一致后比较 OFF/LOAD |

## 几个容易误判的参数

提高 memory_utilization 会增加估计 KV 预算，同时压缩临时开销的安全空间。model_cache_size_gb 控制 CPU 权重缓存，GPU KV 容量由 memory_utilization 等 GPU 侧配置决定。nnodes 表示物理节点数。缩容时先 undeploy，再按目标 replica_count 重新部署。

增加 Engine 会增加 GPU 和 Ray CPU 需求。同一 Controller 的 Engine 共享按 slab 配置的 CPU KV 缓存；增加节点或并发时重新评估 pinned cache 容量。Simple 与 P/D 应使用目标负载分别测量，选择 E2E、TTFT、TPOT 和吞吐更符合目标的一种。

## 图模式

`batch_sizes` 应逐项覆盖实际 batch；运行时只匹配相同的 batch size。`seq_len_buckets` 当前要求一个元素；更大的最大长度会增加静态图输入和 metadata 预算。SAVE/LOAD 使用相同布局，修改 `memory_utilization` 后重新录制。

## Profile可信度

profile 应在目标硬件、驱动、频率和模型版本下采集；当前目录只按模型和设备区分，版本与频率需另行记录。调优前确认真实 batch 和长度分布位于采集范围。当前估计器只支持 TP=1。

## 变更验收

每次变更都核对输出 token、异常数、资源和停止清理。性能结论应来自独占 GPU 上的多轮重复测量，并同时报告样本数、分位数和波动范围。
