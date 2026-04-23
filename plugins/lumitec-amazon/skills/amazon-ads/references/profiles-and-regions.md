# Profiles & Regions

The **profile** is the advertiser × country identity. One Amazon Ads
account has one profile per country it runs ads in. The `profileId` is passed
as the `Amazon-Advertising-API-Scope` header on every call except the
profile-listing endpoint itself.

## The three regions

| Region | Endpoint host | Countries |
|---|---|---|
| `na` | `advertising-api.amazon.com` | US, CA, MX, BR |
| `eu` | `advertising-api-eu.amazon.com` | UK, DE, FR, IT, ES, NL, SE, PL, TR, AE, EG, SA, IN (via separate token — see below) |
| `fe` | `advertising-api-fe.amazon.com` | JP, AU, SG |

India (`IN`) has historically been in `eu` for Ads but SP-API puts it in
`fe`. If you get 404s on IN profiles, try `searchProfiles` with
`countryCode: 'IN'` across both regions and use whichever returns the match.

**Global seat (since 2025)**: Amazon now provisions a **Global Manager
Account** with one profileId per region for advertisers who run ads in
multiple regions under a unified seat. A single advertiser may
legitimately own three profiles — NA, EU, and FE — each with its own
`profileId` and currency. This is by design, not a data-inconsistency
artefact. `summarizeProfiles` will show counts > 0 across multiple
regions; treat this as normal and switch profile per region as you work.

## Tokens and regions

**ADS-001**. *Access tokens* are region-scoped — each one is bound to the
regional endpoint host it was minted for. *Refresh tokens* from a single
LWA consent usually work across all three regions; per-region refresh
tokens are only needed for separate LWA apps per region or legacy
narrowly-scoped tokens.

The user configures their refresh token at install time (via the
`setup-amazon` skill or the docs-site setup form). It travels on every
request as the `X-Ads-Refresh-Token` header. For users running separate
LWA apps per region, they'll need to re-run `setup-amazon` when switching
regions, or maintain separate Claude Desktop config entries per region.

The MCP picks the right endpoint and mints a fresh access token for it
based on the active profile's `countryCode`. You don't need to think about
this if you always call `setActiveProfile` before anything else. On a 401
mid-session, `invalidateCaches` clears the stale access token.

## Discovery flow

1. **`summarizeProfiles`** — the cheap starting point. Returns counts by
   country for every region with a configured refresh token. Cached 300s.
   Example output:
   ```
   na: { US: 1 }
   eu: { UK: 1, DE: 1, FR: 1 }
   fe: { JP: 0 }   // refresh token set but no profile in JP
   ```
2. **`searchProfiles`** — when you need a specific profile. Filter by
   `countryCode`, `accountType`, or `marketplaceStringId`:
   ```
   searchProfiles({ countryCode: 'UK' })
   → [{ profileId: 1234567890, countryCode: 'UK', currencyCode: 'GBP', accountInfo: {...} }]
   ```
3. **`setActiveProfile`** — stores `profileId` in session state and
   derives region from `countryCode`. Every subsequent tool call uses this
   scope + endpoint automatically.

## Switching profiles mid-session

Just call `setActiveProfile` with the new profileId. The MCP server:
1. Swaps the region if the new profile's country maps to a different region.
2. Swaps the refresh token (picks up the new region's token).
3. Mints a fresh access token on the next tool call.

No need to call `invalidateCaches` unless you hit a 401 during the switch.

## Multi-tenant HTTP mode

If the MCP server is running in HTTP mode with `identity` support:
1. `setActiveIdentity` — picks the tenant.
2. `setActiveProfile` — picks the profile within that tenant.

Identities carry their own refresh tokens. Useful when one server instance
serves multiple agencies.

## Common region traps

- **Calling `/v2/profiles` with scope header** → 400. That endpoint is the
  one path that must be called **without** `Amazon-Advertising-API-Scope`.
  The MCP server already strips it for this path — don't override.
- **Using NA `clientId` with EU refresh token** → 401. LWA app and refresh
  tokens must match. The MCP uses the single `X-Ads-Client-Id` header for all regions —
  make sure your LWA app is registered in every region you work with.
- **`countryCode: 'GB'` vs `'UK'`** — Amazon uses `UK` in profile metadata
  but `GB` in some marketplace IDs. `searchProfiles` matches on `countryCode`
  exactly — use `UK`.
- **Brazil** is in `na` region but uses BRL currency and Portuguese copy —
  don't mix US and BR campaigns in the same analysis without normalising
  currency.
