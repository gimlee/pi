# 分析基线与仓库推进记录

研究和全部图证据固定在提交 `9b3c19da5cffc4c5e8b6bd74c45abc1ab6bfcd16`。最终检查时工作区为 `4196b29608898e61d77ef3d1f9a148eede45771f`，分支 `main`。本轮没有执行 commit、切换分支或更新代码；观察到的仓库推进来自本轮分析之外。

报告中的相对源码链接打开当前工作区；图节点范围和报告的基线行号对应旧提交。重新验证基线可用 `git show 9b3c19da5cffc4c5e8b6bd74c45abc1ab6bfcd16:<path>`。不能把旧图的范围当作最新提交逐行审计证据。

## 已核对的相关差异

本节只根据两个提交间的补丁标记差异，没有进行新版本全面审计或运行验证。

- MCP 配置增加可信项目的局部覆盖：没有 command/url/type 时，只能覆盖全局服务器的 enabled/exposure/toolExposure，保留其连接与鉴权配置。Batch 22/27 的“同名项目项替换”描述适用于原基线的完整服务器定义；新版本还支持该局部覆盖分支。见 [config.ts](../packages/coding-agent/src/extensions/mcp/config.ts)。
- MCP /mcp 管理增加项目启停与 override 文件保存路径；OAuth 配置新增 dcr/cimd 客户端注册选择及校验。见 [index.ts](../packages/coding-agent/src/extensions/mcp/index.ts)、[mcp-servers.ts](../packages/coding-agent/src/core/mcp-servers.ts)。
- Agent Loop、AgentSession、SessionManager、compaction、共享 transcript/transform、普通文件/Shell工具在两提交间未变，核心分析仍对应同一实现。
- 其他变化涉及模型目录生成、Bedrock/Cloudflare适配、Codemode、图像/TUI和OAuth；不将它们声称为本轮全部审计过的最新行为。

## 包内变化文件

