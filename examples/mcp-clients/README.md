# Calling the AlgoVesta MCP server from your own code

The server speaks standard MCP over HTTP, so any MCP client library works. These two minimal programs connect with a secret link, list the tools, read the portfolio, get a quote and place a **paper** order (`venue: "paper"`; use a connected exchange name such as `binance` or `mt5` for a real account). Replace `avmcp_<your-key>` with the key from the dashboard; a `read` or `paper` scoped key is enough for everything here.

Tool names and schemas are in [docs/TOOLS.md](../../docs/TOOLS.md). Every order goes through the server-side policy wall and a stop-loss is required; a request that violates your rules is rejected with a reason, not silently adjusted.

## Python (`mcp` SDK)

```python
import asyncio
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

URL = "https://api.algovesta.com/u/avmcp_<your-key>/mcp"

async def main():
    async with streamablehttp_client(URL) as (read, write, _):
        async with ClientSession(read, write) as s:
            await s.initialize()
            tools = await s.list_tools()
            print([t.name for t in tools.tools])          # 20 tools

            ctx = await s.call_tool("get_portfolio_context", {})
            print(ctx.content[0].text)                     # balances, open positions, risk state

            q = await s.call_tool("get_market_price", {"venue": "paper", "symbol": "BTCUSDT"})
            print(q.content[0].text)

            order = await s.call_tool("place_order", {
                "venue": "paper",
                "symbol": "BTCUSDT",
                "side": "buy",
                "order_type": "market",
                "size_usd": 50,
                "sl_pct": 2,
                "tp_pct": 4,
                "idempotency_key": "example-0001",
            })
            print(order.content[0].text)                   # paper fill + signed receipt

asyncio.run(main())
```

Install: `pip install mcp`.

## Node.js (`@modelcontextprotocol/sdk`)

```js
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

const url = new URL("https://api.algovesta.com/u/avmcp_<your-key>/mcp");
const client = new Client({ name: "example", version: "1.0.0" });
await client.connect(new StreamableHTTPClientTransport(url));

const { tools } = await client.listTools();
console.log(tools.map(t => t.name));

const ctx = await client.callTool({ name: "get_portfolio_context", arguments: {} });
console.log(ctx.content[0].text);

const order = await client.callTool({
  name: "place_order",
  arguments: { venue: "paper", symbol: "BTCUSDT", side: "buy", order_type: "market", size_usd: 50, sl_pct: 2, tp_pct: 4, idempotency_key: "example-0001" },
});
console.log(order.content[0].text);
```

Install: `npm install @modelcontextprotocol/sdk`.

## Notes

- Paper is the default: a new key starts on a paper balance, and `live` scope needs 2FA on the dashboard.
- `idempotency_key` makes a retried request return the original receipt instead of a second order.
- Receipts are signed; verify them offline with the public key at `https://api.algovesta.com/mcp/receipts/pubkey` or with the `verify_receipt` tool.
- Rate limits and timeouts are listed in the README under "Performance and limits".
