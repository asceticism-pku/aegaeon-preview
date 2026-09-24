# 源码模块索引

以下索引列出主要模块及其顶层类和函数，供开发时定位代码。应用接入请优先使用公开的 [Python API](python-api.md)。

| 文件 | 顶层符号 |
|---|---|
| `aegaeon/__init__.py` | `__getattr__` |
| `aegaeon/__main__.py` |  |
| `aegaeon/allocator.py` | `_aligned`, `Allocator`, `initialize_alloc`, `has_alloc`, `release_alloc`, `get_alloc` |
| `aegaeon/api.py` | `ServerContext`, `get_context`, `DecodeRateEMA`, `AccessRateTracker`, `_env_float`, `APIModel`, `FunctionDefinition`, `FunctionTool`, `NamedFunctionToolChoice`, `NamedToolChoice`, `FunctionCall`, `ToolCall`, `Message`, `StreamOptions`, `DeployRequest`, `GreedyGenerationRequest`, `ChatCompletionRequest`, `CompletionRequest`, `TokenizeRequest`, `DetokenizeRequest`, `GenerationResult`, `GenerationHandle`, `_select_startup_model_names`, `create_service`, `lifespan`, `_serverlessllm_url`, `_serverlessllm_proxy_error`, `_local_gpu_power_payload`, `proxy_serverlessllm_models`, `proxy_serverlessllm_gpu_power`, `proxy_serverlessllm_chat_completions`, `_get_tokenizer`, `_error`, `_resolve_model`, `_tokenizer_for`, `_effective_max_tokens`, `_validate_greedy_parameters`, `_truncate_prompt`, `_chat_prompt_token_ids`, `_chat_engine_input`, `_completion_prompt_token_ids`, `_stop_strings`, `_decode_token_ids`, `_decode_result`, `_prepare_generation`, `_generate`, `_stop_token_ids`, `_visible_stream_text`, `_text_delta`, `_release_stream_when_finished`, `_schedule_stream_cleanup`, `_generate_stream`, `_response_id`, `_sse`, `_chat_stream`, `_completion_stream`, `_tool_parser_for`, `_tool_parsing_enabled`, `_effective_tool_choice`, `_serialized_tool_choice`, `_tool_template_kwargs`, `_tool_call_parse_options`, `_parsed_chat_output`, `health`, `list_models`, `retrieve_model`, `deploy_model`, `undeploy_model`, `runtime_snapshot`, `access_rate_metrics`, `gpu_power_metrics`, `runtime_events`, `tokenize`, `detokenize`, `create_chat_completion`, `create_completion`, `parse_args`, `demo_redirect`, `_docs_asset`, `docs_css`, `docs_js`, `arena_js`, `integrated_console`, `console_css`, `console_js`, `integrated_chat`, `chat_css`, `chat_js`, `integrated_docs`, `integrated_demo_redirect` |
| `aegaeon/block_manager.py` | `is_gpu`, `CacheGroupConfig`, `CacheGroupRuntimeConfig`, `BlockTable`, `MoveEvent`, `BlockManager` |
| `aegaeon/cache_groups.py` |  |
| `aegaeon/cache_transfer.py` | `logical_blocks_for_tokens`, `sliding_window_resident_block_bound`, `num_skipped_sliding_window_blocks`, `kernel_block_extent_for_attention_page`, `valid_kernel_blocks_for_attention_page`, `prefix_bytes_for_kernel_blocks`, `byte_extent_for_kernel_blocks` |
| `aegaeon/cli.py` | `main`, `_check_partial_result`, `_cmd_deploy`, `_cmd_undeploy` |
| `aegaeon/config.py` | `_MPTAttnConfig`, `_normalize_mpt_config`, `_normalize_telechat3_config`, `NodeConfig`, `ParallelConfig`, `ModelConfig`, `QuickLoaderConfig`, `ServerConfig`, `make_vllm_cache_config`, `get_server_config`, `set_server_config`, `ModelConfigEntry`, `ModelConfigRegistry`, `get_model_config`, `get_vllm_config`, `register_model_config`, `unregister_model_config` |
| `aegaeon/cuda_graph.py` | `GraphKey`, `select_decode_graph_key` |
| `aegaeon/cuda_wrapper.py` | `cudaIpcMemHandle_t`, `cudaIpcEventHandle_t`, `Function`, `CudaRTLibrary` |
| `aegaeon/decode_dispatcher.py` | `coefficient_of_variation`, `WorkStealingRejectionReason`, `WorkStealingStats`, `WorkStealingPlan`, `WorkStealingDecision`, `select_work_steal`, `DecodeDispatcher` |
| `aegaeon/decode_scheduler.py` | `DecodeSchedulerLoad`, `DecodeSchedulerRuntimeStats`, `StealableDecodeBatch`, `DecodeScheduler` |
| `aegaeon/device_registry.py` | `_compute_pcie_bandwidth_gbs`, `_detect_pcie_bandwidth_gbs`, `DeviceSpec`, `DeviceRegistry`, `get_device_registry`, `get_registry`, `set_registry` |
| `aegaeon/energy.py` | `_env_flag`, `_env_int`, `GpuPowerSample`, `NvidiaSmi`, `GpuClockCommand`, `GpuFrequencyController`, `model_gpu_clock_range`, `query_gpu_power` |
| `aegaeon/estimator.py` | `_resolve_default_device`, `_profile_dir`, `_raise_missing_profiles`, `_load_time_profile`, `_get_prefill_latency`, `_get_decode_latency`, `PrefillEstimator`, `DecodeEstimator`, `test_prefill_estimator`, `test_decode_estimator`, `make_estimator`, `cache_estimators` |
| `aegaeon/foundry_runtime.py` | `FoundryRuntime` |
| `aegaeon/graph_bindings.py` | `cuda_tensor_bindings`, `validate_bindings`, `single_split_attention`, `relocate_cuda_buffers` |
| `aegaeon/graph_config.py` | `CudaGraphConfig` |
| `aegaeon/lifetime.py` | `LifetimeEventType`, `LifetimeEvent`, `json_encode_lifetime_events`, `json_decode_lifetime_events` |
| `aegaeon/llm.py` | `_foundry_worker_env`, `_node_runtime_env`, `_get_worker_num_cpus`, `LLMService`, `Controller` |
| `aegaeon/loader/__init__.py` |  |
| `aegaeon/loader/allocator.py` | `DeviceType`, `Device`, `MemoryTable`, `Allocator`, `CUDAllocator`, `CPUAllocator`, `PinnedAllocator`, `find_feasible_addr`, `is_interleaved`, `get_insert_index` |
| `aegaeon/loader/cache.py` | `QuickCache`, `load_tensors_file_buffer` |
| `aegaeon/loader/handle.py` | `StorageHandle`, `TensorsHandle`, `ShardingHandle`, `ModelHandle` |
| `aegaeon/loader/loader.py` | `_normalize_hf_config`, `get_model` |
| `aegaeon/loader/meta.py` | `ParallelType`, `QuantizationType`, `CheckPointConfig`, `SliceInfo`, `TensorInfo`, `TensorsMeta`, `TensorsContent`, `ShardingMeta`, `ShardingContent`, `ModelMeta`, `ModelContent`, `create_tensor_from_storage`, `dtype_str_to_dtype` |
| `aegaeon/loader/quick_loader.py` | `_patch_olmo_fused_conv_shard_loader`, `managed_parameter_storage`, `ManagedParameter`, `QuickLoader`, `set_default_torch_dtype`, `get_model_architecture`, `create_cuts` |
| `aegaeon/logger.py` | `NewLineFormatter`, `_setup_logger`, `init_logger` |
| `aegaeon/model_downloader.py` | `download_model`, `_get_repo_type`, `_snapshot` |
| `aegaeon/model_placement.py` | `ModelNotReadyError`, `RequestRoutingPolicy`, `LeastOutstandingRequestsPolicy`, `RoundRobinRoutingPolicy`, `create_request_routing_policy`, `RequestRoutingTable`, `NodePlacementMetrics`, `ModelPlacementPolicy`, `LeastLoadedPlacementPolicy`, `RoundRobinPlacementPolicy`, `create_model_placement_policy`, `ModelReplicaState`, `ModelReplicaPlacement`, `ModelPlacementTable` |
| `aegaeon/model_registry.py` | `ModelSpec`, `ModelSpecRegistry`, `_validate_and_assign_ids`, `_compute_params`, `get_registry`, `set_registry` |
| `aegaeon/models.py` | `_ModelNamespace`, `set_model_registry`, `load_config_from_content` |
| `aegaeon/prefill_dispatcher.py` | `PrefillDispatcher` |
| `aegaeon/prefill_scheduler.py` | `PrefillScheduler`, `PrefillStageUniBatchScheduler`, `get_prefill_stage_scheduler` |
| `aegaeon/request.py` | `Request`, `BatchedRequests` |
| `aegaeon/simple_dispatcher.py` | `SimpleDispatcher` |
| `aegaeon/simple_scheduler.py` | `calculate_decode_quotas`, `SimpleScheduler` |
| `aegaeon/stage_engine.py` | `Stage`, `StepOutput`, `StageEngine`, `PrefillEngine`, `DecodeEngine`, `SimpleEngine` |
| `aegaeon/test_api.py` | `send_request`, `main` |
| `aegaeon/utils.py` | `_DeviceNamespace`, `set_device_registry`, `Counter`, `set_random_seed`, `prod`, `get_distributed_init_method`, `get_ip`, `get_open_port`, `make_tensor_with_pad`, `estimate_switch_time`, `reduce_shared_cpu_tensor`, `rebuild_shared_cpu_tensor`, `reduce_cuda_event`, `rebuild_cuda_event`, `get_logits_processor`, `get_lm_head`, `compute_request_metrics`, `compute_request_latencies`, `ensure_infile`, `ensure_outfile`, `get_tokenizer` |
| `aegaeon/vllm_cache.py` | `CacheSegment`, `CompositeCache`, `make_composite_config`, `initialize_composite_cache` |
| `aegaeon/worker.py` | `_guard_vllm_c_rms_norm`, `Worker`, `_init_worker_distributed_environment` |
| `aegaeon/workload.py` | `get_dataset`, `syn_workload`, `save_workload` |