- [packages/ai/CHANGELOG.md](../packages/ai/CHANGELOG.md)
- [packages/ai/README.md](../packages/ai/README.md)
- [packages/ai/scripts/generate-models.ts](../packages/ai/scripts/generate-models.ts)
- [packages/ai/scripts/hydrate-model-catalog.ts](../packages/ai/scripts/hydrate-model-catalog.ts)
- [packages/ai/scripts/model-data.ts](../packages/ai/scripts/model-data.ts)
- [packages/ai/src/api/bedrock-converse-stream.ts](../packages/ai/src/api/bedrock-converse-stream.ts)
- [packages/ai/src/api/cloudflare-workers-ai-system-one.ts](../packages/ai/src/api/cloudflare-workers-ai-system-one.ts)
- [packages/ai/src/auth/oauth/openai-chatgpt.ts](../packages/ai/src/auth/oauth/openai-chatgpt.ts)
- [packages/ai/test/bedrock-thinking-payload.test.ts](../packages/ai/test/bedrock-thinking-payload.test.ts)
- [packages/ai/test/cloudflare-workers-ai-system-one.test.ts](../packages/ai/test/cloudflare-workers-ai-system-one.test.ts)
- [packages/ai/test/model-data-validation.test.ts](../packages/ai/test/model-data-validation.test.ts)
- [packages/ai/test/stream.test.ts](../packages/ai/test/stream.test.ts)
- [packages/codemode/CHANGELOG.md](../packages/codemode/CHANGELOG.md)
- [packages/codemode/README.md](../packages/codemode/README.md)
- [packages/codemode/src/index.ts](../packages/codemode/src/index.ts)
- [packages/codemode/src/runtime/prelude-source.ts](../packages/codemode/src/runtime/prelude-source.ts)
- [packages/codemode/test/sandbox.test.ts](../packages/codemode/test/sandbox.test.ts)
- [packages/coding-agent/CHANGELOG.md](../packages/coding-agent/CHANGELOG.md)
- [packages/coding-agent/README.md](../packages/coding-agent/README.md)
- [packages/coding-agent/docs/cli.md](../packages/coding-agent/docs/cli.md)
- [packages/coding-agent/docs/codemode.md](../packages/coding-agent/docs/codemode.md)
- [packages/coding-agent/docs/mcp.md](../packages/coding-agent/docs/mcp.md)
- [packages/coding-agent/docs/models.md](../packages/coding-agent/docs/models.md)
- [packages/coding-agent/docs/quickstart.md](../packages/coding-agent/docs/quickstart.md)
- `packages/coding-agent/npm-shrinkwrap.json`（现提交已删除）
- [packages/coding-agent/package.json](../packages/coding-agent/package.json)
- [packages/coding-agent/src/config.ts](../packages/coding-agent/src/config.ts)
- [packages/coding-agent/src/core/mcp-servers.ts](../packages/coding-agent/src/core/mcp-servers.ts)
- [packages/coding-agent/src/extensions/mcp/cli.ts](../packages/coding-agent/src/extensions/mcp/cli.ts)
- [packages/coding-agent/src/extensions/mcp/config.ts](../packages/coding-agent/src/extensions/mcp/config.ts)
- [packages/coding-agent/src/extensions/mcp/index.ts](../packages/coding-agent/src/extensions/mcp/index.ts)
- [packages/coding-agent/src/extensions/mcp/oauth.ts](../packages/coding-agent/src/extensions/mcp/oauth.ts)
- [packages/coding-agent/src/extensions/mcp/runtime.ts](../packages/coding-agent/src/extensions/mcp/runtime.ts)
- [packages/coding-agent/src/modes/interactive/components/tool-execution.ts](../packages/coding-agent/src/modes/interactive/components/tool-execution.ts)
- [packages/coding-agent/src/modes/interactive/interactive-mode.ts](../packages/coding-agent/src/modes/interactive/interactive-mode.ts)
- [packages/coding-agent/src/package-manager-cli.ts](../packages/coding-agent/src/package-manager-cli.ts)
- [packages/coding-agent/src/utils/image-convert.ts](../packages/coding-agent/src/utils/image-convert.ts)
- [packages/coding-agent/test/image-processing.test.ts](../packages/coding-agent/test/image-processing.test.ts)
- [packages/coding-agent/test/mcp-extension.test.ts](../packages/coding-agent/test/mcp-extension.test.ts)
- [packages/coding-agent/test/mcp-oauth-refresh.test.ts](../packages/coding-agent/test/mcp-oauth-refresh.test.ts)
- [packages/coding-agent/test/suite/mcp-oauth-server.ts](../packages/coding-agent/test/suite/mcp-oauth-server.ts)
- [packages/coding-agent/test/tool-execution-component.test.ts](../packages/coding-agent/test/tool-execution-component.test.ts)
- [packages/evals/docker/entrypoint.ts](../packages/evals/docker/entrypoint.ts)
- [packages/mcp/CHANGELOG.md](../packages/mcp/CHANGELOG.md)
- [packages/mcp/src/oauth/callback.ts](../packages/mcp/src/oauth/callback.ts)
- [packages/mcp/src/oauth/flow.ts](../packages/mcp/src/oauth/flow.ts)
- [packages/mcp/src/oauth/index.ts](../packages/mcp/src/oauth/index.ts)
- [packages/mcp/src/oauth/provider.ts](../packages/mcp/src/oauth/provider.ts)
- [packages/mcp/test/oauth.test.ts](../packages/mcp/test/oauth.test.ts)
- [packages/tui/CHANGELOG.md](../packages/tui/CHANGELOG.md)
- [packages/tui/src/components/image.ts](../packages/tui/src/components/image.ts)
- [packages/tui/src/index.ts](../packages/tui/src/index.ts)
- [packages/tui/src/terminal-image.ts](../packages/tui/src/terminal-image.ts)
- [packages/tui/src/tui-alt-screen.ts](../packages/tui/src/tui-alt-screen.ts)
- [packages/tui/test/terminal-image.test.ts](../packages/tui/test/terminal-image.test.ts)
- [packages/tui/test/tui-alt-screen.test.ts](../packages/tui/test/tui-alt-screen.test.ts)
