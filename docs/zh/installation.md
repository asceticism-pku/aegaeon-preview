# 安装

Aegaeon 运行于 Linux x86_64，需要 NVIDIA GPU、Python 3.10 和 glibc 2.28 或更新版本。驱动需兼容所选的 CUDA 版本。

## 安装步骤

在新的 Conda 环境中，从仓库根目录执行：

```bash
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
bash scripts/install-quickstart.sh
```

安装脚本在当前环境中准备依赖，并编译 CUDA 和 Rust 扩展。

| 选项 | CUDA 版本 |
|---|---|
| `--cuda auto` | 自动选择，默认选项 |
| `--cuda cu129` | CUDA 12.9 |
| `--cuda cu130` | CUDA 13.0 |


## 环境检查

```bash
aegaeon doctor
aegaeon doctor --check-cuda
```

接下来阅读[快速开始](quickstart.md)。安装错误见[故障排查](troubleshooting.md)。
