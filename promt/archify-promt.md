# Pi Agent 深度源码分析与 Archify 可视化任务

你现在需要对当前 **Pi Agent** 代码仓库进行一次系统性的源码级逆向分析。

这不是普通的“代码仓库介绍”，也不是只生成一张架构图。

最终目标是：

> 真正理解 Pi Agent 作为一个 Agent / Coding Agent / Agent Harness 是如何工作的，并达到可以修改核心机制、增加 Provider、增加 Tool、修改 Agent Loop、修改 Context 管理、开发 Extension、替换模型调用逻辑的程度。

整个分析必须基于真实源码。

Archify 用于把源码分析结果生成：

- Architecture
- Workflow
- Sequence
- Data Flow
- Lifecycle

等技术图。

Codex 负责源码调查和调用链追踪。

---

# 一、最终需要回答的核心问题

整个分析最终必须能够回答：

1. Pi Agent 本质上是什么？
2. 它到底属于：
   - Coding Agent
   - Agent Harness
   - Agent Runtime
   - CLI Agent
   - Agent Framework
   - Model Harness
   - 其他
3. Pi 最核心的 Runtime 在哪里？
4. Agent Loop 在哪里？
5. 一次用户输入是如何最终变成 LLM 请求的？
6. LLM 返回 Tool Call 后，Pi 如何执行？
7. Tool Result 如何重新进入模型上下文？
8. Pi 如何判断继续执行还是结束？
9. Context 如何管理？
10. Prompt 如何构造？
11. Conversation History 如何管理？
12. 大上下文如何压缩？
13. Pi 如何适配不同模型？
14. Provider 层到底有多厚？
15. Pi 是否针对不同模型进行了 Harness 优化？
16. Tool 系统如何设计？
17. File / Edit / Shell 等工具如何工作？
18. Pi 如何理解代码仓库？
19. Pi 是否使用 AST / LSP / Tree-sitter / Repo Map？
20. Pi 如何管理 Session？
21. Pi 是否支持长期/短期 Memory？
22. Extension / Plugin / Skill 如何工作？
23. MCP 如何接入？
24. CLI / TUI 与核心 Runtime 是否解耦？
25. Pi 真正有技术含量的核心代码在哪里？
26. 如果只保留 20% 代码，哪些属于 Pi 的 Minimal Core？
27. 如果我要二次开发 Pi，应从哪些代码开始？

---

# 二、源码分析原则

## 1. Source First

README 和官方文档只能作为线索。

最终结论必须优先使用真实源码验证。

优先查看：

- Entry Point
- Class
- Function
- Method
- Type
- Interface
- Factory
- Registry
- Adapter
- Runtime Registration
- Event Handler
- Actual Call Site

---

## 2. 不允许根据目录名直接推测

禁止出现这种低价值分析：

> agents 目录负责 Agent。

> tools 目录负责工具。

> providers 目录负责 Provider。

必须继续深入：

谁实例化谁？

谁调用谁？

什么时候调用？

传递什么对象？

状态在哪里？

---

## 3. 所有重要结论给出源码位置

例如：

`packages/xxx/src/xxx.ts`

`Agent.run()`

`Session.create()`

`ToolRegistry.execute()`

如果存在。

---

## 4. 标记证据等级

使用：

### Confirmed

源码明确确认。

### Inference

基于源码结构合理推断。

### Unknown

当前无法确认。

不要把推测写成事实。

---

# 三、Archify 使用原则

Archify 不负责代替源码分析。

正确流程应该是：

Codex 阅读源码

↓

追踪调用链

↓

确认 Runtime / Data Flow / State

↓

使用 Archify 生成对应技术图

不要根据 README 直接让 Archify 猜系统架构。

---

# Batch 0：Repository Reconnaissance

首先不要直接深入所有代码。

快速侦察整个 Pi 仓库。

分析：

- Repository structure
- Workspace / Monorepo
- Packages
- Apps
- CLI
- TUI
- Core
- Agent
- Model
- Provider
- Tools
- Extensions
- Sessions
- Storage
- Tests

实际存在什么分析什么。

---

## 找出入口

必须找到：

- CLI Entry
- main
- bootstrap
- interactive mode
- non-interactive mode
- library entry
- TUI entry

如果 Pi 有多个运行模式，分别记录。

---

## 初步找出核心模块

