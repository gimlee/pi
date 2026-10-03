# 《Pi Agent 源码深度研究报告》

分析基线：2026-10-03；HEAD `9b3c19da5cffc4c5e8b6bd74c45abc1ab6bfcd16`。Confirmed 是已阅读实现，Inference 是架构解释，Unknown 是未验证边界。未运行项目测试/真实模型；图只有自动校验，不声称人工视觉审查。当前工作区已推进；差异与新版本 MCP 局部覆盖见 [基线推进说明](baseline-drift.md)。

## 1. Pi 到底是什么

**Inference**：以默认 CLI 和编码 SDK 为分析对象，Pi 首先是 **Coding Agent 与 Agent Harness**。Harness 是模型外部的执行环境：负责输入处理、上下文、工具、会话、恢复和扩展。它包含一个可复用的 **Agent Runtime**，CLI/TUI/RPC 是面向不同调用者的 Host。仓库同时提供框架式的组合/扩展 API，但不能把所有包都视为默认编码 Runtime。

#### 定位依据

| 定位 | 判断 | 直接观察到的行为 |
|---|---|---|
| Model Wrapper | 有这一层，但不足以描述默认 Pi | `ModelRuntime.streamSimple()` 准备鉴权并分发 Provider；其上有 Session、循环和工具执行 |
| Agent Loop | 是必要内核 | `runLoop()` 多次发起模型请求，在工具结果、队列与 hooks 的驱动下继续 |
| Agent Runtime | 是可复用执行层 | `Agent` 持有 active run、消息、工具、队列和事件，`streamFn` 可注入 |
| Coding Agent | 是默认应用形态 | SDK 安装 coding Session，内置 read/bash/edit/write 等定义，加载当前项目资源 |
| Agent Harness | 是主要系统价值 | 请求投影、工具装载、重试/压缩、树形会话、资源/扩展事件均由 Pi 编码层协调 |
| CLI Agent | 是产品入口 | `main` 选择 interactive/print/RPC，不同模式复用 `AgentSession` |
| Agent Framework | 有相关能力；“整个仓库只是一套框架”不充分 | Agent/SDK/Extension 公共 API，以及独立 Chord/durable API；默认产品有明确启动和编码工作流 |
| Model Harness | 可作局部描述 | thinking/image/request hooks/virtual route 等管理模型使用，但不覆盖文件工具和用户交互 |

以上分类是 **Inference**；右列行为及下文调用点是 **Confirmed**。

#### 一条具体执行轨迹

问题：裸 LLM API 能返回一个 `read` tool call，却不会自动知道 Pi 的当前会话、工具实例、上下文投影或终端状态。

**Confirmed** 的缩短轨迹：

```text
Host 收到用户文本
  → AgentSession.prompt()
    → 扩展/input、skill/template、请求前上下文与提示/工具装载
    → _runAgentPrompt() → Agent.prompt()
      → runAgentLoop() → runLoop()
        → streamAssistantResponse() → 注入的 streamFn
          → ModelRuntime.streamSimple() → Provider.streamSimple()
        ← assistant.content 中的 toolCall
        → executeToolCalls()：预检、验证、execute、完成 hooks
        → ToolResultMessage 加回 currentContext.messages
        → 下一轮模型请求或退出底层循环
    → 编码层继续处理恢复、队列与 settle
  ← AgentSession events → Host
  → message_end → SessionManager.appendMessage()
```

这不是新造的 `Agent.run()` API。实际入口是 `prompt()` / `continue()`，实际 while 循环是 `runLoop()`。上图为定位所需的概览；参数验证细节、所有继续/结束条件留给 Batch 3。

#### 价值分布

不按代码行数分配价值百分比；这些类别是阅读和改动风险的功能分区，同一文件可跨类别。

| 类别 | 关键源码 / 行为 | 为什么影响结果 |
|---|---|---|
| Core Intelligence / Harness | `agent-loop.ts`；`AgentSession.prompt`、request/next-turn/tool hooks；Session projection、压缩协调 | 决定模型看到什么、工具能否执行、何时继续/恢复/结束；这里的 intelligence 是行为控制，不是模型权重或训练 |
| Infrastructure | SettingsManager、SessionManager、config paths、output guard、HTTP dispatcher | 配置、会话恢复和输出协议可靠性；错误会改变有效模型/工具和恢复行为 |
| Integration | ModelRuntime、Provider 工厂/API adapter、MCP、codemode | 连接模型/外部工具，转换鉴权、事件、schema 与请求约束 |
| Presentation | InteractiveMode、TUI 组件、print/RPC 输出 | 交互编辑、流展示、命令入口；TUI Host 也处理命令和资源准备，不是完全被动的 View |
| Glue Code | `cli.ts`、services/SDK 工厂、`main` 的分发连接 | 大量工作是装配，但顺序、cwd 和注册时机影响行为；不能把整个 `main` 简单视为可随意删除的样板代码 |

职责存在性为 **Confirmed**；“核心价值/改动风险更高”属于 **Inference**。没有通过性能测试或用户数据量化价值。

#### Model capability / Harness capability / Hybrid

| 能力 | 归属判断 | 当前证据与限制 |
|---|---|---|
| 推理、计划文本、选择下一步工具 | Model capability | Pi 消费 assistant 文本和 toolCall；未在已读主路径发现必须经过的独立规划器，不据此否认扩展可实现规划 |
| 根据任务选择文件/搜索参数 | Hybrid | 模型选择 toolCall，Pi 提供工具定义并执行；具体搜索与仓库理解算法待 Tool Batch |
| 工具执行、结果规范化与回灌 | Harness capability | `executeToolCalls`、Session tool hooks、ToolResultMessage；参数选择是模型部分 |
| 修改代码 | Hybrid | 模型生成工具参数，Pi 执行 edit/write；修改算法、原子性和冲突检测尚未深审 |
| 上下文压缩 | Hybrid | Pi 决定时机和保留结构，内置 compact 路径可调用模型生成摘要；摘要质量来自模型，投影/日志机制来自 Harness |
| 模型切换与鉴权 | Harness capability | Session / ModelRuntime 管理选型、thinking 和鉴权；不等于训练模型 |
| 错误恢复 | Hybrid | Pi 有自动 retry/overflow/continue 协调；模型利用错误 tool result 修订行动；具体分类与上限留后续审计 |
| 会话继续、树导航、fork | Harness capability | SessionManager 和 Runtime 提供日志树/替换，独立于模型生成能力 |
| 输入排队、取消、事件与 Host 协议 | Harness capability | Agent 队列/AbortController，Session 与 TUI/print/RPC 的订阅和输入管理 |

归属为 **Inference**，所列执行点为 **Confirmed**。

专题证据与完整控制/数据/状态边界：[Batch 1](batch-1-positioning.md)。

## 2. Pi 的设计哲学

**Inference**：从可替换streamFn、追加树投影、共享工具执行管线与Extension hooks可推导：核心提供运行契约，策略可组合；保留canonical协议再做原生转换；让模型决定任务行动，让Harness管理可审计执行。不是源码声明的哲学口号。

