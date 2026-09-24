# Scheduling and SLOs

## Token-level multiplexing

Long requests iterate repeatedly during Decode. Aegaeon can assign the GPU to other models without waiting for all requests of one model to finish, switching at batch-round and execution-quota boundaries. This schedules work on fixed engines; it does not create or delete a Ray actor for every token.

Higher switch frequency raises weight-loading, KV-transfer, and control overhead. Larger quotas increase wait time for other models. Schedulers use estimated execution time, switching costs, and SLOs to choose quotas. Measure actual TTFT and TPOT to determine SLO attainment.

## Simple mode

SimpleScheduler handles Prefill and Decode in the same engine. Decode quotas use predicted step time, switching time, and a maximum quota. Invalid estimates enter a fallback; normal scheduling still requires complete profiles for the target device.

## P/D mode

PrefillDispatcher implements fcfs-avgload and sjf-avgload paths. PrefillScheduler currently selects through `policy='uni'`. NodeConfig retains the related fields, while ServerConfig YAML does not map these options.

DecodeScheduler manages model batches, rounds/turns, and alpha/quotas. Work Stealing moves one complete batch at a safe boundary after a round; it does not replace the quota formula.

## SLO settings

```bash
AEGAEON_TTFT_SLO=3 AEGAEON_TPOT_SLO=0.03 \
  aegaeon start --config /srv/aegaeon/config.yaml --ray-address 127.0.0.1:6789
```

TTFT defaults to 10 seconds and TPOT to 0.1 seconds. utils reads them during import, so changes require restarting the relevant processes. LLMService's actor environment allowlist omits these variables; set identical values in every node process environment for multi-node and existing Ray deployments.

## Measurement and evaluation

TTFT includes queueing, model switching, Prefill, and required transfers. Decode intervals are also affected by other models. Client text-chunk timestamps and backend token StepOutput timestamps measure different things.

`compute_request_metrics()` defines qos as a cumulative time ratio, rather than the usual fraction of requests meeting both TTFT and TPOT thresholds. Incomplete or mismatched outputs have special return branches. Report it as the project's qos metric with its definition rather than renaming it standard SLO attainment.

## Policy scopes

| Policy | Scope / trigger |
|---|---|
| model_placement_policy | Selects CPU cache nodes during deploy |
| request_routing_policy | Selects a READY physical node for a new request |
| Prefill / Decode Scheduler | Chooses batches and model quotas within a node |
| Decode Work Stealing | An idle node-local Decode receiver takes an eligible batch |

Control these policies separately for interpretable A/B experiments; they are not interchangeable.
