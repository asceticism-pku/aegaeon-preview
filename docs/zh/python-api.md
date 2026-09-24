# Python API

## 对外导出

包通过惰性导出提供 `LLMService`、`NodeConfig` 和 `Request`。Python 接口面向 Aegaeon 控制面，入口是 token 化后的 `Request` 和 `LLMService.serve()`；它采用自身的请求模型与服务生命周期。

## 同步推理示例

在真实Linux/CUDA环境、可用profile与Ray资源具备时，从仓库根目录运行：

```python
import os
import time
import ray
from aegaeon import LLMService, NodeConfig, Request
from aegaeon.models import set_model_registry
from aegaeon.model_registry import get_registry
from aegaeon.utils import get_tokenizer

config_path = os.path.abspath('docs/zh/examples/simple.yaml')
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

这是 Completion 风格的 token 输入。Chat 调用方需要先应用正确的聊天模板。HTTP 层负责的字符串 stop 和响应包装也需要由直接 Python 调用方自行实现。

## LLMService

| 入口 | 用途 / 说明 |
|---|---|
| LLMService(cluster_config, model_placement_policy=None, request_routing_policy=None) | 初始化Controller和event loops，会同步等待初始化 |
| serve(requests) | 为各请求 reserve 节点并最终 release；用 asyncio.run 并发提交，返回每请求 StepOutput 列表或异常对象；适用于同步入口，异步应用需放在线程或独立进程中调用 |
| replay(out_path, ...) | 按到达时间重放并写项目统计；阅读源码了解计时和qos口径 |
| deploy_model_replicas(...) | async副本部署入口 |
| undeploy_model_replicas(...) | async副本卸载入口 |
| reserve_node / release_node_reservation | HTTP生命周期的路由记账；直接使用时负责成对释放 |
| request_routing_snapshot / model_placement_stats_snapshot | 当前与累计统计 |
| reset() | 调用各节点的 reset，重置 BlockManager、Prefill/Decode Dispatcher 和引擎，并清空请求输出与模型映射；Ray 集群生命周期由集群管理命令负责 |

NodeConfig是每Controller资源与策略配置；必填node_id、num_prefill_engines、num_decode_engines。model_cache_size单位bytes，与server YAML的GiB不同。

`Request` 包含 model、arrival_time、request_id、prompt_token_ids、decode_tokens，以及可选 EOS、stop、min 和 context 参数。构造函数还保留 `multimodal_input` 字段，但当前支持范围要求该字段为 `None`。HTTP API 会补充 EOS 集合与 max_model_len；直接 Python 调用应显式提供这些值。

## 稳定性

Python 接口随控制面实现同步演进，公开入口为本章列出的 `LLMService`、`NodeConfig` 和 `Request`。应用集成应通过这些入口完成；新增能力时在包级 API 增加公共封装和测试。
