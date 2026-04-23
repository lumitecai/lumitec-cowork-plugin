---
name: amazon-ads
description: >
  Operate Amazon Advertising campaigns through the Lumitec Ads-API MCP —
  Sponsored Products / Brands / Display (PPC), DSP, AMC, Reporting v3, and
  Exports. Use this skill whenever the user mentions ACoS, TACoS, ad spend,
  keyword bids, campaign budgets, pacing, search-term reports, negative
  keyword harvesting, campaign audits, pausing or archiving campaigns,
  "PPC performance", "my ads", "advertising profile", DSP, AMC, or any
  Amazon advertising task. Also triggers on casual phrasing like "check
  ads", "why is my acos so high", "are campaigns on budget", "find waste",
  "keyword research for my listings". NOT for SP-API / seller operations
  (orders, inventory, listings) — that's the amazon-seller skill.
argument-hint: "[performance|budget|bid|negatives|setup|search-terms|audit] [query]"
---

# Amazon Advertising API

Manages Amazon advertising operations through the hosted `amazon-ads-api` MCP
at `https://lumitec-ads-api-mcp.fly.dev/mcp`. All data access goes through
MCP tools — never call the Ads API directly.

Separate from SP-API. Separate LWA app registered in Amazon's Advertising
console, separate refresh tokens, separate scopes. You cannot reuse SP-API
credentials.

## Prerequisites

Before any workflow, verify the MCP is connected:

1. Call `checkCredentials`. Possible outcomes:
   - ✅ Success → proceed.
   - ❌ `Missing or invalid X-Lumitec-Key` → the user's Lumitec access key
     is wrong or missing. Point them at the `setup-amazon` skill to
     re-provision, or email `a.walters@lumitec.ai` for a fresh key.
   - ❌ `Failed to authenticate with Amazon Ads API` → their Ads
     credentials (client ID / secret / refresh token) are wrong, expired,
     or for the wrong region. Re-run `setup-amazon`. They can also paste
     their creds into the validator at
     <https://lumitecai.github.io/lumitec-cowork-plugin/validate.html>
     to sanity-check outside the MCP path.
   - ❌ Tool not found at all → the plugin isn't installed or the MCP
     isn't connected. Suggest the `setup-amazon` skill.

2. **`summarizeProfiles`** — lists the user's advertiser profiles across
   every region their refresh token covers. Cheap (cached 300s). Shows
   counts by country. Most follow-up work starts here to pick the right
   profile.

3. **`setActiveProfile`** with a `profileId` — auto-derives the region
   from the profile's `countryCode` and sets the
   `Amazon-Advertising-API-Scope` header used on every subsequent call.

4. From here, any `sp_*` / `sb_*` / `sd_*` / `dsp_*` / `rp_*` tool runs
   against the right endpoint with the right scope.

**If the user asks for a specific country** (e.g. "UK ads performance"),
call `searchProfiles` with `countryCode: 'UK'` to find the matching
profileId, then `setActiveProfile`.

**How the MCP is wired:** the user's Claude client talks to
`https://lumitec-ads-api-mcp.fly.dev/mcp` over HTTPS. Their Ads credentials
travel as request headers on every call; the server holds nothing at rest
(only an in-memory LWA token cache keyed by a hash of their client ID +
refresh token). If the user hasn't set up the MCP yet, invoke the
`setup-amazon` skill before continuing.

---

## ⚠️ Confirmation Policy — applies to every workflow

**Never execute a write operation without explicit per-item confirmation
from the user.** Ad spend is real money; bid changes and budget changes
can burn through thousands of dollars quickly if applied incorrectly.

**Every mutating Ads-API tool requires confirmation:**

