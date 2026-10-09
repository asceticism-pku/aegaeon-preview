# Installation and Environment Setup

## Prerequisites

These steps target Linux x86_64, an NVIDIA GPU, and Python 3.10. The prebuilt wheels require glibc 2.28 or newer; the selected CUDA Toolkit also has Linux distribution requirements, listed below for cu130. Obtain this repository, install and initialize Conda, then run the commands from the repository root.

The installer defaults to `--cuda auto`. Before installing packages, it checks the driver and GPUs, chooses CUDA 13.0 when the cu130 prerequisites below pass, and otherwise falls back to cu129 / CUDA 12.9. It prepares the matching toolchain for the selected route; no preinstalled CUDA Toolkit or manual Toolkit selection is required. PyTorch 2.11.0 is required by vLLM 0.26.0.

The cu129 combination of PyTorch `2.11.0+cu129`, vLLM `0.26.0+cu129`, and CUDA 12.9 has completed fresh-install and actual serving-request tests. cu130 has not completed project end-to-end testing.

Fresh installation and actual serving requests were validated on an A100 PCIe 40GB with NVIDIA driver `535.247.01`, in single-node Simple mode with TP=1, PP=1, and `cuda_graph.mode: off`.

You also need enough host memory and `/dev/shm` capacity. The QuickStart example uses a 20 GiB model cache and an 8 GiB CPU KV cache; allow additional capacity for Ray and other processes.

```bash
nvidia-smi
free -h
df -h /dev/shm
```

`nvidia-smi` shows the maximum CUDA version supported by the driver, not the installed CUDA Toolkit version. Before package installation, the cu129 route requires a driver branch number of at least 525. It verifies a real CUDA operation after installation. The branch check is an initial filter; see the [NVIDIA CUDA compatibility requirements](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html) for the full conditions. The cu130 prerequisites and test coverage are listed below.

## Install

Run the default automatic selection in a new environment:

```bash
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
bash scripts/install-quickstart.sh
```

The installer installs a matching CUDA Toolkit, GCC/G++, and Rust into the active environment. If `CONDA_PKGS_DIRS` is unset, it uses `$HOME/.conda/pkgs-aegaeon` as the package cache. Toolchain installation uses the classic solver. You do not need to enter toolchain versions manually. See [troubleshooting](troubleshooting.md) if installation fails.

The installer sets `CUDA_HOME`, installs vLLM and its required PyTorch dependency, repository dependencies and the matching CuPy package, and builds `aegaeon.ops` and `quick_model_loader` with CUDA/C++ and Rust/Cargo. A local CUDA Toolkit is still required to compile the extensions and must match the PyTorch CUDA build. The installer checks versions, runs a real CUDA operation, imports the extensions and core components, and runs `pip check`. For the cu129 route, after all checks pass it prints `Aegaeon quickstart installation verified`.

**Use the installer to select a consistent wheel and compiler combination; `pip install -r requirements.txt` only installs the repository's general dependencies.** Arbitrary PyTorch/vLLM versions are not supported. After changing either version, rebuild the native extensions and run the test suite.

## CUDA route selection

The default command is equivalent to `bash scripts/install-quickstart.sh --cuda auto`. Selection happens before installing packages. If the cu130 driver or GPU prerequisites do not pass, or GPU compute capability cannot be confirmed, the installer chooses cu129 and prints the reason. Download, installation, or compilation failures terminate the selected route and display the error.

Force the tested cu129 route:

```bash
bash scripts/install-quickstart.sh --cuda cu129
```

The auto precheck covers the driver and GPU compute capability. Check the selected Toolkit's Linux distribution support table before running the installer. An existing CUDA Toolkit does not determine the route. The script installs a matching Toolkit into the active Conda environment for the selected route. Use a separate new environment when switching routes to retain an existing installation.

## CUDA 13.0 route (not end-to-end tested)

Selecting cu130 automatically or explicitly with `--cuda cu130` uses the CUDA 13.0.2 Toolkit, PyTorch `2.11.0` (CUDA 13.0 build), and the public PyPI vLLM `0.26.0` wheel; `torch.version.cuda` reports `13.0`. This experimental route has not completed Aegaeon fresh-install, real-inference, or Foundry CUDA Graph SAVE/LOAD end-to-end testing. cu129 is the tested fallback route.

This experimental install route uses a conservative driver prerequisite: Linux NVIDIA driver `580.95.05` or newer to match the CUDA 13.0 Update 2 toolchain. See the [CUDA 13.0 Update 2 release notes](https://docs.nvidia.com/cuda/archive/13.0.2/cuda-toolkit-release-notes/index.html). Every detected GPU must have compute capability of at least `7.5`. This driver/GPU precheck selects the installation route; project end-to-end test coverage remains as stated above. Servers with driver `535.247.01` fall back to cu129 in the default auto mode. With an explicit `--cuda cu130`, failing prerequisites terminate the installer.

CUDA 13.0.2 also requires an [officially supported Linux distribution](https://docs.nvidia.com/cuda/archive/13.0.2/cuda-installation-guide-linux/index.html), such as Ubuntu 22.04 or 24.04. Ubuntu 20.04 was removed from this version's support table. Auto checks only the driver and GPUs. If the distribution requirement is not met, select `--cuda cu129` explicitly in a new environment.

Select cu130 explicitly in a separate new environment to retain a working cu129 installation:

```bash
conda create --solver classic -n aegaeon-cu130 python=3.10 -y
conda activate aegaeon-cu130
bash scripts/install-quickstart.sh --cuda cu130
```

The script checks dependency versions, CUDA execution, and core imports. Passing those checks establishes installation checks only; it does not establish project end-to-end test coverage for this route. See the [QuickStart](quickstart.md) for service configuration and requests, and the [graph archive guide](cuda-graphs.md) for CUDA Graph versions and test coverage.

## Next steps

Follow the [QuickStart](quickstart.md) to prepare a model and start serving. The default `cuda_graph.mode: off` does not require Foundry. For `save` or `load`, install Foundry into the same Python environment following the [graph archive guide](cuda-graphs.md).

For multiple nodes, install matching Python versions, dependencies, and code, and prepare models and profiles on each node. See the [deployment guide](deployment.md).
