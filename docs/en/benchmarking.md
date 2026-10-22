# Benchmarking

## Metrics

| Metric | Meaning |
|---|---|
| TTFT | Request arrival to first token |
| TPOT | Average interval between Decode tokens |
| E2E | Request arrival to completion |
| Throughput | Completed requests or tokens per second |
| CV | Population standard deviation divided by the mean |

Use actual token counts; SSE text chunks can contain multiple tokens. The project's `qos` statistic uses a cumulative-time ratio.

## Tools

Run from the repository root against a configured service:

```bash
python benchmark/demo_api.py --help
python benchmark/demo_api.py --host 127.0.0.1 --port 8000 \
  --model-config /srv/aegaeon/config.yaml --model Qwen/Qwen3-4B
```

| Tool | Purpose |
|---|---|
| `benchmark/work_stealing_gpu_smoke.py` | Work Stealing workload |
| `benchmark/compare_work_stealing_results.py` | Compare saved Work Stealing results |
| `benchmark/model_placement_gpu_smoke.py` | Placement and routing workloads |
| `tools/benchmark_foundry_three_model_latency.py` | Model-switch latency with CUDA Graphs |
| `sim/` | CPU-only scheduling simulation; see `sim/README.md` |

Each tool exposes `--help` for arguments. Use matching [profiles](profiling.md) and distinguish startup, model switching, and steady-state measurements.

## Tuning

High TTFT points to queueing, loading, or Prefill capacity. Long Decode intervals point to model competition, transfers, or quota settings. Adjust engine counts and memory budgets for the workload; use [Work Stealing](work-stealing.md) for uneven Decode load and [CUDA Graphs](cuda-graphs.md) for launch overhead.
