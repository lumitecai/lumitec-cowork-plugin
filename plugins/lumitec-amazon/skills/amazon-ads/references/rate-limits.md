# Rate Limits

The Amazon Ads API publishes per-endpoint limits that are not uniform. The
MCP HTTP client handles the common case automatically, but knowing the
limits helps you plan bulk operations.

## The MCP's built-in behaviour

The HTTP client in this MCP:
1. Retries on `425`, `429`, `500`, `502`, `503`, `504` — up to 3 attempts.
2. Honours the `Retry-After` header when present (seconds or HTTP-date).
3. Falls back to exponential backoff with 50% jitter (200ms → 400ms → 800ms).
4. Never retries `4xx` other than 425/429 — those are your problem.

You generally don't need to sleep between calls yourself. But for **bulk
writes** that will definitely hit limits, pacing upstream of the tool call
reduces retry churn.

## Approximate limits by surface

Published numbers vary by region and advertiser tier — these are
rule-of-thumb defaults. Always trust the `Retry-After` header over these.

| Surface | Typical limit |
|---|---|
| Sponsored Products v3 list/create/update | 1–2 requests/second per profile per resource |
| Sponsored Brands v4 | ~1 req/s |
| Sponsored Display | ~1 req/s |
| Reporting v3 `createAsyncReport` | 1 req/s — don't spam creates |
| Reporting v3 `getAsyncReport` (poll) | 10 req/s — polling is cheap |
| Profiles `/v2/profiles` | 2 req/s |
| DSP | tier-dependent; usually 1 req/s |
| AMC | workflow executions are rate-limited per workflow, not per account |

## Batch sizes (the real lever)

The Ads API v3 list/create/update endpoints take an **array body**. Every
entry is 1 unit of work, but it's 1 HTTP request. Pack the arrays:

- **sp_CreateSponsoredProductsKeywords** — up to 1000 keywords per call.
- **sp_UpdateSponsoredProductsCampaigns** — up to 1000 campaigns per call.
- **sp_CreateSponsoredProductsProductAds** — up to 1000 ads per call.

Batching 1000 changes into 1 call vs. 1000 calls is ~1000× the throughput
and dodges rate limits entirely.

The response shape is always `{ success: [...], error: [...] }` (ADS-009) —
a single failed row does not fail the whole batch.

## When to fan out

If you genuinely need thousands of **reads** (e.g. pulling keywords across
50 campaigns), use the `campaignIdFilter.include` array in the list body to
fetch many campaigns' worth of keywords in one request. Don't loop one
campaign at a time.

If you do loop, pace at 1 req/s with small jitter. Don't Promise.all 50
requests — that will trip the bucket and waste retries.

## 429 vs 425

- **429** — you're over the rate limit. Retry-After says how long to wait.
- **425 "Too Early"** — a follow-up request (usually a poll) is too soon
  after a state change. Amazon returns this on reports that aren't yet in
  the queue. Same retry logic works.

Both are automatically retried by the MCP client.

## Burst vs sustained

Amazon's buckets are token-bucket style. You can burst a handful of
requests, then must wait for refill. The MCP doesn't pre-throttle — it
reacts to 429s. For long loops, 1 req/s is safe on almost every endpoint.

## Reporting-specific pacing

- Don't create 20 reports in parallel — many will fail with 429.
- Do create 5 reports, then start polling them. Polls are cheap.
- Use `timeUnit: 'SUMMARY'` + broader `groupBy` to reduce the number of
  report creates needed.

## Debugging rate-limit issues

1. Check the response headers: Amazon returns `x-amzn-RequestId` — log it
   when reporting a 429 upstream.
2. `invalidateCaches` + retry will NOT help on 429 — it's a server-side
   bucket, not a client cache.
3. If you're seeing 429s on read endpoints you haven't hit hard, another
   tenant on the same LWA app may be burning the bucket. Multi-tenant HTTP
   mode isolates contexts but not Amazon-side rate buckets.
