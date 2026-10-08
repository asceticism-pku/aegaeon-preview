# 功能与兼容性

## 支持范围

| 能力 | 当前状态 | 实现位置 |
|---|---|---|
| 多模型交错推理 | 支持。按批次、轮次和执行配额切换模型 | `simple_scheduler.py`、`decode_scheduler.py` |
| Simple 模式 | 支持。一个 Engine 完成 Prefill 和 Decode | `stage_engine.py`、`simple_dispatcher.py` |
| Prefill/Decode 分离 | 支持。节点内 CPU KV Cache 负责阶段间中转 | `llm.py`、`block_manager.py` |
| 多物理节点 | 支持。每个节点运行一个 Controller，请求固定在选中的节点内 | `llm.py`、`model_placement.py` |
| 动态部署与卸载 | 支持节点级缓存副本、排空状态和部分失败结果 | `api.py` |
| 多副本请求路由 | 支持 READY 副本间的 least-outstanding 和 round-robin | `model_placement.py` |
| Tensor Parallel | **仅支持 `tensor_parallel_size=1`** | `estimator.py`、`config.py` |
| Pipeline Parallel | 未实现；`pipeline_parallel_size` 固定为 1 | `config.py` |
| Chat / Completion | 支持部分 OpenAI 风格接口，字段范围见[请求 Schema](request-schema.md) | `api.py` |
| SSE 流式输出 | 支持。每个事件携带文本增量或最终状态；一个事件可对应多个 token | `api.py` |
| 贪心解码 | 支持。Worker 对 logits 执行 argmax | `worker.py` |
| 随机采样 / 多候选 | 当前仅提供贪心解码，`n` 固定为 1 | `_validate_greedy_parameters` |
| Logprobs / beam search | 未实现 | `api.py` |
| 多模态输入 | 当前仅提供文本输入与文本生成的公开支持；存在内容块、renderer 与 Worker 处理路径，具体多模态模型未纳入公开验收 | `api.py`、`worker.py` |
| 前缀缓存 | 当前关闭；在 vLLM CacheConfig 中显式禁用 | `config.py` |
| 量化配置 | 当前固定为 `quantization=None` | `config.py` |
| Decode Work Stealing | 可选，默认关闭；仅在同一 Controller 内转移符合条件的完整 batch | `decode_dispatcher.py` |
| Foundry 图存档 | 支持 P/D 模式的 Decode Worker；Simple Worker 走常规路径 | `stage_engine.py` |
| Runtime 快照 / 事件 | 支持 JSON 快照与 SSE 事件流 | `api.py` |
| Prometheus 指标 | 未集成 | `api.py` |
| API key 鉴权 | Aegaeon 服务端省略 key 校验；需要鉴权时由部署侧网关实现 | `api.py` |

## 并行约束

Aegaeon 当前只支持 `tensor_parallel_size=1`。配置解析器虽然接受更大的整数，时延估计器仍执行 `assert tp == 1`；调度、KV Cache 和 Foundry 也只将 TP=1 纳入支持范围。部署配置、性能数据和问题报告都应使用 TP=1。

Pipeline Parallel 同样固定为 1。增加 Simple Engine 或 Prefill/Decode Engine 可提供更多执行单元；吞吐量效果还取决于模型切换、工作负载与资源预算，需要实测。每个 Engine 对应一个 Worker 和一张 GPU。

## 模型与输入范围

公开接口支持纯文本语言模型、字符串消息和文本生成。Chat 请求应用 tokenizer 的聊天模板；Completion 接受字符串或一组 token ID。

`Message.content` 也接受内容块列表，`mm_processor_kwargs` 会随这类输入传给 vLLM renderer/processor，Worker 中也有相应处理路径。具体多模态模型尚未纳入公开验收范围。

接入一个模型前，需要确认其配置、权重加载、性能 profile、聊天模板、EOS、KV 类型和目标拓扑均正常。新模型的验收步骤见[模型准备与注册](models.md)。

## 接口边界

HTTP 服务提供 Chat Completions、Completions、Tokenize、Detokenize、模型管理和运行时观测接口。Chat Completions 接受 `tools`、`tool_choice` 和历史 `tool_calls`。解析模型输出中的函数调用时，需要可用的 vLLM tool parser，或显式设置 `tool_parser`。Responses、Embeddings、Rerank、Audio、文件上传、Batch Jobs 和 JSON Schema 约束输出当前尚未实现。
