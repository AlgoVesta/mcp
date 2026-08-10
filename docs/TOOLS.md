# Tool reference — all 11 tools

> These are the exact tool descriptions the server sends to every MCP client in `tools/list` ([machine-readable copy](https://algovesta.com/mcp/tools.json)). They are written as instructions **to the AI assistant** — which is why they use imperatives and emphasis. This file is generated from the live server output; edits belong in the server, not here.

Two conventions are deliberate and worth knowing before you read the table. **`place_order` is titled in capitals** — MCP clients render the tool title in the confirmation dialog, and the one tool that spends real money is meant to look different from the ten that do not. **Required fields accept `null` in the JSON Schema**: rather than failing on a raw schema error, the server collects every missing field and returns a single `MISSING_FIELDS` response naming all of them at once, so the assistant can ask you for everything in one message instead of discovering the gaps one at a time.

| Tool | Title | Scope | Kind | Rate limit (per key) |
|---|---|---|---|---|
| [`get_portfolio_context`](#get_portfolio_context) | Read portfolio | `read` | read-only | 60/min |
| [`get_market_price`](#get_market_price) | Live price | `read` | read-only | 60/min |
| [`simulate_order`](#simulate_order) | Simulate order | `read` | read-only | 60/min |
| [`place_order`](#place_order) | PLACE ORDER | `paper\|live` | destructive | 10/min |
| [`compile_policy`](#compile_policy) | Compile policy | `read` | read-only | 60/min |
| [`list_open_orders`](#list_open_orders) | Pending orders | `read` | read-only | 60/min |
| [`cancel_order`](#cancel_order) | Cancel order | `paper\|live` | destructive | 60/min |
| [`close_position`](#close_position) | Close position | `paper\|live` | destructive | 60/min |
| [`modify_position`](#modify_position) | Modify SL/TP | `paper\|live` | write | 60/min |
| [`verify_receipt`](#verify_receipt) | Verify receipt | `read` | read-only | 60/min |
| [`replay_channel`](#replay_channel) | Replay channel | `read` | read-only | 5/hour |

## get_portfolio_context

**Title:** Read portfolio &nbsp;·&nbsp; **Scope:** `read` &nbsp;·&nbsp; **Kind:** read-only<br>
**Annotations:** `readOnlyHint=true` `destructiveHint=false` `idempotentHint=true`

Normalized portfolio view of all connected exchange accounts + MT5 + paper.
Takes no parameters; returns ONLY the accounts of the user whose key the
connection was made with.

IMPORTANT (crypto accounts): "balance"/"equity" is the futures wallet ONLY.
The spot wallet balance is in the separate "spot_balance" field (there MAY be
money on spot even when futures shows 0 — check spot_balance before telling a
user they have no balance).

IMPORTANT (forex/MT5 accounts): check the "positions_source" field on each
account. "live_ea" means the position list is real data verified by the MetaTrader 5
Expert Advisor (EA) AlgoVesta runs on its managed terminal.
"unavailable" means the EA/terminal could not be reached — in that case
positions=[] does NOT mean "no open positions", only that it could not be
verified. For such an account never tell the user definitively "you have no
open positions"; say "this cannot be verified right now, try again or check
the terminal".

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {},
  "title": "get_portfolio_contextArguments",
  "type": "object"
}
```

</details>

## get_market_price

**Title:** Live price &nbsp;·&nbsp; **Scope:** `read` &nbsp;·&nbsp; **Kind:** read-only<br>
**Annotations:** `readOnlyHint=true` `destructiveHint=false` `idempotentHint=true`

Returns the LIVE price of a crypto or MT5 symbol: last (+ bid/ask if available) + ts + source.
Take the price FROM HERE before opening an order — do NOT use your own estimate or stale
knowledge. A current price is essential when computing SL/TP/limit levels. Source order:
(1) the shared price cache (~1s fresh), (2) if not cached, a single live REST call to the
exchange. If the price is older than 10s OR cannot be fetched, that is stated EXPLICITLY —
a stale price is never presented as live.

If venue is given, that exchange is used; otherwise the user's CONNECTED exchanges are
searched. If found on no CEX, a DEX (DexScreener) informational price is returned together
with a 'you CANNOT trade this on your connected exchanges' warning.
Returns: {ok, venue, symbol, last, bid, ask, ts, source, age_sec} or {ok:false, error,...}.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "venue": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Exchange/venue name — must be one of the user's CONNECTED accounts (e.g. binance, bybit, okx, mt5, paper). Never choose one the user did not name.",
      "title": "Venue"
    },
    "symbol": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Trading symbol, e.g. BTCUSDT, ETHUSDT, EURUSD.",
      "title": "Symbol"
    }
  },
  "title": "get_market_priceArguments",
  "type": "object"
}
```

</details>

## simulate_order

**Title:** Simulate order &nbsp;·&nbsp; **Scope:** `read` &nbsp;·&nbsp; **Kind:** read-only<br>
**Annotations:** `readOnlyHint=true` `destructiveHint=false` `idempotentHint=true`

SIMULATES an order: expected fill, margin impact, policy check result.
Sends NO real order.

This is a READ operation — do NOT ask the user for confirmation before calling
simulate_order. Show its result to the user as ONE summary and ask for ONE
confirmation; call place_order once confirmed.

Ask for all missing fields in ONE message; NEVER invent a value for any field.
AMOUNT DISTINCTION (CRITICAL): if the user says 'X dollars' with leverage (e.g.
'$20 at 5x') and it is NOT clear whether they mean MARGIN (margin_usd=20 -> a $100
position) or POSITION VALUE (size_usd=20 -> a $20 position), ASK — do not open an
order of the wrong size.
SIZE FIELD DEPENDS ON MARKET: forex/MT5 -> `lots` (0.05, 0.10... exactly what the
customer said); crypto -> EXACTLY ONE of size_usd/margin_usd/risk_pct. Do not send
`leverage` for forex.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "venue": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Exchange/venue name — must be one of the user's CONNECTED accounts (e.g. binance, bybit, okx, mt5, paper). Never choose one the user did not name.",
      "title": "Venue"
    },
    "symbol": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Trading symbol, e.g. BTCUSDT, ETHUSDT, EURUSD.",
      "title": "Symbol"
    },
    "side": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "'buy' (long) or 'sell' (short). Never choose without an EXPLICIT user instruction.",
      "title": "Side"
    },
    "order_type": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "'market' or 'limit'. If 'limit', entry_price is REQUIRED. Live CRYPTO venues accept market orders only — a live crypto limit order is refused with live_limit_not_supported (never silently converted). Limit orders work on paper and on live MT5.",
      "title": "Order Type"
    },
    "leverage": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Leverage (crypto). Send it if the user STATED one; if they did NOT, LEAVE IT EMPTY — the server applies the leverage saved in the user's panel. NEVER pick a value yourself: leverage changes position size and overrides the user's setting. For FOREX/MT5 do NOT ask and do NOT send: the product is unleveraged (1x) and size is given via lots.",
      "title": "Leverage"
    },
    "sl": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Stop-loss PRICE. Below entry for long, above entry for short. If the user did NOT state one, LEAVE IT EMPTY — the user's saved SL percentage is applied (use `sl_pct` if you want to pass a percentage). NEVER invent a price. SL remains mandatory: if neither a value nor a saved setting exists, the order is NOT opened.",
      "title": "Sl"
    },
    "tp": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Take-profit PRICE. Above entry for long, below entry for short. If the user did NOT state one, LEAVE IT EMPTY — the user's saved TP percentage is applied. NEVER invent a price.",
      "title": "Tp"
    },
    "idempotency_key": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Client-generated unique idempotency key (min 8 characters). Reuse the SAME key when retrying the same action — the stored response is replayed and the action is NOT performed a second time.",
      "title": "Idempotency Key"
    },
    "size_usd": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "POSITION VALUE / notional (USD) — NOT the margin you put up. E.g. size_usd=100 => a $100 position (margin = 100/leverage). If the user says 'X dollars at 5x' meaning MARGIN, use margin_usd instead. Provide EXACTLY ONE of size_usd / margin_usd / risk_pct; never invent a value the user did not give.",
      "title": "Size Usd"
    },
    "margin_usd": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "MARGIN / collateral (USD) — the money out of your own pocket. notional = margin_usd x leverage. E.g. '$20 at 5x' => margin_usd=20 => a $100 position. This is usually what users mean by 'X dollars' with leverage. EXACTLY ONE of size_usd / margin_usd / risk_pct; never invent a value.",
      "title": "Margin Usd"
    },
    "risk_pct": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Percentage of the account's FREE balance to use as MARGIN (0, 100]. The position value becomes margin x leverage — this is NOT risk-based sizing and the SL distance is not used in the calculation. Can be used instead of size_usd/margin_usd — EXACTLY ONE of the three.",
      "title": "Risk Pct"
    },
    "entry_price": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Limit order entry price. REQUIRED when order_type='limit'; not used for market orders.",
      "title": "Entry Price"
    },
    "lots": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "FOREX/MT5 trade volume in LOTS (e.g. 0.05, 0.10, 1.5). This is how size is given for forex orders — instead of size_usd/margin_usd/risk_pct. The lot size the user stated is used EXACTLY, never rounded to a fixed value; if it falls outside the broker/account limits the order is not opened and the allowed range is reported. NOT used for crypto orders.",
      "title": "Lots"
    },
    "market": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "CRYPTO market type: 'futures' (default, leveraged perpetual) or 'spot' (unleveraged, actually buys the coin). Send 'spot' if the user says spot or if their account supports spot only; on spot leverage must be 1. NOT used for forex/MT5.",
      "title": "Market"
    },
    "take_profits": {
      "anyOf": [
        {
          "items": {},
          "type": "array"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "SCALED take-profit (partial TP). A list: [{\"price\": 70000, \"close_pct\": 50}, {\"price\": 75000, \"close_pct\": 50}] — close_pct is the percentage of the position closed at that target and the TOTAL may not exceed 100%. Prices must follow target order (ascending for long, descending for short). For a single target use `tp` instead; the two cannot be sent together. Never invent targets the user did not give.",
      "title": "Take Profits"
    },
    "sl_pct": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Give SL as a PERCENTAGE (e.g. 3 = 3% away from entry). If the user said '3% SL', use THIS — do not compute the price yourself and do NOT call get_market_price; the server computes it from the live entry. Cannot be combined with `sl`. If the user did NOT state a value, LEAVE IT EMPTY: the server applies the SL percentage saved in the user's AlgoVesta account (the same default used everywhere else in the product). Producing your own value overrides the user's setting.",
      "title": "Sl Pct"
    },
    "tp_pct": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Give TP as a PERCENTAGE (e.g. 5 = 5% away from entry). If the user said '5% TP', use THIS — do not compute the price yourself. Cannot be combined with `tp`. If the user did not state one, LEAVE IT EMPTY — the user's saved TP percentage is applied.",
      "title": "Tp Pct"
    },
    "account": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "TARGET ACCOUNT — the account this call acts on (for place_order/simulate_order the account the order is opened in; for close/modify the account holding the position). For MT5 the login number (e.g. '12345678'), for exchanges the account label (e.g. 'Binance2'). Copy it EXACTLY from the 'account' field in the get_portfolio_context output. REQUIRED when the user has more than one account in the same market: if left empty the action is NOT performed and the user is asked which account — it is never silently applied to the default/primary account. Never choose without the user saying so.",
      "title": "Account"
    }
  },
  "required": [
    "venue",
    "symbol",
    "side",
    "order_type",
    "idempotency_key"
  ],
  "title": "simulate_orderArguments",
  "type": "object"
}
```

</details>

## place_order

**Title:** PLACE ORDER &nbsp;·&nbsp; **Scope:** `paper|live` &nbsp;·&nbsp; **Kind:** destructive (asks for confirmation in capable clients)<br>
**Annotations:** `readOnlyHint=false` `destructiveHint=true` `idempotentHint=true`<br>
**Rate limit:** 10/min

Places an order. idempotency_key is REQUIRED: if the same (user, key) arrives a
second time the stored response is returned and a second order is NEVER opened.
scope=paper runs on the paper engine, scope=live runs through live execution.
The policy wall runs server-side; a violation means a hard reject + audit entry.

CONFIRMATION: if the user has ALREADY confirmed the simulate_order summary once,
call place_order DIRECTLY — do not ask again, do not reprint the summary, do not
re-ask for information.
Ask for all missing fields in ONE message; NEVER invent a value for any field.
AMOUNT DISTINCTION (CRITICAL): if 'X dollars' with leverage is ambiguous between
MARGIN (margin_usd) and POSITION VALUE (size_usd), ASK — do not open an order of the
wrong size.
SIZE FIELD DEPENDS ON MARKET: forex/MT5 -> `lots` (exactly the lot size the customer
stated, never rounded to a fixed value); crypto -> EXACTLY ONE of
size_usd/margin_usd/risk_pct. Do not send `leverage` for forex (the product is
unleveraged).
ACCOUNT DISTINCTION (CRITICAL): the user may have several accounts in the same
market (two MT5 accounts, 'Binance' + 'Binance2'). Pass WHICH account via `account`;
if it is not clear, ASK. Opening an order in the wrong account is WORSE than not
opening one at all.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "venue": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Exchange/venue name — must be one of the user's CONNECTED accounts (e.g. binance, bybit, okx, mt5, paper). Never choose one the user did not name.",
      "title": "Venue"
    },
    "symbol": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Trading symbol, e.g. BTCUSDT, ETHUSDT, EURUSD.",
      "title": "Symbol"
    },
    "side": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "'buy' (long) or 'sell' (short). Never choose without an EXPLICIT user instruction.",
      "title": "Side"
    },
    "order_type": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "'market' or 'limit'. If 'limit', entry_price is REQUIRED. Live CRYPTO venues accept market orders only — a live crypto limit order is refused with live_limit_not_supported (never silently converted). Limit orders work on paper and on live MT5.",
      "title": "Order Type"
    },
    "leverage": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Leverage (crypto). Send it if the user STATED one; if they did NOT, LEAVE IT EMPTY — the server applies the leverage saved in the user's panel. NEVER pick a value yourself: leverage changes position size and overrides the user's setting. For FOREX/MT5 do NOT ask and do NOT send: the product is unleveraged (1x) and size is given via lots.",
      "title": "Leverage"
    },
    "sl": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Stop-loss PRICE. Below entry for long, above entry for short. If the user did NOT state one, LEAVE IT EMPTY — the user's saved SL percentage is applied (use `sl_pct` if you want to pass a percentage). NEVER invent a price. SL remains mandatory: if neither a value nor a saved setting exists, the order is NOT opened.",
      "title": "Sl"
    },
    "tp": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Take-profit PRICE. Above entry for long, below entry for short. If the user did NOT state one, LEAVE IT EMPTY — the user's saved TP percentage is applied. NEVER invent a price.",
      "title": "Tp"
    },
    "idempotency_key": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Client-generated unique idempotency key (min 8 characters). Reuse the SAME key when retrying the same action — the stored response is replayed and the action is NOT performed a second time.",
      "title": "Idempotency Key"
    },
    "size_usd": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "POSITION VALUE / notional (USD) — NOT the margin you put up. E.g. size_usd=100 => a $100 position (margin = 100/leverage). If the user says 'X dollars at 5x' meaning MARGIN, use margin_usd instead. Provide EXACTLY ONE of size_usd / margin_usd / risk_pct; never invent a value the user did not give.",
      "title": "Size Usd"
    },
    "margin_usd": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "MARGIN / collateral (USD) — the money out of your own pocket. notional = margin_usd x leverage. E.g. '$20 at 5x' => margin_usd=20 => a $100 position. This is usually what users mean by 'X dollars' with leverage. EXACTLY ONE of size_usd / margin_usd / risk_pct; never invent a value.",
      "title": "Margin Usd"
    },
    "risk_pct": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Percentage of the account's FREE balance to use as MARGIN (0, 100]. The position value becomes margin x leverage — this is NOT risk-based sizing and the SL distance is not used in the calculation. Can be used instead of size_usd/margin_usd — EXACTLY ONE of the three.",
      "title": "Risk Pct"
    },
    "entry_price": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Limit order entry price. REQUIRED when order_type='limit'; not used for market orders.",
      "title": "Entry Price"
    },
    "lots": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "FOREX/MT5 trade volume in LOTS (e.g. 0.05, 0.10, 1.5). This is how size is given for forex orders — instead of size_usd/margin_usd/risk_pct. The lot size the user stated is used EXACTLY, never rounded to a fixed value; if it falls outside the broker/account limits the order is not opened and the allowed range is reported. NOT used for crypto orders.",
      "title": "Lots"
    },
    "market": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "CRYPTO market type: 'futures' (default, leveraged perpetual) or 'spot' (unleveraged, actually buys the coin). Send 'spot' if the user says spot or if their account supports spot only; on spot leverage must be 1. NOT used for forex/MT5.",
      "title": "Market"
    },
    "take_profits": {
      "anyOf": [
        {
          "items": {},
          "type": "array"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "SCALED take-profit (partial TP). A list: [{\"price\": 70000, \"close_pct\": 50}, {\"price\": 75000, \"close_pct\": 50}] — close_pct is the percentage of the position closed at that target and the TOTAL may not exceed 100%. Prices must follow target order (ascending for long, descending for short). For a single target use `tp` instead; the two cannot be sent together. Never invent targets the user did not give.",
      "title": "Take Profits"
    },
    "sl_pct": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Give SL as a PERCENTAGE (e.g. 3 = 3% away from entry). If the user said '3% SL', use THIS — do not compute the price yourself and do NOT call get_market_price; the server computes it from the live entry. Cannot be combined with `sl`. If the user did NOT state a value, LEAVE IT EMPTY: the server applies the SL percentage saved in the user's AlgoVesta account (the same default used everywhere else in the product). Producing your own value overrides the user's setting.",
      "title": "Sl Pct"
    },
    "tp_pct": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Give TP as a PERCENTAGE (e.g. 5 = 5% away from entry). If the user said '5% TP', use THIS — do not compute the price yourself. Cannot be combined with `tp`. If the user did not state one, LEAVE IT EMPTY — the user's saved TP percentage is applied.",
      "title": "Tp Pct"
    },
    "account": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "TARGET ACCOUNT — the account this call acts on (for place_order/simulate_order the account the order is opened in; for close/modify the account holding the position). For MT5 the login number (e.g. '12345678'), for exchanges the account label (e.g. 'Binance2'). Copy it EXACTLY from the 'account' field in the get_portfolio_context output. REQUIRED when the user has more than one account in the same market: if left empty the action is NOT performed and the user is asked which account — it is never silently applied to the default/primary account. Never choose without the user saying so.",
      "title": "Account"
    }
  },
  "required": [
    "venue",
    "symbol",
    "side",
    "order_type",
    "idempotency_key"
  ],
  "title": "place_orderArguments",
  "type": "object"
}
```

