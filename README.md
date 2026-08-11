# Kylin Agent — 面向信创运维场景的可控式 AIOps Agent

> 第十五届“中国软件杯”A2 赛道 · 全国三等奖（国三）
> 龙芯 3A5000 · LoongArch64 · 银河麒麟高级服务器 V11

让 Agent 在国产信创环境中**可控、可审计、可回放、可评测**地执行运维任务。

![Kylin Agent 六服务架构](assets/architecture.svg)

本仓库是 Kylin Agent 的 Showcase，不包含源码、模型密钥、生产数据或可直接部署的二进制包。源码暂不公开，是因为项目将作为毕业设计继续迭代；这里公开的是可复核的架构决策、失败边界、评测口径和脱敏结果。

## 当前验证基线

主项目版本：**v0.6.280**（2026-08-09）。
生产执行路径默认使用 `agent-runtime-governance` v0.9.1；原生治理实现仅作为显式 legacy fallback 保留。

- 全量 Python 测试：**2352 passed / 4 skipped**；总覆盖率 **80.58%**，覆盖率门禁为 80%。
- GitHub Actions 自动化链路：Python 3.10/3.11/3.14、Java、前端真实浏览器、评测、打包、Future AGI、LoongArch OTLP 和 Docker 双拓扑 smoke 全部通过。
- Docker 多 Agent smoke 覆盖 RabbitMQ、控制面、两个节点、节点绑定、远程只读执行、监控、RAG 去重和审计导出。
- 供应链摘要按 LF 规范化，Windows 与 Ubuntu 检出同一文本内容时得到一致 SHA-256；RabbitMQ 健康探针不依赖脚本执行位。

## 快速了解

Kylin Agent 把一条自然语言运维请求收敛为：

```text
请求 → 认证/限流 → 意图与计划 → 风险审批 → 有界执行 → 证据审计 → 评测判定
```

它解决的不是“模型能不能调用工具”，而是 Agent 产生真实副作用之后，如何保证：

- 同一请求跨 REST/A2A/SSE 不重复执行；
- worker 崩溃、超时或断线后，状态不会永久卡死或被旧结果覆盖；
- 危险命令、路径混淆、容器标识注入和超大输出在执行边界被拒绝或限界；
- 结果必须绑定工具回执、检索证据或审计记录，安全关键失败可以阻断发布。

## 六服务架构

```text
Browser / Vue Panel
        │ HTTP / SSE
        ▼
Nginx ──► Spring Boot Authentication Gateway
          Sa-Token · RBAC · CSRF · HMAC v2
        │ trusted user context
        ▼
FastAPI Control Plane
  Planning · Approval · Audit · Evaluation
        │ HMAC v2 + Node Token
        ├──────────────► Agent Node A
        └──────────────► Agent Node B
                Collection · Preflight · Execution

Control Plane ── SQLite Outbox ── AMQP ── RabbitMQ durable topic
                                      ├─ publisher confirm
                                      ├─ manual ACK
                                      └─ DLX / DLQ

SQLite WAL / etcd：执行记录、租约、幂等账本、审计链与事件回放
```

可编辑图：[`diagrams/architecture.mmd`](diagrams/architecture.mmd)；安全、消息和评测图位于 [`assets/`](assets/)。

## 四个工程决策

| 常见实现 | Kylin Agent 的机制 | 避免的具体故障 |
| --- | --- | --- |
| REST、A2A、SSE 各自维护执行状态 | `ChatExecutionService` + canonical JSON SHA-256 指纹 + SQLite WAL | 同一 `client_request_id` 产生两次副作用，或一个协议已完成而另一个仍显示 running |
| 用进程内永久锁表示“正在执行” | owner token、租约 TTL、心跳、过期接管与 `unknown_outcome` | worker 崩溃后请求永久卡死；变更任务在结果未知时被错误重放 |
| 每条 SSE 请求创建 daemon thread | 共享 64 worker / 512 queue 的背压池，普通事件合并，终态可靠缓冲 | 100 个并发连接无限增线程，背压时 final/approval/error 事件被挤掉 |
| `capture_output=True` 后再裁剪 | 分块读取 + 有界尾部环形缓冲；单次命令/执行器返回 `truncated` | 100 MiB 持续输出先占满内存，应用来不及执行裁剪 |
| 每次审计校验计算全表摘要 | `audit_chain_state` 尾部元数据、唯一幂等账本、增量后缀校验 | 百万条审计记录下健康检查仍 O(n)，相同 key 的冲突 payload 被静默复用 |
| 文件下载先 Base64 再由浏览器解码 | `FileResponse` / `StreamingResponse` + `Range` + `AbortController` | 20 MiB 文件膨胀到约 26.7 MiB，并同时保留 raw、字符串、Uint8Array 与 Blob |

安全防御的分层图：[`assets/security-layers.svg`](assets/security-layers.svg)。可靠事件时序：[`assets/reliability-sequence.svg`](assets/reliability-sequence.svg)。

## 评测证据：

当前公开结果分为“数据集清单”和“历史运行产物”两种口径：

