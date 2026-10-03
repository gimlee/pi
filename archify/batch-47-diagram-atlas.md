# Batch 47 · Core Diagram Atlas

## Findings

至少28个要求的概念均由对应图覆盖；完整映射及每图自动校验状态见 [diagram-atlas.md](diagram-atlas.md)。额外机制图保留在diagrams。

## Evidence

来源绑定固定HEAD、每节点源码范围；[diagram-manifest.json](diagram-manifest.json)记录本轮图。

## Key Source Files

来源绑定固定HEAD、每节点源码范围；[diagram-manifest.json](diagram-manifest.json)记录本轮图。

## Key Types / Classes

architecture/workflow/sequence/dataflow/lifecycle五种图型。

## Key Functions

每个candidate经finalize执行validate/deliver/strict check/browser-check。

## Control Flow

图先由源码trace确定，再生成candidate；失败修复不改变未受影响语义。

## Data Flow

candidate→HTML→hash绑定delivery/browser receipts，不把旧receipt当新文件证据。

## State

各图当前sha与passing receipt；校验结果在analysis-checks.json。

## Design Decisions

**Inference**：atlas是阅读入口，图不能替代具体参数/错误边界报告。

## Unknowns

未做截图人工视觉评审；自动passing不证明人工视觉质量。

## Archify Diagram

[完整图集](diagram-atlas.md)；[Pi Agent Master Architecture](diagrams/diagram-46-master/diagram-46-master.html)

## Follow-up

最终研究报告给整体结论和五问。
