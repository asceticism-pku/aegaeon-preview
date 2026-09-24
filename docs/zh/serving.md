# 在线推理与客户端接入

## OpenAI 风格客户端

Aegaeon 提供 `/v1/chat/completions` 和 `/v1/completions` 的部分兼容实现。请求使用贪心解码参数：`temperature=0`、`top_p=1`、`top_k=1` 和 `n=1`。

```python
from openai import OpenAI

client = OpenAI(base_url='http://127.0.0.1:8000/v1', api_key='local-placeholder')
response = client.chat.completions.create(
    model='Qwen/Qwen3-4B',
    messages=[{'role': 'user', 'content': '解释模型切换。'}],
    temperature=0,
    max_tokens=128,
)
print(response.choices[0].message.content)
```

`openai` 是示例客户端依赖，请在客户端环境单独安装并固定版本。示例中的 `api_key` 用于满足客户端构造要求；生产鉴权由部署侧网关负责。

## 流式客户端

```python
stream = client.chat.completions.create(
    model='Qwen/Qwen3-4B',
    messages=[{'role': 'user', 'content': '解释 CPU 与 GPU KV Cache。'}],
    temperature=0,
    max_tokens=128,
    stream=True,
    stream_options={'include_usage': True},
)
for chunk in stream:
    if chunk.choices:
        print(chunk.choices[0].delta.content or '', end='', flush=True)
    if chunk.usage is not None:
        print('\nusage:', chunk.usage)
```

必须允许 usage chunk 的 choices 为空。SSE 使用 `data: <JSON>` 与最终 `data: [DONE]`。Chat 增量在 delta 中，Completion 增量在 text 中；扩展字段应按可选字段处理。

## Legacy Completion

```bash
curl -sS http://127.0.0.1:8000/v1/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","prompt":"KV Cache is","temperature":0,"max_tokens":32}'
```

`prompt` 支持非空字符串或一组 token ID。批量 prompt、批量 token 列表和 Chat 模板由调用侧处理；instruction 模型请使用 Chat 接口。

## 输入类型

已验证的公开接入范围是纯文本输入和文本生成，Chat 消息的 `content` 应使用字符串。请求 schema 也接受 content parts 列表和 `mm_processor_kwargs`；列表会进入 vLLM renderer/processor，再把处理结果传给后端。这条路径尚未纳入公开支持范围；图像、视频或音频模型需要单独完成端到端验收。

## 请求生命周期

请求先选择 READY 节点并建立 outstanding reservation，节点执行结束后释放。流客户端断开后，后台清理任务会等待后端到达停止条件，再释放 reservation；GPU 执行与客户端连接采用独立生命周期。

字符串 `stop` 命中后会停止发送可见文本，底层请求继续执行到后端停止条件。评估和限流应以请求完成状态为准。

## 常见接入问题

生成与 tokenizer 请求中的未知字段触发 Pydantic 校验错误（通常为 422）；部署请求的未知字段按 Pydantic 默认行为被忽略。非贪心参数值返回 400。Chat 请求可使用 `tools`、`tool_choice` 和 `parallel_tool_calls`，前提是模型名可自动解析出 vLLM tool parser，或模型配置/部署请求显式设置 `tool_parser`。`tool_choice='none'` 不启用工具解析；`required` 或指定函数会传入模板并用于解析校验，但没有约束解码保证一定生成调用。`response_format` 和 `chat_template_kwargs` 仍属于接口范围外字段。

自动重试和持久化幂等由调用侧实现。重试前请根据调用记录确认原请求状态；`request_id` 只影响响应 ID。
