# HTTP API 参考

CLI 默认监听 0.0.0.0:8000，下文示例使用 `http://127.0.0.1:8000`。HTTP 服务允许所有 CORS origins、methods 和 headers；公网部署时，网关负责鉴权与访问限制。

## 接口清单

| 方法 | 路径 | 功能 |
|---|---|---|
| GET | /health | HTTP 进程健康响应；actor 状态见 runtime 接口 |
| GET | /v1/models | 已部署模型列表 |
| GET | /v1/models/{model_name:path} | 已部署模型元信息，支持名字含 / |
| POST | /v1/models/deploy | 部署 CPU 权重缓存副本 |
| POST | /v1/models/undeploy | 卸载指定或全部副本 |
| POST | /v1/chat/completions | Chat，非流式或 SSE |
| POST | /v1/completions | Legacy Completion，非流式或 SSE |
| POST | /tokenize | 将普通 prompt 编码为 token ID |
| POST | /detokenize | token IDs 解码 |
| GET | /v1/aegaeon/runtime | 集群状态快照 |
| GET | /v1/aegaeon/events | runtime SSE |
| GET | /v1/aegaeon/metrics | 60 秒窗口访问统计与限流设置 |
| GET | /v1/aegaeon/gpu-power | API 所在机器的 GPU 采样 |

FastAPI 还提供 `/docs`、`/redoc` 和 `/openapi.json`，用于查看 schema。生成参数的允许值、请求生命周期和部分失败时的返回结果见本章说明。

## 模型列表

```json
{"object":"list","data":[{"id":"Qwen/Qwen3-4B","object":"model","created":0,"owned_by":"aegaeon"}]}
```

示例用 `created=0` 展示字段结构，实际值取查询时间。模型列表显示服务登记的模型；各节点的 READY、DRAINING 等状态见 runtime 快照。

## Deploy / Undeploy

```bash
curl -sS http://127.0.0.1:8000/v1/models/deploy \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","model_path":"/srv/models/Qwen3-4B","replica_count":1,"node_ids":["node_0"]}'
curl -sS http://127.0.0.1:8000/v1/models/undeploy \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","node_ids":["node_0"]}'
```

Deploy 请求包含 `model`（必填）、`model_path`、`max_model_len`、`node_ids`、`replica_count`（至少为 1，默认 1）和 `tool_parser`。Undeploy 使用同一 schema，但只读取 `model` 和 `node_ids`。模型已有副本时，`model_path` 沿用原值；修改 `max_model_len` 会触发 force 部署流程，`tool_parser` 只更新该模型别名的解析器元数据。

完整成功和部分成功都返回 HTTP 200，JSON `status` 分别为 `ok` 和 `partial`。部署响应包含 `ready_nodes`、`failed` 和操作结果；卸载响应包含 `remaining_nodes`、`failed` 和操作结果。客户端应读取 `status`，判断是否所有节点都成功。若所有部署目标失败，且没有现存的 READY 副本，接口返回 507。副本状态的变化见[生命周期](model-management.md)。

## Tokenize / Detokenize

```bash
curl -sS http://127.0.0.1:8000/tokenize \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","prompt":"hello","add_special_tokens":true}'
curl -sS http://127.0.0.1:8000/detokenize \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","tokens":[123],"skip_special_tokens":true}'
```

Tokenize 返回 `count`、`tokens` 和 `max_model_len`，其中 `max_model_len` 当前固定为 null。它直接处理普通字符串；聊天模板由 Chat Completions 接口应用。Detokenize 返回 `prompt`。token ID 的含义取决于所选 tokenizer。

## Chat / Completion 响应

Chat 响应包含 `id`、`object='chat.completion'`、`created`、`model`、`choices` 和 `usage`。每个 choice 包含 `index`、`message`、`logprobs=null` 和 `finish_reason`；Completion 的 `object='text_completion'`，choice 中使用 `text`。普通生成的 `finish_reason` 为 `stop` 或 `length`。Chat 成功解析出函数调用时，`finish_reason='tool_calls'`，调用内容放在 `message.tool_calls`。服务端返回调用信息，但不执行函数；客户端执行后，须自行追加 assistant 调用和 tool 结果消息。

