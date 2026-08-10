# Claude (web, desktop, iOS, Android)

Claude connects through **Connectors** — no config file needed. Custom connectors are available on Free, Pro, Max, Team and Enterprise plans.

1. Open **Customize → Connectors**, then **Add custom connector**. (Team/Enterprise owners: **Organization settings → Connectors**.)
2. Name: `AlgoVesta`.
3. URL: your secret link from the AlgoVesta panel:

   ```
   https://api.algovesta.com/u/avmcp_<your-key>/mcp
   ```

   Or use the OAuth endpoint and let Claude run the authorization flow (PKCE + Dynamic Client Registration — no manual client setup):

   ```
   https://api.algovesta.com/mcp
   ```

4. Ask: *"How is my portfolio?"* — Claude calls `get_portfolio_context` and shows every connected account.

Tool titles and confirmation dialogs come straight from the server; the write tools (`place_order`, `close_position`, `cancel_order`) are marked destructive, so Claude asks before executing them.
