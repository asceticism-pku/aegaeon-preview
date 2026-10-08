# Installation and Environment Setup

## Prerequisites

These steps target Linux x86_64 with glibc 2.28 or newer, an NVIDIA GPU, and Python 3.10. Obtain this repository, install and initialize Conda, then run the commands from the repository root.

The installer uses the validated combination of PyTorch `2.11.0+cu129`, vLLM `0.26.0+cu129`, and CUDA 12.9. It prepares the matching toolchain automatically, so you do not need to choose or install these versions separately. PyTorch 2.11.0 is required by vLLM 0.26.0.

Fresh installation and actual serving requests were validated on an A100 PCIe 40GB with NVIDIA driver `535.247.01`, in single-node Simple mode with TP=1, PP=1, and `cuda_graph.mode: off`.

You also need enough host memory and `/dev/shm` capacity. The QuickStart example uses a 20 GiB model cache and an 8 GiB CPU KV cache; allow additional capacity for Ray and other processes.

```bash
nvidia-smi
free -h
df -h /dev/shm
```

`nvidia-smi` shows the CUDA version supported by the driver. The installer checks driver compatibility before installing packages, prepares a matching compiler, and performs a real CUDA operation.

## Install

Run in a new environment:

```bash
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
bash scripts/install-quickstart.sh
```

The installer prepares an independent Conda package cache and installs a matching CUDA Toolkit, GCC/G++, and Rust into the active environment. It uses the classic solver to avoid the Conda cache-lock errors seen in the tested environment. You do not need to enter toolchain versions manually. See [troubleshooting](troubleshooting.md) if installation fails.

The installer sets `CUDA_HOME`, installs vLLM and its required PyTorch dependency, repository dependencies and the matching CuPy package, and builds `aegaeon.ops` and `quick_model_loader` with CUDA/C++ and Rust/Cargo. A local CUDA Toolkit is still required to compile the extensions and must match the PyTorch CUDA build. The installer checks versions, runs a real CUDA operation, imports the extensions and core components, and runs `pip check`. After all checks pass, it prints `Aegaeon quickstart installation verified`.

**Use the installer to select a consistent wheel and compiler combination; `pip install -r requirements.txt` only installs the repository's general dependencies.** Arbitrary PyTorch/vLLM versions are not supported. After changing either version, rebuild the native extensions and run the test suite.

## Next steps

Follow the [QuickStart](quickstart.md) to prepare a model and start serving. The default `cuda_graph.mode: off` does not require Foundry. For `save` or `load`, install Foundry into the same Python environment following the [graph archive guide](cuda-graphs.md).

For multiple nodes, install matching Python versions, dependencies, and code, and prepare models and profiles on each node. See the [deployment guide](deployment.md).
