# Performance Tuning

## Establish a baseline

Start with TP=1, graphs disabled, one model, and explicit budgets. Measure actual token lengths, TTFT, E2E, throughput, and CPU/GPU usage. Separate first loading from steady state and preserve raw responses and failure counts.

## Tune by bottleneck

| Symptom | Check first | Possible adjustments |
|---|---|---|
| High TTFT | Queueing, first GPU loading, switches, Prefill length | Add Prefill/Simple resources, reduce inputs, adjust arrival rate |
| Long Decode intervals | Multiplexing quotas, switching, KV transfers | Add Decode resources, reduce active-model competition, recollect the matching profile |
| Uneven node routing | READY set, outstanding requests, cumulative assignments | Compare least-outstanding and round-robin in paired runs |
| Insufficient CPU KV | Slab count, actual concurrency and lengths | Increase CPU KV or reduce concurrency/length |
| Insufficient weight cache | Parameter counts, slice fragmentation, active-model protection | Increase CPU weight cache or manage replicas explicitly |
| GPU OOM | Current/prefetched weights, temporary workspace, KV | Leave headroom, reduce batch/context, check model integration |
| Uneven Decode idleness | Eligible batches and idle receivers | Evaluate Work Stealing on a suitable P/D topology |
| Significant control overhead for small Decode batches | Eager baseline and actual replay hits | Compare OFF/LOAD after token IDs match |

## Frequently misinterpreted settings

Increasing memory_utilization raises the estimated KV capacity while reducing headroom for temporary allocations. model_cache_size_gb controls the CPU weight cache; GPU KV capacity comes from GPU-side settings such as memory_utilization. nnodes counts physical nodes. To scale down, undeploy the model and deploy the target replica count.

More Engines require additional GPUs, Ray CPUs, and pinned-cache capacity. Measure Simple and P/D with the target workload, then choose from E2E, TTFT, TPOT, and throughput results.

## Graph mode

List every actual batch size in `batch_sizes`; replay uses an exact batch-size match. `seq_len_buckets` currently contains one element. Larger maximum lengths increase static graph input and metadata budgets. Keep SAVE/LOAD layouts identical and recapture after changing `memory_utilization`.

## Profile reliability

Profiles are specific to hardware, drivers, clocks, and model versions. Confirm that production batch and length distributions fall within the collected range. The current release supports TP=1 only.

## Validate each change

Check output tokens, failures, resources, and shutdown cleanup. Base performance conclusions on repeated runs with an exclusive GPU, and report sample counts, percentiles, and variability.
