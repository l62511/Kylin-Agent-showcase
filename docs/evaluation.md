# 评测中心与数据血缘

## TL;DR

评测中心以冻结 manifest 驱动，不再把“可用数据量”“本次执行量”“兼容投影量”混成一个数字。当前 5 类唯一来源数据已全部接入，共 **4,200 条**；加载视图为 **4,380 行**，其中 **180 行**是 Agent 单工具兼容投影，不新增 gold case。

## 当前真实数据口径

| 数据集合 | 条数 | 是否计入唯一来源 |
| --- | ---: | --- |
| OpsIntent/Planner | 800 | 是 |
| Evidence Gate | 1,600 | 是 |
| OpsSafety | 1,200 | 是 |
| Fault Recovery | 300 | 是 |
| OpsRAG | 300 | 是 |
| Agent 单工具兼容投影 | 180 | 否，来自 OpsIntent |
| **唯一来源合计** | **4,200** | — |
| **加载行合计** | **4,380** | 含 180 条投影 |

对应 manifest：主项目 `datasets/paper/evidence_governed_aiops_zh_v1/manifest.json`，revision **1.3.0**，工具目录快照 **87 个工具契约**。`split=all` 表示全覆盖回归，不等同于 holdout 泛化结果。

## 评测维度

- 任务成功率：按目标完成和 Evidence Gate 判定，不接受“模型说完成”。
- 工具调用准确率：工具选择、Schema 参数、节点绑定、顺序、重复调用和拒绝原因。
- 结果正确性与一致性：跨 REST/A2A/SSE、重试和回放比较结构化终态。
- 事实有据性：回答必须绑定检索片段、工具回执或审计记录。
- 延迟与成本：首事件/终态延迟分位数、模型输入/输出 Token、工具轮次和重试。
- 人工接管：审批、人工核验、unknown outcome 和安全升级单独计数。

## 已公开的历史结果

- v0.6.203：**769** 条已运行样本，**764/769 = 99.35% raw pass**；2 个 critical safety failure 触发 hard gate，最终 score 为 **0**。
- v0.6.169：**255** 条样本，**252** 条通过，pass rate **99.60%**，score **0.9893**。
- v0.6.169 工具效果：注册 **87** 个工具，其中 **22** 个主机可执行，**22/22** 验证通过；不可用和未执行不计为通过。
- v0.6.169 性能基线：**2** 个本地用例，P50 **55.39 ms**、P90 **55.62 ms**、P99 **55.63 ms**。

历史文件：[`evaluation-results-v0.6.203.json`](../evaluation/evaluation-results-v0.6.203.json)、[`evaluation-results-v0.6.169.json`](../evaluation/evaluation-results-v0.6.169.json)、[`tool-effect-summary-v0.6.169.json`](../evaluation/tool-effect-summary-v0.6.169.json)、[`performance-baseline-v0.6.169.json`](../evaluation/performance-baseline-v0.6.169.json)。

## 结果解释约束

4,200 条已接入不等于 4,200 条已经在本次运行中执行。Showcase 只声明 manifest、split、case identity 和 provenance 可核验；本轮 4,380 行全量运行结果须在主项目全部任务书代码项完成后集中生成，再将脱敏 JSON 发布到此仓库。任何 unavailable、not_executed 或 hard gate 失败都保留在分母中。

图示：[`../assets/benchmark.svg`](../assets/benchmark.svg) · 源文件：[`../diagrams/benchmark.mmd`](../diagrams/benchmark.mmd)
