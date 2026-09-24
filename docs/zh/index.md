# 欢迎使用 Aegaeon

<div class="home-hero">
<div class="hero-wordmark"><svg viewBox="0 0 72 80" aria-hidden="true"><path d="M36 3 69 71H54L36 33 18 71H3Z" fill="#5368d6"/><path d="M27 56h18l7 15H20Z" fill="#91a1ef"/><path d="M36 3 45 22 36 33 27 22Z" fill="#bdc8ff"/></svg><span>Aegaeon</span></div>
<p class="hero-tagline">面向多模型服务的 Token 级推理引擎</p>
<p class="hero-subtitle">在共享 GPU 资源上交错执行多个大语言模型，统一管理调度、权重缓存与请求生命周期。</p>
<div class="hero-actions"><a class="primary-action" href="quickstart.md">快速开始 <span aria-hidden="true">→</span></a><a class="secondary-action" href="https://github.com/pkusys/aegaeon">GitHub <span aria-hidden="true">↗</span></a></div>
</div>

Aegaeon 是一个多模型推理与服务系统。它使用 vLLM 的模型实现与计算内核，通过 Ray 和 asyncio 管理执行资源，在模型切换、批次调度和 KV Cache 搬运之间协调多个模型的请求。

当前版本面向纯文本模型，采用贪心解码，Tensor Parallel 固定为 1。公开能力覆盖文本生成、模型管理和运行时观测；其余接口与并行模式的实现状态见[功能与兼容性](capabilities.md)。

## 从这里开始

<div class="home-cards">
<a href="quickstart.md"><span class="card-kicker">GET STARTED</span><strong>运行第一个模型</strong><span>安装环境，启动服务，发送第一条推理请求。</span><b aria-hidden="true">→</b></a>
<a href="deployment.md"><span class="card-kicker">SERVE MODELS</span><strong>部署多模型服务</strong><span>配置 GPU 引擎，管理模型副本与请求路由。</span><b aria-hidden="true">→</b></a>
<a href="architecture.md"><span class="card-kicker">UNDER THE HOOD</span><strong>了解系统架构</strong><span>探索 Token 级调度、权重缓存与 KV 管理。</span><b aria-hidden="true">→</b></a>
</div>

## 核心功能

- **多模型复用**：通过模型切换和执行配额，在固定引擎资源上交错处理多个模型。
- **灵活的执行模式**：使用 Simple Engine 完成两阶段推理，或分别部署 Prefill 与 Decode Engine。
- **动态模型管理**：部署和卸载节点级缓存副本，并在 READY 副本之间路由请求。
- **在线与流式推理**：提供 Chat、Completion 和 SSE 流式接口，支持贪心解码。
- **运行时可观测性**：查看引擎状态、模型放置、请求负载和调度事件。
- **可选优化**：在适用的 Decode 路径上使用 Work Stealing 或 Foundry 图存档。

配置参数和接口边界请参阅[功能与兼容性](capabilities.md)。

## 文档导航

| 文档 | 内容 |
|---|---|
| [用户指南](quickstart.md) | 安装、模型准备、在线推理、部署与排障 |
| [开发者指南](development.md) | 架构、调度、缓存实现与贡献流程 |
| [基准测试](benchmarking.md) | Profiling、测量口径与性能调优 |
| [API 参考](api.md) | HTTP 接口、请求字段与 Python API |
| [CLI 参考](cli.md) | 启动服务、部署和卸载模型 |
| [社区](community.md) | 问题反馈、功能建议与贡献 |
