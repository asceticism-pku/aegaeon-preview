# 基准测试与可复现评估

## 先定义测量口径

| 指标 | 推荐定义与注意事项 |
|---|---|
| TTFT | 请求发出/到达至首 token；客户端文本与后端 token 分开命名 |
| E2E | 请求开始至明确结束事件；写清 finish_reason 还是 [DONE] |
| TPOT | 多 token 输出中 Decode 平均 token 间隔；HTTP chunk 间隔属于另一项指标 |
| Throughput | 实际完成 tokens 或 requests / 明确的墙钟时间区间 |
| Makespan | 固定 workload 从开始到全部完成 |
| SLO / qos | 使用具体源码和公式定义，每种定义单独列示 |
| CV | population stddev/mean，说明引擎集合与 token计数范围 |

## 在线接口演示工具

```bash
python benchmark/demo_api.py --help
python benchmark/demo_api.py \
  --host 127.0.0.1 --port 8000 \
  --model-config /srv/aegaeon/config.yaml --arrival-rate 0.1 --duration 60
```

该工具连接已经启动的 API，结果会写到 benchmark/demo-output.json，重复运行会覆盖同一路径。它是示例负载工具；应先核对模型筛选、输入长度与参数，再用于实验。

## 当前历史脚本的边界

端到端重放脚本为 `benchmark/benchmark_e2e_from_workload.py`。

`benchmark/benchmark_e2e_from_workload.py` 的 Ray 地址固定为原实验集群，NodeConfig 的 P/D 和 cache 参数也在脚本中设置。使用前应改成当前集群地址和目标配置。

CSV 字段为 request_id、timestamp、input_tokens、output_tokens。脚本按 seed 随机分配模型，并从固定预编码 token 列表切片；超出列表长度的输入会变短，decode 长度还受 4096 规则截断。因此该脚本测量的是经过脚本转换的工作负载，报告中应使用实际 prompt 和输出长度。

## 推荐实验设计

1. 固定模型 checkpoint、tokenizer、完整 YAML、依赖和源码 revision。
2. 定义同一请求序列、真实 token长度、到达模型分布、并发和持续时间。
3. 单独记录 cold start、CPU cache命中、GPU切换、warm steady state。
4. 所有副本按实验定义一致预热；零预热实验则保持冷启动。
5. OFF/LOAD 或策略A/B 交替顺序，多次重复，记录P50/P95与样本数。
6. 核对输出token IDs、请求完成数、异常数、资源与退出清理。

## 专项工具

Work Stealing 使用 off/on 成对 smoke 与 compare_work_stealing_results.py；模型放置使用 model_placement_gpu_smoke.py，可比较 least-loaded / round-robin 和 busy-avoidance 路由。

Graph 工具有 benchmark_cuda_graph_latency.py、benchmark_foundry_three_model_latency.py 和 roundtrip tests；它们的默认路径及 archive 前提见脚本和 `tools/README_three_model_latency.md`。后者测量已启动服务、CPU 权重可用场景下的 GPU 真实切换；CPU cache miss 部署和完整启动成本需要单独测量。

## 性能声明规范

报告应列出 GPU/CPU/互联、驱动、torch/vLLM 版本、源码、模型、模式、预算、batch、真实长度、到达率、预热、样本数和统计方法。性能对比只使用正确性检查通过的样本，并随结果保留硬件、配置与测量条件。

## 模拟器

`sim/` 基于 SimPy，使用估计执行时间和 trace 研究调度策略，详见 `sim/README.md`。报告中请将仿真结果与 GPU 实测分开，并记录仿真采用的模型、设备和策略定义。
