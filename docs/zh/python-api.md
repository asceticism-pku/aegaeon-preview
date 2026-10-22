# Python API

`LLMService`、`NodeConfig` 和 `Request` 提供直接传入 token 的推理接口。先配置 `docs/zh/examples/simple.yaml` 中的模型路径和 profile，再按[部署说明](deployment.md)启动带 `node_0` 资源的 Ray。

## 示例

```python
import os
import time
import ray
from aegaeon import LLMService, NodeConfig, Request
from aegaeon.config import get_model_config
from aegaeon.models import set_model_registry
from aegaeon.model_registry import get_registry
from aegaeon.utils import get_tokenizer

config_path = os.path.abspath('docs/zh/examples/simple.yaml')
os.environ['AEGAEON_CONFIG_PATH'] = config_path
set_model_registry(config_path)
ray.init(address='127.0.0.1:6789')
spec = get_registry().get_by_name('Qwen/Qwen3-4B')
assert spec is not None
model_config = get_model_config(spec)
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
    eos_token_ids=model_config.eos_token_ids,
    max_model_len=model_config.max_model_len,
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

## 接口

- `NodeConfig` 配置节点的 Engine、缓存和模型，内存大小以字节计。
- `Request` 提供输入 token ID、输出长度、EOS 和上下文限制；Chat 消息先应用聊天模板。
- `LLMService.serve()` 返回各请求的 `StepOutput` 列表或异常对象，在同步代码中调用。
- `deploy_model_replicas()` 和 `undeploy_model_replicas()` 异步管理副本。

聊天格式、流式输出和响应 schema 可直接使用 [HTTP API](api.md)。
