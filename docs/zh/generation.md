# 生成参数与停止语义

## 贪心解码

Worker 计算 logits 后直接执行 argmax，因此采样字段只接受与贪心行为一致的取值。seed 会通过请求校验，解码结果仍采用确定性的 argmax。

| 参数 | 接受的值 |
|---|---|
| n | 1 |
| temperature | 0 或 null |
| top_p | 1 或 null |
| top_k | 1 或 null |
| min_p | 0 或 null |
| frequency_penalty / presence_penalty | 0 或 null |
| repetition_penalty | 1 或 null |
| logit_bias | 空字典或 null |
| use_beam_search | false |
| length_penalty | 1 |
| priority | 0 |
| Chat logprobs | false / null |
| Chat top_logprobs | 0 / null |
| Completion logprobs | 只能 null / 省略 |

## 输出长度

Chat 默认 max_tokens=128，Completion 默认 16。Chat 可用 max_completion_tokens；同时设置两者时必须相等。长度至少 1。

上下文长度是 prompt 与生成 token 的总和；`max_tokens` 表示生成上限。命中 EOS、stop token、字符串 stop 或上下文边界时，生成会提前结束。

## 停止与解码

| 参数 | 行为 |
|---|---|
| stop | 一个字符串或字符串列表，匹配后截断可见输出 |
| stop_token_ids | 非负 token ID 列表，加入停止 token 集合 |
| ignore_eos | 默认 false；true 跳过 EOS 停止条件，输出上限仍然生效 |
| min_tokens | 0 到 max_tokens；控制何时接受停止 token / stop |
| include_stop_str_in_output | 默认 false；是否保留匹配到的 stop 字符串 |
| skip_special_tokens | 默认 true |
| spaces_between_special_tokens | 默认 true，传给 tokenizer.decode |
| return_token_ids | 默认 false；返回额外 token ID 字段 |

EOS/stop token 命中后会从可见文本中去除。`usage.completion_tokens` 按处理后的 token IDs 计算；字符串截断会重新编码文本，因此该值与 Worker 原始执行步数采用不同口径。

普通文本生成的 `finish_reason` 为 `stop` 或 `length`。`stop` 表示命中 EOS、stop token 或 stop 字符串；`length` 表示后端执行完本次长度预算，其中预算已经受上下文剩余空间限制。Chat 成功解析出函数调用时会改为 `tool_calls`。

## Prompt 截断

`truncate_prompt_tokens` 可为正整数、-1 或 null。纯文本 `_truncate_prompt` 对 -1 不执行截断；正整数默认保留末尾，`truncation_side='right'` 时保留开头。当前公开接口只支持这条纯文本路径。

`add_special_tokens` 在 Chat 默认为 false，在 Completion 默认为 true。Chat 默认 add_generation_prompt=true；continue_final_message=true 时必须同时设 add_generation_prompt=false。

## ID 与用户标识

`request_id` 和 `user` 可出现在请求 schema 中，后端仍使用自身生成的内部 request ID。用户级鉴权由网关负责；客户端重试与跨进程幂等需要在调用侧实现，调度优先级沿用服务的现有策略。

完整字段及默认值见 [自动提取的请求 Schema](request-schema.md)。
