# Reporting v3 — Report Types

All Reporting v3 requests use `rp_createAsyncReport` with a
`configuration` object. This file lists the valid `reportTypeId` values,
their supported `groupBy` keys, and the most-useful columns.

**Complete column lists are in Amazon's docs** — this file covers the
frequently-used subset. If a column name returns `INVALID_PARAMETER_VALUE`,
cross-check Amazon's current schema.

**Most callers should use the preset tools instead**, which encode the
Amazon-accepted column lists for you:

- `rp_campaignPerformance` → `spCampaigns`
- `rp_searchTermPerformance` → `spSearchTerm`
- `rp_advertisedProductPerformance` → `spAdvertisedProduct`
- `rp_targetingPerformance` → `spTargeting`

Drop back to `rp_createAsyncReport` only for SB/SD/DSP/AMC, bespoke
groupings, or `MULTI_AD_PRODUCT` unified reports.

**Column-name gotchas** (validated against Amazon's accept/reject
responses as of April 2026):
- `acosClicks7d` and `roasClicks7d` are **invalid** on `groupBy=campaign`.
  Use `roasClicks14d` and compute ACoS as `cost / sales7d` yourself.
- There is **no standalone `spAdGroups`** reportTypeId. Use
  `spAdvertisedProduct` with `groupBy=advertiser` and read
  `adGroupId`/`adGroupName` off the rows.
- `spTargeting` rejects `targetId` and `targetingExpression`. Use
  `keyword`, `matchType`, and `targeting` instead.

## Shared config shape

```json
{
  "name": "descriptive-name",
  "startDate": "YYYY-MM-DD",
  "endDate": "YYYY-MM-DD",
  "configuration": {
    "adProduct": "SPONSORED_PRODUCTS | SPONSORED_BRANDS | SPONSORED_DISPLAY | SPONSORED_TELEVISION",
    "groupBy": ["..."],
    "columns": ["..."],
    "reportTypeId": "...",
    "timeUnit": "SUMMARY | DAILY",
    "format": "GZIP_JSON | JSON"
  }
}
```

Rules:
- `endDate` is inclusive.
- `timeUnit: 'DAILY'` adds a `date` column to every row.
- `format: 'GZIP_JSON'` is newline-delimited JSON when decompressed; much smaller.
- `groupBy` determines the row grain — pick the **lowest** grain you need.

## Window limits

| reportTypeId | Max window |
|---|---|
| `spSearchTerm`, `sbSearchTerm`, `sdSearchTerm` | 60 days |
| Campaign / ad-group / product-ad / keyword / targeting | 95 days |
| Purchased-product, advertised-product | 95 days |

Exceed the window → `400 INVALID_PARAMETER_VALUE` on `endDate`.

## Sponsored Products (`adProduct: "SPONSORED_PRODUCTS"`)

### `spCampaigns` — preset: `rp_campaignPerformance`

Campaign-level performance.

- `groupBy`: `["campaign"]` or `["campaign", "campaignPlacement"]`
- Columns (common, accepted for `groupBy=campaign`): `campaignId`,
  `campaignName`, `campaignStatus`, `campaignBiddingStrategy`,
  `campaignBudgetAmount`, `impressions`, `clicks`, `cost`,
  `clickThroughRate`, `costPerClick`, `purchases7d`, `sales7d`,
  `unitsSoldClicks1d`, `roasClicks14d`.
- **Do not** request `acosClicks7d`, `roasClicks7d`, or
  `unitsSoldClicks7d` for this groupBy — Amazon rejects them. Compute
  ACoS/ROAS from `cost` + `sales7d` instead.

### Ad-group rollup — use `spAdvertisedProduct`

There is no `spAdGroups` reportTypeId. For ad-group-level metrics, use
`spAdvertisedProduct` with `groupBy=["advertiser"]` and aggregate on the
`adGroupId` / `adGroupName` columns present in each row.

### `spTargeting` — preset: `rp_targetingPerformance`

Keyword + auto-target + product-target performance.

- `groupBy`: `["targeting"]`
- Accepted columns: `keywordId`, `keyword`, `matchType`, `targeting`,
  `keywordBid`, `campaignId`, `campaignName`, `adGroupId`, `adGroupName`,
  `impressions`, `clicks`, `cost`, `purchases7d`, `sales7d`,
  `unitsSoldClicks1d`, `roasClicks14d`.
- **Do not** request `targetId` or `targetingExpression` — Amazon
  rejects them. The `targeting` column carries the human-readable
  expression; `keywordId` serves as the unique row ID.

### `spSearchTerm` — preset: `rp_searchTermPerformance`

The report for harvesting + negatives.

