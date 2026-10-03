# Batch 16 · Session Lifecycle

## Findings

**Confirmed**：SessionManager 保存的是带 id/parentId 的追加日志树；active leaf 指向当前分支。AgentSession 提供编码行为，AgentSessionRuntime 负责替换会话并重绑定 Host。恢复读取文件并重建投影，不恢复之前正在执行的 OS 进程或模型网络流。

branch 在同一日志树内改变 leaf；fork 生成新 session id / 文件，携带选定祖先路径和 parentSession 关系。这与复制整个树不同。

## Evidence

- [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)：buildSessionProjection / branch / forkFrom。
- [agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)：replace session 与绑定。
- [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)：navigateTree / dispose。

## Key Source Files

[packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[packages/coding-agent/src/core/agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)、[packages/coding-agent/src/core/agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)。

## Key Types / Classes

SessionHeader、SessionEntry、SessionProjection、SessionManager、AgentSessionRuntime。

## Key Functions

SessionManager.create/open/continueRecent/inMemory/forkFrom；branch；AgentSession.navigateTree；Runtime 会话替换方法。

## Control Flow

创建或打开日志 → 获取 leaf 分支 → buildSessionProjection → 初始化 Agent → Host bind → prompt → 追加 entries；切换前 shutdown/dispose，切换后重新 bind/start。

## Data Flow

header 记录版本/id/cwd/timestamp/parentSession；entries 包括 message、model/thinking change、compaction、branch summary、custom、usage、context edit、label。投影不是文件中所有消息平铺。

## State

fileEntries、byId、leafId、sessionId、sessionFile 与运行中 Agent/Session 控制器。setup-only entries 可暂留内存，真实对话开始后才落盘。

## Design Decisions

**Inference**：追加树保留替代历史，投影把“可审计记录”与“当前模型可见内容”分开。持久化会话不等于 durable execution checkpoint。

## Unknowns

未实际执行 fork/resume 或损坏文件恢复；中断中的外部工具是否已产生副作用不能由 resume 消除。

## Archify Diagram

[diagram-16-session](diagrams/diagram-16-session/diagram-16-session.html)

## Follow-up

Batch 17～19 拆分状态、持久化与记忆。
