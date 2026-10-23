# Installation

Aegaeon runs on Linux x86_64 with an NVIDIA GPU, Python 3.10, and glibc 2.28 or newer. Use a driver compatible with the selected CUDA version.

## Install

Run from the repository root in a new Conda environment:

```bash
conda create --solver classic -n aegaeon python=3.10 -y
conda activate aegaeon
bash scripts/install-quickstart.sh
```

The installer prepares dependencies and builds the CUDA and Rust extensions in the active environment.

| Option | CUDA version |
|---|---|
| `--cuda auto` | Automatic selection; default |
| `--cuda cu129` | CUDA 12.9 |
| `--cuda cu130` | CUDA 13.0 |


## Check the environment

```bash
aegaeon doctor
aegaeon doctor --check-cuda
```

Continue with the [Quickstart](quickstart.md). For installation errors, see [Troubleshooting](troubleshooting.md).
