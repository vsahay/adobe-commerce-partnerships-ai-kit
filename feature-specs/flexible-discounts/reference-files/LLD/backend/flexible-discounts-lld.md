# LLD — Flexible Discounts (Backend)
**Target repo:** bridge  
**Feature spec:** `feature-specs/flexible-discounts/experience-card/flexible-discounts.md`  
**API spec:** `feature-specs/flexible-discounts/APISpec/flexible-discounts-apispec.md`  
**Service cards:** `feature-specs/flexible-discounts/reference-files/service-cards/backend/` (reference)

---

## §1 — Summary

This feature introduces a single new BFF endpoint — `GET /api/flex-discounts` — that proxies two upstream Partner API operations: market-scoped discount browsing (`GET /v3/flex-discounts`) and customer-scoped reusable discount lookup (`GET /v3/customers/{customerId}/flex-discounts`). The route dispatches on a `type` query parameter, following the established `pages/api/` pattern used by every existing handler in bridge.

---

## §2 — Data Flow

**Operation 1: Get market-scoped flex discounts**

```
UI: GET /api/flex-discounts?type=getFlexDiscounts&market-segment=COM&country=US&[optional filters]
  → pages/api/flex-discounts.ts
  → handlePrerequisites(req) → getAccessToken() → IMS bearer token
  → type === FLEX_DISCOUNT_API_TYPE.GET_FLEX_DISCOUNTS
  → flexDiscountsController.getFlexDiscounts(params, token)
  → GET {PARTNER_API_BASE_URL}/v3/flex-discounts?market-segment=...&country=...&[forwarded params]
  ← FlexDiscountsResponseSchema.parse(data)
  ← HTTP 200 + { limit, offset, count, totalCount, flexDiscounts[] }
```

**Operation 2: Get customer-scoped reusable flex discounts**

```
UI: GET /api/flex-discounts?type=getCustomerFlexDiscounts&customerId=XXX&limit=50
  → pages/api/flex-discounts.ts
  → handlePrerequisites(req) → IMS bearer token
  → type === FLEX_DISCOUNT_API_TYPE.GET_CUSTOMER_FLEX_DISCOUNTS
  → flexDiscountsController.getCustomerFlexDiscounts(params, token)
  → GET {PARTNER_API_BASE_URL}/v3/customers/{customerId}/flex-discounts?limit=50&offset=0
  ← FlexDiscountsResponseSchema.parse(data)
  ← HTTP 200 + { limit, offset, count, totalCount, flexDiscounts[] }
```

**Operations 3+: Order and subscription submission (no backend changes)**

```
[Already implemented — flexDiscountCodes passed through lineItems in POST /api/orders]
[Already implemented — flexDiscountCodes in autoRenewal body via PATCH /api/subscriptions]
```

---

## §3 — Change Summary

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `pages/api/flex-discounts.ts` | CREATE | Route | New BFF route to expose flex discount discovery to the UI |
| `controllers/flexDiscountsController.ts` | CREATE | Controller | Business logic for both flex discount fetch operations |
| `models/FlexDiscounts.ts` | CREATE | Model | Zod schemas for request validation and response parsing |
| `utils/constants.ts` | MODIFY | Constants | Add `FLEX_DISCOUNT_API_TYPE` routing constants |

---

### `pages/api/flex-discounts.ts` — CREATE

GET-only handler following the standard `pages/api/` pattern from `CODE_PATTERNS.md`.

- Call `handlePrerequisites(req)` first — throws `ApiError(500)` if IMS token cannot be obtained
- Return `405` for any non-GET method via `res.status(405).end(...)`
- Dispatch on `type` query param:

| `type` value | Controller call |
|---|---|
| `FLEX_DISCOUNT_API_TYPE.GET_FLEX_DISCOUNTS` | `flexDiscountsController.getFlexDiscounts(params, token)` |
| `FLEX_DISCOUNT_API_TYPE.GET_CUSTOMER_FLEX_DISCOUNTS` | `flexDiscountsController.getCustomerFlexDiscounts(params, token)` |
| anything else | `throw new ApiError('Invalid type parameter', 400)` |

