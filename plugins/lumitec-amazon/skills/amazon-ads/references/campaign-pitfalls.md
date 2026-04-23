# Campaign Pitfalls — ADS-001 through ADS-010

Production-tested rules. Each has been the root cause of a real incident.

---

## ADS-001: Access tokens are region-scoped — refresh tokens usually aren't

**Rule**: A refresh token minted via Login-with-Amazon for the Ads API **can**
typically mint access tokens for any regional endpoint — one consent covers
NA / EU / FE. The thing that's region-bound is the **access token** (and the
regional endpoint host it's used against). Legacy / older refresh tokens
minted under a narrower scope may be region-locked; newer ones generally
aren't.

**Symptom**: `checkCredentials` passes in one region; a call in another
region returns `401 Unauthorized`. This is usually **not** a "wrong refresh
token" problem — it's either (a) a stale access token cached for the wrong
region, (b) the LWA app missing a region's scope, or (c) the profile you're
targeting genuinely not existing in that region.

**Fix**:
- The MCP receives a single refresh token per session (via the
  `X-Ads-Refresh-Token` header). Ensure it was issued for the region you're
  targeting — refresh tokens are region-locked, so an EU token won't mint
  access tokens for NA and vice versa.
- Advertisers with separate LWA apps per region need to re-run the
  `setup-amazon` skill with the correct region's refresh token, or use the
  validator at <https://lumitecai.github.io/lumitec-cowork-plugin/validate.html>
  to confirm which region the token is valid for.
- Always call `setActiveProfile` before tool calls — the MCP picks the
  correct endpoint from the profile's `countryCode`, and mints a fresh
  access token bound to that endpoint.
- On a 401 mid-session, `invalidateCaches` + retry clears a stale access
  token.

**Source**: Empirical — single-token setups work cross-region against this
MCP's EU + NA + FE endpoints. Amazon's docs describe access tokens as
region-scoped; refresh-token scoping is LWA-app-dependent.

---

## ADS-002: Scope header required on every call except `/v2/profiles`

**Rule**: `Amazon-Advertising-API-Scope: {profileId}` must accompany every
request. The **one exception** is `/v2/profiles` itself — with the scope
header, that endpoint returns 400.

**Symptom**: `sp_ListSponsoredProductsCampaigns` returns `404` on a
campaign you just created; or `/v2/profiles` returns `400 Bad Request`.

**Fix**: The MCP handles this automatically. The scope is injected from
the session state on every call **except** paths matching
`/v2/profiles`. Don't override `headers.Amazon-Advertising-API-Scope` from
caller code — it'll be scrubbed.

**Source**: Amazon Ads API docs — Scope header rules.

---

## ADS-003: v3 list endpoints are POST, not GET

**Rule**: v3 Sponsored Products list endpoints accept a **filter body** via
POST. There is no GET equivalent.

**Symptom**: Trying to page campaigns via query-string params returns 405
or 404. Confusing because v2 was GET.

**Fix**: Always pass `{ body: { campaignIdFilter, stateFilter, maxResults, nextToken } }`. The
OpenAPI-generated tools already use POST — this is more a trap when reading
old docs/examples.

**Source**: Sponsored Products v3 migration guide.

---

## ADS-004: Versioned media types are per-operation

**Rule**: Many Ads API endpoints require specific versioned
`Content-Type` / `Accept` headers like
`application/vnd.spCampaign.v3+json`. The right header differs by operation
within the same spec.

**Symptom**: `415 Unsupported Media Type` when sending
`application/json` to a v3 endpoint that wants the versioned variant.

**Fix**: The MCP resolves the correct media type from each operation's
OpenAPI `requestBody.content` keys and applies it automatically. **Don't
override `Content-Type` from caller code** — it'll be overwritten by the
client.

**Source**: Amazon Ads API docs — versioned media types.

---

## ADS-005: Campaign-level negatives don't stop ad-group targeting

**Rule**: A `campaignNegativeKeyword` blocks **future matches** against
that campaign, but does **not** override existing ad-group keyword
targets. If an ad group inside the campaign has the same term as a
positive keyword, the ad-group target still wins.

**Symptom**: You add `negativeExact: 'cheap widget'` at campaign level
expecting all ads in that campaign to stop showing for "cheap widget", but
they keep serving because an ad group has `'cheap widget'` as a broad
match keyword.

**Fix**: For guaranteed blocking, add the negative at **ad-group level**
too. Or audit positive keywords across all ad groups in the campaign
before relying on the campaign-level negative.

**Source**: Amazon Ads operational wisdom — surfaces in Seller Central Ads
Console support threads.

---

## ADS-006: Marketplace bid floors

**Rule**: Each marketplace has a minimum bid. Below-floor bids return
`422`. Approximate floors:

| Marketplace | Floor |
|---|---|
| US | $0.02 |
| UK | £0.02 |
| DE, FR, IT, ES, NL | €0.02 |
| JP | ¥2 |
| AU | AU$0.10 |
| CA | CA$0.02 |
| IN | ₹1 |

**Symptom**: `sp_CreateSponsoredProductsKeywords` with `{ bid: 0.01 }` in
US returns 422 `Bid below minimum`.

**Fix**: Validate before sending. For dynamic bid changes (e.g.
"multiply bids by 0.5"), floor the result at the marketplace minimum.

**Source**: Amazon Ads console — per-marketplace minimum bid rules. Floors
last verified April 2026 — Amazon publishes authoritative values in the
console UI; a 422 error body is always the source of truth over this
table.

