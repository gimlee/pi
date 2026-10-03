# Batch 14 · Repository Understanding

## Findings

**Confirmed**：默认仓库理解依赖提示、工作目录说明与可执行工具，不是预先构建代码语义索引。模型选择搜索词、文件和局部读取范围，Harness 执行并限制输出；两者构成 Hybrid。

| 类别 | 当前内置证据 / 判断 |
|---|---|
| A 文件枚举 | ls/find；Shell 可执行 tree、git ls-files，调用由模型决定 |
| B 文本搜索 | grep 使用 rg；Shell 亦可运行 rg |
| C 文件阅读 | read 支持 offset/limit，图片和截断 |
| D Git 信息 | bash/powershell 可以执行 git；不是独立 Git 语义工具 |
| E 符号 / AST | 本轮检查的核心工具表和包依赖未见内置 LSP、AST 查询或 tree-sitter 工具：Not Present（限定默认内置管线） |
| F Embedding / RAG | 未见仓库 embedding 检索管线；tool_search 的 BM25 是搜索工具描述，不是源码语义索引 |
| G Repo Map | 无内置静态语义 repo-map 生成阶段；模型可通过工具自己建立理解 |

## Evidence

[工具集合](../packages/coding-agent/src/core/tools/index.ts)、[grep](../packages/coding-agent/src/core/tools/grep.ts)、[find](../packages/coding-agent/src/core/tools/find.ts)、[ls](../packages/coding-agent/src/core/tools/ls.ts)、[提示贡献](../packages/coding-agent/src/core/system-prompt.ts)、[tool_search](../packages/coding-agent/src/extensions/tool-search/tool.ts)。对 `packages/coding-agent/src` 与包 manifest 搜索 LSP/AST/tree-sitter/embedding/repomap，并交叉检查内置集合；搜索未命中本身不证明第三方能力不存在。

## Key Source Files

[resource-loader.ts](../packages/coding-agent/src/core/resource-loader.ts) 加载目录指令；read、grep、find 的源码链接见以上。

## Key Types / Classes

ResourceLoader、ToolDefinition、AgentMessage、ToolResultMessage。未发现默认仓库符号数据库类型。

## Key Functions

`createGrepToolDefinition()`、`createFindToolDefinition()`、`createLsToolDefinition()`、`createReadToolDefinition()`、`buildSystemPrompt()`。

## Control Flow

任务 + cwd/AGENTS → 模型选择搜索 → grep/find/ls → 模型选择 read → 内容回灌 → 后续搜索或修改。循环没有“仓库理解完成”的强制系统状态。

## Data Flow

目录指令和截断的工具结果进入消息历史，随后经过投影/压缩/转换形成下一次请求。不会自动把整个 repository 塞入每次上下文。

## State

当前 messages、操作 cwd、工具集合、模型内部推断；没有默认持久 embedding index。

## Design Decisions

**Inference**：这使 Harness 可跨语言工作，代价是文件选择与依赖关系判断依赖模型能力；索引可通过扩展/MCP 增强，并非必须塞进核心。

## Unknowns

第三方扩展或用户 Shell 中可安装 LSP/索引器；本轮没有枚举外部环境，也不评价模型实际检索准确率。

## Archify Diagram

[14A 仓库探索流程](diagrams/diagram-14a-search/diagram-14a-search.html)、[14B 上下文回灌](diagrams/diagram-14b-context/diagram-14b-context.html)。

## Follow-up

Batch 15 明确测试与修复路径的责任归属。
