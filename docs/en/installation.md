# Installation

Aegaeon targets Linux x86_64, Python 3.10, and an NVIDIA GPU. Source installations require glibc 2.28 or newer. The first precompiled wheel targets `manylinux_2_31_x86_64` and requires glibc 2.31 or newer. The driver must support the selected CUDA stack.

The precompiled Aegaeon distribution has not been published to PyPI. The `pip install aegaeon` release workflow is under validation; use a source installation or an explicitly provided local wheel for now.

| Installation path | CUDA | Status |
|---|---|---|
| Source installation | `cu129` or `cu130` | Builds the Aegaeon CUDA and Rust extensions locally |
| Compatibility wheel | `cu129` | Preserves the external dependency wheel URLs; not suitable for upload to PyPI |
| PyPI release candidate | `cu130` | Python 3.10, Torch 2.11.0, vLLM 0.26.0; requires GPU driver >=580.95.05 |

GPU execution on the validation server with driver 575 has been checked only with `cu129`. The `cu130` candidate still needs GPU and end-to-end inference acceptance testing on a compatible driver; a successful build does not establish runtime support.

## Install from source

Run from the repository root in a new Conda environment:

```bash
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
bash scripts/install-quickstart.sh --cuda cu129
```

The installer prepares the matching Torch/vLLM dependencies, CUDA Toolkit, C++ compiler, and Rust toolchain. Installing the main project builds both `aegaeon.ops` and `quick_model_loader._rlib`; a separate editable installation of `quick_model_loader` is no longer required.

| Option | CUDA version |
|---|---|
| `--cuda auto` | Automatic selection; default |
| `--cuda cu129` | CUDA 12.9 |
| `--cuda cu130` | CUDA 13.0; requires GPU driver >=580.95.05 |

Use a separate environment when changing CUDA versions. This script remains a source-build workflow even though it downloads precompiled Torch and vLLM dependencies.

## Install a precompiled local wheel

Use a fresh Python 3.10 environment with glibc >=2.31 outside the source checkout. Set `AEGAEON_WHEEL` below to the actual absolute path and filename of the wheel you built or received; the displayed path is a placeholder.

```bash
AEGAEON_WHEEL=/absolute/path/to/aegaeon-0.1.0-cp310-cp310-manylinux_2_31_x86_64.whl
python -m pip install "$AEGAEON_WHEEL"
python -m pip check
aegaeon --help
aegaeon doctor
```

The wheel contains the compiled CUDA and Rust extensions and bundled latency-profile JSON files. Installing it does not build those extensions and does not require the source checkout, NVCC, GCC, or Rust. Pip still downloads the declared runtime dependencies. A `cu129` compatibility wheel requires access to its external dependency URLs; the `cu130` release candidate uses registry dependencies.

Profiles are located in this order: `AEGAEON_PROFILE_ROOT`, bundled wheel profiles, then the source checkout's `profiles` directory. Set `AEGAEON_PROFILE_ROOT` to use measurements for another model/device combination.

## Build the PyPI candidate wheel

This command is for maintainers with a prepared Linux x86_64 / Python 3.10 build environment: Torch 2.11.0 for CUDA 13.0, matching vLLM 0.26.0, CUDA Toolkit 13.0, a supported GCC C++ toolchain, Rust/Cargo, and the `build`, `wheel`, `setuptools`, `setuptools-rust`, `auditwheel`, and `patchelf` build tools. Prepare the build-system requirements from `pyproject.toml` before using `--no-isolation`. A CUDA 12.9 environment cannot build this candidate. Run from the repository root:

```bash
AEGAEON_RELEASE=1 AEGAEON_CUDA=cu130 AEGAEON_BUILD_CUDA=1 \
  python -m build --wheel --no-isolation --outdir "$HOME/aegaeon-wheelhouse/cu130"
```

The explicit `cu130` target fixes the wheel's dependency metadata and ABI. The release build rejects unsupported targets and direct URL runtime dependencies. The backend repairs the extensions' dynamic-library paths with `auditwheel`/`patchelf`, removes absolute Conda build-environment RPATHs, and outputs only the repaired `manylinux_2_31_x86_64` wheel. Its glibc floor follows the native extensions' actual ELF requirements. This creates a local artifact; it does not upload or publish anything. Install the exact generated wheel in a clean environment, then validate imports, native operations, and minimal inference before publishing.

## Foundry graph support

The default CUDA Graph mode is `off`, so ordinary execution does not require Foundry. Graph `save`/`load` modes still require a separate installation of the pinned Foundry fork from the repository submodule, plus a valid absolute `AEGAEON_FOUNDRY_HOOK_PATH` to its hook library. These features are not included in the base wheel's installation promise. See [CUDA Graph](cuda-graphs.md).

## Check the environment

```bash
aegaeon doctor
aegaeon doctor --check-cuda
```

The default doctor checks imports and the installed wheel's CUDA ABI against `torch.version.cuda` without requiring a GPU. `--check-cuda` also executes GPU operations and therefore needs a compatible GPU and driver.

Continue with the [Quickstart](quickstart.md). For installation errors, see [Troubleshooting](troubleshooting.md).
