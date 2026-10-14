# Installation and Environment Setup

## Prerequisites

These steps target Linux x86_64, an NVIDIA GPU, and standard CPython 3.10. The prebuilt wheels require glibc 2.28 or newer; the selected CUDA Toolkit also has Linux distribution requirements, listed below for cu130. Obtain this repository, install and initialize Conda, then run the commands from the repository root.

The installer defaults to `--cuda auto`. Before installing packages, it checks the driver and GPUs, chooses CUDA 13.0 when the cu130 prerequisites below pass, and otherwise falls back to cu129 / CUDA 12.9. It prepares the matching toolchain for the selected route; no preinstalled CUDA Toolkit or manual Toolkit selection is required. PyTorch 2.11.0 is required by vLLM 0.26.0.

The cu129 combination of PyTorch `2.11.0+cu129`, vLLM `0.26.0+cu129`, and CUDA 12.9 has completed fresh-install and actual serving-request tests. cu130 has not completed project end-to-end testing.

Fresh installation and actual serving requests were validated on an A100 PCIe 40GB with NVIDIA driver `535.247.01`, in single-node Simple mode with TP=1, PP=1, and `cuda_graph.mode: off`.

### Python version support

Aegaeon installation and runtime support is restricted to standard,
GIL-enabled **CPython 3.10**. Create environments with `python=3.10`;
other Python minor versions, PyPy, and free-threaded CPython are unsupported.
The installer and PEP 517 source backend use the CPython 3.10 (`cp310`)
Torch-family wheels. CUDA route selection still prefers CUDA 13 and falls
back to cu129 using driver/GPU checks.

Build native extensions with CPython 3.10 and matching Torch/CUDA, and keep
Foundry graph archives in their matching runtime. Earlier revisions were
tested on Python 3.11–3.14, including an isolated Python 3.14/cu129 native
check; those historical experiments do not extend the current support policy.

### Installer recipe and validation scope

The table separates the versions selected by the installer from the reported serving test coverage. It is not an end-to-end certification of every GPU, driver, or distribution that passes the prerequisite checks.

| Component | cu129 recipe | cu130 recipe | Who prepares it / validation scope |
|---|---|---|---|
| Platform | Linux x86_64, glibc >=2.28; satisfy the Toolkit's distribution requirements | Same base checks; CUDA 13.0.2 distribution requirements also apply | User supplies the OS. The script checks OS, architecture, and glibc, not distribution support. |
| Conda and Python | Activated Conda environment, CPython 3.10 | Same | User installs/initializes Conda and creates the environment before running the script. |
| NVIDIA driver | Script's initial filter: branch >=525; serving tested on `535.247.01` | Script requires >=`580.95.05` | User or administrator installs the driver. The branch-only cu129 filter is not a guarantee for every CUDA feature. |
| GPU and serving coverage | A100 PCIe 40GB; single-node Simple, TP=1, PP=1, graphs off | Every detected GPU must have compute capability >=7.5; serving untested | User supplies GPUs. Passing the cu130 driver/GPU selection check does not establish serving support. |
| CUDA Toolkit | Conda `cuda-toolkit=12.9` | Conda `cuda-toolkit=13.0.2` | Main installer installs it and sets `CUDA_HOME`. |
| C/C++ compiler | Conda GCC/G++ `11.2.0` | Same pinned recipe | Main installer installs it; the cu129 helper accepts GCC/G++ >=9 and <14. |
| Rust/Cargo | Conda `rust=1.97.1` | Same pinned recipe | Main installer installs the toolchain. This is a recipe pin, not evidence of cu130 end-to-end testing. |
| Backend wheels | PyTorch `2.11.0+cu129`, vLLM `0.26.0+cu129` from explicit wheel URLs | PyTorch `2.11.0` CUDA 13.0 build, public PyPI vLLM `0.26.0` | Installer installs and verifies the selected backend. |
| CuPy | `cupy-cuda12x` | `cupy-cuda13x` | Installer selects the matching quick-loader extra and checks CUDA interoperability. |
| Native extensions | Builds/imports `aegaeon.ops` and `quick_model_loader._rlib` | Same local build/import checks | Installer builds them locally; cu130 fresh-install and serving remain untested. |
| Foundry | Separate installation and cu129 tests described in the graph guide | No Foundry end-to-end validation | Not installed by the quickstart script; see [CUDA Graphs](cuda-graphs.md). |

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

