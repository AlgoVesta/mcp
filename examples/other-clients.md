# Any other MCP client

There is no AlgoVesta-specific setup. This is a remote MCP server over **Streamable HTTP**, so whatever your client calls a "remote server", "HTTP server", "custom connector" or "URL server" is the right field.

Two URLs, pick one:

| | URL | When |
|---|---|---|
| Secret link | `https://api.algovesta.com/u/avmcp_<your-key>/mcp` | Simplest. The URL itself is the credential. |
| OAuth 2.1 | `https://api.algovesta.com/mcp` | When your client can run an authorization flow. PKCE (`S256`) is mandatory; Dynamic Client Registration (RFC 7591) is supported, so most clients configure themselves. |

## The three config shapes you will meet

Almost every JSON-configured client uses one of these. Check your client's own MCP documentation for which one it expects — the [README](../README.md#which-ai-clients-can-use-this-server) links to that documentation for each client we know of.

```json
{ "mcpServers": { "algovesta": { "url": "https://api.algovesta.com/u/avmcp_<your-key>/mcp" } } }
```

```json
{ "mcpServers": { "algovesta": { "httpUrl": "https://api.algovesta.com/u/avmcp_<your-key>/mcp" } } }
```

```json
{ "servers": { "algovesta": { "type": "http", "url": "https://api.algovesta.com/u/avmcp_<your-key>/mcp" } } }
```

## Checking the connection without a client

The endpoint speaks plain MCP over HTTP, so you can talk to it with `curl`:

```bash
curl -s https://api.algovesta.com/u/avmcp_<your-key>/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"1"}}}'
```

A valid key returns the server info and capabilities. A revoked or mistyped key returns an authorization error rather than a partial session — there is no anonymous mode.

To see the tool list without any key at all, read the published schemas:

```bash
curl -s https://algovesta.com/mcp/tools.json | head
```

## If it still will not connect

- The URL is a credential. If it was pasted into a chat, a screenshot or a shared config file, revoke it in the panel and generate a new one.
- New keys are `paper` scope. That is not a failure — the tools are identical, they just fill against the $5,000 virtual balance until you issue a `live` key with a second factor.
- Some clients only support remote MCP in their CLI or desktop build, not on the web.

Still stuck? [Open an issue](../../../issues) with the client name and version, or write to [support@algovesta.com](mailto:support@algovesta.com). Never include your key.
