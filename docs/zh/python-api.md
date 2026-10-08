# Python API

## 对外导出

包以惰性导出方式提供 `LLMService`、`NodeConfig` 和 `Request`。调用方先将输入编码为 token ID，构造 `Request`，再传给 `LLMService.serve()`。这个接口使用 Aegaeon 的请求类型和控制逻辑，不提供 `vllm.LLM.generate(prompt, SamplingParams)`。

## 同步推理示例

先完成[安装](installation.md)，将 `docs/zh/examples/simple.yaml` 的模型路径、设备与 profile 调整为当前环境，并启动地址为 `127.0.0.1:6789`、具有 `node_0` 自定义资源的 Ray 集群，参见[运行环境](environment.md)。从仓库根目录在 Linux/CUDA 环境运行：

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

示例传入的是 Completion 风格的 token。Chat 调用方须先应用正确的聊天模板。直接使用 Python 接口时，还须自行处理 HTTP 层提供的字符串 stop 和响应包装。

## LLMService

| 入口 | 用途 / 说明 |
|---|---|
| LLMService(cluster_config, model_placement_policy=None, request_routing_policy=None) | 初始化 Controller 和 event loops，并同步等待初始化完成 |
| serve(requests) | 为每个请求预留节点，结束后释放；通过 `asyncio.run` 并发提交，按请求返回 `StepOutput` 列表或异常对象；异步应用需在线程或独立进程中调用 |
| replay(result_file, num_models, arrival_rate, ...) | 按请求到达时间重放并写入项目统计；计时方法和 QoS 定义见实现 |
| deploy_model_replicas(...) | async 副本部署入口 |
| undeploy_model_replicas(...) | async 副本卸载入口 |
| reserve_node / release_node_reservation | 维护 HTTP 请求的路由占用记录；直接调用时须成对使用 |
| request_routing_snapshot / model_placement_stats_snapshot | 当前与累计统计 |
| reset() | 遗留辅助入口；Controller 的实现引用了 Dispatcher / Engine 中缺失的 reset 方法。重置运行中的服务时，停止进程、确认资源释放后重新启动 |

`NodeConfig` 配置每个 Controller 的资源与策略；`node_id`、`num_prefill_engines`、`num_decode_engines` 为必填项。`model_cache_size` 的单位是 bytes，server YAML 使用 GiB。

`Request` 包含 `model`、`arrival_time`、`request_id`、`prompt_token_ids`、`decode_tokens`，以及可选的 EOS、stop、min 和 context 参数。构造函数保留了 `multimodal_input` 字段，当前支持范围要求它为 `None`。HTTP API 会补充 EOS 集合与 `max_model_len`；直接调用 Python 接口时应显式提供这些值。

## 稳定性

Python 接口随控制面实现演进。应用集成应使用本章列出的 `LLMService`、`NodeConfig` 和 `Request`；新增能力时，应在包级 API 中补充公共封装和测试。
