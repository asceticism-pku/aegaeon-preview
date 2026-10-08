# Dynamic Model Placement

## Scope

Dynamic model placement removes the previous one-model/one-node routing rule.
The API server maintains a control-plane table from model name to physical Ray
nodes, and a model can have multiple independently managed QuickCache replicas.

`node_0`, `node_1`, and so on identify physical Ray nodes. Multiple GPUs in one
machine remain one node and share one Controller and one QuickCache.

## Replica lifecycle

Each `(model, node)` pair has one state:

- `ABSENT`: no replica is recorded on the node.
- `LOADING`: QuickCache deployment is in progress.
- `READY`: the replica is eligible for new requests.
- `DRAINING`: unload was requested; new requests are blocked while existing
  work drains.
- `FAILED`: deployment failed and the error is retained for inspection.

Only `READY` replicas participate in request routing. A request is assigned to
one physical node for its complete Prefill and Decode lifetime.

## Request routing

For a model with multiple `READY` replicas, the API process reserves the node
with the fewest outstanding routed requests across all models. Equal-load nodes use the request
ID as a deterministic rotating tie-breaker, so an idle replica set is filled
evenly without a shared round-robin counter.

This default is implemented by `LeastOutstandingRequestsPolicy` behind the
`RequestRoutingPolicy` interface. `LLMService` accepts an alternative policy at
construction time, allowing later routing experiments without changing request
lifecycle bookkeeping.

The API server selects the policy from YAML:

```yaml
server:
  request_routing_policy: least-outstanding
```

Set the value to `round-robin` for a deterministic, load-oblivious control
policy. It uses `request_id % ready_replica_count`, matching the previous
multi-replica routing behavior while retaining the same lifecycle accounting.

Unknown policy names fail during startup instead of silently falling back.

The reservation is local to `LLMService`, so node selection stays within the
API process. It is released after the remote request
finishes, including exceptional completion. For a disconnected streaming
client, a background cleanup task keeps the reservation until the Controller
has actually finished and released the request.

The selected node owns the complete Prefill and Decode lifetime, preserving
node-local KV cache and scheduler state.

## Automatic placement

The API server selects the placement policy from YAML:

```yaml
server:
  model_placement_policy: least-loaded
```

Set it to `round-robin` for a load- and cache-oblivious cyclic baseline.
Unknown policy names fail during startup.

When `node_ids` is omitted, `LeastLoadedPlacementPolicy` ranks candidate nodes
by:

1. the model fits in the current QuickCache layout without eviction;
2. if eviction is required, whether enough inactive models can be evicted;
3. fewer estimated bytes and then fewer models evicted;
4. fewer active requests;
5. more free QuickCache bytes;
6. fewer cached models;
7. node ID as a deterministic tie-breaker.

The load term is the maximum of the Controller's active-request count and the
API-side outstanding reservation count. This covers requests that have been
routed but are not yet visible to the remote Controller without counting the
same in-flight request twice when the two counts describe the same requests. The maximum is a snapshot heuristic; different request sets can make it lower than their union. Both source values and the effective value are
included in the latest placement decision snapshot.

The fit check uses the node-local QuickCache allocator and safetensors slice
layout, so fragmentation is included rather than inferred from aggregate free
bytes alone.

Eviction cost follows the existing QuickCache eviction order, skips models with
active requests, and simulates freeing their exact allocator slices in a copy
of the current memory table until the target slice layout fits. If the slices
still do not fit after removable models are exhausted, enough aggregate free
bytes makes the estimate eligible for the Controller's defragmentation path.
It is a placement-time estimate; the deployment response reports the models
actually evicted.

`replica_count` is a minimum. Existing `READY` replicas are retained, and the
policy selects enough additional nodes to reach the requested count. Scaling
down uses an explicit unload operation.

Explicit `node_ids` bypass automatic placement. This provides a stable control
for policy experiments.

## Commands

Automatic two-replica deployment:

```bash
aegaeon deploy Qwen/Qwen2.5-7B --replicas 2
```

Explicit placement:

```bash
aegaeon deploy Qwen/Qwen2.5-7B --nodes node_0 node_2
```

Remove one replica:

```bash
aegaeon undeploy Qwen/Qwen2.5-7B --nodes node_2
```

Remove every known replica:

```bash
aegaeon undeploy Qwen/Qwen2.5-7B
```

The equivalent HTTP request is:

```json
{
  "model": "Qwen/Qwen2.5-7B",
  "replica_count": 2
}
```

Send it to `POST /v1/models/deploy`. `GET /v1/aegaeon/runtime` reports the
current placement policy, the most recent placement decision and its candidate
metrics, replica states, node cache usage, and request load. In particular:

- `model_placement_stats` reports cumulative deployment attempts, successes,
  failures, evictions, and successful unloads by node and logical model;

