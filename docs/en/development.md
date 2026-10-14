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
| plots/ | Plot scripts; prepare workload files at the paths required by each tool |
| web/ | Built-in pages |
| docs/zh/, docs/en/ | Chinese and English documentation and examples |

The main project uses Apache-2.0. Preserve license information for additional components and dependencies. See Foundry's subproject documentation for standalone integrations.

## Develop a feature

Wire configuration through parsing → configuration classes → API creation → Ray environment / Controller → Engine/Worker. Cover success paths, failure paths, scope, defaults, and limitations together.

Extend models and devices through registries/YAML. `aegaeon.models.ModelType` and `aegaeon.utils.DeviceType` are namespaces preserving older attribute access. A new KV type needs tests for descriptions, block counts, transfer layouts, sliding windows, and state restoration.

## Validation layers

1. GPU-free logic: graph key selection, catalog paths, configuration schemas, policy decisions, and cache mappings.
2. Mocked control plane: deployment, reservation release, busy rejection, partial failures, and SSE cleanup.
3. One real GPU model: loading, EOS, context, correct output, and shutdown.
4. Model switching/concurrency: cache transfers, request conservation, identical outputs, and resource recovery.
5. Target topology: P/D, multiple nodes, Work Stealing, and Foundry. Use a TP=1 baseline. See [capabilities](capabilities.md) for model and input-type scope; new input types need corresponding processing and inference tests.

`tests/` contains both pure-logic tests and tests that require real models and CUDA. Select test files according to their documented dependencies.

```bash
python -m pytest tests/test_cuda_graph_catalog_config.py
```

This module checks archive catalog rules and requires pytest with the correct project import path. Runtime changes need meaningful failure and boundary checks rather than assertions mirroring implementation details.

The installer regression tests create their temporary environment with the interpreter running pytest. Use the supported standard CPython 3.10 environment. The selector uses the CPython 3.10 wheel ABI, and the tests use the current interpreter without installing GPU packages.

```bash
python --version
python -m pytest tests/test_docs_checker.py tests/test_install_quickstart.py -q
python -m pytest tests/test_foundry_runtime_active_model.py -q
```

The first command should report Python 3.10. These checks validate documentation, installer preflight, and mocked Foundry archive handling; they do not build native extensions or validate GPU serving. Native builds and imports require the [installation toolchain](installation.md).

`tests/test_init.py` runs the real `LLMService` constructor with an explicitly initialized server configuration and isolated Ray/Controller boundaries. It checks propagation of each node's configuration, ready replica placement and routing, initialization failure propagation, and the error contract for a missing server configuration. The test environment must support importing the full serving dependencies; the tests do not start Ray, reserve GPUs, load checkpoints, or allocate model caches.

```bash
python -m pytest tests/test_init.py -q
```

The `tests/test_serve*.py` files remain historical Ray/GPU integration examples. Adapt them to the current server configuration and model/device registries, with real checkpoint paths and sufficient Ray/cache resources, before using them for integration validation. Passing these isolated constructor and logic tests does not establish end-to-end GPU serving or that every test in `tests/` passes.

## Documentation maintenance

MkDocs Material builds the Chinese and English sites with `mkdocs.zh.yml` and `mkdocs.en.yml`. Install documentation dependencies in an isolated environment:

```bash
python -m pip install -r docs/requirements-docs.txt
python -m mkdocs serve -f mkdocs.zh.yml
python -m mkdocs build --strict -f mkdocs.zh.yml
```

For English, use `python -m mkdocs serve -f mkdocs.en.yml` or `python -m mkdocs build --strict -f mkdocs.en.yml`. Publish the two outputs together as sibling `zh/` and `en/` directories to enable page-preserving language links.

Update configuration, CLI, HTTP fields, and examples together in both languages. When adding or removing top-level classes or functions, also update the source module index. Run `python docs/check_docs.py` and complete strict builds before submitting changes.

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
