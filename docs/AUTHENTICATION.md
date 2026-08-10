# Authentication and scopes

There are two ways to connect, and both resolve to the same tenant context. Tools never accept a user ID as a parameter — identity is read only from the authenticated connection, so there is no request field through which one account could name another.

## Secret link

A key of the form `avmcp_<32-byte urlsafe random>`, embedded in the URL path:

```
https://api.algovesta.com/u/avmcp_<your-key>/mcp
```

- Stored as an **Argon2id hash** plus a SHA-256 lookup hash; the plaintext exists only at the moment of creation and is never recoverable afterwards.
- Each key carries its own scope, label and revocation state — run one paper key in Cursor and one live key in Claude, and kill either independently.
- Revocation takes effect immediately for new connections and drops open event streams within seconds.

Keys are created in the AlgoVesta panel (**MCP Connection** tab). Creating a `live` key requires a second factor: a TOTP authenticator code, or a confirmation code sent to the account email (valid 10 minutes). This is enforced server-side, with no exceptions.

## OAuth 2.1

For clients that prefer a proper authorization flow:

```
https://api.algovesta.com/mcp
```

- Grants: `authorization_code` and `refresh_token`, with rotating refresh tokens.
- **PKCE with `S256` is mandatory** — a request without it is rejected.
- **Dynamic Client Registration** (RFC 7591) is available, so most clients configure themselves.
- The consent page defaults to the `paper` scope; `live` over OAuth additionally requires TOTP to be enabled on the account.

Discovery documents:

```
GET  https://api.algovesta.com/.well-known/oauth-authorization-server
GET  https://api.algovesta.com/.well-known/oauth-protected-resource
```

OAuth endpoints:

```
POST https://api.algovesta.com/mcp/oauth/register
GET  https://api.algovesta.com/mcp/oauth/authorize
POST https://api.algovesta.com/mcp/oauth/token
```

## The three scopes

| Scope | What it can do | How you get it |
|---|---|---|
| `read` | Portfolio, prices, pending orders, simulations, policy previews, receipt verification, channel replay. **No order can be placed.** | Created directly. |
| `paper` | Everything in `read`, plus orders executed against the paper engine on a $5,000 virtual balance. **The default for new keys.** | Created directly. |
| `live` | Everything above, plus real orders on your connected exchanges and MetaTrader 5 accounts. | **Second factor required** (TOTP, or an email code for panel-created keys; OAuth requires TOTP). Enforced server-side. |

Scopes are ranked: a tool requiring `paper` refuses a `read` key with `insufficient_scope`. The boundary between simulated and real money is a property of the key itself — not of a prompt, a setting, or the model's judgment.

## Unauthenticated requests

A request without valid credentials receives `401`. On the OAuth endpoint (`/mcp`) the response also carries a `WWW-Authenticate` header pointing at the protected-resource metadata, per the MCP authorization specification.
