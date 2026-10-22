# Decode Work Stealing

Work Stealing balances Decode batches within one Controller. It is disabled by default and requires at least two Decode Engines.

## Enable

```yaml
server:
  num_engines: 0
  num_prefill_engines: 1
  num_decode_engines: 2
  decode_work_stealing: true
  decode_work_stealing_min_batch_gap: 2
```

## Transfer rules

After a scheduling round, a donor can transfer one complete waiting batch to an engine with an empty queue. The batch has no iteration in flight and its KV state is on CPU. The batch-count gap is at least 2; a receiver already holding the same model is preferred.

Transfers stay within a physical node and preserve the existing Decode quota policy.

## Monitoring

`GET /v1/aegaeon/runtime` exposes `work_stealing` transfer counters and `decode_load` per-engine work and queue metrics. See [Monitoring](observability.md) for runtime access.