重点寻找：

Agent Runtime

Agent Loop

Session

Message

Context

Prompt

Model

Provider

Tool

Extension

Workspace

Event

Storage

---

## 输出

### Pi Repository Overview

### Technology Stack

### Package / Module Structure

### Entry Points

### Core Module Candidates

### Important External Dependencies

### Investigation Plan

---

## Archify

生成：

# Diagram 0：Pi Repository High-Level Map

要求：

只保留 10～20 个一级重要组件。

不要把普通 utils 画进去。

---

# Batch 1：Pi 到底是什么

这是概念定位阶段。

不能照抄 README。

基于代码判断：

Pi 的核心本质到底是什么？

重点判断：

Pi 是：

Model Wrapper？

Agent Loop？

Coding Agent？

Agent Harness？

CLI？

Agent Runtime？

Framework？

这些里面哪些是核心，哪些只是外壳？

---

## 分析项目的价值分布

把代码分成：

### Core Intelligence / Harness

真正决定 Agent 行为。

### Infrastructure

配置、日志、存储等。

### Integration

Provider / MCP / 外部服务。

### Presentation

CLI / TUI / UI。

### Glue Code

普通连接代码。

---

## 输出

最后用一段话回答：

> 如果不用 Pi，而直接调用 LLM API + Tool Calling，我究竟缺少了什么？

这是整个分析的重要问题。

---

# Batch 2：Startup 与 Runtime

从真实入口开始追踪。

---

## Startup

完整追踪：

CLI Entry

↓

Args

↓

Config

↓

Environment

↓

Runtime

↓

Model / Provider

↓

Tools

↓

Extensions

↓

Session

↓

Agent

↓

Interactive Loop

---

## 找出 Runtime Owner

必须回答：

谁是整个 Pi 应用生命周期的主人？

可能是：

Application

Runtime

AgentSession

Runner

Context

或者其他对象。

以源码为准。

---

## Object Ownership

分析：

谁创建：

Agent？

Session？

Model？

Provider？

Tools？

Extensions？

---

## Archify

生成：

### Diagram 2A：Pi Startup Lifecycle

### Diagram 2B：Pi Runtime Architecture

---

# Batch 3：Agent Loop

这是最高优先级模块。

必须深入。

---

## 找到真正的 Agent Loop

不要只找名为 Agent 的类。

搜索实际循环：

while

for

recursive

generator

async iterator

event driven loop

---

## 从用户输入开始追踪

例如：

User：

“修改这个 Bug。”

然后追踪：

Input

↓

Session

↓

Message

↓

Context

↓

Model

↓

Model Response

↓

Tool Call

↓

Tool Execution

↓

Tool Result

↓

Model

↓

Final Response

---

## 必须回答

谁调用 Model？

谁解析 Response？

谁决定执行 Tool？

谁追加 Tool Result？

谁发起下一轮？

谁判断完成？

---

## Stop Conditions

分析：

final response

stop reason

max turns

max tokens

abort

cancel

error

timeout

---

## Agent Strategy

检查是否存在：

ReAct

Plan-and-Execute

Planner

Reflection

Reviewer

Critic

Todo

Task

Sub-agent

如果不存在明确说明。

---

## Archify

生成：

### Diagram 3A：Pi Agent Runtime Workflow

### Diagram 3B：Pi Agent Loop Lifecycle

### Diagram 3C：Pi Agent State Machine

如果存在明确状态。

---

# Batch 4：Message Model

专门研究 Pi 内部消息体系。

寻找：

Message

UserMessage

AssistantMessage

SystemMessage

ToolMessage

ToolResult

ContentBlock

ReasoningBlock

---

## 分析

Pi 内部消息模型是什么？

和 OpenAI / Anthropic / Gemini 的消息格式是否不同？

---

## Message Lifecycle

追踪：

User Input

↓

Internal Message

↓

Provider Message

↓

Model Response

↓

Internal Message

↓

Tool Message

---

## 特别分析

是否支持：

Text

Thinking

Reasoning

Image

Tool Call

Tool Result

Error

Metadata

Usage

---

## Archify

生成：

### Diagram 4：Message Data Flow

---

# Batch 5：Prompt System

研究模型真正收到什么。

---

## 找出

System Prompt

Agent Instructions

Default Instructions

Project Instructions

User Rules

