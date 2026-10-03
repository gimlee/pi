# Batch 2：Startup 与 Runtime

## Findings

**Confirmed**：任务说明中的 `CLI → Args → Config → Environment → Runtime → Model → Tools → Extensions → Session → Agent` 是调查清单，不是本仓库的真实执行顺序。默认 CLI 先选择/恢复 `SessionManager` 并确定实际 `cwd`，再创建服务、加载扩展注册、选择模型，随后 SDK **先创建 Agent，再创建 AgentSession**，最后由 `AgentSessionRuntime` 包装当前 Session。

**Confirmed**：扩展初始化至少有两个可观察阶段：ResourceLoader 加载 factory 并收集注册；运行模式 Host 后续调用 `session.bindExtensions()` 绑定 UI/命令/退出能力并触发 `session_start`。MCP 在 `session_start` 可继续异步连接，不能将“扩展已加载”解释成“全部外部工具已就绪”。

**Confirmed**：应用生命周期分层拥有。`main`/Host 管理进程入口、I/O 和退出；`AgentSessionRuntime` 管理当前 Session 及 services 的替换；`AgentSession` 管理编码行为；`Agent` 管理当前 active run。`ExtensionRunner` 不是整个应用的 owner，`SessionManager` 也不是执行 Agent 的对象。

## Evidence

| 编号 | 已读来源 / 范围 | 直接支持 |
|---|---|---|
| S1 | [cli.ts](../packages/coding-agent/src/cli.ts) 1～6；[cli/setup.ts](../packages/coding-agent/src/cli/setup.ts) | 原始 CLI 调用和环境/HTTP 初始设置 |
| S2 | [main.ts](../packages/coding-agent/src/main.ts) 573～715 | bootstrap、短路命令、args、模式、startup 配置与 SessionManager |
| S3 | `main.ts` 717～867，`createRuntime` 与 `createAgentSessionRuntime` 调用 | 有效 cwd、project trust、services、选择和组装顺序 |
| S4 | [agent-session-services.ts](../packages/coding-agent/src/core/agent-session-services.ts) 135～230 | 模型运行服务、资源加载、pending 注册、本地 refresh、SDK 传参 |
| S5 | [sdk.ts](../packages/coding-agent/src/core/sdk.ts) 175～273、306～455 | restored messages/model/thinking、工具选择、Agent/Session 构造、streamFn |
| S6 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 462～497、3448～3610 | 订阅/安装 hooks、定义与工具包装、Runner 创建 |
| S7 | `agent-session.ts` 3211～3258 | `bindExtensions`、`session_start` 与资源发现 |
| S8 | [resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) 497～550、687～765 | pre-trust 与最终加载阶段 |
| S9 | [model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) 215～260、290～321、650～744 | Provider catalog/覆盖组合、鉴权、流请求分发 |
| S10 | [agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts) 74～226、404～440 | owner 字段、teardown/apply/rebind、switch 和 initial factory |
| S11 | [interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts) 596～606、931～1249、1959～2110 | TUI 创建、启动、绑定、run/getUserInput；绑定前已启动终端 |
| S12 | [print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts) 33～171；[rpc-mode.ts](../packages/coding-agent/src/modes/rpc/rpc-mode.ts) 54～819 | 无 TUI Host、绑定/订阅、prompt、背压、退出 |
| S13 | [session-manager.ts](../packages/coding-agent/src/core/session-manager.ts) 1000～1090、1165～1215、1755～1806 | SessionManager 工厂、header cwd、延迟首次写入、内存模式 |
| S14 | [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts) 215～265、416～516、582～633 | default tools、global/project 合并、project trust 与 overrides |
| S15 | [extensions/index.ts](../packages/coding-agent/src/extensions/index.ts)；[extensions/mcp/index.ts](../packages/coding-agent/src/extensions/mcp/index.ts) 960～997、1094 起 | CLI 内置扩展名单、MCP 异步启动与 shutdown |

