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

## Online API demo tool

```bash
python benchmark/demo_api.py --help
python benchmark/demo_api.py \
  --host 127.0.0.1 --port 8000 \
  --model-config /srv/aegaeon/config.yaml --model Qwen/Qwen3-4B \
  --arrival-rate 0.1 --duration 60
```

This tool connects to a running API; replace the example model name with a configured, deployed model. It writes benchmark/demo-output.json, overwriting that path on subsequent runs. Default arrival intervals are exponentially distributed and models rotate; timed mode fixes max_tokens=500, while `--max-tokens` applies only to `--single`. Input is fixed text: the CSV input_tokens field is read but does not construct the prompt. Streaming results count nonempty text chunks in `tokens`; nonstreaming results use usage.completion_tokens. Distinguish these units in reports.

## Limitations of historical scripts

The end-to-end replay script is `benchmark/benchmark_e2e_from_workload.py`.

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

`sim/` uses SimPy and estimated execution times to study policies; see `sim/README.md`. Report simulations separately from GPU measurements and record the model, device, and policy definitions used by the simulation.
