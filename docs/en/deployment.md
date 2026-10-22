# Deployment

A physical Ray node runs one Controller and shares its CPU caches among engines. Each engine uses one GPU; `nnodes` counts physical nodes.

## Execution modes

| Mode | Server settings |
|---|---|
| Simple | `num_engines > 0`; both P/D counts are 0 |
| P/D | `num_engines: 0`; at least one Prefill and one Decode Engine |

Use the [configuration examples](examples/README.md) for single-GPU, P/D, or multi-node layouts. Prepare model files and profiles on every node.

## Ray cluster

For a single host, pass `--ray-address local`. For two hosts, start Ray on each:

```bash
# Head node
ray start --head --port=6789 --resources='{"node_0":1}'
```

```bash
# Worker node
ray start --address=HEAD_IP:6789 --resources='{"node_1":1}'
```

Then start the API from the head node with `server.nnodes: 2` in its configuration:

```bash
aegaeon start --config /srv/aegaeon/config.yaml --ray-address HEAD_IP:6789
```

## Resources

Set GPU visibility consistently on each node. Allocate Ray CPUs for each Worker's `worker_num_cpus` and the Controller; use `ray status` to inspect resources.

Set [SLO variables](scheduling.md#slo-settings) on all nodes before starting Ray. Independent services need isolated Ray instances and shared-memory directories. Stop the foreground service to disconnect its driver; manage an existing Ray cluster separately.
