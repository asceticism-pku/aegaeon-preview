# Installation and Environment Setup

## Target environment

Aegaeon targets Linux with NVIDIA CUDA; Ubuntu is recommended. The package requires Python 3.10 or later and currently pins `torch==2.11.0` and `vllm==0.26.0`. Check compatibility between the corresponding wheels, CUDA Toolkit, and NVIDIA driver before installation.

The main project compiles the `aegaeon.ops` CUDA extension, requiring a CUDA Toolkit with nvcc, a compatible NVIDIA driver, a C++ compiler, and an importable PyTorch installation. `quick_model_loader` additionally requires Rust/Cargo and depends on `cupy-cuda12x`; use a runtime that matches the CUDA 12 wheel.

## Check prerequisites

Run these Linux shell commands from the repository root:

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

Record the GPU model, available memory, driver, CUDA Toolkit, and Python version. The CUDA version shown by `nvidia-smi` describes driver support; it does not replace `nvcc --version`.

QuickLoader registers the model cache and CPU KV cache as pinned host memory. On Linux, `ulimit -l` is reported in KiB. Configure the host, service, or container so the Ray processes inherit a soft memlock limit large enough for both caches; the hard limit must allow that value. `unlimited` is recommended for a dedicated serving process. A low hard limit requires an administrator change before Ray starts.

## Create an isolated environment

```bash
conda create -n aegaeon python=3.10 -y
conda activate aegaeon
python -m pip install --upgrade pip setuptools wheel
```

A venv also works. Select a Python version from the published metadata for torch 2.11.0 and vLLM 0.26.0. Treat a vLLM or PyTorch change as a dependency upgrade: rebuild native extensions and run the test suite.

## Install pinned dependencies and extensions

The main setup.py imports torch while reading package metadata, so install the matching torch before the editable build:

```bash
# Select official wheel sources available for these pinned versions and target CUDA.
python -m pip install 'torch==2.11.0' 'vllm==0.26.0'
python -m pip install -r requirements.txt

# Install PyYAML explicitly for the source code import.
python -m pip install PyYAML

# Use the matching torch already installed, without build isolation.
python -m pip install --no-build-isolation -e .
```

If the pinned versions conflict or have no suitable wheel for the target CUDA environment, check release requirements and select a compatible environment.

Install the Rust toolchain using your environment's usual tooling, then run:

```bash
rustc --version
cargo --version
python -m pip install -e ./quick_model_loader
python -m pip check
```

## Verify core components

```bash
python -c "import torch, vllm; print(torch.__version__, vllm.__version__); print(torch.version.cuda, torch.cuda.is_available())"
python -c "import torch; import quick_model_loader; import aegaeon.ops; from aegaeon import LLMService, NodeConfig, Request; print('core imports OK')"
aegaeon --help
aegaeon start --help
```

`import aegaeon` alone does not verify the CUDA extension, Ray, vLLM, or loader because the package uses lazy exports.

## Install Foundry for CUDA Graph modes

The default `cuda_graph.mode: off` does not require Foundry. With `save` or `load`, install the repository's Foundry subproject into the same Python environment; the main editable install does not include it:

```bash
conda install -c conda-forge 'boost-cpp>=1.83' 'boost>=1.83' 'cmake>=4.0' ninja
python -m pip install --no-build-isolation -e ./foundry
python -c "import foundry, foundry.ops; print(foundry.__file__); print(foundry.ops.__file__)"
```

The first command is for Conda environments and can be omitted when a compatible system or local Boost is already available. Foundry requires CMake>=4.0, Boost>=1.83.0 filesystem/json components, and the dynamic-linker hook. A custom Boost prefix belongs in `CMAKE_PREFIX_PATH` during the build and `LD_LIBRARY_PATH` at runtime. Keep the main project's pinned PyTorch/vLLM combination rather than replacing it with a standalone Foundry example. See the [graph archive guide](cuda-graphs.md) for hook, Boost/CUDA shared-library discovery, environment variables, and SAVE/LOAD validation. Install it locally on every node that runs a Decode Worker.

## Reproducible deployments

```bash
mkdir -p logs
python -m pip freeze > logs/environment.txt
git rev-parse HEAD > logs/source-revision.txt
```

All nodes need matching Python versions, dependencies, and project code. Editable installs depend on local source paths. Ray does not automatically distribute all source code, models, or profiles to other physical machines.
