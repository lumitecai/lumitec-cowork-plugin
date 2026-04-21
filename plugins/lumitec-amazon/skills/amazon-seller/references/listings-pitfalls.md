# Listings Pitfalls — Production Learnings

## LIST-001: Multi-SKU ASIN Trap
- **Trigger**: ASIN has both FBA and MFN SKUs
- **Problem**: Only the content-contributing SKU returns full attributes (title, bullets, description). Other SKUs return sparse data.
- **Action**: Always query the FBA SKU first. If MCP returns empty bullets/description but the live product page shows content → you queried the wrong SKU.
- **Detection**: `grep "B0XXXXXXXX" cache/{region}-{country}-listings.tsv` — if multiple rows appear with different `fulfillment-channel` values (AMAZON vs DEFAULT), this ASIN has the multi-SKU trap.
- **Source**: Production experience

## LIST-002: Patch vs PUT
- **Rule**: Use `patchListingsItem` for partial updates (price, quantity, specific attributes). `putListingsItem` is a full PUT that **overwrites everything**.
- **When to use PUT**: Only for creating a new listing or intentionally replacing all attributes.
- **When to use PATCH**: Price changes, quantity updates, single attribute edits.

## LIST-003: B2B Audience Separation
- **Rule**: Patching `audience: "ALL"` does NOT touch `audience: "B2B"`. They are separate patch targets.
- **Action**: If updating B2B pricing, make a separate patch specifically for the B2B audience.

## LIST-004: No Keyword Search in Listings API
- **Rule**: `searchListingsItems` has no keyword filter. You cannot search listings by product name or description via the API.
- **Workaround**: Grep the listings cache: `grep -i "keyword" cache/{region}-{country}-listings.tsv`
- **Setup**: Refresh with `createReport` → `getReport` → `getReportDocument` → download TSV. See `references/caching-strategy.md` for full workflow.
- **Why not searchCatalogItems?**: Returns the entire Amazon catalogue, not just your listings — your products get buried among competitor results.

## LIST-005: Repricer Conflict
- **Trigger**: Third-party repricer (BQool, Aura, Informed, etc.) is active on the account
- **Problem**: Manual `patchListingsItem` price changes get overwritten by the repricer within seconds to minutes.
- **Action**: Pause the repricer FIRST, or update the repricer's floor/ceiling prices instead of patching directly.
- **Detection**: If a price change reverts within minutes, suspect a repricer.

## LIST-006: Strikethrough Pricing
- **Rule**: `list_price` is the was-price / strikethrough price displayed on the product page.
- **Action**: Set `list_price` HIGHER than the selling price to get the discount badge (e.g., "Was: $29.99, Now: $24.99").
- **Trap**: If `list_price` equals or is lower than selling price, no discount badge appears.

## LIST-007: Charm Pricing & Pack Ladder
- **Rule**: Bigger pack sizes must be cheaper per unit. If a 3-pack is $30 ($10/unit) but a 6-pack is $66 ($11/unit), the pricing ladder is broken.
- **Action**: Always verify per-unit economics when updating pack pricing.
- **Charm pricing**: End prices in .99 or .97 for consumer products.
