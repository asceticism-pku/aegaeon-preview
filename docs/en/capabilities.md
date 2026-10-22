# Features and Compatibility

## Supported scope

| Capability | Support |
|---|---|
| Multi-model inference | Token-level scheduling and model switching |
| Deployment | Simple or Prefill/Decode mode across one or more nodes |
| Model management | Dynamic deployment, unloading, replicas, and request routing |
| HTTP | OpenAI-style Chat/Completion, SSE, tokenization, and runtime monitoring |
| Generation | Greedy decoding with one candidate |
| Function tools | Model-specific vLLM parsers; client-side execution |
| Decode Work Stealing | Optional node-local batch balancing |
| CUDA Graphs | Foundry save/load on P/D Decode Workers |
| Multimodal input | **Unsupported** in the public serving interface |
| Quantization, random sampling, beam search, logprobs | Outside the supported configuration |

Only `tensor_parallel_size=1` is supported. Pipeline parallelism is fixed to 1; use additional engines for more GPUs.

See [Model Preparation](models.md), [Generation Parameters](generation.md), and the [HTTP API](api.md) for usage.
