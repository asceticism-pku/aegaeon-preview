# 快速开始：完成第一条请求

本示例用一张 A100 PCIe 40GB GPU，以 Simple 模式运行 Qwen3-4B（TP=1，CUDA Graph 关闭）。先按[安装指南](installation.md)使用 `--cuda cu129` 完成已测试路线的安装，激活已安装的 Conda 环境，在源码仓库根目录执行命令。

## 1. 准备模型与配置

下载完整 checkpoint；已有本地模型时，将 `MODEL_DIR` 设为其绝对路径并跳过 `hf download`：

```bash
MODEL_DIR="$HOME/aegaeon-models/Qwen3-4B"
hf download Qwen/Qwen3-4B --local-dir "$MODEL_DIR"
cp docs/zh/examples/simple.yaml aegaeon.yaml
sed -i "s|/srv/models/Qwen3-4B|$MODEL_DIR|" aegaeon.yaml
```

示例在主机内存和 `/dev/shm` 中预留 20 GiB 模型缓存、8 GiB CPU KV 缓存，还需为 Ray 等进程留出余量。仓库已附匹配的 profile；更换模型或 GPU 时，调整[配置](configuration.md)并采集匹配的 [profile](profiling.md)。下载问题见[模型准备](models.md#下载与代理排障)。

## 2. 启动服务

选择一张空闲 GPU；`--ray-address local` 自动启动 Ray：

```bash
export CUDA_HOME="$CONDA_PREFIX"
CUDA_VISIBLE_DEVICES=0 aegaeon start --config aegaeon.yaml \
  --host 127.0.0.1 --port 8000 --ray-address local
```

等待 `Application startup complete`。同一组节点运行一个使用默认 actor / 缓存命名的 API 实例；其他部署方式见[部署指南](deployment.md)。

## 3. 发送首个 Chat 请求

在另一个终端的仓库根目录执行：

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  --data-binary @docs/zh/examples/chat-request.json
```

成功响应包含 `choices[0].message.content` 和 token `usage`。首次请求还会将权重加载到 GPU。流式输出见 [API 参考](api.md)，启动或请求问题见[故障排查](troubleshooting.md)。
