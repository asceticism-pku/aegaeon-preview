# 部署

每个物理 Ray 节点运行一个 Controller，各 Engine 共享节点的 CPU 缓存。每个 Engine 使用一张 GPU，`nnodes` 表示物理节点数。

## 执行模式

| 模式 | server 配置 |
|---|---|
| Simple | `num_engines > 0`，P/D Engine 数量均为 0 |
| P/D | `num_engines: 0`，至少一个 Prefill 和一个 Decode Engine |

单卡、P/D 和多节点布局见[配置示例](examples/README.md)。各节点提前准备模型文件和 profile。

## Ray 集群

单机可使用 `--ray-address local`。两台主机分别启动 Ray：

```bash
# Head 节点
ray start --head --port=6789 --resources='{"node_0":1}'
```

```bash
# Worker 节点
ray start --address=HEAD_IP:6789 --resources='{"node_1":1}'
```

在配置中设置 `server.nnodes: 2`，然后从 Head 节点启动 API：

```bash
aegaeon start --config /srv/aegaeon/config.yaml --ray-address HEAD_IP:6789
```

## 资源

各节点保持一致的 GPU 可见性配置。为 Worker 的 `worker_num_cpus` 和 Controller 分配足够 Ray CPU，通过 `ray status` 查看资源。

[SLO 环境变量](scheduling.md#slo-设置)在启动 Ray 前于所有节点设置。独立服务使用隔离的 Ray 实例和共享内存目录。停止前台服务会断开 driver；已有 Ray 集群单独管理。
