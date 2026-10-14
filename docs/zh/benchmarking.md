# 基准测试与可复现评估

## 先定义测量口径

| 指标 | 推荐定义与注意事项 |
|---|---|
| TTFT | 请求发出/到达至首 token；客户端文本与后端 token 分开命名 |
| E2E | 请求开始至明确结束事件；写清 finish_reason 还是 [DONE] |
| TPOT | 多 token 输出中 Decode 平均 token 间隔；HTTP chunk 间隔属于另一项指标 |
| Throughput | 实际完成 tokens 或 requests / 明确的墙钟时间区间 |
| Makespan | 固定 workload 从开始到全部完成 |
| SLO / qos | 使用具体源码和公式定义，每种定义单独列示 |
| CV | population stddev/mean，说明引擎集合与 token计数范围 |

## 通过 HTTP 重放仓库中的 workload

这三份 HTTP CSV 示例随源码交付；生成的 CSV 结果仍由 Git 忽略。表头、模型映射和请求数量见 `workloads/README.md`。

先按[安装指南](installation.md)和[快速开始](quickstart.md)准备运行环境，在 Aegaeon 源码仓库根目录执行。
`benchmark/config-pd-2gpu.yaml` 是双 GPU 的 PD 示例，注册了 `qwen2_5_7b` 和
`qwen2_1_5b`；启动前必须将其中特定主机的模型路径和设备设置改为本机实际配置。
在独立终端启动服务：

```bash
aegaeon start --config benchmark/config-pd-2gpu.yaml --ray-address local
```

服务启动完成后，检查 API 并重放已有 CSV：

```bash
curl --fail http://127.0.0.1:8000/health
curl --fail http://127.0.0.1:8000/v1/models
python aegaeon/test_api.py --csv workloads/synth-1-0.1-1.0-1.0.csv \
  --models qwen2_5_7b qwen2_1_5b --request-timeout 300
```

另两份 HTTP workload 是 `workloads/synth-2-0.1-1.0-1.0.csv` 和
`workloads/synth-2-2.0-1.0-1.0.csv`，使用同一命令替换 CSV 路径即可。表头为
`Request Time,Model Name,Input Length,Output Length`；到达时间是相对开始时刻的
`HH:MM:SS.mmm`，`synth_model_00/01` 按顺序映射到 `--models` 的部署名称。
客户端将 `"test "` 重复 Input Length 次，**不保证实际 tokenizer token 数等于该字段**。
Output Length 是请求的 `max_tokens` 上限，遇到 EOS 可以提前结束。

客户端发送非流式请求，统计可读取的 HTTP 200 响应、逐请求延迟和失败/超时数；有任意失败就以非零状态退出。
300 秒超时从各请求的到达等待结束后起算；总耗时包括所有到达等待。
该工具不测量 TTFT、TPOT 或实际输出 tokens/s。报告性能时保存客户端和服务日志，并核对实际 token 数；
HTTP 成功本身也不等于生成内容正确。

## 在线接口演示工具

```bash
python benchmark/demo_api.py --help
python benchmark/demo_api.py \
  --host 127.0.0.1 --port 8000 \
  --model-config /srv/aegaeon/config.yaml --model Qwen/Qwen3-4B \
  --arrival-rate 0.1 --duration 60
```

该工具连接已经启动的 API；示例模型名需替换为配置中已部署的模型名。结果写到 benchmark/demo-output.json，重复运行会覆盖同一路径。默认到达间隔服从指数分布、按模型轮转，普通定时模式固定 max_tokens=500；`--max-tokens` 只作用于 `--single`。输入是固定文本，CSV 的 input_tokens 被读取但未用于构造输入。流式结果的 `tokens` 是非空文本 chunk 数，非流式结果取 usage.completion_tokens；实验报告需区分两种口径。

它的 `--csv` 表头为 `request_id,timestamp,input_tokens,output_tokens`。
`timestamp` 是以秒为单位的到达延迟，`request_id` 对配置模型数取余后选择模型，
`output_tokens` 设置 `max_tokens`。固定 prompt 不兑现 `input_tokens` 指定的输入长度。
仓库的 `workloads/synth-*.csv` 使用不同表头，应使用上节的 HTTP 重放客户端。
这个流式演示工具的帮助仍需先安装 aiohttp。
下列 latency、模型放置、Work Stealing、结果比较、profiling 和 HTTP workload
帮助命令只需标准库，不会初始化 CUDA、Ray 或模型。实际测量和 GPU smoke
仍需兼容的运行环境、checkpoint、profiles 和足够资源；HTTP 客户端发送请求时
需要 aiohttp，GPU 配置辅助函数被调用时需要 PyYAML。

```bash
python benchmark/benchmark_latency.py --help
python benchmark/model_placement_gpu_smoke.py --help
python benchmark/work_stealing_gpu_smoke.py --help
python benchmark/compare_work_stealing_results.py --help
python profiles/do_profile.py --help
python aegaeon/test_api.py --help
```

