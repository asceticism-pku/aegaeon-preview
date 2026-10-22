# 模型准备

使用完整的本地 checkpoint，包括模型配置、tokenizer、聊天模板及所有 safetensors 权重分片。多节点部署时，各节点使用相同的绝对路径。

## 下载

```bash
hf download Qwen/Qwen3-4B --local-dir /srv/models/Qwen3-4B
```

下载器使用标准的 `HTTP_PROXY` 和 `HTTPS_PROXY` 配置。下载大模型前先配置代理访问。

## 注册模型

```yaml
models:
- name: Qwen/Qwen3-4B
  path: /srv/models/Qwen3-4B
  profile: Qwen/Qwen3-4B
  max_model_len: 4096
```

- `name`：客户端请求中的模型标识。
- `path`：checkpoint 路径。
- `profile`：性能数据目录，见 [Profiling](profiling.md)。
- `max_model_len`：包含输入与输出的总上下文长度。

通常省略 `params`，由 Aegaeon 读取权重元数据；显式填写时以 `2^30` 个参数为单位。

Chat 使用 tokenizer 的聊天模板和 checkpoint 中的 EOS 配置。需要函数调用解析时，可通过 `tool_parser` 指定 vLLM parser，见[在线推理](serving.md)。

完整模型字段见[配置参考](configuration.md)。模型也可以[动态部署](model-management.md)。
