# Batch 0：Repository Reconnaissance

## Findings

**Confirmed**：这是 npm workspace 的 TypeScript/ESM monorepo，当前实际存在 13 个一级包。默认产品入口在 `coding-agent`，通用运行循环在 `agent`，模型协议与 Provider 在 `ai`，终端渲染在 `tui`。新增的 `durable`、`chord`、`client`、`server`、`protocol` 必须纳入仓库地图，但不能因此假设默认 CLI 已改用持久化 Task Runtime 或远程服务器。

**Confirmed**：默认 CLI 的三类 Host 是交互模式、一次性 print（text/JSON）和长期 RPC。SDK 可绕过 CLI 组装会话。开发入口额外分发 client/server，发布配置排除了 experimental 目录。

### Pi Repository Overview

| 位置 | 源码/配置确认的职责 | 等级 |
|---|---|---|
| 根 `package.json`、`tsconfig*.json` | workspace、统一检查、源代码类型配置 | Confirmed |
| `packages/` | 13 个包，见下表；没有独立 `apps/` 一级目录 | Confirmed |
| `scripts/` | 模型生成、发布/安装产物等仓库脚本入口；本次只定位，不执行 | Confirmed（存在性） |
| `.pi/` | 本仓库的 pi 资源与开发技能；不是运行时类名 | Confirmed（存在性） |
| `packages/coding-agent/examples/` | SDK 与扩展样例，其中五个 extension 样例也属于根 workspace | Confirmed |
| `packages/*/test`、`packages/evals/` | 单元、集成、e2e 与评估配置分布于各包 | Confirmed（目录与脚本），质量未知 |
| `archify/` | 本任务的报告、图和检查记录 | Confirmed |

### Technology Stack

| 项目 | 本快照中的事实 / 证据 |
|---|---|
| Runtime | 包声明 Node `>=22.19.0`；ESM；源码开发入口使用 Node strip-types 与 source resolver，另有 Bun 单文件发布路径。见根与 `coding-agent/package.json`、`pi-test.sh`。 |
| 语言 | TypeScript，根开发依赖 `7.0.2`；ES2024、严格类型检查、仅可擦除 TS 语法约束。见 `tsconfig.base.json`、`tsconfig.json`。 |
| Workspace | `packages/*` 加五个扩展示例 workspace。根包版本 `0.0.3`，13 个包版本均为 `1.0.0`；`evals` 为 private。 |
| 接口与验证 | `typebox` 提供工具/数据 schema；模型流与 Agent 事件是 TS 类型化 API。验证语义需在 Tool Batch 深查。 |
| 测试与检查 | 多包使用 Vitest `4.1.11`，TUI 使用 `node:test`；Biome 与 TypeScript 等由根 `check` 驱动。`check` 含写入操作，文档分析不执行。 |
| CLI / UI | Node CLI，`pi-tui` 提供终端输入、组件与主屏/备用屏渲染。不是浏览器 SPA。 |
| 持久化 | 默认编码会话由 `SessionManager` 使用 JSONL/内存；独立 `durable` 包另导出 memory/jsonl/sqlite 存储入口。 |
| 代码执行 | 内置 shell/file 工具；codemode 依赖 QuickJS WASM。不能把脚本沙箱边界推广为所有工具的隔离边界。 |

以上为 **Confirmed**，来自配置或导出/调用点；未实测当前机器所有运行环境。

### Package / Module Structure

下表的依赖箭头表示 manifest 的直接依赖，不自动表示启动必经路径。

