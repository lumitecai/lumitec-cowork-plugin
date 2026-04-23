# amazon-ads skill reference index

Load these on demand — SKILL.md (always loaded) tells you which file to read
for which workflow.

| File | When to read |
|---|---|
| `tool-reference.md` | Before building a workflow — full tool catalogue grouped by package |
| `profiles-and-regions.md` | Region-lock errors, multi-country accounts, profile discovery |
| `report-types.md` | Before calling `rp_createAsyncReport` — valid columns, groupBy, reportTypeId |
| `reporting-workflow.md` | Async create → poll → download → parse lifecycle |
| `rate-limits.md` | 429/425 responses, burst budgets, Retry-After handling |
| `error-codes.md` | 4xx/5xx interpretation and recovery |
| `campaign-pitfalls.md` | ADS-001..010 production rules — read before any write |
| `packages-and-allowlist.md` | Discovery-mode selection, `ADS_API_PACKAGES` env var, lazy-mode `enablePackage()` |
| `v2-sunset-migration.md` | Jan 2026 cutover — migrate away from `/v2/*` |
| `api-gaps.md` | What the Ads API can't do — avoid suggesting impossible workarounds |

## Changelog

- 2026-04 — initial release. Covers v3 Reporting, v3 SP, v4 SB, DSP, AMC.
