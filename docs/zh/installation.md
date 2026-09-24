# 安装与环境准备

## 目标环境

Aegaeon 面向 Linux 与 NVIDIA CUDA 环境，推荐使用 Ubuntu。Python 版本要求为 3.10 或更高，当前依赖固定为 `torch==2.11.0` 和 `vllm==0.26.0`。安装前请确认对应 wheel、CUDA Toolkit 与 NVIDIA 驱动兼容。

主项目编译 `aegaeon.ops` CUDA 扩展，因此需要 CUDA Toolkit（包含 nvcc）、兼容的 NVIDIA 驱动、C++ 编译器，以及可导入的 PyTorch。`quick_model_loader` 还需要 Rust/Cargo，并依赖 `cupy-cuda12x`；请使用与 CUDA 12 wheel 匹配的运行环境。

## 先检查

以下为 Linux Shell 命令，在仓库根目录执行：

```bash
python --version
nvidia-smi
nvcc --version
g++ --version
df -h /dev/shm
free -h
ulimit -Sl
ulimit -Hl
```

记录 GPU 产品名、可用显存、驱动、CUDA Toolkit 和 Python 版本。`nvidia-smi` 给出驱动支持的 CUDA 上限，`nvcc --version` 给出实际安装的 Toolkit 版本，两项都应保留。

QuickLoader 会将模型 cache 和 CPU KV cache 注册为 pinned host memory。Linux 中 `ulimit -l` 的单位是 KiB。请在启动 Ray 前配置主机、服务或容器，使 Ray 进程继承的 soft memlock 上限能够覆盖两块 cache，hard 上限也必须允许该数值；专用服务建议设为 `unlimited`。若 hard 上限过低，需要管理员先调整系统配置。

## 创建独立环境

```bash
conda create -n aegaeon python=3.10 -y
conda activate aegaeon
python -m pip install --upgrade pip setuptools wheel
```

也可使用 venv。创建环境时请按 torch 2.11.0 和 vLLM 0.26.0 的发行元数据选择 Python 版本。更换 vLLM 或 PyTorch 版本属于依赖升级，需要重新编译原生扩展并运行测试集。

## 安装固定依赖与扩展

主项目的 setup.py 在读取 metadata 时导入 torch，因此安装顺序是先安装 torch，再执行 editable 构建：

```bash
# 根据目标 CUDA 选择这些固定版本实际可用的官方 wheel 源。
python -m pip install 'torch==2.11.0' 'vllm==0.26.0'
python -m pip install -r requirements.txt

# 当前源码直接 import yaml，显式安装 PyYAML，避免依赖偶然的传递安装。
python -m pip install PyYAML

# 已准备匹配的 torch 后，关闭构建隔离，避免隔离环境使用另一版本 torch。
python -m pip install --no-build-isolation -e .
```

遇到固定版本依赖冲突或目标 CUDA 缺少对应 wheel 时，请按各依赖的发行要求选择匹配组合。

Rust 工具链安装方式可按团队环境管理规范选择。安装后：

```bash
rustc --version
cargo --version
python -m pip install -e ./quick_model_loader
python -m pip check
```

## 验证核心组件

```bash
python -c "import torch, vllm; print(torch.__version__, vllm.__version__); print(torch.version.cuda, torch.cuda.is_available())"
python -c "import torch; import quick_model_loader; import aegaeon.ops; from aegaeon import LLMService, NodeConfig, Request; print('core imports OK')"
aegaeon --help
aegaeon start --help
```

包使用惰性导出，因此 `import aegaeon` 只检查顶层包；请按上面的命令分别检查 CUDA 扩展、Ray、vLLM 和 loader。

## CUDA Graph 模式下安装 Foundry

默认 `cuda_graph.mode: off` 不需要 Foundry。配置为 `save` 或 `load` 时，必须在同一个 Python 环境中额外安装仓库内的 Foundry 子项目；主项目的 editable 安装不会包含它：

```bash
conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
python -m pip install --no-build-isolation -e ./foundry
python -c "import foundry, foundry.ops; print(foundry.__file__); print(foundry.ops.__file__)"
```

第一条适用于 Conda 环境；已有满足要求的系统或本地 Boost 时可省略。Foundry 要求 CMake>=4.0、Boost>=1.83.0 的 filesystem/json 组件和动态链接 hook。自定义 Boost 前缀需要在构建期加入 `CMAKE_PREFIX_PATH`、运行期加入 `LD_LIBRARY_PATH`。构建时应保持主项目已固定的 PyTorch/vLLM 组合；Foundry 独立安装示例中的其他依赖组合只适用于对应的独立环境。hook、Boost/CUDA 动态库定位及 SAVE/LOAD 验收见[图存档指南](cuda-graphs.md)。多节点时每台 Decode Worker 节点都必须分别完成这项本地安装。

## 部署可复现性

```bash
mkdir -p logs
python -m pip freeze > logs/environment.txt
git rev-parse HEAD > logs/source-revision.txt
```

多节点需要相同的 Python、包版本和项目代码。editable 安装引用各节点的本地源码路径；源码、模型和 profile 请通过共享存储或部署流程分发到每台物理机器。
