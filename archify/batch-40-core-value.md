# Batch 40 · Harness Value

## Findings

**Inference**：困难主要在持续运行的契约，而不在调用一次 SDK。具体 trace：assistant 工具流 → 参数修复/验证 → 并行操作 → 有序回灌 → 失败 request 从投影省略 → 压缩 → 重发 → Session settled。每一步都需处理取消、历史一致性与多个Host。

| 能力 | 归属 | Pi增加的机制 |
|---|---|---|
| 规划/选文件/改法 | Model | 工具与规则供其选择，无强制Planner |
| 工具执行/错误关联 | Harness | schema、hooks、调度、输出限制 |
| 仓库理解/编码闭环 | Hybrid | 检索/读写/测试I/O，模型判断意义 |
| 压缩 | Hybrid | 触发/切点/持久边界由Pi，摘要语义由模型 |
| 会话/切换/恢复 | Harness | 日志树、projection、route/compat/retry |
| Prompt遵守/任务完成 | Hybrid | 指令与执行环境存在，但没有通用自动验收 |

## Evidence

[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[agent.ts](../packages/agent/src/agent.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[types.ts](../packages/ai/src/types.ts)、[transcript.ts](../packages/ai/src/utils/transcript.ts)、[transform-messages.ts](../packages/ai/src/api/transform-messages.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[models.ts](../packages/ai/src/models.ts)

## Key Source Files

[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[agent.ts](../packages/agent/src/agent.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[types.ts](../packages/ai/src/types.ts)、[transcript.ts](../packages/ai/src/utils/transcript.ts)、[transform-messages.ts](../packages/ai/src/api/transform-messages.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[models.ts](../packages/ai/src/models.ts)

## Key Types / Classes

Agent/Session/Projection/Tool/Provider/Runner。

## Key Functions

runLoop；buildSessionProjection；prepareCompaction；prepareToolCall；processEvents。

## Control Flow

用户输入→持续多次模型/工具往返→可保存/可取消/可恢复结束。

## Data Flow

canonical协议隔离模型差异；日志与请求视图分开。

## State

多owner不同寿命使清理与恢复成为核心复杂点。

## Design Decisions

**Inference**：UI/配置并非无价值，但核心可复用价值集中于执行与兼容契约。

## Unknowns

没有量化模型与Harness的价值占比，也没有实现成本实测。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch 41 用行为目标定义最小集。
