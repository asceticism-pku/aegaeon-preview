# System Architecture

```mermaid
flowchart TD
    Client[Application client] --> API[FastAPI interface]
    API --> Service[LLMService routing and replicas]
    Service --> Ctrl[Selected physical node Controller]
    Ctrl --> PD[PrefillDispatcher]
    PD --> PE[PrefillEngine and Workers]
    PE -->|First token| Out[StepOutput and responses]
    PE --> DD[DecodeDispatcher]
    DD --> DE[DecodeEngine and Workers]
    DE --> Out
    Ctrl --> Simple[Alternative SimpleDispatcher / Engine]
    Simple --> Out
    Ctrl --- BM[Node-level BlockManager]
    BM --- CPU[Node-shared CPU KV Cache]
    PE --- PKV[Prefill GPU KV cache per Worker]
    DE --- DKV[Decode GPU KV cache per Worker]
    Simple --- SKV[Simple GPU KV cache per Worker]
    BM -. coordinates .-> PKV
    BM -. coordinates .-> DKV
    BM -. coordinates .-> SKV
    Ctrl --- QC[QuickCache CPU weights]
    QC --> PE
    QC --> DE
    QC --> Simple
```

Simple and P/D are alternative execution modes. Each physical node has one Controller and shared CPU caches; each Worker has its own GPU KV cache, coordinated by the node's BlockManager.

## Components

- **API / LLMService:** request formatting, replica placement, and routing.
- **Controller / Schedulers:** node-local dispatch, batches, and model quotas.
- **StageEngine / Worker:** GPU execution using vLLM models and kernels.
- **QuickCache:** CPU model weights and fast loading.
- **BlockManager:** request KV state and CPU/GPU transfers.

A request selects one READY node. Prefill produces the first token; unfinished requests continue with Decode. Model switches preserve KV state through the node's CPU cache. See [Scheduling](scheduling.md) and [Memory](memory.md) for details.