- Call `forwardRequestIdHeader(res, result.requestId)` before returning HTTP 200
- Standard `ApiError` catch block per `CODE_PATTERNS.md §Error Handling`

---

### `controllers/flexDiscountsController.ts` — CREATE

Follows the controller pattern from `CODE_PATTERNS.md §Controller Pattern`. Both functions use the existing PartnerAPI connector (Bearer + `x-api-key` + `X-Correlation-Id` + `X-Request-Id` headers, `PARTNER_API_BASE_URL`).

**`getFlexDiscounts(params, token): Promise<BackendResult<FlexDiscountsResponse>>`**

1. Validate with `GetFlexDiscountsRequestSchema.parse(params)`; throw `ApiError(400)` on failure
2. Build query string forwarding all provided params — required: `market-segment`, `country`; optional: `categories`, `offer-ids`, `flex-discount-id`, `flex-discount-code`, `include-eligible-reusable-discounts`, `start-date`, `end-date`, `limit`, `offset`
3. `fetch GET {PARTNER_API_BASE_URL}/v3/flex-discounts?{queryString}` with standard headers
4. `FlexDiscountsResponseSchema.parse(data)`; throw `ApiError(500, requestId)` on failure
5. `extractRequestIdFromResponse(result)` and return `{ data, requestId }`

**`getCustomerFlexDiscounts(params, token): Promise<BackendResult<FlexDiscountsResponse>>`**

1. Validate with `GetCustomerFlexDiscountsRequestSchema.parse(params)`; throw `ApiError(400)` on failure
2. Build query string from `limit` and `offset` when provided
3. `fetch GET {PARTNER_API_BASE_URL}/v3/customers/{customerId}/flex-discounts?{queryString}` with standard headers
4. `FlexDiscountsResponseSchema.parse(data)`; throw `ApiError(500, requestId)` on failure
5. `extractRequestIdFromResponse(result)` and return `{ data, requestId }`

---

### `models/FlexDiscounts.ts` — CREATE

All response schemas use `.passthrough()` per `CODE_PATTERNS.md §Validation` — unexpected fields from the Partner API are preserved.

```typescript
const FlexDiscountOutcomeValueSchema = z.object({
  country: z.string().optional(),
  currency: z.string().optional(),
  value: z.number(),
}).passthrough();

const FlexDiscountOutcomeSchema = z.object({
  type: z.enum(['FIXED_PRICE', 'FIXED_DISCOUNT', 'PERCENTAGE_DISCOUNT']),
  discountValues: z.array(FlexDiscountOutcomeValueSchema),
}).passthrough();

const FlexDiscountQualificationSchema = z.object({
  baseOfferIds: z.array(z.string()),
}).passthrough();

const FlexDiscountSchema = z.object({
  id: z.string().optional(),
  category: z.string().optional(),
  code: z.string().optional(),
  name: z.string().optional(),
  description: z.string().optional(),
  startDate: z.string().optional(),
  endDate: z.string().optional(),
  discountLockEndDate: z.string().optional(),
  status: z.string().optional(),
  qualification: FlexDiscountQualificationSchema.optional(),
  outcomes: z.array(FlexDiscountOutcomeSchema).optional(),
}).passthrough();

export const FlexDiscountsResponseSchema = z.object({
  limit: z.number(),
  offset: z.number(),
  count: z.number(),
  totalCount: z.number(),
  flexDiscounts: z.array(FlexDiscountSchema),
}).passthrough();

export const GetFlexDiscountsRequestSchema = z.object({
  'market-segment': z.string(),
  country: z.string(),
  categories: z.string().optional(),
  'offer-ids': z.string().optional(),
  'flex-discount-id': z.string().optional(),
  'flex-discount-code': z.string().optional(),
  'include-eligible-reusable-discounts': z.string().optional(),
  'start-date': z.string().optional(),
  'end-date': z.string().optional(),
  limit: z.string().optional(),
  offset: z.string().optional(),
});

export const GetCustomerFlexDiscountsRequestSchema = z.object({
  customerId: z.string(),
  limit: z.string().optional(),
  offset: z.string().optional(),
});

export type FlexDiscountsResponse = z.infer<typeof FlexDiscountsResponseSchema>;
```

