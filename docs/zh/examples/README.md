# 配套示例

使用示例前，请替换模型路径、GPU 设备和 profile，并根据可用内存调整预算。

| 文件 | 拓扑 |
|---|---|
| simple.yaml | 单节点、1 Simple Engine、TP=1 |
| pd.yaml | 单节点、1P+1D、TP=1 |
| multi-node.yaml | 两物理节点，每节点1 Simple Engine |
| dynamic.yaml | 空服务后显式部署 |
| work-stealing.yaml | 单节点1P+2D |
| foundry-save.yaml / foundry-load.yaml | P/D Decode、batch=1、单4096桶；需要 Foundry/hook；SAVE 创建存档，LOAD 要求完整存档 |
| chat-request.json | 最小非流式贪心请求 |

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions -H 'Content-Type: application/json' --data-binary @docs/zh/examples/chat-request.json
```

这些配置按顺序在独占 GPU 集群中运行。SAVE 和 LOAD 使用相同的布局、模型与依赖。
