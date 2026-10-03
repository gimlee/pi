# Batch 19 · Memory Boundaries

## Findings

**Confirmed**：短期记忆是模型实际请求的消息投影；持久 session、compaction summary、branch summary、AGENTS/Skill 指令是不同信息来源。核心工具表和 resource 加载路径未见自动跨会话记忆抽取、向量记忆库或用户事实召回服务：Not Present（默认内置功能）。

但不能说“没有可保存状态”：Codemode 的 store/load 通过 codemode-store custom entries 按当前 branch 重放；virtual model router 也保存 branch-local custom state。这是程序状态，不是自动语义长期记忆。

## Evidence

- [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)：投影与 custom entries。
- [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)：摘要。
- [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)：目录上下文。
- [execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts)：readCodemodeStore 与成功后的 storeWrites。
- [virtual-models.ts](../packages/coding-agent/src/core/virtual-models.ts)：getVirtualModelState。

## Key Source Files

[packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[packages/coding-agent/src/core/compaction/compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[packages/coding-agent/src/core/resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)、[packages/coding-agent/src/extensions/codemode/execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts)、[packages/coding-agent/src/core/virtual-models.ts](../packages/coding-agent/src/core/virtual-models.ts)。

## Key Types / Classes

SessionProjection、CompactionEntry、BranchSummaryEntry、CodemodeStoreEntryData、VirtualModelStateData。

## Key Functions

buildSessionProjection；readCodemodeStore；getVirtualModelState；appendCustomEntry；prepareCompaction。

## Control Flow

恢复当前分支 → 重建 messages/custom state → 请求；Codemode 成功 → storeWrites entry → 后续脚本 load。脚本失败不提交 storeWrites，已执行工具的外部副作用不回滚。

## Data Flow

历史消息和摘要影响模型；Codemode custom store 仅被脚本显式读取；它不会自动作为全量提示注入。项目规则静态加载，用户可通过文件工具人工维护。

## State

Branch-local 记忆随树路径变化；长期配置文件跨会话，但没有自动“事实可信度/过期/冲突”管理。

## Design Decisions

**Inference**：把不同记忆机制分开能避免将 session resume 误解成无损的模型内部记忆。

## Unknowns

第三方扩展、MCP 服务可以提供长期记忆；本轮未审计这些外部实现。

## Archify Diagram

本 Batch 不要求独立图；参考相邻机制图，避免把不存在的能力画成实现。

## Follow-up

Batch 20 检查扩展实现这些能力的入口。
