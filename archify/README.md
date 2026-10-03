# Pi Agent 源码分析：Batch 0～48

任务：[promt/archify-promt.md](../promt/archify-promt.md)。49 个 Batch 的固定十二节报告已齐全；本轮续完 Batch 6～48，增加机制图、综合分析、20 个核心抽象、30 个阅读文件、17 条调用链、20 项开发动作和最终研究报告。

## 阅读入口

- [《Pi Agent 源码深度研究报告》](pi-agent-source-study.md)：43 节及最后单独回答的 Q1～Q5。
- [Pi Agent Master Architecture Map](diagrams/diagram-46-master/diagram-46-master.html)：20 个核心节点。
- [完整图集](diagram-atlas.md)：43 张 HTML 图、28 项必需概念映射和逐图检查记录。
- [二次开发地图](batch-45-development-map.md)、[阅读路线](batch-43-reading-path.md)、[关键调用链](batch-44-call-chains.md)。

HTML 可直接用浏览器打开；候选 JSON 和检查记录保留在各图目录。

## 分析基线与证据等级

研究快照：2026-10-03，Asia/Shanghai；仓库 F:/github/pi，origin https://github.com/gimlee/pi；固定提交 `9b3c19da5cffc4c5e8b6bd74c45abc1ab6bfcd16`。此前记录的分支为 issue-fix/9946-0bd6fea8。最终检查时工作区已推进到 `4196b29608898e61d77ef3d1f9a148eede45771f`，分支 `main`；本轮未提交或切换分支。

报告基于调查时读取的源码，图引用固定提交并使用 local-only。相对源码链接打开当前工作区，基线行号应通过旧提交确认。后续 MCP 局部覆盖/OAuth、模型生成及其他变化见 [基线推进说明](baseline-drift.md)。原有三个 Bash/TUI 文件修改未由本轮修改；它们的工作区状态随后由外部仓库推进改变。

**Confirmed**：读取的实现或调用点直接支持。**Inference**：结构推导、价值判断或设计解释。**Unknown**：未验证的边界；不代表能力不存在。典型四 API 族深入分析，未逐行审计全部 42 Provider 的所有边界。

## 当前结论

**Inference**：默认 Pi 产品是由可复用 Agent Runtime 支撑的 Coding Agent，其主要系统价值在 Harness。这里的 Harness 指模型外部负责上下文、工具执行、恢复、会话和扩展的执行环境。模型负责推理和生成；Pi 负责把这些输出变成可继续、可取消、可保存、可扩展的编码过程。

**Confirmed**：没有一个对象同时拥有进程、终端、所有会话和每次模型运行。`main` 与运行模式 Host 管理应用 I/O 和退出，`AgentSessionRuntime` 持有并替换当前 session/services，`AgentSession` 管理编码会话行为，`Agent` 管理当前运行与底层循环。

**Confirmed**：底层 `runLoop()` 决定工具与队列续轮；`AgentSession._runAgentPrompt()` 还可以在底层结束后重试、压缩恢复和边界续轮。`agent_end`、Agent finally 清理后的 idle、Session 的 `agent_settled` 是不同结束边界。默认循环没有全局 maxTurns，也没有固定 Planner / Reviewer 验收阶段。

**Confirmed**：内部统一消息不等同供应商 wire format。系统段落与工具声明记录在 transcript，Provider 再按能力重放或折叠；统一 `thinking`、签名和工具关联需要协议适配。默认 Coding Prompt builder 没有三套 Claude/GPT/Gemini 模板，差异还存在于 adapter 的角色、系统更新、图像、推理与 OAuth 身份前缀处理。


## 全部 Batch

