# Pi 源码分析：Batch 0～5

任务来源：[promt/archify-promt.md](../promt/archify-promt.md)。已完成仓库侦察、概念定位、Startup / Runtime、Agent Loop、Message Model 和 Prompt System；Batch 6 及以后只列调查方向，不作为已完成成果。

## 阅读入口

| 报告 | 内容 | 图 |
|---|---|---|
| [Batch 0](batch-0-repository.md) | 13 个包、入口、核心候选、依赖与阅读计划 | [Diagram 0：仓库一级模块](diagrams/diagram-0-repository/diagram-0-repository.html) |
| [Batch 1](batch-1-positioning.md) | Coding Agent / Harness / Runtime 的定位和价值分布 | 复用 Diagram 0 与 2B；任务没有要求 Diagram 1 |
| [Batch 2](batch-2-startup-runtime.md) | 启动顺序、对象创建、所有权、模式绑定和退出 | [Diagram 2A：启动过程](diagrams/diagram-2a-startup/diagram-2a-startup.html)、[Diagram 2B：运行时职责](diagrams/diagram-2b-runtime/diagram-2b-runtime.html) |
| [Batch 3](batch-3-agent-loop.md) | 双层循环、六个执行责任、工具并发、停止条件、策略与结束边界 | [Diagram 3A：运行工作流](diagrams/diagram-3a-runtime/diagram-3a-runtime.html)、[Diagram 3B：循环生命周期](diagrams/diagram-3b-loop/diagram-3b-loop.html)、[Diagram 3C：运行状态](diagrams/diagram-3c-state/diagram-3c-state.html) |
| [Batch 4](batch-4-message-model.md) | 统一消息、内容/元数据、流事件、工具结果和三种协议映射 | [Diagram 4：消息数据流](diagrams/diagram-4-messages/diagram-4-messages.html) |
| [Batch 5](batch-5-prompt-system.md) | 指令来源、段落顺序、动态 hooks、请求构造与供应商差异 | [Diagram 5A：提示组成](diagrams/diagram-5a-composition/diagram-5a-composition.html)、[Diagram 5B：提示构建](diagrams/diagram-5b-build/diagram-5b-build.html) |

HTML 可直接用浏览器打开。每张图保留 `candidate.json`、交付记录和浏览器检查记录，可继续编辑与复验。

## 分析基线与证据等级

- 分析日期：2026-10-03，Asia/Shanghai。
- 仓库：`F:/github/pi`；origin：`https://github.com/gimlee/pi`。
- 分支：`issue-fix/9946-0bd6fea8`。
- HEAD：`9b3c19da5cffc4c5e8b6bd74c45abc1ab6bfcd16`。
- 报告以当前工作区源码为准，源码链接为仓库相对路径，行号以本次快照为准。HTML 引用固定 HEAD 的已提交内容，使用 `local-only`，不把未提交改动当作该提交证据。
- 开始分析时已存在三处修改：`packages/coding-agent/src/modes/interactive/components/bash-execution.ts`、`packages/coding-agent/src/modes/interactive/interactive-mode.ts`、`packages/coding-agent/test/bash-execution-width.test.ts`；另有未跟踪 `promt/`。这些内容未被本次修改。相关改动是 Bash 显示宽度/边距，未改变本报告涉及的启动和 Runtime 创建路径。

**Confirmed**：已阅读的实现或实际调用点直接支持。**Inference**：基于这些行为作出的架构解释或设计动机判断。**Unknown**：本次没有验证，不能理解为能力不存在。报告不使用未经运行的测试作为通过证据。

## 当前结论

**Inference**：默认 Pi 产品是由可复用 Agent Runtime 支撑的 Coding Agent，其主要系统价值在 Harness。这里的 Harness 指模型外部负责上下文、工具执行、恢复、会话和扩展的执行环境。模型负责推理和生成；Pi 负责把这些输出变成可继续、可取消、可保存、可扩展的编码过程。

**Confirmed**：没有一个对象同时拥有进程、终端、所有会话和每次模型运行。`main` 与运行模式 Host 管理应用 I/O 和退出，`AgentSessionRuntime` 持有并替换当前 session/services，`AgentSession` 管理编码会话行为，`Agent` 管理当前运行与底层循环。

**Confirmed**：底层 `runLoop()` 决定工具与队列续轮；`AgentSession._runAgentPrompt()` 还可以在底层结束后重试、压缩恢复和边界续轮。`agent_end`、Agent finally 清理后的 idle、Session 的 `agent_settled` 是不同结束边界。默认循环没有全局 maxTurns，也没有固定 Planner / Reviewer 验收阶段。

**Confirmed**：内部统一消息不等同供应商 wire format。系统段落与工具声明记录在 transcript，Provider 再按能力重放或折叠；统一 `thinking`、签名和工具关联需要协议适配。默认 Coding Prompt builder 没有三套 Claude/GPT/Gemini 模板，差异还存在于 adapter 的角色、系统更新、图像、推理与 OAuth 身份前缀处理。

## 五个核心问题：阶段性答案

任务要求的最终五问将在后续深度分析后定稿；以下只反映 Batch 0～5 的证据。

