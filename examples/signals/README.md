# Telegram signal parsing test corpus

`corpus.json` holds 30 synthetic Telegram signal messages with the parse result a signal copier should produce for each of them. Use it to test your own parser, or to see which formats AlgoVesta's parser is built to handle.

None of the messages is copied from a real channel. They imitate the formats that appear in public channels: free text, labelled lines, emoji bullets, entry zones, numbered take-profit lists, pending orders, management messages and plain chatter.

## Fields

| Field | Values |
|---|---|
| `symbol` | Canonical symbol, upper case, no broker suffix. `GOLD` and `Gold` resolve to `XAUUSD`; `#SOL` and `SOL/USDT` resolve to `SOLUSDT`. |
| `side` | `buy`, `sell`, or `null` for management messages. |
| `entry` / `entry_range` | A single price, or a `[low, high]` zone. A zone is normalised so that `low <= high` regardless of the order in the message. |
| `sl` | Stop-loss price, or the new stop level for a `move_sl` message. |
| `tp_list` | Take-profit prices in the order given. |
| `order_type` | `market` when the message says "now" or gives a single price near the market, `limit` for a zone, `pending` for an explicit buy/sell limit or stop. |
| `intent` | `entry`, `close`, `close_partial`, `move_sl` or `not_a_signal`. |

## Cases worth testing

- Thousands separators (`3,420`) and decimal commas must not break prices.
- `SL.` and `TP.` with a trailing dot, and `TP1`/`TP 1`/`Take profit 1` variants.
- Numbered target lists (`1) 2633`) where the index must not be read as a price.
- Result posts (`TP1 hit`, `+40 pips`) and recaps are **not** entries, even though they mention a symbol and a number.
- A `close half` message is a partial close; the copier that cannot do partial closes should say so rather than closing everything silently.

## How AlgoVesta uses this

AlgoVesta's parser targets the same schema and is measured against a live corpus before every change, with the rule that a parser change may not lose any entry that was previously found. Parsing is only the first step: every parsed signal then passes the account's risk rules (stop-loss required, daily loss limit, position caps) before an order is placed on the exchange or MetaTrader 5.
