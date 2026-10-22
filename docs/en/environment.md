# Environment Variables

Set variables before starting the service. For existing Ray clusters, also configure the relevant node environments before starting Ray.

| Variable | Default | Purpose |
|---|---|---|
| `AEGAEON_WEB_DIR` | Unset | Optional external UI asset directory |
| `AEGAEON_LOG_FILE` | `output.log` | Service log path |
| `AEGAEON_TTFT_SLO` | `10` | TTFT target in seconds |
| `AEGAEON_TPOT_SLO` | `0.1` | TPOT target in seconds |
| `AEGAEON_DEFAULT_MODEL_REPO` | `modelscope` | Model source: modelscope or huggingface |
| `AEGAEON_PROFILE_ROOT` | Repository `profiles/` | Model/device performance profiles |
| `AEGAEON_SHM_DIR` | `/dev/shm` | Shared-memory cache directory |
| `AEGAEON_RAY_ADDRESS` | `RAY_ADDRESS` / `auto` | Ray cluster address; CLI overrides it |
| `AEGAEON_RAY_TEMP_DIR` | Unset | Local Ray temporary directory |
| `AEGAEON_ARENA_FRACTION` | `0.875` | QuickLoader GPU arena fraction |
| `CUDA_VISIBLE_DEVICES` | Process environment | Visible GPU set |
| `AEGAEON_MAX_REQUESTS_PER_SECOND` | Unset | Per-model request limit over a 60-second window |
| `AEGAEON_MAX_TOKENS_PER_SECOND` | Unset | Per-model recorded-token rate limit |
| `AEGAEON_FOUNDRY_HOOK_PATH` | Unset | Absolute path to `libcuda_hook.so` |
| `AEGAEON_CUDA_COMPAT_PATH` | Probed paths | CUDA compatibility library directory |
| `LD_LIBRARY_PATH` | Process environment | Dynamic-library search path |
| `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` | Process environment | Download/network proxy configuration |

Graph settings belong in [YAML configuration](configuration.md); startup commands are in the [CLI reference](cli.md).
