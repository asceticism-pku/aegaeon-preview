# 安装

Aegaeon 面向 Linux x86_64、Python 3.10 和 NVIDIA GPU。源码安装要求 glibc 2.28 或更新版本；首版预编译 wheel 面向 `manylinux_2_31_x86_64`，要求 glibc 2.31 或更新版本。驱动必须支持所选的 CUDA 运行环境。

Aegaeon 的预编译发行包尚未发布到 PyPI。`pip install aegaeon` 的发行流程仍在验收，目前请使用源码安装，或安装明确提供的本地 wheel。

| 安装方式 | CUDA | 当前状态 |
|---|---|---|
| 源码安装 | `cu129` 或 `cu130` | 在本机编译 Aegaeon 的 CUDA 和 Rust 扩展 |
| 兼容 wheel | `cu129` | 保留依赖的外部 wheel URL，不适合上传到 PyPI |
| PyPI 候选发行包 | `cu130` | Python 3.10、Torch 2.11.0、vLLM 0.26.0；GPU 驱动要求 >=580.95.05 |

驱动为 575 的验证服务器仅验证过 `cu129` 的 GPU 执行。`cu130` 候选包仍需在兼容驱动上完成 GPU 和端到端推理验收；构建成功不代表运行验证通过。

## 从源码安装

在仓库根目录创建新的 Conda 环境并运行：

```bash
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
bash scripts/install-quickstart.sh --cuda cu129
```

脚本会准备匹配的 Torch/vLLM 依赖、CUDA Toolkit、C++ 编译器和 Rust 工具链。安装主项目时会同时编译 `aegaeon.ops` 和 `quick_model_loader._rlib`，不再需要单独以 editable 方式安装 `quick_model_loader`。

| 选项 | CUDA 版本 |
|---|---|
| `--cuda auto` | 自动选择，默认值 |
| `--cuda cu129` | CUDA 12.9 |
| `--cuda cu130` | CUDA 13.0；GPU 驱动要求 >=580.95.05 |

切换 CUDA 版本时使用独立环境。这个脚本仍是源码构建流程，尽管它会下载 Torch 和 vLLM 的预编译依赖。

## 安装预编译的本地 wheel

在源码目录之外使用干净的 Python 3.10、glibc >=2.31 环境。将下面的 `AEGAEON_WHEEL` 设置为实际构建或取得的 wheel 的绝对路径和文件名；示例路径仅为占位。

```bash
AEGAEON_WHEEL=/absolute/path/to/aegaeon-0.1.0-cp310-cp310-manylinux_2_31_x86_64.whl
python -m pip install "$AEGAEON_WHEEL"
python -m pip check
aegaeon --help
aegaeon doctor
```

wheel 包含已编译的 CUDA 和 Rust 扩展，以及延迟 profile 的 JSON 数据。安装时不会编译这些扩展，无需源码仓库、NVCC、GCC 或 Rust。Pip 仍需下载声明的运行依赖。`cu129` 兼容 wheel 需要访问其依赖的外部 URL；`cu130` 候选发行包使用注册表中的依赖。

profile 按以下顺序查找：`AEGAEON_PROFILE_ROOT`、wheel 内置数据、源码目录下的 `profiles`。要使用其他模型或设备的测量结果，可设置 `AEGAEON_PROFILE_ROOT`。

## 构建 PyPI 候选 wheel

维护者需要先准备 Linux x86_64 / Python 3.10 构建环境：面向 CUDA 13.0 的 Torch 2.11.0、匹配的 vLLM 0.26.0、CUDA Toolkit 13.0、受支持的 GCC C++ 工具链、Rust/Cargo，以及 `build`、`wheel`、`setuptools`、`setuptools-rust`、`auditwheel` 和 `patchelf` 构建工具。使用 `--no-isolation` 前，应先准备好 `pyproject.toml` 中声明的构建依赖。该候选包的构建要求 CUDA 13.0 环境。在仓库根目录运行：

```bash
AEGAEON_RELEASE=1 AEGAEON_CUDA=cu130 AEGAEON_BUILD_CUDA=1 \
  python -m build --wheel --no-isolation --outdir "$HOME/aegaeon-wheelhouse/cu130"
```

显式的 `cu130` 目标固定 wheel 的依赖元数据和 ABI。发行构建会拒绝不支持的目标和直接 URL 运行依赖。构建后端通过 `auditwheel`/`patchelf` 修复扩展的动态库路径，移除 Conda 构建环境的绝对 RPATH，并且仅输出修复后的 `manylinux_2_31_x86_64` wheel。glibc 下限依据原生扩展的实际 ELF 需求确定。这一步只生成本地产物，不会上传或发布。发布前应在干净环境安装实际生成的 wheel，并完成导入、原生运算和最小推理验收。

## Foundry 图功能

CUDA Graph 默认模式为 `off`，普通运行不依赖 Foundry。`save`/`load` 图模式仍需单独安装仓库 submodule 固定的 Foundry fork，并将 `AEGAEON_FOUNDRY_HOOK_PATH` 设置为其钩子库的有效绝对路径。这些功能不属于基础 wheel 的安装承诺，详见 [CUDA Graph](cuda-graphs.md)。

## 检查环境

```bash
aegaeon doctor
aegaeon doctor --check-cuda
```

默认 doctor 检查导入，以及已安装 wheel 的 CUDA ABI 是否与 `torch.version.cuda` 匹配，无需 GPU。`--check-cuda` 还会执行 GPU 运算，因此需要兼容的 GPU 和驱动。

接下来阅读[快速开始](quickstart.md)。安装错误见[故障排查](troubleshooting.md)。