范围是证据定位，不声称这几个片段可替代整文件阅读。交互源码有工作区改动，S11 的关键初始化段在 HEAD 未变；报告其他较后行号以工作区为准。

## Key Source Files

创建链核心：[main.ts](../packages/coding-agent/src/main.ts) → [agent-session-runtime.ts](../packages/coding-agent/src/core/agent-session-runtime.ts) → [agent-session-services.ts](../packages/coding-agent/src/core/agent-session-services.ts) → [sdk.ts](../packages/coding-agent/src/core/sdk.ts) → [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts)。

环境和装载核心：[config.ts](../packages/coding-agent/src/config.ts)、[settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)、[resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts)、[model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts)、[tools/index.ts](../packages/coding-agent/src/core/tools/index.ts)。

Host 核心：[interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)、[print-mode.ts](../packages/coding-agent/src/modes/print-mode.ts)、[rpc-mode.ts](../packages/coding-agent/src/modes/rpc/rpc-mode.ts)。

## Key Types / Classes

| 类型 / 类 | 对象含义 |
|---|---|
| `Args` / `AppMode` | 解析结果 / 实际应用模式；CLI 的 text mode 与 print 判定不能混同 |
| `CreateAgentSessionRuntimeFactory` | 根据目标 cwd/SessionManager 再创建 session/services 的工厂函数 |
| `AgentSessionServices` | cwd、agentDir、ModelRuntime、SettingsManager、ResourceLoader、诊断的依赖集合 |
| `AgentSessionRuntime` | 可替换的当前 Session 和 services，以及 Host rebind 回调 |
| `CreateAgentSessionOptions` / `AgentSessionConfig` | SDK 外部选项 / 编码 Session 的构造依赖 |
| `SessionManager` / `SessionContext` / `SessionProjection` | 历史树管理 / 恢复上下文 / 带来源的模型上下文投影 |
| `ModelRuntime` / `Model` / `Provider` | catalog/auth/route 服务 / 模型描述值 / 可处理请求的 Provider 实现 |
| `Agent` / `AgentState` | 通用 Agent 对象 / 当前消息、模型、工具、streaming 等状态 |
| `LoadExtensionsResult` / `ExtensionRuntime` / `ExtensionRunner` | 加载注册结果 / 注册共享状态 / 运行期事件与命令调用 |
| `ToolDefinition` / `AgentTool` / `ToolLoadout` | 注册定义 / 可执行契约 / 当前请求可用工具装载 |

## Key Functions

`setupCli`、`main`、`parseArgs`、`resolveAppMode`、`createSessionManager`、`buildSessionOptions`、`createAgentSessionRuntime`、`createAgentSessionServices`、`createAgentSessionFromServices`、`createAgentSession`、`ModelRuntime.create`、`DefaultResourceLoader.reload`、`AgentSession._buildRuntime`、`bindExtensions`、`InteractiveMode.init/run/getUserInput`、`runPrintMode`、`runRpcMode`、Runtime 的 `teardownCurrent/apply/finishSessionReplacement/dispose`。

## Control Flow

### 1. 默认 CLI 的实际启动顺序

以下顺序来自调用点，均为 **Confirmed**。它只描述正常进入会话的路径，短路和错误分支见下一节。

