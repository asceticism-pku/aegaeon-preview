# 开发与贡献

## 项目布局

| 目录 | 内容 |
|---|---|
| `aegaeon/` | API、路由、调度、Worker 与缓存 |
| `ops/` | C++/CUDA KV 搬运 |
| `quick_model_loader/` | Rust/Python 权重加载 |
| `foundry/` | CUDA Graph 持久化 |
| `profiles/`、`benchmark/`、`tools/` | Profiling 与性能工具 |
| `sim/` | CPU 调度模拟器 |
| `tests/` | 逻辑测试与 GPU 测试 |
| `docs/en/`、`docs/zh/` | 文档与示例 |

## 贡献

通过 issue 反馈问题或提出功能建议。PR 应说明行为变化并包含相关测试，配置、CLI、API 和中英文文档保持一致。

在仓库根目录运行相关测试：

```bash
python -m pytest tests/test_cuda_graph_catalog_config.py
```

GPU 测试需要各测试指定的模型和设备。源码入口见[系统架构](architecture.md)。

## 文档维护

```bash
python -m pip install -r docs/requirements-docs.txt
python docs/check_docs.py
python -m mkdocs build --strict -f mkdocs.en.yml
python -m mkdocs build --strict -f mkdocs.zh.yml
python docs/build_portable.py --output /tmp/aegaeon-docs.html --web-dir build/docs
```

使用 `mkdocs serve` 并指定配置文件可本地预览。发布时将输出放在同级的 `en/`、`zh/` 目录。项目使用 Apache-2.0，保留各组件及依赖的许可证。
