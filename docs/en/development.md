# Development and Contributions

## Project layout

| Directory | Contents |
|---|---|
| aegaeon/ | Core system, HTTP, configuration, routing, scheduling, Workers, and caches |
| ops/ | C++/CUDA KV transfer extension |
| quick_model_loader/ | Rust/Python fast weight loading |
| foundry/ | Graph serialization/restoration and standalone engine integrations |
| profiles/ | Collection scripts and performance data |
| benchmark/, tools/ | Benchmark and specialized test tools; each script documents its parameters and environment |
| tests/ | Control logic and GPU-dependent tests |
| sim/ | SimPy policy simulation |
| plots/, workloads/ | Plots and workload resources |
| web/ | Built-in pages |
| docs/zh/, docs/en/ | Chinese and English documentation and examples |

The main project uses Apache-2.0. Preserve license information for additional components and dependencies. See Foundry's subproject documentation for standalone integrations.

## Develop a feature

Wire configuration through parsing → configuration classes → API creation → Ray environment / Controller → Engine/Worker. Cover success paths, failure paths, scope, defaults, and limitations together.

Extend models and devices through registries/YAML; `ModelType` and `DeviceType` are no longer enums. A new KV type needs tests for descriptions, block counts, transfer layouts, sliding windows, and state restoration.

## Validation layers

1. GPU-free logic: graph key selection, catalog paths, configuration schemas, policy decisions, and cache mappings.
2. Mocked control plane: deployment, reservation release, busy rejection, partial failures, and SSE cleanup.
3. One real GPU model: loading, EOS, context, correct output, and shutdown.
4. Model switching/concurrency: cache transfers, request conservation, identical outputs, and resource recovery.
5. Target topology: P/D, multiple nodes, Work Stealing, and Foundry. The release scope fixes TP at 1 and supports text input only.

`tests/` contains both pure-logic tests and tests that require real models and CUDA. Select test files according to their documented dependencies.

```bash
python -m pytest tests/test_cuda_graph_catalog_config.py
```

This module checks archive catalog rules and requires pytest with the correct project import path. Runtime changes need meaningful failure and boundary checks rather than assertions mirroring implementation details.

## Documentation maintenance

MkDocs Material builds the Chinese and English sites with `mkdocs.zh.yml` and `mkdocs.en.yml`. Install documentation dependencies in an isolated environment:

```bash
python -m pip install -r docs/requirements-docs.txt
python -m mkdocs serve -f mkdocs.zh.yml
python -m mkdocs build --strict -f mkdocs.zh.yml
```

For English, use `python -m mkdocs serve -f mkdocs.en.yml` or `python -m mkdocs build --strict -f mkdocs.en.yml`. Publish the two outputs together as sibling `zh/` and `en/` directories to enable page-preserving language links.

Update configuration, CLI, HTTP fields, and examples together in both languages. Run `python docs/check_docs.py` and complete strict builds before submitting changes.

Generate the bilingual offline page:

```bash
python docs/build_portable.py --output /tmp/aegaeon-docs.html
```

To update the service's built-in documentation at the same time:

```bash
python docs/build_portable.py --output /tmp/aegaeon-docs.html --web-dir web
```

## Pull Requests

Explain user-visible behavior, defaults, compatibility, and failure handling, with relevant tests. Documentation changes should include accurate examples and clear prerequisites.