Tool Instructions

Extension Instructions

Environment Instructions

Repository Instructions

---

## Prompt Composition

必须追踪真正构造 request 的函数。

回答：

System Prompt

+

Environment

+

Project Rules

+

Extension

+

Conversation

+

Tools

+

User

实际顺序是什么？

---

## Dynamic Prompt

检查 Prompt 是否根据：

Model

Provider

Environment

Tools

Repository

User Config

动态变化。

---

## Provider-specific Prompt

特别检查：

Claude

GPT

Gemini

等是否使用不同 Prompt。

如果有，这可能说明 Pi 对模型进行了 Harness 优化。

---

## Archify

生成：

### Diagram 5A：Prompt Composition Data Flow

### Diagram 5B：Prompt Build Workflow

---

# Batch 6：Context Management

这是 Pi 分析第二重要模块。

---

## Conversation History

回答：

所有 History 是否都会发送给模型？

---

## Context Window

找出：

token counting

context length

truncate

prune

compact

summary

compression

---

## Context Overflow

如果超过模型 Context：

Pi 做什么？

---

## Tool Result

特别关注大型 Tool 输出：

Shell 输出几万行怎么办？

Read File 很大怎么办？

Search 返回很多结果怎么办？

---

## Compression

分析是否存在：

conversation compression

summary

message pruning

tool output truncation

context compaction

---

## Compression Trigger

什么时候触发？

Token 百分比？

固定阈值？

模型报错后？

手工触发？

---

## Compression Result

压缩之后保留什么？

丢弃什么？

---

## Archify

生成：

### Diagram 6A：Pi Context Data Flow

### Diagram 6B：Context Compaction Lifecycle

### Diagram 6C：Model Context Composition

---

# Batch 7：Model Abstraction

研究 Pi 内部如何抽象模型。

---

## 寻找

Model

LLM

Provider

API

Adapter

Client

Stream

---

## Model Interface

找到 Agent 调用模型时真正依赖的接口。

---

## 分析

Agent 是否知道具体 Provider？

还是只面对统一 Model abstraction？

---

## Model Configuration

检查：

model ID

context window

max output

temperature

reasoning

thinking

tool capability

image capability

---

## Archify

生成：

### Diagram 7：Pi Model Abstraction Architecture

---

# Batch 8：Provider System

深入不同 Provider。

实际支持哪些 Provider，以源码为准。

---

## Provider Interface

分析统一 Provider 接口。

---

## Provider Adapter

分别研究典型：

OpenAI

Anthropic

Gemini

OpenRouter

OpenAI Compatible

如果存在。

---

## Message Conversion

内部 Message

如何转成：

OpenAI format

Anthropic format

Gemini format

---

## Tool Conversion

内部 Tool schema

如何转换 Provider 格式。

---

## Streaming

研究：

text delta

reasoning delta

thinking delta

tool delta

usage

finish reason

---

## Provider Differences

这是很重要的一点。

搜索是否存在：

if provider === ...

if model.startsWith(...)

capabilities

provider-specific handling

---

## Harness Optimization

重点判断：

Pi 是否只是做统一 API Adapter，

还是针对具体模型：

Claude

GPT

Gemini

做了 Prompt / Tool / Context / Streaming / Reasoning 优化。

必须给源码证据。

---

## Archify

生成：

### Diagram 8A：Provider Architecture

### Diagram 8B：LLM Request Sequence

### Diagram 8C：Provider Conversion Data Flow

---

# Batch 9：Reasoning / Thinking

单独研究推理模型支持。

因为这会影响 Harness 设计。

寻找：

reasoning

thinking

thinkingBudget

reasoningEffort

reasoningTokens

extendedThinking

---

## 分析

内部是否有统一 Reasoning abstraction？

不同 Provider 如何映射？

---

## Reasoning Content

是否保存进 Conversation？

下一轮是否重新发送？

---

## Hidden / Visible Reasoning

代码中如何处理？

只分析程序行为，不推断模型内部思维内容。

---

# Batch 10：Tool System

这是核心模块。

---

## 找出

Tool

ToolDefinition

ToolSchema

ToolRegistry

ToolExecutor

ToolResult

---

## Tool Registration

工具什么时候注册？

静态？

动态？

Extension？

MCP？

---

## Tool Schema

工具参数如何转换给模型？

