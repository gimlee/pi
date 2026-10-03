# Batch 23 · Event / Streaming Flow

## Findings

**Confirmed**：Provider 原生事件归一为 AssistantMessageEventStream；Agent 消费后发 AgentEvent；AgentSession 增加压缩、重试和 settled 等产品事件；Host、ExtensionRunner 等订阅。Extension EventBus 是额外的任意频道通信，不是这些核心 typed events 的持久可靠队列。

## Evidence

- [event-stream.ts](../packages/ai/src/utils/event-stream.ts)：异步流/result。
- [agent-loop.ts](../packages/agent/src/agent-loop.ts)：streamAssistantResponse 与 emit。
- [agent.ts](../packages/agent/src/agent.ts)：事件 reducer 与监听。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：_handleAgentEvent / AgentSessionEvent。
- [event-bus.ts](../packages/coding-agent/src/core/event-bus.ts)：EventEmitter 的 safeHandler。

## Key Source Files

[packages/ai/src/utils/event-stream.ts](../packages/ai/src/utils/event-stream.ts)、[packages/agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts)、[packages/agent/src/agent.ts](../packages/agent/src/agent.ts)、[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[packages/coding-agent/src/core/event-bus.ts](../packages/coding-agent/src/core/event-bus.ts)。

## Key Types / Classes

AssistantMessageEvent、AgentEvent、AgentSessionEvent、EventStream、EventBus。

## Key Functions

streamAssistantResponse；Agent.processEvents；AgentSession._handleAgentEvent / subscribe；Runner.emit；createEventBus。

## Control Flow

Provider delta → canonical partial → message_update → Agent streamingMessage/Host render；done → message_end → persistence；工具 start/update/end → pending/tool UI；底层 agent_end → Session 恢复/续轮 → agent_settled。

## Data Flow

局部参数 streaming 使用 tolerant JSON 解析，不代表最终参数通过 schema。结束 AssistantMessage 才有最终 usage/stopReason；原生 provider_stream_event hook 可另行观察。

## State

stream buffer、listener collections、streamingMessage、pendingToolCalls、Session event queue/控制器。

## Design Decisions

**Inference**：流协议隔离 Provider 差异并服务多个 Host。EventBus.emit 返回 void，不 await 所有 async listeners；其异常写 console，不提供事务/重放/确认机制。

## Unknowns

未测所有 listener 抛错与 UI 背压；可选 telemetry 不是每个事件自动进入远端分析的证明。

## Archify Diagram

[diagram-23-events](diagrams/diagram-23-events/diagram-23-events.html)

## Follow-up

Batch 24 检查异步执行的所有权。
