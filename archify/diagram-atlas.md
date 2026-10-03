# Pi Agent 核心图集

43 张独立 HTML 图，覆盖五种图型；任务指定的 28 项均已映射。全部当前候选、HTML 与 browser-check 哈希绑定通过；人工视觉审查未执行。

## 必需概念映射

| 序号 | 任务要求 | 实际图 |
| --- | --- | --- |
| 1 | Repository Map | [diagram-0-repository](diagrams/diagram-0-repository/diagram-0-repository.html) |
| 2 | Runtime Architecture | [diagram-2b-runtime](diagrams/diagram-2b-runtime/diagram-2b-runtime.html) |
| 3 | Startup Lifecycle | [diagram-2a-startup](diagrams/diagram-2a-startup/diagram-2a-startup.html) |
| 4 | Agent Runtime Workflow | [diagram-3a-runtime](diagrams/diagram-3a-runtime/diagram-3a-runtime.html) |
| 5 | Agent Loop Lifecycle | [diagram-3b-loop](diagrams/diagram-3b-loop/diagram-3b-loop.html) |
| 6 | Agent Loop State Machine | [diagram-3c-state](diagrams/diagram-3c-state/diagram-3c-state.html) |
| 7 | Message Data Flow | [diagram-4-messages](diagrams/diagram-4-messages/diagram-4-messages.html) |
| 8 | Prompt Composition | [diagram-5a-composition](diagrams/diagram-5a-composition/diagram-5a-composition.html) |
| 9 | Context Data Flow | [diagram-6a-context](diagrams/diagram-6a-context/diagram-6a-context.html) |
| 10 | Context Compaction Lifecycle | [diagram-6b-compaction](diagrams/diagram-6b-compaction/diagram-6b-compaction.html) |
| 11 | Model Architecture | [diagram-7-model](diagrams/diagram-7-model/diagram-7-model.html) |
| 12 | Provider Architecture | [diagram-8a-provider](diagrams/diagram-8a-provider/diagram-8a-provider.html) |
| 13 | LLM Request Sequence | [diagram-8b-request](diagrams/diagram-8b-request/diagram-8b-request.html) |
| 14 | Tool Architecture | [diagram-10a-tools](diagrams/diagram-10a-tools/diagram-10a-tools.html) |
| 15 | Tool Call Sequence | [diagram-10b-call](diagrams/diagram-10b-call/diagram-10b-call.html) |
| 16 | File Edit Sequence | [diagram-12b-edit](diagrams/diagram-12b-edit/diagram-12b-edit.html) |
| 17 | Shell Lifecycle | [diagram-13-shell](diagrams/diagram-13-shell/diagram-13-shell.html) |
| 18 | Repository Understanding Workflow | [diagram-14a-search](diagrams/diagram-14a-search/diagram-14a-search.html) |
| 19 | Coding Task Workflow | [diagram-15-coding](diagrams/diagram-15-coding/diagram-15-coding.html) |
| 20 | Session Lifecycle | [diagram-16-session](diagrams/diagram-16-session/diagram-16-session.html) |
| 21 | State Architecture | [diagram-17-state](diagrams/diagram-17-state/diagram-17-state.html) |
| 22 | Extension Architecture | [diagram-20a-extensions](diagrams/diagram-20a-extensions/diagram-20a-extensions.html) |
| 23 | MCP Architecture | [diagram-22a-mcp](diagrams/diagram-22a-mcp/diagram-22a-mcp.html) |
| 24 | Event Flow | [diagram-23-events](diagrams/diagram-23-events/diagram-23-events.html) |
| 25 | Configuration Flow | [diagram-27-config](diagrams/diagram-27-config/diagram-27-config.html) |
| 26 | Retry / Recovery Lifecycle | [diagram-29-retry](diagrams/diagram-29-retry/diagram-29-retry.html) |
| 27 | Compatibility Layer | [diagram-33-compatibility](diagrams/diagram-33-compatibility/diagram-33-compatibility.html) |
| 28 | Master Architecture Map | [diagram-46-master](diagrams/diagram-46-master/diagram-46-master.html) |

