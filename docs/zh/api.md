# HTTP API

服务默认使用 8000 端口。运行中服务的 `/docs`、`/redoc` 和 `/openapi.json` 提供完整请求类型和默认值。鉴权由部署侧网关配置。

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

## 模型列表

`GET /v1/models` 返回已注册的模型名称；Runtime 快照包含各节点的副本状态。

## 部署与卸载

```bash
curl -sS http://127.0.0.1:8000/v1/models/deploy \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","model_path":"/srv/models/Qwen3-4B","replica_count":1}'
```

向 `/v1/models/undeploy` 传入 `model` 和可选的 `node_ids` 卸载副本。响应包含 `status`、各节点结果及 `failed`；HTTP 200 也应检查 `status: partial`。详见[模型管理](model-management.md)。

## 生成

Chat 接受 `messages`，Completion 的 `prompt` 接受字符串或 token ID 列表，使用[贪心生成参数](generation.md)。响应沿用 OpenAI 风格的 `choices` 和 `usage`，函数调用通过 `tool_calls` 返回，由客户端执行。

设置 `stream=true` 后使用 SSE，以 `data: [DONE]` 结束；`stream_options.include_usage=true` 添加最终 usage chunk。客户端示例见[在线推理](serving.md)。

## 错误

| 状态码 | 含义 |
|---|---|
| 400 | 模型、生成取值或操作参数错误 |
| 422 | 请求 schema 错误 |
| 429 | 访问限流 |
| 503 | 没有 READY 副本 |
| 507 | 部署失败，且没有 READY 副本 |
| 500 | 服务端错误，查看服务日志 |

响应的 `detail` 字段提供错误信息。Runtime 接口见[日志与监控](observability.md)。
