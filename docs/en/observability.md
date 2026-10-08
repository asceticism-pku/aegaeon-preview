# Logs, Monitoring, and Health Checks

## Logs

```bash
mkdir -p logs
AEGAEON_LOG_FILE="$PWD/logs/server.log" \
  aegaeon start --config /srv/aegaeon/config.yaml --ray-address 127.0.0.1:6789
```

The default log file is output.log. With multiple processes and nodes, identify which machine owns each path. There is no general built-in log rotation setting; use process management and log collection tools to manage storage.

## Runtime snapshots

```bash
curl -sS http://127.0.0.1:8000/v1/aegaeon/runtime
```

Main top-level fields are timestamp, model_placement_policy, request_routing_policy, last_model_placement_decision, model_placement_stats, model_placements, request_routing, engines, events, work_stealing, decode_load, and nodes.

request_routing tracks API reservations that are assigned but incomplete. nodes.request_load reports Controller-observed active requests, sampled at a different time. cumulative_assignments is a historical total; use outstanding or active-request fields for current request load, including queued and executing requests.

runtime queries Controllers in parallel. Each node and the API still sample their fields independently, so the response is not a synchronized atomic snapshot. Cross-node wall times use each node's system clock. A Controller RPC failure causes the runtime request to return an error.

## Runtime SSE

```bash
curl -N http://127.0.0.1:8000/v1/aegaeon/events
```

The event name is runtime and data contains engines/events. Each Controller query is followed by a 0.25-second wait. New events or engine-state changes send data; about 15 seconds without updates sends keep-alive. A new connection starts from the events currently retained by each Controller and then uses connection-local cursors. Durable history and `Last-Event-ID` recovery require an external observation system.

## Access metrics and rate limits

metrics returns timestamp, window_seconds, limits, and metrics. Statistics are per model, with a default 60-second window. Configure limits through:

```bash
export AEGAEON_MAX_REQUESTS_PER_SECOND=10
export AEGAEON_MAX_TOKENS_PER_SECOND=1000
```

Limits apply to each model's average rate over the last 60 seconds: requests / 60 and recorded generated tokens / 60, rather than an independent quota for each second. Admission checks include the pending request in the estimated request rate. Tokens are recorded when the API obtains a final result normally; an early stream disconnect can leave subsequent generated tokens unrecorded. Check and record are separate operations. Accounting follows model resolution and admission checks but precedes generation-parameter validation, so invalid generation parameters also count; model-resolution failures and rate-limit rejections do not. Each API process keeps independent statistics.

429 rejects requests at admission; it does not immediately cancel every in-flight request when token rates exceed a threshold.

## GPU power

gpu-power queries nvidia-smi on the API host and does not aggregate remote Ray nodes. Sampled power is not per-request energy or proof of process GPU utilization. See the [power guide](energy.md).

## Health-check levels

`/health` only returns status=ok. External monitoring can check HTTP, runtime node/Engine state, and occasional short real inference separately. Real inference consumes resources, so use a confirmed short request. The repository does not currently provide a complete deep-readiness endpoint.
