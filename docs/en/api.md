# HTTP API Reference

The CLI listens on 0.0.0.0:8000 by default; examples use `http://127.0.0.1:8000`. The service permits all CORS origins, methods, and headers and does not validate API keys. Public deployments should add authentication and access controls at the gateway.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | /health | Fixed health response; no deep actor checks |
| GET | /v1/models | Deployed model list |
| GET | /v1/models/{model_name:path} | Deployed model metadata; the path converter accepts names containing / |
| POST | /v1/models/deploy | Deploy CPU weight cache replicas |
| POST | /v1/models/undeploy | Unload selected or all replicas |
| POST | /v1/chat/completions | Chat, nonstreaming or SSE |
| POST | /v1/completions | Legacy Completion, nonstreaming or SSE |
| POST | /tokenize | Tokenize a plain prompt |
| POST | /detokenize | Decode token IDs |
| GET | /v1/aegaeon/runtime | Cluster runtime snapshot |
| GET | /v1/aegaeon/events | Runtime SSE |
| GET | /v1/aegaeon/metrics | 60-second access metrics and rate-limit settings |
| GET | /v1/aegaeon/gpu-power | GPU samples on the API host |

FastAPI also exposes `/docs`, `/redoc`, and `/openapi.json`. Those pages show schemas; this chapter defines generation values, lifecycle behavior, and partial failures.

## Model listing

```json
{"object":"list","data":[{"id":"Qwen/Qwen3-4B","object":"model","created":0,"owned_by":"aegaeon"}]}
```

The example uses `created=0` to show the structure; actual values use query time. Model listing shows service registration, while runtime snapshots carry READY and DRAINING node states.

## Deploy and undeploy

```bash
curl -sS http://127.0.0.1:8000/v1/models/deploy \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","model_path":"/srv/models/Qwen3-4B","replica_count":1,"node_ids":["node_0"]}'
curl -sS http://127.0.0.1:8000/v1/models/undeploy \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","node_ids":["node_0"]}'
```

Deploy fields are model (required), model_path, max_model_len, node_ids, replica_count (>=1, default 1), and tool_parser. Undeploy reuses this schema and reads model / node_ids. A model with existing replicas keeps its original model_path; changing max_model_len triggers the forced-deployment path, while tool_parser only updates parser metadata for the model alias.

Complete and partial success both return HTTP 200, with JSON `status` of `ok` or `partial`. Deploy responses include `ready_nodes`, `failed`, and operation results; undeploy responses include `remaining_nodes`, `failed`, and operation results. HTTP 200 alone does not mean every node succeeded. Complete deployment failure with no READY replica returns 507. See [model lifecycle](model-management.md).

## Tokenize and detokenize

```bash
curl -sS http://127.0.0.1:8000/tokenize \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","prompt":"hello","add_special_tokens":true}'
curl -sS http://127.0.0.1:8000/detokenize \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","tokens":[123],"skip_special_tokens":true}'
```

Tokenize returns count, tokens, and max_model_len; max_model_len is currently always null. It tokenizes plain strings without applying a chat template. Detokenize returns prompt, and token-ID meaning follows the selected tokenizer.

## Chat and Completion responses

Chat includes id, object='chat.completion', created, model, choices, and usage. Choices include index, message, logprobs=null, and finish_reason. Completion uses object='text_completion' and text inside each choice. Ordinary generation finishes with `stop` or `length`. When a function call is parsed successfully, Chat uses `finish_reason='tool_calls'` and returns `message.tool_calls`. The server does not execute functions; clients execute them and send the assistant call and tool result as subsequent messages.

With `return_token_ids=true`, nonstreaming Chat returns prompt_token_ids at the top level and token_ids inside choices. Nonstreaming Completion returns both inside choices. Streaming Chat returns prompt_token_ids in the first chunk and token_ids in the final choice. Streaming Completion returns both IDs in the final choice; with echo enabled, the first echo choice also contains prompt_token_ids.

`stream=true` uses `data: <JSON>` SSE and ends with `data: [DONE]`. `stream_options.include_usage=true` adds a usage chunk with `choices=[]` before the end. Chat chunks use `delta`; Completion chunks use `text`. Chat with an enabled tool parser waits until generation completes, then emits parsed `delta.content` and/or `delta.tool_calls`, so text is not guaranteed to arrive token by token. If no call is parsed successfully, the response can remain ordinary text. See [online inference](serving.md#common-integration-issues) for tool requirements.

## Runtime and metrics

`GET /v1/aegaeon/runtime` returns a snapshot including `timestamp`, `nodes`, `engines`, `model_placements`, `request_routing`, `model_placement_stats`, `decode_load`, `work_stealing`, and `events`. `GET /v1/aegaeon/events` is a separate SSE stream with `event: runtime` and JSON `data` containing `engines` and `events`; it can send comment heartbeats when idle. `GET /v1/aegaeon/metrics` returns 60-second-window `limits` and per-model `metrics`. `GET /v1/aegaeon/gpu-power` samples GPUs on the API host, not every cluster node.

## Errors

| HTTP status | Common causes |
|---|---|
| 400 | Undeployed model, invalid greedy parameters, template failures, invalid nodes/model paths |
| 422 | Schema types, unknown generation or tokenizer fields, or Field constraints |
| 429 | Access-window rate limits |
| 503 | Registered model has no routable READY replica |
| 507 | All deployment targets fail with no existing READY replica |
| 500 and others | Internal, download, or Ray exceptions without uniform mapping |

Explicit `_error()` responses use FastAPI detail rather than a uniform OpenAI top-level error:

```json
{"detail":{"message":"...","type":"invalid_request_error","param":"temperature","code":null}}
```

FastAPI schema errors use a list in `detail`, explicit `_error()` responses use an object, and some HTTPException branches use a string. Preserve HTTP status and the original body.

Generation and tokenizer requests use `extra="forbid"`, so undeclared fields such as `response_format` and `chat_template_kwargs` return 422. Deploy and undeploy use a different request model and ignore unknown fields under Pydantic's default behavior. Nongreedy values and invalid tool settings usually return 400. See the [request schema](request-schema.md) for fields and defaults and [generation parameters](generation.md) for accepted values.

## Built-in pages and external proxies

`/aegaeon_console.html`, `/aegaeon_chat.html`, and `/aegaeon_docs.html` are built-in static pages. The main entry and `/demo` redirect to pages. Documentation is maintained in `docs/zh` and `docs/en`; regenerate the built-in page with `docs/build_portable.py` after edits. See [documentation maintenance](development.md#documentation-maintenance).

The demo proxies `/v1/serverlessllm/models`, `/v1/serverlessllm/gpu-power`, and `/v1/serverlessllm/chat/completions` target an external service configured by SERVERLESSLLM_URL (default 127.0.0.1:8343). They are not native Aegaeon inference features or benchmark results.
