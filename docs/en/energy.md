# Power Sampling and GPU Clock Control

## Power sampling

`GET /v1/aegaeon/gpu-power` collects GPU samples through nvidia-smi. Each sample contains index, timestamp, power_w, sm_clock_mhz, mem_clock_mhz, and utilization_gpu_pct; unsupported numeric values are `null`. A missing executable or failed command returns an empty `gpus` list and logs a warning. Empty stdout also produces an empty list.

This endpoint queries the API host rather than all GPUs across the cluster. Power in W differs from energy in J. Energy evaluation requires integration over a defined interval, with a sampling frequency and idle baseline, alongside real completed-token counts.

## Clock-control component

GpuFrequencyController is explicitly enabled by `AEGAEON_GPU_FREQ_CONTROL` and disabled by default. A Worker calls `set_for_model()` while loading a model: a clock range locks the device, while `None` resets an existing lock. `AEGAEON_GPU_CLOCK_COMMAND` can specify a wrapper; execution requires the corresponding device, driver, and system permissions.

## Configuration wiring limitations

`model_gpu_clock_range()` reads gpu_clock_range_mhz or gpu_clock_mhz from `model.extra`, including stage-specific mappings. However, ModelSpec currently has no extra field, and YAML model parsing does not retain these settings.

The service currently **does not support per-model clock settings through models YAML**. `gpu_clock_mhz` and `gpu_clock_range_mhz` never reach ModelSpec, so adding either field has no effect. The controller and scripts are experimental components; service integration requires ModelSpec fields, YAML parsing, and Worker mapping.

## Experimental tools

`tools/gpu_clock_search.py`, `tools/pd_clock_search.py`, and `benchmark/benchmark_energy.py` provide `--help`. Check clock changes, output paths, and model settings before experiments. `tools/energy_request.py` is a request helper module. `tools/aegaeon-gpu-clock` accepts `<gpu_id> set <min_mhz> <max_mhz>` or `<gpu_id> reset`. These experimental tools do not provide automatic energy-aware scheduling.

Currently, `energy_request.py` counts nonempty SSE text chunks in `tokens` and their arrival intervals in `per_token`. Clock-search and energy scripts derive rates and SLO results from these fields; label results as chunk-based. Real energy/token and token-level TPOT require complete token counts and timing data. Multi-sample integration in `benchmark_energy.py` covers only the first-to-last valid sample interval; one sample is multiplied by the experiment duration. Retain the sample coverage interval and measure boundary coverage and the idle baseline for reports.

## Reporting results

Keep output, load, and request order identical across comparisons. Report throughput, E2E, TTFT, TPOT, energy/request, and energy/token together, plus the clock-command result and restoration state. Compute energy by integrating power over the complete execution interval.
