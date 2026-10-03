# Batch 25 · CLI / TUI Architecture

## Findings

**Confirmed**：main 解析命令行、stdin/附件、资源/服务和会话，再绑定 interactive/print/RPC 等模式。TUI 管编辑器、快捷键、组件/布局与显示；Session/Agent 核心可 headless 使用。不是所有产品策略都从 TUI 完全抽离：内置 slash command 的 selector/确认/会话导航 Host 行为仍在 InteractiveMode。

## Evidence

- [main.ts](../packages/coding-agent/src/main.ts)：main startup 与模式分发。
- [args.ts](../packages/coding-agent/src/cli/args.ts)：参数解析。
- [interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)：输入/事件/command。
- [print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts)：headless print。
- [rpc-mode.ts](../packages/coding-agent/src/modes/rpc/rpc-mode.ts)：JSON line commands。
- [sdk.ts](../packages/coding-agent/src/core/sdk.ts)：createAgentSession。

## Key Source Files

[packages/coding-agent/src/main.ts](../packages/coding-agent/src/main.ts)、[packages/coding-agent/src/cli/args.ts](../packages/coding-agent/src/cli/args.ts)、[packages/coding-agent/src/modes/interactive/interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)、[packages/coding-agent/src/modes/print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts)、[packages/coding-agent/src/modes/rpc/rpc-mode.ts](../packages/coding-agent/src/modes/rpc/rpc-mode.ts)、[packages/coding-agent/src/core/sdk.ts](../packages/coding-agent/src/core/sdk.ts)。

## Key Types / Classes

InteractiveMode、TUI、Editor、AgentSessionRuntime、AgentSession、RPC commands。

## Key Functions

main；parseArgs；createAgentSession；runPrintMode；runRpcMode；InteractiveMode.init/run 与事件订阅。

## Control Flow

CLI flags + stdin → services/session → mode Host → submit → Session.prompt → Agent stream/tools → Session events → render/JSON/stdout。Ctrl interrupt → Session.abort，而非 UI 自己终止模型协议。

## Data Flow

TUI 组件持有显示状态，输出内容来自消息与工具 details；print/RPC 使用同一核心但不同呈现与 command capabilities。

## State

Editor buffer、历史、折叠、image capability、fullscreen scroll、subscription/dispose；业务 messages 的 owner 仍是核心。

## Design Decisions

**Inference**：可替换 Host 有利于 SDK 嵌入；修改 TUI 显示并不需要修改 Agent Loop。

## Unknowns

未启动交互终端。本工作区原有 InteractiveMode/Bash 组件修改未被编辑，图引用固定 HEAD；不保证未提交 UI 改动经过验证。

## Archify Diagram

[diagram-25a-ui](diagrams/diagram-25a-ui/diagram-25a-ui.html)、[diagram-25b-input](diagrams/diagram-25b-input/diagram-25b-input.html)

## Follow-up

Batch 26 分析命令分发。
