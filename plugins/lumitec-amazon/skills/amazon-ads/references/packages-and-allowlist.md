# Packages & Discovery Modes

The Ads API has **~580 OpenAPI-generated tools** in this MCP. Loading every
single one into the tool catalogue eats context for no reason — most
sessions need 20-100 tools, not 580. Discovery modes let you pick.

## The three modes

Set via `ADS_API_TOOL_DISCOVERY` at MCP deploy time. On Lumitec's hosted
deploy this is fixed to `allowlist` by the operator — end users don't change
it. Listed here so you understand what you're seeing when tools don't appear:

### `eager` — everything at startup

- **Visible at startup**: all ~580 tools.
- **Use when**: unlimited context budget (Claude 200k+ sessions), or
  exploring the full API surface.
- **Trade-off**: LLM tool-selection accuracy drops with catalogue size.
  Fine-grained queries may miss the right tool amongst similarly-named
  siblings.

### `allowlist` (default) — packages you whitelist

- **Visible at startup**: only tools whose package ∈ `ADS_API_PACKAGES`
  (comma-separated).
- **Use when**: you know upfront which API surfaces you'll touch. Most
  users, most of the time.
- **Default allowlist** in `.env.example`:
  ```
  ADS_API_PACKAGES=profiles,sponsored-products,reporting-version-3
  ```
  That's ~100 tools — SP management + reporting, enough for most PPC work.

### `lazy` — meta only, grow on demand

- **Visible at startup**: ~22 tools — all hand-coded always-on tools
  (auth, profile, discovery) plus the meta-tools.
- **Use when**: very context-constrained, or the LLM should explore
  dynamically based on user intent.
- **Grow catalogue**: call `enablePackage('sponsored-products')` →
  MCP emits `notifications/tools/list_changed` → client re-fetches list →
  SP tools are now callable.

## Available packages

Authoritative list: call `listAdPackages()`. The most-used ones:

| Package | Prefix | Surface |
|---|---|---|
| `profiles` | `pr_` | raw `/v2/profiles` (prefer hand-coded `summarizeProfiles`) |
| `sponsored-products` | `sp_` | SP campaigns, ad groups, keywords, ads, targets |
| `sponsored-brands` | `sb_` | SB v4 campaigns, brand video, store spotlight |
| `sponsored-display` | `sd_` | SD audience / contextual targeting |
| `reporting-version-3` | `rp_` | async report create/poll |
| `exports-snapshots` | `ex_` | bulk full-catalogue exports |
| `accounts-account-budgets` | `ab_` | portfolio / top-up budgets |
| `dsp-campaigns` | `dsp_` | DSP orders, line items |
| `dsp-advertisers` | `dsp_` | DSP advertiser config |
| `amc-admin` | `amc_` | AMC instance + user admin |
| `amc-reporting` | `amc_` | AMC workflow create/execute |
| `creatives` | `cr_` | moderation, asset library |

## Choosing a mode for your workflow

| Workflow | Recommended mode | Packages |
|---|---|---|
| Daily performance check | `allowlist` | `reporting-version-3`, `sponsored-products` |
| Campaign management | `allowlist` | `sponsored-products`, `accounts-account-budgets`, `reporting-version-3` |
| Full audit | `eager` or `allowlist` with many | everything you'll touch |
| Exploratory — user intent unknown | `lazy` | start empty, `enablePackage()` as needed |
| DSP work | `allowlist` | `dsp-campaigns`, `dsp-advertisers`, `reporting-version-3` |
| AMC analytics | `allowlist` | `amc-admin`, `amc-reporting` |

## Lazy-mode command patterns

At session start:
```
listAdPackages()
→ [{ name: 'sponsored-products', toolCount: 82, description: '...' }, ...]
```

When user says "show me last week's ACoS":
```
enablePackage('reporting-version-3')
enablePackage('sponsored-products')
// ListToolsChanged fires; rp_* and sp_* now visible
```

