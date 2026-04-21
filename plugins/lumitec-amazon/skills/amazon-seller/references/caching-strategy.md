# Caching Strategy

## Principle: Caches First, API as Fallback, Refresh at Session Start

SP-API charges per call (starting April 30, 2026) and has strict rate limits.
Cache report data locally and grep it instead of hitting the API.

## Session-Start Refresh

At the start of any session involving sales, inventory, or product data:
1. Check if the relevant cache file exists and is fresh (see staleness table below)
2. If stale or missing → run the refresh workflow
3. Then grep the cache for all read queries during the session

## Key Reports to Cache

| Report | Type String | Refresh | Format |
|---|---|---|---|
| Full Listings | `GET_MERCHANT_LISTINGS_ALL_DATA` | Daily | TSV |
| All Orders (30d) | `GET_FLAT_FILE_ALL_ORDERS_DATA_BY_LAST_UPDATE_GENERAL` | Every session | TSV |
| FBA Inventory | `GET_FBA_MYI_ALL_INVENTORY_DATA` | Daily | TSV |
| Returns | `GET_FBA_FULFILLMENT_CUSTOMER_RETURNS_DATA` | Weekly | TSV |
| Reimbursements | `GET_FBA_REIMBURSEMENTS_DATA` | Weekly | TSV |
| Financial Events | `GET_V2_SETTLEMENT_REPORT_DATA_FLAT_FILE` | Weekly | TSV |

### Cache File Naming

Pattern: **`cache/{region}-{country}-{type}.tsv`**

Derive `{region}` from the session's configured region (na/eu/fe — visible in `checkCredentials` output or from install-time config) and `{country}` from the marketplace ID:

| Marketplace ID | Country | Example cache file |
|---|---|---|
| ATVPDKIKX0DER | us | `cache/na-us-listings.tsv` |
| A2EUQ1WTGCTBG2 | ca | `cache/na-ca-listings.tsv` |
| A1AM78C64UM0Y8 | mx | `cache/na-mx-listings.tsv` |
| A1F83G8C2ARO7P | uk | `cache/eu-uk-listings.tsv` |
| A1PA6795UKMFR9 | de | `cache/eu-de-listings.tsv` |
| A13V1IB3VIYZZH | fr | `cache/eu-fr-listings.tsv` |
| APJ6JRA9NG5V4 | it | `cache/eu-it-listings.tsv` |
| A1RKKUPIHCS9HS | es | `cache/eu-es-listings.tsv` |
| A1VC38T7YXB528 | jp | `cache/fe-jp-listings.tsv` |
| A39IBJ37TRP1C6 | au | `cache/fe-au-listings.tsv` |

Each marketplace needs its own report run per marketplace ID. Multi-marketplace sellers (e.g. Pan-EU) need separate cache files per country.

