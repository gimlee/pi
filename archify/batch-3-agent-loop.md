# Batch 3：Agent Loop

## Findings

**Confirmed**：真正循环是 `packages/agent/src/agent-loop.ts` 的 `runLoop()`，不是 CLI 输入循环，也不是名为 Agent 的类本身。内层处理工具调用和 steering，外层处理本来将结束时到来的 follow-up；`Agent` 管理这次运行的状态、取消信号和事件订阅。

**Confirmed**：编码产品还有一层 `AgentSession._runAgentPrompt()`：底层运行结束后检查重试、压缩恢复、剩余队列和 `agent_before_settle`，必要时调用 `Agent.continue()`。因此，一次用户 prompt 可以包含多个底层 run；每个 run 又可以包含多个 turn，每个 turn 有一次主模型响应及其工具结果。

**Confirmed**：模型决定输出什么工具名和参数，Harness 决定是否允许执行、如何验证、执行顺序、异常转结果及是否发起下一请求。Provider 解析协议流；Loop 消费统一事件，不直接解析供应商 HTTP/SSE 数据。

**Inference**：它是模型驱动的“响应 → 行动 → 观察 → 再响应”循环，可称为 ReAct 风格，但没有固定的 ReAct 文本语法或必须先产生计划的算法。停止是执行协议上的停止，不能证明用户目标已被语义验证。

## Evidence

| 编号 | 已读来源 / 定位范围 | 支持的事实 |
|---|---|---|
| L1 | [agent-loop.ts](../packages/agent/src/agent-loop.ts) 102～331 | 两种入口、真实双层循环、请求准备、完成与队列判定 |
| L2 | 同文件 333～476 | 工具声明差分、上下文转换、统一模型事件消费 |
| L3 | 同文件 478～681 | 截断调用失败、顺序/并行执行、事件与结果顺序 |
| L4 | 同文件 683～940 | 参数准备、schema 验证、前后 hooks、异常、terminate、结果消息 |
| L5 | [agent.ts](../packages/agent/src/agent.ts) 341～613 | abort、prompt/continue、activeRun、事件 reducer、最终清理 |
| L6 | [types.ts](../packages/agent/src/types.ts) 135～347、382～529 | turn/request hooks、队列、工具契约、AgentState/AgentEvent |
| L7 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 759～906、1073～1159 | 每请求投影、下一 turn 刷新、边界 hook、持久化时点 |
| L8 | 同文件 1775～2064、2387～2404 | 用户输入、Session 外层恢复循环、settle 和取消 |
| L9 | 同文件 2900～3039、3713～3766 | 压缩/length 恢复条件、一次 overflow 恢复、重试预算 |
| L10 | [sdk.ts](../packages/coding-agent/src/core/sdk.ts) 314～421；[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) 715～744 | streamFn 注入、超时/重试设置、Provider 分发 |
| L11 | [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) 570～900；[openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts) 433～809；[google-generative-ai.ts](../packages/ai/src/api/google-generative-ai.ts) 57～301 | 原生流 → 统一消息、usage 与停止原因 |
| L12 | [plan-mode/index.ts](../packages/coding-agent/examples/extensions/plan-mode/index.ts)；[subagent/README.md](../packages/coding-agent/examples/extensions/subagent/README.md)；[planner.md](../packages/coding-agent/examples/extensions/subagent/agents/planner.md)、[reviewer.md](../packages/coding-agent/examples/extensions/subagent/agents/reviewer.md) | 可选策略示例；不属于默认 Loop 固定阶段 |

核心实现整文件阅读；定位范围用于复查。分析基线与工作区边界见 [README](README.md)。本批是静态源码追踪，未调用真实模型或执行示例扩展。

## Key Source Files

