# Request Schema Fields

Generation requests inherit `GreedyGenerationRequest`. Chat and Completion tables list added or overridden fields. Accepted values are also subject to [generation parameter validation](generation.md).

## FunctionDefinition

| Field | Type | Default / constraints |
|---|---|---|
| name | `str` | `Required`; 1–64 letters, digits, underscores, or hyphens |
| description | `str \| None` | `None` |
| parameters | `dict[str, Any]` | Empty object; JSON Schema for function arguments |
| strict | `bool \| None` | `None` |

## FunctionTool

| Field | Type | Default / constraints |
|---|---|---|
| type | `Literal['function']` | `'function'` |
| function | `FunctionDefinition` | `Required` |

## NamedToolChoice

| Field | Type | Default / constraints |
|---|---|---|
| type | `Literal['function']` | `'function'` |
| function | `NamedFunctionToolChoice` | `Required`; its name follows the function-name constraints |

## ToolCall

| Field | Type | Default / constraints |
|---|---|---|
| id | `str` | `Required` |
| type | `Literal['function']` | `'function'` |
| function | `FunctionCall` | `Required`; contains name and arguments in string or decoded JSON form |

## Message

| Field | Type | Default / constraints |
|---|---|---|
| role | `Literal['system', 'user', 'assistant', 'tool']` | `Required` |
| content | `str \| list[dict[str, Any]] \| None` | `None`; ordinary messages normally use strings; content is optional for assistant tool calls |
| name | `str \| None` | `None` |
| tool_calls | `list[ToolCall] \| None` | `None`; structured function calls issued by an assistant message |
| tool_call_id | `str \| None` | `None`; call ID referenced by a tool message |

## StreamOptions

| Field | Type | Default / constraints |
|---|---|---|
| include_usage | `bool` | `False` |

## DeployRequest

| Field | Type | Default / constraints |
|---|---|---|
| model | `str` | `Required` |
| model_path | `str \| None` | `None` |
| max_model_len | `int \| None` | `None` |
| node_ids | `list[str] \| None` | `None` |
| replica_count | `int` | `1`; minimum `1` |
| tool_parser | `str \| None` | `None`; vLLM tool-parser override for a model alias |

## GreedyGenerationRequest

| Field | Type | Default / constraints |
|---|---|---|
| model | `str` | `Required` |
| max_tokens | `int \| None` | `None` |
| stream | `bool` | `False` |
| stream_options | `StreamOptions \| None` | `None` |
| stop | `str \| list[str] \| None` | `None` |
| n | `int` | `1` |
| temperature | `float \| None` | `0.0` |
| top_p | `float \| None` | `1.0` |
| top_k | `int \| None` | `1` |
| min_p | `float \| None` | `0.0` |
| frequency_penalty | `float \| None` | `0.0` |
| presence_penalty | `float \| None` | `0.0` |
| repetition_penalty | `float \| None` | `1.0` |
| seed | `int \| None` | `None` |
| logit_bias | `dict[str, float] \| None` | `None` |
| use_beam_search | `bool` | `False` |
| length_penalty | `float` | `1.0` |
| stop_token_ids | `list[int] \| None` | `None` |
| include_stop_str_in_output | `bool` | `False` |
| ignore_eos | `bool` | `False` |
| min_tokens | `int` | `0` |
| skip_special_tokens | `bool` | `True` |
| spaces_between_special_tokens | `bool` | `True` |
| truncate_prompt_tokens | `int \| None` | `None` |
| truncation_side | `Literal['left', 'right'] \| None` | `None` |
| add_special_tokens | `bool` | `True` |
| return_token_ids | `bool` | `False` |
| priority | `int` | `0` |
| request_id | `str \| None` | `None` |
| user | `str \| None` | `None` |

## ChatCompletionRequest

| Field | Type | Default / constraints |
|---|---|---|
| messages | `list[Message]` | `Required` |
| max_completion_tokens | `int \| None` | `None` |
| add_generation_prompt | `bool` | `True` |
| continue_final_message | `bool` | `False` |
| add_special_tokens | `bool` | `False` |
| tools | `list[FunctionTool] \| None` | `None`; available function definitions |
| tool_choice | `'none' \| 'auto' \| 'required' \| NamedToolChoice \| None` | `None` |
| parallel_tool_calls | `bool \| None` | `None` |
| logprobs | `bool \| None` | `False` |
| top_logprobs | `int \| None` | `0` |
| mm_processor_kwargs | `dict[str, Any] \| None` | `None`; reserved field, multimodal input is unsupported |

## CompletionRequest

| Field | Type | Default / constraints |
|---|---|---|
| prompt | `str \| list[int]` | `Required` |
| echo | `bool` | `False` |
| logprobs | `int \| None` | `None` |

## TokenizeRequest

| Field | Type | Default / constraints |
|---|---|---|
| model | `str` | `Required` |
| prompt | `str` | `Required` |
| add_special_tokens | `bool` | `True` |

## DetokenizeRequest

| Field | Type | Default / constraints |
|---|---|---|
| model | `str` | `Required` |
| tokens | `list[int]` | `Required` |
| skip_special_tokens | `bool` | `True` |
