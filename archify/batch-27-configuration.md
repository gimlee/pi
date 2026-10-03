# Batch 27 · Configuration Flow

## Findings

**Confirmed**：没有全局统一的 CLI > env > project > user 优先级公式，必须按字段看调用点。

| 配置 | 实际来源与覆盖 |
|---|---|
| settings | 默认 getter + 全局 + 项目可允许字段；嵌套对象合并，特定数组/工具选择有专门逻辑 |
| 模型选择 | SDK/CLI 显式模型、可恢复 branch selection、默认 provider/model 与可用目录共同决定 |
| thinking | 显式值 → 恢复条目（继续会话）→ per-model → 全局 → 默认；最后 clamp |
| tools | options.tools / noTools / settings.defaultTools / DEFAULT_TOOL_NAMES，另做 exclude |
| 请求 retry/timeouts | 显式 request options 优先于 settings provider defaults |
| MCP | 可信项目同名覆盖全局，文件同名覆盖注册项；项目 auth.provider 禁止 |
| 环境 | provider 鉴权/路径/代理有各自解析，不是覆盖所有 settings |


版本边界：以上配置结论对应固定研究基线。当前 HEAD 新增可信项目 enabled/exposure/toolExposure 的局部覆盖分支，见 [基线推进说明](baseline-drift.md)。

## Evidence

- [settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)：Settings / 合并 / trust / getters。
- [sdk.ts](../packages/coding-agent/src/core/sdk.ts)：模型/工具/thinking 解析。
- [model-config.ts](../packages/coding-agent/src/core/model-config.ts)：models.json immutable snapshot。
- [resolve-config-value.ts](../packages/coding-agent/src/core/resolve-config-value.ts)：env 与 !command 配置值。
- [config.ts](../packages/coding-agent/src/extensions/mcp/config.ts)：MCP 合并。

## Key Source Files

[packages/coding-agent/src/core/settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)、[packages/coding-agent/src/core/sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[packages/coding-agent/src/core/model-config.ts](../packages/coding-agent/src/core/model-config.ts)、[packages/coding-agent/src/core/resolve-config-value.ts](../packages/coding-agent/src/core/resolve-config-value.ts)、[packages/coding-agent/src/extensions/mcp/config.ts](../packages/coding-agent/src/extensions/mcp/config.ts)。

## Key Types / Classes

SettingsManager、SettingsStorage、ModelConfig、CreateAgentSessionOptions、ProviderRetrySettings。

## Key Functions

SettingsManager.create/inMemory；deepMergeObjects；ModelConfig.load；resolveConfigValue；createAgentSession。

## Control Flow

文件/环境/CLI → 各 reader 校验 → 服务创建 → SDK 选项解析 → Session/Provider 请求；/settings 修改持久化，再按相关 getter 生效。reload 可重建资源而非自动恢复旧对象。

## Data Flow

不可变 ModelConfig 不读 credential；ModelRuntime composer 组合目录与 config；credential 另由 AuthStorage/Models 解析。项目 trust 过滤执行/隐私相关配置，cacheWarming 为 global-only。

## State

global/project/effective settings、dirty fields、文件存储、runtime overrides；不是每个 getter 每次都重新读取文件。

## Design Decisions

**Inference**：按用途拆分可防项目静默改变部分全局执行策略；复杂性是开发者必须阅读具体字段路径。

## Unknowns

配置键全面支持见 Settings 类型与 getter；本报告没有把未经逐字段动态测试的行为标为运行已验证。

## Archify Diagram

[diagram-27-config](diagrams/diagram-27-config/diagram-27-config.html)

## Follow-up

Batch 28～29 检查错误与恢复。
