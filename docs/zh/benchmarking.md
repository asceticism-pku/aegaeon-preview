# 基准测试

## 指标

| 指标 | 含义 |
|---|---|
| TTFT | 请求到达到首个 token 的时间 |
| TPOT | Decode token 的平均输出间隔 |
| E2E | 请求到达到完成的时间 |
| 吞吐量 | 每秒完成的请求数或 token 数 |
| CV | 总体标准差除以均值 |

使用实际 token 数；一个 SSE 文本 chunk 可包含多个 token。项目的 `qos` 统计采用累计时间比值。

## 工具

在仓库根目录，对已配置的服务运行：

```bash
python benchmark/demo_api.py --help
python benchmark/demo_api.py --host 127.0.0.1 --port 8000 \
  --model-config /srv/aegaeon/config.yaml --model Qwen/Qwen3-4B
```

| 工具 | 用途 |
|---|---|
| `benchmark/work_stealing_gpu_smoke.py` | Work Stealing 负载 |
| `benchmark/compare_work_stealing_results.py` | 比较已保存的 Work Stealing 结果 |
| `benchmark/model_placement_gpu_smoke.py` | 放置与路由负载 |
| `tools/benchmark_foundry_three_model_latency.py` | CUDA Graph 模型切换延迟 |
| `sim/` | CPU 调度仿真，见`sim/README.md` |

各工具通过 `--help` 提供参数说明。使用匹配的 [profile](profiling.md)，区分启动、模型切换和稳态测量。

## 调优

TTFT 较高时检查排队、加载和 Prefill 容量；Decode 间隔较长时检查模型竞争、搬运和配额。根据负载调整 Engine 数量和内存预算，Decode 负载不均可使用 [Work Stealing](work-stealing.md)，提交开销较高可使用 [CUDA Graph](cuda-graphs.md)。