| 模式 | 具体问题 / trace | 实现 / 必要性 |
| --- | --- | --- |
| Adapter | 不同供应商返回角色/工具/流形状不同 → canonical messages/events | [openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[tools.ts](../packages/coding-agent/src/extensions/mcp/tools.ts)；必要协议边界 |
| Strategy / callbacks | 模型 streamFn、tools execute、context hooks 可替换 | [agent.ts](../packages/agent/src/agent.ts)；Loop 不依赖单一 SDK |
| Factory | 工具和 Provider 需要 cwd/options/auth 配置后形成对象 | [index.ts](../packages/coding-agent/src/core/tools/index.ts)、[all.ts](../packages/ai/src/providers/all.ts) |
| Registry | 动态注册 tool/provider/command；按名称查调用 | [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) |
| Observer | 一次流要驱动状态、持久化和多个 Host | [agent.ts](../packages/agent/src/agent.ts)、[event-bus.ts](../packages/coding-agent/src/core/event-bus.ts)；typed 核心流与任意频道 bus 不同 |
| Command | 用户 /xxx 与模型工具均封装行为但入口不同 | [slash-commands.ts](../packages/coding-agent/src/core/slash-commands.ts)、[types.ts](../packages/coding-agent/src/core/extensions/types.ts) |
| Dependency injection | SessionServices/SDK 注入 stores、tools、streamFn；测试替换为 faux | [sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[harness.ts](../packages/coding-agent/test/suite/harness.ts) |
| Append-log / projection | 分支保留历史，请求只看 active leaf/压缩尾部/edits | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)；不是所有状态事件溯源 |
| Generation / stale guard | 旧 refresh 或 extension ctx 完成晚于替换 → 拒绝旧发布/动作 | [models.ts](../packages/ai/src/models.ts)、[loader.ts](../packages/coding-agent/src/core/extensions/loader.ts) |
| Local serialization | 并行工具写同文件 → per-path Promise chain | [file-mutation-queue.ts](../packages/coding-agent/src/core/tools/file-mutation-queue.ts)；不提供跨进程事务 |

专题证据与完整控制/数据/状态边界：[Batch 37](batch-37-design-patterns.md)。

## 3. Runtime

**Confirmed**：任务说明中的 `CLI → Args → Config → Environment → Runtime → Model → Tools → Extensions → Session → Agent` 是调查清单，不是本仓库的真实执行顺序。默认 CLI 先选择/恢复 `SessionManager` 并确定实际 `cwd`，再创建服务、加载扩展注册、选择模型，随后 SDK **先创建 Agent，再创建 AgentSession**，最后由 `AgentSessionRuntime` 包装当前 Session。

**Confirmed**：扩展初始化至少有两个可观察阶段：ResourceLoader 加载 factory 并收集注册；运行模式 Host 后续调用 `session.bindExtensions()` 绑定 UI/命令/退出能力并触发 `session_start`。MCP 在 `session_start` 可继续异步连接，不能将“扩展已加载”解释成“全部外部工具已就绪”。

**Confirmed**：应用生命周期分层拥有。`main`/Host 管理进程入口、I/O 和退出；`AgentSessionRuntime` 管理当前 Session 及 services 的替换；`AgentSession` 管理编码行为；`Agent` 管理当前 active run。`ExtensionRunner` 不是整个应用的 owner，`SessionManager` 也不是执行 Agent 的对象。

专题证据与完整控制/数据/状态边界：[Batch 2](batch-2-startup-runtime.md)。

## 4. Agent Loop

**Confirmed**：真正循环是 `packages/agent/src/agent-loop.ts` 的 `runLoop()`，不是 CLI 输入循环，也不是名为 Agent 的类本身。内层处理工具调用和 steering，外层处理本来将结束时到来的 follow-up；`Agent` 管理这次运行的状态、取消信号和事件订阅。

**Confirmed**：编码产品还有一层 `AgentSession._runAgentPrompt()`：底层运行结束后检查重试、压缩恢复、剩余队列和 `agent_before_settle`，必要时调用 `Agent.continue()`。因此，一次用户 prompt 可以包含多个底层 run；每个 run 又可以包含多个 turn，每个 turn 有一次主模型响应及其工具结果。

**Confirmed**：模型决定输出什么工具名和参数，Harness 决定是否允许执行、如何验证、执行顺序、异常转结果及是否发起下一请求。Provider 解析协议流；Loop 消费统一事件，不直接解析供应商 HTTP/SSE 数据。

**Inference**：它是模型驱动的“响应 → 行动 → 观察 → 再响应”循环，可称为 ReAct 风格，但没有固定的 ReAct 文本语法或必须先产生计划的算法。停止是执行协议上的停止，不能证明用户目标已被语义验证。

专题证据与完整控制/数据/状态边界：[Batch 3](batch-3-agent-loop.md)。

## 5. Message Model

**Confirmed**：Pi 采用自己的统一消息模型，`Message = SystemMessage | UserMessage | AssistantMessage | ToolResultMessage`。工具结果 role 是 `toolResult`；没有独立的通用 `ToolMessage` 类、`ReasoningBlock` 标签或 `ErrorMessage` role。推理内容用 `type="thinking"`，失败用 assistant.stopReason/errorMessage 或 toolResult.isError 表示。

**Confirmed**：系统消息同时声明文本、命名段落和工具变更；它不是只有 role/content 的简单 OpenAI 消息。Assistant 保存 api/provider/model、签名、usage、停止原因等，支持跨供应商历史重放。

**Confirmed**：Coding Agent 通过声明合并扩展 `AgentMessage`，增加 bashExecution、custom、branchSummary、compactionSummary；在请求边界转换成上述统一 Message。模型输出的 toolCall、工具执行结果和模型看到的 ToolResultMessage 是三个不同数据对象。

专题证据与完整控制/数据/状态边界：[Batch 4](batch-4-message-model.md)。

## 6. Prompt

**Confirmed**：模型看到的不是简单 `system + env + project + extension + user` 拼接。Pi 将系统提示分为命名段落，以系统消息记录差分；对话、工具 schema 分别编码；Provider 按能力 replay/折叠系统更新，再构造请求。必须分别看资源来源顺序、系统文字顺序、对话消息顺序和 wire 字段。

**Confirmed**：默认 `buildSystemPromptSections()` 不接收 Model/Provider 参数，没有三套 Claude/GPT/Gemini Coding Prompt。动态部分主要来自 cwd、资源配置、项目指令、skills、工具 loadout 和扩展；供应商层仍存在 wire、thinking、缓存、工具更新以及 Anthropic OAuth 身份前缀差异。

**Confirmed**：SYSTEM/customPrompt 替换默认前缀，仍可附 addendum/project/skills/cwd；before_agent_start 返回 systemPrompt 则成为 forceSystemPrompt，投影到当前 run 请求的首条系统提示，替换整份文字。两种“替换”不是同一种操作。

专题证据与完整控制/数据/状态边界：[Batch 5](batch-5-prompt-system.md)。

## 7. Context

**Confirmed**：原始 JSONL 历史、当前 branch、模型请求 projection 是三层数据。`buildSessionPath()` 从 leaf 沿 parentId 回溯；`buildContextEntries()` 选择最近压缩边界；`buildSessionProjection()` 应用 context_edit 的最后一次替换/省略。其他分支、纯 custom 状态、label/model/thinking 等元数据不作为对话正文全部发送。

**Confirmed**：压缩不是删除日志。新增 CompactionEntry，保存 summary、firstKeptEntryId、tokensBefore、usage/details，以及边界的完整 systemMessage；投影生成系统状态 + 摘要 + 保留尾部 + 边界后的消息。Provider 仍可进一步折叠系统更新和清理跨模型内容。

**Confirmed**：token 估算优先使用有效 assistant usage，再补之后的消息估算；context edit/新 compaction 会使旧 usage 失效。没有逐模型 tokenizer 的精确客户端计数，chars/4 与图片 4800 字符只是启发式，不能保证所有语言或图像都保守高估。

专题证据与完整控制/数据/状态边界：[Batch 6](batch-6-context-management.md)。

## 8. Context Compression

#### 触发、执行和恢复

| 入口 | 条件 | 行为与边界 |
|---|---|---|
| 自动 threshold | enabled 且 contextTokens > contextWindow - reserveTokens | 不是固定百分比；默认 reserve=16384、keepRecent=20000，Settings 可按模型调整 |
| 自动 overflow | 同一实际模型的 overflow 错误/有效 usage；旧边界及已省略尝试被排除 | 完成的 stop 响应保留，只压缩；失败尝试先用 context_edit 省略，再压缩后 continue，最多一次恢复 |
| recoverable length | 同模型、响应仍在投影、输出不足原本期望限制等检测条件 | 不是任何 length 都压缩；采用同一一次恢复预算 |
| 手工 /compact、RPC、扩展 | compact(customInstructions) | 先 await abort，单独建 signal；不自动恢复被打断的 turn |
| 下一 turn 的中途检查 | prepareNextTurn 边界 | 可在工具轮之间使用投影 usage/估算压缩，不等待整个用户活动结束 |

准备从已编辑的模型可见消息生成；不把 system 当作普通旧对话摘要。向后累计近期预算，合法切点为 user-like 或 assistant，不从 toolResult 开始，避免丢掉它的 toolCall。若切在 turn 内，分别生成历史摘要和 turn-prefix 摘要，再合并；调用顺序为先历史、再前缀，并非并行。近期预算是近似保留目标，不能保证恰好 20000 tokens。

`session_before_compact` 可以 cancel 或提供自定义 CompactionResult。默认生成器使用独立摘要 system、文本化对话、可选前一摘要和自定义重点；输出上限为 min(0.8 × reserveTokens, model.maxTokens)，前缀为 0.5 × reserveTokens。明确拒绝 error、length 和 toolCall 摘要；Session 还在落盘前检查取消。`completeSummarization()` 使用 retryAssistantCall，cacheRetention=none，并提供独立路由 sessionId。

成功后 appendCompaction → refresh finalized context → session_compact → compaction_end。自动失败发可见错误/失败事件并返回 false；手工失败还向调用方 throw。失败没有新的摘要边界，但恢复路径此前写入的 omission edit 仍属于日志，不是事务整体回滚。

专题证据与完整控制/数据/状态边界：[Batch 6](batch-6-context-management.md)。

## 9. Model Abstraction

**Confirmed**：`Model<Api>` 是包含模型 ID、provider ID、协议 API、baseUrl、能力、限额、成本与 compat 的数据对象；实际执行行为属于 Provider。Agent 不实例化 OpenAI/Anthropic SDK，调用注入 `StreamFn(model, TranscriptContext, options)`，消费统一 AssistantMessageEventStream。

**Confirmed**：当前 Coding SDK 使用 `ModelRuntime`，内部实例化 `Models` 集合并装配 Provider；`ModelRegistry` 是给扩展保留的同步兼容 facade。`pi-ai/compat` 仍有全局按 api 分发的旧注册表，不能把旧文档中的注册表当作 Coding Agent 唯一运行路径。

专题证据与完整控制/数据/状态边界：[Batch 7](batch-7-model-abstraction.md)。

## 10. Provider

**Confirmed**：Provider ID 与 API ID 是两个维度。OpenRouter factory 可同时提供 anthropic-messages 与 openai-completions；多个供应商共用 Completions adapter；OpenAI 默认 factory 使用 Responses。Provider 负责鉴权/目录与执行接口，API adapter 负责 SDK、payload、协议流、消息转换和错误归一。

**Confirmed**：适配超出改 baseUrl 的 API wrapper。已读实现包含跨模型签名清理、tool ID 修复、严格/grammar schema、图像结果编码、系统/工具中途更新、reasoning 参数、cache routing、usage/停止归一和 endpoint-specific 分支；Coding Prompt builder 没有三套默认供应商模板。

专题证据与完整控制/数据/状态边界：[Batch 8](batch-8-provider-system.md)。

## 11. Cross-provider Compatibility

**Confirmed**：统一 Message/Tool/Usage 只是入口契约，实际兼容来自 transcript + transform + 各 API adapter。

| 维度 | 代表差异 |
|---|---|
| 系统指令 | developer/system、独立 systemInstruction、系统段落折叠与更新 |
| 工具 | function/tool_use/functionCall；ID 清洗、结果角色/名称、bridge assistant |
| schema | strict require/prefer、支持子集与降级、grammar 支持 |
| thinking | budget/effort/level、显式 visible 与 opaque signature、历史重放 |
| 图片 | base64 block / data URL / inlineData，模型上限与过滤 |
| streaming | SSE/SDK 原生事件 → start/delta/end/done/error |
| usage/stop | cache/reasoning 桶与价格、finish_reason 缺失策略 |
| 传输/鉴权 | API key/OAuth、headers、session affinity、initial retry |
| cache | 断点、TTL、prompt key、可选 warm 请求 |

专题证据与完整控制/数据/状态边界：[Batch 33](batch-33-compatibility.md)。

## 12. Reasoning / Thinking

**Confirmed**：统一配置为 off/minimal/low/medium/high/xhigh/max；reasoning boolean、thinkingLevelMap 与 clamp 决定可用级别，adapter 再映射 effort、thinking budget 或 native thinkingLevel。统一级别不意味着各模型计算量、费用或能力相同。

**Confirmed**：可见推理文字与 opaque/replay signature 分开存储。thinking block 可以无可见正文；签名也可能附在 text/toolCall。只分析流/历史行为，不能由签名推断完整模型内部思维。

专题证据与完整控制/数据/状态边界：[Batch 9](batch-9-reasoning-thinking.md)。

## 13. Tool Runtime

**Confirmed**：工具定义、模型可见声明、可调用集合和一次执行不是同一个对象。`AgentSession` 维护定义注册表，工具的 exposure 决定是否直接声明；`wrapToolDefinition()` 生成 AgentTool，底层循环按名称定位、修复参数、验证 schema、执行 hooks、运行工具、回灌结果。MCP 和扩展工具经过同一执行管线。

工具执行是 **Hybrid**：模型选择工具、参数和调用顺序；Harness 提供可用声明、参数验证、调度、取消、错误转换和结果关联。权限策略可由 `beforeToolCall` / 扩展 `tool_call` 阻断，不是每个文件操作内置一个批准弹窗。

专题证据与完整控制/数据/状态边界：[Batch 10](batch-10-tool-runtime.md)。

## 14. Core Tools

**Confirmed**：内置文件/搜索/Shell 工厂共八种：read、write、edit、bash、powershell、grep、find、ls。实际默认声明由 SDK、Settings 和 Session 决定，不能把工厂导出集合当默认工具集合。`createCodingTools()` 的四工具集合为 read/bash/edit/write；产品还支持扩展提供的 Codemode、tool_search、MCP 和资源工具。

| 工具 | 参数 / 执行 | 返回及限制 | 能力归属 |
|---|---|---|---|
| read | path，offset，limit；读文本或识别图片 | 文本头部 2,000 行 / 50 KiB，图片块，后续读取提示 | Harness |
| write | path，content；创建父目录并 UTF-8 覆盖 | 成功消息；文件串行队列 | Harness |
| edit | path，edits 的 oldText/newText | 精确优先、模糊回退；唯一匹配；diff 在 details | Hybrid |
| bash | command，timeout 可选 | stdout/stderr 合并，尾部 2,000 行 / 50 KiB；大输出临时文件 | Harness |
| powershell | 与 Shell 共用执行机制 | Windows 专用配置，UTF-8 前缀 | Harness |
| grep | pattern，path，glob，ignoreCase，literal，context，limit | 默认 100 匹配；每行 500 字符；50 KiB | Hybrid |
| find | pattern，path，limit | fd glob，默认 1,000 结果 / 50 KiB | Hybrid |
| ls | path，limit | 排序目录列表，默认 500 项 / 50 KiB | Harness |

专题证据与完整控制/数据/状态边界：[Batch 11](batch-11-core-tools.md)。

## 15. File Editing

**Confirmed**：read 是文本/图片读取；edit 是文本块替换，既不是 AST 重写，也不是应用模型提供的 unified diff。write 是完整覆盖。Harness 检查唯一匹配、重叠和无变化，模型仍负责选择文件及修改内容。

例：同一个 oldText 在文件内出现两次时 edit 报错，要求更具体上下文；多个 edits 都针对同一原始内容定位，再按位置逆序替换，前一个替换不会成为后一个匹配的输入。

专题证据与完整控制/数据/状态边界：[Batch 12](batch-12-file-operations.md)。

## 16. Shell

**Confirmed**：Shell 工具是一次命令执行及结果收集，没有返回可续接的后台任务 handle。timeout 参数可选，没有工具默认超时。服务/watch 命令会持续占用调用，除非自身退出、外部取消、指定超时或由 Shell 命令显式放到后台。

Harness 管 cwd、环境、输出上限、进程树终止和事件；模型决定执行什么以及如何解释错误。不存在“所有 bash 命令成功即任务完成”的判定。

专题证据与完整控制/数据/状态边界：[Batch 13](batch-13-shell.md)。

## 17. Repository Understanding

**Confirmed**：默认仓库理解依赖提示、工作目录说明与可执行工具，不是预先构建代码语义索引。模型选择搜索词、文件和局部读取范围，Harness 执行并限制输出；两者构成 Hybrid。

| 类别 | 当前内置证据 / 判断 |
|---|---|
| A 文件枚举 | ls/find；Shell 可执行 tree、git ls-files，调用由模型决定 |
| B 文本搜索 | grep 使用 rg；Shell 亦可运行 rg |
| C 文件阅读 | read 支持 offset/limit，图片和截断 |
| D Git 信息 | bash/powershell 可以执行 git；不是独立 Git 语义工具 |
| E 符号 / AST | 本轮检查的核心工具表和包依赖未见内置 LSP、AST 查询或 tree-sitter 工具：Not Present（限定默认内置管线） |
| F Embedding / RAG | 未见仓库 embedding 检索管线；tool_search 的 BM25 是搜索工具描述，不是源码语义索引 |
| G Repo Map | 无内置静态语义 repo-map 生成阶段；模型可通过工具自己建立理解 |

专题证据与完整控制/数据/状态边界：[Batch 14](batch-14-repository-understanding.md)。

## 18. Coding Workflow

**Confirmed**：核心没有固定 search → read → edit → test 的任务状态机。`runLoop()` 仅依据工具调用、输入队列、取消与结束结果推进。模型可以生成这条工作流，但也可以直接回答、跳过测试或重复搜索。

具体路径：用户给 bug → 模型 grep/read → edit → bash 测试失败 → 错误 text 作为 toolResult → 模型修复并再测 → 无工具调用且队列空 → assistant 完成。搜索词、修改方案、测试选择与最终是否足够是模型判断；参数检查、真实执行、错误回灌、日志/取消为 Harness。

专题证据与完整控制/数据/状态边界：[Batch 15](batch-15-coding-workflow.md)。

## 19. Session

**Confirmed**：SessionManager 保存的是带 id/parentId 的追加日志树；active leaf 指向当前分支。AgentSession 提供编码行为，AgentSessionRuntime 负责替换会话并重绑定 Host。恢复读取文件并重建投影，不恢复之前正在执行的 OS 进程或模型网络流。

branch 在同一日志树内改变 leaf；fork 生成新 session id / 文件，携带选定祖先路径和 parentSession 关系。这与复制整个树不同。

专题证据与完整控制/数据/状态边界：[Batch 16](batch-16-session.md)。

## 20. State

**Confirmed**：状态按作用域分散，没有单一全局 AppState。

| Owner | 状态 / 寿命 |
|---|---|
| Host / TUI | Editor、折叠、布局、当前显示组件 / Host 寿命 |
| AgentSessionRuntime | 当前 session/services、替换绑定 / 应用运行 |
| AgentSession | 定义注册表、queue、retry/compaction、扩展 runner / 会话 |
| Agent | messages、model、tools、streamingMessage、pending calls / Agent 实例 |
| SessionManager | 日志树、leaf、投影 / 可持久化会话 |
| ModelRuntime / Models | provider/catalog/auth facade、刷新 generation / 服务 |
| ExtensionRuntime | 注册、flags、上下文有效性 / 扩展装载代次 |

专题证据与完整控制/数据/状态边界：[Batch 17](batch-17-state.md)。

## 21. Persistence

**Confirmed**：默认编码会话使用 JSONL；配置、鉴权、模型缓存使用 JSON。不能把 monorepo 中 durable 包等同默认 AgentSession 用 SQLite 保存所有状态。

| 载体 | 内容 / owner |
|---|---|
| session JSONL | header、树 entries / SessionManager |
| settings.json | 全局与可信项目配置 / SettingsManager |
| auth.json | provider credential / AuthStorage |
| models.json | 人工模型/Provider 定义 / ModelConfig |
| models-store.json | 刷新目录缓存 / FileModelsStore |
| keybindings.json | 用户快捷键 / KeybindingsManager |
| mcp.json / mcp-auth.json | 服务器定义与 OAuth / MCP 扩展 |
| 临时输出文件 | Shell/MCP/Codemode 全输出；不是会话数据库 |

专题证据与完整控制/数据/状态边界：[Batch 18](batch-18-persistence.md)。

## 22. Memory

**Confirmed**：短期记忆是模型实际请求的消息投影；持久 session、compaction summary、branch summary、AGENTS/Skill 指令是不同信息来源。核心工具表和 resource 加载路径未见自动跨会话记忆抽取、向量记忆库或用户事实召回服务：Not Present（默认内置功能）。

但不能说“没有可保存状态”：Codemode 的 store/load 通过 codemode-store custom entries 按当前 branch 重放；virtual model router 也保存 branch-local custom state。这是程序状态，不是自动语义长期记忆。

专题证据与完整控制/数据/状态边界：[Batch 19](batch-19-memory.md)。

## 23. Extension

**Confirmed**：扩展是可信的 TS/JS factory，在宿主进程运行，并非天然隔离插件。Loader 发现/导入/初始化，Runner 绑定核心并派发 hook，ResourceLoader 结合配置、包资源与信任规则。注册可以发生在加载阶段，执行类 action 在 bind 前为抛错 stub。

Factory 失败会 discard 待提交运行时注册和加载期 event-bus 订阅；成功才 commit。reload/会话替换使旧 runtime/ctx stale，并清理追踪订阅。该清理不能撤销扩展顶层任意 OS 副作用。

专题证据与完整控制/数据/状态边界：[Batch 20](batch-20-extensions.md)。

## 24. Skill

**Confirmed**：Skill 是带 frontmatter 的指令文件及相关资源，不是代码注册 factory。ResourceLoader/skills.ts 发现、解析、检查名称/描述与冲突；系统提示先提供技能摘要/路径，正文按需进入上下文。显式 /skill:name 由 Session 展开成用户消息中的 skill block。

技能指导工具调用，但没有独立 Skill.execute() 的强制步骤执行器；是否遵守正文是模型能力，发现和展开是 Harness，整体 Hybrid。

专题证据与完整控制/数据/状态边界：[Batch 21](batch-21-skills.md)。

## 25. MCP

**Confirmed**：此仓库 MCP 是实际实现：编码层内置扩展负责注册/发现/曝光/管理，pi-mcp 包负责 JSON-RPC client、stdio 与 Streamable HTTP。不支持 legacy SSE 配置。默认 exposure=codemode，另有 deferred/direct/hidden 和单工具覆盖；direct 首次 prompt 等连接最多默认 10s，间接工具在调用时等相关 servers。

服务器文件配置优先于扩展同名注册；可信项目 mcp.json 替换全局同名项。项目配置禁止 auth.provider；OAuth 也不等同模型 Provider OAuth。


版本边界：以上配置结论对应固定研究基线。当前 HEAD 新增可信项目 enabled/exposure/toolExposure 的局部覆盖分支，见 [基线推进说明](baseline-drift.md)。

专题证据与完整控制/数据/状态边界：[Batch 22](batch-22-mcp.md)。

## 26. Events

**Confirmed**：Provider 原生事件归一为 AssistantMessageEventStream；Agent 消费后发 AgentEvent；AgentSession 增加压缩、重试和 settled 等产品事件；Host、ExtensionRunner 等订阅。Extension EventBus 是额外的任意频道通信，不是这些核心 typed events 的持久可靠队列。

专题证据与完整控制/数据/状态边界：[Batch 23](batch-23-events.md)。

## 27. Concurrency

**Confirmed**：并发由多种机制实现，不能说只有单线程、也不能说每个工具都 worker 化。默认工具批次 Promise 并行；显式 toolExecution=sequential 或任一 sequential 工具使整批串行；文件队列再按 canonical path 排序。模型流、输入队列、MCP background connect 与 cache warming 可同时存在；Shell/stdio 是 OS 子进程，Codemode 是 QuickJS sandbox worker。

专题证据与完整控制/数据/状态边界：[Batch 24](batch-24-concurrency.md)。

## 28. CLI / TUI

**Confirmed**：main 解析命令行、stdin/附件、资源/服务和会话，再绑定 interactive/print/RPC 等模式。TUI 管编辑器、快捷键、组件/布局与显示；Session/Agent 核心可 headless 使用。不是所有产品策略都从 TUI 完全抽离：内置 slash command 的 selector/确认/会话导航 Host 行为仍在 InteractiveMode。

专题证据与完整控制/数据/状态边界：[Batch 25](batch-25-cli-tui.md)。

## 29. Configuration

**Confirmed**：没有全局统一的 CLI > env > project > user 优先级公式，必须按字段看调用点。

| 配置 | 实际来源与覆盖 |
|---|---|
| settings | 默认 getter + 全局 + 项目可允许字段；嵌套对象合并，特定数组/工具选择有专门逻辑 |
| 模型选择 | SDK/CLI 显式模型、可恢复 branch selection、默认 provider/model 与可用目录共同决定 |
| thinking | 显式值 → 恢复条目（继续会话）→ per-model → 全局 → 默认；最后 clamp |
| tools | options.tools / noTools / settings.defaultTools / DEFAULT_TOOL_NAMES，另做 exclude |
| 请求 retry/timeouts | 显式 request options 优先于 settings provider defaults |
| MCP | 可信项目同名覆盖全局，文件同名覆盖注册项；项目 auth.provider 禁止 |
| 环境 | provider 鉴权/路径/代理有各自解析，不是覆盖所有 settings |


版本边界：以上配置结论对应固定研究基线。当前 HEAD 新增可信项目 enabled/exposure/toolExposure 的局部覆盖分支，见 [基线推进说明](baseline-drift.md)。

专题证据与完整控制/数据/状态边界：[Batch 27](batch-27-configuration.md)。

## 30. Error / Retry

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

**Confirmed**：至少四个恢复域：Provider 初始请求重试、Session assistant 续请求、摘要 retryAssistantCall、MCP connect/read-only/session-expiry 恢复。它们有独立预算与边界，工具执行失败不会由通用 Agent Loop 无条件重跑。

附专题：[batch-28-errors.md](batch-28-errors.md)。

专题证据与完整控制/数据/状态边界：[Batch 29](batch-29-retry-recovery.md)。

## 31. Token / Cost

**Confirmed**：Provider 回报用量与本地预估分开。Usage 的 input/output/cacheRead/cacheWrite 是归一桶，reasoning 是 output 的可选细分，cacheWrite1h 是 cacheWrite 的细分，不应再叠加总量。Cost 来自 Model 的每百万 token 价格/tiers，并非对账 API 的实际账单。

专题证据与完整控制/数据/状态边界：[Batch 30](batch-30-tokens-cost.md)。

## 32. Cache

**Confirmed**：本地缓存包括 catalog models-store.json、credential/model 文件 revision 快照、extension factory cache；供应商 prompt cache 通过 cacheRetention/断点/session affinity 等协议字段请求。默认 coding 设置 cacheWarming=streaming，是真实额外 LLM 请求，不是本地缓存复制。

专题证据与完整控制/数据/状态边界：[Batch 31](batch-31-cache.md)。

## 33. Testing / Eval

**Confirmed**：存在离线单元/集成、faux provider 的 Session suite、真实 API E2E、MCP conformance 与模型 eval。测试名称说明覆盖意图，不证明当前版本测试通过。此分析未运行项目测试。

| 层 | 示例/机制 |
|---|---|
| 算法/工具 | edit/compaction/truncation/cache tests |
| Agent / Session | suite/harness.ts 注入 faux steps、收集 events |
| Provider mapping | api-specific fixtures/mock SDK 与 usage/retry tests |
| MCP | in-memory/stdio/HTTP fixtures、OAuth 与 conformance |
| TUI | node:test、宽度与组件行为 |
| 真实模型 | ai *-e2e 与 evals；需要显式运行边界 |

**Confirmed**：packages/evals 实际存在，包含 vitest-evals、docs variants、Docker 执行、报告与对比。smoke 检查 Paris 和 usage；extensions docs eval 要求创建扩展、reload、调用 hello 并校验结构化输出/工具参数；documentation audit 逐页调查后提交结构化 verdict。不能将此归为“Not Present”。

本轮发现的评测面向产品文档使用/配置/扩展及基本模型行为；未由这些源码确认 SWE-bench 或通用大规模编码修复排行榜。

附专题：[batch-34-testing.md](batch-34-testing.md)。

专题证据与完整控制/数据/状态边界：[Batch 35](batch-35-evaluation.md)。

## 34. Core Abstractions

**Confirmed**：以下职责、创建入口和数据来自实现；抽象选择顺序属于 Inference。

| 抽象 | 责任 | 创建者 / 寿命 | 依赖 | 数据 / owner | 关键调用 | 证据 |
| --- | --- | --- | --- | --- | --- | --- |
| Agent | 单次运行状态/输入队列 | SDK 创建；Agent 实例 | Loop/streamFn | messages/model/tools | prompt/continue/abort | [agent.ts](../packages/agent/src/agent.ts) |
| AgentLoopConfig | 一次 loop 的策略与回调 | Agent 每次 createLoopConfig | hooks / streamFn | model/options/queue readers | runLoop | [types.ts](../packages/agent/src/types.ts) |
| AgentContext | 运行请求快照 | Agent createContextSnapshot | messages/tools | array copies | prepareRequest | [types.ts](../packages/agent/src/types.ts) |
| AgentSession | 编码行为与恢复 | SDK 创建；会话 | Agent/Manager/Resources | queues/registry/controllers | prompt/compact/setModel | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) |
| AgentSessionRuntime | 当前会话替换与绑定 | main/Host；应用 | services/session | 当前 session 引用 | 替换与 shutdown | [agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts) |
| AgentSessionServices | 服务装配 | createAgentSessionServices | Settings/Models/Resources | 共享依赖 | createAgentSessionFromServices | [agent-session-services.ts](../packages/coding-agent/src/core/agent-session-services.ts) |
| SessionManager | 日志树与投影 | SDK/Runtime；持久会话 | JSONL/transcript | entries/leaf/byId | append*/branch/buildSessionProjection | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) |
| SessionProjection | 模型可见重放结果 | Manager 构造；请求 | 树/compaction/context edits | entries/messages/model/thinking | buildContextEntries | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) |
| SystemMessage | 可重放提示/工具声明 | Session/Loop；日志 | transcript | sections/tools/deltas | normalizeContext | [types.ts](../packages/ai/src/types.ts) |
| AssistantMessage | 统一响应结果 | Provider；响应/日志 | API stream | content/usage/stop | streamAssistantResponse | [types.ts](../packages/ai/src/types.ts) |
| ToolDefinition | 产品工具定义 | 内置工厂/Extension；注册 | schema/exposure/renderers | execute/params/metadata | wrapToolDefinition | [types.ts](../packages/coding-agent/src/core/extensions/types.ts) |
| AgentTool | 可执行核心工具 | wrapper；Agent loadout | schema/execute | arguments/result | runToolCall | [types.ts](../packages/agent/src/types.ts) |
| Model | 模型数据契约 | catalog/config；目录快照 | Provider/API | limits/cost/compat | clampThinkingLevel | [types.ts](../packages/ai/src/types.ts) |
| Provider | 鉴权/目录/请求行为 | factory；Models 注册 | API adapters | auth/models/stream | streamSimple | [types.ts](../packages/ai/src/types.ts) |
| Models | 实例 Provider 服务 | createModels；服务寿命 | CredentialStore/ModelsStore | maps/generations | refresh/getAuth/streamSimple | [models.ts](../packages/ai/src/models.ts) |
| ModelRuntime | 编码层模型组合/路由 | services；服务寿命 | Models/ModelConfig | catalog/virtual routes | prepareRequest/resolveModel | [model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) |
| ResourceLoader | 资源发现/信任/加载 | services；资源代次 | packages/extensions/skills | loaded resources/errors | reload | [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) |
| ExtensionRunner | hook/command 执行环境 | Session；绑定代次 | ExtensionRuntime/core/UI | handlers/context | bindCore/emit* | [runner.ts](../packages/coding-agent/src/core/extensions/runner.ts) |
| SettingsManager | 设置合并/保存/getters | services；配置实例 | SettingsStorage | global/project/effective | getCompactionSettings/set* | [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) |
| McpServerConnection | 单服务器连接与恢复 | MCP extension；session | McpClient/transport | state/tools/opening | getClient/callTool/close | [runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts) |

