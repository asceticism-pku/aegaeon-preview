# 欢迎使用 Aegaeon

<div class="home-hero">
<div class="hero-wordmark"><svg viewBox="0 0 72 80" aria-hidden="true"><path d="M36 3 69 71H54L36 33 18 71H3Z" fill="#5368d6"/><path d="M27 56h18l7 15H20Z" fill="#91a1ef"/><path d="M36 3 45 22 36 33 27 22Z" fill="#bdc8ff"/></svg><span>Aegaeon</span></div>
<p class="hero-tagline">面向多模型服务的 Token 级推理引擎</p>
<p class="hero-subtitle">通过 Token 级调度、快速模型切换与请求状态恢复，让多个大语言模型共享 GPU。</p>
<div class="hero-actions"><a class="primary-action" href="quickstart.md">快速开始 <span aria-hidden="true">→</span></a><a class="secondary-action" href="source-reference.md">源码导览 <span aria-hidden="true">↗</span></a></div>
</div>

Aegaeon 是面向多模型服务的 GPU 池化系统，适用于大量模型请求稀疏、负载不均或流量突发的场景。它在 Token 生成边界暂停与恢复进行中的请求，让其他模型在已有请求尚未结束时获得 GPU 执行机会。这样既能让长尾模型共享计算资源，也能减少请求级模型切换造成的队头阻塞。

Aegaeon 将模型权重缓存在主机内存中，并为后续执行预取权重；切换模型时，通过 CPU 与 GPU 之间的 KV Cache 搬运保留请求状态。结合基于性能 Profile 的 Decode 调度，同一组 GPU 引擎可以轮流服务多个模型，而无需让全部模型常驻显存。

[SOSP '25 论文](https://doi.org/10.1145/3731569.3764815)（[作者 PDF](https://ennanzhai.github.io/pub/sosp25-aegaeon.pdf)）报告：在满足论文评测的 SLO 目标时，Aegaeon 可承载的请求到达率为 ServerlessLLM 的 2–2.5 倍；在论文所述的阿里云百炼（Model Studio）beta 部署中，GPU 数量由 1,192 张降至 213 张，减少 82%。

## 从这里开始

<div class="home-cards">
<a href="quickstart.md"><span class="card-kicker">GET STARTED</span><strong>运行第一个模型</strong><span>安装环境，启动服务，发送第一条推理请求。</span><b aria-hidden="true">→</b></a>
<a href="deployment.md"><span class="card-kicker">SERVE MODELS</span><strong>部署多模型服务</strong><span>配置 GPU 引擎，管理模型副本与请求路由。</span><b aria-hidden="true">→</b></a>
<a href="architecture.md"><span class="card-kicker">UNDER THE HOOD</span><strong>了解系统架构</strong><span>探索 Token 级调度、权重缓存与 KV 管理。</span><b aria-hidden="true">→</b></a>
</div>

## 核心功能

- **Token 级 GPU 池化**：在 Decode 轮次与执行配额边界切换活跃模型，让长请求在生成过程中让出 GPU 时间，多个模型共享同一组引擎。详见[调度与 SLO](scheduling.md)。
- **面向 Token 延迟的 Decode 调度**：利用实测性能 Profile 和模型切换开销估计，为批次分配执行配额，权衡切换成本与单 Token 输出时间（TPOT）目标。
- **快速且可恢复的模型切换**：结合主机内存权重缓存、下一模型预取和异步 KV Cache 换出与换入，在切换模型时保留进行中请求的状态。详见[内存、权重缓存与 KV Cache](memory.md)。
- **独立的 Prefill 与 Decode 资源池**：分别为提示词处理和 Token 生成分配 GPU 引擎，减少两阶段之间的干扰；Simple 模式则由同一引擎完成两阶段。详见[系统架构](architecture.md)。

## 当前版本

当前公开验证的支持范围为纯文本模型与贪心解码，使用 vLLM 的模型实现与计算内核，Tensor Parallel 和 Pipeline Parallel 均固定为 1。提供 Chat、Completion、SSE 流式输出、动态模型管理和运行时观测，并通过 vLLM 工具解析器支持模型相关的函数工具调用。多模态输入、随机采样和量化配置均在当前支持范围之外。模型与接口范围、并行约束及可选优化见[功能与兼容性](capabilities.md)。

## 文档导航

| 文档 | 内容 |
|---|---|
| [用户指南](quickstart.md) | 安装、模型准备、在线推理、部署与排障 |
| [开发者指南](development.md) | 架构、调度、缓存实现与贡献流程 |
| [基准测试](benchmarking.md) | Profiling、测量口径与性能调优 |
| [API 参考](api.md) | HTTP 接口、请求字段与 Python API |
| [CLI 参考](cli.md) | 启动服务、部署和卸载模型 |
| [社区](community.md) | 问题反馈、功能建议与贡献 |