At session start, check which marketplace the user is working with (from the session's configured default marketplace or explicitly stated) and use the matching cache file for all grep operations.

## Refresh Workflow (~15 seconds end-to-end)

1. `createReport` with `reportType` + `marketplaceIds` (e.g. `A1F83G8C2ARO7P` for UK)
2. `getReport` — poll every few seconds until `processingStatus: DONE` (~10 seconds)
3. `getReportDocument` — returns a pre-signed S3 URL (**expires in 5 minutes!**)
4. Download the URL (curl/fetch) → save to cache file

**Critical**: Download immediately after getting the URL. Don't dawdle.

---

## The Listings Cache (detailed)

**Why it exists**: `searchListingsItems` has no keyword filter — finding "all products matching X" would mean paginating through every listing (hundreds of API calls at 20/page) and filtering client-side. The alternative, `searchCatalogItems`, returns the entire Amazon catalogue, not just your listings — your products get buried among competitor results. The cache sidesteps both problems.

**What's in it** — tab-separated columns:
- `item-name` — the listing title
- `seller-sku` — your SKU
- `asin1` — the ASIN
- `price` — current selling price
- `quantity` — stock level
- `status` — Active / Inactive / Incomplete
- `fulfillment-channel` — AMAZON (FBA) or DEFAULT (MFN)
- `open-date` — when the listing was created
- `image-url`, `item-description`, `listing-id`, `product-id-type`, `merchant-shipping-group`

**How it's used** (examples use `listings.tsv` — substitute your actual cache filename):
- **Keyword search**: `grep -i "keyword" cache/{region}-{country}-listings.tsv`
- **Find all SKUs on an ASIN**: `grep "B0XXXXXXXX" cache/{region}-{country}-listings.tsv` — crucial for the multi-SKU ASIN trap (one ASIN often has both FBA and MFN variants)
- **Pull current price/stock**: read the price/quantity columns
- **Build a pricing ladder**: for a product family (6-pack vs 12-pack vs 24-pack)
- **Confirm listing is Active**: before doing PPC work on it

**Refresh cadence**: once daily or on demand. Listings don't change minute-to-minute — one refresh at the start of a pricing or product-audit session usually suffices. The orders cache, by contrast, needs refreshing every session because it's time-windowed.

**Limitations**:
- Only flat-file data — no bullets, backend keywords, A+ content, or full attribute details. For those, fall back to `getListingsItem` with the correct SKU.
- Point-in-time snapshot — price changes or stock adjustments after refresh won't appear until next refresh.
- Per-marketplace — each marketplace ID generates a separate report. Multi-marketplace sellers need one cache file per country they sell in.

**Mental model**: a denormalised snapshot — fast for discovery and bulk lookups, but not authoritative for anything time-sensitive or attribute-rich. Use it to find *what* you need, then use the API to act on it.

---

## Grep Patterns (use INSTEAD of API calls)

| Question | Grep command | API equivalent (avoid) |
|---|---|---|
| Units of [SKU] sold this month | `grep SKU orders.tsv \| sum quantity column` | Loop `getOrderItems` — 0.5/s, 300 orders = 10+ min |
| Find all [keyword] products | `grep -i keyword listings.tsv` | `searchListingsItems` has NO keyword filter |
| All SKUs on an ASIN | `grep B0XXXXXXXX listings.tsv` | Multiple `getListingsItem` calls per SKU |
| Current price of [SKU] | `grep SKU listings.tsv \| read price column` | `getListingsItem` per SKU |
| Is listing Active? | `grep SKU listings.tsv \| check status column` | `getListingsItem` |
| Did [SKU] sell in [country]? | `grep SKU orders.tsv \| filter ship-country` | `searchOrders` per country |
| Pan-EU sales breakdown | `grep orders.tsv` (includes translated names) | Multiple API calls per marketplace |
| Inventory coverage | Orders grep (velocity) + 1 `getInventorySummaries` call | Hundreds of API calls |
| Pricing ladder (product family) | `grep "product-name" listings.tsv \| sort by price` | Multiple `getPricing` calls |

## When Cache is Stale vs Fresh

| Data type | Stale after | Why |
|---|---|---|
| Listings catalogue | 24 hours | Products change slowly |
| Orders | 6 hours | Near-real-time needs API, not cache |
| FBA Inventory | 24 hours | Stock reconciled daily |
| Pricing | Immediately | Prices change constantly — always use API |
| Financial events | 7 days | Settlement cycles are weekly |

## When to Use API Instead of Cache

- **Competitor pricing** — always live, never cache (prices change constantly)
- **Order status just placed** — API for pending + unshipped (matches Seller Central dashboard)
- **Real-time stock levels** — `getInventorySummaries` after receiving confirmation
- **Listing after update** — verify changes took effect via `getListingsItem`
- **Data newer than last cache refresh** — API is source of truth
- **Write operations** — `patchListingsItem`, `createReport`, etc. always go to API

## Orders: Max 30-Day Window

`GET_FLAT_FILE_ALL_ORDERS_DATA_BY_LAST_UPDATE_GENERAL` supports a maximum 30-day
window per report request. For longer ranges, create multiple reports with
non-overlapping date ranges.

## TSV Column Reference

Orders TSV key columns: `amazon-order-id`, `sku`, `quantity`, `item-price`, `ship-country`, `purchase-date`, `order-status`

Listings TSV key columns: `seller-sku`, `asin1`, `item-name`, `price`, `quantity`, `fulfillment-channel`, `status`

FBA Inventory TSV key columns: `sku`, `asin`, `fnsku`, `product-name`, `quantity`, `fulfillment-center-id`
