# Claude Code

Add the server from the command line:

```bash
claude mcp add --transport http algovesta https://api.algovesta.com/u/avmcp_<your-key>/mcp
```

Verify:

```bash
claude mcp list
```

Then use it in a session — for example: *"Simulate a $200 long on ETHUSDT at 5x and show me the policy verdict."*

To use OAuth instead of a secret link:

```bash
claude mcp add --transport http algovesta https://api.algovesta.com/mcp
```

Claude Code will open the browser consent flow (PKCE S256, Dynamic Client Registration).
