# What the Ads API Can't Do

Before suggesting a workaround, check here. Some things genuinely aren't
exposed — don't pretend otherwise.

## Not available at all

### Organic search rank / BSR
The Ads API is **advertising only**. Organic keyword rank, category BSR,
and organic search share are not exposed. Use SP-API `getCatalogItem` for
current BSR (point-in-time), or a browser agent for historical rank
tracking.

### Reviews and ratings
Not available. SP-API exposes listing-level avg rating via
`getCatalogItem` (via the `summaries` attribute set on some marketplaces),
but review content is not in any API. Use Seller Central UI.

### Brand Analytics
Amazon's Brand Analytics (search frequency rank, top search terms, etc.)
is Seller Central UI only. No API. Use the browser agent if you need it
programmatically.

### Buy Box winner history
Current Buy Box owner: SP-API `getCompetitivePricing`. Historical winner
rotation: not exposed. Infer from order flow in SP-API orders if you need
it.

### Organic traffic breakdown
Glance views, session data, conversion rate by traffic source — Seller
Central "Business Reports" only. Not in Ads API or SP-API.

### Customer PII
Buyer names, emails, shipping addresses are not in the Ads API. They are
in SP-API Orders (restricted data, requires specific LWA roles).

### Editing a Sponsored Brands creative after launch
Creative assets (headline, logo, images) can be updated via v4, but the
**video asset** of a Brand Video campaign cannot be replaced in place —
you must create a new campaign. This is a platform limitation, not an
API limitation.

## Available but awkward

### "All campaigns across all profiles" in one call
No. Profiles are the scope boundary. You must iterate profiles, calling
`setActiveProfile` for each, then list campaigns. Plan for this in bulk
workflows.

### Cross-profile attribution
Amazon attributes sales to the profile whose ad was clicked. If a customer
sees an EU ad and buys in the US (rare), attribution does not cross. The
Ads API reports always map to single-profile activity.

### Sub-ASIN performance
Search-term reports and targeting reports give **ASIN-level** performance
but not **variant-level** (size / colour child). If you advertise a parent
ASIN and the customer buys a variant, it's reported against the parent.

### Halo sales across products
`spPurchasedProduct` report captures cross-ASIN purchases driven by an ad,
but only from the same advertiser. Halo revenue that spilled to a
different brand/seller after click is not attributable.

### AMC SQL surface
AMC (Amazon Marketing Cloud) is powerful but has its own permission model
and is not free — entitlement-gated. Not every Ads API account has AMC
access.

## Common "can it..." rephrasings

| User asks | Reality | What to do instead |
|---|---|---|
| "Can you track my organic keyword rank?" | No — Ads API is ad-only | Use a browser agent or a third-party tool (Helium 10, DataDive) |
| "Can you read my reviews?" | No | Use Seller Central UI via browser agent |
| "Can you see what keywords competitors bid on?" | No — opaque | Competitive visibility tools are third-party only |
| "Can you get the historical Buy Box owner?" | No | Only current Buy Box via SP-API pricing |
| "Can you track organic vs paid attribution?" | Partial — only paid is reported | Combine Ads reports with SP-API sales; organic = total − paid-attributed |
| "Can you export the entire account as a spreadsheet?" | Yes — use `exports-snapshots` for structural data, reporting for metrics | Combine both |
| "Can you bulk-edit campaigns across 5 profiles?" | Yes but must iterate profiles | Loop: `setActiveProfile` → bulk op → next profile |

## Complementary MCPs

- **SP-API MCP** (`@lumitec/amazon-sp-api-mcp`) — for seller operations:
  orders, listings, catalog, pricing, FBA inventory, finance.
- **Browser agent** (`claude-in-chrome` / computer-use) — for Seller
  Central UI tasks: Brand Analytics, review triage, A+ Content editing,
  account health.

Use this skill (`amazon-ads`) for PPC / ads. Use `amazon-seller` for
seller ops. Use a browser agent for UI-only tasks.