| 问题 | 当前答案 | 等级 / 后续边界 |
|---|---|---|
| Q1：真正的 Agent Loop 在哪里？ | `packages/agent/src/agent-loop.ts` 的 `runLoop()`；内层处理工具/steering，外层处理 follow-up。编码层 `_runAgentPrompt()` 还管理重试、压缩和 settle。 | Confirmed；责任、并发和停止矩阵见 Batch 3，未做真实模型故障实验。 |
| Q2：Context Management 在哪里？ | `SessionManager.buildSessionProjection()` 形成分支与压缩感知的消息投影；`AgentSession` 在请求边界应用它；`transformContext`、`convertToLlm` 和 Provider 规范化继续处理。 | Confirmed；不把 Context 简化为一个数组，完整机制留后续 Batch。 |
| Q3：针对模型做了多少适配？ | 已确认 thinking/签名重放、图片转换、工具 ID/结果修复、系统/工具更新、角色与 schema 编码、usage/stop 归一、OAuth 身份前缀，以及虚拟路由与鉴权。 | Confirmed：Batch 4～5 对 Anthropic Messages / OpenAI Responses / Google GenAI 作具体比较；其他 API 与缓存策略未全部审计，不能量化总体厚度。 |
| Q4：比 API + Shell + File Tools 多什么？ | 持续工具循环、输入队列与取消、分支会话与恢复、上下文投影与压缩、提示/工具动态装载、扩展事件、模型与鉴权管理、多个 Host。 | Confirmed；具体价值分布见 Batch 1。 |
| Q5：Mini Pi 最少需要什么？ | 消息/模型流协议、循环与状态、工具 schema/执行/结果回灌、上下文构造、输入输出 Host；若要继续会话与恢复，再加入日志/投影和取消/重试。 | Inference：功能目标决定最小集。暂不按代码行数声称“20% 就够”；Extension、MCP、TUI 等不是所有 Mini Pi 的必需项。 |

## 校验记录

九图使用 Archify 3.0.1，`quality=showcase`。最终候选均通过 `validate`、`deliver`、严格 `check`、真实浏览器 `browser-check`；零错误、零警告。源码范围按固定提交验证。未进行截图人工视觉审查，`visualReview=not-requested`；自动检查通过不等于人工确认视觉质量。Diagram 2A 与 3A 有路由绕行提示，3B 有已处理交叉的人工审查建议，不影响自动检查通过。3C 是源码明确运行谓词的展示映射，不是源码定义的正式阶段枚举。

[analysis-checks.json](analysis-checks.json) 汇总六份报告的固定标题、文档本地链接和九张图的候选/HTML/浏览器记录哈希核对；它不代替各图的实际交付记录。

| 图型 | 最终检查记录 | 浏览器记录 |
|---|---|---|
| Diagram 0 · architecture，15 个一级组件 | [finalize-summary](diagrams/diagram-0-repository/review-3/diagram-0-repository.finalize-summary.json) | [browser-check](diagrams/diagram-0-repository/review-3/diagram-0-repository.browser-check.json) |
| Diagram 2A · workflow，启动执行顺序 | [finalize-summary](diagrams/diagram-2a-startup/diagram-2a-startup.finalize-summary.json) | [browser-check](diagrams/diagram-2a-startup/diagram-2a-startup.browser-check.json) |
| Diagram 2B · architecture，12 个运行职责 | [finalize-summary](diagrams/diagram-2b-runtime/review-2/diagram-2b-runtime.finalize-summary.json) | [browser-check](diagrams/diagram-2b-runtime/review-2/diagram-2b-runtime.browser-check.json) |
| Diagram 3A · workflow，输入到恢复与 settled | [finalize-summary](diagrams/diagram-3a-runtime/diagram-3a-runtime.finalize-summary.json) | [browser-check](diagrams/diagram-3a-runtime/diagram-3a-runtime.browser-check.json) |
| Diagram 3B · lifecycle，模型/工具/续轮/终止 | [finalize-summary](diagrams/diagram-3b-loop/diagram-3b-loop.finalize-summary.json) | [browser-check](diagrams/diagram-3b-loop/diagram-3b-loop.browser-check.json) |
| Diagram 3C · lifecycle，idle/active/abort requested | [finalize-summary](diagrams/diagram-3c-state/diagram-3c-state.finalize-summary.json) | [browser-check](diagrams/diagram-3c-state/diagram-3c-state.browser-check.json) |
| Diagram 4 · dataflow，消息与协议归一 | [finalize-summary](diagrams/diagram-4-messages/diagram-4-messages.finalize-summary.json) | [browser-check](diagrams/diagram-4-messages/diagram-4-messages.browser-check.json) |
| Diagram 5A · dataflow，指令组成与请求投影 | [finalize-summary](diagrams/diagram-5a-composition/diagram-5a-composition.finalize-summary.json) | [browser-check](diagrams/diagram-5a-composition/diagram-5a-composition.browser-check.json) |
| Diagram 5B · workflow，提示到最终 payload | [finalize-summary](diagrams/diagram-5b-build/diagram-5b-build.finalize-summary.json) | [browser-check](diagrams/diagram-5b-build/diagram-5b-build.browser-check.json) |

只增加分析文档和图，没有修改项目代码、运行构建或项目测试、提交 Git。
