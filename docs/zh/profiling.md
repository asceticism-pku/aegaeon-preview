# Profiling 与时延估计

PrefillEstimator 和 DecodeEstimator 使用目标 GPU 上真实采集的 JSON 拟合 LinearRegression。找不到可用于拟合的 profile 数据时会抛出 FileNotFoundError；JSON 格式错误或单点样本不足会在读取或计算时失败；请先完成采集，再启动依赖 profile 的服务。

## 目录

```text
profiles/
  Qwen/
    Qwen3-4B/
      A100-PCIE-40GB/
        i32b1.json
        i64b1.json
        ...
```

此目录与 [QuickStart](quickstart.md) 的 `profile: Qwen/Qwen3-4B` 及仓库自带的 A100 profile 数据一致。模型键优先显式 profile；未设置时规范化名字，并存在旧目录兼容探测。设备键优先 DeviceSpec.extra.profile，再探测 CUDA 产品名和逻辑设备名。

显式模型 `profile` 按配置原样使用。仓库自带的目录是 `Qwen/Qwen3-4B`，复用时填写该别名；填写 `profile: qwen3_4b` 则需另行采集 `profiles/qwen3_4b/` 下的数据。省略 `profile` 时，先查规范化的模型名目录，再兼容查找原始模型名目录；此回退只适用于未显式设置模型别名的情况。

下方采集命令的 `--model` 是 checkpoint 路径，`--output-json` 决定 profile 文件位置。批量采集按 `models[].profile` 构造模型目录，按所分配 GPU 的产品名构造设备目录。服务配置中的别名应匹配实际产出的目录；这些键与公开 API 模型名、本地 checkpoint 路径分别独立。

`AEGAEON_PROFILE_ROOT` 指定服务与批量采集共用的根目录，默认是仓库的 `profiles/` 目录。单点采集的 `--output-json` 也应指向该根目录下的对应路径。

JSON 需要包含采样器和估计器使用的 prefill_latencies、decode_latencies 等字段。时延提取会排序、去掉首尾值再取平均，因此每组至少采集 3 个样本；实际测量建议增加重复次数。

采集器直接运行 vLLM 的 `LLM`，不是 Aegaeon HTTP 服务。prefill 样本取 batch 中首个请求的 `first_token_ts - scheduled_ts`；decode 样本取该请求首末 token 间隔除以生成 token 数减一，单位均为秒。默认执行 4 次预热和 12 次采样，并使用 `[0, 10000)` 的随机 token ID；用于该脚本的模型需包含这些有效 ID。

采集时显式关闭 prefix caching，使重复 warmup 和测量输入执行实际 prefill。

## 批量采集 TP=1

```bash
CUDA_VISIBLE_DEVICES=0 python profiles/do_profile.py --model-config /srv/aegaeon/config.yaml --tp 1
```

脚本枚举输入长度16到4096、batch集合及过滤条件，调用 benchmark/benchmark_latency.py，以 bf16、enforce_eager、gpu-memory-utilization=0.90、max-model-len=8192 采集。它会遍历配置中的全部模型，不按 startup_models 限制。

批量脚本固定使用 max-model-len=8192 和 gpu-memory-utilization=0.90。原生上下文小于 8192 的模型或显存容量不足的设备应改用单点采集命令。每个分配的 GPU 在独立子进程中执行 benchmark；命令失败、输出缺失或样本数组无效时，采集返回非零退出码。新 JSON 通过校验后才替换原 profile。

## 单点采集

```bash
mkdir -p profiles/Qwen/Qwen3-4B/A100-PCIE-40GB
CUDA_VISIBLE_DEVICES=0 python benchmark/benchmark_latency.py \
  --model /srv/models/Qwen3-4B --dtype bfloat16 --enforce-eager \
  --tensor-parallel-size 1 --max-model-len 4096 --gpu-memory-utilization 0.90 \
  --input-len 32 --output-len 10 --batch-size 1 \
  --output-json profiles/Qwen/Qwen3-4B/A100-PCIE-40GB/i32b1.json
```

一个采样点只用于确认采集链路。用于调度的 profile 应覆盖目标输入长度、batch 和上下文范围，并记录回归误差。

## TP 与异构硬件

`--tp` 固定设为 1。批量脚本虽然暴露了该参数，但 GPU 分配按单设备构造，估计器也明确执行 `assert tp == 1`。

脚本按每个模型分配到的 GPU 产品名构造输出目录，服务配置中的设备 profile 别名应与该目录相同。异构部署需要同一模型在各类目标 GPU 上的 profile 时，应分别采集。

## 估计值用途

预测值用于调度选择，性能报告应使用实测结果。后端、图模式、模型版本、频率、驱动或算子发生变化后，需要在对应环境重新采集 profile。
