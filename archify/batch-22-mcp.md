# Batch 22 · MCP Integration

## Findings

**Confirmed**：此仓库 MCP 是实际实现：编码层内置扩展负责注册/发现/曝光/管理，pi-mcp 包负责 JSON-RPC client、stdio 与 Streamable HTTP。不支持 legacy SSE 配置。默认 exposure=codemode，另有 deferred/direct/hidden 和单工具覆盖；direct 首次 prompt 等连接最多默认 10s，间接工具在调用时等相关 servers。

服务器文件配置优先于扩展同名注册；可信项目 mcp.json 替换全局同名项。项目配置禁止 auth.provider；OAuth 也不等同模型 Provider OAuth。


版本边界：以上配置结论对应固定研究基线。当前 HEAD 新增可信项目 enabled/exposure/toolExposure 的局部覆盖分支，见 [基线推进说明](baseline-drift.md)。

## Evidence

- [index.ts](../packages/coding-agent/src/extensions/mcp/index.ts)：session_start / before_agent_start / tool_call / shutdown。
- [runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts)：McpServerConnection。
- [tools.ts](../packages/coding-agent/src/extensions/mcp/tools.ts)：createMcpToolDefinition / convertMcpResult。
- [config.ts](../packages/coding-agent/src/extensions/mcp/config.ts)：配置优先级与项目鉴权限制。
- [client.ts](../packages/mcp/src/client.ts)：initialize / listTools / callTool。
- [stdio.ts](../packages/mcp/src/transports/stdio.ts)：传输与进程关闭。
- [mcp-servers.ts](../packages/coding-agent/src/core/mcp-servers.ts)：注册验证与 exposure。

## Key Source Files

[packages/coding-agent/src/extensions/mcp/index.ts](../packages/coding-agent/src/extensions/mcp/index.ts)、[packages/coding-agent/src/extensions/mcp/runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts)、[packages/coding-agent/src/extensions/mcp/tools.ts](../packages/coding-agent/src/extensions/mcp/tools.ts)、[packages/coding-agent/src/extensions/mcp/config.ts](../packages/coding-agent/src/extensions/mcp/config.ts)、[packages/mcp/src/client.ts](../packages/mcp/src/client.ts)、[packages/mcp/src/transports/stdio.ts](../packages/mcp/src/transports/stdio.ts)、[packages/coding-agent/src/core/mcp-servers.ts](../packages/coding-agent/src/core/mcp-servers.ts)。

## Key Types / Classes

McpServerRegistry、McpServerConnection、McpClient、McpTransport、CallToolResult、McpExposure。

## Key Functions

createMcpExtension；loadMcpConfig；getClient/connectOnce；McpClient.connect/listTools/callTool；createMcpToolDefinition；convertMcpResult。

## Control Flow

配置/注册 → background connect → initialize/initialized → listTools → 注册 pi tool → 普通管线/hook → connection.callTool → client.request tools/call → transport → server → 结果转换。list_changed 刷新定义；撤回工具以 hidden 重新注册。

## Data Flow

MCP schema 转 ToolDefinition；名称 sanitize/哈希限制 64 字符；模型 text 超 20 KiB 中间截断并保存全文，图片保留；Codemode 获得未经模型截断的 CallToolResult（去顶层 _meta），isError 可保留结构化数据。audio 转提示，binary resource 可存文件。

## State

connecting/connected/disconnected/needs-auth/failed/closed；opening 去重、generation 防旧会话发布、pending 请求 id/progressToken、默认 server timeout 60s，progress 重置超时。

## Design Decisions

**Confirmed**：普通工具调用不对瞬时 HTTP 失败重放，避免重复副作用；readOnly 请求可一次重试；McpSessionExpiredError 特例重建 session 再试一次（实现假定服务端未执行）。取消发 notification 并本地拒绝，不保证远端已撤销副作用。

## Unknowns

未连接服务器、登录或执行 conformance；HTTP/OAuth 全部边界没有动态验证。

## Archify Diagram

[diagram-22a-mcp](diagrams/diagram-22a-mcp/diagram-22a-mcp.html)、[diagram-22b-mcp-call](diagrams/diagram-22b-mcp-call/diagram-22b-mcp-call.html)

## Follow-up

Batch 23～24 跟踪事件与并发。
