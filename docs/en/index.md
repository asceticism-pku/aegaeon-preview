# Welcome to Aegaeon

<div class="home-hero">
<div class="hero-wordmark"><svg viewBox="0 0 72 80" aria-hidden="true"><path d="M36 3 69 71H54L36 33 18 71H3Z" fill="#5368d6"/><path d="M27 56h18l7 15H20Z" fill="#91a1ef"/><path d="M36 3 45 22 36 33 27 22Z" fill="#bdc8ff"/></svg><span>Aegaeon</span></div>
<p class="hero-tagline">Token-Level Inference for Multi-Model Serving</p>
<p class="hero-subtitle">Interleave large language models on shared GPU resources, coordinating scheduling, weight caching, and request lifecycles.</p>
<div class="hero-actions"><a class="primary-action" href="quickstart.md">Get Started <span aria-hidden="true">→</span></a><a class="secondary-action" href="https://github.com/pkusys/aegaeon">GitHub <span aria-hidden="true">↗</span></a></div>
</div>

Aegaeon is a multi-model inference and serving system. It uses vLLM model implementations and compute kernels, with Ray and asyncio managing execution resources across model switches, batch scheduling, and KV Cache transfers.

The current release serves text-only models with greedy decoding and Tensor Parallel fixed at 1. Chat Completions includes model-specific function tool calling through vLLM tool parsers. Multimodal input, random sampling, Pipeline Parallel, and quantization configuration are outside the supported scope. See [Features and Compatibility](capabilities.md) for the complete boundary.

## Start here

<div class="home-cards">
<a href="quickstart.md"><span class="card-kicker">GET STARTED</span><strong>Run your first model</strong><span>Install the environment, start the service, and send your first inference request.</span><b aria-hidden="true">→</b></a>
<a href="deployment.md"><span class="card-kicker">SERVE MODELS</span><strong>Deploy multiple models</strong><span>Configure GPU engines, manage replicas, and route requests.</span><b aria-hidden="true">→</b></a>
<a href="architecture.md"><span class="card-kicker">UNDER THE HOOD</span><strong>Explore the architecture</strong><span>Understand token-level scheduling, weight caches, and KV management.</span><b aria-hidden="true">→</b></a>
</div>

## Core features

- **Multi-model multiplexing**: interleave requests from multiple models on fixed engine resources through model switching and execution quotas.
- **Flexible execution modes**: handle both inference stages with Simple Engines or deploy separate Prefill and Decode Engines.
- **Dynamic model management**: deploy and unload node-level cached replicas and route requests among READY replicas.
- **Online and streaming inference**: Chat, Completion, and SSE streaming interfaces with greedy decoding.
- **Runtime observability**: inspect engine state, model placements, request loads, and scheduling events.
- **Optional optimizations**: Work Stealing or Foundry graph archives on applicable Decode paths.

See [Features and Compatibility](capabilities.md) for configuration and interface details.

## Documentation

| Guide | Topics |
|---|---|
| [User Guide](quickstart.md) | Installation, model preparation, online inference, deployment, and troubleshooting |
| [Developer Guide](development.md) | Architecture, scheduling, cache implementation, and contributions |
| [Benchmarking](benchmarking.md) | Profiling, measurement definitions, and tuning |
| [API Reference](api.md) | HTTP endpoints, request fields, and Python APIs |
| [CLI Reference](cli.md) | Starting services, deploying, and unloading models |
| [Community](community.md) | Bug reports, feature proposals, and contributions |
