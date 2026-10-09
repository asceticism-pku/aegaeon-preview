# 常见问题与排障

## 排障顺序

先保留完整错误栈、启动命令、YAML、源码revision、依赖版本、GPU/驱动和日志。依次检查import/扩展、Ray资源、缓存预算、模型/profile、请求schema，最后定位图恢复或并行路径。

## 安装与导入

**setup时找不到torch**：setup.py顶层导入torch；按安装顺序先准备匹配torch再构建。**aegaeon.ops缺失或undefined symbol**：检查扩展是否以当前torch/CUDA构建，环境是否更换过torch，nvcc与wheel是否兼容。**quick_model_loader._rlib缺失**：检查Rust工具链与editable扩展编译结果。**vLLM内部模块导入失败**：主项目使用版本敏感内部接口，检查vllm==0.26.0而不是直接升级到任意最新版。

**Conda 报 database is locked**：libmamba 的 SQLite 分片缓存锁冲突会触发该错误。创建环境时使用 `--solver classic`；安装脚本在未设置 `CONDA_PKGS_DIRS` 时指定独立包缓存路径，并在工具链安装时使用 classic solver。Foundry 依赖安装若仍在 `shards_cache` 报锁冲突，可在该次 `conda install` 命令前加 `CONDA_PLUGINS_USE_SHARDED_REPODATA=false`，关闭分片 repodata 缓存。实际命令见[CUDA Graph 安装步骤](cuda-graphs.md#安装-foundry)。

**安装脚本提示驱动不兼容**：默认 auto 模式在安装包之前先检查 cu130 的驱动和 GPU 条件，通过时选择 CUDA 13.0，否则回退 cu129。驱动 `535.247.01` 会回退 cu129。cu129 / CUDA 12.9 路线在安装包之前要求驱动分支号至少为 525，安装后验证真实 CUDA 运算。分支检查是初步筛选；GPU、驱动具体版本及应用所用 CUDA 功能须满足 [NVIDIA CUDA 12.x 兼容要求](https://docs.nvidia.com/deploy/cuda-compatibility/minor-version-compatibility.html)。CUDA 13.0 实验安装路线采用保守前置门槛，要求 Linux 驱动 `580.95.05` 或更新版本，以匹配 Update 2 工具链，见 [CUDA 13.0 Update 2 发布说明](https://docs.nvidia.com/cuda/archive/13.0.2/cuda-toolkit-release-notes/index.html)。每张检测到的 GPU 计算能力还需至少为 `7.5`；显式 `--cuda cu130` 时，预检未通过会直接退出。该路线尚未完成项目端到端测试。自动回退发生在安装包之前；下载、安装或编译失败会终止所选路线。强制已测试路线可使用 `--cuda cu129`。PyTorch 2.11.0 是 vLLM 0.26.0 的依赖要求，请使用安装脚本提供的固定版本组合。

auto 初筛驱动和 GPU 计算能力；cu130 还需符合 [CUDA 13.0.2 的 Linux 发行版支持表](https://docs.nvidia.com/cuda/archive/13.0.2/cuda-installation-guide-linux/index.html)，例如 Ubuntu 22.04 或 24.04。Ubuntu 20.04 已从该版本支持表中移除；发行版条件不满足时在新环境中显式选择 `--cuda cu129`。脚本按所选路线安装匹配的 Toolkit，已有 Toolkit 不决定自动选择。

**CUDA Toolkit/编译器不匹配**：本地扩展仍需要与 PyTorch CUDA 构建匹配的 CUDA Toolkit。在目标环境中重新执行安装脚本，准备匹配的 CUDA、GCC/G++ 和 Rust 工具链，再重新编译扩展。

### 可选安装验证

安装脚本包含 CUDA 运算、核心导入和依赖一致性检查；排查安装失败或更换环境后，可在当前环境手动复查：

```bash
python -c "from importlib.metadata import version; import torch, vllm; print(torch.__version__, version('vllm'), torch.version.cuda); print(torch.ones(1, device='cuda'))"
python -c "import torch, quick_model_loader._rlib, aegaeon.ops; from aegaeon import LLMService, NodeConfig, Request; print('core imports OK')"
aegaeon --help
aegaeon start --help
```

已测试的 cu129 路线应输出 PyTorch `2.11.0+cu129`、vLLM `0.26.0+cu129` 和 CUDA `12.9`。未完成端到端测试的 cu130 路线应使用 PyTorch `2.11.0`（显示版本可带 CUDA 后缀）、vLLM `0.26.0` 和 CUDA `13.0`。包使用惰性导出，检查 CUDA 扩展和 loader 时请分别导入上述模块。上述检查只覆盖安装检查；推理与 CUDA Graph 验证分别见 [QuickStart](quickstart.md) 和[图存档指南](cuda-graphs.md)。

## Ray与GPU

**Ray 连接失败**：默认 auto 会连接现有集群；先启动 Ray 或显式传入 --ray-address。**缺少 node_0/node_1 资源**：启动 Ray 时添加相应自定义资源；多 GPU 单机的 nnodes 仍为 1。**Actor 一直 pending**：用 ray status 检查 CPU、GPU 和 placement group 资源，重点核对 worker_num_cpus×Worker 数。**CUDA 状态或编号异常**：检查 nvidia-smi、torch.cuda.is_available、driver 和 actor 可见性传播。

```bash
ray status
nvidia-smi
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.device_count())"
```

## 缓存与内存

**CPU free blocks 不足**：增加 slab 或降低并发与长度，并检查主机内存和共享目录。**部署返回 507**：查看 failed、节点 cache fit、碎片和活动模型保护；GPU OOM 会沿独立错误路径返回。**GPU MemoryError/OOM**：区分估计剩余 <=0 与实际加载峰值，检查 current/prefetch/临时 workspace。**启动占用大量 CPU 内存**：默认预算含 128 GiB CPU KV 和权重 cache，请显式配置适合机器的数值。

### 主机内存注册与 memlock

QuickLoader 使用共享主机内存和 CUDA host registration。出现 host registration 或 pinned memory 错误时，检查共享内存及 memlock 的 soft/hard 上限（`ulimit -l` 单位为 KiB）：

```bash
df -h /dev/shm
ulimit -Sl
ulimit -Hl
```

保留 host registration 的具体错误，同时检查主机共享内存容量、服务或容器的资源限制。根据错误栈定位所触发的限制，调整后重新启动 Ray。

## 模型与profile

**No usable latency profiles**：按错误提示目录检查 model.profile、device.profile 和 JSON 样本字段，并在目标设备重新采集。**TP assert**：当前 profile 读取固定要求 tp==1，请按单 GPU 采集。**params 计算失败**：检查权重 metadata 完整性；手填时按 2^30 换算。**持续输出特殊 token**：检查 EOS 集合、generation_config、模板和停止条件，并保存 token IDs 定位。

## 接口

**400 temperature/top_p 等**：按贪心参数表设置取值。**422 extra_forbidden**：按 openapi.json 和请求 schema 移除范围外字段。**400 model 未部署**：模型名应与服务登记一致，并完成 startup 或 deploy。**503**：检查 READY 副本和 DRAINING 状态。**429**：检查 60 秒窗口统计和环境变量；该限制按窗口内已记录 token 计算。

**流式输出延迟**：客户端使用 curl -N，并检查代理缓冲、网络和首次加载成本。**断开后 outstanding 仍非零**：后端继续运行至停止条件，清理任务等待执行完成。**修改 YAML 后配置仍为旧值**：重启服务进程以重新加载全局 registry。**CLI 引擎数与预期不同**：正常启动读取 ServerConfig，请修改 YAML。

## Foundry

已测试的 Foundry 与 CUDA Graph 路线使用 cu129 / CUDA 12.9。CUDA 13.0 的 Foundry 构建和 SAVE/LOAD 尚未完成端到端测试；下面链接的 CUDA 12.9 compat 包只适用于 CUDA 12.9 环境。切换版本时使用独立环境，重新构建扩展与 Foundry，并使用新的存档目录。

**NumPy 导入失败 / NP_SUPPORTED_MODULES**：若 NumPy 的依赖要求比系统版本更新的 `libstdc++`，加载系统旧库会导致导入失败。遇到后续 `NP_SUPPORTED_MODULES` 错误时，先查看完整 traceback 中较早的 NumPy 或 `GLIBCXX_*` 导入错误。Python 启动前将 `$CONDA_PREFIX/lib` 加入 `LD_LIBRARY_PATH`；旧驱动还需要 compat 时，保持 compat 第一、Conda `lib` 第二，然后重新启动进程：

```bash
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
# 仅在已配置 CUDA forward-compatibility 库的环境执行下一行
export LD_LIBRARY_PATH="$AEGAEON_CUDA_COMPAT_PATH:$LD_LIBRARY_PATH"
```

**需要 hook path**：AEGAEON_FOUNDRY_HOOK_PATH 填写与 `foundry.ops` 同目录 `.so` 文件的绝对路径，并在 Worker 启动前传入。**libboost_json/libboost_filesystem not found**：安装匹配的 Boost>=1.83，或将自定义 Boost 的 `lib` 目录加入 `LD_LIBRARY_PATH`。**undefined symbol: cuFuncGetName**：按 [CUDA 12.9 兼容库步骤](cuda-graphs.md#cuda-129-兼容库)下载、校验和无 root 解包匹配的 NVIDIA forward-compatibility 库，设置 `AEGAEON_CUDA_COMPAT_PATH` 并前置到 `LD_LIBRARY_PATH`，再导入 Foundry；该下载示例限定 Ubuntu 20.04 x86_64、A100、驱动 `535.247.01` 和 CUDA 12.9。**expect one maximum sequence length**：seq_len_buckets 保留一个元素。**archive incomplete**：核对 registry 中全部模型及每个 rank 的文件与图。**obsolete ABI/layout mismatch/binding mismatch**：恢复与 SAVE 相同的环境和布局，或重新录制；strict=false 只放宽文档注明的检查。**LOAD 后走 eager**：Foundry hook 位于 P/D Decode 路径；继续检查 batch、长度、graph_state 和 replay 日志。

## 提交问题的最小材料

提供可复现短请求、去除敏感路径/输入后的配置、完整错误栈、期望与实际行为、资源与依赖版本、是否单模型/TP=1/图关闭可复现。性能问题另提供真实token长度、样本数、预热和负载条件。
