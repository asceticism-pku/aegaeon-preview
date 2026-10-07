# CUDA Graph

Decode 阶段每生成一个 token，都要由 CPU 重复提交一组相似的 GPU 操作；使用图回放可以减少这部分提交开销，从而降低 Decode 的 TPOT。因此，Aegaeon 在 P/D 分离部署的 Decode Worker 中应用 CUDA Graph。

Aegaeon 使用开源项目 [Foundry](https://github.com/foundry-org/foundry) 负责把录制好的图、CUDA 二进制和确定性显存布局保存到磁盘。第一次运行用 `save` 模式生成存档，后续进程用 `load` 模式恢复，不需要重新录制。

## 支持范围

- 仅接入 P/D 模式的 Decode Worker，当前支持范围固定为 TP=1。
- 图中包含模型的 Decode forward；logits 计算和 argmax 仍在图外执行。
- Prefill、混合 Prefill/Decode 批次和 Simple Worker 走普通 eager 路径。

一次 forward 还需要同时满足以下条件才会使用图：

1. 批次全部是单 token Decode；
2. 实际 batch size 位于 `batch_sizes`；
3. 批次中的最大序列长度不超过 `seq_len_buckets` 中配置的上限；
4. 当前模型的图已经成功恢复。

其余情况直接执行 eager forward，不会导致请求失败。是否命中以 Worker 日志中的 `Foundry decode replay active` 为准。

## 安装 Foundry

`foundry/` 是独立子项目，不会随 Aegaeon 的 `pip install -e .` 一起安装。请在运行 Aegaeon 的同一个 Python 环境中构建：

```bash
conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
python -m pip install --no-build-isolation -e ./foundry

export AEGAEON_FOUNDRY_HOOK_PATH="$(python -c 'from pathlib import Path; import foundry.ops; p = Path(foundry.ops.__file__).resolve().parent / "libcuda_hook.so"; assert p.is_file(), p; print(p)')"
test -f "$AEGAEON_FOUNDRY_HOOK_PATH"
```

`libcuda_hook.so` 必须在 Worker 进程启动前通过 `LD_PRELOAD` 装载；Aegaeon 会根据 `AEGAEON_FOUNDRY_HOOK_PATH` 为 Ray Worker 设置该环境。多节点部署时，每个运行 Decode Worker 的节点都要单独安装 Foundry，并提供该节点上的绝对路径。

自定义 Boost 路径、CUDA forward-compatibility 库和常见加载错误见[故障排查](troubleshooting.md)。

## 配置要点

下面的配置只录制 batch size 1、最大序列长度 4096 的 Decode 图：

```yaml
server:
  num_engines: 0
  num_prefill_engines: 1
  num_decode_engines: 1
  tensor_parallel_size: 1
  cuda_graph:
    mode: 'save'
    archive_dir: /srv/aegaeon-archives/qwen3-test
    batch_sizes: [1]
    seq_len_buckets: [4096]
```

当前实现要求 `seq_len_buckets` 只包含一个值，表示录制图支持的最大序列长度。`batch_sizes` 可以包含多个正整数，但运行时 batch size 必须精确匹配其中一个值。录制更多 batch size 会增加模型加载时间和图占用的显存，应按实际流量选择。

完整配置可从 [`foundry-save.yaml`](examples/foundry-save.yaml) 和 [`foundry-load.yaml`](examples/foundry-load.yaml) 开始修改。

## 录制存档

录制前先用 `mode: 'off'` 运行一组固定请求，保存完整 token ID。然后：

1. 为 `archive_dir` 使用新的空目录，并将 `mode` 改为 `'save'`。
2. 启动服务，让之后需要在 `load` 模式使用的每个模型都在 Decode Worker 上完成一次模型加载。启动模型会在服务启动时录制；其他模型可通过部署或请求触发加载。
3. 检查每个模型、每个 rank 都出现 `captured ... Foundry decode graphs` 日志，并保留本次运行的源码 revision、配置和依赖版本。
4. 用与基线相同的请求检查输出 token ID。

图录制发生在模型加载阶段，服务开始处理该模型的 Decode 请求前就已完成。

## 加载并验证

使用存档时，只把配置中的 `mode` 改为 `'load'`，其余模型、图形状和内存布局配置保持不变，并从新进程启动服务。

LOAD 会检查模型注册表中的所有模型，而不只是 `startup_models`。任何模型缺少对应存档都会使初始化失败。模型加载完成后应能看到 `loaded ... Foundry decode graph shapes`；第一个命中图的 Decode 批次会输出 `Foundry decode replay active`。

可以进行以下两项验证：

1. 对相同输入比较 OFF 与 LOAD 的完整 token ID。
2. 多模型部署中执行 A→B→A 切换，再次检查 A 的输出和 replay 日志。

## 何时需要重新录制

Foundry 存档依赖模型权重、Aegaeon/Foundry 代码、PyTorch、vLLM、CUDA、驱动、GPU 架构以及 CUDA Graph 的内存布局配置。以上任一项变化后，都应使用新的空目录重新执行 SAVE，并再次与 OFF 基线比较 token ID。旧存档不保证跨版本可移植。
