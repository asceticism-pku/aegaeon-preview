# Configuration Examples

Replace model paths, GPU devices, and profile aliases before use, and adjust budgets for available memory.

| File | Topology |
|---|---|
| simple.yaml | One node, 1 Simple Engine, TP=1 |
| pd.yaml | One node, 1P+1D, TP=1 |
| multi-node.yaml | Two physical nodes, 1 Simple Engine per node |
| dynamic.yaml | Empty startup followed by explicit deployment |
| work-stealing.yaml | One node, 1P+2D |
| foundry-save.yaml / foundry-load.yaml | P/D Decode, batch=1, one 4096 bucket; requires Foundry and the hook; SAVE creates archives, LOAD requires complete archives |
| chat-request.json | Minimal nonstreaming greedy request |

```bash
curl -sS http://127.0.0.1:8000/v1/chat/completions -H 'Content-Type: application/json' --data-binary @docs/en/examples/chat-request.json
```

Do not run these configurations simultaneously on the same GPU cluster. SAVE and LOAD must retain matching layouts, models, and dependencies.
