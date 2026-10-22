# Development and Contributions

## Project layout

| Directory | Contents |
|---|---|
| `aegaeon/` | API, routing, scheduling, Workers, and caches |
| `ops/` | C++/CUDA KV transfers |
| `quick_model_loader/` | Rust/Python weight loading |
| `foundry/` | CUDA Graph persistence |
| `profiles/`, `benchmark/`, `tools/` | Profiling and performance tools |
| `sim/` | CPU scheduling simulator |
| `tests/` | Logic and GPU tests |
| `docs/en/`, `docs/zh/` | Documentation and examples |

## Contribute

Open an issue for bugs or feature proposals. Pull requests should describe the behavior change and include relevant tests. Keep configuration, CLI, API, and both documentation languages consistent.

Run focused tests from the repository root:

```bash
python -m pytest tests/test_cuda_graph_catalog_config.py
```

GPU tests require the models and devices specified by each test. Source entry points are in [Architecture](architecture.md).

## Documentation maintenance

```bash
python -m pip install -r docs/requirements-docs.txt
python docs/check_docs.py
python -m mkdocs build --strict -f mkdocs.en.yml
python -m mkdocs build --strict -f mkdocs.zh.yml
python docs/build_portable.py --output /tmp/aegaeon-docs.html --web-dir build/docs
```

Use `mkdocs serve` with either configuration for local preview. Publish the outputs as sibling `en/` and `zh/` directories. The project uses Apache-2.0; retain component and dependency licenses.
