# 展示与答辩演示脚本

## TL;DR

演示目标不是“模型能调用几个工具”，而是让评审看到一次请求如何经过身份、计划、审批、限界执行、证据回传和评测判定，并能在断线、重试和危险输入下解释系统行为。

## 演示准备

- 浏览器打开控制台，准备一个只读请求和一个需要审批的变更请求。
- 准备两个节点身份（Node A/Node B），确保 Gateway 到 Control Plane、Control Plane 到节点的 HMAC v2 校验开启。
- 打开审计、事件流和评测面板；展示仓库中脱敏 JSON，不展示生产地址、凭据或源码。

## 五段演示路径

1. **请求闭环**：提交只读请求，依次展示 request fingerprint、计划、工具契约、证据 ID 和终态。重复提交同一 `client_request_id`，应复用既有结果，不产生第二次副作用。
2. **审批边界**：提交变更请求。先展示逐动作审批和目标节点绑定，再拒绝其中一个动作；最终状态必须是 blocked/incomplete，而不是模型口头的“已完成”。
3. **流式断线**：在阶段事件产生时刷新 SSE。使用 `Last-Event-ID` 恢复，普通日志可合并，终态、审批、错误和取消事件仍可回放。
4. **容量与输出**：并发超过同步池 **8 workers / 32 queue** 或 SSE 池 **64 workers / 512 queue**；应看到 503/504。构造大日志时观察 **100 MiB** 命令上限、**20 MiB** SafeExecutor 上限和 `truncated=true`。
5. **评测解释**：展示 4,200 条唯一来源用例、4,380 行加载视图（含 180 条 Agent 兼容投影）以及历史 0.6.203 结果。说明“数据已接入”与“本轮是否已执行”分别由 manifest 和 result provenance 证明。

## 预期观察

| 场景 | 应看到的工程证据 |
| --- | --- |
| 重试/双协议 | 相同 fingerprint、单一执行记录、统一终态 |
| 断线重连 | 单调 event sequence、`Last-Event-ID`、终态不丢失 |
| 危险输入 | 执行前拒绝原因、无副作用审计记录 |
| 大输出/大文件 | 分块读取、截断标志、Range/取消能力 |
| 评测 | 数据集 lineage、case identity、分母、hard gate 和运行 provenance |

详细机制见 [`architecture.md`](architecture.md)、[`security-layers.md`](security-layers.md)、[`reliability.md`](reliability.md)。
