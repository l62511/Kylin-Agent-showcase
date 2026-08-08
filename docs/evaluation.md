# 评测中心：从“有分数”到“能解释”

## 公开的脱敏结果

本目录中的 JSON 是从项目评测产物提炼出的可公开摘要，保留运行 ID、版本、数据规模、section、分母、失败数、hard gate 和解释字段；原始 prompt、业务文档、节点地址、完整工具返回和完整数据集不发布。

| 文件 | 版本/日期 | 关键结果 |
| --- | --- | --- |
| `evaluation-results-v0.6.169.json` | 2026-07-15 | 255 总用例，252 通过，1 失败，2 不可用，pass rate 99.60%，score 0.9893，hard gate 通过 |
| `evaluation-results-v0.6.203.json` | 2026-07-28 | 769 总用例，764 通过，5 失败，2 个安全关键失败，raw pass rate 99.35%，hard gate 失败，最终 score 0 |
| `tool-effect-summary-v0.6.169.json` | 2026-07-15 | 87 个注册工具；22 个当前主机可执行且 22/22 通过；28 不可用、37 未执行，均不计为通过 |
| `performance-baseline-v0.6.169.json` | 2026-07-14 | 2 个本地性能用例；平均 P50 55.39ms、P90 55.62ms、P99 55.63ms |

## 为什么 2,480 条没有全部进入旧结果

2026-07-28 的产物记录了 4 个 benchmark 的可用规模：Agent 180、Planner 800、RAG 300、Safety 1,200，合计 2,480 条。但当时评测运行配置只选择了 Agent 40、Planner 160、RAG 240、Safety 240，共 680 条 benchmark 用例；再加上 baseline、chunking、审批、性能等系统 section，最终总数是 769。

因此：

- `available_records` 不是 `evaluated_records`，不能把数据集规模直接当作评测样本量。
- 旧结果的 99.35% 只能解释为“该次选择集上的 raw pass rate”。
- Safety section 的 5 个失败中有 2 个被标记为 critical，hard gate 将最终 score 置为 0；这比把失败样本排除后给出高分更可信。
- 全量评测重构的验收标准是 manifest 明确记录每个 dataset 的总数、split、选择规则和排除原因，并让 2,480 条全部进入可追踪评测或明确记录为合法排除。

## 维度定义

### 任务成功率

以任务目标断言为准：例如文件状态、容器状态、审计记录和回滚证据都满足，才算成功。HTTP 200、模型输出“已完成”或工具返回非空不能单独构成成功。

### 工具调用准确率

至少拆成 tool selection、参数 schema、参数完整性、调用顺序、重复调用和拒绝原因；对于需要用户补充字段的请求，澄清安全率与执行通过率分开统计。

### 正确性与一致性

将同一 `trace_id` 的重复提交、跨执行实例、SSE 断线重连和 A2A/REST 双入口视为同一业务事实，比较规范化结果、事件序列和终态，不按文本相似度去重。

### 事实有据性

每个关键断言都要绑定检索片段、工具原始结果或审计记录 ID；无法建立证据链的回答应进入 unsupported/needs_review，而不是自动通过。

### 延迟、Token 与人工接管

每次运行记录首事件、首 token、工具完成、终态的时间戳；模型输入/输出 token、工具轮次和重试；以及主动审批、风险升级、超时接管和异常兜底的原因与耗时。

## 结果解释约束

报告必须同时展示：

1. 分子：passed、failed、critical failures。
2. 分母：total、unavailable、not_executed，以及它们是否计入 gate。
3. provenance：数据集指纹、结果 schema、代码版本、运行模式、模型调用开关。
4. 原始失败索引：至少保留内部可回放的 case ID 和失败原因摘要。

展示仓库只公开摘要；完整失败样本和数据集继续保留在主项目及毕业设计迭代环境中。
