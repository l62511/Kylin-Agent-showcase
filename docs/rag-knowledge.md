# RAG 与知识治理

## TL;DR

OpsRAG 不是“把文档切块后相似度搜索”。导入阶段保留来源和版本，检索阶段采用受限 Query 规范化、混合召回、独立 Rerank、最多两跳和证据片段绑定，评测时与执行链隔离。

## 数据路径

```text
文档/经验卡片 -> 解析 -> 结构感知切块 -> 文件/文本/结构去重
             -> FTS5 + 向量召回 -> Query rewrite -> 独立 rerank
             -> 证据片段/来源版本 -> 生成回答或交给计划审查
```

## 防止的具体问题

- 只按文本相似度召回会把同一配置的旧版本混在一起；来源版本和结构指纹用于去重与新鲜度判断。
- 无限 Query rewrite 容易把用户意图越改越远；规范化规则限制改写范围，最多两跳。
- RAG 命中不能证明动作成功；回答中的事实必须绑定片段或工具证据，缺证据时进入 needs_review。
- 评测索引与生产索引混用会污染结果；OpsRAG 评测使用隔离索引和固定数据版本。

## 规模与边界

冻结基准的 OpsRAG 任务为 **300 条**、**75 个文档**，按 dev/test 分区；这 300 条是 4,200 条唯一来源用例中的一个任务集合，不代表全部 Agent 任务。

相关数据：[`../evaluation/evaluation-results-v0.6.203.json`](../evaluation/evaluation-results-v0.6.203.json) · 架构：[`architecture.md`](architecture.md)
