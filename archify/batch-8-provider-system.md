# Batch 8：Provider System

## Findings

**Confirmed**：Provider ID 与 API ID 是两个维度。OpenRouter factory 可同时提供 anthropic-messages 与 openai-completions；多个供应商共用 Completions adapter；OpenAI 默认 factory 使用 Responses。Provider 负责鉴权/目录与执行接口，API adapter 负责 SDK、payload、协议流、消息转换和错误归一。

**Confirmed**：适配超出改 baseUrl 的 API wrapper。已读实现包含跨模型签名清理、tool ID 修复、严格/grammar schema、图像结果编码、系统/工具中途更新、reasoning 参数、cache routing、usage/停止归一和 endpoint-specific 分支；Coding Prompt builder 没有三套默认供应商模板。

## Evidence

| 来源 | 直接证据 |
|---|---|
| [types.ts](../packages/ai/src/types.ts) / [models.ts](../packages/ai/src/models.ts) | KnownProvider、KnownApi、ProviderStreams、Provider / createProvider / dispatch |
| [providers/all.ts](../packages/ai/src/providers/all.ts) | builtinProviders 实际注册 factories，不根据目录猜测 |
| [openai.ts](../packages/ai/src/providers/openai.ts)、[anthropic.ts](../packages/ai/src/providers/anthropic.ts)、[google.ts](../packages/ai/src/providers/google.ts)、[openrouter.ts](../packages/ai/src/providers/openrouter.ts) | 已整文件读；身份、auth、catalog、API 集合 |
| [openai-completions.ts](../packages/ai/src/api/openai-completions.ts) | 已整文件读；stream、buildParams、convertMessages/Tools、detectCompat/getCompat、parseChunkUsage |
| [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) | 已整文件读；createClient、buildParams、convertMessages、native changes、OAuth、stream |
| [openai-responses.ts](../packages/ai/src/api/openai-responses.ts)、[openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts) | 已整文件读；Responses input/reasoning/function items 与终止事件 |
| [google-generative-ai.ts](../packages/ai/src/api/google-generative-ai.ts)、[google-shared.ts](../packages/ai/src/api/google-shared.ts) | 已整文件读；contents/parts、signature、thinkingConfig、usage |
| [constrained-sampling.ts](../packages/ai/src/api/constrained-sampling.ts)、[lazy.ts](../packages/ai/src/api/lazy.ts) | 已整文件读；strict subset、grammar 参数包装、异步 setup 错误变流事件 |

## Key Source Files

工厂装配与运行调用分别看 providers/all、models、Coding ModelRuntime。协议转换见上述四个主要 API 和 [Batch 4](batch-4-message-model.md)，最后 payload hook 顺序见 [Batch 5](batch-5-prompt-system.md)。

## Key Types / Classes

`Provider<Api>` 支持 auth、catalog、stream、streamSimple；`ProviderStreams` 是具体 API 的协议执行集合；`ApiStreamOptions` / `SimpleStreamOptions` 区分 native 与中立选项；`Model.compat` 分协议；AssistantMessage/Event/Usage 是 adapter 输出契约，不直接暴露原生 SDK response。

## Key Functions

`ModelRuntime.prepareRequest()` → `Provider.streamSimple()` → adapter `streamSimple()` / `buildBaseOptions()` → `stream()` → `createClient()` / `buildParams()` → `onPayload()` → `retryProviderRequest()` / SDK → native async iterator →统一 events/result。

## Control Flow

### 实际注册范围

KnownProvider 与 builtinProviders 对应以下内置身份：amazon-bedrock、ant-ling、anthropic、google、google-vertex、openai、azure-openai-responses、openai-codex、radius、typesafe、nvidia、deepseek、github-copilot、xai、groq、cerebras、openrouter、vercel-ai-gateway、zai、zai-coding-cn、mistral、minimax、minimax-cn、moonshotai、moonshotai-cn、huggingface、fireworks、together、baseten、opencode、opencode-go、kimi-coding、meta、cloudflare-workers-ai、cloudflare-ai-gateway、qwen-token-plan、qwen-token-plan-cn、qwen-token-plan-individual、xiaomi、xiaomi-token-plan-cn、xiaomi-token-plan-ams、xiaomi-token-plan-sgp。

