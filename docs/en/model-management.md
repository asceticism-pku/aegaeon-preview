# Dynamic Deployment, Placement, and Unloading

## Cached replica states

A node replica means the QuickCache entry and service registration for `(model, physical Ray node)`, not a permanently GPU-resident weight copy on each engine.

```mermaid
stateDiagram-v2
    [*] --> ABSENT
    ABSENT --> LOADING: deploy
    LOADING --> READY: success
    LOADING --> FAILED: failure
    READY --> DRAINING: undeploy
    DRAINING --> ABSENT: unload succeeds
    FAILED --> LOADING: retry
```

The diagram shows the main lifecycle. Only READY replicas receive new requests. DRAINING replicas reject new requests and continue work that was already submitted.

## Automatic placement

least-loaded ranks nodes using cache layouts, evictable inactive models, estimated eviction costs, request loads, free cache, and cached-model counts. Node IDs break ties deterministically. This policy does not merely compare GPU utilization.

QuickCache fit considers allocator slices, so fragmentation affects placement. Load is the maximum of Controller active count and API outstanding reservations, covering requests routed but not yet submitted without counting them twice.

`replica_count` is a minimum target. Reducing it after reaching the target does not automatically remove excess replicas. Explicit `node_ids` bypass automatic selection. Placement runs on deploy calls rather than continuously autoscaling node counts.

```yaml
server:
  model_placement_policy: least-loaded
  request_routing_policy: least-outstanding
```

Both policies also support round-robin for comparisons. Unknown policy names fail startup.

## Request routing

least-outstanding selects the READY node with the fewest unfinished requests for that model; equal loads rotate deterministically using the internal request ID. round-robin ignores load and selects by request ID modulo the READY node count.

One physical node owns the entire Prefill and Decode lifecycle of a request. Live requests do not migrate across physical nodes. Node-local Decode Work Stealing is a separate mechanism.

## Eviction and failures

When capacity is insufficient, deployment selects removable cached models with no active requests. The response `evicted` field and runtime state record the actual removals.

Multi-node deployment can partially succeed. Complete failure without an existing READY replica returns 507. Operations for each model are serialized; different models can be operated on independently but still share node caches.

Unloading first marks a replica DRAINING. Active requests or API reservations produce a rejection result; run undeploy again after those requests finish. Before replacing model files, changing the path for an existing name, or updating archives, drain and unload the replica, then rerun the model acceptance procedure after deployment.

## Start an empty service

```yaml
server:
  startup_models: []
```

Empty startup still initializes Engines, CPU KV, and QuickCache. With model_cache_size_gb=0 and no startup models, the Controller creates a 20 GiB weight cache; later deployments do not grow it without limit. Foundry LOAD requires complete configured-model archives and differs from ordinary empty startup.

## Operational checks

```bash
curl -sS http://127.0.0.1:8000/v1/aegaeon/runtime
```

Inspect model_placements, last_model_placement_decision, model_placement_stats, request_routing, and node cache/request statistics. Model listing reflects API registration alone and does not guarantee routability on a node.

Implementation details are available in `docs/model-placement.md`.
