# Online Inference and Client Integration

## OpenAI-style clients

Aegaeon implements parts of `/v1/chat/completions` and `/v1/completions`. Use greedy settings for requests: `temperature=0`, `top_p=1`, `top_k=1`, and `n=1`.

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

This example uses the `openai` package as a client dependency; install and pin it in the client environment. The `api_key` value only satisfies the client constructor. Aegaeon does not authenticate API keys.

## Streaming clients

```python
stream = client.chat.completions.create(
    model='Qwen/Qwen3-4B',
    messages=[{'role': 'user', 'content': 'Explain CPU and GPU KV Cache.'}],
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

Usage chunks have empty `choices`, so clients should handle them. SSE sends `data: <JSON>` and ends with `data: [DONE]`. Chat updates appear under `delta`; Completion updates appear under `text`. Treat extension fields as optional.

## Legacy Completion

```bash
curl -sS http://127.0.0.1:8000/v1/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","prompt":"KV Cache is","temperature":0,"max_tokens":32}'
```

`prompt` accepts a nonempty string or a single list of token IDs. The Completion interface accepts one string or one token-ID list per request; batched strings and nested token-ID lists are rejected. It does not apply a chat template automatically. Use Chat for instruction models.

## Input types

The validated public scope is text input and text generation, so use strings for Chat message `content`. The request schema also accepts content-part lists and `mm_processor_kwargs`. Lists pass through the vLLM renderer/processor before the result reaches the backend. This path is outside the supported public scope: accepting the schema fields does not establish support for a particular image, video, or audio model.

## Request lifecycle

The service selects a READY node for each request and records a routing reservation. It releases the reservation after backend execution finishes. If a streaming client disconnects, background cleanup still waits for the backend. The disconnect does not immediately stop GPU execution.

A matched `stop` string truncates visible output; the backend continues until it reaches its own stopping condition. Evaluation and admission control should use request completion state.

## Common integration issues

Unknown fields in generation and tokenizer requests produce Pydantic validation errors, usually 422; unknown deployment fields are ignored under Pydantic's default behavior. Nongreedy parameter values return 400.

Chat requests can use `tools`, `tool_choice`, and `parallel_tool_calls` when the model name resolves to a vLLM tool parser or model configuration/deployment supplies `tool_parser`. `tool_choice='none'` disables tool parsing. `tool_choice='required'` or a named function is passed to the template and checked during parsing, but unconstrained decoding does not guarantee a call. `response_format` and `chat_template_kwargs` remain outside this interface.

The service does not retry requests automatically or provide persistent idempotency. Check the original call state before retrying; `request_id` only affects the response ID.
