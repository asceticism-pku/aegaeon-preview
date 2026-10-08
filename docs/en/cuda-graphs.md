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
4. The graph for the active model was captured or restored successfully.

All other cases execute the eager forward path without failing the request. Confirm a match through the `Foundry decode replay active` Worker log.

## Install Foundry

`foundry/` is a separate subproject and is not installed by Aegaeon's `pip install -e .`. Build it in the same Conda/Python environment as Aegaeon:

```bash
conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
python -m pip install --no-build-isolation -e ./foundry
```

`$CONDA_PREFIX/lib` supplies the environment's Boost and `libstdc++` libraries. Set this search path before starting Python. Loading the system `libstdc++` after dependency installation can cause a NumPy import failure that surfaces as an `NP_SUPPORTED_MODULES` error.

If Conda reports `shards_cache` / SQLite `database is locked`, run the same installation command with sharded repodata caching disabled for that command, then continue:

```bash
CONDA_PLUGINS_USE_SHARDED_REPODATA=false conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
```

### CUDA 12.9 compatibility library

Foundry calls `cuFuncGetName` directly. Importing `foundry.ops` fails when the driver library lacks this symbol. If the driver library already provides this symbol, skip this section and proceed to the hook path below. When a CUDA forward-compatibility library is needed, first confirm that the GPU and driver combination is supported by the [NVIDIA compatibility support table](https://docs.nvidia.com/deploy/cuda-compatibility/forward-compatibility.html), then select a library matching `torch.version.cuda` and the operating system.

The download example below applies to Ubuntu 20.04 x86_64, CUDA 12.9, and a GPU and driver combination allowed by the support table, such as A100 with driver `535.247.01`. Use the corresponding NVIDIA package for other platforms or CUDA versions. Run these commands from the repository root. They download the official NVIDIA package, verify its SHA256, and extract it into a new directory without root privileges. They require `curl`, `sha256sum`, and Ubuntu's `dpkg-deb` command. Resolve any download or checksum failure before continuing.

```bash
python -c "import torch; assert torch.version.cuda == '12.9', torch.version.cuda"
export CUDA_COMPAT_DIR="$(mktemp -d "$PWD/cuda-compat-12.9-XXXXXX")"
curl --fail --location --retry 3 \
  'https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-compat-12-9_575.57.08-0ubuntu1_amd64.deb' \
  -o "$CUDA_COMPAT_DIR/cuda-compat-12-9.deb"
printf '%s  %s\n' \
  '0e67e0011b8cb359cecf5d60c3dd7be9c2a69f1b7eb531a4536b83b02173d87a' \
  "$CUDA_COMPAT_DIR/cuda-compat-12-9.deb" | sha256sum -c -
dpkg-deb -x "$CUDA_COMPAT_DIR/cuda-compat-12-9.deb" "$CUDA_COMPAT_DIR/root"
export AEGAEON_CUDA_COMPAT_PATH="$CUDA_COMPAT_DIR/root/usr/local/cuda-12.9/compat"
test -f "$AEGAEON_CUDA_COMPAT_PATH/libcuda.so.1"
printf '%s\n' "$AEGAEON_CUDA_COMPAT_PATH"
```

Keep this directory and record the absolute path printed at the end. In later terminals or service restarts, set `AEGAEON_CUDA_COMPAT_PATH` to that absolute path before running the path check and search-path setup below. If a compatibility library already exists, you can set its absolute path directly. Prepare the library and pass the check before importing Foundry. The compatibility library must come first in the search path, followed by the Conda `lib` directory.

```bash
AEGAEON_CUDA_COMPAT_PATH=$(python - <<'PY'
import os
from pathlib import Path
import torch
version = torch.version.cuda
configured = os.environ.get("AEGAEON_CUDA_COMPAT_PATH")
candidates = [Path(configured).expanduser()] if configured else [
    Path(f"/usr/local/cuda-{version}/compat"),
    Path.home() / f".local/cuda-{version}/compat",
]
path = next((p.resolve() for p in candidates if (p / "libcuda.so.1").is_file()), None)
assert path is not None, f"Prepare the CUDA {version} compatibility library first"
print(path)
PY
)
export AEGAEON_CUDA_COMPAT_PATH
test -f "$AEGAEON_CUDA_COMPAT_PATH/libcuda.so.1"
export LD_LIBRARY_PATH="$AEGAEON_CUDA_COMPAT_PATH${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

### Locate the hook

After configuring these library paths, import Foundry to locate the hook:

```bash
export AEGAEON_FOUNDRY_HOOK_PATH="$(python -c 'from pathlib import Path; import foundry.ops; p = Path(foundry.ops.__file__).resolve().parent / "libcuda_hook.so"; assert p.is_file(), p; print(p)')"
test -f "$AEGAEON_FOUNDRY_HOOK_PATH"
```

`libcuda_hook.so` must be loaded through `LD_PRELOAD` before a Worker process starts. Aegaeon uses `AEGAEON_FOUNDRY_HOOK_PATH` to set this environment for Ray Workers. In a multi-node deployment, install Foundry on every node that runs a Decode Worker and provide an absolute path local to that node.

See [Troubleshooting](troubleshooting.md) for custom Boost locations, CUDA forward-compatibility libraries, and common loading errors.

## Configuration

This 1P+1D example requires two available GPUs and captures a Decode graph for batch size 1 and a maximum sequence length of 4096. See [Deployment](deployment.md#pd-separation) for GPU requirements.

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
2. Start the service and send at least one generation request that enters Decode for every model needed by the future LOAD configuration. `startup_models` and deploy only register models and cache CPU weights; capture happens when the model is actually loaded on a Decode GPU. Service startup or successful deployment does not prove the archive is complete.
3. Check that every model and rank reports `captured ... Foundry decode graphs`, and record the source revision, configuration, and dependency versions used for the run.
4. Repeat the baseline requests and compare their complete token IDs.

Capture happens during model loading and finishes before the service decodes requests with that model.

## Load and validate

To use the archive, change only `mode` to `'load'`. Keep the models, graph shapes, and memory-layout settings unchanged, then start a fresh service process.

LOAD checks every model in the registry, not only `startup_models`. Initialization fails if any configured model is missing its archive. A successful model load logs `loaded ... Foundry decode graph shapes`; the first matching Decode batch logs `Foundry decode replay active`.

The following two checks can be used:

1. Compare complete token IDs from identical OFF and LOAD requests.
2. In a multi-model deployment, switch A→B→A and verify A's output and replay log again.

## Single-model validation

After installing Aegaeon and Foundry, run these commands from the repository root with a complete local `Qwen/Qwen3-4B` checkpoint. They retain the example's profiles, graph shapes, and memory budgets, changing only the model path, archive path, and mode. `mktemp` creates a new directory so SAVE does not reuse an old archive.

```bash
export MODEL_DIR=/absolute/path/to/Qwen3-4B
export GRAPH_RUN_DIR="$(mktemp -d "$PWD/foundry-check-XXXXXX")"
python - <<'PY'
import copy
import os
from pathlib import Path
import yaml
run = Path(os.environ["GRAPH_RUN_DIR"])
model = Path(os.environ["MODEL_DIR"]).resolve()
assert model.is_dir(), model
template = yaml.safe_load(Path("docs/en/examples/foundry-save.yaml").read_text())
template["models"][0]["path"] = str(model)
template["server"]["cuda_graph"]["archive_dir"] = str(run / "archive")
for mode in ("off", "save", "load"):
    config = copy.deepcopy(template)
    config["server"]["cuda_graph"]["mode"] = mode
    (run / f"{mode}.yaml").write_text(yaml.safe_dump(config, sort_keys=False))
print(run)
PY
```

In the first terminal, select two available GPUs (0 and 1 below) and start the OFF service:

```bash
export CUDA_VISIBLE_DEVICES=0,1
MODE=off
aegaeon start --config "$GRAPH_RUN_DIR/${MODE}.yaml" \
  --host 127.0.0.1 --port 8000 --ray-address local \
  2>&1 | tee "$GRAPH_RUN_DIR/${MODE}.log"
```

Wait for `Application startup complete`. In a second terminal, use the same repository and Python environment, set `GRAPH_RUN_DIR` to the absolute directory printed above, and send this fixed request with a maximum of 128 generated tokens. The validation below requires at least two actual tokens to exercise Decode:

```bash
GRAPH_RUN_DIR=/absolute/path/printed/above
MODE=off
curl --fail-with-body -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"What are the differences between processes and threads?"}],"temperature":0,"max_tokens":128,"return_token_ids":true}' \
  -o "$GRAPH_RUN_DIR/${MODE}.json"
