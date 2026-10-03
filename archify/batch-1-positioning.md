# Batch 1：Pi 到底是什么

## Findings

**Inference**：以默认 CLI 和编码 SDK 为分析对象，Pi 首先是 **Coding Agent 与 Agent Harness**。Harness 是模型外部的执行环境：负责输入处理、上下文、工具、会话、恢复和扩展。它包含一个可复用的 **Agent Runtime**，CLI/TUI/RPC 是面向不同调用者的 Host。仓库同时提供框架式的组合/扩展 API，但不能把所有包都视为默认编码 Runtime。

### 定位依据

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

### 一条具体执行轨迹

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

### 价值分布

不按代码行数分配价值百分比；这些类别是阅读和改动风险的功能分区，同一文件可跨类别。

| 类别 | 关键源码 / 行为 | 为什么影响结果 |
|---|---|---|
| Core Intelligence / Harness | `agent-loop.ts`；`AgentSession.prompt`、request/next-turn/tool hooks；Session projection、压缩协调 | 决定模型看到什么、工具能否执行、何时继续/恢复/结束；这里的 intelligence 是行为控制，不是模型权重或训练 |
| Infrastructure | SettingsManager、SessionManager、config paths、output guard、HTTP dispatcher | 配置、会话恢复和输出协议可靠性；错误会改变有效模型/工具和恢复行为 |
| Integration | ModelRuntime、Provider 工厂/API adapter、MCP、codemode | 连接模型/外部工具，转换鉴权、事件、schema 与请求约束 |
| Presentation | InteractiveMode、TUI 组件、print/RPC 输出 | 交互编辑、流展示、命令入口；TUI Host 也处理命令和资源准备，不是完全被动的 View |
| Glue Code | `cli.ts`、services/SDK 工厂、`main` 的分发连接 | 大量工作是装配，但顺序、cwd 和注册时机影响行为；不能把整个 `main` 简单视为可随意删除的样板代码 |

职责存在性为 **Confirmed**；“核心价值/改动风险更高”属于 **Inference**。没有通过性能测试或用户数据量化价值。

### Model capability / Harness capability / Hybrid

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

## Evidence

| 结论 | 已阅读源码 / 精确符号 |
|---|---|
| Loop 真正执行多轮请求和 tool result 回灌 | [agent-loop.ts](../packages/agent/src/agent-loop.ts) `runLoop`，163～309；`streamAssistantResponse`，381 起；`executeToolCalls`，508 起 |
| Agent 是执行状态对象，不直接创建 UI/文件会话 | [agent.ts](../packages/agent/src/agent.ts) `Agent`，188 起；`prompt`，371 起；`runWithLifecycle`，507 起 |
| SDK 将编码能力装配到通用 Agent | [sdk.ts](../packages/coding-agent/src/core/sdk.ts) `new Agent`，387～418；`new AgentSession`，437～455 |
| Session 安装请求边界、工具和扩展 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) constructor，462～497；`_installAgentRequestProjection`，759 起；`_buildRuntime`，3558～3610 |
| 原始日志和模型 Context 不同 | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) `buildSessionProjection`，532～573；`_persist`，1178 起 |
| Provider 路由不是整个 Agent | [model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) `prepareRequest`，650～689；`streamSimple`，715～744 |
| Host 复用 Session | [main.ts](../packages/coding-agent/src/main.ts) 949～998；[print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts) `bindExtensions` 与 `session.prompt` |
| Core 与 Presentation 不是完全无依赖 | `agent-session.ts` 54 行附近导入交互 theme，用于导出等能力；[coding-agent/index.ts](../packages/coding-agent/src/index.ts) 也导出 UI API |

## Key Source Files

定位最有解释力的是 [agent-loop.ts](../packages/agent/src/agent-loop.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)。仅阅读 CLI 或 Provider wrapper 会漏掉 Harness。

## Key Types / Classes

- `Agent` / `AgentState`：当前运行和执行状态。
- `AgentSession` / `AgentSessionConfig`：编码会话行为与依赖装配。
- `AgentSessionRuntime`：可替换的当前会话与 services。
- `SessionManager` / `SessionProjection`：原始历史与模型上下文贡献。
- `ModelRuntime` / `Model` / `Provider`：模型描述、模型/鉴权运行服务与请求实现。
- `AgentTool` / `ToolDefinition` / `ExtensionRunner`：执行契约、注册定义与扩展事件调用。

## Key Functions

`AgentSession.prompt`、`_runAgentPrompt`、`_preparePromptAndToolLoadout`、`_installAgentRequestProjection`、`Agent.prompt`、`runAgentLoop`、`runLoop`、`executeToolCalls`、`buildSessionProjection`、`ModelRuntime.streamSimple`。