模拟器帮助需安装下节列出的 CPU 依赖；独立能耗 benchmark 仍在解析帮助前
加载可选包。

## 推荐实验设计

1. 固定模型 checkpoint、tokenizer、完整 YAML、依赖和源码 revision。
2. 定义同一请求序列、真实 token长度、到达模型分布、并发和持续时间。
3. 单独记录 cold start、CPU cache命中、GPU切换、warm steady state。
4. 所有副本按实验定义一致预热；零预热实验则保持冷启动。
5. OFF/LOAD 或策略A/B 交替顺序，多次重复，记录P50/P95与样本数。
6. 核对输出token IDs、请求完成数、异常数、资源与退出清理。
7. 保留原始日志、响应、配置和结果，对失败样本明确标记。

## 专项工具

Work Stealing 使用 off/on 成对 smoke 与 compare_work_stealing_results.py；模型放置使用 model_placement_gpu_smoke.py，可比较 least-loaded / round-robin 和 busy-avoidance 路由。

Graph 工具有 benchmark_cuda_graph_latency.py、benchmark_foundry_three_model_latency.py 和 roundtrip tests；它们的默认路径及 archive 前提见脚本和 `tools/README_three_model_latency.md`。后者测量已启动服务、CPU 权重可用场景下的 GPU 真实切换；CPU cache miss 部署和完整启动成本需要单独测量。

## 性能声明规范

报告应列出 GPU/CPU/互联、驱动、torch/vLLM 版本、源码、模型、模式、预算、batch、真实长度、到达率、预热、样本数和统计方法。性能对比只使用正确性检查通过的样本，并随结果保留硬件、配置与测量条件。

## 模拟器

模拟器默认直接使用仓库已有的 `profiles/`。在干净源码检出、标准 CPython 3.10
环境中执行：

```bash
python -m pip install -r sim/requirements.txt
python -m sim.simulate --workload synth-8-0.1-1.0-1.0.csv \
  --devices a100 --seed 0 --job-timeout 30 \
  --output-dir work/sim-results --name historical-a100
python -m pytest tests/test_simulator.py tests/test_sim_repository_profiles.py \
  tests/test_sim_profile_collector.py -q
```

历史八模型 trace 使用六个 Qwen2.5-7B 和两个 Yi-1.5-9B-Chat 别名，通过明确的名称
映射查找 `profiles/qwen2_5_7b/A100` 和 `profiles/yi1_5_9b_chat/A100`。
无需复制数据、创建软链接、下载模型，或安装 Torch、vLLM、Ray、CUDA。
默认根目录相对于模拟器源码位置解析。`--profile-root PATH` 将查找限定在指定根目录内，
同时支持模拟器规范名称和 serving 目录名称。名称映射表和可选采集命令见 `sim/README.md`。

结果保存为 `work/sim-results/historical-a100.json`，按策略和请求 ID 分组，
两种策略各有 81 个请求结果。成功数组依次为模拟 TTFT 和相邻 token 间隔，单位为毫秒；
`[-1]` 表示调度等待超时失败。上述单张模拟 A100 命令中，SeLLM 完成 33 个、失败 48 个；
SeLLM+ 完成 37 个、失败 44 个。相同输入和 seed 可以重复得到相同的 CPU 结果，
仿真跑完不等于所有请求都满足 SLO。

profile 中 `avg_prefill_latency` 和 `avg_decode_latency` 的单位为秒。
已有数据保留原测量条件。历史 serving 采集脚本默认使用 TP=1、float16 和 eager，
但 Qwen2.5-7B、Yi-1.5-9B 文件缺少每次原始运行的完整来源记录，脚本默认值仅作为参考，
每个文件的实际采集设置仍缺少完整依据。不同运行时或 BF16 配置的耗时需要另行采集。`a100`（80GB）与 `a100_40gb` 明确区分，L20 与 H20 也不混用；
缺失的模型／设备组合会明确报错。

`--workload` 运行固定 CSV，与现有 `--num-models`/`--arrival-rate` sweep 参数互斥。
显式输入输出长度按 CSV 行保持配对；历史两列 trace 根据 seed 采样长度。
synthetic 正确性测试继续使用独立的虚构 fixture。

结果是 execution profile 驱动的仿真估计。线性拟合误差、权重和运行时 buffer 的近似，
以及模型交换的带宽假设都会影响精度；GPU 容量或 SLO 预测的准确度需要与目标运行时的实测结果对照。
其他 workload 按 [profiling 指南](profiling.md)采集并保留来源，验证模型和设备对应关系。
可选工具 `tools/collect_sim_profiles.py` 将新采集的 Qwen/A100 40GB profiles 写到
`work/collected-sim-profiles`，本例无需重新进行 GPU 采集。
缺失／空 profile、子进程错误或超时会明确失败并清理子进程。
