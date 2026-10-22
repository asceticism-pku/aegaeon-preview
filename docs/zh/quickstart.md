# 快速开始

完成[安装](installation.md)后，在仓库根目录执行以下命令。示例使用单张 GPU 的 Simple 模式。

## 准备模型

```bash
MODEL_DIR="$HOME/aegaeon-models/Qwen3-4B"
hf download Qwen/Qwen3-4B --local-dir "$MODEL_DIR"
cp docs/zh/examples/simple.yaml aegaeon.yaml
sed -i "s|/srv/models/Qwen3-4B|$MODEL_DIR|" aegaeon.yaml
```

使用其他模型或 GPU 时，在[配置](configuration.md)中调整路径、内存预算和性能 profile。

## 启动服务

```bash
export CUDA_VISIBLE_DEVICES=0
aegaeon start --config aegaeon.yaml --host 127.0.0.1 --port 8000 --ray-address local
```

等待终端显示 `Application startup complete`。

## 发送请求

在另一个终端执行：

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"用一句话解释 KV Cache。"}],"temperature":0,"max_tokens":64}'
```

响应包含 `choices[0].message.content`、`finish_reason` 和 `usage`。客户端接入和流式输出见[在线推理](serving.md)。