- `request_routing.total` and `request_routing.by_node` report API-side
  requests that have been assigned but have not finished;
- `request_routing.cumulative_assignments` reports first-time node assignments
  since this API service started and is not decremented when requests finish;
  its `by_model` view separates the node distribution of each logical model;
- `nodes[].request_load` reports requests currently tracked by each remote
  Controller.

The two views are intentionally separate: the former drives replica routing,
while the latter describes execution state observed on the physical node.

## Single-host logical-node smoke test

The manual smoke test represents several logical Controllers on one physical Ray node while giving
each Controller an isolated GPU and shared-memory directory. For four logical
nodes on physical GPUs 4--7, run:

```bash
CUDA_VISIBLE_DEVICES=4,5,6,7 python benchmark/model_placement_gpu_smoke.py \
  --model-path /path/to/Qwen2.5-7B-Instruct \
  --gpus 4,5,6,7
```

The balanced scenario deploys one replica per logical node, routes one request
to every replica, verifies runtime events, unloads replicas, and cleans its
temporary Ray and shared-memory directories in `finally`. It requires real
weights and one free GPU per engine. `build_config()` fixes the primary model
parameter count at `7.1 × 2^30` and the device at A100 80 GiB. Before using other
models or GPUs, adjust these values and the memory budgets and supply matching
performance profiles.

To exercise two logical P/D nodes with one Prefill and one Decode GPU each:

```bash
CUDA_VISIBLE_DEVICES=4,5,6,7 python benchmark/model_placement_gpu_smoke.py \
  --engine-mode pd \
  --model-path /path/to/Qwen2.5-7B-Instruct \
  --gpus 4,5,6,7
```

In P/D mode consecutive GPU pairs form nodes: the example maps `node_0` to
GPUs 4/5 and `node_1` to GPUs 6/7.

To verify that a new request avoids a replica with an active long request, add:

```bash
--routing-scenario busy-avoidance --busy-output-tokens 64
```

This mode waits until `node_0` reports the first request as active before
routing the second request, which must select idle `node_1`.

For an A/B control run on the identical request IDs, add
`--request-routing-policy round-robin`. The second request has ID 2, so the
control selects busy `node_0`, while `least-outstanding` selects idle `node_1`.
This is a routing-correctness comparison, not a throughput benchmark: sending
the second request adds a cold model-switch cost when it reaches a previously
unused engine. Performance evaluation should warm every replica first and use repeated
requests rather than interpreting this one-shot smoke latency.

Add `--warm-replicas` to run one direct warm-up request on every replica before
the measured requests. The result JSON reports `warmup_seconds` and the
per-request `request_e2e_seconds` separately from the overall inference time.
Use `--repetitions N` with `busy-avoidance` to run `N` sequential long/short
request pairs. The JSON then includes mean, P50, and nearest-rank P95 latency
for each request role under `request_latency_summary`.

To compare automatic placement while holding cache state constant, deploy a
second real model during an active request with:

```bash
--placement-probe-model-path /path/to/Qwen3-0.6B \
--model-placement-policy least-loaded
```

The primary model is cached on both logical nodes and request ID 0 keeps
`node_0` busy. If both nodes fit the probe with equal eviction cost,
`least-loaded` selects idle `node_1`, while the `round-robin` control starts
at `node_0`. The probe scenario asserts this selection, so its model and cache
budgets must satisfy that condition. The result includes the candidate
metrics, selected node, lifecycle counters, and deployment duration.

## Failure semantics

Multi-node deployment is allowed to partially succeed. Successful replicas
become `READY`; failed replicas become `FAILED`, and the response identifies
each failure. If every target fails and no existing `READY` replica remains,
the API returns HTTP 507.

Deployment and unload operations are serialized per model. Concurrent
deployments therefore reuse the replica completed by the first operation,
while lifecycle operations for different models can still proceed in parallel.

Unload first marks replicas `DRAINING`. If a node refuses unload because work is
still active, it remains non-routable and the operation can be retried after the
work completes. API-side routing reservations also protect the short interval
between node selection and Controller submission: a replica with such a
reservation is not remotely unloaded, and the operation reports a retryable
failure until the reservation is released.

## Current scope

- Request routing compares outstanding request counts. Predicted token and
  batch processing costs are outside this policy. Each API process maintains
  an independent routing table and reservation set.
- Routing selects a node when the request arrives, and that node owns Prefill
  and Decode for the request lifetime.
- Placement uses a point-in-time snapshot; concurrent operations for different
  models can make decisions from overlapping snapshots.
- QuickCache retains its existing local eviction order. Automatic placement
  estimates that order and eviction volume; model value and future access
  probability stay outside the score.
- Single-host logical nodes validate correctness and GPU isolation, but a real
  multi-node performance claim requires multiple physical Ray nodes.
