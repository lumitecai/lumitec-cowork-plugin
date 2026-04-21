---
name: setup-amazon
description: |
  Guide a user through connecting the Lumitec Amazon MCPs (SP-API and
  Advertising API) to their Claude client. Collect credentials conversationally,
  validate their shape, optionally verify live via the existing MCP, and
  produce a ready-to-paste claude_desktop_config.json snippet with the
  correct OS-specific path instructions. Use this whenever the user asks to
  "set up Amazon", "connect Amazon", "configure the Amazon MCP", "install
  the Lumitec Amazon plugin", or similar.
---

# Setup Lumitec Amazon MCP

You are walking a user through connecting their Amazon seller account to
Claude via the Lumitec hosted MCP servers. Your goal: produce a correct,
copy-paste-ready JSON snippet they can put into their Claude config file.

## How to run this flow

1. **Greet briefly** and tell them this will take ~5 minutes if they already
   have their Amazon credentials in hand. If they don't have an SP-API /
   Advertising API developer account set up yet, point them to
   <https://developer-docs.amazon.com/sp-api/> first — you can't generate
   those credentials for them.

2. **Collect credentials one block at a time** (don't ask for everything
   at once — people lose track). Ask for each field, verify the shape,
   confirm back. Use the grouping below.

3. **After each field**, validate its shape before moving on (see
   "Validation" section). If it doesn't match, ask them to double-check
   and paste again — don't silently accept malformed values.

4. **Optionally verify live** — if the Lumitec Amazon MCPs are already
   connected in the current Claude session (you'll see tools named
   `checkCredentials`, `getMarketplaceParticipations`, etc. in your tool
   list), offer to run `checkCredentials` with their values to confirm
   they work before writing the final config. If the tools aren't
   available — this is the user's first install — skip verification;
   just produce the config.

5. **Produce the final JSON block** using the template below, with their
   values substituted.

6. **Tell them where to paste it** based on their OS (ask if unclear).

7. **Tell them to fully quit Claude** (⌘Q on Mac, Alt+F4 on Windows) and
   reopen. The first launch takes ~10 seconds while `npx` downloads
   `mcp-remote`.

8. **Offer to verify** after they restart — they can come back and say
   "check my amazon connection" and you'll run `checkCredentials` on
   both MCPs to confirm.

## Credentials to collect

### Block 1 — Shared

