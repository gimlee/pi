# Batch 45 · Pi Development Map

## Findings

| 任务 | 入口 | 符号 | 联动点 |
| --- | --- | --- | --- |
| 修改 Agent Loop | [agent-loop.ts](../packages/agent/src/agent-loop.ts) | runLoop / executeToolCalls | 续轮、终止、队列和工具批次 |
| 修改 System Prompt | [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts) | buildSystemPromptSections / buildSystemPrompt | 段落构建；另检查 Session delta |
| 修改 Context | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | buildSessionProjection / buildContextEntries | 当前分支/edits/压缩重放 |
| 修改 Compression | [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) | prepareCompaction / compact | 切点与摘要；Session 负责触发/提交 |
| 增加 Provider | [all.ts](../packages/ai/src/providers/all.ts) | builtinProviders / builtinModels | 新增 factory；API 新协议另增 adapter |
| 增加模型 | [generate-models.ts](../packages/ai/scripts/generate-models.ts) | 生成模型脚本 | 静态目录更新生成源，不直接改 generated；动态目录检查 Provider refresh |
| 修改 Message Conversion | [transform-messages.ts](../packages/ai/src/api/transform-messages.ts) | transformMessages | 共享重放；wire 编码看对应 api 文件 |
| 修改 Tool Call | [agent-loop.ts](../packages/agent/src/agent-loop.ts) | prepareToolCall / runToolCall | 参数、hooks、错误和结果 |
| 增加 Tool | [types.ts](../packages/coding-agent/src/core/extensions/types.ts) | ToolDefinition / registerTool | 优先扩展；内置加工厂/index |
| 修改 Read Tool | [read.ts](../packages/coding-agent/src/core/tools/read.ts) | createReadToolDefinition | 分页、图片、输出限制 |
| 修改 Edit Tool | [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) | createEditToolDefinition | 配合 edit-diff 与 mutation queue |
| 修改 Shell Tool | [bash.ts](../packages/coding-agent/src/core/tools/bash.ts) | createShellToolDefinition / createLocalShellOperations | spawn、输出与 timeout/abort |
| 修改 Session | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | append*/branch/forkFrom | 日志结构与版本迁移；行为入口在 AgentSession |
| 增加 Extension | [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts) | initializeExtension / ExtensionFactory | 用户 extension factory + Runner API；本机可信执行 |
| 修改 MCP | [runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts) | McpServerConnection / createDefaultTransport | 曝光/管理在 index，协议在 packages/mcp |
| 修改 Config | [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) | Settings / getters | 逐键 trust/default/merge/save 路径 |
| 修改 Events | [types.ts](../packages/agent/src/types.ts) | AgentEvent | 同时更新 producer/reducer/Session/Host |
| 修改 Token Usage | [models.ts](../packages/ai/src/models.ts) | calculateCost | 计数来自 API adapter；统计看 usage-totals |
| 修改 Streaming | [event-stream.ts](../packages/ai/src/utils/event-stream.ts) | EventStream / AssistantMessageEventStream | 原生事件解析看具体 API adapter |
| 修改 Reasoning | [simple-options.ts](../packages/ai/src/api/simple-options.ts) | adjustMaxTokensForThinking | thinking map在Models；budget/effort/signature看具体API |

## Evidence

入口链接均在表内；更改前需完整阅读目标及关联模块。

## Key Source Files

入口链接均在表内；更改前需完整阅读目标及关联模块。

## Key Types / Classes

ToolDefinition/Provider/AgentEvent/Settings 等契约与实现共同修改。

## Key Functions

表内给出20项动作的符号。

## Control Flow

核心修改连带 producer/consumer；扩展新增注册后经统一Session/Loop执行。

## Data Flow

不要直接修改model generated；不要把UI details误送模型正文。

## State

处理运行/注册/日志schema寿命；有migration要求时要明确。

## Design Decisions

**Inference**：开发地图只建议位置；不是本轮修改代码或提交的结果。

## Unknowns

未实施任何功能变更、测试或PR。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch46只保留核心角色生成Master Map。
