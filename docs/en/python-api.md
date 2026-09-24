# Python API

## Public exports

The package lazily exports `LLMService`, `NodeConfig`, and `Request`. The Python interface targets the Aegaeon control plane and accepts tokenized `Request` objects through `LLMService.serve()`; it does not implement `vllm.LLM.generate(prompt, SamplingParams)`.

## Synchronous inference example

Run from the repository root with Linux/CUDA, usable profiles, and sufficient Ray resources:

```python
import os
import time
import ray
from aegaeon import LLMService, NodeConfig, Request
from aegaeon.models import set_model_registry
from aegaeon.model_registry import get_registry
from aegaeon.utils import get_tokenizer

config_path = os.path.abspath('docs/en/examples/simple.yaml')
os.environ['AEGAEON_CONFIG_PATH'] = config_path
set_model_registry(config_path)
ray.init(address='127.0.0.1:6789')
spec = get_registry().get_by_name('Qwen/Qwen3-4B')
assert spec is not None
spec.path()
tokenizer = get_tokenizer(spec.path())

service = LLMService([
    NodeConfig(
        node_id='node_0',
        num_prefill_engines=0,
        num_decode_engines=0,
        num_engines=1,
        tensor_parallel_size=1,
        cpu_num_slabs=8,
        cpu_slab_size_bytes=1024**3,
        model_cache_size=20 * 1024**3,
        cached_models=[spec],
        config_path=config_path,
    )
])
request = Request(
    model=spec,
    arrival_time=time.time(),
    request_id=0,
    prompt_token_ids=tokenizer.encode('KV Cache is'),
    decode_tokens=16,
)
try:
    outputs = service.serve([request])
    if isinstance(outputs[0], BaseException):
        raise outputs[0]
    token_ids = [step.new_token_id for step in outputs[0]]
    print(tokenizer.decode(token_ids, skip_special_tokens=True))
finally:
    ray.shutdown()
```

This uses Completion-style token input. Chat callers apply the appropriate chat template first. Direct Python callers also implement the string-stop and response-wrapping behavior provided by the HTTP layer.

## LLMService

| Entry point | Purpose / behavior |
|---|---|
| LLMService(cluster_config, model_placement_policy=None, request_routing_policy=None) | Initializes Controllers and event loops, waiting synchronously for initialization |
| serve(requests) | Reserves/releases nodes and submits concurrently with asyncio.run; returns StepOutput lists or exception objects per request; cannot be called directly inside a running event loop |
| replay(out_path, ...) | Arrival-time replay with project statistics; inspect implementation timing and qos definitions |
| deploy_model_replicas(...) | Async replica deployment |
| undeploy_model_replicas(...) | Async replica unloading |
| reserve_node / release_node_reservation | HTTP lifecycle routing accounting; direct callers must pair releases |
| request_routing_snapshot / model_placement_stats_snapshot | Current and cumulative statistics |
| reset() | Calls each node's reset, resets BlockManager, Prefill/Decode Dispatchers and engines, and clears request outputs/model mappings; does not stop Ray |

NodeConfig defines each Controller's resources and policies. node_id, num_prefill_engines, and num_decode_engines are required. model_cache_size is in bytes, unlike the server YAML's GiB setting.

`Request` contains model, arrival_time, request_id, prompt_token_ids, decode_tokens, and optional EOS, stop, min, and context parameters. Its constructor retains a `multimodal_input` field, but the supported scope requires `None`. The HTTP API supplies EOS sets and max_model_len; direct Python callers should provide those values explicitly.

## Stability

The Python interface evolves with the control plane. The public entry points documented here are `LLMService`, `NodeConfig`, and `Request`. Applications should integrate through these names; new capabilities belong in tested package-level wrappers.
