# CLI Reference

setup.py registers `aegaeon=aegaeon.cli:main`; `python -m aegaeon` is also available. Commands are start, deploy, and undeploy.

## start

```bash
aegaeon start --config /srv/aegaeon/config.yaml --host 127.0.0.1 --port 8000 --ray-address 127.0.0.1:6789
```

| Option | Parser default | Meaning |
|---|---|---|
| --config | Required | YAML file |
| --host | 0.0.0.0 | Bind address |
| --port | 8000 | Port |
| --models | None | Comma-separated startup models; overrides startup_models |
| --num-prefill-engines | 1 | Normal startup uses YAML ServerConfig |
| --num-decode-engines | 1 | Same |
| --num-engines | 0 | Same |
| --tensor-parallel-size | None | Explicit non-null value overrides YAML |
| --model-cache-size-gb | 20 | Fallback CPU weight capacity in GiB |
| --cpu-num-slabs | 15 | Fallback slab count |
| --cpu-slab-size-gb | 1 | Fallback size in GiB |
| --ray-address | None | Cluster address, ahead of environment variables |

Read [configuration precedence](configuration.md) first. This guide uses the `aegaeon` CLI. `python -m aegaeon.api` has a separate parser and defaults TP to 1.

## deploy

```bash
aegaeon deploy Qwen/Qwen3-4B --path /srv/models/Qwen3-4B --replicas 2
aegaeon deploy Qwen/Qwen3-4B --nodes node_0 node_1
```

| Option | Default | Meaning |
|---|---|---|
| model | Required positional argument | Service model name |
| --path | None | Local weights; otherwise registered path, then download |
| --host | localhost | API host |
| --port | 8000 | API port |
| --max-model-len | None | Context override |
| --replicas | 1 | Minimum READY replica target for automatic placement |
| --nodes | None | Explicit node IDs; bypass automatic placement |
| --tool-parser | None | vLLM tool-parser override for this model alias |

The client uses HTTPConnection with a 300-second timeout. After HTTP 200, inspect JSON `status`, `failed`, and `ready_nodes` to distinguish complete success from partial success.

## undeploy

```bash
aegaeon undeploy Qwen/Qwen3-4B --nodes node_1
aegaeon undeploy Qwen/Qwen3-4B
```

| Option | Default | Meaning |
|---|---|---|
| model | Required positional argument | Service model name |
| --host | localhost | API host |
| --port | 8000 | API port |
| --nodes | All known replicas | Unload only the listed nodes |

Busy replicas enter DRAINING and appear as partial failures; run undeploy again after their requests finish. Use runtime replica and cache state to confirm release.

## Queries

```bash
aegaeon --help
aegaeon start --help
aegaeon deploy --help
aegaeon undeploy --help
```

The CLI currently provides `start`, `deploy`, and `undeploy`. Query status and lists through HTTP, run benchmark/profile repository scripts, and stop the service through the process manager.
