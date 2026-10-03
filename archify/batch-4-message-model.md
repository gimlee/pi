# Batch 4：Message Model

## Findings

**Confirmed**：Pi 采用自己的统一消息模型，`Message = SystemMessage | UserMessage | AssistantMessage | ToolResultMessage`。工具结果 role 是 `toolResult`；没有独立的通用 `ToolMessage` 类、`ReasoningBlock` 标签或 `ErrorMessage` role。推理内容用 `type="thinking"`，失败用 assistant.stopReason/errorMessage 或 toolResult.isError 表示。

**Confirmed**：系统消息同时声明文本、命名段落和工具变更；它不是只有 role/content 的简单 OpenAI 消息。Assistant 保存 api/provider/model、签名、usage、停止原因等，支持跨供应商历史重放。

**Confirmed**：Coding Agent 通过声明合并扩展 `AgentMessage`，增加 bashExecution、custom、branchSummary、compactionSummary；在请求边界转换成上述统一 Message。模型输出的 toolCall、工具执行结果和模型看到的 ToolResultMessage 是三个不同数据对象。

## Evidence

| 编号 | 已读来源 / 范围 | 直接支持 |
|---|---|---|
| M1 | [ai/types.ts](../packages/ai/src/types.ts) 395～621、756～802 | content、usage、stopReason、四种消息、统一事件 |
| M2 | 同文件 714～754 | Tool、Context 与品牌化 TranscriptContext；更后段有 Model/compat 元数据 |
| M3 | [agent/types.ts](../packages/agent/src/types.ts) 365～445、500～529 | 可扩展 AgentMessage、执行结果与 AgentEvent |
| M4 | [coding/messages.ts](../packages/coding-agent/src/core/messages.ts) 全文件 | 自定义消息定义及 convertToLlm |
| M5 | [transcript.ts](../packages/ai/src/utils/transcript.ts) 全文件；[text.ts](../packages/ai/src/utils/text.ts) | 系统/工具 replay、normalize/collapse、渲染 |
| M6 | [transform-messages.ts](../packages/ai/src/api/transform-messages.ts) 全文件 | 跨模型 thinking/签名/ID/图像处理、缺失工具结果与错误消息处理 |
| M7 | [agent-loop.ts](../packages/agent/src/agent-loop.ts) 381～476、922～940；[agent.ts](../packages/agent/src/agent.ts) 565～613 | assistant/result 入上下文和状态；事件时点 |
| M8 | [agent-session.ts](../packages/coding-agent/src/core/agent-session.ts) 1073～1290、1921～2064 | 输入规范化、扩展最终消息修改、Session 持久化 |
| M9 | [anthropic-messages.ts](../packages/ai/src/api/anthropic-messages.ts) 全文件，转换主要在 1124～1533 | Anthropic wire role/content、SSE 与 stop/usage 映射 |
| M10 | [openai-responses.ts](../packages/ai/src/api/openai-responses.ts)、[openai-responses-shared.ts](../packages/ai/src/api/openai-responses-shared.ts) 全文件 | Responses input items、签名、工具 ID、stream slots 与终止事件 |
| M11 | [google-shared.ts](../packages/ai/src/api/google-shared.ts)、[google-generative-ai.ts](../packages/ai/src/api/google-generative-ai.ts) 全文件 | Gemini contents/parts、signature、functionResponse、usage/finishReason |
| M12 | [event-stream.ts](../packages/ai/src/utils/event-stream.ts)、[json-parse.ts](../packages/ai/src/utils/json-parse.ts) 全文件 | final result promise、共享 partial、增量 JSON 修复 |

本批比较固定 HEAD 的本地 adapter 实现，不声称等同供应商当前所有 API 能力；其他协议未逐一审计。

## Key Source Files

字段与转换以 [ai/types.ts](../packages/ai/src/types.ts)、[coding/messages.ts](../packages/coding-agent/src/core/messages.ts)、[transcript.ts](../packages/ai/src/utils/transcript.ts)、[transform-messages.ts](../packages/ai/src/api/transform-messages.ts) 为阅读入口。上述三个供应商实现是具体 wire format 的证据。

