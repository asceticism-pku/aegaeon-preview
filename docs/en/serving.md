# Online Inference

## OpenAI-style client

Point an OpenAI client at the service's `/v1` URL:

```python
from openai import OpenAI

client = OpenAI(base_url='http://127.0.0.1:8000/v1', api_key='local-placeholder')
response = client.chat.completions.create(
    model='Qwen/Qwen3-4B',
    messages=[{'role': 'user', 'content': 'Explain model switching.'}],
    temperature=0,
    max_tokens=128,
)
print(response.choices[0].message.content)
```

The key satisfies the client constructor; configure authentication at your gateway.

## Streaming

Set `stream=True` and consume `chunk.choices[0].delta.content`. Usage chunks have empty `choices`:

```python
stream = client.chat.completions.create(
    model='Qwen/Qwen3-4B', messages=[{'role': 'user', 'content': 'Hello'}],
    temperature=0, stream=True, stream_options={'include_usage': True},
)
for chunk in stream:
    if chunk.choices:
        print(chunk.choices[0].delta.content or '', end='', flush=True)
```

Raw SSE responses end with `data: [DONE]`. Completion requests use `/v1/completions` with a string or token-ID list as `prompt`.

## Function tools

Pass `tools` and `tool_choice` using OpenAI-style messages. The model must have a matching vLLM parser; set `tool_parser` in its YAML entry or deployment request to override selection.

Read `tool_calls`, execute the selected function in the client, and return its result as a tool message. Tool parsing buffers output until completion, so tool responses arrive together.

See [Generation Parameters](generation.md) for accepted values and [HTTP API](api.md) for endpoints and schemas.