`return_token_ids=true` 时，Chat 非流式响应在顶层返回 prompt_token_ids、在 choice 内返回 token_ids；Completion 非流式响应把两种 ID 都放在 choice 内。Chat 流式响应在首个 chunk 返回 prompt_token_ids、在最终 choice 返回 token_ids。Completion 流式响应在最终 choice 返回两种 ID；启用 echo 时，首个 echo choice 还会包含 prompt_token_ids。

`stream=true` 时，接口用 `data: <JSON>` 发送 SSE，最后发送 `data: [DONE]`。设置 `stream_options.include_usage=true` 后，结束前还会收到一个 `choices=[]` 的 usage chunk。Chat chunk 使用 `delta`，Completion chunk 使用 `text`。

启用 tool parser 的 Chat 会缓冲生成文本，完成后才发送解析所得的 `delta.content` 和/或 `delta.tool_calls`，因此不保证逐 token 输出。未解析出调用时返回普通文本。工具参数与解析条件见[在线推理](serving.md#常见接入问题)。

## Runtime 与监控指标

`GET /v1/aegaeon/runtime` 返回运行快照，包括 `timestamp`、`nodes`、`engines`、`model_placements`、`request_routing`、`model_placement_stats`、`decode_load`、`work_stealing` 和 `events` 等字段。`GET /v1/aegaeon/events` 则持续发送 SSE：事件名为 `runtime`，JSON `data` 包含 `engines` 和 `events`。状态不变时，服务端约每 15 秒发送一次 `: keep-alive`。

`GET /v1/aegaeon/metrics` 返回 60 秒窗口内的逐模型 `metrics` 和 `limits`。`GET /v1/aegaeon/gpu-power` 采样 API 进程所在机器的 GPU，范围不包括其他节点。

## 错误形式

| HTTP 状态 | 常见触发 |
|---|---|
| 400 | 模型未部署、非法贪心参数、模板失败、非法节点 / 模型路径 |
| 422 | schema 类型、生成或 tokenizer 请求的未知字段、Field 约束校验失败 |
| 429 | 访问窗口限流 |
| 503 | 模型已登记，当前可路由 READY 副本数为 0 |
| 507 | 所有部署目标失败，且现存 READY 副本数为 0 |
| 500 等 | 内部、下载或 Ray 异常；状态码按原始异常路径返回 |

显式 `_error()` 返回的错误采用 FastAPI `detail` 包装，格式如下：

```json
{"detail":{"message":"...","type":"invalid_request_error","param":"temperature","code":null}}
```

FastAPI 的 schema 校验错误在 `detail` 中返回列表；显式 `_error()` 返回对象，部分 HTTPException 分支返回字符串。客户端应保留 HTTP 状态码和原始响应体。

生成与 tokenizer 请求使用 `extra="forbid"`：传入 `response_format`、`chat_template_kwargs` 等未声明字段会得到 422。部署和卸载使用另一请求模型，未知字段按 Pydantic 默认行为被忽略。非贪心取值及不合法的 tool 配置通常返回 400。字段和默认值见[请求 Schema](request-schema.md)，取值限制见[生成参数](generation.md)。

## 内置页面与外部代理

`/aegaeon_console.html`、`/aegaeon_chat.html`、`/aegaeon_docs.html` 为仓库内置静态页面，主入口及 `/demo` 有重定向。文档内容维护于 `docs/zh` 和 `docs/en`，更新后通过 `docs/build_portable.py` 重新生成内置页面，具体命令见[文档维护](development.md#文档维护)。

另有 `/v1/serverlessllm/models`、`/v1/serverlessllm/gpu-power`、`/v1/serverlessllm/chat/completions` 代理演示接口，由 SERVERLESSLLM_URL 指向外部服务（默认 127.0.0.1:8343）。Aegaeon 原生能力以 `/v1/aegaeon/*` 和标准推理路由为准。
