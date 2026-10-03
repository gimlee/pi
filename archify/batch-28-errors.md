# Batch 28 · Error Boundaries

## Findings

**Confirmed**：错误在不同边界有不同表示。

| 边界 | 表示 / 可观察行为 |
|---|---|
| Provider setup/stream | canonical error/aborted AssistantMessage，保留 errorMessage；流失败不等于无 usage |
| Tool prepare/validate/execute | AgentToolResult isError → 关联 ToolResultMessage |
| Extensions | loader errors / hook-specific emit 策略；部分错误被通知或容错 |
| MCP | JSON-RPC/timeout/transport typed error 或 CallToolResult.isError，两者不同 |
| 文件/命令 | OS error、exit code、timeout/cancel；工具管线规范化 |
| 配置 | diagnostics/error snapshot；不能把失败全部看作 throw 到 main |
| Host | 接受核心事件并呈现；错误文字不是统一 Error enum |

## Evidence

- [agent-loop.ts](../packages/agent/src/agent-loop.ts)：prepare/execute/finalize 错误。
- [lazy.ts](../packages/ai/src/api/lazy.ts)：setup failure。
- [openai-completions.ts](../packages/ai/src/api/openai-completions.ts)：stream catch。
- [runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)：hook error isolation。
- [jsonrpc.ts](../packages/mcp/src/protocol/jsonrpc.ts)：McpError types。
- [model-config.ts](../packages/coding-agent/src/core/model-config.ts)：getError snapshot。

## Key Source Files

[packages/agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts)、[packages/ai/src/api/lazy.ts](../packages/ai/src/api/lazy.ts)、[packages/ai/src/api/openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[packages/coding-agent/src/core/extensions/runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)、[packages/mcp/src/protocol/jsonrpc.ts](../packages/mcp/src/protocol/jsonrpc.ts)、[packages/coding-agent/src/core/model-config.ts](../packages/coding-agent/src/core/model-config.ts)。

## Key Types / Classes

AssistantMessage、AgentToolResult、McpError/McpTimeoutError/McpAbortError、ExtensionError。

## Key Functions

createErrorToolResult；lazyStream；ModelConfig.load/getError；McpClient.cancelPending；AgentSession 的 retry classifier 调用。

## Control Flow

发现错误 → 局部规范化 → 核心事件/结果 → Session 分类重试或结束 → Host 提示。tool error 可以交给模型修正参数，Provider error 则先走恢复策略。

## Data Flow

error text 进入模型工具结果或 assistant diagnostics；被省略的失败 assistant 仍留 canonical log/context_edit 审计关系。

## State

stopReason、isError、retry attempt、MCP state、diagnostics；没有全局事务回滚状态。

## Design Decisions

**Inference**：结果化错误便于模型继续；跨层字符串分类降低精确性，应避免把所有 error 当可重试。

## Unknowns

第三方 hooks 中自行吞错/修改结果的策略无法由核心保证；未进行故障注入。

## Archify Diagram

本 Batch 不要求独立图；参考相邻机制图，避免把不存在的能力画成实现。

## Follow-up

Batch 29 分层追踪重试，而非只统计 catch。
