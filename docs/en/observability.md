# Logs and Monitoring

## Logs

Set the log path before startup:

```bash
mkdir -p logs
export AEGAEON_LOG_FILE="$PWD/logs/server.log"
```

Ray actor logs are also available in its session log directory.

## Runtime interfaces

| Endpoint | Contents |
|---|---|
| `/health` | API liveness |
| `/v1/models` | Registered models |
| `/v1/aegaeon/runtime` | Nodes, engines, replicas, routing, caches, and Decode load |
| `/v1/aegaeon/events` | Runtime SSE events |
| `/v1/aegaeon/metrics` | Per-model access counters and rate limits |

```bash
curl -sS http://127.0.0.1:8000/v1/aegaeon/runtime
curl -N http://127.0.0.1:8000/v1/aegaeon/events
```

Use READY replica state to confirm model availability. Access metrics use a 60-second window; configure limits with `AEGAEON_MAX_REQUESTS_PER_SECOND` and `AEGAEON_MAX_TOKENS_PER_SECOND`.
