# Batch 33 · Cross-provider Compatibility

## Findings

**Confirmed**：统一 Message/Tool/Usage 只是入口契约，实际兼容来自 transcript + transform + 各 API adapter。

| 维度 | 代表差异 |
|---|---|
| 系统指令 | developer/system、独立 systemInstruction、系统段落折叠与更新 |
| 工具 | function/tool_use/functionCall；ID 清洗、结果角色/名称、bridge assistant |
| schema | strict require/prefer、支持子集与降级、grammar 支持 |
| thinking | budget/effort/level、显式 visible 与 opaque signature、历史重放 |
| 图片 | base64 block / data URL / inlineData，模型上限与过滤 |
| streaming | SSE/SDK 原生事件 → start/delta/end/done/error |
| usage/stop | cache/reasoning 桶与价格、finish_reason 缺失策略 |
| 传输/鉴权 | API key/OAuth、headers、session affinity、initial retry |
| cache | 断点、TTL、prompt key、可选 warm 请求 |

## Evidence

- [transcript.ts](../packages/ai/src/utils/transcript.ts)：normalizeContext / replay。
- [transform-messages.ts](../packages/ai/src/api/transform-messages.ts)：signature 与 ID 修复。
- [openai-completions.ts](../packages/ai/src/api/openai-completions.ts)：compat flags 与 reasoning_formats。
- [openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts)：Responses mapping。
- [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts)：Anthropic mapping。
- [google-shared.ts](../packages/ai/src/api/google-shared.ts)：Google mapping。

## Key Source Files

[packages/ai/src/utils/transcript.ts](../packages/ai/src/utils/transcript.ts)、[packages/ai/src/api/transform-messages.ts](../packages/ai/src/api/transform-messages.ts)、[packages/ai/src/api/openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[packages/ai/src/api/openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts)、[packages/ai/src/api/anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts)、[packages/ai/src/api/google-shared.ts](../packages/ai/src/api/google-shared.ts)。

## Key Types / Classes

TranscriptContext、Model.compat、ThinkingContent、ConstrainedSampling、Usage、Provider。

## Key Functions

normalizeContext；transformMessages；convertMessages / convertTools（各 adapter）；streamSimple 的 thinking mapping。

## Control Flow

canonical transcript → 公共重放修复 → 协议专属参数/工具/角色/推理编码 → request hook → 原生请求 → 统一事件与用量。

## Data Flow

OpenAI-compatible URL 不等于 Responses API；OpenRouter 身份可支持多个 API。session选择/provider身份/api协议必须分别跟踪。

## State

per-model compat/capability、current tool declaration、stream partial indexes 与 encrypted reasoning metadata。

## Design Decisions

**Inference**：Harness 适配显著多于简单 SDK 参数转发；没有严谨分母，不能以 LOC 或“百分之多少”量化整体价值。

## Unknowns

典型 Anthropic / Responses / Completions / Google 已深入阅读，未逐行审计所有42 provider工厂及10 chat API全部边界。

## Archify Diagram

[diagram-33-compatibility](diagrams/diagram-33-compatibility/diagram-33-compatibility.html)

## Follow-up

Batch 34～35 明确静态证据与运行验证差异。