## Key Types / Classes

### 1. 统一消息

| 类型 | 内容 / 关键字段 | 是否发送原样 |
|---|---|---|
| SystemMessage | content 为 string/TextContent[]；sections 为有序 name→string/null；toolsAdded 完整声明、toolsRemoved 名称引用；timestamp | adapter replay 或保留更新后编码；timestamp 不作为对话文字 |
| UserMessage | string 或 TextContent/ImageContent[]；timestamp | 转为供应商 user 内容 |
| AssistantMessage | TextContent/ThinkingContent/ToolCall[]；api/provider/model、responseModel/responseId、thinking level、diagnostics、usage、stopReason、deferred、errorMessage/rawStopReason/endTurn、timestamp | 内容选择性重放；其余多数是内部元数据 |
| ToolResultMessage | toolCallId、toolName；text/image 内容；details、usage、nestedCalls、isError、timestamp | content + 调用关联进入 wire；details/usage/nestedCalls 不作为工具输出正文 |

`responseModel` 表示返回的实际模型可能不同于请求模型，例如 Anthropic fallback。`endTurn` 类型注释明确保留用于调试，当前不参与 Agent 控制流；不能把它当作完成判定。

### 2. Content、Error、Metadata、Usage

| 能力 | 实际字段与语义 |
|---|---|
| Text | `type="text"`, text；可带 textSignature，OpenAI 编码 ID/phase，Google 保存签名 |
| Thinking / Reasoning | `type="thinking"`, thinking、thinkingSignature、redacted；可仅有 opaque signature 而没有可见推理全文 |
| Image | `type="image"`, base64 data、mimeType；user/toolResult 支持；统一 assistant chat content 不含生成图像（另有 AssistantImages API） |
| Tool Call | `type="toolCall"`, id、name、JsonObject arguments；可有 thoughtSignature/namespace |
| Tool Result | 单独 role，与 toolCallId 关联；模型可见 content 与程序细节分开 |
| Assistant Error | stopReason=error/aborted + errorMessage；可能保留已生成的部分块；setup 失败可直接 error 而无 start |
| Tool Error | isError=true，错误文字在 content；不会自动终止模型循环 |
| Metadata | 选型、response ID、provider level、diagnostics、时间、details/nestedCalls 等；request options.metadata 也存在，但不是统一消息 content block |
| Usage | input/output/cacheRead/cacheWrite、可选 cacheWrite1h/reasoning、totalTokens、分项 cost；reasoning 是 output 子集，不能再次相加 |
| StopReason | pending/stop/length/toolUse/error/aborted/deferred；不是四种 role 之外的新消息类型 |

**Confirmed**：`AgentToolResult` 还含 structuredContent、terminate 等。`createToolResultMessage()` 复制 content/details/usage/isError 和调用关联，不复制 structuredContent/terminate；structuredContent 为程序调用数据，不能声称它自动进入模型上下文。

### 3. Coding Agent 扩展消息

| AgentMessage role | 请求转换 | 保留在内部的其他字段 |
|---|---|---|
| bashExecution | 包装为 user text：运行命令、输出、取消/退出码、截断文件提示；excludeFromContext=true 则排除 | command、output、exitCode、cancelled、truncated、fullOutputPath、timestamp |
| custom | content 转为 user | customType、display、details；display=false 只是显示属性，仍会送模型 |
| branchSummary | 带 summary wrapper 的 user | fromId、timestamp |
| compactionSummary | 带 summary wrapper 的 user | tokensBefore、timestamp |
| 四种标准 role | pass through | adapter 再转换 |

用户运行 `!command` 的 bashExecution 和模型发出的 bash toolCall/result 是不同路径；`!!` 对应排除上下文选项。通用 Agent 默认 converter 只保留四种标准 role；Coding SDK 注入自己的 converter 才能翻译这些扩展 role。

## Key Functions

构造/消费：`AgentSession.prompt`、`streamAssistantResponse`、`Agent.processEvents`、`AgentSession._handleAgentEvent`、`createToolResultMessage`。

