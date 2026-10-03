# Batch 18 · Persistence

## Findings

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

## Evidence

- [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)：append / load / migration。
- [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)：FileSettingsStorage。
- [auth-storage.ts](../packages/coding-agent/src/core/auth-storage.ts)：FileAuthStorageBackend。
- [models-store.ts](../packages/coding-agent/src/core/models-store.ts)：FileModelsStore。
- [config.ts](../packages/coding-agent/src/extensions/mcp/config.ts)：loadMcpConfig。

## Key Source Files

[packages/coding-agent/src/core/session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[packages/coding-agent/src/core/settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)、[packages/coding-agent/src/core/auth-storage.ts](../packages/coding-agent/src/core/auth-storage.ts)、[packages/coding-agent/src/core/models-store.ts](../packages/coding-agent/src/core/models-store.ts)、[packages/coding-agent/src/extensions/mcp/config.ts](../packages/coding-agent/src/extensions/mcp/config.ts)。

## Key Types / Classes

SessionEntry、AuthStorageBackend、CredentialStore、ModelsStore、ModelConfig。

## Key Functions

appendMessage / appendCompaction / appendUsage / appendContextEdit；loadEntriesFromFile；withLockAsync；FileModelsStore.read/write。

## Control Flow

事件终结 → Session append → JSONL；设置修改 → 文件 storage；provider refresh → ModelsStore write → 目录发布。读取时由各 owner 分别装载，没有一个统一事务。

## Data Flow

JSONL 允许重建分支；Auth/Models store 在锁内读取最新 JSON 并合并更新；缓存读取通过文件 revision 与共享 reload 降低重复 I/O。

## State

内存 stores 供 SDK/测试；文件模式包含锁、revision 和缓存快照。Auth reload 失败可保留最后有效快照。

## Design Decisions

**Inference**：可读日志利于导出/调试；不同文件并非原子提交，跨文件一致性不能由锁单独保证。

## Unknowns

未验证断电/并发多进程写 session 的所有结果；未把其他包数据库后端作完整存储审计。

## Archify Diagram

[diagram-18-persistence](diagrams/diagram-18-persistence/diagram-18-persistence.html)

## Follow-up

Batch 19 区分保存历史与长期记忆。