专题证据与完整控制/数据/状态边界：[Batch 36](batch-36-core-abstractions.md)。

## 35. Design Patterns

| 模式 | 具体问题 / trace | 实现 / 必要性 |
| --- | --- | --- |
| Adapter | 不同供应商返回角色/工具/流形状不同 → canonical messages/events | [openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[tools.ts](../packages/coding-agent/src/extensions/mcp/tools.ts)；必要协议边界 |
| Strategy / callbacks | 模型 streamFn、tools execute、context hooks 可替换 | [agent.ts](../packages/agent/src/agent.ts)；Loop 不依赖单一 SDK |
| Factory | 工具和 Provider 需要 cwd/options/auth 配置后形成对象 | [index.ts](../packages/coding-agent/src/core/tools/index.ts)、[all.ts](../packages/ai/src/providers/all.ts) |
| Registry | 动态注册 tool/provider/command；按名称查调用 | [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) |
| Observer | 一次流要驱动状态、持久化和多个 Host | [agent.ts](../packages/agent/src/agent.ts)、[event-bus.ts](../packages/coding-agent/src/core/event-bus.ts)；typed 核心流与任意频道 bus 不同 |
| Command | 用户 /xxx 与模型工具均封装行为但入口不同 | [slash-commands.ts](../packages/coding-agent/src/core/slash-commands.ts)、[types.ts](../packages/coding-agent/src/core/extensions/types.ts) |
| Dependency injection | SessionServices/SDK 注入 stores、tools、streamFn；测试替换为 faux | [sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[harness.ts](../packages/coding-agent/test/suite/harness.ts) |
| Append-log / projection | 分支保留历史，请求只看 active leaf/压缩尾部/edits | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)；不是所有状态事件溯源 |
| Generation / stale guard | 旧 refresh 或 extension ctx 完成晚于替换 → 拒绝旧发布/动作 | [models.ts](../packages/ai/src/models.ts)、[loader.ts](../packages/coding-agent/src/core/extensions/loader.ts) |
| Local serialization | 并行工具写同文件 → per-path Promise chain | [file-mutation-queue.ts](../packages/coding-agent/src/core/tools/file-mutation-queue.ts)；不提供跨进程事务 |

专题证据与完整控制/数据/状态边界：[Batch 37](batch-37-design-patterns.md)。

## 36. Technical Debt

| 问题（Inference） | 源码事实 / 具体风险 | 建议与边界 |
| --- | --- | --- |
| Session 与 Host 集中职责 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 同时管队列/提示/工具/重试/摘要/扩展；[interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts) 同时管渲染/命令/会话 UI | 按 owner 边界拆分可改善维护；没有仅因文件大就断言 bug |
| 双入口 API 维护成本 | [compat.ts](../packages/ai/src/compat.ts) 的全局 registry 与 [models.ts](../packages/ai/src/models.ts) 的实例服务并存 | 明确迁移边界；不能仅删除 compat 破坏扩展行为 |
| 重复协议转换 | [openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts)、[anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) 各实现 thinking/tools/cache | 共用重放不应抹平 native 差异；先加契约 fixtures 再抽取 |
| 错误文本分类 | [retry.ts](../packages/ai/src/utils/retry.ts) 通过正则区分 transient 与 quota | 结构化原因可减误判；当前兼容网关文字仍有实际价值 |
| 文件外部写竞争 | [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) 读后直接写；队列只约束宿主进程 | 若需更强正确性，加入版本校验或 atomic write；不是本任务实施项 |
| 取消后的副作用 | [write.ts](../packages/coding-agent/src/core/tools/write.ts)、[execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts) 明确未回滚外部工具 | 界面/策略应区分 canceled 与 undone；没有通用事务承诺 |
| 配置优先级分散 | [sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)、[config.ts](../packages/coding-agent/src/extensions/mcp/config.ts) 不同读取路径 | 逐键说明/契约测试比抽象单一总优先级更可靠 |
| 缓存经济估计 | [cache-warmer.ts](../packages/coding-agent/src/core/cache-warmer.ts) 使用固定续请求概率和目录价格 | warm 有额外费用，需实测收益；本轮没有收益数据 |
| 验证缺口仍待测量 | tests/evals 存在但本轮未运行且未统计 coverage | 缺覆盖的具体断言需测试/coverage证据；不宣称仓库无测试 |

专题证据与完整控制/数据/状态边界：[Batch 38](batch-38-technical-debt.md)。

## 37. Extension Points

| 任务 | 入口 | 符号 | 联动点 |
| --- | --- | --- | --- |
| 修改 Agent Loop | [agent-loop.ts](../packages/agent/src/agent-loop.ts) | runLoop / executeToolCalls | 续轮、终止、队列和工具批次 |
| 修改 System Prompt | [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts) | buildSystemPromptSections / buildSystemPrompt | 段落构建；另检查 Session delta |
| 修改 Context | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | buildSessionProjection / buildContextEntries | 当前分支/edits/压缩重放 |
| 修改 Compression | [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) | prepareCompaction / compact | 切点与摘要；Session 负责触发/提交 |
| 增加 Provider | [all.ts](../packages/ai/src/providers/all.ts) | builtinProviders / builtinModels | 新增 factory；API 新协议另增 adapter |
| 增加模型 | [generate-models.ts](../packages/ai/scripts/generate-models.ts) | 生成模型脚本 | 静态目录更新生成源，不直接改 generated；动态目录检查 Provider refresh |
| 修改 Message Conversion | [transform-messages.ts](../packages/ai/src/api/transform-messages.ts) | transformMessages | 共享重放；wire 编码看对应 api 文件 |
| 修改 Tool Call | [agent-loop.ts](../packages/agent/src/agent-loop.ts) | prepareToolCall / runToolCall | 参数、hooks、错误和结果 |
| 增加 Tool | [types.ts](../packages/coding-agent/src/core/extensions/types.ts) | ToolDefinition / registerTool | 优先扩展；内置加工厂/index |
| 修改 Read Tool | [read.ts](../packages/coding-agent/src/core/tools/read.ts) | createReadToolDefinition | 分页、图片、输出限制 |
| 修改 Edit Tool | [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) | createEditToolDefinition | 配合 edit-diff 与 mutation queue |
| 修改 Shell Tool | [bash.ts](../packages/coding-agent/src/core/tools/bash.ts) | createShellToolDefinition / createLocalShellOperations | spawn、输出与 timeout/abort |
| 修改 Session | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | append*/branch/forkFrom | 日志结构与版本迁移；行为入口在 AgentSession |
| 增加 Extension | [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts) | initializeExtension / ExtensionFactory | 用户 extension factory + Runner API；本机可信执行 |
| 修改 MCP | [runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts) | McpServerConnection / createDefaultTransport | 曝光/管理在 index，协议在 packages/mcp |
| 修改 Config | [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) | Settings / getters | 逐键 trust/default/merge/save 路径 |
| 修改 Events | [types.ts](../packages/agent/src/types.ts) | AgentEvent | 同时更新 producer/reducer/Session/Host |
| 修改 Token Usage | [models.ts](../packages/ai/src/models.ts) | calculateCost | 计数来自 API adapter；统计看 usage-totals |
| 修改 Streaming | [event-stream.ts](../packages/ai/src/utils/event-stream.ts) | EventStream / AssistantMessageEventStream | 原生事件解析看具体 API adapter |
| 修改 Reasoning | [simple-options.ts](../packages/ai/src/api/simple-options.ts) | adjustMaxTokensForThinking | thinking map在Models；budget/effort/signature看具体API |

新增 Provider：若现有API可表达，写 Provider factory 或 registerProvider 配置；新协议需 stream adapter、事件归一和测试。新增工具：ToolDefinition + registerTool；替换内置行为可用 SDK override。换摘要：session_before_compact hook 或 compaction策略。MCP：registerMcpServer 或服务器配置；传输协议改 packages/mcp。

专题证据与完整控制/数据/状态边界：[Batch 39](batch-39-extension-points.md)。

## 38. Minimal Core

**Inference（概念最小集，不是已测量的 20% 代码）**：

| 层 | 必须行为 | Class / Function | 可复用证据 |
| --- | --- | --- | --- |
| 消息契约 | user/assistant/toolResult、id关联、usage/stop/error | Message / AssistantMessage / ToolResultMessage | [types.ts](../packages/ai/src/types.ts) |
| 流适配 | 一个Provider实现，文本/工具delta/终结 | AssistantMessageEventStream / stream() | [event-stream.ts](../packages/ai/src/utils/event-stream.ts)、[openai-completions.ts](../packages/ai/src/api/openai-completions.ts) |
| Loop | 请求→工具执行→结果回灌→继续/结束/取消 | runLoop / executeToolCalls / createToolResultMessage | [agent-loop.ts](../packages/agent/src/agent-loop.ts) |
| 运行所有权 | messages/tools/model、单run、queue/abort/finally | Agent.prompt / continue / abort / processEvents | [agent.ts](../packages/agent/src/agent.ts) |
| Context builder | 系统/当前对话/工具声明，输出与window边界 | normalizeContext / getCurrentSystemMessage | [transcript.ts](../packages/ai/src/utils/transcript.ts) |
| 工具层 | schema验证、read/edit/write/Shell、错误关联 | ToolDefinition / wrapToolDefinition / create*ToolDefinition | [index.ts](../packages/coding-agent/src/core/tools/index.ts) |
| Host | 读取输入、呈现流与错误、取消 | runPrintMode | [print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts) |

若要求可恢复长任务，再加入 JSONL/branch projection、压缩和有界重试。TUI、MCP、Skill、复杂扩展、虚拟路由、warm/eval 是目标驱动的可选层；删除它们不是在现有Pi中授权移除功能。

专题证据与完整控制/数据/状态边界：[Batch 41](batch-41-minimal-core.md)。

## 39. 30 Core Files

| 顺序 | 源码 | 类型 / 函数 | 为何重要 | 关联模块 |
| --- | --- | --- | --- | --- |
| 1 | [agent-loop.ts](../packages/agent/src/agent-loop.ts) | runLoop / streamAssistantResponse / executeToolCalls | 真正的模型-工具续轮 | Agent/Provider/tools |
| 2 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) | prompt / _runAgentPrompt / compact / _checkCompaction | Coding Harness 中枢 | Manager/Models/Extensions |
| 3 | [agent.ts](../packages/agent/src/agent.ts) | prompt / continue / processEvents / finishRun | 运行所有权和队列 | Loop/Host |
| 4 | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | buildSessionProjection / appendCompaction / branch / forkFrom | 日志树与可见上下文 | Session/Compaction |
| 5 | [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) | prepareCompaction / compact / estimateProjectedContextTokens | 长会话切点/摘要/估算 | Session/Provider |
| 6 | [types.ts](../packages/ai/src/types.ts) | Message / Model / Provider / Usage | 跨包数据契约 | 全部 adapter |
| 7 | [transcript.ts](../packages/ai/src/utils/transcript.ts) | normalizeContext / getCurrentSystemMessage | 系统/工具重放 | Loop/Adapters |
| 8 | [transform-messages.ts](../packages/ai/src/api/transform-messages.ts) | transformMessages | 跨模型历史修复 | Adapters |
| 9 | [model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) | prepareRequest / resolveModel | 配置、鉴权、路由装配 | Models/Registry |
| 10 | [models.ts](../packages/ai/src/models.ts) | createModels / calculateCost / clampThinkingLevel | 实例模型服务及能力 | Provider/stores |
| 11 | [openai-completions.ts](../packages/ai/src/api/openai-completions.ts) | stream / streamSimple / convertMessages / buildParams | 兼容协议变体厚度 | Models/transcript |
| 12 | [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) | stream / streamSimple | thinking/tools/cache原生映射 | transcript |
| 13 | [openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts) | 消息/工具转换与事件解析 | Responses共享协议行为 | Responses/Codex adapters |
| 14 | [google-shared.ts](../packages/ai/src/api/google-shared.ts) | convertMessages / convertTools | Google消息、签名、工具映射 | GenAI/Vertex |
| 15 | [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts) | buildSystemPrompt / buildSystemPromptSections | 提示组成 | ResourceLoader/Session |
| 16 | [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) | DefaultResourceLoader.reload | 资源与信任装载 | Extensions/Skills/Packages |
| 17 | [runner.ts](../packages/coding-agent/src/core/extensions/runner.ts) | bindCore / emit / emitContext / emitBoundary | 动态策略和边界 | Session/Host |
| 18 | [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts) | initializeExtension / createExtensionRuntime | 装载与有效期 | ResourceLoader/Runner |
| 19 | [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) | createEditToolDefinition | 文件变更管线 | edit-diff/queue |
| 20 | [edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts) | applyEditsToNormalizedContent / fuzzyFindText | 唯一匹配与差异生成 | Edit/Preview |
| 21 | [bash.ts](../packages/coding-agent/src/core/tools/bash.ts) | createLocalShellOperations / createShellToolDefinition | OS执行与输出 | Shell/Accumulator |
| 22 | [read.ts](../packages/coding-agent/src/core/tools/read.ts) | createReadToolDefinition | 上下文文件入口 | images/truncate |
| 23 | [sdk.ts](../packages/coding-agent/src/core/sdk.ts) | createAgentSession | 核心装配与无UI入口 | Services/Session |
| 24 | [agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts) | AgentSessionRuntime | 切换与生命周期Host绑定 | Services/Session |
| 25 | [main.ts](../packages/coding-agent/src/main.ts) | main | 进程启动与模式选择 | CLI/Runtime/Hosts |
| 26 | [interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts) | InteractiveMode | 用户交互/命令/呈现 | Session/TUI |
| 27 | [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) | SettingsManager / getCompactionSettings | 有效配置 | SDK/Session |
| 28 | [index.ts](../packages/coding-agent/src/extensions/mcp/index.ts) | createMcpExtension | 工具发现/曝光/生命周期 | Runner/Connection |
| 29 | [client.ts](../packages/mcp/src/client.ts) | McpClient.connect / callTool / requestInternal | JSON-RPC连接/取消/请求 | Transports/MCP extension |
| 30 | [execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts) | executeCodemode / readCodemodeStore | 脚本编排与嵌套工具 | Sandbox/Session/MCP |

