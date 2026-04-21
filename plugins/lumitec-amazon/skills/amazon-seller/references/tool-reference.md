# Tool Reference

## AUTH

### checkCredentials
- Params: none
- Returns: success message or error
- Use: always call first to verify setup

### getAccessToken
- Params: none
- Returns: success confirmation (token is cached internally, never exposed)
- Rate: cached, only calls Amazon when expired

---

## CATALOG (2022-04-01)

### getCatalogItem
- Params: `asin` (required), `marketplaceId` (optional)
- Returns: product details (title, brand, dimensions, images)
- Rate: 2 req/s burst

### searchCatalogItems
- Params: `keywords` (required), `marketplaceId` (optional), `includedData` (optional array)
- Returns: list of matching items
- Rate: 2 req/s burst

---

## ORDERS (v0 — upgrade to v2026-01-01 when app role granted, deadline Mar 2027)

### searchOrders
- Params: `createdAfter`, `createdBefore`, `lastUpdatedAfter`, `lastUpdatedBefore`, `orderStatuses` (array), `marketplaceIds` (array), `fulfillmentChannels` (array), `maxResultsPerPage`, `nextToken`
- Required: at least one date filter OR nextToken
- Returns: `{ orders: [...], nextToken }`
- Rate: 0.0167 req/s steady (very slow — ~1 per minute)

### getOrder
- Params: `orderId` (required), `includedData` (optional array: BUYER, RECIPIENT, FULFILLMENT, PROCEEDS, EXPENSE, PROMOTION, CANCELLATION, PACKAGES)
- Returns: order details with selected included data in one consolidated call
- Rate: 0.5 req/s
- **Tip**: Use includedData to avoid separate getOrderItems calls

### getOrderItems
- Params: `orderId` (required)
- Returns: line items via getOrder with includedData=FULFILLMENT (convenience wrapper)
- Rate: 0.5 req/s

---

## INVENTORY (v1)

### getInventorySummaries
- Params: `sellerSkus` (optional array), `marketplaceId`, `granularityType` (Marketplace|ASIN|Seller), `granularityId`
- Returns: inventory levels by SKU
- Rate: 2 req/s

### updateInventory
- Params: `sellerSku` (required), `quantity` (required int), `fulfillmentLatency` (optional int)
- Returns: update confirmation
- Rate: 2 req/s

---

## FBA

### getInboundEligibility
- Params: `asin` (required), `programType` (INBOUND|COMMINGLING), `marketplaceId`
- Returns: eligibility status
- Rate: 2 req/s

### getFbaInventorySummaries
- Params: `details` (boolean), `granularityType`, `granularityId`, `marketplaceId`
- Returns: FBA inventory with optional detail breakdown
- Rate: 2 req/s

### listInboundPlans
- Params: `pageSize` (optional, max 25), `paginationToken`, `status` (ACTIVE|VOIDED|SHIPPED|ERRORED), `sortBy` (LastUpdatedDate|CreatedDate), `sortOrder` (ASC|DESC)
- Returns: list of FBA inbound plans
- Replaces removed v0 getShipments endpoint

### getInboundPlan
- Params: `inboundPlanId` (required)
- Returns: inbound plan details

---

## FINANCE (v0 — sunset Aug 28, 2026)

### listFinancialEventGroups
- Params: `financialEventGroupStartedAfter` (**required**), `maxResultsPerPage`, `financialEventGroupStartedBefore`
- Returns: event groups (settlement periods)
- **Note**: date param is required by the API — omitting it returns an error

### listFinancialEvents
- Params: `maxResultsPerPage`, `postedAfter`, `postedBefore`
- Returns: individual financial events

### listFinancialEventsByOrderId
- Params: `orderId` (required), `maxResultsPerPage`, `nextToken`
- Returns: fees, charges, refunds for a specific order
- **Use for per-order profitability analysis**

### getFinancialEventGroup
- Params: `eventGroupId` (required)
- Returns: all events in a specific group

---

## LISTINGS (2021-08-01)

### getListingsItem
- Params: `sellerId` (optional, auto-filled from the install-time `X-SP-API-Seller-Id` header), `sku`, `marketplaceIds` (array), `issueLocale`
- Returns: listing attributes, issues, status

### putListingsItem
- Params: `sellerId` (optional, defaults from env), `sku`, `marketplaceIds`, `issueLocale`, `productType`, `requirements`, `attributes` (object)
- **WARNING**: PUT overwrites ALL data. Use for create or full replace only.

### patchListingsItem
- Params: `sellerId` (optional, defaults from env), `sku`, `marketplaceIds`, `issueLocale`, `productType`, `patches` (array of {op, path, value})
- `op`: add | replace | delete
- `path`: JSON Pointer (e.g. `/attributes/purchasable_offer`)
- `value`: array of attribute values (for add/replace)
- **Use this for partial updates** (price, quantity, attributes). Never use PUT for partial changes.

