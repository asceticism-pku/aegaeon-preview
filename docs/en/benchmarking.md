# Benchmarking

## Metrics

| Metric | Meaning |
|---|---|
| TTFT | Request arrival to first token |
| TPOT | Mean interval between output tokens after the first |
| E2E | Request arrival to completion |
| Throughput | Completed requests or generated tokens per second |
| CV | Population standard deviation divided by the mean |

Aegaeon's `qos` statistic uses a cumulative-time ratio based on TTFT and TPOT targets. The streaming demo below measures time to the first nonempty text chunk.

## Tools

The API demo sends requests to an existing service. Run it from the repository root:

```bash
python benchmark/demo_api.py --help
python benchmark/demo_api.py --host 127.0.0.1 --port 8000 \
  --model-config /srv/aegaeon/config.yaml --model Qwen/Qwen3-4B
```

| Tool | Purpose |
|---|---|
| `benchmark/work_stealing_gpu_smoke.py` | Run a local 1P+2D Work Stealing workload |
| `benchmark/compare_work_stealing_results.py` | Compare saved Work Stealing results |
| `benchmark/model_placement_gpu_smoke.py` | Run local placement and routing workloads |
| `tools/benchmark_foundry_three_model_latency.py` | Replay a three-model CUDA Graph experiment from an existing run |
| `sim/` | CPU-only scheduling simulation; see `sim/README.md` |

Each tool exposes `--help` for arguments. Use matching [profiles](profiling.md) and distinguish startup, model switching, and steady-state measurements.

## Tuning

High TTFT can reflect queueing, loading, or limited Prefill capacity. Long Decode intervals can reflect model competition, transfers, or quota settings. Check these causes before adjusting engine counts and memory budgets. [Work Stealing](work-stealing.md) addresses uneven Decode load; [CUDA Graphs](cuda-graphs.md) reduce launch overhead.
