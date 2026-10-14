# TradingView webhook alert templates

Copy one of these JSON messages into a TradingView alert, paste your AlgoVesta webhook URL from the dashboard, and the alert becomes an order on the exchange or MetaTrader 5 account you selected for that strategy.

The templates are the ones the AlgoVesta webhook receiver accepts today. They are published here so you can version them, diff them and test them outside the dashboard.

## How the receiver reads a message

| Field | Required | Meaning |
|---|---|---|
| `symbol` | yes | `{{ticker}}` or a literal such as `BTCUSDT`, `EURUSD`, `XAUUSD`. Exchange suffixes and MT5 broker suffixes are resolved on the server. |
| `action` | yes | `buy`, `sell`, `long`, `short` or `close`. From a Pine strategy use `{{strategy.order.action}}` together with the position fields (see `pine-strategy.json`). |
| `price` | recommended | `{{close}}`. Used as the entry reference and for position sizing; the fill price comes from the venue. |
| `stop_loss` | recommended | Absolute price. If omitted, the stop-loss percentage configured on the dashboard applies. A stop-loss is always placed on MetaTrader 5. |
| `take_profit` | optional | Single absolute price. |
| `take_profits` | optional | Array of `{"price": ..., "close_pct": ...}`. Percentages are capped at 100 in total. On MetaTrader 5 each target becomes its own position with the same stop-loss. |
| `leverage` | optional | Integer, crypto futures only. Capped at the dashboard setting and the exchange bracket. |
| `amount_usd` | optional | Margin in USD for the order, crypto only. Default: the dashboard setting. |
| `lots` | optional | Lot size for MetaTrader 5. Default: the lot mode configured on the dashboard. |
| `market` | optional | `crypto` or `forex`. Default: the target configured for the strategy on the dashboard. |
| `market_position`, `prev_market_position`, `position_size` | Pine strategies | Lets the receiver tell an exit from a short entry. Without them, a strategy's `sell` on exit would open a short. |

Fields that are present but not listed above are ignored. An `exchange` field in the message is ignored too: the destination account is chosen on the dashboard, not in the alert.

## What TradingView cannot do

TradingView alerts send only the URL and the message body. They cannot set HTTP headers, so the AlgoVesta receiver does not ask for an HMAC signature from TradingView. The webhook URL itself is the secret: it is a 256-bit random token, and the sender IP is checked against the published TradingView webhook addresses. Treat the URL like a password and rotate it from the dashboard if it leaks.

## Templates

| File | Use |
|---|---|
| `basic-buy.json` | Indicator alert, market entry with absolute stop-loss and take-profit. |
| `multi-take-profit.json` | Three targets closing 50 / 30 / 20 percent. |
| `pine-strategy.json` | Pine `strategy()` alert with position-state fields; entries, exits and reversals are derived from the position. |
| `close.json` | Close the open position for a symbol from any alert type. |
| `forex-mt5.json` | Forex or gold alert routed to MetaTrader 5 with a fixed lot. |
| `crypto-margin-leverage.json` | Crypto futures alert with explicit margin and leverage. |

Test every template with the dashboard's "Send test signal" before saving the alert: the test runs the full parse and risk checks without placing an order.
