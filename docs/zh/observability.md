# 日志、监控与健康检查

## 日志

```bash
mkdir -p logs
AEGAEON_LOG_FILE="$PWD/logs/server.log" \
  aegaeon start --config /srv/aegaeon/config.yaml --ray-address 127.0.0.1:6789
```

默认日志目标为 output.log。多进程节点写同名路径时要明确文件位于哪台机器。日志轮转和容量管理由生产环境的进程管理器与日志采集系统负责。

## Runtime

```bash
curl -sS http://127.0.0.1:8000/v1/aegaeon/runtime
```

主要顶层字段：timestamp、model_placement_policy、request_routing_policy、last_model_placement_decision、model_placement_stats、model_placements、request_routing、engines、events、work_stealing、decode_load、nodes。

request_routing 是 API 已分配但未完成的 reservation；nodes.request_load 是 Controller 观测的活动请求，两者采样时间不同。cumulative_assignments 是历史累计分配量，当前队列深度请读取活动请求字段。

runtime 依次向各 Controller 获取快照，因此字段来自相邻但不同的采样时刻；跨节点墙钟使用各节点系统时间。Controller RPC 失败时本次 runtime 请求返回错误。

## Runtime SSE

```bash
curl -N http://127.0.0.1:8000/v1/aegaeon/events
```

事件名为 runtime，data 包含 engines/events；约每 0.25 秒轮询一次，状态保持不变时约每 15 秒发送 keep-alive。该接口提供实时 SSE；历史存储和断点恢复可由外部观测系统实现。

## 访问统计与限流

metrics返回timestamp、window_seconds、limits、metrics；默认窗口60秒，按模型统计。环境变量：

```bash
export AEGAEON_MAX_REQUESTS_PER_SECOND=10
export AEGAEON_MAX_TOKENS_PER_SECOND=1000
```

限制按模型和当前 60 秒窗口计算。token 限制使用已经记录的生成量，在途输出完成后进入统计；check 与 record 是两次独立操作。请求计数发生在生成参数校验之前，因此校验失败的请求也计入窗口。每个 API 进程独立保存统计。

429 发生在请求接入检查阶段；已经进入执行阶段的请求按各自停止条件完成。

## GPU power

gpu-power 查询 API 所在主机的 nvidia-smi。远端 Ray 节点需要分别采集后再聚合；每请求能量还需要按请求时间窗口积分，进程 GPU 利用率则使用进程级指标。见[功耗指南](energy.md)。

## 健康层级

`/health` 返回 status=ok，用于检查 HTTP 进程。外部监控可继续分层检查 runtime 节点、Engine 状态和低频真实推理；真实推理检查使用已确认的短请求。完整 readiness 可在部署侧组合这些信号。
