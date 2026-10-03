# Batch 17 · State Ownership

## Findings

**Confirmed**：状态按作用域分散，没有单一全局 AppState。

| Owner | 状态 / 寿命 |
|---|---|
| Host / TUI | Editor、折叠、布局、当前显示组件 / Host 寿命 |
| AgentSessionRuntime | 当前 session/services、替换绑定 / 应用运行 |
| AgentSession | 定义注册表、queue、retry/compaction、扩展 runner / 会话 |
| Agent | messages、model、tools、streamingMessage、pending calls / Agent 实例 |
| SessionManager | 日志树、leaf、投影 / 可持久化会话 |
| ModelRuntime / Models | provider/catalog/auth facade、刷新 generation / 服务 |
| ExtensionRuntime | 注册、flags、上下文有效性 / 扩展装载代次 |

## Evidence

- [agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)：当前 session/services。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：会话字段及 dispose。
- [agent.ts](../packages/agent/src/agent.ts)：AgentState reducer。
- [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)：树与 leaf。
- [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)：invalidate 与注册状态。

## Key Source Files

[packages/coding-agent/src/core/agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)、[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[packages/agent/src/agent.ts](../packages/agent/src/agent.ts)、[packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[packages/coding-agent/src/core/extensions/loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)。

## Key Types / Classes

AgentState、AgentSession、AgentSessionRuntime、ExtensionRuntime、SettingsManager、ModelRuntime。

## Key Functions

Agent.processEvents；AgentSession._handleAgentEvent / dispose；SessionManager.buildSessionProjection；ExtensionRuntime.invalidate。

## Control Flow

Agent 事件更新运行状态 → Session 处理/persist → Host 订阅重绘。替换会话使旧 extension context 失效，旧捕获 ctx 不能继续修改新会话。

## Data Flow

同一 assistant 的流中 partial、最终消息、日志 entry 和 UI component 是不同表示；只有规定终结事件成为稳定历史。

## State

内存执行状态不能全部序列化；日志保留消息/选择/自定义数据，OS PID、timer、监听器、UI 对象不属于重放状态。

## Design Decisions

**Inference**：所有权分离允许 headless Host 与同一 Session 核心复用；也要求替换和清理时跨层协调。

## Unknowns

未验证任意第三方扩展自行创建的全局对象能否在 reload 时释放。

## Archify Diagram

[diagram-17-state](diagrams/diagram-17-state/diagram-17-state.html)

## Follow-up

Batch 18 记录各持久化载体。