---

## Tool Invocation

完整追踪：

Model

↓

Tool Call

↓

Parser

↓

Registry

↓

Validation

↓

Execution

↓

Result

↓

Message

↓

Model

---

## Tool Error

Tool 报错：

返回模型什么？

---

## Parallel Tool Calls

是否支持？

如何并发？

---

## Archify

生成：

### Diagram 10A：Pi Tool Architecture

### Diagram 10B：Tool Call Sequence

### Diagram 10C：Tool Result Data Flow

---

# Batch 11：内置 Tools

对 Pi 的内置核心工具逐个分析。

实际存在什么分析什么。

重点寻找：

Read

Write

Edit

Patch

Bash

Shell

Search

Grep

Glob

Find

Git

Web

Browser

Task

---

## 对每一个核心 Tool 分析

入口

参数

权限

执行方式

Result Format

Error Handling

Output Limit

---

## 找出工具之间是否共享抽象。

---

# Batch 12：File Read / Edit

Coding Agent 的重点。

---

## File Read

研究：

encoding

size limit

binary detection

line range

output truncation

---

## File Edit

判断使用：

overwrite

replace

search / replace

patch

unified diff

AST edit

---

## Edit Safety

检查：

old content validation

conflict detection

line matching

hash

version

---

## Diff

修改后是否生成 Diff？

---

## Archify

生成：

### Diagram 12A：File Read Sequence

### Diagram 12B：File Edit Sequence

### Diagram 12C：Edit Data Flow

---

# Batch 13：Shell / Bash Tool

深入研究终端执行。

---

## 分析

process creation

shell

cwd

environment

stdin

stdout

stderr

exit code

timeout

cancel

streaming

---

## Long Running Process

如何处理：

server

watch

tail

interactive command

---

## Process State

是否支持：

background process

process ID

continue

terminate

---

## Archify

生成：

### Diagram 13：Shell Execution Lifecycle

---

# Batch 14：Repository Understanding

核心问题：

Pi 到底怎么理解大型代码仓库？

---

## 查找

grep

ripgrep

glob

find

tree

git ls-files

symbol

LSP

Tree-sitter

AST

embedding

repo map

semantic search

---

## 判断机制

A. 按需文件搜索

B. 全库预索引

C. Symbol Index

D. LSP

E. Semantic Retrieval

F. Repo Map

G. Hybrid

---

## Agent 探索代码过程

研究：

User Task

↓

Search

↓

Read

↓

Follow Import

↓

Read Related Files

↓

Modify

这到底是 LLM 自主完成，

还是 Runtime 有专门 Repository Intelligence？

---

## 关键结论

必须回答：

> Pi 理解代码仓库主要依靠 Harness 的代码智能，还是依靠 LLM + 普通搜索工具？

---

## Archify

生成：

### Diagram 14A：Repository Understanding Workflow

### Diagram 14B：Repository Context Data Flow

---

# Batch 15：Coding Task 全链路

选择一个典型任务：

> “修复当前项目中的一个 Bug。”

追踪真实运行过程：

User

↓

Task

↓

Search Repository

↓

Read

↓

Model Reasoning

↓

Edit

↓

Run Test

↓

Failure

↓

Read Output

↓

Modify

↓

Run Test

↓

Success

↓

Final Response

---

## 分析

这个 Workflow 是系统规定的，

还是完全由模型自己决定？

这是判断 Harness 强弱的重要问题。

---

## Archify

生成：

### Diagram 15：Pi Coding Task Workflow

---

# Batch 16：Session

研究 Session 是什么。

---

## Session 保存什么

Model

Provider

Conversation

Tools

Extensions

CWD

Repository

Token Usage

State

---

## Lifecycle

Create

↓

Active

↓

Model

↓

Tool

↓

Active

↓

Complete

---

## Session Persistence

关闭 Pi 后 Session 是否存在？

---

## Resume

是否可以继续？

---

## Branch / Fork

Conversation 是否支持分叉？

---

## Archify

生成：

### Diagram 16：Pi Session Lifecycle

---

# Batch 17：State Management

区分：

Application State

Session State

Agent State

Conversation State

Tool State

UI State

Extension State

---

## 分析

状态在哪里存？

内存？

文件？

数据库？

---

## State Ownership

谁能修改什么 State？

---

## Archify

生成：