| Batch | 报告 / 内容 |
| --- | --- |
| 0 | [Batch 0：Repository Reconnaissance](batch-0-repository.md) |
| 1 | [Batch 1：Pi 到底是什么](batch-1-positioning.md) |
| 2 | [Batch 2：Startup 与 Runtime](batch-2-startup-runtime.md) |
| 3 | [Batch 3：Agent Loop](batch-3-agent-loop.md) |
| 4 | [Batch 4：Message Model](batch-4-message-model.md) |
| 5 | [Batch 5：Prompt System](batch-5-prompt-system.md) |
| 6 | [Batch 6：Context Management](batch-6-context-management.md) |
| 7 | [Batch 7：Model Abstraction](batch-7-model-abstraction.md) |
| 8 | [Batch 8：Provider System](batch-8-provider-system.md) |
| 9 | [Batch 9：Reasoning / Thinking](batch-9-reasoning-thinking.md) |
| 10 | [Batch 10 · Tool Runtime](batch-10-tool-runtime.md) |
| 11 | [Batch 11 · Core Tools](batch-11-core-tools.md) |
| 12 | [Batch 12 · File Read / Edit / Write](batch-12-file-operations.md) |
| 13 | [Batch 13 · Shell Lifecycle](batch-13-shell.md) |
| 14 | [Batch 14 · Repository Understanding](batch-14-repository-understanding.md) |
| 15 | [Batch 15 · Coding / Bugfix Workflow](batch-15-coding-workflow.md) |
| 16 | [Batch 16 · Session Lifecycle](batch-16-session.md) |
| 17 | [Batch 17 · State Ownership](batch-17-state.md) |
| 18 | [Batch 18 · Persistence](batch-18-persistence.md) |
| 19 | [Batch 19 · Memory Boundaries](batch-19-memory.md) |
| 20 | [Batch 20 · Extension Architecture / Lifecycle](batch-20-extensions.md) |
| 21 | [Batch 21 · Skill Lifecycle](batch-21-skills.md) |
| 22 | [Batch 22 · MCP Integration](batch-22-mcp.md) |
| 23 | [Batch 23 · Event / Streaming Flow](batch-23-events.md) |
| 24 | [Batch 24 · Concurrency / Cancellation](batch-24-concurrency.md) |
| 25 | [Batch 25 · CLI / TUI Architecture](batch-25-cli-tui.md) |
| 26 | [Batch 26 · Commands / Keybindings](batch-26-commands.md) |
| 27 | [Batch 27 · Configuration Flow](batch-27-configuration.md) |
| 28 | [Batch 28 · Error Boundaries](batch-28-errors.md) |
| 29 | [Batch 29 · Retry / Recovery](batch-29-retry-recovery.md) |
| 30 | [Batch 30 · Tokens / Usage / Cost](batch-30-tokens-cost.md) |
| 31 | [Batch 31 · Cache / Warming](batch-31-cache.md) |
| 32 | [Batch 32 · Model Switching](batch-32-model-switching.md) |
| 33 | [Batch 33 · Cross-provider Compatibility](batch-33-compatibility.md) |
| 34 | [Batch 34 · Testing Infrastructure](batch-34-testing.md) |
| 35 | [Batch 35 · Evaluation / Benchmark](batch-35-evaluation.md) |
| 36 | [Batch 36 · Core Abstractions](batch-36-core-abstractions.md) |
| 37 | [Batch 37 · Design Patterns](batch-37-design-patterns.md) |
| 38 | [Batch 38 · Technical Debt](batch-38-technical-debt.md) |
| 39 | [Batch 39 · Extension Points](batch-39-extension-points.md) |
| 40 | [Batch 40 · Harness Value](batch-40-core-value.md) |
| 41 | [Batch 41 · Minimal Core](batch-41-minimal-core.md) |
| 42 | [Batch 42 · 30 Core Files](batch-42-top-30-files.md) |
| 43 | [Batch 43 · Reading Path](batch-43-reading-path.md) |
| 44 | [Batch 44 · Critical Call Chains](batch-44-call-chains.md) |
| 45 | [Batch 45 · Pi Development Map](batch-45-development-map.md) |
| 46 | [Batch 46 · Pi Agent Master Architecture Map](batch-46-master-map.md) |
| 47 | [Batch 47 · Core Diagram Atlas](batch-47-diagram-atlas.md) |
| 48 | [Batch 48 · Final Research Report](batch-48-final-study.md) |

## 校验与限制

Archify 3.0.1，quality=showcase；43 图全部通过 validate、deliver、严格 check 和 browser-check，最终记录零错误、零警告。文档本地链接、源码范围与候选/HTML/浏览器记录哈希核对见 [analysis-checks.json](analysis-checks.json)。[diagram-manifest.json](diagram-manifest.json) 列出全部图与当前有效记录。

人工截图视觉审查未执行。Master 保留两处路由交叉建议，部分其他图仍有路由绕行建议，具体见图集。自动通过不等同人工视觉确认。项目测试、真实模型、付费 eval、压缩质量及性能实验未运行；不报告它们的通过率或费用实测。本轮没有修改项目代码、运行构建或提交 Git。
