# 运行 Trace Replay

Replay 从请求 trace 恢复会话、Agent、请求依赖、间隔和 token 目标，向后端发送请求并记录性能证据。
它不启动原始 Agent，不执行工具或 SWE-bench 正确性测试。合成提示词回放不代表原始文本或性能完全等价。

## 准备后端

安装项目后，启动支持目标模型的 vLLM 服务。后端需要提供 `/tokenize`、`/detokenize`、`/metrics` 和配置的推理端点。
通过 Router 访问时，在 YAML 中设置 `backend.tokenizer_base_url` 为分词服务地址，
`backend.metrics_url` 为完整的 vLLM 指标 URL。使用 `/v1/messages` 时，启动 vLLM 时加载
`agentinfer.agentcache.core.api_adapter.AgentCacheIdentityMiddleware`，以传递身份和采样参数。
具体服务参数见[接入 vLLM](integrate-vllm.md)。

## 回放内置样例

在仓库根目录执行，使用实际后端模型名替换 `MODEL_NAME`：

```bash
vllm bench serve --agentinfer replay \
  --config agentinfer/agentbench/configs/replay_benchmark.yaml \
  --trace-path tests/agentbench/replay/claude_trace_8session_requests.jsonl \
  --base-url http://127.0.0.1:8000 \
  --model MODEL_NAME --task-num 1 \
  --result-dir results/replay-smoke
```

结果目录必须不存在。CLI 路径相对于当前目录，YAML 路径相对于配置文件目录。
默认配置包含用于样例的合成前缀预算；换用其他录制源时需重新标定，后端上下文长度须容纳请求目标。

## 回放 TraceLab

输入每行表示一次 LLM round，包含 `provider`、`session_id`、`round_index`、
`input_tokens_total`、`prefix_tokens`、`newly_append_tokens`、`output_tokens`、`timing_events` 和 `tools`。
Converter 原样保留源 token 统计，按 Provider、Session 分组并按 round 排序，生成 token 配方 IR。
输入/输出目标直接使用记录的总量，前缀与新增 token 数作为来源证据保留，不校验两者之和。
工具元数据只用于来源审计，不会实际执行。

先将示例 YAML 复制到 `/path/to/tracelab.yaml`，并设置 `replay.prompt_calibration_tolerance_tokens: 0`，
然后使用以下 CLI 参数覆盖输入源和模型：

```bash
vllm bench serve --agentinfer replay \
  --config /path/to/tracelab.yaml \
  --trace-type tracelab --trace-path /path/to/round_trace.jsonl \
  --prompt-shape tracelab_synthetic --endpoint /v1/chat/completions \
  --base-url http://127.0.0.1:8000 --model MODEL_NAME \
  --task-num 8 --max-concurrency 4 \
  --result-dir results/replay-tracelab-8x4
```

保留完整输入/输出时，配置 `max_input_tokens: null`、`max_output_tokens: null`；
`prompt_calibration_tolerance_tokens` 必须为 `0`。支持 `trace` 和 `lognormal` 间隔模式。
`task_num` 表示采样后的 Runtime Session 数，`max_concurrency` 限制同时运行的 Session 数。
重复采样使用不同的 Runtime Session ID 和私有合成内容。

首轮发送一条合成 `user` 消息。续接轮通过 `context_after` 保留前一轮消息，追加 Backend 实时
Assistant 回答，再增加新的合成 `user` 消息；`send_after` 仍控制前驱完成后加 interval 的释放时间。
TraceLab 复用现有图执行器，要求上下文前驱响应成功；请求失败或 Prompt 构造失败时，后续上下文依赖请求会跳过。

后端 tokenizer 对包含 Chat Template 的完整对话计数，校准通过调整合成 user 文本满足输入目标，
不会截断实时 Assistant 回答。strict 模式下历史上下文超出目标会失败；adaptive 模式允许有限裁剪历史
合成 user 文本，但 TraceLab 不会为了满足目标重置或静默丢弃 Assistant 历史。无法校准时请求失败。
缺少 SSE `[DONE]`、usage 缺失或实际输入/输出 token 数与计划不同，同样记为请求失败。

