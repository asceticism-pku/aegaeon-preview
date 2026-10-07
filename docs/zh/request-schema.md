# 请求 Schema 字段

Chat 和 Completion 请求都继承 `GreedyGenerationRequest`。各自的表格只列新增或覆盖的字段；字段的实际允许值还受生成参数校验限制，见[生成参数](generation.md)。

## FunctionDefinition

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| name | `str` | `必填`；1–64 个字母、数字、下划线或连字符 |
| description | `str \| None` | `None` |
| parameters | `dict[str, Any]` | 空对象；函数参数的 JSON Schema |
| strict | `bool \| None` | `None` |

## FunctionTool

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| type | `Literal['function']` | `'function'` |
| function | `FunctionDefinition` | `必填` |

## NamedToolChoice

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| type | `Literal['function']` | `'function'` |
| function | `NamedFunctionToolChoice` | `必填`；其中 name 使用与函数名相同的约束 |

## ToolCall

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| id | `str` | `必填` |
| type | `Literal['function']` | `'function'` |
| function | `FunctionCall` | `必填`；包含 name 和字符串或 JSON 形式的 arguments |

## Message

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| role | `Literal['system', 'user', 'assistant', 'tool']` | `必填` |
| content | `str \| list[dict[str, Any]] \| None` | `None`；普通消息通常使用字符串，assistant tool call 可省略 |
| name | `str \| None` | `None` |
| tool_calls | `list[ToolCall] \| None` | `None`；assistant 发起的结构化函数调用 |
| tool_call_id | `str \| None` | `None`；tool 消息对应的调用 ID |

## StreamOptions

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| include_usage | `bool` | `False` |

## DeployRequest

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| model | `str` | `必填` |
| model_path | `str \| None` | `None` |
| max_model_len | `int \| None` | `None` |
| node_ids | `list[str] \| None` | `None` |
| replica_count | `int` | `1`，至少为 `1` |
| tool_parser | `str \| None` | `None`；为模型别名覆盖 vLLM tool parser |

## GreedyGenerationRequest

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| model | `str` | `必填` |
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

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| messages | `list[Message]` | `必填` |
| max_completion_tokens | `int \| None` | `None` |
| add_generation_prompt | `bool` | `True` |
| continue_final_message | `bool` | `False` |
| add_special_tokens | `bool` | `False` |
| tools | `list[FunctionTool] \| None` | `None`；可调用的函数定义 |
| tool_choice | `'none' \| 'auto' \| 'required' \| NamedToolChoice \| None` | `None` |
| parallel_tool_calls | `bool \| None` | `None` |
| logprobs | `bool \| None` | `False` |
| top_logprobs | `int \| None` | `0` |
| mm_processor_kwargs | `dict[str, Any] \| None` | `None`；内容块输入时传给 vLLM renderer/processor；公开验收范围仅包括文本 |

## CompletionRequest

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| prompt | `str \| list[int]` | `必填` |
| echo | `bool` | `False` |
| logprobs | `int \| None` | `None` |

## TokenizeRequest

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| model | `str` | `必填` |
| prompt | `str` | `必填` |
| add_special_tokens | `bool` | `True` |

## DetokenizeRequest

| 字段 | 类型 | 默认值与约束 |
|---|---|---|
| model | `str` | `必填` |
| tokens | `list[int]` | `必填` |
| skip_special_tokens | `bool` | `True` |
