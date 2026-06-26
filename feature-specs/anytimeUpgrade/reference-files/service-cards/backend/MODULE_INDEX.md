# Module Index — bridge

---

## Overview

Bridge is a Next.js BFF for the Adobe VIPMP partner portal. It owns the browser UI and a server-side API proxy layer. Its sole job is to acquire an IMS access token via client credentials and forward every domain operation to the Adobe Partner API. Bridge holds no domain data of its own. Eight capabilities are implemented across seven controller modules, each corresponding to a Next.js API route.

---

## Source Structure

| Layer | Path | File count |
|-------|------|-----------|
| API route handlers | `pages/api/` | 9 |
| Controllers | `controllers/` | 7 |
| Zod models + types | `models/` | 11 |
| React pages | `pages/` | 7 |
| React components | `components/` | ~30 |
| React hooks | `hooks/` | 7 |
| React contexts | `contexts/` | 2 |
| Shared utilities | `utils/` | 18 |
| TypeScript types | `types/` | 7 |
| Tests | `tests/` | 16 |
| Static lib | `lib/` | 1 |

---

## Capability List

| Capability | Primary Handler | Stability |
|-----------|----------------|-----------|
| CustomerManagement | `pages/api/customers.ts` | STABLE |
| ResellerManagement | `pages/api/resellers.ts` | STABLE |
| OrderManagement | `pages/api/orders.ts` | STABLE |
| SubscriptionManagement | `pages/api/subscriptions.ts` | STABLE |
| PricelistFetch | `pages/api/pricelist.ts` | STABLE |
| RecommendationFetch | `pages/api/recommendation.ts` | STABLE |
| PartnerDetails | `pages/api/partnerDetails.ts` | STABLE |
| UnifiedSearch | `pages/api/search.ts` | STABLE |

---

## Capability Catalogue

### CustomerManagement