专题证据与完整控制/数据/状态边界：[Batch 42](batch-42-top-30-files.md)。

## 40. Reading Path

| 时间预算 | 顺序 | 阅读目标 |
| --- | --- | --- |
| 30 分钟 | Batch 1/46 → AgentLoop runLoop → Agent.processEvents → ToolResultMessage | 定位模型/工具续轮和状态owner，先不读所有Provider |
| 2 小时 | 以上 + SDK → AgentSession.prompt/_runAgentPrompt → SessionManager projection → compaction prepare/commit | 看清请求边界、结束边界和上下文 |
| 1 天 | 以上 + transcript/transform → 典型4协议 → edit/queue/bash → ExtensionRunner/Loader → MCP client | 串起真实请求和I/O，注意错误/取消 |
| 3 天 | 以上 + ResourceLoader/Settings → TUI/print/RPC → virtual models/cache → suite/evals → Batch38债务验证 | 按任务做有针对性的运行验证；真实API实验单独安排 |

专题证据与完整控制/数据/状态边界：[Batch 43](batch-43-reading-path.md)。

## 41. Development Map

| 任务 | 入口 | 符号 | 联动点 |
| --- | --- | --- | --- |
| 修改 Agent Loop | [agent-loop.ts](../packages/agent/src/agent-loop.ts) | runLoop / executeToolCalls | 续轮、终止、队列和工具批次 |
| 修改 System Prompt | [system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts) | buildSystemPromptSections / buildSystemPrompt | 段落构建；另检查 Session delta |
| 修改 Context | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | buildSessionProjection / buildContextEntries | 当前分支/edits/压缩重放 |
| 修改 Compression | [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) | prepareCompaction / compact | 切点与摘要；Session 负责触发/提交 |
| 增加 Provider | [all.ts](../packages/ai/src/providers/all.ts) | builtinProviders / builtinModels | 新增 factory；API 新协议另增 adapter |
| 增加模型 | [generate-models.ts](../packages/ai/scripts/generate-models.ts) | 生成模型脚本 | 静态目录更新生成源，不直接改 generated；动态目录检查 Provider refresh |
| 修改 Message Conversion | [transform-messages.ts](../packages/ai/src/api/transform-messages.ts) | transformMessages | 共享重放；wire 编码看对应 api 文件 |
| 修改 Tool Call | [agent-loop.ts](../packages/agent/src/agent-loop.ts) | prepareToolCall / runToolCall | 参数、hooks、错误和结果 |
| 增加 Tool | [types.ts](../packages/coding-agent/src/core/extensions/types.ts) | ToolDefinition / registerTool | 优先扩展；内置加工厂/index |
| 修改 Read Tool | [read.ts](../packages/coding-agent/src/core/tools/read.ts) | createReadToolDefinition | 分页、图片、输出限制 |
| 修改 Edit Tool | [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) | createEditToolDefinition | 配合 edit-diff 与 mutation queue |
| 修改 Shell Tool | [bash.ts](../packages/coding-agent/src/core/tools/bash.ts) | createShellToolDefinition / createLocalShellOperations | spawn、输出与 timeout/abort |
| 修改 Session | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | append*/branch/forkFrom | 日志结构与版本迁移；行为入口在 AgentSession |
| 增加 Extension | [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts) | initializeExtension / ExtensionFactory | 用户 extension factory + Runner API；本机可信执行 |
| 修改 MCP | [runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts) | McpServerConnection / createDefaultTransport | 曝光/管理在 index，协议在 packages/mcp |
| 修改 Config | [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) | Settings / getters | 逐键 trust/default/merge/save 路径 |
| 修改 Events | [types.ts](../packages/agent/src/types.ts) | AgentEvent | 同时更新 producer/reducer/Session/Host |
| 修改 Token Usage | [models.ts](../packages/ai/src/models.ts) | calculateCost | 计数来自 API adapter；统计看 usage-totals |
| 修改 Streaming | [event-stream.ts](../packages/ai/src/utils/event-stream.ts) | EventStream / AssistantMessageEventStream | 原生事件解析看具体 API adapter |
| 修改 Reasoning | [simple-options.ts](../packages/ai/src/api/simple-options.ts) | adjustMaxTokensForThinking | thinking map在Models；budget/effort/signature看具体API |

