# Security policy

## Reporting a vulnerability

Report suspected vulnerabilities to **support@algovesta.com** with `[SECURITY]` in the subject line. Include reproduction steps and the endpoint involved. You will receive an acknowledgement, and we ask that you give us reasonable time to remediate before public disclosure.

## Scope

- **In scope:** the hosted MCP endpoints (`api.algovesta.com/mcp`, `api.algovesta.com/u/.../mcp`), the OAuth flow, the panel API under `/api/mcp/*`, and the documentation in this repository.
- **Out of scope:** testing against accounts you do not own, any attempt to access another tenant's data, denial-of-service traffic, and social engineering of AlgoVesta staff or customers.

Please test only with your own account and paper-scope keys. The paper engine exists precisely so that the full workflow can be exercised with zero real-money risk.

## Key handling

- The MCP connection URL is a credential. Never publish it — not in issues in this repository, screenshots, shared configs or support tickets. Config examples here always use the `avmcp_<your-key>` placeholder.
- If a key leaks, revoke it in the panel (takes effect immediately for new connections; open event streams drop within seconds), or freeze all MCP activity with the kill switch.
- Live scope always requires a second factor and can be revoked independently of paper keys.

## Verifying server actions

Every action returns an ed25519-signed, hash-chained receipt. The signing public key is served without authentication at:

```
https://api.algovesta.com/mcp/receipts/pubkey
```

so receipts can be verified independently of the server that issued them.

## About this repository

This repository contains documentation and client configuration examples only. The server is hosted and closed-source; no server code, credentials or infrastructure details live here.
