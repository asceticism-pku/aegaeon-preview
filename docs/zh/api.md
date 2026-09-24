# HTTP API 参考

默认 CLI 监听 0.0.0.0:8000；本文示例统一使用 `http://127.0.0.1:8000`。HTTP 服务允许全部 CORS origins、methods 和 headers，鉴权与访问限制由公网部署的网关负责。

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
| POST | /tokenize | token 化普通 prompt |
| POST | /detokenize | token IDs 解码 |
| GET | /v1/aegaeon/runtime | 集群运行快照 |
| GET | /v1/aegaeon/events | runtime SSE |
| GET | /v1/aegaeon/metrics | 60 秒窗口访问统计与限流设置 |
| GET | /v1/aegaeon/gpu-power | API 所在机器的 GPU 采样 |

FastAPI 同时提供 `/docs`、`/redoc` 和 `/openapi.json`。这些页面展示 schema；生成参数取值、生命周期和部分失败语义以本章说明为准。

## 模型列表

```json
{"object":"list","data":[{"id":"Qwen/Qwen3-4B","object":"model","created":0,"owned_by":"aegaeon"}]}
```

示例中的 `created=0` 仅用于展示结构，实际值取查询时间。模型列表展示服务登记项；READY、DRAINING 等节点状态位于 runtime 快照。

## Deploy / Undeploy

```bash
curl -sS http://127.0.0.1:8000/v1/models/deploy \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","model_path":"/srv/models/Qwen3-4B","replica_count":1,"node_ids":["node_0"]}'
curl -sS http://127.0.0.1:8000/v1/models/undeploy \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","node_ids":["node_0"]}'
```

Deploy 字段为 model（必填）、model_path、max_model_len、node_ids、replica_count（>=1，默认1）和 tool_parser。Undeploy 复用该 schema，并读取 model / node_ids。已有副本的模型沿用原 model_path；更新 max_model_len 会触发 force 部署流程，tool_parser 仅更新该模型别名使用的解析器元数据。

完整成功和部分成功都返回 HTTP 200，JSON `status` 分别为 `ok` 和 `partial`。部署响应包含 `ready_nodes`、`failed` 和操作结果；卸载响应包含 `remaining_nodes`、`failed` 和操作结果。客户端应检查 JSON `status` 判断是否全部节点成功。全部部署目标失败且 READY 副本数为 0 时返回 507。见[生命周期](model-management.md)。

## Tokenize / Detokenize

```bash
curl -sS http://127.0.0.1:8000/tokenize \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","prompt":"hello","add_special_tokens":true}'
curl -sS http://127.0.0.1:8000/detokenize \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","tokens":[123],"skip_special_tokens":true}'
```

Tokenize 返回 count、tokens、max_model_len，其中 max_model_len 当前固定为 null。输入按普通字符串直接执行 tokenization；聊天消息的模板处理位于 Chat Completions 路径。Detokenize 返回 prompt；token ID 的含义由具体 tokenizer 决定。

## Chat / Completion 响应

Chat 包含 id、object='chat.completion'、created、model、choices、usage；每个 choice 包含 index、message、logprobs=null、finish_reason。Completion 使用 object='text_completion'，choice 中是 text。普通生成的 `finish_reason` 为 `stop` 或 `length`；成功解析出函数调用时，Chat 返回 `finish_reason='tool_calls'`，并在 `message.tool_calls` 中给出调用。服务端不执行函数；客户端执行后须自行追加 assistant 调用与 tool 结果消息。

`return_token_ids=true` 时，Chat 非流式响应在顶层返回 prompt_token_ids、在 choice 内返回 token_ids；Completion 非流式响应把两种 ID 都放在 choice 内。Chat 流式响应在首个 chunk 返回 prompt_token_ids、在最终 choice 返回 token_ids。Completion 流式响应在最终 choice 返回两种 ID；启用 echo 时，首个 echo choice 还会包含 prompt_token_ids。

`stream=true` 使用 `data: <JSON>` SSE，并以 `data: [DONE]` 结束；`stream_options.include_usage=true` 会在结束前发送 `choices=[]` 的 usage chunk。Chat chunk 使用 `delta`，Completion chunk 使用 `text`。启用 tool parser 的 Chat 会等生成完成后发送解析所得的 `delta.content` 和/或 `delta.tool_calls`，不保证逐 token 输出；未解析出调用时返回普通文本。工具参数与解析条件见[在线推理](serving.md#常见接入问题)。

## 运行时与指标

`GET /v1/aegaeon/runtime` 返回 `timestamp`、`nodes`、`engines`、`model_placements`、`request_routing`、`model_placement_stats`、`decode_load`、`work_stealing` 和 `events` 等快照字段。`GET /v1/aegaeon/events` 是另一条 SSE 流，使用 `event: runtime` 和包含 `engines`、`events` 的 JSON `data`；空闲时可发送注释心跳。`GET /v1/aegaeon/metrics` 返回 60 秒窗口的 `limits` 与逐模型 `metrics`。`GET /v1/aegaeon/gpu-power` 读取 API 进程所在机器的 GPU 采样，不代表所有节点。

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

FastAPI schema 校验错误的 `detail` 为列表，显式 `_error()` 的 `detail` 为对象，部分 HTTPException 分支使用字符串。客户端应保留 HTTP status 和原始 body。

生成与 tokenizer 请求使用 `extra="forbid"`，因此 `response_format`、`chat_template_kwargs` 等未声明字段返回 422；部署和卸载使用另一请求模型，未知字段按 Pydantic 默认行为被忽略。非贪心取值及不合法的 tool 配置通常返回 400。字段和默认值见[请求 Schema](request-schema.md)，取值限制见[生成参数](generation.md)。

## 内置页面与外部代理

`/aegaeon_console.html`、`/aegaeon_chat.html`、`/aegaeon_docs.html` 为仓库内置静态页面，主入口及 `/demo` 有重定向。文档内容维护于 `docs/zh` 和 `docs/en`，更新后通过 `docs/build_portable.py` 重新生成内置页面，具体命令见[文档维护](development.md#文档维护)。

另有 `/v1/serverlessllm/models`、`/v1/serverlessllm/gpu-power`、`/v1/serverlessllm/chat/completions` 代理演示接口，由 SERVERLESSLLM_URL 指向外部服务（默认 127.0.0.1:8343）。Aegaeon 原生能力以 `/v1/aegaeon/*` 和标准推理路由为准。