## Control Flow

**Confirmed**：Session 可在底层 `agent_end` 后继续执行 retry/overflow 恢复和排队输入，最终才发出 `agent_settled`。因此 UI 把“模型一次循环结束”等同于“整个编码会话已空闲”会丢失编码层工作。来源：`AgentSession._runAgentPrompt`、`_emitAgentSettled`；RPC 通过 `agent_settled` 检查扩展 shutdown 请求。

**Confirmed**：底层 loop 的 hooks 可以改变下一轮模型、Context 和 thinking，以及 turn 的继续/停止决定。模型输出提供行动意图，Harness 执行边界决定如何使用它。

## Data Flow

**Confirmed**：SessionManager 的 entries 含状态记录、分支/压缩摘要、context_edit 等；这些 entries 并非原样发送模型。`buildSessionProjection` 选择当前 leaf 路径、应用最新压缩与 context edits，再形成 model-visible messages。SDK 的 `convertToLlm` 和 request hooks 继续投影；Provider 再处理协议。

具体例子：先前 tool result 可被 append-only `context_edit` 排除出模型 Context，同时原始日志仍保留该结果。这样恢复/展示所需历史与下一次模型请求内容不必相同。这是 Harness 数据管理，不是模型端记忆。

## State

**Confirmed**：Agent 保存进行中的 messages/tools/model 与 steering/follow-up 队列；SessionManager 保存历史树/leaf；Session 持有恢复控制器、工具注册和 Prompt 状态；Runtime 保存当前 Session 的对象引用；Host 保存 I/O 和 UI 状态。恢复会话需要重建这些层，不只是重放最后一条文本。

## Design Decisions

| 问题 / 具体例子 | 解决方式与必要性 | 等级 |
|---|---|---|
| CLI、RPC 与 SDK 都要使用相同工具/上下文行为 | `AgentSession` 集中 coding policy，Host 提供 I/O 与 extension bindings；避免每个入口各写一套 Harness | Inference；复用调用点 Confirmed |
| 通用循环要能由不同模型库驱动 | 注入 `streamFn` 和消息转换/hook；对可复用 Agent 必要，复杂 Provider catalog 属于更高层可选机制 | Inference；注入行为 Confirmed |
| 历史要保留但下一请求不能总带全部内容 | append-only 日志 + 分支/压缩感知 projection；需要恢复和分支时有价值，最小一次性 Agent 可简化 | Inference；实现 Confirmed |
| extension 在 load 时还没有 UI/session 上下文 | 先注册，再由 Host `bindExtensions`；多 Host 支持需要这条边界 | Inference；绑定顺序 Confirmed |

这些是从实现推导的设计解释，不是已查证的作者原话。

## Unknowns

- 完整 Provider adapter 是否存在模型专用 prompting/序列化策略，以及其数量和收益。
- 所有恢复路径、队列竞争、tool side effects 和故障一致性保证。
- 所有工具对代码仓库的实际理解深度；不能仅因目录叫 tools 就判断有语义代码索引。
- durable Harness 与默认 AgentSession 是否有未来统一计划；源码不能证明路线图。
- 在给定用户任务上的效果/成本优势；没有进行 eval 或模型调用。

## Archify Diagram

任务未要求单独 Diagram 1。定位判断使用 [Diagram 0](diagrams/diagram-0-repository/diagram-0-repository.html) 对照包边界，并使用 [Diagram 2B](diagrams/diagram-2b-runtime/diagram-2b-runtime.html) 对照对象职责。图中的调用/持有事实为 Confirmed；“哪层价值最高”不作为已证实的定量图形。

## Follow-up

[Batch 2](batch-2-startup-runtime.md) 从入口确认创建顺序与 lifecycle owner；Batch 3 再逐条验证本报告缩短轨迹中的每个 loop 分支。后续 Minimal Core 应先确定目标能力，再决定删减模块，不直接按 20% 行数裁剪。

如果不用 Pi，而直接调用 LLM API + Tool Calling，你仍需实现把 tool call 变成多轮可执行任务的循环、schema 验证与结果回灌、取消和输入队列、模型/鉴权与请求约束、提示和上下文装载、分支会话及恢复、压缩/重试边界、扩展 hooks，以及适合你的输入输出 Host。模型已经能推理和选择工具；Pi 的主要补充是让这些能力在真实编码会话中持续运行。是否需要 TUI、MCP、codemode 或 durable 存储，则取决于你的应用目标。
