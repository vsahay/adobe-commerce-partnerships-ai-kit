# Backend LLD — Anytime Upgrade

**Feature:** `anytimeUpgrade`
**Source:** `feature-specs/anytimeUpgrade/experience-card/anytime-upgrade.md` + `feature-specs/anytimeUpgrade/APISpec/anytime-upgrade-apispec.md`
**Target repo:** `/Users/ankitasaha/Documents/openSource/bridge`

---

## Section 1: Summary

The Anytime Upgrade feature lets a partner upgrade a customer's active subscription mid-term to a higher-tier product. It introduces three new external API operations: a `GET /v3/offer-switch-paths` query to discover which subscriptions are eligible for upgrade, a `POST /v3/customers/{id}/orders` with `orderType: PREVIEW_SWITCH` for a non-destructive price estimate, and the same POST endpoint with `orderType: SWITCH` to place the order. The end-to-end data flow is: discover eligible subscriptions at product-listing render time → user picks a target offer → order summary page previews the prorated price → user confirms → switch order is placed and the existing subscription is cancelled and replaced.

### Unit Inventory

| Unit | Action | Location |
|------|--------|----------|
| `models/SwitchOrder.ts` | CREATE | `models/SwitchOrder.ts` |
| `controllers/switchOrderController.ts` | CREATE | `controllers/switchOrderController.ts` |
| `pages/api/offer-switch-paths.ts` | CREATE | `pages/api/offer-switch-paths.ts` |
| `models/Order.ts` | MODIFY | `models/Order.ts` |
| `pages/api/orders.ts` | MODIFY | `pages/api/orders.ts` |
| `utils/constants.ts` | MODIFY | `utils/constants.ts` |
| `utils/cartUtils.ts` | MODIFY | `utils/cartUtils.ts` |

---

## Section 2: Data Flow Diagram

```
Operation: Product listing renders (per subscription)
  → GET /api/offer-switch-paths?subscription-id=<id>&customer-id=<id>
  → getOfferSwitchPaths (switchOrderController)
  → GET /v3/offer-switch-paths?subscription-id=...&customer-id=... (Adobe Partner API)
  ← response: { productUpgrades: [{ sourceBaseOfferId, targetList[] }] }
  ← handler returns: OfferSwitchPathsResponse (Zod-validated)
  ← route returns: 200 { productUpgrades[] }

Operation: User navigates to order summary (page load)
  → POST /api/orders?type=PREVIEW_SWITCH&fetch-price=true
  → createPreviewSwitchOrder (switchOrderController)
  → POST /v3/customers/{customerId}/orders?fetch-price=true (Adobe Partner API)
    body: { orderType: "PREVIEW_SWITCH", currencyCode, lineItems, cancellingItems }
  ← response: { pricingSummary[].totalLineItemPartnerPrice, lineItems[].pricing }
  ← handler returns: SwitchOrderResponse (Zod-validated)
  ← route returns: 201 { pricingSummary, lineItems, cancellingItems }

Operation: User clicks Place order
  → POST /api/orders?type=SWITCH
  → createSwitchOrder (switchOrderController)
  → POST /v3/customers/{customerId}/orders (Adobe Partner API)
    body: { orderType: "SWITCH", currencyCode, lineItems, cancellingItems }
  ← response: { orderId, status, lineItems, cancellingItems }
  ← handler returns: SwitchOrderResponse (Zod-validated)
  ← route returns: 201 { orderId, status, ... }
```

---

## Section 3: Backend Changes

### New Files

#### `models/SwitchOrder.ts` — CREATE

**Why:** Zod schemas and TypeScript types are required for all three new API operations; not covered by the existing `models/Order.ts`.

**Schemas defined:**