- **Handler:** `pages/api/customers.ts · handler`
- **Scope:** GET (getCustomer, getAllCustomers), POST (createCustomer), PATCH (updateCustomer). No subscription or order management here.
- **Services:** `customerController.getCustomer`, `customerController.getAllCustomers`, `customerController.createCustomer`, `customerController.updateCustomer`, `customerController.findCustomerByName`
- **Connector(s):** [AdobePartnerAPI](#2-adobepartnerapi-vipmp) (all operations); [AdobeIMS](#1-adobeims-oauth2-token-endpoint) (token, via `handlePrerequisites`)
- **Domain:** `CustomerDetails`, `CustomerMinimal`, `CreateCustomer`, `UpdateCustomer`, `PaginatedCustomersResponse`
- **Validation:** Zod — `CreateCustomerSchema` / `UpdateCustomerSchema` validated in controller before fetch; `CustomerDetailsSchema` / `CustomerMinimalSchema` validated on response
- **Events:** none
- **Key Decisions:**
  - **System-of-record:** Adobe Partner API — Bridge is a pass-through. **Enforcement:** Bridge never writes to a local store; all mutations go directly to `PARTNER_API_BASE_URL/v3/customers`.
  - **Idempotency:** Not enforced. Duplicate POST calls create duplicate customers. **Enforcement:** not enforced in code — relies on caller discipline.
  - **Fail posture:** Fail-closed. If Partner API is down, all customer operations fail with the upstream status code. **Enforcement:** no fallback path in any customer controller function.
  - **Stability:** STABLE
- **Call graph (getAllCustomers):**
  1. `handler` calls `handlePrerequisites(req)` → `getAccessToken()` → POST to IMS token endpoint
  2. `handler` calls `getAllCustomers(resellerId, offset, limit, accessToken)`
  3. `getAllCustomers` issues `GET /v3/resellers/{resellerId}/customers?offset=&limit=` with Bearer + x-api-key headers
  4. Response parsed, each account mapped through `CustomerMinimalSchema.parse` (invalid entries skipped with warning)
  5. Returns `PaginatedCustomersResponse` with `hasMore` computed from `offset + limit < totalCount`

---

### ResellerManagement

- **Handler:** `pages/api/resellers.ts · handler`
- **Scope:** GET (getResellerDetails, getAllResellers), POST (createReseller). Does not handle reseller search (delegated to UnifiedSearch).
- **Services:** `resellerController.getAllResellers`, `resellerController.getResellerDetails`, `resellerController.createReseller`, `resellerController.findResellerByName`
- **Connector(s):** [AdobePartnerAPI](#2-adobepartnerapi-vipmp); [AdobeIMS](#1-adobeims-oauth2-token-endpoint) (token)
- **Domain:** `Reseller`, `ResellerMinimal`, `ResellerDetails`
- **Validation:** Zod — `ResellerSchema` validated in controller before POST; `ResellerMinimalSchema` validated per item on list responses; `ResellerDetailsSchema` validated on getResellerDetails response
- **Events:** none
- **Key Decisions:**
  - **System-of-record:** Adobe Partner API. **Enforcement:** all mutations proxied directly to `PARTNER_API_BASE_URL/v3/resellers`.
  - **Idempotency:** Not enforced for POST. **Enforcement:** not enforced in code.
  - **Fail posture:** Fail-closed. **Enforcement:** no fallback path.
  - **Stability:** STABLE

---

### OrderManagement

- **Handler:** `pages/api/orders.ts · handler`
- **Scope:** GET (getOrders, getOrdersHistory), POST (NEW, RETURN, PREVIEW, PREVIEW_RENEWAL, RENEWAL). All order types share a single `createOrder` core function.
- **Services:** `orderController.getOrders`, `orderController.getOrdersHistoryForCustomer`, `orderController.createNewOrder`, `orderController.createReturnOrder`, `orderController.createPreviewOrder`, `orderController.createPreviewRenewalOrder`, `orderController.createRenewalOrder`
- **Connector(s):** [AdobePartnerAPI](#2-adobepartnerapi-vipmp); [AdobeIMS](#1-adobeims-oauth2-token-endpoint) (token)
- **Domain:** `Order`, `LineItem`, `Pricing`, `OrdersHistoryResponseWithPagination`, `NewOrderSchema`, `ReturnOrderSchema`, `PreviewOrderSchema`, `PreviewRenewalOrderSchema`, `RenewalOrderSchema`
- **Validation:** Zod — each order type validated with its own schema before `createOrder` is called; `OrderSchema` validated on response (passthrough — extra fields preserved); `OrdersHistoryResponseSchema` on history response
- **Events:** none
- **Key Decisions:**
  - **System-of-record:** Adobe Partner API. **Enforcement:** all order mutations forwarded directly to `PARTNER_API_BASE_URL/v3/customers/{id}/orders`.
  - **Idempotency:** `externalReferenceId` is accepted and forwarded to Partner API; dedup is Partner API's responsibility. **Enforcement:** not enforced locally — relies on Partner API idempotency.
  - **Fail posture:** Fail-closed. For non-preview orders, errors propagate upstream error body verbatim to browser. **Enforcement:** `orders.ts` JSON-parses `ApiError.message` and returns the parsed object.
  - **Stability:** STABLE
- **Call graph (createNewOrder):**
  1. `handler` calls `handlePrerequisites` → IMS token
  2. `handler` calls `createNewOrder(customerId, externalReferenceId, currencyCode, lineItems, accessToken)`
  3. `createNewOrder` builds body `{customerId, orderType:'NEW', ...}`, validates with `NewOrderSchema.parse`
  4. Calls `createOrder(body, accessToken)`
  5. `createOrder` issues `POST /v3/customers/{customerId}/orders` with Bearer + x-api-key
  6. Response parsed and validated with `OrderSchema.parse`
  7. Returns `{ data: Order, requestId }`

---

### SubscriptionManagement

- **Handler:** `pages/api/subscriptions.ts · handler`
- **Scope:** GET (getSubscriptionDetails, getCustomerSubscriptions), POST (createSubscription), PATCH (updateSubscription).
- **Services:** `subscriptionController.getSubscriptionDetails`, `subscriptionController.getCustomerSubscriptions`, `subscriptionController.createSubscription`, `subscriptionController.updateSubscription`
- **Connector(s):** [AdobePartnerAPI](#2-adobepartnerapi-vipmp); [AdobeIMS](#1-adobeims-oauth2-token-endpoint) (token)
- **Domain:** `Subscription`, `CreateSubscriptionRequestSchema`, `UpdateSubscriptionRequestSchema`
- **Validation:** Zod — `CreateSubscriptionRequestSchema` validated in controller before POST; `UpdateSubscriptionRequestSchema` validated before PATCH; `SubscriptionSchema` validated on getSubscriptionDetails and getCustomerSubscriptions responses (per-item on list)
- **Events:** none
- **Key Decisions:**
  - **System-of-record:** Adobe Partner API. **Enforcement:** all mutations proxied directly.
  - **Idempotency:** Not enforced for POST. **Enforcement:** not enforced in code.
  - **`reset-flex-discount-codes`:** PATCH accepts `?reset-flex-discount-codes=true` query param which is forwarded to Partner API. **Enforcement:** `subscriptionController.ts:133-135`.
  - **Known bug:** `getSubscriptionDetails` URL is missing `/v3` (`subscriptionController.ts:199`). **Enforcement:** not enforced — relies on manual fix.
  - **Fail posture:** Fail-closed. **Enforcement:** no fallback path.
  - **Stability:** STABLE

---

### PricelistFetch

- **Handler:** `pages/api/pricelist.ts · handler`
- **Scope:** POST only — fetch paginated product price list for a market segment + region + currency.
- **Services:** `priceController.getPriceList`
- **Connector(s):** [AdobePartnerAPI](#2-adobepartnerapi-vipmp); [AdobeIMS](#1-adobeims-oauth2-token-endpoint) (token)
- **Domain:** `PriceListRequest`, `PriceListResponse`, `PriceListOffer`, `CatalogProduct`
- **Validation:** Zod — `PriceListRequestSchema.parse(body)` before fetch; explicit guard for empty/whitespace `currency` after Zod (catches whitespace that passes `.min(1)`); `PriceListResponseSchema.parse(response)` on response (passthrough)
- **Events:** none
- **Key Decisions:**
  - **System-of-record:** Adobe Partner API. **Enforcement:** pass-through.
  - **Currency guard:** An explicit `parsedRequest.currency.trim() === ''` check rejects whitespace-only currency **before** the upstream call to avoid billing errors. **Enforcement:** `priceController.ts:39-42`.
  - **`hasMore` computation:** Computed locally as `data.items.length === limit` before returning. **Enforcement:** `priceController.ts:102`.
  - **Fail posture:** Fail-closed. **Enforcement:** no fallback.
  - **Stability:** STABLE

---

### RecommendationFetch

- **Handler:** `pages/api/recommendation.ts · handler`
- **Scope:** GET only — fetch personalised product recommendations for a customer.
- **Services:** `recommendationController.getRecommendations`
- **Connector(s):** [AdobePartnerAPI](#2-adobepartnerapi-vipmp); [AdobeIMS](#1-adobeims-oauth2-token-endpoint) (token)
- **Domain:** `RecommendationRequest`, `RecommendationResponse`, `RecommendationItem`, `RecommendationProduct`
- **Validation:** Zod — `RecommendationRequestSchema.parse` on input (coerces `customerId` to number); `RecommendationResponseSchema.parse` on response — **soft validation only**: if schema fails, raw data is returned without throwing
- **Events:** none
- **Key Decisions:**
  - **System-of-record:** Adobe Partner API. **Enforcement:** pass-through.
  - **Soft response validation:** Response schema failure does not fail the request — raw data returned. **Enforcement:** `recommendationController.ts:111-119` (catch block returns raw `data`).
  - **Fail posture:** Fail-closed for IMS / fetch errors; soft-open for schema validation. **Enforcement:** as above.
  - **Stability:** STABLE

---

### PartnerDetails

- **Handler:** `pages/api/partnerDetails.ts · handler`
- **Scope:** GET only — returns partner contract (name, market segments, currencies) derived entirely from environment variables. No IMS token required.
- **Services:** `partnerDetailsController.getPartnerDetails`, `getPartnerDetailsMarketSegments`, `getPartnerDetailsCurrencies`
- **Connector(s):** none — reads env vars only
- **Domain:** `PartnerDetails`, `Currency`, `MarketSegmentSchema`
- **Validation:** Zod — `PartnerDetailsSchema.parse(partnerData)` validates the assembled env-var data; `parseEnvArray` validates that `MARKET_SEGMENTS` and `CURRENCIES` are valid JSON arrays
- **Events:** none
- **Key Decisions:**
  - **System-of-record:** Environment variables (`PARTNER_NAME`, `MARKET_SEGMENTS`, `CURRENCIES`, `REGION`). No external API call. **Enforcement:** `partnerDetailsController.ts:19-72`.
  - **Fail posture:** Fail-closed. Missing or malformed env vars throw `ApiError(500)`. **Enforcement:** `parseEnvArray` at lines 8-16.
  - **Stability:** STABLE

---

### UnifiedSearch

- **Handler:** `pages/api/search.ts · handler`
- **Scope:** GET only — search resellers or customers by company name (wildcard) with optional numeric ID fallback. Combines and deduplicates results.
- **Services:** `resellerController.findResellerByName`, `resellerController.getResellerDetails`, `customerController.findCustomerByName`, `customerController.getCustomer`
- **Connector(s):** [AdobePartnerAPI](#2-adobepartnerapi-vipmp); [AdobeIMS](#1-adobeims-oauth2-token-endpoint) (token)
- **Domain:** `ResellerMinimal`, `CustomerMinimal`, `PaginatedCustomersResponse`, `PaginatedResellersResponse`
- **Validation:** Zod — delegated to controller functions; inline query param validation in handler (`isNaN`, range checks)
- **Events:** none
- **Key Decisions:**
  - **Parallel search:** Name search and numeric-ID lookup run with `Promise.allSettled` — a failure in one path does not fail the other. **Enforcement:** `search.ts:32`, `search.ts:89`.
  - **Dedup by ID:** Results deduplicated by `resellerId`/`customerId` using a `Set` before merging. **Enforcement:** `search.ts:43-51`, `search.ts:105-113`.
  - **Fail posture:** Fail-open for individual sub-queries (Promise.allSettled); fail-closed if IMS acquisition fails. **Enforcement:** `Promise.allSettled` in `searchResellers` and `searchCustomers`.
  - **Stability:** STABLE
- **Call graph (searchResellers):**
  1. `handler` calls `handlePrerequisites` → IMS token
  2. `handler` calls `searchResellers(query, accessToken, offsetNum, limitNum)`
  3. `searchResellers` starts `findResellerByName('*query*', ...)` in parallel
  4. If `query` is 3+ digits, also starts `getResellerDetails(query, ...)` in parallel
  5. `Promise.allSettled([nameSearchPromise, ...numericPromise])` waits for both
  6. Results merged and deduplicated by `resellerId`
  7. Returns combined paginated response

---

## Cross-Reference: Connector → Capabilities

| Connector | Capabilities that use it |
|-----------|--------------------------|
| AdobeIMS | CustomerManagement, ResellerManagement, OrderManagement, SubscriptionManagement, PricelistFetch, RecommendationFetch, UnifiedSearch |
| AdobePartnerAPI | CustomerManagement, ResellerManagement, OrderManagement, SubscriptionManagement, PricelistFetch, RecommendationFetch, UnifiedSearch |

PartnerDetails uses neither connector — env vars only.

---

## Type Inventory

| Type name | Kind | Used by capability |
|-----------|------|--------------------|
| `CustomerDetails` | Response | CustomerManagement |
| `CustomerMinimal` | Response | CustomerManagement, UnifiedSearch |
| `CreateCustomer` | Request | CustomerManagement |
| `UpdateCustomer` | Request | CustomerManagement |
| `PaginatedCustomersResponse` | Response | CustomerManagement, UnifiedSearch |
| `Reseller` | Request / Response | ResellerManagement |
| `ResellerMinimal` | Response | ResellerManagement, UnifiedSearch |
| `ResellerDetails` | Response | ResellerManagement, UnifiedSearch |
| `Order` | Response | OrderManagement |
| `LineItem` | Domain | OrderManagement |
| `Pricing` | Domain | OrderManagement |
| `OrdersHistoryResponseWithPagination` | Response | OrderManagement |
| `Subscription` | Response | SubscriptionManagement |
| `PartnerDetails` | Response | PartnerDetails |
| `Currency` | Domain | PartnerDetails |
| `PriceListRequest` | Request | PricelistFetch |
| `PriceListResponse` | Response | PricelistFetch |
| `PriceListOffer` | Domain | PricelistFetch |
| `CatalogProduct` | Domain | PricelistFetch (UI layer) |
| `CatalogFilters` | Domain | PricelistFetch (UI layer) |
| `RecommendationRequest` | Request | RecommendationFetch |
| `RecommendationResponse` | Response | RecommendationFetch |
| `RecommendationItem` | Domain | RecommendationFetch, OrderManagement |
| `RecommendationProduct` | Domain | RecommendationFetch |
| `BackendResult<T>` | Wrapper | All capabilities |
