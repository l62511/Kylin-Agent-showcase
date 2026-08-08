# LoongArch64 / 麒麟适配

## TL;DR

目标部署环境是龙芯 3A5000、LoongArch64 和银河麒麟 V11。适配策略是把架构相关依赖集中在文档解析、OCR、推理和镜像层，业务治理逻辑保持平台无关，并用 wheel、镜像 content ID 和 SHA-256 记录可复现边界。

## 关键取舍

- 文档解析采用 **ONNX-only** 路径，组合 Layout Heron 与 RapidOCR，避免依赖在 LoongArch64 上不可用的闭源推理运行时。
- 二进制依赖优先使用 Loongnix 官方 wheel；无法安装时在构建阶段显式失败，不在运行时静默降级到不兼容实现。
- 节点与控制面通过能力清单协商；缺失能力进入 unavailable/needs_review，不让模型猜测“应该能执行”。
- 镜像以 content ID + SHA-256 联合校验，防止同名标签覆盖导致节点执行了错误版本。

## 验证边界

展示仓库不包含镜像、wheel、模型权重或节点凭据。适配结论应在目标 LoongArch64/麒麟 V11 环境中复测；本仓库只保留架构决策、校验字段和可审计的结果摘要。
