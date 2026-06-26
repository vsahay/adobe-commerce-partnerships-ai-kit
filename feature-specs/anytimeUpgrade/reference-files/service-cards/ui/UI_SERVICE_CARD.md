---
service: Bridge
owner: ⚠ NOT CAPTURED — no author field in package.json
status: active
repo: ⚠ NOT CAPTURED — no repository field in package.json
tier: 1
stack: Next.js 15 · React 18 · TypeScript 5.4 · TanStack Query v5 · React Spectrum v2
rendering: CSR
date: 2026-06-15
---

# UI_SERVICE_CARD — Bridge

## §1 What We Do

Bridge is a Next.js 15 ordering interface for Adobe direct partners, built on top of the VIP Marketplace (VIP MP) APIs. It is used by partner organizations (such as Ingram US) to browse the Adobe product catalog, manage their downstream resellers and end customers, place new product orders, handle subscription renewals, and manage Three-Year Commit (3YC) enrollments. The application renders entirely client-side (CSR) — all data is fetched after mount via TanStack Query v5. There is no server-side rendering of page content; this keeps the deployment model simple (Next.js Pages Router, custom Node server on port 9000).

Bridge communicates exclusively with two backends: the VIP Marketplace Partner API (`PARTNER_API_BASE_URL`) and the Adobe IMS token endpoint (`IMS_TOKEN_URL`). Both are accessed server-side only through Next.js API routes — the browser never holds credentials or calls external APIs directly. Authentication uses OAuth 2.0 client credentials flow, handled entirely in `utils/imsTokenService.ts` within the server layer.

---

## §2 What We Explicitly Do Not Do

- **Business rule validation** — the VIP MP API owns pricing, discount level calculation, and order eligibility. Bridge displays and proxies; it does not re-implement or override these rules.
- **Direct external API calls from the browser** — every VIP MP API call is proxied through a Next.js API route. The browser only ever calls `/api/*` endpoints on the same origin.
- **Auth UI / login flow** — Bridge has no login page or session management. Auth is entirely server-side (client credentials OAuth); there is no concept of a logged-in user at the UI layer.
- **Data persistence beyond a session's cart** — the only browser storage used is `localStorage` for the cart (`bridge_cart`). No user data, session tokens, or API responses are stored in the browser.
- **Email / notification delivery** — order confirmation tells the partner to notify the reseller and customer manually; Bridge itself sends no emails.

---

## §3 Data Contracts

### Fetch Layer

| Domain | Fetch unit | Endpoint | Cache key |
|---|---|---|---|
| Partner contract | `fetchPartnerDetails` | GET `/api/partnerDetails` | `['partnerDetails']` |
| Customers — list | `fetchCustomersPage` | GET `/api/customers?type=getAllCustomers&...` | `['customers','list', resellerId, page, limit]` |
| Customers — search | `searchCustomers` | GET `/api/search?type=customer&...` | `['customers','search', resellerId, term, page, limit]` |
| Customer detail | `fetchCustomerDetails` | GET `/api/customers?type=getCustomer&customerId=...` | `['customerDetails','customer', customerId]` |
| Resellers — list | `fetchResellersPage` | GET `/api/resellers?type=getAllResellers&...` | `['resellers','list', page, limit]` |
| Resellers — search | `searchResellers` | GET `/api/search?type=reseller&...` | `['resellers','search', term, page, limit]` |
| Reseller detail | `getResellerDetails` | GET `/api/resellers?type=getResellerDetails&id=...` | `['reseller', resellerId]` |
| Pricelist | `fetchPricelistPage` | POST `/api/pricelist` | `['pricelist-infinite', segmentCode, currency]` |
| Subscription pricing | `fetchProductPricing` | POST `/api/pricelist` | `['subscriptionPricing', offerId, region, currency, type]` |
| Subscriptions | `fetchCustomerSubscriptions` | GET `/api/subscriptions?...` | `['customerDetails','subscriptions', customerId]` |
| Order preview | `fetchOrderPreview` | POST `/api/orders?type=Preview` | No cache (mutation) |
| Create order | `useCreateOrder` | POST `/api/orders?type=NEW` | No cache (mutation) |

### Browser Storage Registry

| Key | TTL | Contents |
|---|---|---|
| `bridge_cart` | Until `clearCart()` | `{ cartItemIdToQuantityMap, cartItemIdToCartItemDetailsMap, customerInfoInCart }` |

---

## §4 Module Index (Summary)

| Domain | Capabilities | Details |
|---|---|---|
| Navigation | 2 | [UI_MODULE_INDEX §4.1–4.2](UI_MODULE_INDEX.md#41-root-redirect) |
| Account Management | 2 | [UI_MODULE_INDEX §4.3–4.4](UI_MODULE_INDEX.md#43-customer-list) |
| Catalog | 2 | [UI_MODULE_INDEX §4.5–4.6](UI_MODULE_INDEX.md#45-product-catalog-browse) |
| Ordering | 2 | [UI_MODULE_INDEX §4.7–4.8](UI_MODULE_INDEX.md#47-checkout--order-review) |
| Partner Config | 1 | [UI_MODULE_INDEX §4.9](UI_MODULE_INDEX.md#49-partner-config-load) |

---

## Companion Files

| File | When to load | Purpose |
|---|---|---|
| [UI_MODULE_INDEX.md](UI_MODULE_INDEX.md) | When adding a capability, understanding what a page does, or tracing a call graph | Full capability catalogue — every feature mapped to its files |
| [ROUTES.md](ROUTES.md) | When adding a page, adding navigation, or understanding URL structure | Route registry, auth guard, navigation patterns |
| [DATA_LAYER.md](DATA_LAYER.md) | When adding a fetch call, modifying an API integration, or understanding cache keys | Fetch contracts, mutation pattern, auth injection |
| [STATE_MANAGEMENT.md](STATE_MANAGEMENT.md) | When reading/writing cart or partner state, or understanding cache TTLs | Store shapes, actions, init order, cache strategy |
| [UI_CODE_PATTERNS.md](UI_CODE_PATTERNS.md) | When writing any new component, hook, or utility | Naming conventions, directory structure, code patterns, design system usage |
| [UI_PLATFORM.md](UI_PLATFORM.md) | When setting up locally, checking dependencies, or adding env vars | Runtime stack, build scripts, env vars, monitoring |