### Diagram 17：Pi State Architecture

---

# Batch 18：Persistence

寻找：

JSON

JSONL

SQLite

DB

Filesystem

Config

Session File

History

---

## 分析

Pi 持久化：

Conversation？

Session？

Config？

Model Selection？

Usage？

Memory？

---

## Archify

如有必要生成：

### Diagram 18：Persistence Data Flow

---

# Batch 19：Memory

不要因为项目叫 Agent 就默认存在 Memory。

先确认。

---

## 检查

Short-term Memory

Long-term Memory

User Memory

Project Memory

Session Memory

---

## 如果没有

明确区分：

Conversation History

和

真正意义上的 Memory。

---

## 如果存在

分析：

Read

Write

Retrieval

Injection

Persistence

---

# Batch 20：Extension System

Pi 如果有 Extension，这是重点。

寻找：

Extension

Plugin

Hook

Middleware

Command

Custom Tool

Custom Provider

Event Handler

---

## Extension Lifecycle

Discovery

↓

Load

↓

Initialize

↓

Register

↓

Execute

↓

Dispose

---

## Extension 能修改什么？

Tools？

Prompt？

Model？

Session？

Events？

UI？

Commands？

---

## Isolation

Extension 是否有权限边界？

---

## Archify

生成：

### Diagram 20A：Extension Architecture

### Diagram 20B：Extension Lifecycle

---

# Batch 21：Skill / Instruction System

如果存在 Skill 或类似机制，分析：

Skill definition

discovery

loading

activation

prompt injection

tool association

---

## 重点回答

Skill 是：

静态 Prompt？

动态 Prompt？

工具集合？

Workflow？

代码？

---

## Skill 什么时候进入 Context？

---

## Skill 是否会增加大量 Token？

如果可以从源码确认其 Prompt 注入方式，分析 Token 影响机制。

---

## Archify

生成：

### Diagram 21：Skill Lifecycle

如果项目没有 Skill，明确说明并跳过绘图。

---

# Batch 22：MCP

如果 Pi 支持 MCP：

研究：

MCP Client

Server Config

Discovery

Transport

Tool Discovery

Tool Registration

Execution

Result

---

## 最重要的问题

MCP Tool 最终是否被转成普通 Pi Tool？

---

## 调用链

Model

↓

Pi Tool Call

↓

MCP Adapter

↓

MCP Client

↓

MCP Server

↓

Result

↓

Pi

↓

Model

---

## Archify

生成：

### Diagram 22A：MCP Architecture

### Diagram 22B：MCP Tool Sequence

---

# Batch 23：Event System

寻找：

Event

Emitter

Observer

Subscription

Hook

Stream

---

## 找出核心事件

例如：

session start

message

model request

model response

tool start

tool end

error

usage

---

## Event Consumers

TUI？

Extensions？

Logger？

Telemetry？

---

## Archify

生成：

### Diagram 23：Event Flow

---

# Batch 24：Concurrency

研究：

async / await

Promise

worker

thread

process

queue

---

## 分析

Model Stream 和 Tool 是否异步？

多个 Tool 是否并行？

多个 Session 是否并发？

Extension 是否并发？

---

## Cancellation

AbortController？

Signal？

其他机制？

---

## Archify

如果机制足够复杂：

### Diagram 24：Concurrency Architecture

---

# Batch 25：CLI / TUI

专门研究用户交互层。

---

## CLI

command

flags

interactive

non-interactive

stdin

stdout

---

## TUI

如果存在：

rendering

input

state

events

runtime communication

---

## 关键问题

CLI/TUI 是否只是 View？

还是其中存在大量 Agent 业务逻辑？

---

## Runtime Separation

能否在没有 CLI/TUI 的情况下直接调用 Pi Core？

---

## Archify

生成：

### Diagram 25A：CLI / Runtime Architecture

### Diagram 25B：Interactive Request Sequence

---

# Batch 26：Commands / Shortcuts

如果 Pi 存在：

slash commands

command palette

custom command

分析：

Command 注册

解析

dispatch

handler

extension command

---

# Batch 27：Config

研究所有配置来源：

CLI

Environment

Config File

Project Config

User Config

Session Config

Extension Config

---

## Config Priority

必须从源码得出真正优先级。

例如：

CLI

>

Project

>

User

>

Environment

>

Default

