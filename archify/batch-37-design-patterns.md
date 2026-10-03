# Batch 37 · Design Patterns

## Findings

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

## Evidence

表内源码链接直接支持对应结构；设计动机属于 Inference。

## Key Source Files

表内源码链接直接支持对应结构；设计动机属于 Inference。

## Key Types / Classes

Adapter、Strategy、Registry 是结构判断；不是项目必须使用的框架术语。

## Key Functions

runLoop / initializeExtension / transformMessages / buildSessionProjection / withFileMutationQueue。

## Control Flow

同一请求穿过几种模式，不是模式名字形成新的运行阶段。

## Data Flow

Adapter 变表示，projection 变可见视图，registry 选实现。

## State

缓存代次/有效期与日志树状态分开。

## Design Decisions

**Inference**：必要性见表中具体问题；统一状态机框架或额外DI容器并非源码已采用的设计。

## Unknowns

未做模式重构，不根据模式名推导性能或可靠性。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch 38 只基于具体事实讨论维护风险。
