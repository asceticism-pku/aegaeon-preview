# CUDA Graphs

During Decode, the CPU submits a similar sequence of GPU operations for every generated token. Graph replay can reduce this launch overhead and lower Decode TPOT. Aegaeon therefore applies CUDA Graphs to Decode Workers in P/D deployments.

Aegaeon uses the open-source [Foundry](https://github.com/foundry-org/foundry) project to persist captured graphs, CUDA binaries, and deterministic device-memory layouts. The first run creates an archive in `save` mode; later processes restore it in `load` mode instead of capturing the graphs again.

## Supported path

- The integration applies only to P/D Decode Workers. The current supported configuration is TP=1.
- The graph contains the model's decode forward pass. Logits computation and argmax remain outside the graph.
- Prefill, mixed Prefill/Decode batches, and Simple Workers use the regular eager path.

A forward also uses a graph only when all of the following are true:

1. Every row in the batch is a single-token Decode step.
2. The actual batch size is listed in `batch_sizes`.
3. The largest sequence length in the batch does not exceed the configured `seq_len_buckets` limit.
4. The graph for the active model was restored successfully.

All other cases execute the eager forward path without failing the request. Confirm a match through the `Foundry decode replay active` Worker log.

## Install Foundry

`foundry/` is a separate subproject and is not installed by Aegaeon's `pip install -e .`. Build it in the same Python environment as Aegaeon:

```bash
conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
python -m pip install --no-build-isolation -e ./foundry

export AEGAEON_FOUNDRY_HOOK_PATH="$(python -c 'from pathlib import Path; import foundry.ops; p = Path(foundry.ops.__file__).resolve().parent / "libcuda_hook.so"; assert p.is_file(), p; print(p)')"
test -f "$AEGAEON_FOUNDRY_HOOK_PATH"
```

`libcuda_hook.so` must be loaded through `LD_PRELOAD` before a Worker process starts. Aegaeon uses `AEGAEON_FOUNDRY_HOOK_PATH` to set this environment for Ray Workers. In a multi-node deployment, install Foundry on every node that runs a Decode Worker and provide an absolute path local to that node.

See [Troubleshooting](troubleshooting.md) for custom Boost locations, CUDA forward-compatibility libraries, and common loading errors.

## Configuration

This example captures a Decode graph for batch size 1 and a maximum sequence length of 4096:

```yaml
server:
  num_engines: 0
  num_prefill_engines: 1
  num_decode_engines: 1
  tensor_parallel_size: 1
  cuda_graph:
    mode: 'save'
    archive_dir: /srv/aegaeon-archives/qwen3-test
    batch_sizes: [1]
    seq_len_buckets: [4096]
```

The current Worker implementation requires exactly one value in `seq_len_buckets`; it is the maximum sequence length supported by the captured graphs. `batch_sizes` accepts several positive integers, but the runtime batch size must match one of them exactly. Capturing more batch sizes increases model-load time and graph memory, so choose them from the workload you expect to serve.

Use [`foundry-save.yaml`](examples/foundry-save.yaml) and [`foundry-load.yaml`](examples/foundry-load.yaml) as complete configuration examples.

## Record an archive

First run a fixed set of requests with `mode: 'off'` and save the complete token IDs. Then:

1. Point `archive_dir` to a new empty directory and set `mode: 'save'`.
2. Start the service and make every model needed by the future LOAD configuration complete one model load on a Decode Worker. Startup models are captured during service startup; other models can be loaded through deployment or a request.
3. Check that every model and rank reports `captured ... Foundry decode graphs`, and record the source revision, configuration, and dependency versions used for the run.
4. Repeat the baseline requests and compare their complete token IDs.

Capture happens during model loading and finishes before the service decodes requests with that model.

## Load and validate

To use the archive, change only `mode` to `'load'`. Keep the models, graph shapes, and memory-layout settings unchanged, then start a fresh service process.

LOAD checks every model in the registry, not only `startup_models`. Initialization fails if any configured model is missing its archive. A successful model load logs `loaded ... Foundry decode graph shapes`; the first matching Decode batch logs `Foundry decode replay active`.

The following two checks can be used:

1. Compare complete token IDs from identical OFF and LOAD requests.
2. In a multi-model deployment, switch A→B→A and verify A's output and replay log again.

## When to record again

Foundry archives depend on the model weights, Aegaeon and Foundry revisions, PyTorch, vLLM, CUDA, the driver, GPU architecture, and CUDA Graph memory-layout settings. If any of these change, create a new archive in an empty directory and compare its LOAD output with the OFF baseline again. Treat archives as environment-specific artifacts, not as a portable cross-version format.
