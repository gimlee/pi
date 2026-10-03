# Batch 7：Model Abstraction

## Findings

**Confirmed**：`Model<Api>` 是包含模型 ID、provider ID、协议 API、baseUrl、能力、限额、成本与 compat 的数据对象；实际执行行为属于 Provider。Agent 不实例化 OpenAI/Anthropic SDK，调用注入 `StreamFn(model, TranscriptContext, options)`，消费统一 AssistantMessageEventStream。

**Confirmed**：当前 Coding SDK 使用 `ModelRuntime`，内部实例化 `Models` 集合并装配 Provider；`ModelRegistry` 是给扩展保留的同步兼容 facade。`pi-ai/compat` 仍有全局按 api 分发的旧注册表，不能把旧文档中的注册表当作 Coding Agent 唯一运行路径。

## Evidence

| 已读源码 | 实际职责 / 定位 |
|---|---|
| [ai/types.ts](../packages/ai/src/types.ts) | Model、ProviderStreams、SimpleStreamOptions、TranscriptContext、ThinkingLevelMap |
| [ai/models.ts](../packages/ai/src/models.ts) | Provider / Models / MutableModels，createModels、createProvider、applyAuth、refresh 全文件 |
| [model-runtime.ts](../packages/coding-agent/src/core/model-runtime.ts) | create、prepareRequest、streamSimple、resolveModel、registerProvider |
| [model-registry.ts](../packages/coding-agent/src/core/model-registry.ts) | 全文件；同步快照读与转发到 ModelRuntime |
| [sdk.ts](../packages/coding-agent/src/core/sdk.ts)、[agent-loop.ts](../packages/agent/src/agent-loop.ts) | 默认 streamFn 注入及实际调用 |
| [simple-options.ts](../packages/ai/src/api/simple-options.ts) | 单次输出预算、thinking 共用上限、请求选项传递 |
| [compat.ts](../packages/ai/src/compat.ts)、[providers/all.ts](../packages/ai/src/providers/all.ts) | 旧 API registry 与新的 builtinModels/provider factories |

## Key Source Files

类型入口是 ai/types；行为入口是 ai/models；产品配置入口是 ModelRuntime。具体参数最终进入 API adapter，消息转换见 [Batch 4](batch-4-message-model.md)，Provider 见 [Batch 8](batch-8-provider-system.md)。

## Key Types / Classes

| 抽象 | 内容与 owner |
|---|---|
| Model | id/name/provider/api/baseUrl；contextWindow/maxTokens；input；reasoning/thinkingLevelMap；cost、headers、samplingParams、compat、inputLimits、promptCache 等 |
| Provider | id/name/auth/catalog；stream/streamSimple；可选 refreshModels、images/classifier/deferred；由 Models 集合持有 |
| ModelsImpl | provider Map、CredentialStore、ModelsStore、refresh generations/controllers/publication chains |
| ModelRuntime | 用户 models.json、内置/扩展装配、凭据、可用模型快照、虚拟路由；由 Session services 使用 |
| ModelRegistry | runtime facade，getAvailable 是快照读而非每次网络探测 |
| StreamFn | Agent 注入点；模型数据 + 统一 transcript + 请求选项 → 事件流 |

## Key Functions

`builtinProviders()` → `createProvider()`；`createModels()` / `setProvider()`；`ModelRuntime.create()` / `refresh()` / `prepareRequest()`；`ModelRuntime.streamSimple()`；`ModelsImpl.streamSimple()`；`lazyStream()`；`Agent.streamAssistantResponse()` 调用配置中的 streamFn。

## Control Flow

正常请求：Agent Loop → SDK 注入 streamFn → ModelRuntime.streamSimple → normalizeContext → lazyStream 内 prepareRequest → 按 model.provider 取得 Provider → resolve auth/header/env/baseUrl → Provider.streamSimple → 按 model.api 选择 API 实现。底层 pi-ai Models 的独立调用走同形 applyAuth，Coding Runtime 自己准备鉴权后直接分发，不能画成两次必经鉴权。

虚拟模型不是额外供应商 API。Loop 的 prepareRequest 可以按当前消息、previous/failed、路由 state 选择有凭据的物理模型并 clamp thinking；直接的 Runtime.streamSimple 也支持虚拟路由，并限制目标输出预算。跨 provider 路由移除为原 provider 准备的 apiKey/headers/env，避免误传。路由函数由注册者提供，默认不是固定“失败就换某模型”。

## Data Flow

temperature、samplingParams、maxTokens 属于单次选项/模型默认，contextWindow/maxTokens 是模型能力数据。`buildBaseOptions()` 用输入估算和 4096 安全余量限制单次 maxTokens，最低 1；这不是自动删除消息或整任务预算。

reasoning boolean 决定是否支持统一级别，thinkingLevelMap 可以声明 null 不支持或映射 native effort。input 包含 image 表示输入能力；Tool Calling 不等于每个 Model 有一个通用 toolCapability boolean，schema/strict/grammar/动态工具等走具体 compat 与注册工具集合。ImageModel 与 ClassifierModel 是独立操作类型，不应当作 chat assistant 的新 content block。

## State

Models 的 refresh 可并行刷新不同 provider，但同 provider 新 generation 会取消旧 controller；publish 串行且前后检查 generation/signal，避免旧网络结果覆盖新注册。ModelRuntime 的 availability/catalog 是快照，未完成同步不保证立即反映所有动态结果。认证配置状态也不证明 endpoint 可达或请求成功。

## Design Decisions

**Inference**：模型能力数据、Provider 行为与 Agent 算法分开，使单 Provider 可服务多种 API，Agent 可复用。类型统一仍不能免除 native 协议差异。实例化集合减少独立 SDK 消费者之间共享注册状态；compat 全局表保留了另一种生命周期，需要迁移时明确调用路径。

鉴权、限额、路由与能力映射属于 Harness；模型生成输出属于 Model；选型后继续任务属于 Hybrid。

## Unknowns

未验证每个目录模型的 upstream 能力，也未访问模型目录网络源。动态注册、路由函数、availability refresh 的故障行为来自源码，未运行并发实验。目录能力是固定快照，不代表当前在线供应商状态。

## Archify Diagram

[Diagram 7：Pi Model Abstraction Architecture](diagrams/diagram-7-model/diagram-7-model.html) · architecture，区分 Agent、Runtime、Models、Provider、API 与模型元数据。最终校验记录见总索引。

## Follow-up

Batch 8 深入典型 API 协议及兼容层；Batch 9 说明 thinking 级别与内容重放；Config/Selection 批次补配置优先级和模型切换。
