# Batch 44 · Critical Call Chains

## Findings

### 1. Startup

`main() → createAgentSessionServices() → createAgentSessionFromServices()/createAgentSession() → AgentSession → Host bind`

证据：[main.ts](../packages/coding-agent/src/main.ts)、[agent-session-services.ts](../packages/coding-agent/src/core/agent-session-services.ts)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)。

### 2. User input

`Host submit/print prompt → AgentSession.prompt() → _runAgentPrompt() → Agent.prompt() → runPromptMessages() → runAgentLoop()`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[agent.ts](../packages/agent/src/agent.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts)。

### 3. Context

`Agent.prepareRequest hook（Session安装） → SessionManager.buildSessionProjection() → buildContextEntries()/projectContextEntry() → transformContext → convertToLlm`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)。

### 4. Prompt

`AgentSession._preparePromptAndToolLoadout() → buildSystemPromptSections() → system patch / tool delta → transcript normalizeContext()`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)、[transcript.ts](../packages/ai/src/utils/transcript.ts)。

### 5. Model request

`streamAssistantResponse() → 注入SDK streamFn → ModelRuntime.streamSimple()/prepareRequest() → Provider.streamSimple() → API.streamSimple()/stream()`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[models.ts](../packages/ai/src/models.ts)、[openai-completions.ts](../packages/ai/src/api/openai-completions.ts)。

### 6. Streaming

`API stream() chunk parser → AssistantMessageEventStream → streamAssistantResponse() → Agent.processEvents() → AgentSession._handleAgentEvent()`

证据：[openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[event-stream.ts](../packages/ai/src/utils/event-stream.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent.ts](../packages/agent/src/agent.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)。

### 7. Tool call

`runLoop() → executeToolCalls() → prepareToolCallArguments()/prepareToolCall() → executePreparedToolCall() → finalizeExecutedToolCall()`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[tool-definition-wrapper.ts](../packages/coding-agent/src/core/tools/tool-definition-wrapper.ts)。

### 8. Tool result

`finalizeExecutedToolCall() → createToolResultMessage() → emitToolResultMessage() → processEvents() → _handleAgentEvent() → appendMessage()`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent.ts](../packages/agent/src/agent.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)。

### 9. File read

`executePreparedToolCall() → createReadToolDefinition().execute → resolveReadPathAsync() → operations.access/readFile → truncateHead()`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[read.ts](../packages/coding-agent/src/core/tools/read.ts)、[path-utils.ts](../packages/coding-agent/src/core/tools/path-utils.ts)、[truncate.ts](../packages/coding-agent/src/core/tools/truncate.ts)。

### 10. File edit

`prepareEditArguments() → createEditToolDefinition().execute → withFileMutationQueue() → operations.readFile → applyEditsToNormalizedContent() → operations.writeFile`

证据：[edit.ts](../packages/coding-agent/src/core/tools/edit.ts)、[file-mutation-queue.ts](../packages/coding-agent/src/core/tools/file-mutation-queue.ts)、[edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts)。

### 11. Shell

`createShellToolDefinition().execute → resolveSpawnContext() → BashOperations.exec() → createLocalShellOperations() spawn → waitForChildProcess() → accumulator结果`

证据：[bash.ts](../packages/coding-agent/src/core/tools/bash.ts)、[output-accumulator.ts](../packages/coding-agent/src/core/tools/output-accumulator.ts)、[child-process.ts](../packages/coding-agent/src/utils/child-process.ts)。

### 12. Compression

`AgentSession.compact() / _checkCompaction() → prepareCompaction() → extension session_before_compact/default compact() → completeSummarization() → appendCompaction() → _refreshFinalizedContext()`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)。

### 13. Extension

`ResourceLoader.reload() → loadExtensionsCached()/loadExtensionFromFactory() → initializeExtension() → createExtensionAPI() commit → Runner.bindCore() → session_start/后续hook`

证据：[resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)、[loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)、[runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)。

### 14. MCP

`createMcpToolDefinition().execute → McpServerConnection.callTool()/withClient() → McpClient.callTool()/requestInternal() → McpTransport.send() → convertMcpResult()`

证据：[tools.ts](../packages/coding-agent/src/extensions/mcp/tools.ts)、[runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts)、[client.ts](../packages/mcp/src/client.ts)。

### 15. Final answer

`runLoop() 无工具/队列结束 → agent_end → Agent.processEvents()/finishRun() → Session._runAgentPrompt() 恢复/边界检查 → agent_settled → Host 输出`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent.ts](../packages/agent/src/agent.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts)。

### 16. Model switch

`AgentSession.setModel() → checkAuth → Agent model更新/appendModelChange → setThinkingLevel → 新prepareRequest/route → transformMessages`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[virtual-models.ts](../packages/coding-agent/src/core/virtual-models.ts)、[transform-messages.ts](../packages/ai/src/api/transform-messages.ts)。

### 17. Nested Codemode

`executeCodemode() → CodemodeSandbox script → ctx.executeTool() → runToolCall() → toScriptValue() → 输出截断 / storeWrites`

证据：[execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts)、[runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)。

## Evidence

以上17条为真实符号/调用面；箭头含同步调用、hook安装和事件回调，区别在文字中说明。

## Key Source Files

以上17条为真实符号/调用面；箭头含同步调用、hook安装和事件回调，区别在文字中说明。

## Key Types / Classes

类名/自由函数见每条trace；free function不伪装成Class.method。

## Key Functions

各条执行符号见 Findings；厂工具execute为返回定义对象的方法。

## Control Flow

典型路径不代表每次运行都有全部阶段；extension短路、无tools、manual compact可走不同分支。

## Data Flow

贯穿canonical Message、ToolCall、ToolResult、Projection、wire payload和events。

## State

外层Session可跨底层Agent run继续；旧runtime的ctx被invalidate。

## Design Decisions

**Inference**：把函数链与owner边界一起看，比“LLM→Tool→LLM”框图更能解释Pi。

## Unknowns

静态trace未用真实模型运行；Provider族入口不同，代表链不穷举。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch45把阅读链转成修改地图。