| Category | Examples |
|---|---|
| Campaign lifecycle | `sp_UpdateSponsoredProductsCampaigns` (pause/archive/budget), `sb_UpdateSponsoredBrandsCampaigns`, `sd_UpdateSponsoredDisplayCampaigns` |
| Bid changes | `sp_UpdateSponsoredProductsKeywords`, `sp_UpdateSponsoredProductsProductTargeting`, ad-group-level default bid updates |
| Negative targeting | `sp_CreateSponsoredProductsNegativeKeywords`, `sp_CreateSponsoredProductsNegativeProductTargeting` |
| New campaigns / ad groups / keywords | `sp_CreateSponsoredProductsCampaigns`, `sp_CreateSponsoredProductsAdGroups`, `sp_CreateSponsoredProductsKeywords` |
| Budget rules | Any tool under `budget-rules` or `accounts-account-budgets` that creates / updates / deletes |
| DSP | `dsp_*` create/update/delete (DSP budgets are large) |
| Profile scope | `setActiveProfile` is read-only; profile *edits* (`accounts-ads-accounts` updates) require confirmation |
| Async exports | `createReport` / `checkAndDownloadExport` are read operations — confirm *before* a batched series if the user is running many in parallel (rate-limit courtesy) |

**The confirmation rule:**
- Present your recommendation in plain English — "I'm about to lower the
  bid on keyword 'reading glasses' from £0.85 to £0.65 in campaign X" —
  and **wait** for a clear "yes, do it" / "go ahead" / "confirmed".
- Batched operations: confirm **per item** unless the user explicitly
  says "do all of these" after seeing the full list.
- For irreversible actions (campaign archive, negative-keyword creation —
  negatives can be deleted but the learning from the gap is lost), flag
  the irreversibility in the confirmation prompt.
- If the user says "just fix it" without specifics, stop and ask exactly
  which keywords / bids / campaigns and what values.

Read-only tools (`sp_ListSponsoredProductsCampaigns`, `rp_*` reports,
`summarizeProfiles`, `pageProfiles`, etc.) run directly without asking —
that's the point of the skill.

---

## Tool-Discovery Modes

The MCP server has **~580 tools** in total. To keep the context budget sane,
tools are gated by a discovery mode set at deploy time via the Fly env var
`ADS_API_TOOL_DISCOVERY`. End users don't touch this — it's server config —
but you should know which mode is active so you know what to expect:

