# Safety model

The design premise: the AI is the caller — never the authority. Everything that must hold, holds on the server, after the request leaves the model.

## The policy wall

Your rules are compiled once into JSON, validated against a fixed schema, and then evaluated **server-side and deterministically** on every order. The model does not evaluate them and cannot relax them: the check runs after the request leaves the model, in code the prompt never reaches — so neither you in a moment of impatience nor a prompt injected through a webpage changes the outcome. A violation is a hard reject with an audit entry.

| Rule | Type | Meaning |
|---|---|---|
| `max_risk_per_trade_pct` | number, >0–100 | Ceiling on a single trade's share of the account |
| `max_order_size_usd` | number > 0 | Absolute cap on order value |
| `max_daily_loss_usd` | number > 0 | Stop trading for the day past this loss |
| `max_open_positions` | integer | Concurrency limit |
| `leverage_cap` | number, 1–1000 | Your own leverage ceiling |
| `venue_scope` | array | Restrict the AI to named venues |
| `symbol_whitelist` | array | Only these symbols may be traded |
| `symbol_blacklist` | array | These symbols are never traded |
| `allowed_sides` | array | Long only, short only, or both |
| `notes` | string | Your own annotation |

`compile_policy` turns plain language into a policy **preview** — compiling never activates anything. Activation is a separate, deliberate step from the panel, which means a model cannot loosen your rules by talking about them. A policy that fails schema validation cannot be activated at all; there is no partially-valid policy.

Risk-reducing operations — `close_position` and `cancel_order` — are never blocked by the policy wall. Only the kill switch stops them.

## Layered protections

| Layer | What it does |
|---|---|
| Paper by default | Every new key starts in `paper` scope with a $5,000 virtual balance. Reaching real money is an explicit, separate act behind a second factor. |
| Mandatory stop-loss | If neither an explicit stop-loss nor a saved default exists, the order is refused. It cannot be removed later either (`SL_REMOVAL_FORBIDDEN`). On forex/MT5, a take-profit is mandatory too (`hard_forex_tp_required`). |
| Idempotency | Every order-shaped tool (including the read-only `simulate_order`) requires a client-generated key of at least 8 characters. Repeats replay the stored response instead of acting twice — which is what makes client retries after a timeout safe. |
| Kill switch | `POST /api/mcp/freeze` stops everything at once; every tool then returns `user_frozen`. `/unfreeze` reverses it. |
| Per-key revocation | Revoke one client without touching the others. Open event streams drop within seconds. |
| Signed receipts | ed25519 signature plus a per-user hash chain on every action, verifiable with the `verify_receipt` tool or independently against the public key at `/mcp/receipts/pubkey`. Editing an old receipt breaks the chain for every receipt after it. |
| Audit log | Every call is recorded with tool name, arguments, result and latency — readable in the panel and at `GET /api/mcp/audit`. |
| Tenant isolation | Tools cannot accept a user ID; identity comes only from the authenticated connection. |
| Trade-only exchange keys | Exchange API keys are created without withdrawal permission and stored AES-256 encrypted. There is no funds-movement path through MCP. |
| Strategies cannot be armed | `create_strategy` always creates with `auto_trade` off, and `auto_trade` is absent from the writable field set of `update_strategy` — an assistant cannot switch a strategy to real money whatever it is asked. `ip_allowlist` (the second factor that verifies where signals come from) and deletion are refused for the same reason. Refused fields come back in `refused_fields` rather than being silently dropped. |
| Webhook URLs are never returned | A strategy's webhook URL is a password: anyone holding it can send signals into the account. `list_strategies` reports only *whether* one is configured, so the address never enters an AI client's context. |
| No automatic venue routing | `compare_venues` ranks exchanges but never selects one. The order still names its venue explicitly, and an unconnected venue returns `VENUE_NOT_CONNECTED` instead of a substitution. |

## Honest refusals, no silent downgrades

The server refuses rather than guesses:

- Sizing must be explicit: exactly one of `size_usd` / `margin_usd` / `risk_pct` (crypto) or `lots` (MT5). On crypto, the requested amount is matched to the nearest valid lot step; if the deviation exceeds 20%, the order is **not** opened and the nearest workable sizes are reported as concrete numbers. MT5 lot sizes are used exactly as stated, never rounded — outside the broker's limits the order is refused and the permitted range is reported.
- Live crypto venues accept market orders only: a live crypto limit order is refused with `live_limit_not_supported`, never silently converted to market. MT5 positions can only be closed in full — a partial close there is refused rather than converted.
- Omitted stop-loss / take-profit / leverage fall back to **your saved panel settings** — the response reports which fields came from saved settings in `prefs_used`. The model is instructed never to invent them.
- Ambiguous targets are refused: multiple MT5 positions on one symbol require a `ticket`; multiple accounts in one market require `account`. While the target is ambiguous, nothing is executed.
- Unverifiable data is labeled: if an MT5 terminal cannot be reached, `positions_source` is `unavailable` and an empty position list means *unknown*, not "no positions" — and the tool description instructs the model to say so.
- Stale prices are stated as stale, never dressed up as live.
- Comparisons say what they could not measure. `compare_venues` reports trading fees, order book depth and slippage as **not measured** — they are not available on this path, and no default fee table is substituted for them. An exchange that did not publish bid/ask is listed under `not_comparable_on_spread` instead of being ranked as if its spread were zero, so `cheapest_measured` means "lowest measured spread", not "cheapest overall".
- History is reported with its gaps. In `get_trade_history`, the average R-multiple is computed only from trades where entry, stop-loss and exit are all known, and `rr_sample` states how many that was; `total_pnl` is `null` when several account currencies are mixed, with `pnl_by_currency` given instead of a converted single figure; commission is not recorded, so crypto PnL is gross and `fee` stays `null`. If a source could not be read, `incomplete_sources` says which — an empty list is never presented as "no trades".

## Measured, not promised

Latency figures in the documentation are measurements taken in August 2026 (about 1 s end to end on MetaTrader 5, about 3 s on a crypto exchange, median 318 ms on paper over 18 runs) — see the table in the [README](../README.md#rate-limits-and-latency) for sample sizes. Six of the sixteen exchanges — Binance, Bybit, OKX, Gate.io, KuCoin, Bitget — have been verified end to end with real money on both futures and spot, with stop-loss and take-profit confirmed on the exchange itself. Per-exchange spot quirks are deliberate rather than gaps: Bybit and Bitget do not accept a second take-profit leg on spot, and each venue's minimums and order modes are enforced as that exchange defines them.