源 `prefix_tokens` 保留为审计证据，不再作为精确 LCP 目标：保留实时对话历史可能产生不同于源 trace 的前缀。
不再生成固定前缀模板 profile 和精确 LCP 指标。缓存 usage 缺失仍表示不可用，首轮/续接轮缓存统计依据 `context_after` 分组。

TraceLab IR 使用 `requests.jsonl` 和 `manifest.json`，不生成文本 sidecar。
显式分析复用 `unified_trace_ir.py`，原 Analyzer 和 Inferact 文本 IR 路径保持兼容。
旧 TraceLab IR 和计划需要重新生成：`context_after` 替代 `input_after`。
转换器、Planner、采样器和时间模型标识不再附带版本后缀；数据格式校验仍保留 `schema_version`。
标识和哈希材料变化会改变 workload fingerprint、Runtime ID 和 seed。
历史前缀复制模式的运行不能视为相同工作负载。

## 输入与配置

旧配置中的 `claude_code_minimal_v1`、`trace_record`、`token_recipe` 应分别迁移为
`agentinfer_synthetic`、`inferact_synthetic`、`tracelab_synthetic`，不提供旧名称别名。
名称参与 workload fingerprint，迁移后不应按相同 workload ID 比较。
IR 的 `prompt_source.kind=token_recipe` 保持不变，token 目标用于构造合成 user 轮次并续接实时 Assistant 上下文。
Inferact 仍保留源 Human 文本及实时 Assistant 历史；超出校准容差记录到
`trace-record-validation.json`，不会重写文本或因此终止 Session。

| 配置 | 含义 |
| --- | --- |
| `replay.trace_type: agentinfer` | 输入为 AgentInfer `requests.jsonl`；使用默认 `prompt_shape: agentinfer_synthetic`。 |
| `replay.trace_type: inferact_codex_swebenchpro` | 输入为 Inferact 原始 JSON；要求 `prompt_shape: inferact_synthetic`、`interval_mode: lognormal` 和 `/v1/chat/completions`。 |
| `replay.trace_type: tracelab` | 归一化、未压缩的 JSONL；要求 `prompt_shape: tracelab_synthetic`、零校准容差和 `/v1/chat/completions`。 |
| `replay.trace_type: agentX` | `prompt_shape: agentX_synthetic` 为预留入口，执行时抛出 `NotImplementedError`。 |
| `replay.interval_mode` | `trace` 保留历史间隔，`lognormal` 按配置的分布生成间隔。 |
| `replay.sample_seed` | 可重复的会话抽样和后端采样种子。 |
| `replay.max_input_tokens` / `max_output_tokens` | 可选 token 目标上限；`null` 保留 trace 目标。 |
| `replay.context_adjustment_mode` | `strict` 拒绝非追加上下文；`adaptive` 审计裁剪和上下文重置。 |
| `replay.request_timeout_seconds` | 单请求超时。 |

完整字段、默认值和约束见[Replay 配置模型](../../../agentinfer/agentbench/replay/config.py)与
[示例 YAML](../../../agentinfer/agentbench/configs/replay_benchmark.yaml)。运行
`vllm bench serve --agentinfer replay --help` 查看支持的 CLI 覆盖参数。

## 检查和比较结果

检查 `manifest.json` 的状态和 `summary.json`，并保留 `requests.jsonl`、`replay-source-analysis.json`、
`replay-plan.json`、`replay-execution.json` 和 `evidence/`。失败时检查 `replay-error.json`；
中途失败时部分产物可能不存在。Inferact 转换产物保存在结果目录的 `convert_result/` 下。

```bash
vllm bench serve --agentinfer compare \
  --baseline results/replay-baseline --candidate results/replay-candidate
```

公平比较要求使用相同 trace、抽样种子、前缀预算、并发和部署参数。每次冷启动比较前重启服务，
并检查起始 Prefix Cache 指标。Replay 不提供任务正确性结论，详见[基准方法](../explanation/benchmark-methodology.md)。
