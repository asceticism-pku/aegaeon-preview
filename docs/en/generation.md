# Generation Parameters

## Greedy decoding

| Parameter | Accepted values |
|---|---|
| `n` | `1` |
| `temperature` | `0` or `null` |
| `top_p`, `top_k` | `1` or `null` |
| `min_p` | `0` or `null` |
| `frequency_penalty`, `presence_penalty` | `0` or `null` |
| `repetition_penalty` | `1` or `null` |
| `logit_bias` | Empty dictionary or `null` |
| `use_beam_search` | `false` |
| `length_penalty` | `1` |
| `priority` | `0` |
| Chat `logprobs` / `top_logprobs` | `false` or `null` / `0` or `null` |
| Completion `logprobs` | `null` or omitted |

## Length and stopping

Chat defaults to `max_tokens=128`; Completion defaults to `max_tokens=16`. Chat also accepts `max_completion_tokens`; when both are supplied they must match. Prompt and output together fit within `max_model_len`.

| Parameter | Meaning |
|---|---|
| `stop` | Stop string or strings applied to visible text |
| `stop_token_ids` | Additional token IDs that end generation |
| `ignore_eos` | Ignore EOS while retaining the length limit |
| `min_tokens` | Minimum output before accepting stop conditions |
| `include_stop_str_in_output` | Retain the matched stop string |
| `skip_special_tokens` | Omit special tokens when decoding |
| `return_token_ids` | Include token IDs in the response |

`finish_reason` is `stop`, `length`, or `tool_calls` for a parsed function call.

## Prompt formatting

`truncate_prompt_tokens` limits input tokens; `truncation_side='left'` keeps the end and `'right'` keeps the beginning. `-1` leaves plain-text input unchanged.

Chat defaults to `add_special_tokens=false` and `add_generation_prompt=true`. Set `add_generation_prompt=false` when using `continue_final_message=true`.

Complete request types and defaults are available at the running service's `/docs` and `/openapi.json`.
