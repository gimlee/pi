# Batch 30 · Tokens / Usage / Cost

## Findings

**Confirmed**：Provider 回报用量与本地预估分开。Usage 的 input/output/cacheRead/cacheWrite 是归一桶，reasoning 是 output 的可选细分，cacheWrite1h 是 cacheWrite 的细分，不应再叠加总量。Cost 来自 Model 的每百万 token 价格/tiers，并非对账 API 的实际账单。

## Evidence

- [types.ts](../packages/ai/src/types.ts)：Usage / ModelCost。
- [models.ts](../packages/ai/src/models.ts)：calculateCost。
- [usage-totals.ts](../packages/coding-agent/src/core/usage-totals.ts)：combineUsage / breakdown。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：getSessionStats / getContextUsage。
- [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)：estimateProjectedContextTokens。

## Key Source Files

[packages/ai/src/types.ts](../packages/ai/src/types.ts)、[packages/ai/src/models.ts](../packages/ai/src/models.ts)、[packages/coding-agent/src/core/usage-totals.ts](../packages/coding-agent/src/core/usage-totals.ts)、[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[packages/coding-agent/src/core/compaction/compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)。

## Key Types / Classes

Usage、UsageTotals、ModelCost、ContextUsage、UsageEntry、CacheWasteTotals。

## Key Functions

calculateCost；combineUsage；addUsageToTotals；getUsageCostBreakdown；getSessionStats；estimateProjectedContextTokens。

## Control Flow

Provider counters → canonical Usage → calculateCost → assistant message、摘要/branch summary/独立 usage entry → Session 统计与 UI；下一次 compaction 的阈值使用实际 usage 或 chars/4 fallback。

## Data Flow

prompt tokens=input+cacheRead+cacheWrite；usage.totalTokens 与 fallback 四桶总和使用需看函数；上下文 percentage 以实际请求物理模型 window 为分母，不拿历史总 tokens 当当前上下文。

## State

branch usage 与全树累计统计的调用集合不同；warm 请求有单独 usage entry；模型价为 0 时不能推导免费。

## Design Decisions

**Inference**：透明分桶帮助诊断 cache 命中和摘要开销；token 预估不保证所有语言准确，也没有默认全局金额硬预算执行器。

## Unknowns

没与供应商账单核对；模型目录价格可能随时间改变，本报告只说明固定源码快照算法。

## Archify Diagram

本 Batch 不要求独立图；参考相邻机制图，避免把不存在的能力画成实现。

## Follow-up

Batch 31 区分本地缓存与供应商 prompt cache。
