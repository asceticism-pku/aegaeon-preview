# CLI 参考

入口由 setup.py 注册为 `aegaeon=aegaeon.cli:main`；也可使用 `python -m aegaeon`。主命令为 start、deploy、undeploy。

## start

```bash
aegaeon start --config /srv/aegaeon/config.yaml --host 127.0.0.1 --port 8000 --ray-address 127.0.0.1:6789
```

| 参数 | 解析器默认值 | 说明 |
|---|---|---|
| --config | 必填 | YAML 文件 |
| --host | 0.0.0.0 | 监听地址 |
| --port | 8000 | 监听端口 |
| --models | None | 逗号分隔启动模型；覆盖 startup_models |
| --num-prefill-engines | 1 | 正常路径以 YAML ServerConfig 为准 |
| --num-decode-engines | 1 | 同上 |
| --num-engines | 0 | 同上 |
| --tensor-parallel-size | None | 显式非零值覆盖 YAML；0 沿用 YAML |
| --model-cache-size-gb | 20 | fallback 值，CPU 权重缓存 GiB |
| --cpu-num-slabs | 15 | fallback 值 |
| --cpu-slab-size-gb | 1 | fallback 值，GiB |
| --ray-address | None | 指定集群，优先于环境变量 |

请先阅读[配置优先级](configuration.md)。本指南使用 `aegaeon` CLI；`python -m aegaeon.api` 使用另一套参数解析器，默认 TP=1。

## deploy

```bash
aegaeon deploy Qwen/Qwen3-4B --path /srv/models/Qwen3-4B --replicas 2
aegaeon deploy Qwen/Qwen3-4B --nodes node_0 node_1
```

| 参数 | 默认值 | 说明 |
|---|---|---|
| model | 位置参数，必填 | 服务中的模型名字 |
| --path | None | 本地权重目录；未指定时先查注册配置，再尝试下载 |
| --host | localhost | API 服务主机 |
| --port | 8000 | API 端口 |
| --max-model-len | None | 上下文 override |
| --replicas | 1 | 自动部署时的最小 READY 副本数 |
| --nodes | None | 一个或多个明确节点 ID；绕过自动 placement |
| --tool-parser | None | 为该模型别名覆盖 vLLM tool parser |

客户端使用 HTTPConnection，超时为 300 秒。HTTP 200 且 JSON `status` 为 `partial` 或 `failed` 非空时，CLI 报告部分完成并以状态码 1 退出；完整成功才显示成功信息。直接使用 HTTP API 时，读取 `status`、`failed` 和 `ready_nodes` 确认结果。

## undeploy

```bash
aegaeon undeploy Qwen/Qwen3-4B --nodes node_1
aegaeon undeploy Qwen/Qwen3-4B
```

| 参数 | 默认值 | 含义 |
|---|---|---|
| model | 必填位置参数 | 服务中的模型名 |
| --host | localhost | API 主机 |
| --port | 8000 | API 端口 |
| --nodes | 全部已知副本 | 只卸载列出的节点 |

忙副本会进入 DRAINING，HTTP 响应报告部分失败，CLI 以状态码 1 退出；请求结束后再次执行卸载。以 runtime 中的副本和缓存状态确认资源已释放。

## 查询

```bash
aegaeon --help
aegaeon start --help
aegaeon deploy --help
aegaeon undeploy --help
```

CLI 当前包含 `start`、`deploy` 和 `undeploy`。状态与列表通过 HTTP 查询，benchmark/profile 使用仓库脚本，服务停止由进程管理器负责。
