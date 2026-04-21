# SP-API Rate Limits

## Token Bucket Model

SP-API uses **token bucket throttling** per endpoint per seller — not a global quota.
Each endpoint has a steady-state rate (tokens added/second) and a burst limit (max bucket size).
When the bucket empties → 429 response.

## Key Endpoint Rates

| Endpoint | Steady Rate | Burst | Notes |
|---|---|---|---|
| searchOrders | 0.0167/s | 20 | ~1 per minute — very slow |
| getOrder | 0.5/s | 30 | |
| getOrderItems | 0.5/s | 30 | Heavily throttled — batch carefully |
| getCatalogItem | 2/s | 2 | |
| searchCatalogItems | 2/s | 2 | |
| getPricing | 0.5/s | 1 | Takes up to 20 items per call |
| getCompetitivePricing | 0.5/s | 1 | Takes up to 20 items per call |
| getListingOffers | 1/s | 1 | Single SKU only |
| getInventorySummaries | 2/s | 2 | |
| createReport | 0.0167/s | 15 | |
| getReport | 2/s | 15 | Use for polling |
| getReportDocument | 0.0167/s | 15 | |
| putListingsItem | 5/s | 10 | |
| getListingsItem | 5/s | 10 | |

## Batch Endpoints (always prefer over loops)

- `getPricing` — up to **20 ASINs or SKUs** per call
- `getCompetitivePricing` — up to **20 ASINs or SKUs** per call
- Never loop single-item calls when a batch endpoint exists

## Handling 429 Responses

The MCP server has **built-in retry with exponential backoff + jitter** for 429, 500, and 503 responses (up to 3 retries). It reads the `x-amzn-RateLimit-Limit` header and logs it on throttle.

You don't need to handle retries yourself — but you should still:
1. Space requests evenly (don't fire 20 calls in parallel)
2. Don't retry-loop on top of the built-in retry

## Best Practices

- Use batch endpoints wherever available (getPricing, getCompetitivePricing: 20 items/call)
- For today's orders / recent status: use API (includes Pending + Unshipped)
- For bulk order data (7+ days): use reports instead of API — see `caching-strategy.md`
- For bulk reads: use cached report data (grep locally instead of API calls)
- Use notifications (SQS/EventBridge) instead of polling where possible
