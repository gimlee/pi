# Batch 46 · Pi Agent Master Architecture Map

## Findings

**Confirmed**：Master Map 在前面机制分析后收敛，包含用户/Host、Runtime/Session、Agent/Context/Message/Model/Provider/LLM、工具与文件/Shell/搜索/MCP、Extension/Event/Config/Persistence/Workspace。普通helper不单独成架构节点。节点数与来源见 candidate 和全量校验。

## Evidence

[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[agent.ts](../packages/agent/src/agent.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[types.ts](../packages/ai/src/types.ts)、[transcript.ts](../packages/ai/src/utils/transcript.ts)、[transform-messages.ts](../packages/ai/src/api/transform-messages.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[models.ts](../packages/ai/src/models.ts)、[openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts)、[openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts)、[google-shared.ts](../packages/ai/src/api/google-shared.ts)、[system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)、[resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)、[runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)、[loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)、[edit.ts](../packages/coding-agent/src/core/tools/edit.ts)、[edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts)、[bash.ts](../packages/coding-agent/src/core/tools/bash.ts)、[read.ts](../packages/coding-agent/src/core/tools/read.ts)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)、[main.ts](../packages/coding-agent/src/main.ts)、[interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)、[settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)、[index.ts](../packages/coding-agent/src/extensions/mcp/index.ts)、[client.ts](../packages/mcp/src/client.ts)、[execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts)

## Key Source Files

[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[agent.ts](../packages/agent/src/agent.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[types.ts](../packages/ai/src/types.ts)、[transcript.ts](../packages/ai/src/utils/transcript.ts)、[transform-messages.ts](../packages/ai/src/api/transform-messages.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[models.ts](../packages/ai/src/models.ts)、[openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts)、[openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts)、[google-shared.ts](../packages/ai/src/api/google-shared.ts)、[system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)、[resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)、[runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)、[loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)、[edit.ts](../packages/coding-agent/src/core/tools/edit.ts)、[edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts)、[bash.ts](../packages/coding-agent/src/core/tools/bash.ts)、[read.ts](../packages/coding-agent/src/core/tools/read.ts)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)、[main.ts](../packages/coding-agent/src/main.ts)、[interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)、[settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)、[index.ts](../packages/coding-agent/src/extensions/mcp/index.ts)、[client.ts](../packages/mcp/src/client.ts)、[execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts)

## Key Types / Classes

核心owner见Batch36；Master是组件关系图，不是逐语句sequence。

## Key Functions

运行主链、工具分支与辅助配置/事件/日志关系见图。

## Control Flow

用户→Host→Session→Agent→Context/Message→模型请求→LLM；工具结果回到同一上下文。

## Data Flow

canonical日志/投影与供应商wire分开；File/Shell/Search/MCP是执行能力而非模型大脑。

## State

Config/资源服务跨请求，Agent是run状态，Manager是branch日志；Extension ctx有装载代次。

## Design Decisions

**Inference**：在图中保留真实执行owner与行为边界，省略重复工厂/helper细节。

## Unknowns

图的LLM节点表示外部服务边界，来源引用实际请求代码；没有连接外部服务验证。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch47汇总图集与receipt，Batch48汇总43节报告与五问。
