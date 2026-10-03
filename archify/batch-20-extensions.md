# Batch 20 · Extension Architecture / Lifecycle

## Findings

**Confirmed**：扩展是可信的 TS/JS factory，在宿主进程运行，并非天然隔离插件。Loader 发现/导入/初始化，Runner 绑定核心并派发 hook，ResourceLoader 结合配置、包资源与信任规则。注册可以发生在加载阶段，执行类 action 在 bind 前为抛错 stub。

Factory 失败会 discard 待提交运行时注册和加载期 event-bus 订阅；成功才 commit。reload/会话替换使旧 runtime/ctx stale，并清理追踪订阅。该清理不能撤销扩展顶层任意 OS 副作用。

## Evidence

- [loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)：loadExtensionModule / initializeExtension / createExtensionRuntime。
- [runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)：bindCore 与各 emit。
- [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)：装载、信任、可替换内置扩展。
- [index.ts](../packages/coding-agent/src/extensions/index.ts)：llama.cpp / codemode / tool-search / mcp。

## Key Source Files

[packages/coding-agent/src/core/extensions/loader.ts](../packages/coding-agent/src/core/extensions/loader.ts)、[packages/coding-agent/src/core/extensions/runner.ts](../packages/coding-agent/src/core/extensions/runner.ts)、[packages/coding-agent/src/core/resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)、[packages/coding-agent/src/extensions/index.ts](../packages/coding-agent/src/extensions/index.ts)。

## Key Types / Classes

ExtensionFactory、ExtensionAPI、ExtensionRuntime、ExtensionRunner、ToolDefinition、RegisteredCommand。

## Key Functions

discoverAndLoadExtensions；loadExtensionsCached；initializeExtension；createExtensionAPI；Runner.bindCore；AgentSession.reload。

## Control Flow

发现入口（manifest/index/直接文件）→ jiti 导入 factory → registration → commit → bindCore/UI → session_start → hooks/command/tool → session_shutdown/invalidate。文件模块 factory 可缓存，扩展状态每次初始化。

## Data Flow

注册 tool/command/shortcut/flag/renderer/provider/virtualModel/MCP；context、system prompt、provider payload、headers、tool结果可由不同 hooks 改写。

## State

extension maps、pending provider registrations、factory cache cwd/generation、stale flag、订阅集合。默认目录发现只进入一层，复杂包由 manifest 指定入口。

## Design Decisions

**Inference**：大部分产品策略可在扩展实现，核心保留统一边界；本机权限与信任过滤是必要边界，不能把事件 hooks 当强隔离。

## Unknowns

未运行任意第三方 factory。某个 hook 能否阻断/聚合由对应 emit 方法决定，不能统一假设所有事件返回值可修改核心。

## Archify Diagram

[diagram-20a-extensions](diagrams/diagram-20a-extensions/diagram-20a-extensions.html)、[diagram-20b-extension-life](diagrams/diagram-20b-extension-life/diagram-20b-extension-life.html)

## Follow-up

Batch 21～22 分析 Skill 和 MCP 的实际差异。