| Mode | Visible at startup | When to use |
|---|---|---|
| `eager` | all ~580 | unlimited context budget |
| `allowlist` (default on Lumitec's deploy) | only tools whose package ∈ the deploy-time `ADS_API_PACKAGES` list | most users |
| `lazy` | ~22 meta + essentials | context-constrained sessions |

**The Lumitec deploy runs in `allowlist` mode** with these packages
enabled by default: `profiles`, `sponsored-products`, `sp-suggested-keywords`,
`sponsored-brands-v4`, `sponsored-display`, `reporting-version-3`,
`accounts-ads-accounts`, `report-presets`. Other packages (DSP, AMC, some
utilities) aren't active by default — if you need one, ask the operator
to widen the allowlist.

In `lazy` mode (rare for users of Lumitec's hosted deploy), grow the
catalogue on demand:
- `listAdPackages()` — all packages with name + tool count
- `enablePackage('sponsored-products')` — activates that package's tools
  (emits `ListToolsChanged`)
- `disablePackage('sponsored-products')` — remove when done
- `getToolSchema('sp_CreateSponsoredProductsCampaigns')` — inspect inputs
  without enabling the whole package

Every workflow below lists the **packages it assumes** — if a tool isn't
visible when you try to use it, the package isn't allowlisted.

Full reference: `references/packages-and-allowlist.md`.

---

## Workflow Routing

| User says... | Workflow | Enable first | Read first |
|---|---|---|---|
| "ACoS", "TACoS", "spend", "sales last N days", "performance" | **Performance Analysis** | `reporting-version-3`, `sponsored-products` | `references/report-types.md` + `references/reporting-workflow.md` |
| "budget", "daily budget", "budget pacing", "out of budget" | **Budget Management** | `sponsored-products`, `accounts-account-budgets` | `references/tool-reference.md` |
| "bid", "CPC", "increase bid", "lower bid", "bid floor" | **Bid Management** | `sponsored-products` | `references/campaign-pitfalls.md` |
| "negative keyword", "wasted spend", "bad search terms" | **Negative Keyword Hygiene** | `sponsored-products`, `reporting-version-3` | `references/campaign-pitfalls.md` |
| "create campaign", "new ad group", "launch" | **Campaign Setup** | `sponsored-products` | `references/tool-reference.md` + `references/campaign-pitfalls.md` |
| "search terms", "search query report", "keyword harvesting" | **Search-Term Analysis** | `reporting-version-3` | `references/report-types.md` |
| "audit", "review campaigns", "structure check" | **Campaign Audit** | `sponsored-products` | `references/tool-reference.md` |
| "DSP", "programmatic", "audience" | **DSP** (advanced) | `dsp-campaigns`, `dsp-advertisers` | `references/tool-reference.md` |
| "AMC", "clean room", "Amazon Marketing Cloud" | **AMC** (advanced) | `amc-admin`, `amc-reporting` | `references/tool-reference.md` |

If ambiguous, default to **Performance Analysis**.

---

## Workflow 1: Performance Analysis

**Enable first**: `reporting-version-3`, `sponsored-products`
**Read first**: `references/report-types.md` + `references/reporting-workflow.md`

**Cross-product analysis**: Amazon launched **Unified Reporting** at
unBoxed 2025 to consolidate SP + SB + SD + DSP into one report. **This
MCP build doesn't yet bundle the unified reporting spec** — until upstream
adds it, you must pull SP/SB/SD reports separately and merge locally.
Track upstream for the `adProduct: "MULTI_AD_PRODUCT"` variant.

Reporting v3 single-product is async. **Prefer the preset helpers over raw
`rp_createAsyncReport`** — they encode Amazon's column catalogue so you don't
have to memorise it (e.g. `acosClicks7d` is rejected for `groupBy=campaign` —
use `roasClicks14d`). All presets return `{reportId, status, startDate, endDate, columns}`
— poll + download with the same flow as the raw tool.

| Intent | Preset | reportTypeId | groupBy |
|---|---|---|---|
| Campaign-level spend / sales / ROAS | `rp_campaignPerformance({days:7})` | `spCampaigns` | `campaign` |
| Search-term analysis (negatives + harvest) | `rp_searchTermPerformance({days:30})` | `spSearchTerm` | `searchTerm` |
| Per-ASIN / per-SKU performance | `rp_advertisedProductPerformance({days:30})` | `spAdvertisedProduct` | `advertiser` |
| Target/keyword granularity (bid decisions) | `rp_targetingPerformance({days:30})` | `spTargeting` | — |

**Presets support `extraColumns`** (append to curated list, e.g. add NTB or
`attributedSalesSameSku14d`) and `overrideColumns` (replace entirely). For
SB/SD/DSP reports or bespoke groupings, drop back to `rp_createAsyncReport`.

**Date windows** end 2 days ago (attribution maturity). Max windows:
- Search-term report: **60 days** max.
- Campaign / advertisedProduct / targeting: **95 days** max.

**Lifecycle** (same for preset or raw):
1. Kick off → `{reportId, status: "PENDING"}`.
2. Poll `rp_getAsyncReport({reportId})` every ~10s. Typical completion: 30s–15min depending on queue congestion.
3. On `status === "COMPLETED"`, call `checkAndDownloadExport({statusPath: "/reporting/reports/<reportId>", exportId: <reportId>})`. The tool streams the S3 payload into the MCP response — for reports under 2 MB decompressed, the rows come back inline as text in the `payload` field (format-sniffed as `ndjson` / `json-array` / `csv`). For larger reports, the tool returns the Amazon presigned URL and leaves the download to the client. Lumitec does not persist the payload on its server.
4. Gunzip + parse NDJSON or JSON array. Compute ACoS = cost / sales, TACoS = cost / totalSales.
5. Delete with `rp_deleteAsyncReport({reportId})` when done (stops config-hash dedup collisions on re-runs).

425/429 on rate-limit — HTTP client honours `Retry-After` automatically.

---

## Workflow 2: Budget Management

**Enable first**: `sponsored-products`, `accounts-account-budgets`
**Read first**: `references/tool-reference.md` (budgets section)

Two levels:
- **Campaign daily budget** — `sp_UpdateSponsoredProductsCampaigns` with `body.campaigns[].budget.budget`.
- **Account portfolio / top-up budget** — `accounts-account-budgets` package.

**⚠️ CONFIRMATION REQUIRED**: Never change budgets without explicit per-item
confirmation. Present the proposed change as a table (campaignId, current,
proposed, delta) and wait for "yes, do it" before calling the update.

Read current state first with `sp_ListSponsoredProductsCampaigns` (POST with
filter body) — batch writes return `{success:[], error:[]}` per-row, so
always confirm the success array matches what you requested.

---

## Workflow 3: Bid Management

**Enable first**: `sponsored-products`
**Read first**: `references/campaign-pitfalls.md` (ADS-006, ADS-007, ADS-008, ADS-011)

**Decide the shape first**:
- **Recurring time-of-day / event window** (e.g. "Tuesday evenings +50%",
  "+30% during Prime Day") → use **schedule-based bid rules**
  (`sp_CreateBidRule` — ADS-011). GA since Jan 2026.
- **Conditional on performance** (e.g. "raise bid if ROAS > 4") →
  performance-based bid rule.
- **One-off adjustment based on last-7d data** → manual bid change below.

**Manual bid-change flow**:
1. Read current bids via `sp_ListSponsoredProductsKeywords` (POST with filter body).
2. Respect marketplace bid floors (~$0.02 US, £0.02 UK, €0.02 EU) — below-floor returns 422.
3. For new campaigns, default bidding strategy to `DYNAMIC_BIDS_DOWN_ONLY` (ADS-007).
4. Placement modifiers are 0–900% — if you see "adjust TOS bid by 50%" that means `percentage: 50`, not `0.5`.
5. **⚠️ CONFIRMATION REQUIRED**: present the per-keyword bid-change table before `sp_UpdateSponsoredProductsKeywords`.

Typical bid-change workflow:
- Pull search-term performance (Workflow 6).
- Score each keyword: if ACoS > target and bid > floor → lower by X%. If ACoS < target and impressions thin → raise by Y%.
- Present table: keywordId | keyword | currentBid | proposedBid | reason. Confirm.
- `sp_UpdateSponsoredProductsKeywords` with `body.keywords[{keywordId, bid}]`.

---

## Workflow 4: Negative Keyword Hygiene

**Enable first**: `sponsored-products`, `reporting-version-3`
**Read first**: `references/campaign-pitfalls.md` (ADS-005)

1. Pull **search-term report** via Workflow 6.
2. Filter: spend > $X and 0 conversions, OR ACoS > 2× target.
3. Decide scope:
   - **Ad-group negative** — blocks only that ad group (most common).
   - **Campaign negative** — blocks every ad group in the campaign.
   - **ADS-005 trap**: campaign-level negatives do NOT auto-propagate to targeting. They only block future matches; existing ad-group targets still serve.
4. Match type: `negativeExact` for the exact phrase, `negativePhrase` for the phrase + modifiers.
5. **⚠️ CONFIRMATION REQUIRED**: present the proposed negatives grouped by campaign/ad-group. Confirm per-campaign.
6. `sp_CreateSponsoredProductsNegativeKeywords` or `sp_CreateSponsoredProductsCampaignNegativeKeywords`.

---

## Workflow 5: Campaign Setup

**Enable first**: `sponsored-products`
**Read first**: `references/tool-reference.md` (sp-campaigns) + `references/campaign-pitfalls.md`

Never create without confirmation. Minimum viable campaign:

1. `sp_CreateSponsoredProductsCampaigns` — name, targetingType (`MANUAL`|`AUTO`), budget, bidding strategy.
2. `sp_CreateSponsoredProductsAdGroups` under that campaign — default bid.
3. `sp_CreateSponsoredProductsProductAds` — the SKUs/ASINs.
4. `sp_CreateSponsoredProductsKeywords` (manual) or auto-targeting stays on for AUTO.
5. Optional: `sp_CreateSponsoredProductsCampaignNegativeKeywords` brand-term list.

Each returns `{success: [...], error: [...]}` — **always check both arrays**
(ADS-009). Don't assume atomic success.

---

## Workflow 6: Search-Term Analysis

**Enable first**: `reporting-version-3`
**Read first**: `references/report-types.md`

1. `rp_searchTermPerformance({days: 30})` — returns `{reportId, status: "PENDING"}`.
   (Raw equivalent: `rp_createAsyncReport` with `reportTypeId: 'spSearchTerm'`, 60-day max.)
2. Poll `rp_getAsyncReport({reportId})` → download via `checkAndDownloadExport` → gunzip → parse.
3. Group by search-term string, sum impressions/clicks/cost/sales.
4. Flag:
   - **Harvest candidates**: high sales, not yet a keyword → promote to exact-match keyword in the same ad group.
   - **Negative candidates**: high clicks, 0 sales → negative-exact in the ad group.
5. Present two tables — harvest list and negative list — for confirmation.

---

## Workflow 7: Campaign Audit

**Enable first**: `sponsored-products`
**Read first**: `references/tool-reference.md` (sp-campaigns)

Read-only pass. Flag problems, don't change anything:

1. `sp_ListSponsoredProductsCampaigns` — all campaigns + state.
2. `sp_ListSponsoredProductsAdGroups` — all ad groups.
3. Check each campaign:
   - Paused campaigns older than 90d — archive candidates
   - Campaigns with `dynamicBidding.strategy === 'DYNAMIC_BIDS_UP_AND_DOWN'` on new/unproven ad groups — ADS-007 risk
   - Zero-spend ad groups — misconfigured targeting
   - Ad groups with no negatives and no campaign-level negatives — ADS-005 risk
   - Budget-capped campaigns (daily spend = daily budget consistently)
4. Produce audit report: campaign | issue | severity | recommendation. No writes.

---

## Output Formatting

- **Campaign list** → table: campaignId | name | state | budget | spend7d | sales7d | ACoS
- **Keyword list** → table: keywordId | keyword | matchType | bid | clicks7d | sales7d | ACoS
- **Search-term report** → table: searchTerm | impressions | clicks | cost | sales | ACoS
- **Currency** — use the active profile's marketplace symbol ($, £, €, ¥)
- **Percentages** — ACoS/TACoS to 1 decimal place (e.g. `23.4%`)
- **Dates** — ISO (`2026-04-17`), not locale-specific

---

## Error Handling (quick reference)

| Code | Meaning | Action |
|---|---|---|
| 400 `INVALID_PARAMETER_VALUE` | usually column name or date range | check `references/report-types.md` |
| 401 | wrong region OR expired token | `invalidateCaches` + retry; region auto-derives from profile |
| 403 | LWA app missing advertising scope | server config — not a credentials issue |
| 404 | unknown campaign/ad-group/keyword ID | verify scope matches the profile |
| 422 | bid below floor, budget below min, invalid enum | read error body — it's specific |
| 425/429 | rate limit | HTTP client honours `Retry-After` automatically |
| 5xx | Amazon transient | retried 3× with jitter |

Batch writes always return `{success: [], error: []}` — check BOTH arrays.

Full guide: `references/error-codes.md`.

---

## Write-Operation Confirmation Rule

**Every PUT / POST / DELETE that mutates advertising data must be confirmed
item-by-item** before calling the tool. Applies to:

- campaign create/update/archive
- ad-group create/update
- keyword / product-ad / target create/update/archive
- negative-keyword create
- bid changes, budget changes
- DSP line-item changes

Present the proposed change as a table. Wait for explicit user approval
("yes", "go ahead", "do it"). Analysing is not authorising.

---

## v2 Sunset — done (Jan 2026)

**Read**: `references/v2-sunset-migration.md`

All `/v2/sp/*`, `/v2/hsa/*`, v2 reports, and AMC v2 workflow endpoints were
sunset in January 2026 (the final wave after earlier 2025 deprecations).
They now return `410 Gone`. Build only on v3 Reporting, v3 Sponsored Products
(POST-based list), v4 Sponsored Brands, AMC v3. If a user's legacy script
still hits a v2 path, it's already broken — rewrite against v3/v4.

---

## What the Ads API Can't Do

**Read**: `references/api-gaps.md` before suggesting workarounds.

No: organic search rank, reviews, Brand Analytics, customer PII, Seller
Central UI flows. Use SP-API for seller operations, Brand Analytics via
Seller Central UI, and a browser agent for anything UI-only.
