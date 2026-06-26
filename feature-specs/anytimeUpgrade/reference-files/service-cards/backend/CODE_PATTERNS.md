# Code Patterns — bridge

---

## 1. Package Layout

| Layer | Path | Convention |
|-------|------|-----------|
| HTTP surface (Next.js API routes) | `pages/api/` | One file per resource (`customers.ts`, `orders.ts`, …). Each exports a single `handler(req, res)` function. |
| Controllers (outbound business logic) | `controllers/` | One file per domain (`customerController.ts`, `orderController.ts`, …). Exported async functions — no classes. |
| Zod schemas + TypeScript types | `models/` | One file per domain. Each file exports both the Zod schema (`*Schema`) and the inferred TypeScript type (`type Foo = z.infer<typeof FooSchema>`). |
| Shared utilities | `utils/` | Exported functions — no classes except `ApiError`. Utility scope per file name (`logger.ts`, `constants.ts`, `imsTokenService.ts`, …). |
| TypeScript type definitions | `types/` | Plain TypeScript `interface` and `type` declarations. |
| React hooks | `hooks/` | `use*` naming convention per React conventions. |
| React contexts | `contexts/` | One file per context (`CartContext.tsx`, `PartnerContext.tsx`). |
| React components | `components/` | PascalCase filenames, grouped by feature subdirectory. |

---

## 2. Naming Conventions

| Kind | Convention | Example |
|------|-----------|---------|
| Controller functions | `camelCase` verb + noun | `getCustomer`, `createNewOrder`, `updateSubscription` |
| Zod schemas | PascalCase + `Schema` suffix | `CustomerDetailsSchema`, `NewOrderSchema` |
| TypeScript types (inferred) | PascalCase without suffix | `CustomerDetails`, `Order`, `Subscription` |
| Env var keys | `SCREAMING_SNAKE_CASE` | `PARTNER_API_BASE_URL`, `PARTNER_CLIENT_ID` |
| HTTP method constants | `HTTP_METHOD.GET` etc. | `utils/constants.ts` |
| Operation type constants | `*_API_TYPE.OP_NAME` | `CUSTOMER_API_TYPE.GET_CUSTOMER` |

---

## 3. Error Handling

**`ApiError` class** (`utils/apiError.ts`):
```typescript
throw new ApiError('Failed to fetch customer', 404, requestId);
// .status → HTTP status code
// .requestId → upstream X-Request-Id (forwarded to caller via X-Request-Id response header)
```

**Controller posture:**
- Non-2xx from upstream → `throw new ApiError(message, status, requestId)`
- Non-JSON response → `throw new ApiError('Non-JSON response from Adobe API', status, requestId)`
- Zod validation failure on **input** → `throw new ApiError('Input validation failed', 400)`
- Zod validation failure on **response** → `throw new ApiError('...data validation failed', 500, requestId)`
- For orders: upstream error body is JSON.stringified and stored as the `ApiError.message` — API route parses it back

**API route posture:**
```typescript
if (error instanceof ZodError) return res.status(400).json({ error: error.errors });
if (error instanceof ApiError) return res.status(error.status).json({ error: error.message });
return res.status(500).json({ error: 'Internal Server Error' });
```

`X-Request-Id` from upstream is always forwarded to the browser via `forwardRequestIdHeader(res, error.requestId)`.

---

## 4. Validation Approach

- **Library:** Zod 3.x
- **Invocation point:** Dual — inputs validated in controller **before** fetch; responses validated in controller **after** fetch
- **Pattern:**
```typescript
const parsed = MySchema.parse(body);    // throws ZodError on failure
// ... call fetch ...
const validated = ResponseSchema.parse(data);  // throws ApiError 500 on failure
```
- Zod `passthrough()` used on response schemas where extra fields are expected from the upstream API (Order, Subscription, PriceList).
- `safeParse` used only in unit tests; `parse` (throwing) used in all production paths.

---

## 5. DTO Patterns

**`BackendResult<T>`** (`models/Result.ts`) — all controller return types:
```typescript
interface BackendResult<T> { data: T; requestId?: string; }
```

**Request DTOs** are the `z.infer` of their schema and live in `models/`:
- `CreateCustomer`, `UpdateCustomer` → `models/CustomerDetails.ts`
- `NewOrderSchema`, `ReturnOrderSchema`, `PreviewOrderSchema`, `RenewalOrderSchema`, `PreviewRenewalOrderSchema` → `models/Order.ts`
- `PriceListRequest` → `models/PriceList.ts`

**Response DTOs** are validated on receipt and typed via `z.infer`:
- `CustomerDetails`, `CustomerMinimal` → `models/CustomerDetails.ts`, `models/Customer.ts`
- `Order`, `OrdersHistoryResponseWithPagination` → `models/Order.ts`
- `Subscription` → `models/Subscription.ts`
- `ResellerDetails`, `ResellerMinimal` → `models/ResellerDetails.ts`, `models/Reseller.ts`
- `PartnerDetails` → `models/PartnerDetails.ts`
- `PriceListResponse` → `models/PriceList.ts`
- `RecommendationResponse` → `models/recommendation.ts`

---

## 6. Logging

- **Library:** Pino 9.x
- **Logger creation:** `createAPILogger(req)` for route handlers; `createControllerLogger(name, operation)` for controllers
- **Log fields (standard):**
  - Route handlers: `component: 'api'`, `method`, `path`, `correlationId`
  - Controllers: `component: 'controller'`, `controller`, `operation`
- **Log functions:** `logRequest(logger, body, meta)` → INFO; `logResponse(logger, data, requestId, meta)` → INFO; `logErrorResponse(logger, error, requestId, meta)` → ERROR
- **Production format:** Structured JSON (Pino default)
- **Dev format:** pino-pretty with colour, timestamp, and `messageFormat: '{service}[{environment}] {msg}'`
- **Test format:** Pino level `error` only (minimal output)

---