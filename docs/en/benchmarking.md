# Benchmarking and Reproducible Evaluation

## Define measurements first

| Metric | Definition and considerations |
|---|---|
| TTFT | Request send/arrival to first token; distinguish client text from backend tokens |
| E2E | Request start to an explicit completion event; specify finish_reason or [DONE] |
| TPOT | Average Decode token interval for multi-token output; HTTP chunk interval is a separate metric |
| Throughput | Actual completed tokens or requests / a specified wall-time interval |
| Makespan | Time from workload start until all requests complete |
| SLO / qos | State the exact source and formula, with each definition in its own column |
| CV | Population standard deviation / mean; specify engine set and token-count scope |

## Replay the bundled workloads through HTTP

Run from the Aegaeon source checkout root after completing the [installation](installation.md)
and [quickstart](quickstart.md). `benchmark/config-pd-2gpu.yaml` is a two-GPU PD
example with `qwen2_5_7b` and `qwen2_1_5b`; replace its host-specific checkpoint
paths and device settings before starting it in a separate terminal:

```bash
aegaeon start --config benchmark/config-pd-2gpu.yaml --ray-address local
```

After startup, check the API and replay an existing CSV:

```bash
curl --fail http://127.0.0.1:8000/health
curl --fail http://127.0.0.1:8000/v1/models
python aegaeon/test_api.py --csv workloads/synth-1-0.1-1.0-1.0.csv \
  --models qwen2_5_7b qwen2_1_5b --request-timeout 300
```

The other bundled HTTP workloads are `workloads/synth-2-0.1-1.0-1.0.csv` and
`workloads/synth-2-2.0-1.0-1.0.csv`; use the same command with their paths. The
header is `Request Time,Model Name,Input Length,Output Length`. Arrivals are
relative `HH:MM:SS.mmm` timestamps, and `synth_model_00/01` map to the ordered
`--models` aliases. The client repeats `"test "` Input Length times, so this
field does not guarantee that many tokenizer tokens. Output Length is the
requested `max_tokens`, and generation ends earlier when it reaches EOS.

The runner uses non-streaming requests and reports readable HTTP-200 responses,
per-request latency, and failure/timeout totals; any failure makes it exit
nonzero. The 300-second timeout starts after the arrival delay. Its makespan
includes all arrival delays. It does not measure TTFT, TPOT, or actual output
token throughput; preserve client/service logs and measure actual tokens when
reporting performance. A successful HTTP response alone does not validate the
generated content.

## Online API demo tool

```bash
python benchmark/demo_api.py --help
python benchmark/demo_api.py \
  --host 127.0.0.1 --port 8000 \
  --model-config /srv/aegaeon/config.yaml --model Qwen/Qwen3-4B \
  --arrival-rate 0.1 --duration 60
```

This tool connects to a running API; replace the example model name with a configured, deployed model. It writes benchmark/demo-output.json, overwriting that path on subsequent runs. Default arrival intervals are exponentially distributed and models rotate; timed mode fixes max_tokens=500, while `--max-tokens` applies only to `--single`. Input is fixed text: the CSV input_tokens field is read but does not construct the prompt. Streaming results count nonempty text chunks in `tokens`; nonstreaming results use usage.completion_tokens. Distinguish these units in reports.

Its `--csv` option expects the historical snake_case schema below, **not** the
bundled `workloads/synth-*.csv` schema. Run this tool's help in an environment
with aiohttp installed. Several benchmark scripts still eagerly import optional
packages before parsing help: latency needs vLLM; direct E2E needs Ray and
serving-runtime imports; placement/Work Stealing help needs Ray, PyYAML, and
psutil, while execution additionally loads the serving runtime. Lazy package initialization does not
make these scripts dependency-free. `compare_work_stealing_results.py --help`
and `profiles/do_profile.py --help` can run with the standard library alone;
simulation help uses the CPU dependencies listed below.

## Limitations of historical scripts

The historical direct end-to-end replay script is
`benchmark/benchmark_e2e_from_workload.py`. It is separate from the current
HTTP runner above. Its default workload file is not bundled: pass an existing
`--workload-csv` and explicit `--model-config`/`--output-json` rather than relying
on the old external artifact directory or missing benchmark assets.

Its Ray address is fixed to the original experimental cluster. Each node fixes 1 Prefill + 1 Decode and model_cache_size=0, with CPU slab counts supplied by a command argument; these settings are not read from the YAML server topology. Change the address and topology to the current cluster and target configuration before use.

CSV fields are request_id, timestamp, input_tokens, and output_tokens. Models are assigned from a seed, and prompts are slices of a fixed pre-encoded token list. Inputs beyond that list become shorter, and decode length follows a 4096-based truncation rule. This script therefore measures its transformed workload; report the actual prompt and output lengths.

## Recommended experiment design

1. Fix checkpoints, tokenizers, complete YAML, dependencies, and source revision.
2. Define the same request sequence, actual token lengths, model arrival distribution, concurrency, and duration.
3. Record cold start, CPU cache hits, GPU switches, and warm steady state separately.
4. Warm all replicas consistently, or explicitly use no warmup requests for zero-warmup experiments.
5. Alternate OFF/LOAD or policy A/B order, repeat runs, and report P50/P95 and sample counts.
6. Check token IDs, completed requests, failures, resources, and shutdown cleanup.
7. Preserve raw logs, responses, configuration, and results. Label failed samples rather than silently removing them.