</details>

## compile_policy

**Title:** Compile policy &nbsp;·&nbsp; **Scope:** `read` &nbsp;·&nbsp; **Kind:** read-only<br>
**Annotations:** `readOnlyHint=true` `destructiveHint=false` `idempotentHint=true`

Compiles natural-language risk rules into a JSON policy and returns a PREVIEW.
Activation requires separate approval: from the panel or via
POST /api/mcp/policies/{policy_id}/activate.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "natural_text": {
      "description": "The user's risk rules in plain language, e.g. 'never risk more than 2% per trade, max 3 open positions, no leverage above 10x, long only, BTC and ETH only'.",
      "title": "Natural Text",
      "type": "string"
    }
  },
  "required": [
    "natural_text"
  ],
  "title": "compile_policyArguments",
  "type": "object"
}
```

</details>

## list_open_orders

**Title:** Pending orders &nbsp;·&nbsp; **Scope:** `read` &nbsp;·&nbsp; **Kind:** read-only<br>
**Annotations:** `readOnlyHint=true` `destructiveHint=false` `idempotentHint=true`

Lists the user's pending limit orders (order_ref, venue, symbol, side,
entry_price, size, created_at). Only PAPER limit orders can rest here: live
crypto venues accept market orders only (a live crypto limit order is refused
with live_limit_not_supported, never silently converted), so an empty list on
a live account means nothing is pending — not that something disappeared.

VERIFICATION: call get_portfolio_context / list_open_orders WITHOUT asking for
confirmation (they are read operations). Ask the user only ONE confirmation
question.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "venue": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Optional venue filter (e.g. paper, binance, mt5). If omitted, all pending orders are returned.",
      "title": "Venue"
    }
  },
  "title": "list_open_ordersArguments",
  "type": "object"
}
```