专题证据与完整控制/数据/状态边界：[Batch 45](batch-45-development-map.md)。

## 42. Critical Call Chains

#### 1. Startup

`main() → createAgentSessionServices() → createAgentSessionFromServices()/createAgentSession() → AgentSession → Host bind`

证据：[main.ts](../packages/coding-agent/src/main.ts)、[agent-session-services.ts](../packages/coding-agent/src/core/agent-session-services.ts)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)。

#### 2. User input

`Host submit/print prompt → AgentSession.prompt() → _runAgentPrompt() → Agent.prompt() → runPromptMessages() → runAgentLoop()`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[agent.ts](../packages/agent/src/agent.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts)。

#### 3. Context

`Agent.prepareRequest hook（Session安装） → SessionManager.buildSessionProjection() → buildContextEntries()/projectContextEntry() → transformContext → convertToLlm`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)。

#### 4. Prompt

`AgentSession._preparePromptAndToolLoadout() → buildSystemPromptSections() → system patch / tool delta → transcript normalizeContext()`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[system-prompt.ts](../packages/coding-agent/src/core/system-prompt.ts)、[transcript.ts](../packages/ai/src/utils/transcript.ts)。

#### 5. Model request

`streamAssistantResponse() → 注入SDK streamFn → ModelRuntime.streamSimple()/prepareRequest() → Provider.streamSimple() → API.streamSimple()/stream()`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[models.ts](../packages/ai/src/models.ts)、[openai-completions.ts](../packages/ai/src/api/openai-completions.ts)。

