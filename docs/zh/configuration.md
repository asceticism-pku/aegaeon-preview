# 配置参考

## YAML 结构与生效规则

配置文件包含 `server`、`models`、`devices` 三个主要段。`aegaeon start --config` 必填；请使用绝对模型路径并显式填写关键预算。每个 Ray 节点都应预先准备相同的模型文件和性能数据。

`create_service()` 按字段使用以下优先级：

| 设置 | 最终规则 |
|---|---|
| host / port | 由 CLI 直接传给 uvicorn；server YAML 省略这两个字段 |
| Ray 地址 | `--ray-address` → `AEGAEON_RAY_ADDRESS` → `RAY_ADDRESS` → `auto` |
| 启动模型集合 | `--models` → `server.startup_models` → 配置中全部模型 |
| tensor_parallel_size | 必须为 1；非零 CLI 值优先；未设置或为 0 时读取 YAML / ServerConfig |
| 引擎数、CPU slab、权重缓存大小 | 正常路径取 ServerConfig；CLI 对应参数主要用于 fallback |
| SLO | utils 模块导入时从环境变量读取；server YAML 省略相关字段 |

YAML 省略 server 字段时，`getattr` 会读取 ServerConfig 默认值。引擎拓扑请直接写入 YAML；`--num-engines` 等同名 CLI 参数只用于 fallback 路径。

`startup_models` 省略或设为 null 时启动全部配置模型；`[]` 启动空服务，之后显式 deploy；非空列表只选择指定模型。CLI `--models ''` 可以表达空集合。未知启动模型会报错，重复 CLI 名称会去重；YAML 列表要求填写唯一的非空字符串。

## ServerConfig 默认值

以下是配置类默认值。生产环境应按主机内存、模型数量和并发量重新计算预算。

| 字段 | 默认值 | 含义 / 注意事项 |
|---|---:|---|
| cpu_num_slabs | 128 | 每节点 CPU KV slab 数量 |
| cpu_slab_size_bytes | 1073741824 | 每 slab 1 GiB，单位 bytes |
| memory_utilization | 0.775 | 显存容量估算比例；CUDA allocator 另行管理实际分配 |
| worker_num_cpus | 16 | 每 Worker Ray CPU 资源请求 |
| nnodes | 1 | 物理 Ray 节点数 |
| num_prefill_engines | 1 | 每节点 Prefill Engine 数 |
| num_decode_engines | 1 | 每节点 Decode Engine 数 |
| num_engines | 0 | 每节点 Simple Engine 数；与 P/D 引擎互斥 |
| tensor_parallel_size | 1 | 每 Engine 一个 Worker；当前只支持值 1 |
| model_cache_size_gb | 0 | CPU 权重缓存 GiB；0 进入自动估算路径 |
| dummy_weights | false | 测试占位权重；仅用于控制路径测试 |
| decode_work_stealing | false | 节点内 Decode 批次转移 |
| decode_work_stealing_min_batch_gap | 2 | donor/receiver 批次数差阈值，至少 2 |
| model_placement_policy | least-loaded | 或 round-robin |
| request_routing_policy | least-outstanding | 或 round-robin |
| startup_models | null | null=全部；[]=空服务 |
| cuda_graph | CudaGraphConfig() | 默认关闭；详见下表 |

默认 CPU KV 为 128 GiB/节点，此外还有 CPU 权重缓存、权重文件映射与进程内存。内存少于这些预算的机器应显式调小 slab 和缓存配置。

P/D 推理配置至少包含一个 Prefill Engine 和一个 Decode Engine。Simple 模式则设置 `num_engines>0`，并将两种 P/D Engine 数量设为 0。

## models 条目

| 字段 | 默认 | 说明 |
|---|---|---|
| name | 必填 | 对外模型名字；profile 文件夹可使用独立名称 |
| path | 空字符串 | 本地权重目录；省略时 `ModelSpec.path()` 触发下载 |
| params | null | **输入数字乘以 2^30 后作为参数量** |
| profile | 空字符串 | 显式性能目录别名；否则由模型名字最后一段规范化 |
| max_model_len | null | 上下文总长度 override；仍需符合模型能力 |
| id | 省略 | 所有模型都省略时按顺序生成；显式设置时为每个模型都填写 ID |
| tool_parser | null | 非空 vLLM tool parser 名称；为模型别名显式选择输出函数调用解析器 |

例如 `params: 4` 表示 `4 × 2^30` 个参数。通常可省略 params，由注册表读取权重 metadata；读取失败时将真实参数总数除以 `2^30` 后填写 `params`。models 条目只解析上表字段；GPU 频率由实验脚本处理，当前 ModelSpec 省略相关字段。

## devices 条目

| 字段 | 说明 |
|---|---|
| name | 必填，设备逻辑名；应与检测/别名匹配 |
| memory | 必填，乘以 2^30 得到容量 bytes |
| bandwidth | 可选，乘以 2^30 得到 bytes/s；省略会尝试 NVML 检测 |
| profile | 通过 DeviceSpec.extra 保存，指定性能目录别名 |
| 其他字段 | 保存到 DeviceSpec.extra；只有实现中显式读取的字段才会生效 |

PCIe 最大链路代际和宽度只用于估算带宽，H2D 实测值应通过性能采集获得。省略 devices 后 registry 为空；运行时通过 PyTorch CUDA 查询设备名称与容量，并通过 NVML 估算 PCIe 带宽。设备查询或带宽检测失败时，应在 YAML 中显式配置。

## cuda_graph 默认与约束

| 字段 | 配置类默认值 | 约束 |
|---|---|---|
| mode | 'off' | 'off' / save / load；建议均加引号 |
| archive_dir | '' | save/load 必填 |
| batch_sizes | [1,2,4,8] | 正整数、非空；会排序去重 |
| seq_len_buckets | [128,256,512,1024,2048] | 配置类接受多个；**当前 Worker save/load 必须只填一个最大长度桶** |
| strict | true | 控制初始化 scratch 等严格检查；ABI/layout/binding 校验始终执行 |
| base_address | 0x400000000000 | 正整数、2 MiB 对齐；可写带引号十六进制字符串 |
| region_size_bytes | 274877906944 | 256 GiB 虚拟地址范围；正整数、2 MiB 对齐 |
| scratch_size_bytes | 4294967296 | 4 GiB；正整数、2 MiB 对齐，且小于 region_size_bytes |

Foundry hook 当前接入 P/D Decode 路径，正常 Simple 路径使用常规执行。见 [CUDA Graph](cuda-graphs.md)。

## 修改配置后

配置注册保存在进程全局状态中，`set_model_registry()` 初始化后直接返回。修改 YAML 后请重启服务。部署 API 只修改运行时状态，YAML 文件保持原样。