| 包目录 / npm 名称 | 验证到的公共能力 | 直接内部依赖 / 启动边界 |
|---|---|---|
| `agent` / `pi-agent-core` | `Agent`、`runAgentLoop`、工具执行、队列与事件 | `ai`；默认编码运行使用 |
| `ai` / `pi-ai` | 消息/模型协议、`Models`、Provider 工厂、API 与流 | `telemetry`；core 入口与 compat 入口需区分 |
| `coding-agent` / `pi-coding-agent` | CLI、SDK、AgentSession、资源/工具/配置/会话与三种 Host | `agent`、`ai`、`tui`、`chord`、`codemode`、`mcp`；client/protocol/server 在 devDependencies |
| `tui` / `pi-tui` | `ProcessTerminal`、`TuiMainScreen`、`TuiAltScreen`、组件与文本布局 | 无内部运行依赖；由交互 Host 使用 |
| `mcp` / `pi-mcp` | `McpClient`、Stdio/Streamable HTTP Transport、JSON-RPC 类型 | 无内部直接运行依赖；编码端内置 MCP extension 接入 |
| `codemode` / `pi-codemode` | `CodemodeSandbox`、声明生成与 QuickJS WASM 加载 | 无内部直接运行依赖；内置 extension 装配 |
| `chord` / `chord` | Facet/Service（应用组合与服务接口）、远程服务、复制状态 | 无内部直接运行依赖；不能与 Agent Loop 混为一层 |
| `protocol` / `pi-protocol` | CBOR、framing、请求/响应/握手与 Session target | `chord`；远程通道协议 |
| `client` / `pi-client` | `Client`、transport、连接与订阅 API | `chord`、`protocol`；开发 client 路径 |
| `server` / `pi-server` | server/listener/error/type 公共入口 | `chord`、`protocol`；开发 server 路径 |
| `durable` / `pi-durable` | 独立 `Harness`、`AgentDoc`、Generation/Tool/Compaction Task 与 Storage | `chord`、`ai`；默认 CLI 创建链未接入此 Harness |
| `telemetry` / `pi-telemetry` | typed spans、显式 parent context、内存记录器 | 无运行依赖；不据此声称默认向外部平台上传 |
| `evals` / `pi-evals` | host/docs eval、报告与隔离运行配置入口 | dev 依赖 `ai`、`coding-agent`；不属于生产对话循环 |

全名统一使用 `@earendil-works/` 前缀。全部包表项为 **Confirmed**；`durable`、远程通道和 eval 的内部算法不是本次深度审计范围。

### Entry Points

| 入口 | 实际调用 / 导出 | 意义 |
|---|---|---|
| 发布 CLI：`coding-agent/package.json` 的 `bin.pi` | `dist/bundle/cli.js`，源入口 [cli.ts](../packages/coding-agent/src/cli.ts) | `setupCli(); main(process.argv.slice(2))` |
| 应用组装：[main.ts](../packages/coding-agent/src/main.ts) | `main()`，573～1000；`resolveAppMode()`，112～123 | 配置、会话目录、服务、诊断和 Host 选择 |
| bootstrap：[cli/setup.ts](../packages/coding-agent/src/cli/setup.ts) | `setupCli()` | 进程名、环境标志、warnings 与 HTTP dispatcher |
| 交互：[interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts) | `InteractiveMode.init()` / `run()` / `getUserInput()` | UI 初始化、扩展绑定、用户输入循环 |
| 非交互：[print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts) | `runPrintMode()` | 按次序发送初始/追加 prompt，输出，dispose，返回退出码 |
| RPC：[rpc-mode.ts](../packages/coding-agent/src/modes/rpc/rpc-mode.ts) | `runRpcMode()` | JSONL stdin 命令、事件/响应 stdout、持续存活 |
| 独立 RPC 入口：[rpc-entry.ts](../packages/coding-agent/src/rpc-entry.ts) | 环境/HTTP 设置后 `main(["--mode", "rpc", ...])` | 将运行模式固定为 RPC |
| 编码 SDK：[index.ts](../packages/coding-agent/src/index.ts)、[core/sdk.ts](../packages/coding-agent/src/core/sdk.ts) | `createAgentSession()`、services/runtime 工厂公开导出 | 调用者负责 Host I/O 与扩展绑定；不需要进入 `main()` |
| 通用 Agent：[agent/src/index.ts](../packages/agent/src/index.ts) | `Agent`、loop、类型与工具运行入口 | 可注入 `streamFn`；无编码会话必需依赖 |
| 模型库：[ai/src/index.ts](../packages/ai/src/index.ts) | Models、Provider 工厂、消息和 API 类型 | core 入口不会自动注册 compat 全局 Provider |
| TUI 库：[tui/src/index.ts](../packages/tui/src/index.ts) | 渲染器/终端/组件导出 | 它是库入口；编码产品 TUI 入口在 `InteractiveMode` |
| 开发 CLI：[experimental/cli.ts](../packages/coding-agent/src/experimental/cli.ts) | experimental command 命中后分发，否则回到 `main()` | `pi-test.sh` 使用此路径；发布 CLI 不导入此入口 |

**Confirmed**：`--mode json` 走 print Host 的事件流；`--mode rpc` 走长期命令协议。若不是 RPC/JSON，`-p` 或 stdin/stdout 任一个非 TTY 则 print，否则 interactive。`--mode text` 本身不保证非交互。

### Core Module Candidates

