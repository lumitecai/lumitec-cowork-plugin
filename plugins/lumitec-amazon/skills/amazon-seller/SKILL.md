---
name: amazon-seller
description: >
  Operate a user's Amazon Seller Central business through the Lumitec
  SP-API MCP — orders, FBA inventory, listings, pricing, reports, financial
  events, feeds, and notifications. Use this skill whenever the user asks
  anything about their Amazon seller account: checking orders or sales,
  sales velocity, looking up ASINs or SKUs, adjusting prices or stock,
  running reports, investigating suspended listings, analysing margins,
  pan-EU / multi-marketplace work, inventory coverage, settlement
  reconciliation, "my products", "my listings", "how many sold", "days of
  stock", "buy box", or anything in Seller Central. Also applies to
  casual phrasing like "check Amazon", "my amzn sales", "fba numbers" —
  if it's about their seller business, this skill applies even when they
  don't name SP-API explicitly. NOT for Amazon advertising / PPC (that's
  the amazon-ads skill) and NOT for Amazon-as-a-shopper (Claude handles
  that directly).
argument-hint: "[orders|inventory|catalog|pricing|listings|reports|fba|finance] [query]"
---

# Amazon Seller Partner API

Manages Amazon seller operations through the hosted `amazon-sp-api` MCP at
`https://lumitec-sp-api-mcp.fly.dev/mcp`. All data access goes through MCP
tools — never call SP-API directly.

## Prerequisites

Before any workflow, verify the MCP is connected:

1. Call `checkCredentials`. Possible outcomes:
   - ✅ Success → proceed.
   - ❌ `Missing or invalid X-Lumitec-Key` → the user's Lumitec access key
     is wrong or missing. Ask them to re-run the `setup-amazon` skill, or
     email `a.walters@lumitec.ai` if they need a fresh key.
   - ❌ `Failed to authenticate with Amazon SP-API` → their Amazon
     credentials (client ID / secret / refresh token) are wrong, expired,
     or for the wrong region. Re-run `setup-amazon`.
   - ❌ Tool not found at all → the plugin isn't installed or the MCP
     hasn't connected. Suggest the `setup-amazon` skill.
2. If targeting a non-primary marketplace, always pass `marketplaceId`
   explicitly. The MCP has a default marketplace set at install time, but
   that's only used when no marketplaceId is passed. EU sellers often
   default to Germany — UK queries silently return nothing without an
   explicit UK marketplace ID (`A1F83G8C2ARO7P`).
3. `sellerId` is configured at install time and auto-filled by the MCP
   for listings tools. Only pass it explicitly if operating across
   multiple seller accounts (rare — one seller ID per region: NA/EU/FE).

**How the MCP is wired:** the user's Claude client talks to
`https://lumitec-sp-api-mcp.fly.dev/mcp` over HTTPS. Their Amazon
credentials travel as request headers on every call; the server holds
nothing at rest (only an in-memory LWA token cache keyed by a hash of
client ID + refresh token, which isolates tenants from each other). If
the user hasn't set up the MCP yet, invoke the `setup-amazon` skill
before continuing.

---

## Cache-First Principle

**Mental model: caches first, API as fallback, refresh at session start.**

SP-API has strict rate limits and starts charging per call from April 30,
2026. Pull a bulk report once, store it on the user's own disk, and query
it locally rather than looping the API per SKU / order.

**Why client-side caching (not Lumitec-side):** the cache contains order
data with buyer PII. Amazon's SP-API Data Protection Policy places strict
obligations on anyone storing that data at rest. Keeping the cache on the
user's own machine — where they're the registered Amazon developer holding
their own data — keeps Lumitec out of the DPP compliance scope. Lumitec
never sees the cached rows.

### Caches to maintain

| Work type | Cache file | Report type |
|---|---|---|
| Sales / orders analysis | `orders-{region}-{country}.tsv` | `GET_FLAT_FILE_ALL_ORDERS_DATA_BY_LAST_UPDATE_GENERAL` |
| Product / pricing / listing work | `listings-{region}-{country}.tsv` | `GET_MERCHANT_LISTINGS_ALL_DATA` |
| Inventory / stock planning | `fba-{region}-{country}.tsv` | `GET_FBA_MYI_ALL_INVENTORY_DATA` |

