# Batch 9：Reasoning / Thinking

## Findings

**Confirmed**：统一配置为 off/minimal/low/medium/high/xhigh/max；reasoning boolean、thinkingLevelMap 与 clamp 决定可用级别，adapter 再映射 effort、thinking budget 或 native thinkingLevel。统一级别不意味着各模型计算量、费用或能力相同。

**Confirmed**：可见推理文字与 opaque/replay signature 分开存储。thinking block 可以无可见正文；签名也可能附在 text/toolCall。只分析流/历史行为，不能由签名推断完整模型内部思维。

## Evidence

| 已读源码 | 定位 |
|---|---|
| [types.ts](../packages/ai/src/types.ts) | ThinkingLevel/Map/Budgets、ThinkingContent、ToolCall.thoughtSignature、Usage.reasoning |
| [models.ts](../packages/ai/src/models.ts) | getSupportedThinkingLevels、clampThinkingLevel；null 级别排除；xhigh/max 必须显式 map |
| [simple-options.ts](../packages/ai/src/api/simple-options.ts) | 默认 budgets、1024 回答余量、context/maxTokens clamp |
| [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) | streamSimple 928 附近：adaptive effort 与旧 budget 分支；thinking/redacted/signature 流 |
| [openai-responses.ts](../packages/ai/src/api/openai-responses.ts) / [shared](../packages/ai/src/api/openai-responses-shared.ts) | reasoning effort/summary、encrypted_content、reasoning items |
| [google-shared.ts](../packages/ai/src/api/google-shared.ts) / [GenAI](../packages/ai/src/api/google-generative-ai.ts) | usesGoogleThinkingLevel、resolveGoogleThinkingLevel、getDisabledGoogleThinkingConfig、thought/signature |
| [openai-completions.ts](../packages/ai/src/api/openai-completions.ts) | thinkingFormat 多协议映射、thinking budget、reasoning_details 重放 |
| [transform-messages.ts](../packages/ai/src/api/transform-messages.ts) / [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) | 跨模型清理；set/cycle/clamp level 和持久化 |

## Key Source Files

配置能力在 types/models，预算在 simple-options，wire 与 parser 在各 API，历史可迁移性在 transform-messages。Session 的 thinking 设置影响下一请求，CLI/TUI 只提供选择入口。

## Key Types / Classes

`ThinkingContent`：thinking、thinkingSignature、redacted；`ThinkingBudgets`：minimal/low/medium/high 可选 token 数；`ThinkingLevelMap`：off/级别 → string/null；`Usage.reasoning` 是 output 子集，不能重复相加；provider-specific effort/config 不等于统一 enum。

## Key Functions

`AgentSession.setThinkingLevel()` / `cycleThinkingLevel()` → clamp；SDK options.reasoning → adapter streamSimple；`thinkingBudgetForLevel()` / `adjustMaxTokensForThinking()`；`transformMessages()`；Responses `processResponsesStream()`；Google `isThinkingPart()`。

## Control Flow

| adapter | 统一选项的 native 映射 |
|---|---|
| Anthropic adaptive | compat.forceAdaptiveThinking=true：thinkingEnabled + output_config.effort；map 优先 |
| Anthropic budget | 默认 minimal=1024、low=2048、medium=8192、high=16384；xhigh/max budget clamp 为 high；共享输出上限，尽量留 1024 回答 token；最终再按 context 限制 |
| Responses | reasoned model 的 effort/summary；include encrypted_content；无请求 effort 时按 off mapping/供应商例外发送 none 或省略 |
| Gemini | 部分模型 ID 走 discrete thinkingLevel；其他走 thinkingBudget；off 不被模型支持时 clamp 为其最低可用级别，不能保证所有模型真正关闭推理 |
| Completions | openai reasoning_effort；OpenRouter nested reasoning；DeepSeek/Zai thinking.type；Qwen enable_thinking/chat_template；Together enabled；Baseten args；预算字段按 compat 选 thinking_token_budget/thinking_budget/thinking_budget_tokens |

Model map 的 null 表示级别不可用；clamp 优先向更高可用级别搜索，再向更低找。扩展或 native options 可直接改最终 payload，通用级别不能保证最终 wire 不被覆盖。

## Data Flow

Provider 流累积 thinking 字符与签名 → assistant message_end → Session JSONL 保存 → 下次请求 transformMessages 判断同 provider/api/model。同模型可保留原生可重放内容；跨模型 redacted/opaque-only 内容丢弃，可见 thinking 退 text，签名按类型移除。错误/aborted assistant 不重放。Completions 的 reasoning_details 是签名槽中的 JSON；不是默认显示出来的增量推理全文。

Google thought=true 才说明 part 是可见推理；只带 thoughtSignature 的普通 text 或 functionCall 仍保持原类型。不得为方便合并而把签名移动到其他 part。

## State

Session 当前 thinking level 与日志 thinking_level_change 分开；模型切换可使用 per-model/default 并 clamp。流 partial 的 thinking block 为可变累积状态，最终保存后还有 provider replay 清理。TUI 收到统一 thinking 事件，但显示策略不改变保存字段的语义。

## Design Decisions

**Inference**：统一 effort 使 UI/Agent 不必了解每家 wire；opaque signature 保留能继续协议所需信息。不同模型的预算/effort 没有同尺度，不能从“high”比较供应商推理能力。级别映射、预算和签名重放是 Harness；推理内容生成是 Model；跨模型保留有损的工作上下文是 Hybrid。

## Unknowns

没有读取、推断隐藏思维；没有运行费用或延迟对比。未完整审计其他 API 的 thinking 分支。模型是否服从预算、返回可见 summary 或需要某签名组合仍需实际协议验证。

## Archify Diagram

任务没有要求单独 Diagram 9；复用 [Diagram 8C](diagrams/diagram-8c-conversion/diagram-8c-conversion.html) 与 [Diagram 4](diagrams/diagram-4-messages/diagram-4-messages.html)。最终图集不虚构可读的隐藏思维节点。

## Follow-up

Tool System 接着分析模型意图如何被验证与执行；Token/Cache/Switching 批次补 reasoned output 计量和签名迁移。