Use `scripts/install-quickstart.sh` as the entry point. `scripts/install-quickstart-cu129.sh` is its lower-level helper: calling it directly requires CUDA 12.9 `nvcc`, a matching `CUDA_HOME`, GCC/G++, and Rust/Cargo to be prepared already. The main installer provides these; neither script installs Conda or updates the NVIDIA driver.

To check the initial prerequisites and print the planned commands without installing packages:

```bash
bash scripts/install-quickstart.sh --cuda cu129 --dry-run
```

A successful dry-run checks the current environment and route selection only. It does not prove package downloads, compilation, CUDA execution, or model serving will succeed.

The installer sets `CUDA_HOME`, installs vLLM and its required PyTorch dependency, repository dependencies and the matching CuPy package, and builds `aegaeon.ops` and `quick_model_loader` with CUDA/C++ and Rust/Cargo. A local CUDA Toolkit is still required to compile the extensions and must match the PyTorch CUDA build. The installer checks versions, runs a real CUDA operation, imports the extensions and core components, and runs `pip check`. For the cu129 route, after all checks pass it prints `Aegaeon quickstart installation verified`.


## pip source installation

Run from the root of a complete Aegaeon source checkout. Both commands enter
the same PEP 517 backend and use the installer's CUDA 13-first policy:

```bash
python -m pip install --no-cache-dir .
python -m pip install --no-cache-dir -r requirements.txt
```

Use either command. `requirements.txt` now references the local project (`.`),
so it builds and installs Aegaeon instead of only installing a dependency list.
Run it from the repository root, including when the requirements filename is
absolute. For common Python dependencies alone, use:

```bash
python -m pip install -r requirements-common.txt
```

The source backend checks every detected GPU and driver before requesting
Torch/vLLM. It prefers cu130 when the same driver/GPU checks pass, otherwise
selects cu129. Unsupported drivers fail before backend dependency installation.
The selected Torch also supplies the isolated native build environment; cu129
pins matching Torch, vLLM, torchaudio, torchvision, torchcodec wheels and CuPy
12, while cu130 requests exact public versions and CuPy 13. Metadata creation
does not import Torch or create a CUDA extension. Native compilation validates
both `CUDA_HOME`/nvcc and `torch.version.cuda` against the selected route.

pip prepares Python build dependencies, but the CUDA Toolkit and C++ compiler
must already be available and match the selection. The quickstart installer
prepares that toolchain automatically. Serving also needs the separately built
Rust `quick_model_loader`; the pip source entry above installs the main project
and selected backend, while the complete quickstart installs both extensions.
Use the complete repository for quickstart, including `quick_model_loader/`.

Override the source selection with `AEGAEON_CUDA=cu129` or `cu130`. Explicit
routes still check driver/GPU compatibility. A matching prepared environment
can build without isolation:

```bash
AEGAEON_CUDA=cu129 python -m pip install --no-cache-dir --no-build-isolation .
```

Use a fresh environment when changing CUDA routes; retain one matching CuPy
distribution. Before requesting backend packages, the source build rejects an
existing `cupy` source distribution or a CuPy CUDA distribution for the other
route. It reads the invoking interpreter environment independently of pip
build isolation. This check covers the default interpreter installation;
separate `--target` or `--prefix` directories are not visible to the backend. A Python/resource-only package can be built without GPU probing:

```bash
AEGAEON_CUDA=core AEGAEON_BUILD_CUDA=0 python -m pip wheel --no-deps .
```

This core package has no CUDA runtime dependencies or `aegaeon.ops`; GPU serving
requires the complete runtime. `AEGAEON_BUILD_CUDA=0` alone omits the extension
but retains the selected GPU backend dependencies.

Built wheels have a fixed backend recorded in their `Requires-Dist` dependency
metadata. The package version stays `0.1.0` for both source archives and wheels
so pip can resolve source packages consistently. Installing a wheel does not
rerun source selection; choose an artifact matching the target runtime. pip's
wheel compatibility rules are independent of GPU hardware. Use `--no-cache-dir` for source archive or immutable Git
installs to prevent reuse of a wheel built earlier for another CUDA route.
Source archives contain the build selector but not the separate quick-loader
subproject. Selection fallback happens before installation; a download or
compile failure terminates the selected route.

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

## Runtime checks and wheel installations

Check the local runtime, then check the configuration that you will use to start the service:

```bash
aegaeon doctor
CUDA_VISIBLE_DEVICES=0 aegaeon doctor --config /srv/aegaeon/config.yaml --check-cuda
```