| 步骤 | 执行者与调用 | 输入 / 创建结果 |
|---|---|---|
| 1 | `cli.ts` → `setupCli()` → `main(argv)` | 设置 process title、`PI_CODING_AGENT` / `AI_AGENT`，配置 HTTP dispatcher |
| 2 | `main` 的早期环境与 bootstrap | 收集内置 extension factories，识别 offline；auth/维护命令；取得启动 cwd、agentDir；untrusted SettingsManager 只读全局 proxy 并重新配置 HTTP |
| 3 | `parseArgs` → `resolveAppMode` → migrations/startup settings | 参数、诊断、运行模式；非交互输出接管；交互首次设置可先发生 |
| 4 | `createSessionManager` | no-session/help/list-models 使用内存；fork/session/resume/continue/id/default 选择历史与目录；处理不存在的 cwd |
| 5 | `createAgentSessionRuntime(factory, target)` → `createRuntime` | `target.cwd = sessionManager.getCwd()`；目标 cwd 的信任上下文；显式 CLI 资源路径基于启动 cwd 解析一次 |
| 6 | `createRuntime` 创建目标 SettingsManager → `createAgentSessionServices` | ModelRuntime、SettingsManager、DefaultResourceLoader；先有基础 Provider catalog，再加载资源 |
| 7 | `resourceLoader.reload` | pre-trust 全局/显式资源及 factory → 决定 project trust → 最终配置与项目资源；收集 extensions、skills、templates、themes、context |
| 8 | services 应用 pending 注册并 refresh | compatibility Provider、native Provider、virtual Model；清空 pending 数组；`refresh({allowNetwork:false})`；应用扩展 flags |
| 9 | `resolveModelScope`、`buildSessionOptions` → `createAgentSessionFromServices` | CLI model/thinking/tool 选项、scopedModels；`--api-key` 运行期覆盖；复用本步已有 services |
| 10 | `createAgentSession` 读取 Session context 并补齐选型 | CLI/传入 model 优先；否则恢复分支选择；再按 settings/provider defaults 回退；恢复/clamp thinking；选初始工具名单 |
| 11 | SDK `new Agent` | restored messages、model/thinking、空初始 tools；注入转换、request callbacks、ModelRuntime streamFn、队列模式、cacheWarmer |
| 12 | SDK `new AgentSession` | 保存依赖；订阅 Agent；安装请求/工具/turn/boundary hooks；创建工具定义和包装、ExtensionRunner、核心 API 绑定及提示配置 |
| 13 | factory 返回 → `new AgentSessionRuntime` | 当前 session/services/diagnostics/modelFallbackMessage 与 factory 一起被持有 |
| 14 | `main` 后处理并分发模式 | 终端能力/HTTP 设置、help/list 出口、非 RPC stdin 和初始消息/图片、theme、启动诊断；选择 TUI/print/RPC |
| 15 | Host `bindExtensions` | 设置 mode/UI/command actions/errors/shutdown；发出 `session_start`，然后 `resources_discover` 可补充资源与提示 |
| 16 | Host 接收/提交输入 | TUI `getUserInput`；print 的初始/追加 messages；RPC 的 prompt/steer/follow_up 等 JSON 命令 → Session |

`createAgentSessionRuntime` 名称出现在较早的调用栈，但 Runtime 对象的 `new` 发生在 factory 完成之后。ModelRuntime 也不是 AgentSession 的子对象工厂创建的，而是 services 阶段创建后注入。

### 2. 分支与停止点

- **Confirmed**：auth、package/config、`mcp` 子命令、`--version`、export 等可提前退出，不保证创建完整会话。S2。
- **Confirmed**：`--help` / `--list-models` 不同：先创建内存 SessionManager，加载服务/扩展并构造 Runtime，之后在 `main` 的 metadata 分支退出，尚未由正常 Host 绑定 `session_start`。这样 help 可展示扩展 flags。S2、S4、`main` 873～888。
- **Confirmed**：runtime diagnostic error 在 Host 分发前致命退出；非交互没有可用 session model 也失败。交互模式可留给 UI 处理模型/登录选择。`main` 914～934。
- **Confirmed**：RPC 拒绝 CLI `@file` 参数，stdin 留给协议；其他模式可读 piped stdin。模式判定同时考虑 stdin/stdout TTY。`main` / `args.ts`。
- **Confirmed**：初始 ModelRuntime 构造及扩展注册后的 refresh 默认禁止 catalog network；RPC/interactive 可随后后台 refresh。`--offline` 设置内部 offline/version-check 标志。这不等于阻止所有 Provider、MCP 或扩展网络访问。S9、`main` 573～580、936 起。
- **Confirmed**：`PI_STARTUP_BENCHMARK` 的交互特例只 init/stop 并返回，不进入普通输入循环；非交互使用报错。不要把它当成正常会话退出样例。

