# 调度与 SLO

Aegaeon 在 batch 轮次和执行配额边界切换模型，结合[性能 profile](profiling.md)在切换成本与 token 时延之间分配配额。

Simple 模式在同一 Engine 中调度 Prefill 和 Decode。P/D 模式使用独立资源池：Prefill 按到达顺序和负载分发，Decode 调度模型 batch 与 token 配额。

## SLO 设置

启动前在 API 和各 Ray 节点环境中设置：

```bash
export AEGAEON_TTFT_SLO=3
export AEGAEON_TPOT_SLO=0.03
```

单位为秒，TTFT 默认 10 秒，TPOT 默认 0.1 秒。修改后重启相关进程。

## 策略作用域

| 策略 | 作用域 |
|---|---|
| `model_placement_policy` | 部署时选择副本节点 |
| `request_routing_policy` | 为新请求选择 READY 节点 |
| Prefill/Decode 调度 | 在节点内选择 batch 和模型配额 |
| [Decode Work Stealing](work-stealing.md) | 在节点内均衡等待中的 Decode batch |

指标定义和工具见[基准测试](benchmarking.md)。
