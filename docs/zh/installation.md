# 安装与环境准备

## 前提

以下步骤面向 Linux x86_64、NVIDIA GPU 与 Python 3.10。预编译 wheel 要求 glibc 2.28 或更新版本；所选 CUDA Toolkit 的发行版支持要求还需分别满足，cu130 见下节。请先获取本仓库、安装并初始化 Conda，然后在仓库根目录执行命令。

安装脚本默认使用 `--cuda auto`：安装包之前先检查驱动和 GPU，满足下节的 cu130 预检条件时选择 CUDA 13.0，否则回退 cu129 / CUDA 12.9。脚本按所选路线准备匹配的工具链，无需预先安装或手动选择 CUDA Toolkit。PyTorch 2.11.0 是 vLLM 0.26.0 的依赖要求。

cu129 的 PyTorch `2.11.0+cu129`、vLLM `0.26.0+cu129` 和 CUDA 12.9 组合已完成从零安装与实际服务请求测试；cu130 尚未完成项目端到端测试。

从零安装和实际服务请求的验收环境为 A100 PCIe 40GB、NVIDIA 驱动 `535.247.01`、单节点 Simple 模式、TP=1、PP=1 和 `cuda_graph.mode: off`。

### 安装版本与验收范围

下表区分安装脚本选择的版本与已有服务测试范围；它不表示所有通过预检的 GPU、驱动和发行版都完成了端到端验收。

| 组件 | cu129 安装方案 | cu130 安装方案 | 准备方与验收范围 |
|---|---|---|---|
| 平台 | Linux x86_64、glibc >=2.28，并满足 Toolkit 的发行版要求 | 相同基础检查，另需满足 CUDA 13.0.2 的发行版要求 | 用户准备操作系统。脚本检查系统、架构和 glibc，不检查发行版支持表。 |
| Conda 与 Python | 已激活的 Conda 环境、Python 3.10 | 相同 | 用户先安装并初始化 Conda，再创建环境。 |
| NVIDIA 驱动 | 脚本初筛：分支 >=525；服务实测版本为 `535.247.01` | 脚本要求 >=`580.95.05` | 用户或管理员准备驱动。cu129 的分支号检查不保证所有 CUDA 功能兼容。 |
| GPU 与服务验收 | A100 PCIe 40GB；单节点 Simple、TP=1、PP=1、graphs off | 每张检测到的 GPU 计算能力 >=7.5；服务未测试 | 用户准备 GPU。cu130 通过驱动/GPU 路线选择检查，不代表已验证服务支持。 |
| CUDA Toolkit | Conda `cuda-toolkit=12.9` | Conda `cuda-toolkit=13.0.2` | 主安装脚本安装并设置 `CUDA_HOME`。 |
| C/C++ 编译器 | Conda GCC/G++ `11.2.0` | 相同的固定版本方案 | 主脚本安装；cu129 辅助脚本允许 GCC/G++ >=9 且 <14。 |
| Rust/Cargo | Conda `rust=1.97.1` | 相同的固定版本方案 | 主脚本安装工具链。该版本固定不等于 cu130 已完成端到端验收。 |
| 后端 wheel | 显式 wheel URL 的 PyTorch `2.11.0+cu129`、vLLM `0.26.0+cu129` | PyTorch `2.11.0` CUDA 13.0 构建、PyPI 公共 vLLM `0.26.0` | 安装脚本安装并检查所选后端。 |
| CuPy | `cupy-cuda12x` | `cupy-cuda13x` | 脚本选择匹配的 quick-loader extra，并检查 CUDA 互操作。 |
| 原生扩展 | 编译并导入 `aegaeon.ops`、`quick_model_loader._rlib` | 相同的本地编译/导入检查 | 脚本本地编译；cu130 的从零安装和服务仍未测试。 |
| Foundry | 另行安装；cu129 测试范围见图存档指南 | 未完成 Foundry 端到端验证 | quickstart 脚本不安装；见 [CUDA Graph](cuda-graphs.md)。 |

还需要足够的主机内存和 `/dev/shm` 空间；QuickStart 示例使用 20 GiB 模型 cache、8 GiB CPU KV cache，还需为 Ray 和其他进程留出余量。