### 3. 运行模式 Host 的行为

| Host | 初始化和输入 | 输出与退出边界 |
|---|---|---|
| InteractiveMode | 构造 terminal/TUI/editor/footer；`init` 先 `ui.start`，再 `rebindCurrentSession` / `bindExtensions`，因此 startup 扩展可显示对话框；`run` 初始化后处理初始 prompts，再循环 `getUserInput → session.prompt` | 订阅流事件并更新组件；内置 slash/`!` 命令可在 UI handler 处理；注册信号、终端 drain/stop、Runtime dispose 和 process exit |
| print（text / JSON） | `rebindSession` 绑定无 TUI 的 extension context，再订阅；顺序 await 初始和追加 prompts | text 输出最后 assistant 文本；JSON 输出 header/事件；JSON 为 Agent 订阅背压等待；finally disposeRuntime 并 flush，返回 exitCode 给 main |
| RPC | `runRpcMode` 接管 stdout；绑定协议 UI context、重绑定和订阅；逐行 JSON 输入；prompt preflight 成功后回应，Agent 流另发事件 | `extension_ui_request`/response 关联；Agent 订阅支持输出背压；EOF、signal 或扩展 shutdown 走 dispose、detach input、flush、process.exit；最后 unresolved Promise 保持存活 |

以上为 **Confirmed**，但没有本机交互运行测试。TUI 并非唯一 interactive loop：RPC 长期输入循环存在但不读取终端编辑器，底层 Agent Loop 是另一种循环。

### 4. Object Ownership：谁创建谁

| 对象 | 实际创建者 | 注入/持有者 | 更换或释放 |
|---|---|---|---|
| 启动/目标 SettingsManager | `main`，或 services/SDK 在没有传入时创建 | services、Session、ResourceLoader | Runtime factory 在目标 cwd 重新装配；项目信任控制 project settings |
| SessionManager | CLI `createSessionManager`；SDK 可默认 create；Runtime new/switch/fork/import 创建目标 manager | AgentSession；Agent 的初始消息来自它 | Runtime 替换 Session；原 JSONL 保留；manager 本身不执行模型 |
| ModelRuntime | `createAgentSessionServices` 调用 `ModelRuntime.create`；SDK 也有缺省创建分支 | services、Session、SDK streamFn/cacheWarmer 闭包 | CLI Runtime factory 重建 services；SDK 允许调用者传入共享实例 |
| 基础 Provider | `ModelRuntime.create` 调用 `builtinProviderCatalog.builtinProviders()` | ModelRuntime/MutableModels | 配置/native/compat extension 参与重组合与 refresh |
| 扩展 Provider / Virtual Model | 扩展注册，services 应用 pending registrations；运行期 core API 也能注册 | ModelRuntime | unregister/refresh 路径；完整生命周期待 Provider/Extension Batch |
| 当前 Model | SDK 从 options、history 和 ModelRuntime 查询/选择 | Agent state；Session 记录 model/thinking 变化 | `setModel`/cycle/virtual route 改变选择；Model 是描述值，不是每会话 new 的 SDK 客户端 |
| Agent | `createAgentSession()` 的 `new Agent` | AgentSession | Session dispose 请求 abort；每 prompt 是新的 active run，不是新建 Agent 对象 |
| AgentSession | SDK `new AgentSession(config)` | Runtime 的 `_session` 或直接 SDK 调用者 | Runtime teardown/apply；SDK 调用者管理 disposal |
| 内置工具定义 | `AgentSession._buildRuntime → createAllToolDefinitions(cwd, options)` | Session `_baseToolDefinitions` / `_toolDefinitions` | reload 重建；存在八类不代表全部向模型声明 |
| 可执行工具实例 | `_refreshToolRegistry → wrapRegisteredTools` | Session `_toolRegistry`，请求时设置 Agent loadout | extension/custom 同名覆盖；allow/deny/exposure/active 状态共同决定声明 |
| Extension 注册对象 | DefaultResourceLoader 的路径/factory 加载 | `LoadExtensionsResult` / shared ExtensionRuntime | reload 可清 cache 和换 runner；replaceable CLI built-ins 可被同能力扩展接管 |
| ExtensionRunner | `AgentSession._buildRuntime → new ExtensionRunner` | Session；SDK 的 runnerRef 让转换/request callbacks 使用当前 runner | Session reload/replace/dispose 使旧 ctx 失效 |
| AgentSessionRuntime | `createAgentSessionRuntime` 在 factory 返回后 `new` | CLI Host | 本身保存 factory，供 later new/resume/fork/import 使用 |
| TUI/terminal/components | `new InteractiveMode` 及其 `init` | 交互 Host | Host stop/shutdown；print/RPC 不创建同一 TUI 组件树 |

