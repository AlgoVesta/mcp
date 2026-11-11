# AlgoVesta MCP server

This extension connects Gemini CLI to the hosted AlgoVesta MCP server at `https://api.algovesta.com/mcp` (OAuth 2.1). It reaches 16 crypto exchanges and MetaTrader 5. Nothing runs locally.

## Sign in

On first use, run `/mcp auth algovesta`. A browser opens the consent page on algovesta.com; the user signs in there. The consent page defaults to the `paper` scope. Never ask the user for passwords, exchange API keys or MCP secret links in chat.

## Scopes

- `read`: portfolio, prices, history, simulations, policy previews. No orders.
- `paper`: adds orders on a $5,000 virtual balance. The default.
- `live`: real orders on the user's connected accounts. Requires a second factor (TOTP for OAuth).

## Order of operations

1. Read with `get_portfolio_context` and `get_market_price`.
2. If the user states risk limits, `compile_policy` returns a preview only; the user activates it in the panel.
3. `simulate_order` before any order.
4. Paper first. Real money only when the user explicitly asks and the connection has the `live` scope; restate venue, symbol, side, size, stop-loss and take-profit and wait for an explicit yes.

Always:

- Sizing is explicit: one of `size_usd`, `margin_usd`, `risk_pct` (crypto) or `lots` (MT5).
- A stop-loss is mandatory; on MT5 a take-profit is mandatory too. Report a refusal instead of inventing a level.
- Send an idempotency key of at least 8 characters with every order-shaped tool and reuse it on retry.
- Name the venue explicitly; the server never chooses one. `auto_trade` cannot be enabled through MCP.
- If `positions_source` is `unavailable`, an empty list means unknown.
- AlgoVesta does not generate signals, hold client funds or give investment advice.

Trading involves substantial risk of loss. Software, not advice.

Reference: https://github.com/AlgoVesta/mcp/blob/main/docs/TOOLS.md · https://github.com/AlgoVesta/mcp/blob/main/docs/SAFETY.md
