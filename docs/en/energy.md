# Power Sampling and GPU Clock Control

## Power sampling

`GET /v1/aegaeon/gpu-power` collects GPU samples through nvidia-smi. The NvidiaSmi layer parses device identifiers, power, clocks, and related fields. A command failure or empty query produces an empty `gpus` list, with the reason written to the service log.

This endpoint queries the API host rather than all GPUs across the cluster. Power in W differs from energy in J. Energy evaluation requires integration over a defined interval, with a sampling frequency and idle baseline, alongside real completed-token counts.

## Clock-control component

GpuFrequencyController is explicitly enabled by `AEGAEON_GPU_FREQ_CONTROL` and disabled by default. A Worker calls `set_for_model()` while loading a model: a clock range locks the device, while `None` resets an existing lock. `AEGAEON_GPU_CLOCK_COMMAND` can specify a wrapper; execution requires the corresponding device, driver, and system permissions.

## Configuration wiring limitations

`model_gpu_clock_range()` reads gpu_clock_range_mhz or gpu_clock_mhz from `model.extra`, including stage-specific mappings. However, ModelSpec currently has no extra field, and YAML model parsing does not retain these settings.

The service currently **does not support per-model clock settings through models YAML**. `gpu_clock_mhz` and `gpu_clock_range_mhz` never reach ModelSpec, so adding either field has no effect. The controller and scripts are experimental components; service integration requires ModelSpec fields, YAML parsing, and Worker mapping.

## Experimental tools

tools/ includes gpu_clock_search.py, pd_clock_search.py, energy_request.py, and aegaeon-gpu-clock; benchmark/ includes benchmark_energy.py. Read --help and script prerequisites to understand clock changes, output paths, and model settings. These tools do not provide an automatic energy-aware scheduling feature.

## Reporting results

Keep output, load, and request order identical across comparisons. Report throughput, E2E, TTFT, TPOT, energy/request, and energy/token together, plus the clock-command result and restoration state. Compute energy by integrating power over the complete execution interval.
