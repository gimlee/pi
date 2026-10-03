# Batch 5：Prompt System

## Findings

**Confirmed**：模型看到的不是简单 `system + env + project + extension + user` 拼接。Pi 将系统提示分为命名段落，以系统消息记录差分；对话、工具 schema 分别编码；Provider 按能力 replay/折叠系统更新，再构造请求。必须分别看资源来源顺序、系统文字顺序、对话消息顺序和 wire 字段。

**Confirmed**：默认 `buildSystemPromptSections()` 不接收 Model/Provider 参数，没有三套 Claude/GPT/Gemini Coding Prompt。动态部分主要来自 cwd、资源配置、项目指令、skills、工具 loadout 和扩展；供应商层仍存在 wire、thinking、缓存、工具更新以及 Anthropic OAuth 身份前缀差异。

**Confirmed**：SYSTEM/customPrompt 替换默认前缀，仍可附 addendum/project/skills/cwd；before_agent_start 返回 systemPrompt 则成为 forceSystemPrompt，投影到当前 run 请求的首条系统提示，替换整份文字。两种“替换”不是同一种操作。

## Evidence

| 编号 | 已读来源 / 范围 | 直接支持 |
|---|---|---|
| P1 | [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts) 全文件 1～216 | 选项、默认规则、命名段落顺序、custom/force、diff |
| P2 | [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) 167～270、631～666、1201～1228 | prompt 文件/文字读取、AGENTS/CLAUDE 候选、祖先顺序、SYSTEM/APPEND 发现 |
| P3 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 1649～1760 | 资源合并、loadout、隐藏声明与 forced prompt 投影 |
| P4 | 同文件 872～906、1921～2123 | 后续 turn 刷新、输入到最终 user、skill 展开 |
| P5 | [extensions/runner.ts](../packages/coding-agent/src/core/extensions/runner.ts) 1289～1465、1511～1554 | context/context_with_system 两阶段、before_agent_start、input 链式修改 |
| P6 | [skills.ts](../packages/coding-agent/src/core/skills.ts) 345～509；[prompt-templates.ts](../packages/coding-agent/src/core/prompt-templates.ts) 全文件 | skill 元信息与显式正文；模板路径/参数替换 |
| P7 | [sdk.ts](../packages/coding-agent/src/core/sdk.ts) 270～305、314～421 | 图片关闭、request options、onPayload、注入转换/模型流 |
| P8 | [agent-loop.ts](../packages/agent/src/agent-loop.ts) 195～240、333～420 | 工具声明差分与实际请求边界；transform → convert → normalize → streamFn |
| P9 | [transcript.ts](../packages/ai/src/utils/transcript.ts) 全文件；[text.ts](../packages/ai/src/utils/text.ts) | 系统与工具 replay、collapse、rendered updates |
| P10 | [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) 570～900、1124～1310 | 参数构建/onPayload、system OAuth 前缀、thinking/cache/native updates |
| P11 | [openai-responses.ts](../packages/ai/src/api/openai-responses.ts) 305～384；[openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts) 145～350 | input instructions、工具、reasoning、角色与按能力 additions |
| P12 | [google-generative-ai.ts](../packages/ai/src/api/google-generative-ai.ts) 57～100、373～445；[google-shared.ts](../packages/ai/src/api/google-shared.ts) 198～364 | collapsed systemInstruction、contents、tools、thinkingConfig |

证据对应固定仓库实现；不根据在线宣传推断 Prompt，也没有读取本机私有用户配置或实际发送 prompt 来声称运行内容已验证。

## Key Source Files

资源发现：[resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)、[skills.ts](../packages/coding-agent/src/core/skills.ts)、[prompt-templates.ts](../packages/coding-agent/src/core/prompt-templates.ts)。