</details>

## cancel_order

**Title:** Cancel order &nbsp;·&nbsp; **Scope:** `paper|live` &nbsp;·&nbsp; **Kind:** destructive (asks for confirmation in capable clients)<br>
**Annotations:** `readOnlyHint=false` `destructiveHint=true` `idempotentHint=true`

Cancels a pending limit order. If order_ref does not belong to the user,
NOT_FOUND is returned (no information about another user's order is disclosed).
idempotency_key is REQUIRED: calling again with the same key returns the stored
response. This is a RISK-REDUCING operation: the policy wall does not block it
(except the kill switch).

VERIFICATION: call list_open_orders WITHOUT asking for confirmation (it is a read
operation), then cancel with ONE confirmation. Never choose an order the user did
not name.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "venue": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Exchange/venue name — must be one of the user's CONNECTED accounts (e.g. binance, bybit, okx, mt5, paper). Never choose one the user did not name.",
      "title": "Venue"
    },
    "order_ref": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Reference of the pending order to cancel (taken from list_open_orders). Never choose an order the user did not name.",
      "title": "Order Ref"
    },
    "idempotency_key": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Client-generated unique idempotency key (min 8 characters). Reuse the SAME key when retrying the same action — the stored response is replayed and the action is NOT performed a second time.",
      "title": "Idempotency Key"
    }
  },
  "required": [
    "venue",
    "order_ref",
    "idempotency_key"
  ],
  "title": "cancel_orderArguments",
  "type": "object"
}
```

</details>

## close_position

**Title:** Close position &nbsp;·&nbsp; **Scope:** `paper|live` &nbsp;·&nbsp; **Kind:** destructive (asks for confirmation in capable clients)<br>
**Annotations:** `readOnlyHint=false` `destructiveHint=true` `idempotentHint=true`

Closes an open position fully or partially (fraction (0,1]).
Works for crypto and FOREX/MT5. This is a RISK-REDUCING operation: the policy wall
NEVER blocks this tool (except the kill switch). idempotency_key is REQUIRED: even
10 calls with the same key perform ONE close. If no position exists,
POSITION_NOT_FOUND is returned together with the open positions on that venue.

MT5 positions can only be closed IN FULL (no partial close) -> use fraction=1. If several
MT5 positions are open on the same symbol, `ticket` becomes REQUIRED; while it is
ambiguous, none are closed.

VERIFICATION: call get_portfolio_context / list_open_orders WITHOUT asking for
confirmation (they are read operations). Ask the user only ONE confirmation
question.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "venue": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Exchange/venue name — must be one of the user's CONNECTED accounts (e.g. binance, bybit, okx, mt5, paper). Never choose one the user did not name.",
      "title": "Venue"
    },
    "symbol": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Trading symbol, e.g. BTCUSDT, ETHUSDT, EURUSD.",
      "title": "Symbol"
    },
    "side": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "'buy' (long) or 'sell' (short). Never choose without an EXPLICIT user instruction.",
      "title": "Side"
    },
    "idempotency_key": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Client-generated unique idempotency key (min 8 characters). Reuse the SAME key when retrying the same action — the stored response is replayed and the action is NOT performed a second time.",
      "title": "Idempotency Key"
    },
    "fraction": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": 1.0,
      "description": "Close fraction (0,1]. 1.0 = full close, 0.5 = half. MT5 positions can only be closed in full — use 1.0 there; a partial close on MT5 is refused rather than silently converted to a full close.",
      "title": "Fraction"
    },
    "account": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "TARGET ACCOUNT — the account this call acts on (for place_order/simulate_order the account the order is opened in; for close/modify the account holding the position). For MT5 the login number (e.g. '12345678'), for exchanges the account label (e.g. 'Binance2'). Copy it EXACTLY from the 'account' field in the get_portfolio_context output. REQUIRED when the user has more than one account in the same market: if left empty the action is NOT performed and the user is asked which account — it is never silently applied to the default/primary account. Never choose without the user saying so.",
      "title": "Account"
    },
    "ticket": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "FOREX/MT5 only: the ticket number of the position this action targets (the 'ticket' field on forex positions in get_portfolio_context). REQUIRED when several positions are open on the same symbol — if which one is meant is not certain, nothing is done. May be left empty when there is only one position.",
      "title": "Ticket"
    }
  },
  "required": [
    "venue",
    "symbol",
    "side",
    "idempotency_key"
  ],
  "title": "close_positionArguments",
  "type": "object"
}
```

