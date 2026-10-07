# Frontend / parameter compatibility review

Reviewed against `origin/migration/vllm-0.26.0-upgrade` at `0c38c04`.
This records the frontend and API adaptations and verification performed on
2026-09-17. It is a point-in-time test record, not a guarantee for later changes.

## Adaptations

- Resolve API conflicts by retaining both model-specific tool parsing and the
  upstream replica deployment, routing reservations, stop-token and minimum-token
  handling. Preserve tool-only assistant messages and tool template options.
- Update the [API compatibility reference](../aegaeon/API_COMPATABILITY.md) and
  retain startup tests for both tool-parser configuration and work stealing.
- Console deployment now accepts replica count, explicit nodes and a tool parser.
  Node names accept commas or whitespace and are deduplicated.
- Console and CLI report partially failed deploy/undeploy operations. CLI exits
  nonzero so scripts cannot mistake HTTP 200 partial results for full success.
- Parameter testing now checks minimum output length, distributed runtime fields,
  invalid replica counts and empty node lists, and rejects partial deployment
  success. Documentation reflects routing to ready replicas.

## Local verification (Windows)

- 50 tests passed across tool calls, placement, routing tables, dynamic registry,
  CLI and parameter-matrix checks.
- Python compilation, JavaScript syntax, undefined-name checks for modified
  Python code, and `git diff --check` passed.
- Browser verification used simulated HTTP responses, not a running GPU cluster:
  deployment payload, node deduplication, parser forwarding, partial deploy and
  undeploy failures, and a 390-pixel viewport without horizontal overflow passed.
- Full API and startup tests could not run: this environment lacks PyTorch,
  Ray and vLLM. Added API regression tests are included for the server environment.

## Remaining server verification

Run from the repository root on the Linux inference server, with its Aegaeon
environment activated:

```bash
python -m pytest -q tests/test_api_routing.py tests/test_startup_config.py \
  tests/test_model_replica_management.py tests/test_model_runtime_snapshot.py \
  tests/test_model_placement_cli.py tests/test_param_matrix.py

# Against an already running service and an already deployed model:
python tools/param_test.py --host 127.0.0.1 --port 8000 --models Qwen/Qwen3-4B
```

Real GPU inference, vLLM parser behavior and multi-node failure recovery still
require that environment. Existing upstream GPU smoke scripts are documented in
[model placement](model-placement.md) and [work stealing](work-stealing.md).
