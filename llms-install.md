# Installing the AlgoVesta MCP server (instructions for AI agents)

AlgoVesta is an MCP server for 16 crypto exchanges + MetaTrader 5. It is **hosted**: there is no
package to install, no process to start and no API key to put in an environment variable. Setup is
adding one remote Streamable HTTP endpoint to the client's MCP configuration and letting the user
authorize it.

## What you need from the user

Exactly one of these two:

1. **OAuth (recommended, nothing to copy).** The user only needs an AlgoVesta account. The client
   uses the endpoint `https://api.algovesta.com/mcp`; it discovers the authorization server, registers
   itself (RFC 7591 Dynamic Client Registration) and opens a browser consent page. PKCE `S256` is
   mandatory.
2. **Secret link.** The user creates a key in the AlgoVesta panel (`https://algovesta.com/app.html`,
   tab **MCP Connection**) and gives you a URL of the form
   `https://api.algovesta.com/u/avmcp_<key>/mcp`. That URL is a credential: do not print it back,
   log it, or commit it to a repository.

If the user has no account yet, send them to `https://algovesta.com/register.html` first. Do not
create an account for them.

## Step 1 — add the server

### Cline

Use the **Remote Servers** tab (Server Name `algovesta`, Server URL as above, Transport
**Streamable HTTP**), or edit the MCP settings JSON:

```json
{
  "mcpServers": {
    "algovesta": {
      "type": "streamableHttp",
      "url": "https://api.algovesta.com/mcp",
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Set `"type": "streamableHttp"` explicitly; without it Cline falls back to legacy SSE. Leave
`autoApprove` empty: the write tools can place real orders on a live key.

From the Cline CLI:

```bash
cline mcp add algovesta https://api.algovesta.com/mcp --transport http
```

### Claude (web, desktop, mobile)

Customize -> Connectors -> **Add custom connector**, name `AlgoVesta`, URL
`https://api.algovesta.com/mcp`. Claude runs the OAuth flow itself.

### Claude Code

```bash
claude mcp add --transport http algovesta https://api.algovesta.com/mcp
```

### Cursor

`~/.cursor/mcp.json` (or `.cursor/mcp.json` in a project):

```json
{
  "mcpServers": {
    "algovesta": { "url": "https://api.algovesta.com/mcp" }
  }
}
```

### ChatGPT

https://chatgpt.com/plugins -> **+** -> **Add custom MCP server** -> Connection: public endpoint
`https://api.algovesta.com/mcp` (Streamable HTTP, OAuth 2.1 + PKCE; the user approves on the AlgoVesta
consent page). Account and workspace policies on OpenAI's side decide who can add custom MCP servers.

### Any other MCP client

Use the same URL in whatever field the client calls a remote / HTTP / Streamable HTTP server. The
three config shapes in common use are listed in
[examples/other-clients.md](examples/other-clients.md).

## Step 2 — authorize

With the OAuth endpoint the client opens the consent page; the user signs in and approves. The
consent page grants the `paper` scope by default.

## Step 3 — verify

Call `get_portfolio_context` (read-only, no confirmation needed). A successful response lists the
user's connected exchange accounts, MetaTrader 5 accounts and the paper account. Then list tools:
there should be 20.

A `401` means the connection is not authorized yet: re-run the OAuth flow, or check that the secret
link was pasted completely.

## Scopes and money

| Scope | Effect |
|---|---|
| `read` | Portfolio, prices, history, simulations. No orders. |
| `paper` | **Default.** Orders run on a $5,000 virtual balance with the same 20 tools. |
| `live` | Real orders on the user's connected exchanges and MetaTrader 5 accounts. Issued only after a second factor (TOTP, or an e-mail code for panel-created keys; OAuth requires TOTP). |

Do not try to obtain the `live` scope on the user's behalf. Every order also passes a server-side
risk policy and returns an ed25519-signed receipt. AlgoVesta does not close a user's position on its
own initiative; the one automatic close is undoing an order it has just opened when the stop-loss
could not be placed on the exchange.

## References

- Full documentation: https://algovesta.com/mcp-docs.html
- Tool reference: [docs/TOOLS.md](docs/TOOLS.md)
- Authentication details: [docs/AUTHENTICATION.md](docs/AUTHENTICATION.md)
- Safety model: [docs/SAFETY.md](docs/SAFETY.md)
- Support: support@algovesta.com
