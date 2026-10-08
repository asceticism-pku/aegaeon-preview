# Single-Node, Multi-GPU, and Multi-Node Deployment

## Three resource levels

A physical Ray node has a Controller. An Engine is a scheduling and execution unit. A Worker is a Ray actor that occupies a GPU. Aegaeon supports TP=1 only, so one Engine maps to one Worker and one GPU.

GPU requirements per node are `num_engines` for Simple and `num_prefill_engines + num_decode_engines` for P/D.

A host with four GPUs still uses nnodes=1; num_engines=4 creates four Simple Engines. nnodes=4 means four physical nodes, requiring Ray custom resources node_0 through node_3.

## Simple mode

```yaml
server:
  nnodes: 1
  num_engines: 2
  num_prefill_engines: 0
  num_decode_engines: 0
  tensor_parallel_size: 1
```

One engine handles both request stages on one GPU. Multi-model scheduling switches weights on that engine and swaps KV state when required.

## P/D separation

```yaml
server:
  nnodes: 1
  num_engines: 0
  num_prefill_engines: 1
  num_decode_engines: 1
  tensor_parallel_size: 1
```

1P+1D needs two available GPUs. CPU KV transfers data between Prefill and Decode within the node. 1P+2D uses three GPUs and can run Work Stealing. This transfer path is node-local.

## Two physical nodes

Install identical dependencies and code on all nodes, prepare models at the same paths with matching profiles, and verify network and port reachability. Replace HEAD_IP below with the actual reachable address:

```bash
# Head node
ray start --head --port=6789 --num-cpus="$(nproc)" --resources='{"node_0": 1}'
# Worker node
ray start --address=HEAD_IP:6789 --num-cpus="$(nproc)" --resources='{"node_1": 1}'
```

```yaml
server:
  nnodes: 2
  num_engines: 1
  num_prefill_engines: 0
  num_decode_engines: 0
  tensor_parallel_size: 1
```

```bash
aegaeon start --config /srv/aegaeon/config.yaml --ray-address HEAD_IP:6789 --host 127.0.0.1
```

Engine settings apply uniformly to every node, and the API builds a homogeneous NodeConfig list. YAML has no per-node heterogeneous topology section; the documented support scope covers homogeneous node configuration.

Startup models are cached on every configured Controller. To place models on selected nodes later, start an empty service and deploy explicitly.

## GPU visibility and CPU resources

Controller / Worker clear Ray's CUDA_VISIBLE_DEVICES and reset it from AEGAEON_CUDA_VISIBLE_DEVICES. LLMService also converts the driver's CUDA_VISIBLE_DEVICES into actor environment settings. This custom GPU namespace differs from normal Ray actor visibility behavior.

Use consistent GPU numbering and visibility on every node. Physical GPU IDs from the driver propagate to actors, so each node must expose the corresponding devices. Production topology uses real Ray node resources.

`worker_num_cpus` controls each Worker's Ray CPU request. Per-node Engine placement groups require `total Engine count × worker_num_cpus` Ray CPUs; the Controller also needs the default actor CPU scheduling resource. Provide enough resources when adding Engines and use `ray status` to inspect resource demand. Aegaeon assigns GPUs through its device-ID mapping; Worker Ray options do not request `num_gpus`, so deployment must also give the service exclusive access to the selected GPUs.

## Shutdown and isolation

The CLI has no stop command. Interrupt a foreground service normally; lifespan cleanup calls ray.shutdown to disconnect the driver. When connected to an existing cluster, driver exit does not imply that the cluster stops or every cache is immediately removed.

Shared memory files and actors use fixed names. Give independent services isolated processes, clusters, shared directories, and resource allocations. `ray stop` affects Ray processes on that machine; use it only when you control that instance's lifecycle.

## Containers and Kubernetes

This guide uses source installation; the repository contains no standard Dockerfile or Helm chart. A custom image needs compatible CUDA/driver interfaces, shared and pinned memory, model/profile mounts, and a consistent Ray namespace. Run a real text request and shutdown-cleanup test before release.
