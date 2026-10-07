# Installation and Environment Setup

## Prerequisites

These steps target Linux x86_64 with an NVIDIA GPU. The tested combination is Python 3.10, PyTorch `2.11.0+cu129`, vLLM `0.26.0+cu129`, and CUDA 12.9. Obtain this repository, install and initialize Conda, then run the commands from the repository root.

You need an NVIDIA driver compatible with CUDA 12.9 and enough host memory and `/dev/shm` capacity. The QuickStart example uses a 20 GiB model cache and an 8 GiB CPU KV cache; allow additional capacity for Ray and other processes.

```bash
nvidia-smi
free -h
df -h /dev/shm
```

`nvidia-smi` shows the CUDA version supported by the driver; the installer separately checks the compiler and performs a real CUDA operation.

## Install

Run in a new environment:

```bash
export CONDA_PKGS_DIRS="$HOME/.conda/pkgs-aegaeon"
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
conda install --solver classic -c nvidia cuda-toolkit=12.9 gcc_linux-64=11.2.0 gxx_linux-64=11.2.0 -y
conda install --solver classic rust=1.97.1 -y
bash scripts/install-quickstart-cu129.sh
```

The private package cache and classic solver avoid Conda cache-lock errors. GCC/G++ 11.2 matches the CUDA 12.9 build requirements. See [troubleshooting](troubleshooting.md) if installation fails.

The installer sets `CUDA_HOME`, installs the pinned cu129 PyTorch/vLLM wheels and repository dependencies (including `cupy-cuda12x`), and builds `aegaeon.ops` and `quick_model_loader` with CUDA/C++ and Rust/Cargo. It already checks versions, runs a real CUDA operation, imports the extensions and core components, and runs `pip check`. After all checks pass, it prints `Aegaeon quickstart installation verified`.

**Use the installer above to select the required CUDA 12.9 wheel variants; `pip install -r requirements.txt` only installs the repository's general dependencies.** After changing PyTorch or vLLM versions, rebuild the native extensions and run the test suite.

## Next steps

Follow the [QuickStart](quickstart.md) to prepare a model and start serving. The default `cuda_graph.mode: off` does not require Foundry. For `save` or `load`, install Foundry into the same Python environment following the [graph archive guide](cuda-graphs.md).

For multiple nodes, install matching Python versions, dependencies, and code, and prepare models and profiles on each node. See the [deployment guide](deployment.md).
