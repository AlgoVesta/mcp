# ChatGPT

**Requires a paid ChatGPT plan.** Custom MCP connectors are not offered on the free plan, and developer mode may need to be enabled first — both are OpenAI restrictions that apply to every MCP server, not only this one.

1. Enable **Developer mode** (Settings → Security and login) if it is not already on.
2. Add a custom connector with your secret link:

   ```
   https://api.algovesta.com/u/avmcp_<your-key>/mcp
   ```

3. Transport is Streamable HTTP; no additional headers are needed — the key travels in the URL path.

Start on a `paper`-scope key: the full toolset runs against a $5,000 virtual balance, so you can rehearse the workflow before any real money is reachable.
