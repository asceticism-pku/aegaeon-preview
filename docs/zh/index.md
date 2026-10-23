# 欢迎使用 Aegaeon

<div class="home-hero">
<div class="hero-wordmark"><svg viewBox="0 0 72 80" aria-hidden="true"><path d="M36 3 69 71H54L36 33 18 71H3Z" fill="#5368d6"/><path d="M27 56h18l7 15H20Z" fill="#91a1ef"/><path d="M36 3 45 22 36 33 27 22Z" fill="#bdc8ff"/></svg><span>Aegaeon</span></div>
<p class="hero-tagline">面向多模型服务的 Token 级推理引擎</p>
<p class="hero-subtitle">通过 Token 级调度、快速模型切换与请求状态恢复，让多个大语言模型共享 GPU。</p>
<div class="hero-actions"><a class="primary-action" href="quickstart.md">快速开始 <span aria-hidden="true">→</span></a><a class="secondary-action" href="architecture.md">系统架构 <span aria-hidden="true">↗</span></a></div>
</div>

Aegaeon 面向并发 LLM 服务，通过 Token 级调度、快速权重加载和 KV 状态恢复，让多个模型共享 GPU Engine。[SOSP '25 论文](https://doi.org/10.1145/3731569.3764815)介绍了系统设计。

## 核心功能

- **Token 级 GPU 共享**：活跃模型轮流使用共享 GPU Engine，长请求可在生成过程中让出算力。详见[调度与 SLO](scheduling.md)。
- **Decode 时延调度**：实测延迟 profile 和估算的切换成本共同决定 batch 执行时长配额，平衡切换开销与输出 token 间隔（TPOT）目标。
- **可恢复的模型切换**：CPU 权重缓存、预取和异步 KV 卸载/重载，在模型切换期间保留进行中的请求状态。详见[内存与 KV Cache](memory.md)。
- **Prefill/Decode 分离**：P/D 模式为 Prefill 和 Decode 分配独立 Engine 池，Simple 模式在同一 Engine 内完成两个阶段。详见[系统架构](architecture.md)。

## 文档

| 指南 | 内容 |
|---|---|
| [快速开始](quickstart.md) | 安装、配置与发送请求 |
| [部署](deployment.md) | 单节点与多节点服务 |
| [系统架构](architecture.md) | 请求流程与运行组件 |
| [基准测试](benchmarking.md) | 指标、Profiling 和性能工具 |
| [HTTP API](api.md) | 端点、schema 与响应 |
| [CLI](cli.md) | 启动、部署与查询 |

配置范围见[功能与兼容性](capabilities.md)。
