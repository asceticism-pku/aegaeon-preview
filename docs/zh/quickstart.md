# 快速开始：完成第一条请求

本流程使用一台 Linux 主机、一张 NVIDIA GPU、一个本地模型、Simple 模式和 TP=1。请先完成 [安装](installation.md)，并具备足够 CPU 内存与共享内存。

## 1. 准备模型与配置

先下载 `Qwen/Qwen3-4B` 完整 checkpoint 到 `/srv/models/Qwen3-4B`。模型目录应包含 config、tokenizer、chat template 及全部权重分片。

先将单节点示例复制为实际使用的配置，再修改模型路径、GPU 名称、容量和 profile 名称：

```bash
cp docs/zh/examples/simple.yaml aegaeon.yaml
```

以下为配置的关键部分：

```yaml
server:
  nnodes: 1
  num_engines: 1
  num_prefill_engines: 0
  num_decode_engines: 0
  tensor_parallel_size: 1
  cpu_num_slabs: 8
  cpu_slab_size_bytes: 1073741824
  model_cache_size_gb: 20
  memory_utilization: 0.775
  startup_models: [Qwen/Qwen3-4B]
  cuda_graph:
    mode: 'off'
models:
  - name: Qwen/Qwen3-4B
    path: /srv/models/Qwen3-4B
    profile: Qwen/Qwen3-4B
    max_model_len: 4096
devices:
  - name: A100_40GB
    memory: 39
    bandwidth: 32
    profile: A100-PCIE-40GB
```

模型和设备的 profile 名称共同对应 `profiles/Qwen/Qwen3-4B/A100-PCIE-40GB/`。`mode: 'off'` 使用带引号的字符串，以兼容会把裸 `off` 解析为布尔值的 YAML 1.1 解析器。示例中的容量、带宽和预算用于展示配置格式；部署时请按目标硬件重新测量和设置。

## 2. 检查或生成 profile

调度估计依赖 `profiles/<model-profile>/<device-profile>/i<input>b<batch>.json`。请按 [Profiling 指南](profiling.md) 检查数据是否完整，并与模型及 GPU 匹配。若缺失，先在目标 GPU 上采集。

```bash
CUDA_VISIBLE_DEVICES=0 python profiles/do_profile.py --model-config aegaeon.yaml --tp 1
```

批量采集需要一定时间，并会占用所选 GPU。采集完成后检查日志和生成的 JSON 文件。

## 3. 启动 Ray 与服务

该示例中每个 worker 最多 pin 住 20 GiB 模型 cache 和 8 GiB CPU KV cache。首次启动 Ray 集群时，在该主机执行：

```bash
df -h /dev/shm
ulimit -Sl
ray start --head --port=6789 --num-cpus="$(nproc)" --resources='{"node_0": 1}'
mkdir -p logs
AEGAEON_LOG_FILE="$PWD/logs/server.log" \
  aegaeon start --config aegaeon.yaml \
  --host 127.0.0.1 --port 8000 --ray-address 127.0.0.1:6789
```

服务启动会初始化共享 CPU 缓存和 actors，耗时由模型大小、缓存配置和机器状态决定。同一组节点只运行一个使用默认 actor / 缓存命名的 API 实例。

单节点可使用 `--ray-address local` 创建本地 Ray runtime，并自动提供 `node_0` 资源。已有多节点集群请传入 head 节点地址。

## 4. 验证服务与模型

```bash
curl -sS http://127.0.0.1:8000/health
curl -sS http://127.0.0.1:8000/v1/models
```

`/health` 预期返回 `{"status":"ok"}`，表示 HTTP 进程可响应。引擎和 GPU 状态通过 `/v1/aegaeon/runtime` 查看；模型列表应包含配置中的名字。

## 5. 发送首个 Chat 请求

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"用一句话解释 KV Cache。"}],"temperature":0,"max_tokens":64}'
```

响应包含 `choices[0].message.content`、`finish_reason` 和 `usage`。文本内容由模型和输入决定；首条请求还会计入 GPU 权重加载与初始化时间。

## 6. 流式输出与运行状态

```bash
curl -N http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"解释 Prefill 和 Decode 的区别。"}],"temperature":0,"max_tokens":128,"stream":true,"stream_options":{"include_usage":true}}'
curl -sS http://127.0.0.1:8000/v1/aegaeon/runtime
```

