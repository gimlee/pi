# Batch 26 · Commands / Keybindings

## Findings

**Confirmed**：CLI options、内置 slash command、扩展 command、prompt template、skill command、键绑定是不同入口。slash-commands.ts 是内置可发现信息表，不能当所有命令执行集中在该文件。运行态 extension 命令由 Session 优先检查，/skill 与模板展开后才成为普通 prompt；Host 内置命令由 UI dispatch。

## Evidence

- [slash-commands.ts](../packages/coding-agent/src/core/slash-commands.ts)：BUILTIN_SLASH_COMMANDS。
- [keybindings.ts](../packages/coding-agent/src/core/keybindings.ts)：KEYBINDINGS / manager。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：prompt 与 extension command 优先。
- [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)：registerCommand/registerShortcut。
- [interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)：Host 命令 dispatch。

## Key Source Files

[packages/coding-agent/src/core/slash-commands.ts](../packages/coding-agent/src/core/slash-commands.ts)、[packages/coding-agent/src/core/keybindings.ts](../packages/coding-agent/src/core/keybindings.ts)、[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[packages/coding-agent/src/core/extensions/loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)、[packages/coding-agent/src/modes/interactive/interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)。

## Key Types / Classes

BuiltinSlashCommand、SlashCommandInfo、RegisteredCommand、KeybindingsManager、ExtensionCommandContext。

## Key Functions

registerCommand；registerShortcut；KeybindingsManager.create/reload；AgentSession.prompt；expandPromptTemplate。

## Control Flow

输入 /xxx → Host built-in 或 Session extension command → 若未消费，再 input hook → skill/template → prompt。shortcut → configurable manager 匹配 → 对应 action。

## Data Flow

command 参数文本传给 handler；模板参数转用户提示；skill 正文转 skill block；这些不应统称为模型工具。

## State

命令注册 maps、shortcut definitions、keybindings.json override、命令上下文 session replacement 有效期。

## Design Decisions

**Inference**：分离命令与工具使用户可控制会话而无需模型 round trip。权限/网络等副作用仍由具体 handler 决定。

## Unknowns

不同 Host 的 UI commands 不保证同样可用；未运行快捷键冲突测试。

## Archify Diagram

本 Batch 不要求独立图；参考相邻机制图，避免把不存在的能力画成实现。

## Follow-up

Batch 27 逐来源分析配置优先级。