**Staleness**: orders 6 h, listings 24 h, inventory 24 h, financial events 7 d.
Full table in `references/caching-strategy.md`.

### Where the cache lives, by client

| Client | Cache directory | Persistence | Setup |
|---|---|---|---|
| **Claude Code** (mac/linux/win) | `cache/` in the cwd (project-relative) | Persists as long as the project folder exists | No setup — `Bash`/`Write` tools run directly on the host FS |
| **Cowork** | `/sessions/<name>/mnt/lumitec-amazon-cache/` inside the VM (mapped to a host directory like `~/lumitec-amazon-cache/`) | Persists on the host, survives session deletion | **One-time** per conversation: ask the user to approve a `request_cowork_directory` for a folder like `~/lumitec-amazon-cache/`. Then writes go through to their Mac disk. |
| **Claude Desktop** | Not supported for bulk caching | — | Use targeted API batches instead (`getInventorySummaries` with `sellerSkus`, `getPricing` batch of 20). Recommend the user switch to Claude Code or Cowork for bulk work. |

**Cowork gotcha** — [bug #30364](https://github.com/anthropics/claude-code/issues/30364):
multiple separate writes to a single mount can silently drop all but the
first. Workaround: **write each cache to a separately named file in one
write operation** (don't do three sequential `curl -o` into the same
mount — bundle them, or use three distinct filenames with distinct
downloads spaced apart).

### Freshness check (all clients with filesystem access)

1. Look for the cache file at the path appropriate to the client.
2. Compare its modification time to the staleness table above.
3. Fresh → `grep` it directly. Stale or missing → refresh.

### Refresh workflow (~15 s end-to-end)

1. `createReport` with the report type + `marketplaceIds`.
2. Poll `getReport` until `processingStatus: DONE`.
3. `getReportDocument` → returns a pre-signed S3 URL (**expires in 5 min —
   download immediately**).
4. Download via `Bash curl -o <cache-path> "$URL"` (Code) or
   `mcp__workspace__bash curl -o <mount-path>/... "$URL"` (Cowork).

### Grep instead of API

| Question | Action | Never do |
|---|---|---|
| "How many units of [SKU] sold?" | Grep orders cache, sum quantity column | Loop `getOrderItems` (~0.5 req/s → 10 min for 300 orders) |
| "Find all [keyword] products" | Grep listings cache | `searchListingsItems` has no keyword filter |
| "Current price of [SKU]?" | Grep listings cache, read price column | `getListingsItem` per SKU |
| "Did [SKU] sell in [country]?" | Grep orders cache, filter ship-country | `searchOrders` per country |
| "Pan-EU sales breakdown" | Grep orders cache (includes cross-border translated names) | Multiple API calls per marketplace |

### When to use live API regardless of cache

- **Competitor pricing** — always live, never cache (prices change constantly).
- **Real-time stock levels** — `getInventorySummaries` after receiving confirmation.
- **Write operations** — `patchListingsItem`, `putListingsItem`, etc. always hit API.
- **Order status just placed** — API for Pending + Unshipped (matches Seller Central).
- **Verify a listing update took effect** — `getListingsItem` after patching.
- **Data newer than last cache refresh** — API is source of truth.
- **Claude Desktop users** (no local filesystem tools) — use batched API
  calls throughout. `getInventorySummaries({sellerSkus:[...]})` up to 50
  SKUs; `getPricing` / `getCompetitivePricing` / `getItemOffersBatch` up
  to 20 ASINs. Never loop `getOrderItems` — use a report and recommend
  the user switch to Claude Code or Cowork for bulk work.

---

## Workflow Routing

| User says... | Workflow | Read first |
|---|---|---|
| "check my orders", "recent orders", "order status" | **Order Management** | — |
| "how many sold", "sales this month", "sales velocity" | **Order Management** (grep cache) | — |
| "FBA inventory", "stock levels", "inbound shipments" | **FBA & Inventory** | — |
| "days of stock", "inventory coverage" | **Inventory Coverage** (compound) | — |
| "look up ASIN", "search products", "catalog" | **Product Research** | — |
| "find my products", "search my listings" | **Grep listings cache** (no API keyword search) | — |
| "pricing", "competitor prices", "buy box" | **Pricing Intelligence** | `references/rate-limits.md` |
| "create listing", "update listing", "patch listing" | **Listing Management** | `references/listings-pitfalls.md` |
| "sales report", "generate report", "listings report" | **Reports & Analytics** | `references/report-types.md` |
| "financial events", "settlements" | **Finance** | — |
| "notifications", "subscriptions", "destinations" | **Notifications** | — |

If ambiguous, default to **Order Management** for sales queries or **Product Research** for ASIN/product queries.

---

## Workflow 1: Order Management

**Read first** (for bulk work): `references/caching-strategy.md`,
`references/rate-limits.md`.

**Choose the right approach by volume:**
- **Today's orders / last few days** → use API (includes Pending + Unshipped, matches Seller Central)
- **Bulk / 7+ days / reporting** → use report `GET_FLAT_FILE_ALL_ORDERS_DATA_BY_LAST_UPDATE_GENERAL` (max 30-day window per request, see `references/caching-strategy.md`)

**API path (small volume):**
1. `searchOrders` with date/status filters. Always pass `marketplaceIds`. Rate: ~1 req/min — don't loop aggressively.
2. For details: `getOrder` with orderId
3. For line items: `getOrderItems` with orderId

**Report path (bulk):**
1. `createReport` with type `GET_FLAT_FILE_ALL_ORDERS_DATA_BY_LAST_UPDATE_GENERAL`, date range, marketplaceIds
2. Poll `getReport` until DONE
3. `getReportDocument` → download TSV (**URL expires in 5 min!**)
4. Parse/grep the TSV locally — zero additional API calls

---

## Workflow 2: FBA & Inventory

**Choose the right inventory tool:**
- **`getInventorySummaries`** — use for targeted SKU lookups. Supports `sellerSkus` filter param to fetch specific SKUs in one call. Returns both FBA and MFN inventory.
- **`getFbaInventorySummaries`** — use for broad FBA overview. Does NOT support SKU filtering — returns all FBA inventory paginated at 50 results. You may need multiple pages to find specific SKUs.
- **Rule**: If you know the SKUs, always use `getInventorySummaries` with `sellerSkus`. Only use `getFbaInventorySummaries` for full FBA catalogue scans.

1. Get current stock levels (live API — stock changes in real time)
2. `getInboundEligibility` to check FBA program eligibility
3. `listInboundPlans` for inbound shipment tracking (v2024-03-20)
4. `getInboundPlan` for details on a specific inbound plan

### Inventory Coverage (compound workflow)

Combines cached orders + live inventory in one pass:
1. Grep the **orders cache** for the target SKU(s) over 30 days → sum quantity → daily sales velocity
2. One `getInventorySummaries` API call with `sellerSkus` filter → current FBA stock for those exact SKUs
3. Divide: `currentStock / dailyVelocity` = **days of stock remaining**
4. Flag anything under 14 days as reorder-needed
5. **Velocity-weighted thresholds**: top-3 sellers by velocity should use a 45-day threshold (not 14). Losing your best seller is disproportionately costly — by the time you're at 14 days, FBA inbound lead time may exceed remaining stock.

This uses 1 API call total instead of hundreds.

---

## Workflow 3: Product Research

1. `searchCatalogItems` with keywords
2. `getCatalogItem` with ASIN for full details
3. Chain with `getCompetitivePricing` for price context

---

## Workflow 4: Pricing Intelligence

**Read first**: `references/rate-limits.md` (batch rules)

1. `getPricing` — batch up to 20 ASINs/SKUs per call (always batch, never loop)
2. `getCompetitivePricing` — batch up to 20 for competitor data
3. `getListingOffers` — lowest offers for a single SKU
4. Present as comparison table: ASIN | yourPrice | buyBox | lowestOffer | competitors

---

## Workflow 5: Listing Management

**Read first**: `references/listings-pitfalls.md` (critical traps)

**⚠️ CONFIRMATION REQUIRED**: Never execute write operations (`patchListingsItem`, `putListingsItem`, `deleteListingsItem`) without explicit per-item confirmation from the user. Present recommendations and wait for a clear "go ahead" or "yes, do it." Analysing data is not the same as authorising changes. This applies to all price changes, quantity updates, and listing modifications.

1. **Find the SKU first** — if user gives a product name or ASIN (not a SKU), grep the listings cache to find it:
   - `grep -i "product name" cache/{region}-{country}-listings.tsv`
   - `grep "B0XXXXXXXX" cache/{region}-{country}-listings.tsv` — also reveals if ASIN has multiple SKUs (FBA + MFN)
2. `getListingsItem` to inspect full current state (sellerId auto-fills from install-time config).
3. For partial updates (price, qty): use `patchListingsItem` with JSON Patch operations.
   `putListingsItem` overwrites ALL data — only use for full create/replace.
4. `deleteListingsItem` to remove.
5. Always verify correct SKU (FBA vs MFN — see LIST-001 in pitfalls).

### EU Suspended Listings Investigation

When a user reports suspended or inactive listings in EU marketplaces:
1. `getMarketplaceParticipations` — check which marketplaces show `hasSuspendedListings: true`
2. For each affected marketplace, `getListingsItem` with that marketplace's ID to see the listing status and issues
3. Common causes: missing compliance docs (GPSR, CE marking), VAT registration gaps, product category restrictions
4. Cross-reference with listings cache to identify which SKUs are affected vs active

---

## Workflow 6: Reports & Analytics

**Read first**: `references/report-types.md` + `references/caching-strategy.md`

Reports are **async**:
1. `createReport` with reportType + marketplaceIds
2. Poll `getReport` until status = DONE
3. `getReportDocument` to get download URL (**expires in 5 minutes!**)
4. Download and parse the TSV/CSV data

---

## Workflow 7: Finance

1. `listFinancialEventGroups` for settlement periods (requires `financialEventGroupStartedAfter` date)
2. `listFinancialEvents` for transaction details
3. `listFinancialEventsByOrderId` for per-order fees/charges/refunds (profitability analysis)
4. `getFinancialEventGroup` for events in a specific group

### Financial Events Aggregation Pattern

Raw financial events are verbose — always aggregate before presenting:
1. Call `listFinancialEventsByOrderId` for each order (or `listFinancialEvents` for a date range)
   - **Sample from 2+ weeks ago** — recent orders may not have financial events yet (pending settlement). Orders from the last few days will return empty `ShipmentEventList`.
2. Group charges by fee type from `ShipmentEventList[].ShipmentItemList[].ItemFeeList[]`:
   - **Principal** (`Principal`) — the sale price you receive
   - **Commission** (`Commission`) — Amazon's referral fee
   - **FBA Fulfillment** (`FBAPerUnitFulfillmentFee`, `FBAPerOrderFulfillmentFee`) — pick/pack/ship
   - **Digital Services** (`DigitalServicesFee`) — UK/EU digital services tax
   - **Other** — variable closing fees, giftwrap, promotions
3. Present as summary table: FeeType | Total | Currency
4. Calculate **net per order**: Principal − (Commission + FBA fees + other fees)
5. For profitability analysis across multiple orders, sum nets and divide by order count for average margin

Note: Finance API v0 — sunset August 28, 2026.

---

## Workflow 8: Notifications

1. `getDestinations` to list existing SQS/EventBridge destinations
2. `createDestination` to set up a new one
3. `createSubscription` for a notification type (ORDER_CHANGE, REPORT_PROCESSING_FINISHED, etc.)
4. `getSubscription` to check status

---

## Output Formatting

- **Orders** → table: orderId | status | date | total | items
- **Inventory** → table: SKU | ASIN | quantity | condition | fulfillmentChannel
- **Pricing** → comparison: ASIN | yourPrice | buyBox | lowestOffer | competitorCount
- **Financial events** → grouped by eventGroup with totals
- **Currency** → marketplace-appropriate symbol ($, £, €, ¥)

---

## Error Handling (quick reference)

| Code | Meaning | Action |
|---|---|---|
| 400 | Bad request — surface `errors[].message` | Fix params |
| 401 | Token expired | Retry — `getAccessToken` auto-refreshes |
| 403 | Missing role/scope on LWA app | Check app registration (not credentials) |
| 404 | Invalid ASIN/SKU/orderId | Verify identifier, try search |
| 429 | Rate limit — read `x-amzn-RateLimit-Limit` | Backoff with jitter |
| 500/503 | Amazon transient error | Retry with backoff |

Full guide: `references/error-codes.md`

---

## What SP-API Can't Do

**Read**: `references/api-gaps.md` before suggesting workarounds.

No: Buy Box analytics, A+ Content, PPC/Advertising, reviews, Brand Analytics, keyword ranking.
Use: agent-browser for Seller Central UI tasks, Amazon Ads API for PPC.
