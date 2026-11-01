# ChatGPT

Steps follow OpenAI's "Connect from ChatGPT" guide
(https://developers.openai.com/apps-sdk/deploy/connect-chatgpt, read 2026-11-01). OpenAI changes
this flow often; if a menu name below no longer matches, follow that page.

Account and workspace policies on OpenAI's side decide whether your ChatGPT account can add a
custom MCP server. That restriction applies to every MCP server, this one included.

1. Open https://chatgpt.com/plugins.
2. Select the **+** button, then **Add custom MCP server**.
3. Enter a name (for example `AlgoVesta`) and a short description.
4. Under **Connection**, choose the public endpoint and enter:

   ```
   https://api.algovesta.com/mcp
   ```

   Transport is Streamable HTTP.
5. Configure authentication as OAuth. ChatGPT discovers the authorization server from
   `https://api.algovesta.com/.well-known/oauth-protected-resource` and registers itself
   (dynamic client registration, PKCE S256).
6. Review the risk warning and select **I understand and want to continue**, then
   **Create as a plugin**.
7. A browser window opens the AlgoVesta consent page. Sign in again when asked and approve the
   **paper** scope.
8. ChatGPT lists the 20 tools it discovered. Start a new chat, type `@`, and pick the plugin.

## Secret link (alternative)

If you prefer not to use OAuth, create a key in the AlgoVesta panel (MCP page) and use the secret
link as the URL instead. The key travels in the URL path, so treat the link like a password:

```
https://api.algovesta.com/u/avmcp_<your-key>/mcp
```

## Getting started

Start on the `paper` scope: the full toolset runs against a $5,000 virtual balance, so you can
rehearse the workflow before any real money is reachable. Live trading needs a separate
live-scope key, which requires a second factor.

A suggested starting sequence: `get_portfolio_context`, then `simulate_order`, then a paper
`place_order`, then `verify_receipt` on the receipt it returns.