请求转换：`convertToLlm` → `normalizeContext` → `resolveTranscript/collapseSystemMessages` → `transformMessages` → Anthropic `convertMessages` / `convertResponsesMessages` / Google `convertMessages`。

重放：`getCurrentSystemMessage`、`getCurrentTools`、`getSystemMessageText`、`renderSystemMessageUpdate`、`resolveTranscriptTools`。流归一：`processResponsesStream`、各 adapter 的 stream、`parseStreamingJson`、`AssistantMessageEventStream.result`。

## Control Flow

### 请求到工具结果的一次闭环

1. input/skill/template/hooks 处理后构造 user text/image；Agent 发 message_start/end。此时系统 patch 和 user 已可保存，尚未发网络请求。
2. Session prepareRequest 从历史形成投影；transformContext 应用扩展及隐藏/forced prompt 处理；convertToLlm 把自定义消息转统一 Message。
3. normalizeContext 将 shorthand systemPrompt/tools 折入首条系统消息。TranscriptContext 是只含 messages 的品牌化类型，不是额外一份 prompt+tools 状态。
4. Provider 处理系统/工具 replay 和跨模型历史，再生成原生 request。模型返回流后 adapter 累积一个 AssistantMessage 并发统一事件。
5. Loop 消费 partial 并在 done/error 取得最终 result，发 message_end。Agent 存状态，Session 可应用同 role 替换后持久化。
6. 最终 toolCall 才进入工具流水线；工具 content/result 构造 toolResult，发 message_start/end；下一请求带调用和结果关联。

### 三种已审计协议的映射

| Pi 概念 | Anthropic Messages | OpenAI Responses | Google Generative AI |
|---|---|---|---|
| 初始系统提示 | 顶层 system text blocks | input 中 system/developer；reasoning 模型且兼容 developer 时选 developer | config.systemInstruction，系统历史合并 |
| 用户文字/图片 | user text/image(base64 source) | user input_text/input_image(data URL) | user parts.text/inlineData |
| Assistant | assistant blocks | output message/reasoning/function_call 等 input items | role=model 的 parts |
| 推理重放 | thinking + signature，redacted_thinking；无有效签名时可能退成 text | thinkingSignature 保存整个 reasoning item JSON，包括 encrypted_content | thought=true 与 base64 thoughtSignature；signature 本身不等于 thought |
| Tool Call | tool_use(id/name/input) | function_call 或 grammar custom_tool_call；内部 id 为 call_id\|item_id | functionCall(name/args，可选 id) |
| Tool Result | 相邻多个 tool_result 合成 user，tool_use_id/is_error | function_call_output/custom_tool_call_output，call_id；不独立编码 isError | user functionResponse，response.output/error；可合并连续结果 |
| 工具 schema | 顶层 tools.input_schema，兼容严格字段 | tools 的 function 或 grammar custom；另有按能力 anchored additions/tool search | functionDeclarations.parametersJsonSchema；另有 legacy parameters 转换 |
| 正常停止 | end_turn → stop；tool_use → toolUse；max_tokens → length | completed → stop，有调用再改 toolUse；incomplete.max_output_tokens → length | STOP → stop，有调用再改 toolUse；MAX_TOKENS → length |
| 失败停止 | refusal/sensitive → error；未知原因抛错 | failed 或非 max_output 的 incomplete → error | safety/malformed/其他非正常 finishReason → error |

Anthropic pause_turn 被映射 stop，底层没有只因该字符串自动续轮的专用分支；仍由 toolCall、队列或继续决策驱动。OpenAI Chat Completions、Codex WebSocket、Bedrock、Mistral 等不在本表完整审计范围，不能将 Responses 行套用到所有 GPT transport。

工具图像也有差异：Responses 可放 tool output input_image；Anthropic tool_result content 中放 image；Gemini 3+ 可放 functionResponse.parts，旧 Gemini 分开附 user image。精确行为依本地 model ID/compat 分支，不是抽象类型自动保证。

## Data Flow

### 重放不是无损原样发送

**Confirmed**：`transformMessages()` 在本次请求中执行下列变换，通常不改原始历史：

