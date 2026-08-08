# 可靠性与故障恢复

## TL;DR

系统把“消息已发出”“副作用已发生”“结果已确认”拆成不同状态。SQLite Outbox 解决提交与发布之间的窗口，RabbitMQ confirm/手动 ACK/DLX-DLQ 解决投递与消费失败，幂等账本和 `unknown_outcome` 解决重试歧义。

## 状态机

```text
request accepted -> planned -> approval_pending -> dispatched
       |                |              |                |
       +-------------- cancelled      +-- rejected       +-- running
                                                          |
                                  succeeded / failed / unknown_outcome
```

## 关键机制

- **Outbox**：业务状态和待发布事件在同一 SQLite 事务中提交；进程在 commit 后、publish 前崩溃时，恢复任务仍能从 Outbox 发布。
- **RabbitMQ**：durable topic、publisher confirm、manual ACK；消费者处理失败进入重试/死信链路，不在内存中静默丢弃。
- **幂等**：`client_request_id + canonical fingerprint` 识别相同请求；执行账本记录 owner token、租约、结果摘要和副作用阶段。
- **未知结果**：节点执行后连接断开或超时，状态进入 `unknown_outcome`，要求人工核验或证据补偿后才允许后续动作。
- **事件回放**：事件带单调序号；普通阶段事件可合并，终态/审批/错误/取消事件进入可靠缓冲，客户端按 `Last-Event-ID` 补读。

## 失败场景与处理

| 失败窗口 | 错误做法 | Kylin Agent 行为 |
| --- | --- | --- |
| commit 后 publish 前崩溃 | 状态已变更但消息不存在 | Outbox 重启扫描并补发 |
| consumer 处理后 ACK 前崩溃 | 重复投递造成重复副作用 | 消费幂等键 + 执行账本拒绝第二次副作用 |
| 副作用后回执丢失 | 直接重试写操作 | 标记 `unknown_outcome`，进入人工核验 |
| SSE 背压 | 让所有事件争抢同一队列 | 可靠事件单独保留并暴露 dropped/coalesced |

图示：[`../assets/reliability-sequence.svg`](../assets/reliability-sequence.svg) · 源文件：[`../diagrams/reliability.mmd`](../diagrams/reliability.mmd)