运行状态图是 Agent 的 idle/active/abort-requested 谓词映射；Skill/Coding/Lifecycle 图标明可发生路径或步骤映射，不虚构强制规划器/正式状态枚举。

## 完整图与最终记录

| 图 | 类型 / 节点 | 自动检查 | 最终记录 | 浏览器记录 |
| --- | --- | --- | --- | --- |
| [Diagram 0 · Pi 仓库一级模块](diagrams/diagram-0-repository/diagram-0-repository.html) | architecture / 15 | 4 gates pass | [summary](diagrams/diagram-0-repository/review-4/diagram-0-repository.finalize-summary.json) | [browser](diagrams/diagram-0-repository/review-4/diagram-0-repository.browser-check.json) |
| [Diagram 2A · Pi Startup Lifecycle](diagrams/diagram-2a-startup/diagram-2a-startup.html) | workflow / 17 | 4 gates pass | [summary](diagrams/diagram-2a-startup/review-2/diagram-2a-startup.finalize-summary.json) | [browser](diagrams/diagram-2a-startup/review-2/diagram-2a-startup.browser-check.json) |
| [Diagram 2B · 默认 CLI 运行时职责](diagrams/diagram-2b-runtime/diagram-2b-runtime.html) | architecture / 12 | 4 gates pass | [summary](diagrams/diagram-2b-runtime/review-2/diagram-2b-runtime.finalize-summary.json) | [browser](diagrams/diagram-2b-runtime/review-2/diagram-2b-runtime.browser-check.json) |
| [Diagram 3A · Pi Agent Runtime Workflow](diagrams/diagram-3a-runtime/diagram-3a-runtime.html) | workflow / 14 | 4 gates pass | [summary](diagrams/diagram-3a-runtime/diagram-3a-runtime.finalize-summary.json) | [browser](diagrams/diagram-3a-runtime/diagram-3a-runtime.browser-check.json) |
| [Diagram 3B · Pi Agent Loop Lifecycle](diagrams/diagram-3b-loop/diagram-3b-loop.html) | lifecycle / 9 | 4 gates pass | [summary](diagrams/diagram-3b-loop/diagram-3b-loop.finalize-summary.json) | [browser](diagrams/diagram-3b-loop/diagram-3b-loop.browser-check.json) |
| [Diagram 3C · Pi Agent State Machine（运行谓词映射）](diagrams/diagram-3c-state/diagram-3c-state.html) | lifecycle / 3 | 4 gates pass | [summary](diagrams/diagram-3c-state/diagram-3c-state.finalize-summary.json) | [browser](diagrams/diagram-3c-state/diagram-3c-state.browser-check.json) |
| [Diagram 4 · Message Data Flow](diagrams/diagram-4-messages/diagram-4-messages.html) | dataflow / 10 | 4 gates pass | [summary](diagrams/diagram-4-messages/diagram-4-messages.finalize-summary.json) | [browser](diagrams/diagram-4-messages/diagram-4-messages.browser-check.json) |
| [Diagram 5A · Prompt Composition Data Flow](diagrams/diagram-5a-composition/diagram-5a-composition.html) | dataflow / 9 | 4 gates pass | [summary](diagrams/diagram-5a-composition/diagram-5a-composition.finalize-summary.json) | [browser](diagrams/diagram-5a-composition/diagram-5a-composition.browser-check.json) |
| [Diagram 5B · Prompt Build Workflow](diagrams/diagram-5b-build/diagram-5b-build.html) | workflow / 12 | 4 gates pass | [summary](diagrams/diagram-5b-build/diagram-5b-build.finalize-summary.json) | [browser](diagrams/diagram-5b-build/diagram-5b-build.browser-check.json) |
| [6A · Context Data Flow](diagrams/diagram-6a-context/diagram-6a-context.html) | dataflow / 5 | 4 gates pass | [summary](diagrams/diagram-6a-context/diagram-6a-context.finalize-summary.json) | [browser](diagrams/diagram-6a-context/diagram-6a-context.browser-check.json) |
| [6B · Context Compaction Lifecycle](diagrams/diagram-6b-compaction/diagram-6b-compaction.html) | lifecycle / 5 | 4 gates pass | [summary](diagrams/diagram-6b-compaction/diagram-6b-compaction.finalize-summary.json) | [browser](diagrams/diagram-6b-compaction/diagram-6b-compaction.browser-check.json) |
| [6C · Context Composition](diagrams/diagram-6c-composition/diagram-6c-composition.html) | dataflow / 4 | 4 gates pass | [summary](diagrams/diagram-6c-composition/diagram-6c-composition.finalize-summary.json) | [browser](diagrams/diagram-6c-composition/diagram-6c-composition.browser-check.json) |
| [7 · Model Architecture](diagrams/diagram-7-model/diagram-7-model.html) | architecture / 6 | 4 gates pass | [summary](diagrams/diagram-7-model/diagram-7-model.finalize-summary.json) | [browser](diagrams/diagram-7-model/diagram-7-model.browser-check.json) |
| [8A · Provider Architecture](diagrams/diagram-8a-provider/diagram-8a-provider.html) | architecture / 5 | 4 gates pass | [summary](diagrams/diagram-8a-provider/diagram-8a-provider.finalize-summary.json) | [browser](diagrams/diagram-8a-provider/diagram-8a-provider.browser-check.json) |
| [8B · LLM Request Sequence](diagrams/diagram-8b-request/diagram-8b-request.html) | sequence / 4 | 4 gates pass | [summary](diagrams/diagram-8b-request/diagram-8b-request.finalize-summary.json) | [browser](diagrams/diagram-8b-request/diagram-8b-request.browser-check.json) |
| [8C · Provider Message Conversion](diagrams/diagram-8c-conversion/diagram-8c-conversion.html) | dataflow / 4 | 4 gates pass | [summary](diagrams/diagram-8c-conversion/diagram-8c-conversion.finalize-summary.json) | [browser](diagrams/diagram-8c-conversion/diagram-8c-conversion.browser-check.json) |
| [10A · Tool Architecture](diagrams/diagram-10a-tools/diagram-10a-tools.html) | architecture / 6 | 4 gates pass | [summary](diagrams/diagram-10a-tools/diagram-10a-tools.finalize-summary.json) | [browser](diagrams/diagram-10a-tools/diagram-10a-tools.browser-check.json) |
| [10B · Tool Call Sequence](diagrams/diagram-10b-call/diagram-10b-call.html) | sequence / 4 | 4 gates pass | [summary](diagrams/diagram-10b-call/diagram-10b-call.finalize-summary.json) | [browser](diagrams/diagram-10b-call/diagram-10b-call.browser-check.json) |
| [10C · Tool Result Projection](diagrams/diagram-10c-result/diagram-10c-result.html) | dataflow / 5 | 4 gates pass | [summary](diagrams/diagram-10c-result/diagram-10c-result.finalize-summary.json) | [browser](diagrams/diagram-10c-result/diagram-10c-result.browser-check.json) |
| [12A · File Read Sequence](diagrams/diagram-12a-read/diagram-12a-read.html) | sequence / 3 | 4 gates pass | [summary](diagrams/diagram-12a-read/diagram-12a-read.finalize-summary.json) | [browser](diagrams/diagram-12a-read/diagram-12a-read.browser-check.json) |
| [12B · File Edit Sequence](diagrams/diagram-12b-edit/diagram-12b-edit.html) | sequence / 4 | 4 gates pass | [summary](diagrams/diagram-12b-edit/diagram-12b-edit.finalize-summary.json) | [browser](diagrams/diagram-12b-edit/diagram-12b-edit.browser-check.json) |
| [12C · File Editing Data Flow](diagrams/diagram-12c-data/diagram-12c-data.html) | dataflow / 5 | 4 gates pass | [summary](diagrams/diagram-12c-data/diagram-12c-data.finalize-summary.json) | [browser](diagrams/diagram-12c-data/diagram-12c-data.browser-check.json) |
| [13 · Shell Lifecycle](diagrams/diagram-13-shell/diagram-13-shell.html) | lifecycle / 5 | 4 gates pass | [summary](diagrams/diagram-13-shell/diagram-13-shell.finalize-summary.json) | [browser](diagrams/diagram-13-shell/diagram-13-shell.browser-check.json) |
| [14A · Repository Understanding Workflow](diagrams/diagram-14a-search/diagram-14a-search.html) | workflow / 5 | 4 gates pass | [summary](diagrams/diagram-14a-search/diagram-14a-search.finalize-summary.json) | [browser](diagrams/diagram-14a-search/diagram-14a-search.browser-check.json) |
| [14B · Repository Evidence to Context](diagrams/diagram-14b-context/diagram-14b-context.html) | dataflow / 5 | 4 gates pass | [summary](diagrams/diagram-14b-context/diagram-14b-context.finalize-summary.json) | [browser](diagrams/diagram-14b-context/diagram-14b-context.browser-check.json) |
| [15 · Coding Task Workflow](diagrams/diagram-15-coding/diagram-15-coding.html) | workflow / 6 | 4 gates pass | [summary](diagrams/diagram-15-coding/diagram-15-coding.finalize-summary.json) | [browser](diagrams/diagram-15-coding/diagram-15-coding.browser-check.json) |
| [16 · Session Lifecycle](diagrams/diagram-16-session/diagram-16-session.html) | lifecycle / 5 | 4 gates pass | [summary](diagrams/diagram-16-session/diagram-16-session.finalize-summary.json) | [browser](diagrams/diagram-16-session/diagram-16-session.browser-check.json) |
| [17 · State Architecture](diagrams/diagram-17-state/diagram-17-state.html) | architecture / 6 | 4 gates pass | [summary](diagrams/diagram-17-state/diagram-17-state.finalize-summary.json) | [browser](diagrams/diagram-17-state/diagram-17-state.browser-check.json) |
| [18 · Persistence Flow](diagrams/diagram-18-persistence/diagram-18-persistence.html) | dataflow / 4 | 4 gates pass | [summary](diagrams/diagram-18-persistence/diagram-18-persistence.finalize-summary.json) | [browser](diagrams/diagram-18-persistence/diagram-18-persistence.browser-check.json) |
| [20A · Extension Architecture](diagrams/diagram-20a-extensions/diagram-20a-extensions.html) | architecture / 5 | 4 gates pass | [summary](diagrams/diagram-20a-extensions/diagram-20a-extensions.finalize-summary.json) | [browser](diagrams/diagram-20a-extensions/diagram-20a-extensions.browser-check.json) |
| [20B · Extension Lifecycle](diagrams/diagram-20b-extension-life/diagram-20b-extension-life.html) | lifecycle / 5 | 4 gates pass | [summary](diagrams/diagram-20b-extension-life/diagram-20b-extension-life.finalize-summary.json) | [browser](diagrams/diagram-20b-extension-life/diagram-20b-extension-life.browser-check.json) |
| [21 · Skill Activation Lifecycle](diagrams/diagram-21-skill/diagram-21-skill.html) | lifecycle / 4 | 4 gates pass | [summary](diagrams/diagram-21-skill/diagram-21-skill.finalize-summary.json) | [browser](diagrams/diagram-21-skill/diagram-21-skill.browser-check.json) |
| [22A · MCP Architecture](diagrams/diagram-22a-mcp/diagram-22a-mcp.html) | architecture / 6 | 4 gates pass | [summary](diagrams/diagram-22a-mcp/diagram-22a-mcp.finalize-summary.json) | [browser](diagrams/diagram-22a-mcp/diagram-22a-mcp.browser-check.json) |
| [22B · MCP Tool Call Sequence](diagrams/diagram-22b-mcp-call/diagram-22b-mcp-call.html) | sequence / 5 | 4 gates pass | [summary](diagrams/diagram-22b-mcp-call/diagram-22b-mcp-call.finalize-summary.json) | [browser](diagrams/diagram-22b-mcp-call/diagram-22b-mcp-call.browser-check.json) |
| [23 · Event Flow](diagrams/diagram-23-events/diagram-23-events.html) | dataflow / 5 | 4 gates pass | [summary](diagrams/diagram-23-events/diagram-23-events.finalize-summary.json) | [browser](diagrams/diagram-23-events/diagram-23-events.browser-check.json) |
| [24 · Concurrency Architecture](diagrams/diagram-24-concurrency/diagram-24-concurrency.html) | architecture / 6 | 4 gates pass | [summary](diagrams/diagram-24-concurrency/review-2/diagram-24-concurrency.finalize-summary.json) | [browser](diagrams/diagram-24-concurrency/review-2/diagram-24-concurrency.browser-check.json) |
| [25A · CLI / TUI Architecture](diagrams/diagram-25a-ui/diagram-25a-ui.html) | architecture / 5 | 4 gates pass | [summary](diagrams/diagram-25a-ui/diagram-25a-ui.finalize-summary.json) | [browser](diagrams/diagram-25a-ui/diagram-25a-ui.browser-check.json) |
| [25B · User Input to UI Output](diagrams/diagram-25b-input/diagram-25b-input.html) | sequence / 4 | 4 gates pass | [summary](diagrams/diagram-25b-input/diagram-25b-input.finalize-summary.json) | [browser](diagrams/diagram-25b-input/diagram-25b-input.browser-check.json) |
| [27 · Configuration Flow](diagrams/diagram-27-config/diagram-27-config.html) | dataflow / 5 | 4 gates pass | [summary](diagrams/diagram-27-config/diagram-27-config.finalize-summary.json) | [browser](diagrams/diagram-27-config/diagram-27-config.browser-check.json) |
| [29 · Retry / Recovery Lifecycle](diagrams/diagram-29-retry/diagram-29-retry.html) | lifecycle / 5 | 4 gates pass | [summary](diagrams/diagram-29-retry/diagram-29-retry.finalize-summary.json) | [browser](diagrams/diagram-29-retry/diagram-29-retry.browser-check.json) |
| [33 · Cross-provider Compatibility Layer](diagrams/diagram-33-compatibility/diagram-33-compatibility.html) | dataflow / 5 | 4 gates pass | [summary](diagrams/diagram-33-compatibility/diagram-33-compatibility.finalize-summary.json) | [browser](diagrams/diagram-33-compatibility/diagram-33-compatibility.browser-check.json) |
| [34 · Testing Architecture](diagrams/diagram-34-tests/diagram-34-tests.html) | architecture / 5 | 4 gates pass | [summary](diagrams/diagram-34-tests/diagram-34-tests.finalize-summary.json) | [browser](diagrams/diagram-34-tests/diagram-34-tests.browser-check.json) |
| [Pi Agent Master Architecture Map](diagrams/diagram-46-master/diagram-46-master.html) | architecture / 20 | 4 gates pass | [summary](diagrams/diagram-46-master/review-3/diagram-46-master.finalize-summary.json) | [browser](diagrams/diagram-46-master/review-3/diagram-46-master.browser-check.json) |

