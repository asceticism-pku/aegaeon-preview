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

它的 `--csv` 使用下节的历史 snake_case 表头；仓库的 `workloads/synth-*.csv`
使用上节的 HTTP 重放客户端。该工具的帮助也需先安装 aiohttp。
部分 benchmark 脚本仍在解析帮助前加载可选依赖：latency 需 vLLM，直接 E2E 需 Ray 和服务运行时；
placement/Work Stealing 的帮助需 Ray、PyYAML 和 psutil，实际执行再加载服务运行时。
包入口的 lazy import 不会让这些脚本脱离依赖。
`compare_work_stealing_results.py --help` 和 `profiles/do_profile.py --help`
可仅用标准库执行；模拟器帮助需安装下节列出的 CPU 依赖。

## 当前历史脚本的边界

历史的直接端到端重放脚本为 `benchmark/benchmark_e2e_from_workload.py`，与上面的当前 HTTP
客户端不同。它的默认 workload 文件没有随仓库提供，必须显式传入已有 `--workload-csv` 和
`--model-config`/`--output-json`；当前 HTTP 流程使用已有 CSV 和 YAML 配置。

`benchmark/benchmark_e2e_from_workload.py` 的 Ray 地址固定为原实验集群，每节点固定 1 Prefill + 1 Decode、model_cache_size=0，CPU slab 数来自命令参数；这些设置不会从 YAML 的 server 拓扑读取。使用前应改成当前集群地址和目标配置。

CSV 字段为 request_id、timestamp、input_tokens、output_tokens。脚本按 seed 随机分配模型，并从固定预编码 token 列表切片；超出列表长度的输入会变短，decode 长度还受 4096 规则截断。因此该脚本测量的是经过脚本转换的工作负载，报告中应使用实际 prompt 和输出长度。

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

`sim/` 基于 SimPy，使用估计执行时间研究调度策略。在 Aegaeon 源码仓库根目录、Python 3.10+ 虚拟环境中，
可独立运行的 CPU 正确性检查为：

```bash
python -m pip install aenum typing_extensions simpy==4.1.1 numpy scikit-learn matplotlib pytest
python -m sim.simulate --help
python -m pytest tests/test_simulator.py -q
```

预期为 13 项通过，包含两种策略下仓库的 81 请求 workload。
测试在临时目录生成明确标记为 synthetic 的 profile；这些延迟是虚构的测试数据，不是 GPU 实测。

性能仿真需要真实 profile；仓库未提供 `sim/simdata/execution/`。
按[profiling 指南](profiling.md)在目标 GPU 上采集模型执行时间；当前
`benchmark/benchmark_latency.py` 除服务端采样数组外，还输出以秒为单位的
`avg_prefill_latency` 和 `avg_decode_latency`。
按下方采集、安装步骤，将匹配的实测数据放到
`<profile-root>/<模拟器模型>/<模拟器设备>/i<input_length>b<batch_size>.json`。
服务 YAML alias、CUDA 产品名称与模拟器名称不同；使用前需核对模型 checkpoint、设备参数与测量单位，
再建立匹配的数据目录。名称映射保留原始测量的模型和硬件来源。

### 采集与安装实测 profile

准备包含真实 Qwen2.5-7B、Yi-9B checkpoint 路径和显式 profile 别名的模型 YAML，
配置格式见 [profiling 指南](profiling.md)。采集器在指定 GPU 上以 TP=1 运行 vLLM，
需要完整 GPU 运行环境；该采集步骤与上面的 CPU 模拟器检查分别执行。
在 Aegaeon 源码仓库根目录运行：

```bash
CUDA_VISIBLE_DEVICES=0 AEGAEON_PROFILE_ROOT="$PWD/work/measured-profiles" \
  python profiles/do_profile.py \
  --model-config /absolute/path/to/simulation-profile-models.yaml --tp 1
```

输出位于
`work/measured-profiles/<模型-profile>/<CUDA-产品名>/i<input_length>b<batch_size>.json`，
均值单位为秒。两种延迟估计器均需覆盖不同输入长度和 batch 大小的有效训练点。
随结果保留采集 YAML，以及 checkpoint、硬件、后端和采样信息。

确认测量与模拟器的模型、上下文假设及 **80 GB A100** 设备匹配后，将下面的源路径
替换为实际测量目录，并安装两种模型的数据：

```bash
QWEN_MEASURED_DIR=/absolute/path/to/measured-qwen/product-name
YI_MEASURED_DIR=/absolute/path/to/measured-yi/product-name
mkdir -p work/sim-execution/qwen2.5-7b/a100 work/sim-execution/yi-9b/a100
cp "$QWEN_MEASURED_DIR"/i*b*.json work/sim-execution/qwen2.5-7b/a100/
cp "$YI_MEASURED_DIR"/i*b*.json work/sim-execution/yi-9b/a100/
```

目录映射后仍需保留原始元数据。模拟器检查有效训练 profile 是否存在，
checkpoint 和硬件是否对应则属于实验准备工作。只有原始延迟数组的旧服务 profile
需按明确的平均规则补齐均值后，再作为模拟器输入。

### 运行实测 profile 仿真

准备好 `work/sim-execution` 后运行：

```bash
python -m sim.simulate --num-models 8 --arrival-rate 0.1 --devices a100 \
  --profile-root work/sim-execution --job-timeout 300 \
  --output-dir work/sim-results --name measured-a100
```

八模型 trace 需要 `qwen2.5-7b/a100` 和 `yi-9b/a100` 两组数据；模拟器的 `a100` 表示
80 GB 设备模型。缺失或没有可用训练数据会在启动子进程前明确失败。
结果保存到 `work/sim-results/measured-a100.json`，按 SeLLM/SeLLM+ 策略和请求 ID 分组。
成功请求的数组依次为模拟 TTFT 和相邻 token 间隔，单位均为毫秒；失败请求保存为 `[-1]`。
除进程状态外也要检查这些数组。报告中将仿真与 GPU 实测分开，并记录 profile 来源、模型/设备假设和策略。
只有补齐匹配的实测 profile 后，性能仿真流程才具备完整输入。
