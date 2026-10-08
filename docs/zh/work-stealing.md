# Decode Work Stealing

这是默认关闭的可选节点内负载均衡。启用后，同一 Controller 的空闲 Decode Engine 可以接收另一个 Scheduler 中等待执行的完整批次。

## 启用

```yaml
server:
  num_engines: 0
  num_prefill_engines: 1
  num_decode_engines: 2
  decode_work_stealing: true
  decode_work_stealing_min_batch_gap: 2
```

至少需要两个 Decode Engine 才有实际转移对象；gap 必须>=2。

## 转移条件

donor 在 round 完成后最多尝试一次。receiver 的批次队列必须为空，候选为队列中 head 之后、`start_time is None`、KV 逻辑驻留 CPU 的完整 BatchedRequests。`start_time` 每次 iteration 完成后都会清空，此条件表示当前无执行中的 iteration，已经生成过 token 的等待批次也符合该条件。优先选择与 receiver resident model 相同的候选。

转移前重新检查 owner、队列、负载差与 KV 状态。关键队列变更采用连续的同步操作；检查失败时队列保持原状。候选范围限于等待执行的批次，转移范围限于同一 Controller 和物理节点。

## 指标

runtime 的 work_stealing 包含 `enabled`、`min_batch_gap`、`attempts`、`successes` 和 `rejections`；decode_load 包含每 Engine 的 `output_tokens`、`active_seconds`、`decode_steps`、`rounds_completed`、`turns_completed`、`model_switches`、转移计数和当前队列负载，以及 CV。

Decode output_tokens 从 Prefill 生成的第一个 token 之后开始计数。active_seconds 包括 dispatch 到 Worker 返回的完整墙钟开销；纯 kernel 时间需用 GPU profiler 单独采集。CV 为总体标准差/平均值；输入为空或平均值为 0 时实现返回 0。解释负载均衡效果前应确认计数已有实际工作量，比较时要确保引擎集合与 workload 相同。

## 成对验证

```bash
CUDA_VISIBLE_DEVICES=0,1,2 python benchmark/work_stealing_gpu_smoke.py \
  --model-paths /srv/models/model-a /srv/models/model-b \
  --work-stealing off --result-file /tmp/ws-off.json
CUDA_VISIBLE_DEVICES=0,1,2 python benchmark/work_stealing_gpu_smoke.py \
  --model-paths /srv/models/model-a /srv/models/model-b \
  --work-stealing on --result-file /tmp/ws-on.json
python benchmark/compare_work_stealing_results.py /tmp/ws-off.json /tmp/ws-on.json
```

model-a/model-b 是占位路径。脚本的 `build_config()` 固定按 A100 80 GiB、两个模型参数量分别为 `0.6 × 2^30` 和 `7.1 × 2^30` 生成预算；默认 profile 均为 `qwen2_5_7b`，可用 `--profiles` 和 `--device-profile` 显式选择。默认 prompt 为 16384 tokens，模型上下文还需覆盖输出长度。替换模型或设备时，应先核对并调整脚本中的参数量、设备容量和内存预算，准备对应性能数据。两次测试应顺序运行并独占 GPU；比较工具会检查请求 ID、模型名、输入与输出长度、Engine 集合及 Decode token 守恒。

## 局限

Work Stealing 在出现批次队列为空的 Decode Engine 时触发，每次移动一个完整的等待批次。跨模型移动会增加一次模型切换成本。配对测试使用构造负载；吞吐量、时延和负载均衡效果需要以保留的原始结果评价。
