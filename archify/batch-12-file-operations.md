# Batch 12 · File Read / Edit / Write

## Findings

**Confirmed**：read 是文本/图片读取；edit 是文本块替换，既不是 AST 重写，也不是应用模型提供的 unified diff。write 是完整覆盖。Harness 检查唯一匹配、重叠和无变化，模型仍负责选择文件及修改内容。

例：同一个 oldText 在文件内出现两次时 edit 报错，要求更具体上下文；多个 edits 都针对同一原始内容定位，再按位置逆序替换，前一个替换不会成为后一个匹配的输入。

## Evidence

[read 定义与执行](../packages/coding-agent/src/core/tools/read.ts#L66)、[edit 参数兼容及执行](../packages/coding-agent/src/core/tools/edit.ts#L103)、[匹配与替换](../packages/coding-agent/src/core/tools/edit-diff.ts#L300)、[文件队列](../packages/coding-agent/src/core/tools/file-mutation-queue.ts#L32)、[write](../packages/coding-agent/src/core/tools/write.ts)。

## Key Source Files

[edit-diff.ts](../packages/coding-agent/src/core/tools/edit-diff.ts)、[path-utils.ts](../packages/coding-agent/src/core/tools/path-utils.ts)、[read.ts](../packages/coding-agent/src/core/tools/read.ts)、[edit.ts](../packages/coding-agent/src/core/tools/edit.ts)、[write.ts](../packages/coding-agent/src/core/tools/write.ts)。

## Key Types / Classes

Edit、AppliedEditsResult、EditToolDetails、ReadOperations、EditOperations、WriteOperations。diff/patch/firstChangedLine 是展示与诊断返回，不是自动测试成功证据。

## Key Functions

`resolveReadPath()`；`fuzzyFindText()`；`normalizeForFuzzyMatch()`；`applyEditsToNormalizedContent()`；`applyReplacementsPreservingUnchangedLines()`；`generateUnifiedPatch()`；`withFileMutationQueue()`。

## Control Flow

Read：路径 → access → 图片 magic 检测或 UTF-8 decode → offset/limit → 截断 → 返回。Edit：参数 prepare → 路径队列 → access/read → 去 BOM、统一 LF → 精确/模糊唯一定位 → 拒绝重叠 → 替换 → 恢复换行/BOM → write → diff。Write：队列 → mkdir → write。

## Data Flow

read 的 offset 按一基行号，越界报错；过长第一行会返回提示而非超限正文。普通二进制没有通用拒绝策略，UTF-8 解码可能失真。图片经过模型相关处理，最终能否发送还受请求转换和 blockImages 控制。

edit 的模糊匹配包括 Unicode NFKC、行尾空白、引号/横线/空格归一；若用模糊基线，未修改行仍尽量保留原始字节表现。整体恢复原检测的 LF/CRLF。

## State

进程内全局 per-file 队列按 realpath（不存在时绝对路径）分组，write/edit 同文件串行；不同文件允许并发。队列持续到 I/O 完成，不因调用方先取消就提前放锁。

## Design Decisions

**Inference**：唯一匹配与同文件串行减少误写，但不提供跨进程事务。实现没有写前版本/hash 的 compare-and-swap；其他编辑器在 read/write 之间更新文件仍可被覆盖。取消发生在 write 后时可能已写入却返回取消，没有回滚承诺。

## Unknowns

硬链接别名、外部写竞争及断电原子性未实验。未发现内置 AST 编辑或语法验证管线；扩展可另行实现，不能据此断言生态不存在。

## Archify Diagram

[12A 读取顺序](diagrams/diagram-12a-read/diagram-12a-read.html)、[12B 编辑顺序](diagrams/diagram-12b-edit/diagram-12b-edit.html)、[12C 编辑数据流](diagrams/diagram-12c-data/diagram-12c-data.html)。

## Follow-up

Batch 13 追踪进程取消，Batch 15 区分文件写成功和任务验证成功。