装配与动态更新：[system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[extensions/runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)。实际出网边界：[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) 和三种已读 API adapter。

## Key Types / Classes

`BuildSystemPromptOptions` / `NormalizedBuildSystemPromptOptions` 保存 custom/force、selectedTools、snippets/guidelines、append、sections、cwd、contextFiles、skills；规范化会复制可修改集合。

`SystemPromptSections` 是有序段落表；`SystemMessage` 保存这些段落及工具差分；`ToolDefinition`/`AgentTool` 分别贡献说明/规则与可执行接口；`ResourceLoader` 提供实际加载资源；`ExtensionRunner` 串联修改；`TranscriptContext` 是最终统一消息输入；Provider-native 参数并不与这些内部类型同形。

## Key Functions

`loadProjectContextFiles`、`discoverSystemPromptFile`、`discoverAppendSystemPromptFile`、`resolvePromptInput` → `AgentSession._rebuildSystemPrompt` → `normalizeBuildSystemPromptOptions` / `buildSystemPromptSections`。

`AgentSession.prompt` → `emitInput` → `_expandSkillCommand` / `expandPromptTemplate` → `emitBeforeAgentStart` → `_preparePromptAndToolLoadout` → `diffSystemPromptSections`。

请求：prepareRequest → `emitContext` → hidden/forced projections → convertToLlm → normalizeContext → ModelRuntime.streamSimple → adapter `buildParams` → onPayload / `emitBeforeProviderRequest` → SDK 请求。细节顺序见下一节。

## Control Flow

### 1. 资源来源与优先级

| 资源 | 已确认的默认行为 |
|---|---|
| SYSTEM.md | 显式 systemPrompt（文件存在则读内容，否则按文字）优先于发现；自动发现先受信任项目 cwd/.pi/SYSTEM.md，再全局 agentDir/SYSTEM.md，选一份 |
| APPEND_SYSTEM.md | 显式 append 源数组优先；没有显式源才发现受信任项目，再全局，选一份；多显式源按数组顺序读取并 join |
| AGENTS / CLAUDE | 全局 agentDir 一份，再从文件系统根到 cwd 的祖先一份/目录；不是只到 git root，也不是全仓递归找每个子目录 |
| 单目录候选顺序 | AGENTS.override.md → AGENTS.md → AGENTS.MD → CLAUDE.md → CLAUDE.MD；第一份可读文件赢，不把同目录五份全部拼接 |
| linked worktree | 特定“主仓祖先包含嵌套 linked worktree”情形，worktree 指令副本可遮蔽主仓同作用域副本，避免重复；不是任意同名文件全局覆盖 |
| Skills | 加载资源最终提供 Skill 列表；standalone loader 默认 global→project→explicit，同名先到赢且产生 collision；disabled-model-invocation 不放模型目录 |
| Prompt templates | standalone loader 扫 global→project→explicit；最终 ResourceLoader 还有资源启用/冲突处理；展开时对最终 templates 用 find，取第一匹配 |
| Loader overrides | systemPromptOverride、appendSystemPromptOverride、agentsFilesOverride 等可替换加载结果；SDK 可注入 ResourceLoader |

Project trust 影响自动发现 SYSTEM/APPEND 和项目可执行资源装载；`loadProjectContextFiles()` 自身按 cwd/agentDir 找文本，没有同一 trusted 条件。本批不将可执行资源的信任门套到所有 AGENTS 文本上。

### 2. 默认系统文字的真实顺序

**Confirmed**：从 buildSystemPromptSections 的插入顺序得到以下顺序，存在的段落才输出；正文间以空行连接。

| 次序 | 段落 | 来源 / 条件 |
|---|---|---|
| 1 | preamble，未包 XML 标签 | 默认 expert coding assistant；customPrompt 非空时使用其正文，并跳过下面 tools/rules/docs 默认段 |
| 2 | tools | selectedTools 中有 snippet 且可见的工具；附“可能还有项目 custom tools”的提示；不是完整 JSON schema |
| 3 | rules | 工具组合决定的 shell 搜索建议 → 按 selectedTools 顺序的 toolGuidelines → promptGuidelines → 简洁与显示路径规则；trim/去重 |
| 4 | docs | 本产品 README/docs/examples 的资产路径与使用规则，通过 config helpers 解析 |
| 5 | addendum | appendSystemPrompt；来自用户配置/APPEND 来源，位于项目与 skills 前 |
| 6 | project_context | 逐个 `<project_instructions path="…">` 包装全局/祖先文件正文 |
| 7 | skills | 仅有 read 或 bash 且存在可见 skills；目录含名称、description、location 和按需读取规则，不含全部 SKILL.md 正文 |
| 8 | cwd | 规范化为斜杠的工作目录路径 |
| 9 | 自定义 sections | 按输入表遍历新增；可替换已存在的非 preamble 段，此时已有键位置保留；空 content 不覆盖 |

非 preamble 段再整体包 `<段名>…</段名>`。自定义段名必须匹配小写合法名且不得为 preamble。自定义段落不是只能附最后：替换 project_context/cwd/rules 等会改变已有段正文。forceSystemPrompt !== undefined 时完整渲染直接返回 forced content，包括空串，不是追加段。

环境信息的默认直接文字是 cwd、文档资产路径、可用工具及其说明/规则；没有在该 builder 内自动加入当前日期、完整环境变量、git branch、仓库索引或所有源码。工具选择、Settings/加载路径也间接体现 OS/用户配置；不能据此声称“整份环境已注入”。

### 3. 用户与扩展的实际顺序

空闲的普通 prompt 路径是：扩展 `/command` 检查 → compaction 状态保护 → input handlers → `/skill:name` 展开 → `/template args` 展开 → flush 待处理消息 → 模型/鉴权与压缩检查 → before_agent_start → 图片规范化 → 构造消息 → loadout/系统 patch → Agent。

input 是链式 transform，handled 短路。skill 显式命令读正文、剥 frontmatter，包 skill block 并附参数；template 做非递归 $1/$@/slice/default 替换。普通提问只含技能目录，模型决定是否调用 read/bash 获取技能正文。expandPromptTemplates=false 同时关闭扩展命令优先与这两种展开。

before_agent_start 各 handler 按顺序共享规范化 options，可直接修改 selectedTools/guidelines/sections 等；返回 message 累积为 custom；返回 systemPrompt 设置 forceSystemPrompt，后续返回可覆盖。handler 未显式改 selectedTools 时，setActiveTools 对实时 loadout 的修改仍有效。先运行这些 hooks，再按最终选型处理图像。

发给 Agent 的新消息顺序：`[可选 system sections patch] → user(text + image) → pending nextTurn messages → before_agent_start custom messages`。这是新消息顺序，不是整个历史只有这些；Agent 会先与已有 context 接合。display=false 的扩展 custom 仍经 convertToLlm 成为 user 内容。

Streaming 输入只走入队路径；不是对每条队列输入立即重跑整套 before_agent_start。下一 turn 会 refresh tool loadout/prompt，运行选项跨该 run 保持；before_agent_start 的强制文字不作为新的 raw transcript 全量写入。

### 4. 从系统段落到真实 request

1. `_preparePromptAndToolLoadout()` 应用实际工具集合，对历史 replay 后的 sections 做 diff；输出 system patch。Loop `declareToolChanges()` 在请求前声明可执行集合的增删，替换 pending system 上的工具意图，保证模型声明匹配当前执行集合。
2. `prepareRequest` 每次从 SessionManager.buildSessionProjection 形成 canonical context；已有自定义 prepare hook 可以再替换，虚拟模型可路由到 physical model。
3. `transformContext` 默认 SDK 调用 ExtensionRunner.emitContext：先 structuredClone；context handlers 只见非 system 对话，改列表后恢复 replay 系统状态；再 context_with_system handlers 看全 transcript，返回结果被尊重。删除首 system 会报错但不强行恢复。
4. Session 安装的隐藏声明投影进一步滤掉 hidden 工具；forced prompt 投影随后将所有 system 合成一个首 system，content=forced，保留当前工具声明。它在上述 context hooks 之后生效。
5. convertToLlm 转 Coding roles（必要时 SDK 禁用图像再替换），normalizeContext 形成统一 transcript；streamFn 经 ModelRuntime 做鉴权/Provider 分发。
6. Provider resolve/collapse 系统历史、transformMessages、render system updates，构造原生 payload。tools schema 在原生 tools 字段，不是默认 tools 段落文字。对话原生 input/messages/contents 与系统字段按协议分别存在。
7. adapter 在发请求前 await onPayload；SDK 连到 before_provider_request。handler 按链修改最终 payload，这是文字/字段的最后可变边界；headers 有另一路 before_provider_headers。所谓“模型确切收到”不能只打印 buildSystemPrompt。

下一 turn：prepareNextTurnWithContext 可先压缩，调已有 hook，再更新 selectedTools/snippets/guidelines、生成 sections patch，并保持 session.systemPrompt 同步。没有变化就不额外生成 patch。系统 replay 的 content 是累加，sections 是按名替换/null 删除，tools 是按名增删；普通替换与增量更新要分清。

## Data Flow

`资源文件/用户配置 + 工具说明/规则 + cwd → base options → before_agent_start options → ordered sections → 系统 patch + 工具 declarations + Conversation → request projection/hooks → Provider payload → onPayload → API`。

forceSystemPrompt 是当前 run 的 request projection；raw transcript 仍持久化结构化 sections 差分。后续恢复重放不等价于永久保存 forced string。资源 reload、工具增删、扩展状态改变可使下次 desired sections 不同；不是每个 token delta 重新构造 prompt。

### Dynamic Prompt 与 Provider-specific Prompt

| 因素 | 文字/结构变化 | 范围与证据等级 |
|---|---|---|
| Tools | 说明、规则、skill 可读性随 active loadout；声明新增/移除；hidden 工具不直接声明 | **Confirmed**：Session、builder、Loop |
| Repository | cwd/祖先 context files；worktree 避免同作用域重复 | **Confirmed**：ResourceLoader；不是默认全库索引 |
| User Config | SYSTEM/APPEND、显式资源、overrides、selected tools | **Confirmed**：加载与 builder |
| Environment | cwd/资产路径与装载配置间接变化；shell 工具差异 | **Confirmed**：默认文字范围；其他环境注入靠配置/扩展 |
| Model / Provider | builder 无参数分支；extension context 能读当前 model，自行定制；模型能力影响请求角色、图像与签名 | **Confirmed**：不是三套默认 Coding Prompt |
| Claude / Anthropic | 初始 system 顶层 blocks；OAuth 再在前加入 Claude Code 身份文字，工具名称按规范映射；API-key 路径不加该前缀 | **Confirmed**：实际文字差异，来自 adapter；名称表不创建 Task/Todo 工具 |
| Anthropic native updates | compat 允许时保留 system 更新，转换时延后到下一 assistant 前以保护 tool/result 紧邻；支持 native tool changes 且有 initial tools 时使用 inline add/remove 与 placeholder；否则折叠/current tools | **Confirmed**：按能力有条件，不适用于全部 Claude endpoint |
| GPT / Responses | reasoning 模型且支持 developer 时使用 developer，否则 system；后续系统按 supportsMidConvoSystemMessages 保留/合并；additions/search 按 compat 和非删除/非重定义条件选择 | **Confirmed**：Responses，未泛化到所有 GPT API |
| Gemini | 总是 collapse system；config.systemInstruction + contents；functionDeclarations；thinkingConfig 独立字段 | **Confirmed**：Google Generative AI / shared converter |
| Thinking / Cache | adapter 映射 effort/budget、encrypted signatures、cache markers/key 与 session affinity；可能改变 payload 或重放系统项 | **Confirmed**：协议层 Harness 适配；不是在通用 prompt 中写“请更认真思考” |
| 最终 Payload Hook | before_provider_request 能再次修改 native payload | **Confirmed**：不能保证强制文字不被后续 hook 改掉 |

**Inference**：真正的模型适配同时存在于数据重放和请求协议，不应只用“有没有 Claude 专属系统文字”衡量 Harness 厚度。OAuth 身份前缀是已经确认的文字特例，不是代码推理/规划能力的证明。

## State

`_baseSystemPromptOptions` 保存资源基线；`_runSystemPromptOptions` 保存本次 before_agent_start 后选项，直到 Session run finally 清空；transcript 保存系统 patch；`getCurrentSystemMessage` replay 得到模型当前段落/工具状态；每请求 projection 可与 raw history 不同。selectedTools、hiddenDeclarations、工具 registry 和 schema declarations 不应合为一个数组。

## Design Decisions

- **Confirmed**：命名段落差分与历史中的工具变更。**Inference**：提示可复查、动态更新；支持中途系统项的模型可保持既有前缀，有利缓存，具体收益没有本批量测。
- **Confirmed**：技能目录与按需读取/显式展开。**Inference**：避免所有技能正文一开始占满上下文；Harness 提供发现，模型选择何时读取。
- **Confirmed**：context 与 context_with_system 权限范围不同；forced prompt 更晚投影。**Inference**：普通上下文裁剪较不容易意外丢掉提示/工具，仍允许显式高级改写。
- **Confirmed**：tool snippets/rules 与 schema 分开。**Inference**：文字负责使用建议，schema 负责可调用接口，两者都必须与实时工具集合同步。

构造、来源、差分、工具声明和供应商编码是 Harness capability；理解规则/按需选择技能是 Model capability；在项目约束下执行任务是 Hybrid。

## Unknowns

没有读本机私有 agentDir、实际用户 settings/扩展组合，也没有发网络请求或截取实际 payload。未完整审计所有 API、工具定义贡献的全部规则、package 资源冲突策略及上下文压缩后的提示恢复；这些留对应批次。无法从拼接顺序保证模型遵守规则，或证明所有供应商对 native 中途系统项有同等语义。

## Archify Diagram

- [Diagram 5A：Prompt Composition Data Flow](diagrams/diagram-5a-composition/diagram-5a-composition.html) · dataflow，来源、系统段落/声明、请求投影和原生 payload。
- [Diagram 5B：Prompt Build Workflow](diagrams/diagram-5b-build/diagram-5b-build.html) · workflow，输入展开、前置 hooks、patch、请求 hooks 与实际发送顺序。

候选和自动检查记录在各图目录；最终汇总见 [README](README.md)。

## Follow-up

Batch 6 完整分析 Context/Compression；Batch 7 及后续 Provider/Compatibility 批次补齐模型抽象、其他 API 和兼容分支；Tool/Extension/Skill/Config 批次审计注册贡献、加载优先级、资源更新与 hooks 合约。后续 Q3 汇总须基于这些审计，不能在本批量化整个仓库适配规模。