- **Lumitec access key** — issued by Lumitec support (email
  `a.walters@lumitec.ai` if they don't have one). Format: starts with
  `lk_`, followed by ~42 chars of base64url (letters, digits, `-`, `_`).

### Block 2 — Amazon SP-API

- **Client ID** — from Amazon Developer Central → Your SP-API app → LWA
  Credentials. Format: `amzn1.application-oa2-client.<32 hex chars>`.
- **Client Secret** — same page, revealed once at app creation. Format:
  `amzn1.oa2-cs.v1.<64 hex chars>`. Treat as a password.
- **Refresh Token** — from the seller-authorization OAuth flow. Format:
  `Atzr|<long base64url>`. Region-locked: a token minted for EU won't
  work with `region: na`.
- **Region** — `eu` | `na` | `fe`. Must match the refresh token.
- **Marketplace ID** — default marketplace for tools that need one.
  Common values:
  - UK: `A1F83G8C2ARO7P`
  - Germany: `A1PA6795UKMFR9`
  - France: `A13V1IB3VIYZZH`
  - Italy: `APJ6JRA9NG5V4`
  - Spain: `A1RKKUPIHCS9HS`
  - US: `ATVPDKIKX0DER`
  - Canada: `A2EUQ1WTGCTBG2`
  - Japan: `A1VC38T7YXB528`
  - Australia: `A39IBJ37TRP1C6`
- **Seller ID** — Seller Central → Settings → Account Info → Merchant
  Token. Format: starts with `A`, ~14 chars, all uppercase letters and
  digits (e.g. `A3JEKG1WEL1FC`).

### Block 3 — Amazon Advertising API (optional — skip if user only uses SP-API)

- **Client ID** — from the Advertising API console (different app than
  SP-API). Same format as SP-API Client ID.
- **Client Secret** — same format as SP-API Client Secret.
- **Refresh Token** — same `Atzr|…` format. Also region-locked.
- **Region** — `eu` | `na` | `fe`.

## Validation (do this as each field comes in)

| Field                 | Must start with                          | Must be length |
|-----------------------|------------------------------------------|----------------|
| Lumitec key           | `lk_`                                    | ~45 chars      |
| SP-API Client ID      | `amzn1.application-oa2-client.`          | ~64 chars      |
| SP-API Client Secret  | `amzn1.oa2-cs.v1.`                       | ~80 chars      |
| SP-API Refresh Token  | `Atzr|`                                  | >200 chars     |
| SP-API Seller ID      | `A`, uppercase letters/digits only       | ~13 chars      |
| Ads Client ID         | `amzn1.application-oa2-client.`          | ~64 chars      |
| Ads Client Secret     | `amzn1.oa2-cs.v1.`                       | ~80 chars      |
| Ads Refresh Token     | `Atzr|`                                  | >200 chars     |
| Region                | Exactly `eu`, `na`, or `fe`              | 2 chars        |
| Marketplace ID        | `A`, uppercase letters/digits only       | 12–14 chars    |

If any validation fails, **don't proceed silently** — politely ask them
to double-check that specific value. A common mistake is pasting a
trailing newline or a truncated refresh token.

## Final JSON template

Emit the final config as a single code block, formatted for readability.
Use **double quotes** and valid JSON (no trailing commas, no comments).

```json
{
  "mcpServers": {
    "amazon-sp-api": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote@latest",
        "https://lumitec-sp-api-mcp.fly.dev/mcp",
        "--header", "X-Lumitec-Key:<LUMITEC_KEY>",
        "--header", "X-SP-API-Client-Id:<SP_CLIENT_ID>",
        "--header", "X-SP-API-Client-Secret:<SP_CLIENT_SECRET>",
        "--header", "X-SP-API-Refresh-Token:<SP_REFRESH_TOKEN>",
        "--header", "X-SP-API-Region:<SP_REGION>",
        "--header", "X-SP-API-Marketplace-Id:<SP_MARKETPLACE_ID>",
        "--header", "X-SP-API-Seller-Id:<SP_SELLER_ID>"
      ]
    },
    "amazon-ads-api": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote@latest",
        "https://lumitec-ads-api-mcp.fly.dev/mcp",
        "--header", "X-Lumitec-Key:<LUMITEC_KEY>",
        "--header", "X-Ads-Client-Id:<ADS_CLIENT_ID>",
        "--header", "X-Ads-Client-Secret:<ADS_CLIENT_SECRET>",
        "--header", "X-Ads-Refresh-Token:<ADS_REFRESH_TOKEN>",
        "--header", "X-Ads-Region:<ADS_REGION>"
      ]
    }
  }
}
```

If the user skipped the Ads API block, omit the `amazon-ads-api` entry
entirely (don't emit it with empty values).

**Important:** tell the user that if their config file already contains
other MCPs, they must **merge** these two keys into the existing
`mcpServers` object — not replace the whole file. Show them what their
existing file probably looks like so they can see how to slot ours in.

## OS-specific paths

| OS          | Config file location                                              |
|-------------|-------------------------------------------------------------------|
| **macOS**   | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json`                     |
| **Linux**   | `~/.config/Claude/claude_desktop_config.json`                     |

If the user is on Mac and has a terminal handy, give them:
```
open -a TextEdit ~/Library/Application\ Support/Claude/claude_desktop_config.json
```

If on Windows:
```
notepad %APPDATA%\Claude\claude_desktop_config.json
```

## If the user is in Claude Code (not Desktop)

If they mention they're using `claude` CLI / Claude Code (not the
Desktop app), the install path is different — the plugin is already
installed and uses `${VAR}` expansion. Point them to:

- `claude plugin install lumitec-amazon@lumitec-amazon` (if not yet installed)
- Set the required env vars in their shell: `SP_API_CLIENT_ID=…` etc.
  before launching `claude`
- Or use `claude mcp add --transport http --header 'X-Lumitec-Key:…' …`

The stdio-bridge approach in the main JSON template is specifically for
Claude **Desktop** and **Cowork**.

## Prerequisites to mention upfront

- **Node.js installed** — required by `npx mcp-remote`. If missing,
  point to <https://nodejs.org/en/download>.
- **An Amazon Developer account** with an SP-API application registered
  and a completed seller-authorization flow. This is the 30–60 minute
  hard part of onboarding; you cannot do it for them.
- **A Lumitec access key** (`LUMITEC_MCP_KEY`) issued to them by
  Lumitec support.

## After they restart

Offer to verify with one or both of:

- `checkCredentials` on the SP-API MCP → returns the active region +
  token preview if the creds are valid.
- `checkCredentials` on the Ads API MCP → same pattern.

If either fails, common fixes:
- "Missing or invalid X-Lumitec-Key header" → wrong Lumitec key.
- "Failed to authenticate with Amazon" → wrong Amazon Client ID / Secret /
  Refresh Token combo, or the refresh token is for a different region
  than the one configured.
- Tools don't appear at all after restart → the JSON has a syntax error.
  Walk them through validating at <https://jsonlint.com>.

## Tone

Be direct and efficient. This is setup work, not a tutorial — don't
over-explain. Ask one thing at a time, move fast. Users copying
credentials out of Amazon's developer portal are usually in a hurry
and slightly confused; your job is to make the path obvious.
