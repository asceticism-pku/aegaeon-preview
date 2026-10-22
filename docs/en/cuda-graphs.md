# CUDA Graphs

Aegaeon uses [Foundry](https://github.com/foundry-org/foundry) to save and restore Decode graphs, reducing CPU launch overhead. The integration runs on P/D Decode Workers with TP=1. Prefill and Simple Workers use eager execution.

## Install Foundry

Use the same environment as Aegaeon and run from the repository root:

```bash
conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
python -m pip install --no-build-isolation -e ./foundry
export AEGAEON_FOUNDRY_HOOK_PATH="$(python -c 'from pathlib import Path; import foundry.ops; print(Path(foundry.ops.__file__).resolve().parent / "libcuda_hook.so")')"
```

Install Foundry on every Decode node and use matching absolute hook/library paths. For library loading errors, see [Troubleshooting](troubleshooting.md).

## Configure

```yaml
server:
  num_engines: 0
  num_prefill_engines: 1
  num_decode_engines: 1
  tensor_parallel_size: 1
  cuda_graph:
    mode: 'save'
    archive_dir: /srv/aegaeon-archives/qwen3
    batch_sizes: [1]
    seq_len_buckets: [4096]
```

Use exactly one maximum sequence-length bucket. Graph replay requires an exact batch-size match and a sequence length within that bucket; other batches use eager execution.

## Save and load

The [SAVE and LOAD examples](examples/README.md) provide complete configurations. Adjust model paths and budgets, then start SAVE:

```bash
aegaeon start --config docs/en/examples/foundry-save.yaml --ray-address local
```

After `Application startup complete`, send a request from another terminal to load the model onto the Decode Worker and capture its graphs:

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  --data-binary @docs/en/examples/chat-request.json
```

For multiple models, send a request for each configured model. Wait for the request to finish and the Worker log `captured ... Foundry decode graphs during model load`. Then stop the service and start LOAD with the same model, memory layout, and archive directory:

```bash
aegaeon start --config docs/en/examples/foundry-load.yaml --ray-address local
```

`Foundry decode replay active` identifies graph execution in Worker logs. Record new archives after changing model weights, graph sizes, memory layout, or runtime dependencies.
