# Batch 10 · Tool Runtime

## Findings

**Confirmed**：工具定义、模型可见声明、可调用集合和一次执行不是同一个对象。`AgentSession` 维护定义注册表，工具的 exposure 决定是否直接声明；`wrapToolDefinition()` 生成 AgentTool，底层循环按名称定位、修复参数、验证 schema、执行 hooks、运行工具、回灌结果。MCP 和扩展工具经过同一执行管线。

工具执行是 **Hybrid**：模型选择工具、参数和调用顺序；Harness 提供可用声明、参数验证、调度、取消、错误转换和结果关联。权限策略可由 `beforeToolCall` / 扩展 `tool_call` 阻断，不是每个文件操作内置一个批准弹窗。

## Evidence

- [调度与结果](../packages/agent/src/agent-loop.ts#L508)：配置 `toolExecution=sequential` 或存在任一 `executionMode=sequential` 时整批串行，否则并行；并行完成事件可以交错，最终消息按调用顺序回灌。
- [参数及 hooks](../packages/agent/src/agent-loop.ts#L693)：prepareArguments 在验证前运行；未知工具、验证失败、执行异常成为关联的错误结果。
- [单次执行](../packages/agent/src/agent-loop.ts#L810)：`runToolCall()` 复用同一管线，支持非直接模型入口。
- [包装](../packages/coding-agent/src/core/tools/tool-definition-wrapper.ts)：UI renderer 不属于 AgentTool 的执行接口；调用时注入 ExtensionToolContext。
- [Session 工具管理](../packages/coding-agent/src/core/agent-session.ts#L1453)、[注册](../packages/coding-agent/src/core/extensions/loader.ts#L287)。

## Key Source Files

[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent/types.ts](../packages/agent/src/types.ts)、[extensions/types.ts](../packages/coding-agent/src/core/extensions/types.ts)、[wrapper.ts](../packages/coding-agent/src/core/extensions/wrapper.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)。

## Key Types / Classes

AgentTool、ToolDefinition、ToolExposure、AgentToolResult、AgentToolCallOutcome、ToolResultMessage、ExtensionToolContext。`structuredContent` 面向程序调用；`content` 面向模型；`details` 面向 UI/诊断。三者不能互换。

## Key Functions

`wrapToolDefinition()` → `prepareToolCallArguments()` → `prepareToolCall()` → `executePreparedToolCall()` → `finalizeExecutedToolCall()` → `createToolResultMessage()`。

## Control Flow

assistant 含 toolCall → 选择串行/并行 → 查名称与参数验证 → before hook → execute/onUpdate → after hook → tool_execution_end → message_start/end → 下一轮。被阻断的工具仍有明确结果，不留下无结果调用。

## Data Flow

ToolDefinition.parameters → Provider 工具 schema → toolCall.arguments → 验证后的参数 → AgentToolResult → ToolResultMessage。消息保留 callId、name、content、details、isError、nestedCalls；程序结构化返回不自动复制为模型正文。

## State

注册表、active/callable 名称集合、pendingToolCalls、批次 AbortSignal、每次执行结果与 nestedCalls。只有一批所有最终结果都 `terminate=true` 才结束工具续轮，单个 terminate 不足以结束混合批次。

## Design Decisions

**Inference**：统一执行管线让直接工具、MCP、Codemode 中的嵌套调用共享拦截策略；定义与声明分离降低一次请求携带的工具数量。输出 schema 不意味着执行结果已经获得完整业务语义验证。

## Unknowns

没有执行真实模型或权限扩展实验；第三方工具内部是否遵守 AbortSignal 取决于实现。全局强制权限沙箱不能从这一接口推导。

## Archify Diagram

[10A 工具架构](diagrams/diagram-10a-tools/diagram-10a-tools.html)、[10B 调用顺序](diagrams/diagram-10b-call/diagram-10b-call.html)、[10C 结果回灌](diagrams/diagram-10c-result/diagram-10c-result.html)。

## Follow-up

Batch 11～15 检查内置操作的边界；Batch 20、22、24 说明注册、MCP 与并发。