本表均为 **Confirmed**。Provider 工厂示例：[anthropic.ts](../packages/ai/src/providers/anthropic.ts) 的 `anthropicProvider()` 返回带 auth、model catalog 和 `anthropicMessagesApi()` 的 Provider。不能由此声称所有外部 SDK client 均在启动时创建；adapter 的 lazy 初始化需单独追踪。

### 5. 生命周期主人与会话替换

**Confirmed**：`AgentSessionRuntime` 保存 `_session`、`_services`、factory 和 Host 回调，不拥有进程/terminal。正常 `switchSession` 的短 trace：

```text
session_before_switch（可 cancel）
  → SessionManager.open(target)
  → 检查 target cwd 存在
  → await oldSession.abort()，等待当前响应结束
  → await oldRunner session_shutdown(reason=resume)
  → Host beforeSessionInvalidate 回调
  → oldSession.dispose()
  → await factory(target cwd, manager, session_start 元数据)
  → apply(new session/services)
  → await Host rebindSession(new session)
  → 可选 withSession(new ctx)
```

来源：S10。`newSession`、fork 和 import 也复用此 factory，但目标构造/事件原因和文件操作各有分支，不能假设所有替换都完全同形。

**Confirmed**：`Runtime.dispose()` 发出 `session_shutdown(reason=quit)`、Host invalidation 回调和 `Session.dispose()`；它并不直接执行 `process.exit()`，也不完全等同于替换路径先 await abort。Session dispose 使 runner 失效，取消相关操作并取消事件订阅/清理缓存与 Provider session resources。process/终端退出由 Host 管理。

**Confirmed**：目标 cwd 在旧会话 teardown 前校验，但 factory 实际运行在 teardown 后。当前实现没有在这条链上包一层完整事务回滚。因此“factory 失败自动恢复旧 Session”不是已确认保证；需要失败注入测试才能评估全部后果。

## Data Flow

### 配置、环境与模型选择

**Confirmed**：`getAgentDir()` 默认 `~/.pi/agent`，可由应用派生的 `ENV_AGENT_DIR`（默认 `PI_CODING_AGENT_DIR`）覆盖；Session directory 在 CLI 路径按 `--session-dir` → 对应 env → startup settings 选择；包 assets 由 `config.ts` helper 解析，不能用项目 cwd 代替安装路径。

**Confirmed**：SettingsManager 将 global 与受信任 project settings 深合并，再可应用 overrides；数组通常替换，`defaultTools` 对纯 `+name/-name` 列表有特殊合并/解析规则；`cacheWarming`、`defaultProjectTrust` 等读取 global-only。不存在一个适用于所有字段的“CLI > env > project > global”通用规则。S14。