</details>

## modify_position

**Title:** Modify SL/TP &nbsp;·&nbsp; **Scope:** `paper|live` &nbsp;·&nbsp; **Kind:** write<br>
**Annotations:** `readOnlyHint=false` `destructiveHint=false` `idempotentHint=true`

Updates the SL/TP of an open position — works for crypto and FOREX/MT5.
AT LEAST ONE of new_sl/new_tp must be given. SL CANNOT BE REMOVED (the
mandatory-SL rule stands). Logic: long requires new_sl < mark < new_tp, short the
reverse. idempotency_key is REQUIRED.

If you send only ONE side, the other keeps the position's CURRENT value — the side
you omit is NOT deleted. If several MT5 positions are open on the same symbol,
`ticket` becomes REQUIRED; while it is ambiguous, NONE are modified.

VERIFICATION: call get_portfolio_context WITHOUT asking for confirmation (it is a
read operation), then apply with ONE confirmation. Never change levels the user did
not specify.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "venue": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Exchange/venue name — must be one of the user's CONNECTED accounts (e.g. binance, bybit, okx, mt5, paper). Never choose one the user did not name.",
      "title": "Venue"
    },
    "symbol": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Trading symbol, e.g. BTCUSDT, ETHUSDT, EURUSD.",
      "title": "Symbol"
    },
    "side": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "'buy' (long) or 'sell' (short). Never choose without an EXPLICIT user instruction.",
      "title": "Side"
    },
    "idempotency_key": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Client-generated unique idempotency key (min 8 characters). Reuse the SAME key when retrying the same action — the stored response is replayed and the action is NOT performed a second time.",
      "title": "Idempotency Key"
    },
    "new_sl": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "New SL price. SL CANNOT BE REMOVED (0/negative is rejected) - the mandatory-SL rule stands. At least one of new_sl or new_tp must be given.",
      "title": "New Sl"
    },
    "new_tp": {
      "anyOf": [
        {
          "type": "number"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "New TP price. At least one of new_sl or new_tp must be given.",
      "title": "New Tp"
    },
    "account": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "TARGET ACCOUNT — the account this call acts on (for place_order/simulate_order the account the order is opened in; for close/modify the account holding the position). For MT5 the login number (e.g. '12345678'), for exchanges the account label (e.g. 'Binance2'). Copy it EXACTLY from the 'account' field in the get_portfolio_context output. REQUIRED when the user has more than one account in the same market: if left empty the action is NOT performed and the user is asked which account — it is never silently applied to the default/primary account. Never choose without the user saying so.",
      "title": "Account"
    },
    "ticket": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "FOREX/MT5 only: the ticket number of the position this action targets (the 'ticket' field on forex positions in get_portfolio_context). REQUIRED when several positions are open on the same symbol — if which one is meant is not certain, nothing is done. May be left empty when there is only one position.",
      "title": "Ticket"
    }
  },
  "required": [
    "venue",
    "symbol",
    "side",
    "idempotency_key"
  ],
  "title": "modify_positionArguments",
  "type": "object"
}
```

</details>

## verify_receipt

**Title:** Verify receipt &nbsp;·&nbsp; **Scope:** `read` &nbsp;·&nbsp; **Kind:** read-only<br>
**Annotations:** `readOnlyHint=true` `destructiveHint=false` `idempotentHint=true`

Verifies an order receipt: ed25519 signature + per-user hash chain.
Returns signature_valid + chain_valid (verified = both True). If an earlier receipt
in the chain was altered, chain_valid=False (proof of tamper-evidence).
Public key: /mcp/receipts/pubkey. Receipts issued before the ed25519 rollout
were HMAC-signed and return legacy=True.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "receipt_id": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "receipt_id of the receipt to verify (receipt.receipt_id from a place/close/cancel response). Only your own receipts can be verified.",
      "title": "Receipt Id"
    }
  },
  "required": [
    "receipt_id"
  ],
  "title": "verify_receiptArguments",
  "type": "object"
}
```

