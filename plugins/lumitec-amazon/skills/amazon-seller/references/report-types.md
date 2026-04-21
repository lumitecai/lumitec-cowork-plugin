# Report Types Catalogue

## Commonly Used Reports

| Report | Type String | Notes |
|---|---|---|
| All Listings | `GET_MERCHANT_LISTINGS_ALL_DATA` | Full catalogue — refresh daily |
| Active Listings | `GET_FLAT_FILE_OPEN_LISTINGS_DATA` | Currently active only |
| All Orders (30d) | `GET_FLAT_FILE_ALL_ORDERS_DATA_BY_LAST_UPDATE_GENERAL` | Max 30-day window per report |
| FBA Inventory | `GET_FBA_MYI_ALL_INVENTORY_DATA` | Current FBA stock |
| FBA Inventory Age | `GET_FBA_INVENTORY_AGED_DATA` | Aging analysis |
| FBA Returns | `GET_FBA_FULFILLMENT_CUSTOMER_RETURNS_DATA` | Return details |
| FBA Reimbursements | `GET_FBA_REIMBURSEMENTS_DATA` | Reimbursement transactions |
| FBA Fees | `GET_FBA_ESTIMATED_FBA_FEES_TXT_DATA` | Estimated fee breakdown |
| Settlements | `GET_V2_SETTLEMENT_REPORT_DATA_FLAT_FILE` | Financial settlements |
| Amazon Fulfilled | `GET_AMAZON_FULFILLED_SHIPMENTS_DATA` | FBA-fulfilled orders |

## Async Report Workflow

Reports are **asynchronous**. Follow this sequence:

```
1. createReport(reportType, marketplaceIds)
   → returns { reportId }

2. Poll: getReport(reportId)
   → status: IN_QUEUE → IN_PROGRESS → DONE
   → when DONE, response includes reportDocumentId

3. getReportDocument(reportDocumentId)
   → returns { url } — PRE-SIGNED, EXPIRES IN 5 MINUTES!

4. Download from the URL immediately
   → TSV/CSV format depending on report type
```

## Parameters

- `dataStartTime` / `dataEndTime` — ISO 8601 format, optional for most reports
- `GET_FLAT_FILE_ALL_ORDERS_DATA_BY_LAST_UPDATE_GENERAL` — max 30-day window per request
- `marketplaceIds` — always pass explicitly

## Recommended Refresh Cadence

| Report | Cadence | Reason |
|---|---|---|
| All Listings | Daily | Catalogue changes slowly |
| Orders | On-demand or 4x daily | Near-real-time via API is better |
| FBA Inventory | Daily | Stock reconciliation |
| Returns | Weekly | Low volume usually |
| Reimbursements | Weekly | Settlement cycle |
