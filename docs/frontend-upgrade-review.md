# Frontend / parameter compatibility review

This replaces the earlier point-in-time review narrative with references to the
current source and commands for verification. The earlier narrative's test count
and browser results had no execution logs attached to this file and are omitted.
The commands below describe checks to run; they are not recorded pass results.

## Current implementation references

- `aegaeon/api.py` combines model-specific tool parsing with replica deployment,
  routing reservations, stop tokens, and minimum-token validation.
  `aegaeon/tool_calls.py` normalizes tool messages; API template helpers forward
  tool options. See the [API compatibility reference](../aegaeon/API_COMPATABILITY.md).
- `web/aegaeon_console.js` sends replica counts, explicit node IDs, and optional
  tool-parser names. It splits node names on commas or whitespace and removes
  duplicates before submission.
- The console shows partial deploy/undeploy failures. `aegaeon/cli.py` exits with
  status 1 when a result has `status=partial` or a nonempty `failed` map, including
  HTTP 200 responses.
- `tools/param_test.py` contains minimum-output-length and runtime-field checks,
  deploy-matrix cases for invalid replica counts and empty node lists, and a
  success predicate that rejects partial deployment results.

## Verification commands

Run from the repository root on the Linux inference server, with its Aegaeon
environment activated:

```bash
python -m pytest -q tests/test_api_routing.py tests/test_startup_config.py \
  tests/test_model_replica_management.py tests/test_model_runtime_snapshot.py \
  tests/test_model_placement_cli.py tests/test_param_matrix.py

# Against a running test service and a deployed model:
python tools/param_test.py --host 127.0.0.1 --port 8000 --models Qwen/Qwen3-4B
```

The parameter matrix sends generation requests, including invalid-parameter
cases. Run it against a test service with the named model ready, and preserve
its raw results. The pytest modules cover mocked control behavior; passing them
alone does not establish real GPU inference, vLLM parser behavior, or multi-node
failure recovery. GPU smoke procedures are described in
[model management](en/model-management.md) and [work stealing](en/work-stealing.md).
