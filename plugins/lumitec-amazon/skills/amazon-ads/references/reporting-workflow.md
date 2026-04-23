# Reporting v3 — Async Workflow

Every Reporting v3 request follows the same lifecycle:
**create → poll → download → gunzip → parse**.

## 1. Create

```
rp_createAsyncReport({
  body: {
    name: 'sp-campaigns-7d',
    startDate: '2026-04-10',
    endDate: '2026-04-17',
    configuration: {
      adProduct: 'SPONSORED_PRODUCTS',
      groupBy: ['campaign'],
      columns: ['campaignId','campaignName','impressions','clicks','cost','sales7d','acosClicks7d'],
      reportTypeId: 'spCampaigns',
      timeUnit: 'SUMMARY',
      format: 'GZIP_JSON'
    }
  }
})
→ { reportId: 'abc-123', status: 'PENDING', ... }
```

Common create-time errors:
- `400 INVALID_PARAMETER_VALUE` on `columns[X]` — column name typo or column not valid for this `reportTypeId`. Check `report-types.md`.
- `400` on `endDate` — exceeds window limit.
- `400` on `startDate` — future date, or older than the product's retention (usually 95d historical).

## 2. Poll

```
rp_getAsyncReport({ reportId: 'abc-123' })
→ { status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED', url?, failureReason? }
```

Typical pattern:
- First-result latency: 30–90 seconds for most reports.
- `SUMMARY` + small window usually completes inside 30s.
- `DAILY` + 90d window can take 2–5 minutes.
- Backoff: 5s → 10s → 15s → 20s → cap at 30s between polls.
- Give up after 10 minutes; something's wrong upstream.

`status: 'FAILED'` is usually transient (Amazon backfill lag). Re-create.

## 3. Fetch the data

`COMPLETED` responses include a **presigned `url`**. Lifetime ~1 hour.

Use the hand-coded **`checkAndDownloadExport`** tool:

```
checkAndDownloadExport({ statusPath: '/reporting/reports/abc-123', exportId: 'abc-123' })
→ {
    ok: true,
    status: 'COMPLETED',
    exportId: 'abc-123',
    format: 'ndjson',
    encoding: 'gzip (decompressed)',
    compressedBytes: 8432,
    decompressedBytes: 48219,
    inline: true,
    payload: '{"campaignId":...}\n{"campaignId":...}\n...'
  }
```

Behaviour:
1. Polls the status endpoint until `COMPLETED`.
2. Fetches Amazon's presigned S3 URL into the MCP server's RAM.
3. Decompresses (gzip → text).
4. If the decompressed payload is under `maxInlineBytes` (default 2 MB),
   returns it inline as text in the `payload` field — Claude reads it
   directly. `format` is sniffed as `ndjson`, `json-array`, `csv`, or
   `unknown` so you know how to parse.
5. If bigger than the inline cap, the tool returns
   `{ inline: false, presignedUrl, urlExpiresAt, decompressedBytes, hint }` —
   the client (you, or the user via a shell tool) should download Amazon's
   presigned URL directly. Lumitec never persists the payload.

**For reports under 2 MB:** just read `payload`, split on `\n` (for
`ndjson`), `JSON.parse` each line. Done.

**For reports over 2 MB:** hand the `presignedUrl` to the user's shell —
`curl -o report.gz "<presignedUrl>" && gunzip report.gz`. Amazon S3 →
user's machine, no Lumitec hop.

## 4. Parse

`GZIP_JSON` decompresses to **newline-delimited JSON** — one row per line:

```
{"campaignId":123,"campaignName":"US-Prime-TV","impressions":45231,"clicks":892,...}
{"campaignId":124,"campaignName":"US-Kitchen-Gadgets","impressions":98712,...}
```

Parse incrementally if the file is large:
- Node: `readline` over the file stream, `JSON.parse` each line.
- Don't load the whole thing into a single `JSON.parse` — for `timeUnit: 'DAILY'` with a 95-day window across many campaigns, the file can be tens of MB.

## 5. Compute

Standard derived metrics:
- **ACoS** = `cost / sales{N}d` × 100%
- **TACoS** = `cost / totalSales` (requires SP-API sales data for the denominator — Ads API doesn't know total sales)
- **ROAS** = `sales{N}d / cost`
- **CTR** = `clicks / impressions` × 100%
- **CPC** = `cost / clicks`
- **CVR** = `purchases{N}d / clicks` × 100%

## Full example end-to-end

```
// 1. Create
const created = await rp_createAsyncReport({ body: { ... } });

// 2. Poll
let report;
for (let i = 0; i < 20; i++) {
  report = await rp_getAsyncReport({ reportId: created.reportId });
  if (report.status === 'COMPLETED') break;
  if (report.status === 'FAILED') throw new Error(report.failureReason);
  await sleep(Math.min(30000, 5000 + i * 5000));
}

// 3. Fetch (server streams → memory → MCP response)
const result = await checkAndDownloadExport({
  statusPath: `/reporting/reports/${created.reportId}`,
  exportId: created.reportId,
});

// 4. Parse NDJSON (from the inline payload — no file system involved)
if (!result.inline) throw new Error(`Report too large; download directly: ${result.presignedUrl}`);
const rows = result.payload.trim().split('\n').map(JSON.parse);

// 5. Compute + present
const summary = rows.map(r => ({
  campaign: r.campaignName,
  spend: r.cost,
  sales: r.sales7d,
  acos: (r.cost / r.sales7d * 100).toFixed(1) + '%'
}));
```

## Exports vs Reports

If you need the full current state of an advertiser (all campaigns, ad
groups, keywords — not metrics), use the **exports-snapshots** package
instead. Exports are slower but return complete object graphs, not
time-series metrics.

Lifecycle is the same: create → poll → download.
