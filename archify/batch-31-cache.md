# Batch 31 · Cache / Warming

## Findings

**Confirmed**：本地缓存包括 catalog models-store.json、credential/model 文件 revision 快照、extension factory cache；供应商 prompt cache 通过 cacheRetention/断点/session affinity 等协议字段请求。默认 coding 设置 cacheWarming=streaming，是真实额外 LLM 请求，不是本地缓存复制。

## Evidence

- [cache-warmer.ts](../packages/coding-agent/src/core/cache-warmer.ts)：start/evaluate/refresh/cancel。
- [sdk.ts](../packages/coding-agent/src/core/sdk.ts)：warmer 与实际 stream 注入。
- [cache-stats.ts](../packages/coding-agent/src/core/cache-stats.ts)：detectMiss / scan。
- [openai-completions.ts](../packages/ai/src/api/openai-completions.ts)：cache_control 与 prompt_cache_key。
- [models-store.ts](../packages/coding-agent/src/core/models-store.ts)：revision cache。
- [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)：global-only cacheWarming。

## Key Source Files

[packages/coding-agent/src/core/cache-warmer.ts](../packages/coding-agent/src/core/cache-warmer.ts)、[packages/coding-agent/src/core/sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[packages/coding-agent/src/core/cache-stats.ts](../packages/coding-agent/src/core/cache-stats.ts)、[packages/ai/src/api/openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[packages/coding-agent/src/core/models-store.ts](../packages/coding-agent/src/core/models-store.ts)、[packages/coding-agent/src/core/settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)。

## Key Types / Classes

CacheWarmer、CacheWarmRequest、CacheWarmingStatus、CacheMiss、ModelsFileReadState。

## Key Functions

getPromptCacheTtlMs；isReplayable；getCacheWarmingDelayMs；CacheWarmer.evaluate/refresh；computeCacheWaste。

## Control Flow

真实当前模型请求 → 记录可重放 prefix → TTL 90%（保留至少10s余量）定时 → 评估预计节约至少 $0.05 / hook override → maxTokens=1,maxRetries=0 refresh → appendUsage cache_warm → 重排；上下文改变/过晚/取消/超时域停止。

## Data Flow

刷新复用原 request options/context，不追加其文本为对话。streaming 最长1h；idle 最长30min，continuation probability 常量0.15。预算式 Anthropic thinking 不安全重放，除 forceAdaptiveThinking 情形。

## State

inactive/scheduled/refreshing，timer/controller/isCurrent；miss 统计在压缩/branch summary 后重置，model switch 不豁免。noise floor=1024 tokens。

## Design Decisions

**Inference**：warmer 用价格与续请求概率估算收益，仍可能花费不节省。cache miss 推断不是服务端 cache 内部状态证明。

## Unknowns

未实际发 warm 请求或验证收益；全部 Provider cache 行为没有实测，TTL 元数据并非服务端保证。

## Archify Diagram

本 Batch 不要求独立图；参考相邻机制图，避免把不存在的能力画成实现。

## Follow-up

Batch 32～33 分析切换与协议兼容。