[agent-loop.ts](../packages/agent/src/agent-loop.ts) 是算法；[agent.ts](../packages/agent/src/agent.ts) 是运行状态 owner；[types.ts](../packages/agent/src/types.ts) 是 hooks/事件/工具契约；[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 是编码层输入、历史、恢复与扩展集成；[sdk.ts](../packages/coding-agent/src/core/sdk.ts) 和 [model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) 连接模型。

## Key Types / Classes

| 类型 / 类 | 职责与边界 |
|---|---|
| `AgentSession` | 一次用户活动及其重试/恢复/settle；保存到 SessionManager |
| `Agent` / `ActiveRun` | 一次低层 run，拥有 AbortController、完成 promise、状态和队列 |
| `AgentLoopConfig` / `AgentContext` | 本次模型/选项/hooks；消息和可执行工具快照 |
| `PrepareRequestContext` / `AgentRequestUpdate` | 每次模型调用前可替换 context/model/thinking；不能直接返回待注入 messages |
| `PrepareNextTurnContext` / `AgentLoopTurnUpdate` | 已完成 turn 的消息、工具结果、上下文；可为下一 turn 返回 context/messages/选型 |
| `AgentTurnDecision` | `continue` 或 `end`；与供应商 stopReason 不同 |
| `AgentTool` / `AgentToolCall` | schema + execute 的实现；模型输出的调用数据 |
| `AgentToolResult` / `ToolResultMessage` | 执行结果；进入上下文的消息，字段不是完全相同 |
| `AgentEventSink` / `StreamFn` | 可等待的事件接收者；注入的模型流调用函数 |

## Key Functions

`AgentSession.prompt()` → `_runAgentPrompt()` → `Agent.prompt()` → `runPromptMessages()` → `runWithLifecycle()` → `runAgentLoop()` → `runLoop()` → `streamAssistantResponse()` → 注入的 `streamFn()`。

工具路径：`executeToolCalls()` → sequential/parallel → `prepareToolCall()` → `executePreparedToolCall()` → `finalizeExecutedToolCall()` → `createToolResultMessage()`。

继续路径：`finishTurn()`、`getSteeringMessages()`、`getFollowUpMessages()`、`prepareNextTurn()`；Session 的 `_handlePostAgentRun()` / `_runBeforeSettleBoundary()` → `Agent.continue()`。

## Control Flow

### 1. 从“修改这个 Bug”开始

以下正常路径为 **Confirmed**，可选 hooks 在没有 handler 时直接通过。

| 次序 | 调用 / 行为 | 实际结果 |
|---|---|---|
| 1 | Host → `AgentSession.prompt(text, options)` | 扩展命令先行；input handler 可 handled/transform；再展开 skill/template |
| 2 | 已在 streaming 时 | 必须指定 steer/followUp，入队后返回；不同时创建第二个 activeRun |
| 3 | 空闲路径 | flush 待处理消息，检查模型/鉴权，检查上次响应压缩；执行 before_agent_start 与图片规范化 |
| 4 | 构造待发送消息 | 系统段落 patch 在前，然后 user、pending nextTurn、扩展 custom 消息；准备可执行工具 loadout |
| 5 | `_runAgentPrompt` → `Agent.prompt` | Agent 创建 activeRun、signal，isStreaming=true；发 agent_start/turn_start 和输入 message_start/end |
| 6 | `runLoop` → prepareRequest | Session 以历史投影替换 context；可路由虚拟模型；随后转换 request context |
| 7 | `streamAssistantResponse` | transformContext → convertToLlm → normalizeContext → streamFn；SDK streamFn → ModelRuntime → Provider |
| 8 | Provider → 统一流事件 | start、text/thinking/toolcall 的 start/delta/end、done/error；Loop 更新部分消息并发 message_update |
| 9 | message_end | Loop 保存最终 assistant 到局部 context；Agent reducer 加入 state.messages；Session 先扩展/公开事件，再持久化 |
| 10 | 查看 `content` 内的 toolCall | 错误/aborted 先退出；length 调用全部失败；否则准备并执行工具 |
| 11 | 工具结果 | execution_end → toolResult message_start/end；Session 保存；整批结束后 Loop 追加局部 context/newMessages |
| 12 | finishTurn / turn_end | end 立即终止；否则工具要求继续、steering、follow-up、显式 continue 依次影响下一请求 |
| 13 | 下一 turn | prepareNextTurn，可压缩/刷新工具提示；turn_start；每次还会再次 prepareRequest |
| 14 | 底层 agent_end | 监听器被等待；finally 才清理 activeRun。Session 检查恢复与边界，最终发 agent_settled |

一个具体消息轨迹是：`system + user("修改 Bug") → assistant(toolCall read) → toolResult(read 输出) → assistant(toolCall edit) → toolResult(edit 输出) → assistant(text)`。模型选择 read/edit 及参数；Harness 按工具定义执行。最后 assistant 没有调用且没有队列/继续决策时，底层 run 才结束。

### 2. 六个“谁”

| 问题 | 精确责任 |
|---|---|
| 谁调用 Model？ | Loop 的 streamAssistantResponse 调用注入 streamFn；默认 SDK 经 ModelRuntime.streamSimple 选择 Provider |
| 谁解析 Response？ | Provider adapter 解析原生流/JSON 和 stopReason；Loop 消费 AssistantMessageEventStream |
| 谁决定执行 Tool？ | Loop 按 assistant.content 检出调用，prepareToolCall 解析注册工具、准备/验证参数并运行 beforeToolCall；模型不直接执行代码 |
| 谁追加 Tool Result？ | Loop 为局部 context/newMessages 追加；Agent.processEvents 对 state.messages 追加；Session 在 message_end 写历史 |
| 谁发起下一轮？ | runLoop 的条件与队列；prepareNextTurn 刷新下一轮。低层结束后的恢复由 Session 调用 Agent.continue |
| 谁判断完成？ | Loop 判断没有调用/待输入/继续要求；Session 判断没有恢复或边界继续。没有默认语义 Reviewer 验收 |

### 3. 工具批次与顺序

**Confirmed**：默认 `parallel`。配置为 sequential，或本批任一工具声明 `executionMode="sequential"`，整批顺序执行；不是只把该工具从并行批次中单独取出。

顺序模式逐个发 execution_start、准备、执行、after hook、execution_end、结果消息；取消后可停止余下工具。并行模式先按调用顺序做全部 preflight，再 `Promise.all` 执行。execution_end 体现实际完成顺序；结果 message_start/end 按 assistant 调用顺序输出；上下文追加也按该顺序。收到工具参数 delta 时不提前执行，等待整条 assistant 最终结果。

prepareArguments 在 schema 验证前；验证后才 beforeToolCall。未知工具、参数验证失败、before hook 阻止/抛错形成即时错误结果，不运行 execute 或 after hook。execute 抛错转为错误结果并经过 after hook；after hook 自身抛错也形成错误结果。部分更新在执行 promise 结束后不再接受，已开始的更新会被等待。

`terminate` 只有整批已经产生的结果全部为 true 才使这批“不要求因工具而继续”；队列或 finishTurn 仍能继续。它不是不可覆盖的全局终止。`runToolCall()` 可复用执行流水线，但自身不发 Loop 事件或写上下文；嵌套调用记录是 Session 的另一个机制。

### 4. Stop Conditions

| 条件 | 底层 Loop 行为 | 编码 Session 的后续 |
|---|---|---|
| 无工具的最终文本 | 不是看到文本就退出；先 finishTurn，再检查 steer/followUp/continue | 仍可恢复/边界继续；正常最终发 settled |
| `stop` / `toolUse` | 实际 toolCall 内容驱动执行，不能只看标签 | stopReason 留作历史与恢复判定 |
| `length` | 有调用：全部形成错误结果，不执行，通常继续向模型反馈；无调用：按普通无工具 turn 处理 | 满足 isRecoverableLength、同一模型、仍在投影且启用压缩时，可一次 compact-and-retry |
| `error` / `aborted` | 调用 finishTurn 但忽略其继续决策，turn_end + agent_end，立即 return | retryable error 按预算退避；overflow 走压缩；用户取消抑制继续 |
| finishTurn `end` | 当前 turn 已完成工具后立即结束，不再排队列 | Session 层仍检查自己的恢复/边界；不是应用退出 |
| finishTurn `continue` | 至少再请求一次；自然工具/队列续轮已经满足时，不额外重复一轮 | 扩展边界只能在可继续的 context 下请求继续 |
| 工具失败 | error toolResult，通常继续模型 | 不等于 assistant error；模型有机会调整调用 |
| 工具 terminate | 整批全 true，取消工具导致的续轮 | 其他继续条件仍有效 |
| max turns | **Not Present：已读默认 AgentLoopConfig/runLoop 没有全局 maxTurns** | 重试预算限制失败重试，不限制正常工具轮次 |
| max tokens | 单次模型输出限制；不是整个任务 token 预算 | 可能触发 length/overflow 恢复；不代表目标完成 |
| abort / cancel | Agent.abort 置 signal；等待执行代码/Provider 合作退出；不强制杀死任意 promise | Session.abort 同时取消 retry/compaction/branch-summary，并等待 Session idle |
| timeout | Loop 没有全局计时器；默认 SDK 传每请求 timeout/连接超时设置 | adapter 支持各异；请求错误可重试。Google GenAI 当前 buildParams 未应用 timeoutMs，不能声称所有 adapter 同样生效 |
| setup / hook / listener 抛错 | Agent.runWithLifecycle 捕获并生成 synthetic error/aborted assistant；finally 清理 | 错误事件监听器再次抛错可能向外传播；不保证任意 handler 故障都被完全吞掉 |

统一类型还有 pending/deferred；默认循环没有专门轮询 deferred 或 pending 的分支。pending 是适配器生成中的初始状态，已读三种适配器检查缺少终止原因；deferred 的其他 API 用法留后续审计。

### 5. Agent Strategy

| 策略 | 默认路径 | 仓库内其他证据 / 等级 |
|---|---|---|
| ReAct | 交替模型/工具/观察，没有固定 ReAct parser | **Inference**：行为可归类为 ReAct 风格 |
| Plan-and-Execute | **Not Present：固定规划/执行两阶段** | **Confirmed**：plan-mode 示例有只读规划、编号步骤、执行模式与 DONE 标记 |
| Planner | **Not Present：默认专用 Planner 对象/阶段** | 示例 planner.md 定义规划角色，明确仅分析 |
| Reflection | **Not Present：默认强制自反思 pass** | 模型可自发反思；扩展可加续轮，不等于内置算法 |
| Reviewer | **Not Present：默认专用验收阶段** | reviewer.md 与 implement-and-review 模板存在；模板要求调用可选 subagent |
| Critic | **Not Present：默认专用 Critic 阶段** | 本批未审计所有外部扩展；不作全仓/生态不存在声明 |
| Todo | **Not Present：默认 Loop 的 Todo 状态/工具** | plan-mode 示例维护 todoItems 并写 custom entry；不是核心状态 |
| Task | **Not Present：默认 Loop 的 Task 调度器** | 示例 subagent 文档定义 agent/task、tasks、chain；Anthropic 名称映射里的 Task/TodoWrite 只是名称兼容，不创建工具 |
| Sub-agent | **Not Present：默认 Loop 自动派生子 Agent** | 示例 README 明确独立 pi 进程、隔离 context、single/parallel/chain；本批未审计其完整执行实现，不宣称已验证进程/并发细节 |

“Not Present”限定已读默认路径及其内置装配，不否认用户装载扩展或模型生成计划的能力。

## Data Flow

`AgentMessage[]` 是历史与自定义消息；`transformContext` 可改变本次请求视图；`convertToLlm` 得到统一 `Message[]`；`normalizeContext` 得到 `TranscriptContext`；adapter 生成原生 request。原生响应反向变成 assistant content/events，工具 execute 返回 AgentToolResult，再变为 ToolResultMessage 回灌。

局部 context、Agent state、Session 原始历史、Session projection 是四个不同层次。错误尝试可保留原始历史，同时用持久化 context_edit 从模型投影省略。扩展 message_end 可以同 role 替换最终消息，Session 原地同步该对象后保存；不能把 Provider 原始输出等同最终持久化字节。

## State

**Confirmed**：没有完整阶段 enum。明确状态是 activeRun、isStreaming、streamingMessage、pendingToolCalls、errorMessage、两条队列；Loop 内还有 hasMoreToolCalls、pendingMessages、lastCompletedTurn、explicitContinuation。

| 可观察状态 | 源码条件 / 变化 |
|---|---|
| Agent idle | activeRun 缺失；finishRun 后 isStreaming=false |
| Agent active | activeRun 存在；整个 run 的 isStreaming=true，包含工具、准备与等待 listener |
| Abort requested | activeRun.signal.aborted=true；仍 active，直到合作退出与 finally |
| 响应部分内容 | message_start/update 设置 streamingMessage；message_end/agent_end 清空 |
| 待工具 | execution_start 加 ID，execution_end 删除；并行可多 ID；finally 清空 |
| Session active / idle | _isAgentRunActive 跨低层 run 和恢复；idle 为 !active && !isCompacting |
| 三个结束边界 | agent_end：不再有低层事件；finishRun：低层 idle；agent_settled：Session 恢复/边界处理结束 |

Diagram 3C 将前三个明确谓词画为状态映射，属于 **Inference 的展示命名**，不是源码定义的正式 FSM。没有虚构 Planning/Reviewing/Approved 状态。

## Design Decisions

- **Confirmed**：事件接收函数可被 await，Agent 先更新状态再逐一等待 listener；Session 能在下一请求前完成持久化。**Inference**：这减少了请求读到未完成历史的风险，但慢 handler 会阻塞继续和 idle。
- **Confirmed**：finishTurn、prepareNextTurn、prepareRequest 分别处在完成、下一轮和每请求边界。**Inference**：通用循环可复用，Coding Session 可加入压缩、路由、提示更新而不复制循环。
- **Confirmed**：工具截断全批失败而不是尝试执行解析出的部分参数。**Inference**：防止输出长度截断导致不完整操作；必要的安全行为位于 Harness。
- **Confirmed**：模型可见工具声明与可执行实现分开。**Inference**：方便持久化可 JSON 化的接口，同时保留执行函数和扩展 hooks。

规划与文件选择主要是 Model capability；工具验证、执行、队列、取消、事件和恢复是 Harness capability；选择工具并利用反馈修复代码属于 Hybrid。

## Unknowns

没有运行真实 Provider、故障注入、并发工具或取消实验；这里的顺序保证来自源码。工具如何取消具体进程、Provider 超时的全部 transport 差异、deferred 完整生命周期、第三方扩展的策略和无限继续行为未在本批证明。语义成功率不能由 stopReason 推断。

## Archify Diagram

- [Diagram 3A：Pi Agent Runtime Workflow](diagrams/diagram-3a-runtime/diagram-3a-runtime.html) · workflow，用户输入到 Session 恢复与 settled。
- [Diagram 3B：Pi Agent Loop Lifecycle](diagrams/diagram-3b-loop/diagram-3b-loop.html) · lifecycle，主请求、工具、续轮和底层终止。
- [Diagram 3C：Pi Agent State Machine](diagrams/diagram-3c-state/diagram-3c-state.html) · lifecycle，明确运行谓词的映射，无虚构阶段。

候选、源码范围与自动交付/浏览器检查记录保存在各图目录；总体校验见 [README](README.md)。

## Follow-up

Batch 4 深入消息字段及 adapter 映射；Batch 5 深入提示构造。Batch 6 完整验证 projection/compaction 算法，Batch 7 分析模型抽象；后续 Tool、Provider、Error/Retry、Concurrency 批次补 transport、取消、嵌套调用与故障实验。
