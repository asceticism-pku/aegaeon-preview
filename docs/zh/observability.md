# 日志与监控

## 日志

启动前设置日志路径：

```bash
mkdir -p logs
export AEGAEON_LOG_FILE="$PWD/logs/server.log"
```

Ray actor 日志也可在对应 session 的日志目录中查看。

## Runtime 接口

| 接口 | 内容 |
|---|---|
| `/health` | API 存活状态 |
| `/v1/models` | 已注册模型 |
| `/v1/aegaeon/runtime` | 节点、Engine、副本、路由、缓存及 Decode 负载 |
| `/v1/aegaeon/events` | Runtime SSE 事件 |
| `/v1/aegaeon/metrics` | 各模型的访问统计与限流设置 |

```bash
curl -sS http://127.0.0.1:8000/v1/aegaeon/runtime
curl -N http://127.0.0.1:8000/v1/aegaeon/events
```

通过副本的 READY 状态确认模型可用性。访问统计采用 60 秒窗口，限流通过 `AEGAEON_MAX_REQUESTS_PER_SECOND` 和 `AEGAEON_MAX_TOKENS_PER_SECOND` 配置。
