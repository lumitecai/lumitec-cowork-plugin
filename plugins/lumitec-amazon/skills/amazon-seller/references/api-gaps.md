# What SP-API Can't Do

Document these gaps upfront so agents don't spin trying to find tools that don't exist.

## Not Available via SP-API

| Feature | Why | Alternative |
|---|---|---|
| Buy Box analytics | Not exposed in any API | Seller Central UI via agent-browser |
| A+ Content / EBC | No API for enhanced brand content | Seller Central UI via agent-browser |
| PPC / Advertising | Separate API entirely | Amazon Ads API (Bearer auth, different base URL) |
| Reviews & ratings | Not available programmatically | Catalog API returns basic review count only |
| Brand Analytics | Restricted to Brand Registry sellers, no API | Seller Central UI via agent-browser |
| Keyword ranking | Not tracked by Amazon's APIs | Third-party tools (Helium 10, Jungle Scout) |
| Search term reports | Only via Brand Analytics UI | Seller Central UI via agent-browser |
| Account health | Limited API, mostly UI-only | Seller Central UI via agent-browser |
| Case management | No API for support cases | Seller Central UI via agent-browser |
| Product images upload | Must use Feeds API with specific feed types | `POST_PRODUCT_IMAGE` feed type |

## When Users Ask for These

1. **Advertising/PPC**: Point to the Amazon Ads API — separate credentials, Bearer auth only, different endpoint (advertising-api.amazon.com)
2. **Seller Central UI tasks**: Suggest agent-browser skill for browser automation
3. **Third-party data**: Acknowledge the gap and suggest appropriate external tools
4. **Brand Analytics**: Only available to Brand Registry sellers in Seller Central

## What SP-API CAN Do (common misconceptions)

- Financial data — YES, via Finance API (settlements, fees, refunds)
- Bulk updates — YES, via Feeds API (price, inventory, listings)
- Real-time order notifications — YES, via Notifications API (ORDER_CHANGE)
- Competitor pricing — YES, via Product Pricing API (getCompetitivePricing)