---

### `utils/constants.ts` — MODIFY

Add after the last `*_API_TYPE` block:

```typescript
export const FLEX_DISCOUNT_API_TYPE = {
  GET_FLEX_DISCOUNTS: 'getFlexDiscounts',
  GET_CUSTOMER_FLEX_DISCOUNTS: 'getCustomerFlexDiscounts',
} as const;
```

---

## §4 — DB Changes

No DB changes — service is stateless.

---

## §5 — Design Decisions

| Decision | Why | Trade-off | Enforcement |
|----------|-----|-----------|-------------|
| Single `/api/flex-discounts` route dispatching on `type` | Consistent with every existing `pages/api/` handler (`orders.ts`, `subscriptions.ts`, etc.) — `CODE_PATTERNS.md §API Route Handler Pattern` establishes this as the project standard | Route file carries two distinct operations; offset by the fact that both share the same response schema and connector | `CODE_PATTERNS.md` pattern; add `/api/flex-discounts` to `CONTRACTS.md` after merge |
| No changes to order or subscription routes | `LineItemSchema.flexDiscountCodes?: string[]` already present per `MODULE_INDEX.md §Order Operations`; `updateSubscription` with `resetFlexDiscount` already implemented per `MODULE_INDEX.md §Subscription Management` | Zero-scope on existing routes means discount application is provably not regrested | Existing `CONTRACTS.md` contracts unchanged |
| Query params forwarded as-is (no transformation) | Both upstream endpoints accept the same param names the UI passes; no mapping needed; avoids a translation layer that could diverge | If the Partner API renames a param, the BFF must be updated in lock-step | No enforcement — relies on API spec alignment |
| `.passthrough()` on all response schemas | Consistent with `CODE_PATTERNS.md §Validation` — all existing response schemas use `.passthrough()` to preserve unexpected fields | `previewFlexDiscount` and other future fields pass through automatically without schema changes | Convention established in `CODE_PATTERNS.md` |

---

## §6 — Acceptance Criteria Coverage

| Goal (from experience card) | Covered by | Status |
|-----------------------------|-----------|--------|
| Partners can quickly discover which discounts are available for a given country and market segment | §3 `pages/api/flex-discounts.ts` + `controllers/flexDiscountsController.ts` (`getFlexDiscounts`) | ✅ Covered |
| Partners can narrow results to discounts relevant to a specific use case (new purchases, renewals, upgrades, 3YC) | §3 `controllers/flexDiscountsController.ts` — `offer-ids`, `categories`, `flex-discount-code`, `start-date`/`end-date` params forwarded to Partner API | ✅ Covered |
| Partners can copy a discount code and understand exactly which products and pricing it applies to | §3 `models/FlexDiscounts.ts` — `FlexDiscountSchema` exposes `code`, `qualification.baseOfferIds`, and `outcomes` | ✅ Covered |
| During checkout, partners can discover and apply an eligible discount without leaving the order flow | §3 `controllers/flexDiscountsController.ts` (`getFlexDiscounts` with `offer-ids`); order submission via existing `/api/orders` (no change) | ✅ Covered |
| During renewal editing, partners can discover and apply renewal-eligible discounts | §3 `controllers/flexDiscountsController.ts` (`getFlexDiscounts` with `start-date=today`); `previewFlexDiscount` passes through existing `/api/orders` `PREVIEW_RENEWAL` response via `.passthrough()` | ✅ Covered |

---

## §7 — Out of Scope
- **BFF-level caching of discount results:** Results are proxied fresh on each request. Caching would require TTL management and invalidation logic not present in any existing route.
- **Rate limiting / quota enforcement:** Handled entirely by the upstream Partner API — the BFF propagates 429 responses as-is via the existing `ApiError` catch block.