不要假设。

---

## Model Config

模型参数在哪里配置？

---

## Provider Config

API Key 从哪里来？

---

## Archify

生成：

### Diagram 27：Configuration Data Flow

---

# Batch 28：Error Handling

研究：

Model Error

Network Error

Rate Limit

Context Overflow

Invalid Response

Tool Error

Shell Error

File Error

Extension Error

MCP Error

---

## Error Propagation

错误：

Throw？

Result？

Event？

Message？

---

## User Visibility

什么错误直接显示？

什么错误被 Agent 自己处理？

---

# Batch 29：Retry / Recovery

研究：

Model Retry

Provider Retry

Tool Retry

Network Retry

---

## Retry Policy

次数

backoff

条件

---

## Recovery

Session Resume

Conversation Recovery

Checkpoint

Fallback Provider

Fallback Model

---

## Archify

生成：

### Diagram 29：Retry / Recovery Lifecycle

---

# Batch 30：Token / Usage / Cost

分析：

Input Tokens

Output Tokens

Cached Tokens

Reasoning Tokens

Context Tokens

---

## Token Counting

Client-side？

Provider 返回？

---

## Cost

Pi 是否计算费用？

---

## Budget

是否存在：

max tokens

max cost

max turns

---

## Context Usage

是否展示 Context 百分比？

数据从哪里计算？

---

# Batch 31：Cache

搜索：

cache

prompt cache

context cache

provider cache

---

## 分析

Pi 自身是否实现 Cache？

还是完全依赖 Provider？

---

## Anthropic / OpenAI 等 Prompt Caching

如果存在 Provider 特殊实现，深入分析。

---

# Batch 32：Model Selection / Switching

分析：

用户如何切换模型？

Session 中间是否可以切换？

---

## 切换模型以后

原 Conversation 是否直接重新发送？

Message Format 是否转换？

Reasoning Block 如何处理？

Tool Call 如何处理？

---

这是研究跨模型 Harness 非常重要的一部分。

---

# Batch 33：Model Compatibility Layer

重点找：

normalization

compatibility

capability

transform

adapter

---

## 分析

Pi 是否有一个真正的：

Canonical Message Model

Canonical Tool Model

Canonical Reasoning Model

Canonical Usage Model

---

如果存在：

说明这可能是 Pi 很核心的设计。

---

## Archify

生成：

### Diagram 33：Cross-Provider Compatibility Layer

---

# Batch 34：Tests

分析：

Unit Test

Integration

E2E

Provider Test

Tool Test

Session Test

Extension Test

Agent Test

---

## LLM Test

是否调用真实模型？

还是 Mock？

---

## Tool Tests

重点看核心 Tool 测试。

---

## Archify

如果体系复杂：

### Diagram 34：Testing Architecture

---

# Batch 35：Agent Eval / Benchmark

寻找：

eval

benchmark

fixture

golden

scenario

---

## 分析

Pi 是否真正衡量：

Coding Ability

Tool Usage

Task Completion

Regression

---

如果没有正式 Eval 系统，明确说明。

---

# Batch 36：核心抽象

最终找出 Pi 最重要的 10～20 个核心抽象。

例如实际可能包括：

Agent

Session

Message

Model

Provider

Tool

Context

Extension

Event

Workspace

但必须根据源码。

---

## 每个对象说明

职责

生命周期

Owner

Dependencies

Data

调用关系

---

# Batch 37：设计模式

识别真实使用：

Adapter

Factory

Strategy

Registry

Observer

Command

State

Dependency Injection

Plugin Architecture

---

不要为了列模式而套模式。

---

# Batch 38：Technical Debt

从源码发现：

God Object

High Coupling

Provider-specific hacks

Global State

Duplicated Conversion Logic

Prompt Coupling

UI Coupling

Hidden Side Effects

Complex Async

Weak Error Handling

Test Gaps

---

每一项尽量提供代码证据。

---

# Batch 39：扩展性

分别模拟：

## 增加一个 Provider

需要改哪里？

## 增加一个模型

需要改哪里？

## 增加 Tool

需要改哪里？

## 增加 Extension

需要改哪里？

## 修改 Agent Loop

需要改哪里？

## 修改 Context Compression

需要改哪里？

## 增加 MCP

需要改哪里？

## 修改 Shell

需要改哪里？

