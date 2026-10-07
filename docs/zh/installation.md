# 安装与环境准备

## 前提

以下步骤适用于 Linux x86_64 与 NVIDIA GPU，已验证的组合为 Python 3.10、PyTorch `2.11.0+cu129`、vLLM `0.26.0+cu129` 和 CUDA 12.9。请先获取本仓库、安装并初始化 Conda，然后在仓库根目录执行命令。

需要兼容 CUDA 12.9 的 NVIDIA 驱动，以及足够的主机内存和 `/dev/shm` 空间；QuickStart 示例使用 20 GiB 模型 cache、8 GiB CPU KV cache，还需为 Ray 和其他进程留出余量。

```bash
nvidia-smi
free -h
df -h /dev/shm
```

`nvidia-smi` 显示驱动支持的 CUDA 版本；安装脚本会另外检查编译器并执行真实 CUDA 运算。

## 安装

在新环境中执行：

```bash
export CONDA_PKGS_DIRS="$HOME/.conda/pkgs-aegaeon"
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
conda install --solver classic -c nvidia cuda-toolkit=12.9 gcc_linux-64=11.2.0 gxx_linux-64=11.2.0 -y
conda install --solver classic rust=1.97.1 -y
bash scripts/install-quickstart-cu129.sh
```

独立包缓存与 classic solver 用于规避 Conda 缓存锁问题，GCC/G++ 11.2 与 CUDA 12.9 构建要求匹配；故障处理见[故障排查](troubleshooting.md)。

安装脚本设置 `CUDA_HOME`，安装固定的 cu129 PyTorch/vLLM wheel、仓库依赖（包括 `cupy-cuda12x`），并使用 CUDA/C++ 和 Rust/Cargo 编译 `aegaeon.ops` 与 `quick_model_loader`。脚本已包含版本检查、真实 CUDA 运算、扩展和核心组件导入检查及 `pip check`，全部通过后会输出 `Aegaeon quickstart installation verified`。

**安装时使用上述脚本选择指定的 CUDA 12.9 wheel 变体；`pip install -r requirements.txt` 只处理仓库通用依赖。** 更换 PyTorch 或 vLLM 版本后，需要重新编译原生扩展并运行测试集。

## 下一步

按[QuickStart](quickstart.md)准备模型并启动服务。默认 `cuda_graph.mode: off` 不需要 Foundry；使用 `save` 或 `load` 时，按[图存档指南](cuda-graphs.md)在同一 Python 环境中额外安装 Foundry。

多节点需要相同的 Python、依赖和代码，并在各节点准备模型与 profile，详见[部署指南](deployment.md)。
