# 在线推理与客户端接入

## OpenAI 风格客户端

Aegaeon 的 `/v1/chat/completions` 和 `/v1/completions` 部分兼容 OpenAI 接口。发送请求时使用贪心解码参数：`temperature=0`、`top_p=1`、`top_k=1`、`n=1`。

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

示例依赖 `openai` 包，需在客户端环境安装并固定版本。`api_key` 在这里仅用于满足客户端构造要求；生产环境的鉴权由部署侧网关负责。

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

usage chunk 的 `choices` 为空，所以上述代码先检查 `chunk.choices`。SSE 以 `data: <JSON>` 发送消息，最后发送 `data: [DONE]`。Chat 增量位于 `delta`，Completion 增量位于 `text`；扩展字段按可选字段处理。

## Legacy Completion

```bash
curl -sS http://127.0.0.1:8000/v1/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","prompt":"KV Cache is","temperature":0,"max_tokens":32}'
```

`prompt` 可传非空字符串或一组 token ID。Completion 一次只处理一个 prompt，不提供批量处理，也不会自动应用 Chat 模板；instruction 模型应使用 Chat 接口。

## 输入类型

目前经过验证的公开接入范围是纯文本输入和文本生成。Chat 消息的 `content` 应使用字符串。请求 schema 也接受内容块列表和 `mm_processor_kwargs`：列表进入 vLLM renderer/processor，处理结果再传给后端。这条路径尚未纳入公开支持范围；图像、视频或音频模型需要单独完成端到端验收。

## 请求生命周期

服务端为请求选择 READY 节点并登记路由占用，节点执行结束后再解除占用。流式客户端断开后，后台清理任务仍会等待后端执行结束。客户端断开不会立刻停止 GPU 执行。

字符串 `stop` 命中后，接口停止发送可见文本，底层请求继续运行，直到满足后端停止条件。评估和限流应以请求完成状态为准。

## 常见接入问题

生成与 tokenizer 请求传入未知字段时，Pydantic 通常返回 422；部署请求中的未知字段按 Pydantic 默认行为被忽略。非贪心参数值返回 400。`response_format` 和 `chat_template_kwargs` 不在当前接口的字段范围内。

Chat 支持 `tools`、`tool_choice` 和 `parallel_tool_calls`，条件是模型名能自动匹配 vLLM tool parser，或模型配置、部署请求显式指定了 `tool_parser`。`tool_choice='none'` 不启用工具解析；`required` 或指定函数会传入模板并用于解析校验，但当前没有约束解码保证一定生成调用。

服务端不提供自动重试或持久化幂等。调用方重试前应确认原请求状态；`request_id` 只影响响应 ID。