| Schema | Purpose | Key fields |
|--------|---------|------------|
| `CancellingItemSchema` | Internal shape for the subscription being cancelled in a switch | `extLineItemNumber: number`, `referenceLineItemNumber: number`, `subscriptionId: string`, `quantity: number`, `status?: string`, `proratedDays?: number`, `pricing?: { partnerPrice, discountedPartnerPrice, netPartnerPrice, lineItemPartnerPrice }` |
| `PreviewSwitchOrderSchema` | Validates `PREVIEW_SWITCH` request body | `orderType: literal('PREVIEW_SWITCH')`, `currencyCode: string`, `lineItems: LineItemSchema[]`, `cancellingItems?: CancellingItemSchema[]` |
| `SwitchOrderSchema` | Validates `SWITCH` request body | Same shape as preview; `orderType: literal('SWITCH')` |
| `SwitchOrderResponseSchema` | Validates Adobe Partner API response for both operations | `orderId?: string`, `customerId: string`, `orderType: string`, `status: string`, `lineItems`, `cancellingItems?`, `pricingSummary?`, `creationDate?` |
| `OfferSwitchPathsResponseSchema` | Validates GET eligibility response | `totalCount`, `count`, `offset`, `limit`, `productUpgrades: [{ sourceBaseOfferId, targetList: [{ targetBaseOfferId, sequence, switchType }] }]` |

**Exported types:** `SwitchOrderResponse`, `OfferSwitchPathsResponse`

**Dependencies on `models/Order.ts`:**

| Import | Why |
|--------|-----|
| `LineItemSchema` | Reused to avoid duplication — requires `export` keyword added (see `models/Order.ts` MODIFY) |
| `PricingSummarySchema` | Same |
| `LinkSchema` | Same |

---

#### `controllers/switchOrderController.ts` — CREATE

**Why:** All external Adobe Partner API calls must go through a controller that handles auth headers, correlation IDs, JSON parsing, Zod validation, and structured error propagation — matching the pattern in `orderController.ts`.

**Functions:**

| Function | Signature | Role |
|----------|-----------|------|
| `postSwitchOrder` | `(body, accessToken, queryParams?)` | Internal helper; builds and fires `POST /v3/customers/{customerId}/orders`; validates response with `SwitchOrderResponseSchema` |
| `createPreviewSwitchOrder` | `(customerId, currencyCode, lineItems, cancellingItems?, accessToken?, queryParams?, externalReferenceId?)` | Validates body with `PreviewSwitchOrderSchema`; delegates to `postSwitchOrder` |
| `createSwitchOrder` | `(customerId, currencyCode, lineItems, cancellingItems?, accessToken?, queryParams?, externalReferenceId?)` | Validates body with `SwitchOrderSchema`; delegates to `postSwitchOrder` |
| `getOfferSwitchPaths` | `(accessToken, queryParams?)` | Calls `GET /v3/offer-switch-paths`; maps camelCase params to kebab-case; validates response with `OfferSwitchPathsResponseSchema` |

**Query param key mapping (for `getOfferSwitchPaths`):**

| Camel | Kebab (sent to API) |
|-------|---------------------|
| `marketSegment` | `market-segment` |
| `subscriptionId` | `subscription-id` |
| `customerId` | `customer-id` |

**Common behaviour:**

| Concern | Detail |
|---------|--------|
| Request headers | `Authorization: Bearer <token>`, `X-Correlation-Id`, `X-Request-Id`, `x-api-key: PARTNER_CLIENT_ID` |
| Env vars | `PARTNER_API_BASE_URL`, `PARTNER_CLIENT_ID` (same as `orderController.ts`) |
| Error — non-OK upstream | `ApiError` thrown with upstream HTTP status forwarded |
| Error — JSON parse failure | 500 |
| Error — Zod validation failure | 500 |

---

#### `pages/api/offer-switch-paths.ts` — CREATE

**Why:** New GET route exposes upgrade eligibility to the frontend; keeps the controller testable in isolation.

| Aspect | Detail |
|--------|--------|
| Allowed methods | `GET` only; returns `405` for all others |
| Auth | `handlePrerequisites(req)` to obtain `accessToken` |
| Query params extracted | `market-segment`, `country`, `language`, `offer-id`, `subscription-id`, `customer-id`, `limit`, `offset` |
| Delegates to | `getOfferSwitchPaths(accessToken, queryParams)` |
| Request ID forwarding | `forwardRequestIdHeader` |
| Success response | `200` with `OfferSwitchPathsResponse` body |
| Error response | `ApiError` → matching HTTP status code forwarded |