## 修改 File Edit

需要改哪里？

---

# Batch 40：Pi 真正的核心竞争力

不要参考宣传语。

只根据源码回答：

Pi 哪些代码只是：

API wrapper

CLI

UI

configuration

glue

而哪些代码是真正的：

Harness Logic

Agent Runtime

Context Management

Cross-provider Abstraction

Tool Runtime

Extension Runtime

---

## 最重要的问题

如果直接用 OpenAI / Anthropic / Gemini SDK，

重新实现 Pi，

最难重新实现的是哪些部分？

---

# Batch 41：Minimal Core

假设删除 Pi 80% 的代码。

只保留最核心 20%。

列出：

必须留下的模块。

---

输出：

# Pi Minimal Core

例如：

Agent Loop

+

Canonical Message Model

+

Context

+

Model Adapter

+

Tool Runtime

但必须根据实际 Pi 源码。

---

## 同时列出

对应：

文件

Class

Function

---

# Batch 42：关键源码文件

找出：

# Pi 最值得阅读的 30 个源码文件

按照：

“理解整个 Pi 的价值”

排序。

不是按照代码量。

---

每个文件说明：

1. 路径
2. 作用
3. 核心 Class / Function
4. 为什么重要
5. 与哪些模块相关

---

# Batch 43：阅读路线

生成四个版本。

## 30 分钟路线

只理解项目本质。

## 2 小时路线

理解核心 Agent Runtime。

## 1 天路线

理解整个 Pi。

## 3 天路线

达到可以二次开发 Pi 的水平。

---

# Batch 44：关键调用链

整理至少以下调用链。

---

## 1. Startup

Entry

↓

Runtime

↓

Session

↓

Agent

---

## 2. User Input

User

↓

CLI/TUI

↓

Session

↓

Agent

---

## 3. Context Build

Message

↓

Context Manager

↓

Model Context

---

## 4. Prompt

Instructions

↓

Prompt

↓

Request

---

## 5. Model Call

Agent

↓

Model

↓

Provider

↓

API

---

## 6. Streaming

Provider

↓

Pi

↓

UI

---

## 7. Tool

Model

↓

Tool Registry

↓

Tool

---

## 8. Tool Result

Tool

↓

Message

↓

Context

↓

Model

---

## 9. File Read

Agent

↓

Read Tool

↓

Filesystem

---

## 10. File Edit

Agent

↓

Edit Tool

↓

Filesystem

---

## 11. Shell

Agent

↓

Shell Tool

↓

Process

---

## 12. Context Compression

Context

↓

Compaction

↓

New Context

---

## 13. Extension

Startup

↓

Extension Load

↓

Registration

↓

Runtime

---

## 14. MCP

Model

↓

Tool

↓

MCP

↓

Server

---

## 15. Final Answer

Agent Loop

↓

Completion

↓

Session

↓

UI

---

每条链必须尽量精确到：

`Class.method()`

并附真实源码路径。

---

# Batch 45：二次开发地图

生成：

# Pi Development Map

直接告诉开发者：

修改 Agent Loop → 哪里

修改 System Prompt → 哪里

修改 Context → 哪里

修改 Compression → 哪里

增加 Provider → 哪里

增加模型 → 哪里

修改 Message Conversion → 哪里

修改 Tool Call → 哪里

增加 Tool → 哪里

修改 Read Tool → 哪里

修改 Edit Tool → 哪里

修改 Shell Tool → 哪里

修改 Session → 哪里

增加 Extension → 哪里

修改 MCP → 哪里

修改 Config → 哪里

修改 Events → 哪里

修改 Token Usage → 哪里

修改 Streaming → 哪里

修改 Reasoning → 哪里

---

# Batch 46：最终 Master Map

完成前面所有分析以后，

使用 Archify 重新生成最终：

# Pi Agent Master Architecture Map

---

这张图只保留真正核心的 15～25 个节点。

建议重点体现：

User

↓

CLI / TUI

↓

Session

↓

Agent Runtime

↓

Context / Prompt

↓

Canonical Message / Model Layer

↓

Provider

↓

LLM

同时：

Agent

↓

Tool Runtime

↓

File / Shell / Search / MCP

同时展示：

Extension

Events

Persistence

Configuration

Workspace

之间的重要关系。

---

不要把普通 helper 和 utility 塞进去。

