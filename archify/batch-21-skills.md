# Batch 21 · Skill Lifecycle

## Findings

**Confirmed**：Skill 是带 frontmatter 的指令文件及相关资源，不是代码注册 factory。ResourceLoader/skills.ts 发现、解析、检查名称/描述与冲突；系统提示先提供技能摘要/路径，正文按需进入上下文。显式 /skill:name 由 Session 展开成用户消息中的 skill block。

技能指导工具调用，但没有独立 Skill.execute() 的强制步骤执行器；是否遵守正文是模型能力，发现和展开是 Harness，整体 Hybrid。

## Evidence

- [skills.ts](../packages/coding-agent/src/core/skills.ts)：loadSkills / formatSkillsForPrompt。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：parseSkillBlock / skill command 展开。
- [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)：技能资源发现。
- [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)：提示段落。

## Key Source Files

[packages/coding-agent/src/core/skills.ts](../packages/coding-agent/src/core/skills.ts)、[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[packages/coding-agent/src/core/resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)、[packages/coding-agent/src/core/system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)。

## Key Types / Classes

Skill、SkillDiagnostic、ParsedSkillBlock、ResourceLoader。

## Key Functions

loadSkills；formatSkillsForPrompt；parseSkillBlock；AgentSession 的 skill command 展开逻辑。

## Control Flow

资源路径 → 发现 SKILL.md → frontmatter 验证 → 描述进入 system → 用户 /skill 或模型 read → 正文进入 messages → 工具普通执行。

## Data Flow

名称/描述/location 为摘要；正文和附属文件通过展开或 read 带来 token 成本。不是启动时所有技能全文自动注入。

## State

加载结果和 diagnostics、可选 enableSkillCommands、当前请求消息；没有核心“Skill running/completed”正式状态。

## Design Decisions

**Inference**：按需读取降低提示大小，代价是激活与阅读依赖用户或模型选择。

## Unknowns

不把技能描述视为可验证能力保证；第三方技能指令和运行成本未评估。

## Archify Diagram

[diagram-21-skill](diagrams/diagram-21-skill/diagram-21-skill.html)

## Follow-up

Batch 22 追踪 MCP 的可调用能力。
