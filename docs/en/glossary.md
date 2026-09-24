# Glossary and FAQ

| Term | Meaning in Aegaeon |
|---|---|
| Prefill | Consumes the prompt, establishes state, and usually generates the first token |
| Decode | Autoregressive iterations for subsequent tokens |
| P/D | Separate Prefill and Decode execution |
| Simple | One Engine handles both stages |
| Controller | Control actor on a physical Ray node |
| Engine | Scheduling/execution unit managing one or more Workers |
| Worker | GPU execution actor |
| QuickCache | CPU weight cache |
| KV Cache | Request-level attention K/V or related model state |
| READY replica | Node-level cached replica accepting new requests, not a permanently GPU-resident model |
| Reservation | API accounting for assigned but unfinished requests |
| Round / Turn / Quota | Scheduler batch cycle, an execution segment, and its quota |
| TTFT | Time To First Token; identify the observation layer |
| TPOT | Time Per Output Token; specify timing and treatment of the first token |
| HOL blocking | Earlier work blocks subsequent work |
| CV | Population standard deviation / mean |
| Foundry SAVE/LOAD | Capture/restore graphs, device code, and deterministic layouts |
| GiB | 2^30 bytes; several gb settings actually use this unit |

## Can Aegaeon replace a vLLM server directly?

Integration is possible for supported endpoints and greedy parameters. Function tools cover the documented schema only for models with a matching vLLM parser. Upstream sampling options, Responses, and other interfaces require separate compatibility work.

## Why does a multi-GPU host still use nnodes=1?

A node is a physical Ray node; engine counts represent GPU execution instances on that host. They share a Controller and QuickCache.

## Why can the first request be slow after READY?

Deploying weights into CPU cache does not imply GPU loading, initialization, or graph restoration has completed.

## Are CUDA Graphs enabled by default?

No; the default is off. The current integration needs P/D Decode, Foundry and its hook, one length bucket, and matching archives. Setting mode=load alone is insufficient.

## Can inference run natively on Windows or CPU?

The implementation requires CUDA and uses Linux .so libraries, /dev/shm, and native CUDA/Rust extensions. Native Windows and CPU inference are not supported.

## Are documented performance numbers guaranteed to reproduce?

Performance depends on models, hardware, workloads, and execution settings. Reproduce under matching conditions and distinguish first loading, model switching, and steady-state inference.