- 不支持图像的模型用说明文字替换连续图片。
- 同模型可保留带签名 thinking；跨模型 redacted thinking 丢弃，可见 thinking 退为 text；textSignature/工具 thoughtSignature 适用时移除。
- 通过 adapter 提供的 normalizeToolCallId 同步调用和结果 ID，避免跨协议字符/长度约束失败。
- error/aborted assistant 不重放；缺少对应结果的调用合成 isError=true、内容为 `No result provided` 的 toolResult。
- 工具调用与结果之间的 system 更新被暂存到结果之后，以维持 wire 顺序；不是把系统消息当作缺失结果的证据。

**Inference**：统一模型保留协议相关签名，同时 adapter 清理不可迁移信息，实现“尽量可继续”而不是协议字节完全等价。上下文编辑还可在此前改变消息，这属于下一批 Context 分析。

### 流事件不是每次完整快照

**Confirmed**：AssistantMessageEvent.partial 是共享的 live response；start/delta/end 的块不断增长。Loop 向 Agent 转发浅拷贝 message，嵌套块并非自动深冻结。消费者若需要历史时刻快照，要自己复制。

增量工具 JSON 尝试完整解析、转义修复、partial-json，再失败则 `{}`；这只帮助呈现和累计，不代表参数有效。执行前 schema 验证另做。Responses 还要求终止 response event 和完成的 tool item，未结束的 scratch buffer 调用会拒绝交给 Loop。adapter 最终清理 index/partialJson 等解析临时字段。

## State

三个阶段的数据状态是：请求的历史视图 → 共享部分 assistant → 最终 assistant/result。pending 是生成初值，done/error 解析出 finalResultPromise；error 也是有结果的终止事件，不是 EventStream.result 自动 reject。

系统 replay 状态是按消息顺序累加 content、按名字 patch/delete sections、按名字增删 tools。变化的工具定义会作为 removal+addition；执行函数/display 字段经 toToolDeclaration 去掉，schema JSON 化。Context.tools shorthand 与 AgentContext.tools 实现集合必须区分。

Usage 在 adapter 累积/归一后形成最终记录。Anthropic 合计 input/output/cacheRead/cacheWrite；Responses 从 input_tokens 扣出 cached/cache-write；Google output 包含 candidates + thoughts，reasoning 另记子集。零值也可能来自未生成就失败，不能视为完整计费数据。

## Design Decisions

- **Confirmed**：统一 Message 包含可 JSON 化的调用、内容和重放元数据。**Inference**：这是跨 Provider 会话、工具循环与 UI/历史共用的契约，价值超出 API wrapper。
- **Confirmed**：ToolResult content 与 details/structuredContent 分离。**Inference**：模型需要观察文字/图片，程序和 UI 需要结构与显示细节；混为一份会暴露不必要数据或丢失结构。
- **Confirmed**：独立统一流事件隔离供应商协议。**Inference**：Loop/TUI 可以处理一套事件，Provider-specific 恢复和兼容仍需 adapter 实现，不能认为统一接口消除了协议差异。

消息归一、签名重放、错误/孤儿工具修复是 Harness capability；模型生成内容/工具意图是 Model capability；跨模型继续任务是 Hybrid，保留程度有明确损失。

## Unknowns

没有用真实账户验证供应商接受所有组合；未完整审计其他 API、外部自定义 provider、版本迁移、deferred 恢复及导入不合法历史的全部路径。统一字段不保证每个 provider 都返回 usage、signature 或 responseId；opaque signature 也不意味着可读的完整思维过程。

## Archify Diagram

[Diagram 4：Message Data Flow](diagrams/diagram-4-messages/diagram-4-messages.html) · dataflow，分开请求归一、原生协议、响应归一、工具结果与历史。节点来源固定 HEAD；候选和校验记录保存在同目录，总体状态见 [README](README.md)。

## Follow-up

Batch 5 追踪系统段落来源、input/hooks 顺序及最终 request。后续 Context/Session/Provider/Reasoning 批次继续验证历史投影、压缩、迁移、跨模型签名和各 transport 的失败路径。