```

After the request finishes, press Ctrl+C in the first terminal to stop the service normally. Confirm that its processes have exited and released their GPU resources. Set `MODE=save` in both terminals, repeat startup and the request, then stop and wait for resources to be released again. Repeat once more with `MODE=load`. Start a fresh process for every mode and keep the other configuration settings unchanged.

After all three phases, compare the complete prompt and generated token IDs in the second terminal and inspect the logs:

```bash
export GRAPH_RUN_DIR
python - <<'PY'
import json
import os
from pathlib import Path
import re
run = Path(os.environ["GRAPH_RUN_DIR"])
results = [json.loads((run / f"{mode}.json").read_text()) for mode in ("off", "save", "load")]
prompts = [r["prompt_token_ids"] for r in results]
tokens = [r["choices"][0]["token_ids"] for r in results]
assert all(len(ids) >= 2 for ids in tokens), "Need at least two generated tokens"
assert all(len(p) + len(t) <= 4096 for p, t in zip(prompts, tokens)), "Sequence exceeds graph bucket"
assert prompts[0] == prompts[1] == prompts[2], "Prompt token IDs differ"
assert tokens[0] == tokens[1] == tokens[2], "Generated token IDs differ"
logs = {mode: (run / f"{mode}.log").read_text(errors="replace") for mode in ("off", "save", "load")}
replay = "Foundry decode replay active"
capture = r"captured(?: \d+)? Foundry decode graphs?"
assert replay not in logs["off"], "OFF unexpectedly replayed a graph"
assert re.search(capture, logs["save"]) and replay in logs["save"], "SAVE capture/replay missing"
assert re.search(r"loaded \d+ Foundry decode graph shapes", logs["load"]), "LOAD marker missing"
assert replay in logs["load"] and not re.search(capture, logs["load"]), "LOAD replay missing or captured again"
print(f"OFF / SAVE / LOAD match: {len(tokens[0])} generated tokens; graph logs verified")
PY
```

SAVE must log both `captured ... Foundry decode graphs` and `Foundry decode replay active`. LOAD must log both `loaded ... Foundry decode graph shapes` and `Foundry decode replay active`, without capturing again. A successful HTTP request without the replay marker does not verify that CUDA Graphs were used.

## When to record again

Foundry archives depend on the model weights, Aegaeon and Foundry revisions, PyTorch, vLLM, CUDA, the driver, GPU architecture, and CUDA Graph memory-layout settings. If any of these change, create a new archive in an empty directory and compare its LOAD output with the OFF baseline again. Treat archives as environment-specific artifacts, not as a portable cross-version format.
