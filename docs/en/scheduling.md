# Scheduling and SLOs

Aegaeon switches models at batch-round and execution-quota boundaries. Quotas balance model-switch cost against token latency using [performance profiles](profiling.md).

Simple mode schedules Prefill and Decode in one engine. P/D mode uses separate engine pools; Prefill dispatches by arrival/load, while Decode schedules model batches and token quotas.

## SLO settings

Set these variables in the API and all Ray node environments before startup:

```bash
export AEGAEON_TTFT_SLO=3
export AEGAEON_TPOT_SLO=0.03
```

Values are seconds. Defaults are 10 seconds for TTFT and 0.1 seconds for TPOT. Restart the relevant processes after changes.

## Policy scopes

| Policy | Scope |
|---|---|
| `model_placement_policy` | Selects replica nodes at deployment |
| `request_routing_policy` | Selects a READY node for a new request |
| Prefill/Decode scheduling | Selects batches and model quotas within a node |
| [Decode Work Stealing](work-stealing.md) | Balances waiting Decode batches within a node |

For metric definitions and tools, see [Benchmarking](benchmarking.md).
