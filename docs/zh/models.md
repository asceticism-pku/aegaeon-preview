# 模型准备与注册

## 本地 checkpoint

请将模型下载到各节点一致的绝对路径，并显式配置 `path`。目录应包含完整 config、tokenizer 文件、模型所需的 chat template / generation config，以及全部 safetensors 或其他受加载器支持的权重分片。

模型名用于客户端请求，profile 别名用于性能数据，path 用于真实权重。例如：

```yaml
models:
  - name: Qwen/Qwen3-4B
    path: /srv/models/Qwen3-4B
    profile: qwen3_4b
    max_model_len: 4096
```

## 下载来源

`model_downloader.py` 默认使用 ModelScope；可选择 Hugging Face：

```bash
export AEGAEON_DEFAULT_MODEL_REPO=modelscope
# 或者
export AEGAEON_DEFAULT_MODEL_REPO=huggingface
```

下载实现覆盖以上两个值。代码先探测 `~/.cache/modelscope/hub/<name>` 或 `~/.cache/huggingface/hub/<name>`，再调用对应 SDK 的 snapshot_download。离线或多节点部署建议配置明确的 path，避免依赖 SDK 缓存布局。

主服务会向 actors 传递部分 MODEL_ROOT/PROXY 环境变量，当前下载函数只使用下载 SDK 识别的标准设置。网络代理请配置 `HTTP_PROXY` 和 `HTTPS_PROXY`，离线部署直接填写本地 `path`。

## 下载与代理排障

已有完整本地 checkpoint 时无需下载。Hugging Face 下载失败时，可先检查连通性；需要代理时在下载命令所在 shell 设置实际可用的地址：

```bash
export HTTPS_PROXY=http://127.0.0.1:7890
export HTTP_PROXY="$HTTPS_PROXY"
curl -fsS --max-time 20 https://huggingface.co/api/models/Qwen/Qwen3-4B >/dev/null
```

代理地址仅为示例，请按环境替换；可直接访问时省略代理设置。若代理环境中 Xet 权重下载超时，可设置 `export HF_HUB_DISABLE_XET=1` 改用标准 HTTP 下载。`hf download` 因网络中断退出时，重新执行相同命令和 `--local-dir` 继续下载，完成后再启动服务。

## 模型配置

当前 ModelConfig 固定使用 bf16（每参数按 2 bytes 估计）、`trust_remote_code=True`、`model_impl='auto'`、`quantization=None`、`revision=None`，并向 vLLM 设置 `enforce_eager=True`。YAML 暴露的模型选项以配置参考表为准，其中省略 dtype、量化和 revision。

省略 `max_model_len` 时，系统从模型 config 的候选长度字段中取最小已知值；候选为空时抛出异常。override 只设置服务侧上限，模型的 RoPE 和训练上下文能力仍由权重与模型配置决定。

## EOS 与聊天模板

EOS 集合合并 hf_config、text_config 和 generation_config 中的 EOS ID，兼顾多结束 token 模型。用短问答检查是否正常停止，尤其含 `<end_of_turn>` 等额外结束符的模型。

Chat 使用 tokenizer 的 apply_chat_template。当前 API 对归一化名称含 qwen3 的模型固定传入 `enable_thinking=False`，默认生成直接回答。请求 schema 当前省略通用的 chat_template_kwargs 和 enable_thinking 字段。

## profile 和参数估算

显式 profile 推荐保持稳定且使用安全相对别名。未设置时，`Qwen/Qwen3.5-9B` 可规范化为 `qwen3_5_9b`。绝对路径或 `..` 路径片段会被拒绝。

参数量自动计算会读取权重 metadata；读取失败时再手工填写 params，并按 [2^30 换算](configuration.md)。自动 CPU 权重缓存估算为参数量 × 2 × 1.2；实际 GPU 峰值还需计入加载与运行时开销。

## 新文本模型验收

1. TP=1、图关闭，单模型短文本生成。
2. 验证 tokenizer/chat template、EOS、停止参数和上下文限制。
3. 采集目标设备 profile，确认调度估计能使用。
4. 两模型串行交替，比较切换前后的 token 输出和缓存释放。
5. 按目标用途检查并发、P/D、多副本、sliding window、MLA 或 Mamba 状态。
6. 使用 Foundry 时完成 SAVE/LOAD、模型切换和 token ID 对比。

六项检查全部通过后，模型才能进入该部署环境的可用清单。当前验收清单面向文本模型。
