# Outbound Dependencies — bridge

> Bridge has **two** outbound connectors: Adobe IMS (token acquisition) and
> Adobe Partner API (all domain operations). Both use the Node.js built-in
> `fetch`. **Neither has timeout or retry configuration** — deviations from
> `CONNECTOR_STANDARDS.md` are documented below.

---

## 1. AdobeIMS (OAuth2 token endpoint)

| Field | Value |
|-------|-------|
| **Purpose** | Obtain an IMS Bearer token via client_credentials grant; token is used on every subsequent Partner API call |
| **Library** | Native `fetch` (Node.js built-in) |
| **Base URL** | `process.env.IMS_TOKEN_URL` — default `https://ims-na1-stg1.adobelogin.com/ims/token/v2` |
| **Auth mechanism** | Client credentials: `client_id=PARTNER_CLIENT_ID`, `client_secret=PARTNER_CLIENT_SECRET` sent as URLSearchParams in POST body |
| **Connect timeout** | none configured |
| **Read timeout** | none configured |
| **Retry config** | none configured |
| **Token cache** | In-process module-level variables (`cachedToken`, `tokenExpiry`). TTL = `expires_in - 300_000ms` (5 min buffer). Single node only — does not share cache across replicas. |
| **Error handling** | Non-2xx throws `ApiError(message, status)` with response body text. No retry. |
| **Unavailability posture** | Fail-fast. Every inbound request that calls `handlePrerequisites` will fail with 500 if IMS is unreachable. |
| **Config prefix** | `IMS_TOKEN_URL`, `PARTNER_CLIENT_ID`, `PARTNER_CLIENT_SECRET`, `IMS_SCOPES` |
| **Implementation** | `utils/imsTokenService.ts · getAccessToken()` |
| **Called from** | `utils/commonUtils.ts · handlePrerequisites()` — invoked by every API route handler except `/api/partnerDetails` and `/api/env` |

**Standards deviation:**
- No timeout (CONNECTOR_STANDARDS §1 requires 2s connect / 5s read)
- No retry (CONNECTOR_STANDARDS §3.1 requires exponential backoff)
- No circuit breaker (CONNECTOR_STANDARDS §3.2)

---

## 2. AdobePartnerAPI (VIPMP)

| Field | Value |
|-------|-------|
| **Purpose** | All domain operations: customer, reseller, order, subscription, pricelist, recommendation management |
| **Library** | Native `fetch` (Node.js built-in) |
| **Base URL** | `process.env.PARTNER_API_BASE_URL` |
| **Auth mechanism** | Bearer token: `Authorization: Bearer <imsToken>` + `x-api-key: <PARTNER_CLIENT_ID>` on every request |
| **Connect timeout** | none configured |
| **Read timeout** | none configured (browser-side `AccountApis.ts` uses a 30s `AbortController`, but server-side controllers do not) |
| **Retry config** | none configured |
| **Correlation headers** | `X-Correlation-Id: uuidv4()` and `X-Request-Id: uuidv4()` generated fresh per call; `X-Request-Id` from the response is extracted and forwarded back to the caller |
| **Error handling** | Non-2xx: throws `ApiError(message, status, requestId)`; non-JSON: throws `ApiError('Non-JSON response', status, requestId)`; Zod validation failure on response: throws `ApiError('...data validation failed', 500, requestId)`. Error body parsed and forwarded for order endpoints. |
| **Unavailability posture** | Fail-fast. No fallback, no cached data. HTTP error status and message propagated to browser. |
| **Config prefix** | `PARTNER_API_BASE_URL`, `PARTNER_CLIENT_ID` |
| **Implementation** | `controllers/customerController.ts`, `controllers/orderController.ts`, `controllers/resellerController.ts`, `controllers/subscriptionController.ts`, `controllers/priceController.ts`, `controllers/recommendationController.ts` |

**Standards deviation:**
- No timeout (CONNECTOR_STANDARDS §1)
- No retry (CONNECTOR_STANDARDS §3.1)
- No circuit breaker (CONNECTOR_STANDARDS §3.2)
- No connection pooling (CONNECTOR_STANDARDS §2) — each request creates a new TCP connection (native fetch default)

### Endpoint inventory

| Operation | Method | URL pattern | Controller fn |
|-----------|--------|-------------|--------------|
| Get customer | GET | `/v3/customers/{customerId}` | `customerController.getCustomer` |
| Create customer | POST | `/v3/customers` | `customerController.createCustomer` |
| Update customer | PATCH | `/v3/customers/{customerId}` | `customerController.updateCustomer` |
| List customers for reseller | GET | `/v3/resellers/{resellerId}/customers?offset=&limit=` | `customerController.getAllCustomers` |
| Search customers by name | GET | `/v3/resellers/{resellerId}/customers?company-name=&offset=&limit=` | `customerController.findCustomerByName` |
| Get order | GET | `/v3/customers/{customerId}/orders/{orderId}` | `orderController.getOrders` |
| List order history | GET | `/v3/customers/{customerId}/orders?offset=&limit=` | `orderController.getOrdersHistoryForCustomer` |
| Create order (all types) | POST | `/v3/customers/{customerId}/orders[?queryParams]` | `orderController.createOrder` |
| List resellers | GET | `/v3/resellers?offset=&limit=` | `resellerController.getAllResellers` |
| Get reseller | GET | `/v3/resellers/{id}` | `resellerController.getResellerDetails` |
| Create reseller | POST | `/v3/resellers` | `resellerController.createReseller` |
| Search resellers by name | GET | `/v3/resellers?company-name=&offset=&limit=` | `resellerController.findResellerByName` |
| Get subscription details | GET | `/customers/{customerId}/subscriptions/{subscriptionId}` ⚠️ **missing `/v3`** | `subscriptionController.getSubscriptionDetails` |
| List customer subscriptions | GET | `/v3/customers/{customerId}/subscriptions` | `subscriptionController.getCustomerSubscriptions` |
| Create subscription | POST | `/v3/customers/{customerId}/subscriptions` | `subscriptionController.createSubscription` |
| Update subscription | PATCH | `/v3/customers/{customerId}/subscriptions/{subscriptionId}[?reset-flex-discount-codes=true]` | `subscriptionController.updateSubscription` |
| Get pricelist | POST | `/v3/pricelist` | `priceController.getPriceList` |
| Get recommendations | POST | `/v3/recommendations` | `recommendationController.getRecommendations` |

> ⚠️ `getSubscriptionDetails` URL is missing the `/v3` path segment — see `subscriptionController.ts:199`. All other endpoints correctly use `/v3`. This may route to a different API version.