When you're done with a package:
```
disablePackage('sponsored-products')
// ListToolsChanged fires; sp_* disappear
```

Inspect a single tool without pulling in a whole package:
```
getToolSchema('sp_CreateSponsoredProductsCampaigns')
→ { name, description, inputSchema (as JSON) }
```

## What stays on in every mode

These are **alwaysOn** — registered regardless of discovery mode:

- `checkCredentials`, `getAccessToken`
- `listIdentities`, `setActiveIdentity`
- `listRegions`, `setActiveRegion`
- `summarizeProfiles`, `searchProfiles`, `pageProfiles`, `refreshProfilesCache`, `setActiveProfile`
- `invalidateCaches`, `cacheStats`
- `checkAndDownloadExport`, `oauthStatus`
- `listAdPackages`, `listToolsInPackage`, `enablePackage`, `disablePackage`, `getToolSchema`, `listActiveTools`
- **Report presets**: `rp_campaignPerformance`, `rp_searchTermPerformance`, `rp_advertisedProductPerformance`, `rp_targetingPerformance`

~26 tools. The MCP server always has these available so bootstrap + meta
operations + common reporting work without `enablePackage` gymnastics.

## Report presets vs raw `rp_createAsyncReport`

Four curated wrappers over `rp_createAsyncReport` in the `report-presets`
package (alwaysOn — available in every mode, including `lazy`). They encode
Amazon's v3 column catalogue and the right `reportTypeId`/`groupBy` so you
don't need to memorise which columns each report type accepts.

| Preset | reportTypeId | Use for |
|---|---|---|
| `rp_campaignPerformance` | `spCampaigns` | Campaign spend/sales/ROAS roll-up |
| `rp_searchTermPerformance` | `spSearchTerm` | Negative-keyword + harvest workflows |
| `rp_advertisedProductPerformance` | `spAdvertisedProduct` | Per-ASIN / per-SKU metrics |
| `rp_targetingPerformance` | `spTargeting` | Keyword/target-level bid analysis |

Each accepts `days` (with sensible max per report type — 60d for search-term,
95d for the others), `extraColumns` (append to preset list) and
`overrideColumns` (replace entirely). Returns `{reportId, status, startDate,
endDate, columns}` — poll + download with `rp_getAsyncReport` +
`checkAndDownloadExport` exactly like the raw tool.

**Drop back to `rp_createAsyncReport`** for SB/SD/DSP reports, AMC workflows,
`MULTI_AD_PRODUCT` unified reporting, or any bespoke `groupBy`/column
combination the presets don't cover. The raw tool lives in the
`reporting-version-3` package and is only visible when that package is
allow-listed or explicitly enabled.

## Why we don't bundle a second "official Amazon" MCP alongside this one

The official Amazon Advertising MCP servers (`amzn-ads-eu`/`amzn-ads-na`)
package tools like `create_campaign_report` that are just opinionated
wrappers over the same Reports v3 endpoints this MCP already exposes. Their
"richer fields" advantage (NTB, DPV, promoted product metrics) is purely a
column-list choice — every column is available via `rp_createAsyncReport` or
an `extraColumns` addition on the presets.

Running two ad MCPs in the same session means two credential setups, two
profile-context states to keep in sync, and tool-name collisions. One is
simpler. This MCP is a strict superset:

- Everything the official MCP reports on, via the presets + `rp_*` raw tool.
- Browsing operations the official MCP lacks (cross-account campaign lists,
  ad-group + keyword + target lists, state/config via Exports API).
- SB / SD / DSP / AMC / AMS / Attribution / Accounts / Creatives — all absent
  from the official MCP.

## Env-var reference

```bash
# Discovery mode
ADS_API_TOOL_DISCOVERY=allowlist           # eager | allowlist | lazy

# Allowlist — comma-separated package names
ADS_API_PACKAGES=profiles,sponsored-products,reporting-version-3,exports-snapshots
```

Unknown package names in `ADS_API_PACKAGES` are silently ignored — call
`listAdPackages()` to check canonical names.
