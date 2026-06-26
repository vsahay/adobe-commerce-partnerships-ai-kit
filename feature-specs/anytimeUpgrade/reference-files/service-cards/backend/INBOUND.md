# Inbound Interface — bridge

> All routes are Next.js API routes under `pages/api/`. Authentication is
> handled server-side: every handler calls `handlePrerequisites(req)` which
> calls `getAccessToken()` (IMS client credentials flow) to obtain a Bearer
> token. The token is **never** read from the incoming request; the caller
> only needs to be able to reach the Next.js server.
>
> Exception: `/api/partnerDetails` and `/api/env` do not require IMS token.

---

## HTTP Routes

### GET /api/customers — Fetch customer or customer list

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/customers.ts` |
| **Controller** | `controllers/customerController.ts` |
| **Auth** | Server-side IMS Bearer (caller unauthenticated) |

**Sub-operations (via `type` query param):**

| `type` | Required params | Response shape | Controller fn |
|--------|----------------|----------------|---------------|
| `getCustomer` | `customerId` | `CustomerDetails` | `getCustomer` |
| `getAllCustomers` | `resellerId`, optional `offset` (default 0), `limit` (default 50) | `PaginatedCustomersResponse` | `getAllCustomers` |

**Error codes:**
- 400 — missing or invalid required query params
- 500 — missing env vars (`PARTNER_CLIENT_ID`, `PARTNER_API_BASE_URL`), IMS failure, or upstream non-JSON response
- Upstream 4xx/5xx — forwarded as-is with `X-Request-Id` header

---

### POST /api/customers — Create customer

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/customers.ts` |
| **Controller** | `controllers/customerController.ts · createCustomer` |
| **Auth** | Server-side IMS Bearer |
| **Query** | `type=createCustomer` |

**Request body — `CreateCustomer`:**
```json
{
  "resellerId": "string",
  "externalReferenceId": "string (optional)",
  "companyProfile": {
    "companyName": "string",
    "preferredLanguage": "string",
    "marketSegment": "string (optional)",
    "marketSubSegments": ["string"],
    "address": {
      "country": "string", "region": "string", "city": "string",
      "addressLine1": "string", "addressLine2": "string",
      "postalCode": "string", "phoneNumber": "string (optional)"
    },
    "contacts": [{ "firstName": "string", "lastName": "string", "email": "string" }]
  }
}
```

**Response (201):** `CustomerDetails` shape

---

### PATCH /api/customers — Update customer

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/customers.ts` |
| **Controller** | `controllers/customerController.ts · updateCustomer` |
| **Auth** | Server-side IMS Bearer |
| **Query** | `type=updateCustomer&customerId=<id>` |

**Request body — `UpdateCustomer`:**
```json
{
  "benefits": [
    { "type": "THREE_YEAR_COMMIT", "commitmentRequest": { "minimumQuantities": [{ "offerType": "LICENSE|CONSUMABLES", "quantity": 1 }] } }
  ],
  "companyProfile": { "...same as CreateCustomer..." },
  "globalSalesEnabled": false
}
```

**Response (200):** `CustomerDetails` shape

---

### GET /api/orders — Fetch order or order history

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/orders.ts` |
| **Controller** | `controllers/orderController.ts` |
| **Auth** | Server-side IMS Bearer |

**Sub-operations:**

| `type` | Required params | Response shape | Controller fn |
|--------|----------------|----------------|---------------|
| `getOrders` | `customerId`, `orderId` | `Order` (passthrough, no strict validation) | `getOrders` |
| `getOrdersHistory` | `customerId`, optional `offset` (default 0), `limit` (default 25) | `OrdersHistoryResponseWithPagination` | `getOrdersHistoryForCustomer` |

---

### POST /api/orders — Create order

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/orders.ts` |
| **Controller** | `controllers/orderController.ts` |
| **Auth** | Server-side IMS Bearer |

**Sub-operations (via `type` query param):**

| `type` | Body fields | Validates with | Controller fn |
|--------|------------|----------------|---------------|
| `NEW` | `customerId`, `externalReferenceId`, `currencyCode`, `lineItems` | `NewOrderSchema` | `createNewOrder` |
| `Return` | `customerId`, `referenceOrderId`, `externalReferenceId`, `currencyCode`, `lineItems` | `ReturnOrderSchema` | `createReturnOrder` |
| `Preview` | `customerId`, `externalReferenceId`, `currencyCode`, `lineItems` | `PreviewOrderSchema` | `createPreviewOrder` |
| `PreviewRenewal` | `customerId`, `currencyCode`, optional `lineItems` | `PreviewRenewalOrderSchema` | `createPreviewRenewalOrder` |
| `RenewalOrder` | `customerId`, `externalReferenceId`, `currencyCode`, `lineItems` | `RenewalOrderSchema` | `createRenewalOrder` |

**Response (201):** `Order` shape (Zod passthrough — extra fields preserved)

**Error handling:** Backend error body is JSON-parsed and forwarded as-is (not wrapped in `{ error: ... }`).

---

### GET /api/partnerDetails — Fetch partner details

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/partnerDetails.ts` |
| **Controller** | `controllers/partnerDetailsController.ts` |
| **Auth** | None — no IMS token required |

**Sub-operations (via `type` query param):**

| `type` | Source | Response shape |
|--------|--------|----------------|
| (default) | Env vars: `PARTNER_NAME`, `MARKET_SEGMENTS`, `CURRENCIES`, `REGION` | `PartnerDetails` |
| `marketSegments` | Derived from `MARKET_SEGMENTS` env var | `string[]` |
| `currencies` | Derived from `CURRENCIES` + `REGION` env vars | `Currency[]` |

