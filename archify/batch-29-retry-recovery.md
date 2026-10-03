# Batch 29 · Retry / Recovery

## Findings

**Confirmed**：至少四个恢复域：Provider 初始请求重试、Session assistant 续请求、摘要 retryAssistantCall、MCP connect/read-only/session-expiry 恢复。它们有独立预算与边界，工具执行失败不会由通用 Agent Loop 无条件重跑。

## Evidence

- [provider-retry.ts](../packages/ai/src/utils/provider-retry.ts)：retryProviderRequest。
- [retry.ts](../packages/ai/src/utils/retry.ts)：classifier / retryDelayMs / retryAssistantCall。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：_runAgentPrompt / retry / _checkCompaction。
- [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)：completeSummarization。
- [runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts)：withClient / open。

## Key Source Files

[packages/ai/src/utils/provider-retry.ts](../packages/ai/src/utils/provider-retry.ts)、[packages/ai/src/utils/retry.ts](../packages/ai/src/utils/retry.ts)、[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[packages/coding-agent/src/core/compaction/compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[packages/coding-agent/src/extensions/mcp/runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts)。

## Key Types / Classes

RetryPolicy、ProviderRetrySettings、AssistantMessage、AbortController、McpSessionExpiredError。

## Key Functions

retryProviderRequest；isRetryableAssistantError；retryDelayMs；retryAssistantCall；AgentSession._runAgentPrompt / _checkCompaction。

## Control Flow

请求失败 → Provider 可中断 backoff（SDK retries=0）→ 若流中或外层仍 error，Session 先处理 context overflow → transient classifier → 有界延时 → fresh request；耗尽/取消/非 transient → settle。

## Data Flow

失败 assistant 可通过 context edit 从请求投影省略；raw log 保留。重新发起模型请求不是从上一个网络 token checkpoint 继续，也不撤销已完成工具。virtual router 可在 retry reason 时选择物理模型。

## State

Session 默认 enabled=true、3 retries、base=2s（2/4/8s）、每次上限 60s；Provider honors retry-after-ms/retry-after/x-should-retry，服务器 delay 超 cap 直接失败。摘要与 MCP 使用自己的调用预算。Provider helper 在未传 maxRetries 时为 0；代表 OpenAI/Anthropic adapter 传 options.maxRetries，不能套用 SDK 默认两次重试。没有服务器 delay 时使用 min(0.5 × 2^retryIndex, 8)s，并乘 0.75～1 的随机因子；可中断等待。

## Design Decisions

**Confirmed**：quota/billing/账户限额分类为不重试，abort 终结。Context overflow 使用压缩恢复，单个 Session run 的失败恢复最多一次；成功 length/overflow 压缩不等同自动重发失败工具。

## Unknowns

字符串 classifier 可误判，完整 Provider 族没有逐个故障实验；重试叠加的总耗时没有全局 max-duration 保证。

## Archify Diagram

[diagram-29-retry](diagrams/diagram-29-retry/diagram-29-retry.html)

## Follow-up

Batch 30～31 跟踪 token 与缓存成本。
