# Welcome to Aegaeon

<div class="home-hero">
<div class="hero-wordmark"><svg viewBox="0 0 72 80" aria-hidden="true"><path d="M36 3 69 71H54L36 33 18 71H3Z" fill="#5368d6"/><path d="M27 56h18l7 15H20Z" fill="#91a1ef"/><path d="M36 3 45 22 36 33 27 22Z" fill="#bdc8ff"/></svg><span>Aegaeon</span></div>
<p class="hero-tagline">Token-Level Inference for Multi-Model Serving</p>
<p class="hero-subtitle">Serve many LLMs on shared GPUs with token-level scheduling and fast, resumable model switching.</p>
<div class="hero-actions"><a class="primary-action" href="quickstart.md">Get Started <span aria-hidden="true">→</span></a><a class="secondary-action" href="https://github.com/pkusys/aegaeon">GitHub <span aria-hidden="true">↗</span></a></div>
</div>

Aegaeon is a multi-LLM serving system for workloads where many models receive sparse, uneven, or bursty traffic. It pools GPU capacity across models and can pause and resume in-flight generation at token boundaries, so another model can run while earlier requests are still active. This makes GPU sharing practical for a long tail of models and reduces head-of-line blocking from request-level model switching.

Model weights are cached in host memory and prefetched for upcoming execution, while request KV Cache is moved between CPU and GPU as models switch. Together with profile-guided Decode scheduling, these mechanisms let the system reuse the same GPU engines across models without keeping every model resident on the GPU.

The [SOSP '25 paper](https://doi.org/10.1145/3731569.3764815) reports that Aegaeon sustained 2–2.5× the request arrival rate of ServerlessLLM under the evaluated SLO targets. In the paper's Alibaba Cloud Model Studio beta deployment, it reduced the GPU count by 82% (1,192 to 213).

## Start here

<div class="home-cards">
<a href="quickstart.md"><span class="card-kicker">GET STARTED</span><strong>Run your first model</strong><span>Install the environment, start the service, and send your first inference request.</span><b aria-hidden="true">→</b></a>
<a href="deployment.md"><span class="card-kicker">SERVE MODELS</span><strong>Deploy multiple models</strong><span>Configure GPU engines, manage replicas, and route requests.</span><b aria-hidden="true">→</b></a>
<a href="architecture.md"><span class="card-kicker">UNDER THE HOOD</span><strong>Explore the architecture</strong><span>Understand token-level scheduling, weight caches, and KV management.</span><b aria-hidden="true">→</b></a>
</div>

## Core features

- **Token-level GPU pooling**: multiplex active models on shared engines at Decode round and execution-quota boundaries, allowing long-running generations to yield GPU time to other models. See [Scheduling and SLOs](scheduling.md).
- **Decode scheduling for token latency**: use measured latency profiles and estimated model-switching costs to set per-batch execution quotas, balancing switching overhead against the target time per output token (TPOT).
- **Fast, resumable model switching**: combine host-side weight caching, next-model prefetching, and asynchronous KV Cache offload/reload to switch models while preserving in-flight requests. See [Memory, Weight Caches, and KV Cache](memory.md).
- **Independent Prefill and Decode pools**: allocate GPU engines to prompt processing and token generation separately to reduce interference between the two stages. Simple mode provides a combined engine for both stages. See [System Architecture](architecture.md).

## Current release

The current release uses vLLM model implementations and compute kernels and serves text-only models with greedy decoding, with Tensor Parallel and Pipeline Parallel fixed at 1. It provides Chat and Completion APIs, SSE streaming, dynamic model management, runtime observability, and model-specific function tool calling through vLLM tool parsers. Multimodal input, random sampling, and quantization configuration are outside the supported scope. See [Features and Compatibility](capabilities.md) for the supported model and interface scope, parallelism constraints, and optional optimizations.

## Documentation

| Guide | Topics |
|---|---|
| [User Guide](quickstart.md) | Installation, model preparation, online inference, deployment, and troubleshooting |
| [Developer Guide](development.md) | Architecture, scheduling, cache implementation, and contributions |
| [Benchmarking](benchmarking.md) | Profiling, measurement definitions, and tuning |
| [API Reference](api.md) | HTTP endpoints, request fields, and Python APIs |
| [CLI Reference](cli.md) | Starting services, deploying, and unloading models |
| [Community](community.md) | Bug reports, feature proposals, and contributions |