| 概念 | 实际源码与符号 | 当前已确认边界 |
|---|---|---|
| Runtime owner | `core/agent-session-runtime.ts`：`AgentSessionRuntime` | 当前 session/services 替换与 Host 重绑定 |
| 编码 Harness | `core/agent-session.ts`：`AgentSession` | prompt、上下文/工具边界、恢复、扩展、事件与持久化协调 |
| Agent / Loop | `agent/src/agent.ts`、`agent-loop.ts`：`Agent`、`runLoop` | 单次运行、tool call/result、steer/follow-up |
| Session / Storage | `core/session-manager.ts`：`SessionManager`、`SessionEntry` | JSONL 追加树、leaf、上下文投影与内存模式 |
| Message / Context | `agent/src/types.ts`；`core/messages.ts`；`SessionProjection` | 通用与编码扩展消息、模型可见投影 |
| Prompt | `AgentSession._rebuildSystemPrompt` / `_preparePromptAndToolLoadout`，`core/system-prompt.ts` | 资源输入、结构化提示段、当前工具声明 |
| Model / Provider | `core/model-runtime.ts`：`ModelRuntime`；`ai/src/providers/*` | 模型选择/鉴权/路由、Provider 与 API 适配器分发 |
| Tool | `core/tools/index.ts`：`createAllToolDefinitions`；`_refreshToolRegistry` | 8 种内置定义、可执行包装、allow/deny/activation |
| Extension / Event | `DefaultResourceLoader`、`ExtensionRunner`、`bindExtensions` | 加载/注册阶段与模式绑定/启动事件阶段分离 |
| Workspace | CLI/Session 的 `cwd`，Settings/ResourceLoader 的 project trust | 当前目录决定工具与资源；尚未确认统一 Workspace 类 |
| MCP / codemode | `extensions/mcp/index.ts`、`extensions/codemode/index.ts` | CLI 内置扩展名单已确认；沙箱和连接细节待深查 |

以上候选职责为 **Confirmed**；将它们视为最优二次开发切入点是 **Inference**。

### Important External Dependencies

依赖版本来自本快照 manifest，不是最新版本推荐。

| 依赖 | 版本 / 位置 | 在架构中的用途 |
|---|---|---|
| `@anthropic-ai/sdk`、`openai`、`@google/genai`、AWS Bedrock SDK | `0.129.0`、`7.19.0`、`2.21.0`、`3.1127.0`；ai | Provider API 的外部客户端依赖；具体请求转换待 Provider Batch |
| `typebox` | `1.3.27`；agent/ai/protocol 等 | schema 与 TS 类型支撑 |
| `quickjs-wasi` | `3.6.2`；codemode | 脚本 VM / WASM |
| `jiti` | `2.7.0`；coding-agent | TypeScript 扩展加载依赖；loader 实现待 Extension Batch |
| `proper-lockfile` | `4.1.2`；coding-agent | `FileSettingsStorage` 的锁定读写 |
| `undici` | `8.10.2`；coding-agent | HTTP dispatcher 依赖；本次不更新依赖 |
| `cross-spawn` | `7.0.6`；mcp | 子进程 Transport 依赖；不是官方 MCP SDK 包 |
| `@silvia-odwyer/photon-node` | `0.3.4`；coding-agent | 图片处理依赖 |
| `get-east-asian-width`、`marked` | `1.6.0`、`18.0.11`；tui | 终端字符宽度与 Markdown |
| `esbuild` | `0.28.2`；chord | 直接 dependencies；包导出 bundler 入口，具体运行用途待查 |
| Vitest / vitest-evals / autoevals | `4.1.11` / `0.15.0` / `0.3.0` | 测试和评估工具；未在本任务运行 |

manifest 的依赖存在性为 **Confirmed**；未读 adapter 代码的具体用法仅作为后续调查线索。

### Investigation Plan

1. Batch 1：从 SDK、AgentSession 和实际 loop 定位产品核心；解释模型能力与 Harness 能力的分工。
2. Batch 2：按默认 CLI 的真实先后顺序追踪；建立对象创建表、状态所有权和 Host 边界。
3. Batch 3：审计 `runLoop` 的完整继续/停止矩阵、tool 预检/执行/完成顺序与队列时机。优先读 `agent-loop.test.ts` 和 specific tests。
4. 后续 Message / Context / Compaction：从原始日志→投影→transform→Provider 请求追踪，验证压缩、分支和系统消息声明。
5. 后续 Model / Provider：逐个检查 API 类型和真实请求序列化；比较 thinking、图片、缓存、transport 与虚拟模型。
6. 后续 Tool / Repository Understanding：完整阅读 read/edit/write/grep/find/shell 实现和测试，再判断 AST/LSP/Tree-sitter/Repo Map 是否在实际主路径。
7. 后续 Extension / Skill / MCP：加载、信任、注册、绑定、异步连接和 shutdown；不要把不同机制统一称为插件。
8. 最后审计 durable/远程/evals 与默认路径的关系，形成 Minimal Core 和 development map。