</details>

## replay_channel

**Title:** Replay channel &nbsp;·&nbsp; **Scope:** `read` &nbsp;·&nbsp; **Kind:** read-only<br>
**Annotations:** `readOnlyHint=true` `destructiveHint=false` `idempotentHint=true`<br>
**Rate limit:** 5/hour

Answers "what if you had followed this Telegram channel for the last N days
under these rules?". Simulates the channel's past signals paper-style; every signal
passes the policy wall (rejected ones are not opened). Progress is published as
replay_progress events. Results are cached for 24 hours (the same channel +
parameters return from cache). Rate limit: 5 replays/hour.

Output: {trades:[...], summary:{total_pnl, win_rate, max_drawdown, avg_rr,
policy_rejections}}.

<details>
<summary>Input schema (JSON Schema)</summary>

```json
{
  "properties": {
    "channel_ref": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "description": "Reference of the Telegram channel to analyze (a channel the user follows or knows). Never choose a channel the user did not name.",
      "title": "Channel Ref"
    },
    "days": {
      "default": 30,
      "description": "Look-back window in days (1-90). The channel's signals in that window are simulated.",
      "title": "Days",
      "type": "integer"
    },
    "policy_override": {
      "anyOf": [
        {
          "additionalProperties": true,
          "type": "object"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Optional policy rules (same schema as the compile_policy output). If given, the simulation runs with these rules instead of the active policy; if omitted, the user's active policy applies.",
      "title": "Policy Override"
    }
  },
  "required": [
    "channel_ref"
  ],
  "title": "replay_channelArguments",
  "type": "object"
}
```

</details>

