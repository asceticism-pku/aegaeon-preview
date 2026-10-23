# Welcome to Aegaeon

<div class="home-hero">
<div class="hero-wordmark"><svg viewBox="0 0 72 80" aria-hidden="true"><path d="M36 3 69 71H54L36 33 18 71H3Z" fill="#5368d6"/><path d="M27 56h18l7 15H20Z" fill="#91a1ef"/><path d="M36 3 45 22 36 33 27 22Z" fill="#bdc8ff"/></svg><span>Aegaeon</span></div>
<p class="hero-tagline">Token-Level Inference for Multi-Model Serving</p>
<p class="hero-subtitle">Serve many LLMs on shared GPUs with token-level scheduling and fast, resumable model switching.</p>
<div class="hero-actions"><a class="primary-action" href="quickstart.md">Get Started <span aria-hidden="true">→</span></a><a class="secondary-action" href="architecture.md">Architecture <span aria-hidden="true">↗</span></a></div>
</div>

Aegaeon pools GPUs for concurrent LLM serving. Token-level scheduling, fast weight loading, and KV restoration let multiple models share the same engines. The [SOSP '25 paper](https://doi.org/10.1145/3731569.3764815) describes the system.

## Core features

- **Token-level GPU pooling**: active models take turns on shared GPU engines, so long generations can yield capacity to other models. See [Scheduling and SLOs](scheduling.md).
- **Decode latency scheduling**: measured latency profiles and estimated switching costs guide batch execution quotas, balancing switching overhead against the target time per output token (TPOT).
- **Resumable model switching**: host-side weight caching, prefetching, and asynchronous KV offload/reload preserve in-flight requests across model switches. See [Memory and KV Cache](memory.md).
- **Prefill/Decode separation**: P/D mode assigns separate engine pools to Prefill and Decode. Simple mode runs both stages in one engine. See [System Architecture](architecture.md).

## Documentation

| Guide | Contents |
|---|---|
| [Quickstart](quickstart.md) | Install, configure, and send a request |
| [Deployment](deployment.md) | Single-node and multi-node serving |
| [Architecture](architecture.md) | Request flow and runtime components |
| [Benchmarking](benchmarking.md) | Metrics, profiling, and performance tools |
| [HTTP API](api.md) | Endpoints, schemas, and responses |
| [CLI](cli.md) | Start, deploy, and query |

See [Features and Compatibility](capabilities.md) for the supported configuration.
