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

The three HTTP CSV examples are shipped with the source checkout; generated CSV results remain ignored by Git. Their header, model mapping and request counts are documented in `workloads/README.md`.

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

Its `--csv` option expects `request_id,timestamp,input_tokens,output_tokens`.
`timestamp` is an arrival delay in seconds, `request_id` selects a model modulo
the configured model count, and `output_tokens` sets `max_tokens`. The fixed
prompt does not honor the requested `input_tokens` length. Use the HTTP replay
client above for `workloads/synth-*.csv`, which has a different schema.
Run this streaming demo's help in an environment with aiohttp installed.
The latency, model-placement, Work Stealing, comparison, profiling, and HTTP
workload help commands below need only the standard library; they do not
initialize CUDA, Ray, or a model. Actual measurement and GPU smoke runs still
need the compatible runtime, checkpoints, profiles, and adequate resources.
The HTTP client needs aiohttp when it sends requests, and GPU configuration
helpers need PyYAML when called.

```bash
python benchmark/benchmark_latency.py --help
python benchmark/model_placement_gpu_smoke.py --help
python benchmark/work_stealing_gpu_smoke.py --help
python benchmark/compare_work_stealing_results.py --help
python profiles/do_profile.py --help
python aegaeon/test_api.py --help
```

Simulation help uses the CPU dependencies listed below. The separate energy
benchmark still imports optional packages before parsing help.

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

The simulator uses the existing repository `profiles/` directory by default.
From a clean source checkout in a Python 3.10–3.14 environment:

```bash
python -m pip install -r sim/requirements.txt
python -m sim.simulate --workload synth-8-0.1-1.0-1.0.csv \
  --devices a100 --seed 0 --job-timeout 30 \
  --output-dir work/sim-results --name historical-a100
python -m pytest tests/test_simulator.py tests/test_sim_repository_profiles.py \
  tests/test_sim_profile_collector.py -q
```

The historical eight-model trace uses six Qwen2.5-7B and two Yi-1.5-9B-Chat
aliases. Explicit name mappings resolve them to `profiles/qwen2_5_7b/A100`
and `profiles/yi1_5_9b_chat/A100`; no data copying, symlinks, checkpoint
downloads, Torch, vLLM, Ray or CUDA are needed. The default root is resolved
relative to the simulator source location. `--profile-root PATH` confines
lookup to that root, accepting canonical and serving directory layouts.
See `sim/README.md` for the alias table and optional collection commands.

Output is `work/sim-results/historical-a100.json`, grouped by policy and
request ID, with 81 results per policy. Successful arrays contain simulated
TTFT followed by token intervals in milliseconds; `[-1]` means a scheduling
deadline failure. With the command above, SeLLM completes 33 requests and
fails 48; SeLLM+ completes 37 and fails 44. Repeating the same inputs and seed
reproduces CPU results; a completed simulation does not mean all requests
meet their SLOs on the single simulated GPU.

Profiles contain `avg_prefill_latency` and `avg_decode_latency` in seconds.
The existing files retain their original measurement conditions. The historical
serving collection script defaulted to TP=1, float16 and eager execution,
but the Qwen2.5-7B and Yi-1.5-9B files lack complete per-run provenance;
the defaults do not confirm every file's collection settings. The timings do
not establish performance for a different runtime or BF16 configuration. `a100` (80GB) and `a100_40gb`
remain distinct, as do L20 and H20. Missing model/device pairs fail explicitly.

`--workload` runs a fixed CSV and is mutually exclusive with the existing
`--num-models`/`--arrival-rate` sweep. Explicit input/output lengths stay paired
with their CSV row; historical two-column traces sample lengths using the seed.
Synthetic correctness fixtures remain separate from repository timings.

The result is an execution-profile-driven simulation. Linear-fit error,
approximate weight/runtime-buffer modeling and assumed swapping bandwidth
limit its accuracy; reproducibility alone does not validate GPU capacity or
SLO predictions. For other workloads, follow [profiling](profiling.md), retain
provenance and validate model/device equivalence. The optional
`tools/collect_sim_profiles.py` writes newly collected Qwen/A100 40GB profiles
under `work/collected-sim-profiles`; GPU collection is not required for this
example. Missing/empty profiles and failed or timed-out workers cause a
nonzero exit with subprocess cleanup.
