# Batch 15 · Coding / Bugfix Workflow

## Findings

**Confirmed**：核心没有固定 search → read → edit → test 的任务状态机。`runLoop()` 仅依据工具调用、输入队列、取消与结束结果推进。模型可以生成这条工作流，但也可以直接回答、跳过测试或重复搜索。

具体路径：用户给 bug → 模型 grep/read → edit → bash 测试失败 → 错误 text 作为 toolResult → 模型修复并再测 → 无工具调用且队列空 → assistant 完成。搜索词、修改方案、测试选择与最终是否足够是模型判断；参数检查、真实执行、错误回灌、日志/取消为 Harness。

## Evidence

[循环条件](../packages/agent/src/agent-loop.ts#L163)、[工具调度](../packages/agent/src/agent-loop.ts#L508)、[Session prompt](../packages/coding-agent/src/core/agent-session.ts#L1921)、[edit](../packages/coding-agent/src/core/tools/edit.ts)、[bash](../packages/coding-agent/src/core/tools/bash.ts)。[plan-mode 示例](../packages/coding-agent/examples/extensions/plan-mode/index.ts) 展示扩展策略，不能当核心必经 Planner 阶段。

## Key Source Files

agent-loop.ts、agent-session.ts、各具体工具；[Agent](../packages/agent/src/agent.ts) 维护运行状态及事件。

## Key Types / Classes

Agent、AgentSession、AgentEvent、AssistantMessage、ToolResultMessage。没有默认 CodingTaskPhase 枚举或任务验收器。

## Key Functions

`AgentSession.prompt()` → `Agent.prompt()` → `runAgentLoop()` → `runLoop()` → `streamAssistantResponse()` / `executeToolCalls()`。

## Control Flow

系统保证模型/工具续轮机制，不能保证每条任务都执行验证。模型不再发工具调用只意味着此轮可结束，不代表用户需求经客观验证。

## Data Flow

命令 exitCode → 工具 content/isError → 模型下一轮。diff、测试日志和最终答案在不同返回层；没有自动把一个 exitCode=0 推导成业务完成。

## State

Agent streaming/pendingToolCalls、Session retry/compaction/queue、日志分支。工作流中“已理解/已修复/已验证”是分析示意，并非核心持久状态。

## Design Decisions

**Inference**：核心容纳多种任务和模型策略，验证策略可以靠指令/扩展加强；若产品需要固定验收，必须在 Harness 另加明确检查，而不是把模型自述当验收证据。

## Unknowns

未运行实际 bugfix 或基准。本报告不宣称模型能稳定完成上述路径。

## Archify Diagram

[15 编码任务流程](diagrams/diagram-15-coding/diagram-15-coding.html)，标注模型自主分支与 Harness 执行边界。

## Follow-up

Batch 16～19 检查长任务的会话与保存；Batch 34～35 检查验证设施。