---

# Batch 47：最终核心图集

最终至少应该得到：

1. Pi Repository Map
2. Pi Runtime Architecture
3. Startup Lifecycle
4. Agent Runtime Workflow
5. Agent Loop Lifecycle
6. Agent Loop State Machine
7. Message Data Flow
8. Prompt Composition
9. Context Data Flow
10. Context Compaction Lifecycle
11. Model Architecture
12. Provider Architecture
13. LLM Request Sequence
14. Tool Architecture
15. Tool Call Sequence
16. File Edit Sequence
17. Shell Lifecycle
18. Repository Understanding Workflow
19. Coding Task Workflow
20. Session Lifecycle
21. State Architecture
22. Extension Architecture
23. MCP Architecture
24. Event Flow
25. Configuration Flow
26. Retry / Recovery Lifecycle
27. Cross-provider Compatibility Layer
28. Pi Agent Master Architecture Map

某个能力不存在时：

明确写：

`Not Present`

不要为了凑图虚构。

---

# Batch 48：最终研究报告

最终生成一份：

# 《Pi Agent 源码深度研究报告》

结构：

## 1. Pi 到底是什么

## 2. Pi 的设计哲学

从源码推导，不照抄宣传。

## 3. Runtime

## 4. Agent Loop

## 5. Message Model

## 6. Prompt

## 7. Context

## 8. Context Compression

## 9. Model Abstraction

## 10. Provider

## 11. Cross-provider Compatibility

## 12. Reasoning / Thinking

## 13. Tool Runtime

## 14. Core Tools

## 15. File Editing

## 16. Shell

## 17. Repository Understanding

## 18. Coding Workflow

## 19. Session

## 20. State

## 21. Persistence

## 22. Memory

## 23. Extension

## 24. Skill

## 25. MCP

## 26. Events

## 27. Concurrency

## 28. CLI / TUI

## 29. Configuration

## 30. Error / Retry

## 31. Token / Cost

## 32. Cache

## 33. Testing / Eval

## 34. Core Abstractions

## 35. Design Patterns

## 36. Technical Debt

## 37. Extension Points

## 38. Minimal Core

## 39. 30 Core Files

## 40. Reading Path

## 41. Development Map

## 42. Critical Call Chains

## 43. Pi Master Architecture

---

# 每个 Batch 固定输出格式

每次分析统一输出：

## Findings

这一批次的核心发现。

## Evidence

源码证据。

## Key Source Files

关键源码。

## Key Types / Classes

核心类型。

## Key Functions

核心函数。

## Control Flow

谁调用谁。

## Data Flow

数据如何传递。

## State

涉及哪些状态。

## Design Decisions

为什么这样设计。

## Unknowns

尚未确认的问题。

## Archify Diagram

生成本 Batch 对应图。

## Follow-up

下一 Batch 应继续验证什么。

---

# 最高优先级

如果分析工作量过大，

优先保证下面这些内容完全分析清楚：

1. Agent Loop
2. Message Model
3. Context Management
4. Context Compression
5. Model Abstraction
6. Provider Adapter
7. Tool Runtime
8. File Edit
9. Shell
10. Repository Understanding
11. Session
12. Extension
13. MCP
14. CLI / TUI

---

# 特别关注：Pi 作为 Harness 的价值

整个分析过程中始终追踪一个问题：

> Pi 的能力到底来自模型本身，还是来自 Pi Harness？

对于每个关键能力尽量判断：

### Model capability

模型自身完成。

### Harness capability

Pi Runtime 实现。

### Hybrid

模型 + Pi 协作实现。

例如：

代码搜索

任务规划

文件选择

Tool 调用

错误恢复

Context 压缩

模型切换

Tool Result 处理

代码修改

都应该尝试进行这个判断。

---

# 最终最重要的五个结论

报告最后必须单独回答以下五个问题。

## Q1

Pi 真正的 Agent Loop 是哪段代码？

## Q2

Pi 真正的 Context Management 是哪段代码？

## Q3

Pi 针对不同模型究竟做了多少 Harness 层适配？

## Q4

Pi 和“LLM API + Shell + File Tools”的简单 Agent 相比，真正多出来了什么？

## Q5

如果我要自己重新实现一个 Mini Pi，最少需要实现哪些模块？

这些问题必须基于前面的源码调查回答。