---

## ADS-007: Default dynamic bidding to DOWN-ONLY for new campaigns

**Rule**: Amazon's dynamic-bidding modes:
- `DYNAMIC_BIDS_DOWN_ONLY` — safe; Amazon lowers bid on low-conversion
  placements.
- `DYNAMIC_BIDS_UP_AND_DOWN` — Amazon can raise bid up to 100% on
  high-conversion placements. Without conversion history, this burns cash.
- `FIXED_BIDS` — Amazon doesn't adjust; whatever you set is what you pay.

**Rule of thumb**: New campaigns with no conversion history default to
`DYNAMIC_BIDS_DOWN_ONLY`. Switch to `UP_AND_DOWN` only after you have a
baseline ACoS.

**Symptom**: Brand-new campaign spends 2× daily budget on day 1 with poor
ACoS — Amazon bid up aggressively on placements that didn't convert.

**Fix**: Always specify `dynamicBidding.strategy` on create. Default to
`DYNAMIC_BIDS_DOWN_ONLY`.

**Source**: Amazon Ads PPC best practices; widely documented.

---

## ADS-008: Placement modifiers are 0–900%, not 0–9.0

**Rule**: `dynamicBidding.placementBidding[].percentage` is an **integer
percent** from 0 to 900. `50` means +50%, not 5000%. `900` means +900%.

**Symptom**: You write "bid TOS +50%" as `{ percentage: 0.5 }` — Amazon
accepts it as +0.5% (basically no modifier). Your top-of-search position
never improves.

**Fix**: Always use whole numbers. `50` for +50%, `100` for +100%,
`900` max.

**Source**: Amazon Ads API docs — placementBidding schema.

---

## ADS-009: Batch writes return `{success, error}` — never assume atomic

**Rule**: Every batch create/update endpoint returns:
```json
{ "success": [...], "error": [...] }
```
HTTP 200 is returned **even if every row failed**. A non-empty `success`
array does not mean the whole batch succeeded.

**Symptom**: Your 500-keyword create reports success, but only 437 actually
got created — the other 63 were duplicates or had invalid match types.

**Fix**: Always check `error[].length` after every batch write. Surface the
per-row errors to the user. If you need atomic writes, split into
single-row calls and fail fast on any error.

**Source**: Amazon Ads API v3 batch-response convention.

---

## ADS-010: Reporting window limits

**Rule**:
- Search-term reports: **60 days** max window.
- Campaign / ad-group / keyword / product-ad / targeting reports:
  **95 days** max window.
- Historical retention: ~95 days for most reports. Older data must come
  from Exports (full snapshot) or AMC.

**Symptom**: `rp_createAsyncReport` with `startDate: '2025-01-01',
endDate: '2025-12-31'` returns 400 `INVALID_PARAMETER_VALUE` on `endDate`
or `startDate`.

**Fix**: Chunk requests into ≤95-day (or ≤60-day for search-term)
windows and concatenate locally. For year-over-year comparisons, pull
monthly reports and merge.

**Source**: Amazon Ads Reporting v3 docs.

---

---

## ADS-011: Rule-based and schedule-based bidding

**Rule**: Since 2025, Amazon has offered **rule-based bidding** for
Sponsored Products — Amazon adjusts bids automatically based on user-defined
rules (performance thresholds, events, time-of-day). **Schedule-based bid
rules** went GA January 2026 — bid up/down for specific windows (e.g.
Prime Day, weekend evenings) without manually toggling.

**When to use**:
- **Static bid change** (Workflow 3) → one-off, ad-hoc optimisation.
- **Schedule-based rule** → recurring time windows (weekly prime-time
  boost, event surges).
- **Performance-based rule** → conditional on ROAS/ACoS thresholds.

**Tools**: at Amazon, the rules API lives under `/sp/rules/...`. However,
**this build of the MCP does not yet bundle the SP bid-rules OpenAPI
spec** — only `sp_*BudgetRules*` tools are present (budget rules ≠ bid
rules). Until the upstream spec is added:

- For **budget rules** (distinct from bid rules), use the existing
  `sp_CreateBudgetRulesForSPCampaigns`, `sp_UpdateBudgetRulesForSPCampaigns`, etc.
- For **bid rules** (including schedule-based), call Amazon directly or
  wait for the spec refresh (track `scripts/sync-upstream.mjs`). Until
  then, suggest manual bid changes and warn the user that rules would be
  a better fit.

Confirm what's available in your build: `listToolsInPackage('sponsored-products')`
then grep for `Rule` and `BidRule`.

**Symptom of misuse**: a user asks for "bid +50% on Tuesday evenings" and
you write a one-off `sp_UpdateSponsoredProductsKeywords` — the change
sticks forever and you never reverse it. Use a schedule-based rule
instead.

**Gotchas**:
- Rules and manual bid overrides coexist — a manual bid change while a
  rule is active is temporary; next rule evaluation overwrites it.
- Delete old ad-hoc one-off changes before creating a recurring rule to
  avoid conflicts.
- Rules themselves count as write operations — confirm per-item before
  creating.

---

## Bonus: Write-operation confirmation

Not a technical rule — a workflow rule. **Every PUT/POST/DELETE on
campaigns / ad-groups / keywords / targets / bids / budgets must be
confirmed item-by-item before calling the tool.** Present the change as a
table, wait for explicit approval. Analysing ≠ authorising.

This is enforced at the skill layer (see SKILL.md) — the MCP server does
not gate writes.
