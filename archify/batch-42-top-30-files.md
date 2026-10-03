# Batch 42 · 30 Core Files

## Findings

| 顺序 | 源码 | 类型 / 函数 | 为何重要 | 关联模块 |
| --- | --- | --- | --- | --- |
| 1 | [agent-loop.ts](../packages/agent/src/agent-loop.ts) | runLoop / streamAssistantResponse / executeToolCalls | 真正的模型-工具续轮 | Agent/Provider/tools |
| 2 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) | prompt / _runAgentPrompt / compact / _checkCompaction | Coding Harness 中枢 | Manager/Models/Extensions |
| 3 | [agent.ts](../packages/agent/src/agent.ts) | prompt / continue / processEvents / finishRun | 运行所有权和队列 | Loop/Host |
| 4 | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | buildSessionProjection / appendCompaction / branch / forkFrom | 日志树与可见上下文 | Session/Compaction |
| 5 | [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) | prepareCompaction / compact / estimateProjectedContextTokens | 长会话切点/摘要/估算 | Session/Provider |
| 6 | [types.ts](../packages/ai/src/types.ts) | Message / Model / Provider / Usage | 跨包数据契约 | 全部 adapter |
| 7 | [transcript.ts](../packages/ai/src/utils/transcript.ts) | normalizeContext / getCurrentSystemMessage | 系统/工具重放 | Loop/Adapters |
| 8 | [transform-messages.ts](../packages/ai/src/api/transform-messages.ts) | transformMessages | 跨模型历史修复 | Adapters |
| 9 | [model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) | prepareRequest / resolveModel | 配置、鉴权、路由装配 | Models/Registry |
| 10 | [models.ts](../packages/ai/src/models.ts) | createModels / calculateCost / clampThinkingLevel | 实例模型服务及能力 | Provider/stores |
| 11 | [openai-completions.ts](../packages/ai/src/api/openai-completions.ts) | stream / streamSimple / convertMessages / buildParams | 兼容协议变体厚度 | Models/transcript |
| 12 | [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) | stream / streamSimple | thinking/tools/cache原生映射 | transcript |
| 13 | [openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts) | 消息/工具转换与事件解析 | Responses共享协议行为 | Responses/Codex adapters |
| 14 | [google-shared.ts](../packages/ai/src/api/google-shared.ts) | convertMessages / convertTools | Google消息、签名、工具映射 | GenAI/Vertex |
| 15 | [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts) | buildSystemPrompt / buildSystemPromptSections | 提示组成 | ResourceLoader/Session |
| 16 | [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) | DefaultResourceLoader.reload | 资源与信任装载 | Extensions/Skills/Packages |
| 17 | [runner.ts](../packages/coding-agent/src/core/extensions/runner.ts) | bindCore / emit / emitContext / emitBoundary | 动态策略和边界 | Session/Host |
| 18 | [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts) | initializeExtension / createExtensionRuntime | 装载与有效期 | ResourceLoader/Runner |
| 19 | [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) | createEditToolDefinition | 文件变更管线 | edit-diff/queue |
| 20 | [edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts) | applyEditsToNormalizedContent / fuzzyFindText | 唯一匹配与差异生成 | Edit/Preview |
| 21 | [bash.ts](../packages/coding-agent/src/core/tools/bash.ts) | createLocalShellOperations / createShellToolDefinition | OS执行与输出 | Shell/Accumulator |
| 22 | [read.ts](../packages/coding-agent/src/core/tools/read.ts) | createReadToolDefinition | 上下文文件入口 | images/truncate |
| 23 | [sdk.ts](../packages/coding-agent/src/core/sdk.ts) | createAgentSession | 核心装配与无UI入口 | Services/Session |
| 24 | [agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts) | AgentSessionRuntime | 切换与生命周期Host绑定 | Services/Session |
| 25 | [main.ts](../packages/coding-agent/src/main.ts) | main | 进程启动与模式选择 | CLI/Runtime/Hosts |
| 26 | [interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts) | InteractiveMode | 用户交互/命令/呈现 | Session/TUI |
| 27 | [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) | SettingsManager / getCompactionSettings | 有效配置 | SDK/Session |
| 28 | [index.ts](../packages/coding-agent/src/extensions/mcp/index.ts) | createMcpExtension | 工具发现/曝光/生命周期 | Runner/Connection |
| 29 | [client.ts](../packages/mcp/src/client.ts) | McpClient.connect / callTool / requestInternal | JSON-RPC连接/取消/请求 | Transports/MCP extension |
| 30 | [execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts) | executeCodemode / readCodemodeStore | 脚本编排与嵌套工具 | Sandbox/Session/MCP |

## Evidence

顺序是阅读/复用价值判断（Inference），每项实现职责为静态Confirmed。

## Key Source Files

顺序是阅读/复用价值判断（Inference），每项实现职责为静态Confirmed。

## Key Types / Classes

类型/函数和关联模块见表。

## Key Functions

各行列出具体符号；某些API共享模块含多转换函数，需沿实际调用读。

## Control Flow

由Loop核心→Session/Context→模型协议→工具→产品装配/扩展阅读。

## Data Flow

沿统一消息与ToolResult追踪，避免从helper反向猜主架构。

## State

表中特别区分运行owner、持久owner和请求转换。

## Design Decisions

**Inference**：前10项解释持续Agent；后20项解释编码产品与可扩展性。

## Unknowns

不是性能热点排行，也不是代码修改授权。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch43按时间组织路径。
