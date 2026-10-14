# CLI 参考

入口由 setup.py 注册为 `aegaeon=aegaeon.cli:main`；也可使用 `python -m aegaeon`。主命令为 doctor、start、deploy、undeploy。

## doctor

```bash
aegaeon doctor
CUDA_VISIBLE_DEVICES=0 aegaeon doctor --config /srv/aegaeon/config.yaml --check-cuda
aegaeon doctor --config /srv/aegaeon/config.yaml --models Qwen/Qwen3-4B --strict
```

| 参数 | 解析器默认值 | 说明 |
|---|---|---|
| --config | None | 用于拓扑、本地模型/profile 和内存预算检查的 YAML 配置 |
| --models | None | 逗号分隔启动模型；与 start 一样覆盖 startup_models |
| --tensor-parallel-size | None | 显式非零值覆盖 YAML；0 沿用 YAML；仅支持 TP=1 |
| --check-cuda | false | 在已分配的 GPU 上执行微型 PyTorch/CuPy CUDA 操作、导入 vLLM 原生扩展，并探测一页主机内存的 CUDA 注册 |
| --strict | false | 警告和失败均以状态码 1 退出 |

默认检查覆盖包括 `aegaeon.ops` 在内的运行时导入、固定后端版本、CUDA/C++
工具链、Rust/Cargo 可用性、共享内存写入/mmap 与容量、主机可用内存和 memlock 限额。
`--config` 另行检查启动模型选择、本地模型资源与权重分片可读性、声明的
decode profile 候选、拓扑约束和配置共享内存预算。检查读取本地文件，无需下载模型或启动 Ray。

结果显示为 `OK`、`WARN` 或 `FAIL`。失败以状态码 1 退出；只有警告时普通
模式返回 0，`--strict` 返回 1。警告示例包括 memlock 限额低于预算、
有限额但预算未知，以及扩展已安装但缺少 Rust/Cargo。普通 `doctor` 省略 CUDA tensor 运算；
请在每台 worker 节点已分配的 GPU 上运行 `--check-cuda`。检查范围为本节点；
完整规模的主机内存注册、Ray 集群就绪状态和真实推理属于独立的验收步骤。
详见[运行检查](installation.md#运行检查与-wheel-安装)。

`start` 在导入 API 前，将配置及模型/TP 覆盖参数传入静态配置和本节点
主机预检，省略 CUDA 查询。本地模型文件错误、cache 内存容量不足等确定
失败会阻止启动；静态 profile 候选缺失或格式错误显示 `WARN`，静态 `OK`
也仍标明实际 worker GPU 尚未检查。通过
`doctor --config PATH --check-cuda` 验证实际 GPU/profile；此时引擎所需
GPU 的 profile 缺失或格式错误会显示 `FAIL`。`--strict` 也会将静态
不确定性所产生的警告转为非零退出状态。

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
aegaeon doctor --help
aegaeon start --help
aegaeon deploy --help
aegaeon undeploy --help
```

CLI 包含 `doctor`、`start`、`deploy` 和 `undeploy`。状态与列表通过 HTTP 查询，benchmark/profile 使用仓库脚本，服务停止由进程管理器负责。
