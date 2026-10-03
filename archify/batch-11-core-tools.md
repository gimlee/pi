# Batch 11 · Core Tools

## Findings

**Confirmed**：内置文件/搜索/Shell 工厂共八种：read、write、edit、bash、powershell、grep、find、ls。实际默认声明由 SDK、Settings 和 Session 决定，不能把工厂导出集合当默认工具集合。`createCodingTools()` 的四工具集合为 read/bash/edit/write；产品还支持扩展提供的 Codemode、tool_search、MCP 和资源工具。

| 工具 | 参数 / 执行 | 返回及限制 | 能力归属 |
|---|---|---|---|
| read | path，offset，limit；读文本或识别图片 | 文本头部 2,000 行 / 50 KiB，图片块，后续读取提示 | Harness |
| write | path，content；创建父目录并 UTF-8 覆盖 | 成功消息；文件串行队列 | Harness |
| edit | path，edits 的 oldText/newText | 精确优先、模糊回退；唯一匹配；diff 在 details | Hybrid |
| bash | command，timeout 可选 | stdout/stderr 合并，尾部 2,000 行 / 50 KiB；大输出临时文件 | Harness |
| powershell | 与 Shell 共用执行机制 | Windows 专用配置，UTF-8 前缀 | Harness |
| grep | pattern，path，glob，ignoreCase，literal，context，limit | 默认 100 匹配；每行 500 字符；50 KiB | Hybrid |
| find | pattern，path，limit | fd glob，默认 1,000 结果 / 50 KiB | Hybrid |
| ls | path，limit | 排序目录列表，默认 500 项 / 50 KiB | Harness |

## Evidence

[全部工厂与集合](../packages/coding-agent/src/core/tools/index.ts)、[SDK 选择](../packages/coding-agent/src/core/sdk.ts#L260)、[read](../packages/coding-agent/src/core/tools/read.ts)、[write](../packages/coding-agent/src/core/tools/write.ts)、[edit](../packages/coding-agent/src/core/tools/edit.ts)、[bash](../packages/coding-agent/src/core/tools/bash.ts)、[powershell](../packages/coding-agent/src/core/tools/powershell.ts)、[grep](../packages/coding-agent/src/core/tools/grep.ts)、[find](../packages/coding-agent/src/core/tools/find.ts)、[ls](../packages/coding-agent/src/core/tools/ls.ts)。这些文件已通读。

## Key Source Files

除以上工具外：[truncate.ts](../packages/coding-agent/src/core/tools/truncate.ts)、[output-accumulator.ts](../packages/coding-agent/src/core/tools/output-accumulator.ts)、[path-utils.ts](../packages/coding-agent/src/core/tools/path-utils.ts)。

## Key Types / Classes

ToolDefinition、各工具的 Input/Options/Operations/Details、TruncationResult、OutputAccumulator。Operations 可注入不同后端，但 grep 的 rg 执行仍在本地，不能仅替换 stat/read 就声称完全远程搜索。

## Key Functions

`create*ToolDefinition()` 负责定义；`create*Tool()` 经统一 wrapper 暴露 AgentTool；`truncateHead/Tail/Middle()` 与 `OutputAccumulator` 管输出。

## Control Flow

Session 选择定义 → 工具管线验证 → 解析路径/操作 → 截断与 details → 统一结果。每个工具的错误交给执行管线回灌，不会自动要求模型重试。

## Data Flow

路径相对 session cwd 解析，也支持绝对路径及 `~`；结果从 OS I/O 变成模型 text/image 与独立 UI details。grep 尊重 rg ignore 机制；ls 不执行 gitignore 过滤。

## State

工具 Options、当前 cwd、文件变更队列、Shell 输出累积器和执行信号；不存在所有工具共用的持久索引。

## Design Decisions

**Inference**：小而具体的工具让模型自己组织任务；限制输出避免单次结果挤满上下文。工具本身没有 workspace 路径封锁，权限范围来自 OS 与上层策略。

## Unknowns

未调用工具执行命令，未下载 rg/fd；各平台二进制探测与外部文件系统行为没有运行验证。

## Archify Diagram

复用 [10A](diagrams/diagram-10a-tools/diagram-10a-tools.html)，文件和 Shell 细节见 Batch 12、13。

## Follow-up

明确文件替换的安全边界以及 Shell 生命周期。
