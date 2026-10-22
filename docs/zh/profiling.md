# Profiling

Aegaeon 使用模型与设备的时延 profile 估计 Prefill 和 Decode 时间。数据位于 `AEGAEON_PROFILE_ROOT`，默认是仓库的 `profiles/`：

```text
profiles/Qwen/Qwen3-4B/A100-PCIE-40GB/i32b1.json
```

模型和设备的 `profile` 字段选择对应目录。在目标 GPU 上采集，并与服务使用相同的运行环境。

## 批量采集

在仓库根目录执行：

```bash
CUDA_VISIBLE_DEVICES=0 python profiles/do_profile.py --model-config /srv/aegaeon/config.yaml --tp 1
```

## 单点采集

需要自定义输入长度或上下文限制时：

```bash
mkdir -p profiles/Qwen/Qwen3-4B/A100-PCIE-40GB
CUDA_VISIBLE_DEVICES=0 python benchmark/benchmark_latency.py \
  --model /srv/models/Qwen3-4B --dtype bfloat16 --enforce-eager \
  --tensor-parallel-size 1 --max-model-len 4096 --gpu-memory-utilization 0.90 \
  --input-len 32 --output-len 10 --batch-size 1 \
  --output-json profiles/Qwen/Qwen3-4B/A100-PCIE-40GB/i32b1.json
```

采集范围应覆盖服务使用的输入长度和 batch size。更换模型、硬件或执行配置后重新采集。
