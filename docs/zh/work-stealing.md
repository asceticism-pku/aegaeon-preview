# Decode Work Stealing

Work Stealing 在同一个 Controller 内均衡 Decode batch，默认关闭，要求至少两个 Decode Engine。

## 启用

```yaml
server:
  num_engines: 0
  num_prefill_engines: 1
  num_decode_engines: 2
  decode_work_stealing: true
  decode_work_stealing_min_batch_gap: 2
```

## 转移规则

一轮调度结束后，donor 可向队列为空的 Engine 转移一个完整的等待 batch。候选 batch 没有正在执行的 iteration，KV 状态位于 CPU；两端 batch 数差至少为 2，优先选择已驻留相同模型的接收端。

转移范围限于单个物理节点，沿用原有的 Decode 配额策略。

## 监控

`GET /v1/aegaeon/runtime` 中，`work_stealing` 提供转移计数，`decode_load` 提供各 Engine 的工作量和队列指标。接口用法见[日志与监控](observability.md)。
