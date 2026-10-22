# 模型管理

Aegaeon 在节点级 CPU 权重缓存中部署模型副本，并在副本之间路由请求。每个请求的 Prefill 和 Decode 始终位于同一个物理节点。

## 副本状态

| 状态 | 含义 |
|---|---|
| ABSENT | 尚无副本 |
| LOADING | 正在部署 |
| READY | 接收新请求 |
| DRAINING | 不接收新请求；已有请求结束后重试卸载 |
| FAILED | 部署失败 |

## 部署与路由

```bash
aegaeon deploy Qwen/Qwen3-4B --replicas 2
aegaeon undeploy Qwen/Qwen3-4B --nodes node_1
```

`replica_count` 是副本数的下限目标；减少副本使用 undeploy。`node_ids` 可直接指定目标节点。

`model_placement_policy: least-loaded` 根据缓存容量、驱逐代价和请求负载选择节点；`request_routing_policy: least-outstanding` 在 READY 副本中选择未完成请求最少的节点。两者也支持 `round-robin`。

## 操作结果

部署和卸载响应包含 `status`、各节点结果及 `failed`。HTTP 200 中仍会出现 `status: partial`，应检查该字段确认操作结果。

卸载时副本进入 DRAINING；已有请求结束后重试。使用 `server.startup_models: []` 可启动空服务，后续动态部署沿用服务的缓存预算。

通过 `GET /v1/aegaeon/runtime` 查看放置和路由状态。命令选项见 [CLI 参考](cli.md)，HTTP 请求见 [API 参考](api.md)。
