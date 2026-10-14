# 安装与环境准备

## 前提

以下步骤面向 Linux x86_64、NVIDIA GPU 与标准 CPython 3.10。预编译 wheel 要求 glibc 2.28 或更新版本；所选 CUDA Toolkit 的发行版支持要求还需分别满足，cu130 见下节。请先获取本仓库、安装并初始化 Conda，然后在仓库根目录执行命令。

安装脚本默认使用 `--cuda auto`：安装包之前先检查驱动和 GPU，满足下节的 cu130 预检条件时选择 CUDA 13.0，否则回退 cu129 / CUDA 12.9。脚本按所选路线准备匹配的工具链，无需预先安装或手动选择 CUDA Toolkit。PyTorch 2.11.0 是 vLLM 0.26.0 的依赖要求。

cu129 的 PyTorch `2.11.0+cu129`、vLLM `0.26.0+cu129` 和 CUDA 12.9 组合已完成从零安装与实际服务请求测试；cu130 尚未完成项目端到端测试。

从零安装和实际服务请求的验收环境为 A100 PCIe 40GB、NVIDIA 驱动 `535.247.01`、单节点 Simple 模式、TP=1、PP=1 和 `cuda_graph.mode: off`。

### Python 版本支持

Aegaeon 的安装和运行仅支持启用 GIL 的标准 **CPython 3.10**。
创建环境时使用 `python=3.10`；其他 Python minor 版本、PyPy 和
free-threaded CPython 不在支持范围内。安装器与 PEP 517 源码后端使用
CPython 3.10（`cp310`）的 Torch 系列 wheel。CUDA 路线仍按驱动／GPU
检查优先 CUDA 13，并在条件不满足时选择 cu129。

原生扩展应使用 CPython 3.10 和匹配的 Torch/CUDA 构建，Foundry 图存档
保留其匹配运行环境。此前版本曾在 Python 3.11–3.14 上进行实验测试，
包括独立的 Python 3.14/cu129 原生检查；这些历史实验不扩展当前支持范围。

### 安装版本与验收范围

下表区分安装脚本选择的版本与已有服务测试范围；它不表示所有通过预检的 GPU、驱动和发行版都完成了端到端验收。

| 组件 | cu129 安装方案 | cu130 安装方案 | 准备方与验收范围 |
|---|---|---|---|
| 平台 | Linux x86_64、glibc >=2.28，并满足 Toolkit 的发行版要求 | 相同基础检查，另需满足 CUDA 13.0.2 的发行版要求 | 用户准备操作系统。脚本检查系统、架构和 glibc，不检查发行版支持表。 |
| Conda 与 Python | 已激活的 Conda 环境、CPython 3.10 | 相同 | 用户先安装并初始化 Conda，再创建环境。 |
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


## pip 源码安装

在完整 Aegaeon 源码仓库的根目录执行。以下两个命令都进入同一个 PEP 517 构建后端，
使用与安装脚本一致的 CUDA 13 优先策略：

```bash
python -m pip install --no-cache-dir .
python -m pip install --no-cache-dir -r requirements.txt
```

任选其中一个命令。`requirements.txt` 现在引用本地项目 `.`，会构建并安装 Aegaeon，
而非只安装依赖列表。即使 requirements 文件名使用绝对路径，也应先进入仓库根目录。
仅安装通用 Python 依赖时，使用：

```bash
python -m pip install -r requirements-common.txt
```

源码构建后端先检查所有检测到的 GPU 和驱动，再请求 Torch/vLLM：满足同一组驱动/GPU
条件时优先 cu130，否则选择 cu129；驱动低于受支持门槛时，在安装后端依赖前报错。
隔离的原生构建环境也使用所选 Torch。cu129 固定配套的 Torch、vLLM、torchaudio、
torchvision、torchcodec wheel 和 CuPy 12；cu130 请求精确的公共版本及 CuPy 13。
生成元数据时不导入 Torch，也不创建 CUDA 扩展；实际编译时核对 `CUDA_HOME`/nvcc
和 `torch.version.cuda` 与所选路线是否一致。

pip 准备 Python 构建依赖；匹配的 CUDA Toolkit 与 C++ 编译器仍需提前准备。
快速开始安装器会自动准备这些工具。serving 还需要单独编译的 Rust
`quick_model_loader`；上述 pip 入口安装主项目及所选后端，完整 quickstart 则安装两个扩展。
运行完整 quickstart 时使用包含 `quick_model_loader/` 的完整源码仓库。

通过 `AEGAEON_CUDA=cu129` 或 `cu130` 显式指定源码路线，显式路线仍检查驱动/GPU。
已有匹配环境可以关闭构建隔离：

```bash
AEGAEON_CUDA=cu129 python -m pip install --no-cache-dir --no-build-isolation .
```

切换 CUDA 路线时使用新环境，并保留一个匹配的 CuPy 分发包。
源码后端在请求后端包前会拒绝已存在的 `cupy` 源码分发包或另一条 CUDA 路线的 CuPy。
它独立于 pip 构建隔离读取调用解释器环境，覆盖默认解释器安装；
单独的 `--target` 或 `--prefix` 目标目录不会传给构建后端。
仅构建 Python/资源包时，可关闭 GPU 探测：

```bash
AEGAEON_CUDA=core AEGAEON_BUILD_CUDA=0 python -m pip wheel --no-deps .
```

core 包没有 CUDA 运行依赖或 `aegaeon.ops`；GPU serving 需要完整运行环境。
单独设置 `AEGAEON_BUILD_CUDA=0` 只省略扩展，仍保留所选 GPU 后端依赖。

