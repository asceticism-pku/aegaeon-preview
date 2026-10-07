# 生成参数与停止语义

## 贪心解码

Worker 计算 logits 后直接取 argmax，所以采样字段只接受与贪心解码一致的值。`seed` 可以通过请求校验，但解码仍采用确定性的 argmax。

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

Chat 的 `max_tokens` 默认为 128，Completion 默认为 16。Chat 也接受 `max_completion_tokens`。若同时设置两个字段，取值必须相等；选定的输出上限至少为 1。

上下文长度包括 prompt 和生成的 token，`max_tokens` 是生成上限。命中 EOS、stop token、字符串 stop 或上下文边界时，生成会提前结束。

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

命中的 EOS 或 stop token 会从可见文本中去除。`usage.completion_tokens` 按处理后的 token IDs 计算。字符串截断时，文本会重新编码，因此这个计数与 Worker 的原始执行步数口径不同。

普通文本生成的 `finish_reason` 为 `stop` 或 `length`：前者表示命中 EOS、stop token 或 stop 字符串，后者表示后端用完本次长度预算。该预算已受上下文剩余空间限制。Chat 成功解析出函数调用时，`finish_reason` 改为 `tool_calls`。

## Prompt 截断

`truncate_prompt_tokens` 可设为正整数、-1 或 null。在纯文本 `_truncate_prompt` 路径中，-1 表示不截断；使用正整数时默认保留末尾，设置 `truncation_side='right'` 则保留开头。当前公开接口只支持这条纯文本路径。

`add_special_tokens` 在 Chat 中默认为 false，在 Completion 中默认为 true。Chat 默认 `add_generation_prompt=true`；设置 `continue_final_message=true` 时，还须设置 `add_generation_prompt=false`。

## ID 与用户标识

请求 schema 接受 `request_id` 和 `user`，后端仍使用自己生成的内部 request ID。用户级鉴权由网关负责，客户端重试与跨进程幂等由调用方实现；调度优先级沿用服务现有策略。

完整字段及默认值见[请求 Schema](request-schema.md)。
