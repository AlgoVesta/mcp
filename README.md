<div align="center">

# AlgoVesta MCP Server

**MCP server for 16 crypto exchanges + MetaTrader 5.**

Trade from Claude, ChatGPT, Cursor or any MCP client over one hosted HTTPS connection. Paper by default.

[![MCP Registry](https://img.shields.io/badge/MCP%20Registry-com.algovesta%2Ftrading-1f6feb)](https://registry.modelcontextprotocol.io/v0/servers?search=algovesta)
[![Glama score](https://glama.ai/mcp/servers/AlgoVesta/mcp/badges/score.svg)](https://glama.ai/mcp/servers/AlgoVesta/mcp)
[![Transport](https://img.shields.io/badge/transport-Streamable%20HTTP-0b7285)](https://modelcontextprotocol.io)
[![Auth](https://img.shields.io/badge/auth-OAuth%202.1%20%C2%B7%20PKCE%20S256-5f3dc4)](docs/AUTHENTICATION.md)
[![Tools](https://img.shields.io/badge/tools-20-2b8a3e)](docs/TOOLS.md)
[![Docs license](https://img.shields.io/badge/docs-CC%20BY%204.0-495057)](LICENSE)

[Documentation](https://algovesta.com/mcp-docs.html) · [Tool reference](docs/TOOLS.md) · [Safety model](docs/SAFETY.md) · [Tool schemas (JSON)](https://algovesta.com/mcp/tools.json) · [Product overview](https://algovesta.com/features-mcp.html)

</div>

---

AlgoVesta runs a hosted [Model Context Protocol](https://modelcontextprotocol.io) server that reaches **16 crypto exchanges and MetaTrader 5** through a single HTTPS endpoint. Claude, ChatGPT, Cursor, Claude Code, Gemini CLI or any other MCP-capable client can read balances, open and close positions, move stop-loss and take-profit, and audit its own actions.

Your risk rules are compiled to JSON and evaluated **on the server, after the request leaves the model** — outside the reach of anything the model can be persuaded to say. Every new key starts on a **$5,000 paper balance**, and every action returns an **ed25519-signed, hash-chained receipt**.

> **Hosted and closed-source.** This repository is the public documentation, tool reference and client configuration set. There is nothing to install, build or self-host — you connect to the hosted endpoint.

**Contents** — [At a glance](#at-a-glance) · [How it works](#how-it-works) · [Install in 2 minutes](#install-in-2-minutes) · [Connect your client](#connect-your-client) · [Supported AI clients](#supported-ai-clients) · [In use](#what-it-looks-like-in-use) · [Tools](#tools) · [Safety model](#safety-model) · [Venues](#venues) · [Performance and limits](#performance-and-limits) · [Endpoints](#endpoints) · [FAQ](#faq)

## At a glance

| | |
|---|---|
| **Transport** | Remote MCP over Streamable HTTP |
| **Authentication** | Secret link, or OAuth 2.1 with mandatory PKCE (`S256`) and Dynamic Client Registration (RFC 7591) |
| **Tools** | 20 — 14 read-only, 1 additive write, 5 destructive ([reference](docs/TOOLS.md)) |
| **Venues** | 16 crypto exchanges · MetaTrader 5 · built-in paper engine |
| **Default scope** | `paper` — $5,000 virtual balance, identical toolset |
| **Live scope** | Issued only after a second factor, enforced server-side |
| **Risk policy** | Compiled to JSON, evaluated deterministically on the server |
| **Audit** | ed25519-signed, hash-chained receipt per action, independently verifiable |
| **Install** | None. Hosted endpoint. |
| **Registry** | [`com.algovesta/trading`](https://registry.modelcontextprotocol.io) — [server.json](server.json) is that manifest |

## How it works

```mermaid
flowchart LR
    A["AI client<br/>Claude, ChatGPT, Cursor, and more"]
    B["AlgoVesta MCP server<br/>scope, rate limit, idempotency"]
    C{"Policy wall<br/>evaluated server-side"}
    R["Refusal<br/>plus audit entry"]
    D["Execution"]
    E["16 crypto exchanges"]
    F["MetaTrader 5<br/>managed terminals"]
    G["Paper engine<br/>5,000 USD virtual"]
    H["ed25519-signed receipt"]

    A -->|MCP over HTTPS| B
    B --> C
    C -->|violates a rule| R
    C -->|allowed| D
    D --> E
    D --> F
    D --> G
    D --> H
```

The model is the **caller**, never the authority. It decides what to ask for; the server decides what happens. Your exchange API keys never leave AlgoVesta and are created without withdrawal permission — the assistant only ever holds a scoped, revocable MCP link.

## Install in 2 minutes

There is nothing to install locally: the server is hosted. An AI agent can follow [llms-install.md](llms-install.md) step by step.

1. Create an AlgoVesta account and open the **MCP Connection** tab in the panel.
2. Generate a key. New keys default to the `paper` scope; the full link is shown once.
3. Paste the link into your AI client as a custom MCP server.

```
https://api.algovesta.com/u/avmcp_<your-key>/mcp
```

> **That URL is a credential.** Treat it like a password. If it leaks, revoke it in the panel — revocation takes effect immediately for new connections and drops open event streams within seconds.

Clients that prefer a full authorization flow can use **OAuth 2.1** instead, at `https://api.algovesta.com/mcp`. PKCE (`S256`) is mandatory and Dynamic Client Registration is supported, so most clients configure themselves. See [docs/AUTHENTICATION.md](docs/AUTHENTICATION.md).

## Connect your client

Six walkthroughs below; anything else uses the same URL in whatever field it calls a remote MCP server.

### Claude — web, desktop, iOS, Android

Customize -> Connectors -> **+** -> **Add custom connector**, paste the URL, then allow the tools when Claude asks. [Walkthrough](examples/claude.md)

### Claude Code

```bash
claude mcp add --transport http algovesta https://api.algovesta.com/u/avmcp_<your-key>/mcp
```

[Walkthrough](examples/claude-code.md)

### ChatGPT

[chatgpt.com/plugins](https://chatgpt.com/plugins) -> **+** -> **Add custom MCP server** -> public endpoint `https://api.algovesta.com/mcp` (Streamable HTTP, OAuth). Account and workspace policies on OpenAI's side decide who can add custom MCP servers. [Walkthrough](examples/chatgpt.md)

### Cursor

`~/.cursor/mcp.json`, or `.cursor/mcp.json` inside a project — [file](examples/cursor/mcp.json)

```json
{
  "mcpServers": {
    "algovesta": { "url": "https://api.algovesta.com/u/avmcp_<your-key>/mcp" }
  }
}
```

### VS Code — GitHub Copilot

`.vscode/mcp.json` — [file](examples/vscode/mcp.json)

```json
{
  "servers": {
    "algovesta": {
      "type": "http",
      "url": "https://api.algovesta.com/u/avmcp_<your-key>/mcp"
    }
  }
}
```

### Gemini CLI

**As an extension (one command).** Installs the MCP server definition (OAuth endpoint), a context file and the `algovesta-trading` agent skill:

```
gemini extensions install https://github.com/AlgoVesta/mcp
```

Then run `/mcp auth algovesta` inside Gemini CLI to sign in through the browser consent page (paper scope by default).

**Or by hand:** `~/.gemini/settings.json` — [file](examples/gemini-cli/settings.json). CLI only; the Gemini web app does not accept custom MCP servers.

```json
{
  "mcpServers": {
    "algovesta": { "httpUrl": "https://api.algovesta.com/u/avmcp_<your-key>/mcp" }
  }
}
```

### Anything else

The three config shapes you will meet, plus a `curl` connection check: [examples/other-clients.md](examples/other-clients.md)

## Agent skill

`skills/algovesta-trading/SKILL.md` teaches coding agents (Claude Code, Cursor, Codex, Gemini CLI and others supported by the [skills CLI](https://github.com/vercel-labs/skills)) how to connect and the safe order of operations: read, simulate, paper first, live only with a second-factor key. It contains instructions only and runs no commands.

```
npx skills add AlgoVesta/mcp --skill algovesta-trading
```

## Supported AI clients

This is a standard remote MCP server over Streamable HTTP, so anything that speaks remote MCP can connect. The clients below document that support themselves — each row links to the client's own configuration docs, which is where the exact field name lives.

<details>
<summary><b>Assistants and chat apps</b> — 8</summary>

| Client | MCP documentation |
|---|---|
| Claude — web, desktop, iOS, Android | [Custom connectors](https://support.claude.com/en/articles/11175166-about-custom-connectors-remote-mcp-servers) |
| ChatGPT — custom MCP servers | [OpenAI MCP docs](https://platform.openai.com/docs/mcp) |
| Microsoft Copilot Studio agents | [Extend with MCP](https://learn.microsoft.com/en-us/microsoft-copilot-studio/agent-extend-action-mcp) |
| Goose (Block) | [block.github.io/goose](https://block.github.io/goose/) |
| LibreChat | [MCP in LibreChat](https://www.librechat.ai/docs/features/mcp) |
| Open WebUI | [docs.openwebui.com](https://docs.openwebui.com/) |
| Cherry Studio | [CherryHQ/cherry-studio](https://github.com/CherryHQ/cherry-studio) |
| Raycast | [raycast.com](https://www.raycast.com/) |

</details>

<details>
<summary><b>Coding agents and IDEs</b> — 19</summary>

| Client | MCP documentation |
|---|---|
| Claude Code | [Claude Code MCP](https://docs.anthropic.com/en/docs/claude-code/mcp) |
| OpenAI Codex CLI | [Codex MCP](https://developers.openai.com/codex/mcp/) |
| Gemini CLI | [MCP servers in Gemini CLI](https://google-gemini.github.io/gemini-cli/docs/tools/mcp-server.html) |
| Cursor | [Cursor MCP](https://cursor.com/docs/context/mcp) |
| VS Code — GitHub Copilot | [MCP servers in VS Code](https://code.visualstudio.com/docs/copilot/customization/mcp-servers) |
| Windsurf | [Cascade MCP](https://docs.windsurf.com/windsurf/cascade/mcp) |
| Zed | [Zed MCP](https://zed.dev/docs/ai/mcp) |
| Cline | [Connecting to a remote server](https://docs.cline.bot/mcp/connecting-to-a-remote-server) |
| Roo Code | [Using MCP in Roo](https://docs.roocode.com/features/mcp/using-mcp-in-roo) |
| Kilo Code | [Using MCP in Kilo Code](https://kilocode.ai/docs/features/mcp/using-mcp-in-kilo-code) |
| Continue | [MCP deep dive](https://docs.continue.dev/customize/deep-dives/mcp) |
| JetBrains AI Assistant | [MCP in JetBrains IDEs](https://www.jetbrains.com/help/ai-assistant/mcp.html) |
| Warp | [Warp MCP](https://docs.warp.dev/knowledge-and-collaboration/mcp) |
| Amazon Q Developer | [MCP with Amazon Q](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/qdev-mcp.html) |
| Kiro (AWS) | [Kiro MCP](https://kiro.dev/docs/mcp/) |
| Sourcegraph Amp | [Amp manual](https://ampcode.com/manual#mcp) |
| Trae | [Trae MCP](https://docs.trae.ai/ide/model-context-protocol) |
| PostHog Code | [posthog.com/code](https://posthog.com/code/) |
| Archestra.AI | [archestra.ai](https://www.archestra.ai/) |

</details>

<details>
<summary><b>Frameworks, automation and developer tools</b> — 6</summary>

| Client | MCP documentation |
|---|---|
| OpenAI Agents SDK | [MCP in the Agents SDK](https://openai.github.io/openai-agents-python/mcp/) |
| fast-agent | [evalstate/fast-agent](https://github.com/evalstate/fast-agent) |
| n8n — MCP Client Tool node | [docs.n8n.io](https://docs.n8n.io/) |
| Postman | [MCP requests](https://learning.postman.com/docs/postman-ai-agent-builder/mcp-requests/overview/) |
| MCP Inspector | [modelcontextprotocol/inspector](https://github.com/modelcontextprotocol/inspector) |
| MCPJam | [mcpjam.com](https://www.mcpjam.com/) |

</details>

> **Scope of that claim.** We have verified **Claude** end to end against this server. The others are listed because they document remote MCP support — not because we individually tested each one. Client support also moves fast, and a given client may gate custom servers behind account or workspace policy (ChatGPT does) or behind its CLI only (the Gemini web app does not accept custom MCP servers). If a listed client misbehaves with us, or one is missing, [open an issue](https://github.com/AlgoVesta/mcp/issues).

## What it looks like in use

You never call a tool by name. You ask in plain language; the assistant picks the tool.

| You say | The assistant reaches for |
|---|---|
| *"What's in my accounts right now, and how has this month gone?"* | `get_portfolio_context` — every exchange, MT5 account and the paper book in one call — then `get_trade_history` |
| *"Where is BTC trading on the exchanges I've connected?"* | `compare_venues` — measured spread, taker fee and, for a given size, slippage |
| *"If I opened 0.05 BTC long on Binance with a 2% stop, what would it cost and would my rules allow it?"* | `simulate_order` — a dry run **including the policy verdict** |
| *"OK, open it — stop 2% below, target 4% above."* | `place_order` |
| *"Move the stop on my ETH position up to break-even."* | `modify_position` |
| *"Close half of the SOL long."* | `close_position` — partial on crypto, full close on MT5 |
| *"Never risk more than 1% per trade, max 3 positions, stop me after a 5% daily drawdown."* | `compile_policy` — returns readable JSON you approve |
| *"Show me the receipt for that last order and verify it."* | `verify_receipt` — signature and hash chain, against a [public key](https://api.algovesta.com/mcp/receipts/pubkey) you can fetch yourself |
| *"Run my last 90 days of signals again with a 1% risk cap."* | `simulate_policy`, then `get_job_status` |

**And what it will refuse.** *"Turn on auto-trading for that strategy"* — refused; `auto_trade` is not a writable field in any scope. *"Trade on Kraken for me"* when Kraken is not connected — `VENUE_NOT_CONNECTED`, not a guess. *"Open it without a stop"* — refused, with no saved default to fall back on.

On a `paper` key every one of those runs against the $5,000 virtual balance with the identical toolset, so the whole workflow can be rehearsed before real money is reachable.

## Tools

20 tools. Scope `read` works on any key; `paper` / `live` gates the rest. *Kind* follows each tool's MCP annotations: `destructive` means the tool can overwrite or remove state (open, close, cancel, move a stop, change a strategy); `write (additive)` only creates something new. The table is generated from the live `tools/list` output.

<details open>
<summary><b>Full tool list</b></summary>

| Tool | Title | Scope | Kind | What it does |
|---|---|---|---|---|
| `get_portfolio_context` | Read portfolio | read | read-only | Normalized portfolio view of all connected exchange accounts + MT5 + paper. |
| `get_market_price` | Live price | read | read-only | Returns the LIVE price of a crypto or MT5 symbol: last (+ bid/ask if available) + ts + source. |
| `simulate_order` | Simulate order | read | read-only | SIMULATES an order: expected fill, margin impact, policy check result. |
| `place_order` | PLACE ORDER | paper / live | destructive | Places an order. |
| `compile_policy` | Compile policy | read | read-only | Compiles natural-language risk rules into a JSON policy and returns a PREVIEW. |
| `list_open_orders` | Pending orders | read | read-only | Lists the user's pending limit orders (order_ref, venue, symbol, side, entry_price, size, created_at). |
| `cancel_order` | Cancel order | paper / live | destructive | Cancels a pending limit order. |
| `close_position` | Close position | paper / live | destructive | Closes an open position fully or partially (fraction (0,1]). |
| `modify_position` | Modify SL/TP | paper / live | destructive | Updates the SL/TP of an open position on a crypto exchange or FOREX/MT5. |
| `verify_receipt` | Verify receipt | read | read-only | Verifies an order receipt: ed25519 signature + per-user hash chain. |
| `replay_channel` | Replay channel | read | read-only | Answers "what if you had followed this Telegram channel for the last N days under these rules?". |
| `get_trade_history` | Trade history | read | read-only | Returns the user's CLOSED trades and a performance summary, across crypto exchanges, MetaTrader 5 and the paper account in one list. |
| `compare_venues` | Compare exchanges | read | read-only | Compares the user's CONNECTED crypto exchanges for one symbol on measured execution cost. |
| `list_strategies` | List strategies | read | read-only | Lists the user's TradingView strategies with their settings, plan limit and whether real-money execution is currently on. |
| `create_strategy` | Create strategy | paper / live | write (additive) | Creates a new TradingView strategy. |
| `update_strategy` | Update strategy | paper / live | destructive | Changes the settings of an existing strategy (leverage, risk, SL/TP percentages, trailing, allowed symbols, target account, and whether it accepts signals at all). |
| `backtest_my_signals` | Backtest my signals | read | read-only | Answers "what would have happened to MY OWN past signals with different settings?" - for example "5x instead of 10x" or "a 2% stop instead of 1%". |
| `simulate_policy` | Simulate risk policy | read | read-only | Answers "if I had had this risk rule in place, which of my trades would it have blocked, and what would that have done to my PnL?". |
| `import_tradingview_backtest` | Import TradingView backtest | read | read-only | Answers "TradingView says my strategy made X - what would it have made through AlgoVesta?". |
| `get_job_status` | Job status | read | read-only | Returns the state and, once finished, the result of a background job started by backtest_my_signals, simulate_policy or import_tradingview_backtest. |

</details>

Two of them deserve a note, because what they **refuse** to do is the point:

- **`compare_venues` does not route your order.** For each connected exchange it reports the bid/ask spread, the public taker fee and, when you give an order size, the slippage that size would pay against the live order book. It does not know your own VIP fee tier, and a venue whose cost could not be measured is left out of the ranking rather than treated as free. You still name the exchange yourself in `place_order`.
- **`create_strategy` / `update_strategy` cannot turn real money on.** A new strategy is always created with `auto_trade` off, and `auto_trade` is not in the writable field set — an assistant cannot set it, whatever it is asked or persuaded to do. A single order is one action you can see; a strategy keeps trading after the conversation ends, so arming one stays a human decision made in the panel. `ip_allowlist` is refused for the same reason: it is the second factor that verifies where signals come from.

Full reference with descriptions and JSON Schemas: [docs/TOOLS.md](docs/TOOLS.md) · machine-readable: [tools.json](https://algovesta.com/mcp/tools.json)

## Safety model

The AI is the caller — never the authority. Full model: [docs/SAFETY.md](docs/SAFETY.md).

| Control | What it means |
|---|---|
| **Paper by default** | Every new key starts in `paper` scope on a $5,000 virtual balance. Real money is a separate, deliberate act: the `live` scope is issued only after a second factor (TOTP, or an email code for panel-created keys; over OAuth, TOTP is required), enforced server-side with no exceptions. |
| **Server-side policy wall** | Max risk per trade, order-size cap, daily-loss cap, position count, leverage cap, venue/symbol/side restrictions — compiled to JSON and evaluated deterministically after the request leaves the model. A prompt injection can change what the model *sends*; it cannot change how the server evaluates it. |
| **Stop-loss is mandatory** | No stop-loss and no saved default means the order is refused. The stop cannot be removed later either. On forex/MT5, a take-profit is mandatory too. |
| **Idempotency** | Every order-shaped tool — including the read-only `simulate_order`, so one key carries from simulation to placement — requires a client-generated idempotency key. Repeats replay the stored response instead of acting twice. |
| **Signed receipts** | Every action returns an ed25519-signed, hash-chained receipt, verifiable against the [public key endpoint](https://api.algovesta.com/mcp/receipts/pubkey). |
| **Kill switch and per-key revocation** | Freeze everything at once, or revoke one client without touching the others. |
| **Structural tenant isolation** | Tools have no identity parameter at all. The account is derived from the authenticated connection, so there is no argument a confused model or an attacker could supply to reach another account. |
| **No withdrawal path** | Exchange API keys are created without withdrawal permission and stay inside AlgoVesta, AES-256 encrypted. The AI only ever holds a scoped, revocable MCP link. |

## Venues

**Crypto (16)** — Binance · Bybit · OKX · KuCoin · Gate.io · Bitget · Kraken · Coinbase · BingX · Hyperliquid · Backpack · HTX · BloFin · Phemex · WOO X · CoinEx

**Forex, metals, indices** — MetaTrader 5, zero installation: AlgoVesta runs the MT5 terminals on its own managed servers, connected to your broker 24/7.

**Simulation** — built-in paper engine, $5,000 virtual.

Six exchanges — Binance, Bybit, OKX, Gate.io, KuCoin and Bitget — have been verified end to end with real money on both futures and spot, with stop-loss and take-profit confirmed on the exchange itself. Each exchange has its own quirks, and the differences are deliberate rather than gaps: Bybit and Bitget do not accept a second take-profit leg on spot; OKX spot is routed through the raw API to keep a cash account from silently becoming a margin account; Binance spot enforces a minimum notional before buying; KuCoin market buys are placed in cost mode. A venue becomes available to the AI only after you connect it — asking for one you have not connected returns `VENUE_NOT_CONNECTED` rather than a guess.

## Performance and limits

| Limit | Value |
|---|---|
| All tool calls, per key | 60 / minute |
| `place_order` | 10 / minute |
| `replay_channel` | 5 / hour (results cached 24 h) |

Measured, not marketing — figures measured in August 2026; the averages match the ones published on algovesta.com:

| Stage | Measured |
|---|---|
| Request intake and parsing | 17–67 ms (median 38, n=6) |
| End to end on MetaTrader 5 | about 1.2 seconds on average, stop-loss and take-profit included (single live demo order: 849 ms) |
| End to end on a crypto exchange | about 4.1 seconds on average, stop-loss and take-profit included (single live order: 2,785 ms) |
| Paper engine | median 318 ms (n=18, min 284, max 769) |

Time spent inside your AI client — the model thinking, and you confirming — is not included and usually dominates. This server is not a low-latency execution venue and is not sold as one.

## Endpoints

| Purpose | URL |
|---|---|
| MCP — secret link | `https://api.algovesta.com/u/avmcp_<key>/mcp` |
| MCP — OAuth 2.1 | `https://api.algovesta.com/mcp` |
| Live events (SSE) | `https://api.algovesta.com/mcp/events` · `/u/avmcp_<key>/events` |
| OAuth metadata | [`/.well-known/oauth-authorization-server`](https://api.algovesta.com/.well-known/oauth-authorization-server) · [`/.well-known/oauth-protected-resource`](https://api.algovesta.com/.well-known/oauth-protected-resource) |
| Receipt public key | [`/mcp/receipts/pubkey`](https://api.algovesta.com/mcp/receipts/pubkey) |
| Tool schemas | [`https://algovesta.com/mcp/tools.json`](https://algovesta.com/mcp/tools.json) |

Published in the official [MCP Registry](https://registry.modelcontextprotocol.io) as `com.algovesta/trading`; [server.json](server.json) in this repository is that manifest.

## FAQ

<details>
<summary><b>Can Claude, ChatGPT or Cursor actually place a real trade on my exchange account?</b></summary>

Yes — once you give it a key with the `live` scope, and that scope is only issued after a second factor. Until then the same assistant runs against a $5,000 paper balance with identical tools, so you can rehearse the entire workflow before any real money is reachable.

</details>

<details>
<summary><b>Do I have to give my exchange API keys to the AI?</b></summary>

No, and you should never do that with any tool. Your API keys stay inside AlgoVesta, encrypted, created without withdrawal permission. The assistant only ever holds an MCP link — scoped, rate-limited, policy-checked, individually revocable, and useless for moving funds off an exchange.

</details>

<details>
<summary><b>Can a prompt injection make the AI ignore my risk rules?</b></summary>

It can change what the model asks for. It does not change how the server answers: your rules are evaluated after the request leaves the model, by code the prompt never reaches, and a violation is rejected and written to the audit log. The limits of that guarantee are worth stating plainly — it covers rule evaluation, scope and account isolation, not the wording of what the assistant tells you afterwards.

</details>

<details>
<summary><b>What happens if the model calls <code>place_order</code> twice by mistake?</b></summary>

Nothing happens twice. Every write tool requires an idempotency key, and a repeat of the same key returns the stored response instead of acting again.

</details>

<details>
<summary><b>How do I stop everything immediately?</b></summary>

The kill switch in the panel (`POST /api/mcp/freeze`) stops everything at once; every tool then returns `user_frozen`. To cut off a single client, revoke just that key.

</details>

## Examples

Ready-to-use files in [`examples/`](examples/):

- [TradingView alert templates](examples/tradingview/README.md): six JSON messages (basic entry, multi take-profit, Pine strategy with position state, close, forex on MT5, crypto with margin and leverage) and the field reference the webhook receiver uses.
- [Telegram signal parsing corpus](examples/signals/README.md): 30 synthetic signal messages with the expected parse for each, for testing your own copier or seeing which formats the AlgoVesta parser is built for.
- [MCP client code](examples/mcp-clients/README.md): minimal Python and Node.js programs that connect with a secret link, read the portfolio and place a paper order.
- Client configs: [Cursor](examples/cursor/mcp.json), [VS Code](examples/vscode/mcp.json), [Gemini CLI](examples/gemini-cli/settings.json).

## Support

Account and trading questions: [support](https://algovesta.com/support-ticket.html) · [support@algovesta.com](mailto:support@algovesta.com)
Vulnerability reports: [SECURITY.md](SECURITY.md)
Client that will not connect, or one missing from the list above: [open an issue](https://github.com/AlgoVesta/mcp/issues)

## Compliance

AlgoVesta does not generate signals, does not hold, receive or move client funds, and does not provide investment advice. Trading carries risk; automation does not remove it. Every new key starts on paper.

## License

Copyright (c) 2026 AlgoVesta. Documentation and configuration examples in this repository are licensed under [CC BY 4.0](LICENSE). The AlgoVesta MCP server itself is proprietary and hosted; this repository contains no server source code.
