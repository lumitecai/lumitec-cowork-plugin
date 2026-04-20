# lumitec-amazon Cowork plugin

Two MCP servers for Amazon operations, connected as **remote HTTP MCPs** hosted on Fly.io:

| MCP | URL | Purpose |
|-----|-----|---------|
| `amazon-sp-api`  | `https://lumitec-sp-api-mcp.fly.dev/mcp`  | Orders, listings, inventory, pricing, catalog, finance, FBA, reports, feeds |
| `amazon-ads-api` | `https://lumitec-ads-api-mcp.fly.dev/mcp` | Sponsored Products / Brands / Display, DSP, reporting v3 |

Credentials are **per-request**, pass-through: each MCP call carries your SP-API / Ads API keys in HTTP headers. The server is stateless — nothing is persisted for your account. No OAuth flow, no bundled binaries, no cross-compile.

## Install

```bash
claude plugin marketplace add /Users/aw/lumitec-cowork-plugin --scope user
claude plugin install lumitec-amazon@lumitec-amazon
```

Or, once pushed to GitHub:

```bash
claude plugin marketplace add https://github.com/lumitec-ltd/lumitec-cowork-plugin --scope user
claude plugin install lumitec-amazon@lumitec-amazon
```

## Secrets the user must create in Cowork

Open Cowork → Settings → Secrets → "Add generic secret". Cowork encrypts these per-user and injects them as env vars; the plugin's HTTP headers are then expanded from those env vars via `${NAME}` syntax at MCP connect time.

### Shared — required for all users

| Secret name        | Notes                                                           |
|--------------------|-----------------------------------------------------------------|
| `LUMITEC_MCP_KEY`  | Shared access key issued by Lumitec; gates both Fly endpoints. Ask support@lumitec.ai for your value. |

### SP-API

| Secret name              | Example / notes                                           |
|--------------------------|-----------------------------------------------------------|
| `SP_API_CLIENT_ID`       | `amzn1.application-oa2-client.…`                          |
| `SP_API_CLIENT_SECRET`   | `amzn1.oa2-cs.v1.…`                                       |
| `SP_API_REFRESH_TOKEN`   | `Atzr\|IwE…` (seller-authorised refresh token)            |
| `SP_API_REGION`          | `na` / `eu` / `fe`  (default: `eu`)                       |
| `SP_API_MARKETPLACE_ID`  | e.g. `A1F83G8C2ARO7P` (UK) — primary marketplace          |
| `SP_API_SELLER_ID`       | e.g. `A3JEKG1WEL1FC`                                      |

### Ads API

| Secret name              | Example / notes                                     |
|--------------------------|-----------------------------------------------------|
| `ADS_API_CLIENT_ID`      | `amzn1.application-oa2-client.…`                    |
| `ADS_API_CLIENT_SECRET`  | `amzn1.oa2-cs.v1.…`                                 |
| `ADS_API_REFRESH_TOKEN`  | `Atzr\|IwE…`                                        |
| `ADS_API_REGION`         | `na` / `eu` / `fe`  (default: `eu`)                 |

Restart any running agent after adding or changing secrets.

## How it works

```
Cowork VM ─── HTTPS + X-*-Client-Id / …-Refresh-Token ───► Fly.io MCP server ──► Amazon APIs
                                                              (stateless, multi-tenant)
```

- Each MCP invocation includes your credentials in headers.
- The server derives a per-tenant `identityId` from a SHA-256 of `(clientId, refreshToken)` so its LWA access-token cache is isolated per user — one tenant's token never serves another tenant's request.
- Nothing is stored server-side. Credentials live only in your Cowork secrets vault.

## Known gaps

- **No per-op gating.** Destructive Amazon tools (delete listing, cancel order, archive campaign, etc.) are not gated at the plugin layer — the Cowork `clis.commands` confirmation block isn't accepted by the public plugin.json schema. If you need it, add Claude permission rules that match `mcp__amazon-*` tools.
- **No OAuth.** Users bring their own refresh token. If you want a Connect button UX, that's a v3 change — see the internal plan doc.

## Custom domains (optional)

The default URLs are `*.fly.dev`. To use branded domains (e.g. `sp-api-mcp.lumitec.ai`):

```bash
fly certs create sp-api-mcp.lumitec.ai   -a lumitec-sp-api-mcp
fly certs create ads-api-mcp.lumitec.ai  -a lumitec-ads-api-mcp
```

Then add the CNAME records Fly prints to your DNS, and update the `url` fields in `plugin.json`.

## Changelog

- **2.0.0** — rewrite to remote HTTP MCPs on Fly.io. Removed 380 MB of bundled binaries. Multi-tenant, stateless, pass-through credentials.
- **1.0.0** — bundled Linux musl binaries (deprecated).
