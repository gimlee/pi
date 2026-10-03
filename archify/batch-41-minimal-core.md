# Batch 41 · Minimal Core

## Findings

**Inference（概念最小集，不是已测量的 20% 代码）**：

| 层 | 必须行为 | Class / Function | 可复用证据 |
| --- | --- | --- | --- |
| 消息契约 | user/assistant/toolResult、id关联、usage/stop/error | Message / AssistantMessage / ToolResultMessage | [types.ts](../packages/ai/src/types.ts) |
| 流适配 | 一个Provider实现，文本/工具delta/终结 | AssistantMessageEventStream / stream() | [event-stream.ts](../packages/ai/src/utils/event-stream.ts)、[openai-completions.ts](../packages/ai/src/api/openai-completions.ts) |
| Loop | 请求→工具执行→结果回灌→继续/结束/取消 | runLoop / executeToolCalls / createToolResultMessage | [agent-loop.ts](../packages/agent/src/agent-loop.ts) |
| 运行所有权 | messages/tools/model、单run、queue/abort/finally | Agent.prompt / continue / abort / processEvents | [agent.ts](../packages/agent/src/agent.ts) |
| Context builder | 系统/当前对话/工具声明，输出与window边界 | normalizeContext / getCurrentSystemMessage | [transcript.ts](../packages/ai/src/utils/transcript.ts) |
| 工具层 | schema验证、read/edit/write/Shell、错误关联 | ToolDefinition / wrapToolDefinition / create*ToolDefinition | [index.ts](../packages/coding-agent/src/core/tools/index.ts) |
| Host | 读取输入、呈现流与错误、取消 | runPrintMode | [print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts) |

若要求可恢复长任务，再加入 JSONL/branch projection、压缩和有界重试。TUI、MCP、Skill、复杂扩展、虚拟路由、warm/eval 是目标驱动的可选层；删除它们不是在现有Pi中授权移除功能。

## Evidence

[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[agent.ts](../packages/agent/src/agent.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[types.ts](../packages/ai/src/types.ts)、[transcript.ts](../packages/ai/src/utils/transcript.ts)、[transform-messages.ts](../packages/ai/src/api/transform-messages.ts)

## Key Source Files

[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[agent.ts](../packages/agent/src/agent.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[types.ts](../packages/ai/src/types.ts)、[transcript.ts](../packages/ai/src/utils/transcript.ts)、[transform-messages.ts](../packages/ai/src/api/transform-messages.ts)

## Key Types / Classes

Message、Model、AgentTool、AgentContext、EventStream、Agent。

## Key Functions

runLoop / streamAssistantResponse / executeToolCalls / createToolResultMessage / Agent.processEvents。

## Control Flow

单请求适配→循环→工具→结果→续轮；取消在finally收尾。

## Data Flow

只支持一个Provider也需一致消息/工具ID，不能省掉错误结果。

## State

最低run状态与messages；持久恢复是额外目标的必要组件。

## Design Decisions

**Inference**：20%是任务中的概念问法，未用LOC证明；避免机械裁剪当前仓库。

## Unknowns

没有构建MiniPi或运行性能/规模对比。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch42给可执行阅读入口。
