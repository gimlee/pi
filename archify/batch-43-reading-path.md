# Batch 43 · Reading Path

## Findings

| 时间预算 | 顺序 | 阅读目标 |
| --- | --- | --- |
| 30 分钟 | Batch 1/46 → AgentLoop runLoop → Agent.processEvents → ToolResultMessage | 定位模型/工具续轮和状态owner，先不读所有Provider |
| 2 小时 | 以上 + SDK → AgentSession.prompt/_runAgentPrompt → SessionManager projection → compaction prepare/commit | 看清请求边界、结束边界和上下文 |
| 1 天 | 以上 + transcript/transform → 典型4协议 → edit/queue/bash → ExtensionRunner/Loader → MCP client | 串起真实请求和I/O，注意错误/取消 |
| 3 天 | 以上 + ResourceLoader/Settings → TUI/print/RPC → virtual models/cache → suite/evals → Batch38债务验证 | 按任务做有针对性的运行验证；真实API实验单独安排 |

## Evidence

路径依据 [Batch42](batch-42-top-30-files.md) 的实际入口与关联。

## Key Source Files

路径依据 [Batch42](batch-42-top-30-files.md) 的实际入口与关联。

## Key Types / Classes

沿Agent、Session、Manager、Models、Tool、Runner、Host顺序理解。

## Key Functions

先runLoop，再prompt/projection/compact，再API转换与工具。

## Control Flow

每条路线都先主调用链，再追旁支。

## Data Flow

画清同一用户输入在raw log、projected context、wire payload中的不同表示。

## State

先看owner，再看同步/异步/持久寿命。

## Design Decisions

**Inference**：阅读时间是建议预算，不是实测学习时间。

## Unknowns

读者经验不同；无真实模型实验时只能确认实现契约，不能评价任务可靠性。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch44给至少15条可追踪调用链。
