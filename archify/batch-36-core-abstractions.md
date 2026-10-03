# Batch 36 · Core Abstractions

## Findings

**Confirmed**：以下职责、创建入口和数据来自实现；抽象选择顺序属于 Inference。

| 抽象 | 责任 | 创建者 / 寿命 | 依赖 | 数据 / owner | 关键调用 | 证据 |
| --- | --- | --- | --- | --- | --- | --- |
| Agent | 单次运行状态/输入队列 | SDK 创建；Agent 实例 | Loop/streamFn | messages/model/tools | prompt/continue/abort | [agent.ts](../packages/agent/src/agent.ts) |
| AgentLoopConfig | 一次 loop 的策略与回调 | Agent 每次 createLoopConfig | hooks / streamFn | model/options/queue readers | runLoop | [types.ts](../packages/agent/src/types.ts) |
| AgentContext | 运行请求快照 | Agent createContextSnapshot | messages/tools | array copies | prepareRequest | [types.ts](../packages/agent/src/types.ts) |
| AgentSession | 编码行为与恢复 | SDK 创建；会话 | Agent/Manager/Resources | queues/registry/controllers | prompt/compact/setModel | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) |
| AgentSessionRuntime | 当前会话替换与绑定 | main/Host；应用 | services/session | 当前 session 引用 | 替换与 shutdown | [agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts) |
| AgentSessionServices | 服务装配 | createAgentSessionServices | Settings/Models/Resources | 共享依赖 | createAgentSessionFromServices | [agent-session-services.ts](../packages/coding-agent/src/core/agent-session-services.ts) |
| SessionManager | 日志树与投影 | SDK/Runtime；持久会话 | JSONL/transcript | entries/leaf/byId | append*/branch/buildSessionProjection | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) |
| SessionProjection | 模型可见重放结果 | Manager 构造；请求 | 树/compaction/context edits | entries/messages/model/thinking | buildContextEntries | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) |
| SystemMessage | 可重放提示/工具声明 | Session/Loop；日志 | transcript | sections/tools/deltas | normalizeContext | [types.ts](../packages/ai/src/types.ts) |
| AssistantMessage | 统一响应结果 | Provider；响应/日志 | API stream | content/usage/stop | streamAssistantResponse | [types.ts](../packages/ai/src/types.ts) |
| ToolDefinition | 产品工具定义 | 内置工厂/Extension；注册 | schema/exposure/renderers | execute/params/metadata | wrapToolDefinition | [types.ts](../packages/coding-agent/src/core/extensions/types.ts) |
| AgentTool | 可执行核心工具 | wrapper；Agent loadout | schema/execute | arguments/result | runToolCall | [types.ts](../packages/agent/src/types.ts) |
| Model | 模型数据契约 | catalog/config；目录快照 | Provider/API | limits/cost/compat | clampThinkingLevel | [types.ts](../packages/ai/src/types.ts) |
| Provider | 鉴权/目录/请求行为 | factory；Models 注册 | API adapters | auth/models/stream | streamSimple | [types.ts](../packages/ai/src/types.ts) |
| Models | 实例 Provider 服务 | createModels；服务寿命 | CredentialStore/ModelsStore | maps/generations | refresh/getAuth/streamSimple | [models.ts](../packages/ai/src/models.ts) |
| ModelRuntime | 编码层模型组合/路由 | services；服务寿命 | Models/ModelConfig | catalog/virtual routes | prepareRequest/resolveModel | [model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) |
| ResourceLoader | 资源发现/信任/加载 | services；资源代次 | packages/extensions/skills | loaded resources/errors | reload | [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) |
| ExtensionRunner | hook/command 执行环境 | Session；绑定代次 | ExtensionRuntime/core/UI | handlers/context | bindCore/emit* | [runner.ts](../packages/coding-agent/src/core/extensions/runner.ts) |
| SettingsManager | 设置合并/保存/getters | services；配置实例 | SettingsStorage | global/project/effective | getCompactionSettings/set* | [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) |
| McpServerConnection | 单服务器连接与恢复 | MCP extension；session | McpClient/transport | state/tools/opening | getClient/callTool/close | [runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts) |

## Evidence

以上表格逐项给出实现链接；抽象取自实际接口/owner，未虚构 TaskPlanner 或 MemoryDatabase。

## Key Source Files

以上表格逐项给出实现链接；抽象取自实际接口/owner，未虚构 TaskPlanner 或 MemoryDatabase。

## Key Types / Classes

20 项见 Findings；类型和实例owner不等同。

## Key Functions

表中关键调用对应 owner；私有方法表示内部路径，不作为公开SDK接口。

## Control Flow

Services → Runtime/Session → Agent/Loop → Provider/tools；Manager/Runner 从侧面参与。

## Data Flow

Message/ToolResult/Projection/Model为跨层传递数据；stream只负责流，不拥有完整产品会话。

## State

应用、会话、run、request、tool call 与资源装载代次各有寿命。

## Design Decisions

**Inference**：优先按责任和所有权理解核心，而不是按文件大小选择架构节点。

## Unknowns

抽象表不是所有类型穷举；未验证每个第三方实现遵循契约。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch 37 总结这些路径中实际采用的模式。