#### 6. Streaming

`API stream() chunk parser → AssistantMessageEventStream → streamAssistantResponse() → Agent.processEvents() → AgentSession._handleAgentEvent()`

证据：[openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[event-stream.ts](../packages/ai/src/utils/event-stream.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent.ts](../packages/agent/src/agent.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)。

#### 7. Tool call

`runLoop() → executeToolCalls() → prepareToolCallArguments()/prepareToolCall() → executePreparedToolCall() → finalizeExecutedToolCall()`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[tool-definition-wrapper.ts](../packages/coding-agent/src/core/tools/tool-definition-wrapper.ts)。

#### 8. Tool result

`finalizeExecutedToolCall() → createToolResultMessage() → emitToolResultMessage() → processEvents() → _handleAgentEvent() → appendMessage()`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent.ts](../packages/agent/src/agent.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)。

#### 9. File read

`executePreparedToolCall() → createReadToolDefinition().execute → resolveReadPathAsync() → operations.access/readFile → truncateHead()`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[read.ts](../packages/coding-agent/src/core/tools/read.ts)、[path-utils.ts](../packages/coding-agent/src/core/tools/path-utils.ts)、[truncate.ts](../packages/coding-agent/src/core/tools/truncate.ts)。

#### 10. File edit

`prepareEditArguments() → createEditToolDefinition().execute → withFileMutationQueue() → operations.readFile → applyEditsToNormalizedContent() → operations.writeFile`