已构建 wheel 的 CUDA 路线固定，由 `Requires-Dist` 依赖元数据记录。
源码归档和 wheel 都保持包版本 `0.1.0`，保证 pip 解析源码包时版本一致。
安装 wheel 时不再执行源码选择，应选择与目标运行环境匹配的产物；
pip 的 wheel 兼容性规则仍独立于 GPU 硬件。
从源码归档或固定 Git commit 安装时使用 `--no-cache-dir`，避免复用此前按其他 CUDA
路线构建的 wheel。源码归档包含构建选择器，独立的 quick-loader 子项目仍需另行获取。
回退发生在安装前的路线选择阶段；下载或编译失败会终止所选路线。

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

先检查本节点的运行环境，再检查实际用于启动服务的配置：

```bash
aegaeon doctor
CUDA_VISIBLE_DEVICES=0 aegaeon doctor --config /srv/aegaeon/config.yaml --check-cuda
```

| 检查 | 检查内容 |
|---|---|
| 运行时导入 | PyTorch、vLLM、Ray、PyYAML、CuPy、FastAPI、Uvicorn、`aegaeon.ops` 和 Rust loader，以及固定的 PyTorch/vLLM 版本 |
| 本地工具链 | `CUDA_HOME`、`nvcc`、Toolkit 与 `torch.version.cuda` 的版本匹配、C++ 编译器；Rust/Cargo 按编译前提报告 |
| 主机内存 | 共享内存和主机内存可用空间、可读取的 cgroup v2 限额、临时文件写入和 mmap 探针，以及 memlock 软限额和硬限额 |
| 带 `--config` | YAML 拓扑与 TP=1 约束、启动模型选择、本地模型配置/tokenizer/safetensors 文件及索引分片可读性、声明的模型/设备 profile 候选目录及可用的 DecodeEstimator 训练样本（`decode_latencies` 至少含三个有限正数，单位秒） |
| 配置内存预算 | CPU KV slab 分配量加显式或估算的模型 cache，与本节点共享内存和主机可用内存比较；诊断标明估算值及尚未确定的预算 |
| 带 `--check-cuda` | 在已分配的 GPU 上执行微型 PyTorch、CuPy CUDA 运算，导入 vLLM 原生扩展，并对一页共享内存执行 CUDA 注册和注销 |

`--models` 与 `--tensor-parallel-size` 的覆盖规则和 `start` 一致。
配置检查读取本地文件，全程无需下载模型或启动 Ray。
普通 `doctor` 省略 GPU tensor 运算；每台 worker 节点应在已分配的 GPU 上执行
`--check-cuda`。`start` 在导入 API 前执行静态配置和本节点主机预检，
driver 进程省略 CUDA 查询。声明的设备仅作为 profile 候选；候选数据缺失或
格式错误显示 `WARN`，静态 `OK` 也仍标明实际 worker GPU 尚未检查。
`doctor --config PATH --check-cuda` 读取已分配的实际 GPU，并将引擎所需
GPU 的 profile 缺失或格式错误作为 `FAIL`。
共享内存检查优先使用 `AEGAEON_SHM_DIR`，默认使用
`/dev/shm`；检查自定义节点目录时，将该变量设置为 worker 的 `shm_dir`。
微型临时映射会在探针结束后清理。已有固定名称 cache 文件会检查类型和访问权限；
其存在会产生警告，文件内容和大小保持原样。容量检查同时估算本挂载上
既有 cache 文件可复用的实际分配空间。空闲容量仅在复用 cache 后满足预算时，
结果显示 `WARN` 并要求确认复用；最佳复用仍不足时显示 `FAIL`。分配元数据
读取失败时，复用状态显示 `WARN`。空闲空间少于一页且复用状态尚未确定时，
mmap/CUDA 注册探针会跳过并显示 `WARN`，检查状态保持为未验证。

每项结果显示为 `OK`、`WARN` 或 `FAIL`。失败以状态码 1 退出，并阻止
`start`，包括模型文件或内存容量等确定的失败。警告会说明静态 profile
候选尚未确认、memlock 限额低于预算、有限额但预算未知、扩展已安装
但缺少 Rust/Cargo、预算尚未确定等情况；普通启动继续执行。`aegaeon doctor --strict` 会将
警告也视为状态码 1。传入 `--config` 才会启用模型、profile 和内存预算检查。

预检范围是执行命令的本节点。远端 Ray worker 状态、完整配置规模的锁页内存分配、
真实模型执行仍属于单独的验收步骤。预检通过表示上表所列检查通过；随后完成
[QuickStart](quickstart.md) 请求测试，并按配置规模核对
[主机内存注册与 memlock 指南](troubleshooting.md#主机内存注册与-memlock)。

wheel 包含 loader Python 子包和内置网页。每台 worker 通过 `AEGAEON_PROFILE_ROOT` 提供实测 profile，根目录下按 `<model-profile>/<device-profile>/i<input>b<batch>.json` 组织；批量 profiling 也使用此根目录。

需要省略可选的 `aegaeon.ops` 扩展时，先准备匹配的运行时依赖，再在仓库根目录构建 Python wheel：

```bash
AEGAEON_BUILD_CUDA=0 python -m pip wheel --no-build-isolation --no-deps .
```

默认构建仍包含 `aegaeon.ops`。serving 需要 Rust loader 和匹配的 GPU 依赖；执行 vLLM/FlashInfer JIT 编译时需要匹配的 Toolkit。

## 下一步

按[QuickStart](quickstart.md)准备模型并启动服务。默认 `cuda_graph.mode: off` 不需要 Foundry；使用 `save` 或 `load` 时，按[图存档指南](cuda-graphs.md)在同一 Python 环境中额外安装 Foundry。

多节点需要相同的 Python、依赖和代码，并在各节点准备模型与 profile，详见[部署指南](deployment.md)。
