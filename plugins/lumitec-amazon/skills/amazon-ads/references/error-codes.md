# Error Codes

## Quick reference

| Code | Typical cause | Fix |
|---|---|---|
| 400 `INVALID_PARAMETER_VALUE` | column name typo, invalid date range, enum mismatch | read `error.details[]` — it names the field |
| 400 `MISSING_REQUIRED_PARAMETER` | missing body field | check tool schema via `getToolSchema` |
| 401 `Unauthorized` | expired/stale access token, or LWA scope missing for the region | `invalidateCaches` + retry; if persistent, see ADS-001 |
| 403 `Forbidden` | LWA app missing advertising scope | server config — not a credentials issue |
| 404 | unknown ID (campaign/ad-group/keyword) OR scope header doesn't match profile | verify `profileId` is set and matches the IDs |
| 422 | bid below marketplace floor, budget below min, invalid state transition | read body — Amazon names the rule |
| 425 `Too Early` | report-poll before it's queued, state-change follow-up too fast | retry with backoff (auto-retried) |
| 429 `Too Many Requests` | rate limit | honour `Retry-After` (auto-retried) |
| 5xx | Amazon transient | retried 3× (auto) |

## 400 subcodes — the important ones

### `INVALID_PARAMETER_VALUE`

Often fired from Reporting v3. Check:
- **Column name** — `sales7d` is valid on SP; `sales14d` isn't (SP default is 7d). Cross-check `report-types.md`.
- **Date range** — 60d on search-term, 95d on everything else.
- **Enum case** — `matchType: 'EXACT'` not `'exact'`. Most Ads API enums are UPPERCASE.
- **ASIN format** — must be 10 chars, start with `B0`.

### `MISSING_REQUIRED_PARAMETER`

v3 list endpoints often require `maxResults` or a non-empty filter. If you
call `sp_ListSponsoredProductsCampaigns` with `{ body: {} }` expecting
"give me everything", some versions return this. Pass
`{ body: { maxResults: 100 } }`.

### `CONFLICTING_FIELDS`

You passed two mutually-exclusive fields. Example: `adGroupId` and
`campaignId` in the same keyword filter. Pick one grain.

## 401 — stale access token or mis-scoped LWA app (ADS-001)

If `checkCredentials` passes for EU but a cross-region tool call returns
401:

1. `invalidateCaches` + retry — clears a stale access token bound to the
   previous region.
2. If still failing, confirm your LWA app has the advertising scopes
   attached for every region you use. A single refresh token usually
   covers all three, but only if the consent was granted with the right
   scopes.
3. Only if (1) and (2) don't help: set `ADS_API_{REGION}_REFRESH_TOKEN` to
   a region-specific token. This is rare — see ADS-001 in pitfalls.

Amazon does not tell you "wrong region" explicitly — the error message is
just `Unauthorized`, which is why it used to be misdiagnosed as a
refresh-token problem.

## 403 — scope missing from LWA app

Your LWA app needs the `advertising::campaign_management` scope (and
`advertising::account_management` for profiles). These are requested at
authorisation time and baked into the refresh token. You can't add scopes
to an existing refresh token — you must re-authorise.

Symptom: `checkCredentials` returns a valid access token, but every tool
call returns 403. Fix: re-mint refresh tokens with the correct scopes.

## 404 — scope / ID mismatch

Most common: the active profile doesn't own the campaign/ad-group/keyword
you're asking about. Two common cases:

- You called `setActiveProfile` with one profile, but the campaignId
  belongs to a different profile. Switch profile.
- You're on the right profile but the ID was moved to an archived state
  that hides it from reads. Pass `stateFilter: { include: ['ARCHIVED'] }`.

## 422 — business-rule violations

The Ads API uses 422 for "your input is syntactically fine but violates a
business rule". Common ones:

- **Bid below floor** — `{ bid: 0.01 }` in US (floor ~$0.02). Check
  `campaign-pitfalls.md` ADS-006.
- **Budget below minimum** — US daily minimum is $1.
- **Negative bid change on a paused campaign** — some state transitions
  are blocked.
- **Invalid date on end-dated campaign** — `endDate` before `startDate`.

Read the error body — Amazon names the rule.

## Batch-write partial failure

Every batch-write endpoint returns:

```json
{
  "success": [{ "campaignId": "...", "index": 0 }],
  "error": [{ "code": "422", "details": "...", "index": 1 }]
}
```

HTTP status will be **200** even if every row failed. Always check
`error[].length` — `success` being non-empty doesn't mean the batch was OK.

## 5xx — what the MCP does

The HTTP client retries 500/502/503/504 three times with exp backoff + 50%
jitter. If all three fail, the error surfaces to the tool caller. Don't
wrap your own retry on top — the MCP already has one.

## Debugging checklist

When a tool returns an unexpected error:
1. **`x-amzn-RequestId`** — log it (the MCP surfaces it in the error).
2. Cross-check the tool schema: `getToolSchema('sp_...')`.
3. `invalidateCaches` and retry once — covers stale token / profile cache.
4. If the error is about a column/field name, check `report-types.md` or
   the tool's OpenAPI spec (in `openapi/resources/`).
5. If about state/region, confirm `setActiveProfile` was called and the
   profile's country is what you expect.
