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

donor 在 round 完成后最多尝试一次。receiver 必须完全空闲，候选为队列中 head 之后、`start_time is None`、KV 逻辑驻留 CPU 的完整 BatchedRequests。优先选择与 receiver resident model 相同的候选。

转移前重新检查 owner、队列、负载差与 KV 状态。关键队列变更采用连续的同步操作；检查失败时队列保持原状。候选范围限于等待执行的批次，转移范围限于同一 Controller 和物理节点。

## 指标

runtime 的 work_stealing 包含 enabled、gap、attempts、successes 和 rejection；decode_load 包含每 Engine tokens、active_seconds、steps、rounds、turns、switches 与转移计数，以及 CV。

Decode output_tokens 从 Prefill 生成的第一个 token 之后开始计数。active_seconds 包括 dispatch 到 Worker 返回的完整墙钟开销；纯 kernel 时间需用 GPU profiler 单独采集。CV 为总体标准差/平均值；比较时要确保引擎集合与 workload 相同。

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

model-a/model-b 是占位路径，请换成满足脚本前提的真实兼容模型。两次测试应顺序运行并独占 GPU；比较工具会检查 workload、Engine 集合、输出长度与 token 守恒。

## 局限

Work Stealing 在出现完全空闲的 Decode Engine 时触发，每次移动一个完整的等待批次。跨模型移动会增加一次模型切换成本。`docs/work-stealing.md` 中的数据来自其注明的小型构造负载。