## Specialized tools

Work Stealing provides paired off/on smoke tests and compare_work_stealing_results.py. Model placement uses model_placement_gpu_smoke.py to compare least-loaded / round-robin and busy-avoidance routing.

Graph tools include benchmark_cuda_graph_latency.py, benchmark_foundry_three_model_latency.py, and roundtrip tests. Check scripts and `tools/README_three_model_latency.md` for default paths and archive prerequisites. The three-model tool measures real GPU switches with the service running and CPU weights available; it excludes CPU cache-miss deployment and complete startup costs.

## Performance reporting

Report GPU/CPU/interconnect, driver, torch/vLLM versions, source, models, mode, budgets, batches, actual lengths, arrival rates, warmup, sample counts, and statistics. Use samples that pass correctness checks, and retain the hardware, configuration, and measurement conditions with every cited result.

## Simulator

`sim/` uses SimPy and estimated execution times to study policies. From the
Aegaeon source checkout root in a Python 3.10+ virtual environment, a self-contained CPU
correctness check is:

```bash
python -m pip install aenum typing_extensions simpy==4.1.1 numpy scikit-learn matplotlib pytest
python -m sim.simulate --help
python -m pytest tests/test_simulator.py -q
```

Expect 13 passing tests, including both policies on the 81-request bundled
workload. The tests use explicitly marked synthetic profiles in temporary
directories; their latency values are invented and do not measure a GPU.

Performance simulation needs measured profiles. `sim/simdata/execution/` is
not bundled. Follow [profiling](profiling.md) to collect model execution times
on the target GPU; current `benchmark/benchmark_latency.py` emits
`avg_prefill_latency` and `avg_decode_latency` in seconds in addition to the
serving sample arrays. Use the collection and staging steps below to place matching measurements under
`<profile-root>/<simulator-model>/<simulator-device>/i<input_length>b<batch_size>.json`.
Serving YAML aliases and CUDA product names differ from simulator names, so
do not point the simulator directly at arbitrary serving profiles or rename
unmatched hardware measurements.

### Collect and stage measured profiles

Prepare a model YAML containing the actual Qwen2.5-7B and Yi-9B checkpoint
paths and explicit profile aliases. Follow the [profiling guide](profiling.md)
for the configuration format. The collector runs vLLM on the selected GPU at
TP=1 and requires the complete GPU runtime; this collection step is separate
from the CPU simulator check above. Run from the Aegaeon source checkout:

```bash
CUDA_VISIBLE_DEVICES=0 AEGAEON_PROFILE_ROOT="$PWD/work/measured-profiles" \
  python profiles/do_profile.py \
  --model-config /absolute/path/to/simulation-profile-models.yaml --tp 1
```

Output is written under
`work/measured-profiles/<model-profile>/<CUDA-product-name>/i<input_length>b<batch_size>.json`.
The averages are in seconds. Collect usable training points with independent
input lengths and batch sizes for both latency estimators. Keep the collection
YAML and checkpoint, hardware, backend, and sampling metadata with the results.

After confirming that the measurements match the simulator's model/context
assumptions and **80 GB A100** device, replace the source paths below with
the actual measured directories and stage both models:

```bash
QWEN_MEASURED_DIR=/absolute/path/to/measured-qwen/product-name
YI_MEASURED_DIR=/absolute/path/to/measured-yi/product-name
mkdir -p work/sim-execution/qwen2.5-7b/a100 work/sim-execution/yi-9b/a100
cp "$QWEN_MEASURED_DIR"/i*b*.json work/sim-execution/qwen2.5-7b/a100/
cp "$YI_MEASURED_DIR"/i*b*.json work/sim-execution/yi-9b/a100/
```

Preserve the original metadata alongside the staged tree. The simulator checks
that usable training profiles exist, while checkpoint and hardware equivalence
remain part of experiment preparation. Serving files that contain only raw
latency arrays need an explicit averaging rule before use as simulator inputs.

### Run the measured simulation

After preparing `work/sim-execution`, run:

```bash
python -m sim.simulate --num-models 8 --arrival-rate 0.1 --devices a100 \
  --profile-root work/sim-execution --job-timeout 300 \
  --output-dir work/sim-results --name measured-a100
```

The eight-model trace requires both `qwen2.5-7b/a100` and `yi-9b/a100` data;
the simulator's `a100` is an 80 GB device model. Missing/empty training data
fails before workers launch. Output is `work/sim-results/measured-a100.json`
with results grouped by policy and request ID under SeLLM and SeLLM+. Each
successful request's array contains simulated TTFT followed by consecutive
token intervals, all in milliseconds; `[-1]` marks a failed request. Inspect
those arrays as well as process status. Report simulation separately from GPU
measurements and record profile provenance, model/device assumptions, and
policies. No measured-performance simulation is self-contained until suitable
profiles are supplied.
