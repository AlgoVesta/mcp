# AlgoVesta MCP Server

**MCP server for 16 exchanges + MT5.** Crypto + MT5 in one MCP connection — your AI assistant trades for you.

[Documentation](https://algovesta.com/mcp-docs.html) · [Machine-readable tool schemas](https://algovesta.com/mcp/tools.json) · [Product overview](https://algovesta.com/features-mcp.html) · [Website](https://algovesta.com)

AlgoVesta runs a hosted [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server that reaches **16 crypto exchanges and MetaTrader 5** through one HTTPS link. Claude, ChatGPT, Cursor, Claude Code, Gemini CLI or any MCP-capable client can read balances, open and close positions, move stop-loss and take-profit, and audit its own actions — with your risk rules evaluated on the server, outside the model's reach. Every new key starts on a **$5,000 paper balance**, and every action returns an **ed25519-signed receipt**.

> The server is hosted and closed-source. This repository contains its public documentation, tool reference and client configuration examples. There is nothing to install or build — you connect to the hosted endpoint.

## Quick start

1. Create an AlgoVesta account and open the **MCP Connection** tab in the panel.
2. Generate a key. New keys default to the `paper` scope; the full link is shown once.
3. Paste the link into your AI client as a custom MCP server. Done — no code, no local install, no API keys handed to the AI.

Your connection URL looks like this:

```
https://api.algovesta.com/u/avmcp_<your-key>/mcp
```

That URL is a credential — treat it like a password. If it leaks, revoke it in the panel; revocation takes effect immediately for new connections and drops open event streams within seconds.

Clients that prefer a full authorization flow can use **OAuth 2.1** instead:

```
https://api.algovesta.com/mcp
```

PKCE (`S256`) is mandatory and Dynamic Client Registration (RFC 7591) is supported, so most clients configure themselves. See [docs/AUTHENTICATION.md](docs/AUTHENTICATION.md).

## Connect your client

| AI client | Where the link goes | Notes |
|---|---|---|
| Claude (web, desktop, iOS, Android) | Customize → Connectors → Add custom connector | Verified end to end against this server. |
| Claude Code | `claude mcp add --transport http algovesta <url>` | Command-line; good for scripted workflows. |
| Cursor | `mcp.json`, the `"url"` field | — |
| VS Code (Copilot) | `.vscode/mcp.json`, `"servers"` with `"type": "http"` | — |
| ChatGPT | Developer mode → custom connector | **Paid plans only** (an OpenAI restriction). |
| Gemini CLI | `~/.gemini/settings.json`, the `"httpUrl"` field | **CLI only** — the Gemini web app does not support custom MCP servers. |
| Any other MCP client | Its own MCP settings | Standard remote MCP over Streamable HTTP. |

Ready-made snippets: [examples/](examples/).

Only the Claude row has been verified end to end by us. The others follow each client's own official configuration documentation and this is a standard remote MCP server over Streamable HTTP, but we have not individually tested every client — if one of them misbehaves, [open an issue](../../issues) and we will look at it.

## The 11 tools

| Tool | Scope | Kind | Purpose |
|---|---|---|---|
| `get_portfolio_context` | read | read-only | Every connected account in one call |
| `get_market_price` | read | read-only | Live price with freshness reported |
| `simulate_order` | read | read-only | Dry-run including the policy verdict |
| `place_order` | paper / live | destructive | Opens a position (live crypto: market orders only) |
| `compile_policy` | read | read-only | Turns plain-language rules into a policy preview |
| `list_open_orders` | read | read-only | Pending limit orders (paper book) |
| `cancel_order` | paper / live | destructive | Cancels a pending order |
| `close_position` | paper / live | destructive | Closes fully or partially (MT5: full close only) |
| `modify_position` | paper / live | write | Moves stop-loss and take-profit |
| `verify_receipt` | read | read-only | Checks signature and hash chain |
| `replay_channel` | read | read-only | Backtests a Telegram channel against your rules |

Full reference with descriptions and JSON Schemas: [docs/TOOLS.md](docs/TOOLS.md).

## Safety model

The AI is the caller — never the authority. See [docs/SAFETY.md](docs/SAFETY.md) for the full model.

- **Paper by default.** Every new key starts in `paper` scope on a $5,000 virtual balance. Real money is a separate, deliberate act: the `live` scope is only issued after a second factor (a TOTP code, or an email code for panel-created keys; over OAuth, TOTP is required), enforced server-side with no exceptions.
- **Server-side policy wall.** Your risk rules (max risk per trade, order-size cap, daily-loss cap, position count, leverage cap, venue/symbol/side restrictions) are compiled to JSON and evaluated deterministically on the server, after the request leaves the model. A prompt injection can change what the model *sends*; it does not change how the server evaluates it, and a rule violation is rejected regardless of what the model was persuaded to ask for.
- **Stop-loss is mandatory.** No stop-loss and no saved default means the order is refused. The stop-loss cannot be removed later either. On forex/MT5, a take-profit is mandatory too.
- **Idempotency.** Every order-shaped tool — including the read-only `simulate_order`, so the same key can carry from simulation to placement — requires a client-generated idempotency key; repeats replay the stored response instead of acting twice.
- **Signed receipts.** Every action returns an ed25519-signed, hash-chained receipt, independently verifiable against the [public key endpoint](https://api.algovesta.com/mcp/receipts/pubkey).
- **Kill switch & per-key revocation.** Freeze everything at once, or revoke one client without touching the others.
- **Structural tenant isolation.** Tools have no identity parameter at all — the account is derived from the authenticated connection, so there is no argument a confused model or an attacker could supply to reach someone else's account.
- **No withdrawal path.** Exchange API keys are created without withdrawal permission and stay inside AlgoVesta, AES-256 encrypted. The AI only ever holds a scoped, revocable MCP link.

## Venues

**Crypto (16):** Binance, Bybit, OKX, KuCoin, Gate.io, Bitget, Kraken, Coinbase, BingX, Hyperliquid, Backpack, HTX, BloFin, Phemex, WOO X, CoinEx.
**Forex, metals, indices:** MetaTrader 5 — zero installation; AlgoVesta runs the MT5 terminals on its own managed servers, connected to your broker 24/7.
**Simulation:** built-in paper engine ($5,000 virtual).

Six exchanges — Binance, Bybit, OKX, Gate.io, KuCoin and Bitget — have been verified end to end with real money on both futures and spot, with stop-loss and take-profit confirmed on the exchange itself. Each exchange has its own quirks, and the differences are deliberate rather than gaps: Bybit and Bitget do not accept a second take-profit leg on spot, OKX spot is routed through the raw API to keep a cash account from silently becoming a margin account, Binance spot enforces a minimum notional before buying, and KuCoin market buys are placed in cost mode. A venue becomes available to the AI only after you connect it; asking for one you have not connected returns `VENUE_NOT_CONNECTED` rather than a guess.

## Rate limits and latency

| Limit | Value |
|---|---|
| All tool calls, per key | 60 / minute |
| `place_order` | 10 / minute |
| `replay_channel` | 5 / hour (results cached 24 h) |

Measured latency, not marketing numbers — all figures measured in August 2026:

| Stage | Measured |
|---|---|
| Request intake and parsing | 17–67 ms (median 38, n=6) |
| End to end on MetaTrader 5 | about 1 second (849 ms on a live demo order) |
| End to end on a crypto exchange | about 3 seconds (2,785 ms on a live order) |
| Paper engine | median 318 ms (n=18, min 284, max 769) |

Time spent inside your AI client — the model thinking, and you confirming — is not included and usually dominates. This server is not a low-latency execution venue and is not sold as one.

## Endpoints

| Purpose | URL |
|---|---|
| MCP (secret link) | `https://api.algovesta.com/u/avmcp_<key>/mcp` |
| MCP (OAuth 2.1) | `https://api.algovesta.com/mcp` |
| Live events (SSE) | `https://api.algovesta.com/mcp/events` · `/u/avmcp_<key>/events` |
| OAuth metadata | [`/.well-known/oauth-authorization-server`](https://api.algovesta.com/.well-known/oauth-authorization-server) · [`/.well-known/oauth-protected-resource`](https://api.algovesta.com/.well-known/oauth-protected-resource) |
| Receipt public key | [`/mcp/receipts/pubkey`](https://api.algovesta.com/mcp/receipts/pubkey) |
| Tool schemas | [`https://algovesta.com/mcp/tools.json`](https://algovesta.com/mcp/tools.json) |

The server is published in the official [MCP Registry](https://registry.modelcontextprotocol.io) as `com.algovesta/trading` — [server.json](server.json) in this repository is that registry manifest.

## FAQ

**Can Claude, ChatGPT or Cursor actually place a real trade on my exchange account?**
Yes, once you give it a key with the `live` scope — and that scope is only issued after a second factor. Until then the same assistant runs against a $5,000 paper balance with identical tools, so you can rehearse the entire workflow before any real money is reachable.

**Do I have to give my exchange API keys to the AI?**
No, and you should never do that with any tool. Your API keys stay inside AlgoVesta, encrypted, created without withdrawal permission. The assistant only ever holds an MCP link — scoped, rate-limited, policy-checked, individually revocable, and useless for moving funds off an exchange.

**Can a prompt injection make the AI ignore my risk rules?**
It can change what the model asks for. It does not change how the server answers: your rules are evaluated after the request leaves the model, by code the prompt never reaches, and a violation is rejected and written to the audit log. The limits of that guarantee are worth stating plainly — it covers rule evaluation, scope and account isolation, not the wording of what the assistant tells you afterwards.

**What happens if the model calls `place_order` twice by mistake?**
Nothing happens twice. Every write tool requires an idempotency key, and a repeat of the same key returns the stored response instead of acting again.

**How do I stop everything immediately?**
The kill switch in the panel (`POST /api/mcp/freeze`) stops everything at once; every tool then returns `user_frozen`. To cut off a single client, revoke just that key.

## Support

Account and trading questions: [support](https://algovesta.com/support-ticket.html) · [support@algovesta.com](mailto:support@algovesta.com). Vulnerability reports: [SECURITY.md](SECURITY.md).

## Compliance

AlgoVesta does not generate signals, hold, receive, or move client funds, and does not provide investment advice. Trading carries risk; automation does not remove it. Every new key starts on paper.

## License

Copyright (c) 2026 AlgoVesta. Documentation and configuration examples in this repository are licensed under [CC BY 4.0](LICENSE). The AlgoVesta MCP server itself is proprietary and hosted; this repository contains no server source code.
