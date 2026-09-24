# Profiling 与时延估计

PrefillEstimator 和 DecodeEstimator 使用目标 GPU 上真实采集的 JSON 拟合 LinearRegression。缺少有效样本时会抛出 FileNotFoundError；请先完成采集，再启动依赖 profile 的服务。

## 目录

```text
profiles/
  qwen3_4b/
    A100-PCIE-40GB/
      i32b1.json
      i64b1.json
      ...
```

模型键优先显式 profile；未设置时规范化名字，并存在旧目录兼容探测。设备键优先 DeviceSpec.extra.profile，再探测 CUDA 产品名和逻辑设备名。

JSON 需要包含采样器和估计器使用的 prefill_latencies、decode_latencies 等字段。时延提取会排序、去掉首尾值再取平均，因此每组至少采集 3 个样本；实际测量建议增加重复次数。

## 批量采集 TP=1

```bash
CUDA_VISIBLE_DEVICES=0 python profiles/do_profile.py --model-config /srv/aegaeon/config.yaml --tp 1
```

脚本枚举输入长度16到4096、batch集合及过滤条件，调用 benchmark/benchmark_latency.py，以 bf16、enforce_eager、gpu-memory-utilization=0.90、max-model-len=8192 采集。它会遍历配置中的全部模型，不按 startup_models 限制。

批量脚本固定使用 max-model-len=8192 和 gpu-memory-utilization=0.90。原生上下文小于 8192 的模型或显存容量不足的设备应改用单点采集命令。脚本通过 shell 启动子命令，完成后需要检查日志和输出 JSON 的样本数。

## 单点采集

```bash
mkdir -p profiles/qwen3_4b/A100-PCIE-40GB
CUDA_VISIBLE_DEVICES=0 python benchmark/benchmark_latency.py \
  --model /srv/models/Qwen3-4B --dtype bfloat16 --enforce-eager \
  --tensor-parallel-size 1 --max-model-len 4096 \
  --input-len 32 --output-len 10 --batch-size 1 \
  --output-json profiles/qwen3_4b/A100-PCIE-40GB/i32b1.json
```

一个采样点只用于确认采集链路。用于调度的 profile 应覆盖目标输入长度、batch 和上下文范围，并记录回归误差。

## TP 与异构硬件

`--tp` 固定设为 1。批量脚本虽然暴露了该参数，但 GPU 分配按单设备构造，估计器也明确执行 `assert tp == 1`。

脚本还用可见 GPU0 的产品名构造目录；异构 GPU 要分别采集并核对目录，避免为其他设备写入错误标签。

## 估计值用途

预测值用于调度选择，性能报告应使用实测结果。后端、图模式、模型版本、频率、驱动或算子发生变化后，需要在对应环境重新采集 profile。
