# 功耗采样与 GPU 频率控制

## 功耗采样

`GET /v1/aegaeon/gpu-power` 通过 nvidia-smi 读取 GPU 采样。NvidiaSmi 层解析设备标识、功率和频率等字段；命令失败或未采集到设备时，响应中的 `gpus` 为空列表，详细原因写入服务日志。

该接口采集 API 所在主机。分布式部署需要在各 Ray 节点分别采集再聚合。能量评估通过功率积分计算，并明确时间区间、采样频率、空载基线和真实完成 tokens。

## 频率控制组件

GpuFrequencyController 由 `AEGAEON_GPU_FREQ_CONTROL` 显式启用，默认关闭。Worker 在加载模型时调用 `set_for_model()`；传入频率范围时执行锁频，传入 `None` 时重置已有锁频。`AEGAEON_GPU_CLOCK_COMMAND` 可指定 wrapper；命令执行需要相应的设备、驱动和系统权限。

## 当前配置接线限制

`model_gpu_clock_range()` 从 `model.extra` 读取 gpu_clock_range_mhz 或 gpu_clock_mhz，并支持按 stage 映射。当前 ModelSpec 与 YAML 模型解析省略 extra，因此这条读取路径只服务于实验代码。

每模型频率当前由实验脚本控制，models YAML 尚未接入这项配置。完整服务接入需要补充 ModelSpec 字段、YAML 解析和 Worker 映射。

## 实验工具

tools/ 包含 gpu_clock_search.py、pd_clock_search.py、energy_request.py 与 aegaeon-gpu-clock，benchmark/ 包含 benchmark_energy.py。先读取 --help 和脚本前提，确认频率修改、输出路径和模型配置。这些工具用于实验与测量；自动节能调度需要单独实现。

## 合理报告

对比实验应保持输出、负载和请求顺序一致，并同时报告吞吐、E2E、TTFT、TPOT、能量/请求和能量/token。记录锁频命令结果与恢复状态；能量以功率在完整执行时间上的积分计算。
