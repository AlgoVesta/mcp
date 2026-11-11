---
name: algovesta-trading
description: Connect an AI coding agent to the hosted AlgoVesta MCP server (16 crypto exchanges and MetaTrader 5) and use it in the safe order - read first, simulate, trade on the $5,000 paper balance, and move to real money only through a live key protected by a second factor. Use when the user asks to check a trading portfolio, place or manage a crypto or MT5 order, set risk rules, or review trade history through AlgoVesta.
---

# AlgoVesta trading via MCP

AlgoVesta runs a hosted Model Context Protocol server. There is nothing to install or run locally: the agent connects to an HTTPS endpoint. This skill contains instructions only. It never runs shell commands and never asks for exchange API keys, passwords or MCP keys in chat.

## 1. Connect

Add the server to the agent's MCP configuration. Preferred endpoint (OAuth 2.1, PKCE S256, Dynamic Client Registration):

```
https://api.algovesta.com/mcp
```

The first connection opens a browser consent page on algovesta.com where the user signs in. The consent page defaults to the `paper` scope.

Alternative for clients without OAuth: a per-user secret link created by the user in the AlgoVesta panel (MCP Connection tab):

```
https://api.algovesta.com/u/avmcp_<your-key>/mcp
```

Treat that link as a password. Never print it, log it, commit it or paste it into a shared file. If the user pastes one into the conversation, tell them to revoke it in the panel and create a new one.

Client configuration examples: https://github.com/AlgoVesta/mcp/tree/main/examples

## 2. Know the scope before acting

Every key or OAuth grant has exactly one scope. The boundary between simulated and real money is a property of the key, not of a prompt or a setting.

| Scope | Can do |
|---|---|
| `read` | Portfolio, prices, history, simulations, policy previews. No order can be placed. |
| `paper` | Everything in `read`, plus orders on a $5,000 virtual balance. The default for new keys. |
| `live` | Real orders on the user's connected exchanges and MetaTrader 5 accounts. Creating a live key requires a second factor (TOTP or an email code); live over OAuth requires TOTP. |

If a tool returns `insufficient_scope`, report it. Do not try to work around it.

## 3. Safe order of operations

1. Read: `get_portfolio_context`, then `get_market_price` for the symbol.
2. Rules: if the user states risk limits, call `compile_policy`. It returns a preview only; activation is a separate step the user takes in the panel.
3. Simulate: `simulate_order` shows the expected fill, margin impact and the policy check result. It places nothing.
4. Paper: place the order with a `paper` key first and confirm the result with the user.
5. Live: only when the user explicitly asks for real money and the connection is a `live` key. Restate venue, symbol, side, size, stop-loss and take-profit, and wait for an explicit yes before calling `place_order`.

Rules that always apply:

- Sizing must be explicit: exactly one of `size_usd`, `margin_usd` or `risk_pct` on crypto, or `lots` on MetaTrader 5. Never invent a size.
- A stop-loss is mandatory; on forex/MT5 a take-profit is mandatory too. If neither the user nor the saved panel settings provide one, the server refuses the order. Report the refusal instead of guessing a level.
- Every order-shaped tool needs a client-generated idempotency key of at least 8 characters. Reuse the same key when retrying after a timeout so the order is not placed twice.
- Name the venue explicitly. The server never picks an exchange; `compare_venues` ranks connected exchanges on measured spread only and does not choose one.
- Strategy tools cannot arm real-money execution: `auto_trade` cannot be switched on through MCP.
- `close_position` and `cancel_order` reduce risk and are not blocked by the policy wall.
- If the user wants everything stopped, point them to the kill switch in the panel; after it, every tool returns `user_frozen`.

## 4. Report honestly

- If `positions_source` is `unavailable`, an empty position list means unknown, not "no positions". Say so.
- Fees, order book depth and slippage are not measured by `compare_venues`; do not describe the cheapest measured spread as the cheapest venue overall.
- Crypto PnL in `get_trade_history` is gross (commission is not recorded).
- AlgoVesta does not generate signals, does not hold or move client funds, and does not provide investment advice. Do not present any output as a recommendation.

Trading involves substantial risk of loss. Software, not advice.

## Reference

- Tool reference (20 tools, JSON Schemas): https://github.com/AlgoVesta/mcp/blob/main/docs/TOOLS.md
- Authentication and scopes: https://github.com/AlgoVesta/mcp/blob/main/docs/AUTHENTICATION.md
- Safety model: https://github.com/AlgoVesta/mcp/blob/main/docs/SAFETY.md
- Developer documentation: https://algovesta.com/mcp-docs.html
