# CUDA Graph

Aegaeon 使用 [Foundry](https://github.com/foundry-org/foundry) 保存和恢复 Decode graph，减少 CPU 提交开销。集成用于 P/D 模式的 Decode Worker，TP=1；Prefill 和 Simple Worker 使用 eager 执行。

## 安装 Foundry

在 Aegaeon 环境中，从仓库根目录执行：

```bash
conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
python -m pip install --no-build-isolation -e ./foundry
export AEGAEON_FOUNDRY_HOOK_PATH="$(python -c 'from pathlib import Path; import foundry.ops; print(Path(foundry.ops.__file__).resolve().parent / "libcuda_hook.so")')"
```

每个 Decode 节点都需安装 Foundry，并使用一致的 hook 和动态库绝对路径。动态库加载错误见[故障排查](troubleshooting.md)。

## 配置

```yaml
server:
  num_engines: 0
  num_prefill_engines: 1
  num_decode_engines: 1
  tensor_parallel_size: 1
  cuda_graph:
    mode: 'save'
    archive_dir: /srv/aegaeon-archives/qwen3
    batch_sizes: [1]
    seq_len_buckets: [4096]
```

`seq_len_buckets` 使用一个最大序列长度。实际 batch size 精确匹配且序列长度在桶内时使用 graph，其余 batch 使用 eager 执行。

## 保存与加载

[SAVE 和 LOAD 示例](examples/README.md)提供完整配置。调整模型路径和预算后启动 SAVE：

```bash
aegaeon start --config docs/zh/examples/foundry-save.yaml --ray-address local
```

出现 `Application startup complete` 后，在另一终端发送请求，将模型加载到 Decode Worker 并捕获图：

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  --data-binary @docs/zh/examples/chat-request.json
```

多模型配置需为每个模型发送请求。等待请求完成，并确认 Worker 日志出现 `captured ... Foundry decode graphs during model load`，再停止服务，以相同的模型、内存布局和存档目录启动 LOAD：

```bash
aegaeon start --config docs/zh/examples/foundry-load.yaml --ray-address local
```

Worker 日志中的 `Foundry decode replay active` 表示正在执行 graph。更改权重、图尺寸、内存布局或运行时依赖后重新录制存档。
