# CUDA Graph

Decode 阶段每生成一个 token，都要由 CPU 重复提交一组相似的 GPU 操作；使用图回放可以减少这部分提交开销，从而降低 Decode 的 TPOT。因此，Aegaeon 在 P/D 分离部署的 Decode Worker 中应用 CUDA Graph。

Aegaeon 使用开源项目 [Foundry](https://github.com/foundry-org/foundry) 负责把录制好的图、CUDA 二进制和确定性显存布局保存到磁盘。第一次运行用 `save` 模式生成存档，后续进程用 `load` 模式恢复，不需要重新录制。

本文的 Foundry 安装及 OFF / SAVE / LOAD、完整 token ID 对比和 replay 日志验证已在 cu129 / CUDA 12.9、A100 PCIe 40GB、驱动 `535.247.01` 环境中测试。默认安装器按驱动和 GPU 预检选择 CUDA 13.0，条件未通过时在安装包之前回退 cu129；使用 `--cuda cu129` 可强制选择本文已测试的路线。CUDA 13.0 路线尚未完成 Foundry 构建或 CUDA Graph 端到端测试。下面的 CUDA 12.9 compat 包仅用于 CUDA 12.9 环境；CUDA 13.0 环境应使用与该版本匹配的驱动库，分别构建 Foundry 并重新生成存档。

## 支持范围

- 仅接入 P/D 模式的 Decode Worker，当前支持范围固定为 TP=1。
- 图中包含模型的 Decode forward；logits 计算和 argmax 仍在图外执行。
- Prefill、混合 Prefill/Decode 批次和 Simple Worker 走普通 eager 路径。

一次 forward 还需要同时满足以下条件才会使用图：

1. 批次全部是单 token Decode；
2. 实际 batch size 位于 `batch_sizes`；
3. 批次中的最大序列长度不超过 `seq_len_buckets` 中配置的上限；
4. 当前模型的图已经成功录制或恢复。

未命中上述图条件时，执行 eager forward。是否命中以 Worker 日志中的 `Foundry decode replay active` 为准。

## 安装 Foundry

`foundry/` 是独立子项目，不会随 Aegaeon 的 `pip install -e .` 一起安装。在运行 Aegaeon 的同一个 Conda/Python 环境中构建：

```bash
conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
python -m pip install --no-build-isolation -e ./foundry
```

`$CONDA_PREFIX/lib` 提供环境中的 Boost 和 `libstdc++`。请在 Python 启动前设置上述搜索路径；系统 `libstdc++` 缺少 NumPy 所需的 `GLIBCXX_*` 符号时，NumPy 导入失败，后续显示 `NP_SUPPORTED_MODULES` 错误。

Conda 安装出现 `shards_cache` / SQLite `database is locked` 时，使用下面的同一安装命令关闭该命令的分片 repodata 缓存，再继续后续步骤：

```bash
CONDA_PLUGINS_USE_SHARDED_REPODATA=false conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
```

### CUDA 12.9 兼容库

Foundry 直接调用 `cuFuncGetName`；驱动库缺少该符号时，导入 `foundry.ops` 会失败。驱动库已经提供该符号的环境跳过本节，直接取得下节的 hook 路径。需要 CUDA forward-compatibility 库时，先按 [NVIDIA 官方兼容支持表](https://docs.nvidia.com/deploy/cuda-compatibility/forward-compatibility.html)确认 GPU 和驱动组合受支持，再选择与 `torch.version.cuda` 和操作系统匹配的库。

下面的下载示例适用于 Ubuntu 20.04 x86_64、CUDA 12.9，以及官方支持表允许的 GPU 和驱动组合，例如 A100 配合驱动 `535.247.01`。其他平台或 CUDA 版本应使用对应的 NVIDIA 包。在仓库根目录执行，使用 NVIDIA 官方包及其 SHA256 校验值，解包到新的独立目录，无需 root 权限。需要 `curl`、`sha256sum` 和 Ubuntu 的 `dpkg-deb` 命令；下载或校验失败时先解决错误，再继续后续步骤。

```bash
python -c "import torch; assert torch.version.cuda == '12.9', torch.version.cuda"
export CUDA_COMPAT_DIR="$(mktemp -d "$PWD/cuda-compat-12.9-XXXXXX")"
curl --fail --location --retry 3 \
  'https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-compat-12-9_575.57.08-0ubuntu1_amd64.deb' \
  -o "$CUDA_COMPAT_DIR/cuda-compat-12-9.deb"
printf '%s  %s\n' \
  '0e67e0011b8cb359cecf5d60c3dd7be9c2a69f1b7eb531a4536b83b02173d87a' \
  "$CUDA_COMPAT_DIR/cuda-compat-12-9.deb" | sha256sum -c -
dpkg-deb -x "$CUDA_COMPAT_DIR/cuda-compat-12-9.deb" "$CUDA_COMPAT_DIR/root"
export AEGAEON_CUDA_COMPAT_PATH="$CUDA_COMPAT_DIR/root/usr/local/cuda-12.9/compat"
test -f "$AEGAEON_CUDA_COMPAT_PATH/libcuda.so.1"
printf '%s\n' "$AEGAEON_CUDA_COMPAT_PATH"
```

保留这个目录，并记录最后打印的绝对路径。后续终端或服务重启时，先将 `AEGAEON_CUDA_COMPAT_PATH` 重新设为该绝对路径，再执行下面的路径检查和搜索路径设置。已有兼容库时也可直接设置它的绝对路径；检查失败时先补齐库，检查通过后再导入 Foundry。compat 库应位于搜索路径最前面，Conda 的 `lib` 目录紧随其后。

```bash
AEGAEON_CUDA_COMPAT_PATH=$(python - <<'PY'
import os
from pathlib import Path
import torch
version = torch.version.cuda
configured = os.environ.get("AEGAEON_CUDA_COMPAT_PATH")
candidates = [Path(configured).expanduser()] if configured else [
    Path(f"/usr/local/cuda-{version}/compat"),
    Path.home() / f".local/cuda-{version}/compat",
]
path = next((p.resolve() for p in candidates if (p / "libcuda.so.1").is_file()), None)
assert path is not None, f"Prepare the CUDA {version} compatibility library first"
print(path)
PY
)
export AEGAEON_CUDA_COMPAT_PATH
test -f "$AEGAEON_CUDA_COMPAT_PATH/libcuda.so.1"
export LD_LIBRARY_PATH="$AEGAEON_CUDA_COMPAT_PATH${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

### 取得 hook 路径

以上路径设置完成后，再导入 Foundry 并取得 hook 路径：

```bash
export AEGAEON_FOUNDRY_HOOK_PATH="$(python -c 'from pathlib import Path; import foundry.ops; p = Path(foundry.ops.__file__).resolve().parent / "libcuda_hook.so"; assert p.is_file(), p; print(p)')"
test -f "$AEGAEON_FOUNDRY_HOOK_PATH"
```

`libcuda_hook.so` 必须在 Worker 进程启动前通过 `LD_PRELOAD` 装载；Aegaeon 会根据 `AEGAEON_FOUNDRY_HOOK_PATH` 为 Ray Worker 设置该环境。多节点部署时，每个运行 Decode Worker 的节点都要安装 Foundry。API 进程设置的 hook、compat 和 `LD_LIBRARY_PATH` 会统一传给各节点；这些绝对路径必须在所有目标节点上同名有效。当前 NodeConfig 未提供每节点 hook 或 compat 路径覆盖。

自定义 Boost 路径、CUDA forward-compatibility 库和常见加载错误见[故障排查](troubleshooting.md)。

## 配置要点

下面的 1P+1D 配置需要两张空闲 GPU，只录制 batch size 1、最大序列长度 4096 的 Decode 图。GPU 需求见[部署指南](deployment.md#pd-分离)。

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
2. 启动服务，并为之后需要在 `load` 模式使用的每个模型发送至少一个进入 Decode 的生成请求。`startup_models` 和 deploy 只注册模型并缓存 CPU 权重；实际 Decode GPU 模型加载时才会录制，以 capture 日志确认存档完成。
3. 检查每个模型、每个 rank 都出现 `captured ... Foundry decode graphs` 日志，并保留该次运行的源码 revision、配置和依赖版本。
4. 用与基线相同的请求检查输出 token ID。

图录制发生在模型加载阶段，服务开始处理该模型的 Decode 请求前就已完成。

## 加载并验证

使用存档时，只把配置中的 `mode` 改为 `'load'`，其余模型、图形状和内存布局配置保持不变，并从新进程启动服务。

LOAD 会检查模型注册表中的所有模型，而不只是 `startup_models`。任何模型缺少对应存档都会使初始化失败。模型加载完成后应能看到 `loaded ... Foundry decode graph shapes`；第一个命中图的 Decode 批次会输出 `Foundry decode replay active`。

可以进行以下两项验证：

1. 对相同输入比较 OFF 与 LOAD 的完整 token ID。
2. 多模型部署中执行 A→B→A 切换，再次检查 A 的输出和 replay 日志。

## 单模型验证示例

完成 Aegaeon 和 Foundry 安装后，在仓库根目录执行。使用完整的本地 `Qwen/Qwen3-4B` checkpoint；下列命令沿用示例中的 profile、图形状和内存预算，只修改模型路径、存档路径和模式。`mktemp` 创建独立的新目录，SAVE 不使用历史存档。

```bash
export MODEL_DIR=/absolute/path/to/Qwen3-4B
export GRAPH_RUN_DIR="$(mktemp -d "$PWD/foundry-check-XXXXXX")"
python - <<'PY'
import copy
import os
from pathlib import Path
import yaml
run = Path(os.environ["GRAPH_RUN_DIR"])
model = Path(os.environ["MODEL_DIR"]).resolve()
assert model.is_dir(), model
template = yaml.safe_load(Path("docs/zh/examples/foundry-save.yaml").read_text())
template["models"][0]["path"] = str(model)
template["server"]["cuda_graph"]["archive_dir"] = str(run / "archive")
for mode in ("off", "save", "load"):
    config = copy.deepcopy(template)
    config["server"]["cuda_graph"]["mode"] = mode
    (run / f"{mode}.yaml").write_text(yaml.safe_dump(config, sort_keys=False))
print(run)
PY
```

第一个终端选择两张空闲 GPU（以下以 0、1 为例），从 OFF 开始启动服务：

```bash
export CUDA_VISIBLE_DEVICES=0,1
MODE=off
aegaeon start --config "$GRAPH_RUN_DIR/${MODE}.yaml" \
  --host 127.0.0.1 --port 8000 --ray-address local \
  2>&1 | tee "$GRAPH_RUN_DIR/${MODE}.log"
```

等待 `Application startup complete`。第二个终端进入同一仓库和 Python 环境，将 `GRAPH_RUN_DIR` 设为上一步打印的绝对目录，再发送固定请求。下面设置最多生成 128 个 token；验证脚本会要求实际至少生成 2 个 token，以覆盖 Decode：

```bash
GRAPH_RUN_DIR=/absolute/path/printed/above
MODE=off
curl --fail-with-body -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-4B","messages":[{"role":"user","content":"What are the differences between processes and threads?"}],"temperature":0,"max_tokens":128,"return_token_ids":true}' \
  -o "$GRAPH_RUN_DIR/${MODE}.json"
```

请求完成后，在第一个终端按 Ctrl+C 正常停止服务，确认本服务进程退出、GPU 资源释放。然后在两个终端都将 `MODE` 改为 `save`，重复启动和请求；再次正常停止、等待资源释放后，以 `MODE=load` 重复。三个模式分别使用新进程，保留其他配置一致。

完成三个阶段后，在第二个终端比较完整输入和输出 token ID，并检查日志：

```bash
export GRAPH_RUN_DIR
python - <<'PY'
import json
import os
from pathlib import Path
import re
run = Path(os.environ["GRAPH_RUN_DIR"])
results = [json.loads((run / f"{mode}.json").read_text()) for mode in ("off", "save", "load")]
prompts = [r["prompt_token_ids"] for r in results]
tokens = [r["choices"][0]["token_ids"] for r in results]
assert all(len(ids) >= 2 for ids in tokens), "Need at least two generated tokens"
assert all(len(p) + len(t) <= 4096 for p, t in zip(prompts, tokens)), "Sequence exceeds graph bucket"
assert prompts[0] == prompts[1] == prompts[2], "Prompt token IDs differ"
assert tokens[0] == tokens[1] == tokens[2], "Generated token IDs differ"
logs = {mode: (run / f"{mode}.log").read_text(errors="replace") for mode in ("off", "save", "load")}
replay = "Foundry decode replay active"
capture = r"captured(?: \d+)? Foundry decode graphs?"
assert replay not in logs["off"], "OFF unexpectedly replayed a graph"
assert re.search(capture, logs["save"]) and replay in logs["save"], "SAVE capture/replay missing"
assert re.search(r"loaded \d+ Foundry decode graph shapes", logs["load"]), "LOAD marker missing"
assert replay in logs["load"] and not re.search(capture, logs["load"]), "LOAD replay missing or captured again"
print(f"OFF / SAVE / LOAD match: {len(tokens[0])} generated tokens; graph logs verified")
PY
```

SAVE 日志须同时出现 `captured ... Foundry decode graphs` 和 `Foundry decode replay active`；LOAD 日志须出现 `loaded ... Foundry decode graph shapes` 和 `Foundry decode replay active`，仅恢复已有存档。图验证以完整 token ID 相等和 replay 日志同时成立为准。

## 何时需要重新录制

Foundry 存档依赖模型权重、Aegaeon/Foundry 代码、PyTorch、vLLM、CUDA、驱动、GPU 架构以及 CUDA Graph 的内存布局配置。以上任一项变化后，都应使用新的空目录重新执行 SAVE，并再次与 OFF 基线比较 token ID。旧存档不保证跨版本可移植。
