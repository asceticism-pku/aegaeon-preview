# 快速开始：完成第一条请求

本示例使用一台 Linux 主机、一张空闲的 A100 40 GB GPU、Simple 模式和 TP=1，关闭 CUDA Graph。先完成[安装](installation.md)，然后在仓库根目录执行以下命令。

仓库已包含示例配置和匹配的 profile。其他模型或 GPU 需要调整[配置](configuration.md)并采集匹配的 [profile](profiling.md)。

## 1. 准备模型与配置

下载完整的 `Qwen/Qwen3-4B` checkpoint。需要代理时，先按[模型下载说明](models.md#下载与代理排障)配置网络。已有完整本地模型时，将 `MODEL_DIR` 改为其绝对路径，并跳过下载命令。

```bash
MODEL_DIR="$HOME/aegaeon-models/Qwen3-4B"
hf download Qwen/Qwen3-4B --local-dir "$MODEL_DIR"
cp docs/zh/examples/simple.yaml aegaeon.yaml
sed -i "s|/srv/models/Qwen3-4B|$MODEL_DIR|" aegaeon.yaml
```

## 2. 启动服务

选择一张空闲 GPU；下方以 GPU 0 为例。`local` 自动创建带 `node_0` 资源的本地 Ray runtime。

```bash
export CUDA_VISIBLE_DEVICES=0
aegaeon start --config aegaeon.yaml \
  --host 127.0.0.1 --port 8000 --ray-address local
```

启动会初始化 CPU 缓存和 actors。等待终端显示 `Application startup complete`；同一组节点只运行一个使用默认 actor / 缓存命名的 API 实例。已有 Ray 集群或多节点部署见[部署指南](deployment.md)。

## 3. 发送首个 Chat 请求

在另一个终端执行：

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"用一句话解释 KV Cache。"}],"temperature":0,"max_tokens":64}'
```

成功响应包含 `choices[0].message.content`、`finish_reason` 和 `usage`。首条请求还包含 GPU 权重加载时间。

流式输出与运行状态查询见 [API 参考](api.md)；启动或请求失败时见[故障排查](troubleshooting.md)。
