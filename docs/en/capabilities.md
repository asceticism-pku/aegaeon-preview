# Features and Compatibility

## Supported scope

| Capability | Current status | Implementation |
|---|---|---|
| Interleaved multi-model inference | Supported. Models switch at batch, round, and execution-quota boundaries | `simple_scheduler.py`, `decode_scheduler.py` |
| Simple mode | Supported. One Engine handles Prefill and Decode | `stage_engine.py`, `simple_dispatcher.py` |
| Prefill/Decode separation | Supported. Node-local CPU KV Cache transfers state between stages | `llm.py`, `block_manager.py` |
| Multiple physical nodes | Supported. Each node has one Controller, and each request stays on its selected node | `llm.py`, `model_placement.py` |
| Dynamic deploy and unload | Supports node-level cached replicas, draining, and partial-failure results | `api.py` |
| Replica routing | Supports least-outstanding and round-robin across READY replicas | `model_placement.py` |
| Tensor Parallel | **Only `tensor_parallel_size=1` is supported** | `estimator.py`, `config.py` |
| Pipeline Parallel | **Unsupported**; `pipeline_parallel_size` is fixed at 1 | `config.py` |
| Chat / Completion | Partial OpenAI-style interface; see the [request schema](request-schema.md) | `api.py` |
| SSE streaming | Supported. Events carry text deltas or final state; events and tokens are not one-to-one | `api.py` |
| Greedy decoding | Supported. Workers select argmax from logits | `worker.py` |
| Random sampling / multiple candidates | **Unsupported**; sampling fields accept greedy-equivalent values and `n` is fixed at 1 | `_validate_greedy_parameters` |
| Logprobs / beam search | **Unsupported** | `api.py` |
| Multimodal input | **Unsupported**. The public serving scope is text input and text generation | `api.py`, `worker.py` |
| Prefix caching | **Unsupported**; explicitly disabled in vLLM CacheConfig | `config.py` |
| Quantization configuration | **Unsupported**; ModelConfig uses `quantization=None` | `config.py` |
| Decode Work Stealing | Optional and disabled by default; transfers eligible complete batches within one Controller | `decode_dispatcher.py` |
| Foundry archives | Supported for P/D Decode Workers; Simple Workers use the regular path | `stage_engine.py` |
| Runtime snapshots and events | JSON snapshots and SSE event streams | `api.py` |
| Prometheus metrics | **Unsupported** | `api.py` |
| API key authentication | Supplied by the deployment gateway; Aegaeon omits server-side key validation | `api.py` |

## Parallelism constraints

Aegaeon currently supports only `tensor_parallel_size=1`. The parser accepts larger integers, but the latency estimator executes `assert tp == 1`, and the scheduler, KV Cache, and Foundry support scope is limited to TP=1. Use TP=1 in every deployment configuration, profile, and issue report.

Pipeline Parallel is also fixed at 1. To add throughput, increase the number of Simple Engines or Prefill/Decode Engines. One Engine maps to one Worker and one GPU.

## Model and input scope

The public interface supports text-only causal language models, string messages, and text generation. Chat requests use the tokenizer's chat template. Completion accepts a string or one list of token IDs.

The public service currently accepts text input and produces text output. `Message.content` and Worker retain reserved code paths for content parts and multimodal processing, and `mm_processor_kwargs` remains a reserved field.

A model enters the supported deployment set after its configuration, weight loading, performance profiles, chat template, EOS handling, KV type, and target topology pass the fixed acceptance procedure in [Model Preparation and Registration](models.md).

## Interface boundaries

Aegaeon uses vLLM model classes and compute kernels while providing its own scheduler, cache management, and HTTP service. vLLM is the current inference backend. Material about vLLM, SGLang, and TensorRT-LLM under `foundry/` belongs to the Foundry subproject.

The HTTP service provides Chat Completions, Completions, Tokenize, Detokenize, model management, and runtime observability. Chat Completions supports model-specific function tool calls: requests accept `tools`, `tool_choice`, and historical `tool_calls`, while output parsing uses an available vLLM tool parser or an explicit `tool_parser` setting. Responses, Embeddings, Rerank, Audio, file uploads, Batch Jobs, and JSON Schema constrained output remain unimplemented.
