# Batch 24 · Concurrency / Cancellation

## Findings

**Confirmed**：并发由多种机制实现，不能说只有单线程、也不能说每个工具都 worker 化。默认工具批次 Promise 并行；显式 toolExecution=sequential 或任一 sequential 工具使整批串行；文件队列再按 canonical path 排序。模型流、输入队列、MCP background connect 与 cache warming 可同时存在；Shell/stdio 是 OS 子进程，Codemode 是 QuickJS sandbox worker。

## Evidence

- [agent-loop.ts](../packages/agent/src/agent-loop.ts)：executeToolCallsParallel / Sequential。
- [file-mutation-queue.ts](../packages/coding-agent/src/core/tools/file-mutation-queue.ts)：per-file queue。
- [execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts)：CodemodeSandbox / limiter。
- [worker.ts](../packages/coding-agent/src/extensions/codemode/worker.ts)：worker entry。
- [index.ts](../packages/coding-agent/src/extensions/mcp/index.ts)：Promise.all connects。
- [cache-warmer.ts](../packages/coding-agent/src/core/cache-warmer.ts)：timer 与 abort controller。

## Key Source Files

[packages/agent/src/agent-loop.ts](../packages/agent/src/agent-loop.ts)、[packages/coding-agent/src/core/tools/file-mutation-queue.ts](../packages/coding-agent/src/core/tools/file-mutation-queue.ts)、[packages/coding-agent/src/extensions/codemode/execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts)、[packages/coding-agent/src/extensions/codemode/worker.ts](../packages/coding-agent/src/extensions/codemode/worker.ts)、[packages/coding-agent/src/extensions/mcp/index.ts](../packages/coding-agent/src/extensions/mcp/index.ts)、[packages/coding-agent/src/core/cache-warmer.ts](../packages/coding-agent/src/core/cache-warmer.ts)。

## Key Types / Classes

AbortController、Agent activeRun、PendingRequest、CodemodeSandbox、per-file Promise chain。

## Key Functions

executeToolCallsParallel；withFileMutationQueue；ctx.executeTool；CacheWarmer.refresh/cancel；McpClient.cancelPending。

## Control Flow

顶层一次 Agent run → 模型流 → 多工具 Promise → 有序回灌；steering 可以改变尚未准备的后续调用，但已运行并行操作不回滚。Session.abort 触发控制器并等待清理。

## Data Flow

跨 worker 使用消息/序列化桥，nested tool 执行回到宿主统一管线；脚本模型调用 limiter 最多 4 个同时进行，不能推导所有工具有全局 4 并发上限。

## State

Agent、Session、重试、压缩、Bash、MCP、warmer 各有控制器/清理责任；取消为合作式，忽略信号的自定义工具可能延迟结束。

## Design Decisions

**Inference**：局部队列避免全局阻塞，但共享注册表、异步发布和旧 ctx 需要 generation/stale 防护。

## Unknowns

没有压力测试、泄漏检测或外部工具取消保证；未审计 pi-codemode 全部 VM 内部实现。

## Archify Diagram

[diagram-24-concurrency](diagrams/diagram-24-concurrency/diagram-24-concurrency.html)

## Follow-up

Batch 25 检查 UI 如何消费同一核心。
