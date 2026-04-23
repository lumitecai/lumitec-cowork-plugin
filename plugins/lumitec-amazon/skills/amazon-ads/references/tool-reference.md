# Tool Reference

Tools are grouped by **package**. The package name is what you pass to
`enablePackage()` in lazy mode, or put in `ADS_API_PACKAGES` in allowlist
mode. Call `listAdPackages()` for the authoritative runtime list with counts.

## Hand-coded tools (always available)

These are registered in every discovery mode — no `enablePackage()` needed.

### Auth & identity
- **`checkCredentials`** — pings LWA with the active region's refresh token. Returns `{ ok, region, expiresInSec }`.
- **`getAccessToken`** — returns the current LWA access token (for debugging; rarely needed).
- **`listIdentities`** — in multi-tenant HTTP mode, lists configured identities.
- **`setActiveIdentity`** — switch the active identity in HTTP mode.

### Region
- **`listRegions`** — all three (`na`, `eu`, `fe`) with their endpoint hosts.
- **`setActiveRegion`** — override the active region (rarely needed — `setActiveProfile` auto-derives).

### Profile (the most-used group)
- **`summarizeProfiles`** — counts by country across every configured region. Cheap — cached 300s.
- **`searchProfiles`** — filter by `countryCode`, `accountType` (`seller` | `vendor`), `marketplaceStringId`.
- **`pageProfiles`** — paginated raw profile list.
- **`refreshProfilesCache`** — force a reload. Use after entitlement changes.
- **`setActiveProfile`** — the one you'll call most. Accepts `profileId`, auto-derives region from `countryCode`.

### Cache
- **`invalidateCaches`** — clears token cache and profile cache. Call after 401 errors or on region switches.
- **`cacheStats`** — debug — shows TTLs and entry counts.

### Discovery (meta-tools for lazy mode)
- **`listAdPackages`** — all packages with tool counts and descriptions.
- **`listToolsInPackage`** — tool names + one-line summaries inside a package.
- **`enablePackage`** — activates every tool in the package; emits `ListToolsChanged`.
- **`disablePackage`** — deactivates a package.
- **`getToolSchema`** — inspect a single tool's Zod schema without enabling its whole package.
- **`listActiveTools`** — debug — what's currently registered on the MCP server.

### Downloads (HTTP mode)
- **`checkAndDownloadExport`** — helper that downloads an export/report URL, gunzips if needed, saves to `data/profiles/{profileId}/`, and returns the local path.

### OAuth
- **`oauthStatus`** — informational — this server uses BYO refresh tokens, not 3-legged auth-code exchange.

---

## OpenAPI-generated packages

Tool names use the package's prefix (`sp_`, `sb_`, `sd_`, `rp_`, `dsp_`,
`amc_`, etc.) plus the OpenAPI `operationId`. Call `listToolsInPackage` for
the authoritative per-package list.

### `sponsored-products` (prefix `sp_`)

v3 is POST-based — list endpoints take a filter body, not query-string
params. This is counter-intuitive vs v2. Core tools:

- **Campaigns**
  - `sp_ListSponsoredProductsCampaigns` — POST with `{ body: { campaignIdFilter, stateFilter, portfolioIdFilter, nameFilter, maxResults, nextToken } }`.
  - `sp_CreateSponsoredProductsCampaigns` — `{ body: { campaigns: [{...}] } }` — batch.
  - `sp_UpdateSponsoredProductsCampaigns` — batch update (state, budget, bidding).
  - `sp_DeleteSponsoredProductsCampaigns` — archive (no hard-delete).
- **Ad groups**
  - `sp_ListSponsoredProductsAdGroups` / `sp_CreateSponsoredProductsAdGroups` / `sp_UpdateSponsoredProductsAdGroups`.
- **Keywords**
  - `sp_ListSponsoredProductsKeywords` — filter by ad group, state, match type.
  - `sp_CreateSponsoredProductsKeywords` — `{ body: { keywords: [{ adGroupId, keywordText, matchType, bid, state }] } }`.
  - `sp_UpdateSponsoredProductsKeywords` — bid / state changes.
- **Negative keywords**
  - `sp_CreateSponsoredProductsNegativeKeywords` — ad-group-level.
  - `sp_CreateSponsoredProductsCampaignNegativeKeywords` — campaign-level (see ADS-005).
- **Product ads**
  - `sp_CreateSponsoredProductsProductAds` — the SKU/ASIN assignments.
- **Targeting (ASIN / category)**
  - `sp_CreateSponsoredProductsTargetingClauses` / `sp_ListSponsoredProductsTargetingClauses`.
- **Budget rules & recommendations**
  - `sp_CreateBudgetRulesForSPCampaigns`, `sp_GetRecommendedBudgetForCampaigns`.

