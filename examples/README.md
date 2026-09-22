# Client configuration examples

Replace `avmcp_<your-key>` with the key from your AlgoVesta panel (**MCP Connection** tab). New keys default to the `paper` scope — a $5,000 virtual balance with the full toolset.

| Client | File | Where it goes |
|---|---|---|
| Claude (web, desktop, iOS, Android) | [claude.md](claude.md) | No file — added in the Connectors UI |
| Claude Code | [claude-code.md](claude-code.md) | One CLI command |
| Cursor | [cursor/mcp.json](cursor/mcp.json) | `.cursor/mcp.json` in your project, or Cursor's MCP settings |
| VS Code (Copilot) | [vscode/mcp.json](vscode/mcp.json) | `.vscode/mcp.json` in your workspace |
| ChatGPT (paid plans) | [chatgpt.md](chatgpt.md) | No file — added as a custom connector in developer mode |
| Gemini CLI | [gemini-cli/settings.json](gemini-cli/settings.json) | `~/.gemini/settings.json` |
| Anything else | [other-clients.md](other-clients.md) | The three config shapes, plus a `curl` connection check |

The [README](../README.md#supported-ai-clients) lists the clients we know document remote MCP support, each linked to its own configuration docs.
