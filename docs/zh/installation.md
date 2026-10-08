# 安装与环境准备

## 前提

以下步骤适用于 Linux x86_64（glibc 2.28 或更新版本）、NVIDIA GPU 与 Python 3.10。请先获取本仓库、安装并初始化 Conda，然后在仓库根目录执行命令。

安装脚本固定使用已验收的 PyTorch `2.11.0+cu129`、vLLM `0.26.0+cu129` 和 CUDA 12.9 组合，自动准备匹配的工具链，无需分别选择或安装这些版本。PyTorch 2.11.0 是 vLLM 0.26.0 的依赖要求。

从零安装和实际服务请求的验收环境为 A100 PCIe 40GB、NVIDIA 驱动 `535.247.01`、单节点 Simple 模式、TP=1、PP=1 和 `cuda_graph.mode: off`。

还需要足够的主机内存和 `/dev/shm` 空间；QuickStart 示例使用 20 GiB 模型 cache、8 GiB CPU KV cache，还需为 Ray 和其他进程留出余量。

```bash
nvidia-smi
free -h
df -h /dev/shm
```

`nvidia-smi` 显示驱动支持的 CUDA 版本；安装脚本会在安装包之前检查驱动兼容性，准备匹配的编译器并执行真实 CUDA 运算。

## 安装

在新环境中执行：

```bash
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
bash scripts/install-quickstart.sh
```

安装脚本会在当前 Conda 环境中自动准备匹配的 CUDA Toolkit、GCC/G++ 和 Rust，并使用独立包缓存与 classic solver 规避测试环境出现过的 Conda 缓存锁问题，无需手动输入工具链版本；故障处理见[故障排查](troubleshooting.md)。

安装脚本设置 `CUDA_HOME`，安装 vLLM 及其要求的 PyTorch、仓库依赖和匹配的 CuPy 包，并使用 CUDA/C++ 和 Rust/Cargo 编译 `aegaeon.ops` 与 `quick_model_loader`。本地 CUDA Toolkit 仍用于编译扩展，必须与 PyTorch 的 CUDA 构建匹配。脚本包含版本检查、真实 CUDA 运算、扩展和核心组件导入检查及 `pip check`，全部通过后会输出 `Aegaeon quickstart installation verified`。

**使用安装脚本选择一致的 wheel 与编译器组合；`pip install -r requirements.txt` 只处理仓库通用依赖。** 不支持任意 PyTorch/vLLM 版本。更换任一版本后，需要重新编译原生扩展并运行测试集。

## 下一步

按[QuickStart](quickstart.md)准备模型并启动服务。默认 `cuda_graph.mode: off` 不需要 Foundry；使用 `save` 或 `load` 时，按[图存档指南](cuda-graphs.md)在同一 Python 环境中额外安装 Foundry。

多节点需要相同的 Python、依赖和代码，并在各节点准备模型与 profile，详见[部署指南](deployment.md)。