### `sponsored-brands` (prefix `sb_`)

v4 SB is the supported surface; avoid `/hsa/v2/*` (sunset Jan 2026).

- `sb_ListSponsoredBrandsCampaigns`, `sb_CreateSponsoredBrandsCampaigns`, `sb_UpdateSponsoredBrandsCampaigns`.
- `sb_ListAdGroups`, `sb_CreateAdGroups`, etc.
- Brand video and Store Spotlight ad formats have their own create endpoints — check `listToolsInPackage('sponsored-brands')`.

### `sponsored-display` (prefix `sd_`)

- `sd_ListCampaigns`, `sd_CreateCampaigns`, `sd_UpdateCampaigns`.
- `sd_CreateTargets` — audience / contextual targeting.
- SD supports retargeting — `audienceTargeting.lookback` controls the window.

### `reporting-version-3` (prefix `rp_`)

- `rp_createAsyncReport` — body: `{ name, startDate, endDate, configuration: { adProduct, groupBy, columns, reportTypeId, timeUnit, format } }`.
- `rp_getAsyncReport` — poll until `status === 'COMPLETED'`; response includes `url` (presigned, valid ~1h).
- `rp_deleteAsyncReport` — cleanup.

Only these three are in the current bundle (no list endpoint shipped
yet) — cache your own reportIds from `rp_createAsyncReport` responses.

See `report-types.md` for the full catalogue of `reportTypeId`s + valid columns.

### `exports-snapshots` (prefix `ex_` / `sn_`)

Bulk full-catalogue exports. Slower than reports but no column limits.

- `createExport`, `getExport`, `downloadExport` — snapshot of campaigns, ad groups, targets, etc.
- Use when you need the entire state of an advertiser, not time-series metrics.

### `accounts-account-budgets` (prefix `ab_`)

- `ab_ListAccountBudgets` / `ab_CreateAccountBudget` / `ab_UpdateAccountBudget`.
- Portfolio/top-up budgets that cap spend across multiple campaigns.

### `profiles` (prefix `pr_` — rarely used, hand-coded tools are better)

The raw Profiles API. Prefer the hand-coded `summarizeProfiles` /
`searchProfiles` — they add caching and cross-region aggregation.

### `dsp-*` (prefix `dsp_`)

Demand-Side Platform. Advanced — requires DSP entitlement on the advertiser.

- `dsp_ListOrders`, `dsp_ListLineItems`, `dsp_UpdateLineItems`.
- `dsp_ListAudiences`, `dsp_CreateAudience`.
- Programmatic display + OTT/video inventory. Talks to different endpoints (same regions, different paths).

### `amc-*` (prefix `amc_`)

Amazon Marketing Cloud — SQL-based clean-room analytics.

- `amc_CreateWorkflow` — register a SQL query definition.
- `amc_CreateWorkflowExecution` — run it.
- `amc_GetWorkflowExecution` — poll.
- Large result sets are truncated to a 50-row preview by the MCP HTTP client — pass a `S3Destination` in the workflow to get the full data set.

### `sponsored-television` (prefix `stv1_`)

Sponsored TV v1. GA in US/CA/MX/BR/UK.

- `stv1_STCreateCampaign`, `stv1_STCreateAdGroup`, `stv1_STCreateAd`, `stv1_STCreateTarget`.
- `stv1_STDeleteAd`, `stv1_STDeleteTarget`.
- CPM-priced, 14d click+view attribution.
- Reporting for STV is **not yet bundled** — see `report-types.md`.

### `ams-*` (prefix `ams_`)

Amazon Marketing Stream — real-time push subscriptions (alternative to
pulling reports).

- `ams_CreateStreamSubscription`, `ams_ListStreamSubscriptions`, `ams_GetStreamSubscription`, `ams_UpdateStreamSubscription`.
- `ams_CreateDspStreamSubscription` / DSP-specific variants.
- Pushes to SQS/Kinesis — good for near-real-time dashboards, bad for
  ad-hoc analysis.

### `creatives` (prefix `cr_`)

- Creative moderation, asset library, video management.

---

## Call-signature conventions

Every OpenAPI-generated tool accepts one input object with these keys:

- **`path`** params — for `/resource/{id}`, pass `{ id: '...' }`.
- **`query`** params — pass directly, e.g. `{ stateFilter: 'ENABLED' }` on `GET` endpoints.
- **`body`** — for POST/PUT/PATCH, pass `{ body: { ... } }`.
- **`headers`** — rarely needed; the MCP client injects `Authorization`, `Amazon-Advertising-API-ClientId`, and `Amazon-Advertising-API-Scope` automatically. Caller-supplied auth headers are scrubbed.

Don't set `Content-Type` or `Accept` — the versioned media type is resolved
from the spec's sidecar and applied for you.
