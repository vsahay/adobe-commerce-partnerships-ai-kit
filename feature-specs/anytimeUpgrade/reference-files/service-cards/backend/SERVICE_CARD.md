# Service Card — bridge

> **Purpose:** Quick-reference for engineers onboarding to this service and for
> agentic LLD generation. Summarises identity, ownership, external surface, and
> fragile areas.

---

## 1. Identity

| Field | Value |
|-------|-------|
| **Service name** | bridge |
| **Version** | 1.0.0 |
| **Type** | Next.js BFF (Backend for Frontend) |
| **Language** | TypeScript 5.4.2 |
| **Runtime** | Node.js 20 |
| **Framework** | Next.js 15.2.6 (Pages Router) |
| **Owner team** | NOT CAPTURED — no deployment spec |
| **Repo** | /Users/ankitasaha/Documents/opensource_demo/bridge |
| **Source branch** | midterm |

---

## 2. What This Service Owns

Bridge is a partner portal BFF for the Adobe VIP Marketplace Program (VIPMP). It serves two roles:

1. **Server-side API proxy:** Next.js API routes (`pages/api/`) acquire IMS OAuth2 tokens via client credentials, then forward browser requests to the Adobe Partner API. No business data is persisted here.
2. **React SPA:** Provides the browser UI for partner (reseller/distributor) operations — browsing the product catalog, managing customers and resellers, placing and reviewing orders, and managing subscription auto-renewal.

Bridge owns **no data store**. All authoritative data lives in the Adobe Partner API.

---

## 3. External Surface

### APIs We Expose

| Endpoint | Method | Purpose | Version | Stability |
|----------|--------|---------|---------|-----------|
| [`/api/customers`](#get-apicustomers--fetch-customer-or-customer-list) | GET | Fetch single customer (`type=getCustomer`) or paginated reseller customer list (`type=getAllCustomers`) | v1 | STABLE |
| [`/api/customers`](#post-apicustomers--create-customer) | POST | Create customer (`type=createCustomer`) | v1 | STABLE |
| [`/api/customers`](#patch-apicustomers--update-customer) | PATCH | Update customer 3YC enrollment / company profile (`type=updateCustomer`) | v1 | STABLE |
| [`/api/orders`](#get-apiorders--fetch-order-or-order-history) | GET | Fetch single order (`type=getOrders`) or paginated history (`type=getOrdersHistory`) | v1 | STABLE |
| [`/api/orders`](#post-apiorders--create-order) | POST | Create NEW / RETURN / PREVIEW / PREVIEW_RENEWAL / RENEWAL order | v1 | STABLE |
| [`/api/partnerDetails`](#get-apipartnerdetails--fetch-partner-details) | GET | Fetch partner contract, market segments, or currencies from env vars | v1 | STABLE |
| [`/api/pricelist`](#post-apipricelist--fetch-product-price-list) | POST | Fetch product price list for a region + currency + market segment | v1 | STABLE |
| [`/api/recommendation`](#get-apirecommendation--fetch-recommendations) | GET | Fetch personalised product recommendations for a customer | v1 | STABLE |
| [`/api/resellers`](#get-apiresellers--fetch-reseller-or-list) | GET | Fetch reseller details (`type=getResellerDetails`) or paginated list (`type=getAllResellers`) | v1 | STABLE |
| [`/api/resellers`](#post-apiresellers--create-reseller) | POST | Create reseller | v1 | STABLE |
| [`/api/search`](#get-apisearch--search-resellers-or-customers) | GET | Search resellers or customers by name and optional numeric ID | v1 | STABLE |
| [`/api/subscriptions`](#get-apisubscriptions--fetch-subscriptions) | GET | Fetch single subscription (`type=getSubscriptionDetails`) or all for a customer (`type=getCustomerSubscriptions`) | v1 | STABLE |
| [`/api/subscriptions`](#post-apisubscriptions--create-subscription) | POST | Create subscription | v1 | STABLE |
| [`/api/subscriptions`](#patch-apisubscriptions--update-subscription) | PATCH | Update subscription auto-renewal settings | v1 | STABLE |
| [`/api/env`](#get-apienv--serve-runtime-env-vars) | GET | Serve cached public runtime env vars | v1 | STABLE |

### APIs We Consume (Upstream)

| Service | Connector Interface | Connector Impl | Auth Type | Purpose |
|---------|--------------------|--------------------|-----------|---------|
| [Adobe IMS](#1-adobeims-oauth2-token-endpoint) | n/a (no interface) | `utils/imsTokenService.ts · getAccessToken` | Client credentials (client_id + client_secret) | Obtain Bearer token for all Partner API calls |
| [Adobe Partner API](#2-adobepartnerapi-vipmp) | n/a (no interface) | `controllers/*Controller.ts` — one module per domain | Bearer (IMS token) + x-api-key (PARTNER_CLIENT_ID) | All customer, reseller, order, subscription, pricelist, and recommendation operations |

---

## 4. Source Structure

| Layer | Path | File count |
|-------|------|-----------|
| API route handlers | `pages/api/` | 9 |
| Controllers (outbound calls) | `controllers/` | 7 |
| Zod models + TypeScript types | `models/` | 11 |
| React pages | `pages/` | 7 |
| React components | `components/` | ~30 |
| React hooks | `hooks/` | 7 |
| React contexts | `contexts/` | 2 |
| Shared utilities | `utils/` | 18 |
| TypeScript type definitions | `types/` | 7 |
| Tests | `tests/` | 16 |
| Static lib data | `lib/` | 1 |

---