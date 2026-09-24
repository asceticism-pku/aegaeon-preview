# 调度与 SLO

## Token 级复用的含义

长请求在 Decode 阶段多次迭代。Aegaeon 在批次轮次和执行配额边界安排模型切换，让多个模型共享固定的一组引擎与 Ray actors。

切换频率越高，权重加载、KV 搬运和控制开销越大；配额越大，其他模型的等待时间越长。Scheduler 使用估计执行时间、切换成本和 SLO 计算配额。SLO 是否达成应从实际 TTFT 和 TPOT 结果统计。

## Simple

SimpleScheduler 在同一引擎完成 Prefill 与 Decode。Decode 配额计算使用预测 step time、切换时间与最大配额；数值无效时进入 fallback。正常调度仍要求完整的目标设备 profile。

## P/D

PrefillDispatcher 实现 fcfs-avgload 和 sjf-avgload 路径；PrefillScheduler 当前固定通过 `policy='uni'` 选择实现。NodeConfig 保留相关字段，ServerConfig YAML 未映射这些选项。

DecodeScheduler 按模型批次、round/turn、alpha/quota 管理执行。Work Stealing 在 round 结束后的安全边界转移一个完整批次，原有配额公式继续生效。

## SLO 设置

```bash
AEGAEON_TTFT_SLO=3 AEGAEON_TPOT_SLO=0.03 \
  aegaeon start --config /srv/aegaeon/config.yaml --ray-address 127.0.0.1:6789
```

TTFT 默认 10 秒，TPOT 默认 0.1 秒。utils 在 import 时读取它们；修改后需要重启相关进程。LLMService 的 actor 环境白名单未包含这两个变量，多节点和已有 Ray 集群应在每个节点的进程环境中设置相同值。

## 测量与评价

TTFT 包括排队、模型切换、Prefill 与必要传输；Decode token 间隔也会受其他模型执行影响。客户端按文本 chunk 计时，后端按单 token StepOutput 计时，报告时应注明口径。

`compute_request_metrics()` 的 qos 按源码定义为时间累计比例。它与“同时满足 TTFT 与 TPOT 阈值的请求比例”采用不同公式，且对输出缺失或数量不匹配设有专门分支。报告中应使用“项目 qos 指标”并附上公式定义。

## 策略作用域

| 策略 | 作用域 / 触发 |
|---|---|
| model_placement_policy | deploy 时选择 CPU 缓存节点 |
| request_routing_policy | 新请求选择 READY 物理节点 |
| Prefill / Decode Scheduler | 节点内选择执行批次与模型配额 |
| Decode Work Stealing | 节点内空闲 Decode receiver 接收安全批次 |

这四种策略分别作用于不同阶段。A/B 实验应一次只调整其中一项，并固定其余配置。
