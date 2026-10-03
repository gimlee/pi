# Batch 6：Context Management

## Findings

**Confirmed**：原始 JSONL 历史、当前 branch、模型请求 projection 是三层数据。`buildSessionPath()` 从 leaf 沿 parentId 回溯；`buildContextEntries()` 选择最近压缩边界；`buildSessionProjection()` 应用 context_edit 的最后一次替换/省略。其他分支、纯 custom 状态、label/model/thinking 等元数据不作为对话正文全部发送。

**Confirmed**：压缩不是删除日志。新增 CompactionEntry，保存 summary、firstKeptEntryId、tokensBefore、usage/details，以及边界的完整 systemMessage；投影生成系统状态 + 摘要 + 保留尾部 + 边界后的消息。Provider 仍可进一步折叠系统更新和清理跨模型内容。

**Confirmed**：token 估算优先使用有效 assistant usage，再补之后的消息估算；context edit/新 compaction 会使旧 usage 失效。没有逐模型 tokenizer 的精确客户端计数，chars/4 与图片 4800 字符只是启发式，不能保证所有语言或图像都保守高估。

## Evidence

| 已读源码 | 定位与直接证据 |
|---|---|
| [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) | buildSessionPath、buildContextEntries、projectContextEntry、buildSessionProjection；439～581；appendCompaction 保存系统 replay |
| [compaction.ts](../packages/coding-agent/src/core/compaction/compaction.ts) | 全文件；calculateContextTokens、estimateProjectedContextTokens、shouldCompact、prepareCompaction、compact |
| [compaction/utils.ts](../packages/coding-agent/src/core/compaction/utils.ts) | 全文件；serializeConversation、文件操作统计、摘要时单个工具结果 2000 字符上限 |
| [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) | prepareRequest/prepareNextTurn；compact 2717 起；_checkCompaction 2900 起；_runAutoCompaction 3050 附近；getContextUsage |
| [truncate.ts](../packages/coding-agent/src/core/tools/truncate.ts)、[output-accumulator.ts](../packages/coding-agent/src/core/tools/output-accumulator.ts) | 全文件；头/尾/中间截断，流式累计与落盘 |
| [read.ts](../packages/coding-agent/src/core/tools/read.ts)、[bash.ts](../packages/coding-agent/src/core/tools/bash.ts)、[grep.ts](../packages/coding-agent/src/core/tools/grep.ts) | 全文件；具体工具的输出限制与继续读取提示 |

分析基线见 [README](README.md)。Confirmed 是静态实现证据；没有真实模型压缩质量实验。

## Key Source Files

SessionManager 决定哪个历史片段参与请求；Compaction 的纯函数决定估算、切点和摘要内容；AgentSession 决定何时压缩、鉴权、扩展拦截、落盘与恢复；Tool 的输出限制在结果进入历史之前发生。完整转换路径另见 [Batch 4](batch-4-message-model.md) 与 [Batch 5](batch-5-prompt-system.md)。

## Key Types / Classes

`SessionEntry` / `ProjectedSessionEntry` 保留原始条目的来源；`ContextEditEntry` 的 null 表示省略，非空值只替换允许 role 的 content；`SessionProjection` 保存 messages 与 replay 的选型；`CompactionPreparation` 保存旧历史、split-turn 前缀、上一摘要、切点与文件操作；`CompactionResult` 是摘要结果，SessionManager 再增加树节点信息。

## Key Functions

`SessionManager.buildSessionProjection()` → `buildSessionPath()` / `buildContextEntries()` → `projectContextEntry()`；`AgentSession._installAgentRequestProjection() 安装的 prepareRequest hook` 每请求接入投影；`estimateProjectedContextTokens()` → `shouldCompact()`；`prepareCompaction()` → `findProjectedCutPoint()` → `compact()` → `generateSummaryWithUsage()` / `generateTurnPrefixSummary()` → `completeSummarization()`。

## Control Flow

### 触发、执行和恢复

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

## Data Flow

例如原始分支为 `system,user1,assistant1,toolResult1,user2,assistant2`。压缩保存 C，firstKept 指 user2；后续 projection 为 `C.systemMessage,compactionSummary(C),user2,assistant2`，原始 user1/工具结果仍留 JSONL。另一个分支不被混入；之后 null context_edit 可再省略 assistant2 的模型贡献。

| 大型输出 | 进入模型前的处理 |
|---|---|
| read | 默认首 2000 行或 50 KiB，offset/limit 分页；单行过大给 shell 处理提示；当前本地实现仍先读整文件 Buffer，输出限制不等于读取内存限制 |
| bash / PowerShell 共用 shell backend | 模型 content 保留尾部 2000 行或 50 KiB；超过阈值保存完整临时日志，提示路径；structuredContent 可取至 1 MiB 的头尾，但不自动作为 ToolResultMessage 正文 |
| grep | 默认 100 个匹配，单行 500 字符，再按 50 KiB 截断；到匹配上限停止子进程，建议缩小查询或提高 limit |
| 摘要输入 | serializeConversation 再把每个工具结果文字裁到 2000 字符；图片不还原成原始附件；可见 thinking 会被序列化 |

这是三种不同机制：工具输出截断、历史投影编辑、模型摘要压缩。没有默认“每次请求把所有旧工具结果统一剪掉”的独立通用算法。

## State

原始 entry/tree 与 leaf 保存在 SessionManager；projection 是重建的请求视图。压缩信号分别为手工与自动 controller；overflowRecoveryAttempted 限制用户活动中的一次恢复；系统 checkpoint、summary 和 usage 持久化。估算遇到后来的 context_edit/compaction 时不继续信任更早的 usage，避免压缩完立即再次误触发。

## Design Decisions

- **Inference**：追加式 edit/compaction 保留调查历史，同时允许模型只接收必要视图；不是删除用户会话证据。
- **Inference**：切点保护调用/结果邻接比仅按最后 N 条消息裁剪更适合工具对话。
- **Confirmed**：summary 文件列表来自 read/write/edit 调用和 nestedCalls，并排除已修改文件的只读分类。**Inference**：提供继续任务的定位线索；没有重新读取、校验文件或生成仓库索引。
- Harness 管触发、切点、投影、重试和落盘；Model 生成摘要；摘要能否保留任务含义是 Hybrid，源码不能证明其语义无损。

## Unknowns

没有测量 chars/4 的误差、摘要信息损失或实际费用；未运行取消/故障注入。Provider-specific context 编码可能继续改变最终长度。扩展自定义摘要的语义与预算不由默认 compact 保证。

## Archify Diagram

- [Diagram 6A：Pi Context Data Flow](diagrams/diagram-6a-context/diagram-6a-context.html) · dataflow。
- [Diagram 6B：Context Compaction Lifecycle](diagrams/diagram-6b-compaction/diagram-6b-compaction.html) · lifecycle。
- [Diagram 6C：Model Context Composition](diagrams/diagram-6c-composition/diagram-6c-composition.html) · dataflow。

交付与校验状态以最终 [总索引](README.md) 和检查记录为准；链接不表示未经校验的图已经通过。

## Follow-up

Batch 7 分析 Model Abstraction，Batch 8 分析 Provider。后续 Session/Persistence、Tool、Retry 批次分别补分支/恢复、工具限流与多层重试；压缩质量实验属于 Unknown 的运行验证。
