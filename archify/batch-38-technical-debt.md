# Batch 38 · Technical Debt

## Findings

| 问题（Inference） | 源码事实 / 具体风险 | 建议与边界 |
| --- | --- | --- |
| Session 与 Host 集中职责 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 同时管队列/提示/工具/重试/摘要/扩展；[interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts) 同时管渲染/命令/会话 UI | 按 owner 边界拆分可改善维护；没有仅因文件大就断言 bug |
| 双入口 API 维护成本 | [compat.ts](../packages/ai/src/compat.ts) 的全局 registry 与 [models.ts](../packages/ai/src/models.ts) 的实例服务并存 | 明确迁移边界；不能仅删除 compat 破坏扩展行为 |
| 重复协议转换 | [openai-completions.ts](../packages/ai/src/api/openai-completions.ts)、[openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts)、[anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) 各实现 thinking/tools/cache | 共用重放不应抹平 native 差异；先加契约 fixtures 再抽取 |
| 错误文本分类 | [retry.ts](../packages/ai/src/utils/retry.ts) 通过正则区分 transient 与 quota | 结构化原因可减误判；当前兼容网关文字仍有实际价值 |
| 文件外部写竞争 | [edit.ts](../packages/coding-agent/src/core/tools/edit.ts) 读后直接写；队列只约束宿主进程 | 若需更强正确性，加入版本校验或 atomic write；不是本任务实施项 |
| 取消后的副作用 | [write.ts](../packages/coding-agent/src/core/tools/write.ts)、[execute.ts](../packages/coding-agent/src/extensions/codemode/execute.ts) 明确未回滚外部工具 | 界面/策略应区分 canceled 与 undone；没有通用事务承诺 |
| 配置优先级分散 | [sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[settings-manager.ts](../packages/coding-agent/src/core/settings-manager.ts)、[config.ts](../packages/coding-agent/src/extensions/mcp/config.ts) 不同读取路径 | 逐键说明/契约测试比抽象单一总优先级更可靠 |
| 缓存经济估计 | [cache-warmer.ts](../packages/coding-agent/src/core/cache-warmer.ts) 使用固定续请求概率和目录价格 | warm 有额外费用，需实测收益；本轮没有收益数据 |
| 验证缺口仍待测量 | tests/evals 存在但本轮未运行且未统计 coverage | 缺覆盖的具体断言需测试/coverage证据；不宣称仓库无测试 |

## Evidence

表中的风险都链接具体路径。它们是静态维护评估，不是已复现GitHub问题。

## Key Source Files

表中的风险都链接具体路径。它们是静态维护评估，不是已复现GitHub问题。

## Key Types / Classes

AgentSession、InteractiveMode、ModelRuntime、compat registry、FileMutationQueue。

## Key Functions

涉及方法见每项来源及 Batch 3/12/20/29/31。

## Control Flow

跨层链越长，修改一次边界越需要检查相邻生产者/消费者。

## Data Flow

重复转换需要协议契约；模型正文、UI details 与结构化输出不可合并。

## State

global cache与实例状态并存；各自有防陈旧机制，不能仅因global就判bug。

## Design Decisions

**Inference**：先用具体trace/fixture验证，再决定拆分或抽取；架构观察不能替代回归证据。

## Unknowns

未跑coverage、profile或故障注入；因此不声称泄漏、竞争bug或全面弱测试已经证实。

## Archify Diagram

[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

Batch 39 给出增加能力的真实切入点。
