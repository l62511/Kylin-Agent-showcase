# 四层安全防线

## TL;DR

安全边界不依赖模型“自觉”。输入层先拒绝明显绕过，规划层只允许注册工具契约，执行层再次校验参数/节点/审批/幂等，审计层保留哈希链和可追溯证据。

## 1. 输入层：拦截看起来合法的绕过

- 请求体按声明长度和 ASGI 分块累计，超过边界返回 413；避免攻击者用超大 JSON 占满解析内存。
- 路径先 `resolve`、再检查允许根目录；`..`、编码斜杠和符号链接不能把读取范围带出工作区。
- Docker 容器标识只接受受限字符集和 **128 字符**以内长度；避免标签注入导致访问非目标容器。
- SQL 表名先与已发现对象集合匹配，再生成受控标识符；避免把表名变成注入片段。

## 2. 规划层：工具契约优先于模型输出

规划器只能从注册表选择工具，参数必须通过 Schema、节点能力和风险等级校验。缺少必填字段的请求进入澄清，不会被“猜一个默认值”直接执行；高风险动作被标记为逐动作审批。

## 3. 执行层：最后一道硬门禁

统一执行服务在真正产生副作用前再次检查 canonical JSON 指纹、工具版本、目标节点、审批摘要和 owner lease。命令、Docker、文件和数据库工具分别设置超时、输出上限和白名单。未知结果保持 `unknown_outcome`，不会把网络超时当成“失败可安全重试”。

## 4. 审计层：证据链而不是日志堆

执行记录、工具回执、审批和评测结果绑定 `trace_id`/`request_id`/`event_id`。SQLite WAL 保存持久状态，Outbox 在事务内落盘，再由消息系统发布；审计哈希链支持增量尾部校验。敏感字段（Password、Token、API key、Cookie、Authorization）在展示 DTO 前统一脱敏。

## 可验证证据

- 命令/容器输出边界：**100 MiB** 与 **20 MiB** 两档，结果带 `truncated`。
- SSE 共享池：**64 workers / 512 queue**，可靠事件与普通事件分离。
- 文件下载：**64 KiB** 分块，支持 Range 和客户端 AbortController。
- 文件流改造 targeted suite：**107 passed / 3 skipped**。

图示：[`../assets/security-layers.svg`](../assets/security-layers.svg) · 可编辑源：[`../diagrams/security-layers.mmd`](../diagrams/security-layers.mmd)