---

### Modified Files

#### `models/Order.ts` — MODIFY

| Symbol | Change | Why |
|--------|--------|-----|
| `LinkSchema` | Add `export` keyword | `SwitchOrder.ts` imports this to avoid duplicating the schema |
| `LineItemSchema` | Add `export` keyword | `SwitchOrder.ts` imports this to avoid duplicating the schema |
| `PricingSummarySchema` | Add `export` keyword | `SwitchOrder.ts` imports this to avoid duplicating the schema |

---

#### `pages/api/orders.ts` — MODIFY

| Location | Change | Why |
|----------|--------|-----|
| Imports | Add `createPreviewSwitchOrder`, `createSwitchOrder` from `../../controllers/switchOrderController` | New branches call these handlers |
| Route handler | Add `else if (type === ORDER_API_TYPE.PREVIEW_SWITCH)` — destructure `customerId`, `externalReferenceId`, `currencyCode`, `lineItems`, `cancellingItems` from body; pass `restQuery` as `queryParams`; call `createPreviewSwitchOrder`; return `201` | Exposes the preview-switch endpoint to the UI |
| Route handler | Add `else if (type === ORDER_API_TYPE.SWITCH)` — same shape; call `createSwitchOrder`; return `201` | Exposes the place-switch-order endpoint to the UI |

---

#### `utils/constants.ts` — MODIFY

| Symbol | Change | Why |
|--------|--------|-----|
| `ORDER_API_TYPE` | Add `PREVIEW_SWITCH: 'PREVIEW_SWITCH'` and `SWITCH: 'SWITCH'` | Orders route handler and hooks switch on these string constants |
| `SwitchType` *(new)* | Add enum — `FULL_ONLY = 'FULL_ONLY'`, `PARTIAL_ALLOWED = 'PARTIAL_ALLOWED'` | Upgrade page uses this to determine whether quantity is editable |

---

#### `utils/cartUtils.ts` — MODIFY

| Symbol | Change | Why |
|--------|--------|-----|
| `cartItemsToSwitchOrderLineItems` *(new)* | Add function `(item: CartItem, sourceSubscriptionId: string): { lineItems, cancellingItems }` | Switch orders need a different shape from regular orders: a target `lineItems` array plus a `cancellingItems` array for the subscription being replaced |

**Return shape of `cartItemsToSwitchOrderLineItems`:**

| Property | Value |
|----------|-------|
| `lineItems` | `[{ extLineItemNumber: 1, offerId, quantity, flexDiscountCodes? }]` |
| `cancellingItems` | `[{ extLineItemNumber: 2, referenceLineItemNumber: 1, subscriptionId: sourceSubscriptionId, quantity }]` |

---

## Section 4: Acceptance Criteria Coverage

| Acceptance criterion (from experience card) | LLD section that satisfies it |
|---------------------------------------------|-------------------------------|
| Eligibility checked per subscription on product listing render | `offer-switch-paths.ts` CREATE; `switchOrderController.getOfferSwitchPaths` |
| No upgrade button shown if eligibility API fails (silent) | `getOfferSwitchPaths` throws `ApiError`; hook surfaces `isError`; UI suppresses button |
| Preview call is non-destructive | `createPreviewSwitchOrder` — `orderType: PREVIEW_SWITCH`; `fetch-price=true` query param |
| Estimated total from `pricingSummary[].totalLineItemPartnerPrice` | `SwitchOrderResponseSchema` captures `pricingSummary`; returned to UI in 201 body |
| Place order uses `orderType: SWITCH` | `createSwitchOrder` — `SwitchOrderSchema` enforces literal |
| `orderId` returned on success | `SwitchOrderResponseSchema.orderId?: string` |

---

## Section 5: Out of Scope

| Item | Reason |
|------|--------|
| Revert switch order (`PREVIEW_REVERT_SWITCH` / `REVERT_SWITCH`) | Defined in API spec but not in experience card flow; not implemented |
| `reassign-users` query parameter | Optional API feature not referenced in experience card |
| Order history display for SWITCH orders | Existing `getOrdersHistory` already returns SWITCH orders; no UI change needed per experience card |

---
