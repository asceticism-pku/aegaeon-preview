# Model Management

Aegaeon deploys node-level CPU weight-cache replicas and routes requests among them. Each request stays on one physical node for its Prefill and Decode lifecycle.

## Replica states

| State | Meaning |
|---|---|
| ABSENT | No replica |
| LOADING | Deployment in progress |
| READY | Accepts new requests |
| DRAINING | Rejects new requests; retry unloading after active requests finish |
| FAILED | Deployment failed |

## Deployment and routing

```bash
aegaeon deploy Qwen/Qwen3-4B --replicas 2
aegaeon undeploy Qwen/Qwen3-4B --nodes node_1
```

`replica_count` sets a minimum target; use undeploy to remove replicas. Explicit `node_ids` select nodes directly.

`model_placement_policy: least-loaded` selects nodes using cache capacity, eviction cost, and request load. `request_routing_policy: least-outstanding` selects a READY node with the fewest outstanding requests. Both accept `round-robin` as an alternative.

## Operation results

Deploy and undeploy return `status`, per-node results, and `failed`. HTTP 200 can contain `status: partial`; check this field before treating the operation as complete.

Unloading marks replicas DRAINING. Retry after active requests finish. An empty service starts with `server.startup_models: []`; later deployments use its configured cache budget.

Inspect placements and routing with `GET /v1/aegaeon/runtime`. Command options are in the [CLI reference](cli.md); HTTP requests are in the [API reference](api.md).