**Confirmed**：`main` 的 CLI model/options 经 services 查询后进入 SDK；SDK 未显式给 model 时才恢复当前分支选择，失败再找 settings/provider defaults。虚拟模型选择需读 `model_change`，不能只用上一条 assistant 的物理模型。thinking 从显式选项、历史/配置补齐并按模型能力 clamp；CLI 可在外层进一步覆盖。

### 扩展、工具与 Prompt

**Confirmed**：CLI 注入四个 built-in extensions：llama.cpp、codemode、tool-search、mcp；后面三个 replaceable。单独 `createAgentSession()` 的缺省 ResourceLoader 不自动带这份 CLI factory 列表，SDK host 想要相同行为应显式提供加载策略，不能假设裸 SDK 与 CLI 完全等价。

**Confirmed**：内置工具定义为 read/bash/powershell/edit/write/grep/find/ls；默认初始启用 read/bash/edit/write。`tools` allowlist、`excludeTools` denylist、`noTools=all/builtin` 与 extension exposure/defaultActive 共同决定当前工具集合。`noTools=builtin` 与禁用所有扩展工具不同。构造时工具注册、请求时工具声明/装载也是不同步骤。

**Confirmed**：`_rebuildSystemPrompt` 汇集 SYSTEM/append、skills、context files、选中 tools 的 snippets/guidelines 到结构化 prompt options；`_preparePromptAndToolLoadout` 才在请求边界生成 system sections 差异、设置当前工具。不能把 Agent 构造中的 `systemPrompt:""` 解释为运行时没有系统提示。before_agent_start 强制 prompt 还有单独 request projection。

**Confirmed**：project trust 控制项目配置/扩展资源加载，显式 CLI temporary 资源可在 pre-trust pass 加载；AGENTS/context files 的发现有自己的逻辑。此处没有证据支持“所有项目文本在 untrusted 状态下一律不读取”，也没有证据把 project trust 当成 shell 工具的操作系统沙箱。

### 消息、事件与持久化

**Confirmed**：SDK 从 SessionManager projection 获取已有 messages → Agent initial state；请求时 Session 可重新投影当前分支 → transformContext / convertToLlm → ModelRuntime → Provider。Agent 的 message_end 由 Session 处理，再写 SessionManager。UI 展示的流片段不等于每个片段都已落盘。

**Confirmed**：普通新 Session 可以先产生 model/thinking/system 记录而没有创建 JSONL。`_persist` 在首次出现 user 或 assistant conversation message 后用 `wx` 写入此前 entries，后续使用 append；内存模式不写文件。不是“new SessionManager 必然立即创建会话文件”，也不宣称 appendFileSync 等同于 fsync/崩溃事务保证。S13。

## State

| 状态 | 主持有者 | 变更边界 |
|---|---|---|
| 启动参数、实际 appMode、initial input | main / Host | CLI 启动和 stdin 处理 |
| target cwd、agentDir、诊断、当前 services/session | AgentSessionRuntime / services | factory 创建、apply 替换；CLI 资源路径起点与目标 cwd 可不同 |
| global/project settings、trust 与写队列 | SettingsManager / ResourceLoader | reload、trust decision、overrides、setter |
| Provider、model snapshot、auth、virtual route | ModelRuntime | 注册/重组合/refresh，以及每请求 auth/route |
| 原始 fileEntries、byId、leaf、persist/flushed | SessionManager | append、branch、open/fork；projection 不重写原日志 |
| messages、model、tools、queue、activeRun/abort | Agent | prompt/continue、loop reducer、steer/follow-up、abort |
| Prompt/loadout、runner、retry/compaction/summary/bash 协调 | AgentSession | 请求边界、事件、恢复/settle、reload/dispose |
| editor、组件、selector、终端状态 | InteractiveMode | init/rebind/input/events/stop |
| RPC UI pending requests 与 shutdown flags | RPC Host | JSON commands、extension responses、EOF/signal |

