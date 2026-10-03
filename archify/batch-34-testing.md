# Batch 34 · Testing Infrastructure

## Findings

**Confirmed**：存在离线单元/集成、faux provider 的 Session suite、真实 API E2E、MCP conformance 与模型 eval。测试名称说明覆盖意图，不证明当前版本测试通过。此分析未运行项目测试。

| 层 | 示例/机制 |
|---|---|
| 算法/工具 | edit/compaction/truncation/cache tests |
| Agent / Session | suite/harness.ts 注入 faux steps、收集 events |
| Provider mapping | api-specific fixtures/mock SDK 与 usage/retry tests |
| MCP | in-memory/stdio/HTTP fixtures、OAuth 与 conformance |
| TUI | node:test、宽度与组件行为 |
| 真实模型 | ai *-e2e 与 evals；需要显式运行边界 |

## Evidence

- [test.sh](../test.sh)：隔离HOME/credentials/env后运行测试。
- [vitest.config.ts](../packages/coding-agent/vitest.config.ts)：PI_OFFLINE=1。
- [harness.ts](../packages/coding-agent/test/suite/harness.ts)：createHarness。
- [client.test.ts](../packages/mcp/test/client.test.ts)：client tests。
- [package.json](../packages/evals/package.json)：eval 与 test 分离。

## Key Source Files

[test.sh](../test.sh)、[packages/coding-agent/vitest.config.ts](../packages/coding-agent/vitest.config.ts)、[packages/coding-agent/test/suite/harness.ts](../packages/coding-agent/test/suite/harness.ts)、[packages/mcp/test/client.test.ts](../packages/mcp/test/client.test.ts)、[packages/evals/package.json](../packages/evals/package.json)。

## Key Types / Classes

FauxResponseStep、Harness、InMemory Auth/Settings/Session、AgentSessionEvent。

## Key Functions

createHarness；registerFauxProvider；setResponses；eventsOfType；cleanup。

## Control Flow

构造 fixtures/provider → 创建 session → prompt/操作 → 收集事件/日志 → assertions → cleanup。模型 E2E 与 eval 是另一路，不应在默认分析时启动。

## Data Flow

faux 捕捉请求/消费预制步骤；模拟协议结果能验证时序，不能验证真实模型决定。

## State

临时 cwd、隔离 env、faux 注册、events 数组、fixture servers。

## Design Decisions

**Inference**：确定性测试保护 Harness 契约，live eval 验证模型协作；两个维度需要分别报告。

## Unknowns

未执行测试或统计覆盖率；不能从目录结构声称所有错误路径均被覆盖。

## Archify Diagram

[diagram-34-tests](diagrams/diagram-34-tests/diagram-34-tests.html)

## Follow-up

Batch 35 分析仓库内真实 eval 的指标与局限。
