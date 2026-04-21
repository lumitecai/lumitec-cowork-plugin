# SP-API Cost Optimization

## Fee Structure (Effective 2026)

| Tier | Monthly GET Calls Included | Price |
|---|---|---|
| Basic | 2.5M | $1,400/year |
| Pro | 25M | $1,000/month |
| Plus | 250M | $10,000/month |
| Enterprise | Custom | Custom |

**Overage**: $0.40 per 1,000 GET calls beyond tier limit (starting April 30, 2026).

## Call Reduction Strategies

### 1. Batch Endpoints
- `getPricing` — 20 items/call instead of 20 separate calls
- `getCompetitivePricing` — 20 items/call
- Always batch when the endpoint supports it

### 2. Event-Driven Over Polling
- Subscribe to `ORDER_CHANGE` notifications instead of polling `searchOrders`
- Subscribe to `REPORT_PROCESSING_FINISHED` instead of polling `getReport`
- Use SQS or EventBridge destinations

### 3. Cache Report Data
- Download listings/orders/inventory reports once, grep locally
- See `caching-strategy.md` for refresh cadence

### 4. Consolidated API Calls
- Orders v2026-01-01 `getOrder` with `includedData` parameter gets everything in ONE call
  (replaces getOrder + getOrderItems = 2 calls → 1 call)

### 5. Minimize Redundant Calls
- Cache seller ID from `getMarketplaceParticipations` (it doesn't change)
- Cache access tokens (built into the MCP server with 1-min buffer)
- Don't re-fetch data you already have in the current session

## Monthly Call Budget Estimation

| Operation | Calls/day | Calls/month | Notes |
|---|---|---|---|
| Daily listings refresh | 3 | 90 | create + poll + download |
| Order monitoring (4x/day) | 12 | 360 | create + poll + download x4 |
| On-demand order lookups | ~50 | ~1,500 | searchOrders + getOrder |
| Pricing checks | ~20 | ~600 | Batched getPricing |
| **Total estimate** | ~85 | ~2,550 | Well within Basic tier |

High-volume sellers with automated repricing or multi-marketplace operations
should budget for Pro tier (25M calls/month).