```bash
nvidia-smi
free -h
df -h /dev/shm
```

`nvidia-smi` 显示驱动支持的最高 CUDA 版本；它不表示本机已安装的 CUDA Toolkit 版本。cu129 路线在安装包之前要求驱动分支号至少为 525，安装后再执行真实 CUDA 运算。驱动分支检查只是初步筛选，完整兼容条件见 [NVIDIA CUDA 兼容说明](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html)。cu130 路线的预检条件和测试范围见下节。

## 安装

在新环境中执行默认自动选择：

```bash
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
bash scripts/install-quickstart.sh
```

安装脚本会在当前 Conda 环境中自动准备匹配的 CUDA Toolkit、GCC/G++ 和 Rust，无需手动输入工具链版本。未设置 `CONDA_PKGS_DIRS` 时，脚本使用 `$HOME/.conda/pkgs-aegaeon` 作为包缓存，并使用 classic solver 安装工具链；故障处理见[故障排查](troubleshooting.md)。

请使用 `scripts/install-quickstart.sh` 作为入口。`scripts/install-quickstart-cu129.sh` 是它调用的底层辅助脚本；直接调用辅助脚本时，必须已经准备 CUDA 12.9 `nvcc`、匹配的 `CUDA_HOME`、GCC/G++ 和 Rust/Cargo。主安装脚本会提供这些工具；两个脚本都不安装 Conda，也不更新 NVIDIA 驱动。

只检查初始前提并打印计划命令、不安装软件时，执行：

```bash
bash scripts/install-quickstart.sh --cuda cu129 --dry-run
```

dry-run 成功只表示当前环境与路线选择的初检通过，不证明后续下载、编译、CUDA 运算或模型服务一定成功。

安装脚本设置 `CUDA_HOME`，安装 vLLM 及其要求的 PyTorch、仓库依赖和匹配的 CuPy 包，并使用 CUDA/C++ 和 Rust/Cargo 编译 `aegaeon.ops` 与 `quick_model_loader`。本地 CUDA Toolkit 仍用于编译扩展，必须与 PyTorch 的 CUDA 构建匹配。脚本包含版本检查、真实 CUDA 运算、扩展和核心组件导入检查及 `pip check`，cu129 路线全部通过后会输出 `Aegaeon quickstart installation verified`。

**后端安装应通过安装脚本选择完整组合，`requirements.txt` 本身没有指定 CUDA 路线。** 该文件除通用依赖外，还包含 `torch==2.11.0` 和 `vllm==0.26.0`，但没有选择 CUDA wheel 变体，也不会安装 Toolkit。公共源在空环境中按这些版本解析出的 CUDA 13 依赖独立于主机驱动的运行能力；该结果与已测试的 cu129 路线不同。安装脚本先选择完整的后端 wheel 组合，再安装该文件。请在新环境中使用安装脚本，并保留它选择的组合。不支持任意 PyTorch/vLLM 版本。更换任一版本后，需要重新编译原生扩展并运行测试集。

## CUDA 路线选择

默认命令等价于 `bash scripts/install-quickstart.sh --cuda auto`。自动选择在安装包之前完成：cu130 的驱动或 GPU 预检条件未通过、或 GPU 计算能力无法确认时，选择 cu129，并打印原因。下载、安装或编译失败时，脚本终止当前路线并显示错误。

强制使用已测试的 cu129 路线：

```bash
bash scripts/install-quickstart.sh --cuda cu129
```

auto 的预检范围是驱动和 GPU 计算能力；运行前还需核对所选 Toolkit 的 Linux 发行版支持表。已有 CUDA Toolkit 不参与路线选择；脚本按所选路线在当前 Conda 环境中安装匹配的 Toolkit。切换路线时使用新的独立环境，保留已有安装。

## CUDA 13.0 路线（未测试）

自动选择 cu130 或显式指定 `--cuda cu130` 时，使用 CUDA 13.0.2 Toolkit、PyTorch `2.11.0`（CUDA 13.0 构建）与 vLLM `0.26.0` 的 PyPI 公共 wheel；`torch.version.cuda` 显示 `13.0`。此路线尚未完成 Aegaeon 从零安装、真实推理或 Foundry CUDA Graph SAVE/LOAD 的端到端测试；cu129 是已测试的回退路线。