这是对象状态边界，不是全局互斥/线程安全证明。尤其“Agent 当前 run 结束”与“Session 所有后续动作 settle”需要区分。

## Design Decisions

| 问题与例子 | 源码采用的方案 | 必要性与推断边界 |
|---|---|---|
| 从目录 A 打开 header cwd=B 的 Session，工具不能仍定位 A | 先选 SessionManager，再按实际 cwd 建 Settings/ResourceLoader/Tools | 对跨 cwd 恢复必要；解释为避免 stale resources 是 Inference |
| 扩展可注册 Provider，初始 model 查询不能先完成再漏掉它 | pre/post trust 资源加载、应用 pending Provider/virtual Model、本地 refresh，再 model selection | 行为 Confirmed；注册依赖选型次序是实际约束 |
| session_start 扩展可能需要 UI 对话框 | 先工厂注册/core bind，再 Host UI bind 和 start event；TUI 先 start renderer | 多 Host 的上下文绑定需要分阶段；作者动机 Inference |
| new/resume/fork 后旧 Extension ctx 不能继续使用旧工具/cwd | dispose/invalidate old runner，factory 重建，Host rebind，withSession 提供新 ctx | 替换边界必要；避免旧上下文误用的效果 Confirmed |
| 启动后马上退出不应生成空聊天文件 | 延迟至 conversation message 首次落盘 | 实现与注释 Confirmed；崩溃/断电完整保证 Unknown |
| 模型调用实现不应锁死通用 Agent | SDK 注入 ModelRuntime streamFn/转换/hooks | 复用结果 Confirmed；“最小 Agent 也必须有全套 catalog/cache warmer”不成立，属于可选复杂度 |

## Unknowns

1. 真实 Windows/Linux/macOS 信号、终端查询、下载 fd/rg、输出 drain 与退出时序的运行结果。本次只读源码。
2. Provider adapter 初始化、全部 env/credential 优先级、网络请求内容、模型专用行为；只追到了 Provider 分发与一个工厂示例。
3. MCP 全部连接失败、重连、auth、异步 tool loadout 与首次请求同步矩阵；已确认异步 startup 的存在，未完成 MCP 深查。
4. extension reload/provider refresh 并发与失败后恢复；Runtime factory failure 的全部宿主处理结果。
5. JSONL 崩溃一致性、多进程写同一 Session 的行为，以及 durable 与默认 Session 的完整关系。
6. 所有 context/compaction/branch summary 算法、成本和质量。owner 已定位，不以此替代后续机制审计。

## Archify Diagram

- [Diagram 2A：Pi Startup Lifecycle](diagrams/diagram-2a-startup/diagram-2a-startup.html)：workflow 图，表达真实执行先后与短路/metadata/错误出口；不是人为新增 lifecycle 状态机。
- [Diagram 2B：Pi Runtime Architecture](diagrams/diagram-2b-runtime/diagram-2b-runtime.html)：architecture 图，12 个职责组件，明确 Host、Runtime、Session、Agent、Runner、ModelRuntime 与 SessionManager 的边界。

每个节点有固定提交的源码范围。结构/交付/严格 artifact/真实浏览器自动检查均通过；未人工截图审查，完整记录见 [README](README.md#校验记录)。图没有为默认路径添加 durable DB、remote server 或强制审批 gate。

## Follow-up

下一步 Batch 3 首先追踪 `Agent.prompt → runAgentLoop → runLoop`，列全 tool calls、truncated length、finishTurn、steer/follow-up、error/abort 与 extension continuation 条件；然后回到 `_runAgentPrompt` 比较底层 end 与编码层 settle。测试应使用包内 specific tests；编码 suite 采用 faux provider，不调用真实模型。之后再单独深查 Context/Compaction、Provider 和工具机制。
