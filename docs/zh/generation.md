# 生成参数

## 贪心解码

| 参数 | 可用取值 |
|---|---|
| `n` | `1` |
| `temperature` | `0` 或 `null` |
| `top_p`、`top_k` | `1` 或 `null` |
| `min_p` | `0` 或 `null` |
| `frequency_penalty`、`presence_penalty` | `0` 或 `null` |
| `repetition_penalty` | `1` 或 `null` |
| `logit_bias` | 空字典或 `null` |
| `use_beam_search` | `false` |
| `length_penalty` | `1` |
| `priority` | `0` |
| Chat `logprobs` / `top_logprobs` | `false` 或 `null` / `0` 或 `null` |
| Completion `logprobs` | `null` 或省略 |

## 长度与停止

Chat 默认 `max_tokens=128`，Completion 默认 `max_tokens=16`。Chat 也接受 `max_completion_tokens`，同时填写时两者一致。输入与输出的总长度受 `max_model_len` 限制。

| 参数 | 含义 |
|---|---|
| `stop` | 在可见文本中匹配的停止字符串 |
| `stop_token_ids` | 额外的停止 token ID |
| `ignore_eos` | 忽略 EOS，仍保留长度上限 |
| `min_tokens` | 接受停止条件前的最少输出长度 |
| `include_stop_str_in_output` | 保留匹配的停止字符串 |
| `skip_special_tokens` | 解码时省略特殊 token |
| `return_token_ids` | 在响应中返回 token ID |

`finish_reason` 为 `stop`、`length`，成功解析函数调用时为 `tool_calls`。

## Prompt 格式

`truncate_prompt_tokens` 限制输入 token 数；`truncation_side='left'` 保留末尾，`'right'` 保留开头。`-1` 保持纯文本输入不变。

Chat 默认 `add_special_tokens=false`、`add_generation_prompt=true`。使用 `continue_final_message=true` 时将 `add_generation_prompt` 设为 `false`。

完整请求类型和默认值可在运行中服务的 `/docs` 和 `/openapi.json` 查看。