证据：[edit.ts](../packages/coding-agent/src/core/tools/edit.ts)、[file-mutation-queue.ts](../packages/coding-agent/src/core/tools/file-mutation-queue.ts)、[edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts)。

#### 11. Shell

`createShellToolDefinition().execute → resolveSpawnContext() → BashOperations.exec() → createLocalShellOperations() spawn → waitForChildProcess() → accumulator结果`

证据：[bash.ts](../packages/coding-agent/src/core/tools/bash.ts)、[output-accumulator.ts](../packages/coding-agent/src/core/tools/output-accumulator.ts)、[child-process.ts](../packages/coding-agent/src/utils/child-process.ts)。

#### 12. Compression

`AgentSession.compact() / _checkCompaction() → prepareCompaction() → extension session_before_compact/default compact() → completeSummarization() → appendCompaction() → _refreshFinalizedContext()`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)。

#### 13. Extension

`ResourceLoader.reload() → loadExtensionsCached()/loadExtensionFromFactory() → initializeExtension() → createExtensionAPI() commit → Runner.bindCore() → session_start/后续hook`

证据：[resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)、[loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)、[runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)。

#### 14. MCP

`createMcpToolDefinition().execute → McpServerConnection.callTool()/withClient() → McpClient.callTool()/requestInternal() → McpTransport.send() → convertMcpResult()`