- `groupBy`: `["searchTerm"]`
- **60-day max window**
- Accepted columns: `searchTerm`, `keywordId`, `keyword`, `matchType`,
  `campaignId`, `campaignName`, `adGroupId`, `adGroupName`,
  `impressions`, `clicks`, `cost`, `purchases7d`, `sales7d`,
  `unitsSoldClicks1d`, `roasClicks14d`.
- Use `keyword` (not `keywordText`).

### `spAdvertisedProduct` — preset: `rp_advertisedProductPerformance`

Performance per advertised SKU/ASIN. Also the correct report type for
ad-group rollups.

- `groupBy`: `["advertiser"]`
- Accepted columns: `advertisedAsin`, `advertisedSku`, `campaignId`,
  `campaignName`, `adGroupId`, `adGroupName`, `impressions`, `clicks`,
  `cost`, `purchases7d`, `sales7d`, `unitsSoldClicks1d`, `roasClicks14d`.

### `spPurchasedProduct`

Bought-via-ad ASINs, including halo (bought a different ASIN after clicking the ad).

- `groupBy`: `["asin"]`
- Columns: `purchasedAsin`, `advertisedAsin`, `impressions`, `clicks`, `unitsSoldOtherSku7d`, `salesOtherSku7d`.

## Sponsored Brands (`adProduct: "SPONSORED_BRANDS"`)

### `sbCampaigns`, `sbAdGroups`, `sbSearchTerm`, `sbPurchasedProduct`

Analogous to SP. SB has richer creative breakdown:

- `sbCampaigns` supports `groupBy: ["campaign", "campaignPlacement"]` — placement key is top-of-search vs. other.
- `sbAds` — per-creative performance (helpful for Brand Video vs Store Spotlight comparison).

## Sponsored Television

Sponsored TV is GA in US/CA/MX/BR/UK as of April 2026. CPM-priced,
streaming + linear video inventory (Prime Video, Freevee, Fire TV,
Twitch).

**Management tools**: available in this MCP under the `stv1_` prefix —
`stv1_STCreateCampaign`, `stv1_STCreateAdGroup`, `stv1_STCreateAd`,
`stv1_STCreateTarget`, etc. Call `listToolsInPackage` on the STV package
to see all.

**Reporting**: Sponsored TV reporting is **not yet bundled** in this
build's OpenAPI specs (upstream KuudoAI hasn't shipped the STV reporting
spec at the time of this port). Amazon's docs document `adProduct:
"SPONSORED_TELEVISION"` on Reporting v3, but until the spec is added,
`rp_createAsyncReport` won't validate STV report configs reliably. Track
`scripts/sync-upstream.mjs` output — add STV reporting when it appears.

Attribution window is **14d** (click + view), same as SD. No
search-term surface.

## Sponsored Display (`adProduct: "SPONSORED_DISPLAY"`)

### `sdCampaigns`, `sdAdGroups`, `sdTargeting`, `sdAdvertisedProduct`

SD doesn't have a search-term equivalent (no keywords — audience/contextual
targeting only). Useful columns:

- `viewableImpressions`, `viewClickThroughRate` — SD is CPM-ish.
- `impressions`, `clicks`, `cost`, `detailPageViews14d`, `purchases14d`, `sales14d`.
- Attribution windows on SD are **14d** by default, not 7d.

## Attribution-window column suffixes

Columns that end in `7d`, `14d`, `30d` reflect the attribution window.
Sponsored Products default is 7d click; SD default is 14d click+view.
For cross-product reporting, pick a consistent suffix or you'll
accidentally compare apples and oranges.

## Time-unit gotchas

- `SUMMARY` collapses the whole window into one row per groupBy key.
- `DAILY` adds `date` and produces one row per day per key — size explodes
  with long windows or fine groupBy. Plan download/parse accordingly.

## Cost fields

- `cost` — what you paid (= sum of clicks × CPC).
- `sales{N}d` — revenue attributed.
- Compute ACoS manually: `cost / sales7d`. For ROAS, `roasClicks14d` is
  the pre-computed column Amazon accepts on `spCampaigns`,
  `spAdvertisedProduct`, `spSearchTerm`, and `spTargeting`. The
  `acosClicks7d` / `roasClicks7d` columns exist only for narrower
  groupBy grains (not `campaign`) — if you need them, swap to a
  day/hour groupBy or compute from `cost` + `sales7d`.

## Minimum sanity payload

When in doubt, start with this and expand:

```json
{
  "name": "sp-campaigns-last-7",
  "startDate": "2026-04-10",
  "endDate": "2026-04-17",
  "configuration": {
    "adProduct": "SPONSORED_PRODUCTS",
    "groupBy": ["campaign"],
    "columns": ["campaignId", "campaignName", "impressions", "clicks", "cost", "sales7d", "roasClicks14d"],
    "reportTypeId": "spCampaigns",
    "timeUnit": "SUMMARY",
    "format": "GZIP_JSON"
  }
}
```