| Check | What it verifies |
|---|---|
| Runtime imports | PyTorch, vLLM, Ray, PyYAML, CuPy, FastAPI, Uvicorn, `aegaeon.ops`, and the Rust loader; pinned PyTorch/vLLM versions |
| Local toolchain | `CUDA_HOME`, `nvcc`, the Toolkit version against `torch.version.cuda`, and a C++ compiler; Rust/Cargo availability is reported as a build prerequisite |
| Host memory | Available shared-memory and host memory, inspectable cgroup v2 limits, a temporary file write/mmap probe, and soft/hard memlock limits |
| With `--config` | YAML topology and TP=1 constraints, startup model selection, readable local model configuration/tokenizer/safetensors files and indexed shards, declared model/device profile candidates and usable DecodeEstimator training samples (`decode_latencies`, at least three positive finite values in seconds) |
| Configured memory budget | CPU KV slab allocations plus the explicit or estimated model cache, compared with the local shared-memory and available host-memory capacity; diagnostics identify estimates and unresolved budgets |
| With `--check-cuda` | Small PyTorch and CuPy CUDA operations, the vLLM native-extension import, and one-page shared-memory CUDA registration/unregistration on an allocated GPU |

`--models` and `--tensor-parallel-size` use the same overrides as `start`.
The configuration check reads local files without downloading models or starting Ray.
Plain `doctor` omits GPU tensor operations; pass `--check-cuda` on an allocated GPU
on every worker node. `start` runs static configuration and local host checks
before importing the API, without querying CUDA in the driver process. Declared
devices are profile candidates: missing or invalid candidate data produces `WARN`,
and a static `OK` still leaves the actual worker GPU unverified.
`doctor --config PATH --check-cuda` resolves the allocated GPUs and treats
missing or invalid profiles for the GPUs required by the engines as `FAIL`.
The shared-memory checks use `AEGAEON_SHM_DIR` when set, otherwise `/dev/shm`;
set this to the worker's `shm_dir` when checking a custom node directory.
The small temporary mappings are removed after the probe. Existing fixed-name
cache files are checked for file type and access permissions; their presence
produces a warning, and their contents and sizes are preserved.
Capacity checks also estimate the allocated bytes of reusable cache files on
the same mount. When free capacity meets the budget only after potential cache
reuse, the result is `WARN` pending confirmation of reuse; if even this best-case
reuse leaves insufficient capacity, the result is `FAIL`. Unreadable allocation
metadata leaves reuse unverified and produces `WARN`. When less than one page
is free and reuse remains unresolved, mmap/CUDA-registration probes are skipped
with `WARN` rather than claiming success.

Each result is `OK`, `WARN`, or `FAIL`. A failure exits with status 1 and blocks
`start`, including definite model-file and memory-capacity failures. Warnings
explain limits such as an unverified static profile candidate, a memlock allowance below the budget, a
finite allowance with an unknown budget, missing Rust/Cargo after the extensions
are installed, or an unresolved memory budget;
they allow normal startup. `aegaeon doctor --strict` also exits with status 1 for
warnings. Supplying `--config` enables the model, profile, and memory-budget checks.

This is a preflight of the node running the command. Remote Ray worker state,
full-size pinned-memory allocations, and real model execution are separate
acceptance steps. A passing result establishes the checks listed above; complete
the [QuickStart](quickstart.md) request test and review the
[host-registration and memlock guidance](troubleshooting.md#host-registration-and-memlock)
for the configured memory sizes.

Wheels include the loader Python subpackage and built-in web pages. Supply measured profiles on every worker through `AEGAEON_PROFILE_ROOT`, using `<model-profile>/<device-profile>/i<input>b<batch>.json` under that directory; batch profiling also uses this root.

For a Python wheel that omits the optional `aegaeon.ops` extension, build from the repository root after preparing the matching runtime dependencies:

```bash
AEGAEON_BUILD_CUDA=0 python -m pip wheel --no-build-isolation --no-deps .
```

The default build includes `aegaeon.ops`. Serving requires the Rust loader and matching GPU dependencies; vLLM/FlashInfer JIT compilation requires the matching Toolkit.

## Next steps

Follow the [QuickStart](quickstart.md) to prepare a model and start serving. The default `cuda_graph.mode: off` does not require Foundry. For `save` or `load`, install Foundry into the same Python environment following the [graph archive guide](cuda-graphs.md).

For multiple nodes, install matching Python versions, dependencies, and code, and prepare models and profiles on each node. See the [deployment guide](deployment.md).