证据：[tools.ts](../packages/coding-agent/src/extensions/mcp/tools.ts)、[runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts)、[client.ts](../packages/mcp/src/client.ts)。

#### 15. Final answer

`runLoop() 无工具/队列结束 → agent_end → Agent.processEvents()/finishRun() → Session._runAgentPrompt() 恢复/边界检查 → agent_settled → Host 输出`

证据：[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent.ts](../packages/agent/src/agent.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts)。

#### 16. Model switch

`AgentSession.setModel() → checkAuth → Agent model更新/appendModelChange → setThinkingLevel → 新prepareRequest/route → transformMessages`

证据：[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[virtual-models.ts](../packages/coding-agent/src/core/virtual-models.ts)、[transform-messages.ts](../packages/ai/src/api/transform-messages.ts)。

#### 17. Nested Codemode

`executeCodemode() → CodemodeSandbox script → ctx.executeTool() → runToolCall() → toScriptValue() → 输出截断 / storeWrites`

证据：[execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts)、[runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)。

专题证据与完整控制/数据/状态边界：[Batch 44](batch-44-call-chains.md)。

## 43. Pi Master Architecture

**Confirmed**：Master Map 在前面机制分析后收敛，包含用户/Host、Runtime/Session、Agent/Context/Message/Model/Provider/LLM、工具与文件/Shell/搜索/MCP、Extension/Event/Config/Persistence/Workspace。普通helper不单独成架构节点。节点数与来源见 candidate 和全量校验。

专题证据与完整控制/数据/状态边界：[Batch 46](batch-46-master-map.md)。

## 最后单独回答的五个问题

### Q1 · 真正的 Agent Loop

**Confirmed**：`packages/agent/src/agent-loop.ts` 的 `runLoop()` 是模型与工具续轮核心，内层工具/steering、外层follow-up。`AgentSession._runAgentPrompt()` 在其外管理重试、压缩与边界续轮。因此agent_end、Agent idle和agent_settled不同。证据：[Batch3](batch-3-agent-loop.md)、[Batch44](batch-44-call-chains.md)。

### Q2 · 真正的 Context Management

**Confirmed**：`SessionManager.buildSessionProjection()` / `buildContextEntries()` / `projectContextEntry()` 负责branch、compaction与context edits；Session安装的request/next-turn hooks刷新；transformContext/convertToLlm/transcript/Provider继续变换。摘要切点和提交由compaction.ts与Session共同实现。证据：[Batch6](batch-6-context-management.md)、[Batch16](batch-16-session.md)。

### Q3 · 模型 Harness 适配厚度

**Confirmed**：不止SDK调用：身份/API分离、鉴权、物理路由、系统/工具delta重放、角色转换、工具ID/结果修复、strict/grammar schema、thinking budget/effort/signature、图片限制、stream/stop/usage归一、cache/affinity/retry均有实现。典型四API族深读；没有审计全部42 identities的每种边界，不能给准确价值百分比。证据：[Batch8](batch-8-provider-system.md)、[Batch33](batch-33-compatibility.md)。

### Q4 · 多于 API + Shell + File Tools 的内容

**Confirmed**：持续执行协议、输入队列/取消、统一工具hooks与并发结果、投影与压缩、分支会话/恢复、动态工具/Skill/Extension/MCP、鉴权/模型路由/跨协议、headless与TUI Hosts、用量/cache/评测设施。模型仍负责代码意义、计划与完成判断；核心无通用强制测试验收。证据：[Batch40](batch-40-core-value.md)、[Batch15](batch-15-coding-workflow.md)。

### Q5 · Mini Pi 最少模块

**Inference（概念最小集，不是已测量的 20% 代码）**：

| 层 | 必须行为 | Class / Function | 可复用证据 |
| --- | --- | --- | --- |
| 消息契约 | user/assistant/toolResult、id关联、usage/stop/error | Message / AssistantMessage / ToolResultMessage | [types.ts](../packages/ai/src/types.ts) |
| 流适配 | 一个Provider实现，文本/工具delta/终结 | AssistantMessageEventStream / stream() | [event-stream.ts](../packages/ai/src/utils/event-stream.ts)、[openai-completions.ts](../packages/ai/src/api/openai-completions.ts) |
| Loop | 请求→工具执行→结果回灌→继续/结束/取消 | runLoop / executeToolCalls / createToolResultMessage | [agent-loop.ts](../packages/agent/src/agent-loop.ts) |
| 运行所有权 | messages/tools/model、单run、queue/abort/finally | Agent.prompt / continue / abort / processEvents | [agent.ts](../packages/agent/src/agent.ts) |
| Context builder | 系统/当前对话/工具声明，输出与window边界 | normalizeContext / getCurrentSystemMessage | [transcript.ts](../packages/ai/src/utils/transcript.ts) |
| 工具层 | schema验证、read/edit/write/Shell、错误关联 | ToolDefinition / wrapToolDefinition / create*ToolDefinition | [index.ts](../packages/coding-agent/src/core/tools/index.ts) |
| Host | 读取输入、呈现流与错误、取消 | runPrintMode | [print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts) |

若要求可恢复长任务，再加入 JSONL/branch projection、压缩和有界重试。TUI、MCP、Skill、复杂扩展、虚拟路由、warm/eval 是目标驱动的可选层；删除它们不是在现有Pi中授权移除功能。

证据：[Batch41](batch-41-minimal-core.md)。
