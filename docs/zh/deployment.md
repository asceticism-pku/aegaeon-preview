# 单节点、多 GPU 与多节点部署

## 三种资源层级

物理 Ray 节点对应 Controller；Engine 是调度和执行单位；Worker 是占用 GPU 的 Ray actor。Aegaeon 只支持 TP=1，因此一个 Engine 对应一个 Worker 和一张 GPU。

GPU 需求：Simple 每节点为 `num_engines` 张；P/D 每节点为 `num_prefill_engines + num_decode_engines` 张。

单机四张 GPU 仍是 nnodes=1；用 num_engines=4 表达四个 Simple Engine。nnodes=4 表示四个物理节点，API 会要求 node_0 到 node_3 的 Ray 自定义资源。

## Simple

```yaml
server:
  nnodes: 1
  num_engines: 2
  num_prefill_engines: 0
  num_decode_engines: 0
  tensor_parallel_size: 1
```

同一引擎负责一个请求的两个阶段，共用一张 GPU。多模型调度会在该引擎上切换权重并按需换出 KV。

## P/D 分离

```yaml
server:
  nnodes: 1
  num_engines: 0
  num_prefill_engines: 1
  num_decode_engines: 1
  tensor_parallel_size: 1
```

1P+1D 需要两张空闲 GPU，CPU KV 负责节点内 Prefill/Decode 数据中转。1P+2D 需要三张 GPU，可用于 Work Stealing。该传输路径只作用于节点内部。

## 两物理节点

所有节点预先安装相同依赖与代码，准备同路径模型、同 profile，并确认端口和网络可达。以下 HEAD_IP 必须换成真实可达地址：

```bash
# Head 节点
ray start --head --port=6789 --num-cpus="$(nproc)" --resources='{"node_0": 1}'
# Worker 节点
ray start --address=HEAD_IP:6789 --num-cpus="$(nproc)" --resources='{"node_1": 1}'
```

```yaml
server:
  nnodes: 2
  num_engines: 1
  num_prefill_engines: 0
  num_decode_engines: 0
  tensor_parallel_size: 1
```

```bash
aegaeon start --config /srv/aegaeon/config.yaml --ray-address HEAD_IP:6789 --host 127.0.0.1
```

引擎设置按节点统一应用，API 会构造同构 NodeConfig 列表。当前 YAML 和文档覆盖同构节点配置；异构拓扑需要在 Python 控制面中自行构造。

启动选中的模型会在每个配置 Controller 上缓存。若希望之后按策略选择单独节点，可启动空服务并用 deploy 明确放置。

## GPU 可见性与 CPU 资源

Controller / Worker 会清除 Ray 设置的 CUDA_VISIBLE_DEVICES，然后依据 AEGAEON_CUDA_VISIBLE_DEVICES 重新设置；LLMService 还把 driver 的 CUDA_VISIBLE_DEVICES 转为 actor 环境。排查设备映射时应按这套自定义 GPU namespace 逐层核对 driver、actor 与 Worker。

多节点应使用一致的 GPU 编号和可见集合。driver 的物理 GPU 编号会传播给 actors，因此每台节点都要提供对应设备。生产拓扑使用真实 Ray 节点资源。

每 Worker 的 CPU 资源由 `worker_num_cpus` 控制。每节点的 Engine placement groups 需要 `Engine 总数 × worker_num_cpus` 个 Ray CPU 资源，Controller 还需要可调度的默认 actor CPU 资源。增加 Engine 后应同步提供足够资源；`ray status` 可用于检查资源需求。GPU 分配由 Aegaeon 的 device ID 映射管理，Worker 的 Ray options 未申请 `num_gpus`，因此还需在部署侧独占所选 GPU。

## 停止与隔离

前台服务通过正常中断停止；lifespan 结束时调用 ray.shutdown 断开 driver。已有 Ray 集群的生命周期由集群管理工具负责，driver 退出后还需按运维流程检查集群和缓存状态。

共享内存文件名与 actor 名称存在固定命名，生产部署应为不同服务提供独立进程/集群/共享目录和资源分配。`ray stop` 影响该机器上的 Ray 进程，仅在确实拥有该实例生命周期时使用。

## 容器 / Kubernetes

本文使用源码安装方式；仓库内未提供标准 Dockerfile 或 Helm chart。自定义镜像需要匹配 CUDA/驱动接口，配置 shared memory 和 pinned memory，挂载模型/profile，并为 Ray 使用一致的 namespace。发布前执行真实文本请求和退出清理测试。
