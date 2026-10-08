# 环境变量参考

以下列出主源码中读取、传播或生成的主要变量。表格分别标明变量的实际消费位置；内部设置随实现一同更新。变量主要在模块导入、服务创建或 actor 初始化时读取，修改后重新启动相应进程以保持设置一致。

| 变量 | 默认 / 来源 | 说明 |
|---|---|---|
| `AEGAEON_LOG_FILE` | output.log | logger 导入时读取，LLMService 显式转发给 Controller |
| `AEGAEON_TTFT_SLO` | 10 | 秒；utils导入时读取，未在LLMService显式actor白名单中 |
| `AEGAEON_TPOT_SLO` | 0.1 | 秒；utils导入时读取，未在LLMService显式actor白名单中 |
| `AEGAEON_DEFAULT_MODEL_REPO` | modelscope | modelscope或huggingface |
| `AEGAEON_SHM_DIR` | /dev/shm | 共享缓存目录，需容量与隔离 |
| `AEGAEON_RAY_ADDRESS` | 下一层RAY_ADDRESS/auto | Ray连接地址；CLI优先 |
| `RAY_ADDRESS` | auto | Ray连接fallback |
| `AEGAEON_RAY_TEMP_DIR` | 省略 | 创建 local Ray 时的临时目录 |
| `AEGAEON_ARENA_FRACTION` | 0.875 | QuickLoader GPU arena 比例；create_service 调用 ray.init 时通过 runtime_env 转发，Ray 已初始化时跳过此步骤；大小还受 GPU 容量减 8 GiB 的上限约束 |
| `AEGAEON_MAX_REQUESTS_PER_SECOND` | 省略 | API 每模型 60 秒窗口访问上限 |
| `AEGAEON_MAX_TOKENS_PER_SECOND` | 省略 | 已记录 token 窗口平均速率上限；正常完成结果进入统计，断流统计边界见日志与监控 |
| `AEGAEON_GPU_FREQ_CONTROL` | false | 频率控制组件开关；当前模型YAML链路未完整接线 |
| `AEGAEON_GPU_CLOCK_COMMAND` | 省略 | 频率控制 wrapper 命令 |
| `AEGAEON_NVIDIA_SMI_COMMAND` | 省略 | 采样/控制命令覆盖 |
| `AEGAEON_CUDA_COMPAT_PATH` | 探测默认路径 | CUDA兼容库目录，必须存在 |
| `AEGAEON_FOUNDRY_HOOK_PATH` | 省略 | save/load Decode Worker 所需的已存在绝对 `.so` 文件路径 |
| `AEGAEON_DUMMY_WEIGHTS` | 省略 | `1` 启用测试占位权重；质量与性能测试使用真实权重 |
| `AEGAEON_CUDA_VISIBLE_DEVICES` | 省略 | Controller/Worker 设备可见集合 |
| `CUDA_VISIBLE_DEVICES` | 进程环境 | driver值会转换为actor自定义可见配置 |
| `AEGAEON_CONFIG_PATH` | 内部设置 | 服务配置路径，主要内部传递 |
| `AEGAEON_CONFIG_CONTENT` | 内部设置 | 完整YAML内容，主要内部传递 |
| `AEGAEON_SHAREGPT_PATH` | /root/ShareGPT_V3_unfiltered_cleaned_split.json | workload 工具的数据路径；API 启动路径省略此依赖 |
| `FAST_SWITCH` | LLMService 设置 1 | 内部快速切换控制，仅供实现内部使用 |
| `SERVERLESSLLM_URL` | http://127.0.0.1:8343 | 外部演示代理地址 |
| `AEGAEON_MODEL_ROOT` | 省略 | actor 会传播该值；主下载函数当前省略此入口 |
| `AEGAEON_MODEL_ROOTS` | 省略 | actor 会传播该值；主下载函数当前省略此入口 |
| `AEGAEON_MODEL_DOWNLOAD_PROXY` | 省略 | 用于传播及 profile 脚本拼接；主下载函数当前省略此入口 |
| `LD_LIBRARY_PATH` | 环境 | 库搜索目录，compat路径可前置 |
| `LD_PRELOAD` | 内部构造 | Foundry hook在Worker启动前设置 |
| `PYTHONPATH` | 环境 | 源码/模块查找路径，actor传播 |
| `FOUNDRY_MODE` | 内部设置 | 由cuda_graph.mode生成 |
| `FOUNDRY_EARLY_RESERVE_BASE` | 内部设置 | 由base_address生成 |
| `FOUNDRY_EARLY_RESERVE_SIZE` | 内部设置 | 由region_size_bytes生成 |
| `TORCH_CUBLASLT_UNIFIED_WORKSPACE` | 图路径设置1 | Foundry统一workspace |
| `NCCL_ASYNC_ERROR_HANDLING` | Worker 移除 | Worker.initialize 中移除 |
| `HTTP_PROXY` | 环境 | 标准代理环境，传播到actor |
| `HTTPS_PROXY` | 环境 | 同上 |
| `http_proxy` | 环境 | 同上 |
| `https_proxy` | 环境 | 同上 |

Foundry 使用自定义 Boost 前缀时，`LD_LIBRARY_PATH` 还要包含该前缀的 `lib` 目录；构建阶段则通过 `CMAKE_PREFIX_PATH` 提供 Boost CMake 配置。需要 CUDA forward-compatibility 库的环境，在直接导入 Foundry 前将 compat 目录前置到 `LD_LIBRARY_PATH`。Worker 启动会前置显式的 `AEGAEON_CUDA_COMPAT_PATH`，或探测 `/usr/local/cuda-<torch.version.cuda>/compat` 和 `~/.local/cuda-<torch.version.cuda>/compat` 中首个已存在的目录。

SLO 跨进程一致性、可见设备、频率链路和下载代理范围分别见配置、部署、功耗与模型章节。RAY_DEDUP_LOGS 等 Ray 专用变量用于上游诊断，与 Aegaeon 性能配置分开管理。
