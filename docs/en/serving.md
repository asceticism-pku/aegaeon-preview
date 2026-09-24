# Online Inference and Client Integration

## OpenAI-style clients

Aegaeon partially implements `/v1/chat/completions` and `/v1/completions`. Requests use greedy settings: `temperature=0`, `top_p=1`, `top_k=1`, and `n=1`.

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

The `openai` package is an example client dependency. Install and pin it in the client environment. The example `api_key` only satisfies the client constructor; Aegaeon does not perform API key authentication.

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

Allow empty choices in usage chunks. SSE uses `data: <JSON>` followed by `data: [DONE]`. Chat deltas appear in delta; Completion deltas appear in text. Treat extension fields as optional.

## Legacy Completion

```bash
curl -sS http://127.0.0.1:8000/v1/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","prompt":"KV Cache is","temperature":0,"max_tokens":32}'
```

`prompt` accepts a nonempty string or one list of token IDs. Batched prompts, batched token lists, and automatic chat templates are outside the Completion interface. Use Chat for instruction models.

## Input types

The validated public scope is text input and text generation; use strings for Chat message `content`. The request schema also accepts content-part lists and `mm_processor_kwargs`. Lists enter the vLLM renderer/processor, and its result is passed to the backend. This path is outside the supported public scope; accepting the schema fields does not establish that a particular image, video, or audio model works.

## Request lifecycle

A request selects a READY node and creates an outstanding reservation. After a streaming client disconnects, background cleanup waits for backend completion and then releases the reservation. Disconnecting the client does not immediately stop GPU execution.

A matched string `stop` ends visible output while the backend continues to its stopping condition. Evaluation and admission control should use request completion state.

## Common integration issues

Unknown fields in generation and tokenizer requests produce Pydantic validation errors, usually 422; unknown deployment fields are ignored under Pydantic's default behavior. Nongreedy parameter values return 400. Chat requests can use `tools`, `tool_choice`, and `parallel_tool_calls` when the model name resolves to a vLLM tool parser or model configuration/deployment supplies `tool_parser`. `tool_choice='none'` disables tool parsing. `required` or a named function is passed to the template and checked during parsing, but unconstrained decoding does not guarantee a call. `response_format` and `chat_template_kwargs` remain outside this interface.

The service provides neither automatic retries nor persistent idempotency. Confirm the original call state before retrying; `request_id` only affects the response ID.