## 复核边界

- Diagram 24 只调整节点位置后复验，原有两处交叉提示已消除。
- Master 的一次位置复核因桌面可读字号失败，恢复原布局后重新完整通过。保留 review-2 失败记录与 review-3 最终记录，没有降低验收阈值。
- 以下建议没有被自动 gate 判为错误；保留供人工路由评审，不能据此宣称视觉质量已人工确认。

| 图 | 路由建议 |
| --- | --- |
| [diagram-2a-startup](diagrams/diagram-2a-startup/diagram-2a-startup.html) | {"routesOverSuggestedBends":2,"routesOverSuggestedStretch":1} |
| [diagram-3a-runtime](diagrams/diagram-3a-runtime/diagram-3a-runtime.html) | {"routesOverSuggestedStretch":2} |
| [diagram-3b-loop](diagrams/diagram-3b-loop/diagram-3b-loop.html) | {"resolvedCrossovers":1} |
| [diagram-10a-tools](diagrams/diagram-10a-tools/diagram-10a-tools.html) | {}；inspect-leading-space |
| [diagram-29-retry](diagrams/diagram-29-retry/diagram-29-retry.html) | {"resolvedCrossovers":2} |
| [diagram-46-master](diagrams/diagram-46-master/diagram-46-master.html) | {"resolvedCrossovers":2,"routesOverSuggestedBends":3} |

每图 candidate.json 保留来源、语义与布局；旧失败 receipt 不是当前交付状态。[analysis-checks.json](analysis-checks.json) 选择与当前 candidate/HTML 哈希相符的最新 passing 记录。[基线推进说明](baseline-drift.md) 区分已研究快照与当前源码。