---

### POST /api/pricelist — Fetch product price list

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/pricelist.ts` |
| **Controller** | `controllers/priceController.ts · getPriceList` |
| **Auth** | Server-side IMS Bearer |

**Request body — `PriceListRequest`:**
```json
{
  "region": "string",
  "marketSegment": "string",
  "priceListType": "string",
  "priceListMonth": "string (YYYYMM)",
  "currency": "string (required, non-empty)",
  "offset": 0,
  "limit": 100,
  "filters": { "offerId": "string (optional)" },
  "includeOfferAttributes": ["string"]
}
```

**Response (200):** `PriceListResponse` with `hasMore` computed field

**Guard:** Empty or whitespace-only `currency` throws 400 before upstream call.

---

### GET /api/recommendation — Fetch recommendations

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/recommendation.ts` |
| **Controller** | `controllers/recommendationController.ts · getRecommendations` |
| **Auth** | Server-side IMS Bearer |

**Query params:**
- `customerId` (required, coerced to number upstream)
- `recommendationContext` (optional)
- `country` (optional — passed as query context, not forwarded to Adobe API)
- `language` (optional — passed as query context, not forwarded to Adobe API)

**Response (200):** `RecommendationResponse` shape; falls back to raw data if schema validation fails (non-throwing fallback).

---

### GET /api/resellers — Fetch reseller or list

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/resellers.ts` |
| **Controller** | `controllers/resellerController.ts` |
| **Auth** | Server-side IMS Bearer |

**Sub-operations:**

| `type` | Required params | Response shape | Controller fn |
|--------|----------------|----------------|---------------|
| `getResellerDetails` | `id` | `ResellerDetails` | `getResellerDetails` |
| `getAllResellers` | optional `offset` (default 0), `limit` (default 50) | paginated resellers | `getAllResellers` |

**Also supports:** `OPTIONS` preflight (returns 200).

---

### POST /api/resellers — Create reseller

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/resellers.ts` |
| **Controller** | `controllers/resellerController.ts · createReseller` |
| **Auth** | Server-side IMS Bearer |
| **Query** | `type=createReseller` (or omitted for backward compat) |

**Request body — `Reseller`:**
```json
{
  "externalReferenceId": "string",
  "resellerId": "string",
  "distributorId": "string",
  "status": "string",
  "companyProfile": {
    "companyName": "string", "preferredLanguage": "string",
    "marketSegments": ["string"],
    "address": { "country": "string", "region": "string", "city": "string", "addressLine1": "string", "addressLine2": "string", "postalCode": "string" },
    "contacts": [{ "firstName": "string", "lastName": "string", "email": "string" }]
  },
  "creationDate": "string"
}
```

**Response (201):** Raw API response (not validated against a schema).

---

### GET /api/search — Search resellers or customers

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/search.ts` |
| **Auth** | Server-side IMS Bearer |

**Query params:**
- `type` (required): `reseller` or `customer`
- `query` (required): search string
- `resellerId` (required when `type=customer`)
- `offset` (default 0), `limit` (default 50)

**Behaviour:** Runs a wildcard name search (`*query*`) in parallel with an exact numeric ID lookup (if `query` is 3+ digits). Results are deduplicated by `resellerId`/`customerId`.

**Response (200):**
```json
{ "resellers|customers": [...], "totalCount": 0, "count": 0, "offset": 0, "limit": 50, "hasMore": false }
```

---

### GET /api/subscriptions — Fetch subscriptions

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/subscriptions.ts` |
| **Controller** | `controllers/subscriptionController.ts` |
| **Auth** | Server-side IMS Bearer |

**Sub-operations:**

| `type` | Required params | Response shape | Controller fn |
|--------|----------------|----------------|---------------|
| `getSubscriptionDetails` | `customerId`, `subscriptionId` | `Subscription` | `getSubscriptionDetails` |
| `getCustomerSubscriptions` | `customerId` | `Subscription[]` | `getCustomerSubscriptions` |

**Known issue:** `getSubscriptionDetails` URL missing `/v3` — see SERVICE_CARD Fragile Areas.

---

### POST /api/subscriptions — Create subscription

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/subscriptions.ts` |
| **Controller** | `controllers/subscriptionController.ts · createSubscription` |
| **Auth** | Server-side IMS Bearer |

**Request body:**
```json
{
  "customerId": "string",
  "offerId": "string",
  "autoRenewal": { "enabled": true, "renewalQuantity": 1, "flexDiscountCodes": [] }
}
```

**Response (201):** Raw API response (not validated against schema).

---

### PATCH /api/subscriptions — Update subscription

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/subscriptions.ts` |
| **Controller** | `controllers/subscriptionController.ts · updateSubscription` |
| **Auth** | Server-side IMS Bearer |
| **Query** | optional `reset-flex-discount-codes=true` |

**Request body:**
```json
{
  "customerId": "string",
  "subscriptionId": "string",
  "autoRenewal": { "enabled": boolean, "renewalQuantity": number, "flexDiscountCodes": [] }
}
```

**Response (200):** Raw API response.

---

### GET /api/env — Serve runtime env vars

| Field | Value |
|-------|-------|
| **Handler** | `pages/api/env.ts` |
| **Auth** | None |
| **Cache** | In-process memory, 60s TTL; `Cache-Control: public, max-age=60` |

**Response (200):** JSON object of public env var key-value pairs (currently empty by default — populated by deployment if needed).