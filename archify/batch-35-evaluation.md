# Batch 35 · Evaluation / Benchmark

## Findings

**Confirmed**：packages/evals 实际存在，包含 vitest-evals、docs variants、Docker 执行、报告与对比。smoke 检查 Paris 和 usage；extensions docs eval 要求创建扩展、reload、调用 hello 并校验结构化输出/工具参数；documentation audit 逐页调查后提交结构化 verdict。不能将此归为“Not Present”。

本轮发现的评测面向产品文档使用/配置/扩展及基本模型行为；未由这些源码确认 SWE-bench 或通用大规模编码修复排行榜。

## Evidence

- [harness.ts](../packages/evals/src/harness.ts)：createPiCodingAgentHarness / runPiCodingAgent。
- [smoke.eval.ts](../packages/evals/evals/smoke.eval.ts)：expected answer。
- [extensions.docs.eval.ts](../packages/evals/evals/extensions.docs.eval.ts)：ToolCallJudge 与 StructuredOutputJudge。
- [documentation-audit.eval.ts](../packages/evals/evals/documentation-audit.eval.ts)：submit_documentation_audit。
- [package.json](../packages/evals/package.json)：eval:host / eval:docs。

## Key Source Files

[packages/evals/src/harness.ts](../packages/evals/src/harness.ts)、[packages/evals/evals/smoke.eval.ts](../packages/evals/evals/smoke.eval.ts)、[packages/evals/evals/extensions.docs.eval.ts](../packages/evals/evals/extensions.docs.eval.ts)、[packages/evals/evals/documentation-audit.eval.ts](../packages/evals/evals/documentation-audit.eval.ts)、[packages/evals/package.json](../packages/evals/package.json)。

## Key Types / Classes

Harness、PiCodingAgentInput、DocumentationVariant、TranscriptEvent、UsageSummary、ToolCallJudge。

## Key Functions

resolveModelSelection；runPiCodingAgent；createPiDocumentationEvalHarness；verifySystemPrompt；describeEval。

## Control Flow

选择 provider/model → 隔离 cwd/home/credentials → 构造 Session → 多 prompt/reload steps → 收集 response/events/usage → judge/assertions → 保存 session snapshot → cleanup。docs variants 要求容器非特权工具用户边界。

## Data Flow

真实会话轨迹、system prompt hash、token/cache/cost、artifact snapshot 与结果；价格缺失时不报告估计美元。失败可附 partial harness run。

## State

fixture workspace、模型选择、隔离环境恢复、run diagnostics、with_docs/without_docs variants。

## Design Decisions

**Inference**：评测可比较文档提示是否帮助实际任务，但有限 fixtures 的通过不等同任意编码任务可靠。

## Unknowns

未运行付费模型、Docker 或 judge；没有报告分数、通过率或成本实测，也未审计所有 eval cases。

## Archify Diagram

本 Batch 不要求独立图；参考相邻机制图，避免把不存在的能力画成实现。

## Follow-up

Batch 36～45 基于以上证据形成核心抽象与开发地图。
