# 欢迎使用 Aegaeon

<div class="home-hero">
<div class="hero-wordmark"><svg viewBox="0 0 72 80" aria-hidden="true"><path d="M36 3 69 71H54L36 33 18 71H3Z" fill="#5368d6"/><path d="M27 56h18l7 15H20Z" fill="#91a1ef"/><path d="M36 3 45 22 36 33 27 22Z" fill="#bdc8ff"/></svg><span>Aegaeon</span></div>
<p class="hero-tagline">面向多模型服务的 Token 级推理引擎</p>
<p class="hero-subtitle">通过 Token 级调度、快速模型切换与请求状态恢复，让多个大语言模型共享 GPU。</p>
<div class="hero-actions"><a class="primary-action" href="quickstart.md">快速开始 <span aria-hidden="true">→</span></a><a class="secondary-action" href="architecture.md">源码导览 <span aria-hidden="true">↗</span></a></div>
</div>

Aegaeon 面向并发 LLM 服务，通过 Token 级调度、快速权重加载和 KV 状态恢复，让多个模型共享 GPU Engine。项目介绍见 [SOSP '25 论文](https://doi.org/10.1145/3731569.3764815)。

## 功能

- [Token 级调度](scheduling.md)：平衡模型切换开销与 token 时延。
- [模型管理](model-management.md)：动态副本与感知负载的请求路由。
- [内存管理](memory.md)：分别管理 CPU 权重、请求 KV 状态和 GPU 执行内存。
- [CUDA Graph](cuda-graphs.md)：减少 Decode 提交开销。

## 文档

| 指南 | 内容 |
|---|---|
| [快速开始](quickstart.md) | 安装、配置与发送请求 |
| [部署](deployment.md) | 单节点与多节点服务 |
| [系统架构](architecture.md) | 组件与源码入口 |
| [基准测试](benchmarking.md) | 指标、Profiling 和性能工具 |
| [HTTP API](api.md) | 端点、schema 与响应 |
| [CLI](cli.md) | 启动、部署与查询 |
| [贡献指南](development.md) | 开发与文档维护 |

配置范围见[功能与兼容性](capabilities.md)。
