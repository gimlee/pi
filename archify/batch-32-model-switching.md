# Batch 32 · Model Switching

## Findings

**Confirmed**：Session 可通过 setModel/cycleModel 与扩展选择不同模型；记录 model_change、调整 thinking，并在未来请求应用新模型。virtual selection 与 response 的 physical model 分开，路由按 user/continuation/retry/direct reason 接收当前 messages 和 branch state。选择 virtual model 不意味着该名字送到 Provider。

## Evidence

- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：setModel / cycleModel / setThinkingLevel。
- [virtual-models.ts](../packages/coding-agent/src/core/virtual-models.ts)：getBranchSelection / route types。
- [model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)：prepareRequest。
- [transform-messages.ts](../packages/ai/src/api/transform-messages.ts)：重放转换。
- [models.ts](../packages/ai/src/models.ts)：clampThinkingLevel。

## Key Source Files

[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[packages/coding-agent/src/core/virtual-models.ts](../packages/coding-agent/src/core/virtual-models.ts)、[packages/coding-agent/src/core/model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[packages/ai/src/api/transform-messages.ts](../packages/ai/src/api/transform-messages.ts)、[packages/ai/src/models.ts](../packages/ai/src/models.ts)。

## Key Types / Classes

ModelMutationOptions、ModelRouteRequest、VirtualModelDefinition、AssistantMessage.provider/model。

## Key Functions

setModel；cycleModel；clampThinkingLevel；getBranchSelection；getVirtualModelState；prepareRequest。

## Control Flow

用户/extension 选择 → registry/auth/capability 检查 → Agent model 与 thinking 更新 → session entry → 下一请求 route/convert → 新 Provider。setModel 本身没有 require-idle 检查；它更新选型与日志，Session 的 prepareRequest 在后续请求读取当前 model。已经发送的网络流继续使用原请求，不能瞬时迁移。

## Data Flow

历史 canonical messages 保留原 provider/api/model；新 adapter 去掉无法重放的 thinking signature、修复工具 id 和孤儿结果、处理图片/系统/工具变更。UI hideThinking 不会自动删除历史。

## State

selected model、last successful physical response、thinking selection、branch router state、available catalog。

## Design Decisions

**Inference**：兼容层让会话可跨模型继续，但没有无损转移模型隐藏推理状态承诺。

## Unknowns

未运行跨模型会话；API-specific 内部 opaque 数据不保证可跨供应商使用。

## Archify Diagram

本 Batch 不要求独立图；参考相邻机制图，避免把不存在的能力画成实现。

## Follow-up

Batch 33 汇总已经验证的适配矩阵。