KnownApi 的十个 chat 协议是 openai-completions、mistral-conversations、openai-responses、azure-openai-responses、openai-codex-responses、anthropic-messages、bedrock-converse-stream、google-generative-ai、google-vertex、pi-messages。API 类型也允许外部字符串；native factories 还可注册 image/classifier 协议。身份/注册已确认，不等于每种协议都作相同深度审计。

### 四条典型路径

| API | 请求 / 响应的关键区别 |
|---|---|
| Anthropic Messages | 顶层 system；user/assistant blocks；tool_use/tool_result；thinking/signature/redacted；OAuth Claude Code identity；native system/tool changes 按 compat 条件；响应映射 stop/length/toolUse/error |
| OpenAI Responses | input message/reasoning/function_call/custom_tool_call items；developer role 按模型能力；call_id 与 item_id 两段关联；reasoning.encrypted_content；必须收到终止 response 与完整 tool item |
| OpenAI Compatible Completions | messages、tools、tool role；max_tokens 或 max_completion_tokens；原生 reasoning 字段优先选择一种避免重复，reasoning_details 保存 opaque 数据；index/id 累积并行调用；检查 finish_reason，只有兼容选项明确关闭才允许无 finish_reason 推断 |
| Google GenAI | collapse systemInstruction；contents role=user/model、parts；functionCall/functionResponse；thought=true 标记可见 thinking，thoughtSignature 可在任意 part；工具图片按模型代际处理 |

OpenRouter 的路由参数、Anthropic cache_control、原生 reasoning_details 和 x-session-id 发生在 adapter/compat 分支，不额外产生一个 Agent Loop。自定义 OpenAI-compatible endpoint 可通过 Model.compat 覆盖自动 provider/URL 探测，未知 URL 不应默认获得所有高级能力。

## Data Flow

Tool schema 不是提示中的 snippet：统一 Tool.parameters → protocol input_schema/parametersJsonSchema/function.parameters。strict 转换复制 schema，所有属性 required、可选项变 nullable、additionalProperties=false；不支持关键词或结构时 prefer 退普通 schema，require 报错。grammar 有能力时用 custom 工具并把原生字符串包回统一单个 string 参数；不支持时仍用普通 function schema。

消息重放先 transformMessages；Completions 把 Responses 的复合 ID 规范化并保留 item 唯一性，必要时插入 assistant bridge、补工具 name、将工具图片另附 user。不同协议并不是把同一 JSON 原样提交。

Native text/thinking/tool deltas → adapter 累积 AssistantMessage →统一 start/delta/end/done/error；usage 再拆 cache/reasoning 并计算成本。partial 共享可变对象，最终工具执行等待完整消息。原生回调 onProviderStreamEvent 与统一事件是不同观察点。

## State

Provider 实例持有静态/动态目录；Models 保存 auth 与刷新状态。adapter 的 toolCall index/id Map、JSON scratch、reasoning metadata 属于单请求解析状态，终止后清除 scratch。并行输出不能按“最后一个工具块”单槽解析。缓存/连接资源的寿命不等于单个 assistant 消息，详见后续 Cache。

## Design Decisions

**Inference**：共用 adapter 节省重复协议实现，compat 把不一致 endpoint 的差异显式化；代价是组合分支需要针对测试。模型 ID/URL 分支是实际证据，不自动证明性能收益或“特定供应商更聪明”。这些归一、恢复、schema 和流处理是 Harness；生成推理/调用意图仍来自模型。

## Unknowns

没有真实账户或网络验证；Bedrock、Vertex、Codex transport、Mistral、Pi Messages 等仅确认接口/装配，本批未逐个声称完整 wire 审计。所有在线能力/费用可能不同于固定目录快照。也没有基准证明 compat 分支提高任务成功率。

## Archify Diagram

- [Diagram 8A：Provider Architecture](diagrams/diagram-8a-provider/diagram-8a-provider.html) · architecture。
- [Diagram 8B：LLM Request Sequence](diagrams/diagram-8b-request/diagram-8b-request.html) · sequence。
- [Diagram 8C：Provider Conversion Data Flow](diagrams/diagram-8c-conversion/diagram-8c-conversion.html) · dataflow。

## Follow-up

Batch 9 深入 thinking；Token/Cost、Cache、Compatibility 补统一计量、缓存与跨模型继续；扩展 Provider 的最小改动见最终开发地图。