此实验安装路线采用保守的驱动前置门槛：Linux NVIDIA 驱动 `580.95.05` 或更新版本，以匹配 CUDA 13.0 Update 2 工具链，见 [CUDA 13.0 Update 2 发布说明](https://docs.nvidia.com/cuda/archive/13.0.2/cuda-toolkit-release-notes/index.html)。每张检测到的 GPU 计算能力还需至少为 `7.5`；该驱动/GPU 预检只用于选择安装路线，完整项目端到端测试范围仍按上文标注。驱动 `535.247.01` 的服务器在默认 auto 模式下回退 cu129。显式 `--cuda cu130` 时，预检条件未通过会直接退出。

CUDA 13.0.2 还要求使用其[官方支持的 Linux 发行版](https://docs.nvidia.com/cuda/archive/13.0.2/cuda-installation-guide-linux/index.html)，例如 Ubuntu 22.04 或 24.04；Ubuntu 20.04 已从该版本支持表中移除。auto 只初筛驱动/GPU；发行版条件不满足时，在新环境中显式选择 `--cuda cu129`。

在独立的新环境中显式选择 cu130，保留已工作的 cu129 环境：

```bash
conda create --solver classic -n aegaeon-cu130 python=3.10 -y
conda activate aegaeon-cu130
bash scripts/install-quickstart.sh --cuda cu130
```

脚本检查依赖版本、CUDA 运算和核心导入；这些检查通过只表示安装检查通过，不代表此路线已经完成项目端到端测试。服务配置和请求步骤见 [QuickStart](quickstart.md)，CUDA Graph 的版本与测试范围见[图存档指南](cuda-graphs.md)。

## 运行检查与 wheel 安装

启动服务前，检查依赖导入和固定的运行时版本：

```bash
aegaeon doctor
CUDA_VISIBLE_DEVICES=0 aegaeon doctor --check-cuda
```

`doctor` 导入 PyTorch、vLLM、Ray、PyYAML、CuPy、FastAPI、Uvicorn 和 Rust loader，并检查 PyTorch/vLLM 的公开版本号，不执行自身的 GPU tensor 检查。`--check-cuda` 还会执行微型 PyTorch、CuPy 运算并导入 `vllm._C_stable_libtorch`；任何检查失败都会返回非零状态。每台 worker 节点应在已分配的 GPU 上执行 CUDA 检查。`start` 在导入 API 前先执行依赖检查。

这些诊断不是完整的部署预检：不检查 `nvcc`/`CUDA_HOME`、编译器或 Rust/Cargo 可执行文件、`aegaeon.ops`、普通 `doctor` 所用 CUDA wheel 变体、Ray 集群资源、共享内存容量、memlock 限额、模型路径或 profile 路径。安装脚本另行检查自身的构建工具链和扩展导入。启动服务前仍需完成 [QuickStart](quickstart.md) 的模型/profile 与内存检查，以及[主机内存注册与 memlock 检查](troubleshooting.md#主机内存注册与-memlock)。`doctor` 通过不表示具体模型配置一定能启动或完成请求。

wheel 包含 loader Python 子包和内置网页。每台 worker 通过 `AEGAEON_PROFILE_ROOT` 提供实测 profile，根目录下按 `<model-profile>/<device-profile>/i<input>b<batch>.json` 组织；批量 profiling 也使用此根目录。

需要省略可选的 `aegaeon.ops` 扩展时，先准备匹配的运行时依赖，再在仓库根目录构建 Python wheel：

```bash
AEGAEON_BUILD_CUDA=0 python -m pip wheel --no-build-isolation --no-deps .
```

默认构建仍包含 `aegaeon.ops`。serving 需要 Rust loader 和匹配的 GPU 依赖；执行 vLLM/FlashInfer JIT 编译时需要匹配的 Toolkit。

## 下一步

按[QuickStart](quickstart.md)准备模型并启动服务。默认 `cuda_graph.mode: off` 不需要 Foundry；使用 `save` 或 `load` 时，按[图存档指南](cuda-graphs.md)在同一 Python 环境中额外安装 Foundry。

多节点需要相同的 Python、依赖和代码，并在各节点准备模型与 profile，详见[部署指南](deployment.md)。
