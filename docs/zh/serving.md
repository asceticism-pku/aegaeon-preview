# 在线推理

## OpenAI 风格客户端

将 OpenAI 客户端指向服务的 `/v1` 地址：

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

这里的 key 用于满足客户端构造要求；鉴权由部署侧网关配置。

## 流式输出

设置 `stream=True`，读取 `chunk.choices[0].delta.content`。usage chunk 的 `choices` 为空：

```python
stream = client.chat.completions.create(
    model='Qwen/Qwen3-4B', messages=[{'role': 'user', 'content': '你好'}],
    temperature=0, stream=True, stream_options={'include_usage': True},
)
for chunk in stream:
    if chunk.choices:
        print(chunk.choices[0].delta.content or '', end='', flush=True)
```

原始 SSE 响应以 `data: [DONE]` 结束。Completion 使用 `/v1/completions`，`prompt` 可为字符串或 token ID 列表。

## 函数工具

按 OpenAI 消息格式传入 `tools` 和 `tool_choice`。模型需有对应的 vLLM parser，可在 YAML 模型项或部署请求中通过 `tool_parser` 指定。

客户端读取 `tool_calls`、执行函数，再将结果作为 tool 消息发回。工具解析会缓冲输出，完成后统一返回。

可用取值见[生成参数](generation.md)，端点与 schema 见 [HTTP API](api.md)。
