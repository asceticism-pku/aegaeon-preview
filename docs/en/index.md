# Welcome to Aegaeon

<div class="home-hero">
<div class="hero-wordmark"><svg viewBox="0 0 72 80" aria-hidden="true"><path d="M36 3 69 71H54L36 33 18 71H3Z" fill="#5368d6"/><path d="M27 56h18l7 15H20Z" fill="#91a1ef"/><path d="M36 3 45 22 36 33 27 22Z" fill="#bdc8ff"/></svg><span>Aegaeon</span></div>
<p class="hero-tagline">Token-Level Inference for Multi-Model Serving</p>
<p class="hero-subtitle">Serve many LLMs on shared GPUs with token-level scheduling and fast, resumable model switching.</p>
<div class="hero-actions"><a class="primary-action" href="quickstart.md">Get Started <span aria-hidden="true">→</span></a><a class="secondary-action" href="architecture.md">Source Guide <span aria-hidden="true">↗</span></a></div>
</div>

Aegaeon pools GPUs for concurrent LLM serving. Token-level scheduling, fast weight loading, and KV restoration let multiple models share the same engines. Read the [SOSP '25 paper](https://doi.org/10.1145/3731569.3764815).

## Features

- [Token-level scheduling](scheduling.md) balances switching costs and token latency.
- [Model management](model-management.md) provides dynamic replicas and load-aware routing.
- [Memory management](memory.md) separates CPU weights, request KV state, and GPU execution.
- [CUDA Graphs](cuda-graphs.md) reduce Decode launch overhead.

## Documentation

| Guide | Contents |
|---|---|
| [Quickstart](quickstart.md) | Install, configure, and send a request |
| [Deployment](deployment.md) | Single-node and multi-node serving |
| [Architecture](architecture.md) | Components and source entry points |
| [Benchmarking](benchmarking.md) | Metrics, profiling, and performance tools |
| [HTTP API](api.md) | Endpoints, schemas, and responses |
| [CLI](cli.md) | Start, deploy, and query |
| [Contributing](development.md) | Development and documentation |

See [Features and Compatibility](capabilities.md) for the supported configuration.