### deleteListingsItem
- Params: `sellerId` (optional, defaults from env), `sku`, `marketplaceIds`, `issueLocale`
- Returns: deletion confirmation

---

## REPORTS (2021-06-30)

### createReport
- Params: `reportType` (required), `marketplaceIds` (array), `dataStartTime`, `dataEndTime`
- Returns: `{ reportId }` — async, must poll

### getReport
- Params: `reportId` (required)
- Returns: status (IN_QUEUE, IN_PROGRESS, DONE, CANCELLED, FATAL)

### getReportDocument
- Params: `reportDocumentId` (required)
- Returns: download URL (**expires in 5 minutes!**)

### cancelReport
- Params: `reportId` (required)
- Returns: cancellation confirmation

### createReportSchedule
- Params: `reportType` (required), `marketplaceIds` (array), `period` (P1D, P7D, P30D, etc.), `nextReportCreationTime`
- Returns: schedule ID — automates recurring report generation

### getReportSchedules
- Params: `reportTypes` (**required** array — API rejects without it)
- Returns: list of active report schedules

### cancelReportSchedule
- Params: `reportScheduleId` (required)
- Returns: cancellation confirmation

### getReports
- Params: `reportTypes` (array), `processingStatuses` (array), `marketplaceIds` (array), `maxResults`
- Returns: list of reports matching filters

---

## FEEDS (2021-06-30)

### createFeed
- Params: `feedType`, `marketplaceIds` (array), `inputFeedDocumentId`
- Returns: `{ feedId }` — async

### getFeed
- Params: `feedId` (required)
- Returns: feed status and details

### getFeedDocument
- Params: `feedDocumentId` (required)
- Returns: document download URL

---

## NOTIFICATIONS (v1)

### getSubscription
- Params: `notificationType` (required), `marketplaceId`

### createSubscription
- Params: `notificationType`, `payloadVersion`, `destinationId`, `marketplaceId`

### deleteSubscription
- Params: `notificationType` (required), `subscriptionId` (required)
- Returns: deletion confirmation

### getDestinations
- Params: none

### deleteDestination
- Params: `destinationId` (required)
- Returns: deletion confirmation

### createDestination
- Params: `name`, `resourceSpecification` (sqs.arn or eventBridge.accountId)

---

## SELLER (v1)

### getMarketplaceParticipations
- Params: none
- Returns: list of marketplaces seller is registered in

---

## PRODUCT PRICING (v0)

### getPricing
- Params: `itemType` (Asin|Sku), `itemIds` (array, **up to 20**), `marketplaceId`
- Always batch — never loop single calls

### getCompetitivePricing
- Params: `itemType` (Asin|Sku), `itemIds` (array, **up to 20**), `marketplaceId`
- Always batch — never loop single calls

### getItemOffersBatch
- Params: `requests` (array of {uri, method, MarketplaceId, ItemCondition, CustomerType})
- Batch get lowest offers for **up to 20 ASINs** at once
- Far more efficient than looping getListingOffers

### getListingOffers
- Params: `sellerSku`, `marketplaceId`, `itemCondition` (New|Used|Collectible|Refurbished|Club)
- Returns: lowest priced offers for a single SKU

---

## PRODUCT FEES (v0)

### getMyFeesEstimateForSKU
- Params: `sellerSku` (required), `marketplaceId`, `listingPrice` (required), `currencyCode`, `isAmazonFulfilled` (required), `shippingPrice`
- Returns: referral fees, FBA/FBM fees, total estimated fees

### getMyFeesEstimateForASIN
- Params: `asin` (required), `marketplaceId`, `listingPrice` (required), `currencyCode`, `isAmazonFulfilled` (required), `shippingPrice`
- Returns: referral fees, FBA/FBM fees, total estimated fees

### getMyFeesEstimates
- Params: `items` (array up to 20: {idType, idValue, listingPrice, currencyCode, isAmazonFulfilled, shippingPrice}), `marketplaceId`
- Batch estimate — always use this for multiple items

---

## PRODUCT TYPE DEFINITIONS (2020-09-01)

### searchDefinitionsProductTypes
- Params: `keywords` (optional), `marketplaceId`, `itemName` (optional)
- Returns: matching product types
- **Use before creating listings** to find the correct productType

### getDefinitionsProductType
- Params: `productType` (required), `marketplaceId`, `requirements` (LISTING|LISTING_PRODUCT_ONLY|LISTING_OFFER_ONLY), `locale`
- Returns: JSON schema of required/optional attributes
- **Essential companion to Listings API** — tells you exactly what fields to populate

---

## LISTINGS RESTRICTIONS (2021-08-01)

### getListingsRestrictions
- Params: `asin` (required), `sellerId` (optional, defaults from env), `marketplaceIds` (array), `conditionType`, `reasonLocale`
- Returns: restriction reasons (gating, approval required, etc.)
- **Always check before attempting to create a new listing**