## Evidence

| 证据 | 可复核来源 |
|---|---|
| workspace、版本、依赖、检查方式 | [根 package.json](../package.json)、全部 `packages/*/package.json`、[tsconfig.base.json](../tsconfig.base.json)、[tsconfig.json](../tsconfig.json) |
| 发布/源码入口与 experimental 排除 | [coding-agent/package.json](../packages/coding-agent/package.json) 9～33；[pi-test.sh](../pi-test.sh)；`experimental/commands.ts` 83～98 |
| 模式判定和分发 | `main.ts` 112～123、949～998 |
| SDK 和核心对象创建 | `core/sdk.ts` 175～194、387～455 |
| 独立 durable 能力 | [durable/src/index.ts](../packages/durable/src/index.ts)、[durable/package.json](../packages/durable/package.json)；未作为默认创建链证据 |
| 测试层次 | [test.sh](../test.sh)、各包 test 脚本与已定位测试文件；只是配置/存在性证据 |

## Key Source Files

下一轮最短阅读集合：[main.ts](../packages/coding-agent/src/main.ts)、[sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts)、[agent-session-services.ts](../packages/coding-agent/src/core/agent-session-services.ts)、[agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)、[agent.ts](../packages/agent/src/agent.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[session-manager.ts](../packages/coding-agent/src/core/session-manager.ts)、[resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)。

## Key Types / Classes

`Args`、`CreateAgentSessionOptions`、`AgentSessionServices`、`AgentSessionRuntime`、`AgentSession`、`Agent`、`AgentState`、`AgentLoopConfig`、`AgentMessage`、`Model`、`Provider`、`AgentTool`、`ToolDefinition`、`ExtensionRunner`、`SessionManager`、`SessionProjection`。`ToolDefinition` 是注册定义，`AgentTool` 是通用循环执行的工具契约；二者不能只按名字视为同一个对象。

## Key Functions

`setupCli` → `main` → `createSessionManager` → `createAgentSessionRuntime` → `createAgentSessionServices` / `createAgentSessionFromServices` → `createAgentSession`；运行入口为 `InteractiveMode.run` / `runPrintMode` / `runRpcMode`，下层为 `Agent.prompt` → `runAgentLoop` → `runLoop`。

## Control Flow

**Confirmed**：默认 CLI 自上而下选择 Host 和服务，Host 调用同一编码 Session；Session 安装通用 Agent 的 hooks 并驱动它；底层 loop 调用注入 `streamFn`，执行工具后把结果放回下一次请求。包依赖图与调用图不同，`durable` 和 remote transport 不应串入这条默认链。

## Data Flow

**Confirmed**：CLI/Host 输入 → `session.prompt` → 编码消息、提示和当前工具装载 → 分支 Context 投影 → 通用消息到模型消息转换 → ModelRuntime / Provider；流和工具事件返回 AgentSession → Host，已完成消息进入 SessionManager。完整转换算法留后续 Batch。

## State

当前目录与 project trust 属于服务/资源配置；日志树和 leaf 属于 SessionManager；messages/model/tools/queues/activeRun 属于 Agent；工具注册/扩展/恢复协调属于 AgentSession；终端组件属于 InteractiveMode。这里只确定边界，不声称所有状态同步已审计。

## Design Decisions

**Inference**：把通用 loop 与编码 Harness 拆开，使其他应用可复用循环；用 Host 适配终端/一次性命令/RPC，使编码能力跨 I/O 入口复用；独立服务构造能在目标 cwd 确定后重新装配资源。源码支持这些结果，作者的历史动机未调查。

## Unknowns

- 各包代码质量、测试覆盖率与真实平台运行结果。
- durable 与远程路径内部的一致性、恢复和并发保证。
- Provider 特例的完整范围、缓存优化收益与 token 成本实测。
- AST/LSP/Tree-sitter/Repo Map 的完整存在与调用情况；本 Batch 不作“全部没有”结论。
- 完整长期记忆能力及其与普通会话恢复的差异。

## Archify Diagram

[Diagram 0：Pi Repository High-Level Map](diagrams/diagram-0-repository/diagram-0-repository.html)，architecture，15 个一级组件。默认链与底部独立/开发包视图分开，虚线明确标注包依赖。JSONL 节点由真实写入点支持，不将 durable 存储当作默认后端。最终自动检查记录见 [README](README.md#校验记录)。

## Follow-up

进入 [Batch 1](batch-1-positioning.md)：先解释 Pi 的主要价值为什么位于模型外部的执行环境，再通过 Batch 2 的创建链验证这一定位。
