# Generation Parameters and Stopping

## Greedy decoding

The Worker takes argmax directly from logits, so accepted sampling values must be equivalent to greedy decoding. Supplying `seed` does not enable random sampling.

| Parameter | Accepted values |
|---|---|
| n | 1 |
| temperature | 0 or null |
| top_p | 1 or null |
| top_k | 1 or null |
| min_p | 0 or null |
| frequency_penalty / presence_penalty | 0 or null |
| repetition_penalty | 1 or null |
| logit_bias | Empty dictionary or null |
| use_beam_search | false |
| length_penalty | 1 |
| priority | 0 |
| Chat logprobs | false / null |
| Chat top_logprobs | 0 / null |
| Completion logprobs | null or omitted |

## Output length

Chat defaults to `max_tokens=128`; Completion defaults to `max_tokens=16`. Chat also accepts `max_completion_tokens`. If both limits are supplied, they must match. The selected limit must be at least 1.

Prompt and generated tokens both count toward context length. `max_tokens` caps output; EOS, stop tokens, string stops, or the context boundary can end it earlier.

## Stopping and decoding

| Parameter | Behavior |
|---|---|
| stop | A string or list of strings; truncates visible output on a match |
| stop_token_ids | Nonnegative token IDs added to the stopping set |
| ignore_eos | Defaults to false; true ignores EOS but retains the output limit |
| min_tokens | From 0 to max_tokens; controls when stop tokens / stop are accepted |
| include_stop_str_in_output | Defaults to false; retains a matched stop string when enabled |
| skip_special_tokens | Defaults to true |
| spaces_between_special_tokens | Defaults to true; forwarded to tokenizer.decode |
| return_token_ids | Defaults to false; adds token ID fields to responses |

Matched EOS and stop tokens are removed from visible text. `usage.completion_tokens` counts processed token IDs. When a stop string truncates the decoded text, the API re-encodes that text for the count instead of counting raw Worker steps.

`finish_reason` is `stop` or `length` for ordinary text generation. `stop` means EOS, a stop token, or a stop string matched. `length` means the backend completed the request's length budget, which already accounts for remaining context space. Chat uses `tool_calls` when a function call is parsed successfully.

## Prompt truncation

`truncate_prompt_tokens` accepts a positive integer, -1, or null. For plain text, `_truncate_prompt` leaves the input unchanged when the value is -1. A positive value keeps the end by default; `truncation_side='right'` keeps the beginning. The public interface supports this text path only.

`add_special_tokens` defaults to false in Chat and true in Completion. Chat defaults to add_generation_prompt=true. Setting continue_final_message=true requires add_generation_prompt=false.

## Request and user identifiers

The request schema accepts `request_id` and `user`. Neither field provides user authentication, cross-process idempotency, or priority scheduling. The backend generates its own internal request ID, so these fields do not make repeated execution safe.

See [request fields](request-schema.md) for the complete schema and defaults.
