# 开发与贡献指南

## 项目布局

| 目录 | 内容 |
|---|---|
| aegaeon/ | 核心系统、HTTP、配置、路由、调度、Worker与缓存 |
| ops/ | KV搬运C++/CUDA扩展 |
| quick_model_loader/ | Rust/Python快速权重加载组件 |
| foundry/ | 图序列化与恢复子项目及独立引擎集成 |
| profiles/ | 采集脚本与性能数据 |
| benchmark/、tools/ | 基准与专项测试工具；每个脚本的参数和环境要求见对应说明 |
| tests/ | 控制逻辑与GPU相关测试 |
| sim/ | SimPy策略仿真 |
| plots/、workloads/ | 绘图与负载资料 |
| web/ | 内置页面 |
| docs/zh/、docs/en/ | 中英文文档及配套示例 |

主项目LICENSE为Apache-2.0；新增组件与依赖需同时保留各自许可信息。Foundry 的独立集成说明见其子项目文档。

## 开发新功能

配置字段需要贯通解析→配置类→API创建→Ray env / Controller→Engine/Worker。新增功能应同时覆盖成功路径、失败路径、作用域、默认值和限制。

模型与设备通过 registry/YAML 扩展；`ModelType` 和 `DeviceType` 已不再使用枚举。新增 KV 类型时同步实现描述、block 数量、搬运布局、sliding window 和状态复原测试。

## 验证层次

1. 无GPU的纯逻辑检查：图key选择、catalog路径、配置schema、策略决策、缓存映射计算。
2. 带mock控制面测试：模型部署、reservation释放、busy拒绝、部分失败与SSE清理。
3. 单模型真实GPU：加载、EOS、上下文、正确输出与退出。
4. 多模型切换与并发：缓存搬运、请求守恒、输出一致、资源回落。
5. 目标拓扑专项：P/D、多节点、Work Stealing 和 Foundry。当前发布范围固定 TP=1，且只支持文本输入。

`tests/` 同时包含纯逻辑测试和依赖真实模型、CUDA 的测试。请按测试文件的依赖选择测试集。

```bash
python -m pytest tests/test_cuda_graph_catalog_config.py
```

该测试模块检查图存档目录规则，运行环境应安装 pytest 并包含正确的项目导入路径。新增运行逻辑应覆盖真实失败分支和边界条件。

## 文档维护

中英文文档使用 MkDocs Material 构建，配置文件分别为 `mkdocs.zh.yml` 和 `mkdocs.en.yml`。在独立文档环境安装依赖后运行：

```bash
python -m pip install -r docs/requirements-docs.txt
python -m mkdocs serve -f mkdocs.zh.yml
python -m mkdocs build --strict -f mkdocs.zh.yml
```

英文站点使用 `python -m mkdocs serve -f mkdocs.en.yml` 或 `python -m mkdocs build --strict -f mkdocs.en.yml`。发布时将两份构建结果放在同级 `zh/`、`en/` 目录，语言菜单即可跳转到对应文章。

新增或修改参数时，同步更新两种语言的配置参考、CLI、HTTP 请求字段和示例。提交文档前运行 `python docs/check_docs.py`，并完成严格构建。

生成支持中英文切换的离线文档页面：

```bash
python docs/build_portable.py --output /tmp/aegaeon-docs.html
```

同时更新服务内置的文档页面：

```bash
python docs/build_portable.py --output /tmp/aegaeon-docs.html --web-dir web
```

## 提交内容

Pull Request 应说明用户可见行为、默认值、兼容性与失败处理，并附相关测试。文档更新应包含适用前提清楚、参数准确的示例。
