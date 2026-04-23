# v2 Sunset — completed January 2026

Amazon **sunset** v2 Sponsored Products, v2 Sponsored Brands
(`/hsa/v2/*`), v2 Reports, and AMC v2 workflow endpoints in **January
2026** (the final of several deprecation waves that ran through 2025).
Calls to these paths now return `410 Gone`.

Authoritative list:
`advertising.amazon.com/API/docs/en-us/release-notes/deprecations`.

Earlier waves already happened — if you see legacy code still targeting
these, it is already broken:
- **2025-06-30**: DSP v1 line-item / targeting endpoints; AMC ad-server
  tables.
- **2025-07-01**: Sponsored Brands `/sb/v2/brands`.
- **2025-07-15**: Sponsored Brands moderation v2; Sponsored Products
  bid-recommendations v2.
- **2026-01**: final v2/v3-legacy wave covered by this document.

## What's already broken

If you find code still calling these, the calls are currently returning
`410 Gone` (or `404` on a handful of paths Amazon removed rather than
gated). Rewrite them against the v3/v4 surface below.

| Old (deprecated) | Replace with |
|---|---|
| `/v2/sp/campaigns` (GET list) | `sp_ListSponsoredProductsCampaigns` (v3 POST with filter body) |
| `/v2/sp/campaigns` (POST create) | `sp_CreateSponsoredProductsCampaigns` (v3) |
| `/v2/sp/adGroups`, `/v2/sp/keywords`, `/v2/sp/productAds` | `sp_*` v3 equivalents |
| `/v2/sp/reports` (report create) | `rp_createAsyncReport` with `adProduct: 'SPONSORED_PRODUCTS'` |
| `/v2/sp/{reportId}` (report poll) | `rp_getAsyncReport` |
| `/v2/hsa/campaigns` (SB v2) | `sb_*` v4 tools |
| `/v2/hsa/reports` | `rp_createAsyncReport` with `adProduct: 'SPONSORED_BRANDS'` |
| AMC v2 workflow executions | AMC v3 — `amc_CreateWorkflowExecution` |

## Key behavioural differences

### v2 GET → v3 POST for list

**v2**: `GET /v2/sp/campaigns?stateFilter=enabled&startIndex=0&count=100`

**v3**: `POST /sp/campaigns/list` with body:
```json
{
  "stateFilter": { "include": ["ENABLED"] },
  "maxResults": 100,
  "nextToken": "..."
}
```

- State names UPPERCASE in v3 (`ENABLED`, not `enabled`).
- `startIndex` → `nextToken` (opaque cursor).
- Filter fields are **objects** with `include` / `exclude` arrays.

### v2 synchronous reports → v3 async

**v2**: `POST /v2/sp/campaigns/report` returned a `reportId`, you polled
`GET /v2/reports/{id}`, then downloaded. Limited columns, fixed schemas.

**v3**: `rp_createAsyncReport` → `rp_getAsyncReport` → download presigned
URL. Configurable `columns`, `groupBy`, `timeUnit`. See `report-types.md`.

Rewrite checklist:
- Replace old fixed report types (`campaigns`, `keywords`) with v3
  `reportTypeId` + `columns` (`spCampaigns`, `spTargeting`, etc.).
- Old report files were CSV; v3 default is GZIP_JSON (newline-delimited).
- Retry behaviour is similar but v3 uses 425 for polls-too-early (v2
  returned `PENDING`).

### v2 SB (`/hsa/v2/*`) → SB v4

SB v4 restructures creative around `brandVideo`, `storeSpotlight`, etc.
Not a 1:1 path rename — treat as a rewrite.

## Timeline (for the record)

- Mid-2025: Amazon stopped accepting v2 sandbox registrations.
- Jun–Jul 2025: DSP v1, SB v2 brands/moderation, SP bid-recommendations v2
  waves (see list above).
- **January 2026**: final wave — v2 SP campaigns/adGroups/keywords/ads,
  v2 reports, AMC v2 workflows. Now returning 410.

Old SDK samples still reference `/v2/*` — ignore them.

## If you find v2 calls in legacy scripts

They're returning 410 right now. Rewrite checklist:

- [ ] Grep for `/v2/sp/`, `/v2/hsa/`, `/v2/reports`, `/v2/sb/`.
- [ ] Replace each with the v3/v4 tool from this MCP.
- [ ] Rewrite report-create calls to use `rp_createAsyncReport` +
      `configuration.columns` array.
- [ ] Verify downloaded reports parse as NDJSON (not CSV).
- [ ] Test state-filter values are UPPERCASE.
- [ ] Confirm batch-write response handling checks both `success` and
      `error` arrays (v3 ADS-009).

## What survived

- `/v2/profiles` — the one v2 endpoint still alive; no v3 replacement
  planned as of April 2026. Call it **without** the scope header
  (ADS-002).
- DSP v2+ endpoints — DSP v1 was sunset in Jun 2025, but current DSP
  (v2/v3) is on its own cadence.
- Portfolio / account-budget endpoints — v1 and stable.

## Confirming a path is dead

If a tool unexpectedly returns 410 or 404 with a deprecation notice:
1. Check Amazon's deprecations page
   (`advertising.amazon.com/API/docs/en-us/release-notes/deprecations`).
2. If it's on the list, the MCP's bundled OpenAPI specs should already
   exclude it — if a tool for that path still exists in this MCP, file
   a spec-update issue upstream (KuudoAI).
3. Rewrite against the v3/v4 replacement.
