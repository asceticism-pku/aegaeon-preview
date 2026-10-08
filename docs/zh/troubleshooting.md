# 常见问题与排障

## 排障顺序

先保留完整错误栈、启动命令、YAML、源码revision、依赖版本、GPU/驱动和日志。依次检查import/扩展、Ray资源、缓存预算、模型/profile、请求schema，最后定位图恢复或并行路径。

## 安装与导入

**setup时找不到torch**：setup.py顶层导入torch；按安装顺序先准备匹配torch再构建。**aegaeon.ops缺失或undefined symbol**：检查扩展是否以当前torch/CUDA构建，环境是否更换过torch，nvcc与wheel是否兼容。**quick_model_loader._rlib缺失**：检查Rust工具链与editable扩展编译结果。**vLLM内部模块导入失败**：主项目使用版本敏感内部接口，检查vllm==0.26.0而不是直接升级到任意最新版。

**Conda 报 database is locked**：测试环境的 libmamba SQLite 分片缓存曾出现该错误，独立包缓存也未完全避免。创建环境时使用 `--solver classic`；安装脚本已自动准备独立包缓存，并在工具链安装时使用 classic solver。

**默认安装提示驱动过旧**：PyPI 的 CUDA 13.0 构建要求驱动 580 或更新版本。可升级驱动，或在驱动与 GPU 满足 CUDA 12.x 兼容要求时执行 `bash scripts/install-quickstart.sh --cuda cu129`；脚本在安装包之前完成该检查。PyTorch 2.11.0 是 vLLM 0.26.0 的依赖要求，选择 cu129 只改变 CUDA 构建变体，不改变支持的 PyTorch/vLLM 版本组合。

**CUDA Toolkit/编译器不匹配**：本地扩展仍需要与 PyTorch CUDA 构建匹配的 CUDA Toolkit。在目标环境中重新执行安装脚本，准备匹配的 CUDA、GCC/G++ 和 Rust 工具链，再重新编译扩展。

### 可选安装验证

安装脚本已检查 CUDA 运算、核心导入和依赖一致性；排查安装失败或更换环境后，可在当前环境手动复查：

```bash
python -c "import torch, vllm; print(torch.__version__, vllm.__version__, torch.version.cuda); print(torch.ones(1, device='cuda'))"
python -c "import torch, quick_model_loader._rlib, aegaeon.ops; from aegaeon import LLMService, NodeConfig, Request; print('core imports OK')"
aegaeon --help
aegaeon start --help
```

默认安装使用 PyPI 的 vLLM `0.26.0` / PyTorch `2.11.0` CUDA 13.0 构建；`--cuda cu129` 使用相同版本的 CUDA 12.9 变体。请按所选路线核对输出版本。包使用惰性导出，检查 CUDA 扩展和 loader 时请分别导入上述模块。

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

一次 A100 验收在 memlock hard 上限为 64 MiB 时通过；其他环境的限制需求以实际启动结果为准。必要时调整服务或容器限制后重新启动 Ray。

## 模型与profile

**No usable latency profiles**：按错误提示目录检查 model.profile、device.profile 和 JSON 样本字段，并在目标设备重新采集。**TP assert**：当前 profile 读取固定要求 tp==1，请按单 GPU 采集。**params 计算失败**：检查权重 metadata 完整性；手填时按 2^30 换算。**持续输出特殊 token**：检查 EOS 集合、generation_config、模板和停止条件，并保存 token IDs 定位。

## 接口

**400 temperature/top_p 等**：按贪心参数表设置取值。**422 extra_forbidden**：按 openapi.json 和请求 schema 移除范围外字段。**400 model 未部署**：模型名应与服务登记一致，并完成 startup 或 deploy。**503**：检查 READY 副本和 DRAINING 状态。**429**：检查 60 秒窗口统计和环境变量；该限制按窗口内已记录 token 计算。

**流式输出延迟**：客户端使用 curl -N，并检查代理缓冲、网络和首次加载成本。**断开后 outstanding 仍非零**：后端继续运行至停止条件，清理任务等待执行完成。**修改 YAML 后配置仍为旧值**：重启服务进程以重新加载全局 registry。**CLI 引擎数与预期不同**：正常启动读取 ServerConfig，请修改 YAML。

## Foundry

**需要 hook path**：AEGAEON_FOUNDRY_HOOK_PATH 填写与 `foundry.ops` 同目录 `.so` 文件的绝对路径，并在 Worker 启动前传入。**libboost_json/libboost_filesystem not found**：安装匹配的 Boost>=1.83，或将自定义 Boost 的 `lib` 目录加入 `LD_LIBRARY_PATH`。**undefined symbol: cuFuncGetName**：使用与 CUDA wheel 匹配的 forward-compatibility 库，并设置 `AEGAEON_CUDA_COMPAT_PATH`；直接导入验证时还要同步加入 `LD_LIBRARY_PATH`。**expect one maximum sequence length**：seq_len_buckets 保留一个元素。**archive incomplete**：核对 registry 中全部模型及每个 rank 的文件与图。**obsolete ABI/layout mismatch/binding mismatch**：恢复与 SAVE 相同的环境和布局，或重新录制；strict=false 只放宽文档注明的检查。**LOAD 后走 eager**：Foundry hook 位于 P/D Decode 路径；继续检查 batch、长度、graph_state 和 replay 日志。

## 提交问题的最小材料

提供可复现短请求、去除敏感路径/输入后的配置、完整错误栈、期望与实际行为、资源与依赖版本、是否单模型/TP=1/图关闭可复现。性能问题另提供真实token长度、样本数、预热和负载条件。
