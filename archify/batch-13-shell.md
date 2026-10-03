# Batch 13 · Shell Lifecycle

## Findings

**Confirmed**：Shell 工具是一次命令执行及结果收集，没有返回可续接的后台任务 handle。timeout 参数可选，没有工具默认超时。服务/watch 命令会持续占用调用，除非自身退出、外部取消、指定超时或由 Shell 命令显式放到后台。

Harness 管 cwd、环境、输出上限、进程树终止和事件；模型决定执行什么以及如何解释错误。不存在“所有 bash 命令成功即任务完成”的判定。

## Evidence

[本地执行](../packages/coding-agent/src/core/tools/bash.ts#L97)、[参数与返回](../packages/coding-agent/src/core/tools/bash.ts#L243)、[Shell 选择](../packages/coding-agent/src/utils/shell.ts)、[子进程等待](../packages/coding-agent/src/utils/child-process.ts)、[输出累积](../packages/coding-agent/src/core/tools/output-accumulator.ts)、[PowerShell](../packages/coding-agent/src/core/tools/powershell.ts)。

## Key Source Files

以上六文件均已通读；[AgentSession.executeBash](../packages/coding-agent/src/core/agent-session.ts#L3792) 是用户命令入口，与模型调用工具入口不同。

## Key Types / Classes

BashOperations、BashSpawnContext、BashToolOutput、OutputAccumulator、ShellConfig、AbortSignal。

## Key Functions

`createLocalShellOperations()`、`resolveSpawnContext()`、`waitForChildProcess()`、`killProcessTree()`、`createShellToolDefinition()`。

## Control Flow

校验 timeout → spawn hook/commandPrefix → cwd 检查 → spawn → stdout/stderr 回调 → 更新事件/累积 → exit/输出收尾 → 返回；取消或超时走 killProcessTree → 捕获尾部输出 → 错误结果。

## Data Flow

继承 process.env 并更新 PATH；清理继承的 PI_SESSION_*、provider/model/reasoning 后按选项写本会话元数据。stdout/stderr 合并，TextDecoder 处理跨 chunk 字符。模型正文保留尾部 2,000 行 / 50 KiB；完整大输出落临时文件。程序结构化输出上限为 1 MiB，不能当模型实际看到了全文。

## State

PID、tracked process、abort listener、timeout timer、输出文件/缓冲。Unix detached 进程组以 SIGKILL 终止，Windows 通过 taskkill /T /F；工具没有 PTY 或持续 stdin 对话接口。旧 WSL bash 使用 stdin 传命令是传输方式例外。

## Design Decisions

**Confirmed**：waitForChildProcess 可在 exit 后等待输出安静 100ms；若后台后代持续输出，会继续延后。不是从 exit 开始固定 100ms 强制完成。

**Inference**：尾部对测试/编译失败有用，完整文件让模型进一步读取；这仍需模型主动调用 read。Shell 本身具有 OS 权限，权限扩展及隔离后端属于额外策略。

## Unknowns

没有启动进程、验证 Windows taskkill、编码或脱离进程行为。远程 BashOperations 的终止保证取决于后端。

## Archify Diagram

[13 Shell 生命周期](diagrams/diagram-13-shell/diagram-13-shell.html)。

## Follow-up

Batch 14～15 说明搜索/测试如何由模型组织。