| 口径 | 数字 | 含义 |
| --- | ---: | --- |
| 当前全量清单 | **4,380 行** | OpsRAG 300 + Agent 单工具投影 180 + Planner/OpsIntent 800 + OpsSafety 1,200 + Evidence Gate 1,600 + Fault Recovery 300 |
| 唯一源案例 | **4,200** | 4,380 行扣除来自 OpsIntent 的 180 条兼容投影 |
| 历史 v0.6.203 可用规模 | **2,480** | Agent 180、Planner 800、RAG 300、Safety 1,200 |
| 历史 v0.6.203 实际运行 | **769** | 680 条 benchmark + 89 条系统/质量用例 |
| 历史 v0.6.203 结果 | **764/769，99.35% raw pass** | 2 个安全关键失败触发 hard gate，最终 score 为 0 |

这意味着 99.35% 不是“全量数据集成绩”，而是整改前受控子集上的阶段性结果；安全关键失败被保留在分母中，没有被转换成 unavailable 或从统计中删除。评测中心当前已经接入 4,380 行（其中 4,200 条唯一来源用例、180 条兼容投影），manifest、split、case identity fingerprint 和 provenance 均可追踪；待主项目全部整改项完成后，再发布对应的本轮全量运行结果。

评测维度固定为：任务成功率、工具调用准确率、结果正确性与一致性、事实有据性、首事件/终态延迟分位数、真实 Token/成本、人工接管混淆矩阵。评测图：[`assets/benchmark.svg`](assets/benchmark.svg)。

公开脱敏结果：

- [`evaluation-results-v0.6.203.json`](evaluation/evaluation-results-v0.6.203.json)：含 hard gate 失败语义与失败分母；
- [`evaluation-results-v0.6.169.json`](evaluation/evaluation-results-v0.6.169.json)：历史基线，255 总用例、252 通过、2 不可用；
- [`tool-effect-summary-v0.6.169.json`](evaluation/tool-effect-summary-v0.6.169.json)：87 个注册工具中 22 个当前主机可执行，22/22 验证通过，不可用和未执行不计为通过；
- [`performance-baseline-v0.6.169.json`](evaluation/performance-baseline-v0.6.169.json)：历史本地 2 用例 P50/P90/P99 和吞吐基线。

## 可验证的工程边界

- 同步执行池默认 **8 workers / 32 queue**；SSE 共享池默认 **64 workers / 512 queue**，容量耗尽显式返回 503，deadline 返回 504。
- 持续 **100 MiB** 命令输出、**20 MiB** SafeExecutor 输出在有界内存内完成，响应标记 `truncated`。
- 文件下载使用 64 KiB 分块，支持 `Accept-Ranges: bytes`、206/416、`Content-Disposition` 和客户端取消。
- 2026-08-08 文件流整改定向回归 **107 passed / 3 skipped**；2026-08-09 任务书完成后的全量项目测试为 **2352 passed / 4 skipped**。

## 技术栈与适配

Python · FastAPI · LangGraph · Spring Boot · Vue · RabbitMQ · SQLite WAL / FTS5 · FAISS · Docling · ONNX · Docker · GitHub Actions

目标平台为 LoongArch64。文档处理采用 ONNX-only 路径（Layout Heron、RapidOCR），二进制依赖优先使用 Loongnix 官方 wheel；镜像以 content ID + SHA-256 联合校验。

## 阅读路径

| 想回答的问题 | 文档 |
| --- | --- |
| 六服务如何分工，数据怎么流？ | [`docs/architecture.md`](docs/architecture.md) |
| Agent 为什么不会直接裸奔执行？ | [`docs/security-layers.md`](docs/security-layers.md) |
| 消息为什么不会因重启丢失？ | [`docs/reliability.md`](docs/reliability.md) |
| 文档如何进入混合检索和 RCA？ | [`docs/rag-knowledge.md`](docs/rag-knowledge.md) |
| 如何重放一次执行并定位问题？ | [`docs/trace-replay.md`](docs/trace-replay.md) |
| 评测分母、失败和 gate 如何解释？ | [`docs/evaluation.md`](docs/evaluation.md) |
| 龙芯/麒麟环境的限制是什么？ | [`docs/loongarch-adaptation.md`](docs/loongarch-adaptation.md) |

## 从 Kylin Agent 到通用治理 SDK

在项目中反复遇到的审批一致性、副作用幂等、未知结果对账和证据链问题，被进一步抽象为框架无关的 [Agent Runtime Governance SDK](https://github.com/Success6666/agent-runtime-governance)。这条路径体现的是：真实系统故障 → 提炼跨框架契约 → 独立运行时治理组件，而不是简单复制项目代码。

当前主路径由 SDK 负责 action binding、Schema 校验、审批决策、持久化幂等、UNKNOWN 结果和 JSONL 审计；Kylin Agent Adapter 负责业务注册表、节点上下文和公开结果映射。专项测试覆盖安全决策透传、预演/正式执行幂等键隔离、内部参数隔离、Schema 拒绝、审计脱敏和 legacy fallback；Graph、MCP、ReAct 通用回归验证 SDK 接入后的业务语义。SDK 本身未发现缺陷，项目没有复制修补 SDK 代码。

## 源码边界

本仓库明确是 Showcase only。Kylin Agent 是第十五届“中国软件杯”全国三等奖项目，源码不在此仓库发布；后续将作为毕业设计持续迭代，待研究内容、授权边界和部署资产稳定后再决定开放范围。仓库只发布架构图、设计决策、脱敏评测 JSON 和可复核的限制说明。

演示入口：[`docs/demo.md`](docs/demo.md)。
