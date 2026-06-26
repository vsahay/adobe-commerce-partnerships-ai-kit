# UI_MODULE_INDEX — Bridge

## §1 Workflow

| Step | What happens |
|---|---|
| 1. User navigates to `/resellers` | App shell loads; PartnerContext fires GET /api/partnerDetails |
| 2. User selects a reseller | Router pushes to `/customers?resellerId=<id>` |
| 3. User selects a customer | Router pushes to `/customerdetails?customerId=<id>&resellerId=<id>` |
| 4. User browses to `/catalog` | Infinite pricelist query fires after PartnerContext is ready |
| 5. User adds items to cart | CartContext updated; persisted to localStorage |
| 6. User selects customer from cart | FindOrCreateCustomer sets customerInfoInCart |
| 7. User navigates to `/checkout` | Order preview fires; cart items reconciled with server prices |
| 8. User places order | POST /api/orders?type=NEW; cart cleared; redirect to `/orderConfirmation` |

**Resolution order for new code:** `ROUTES.md` → `DATA_LAYER.md` → `STATE_MANAGEMENT.md` → this file.

---

## §2 Module Taxonomy

| Layer | Pattern | Example |
|---|---|---|
| Pages | `pages/*.tsx` (file = route) | `pages/catalog.tsx` |
| Components | `components/**/*.tsx` | `components/CatalogProductCard.tsx` |
| Hooks | `hooks/use*.ts` | `hooks/usePaginatedCustomersList.ts` |
| Context (store) | `contexts/*Context.tsx` | `contexts/CartContext.tsx` |
| Fetch functions | `utils/AccountApis.ts`, `utils/*Utils.ts` | `fetchCustomersPage` in `utils/AccountApis.ts` |
| Utils | `utils/*.ts` (no I/O, no side effects) | `getSKUFromOfferId` in `utils/commonUtils.ts` |
| Models | `models/*.ts` (Zod schemas) | `models/CustomerDetails.ts` |

---

## §3 Capability Taxonomy

| Domain | Capabilities |
|---|---|
| Navigation | Redirect from root, Reseller list |
| Account Management | Customer list, Customer details (products, account, purchase history) |
| Catalog | Product catalog browse, Cart management |
| Ordering | Checkout / order review, Order confirmation |
| Partner Config | Partner details loading |

---

## §4 Capability Catalogue

### 4.1 Root Redirect

| Field | Value |
|---|---|
| Domain | Navigation |
| Page | `pages/index.tsx` → [ROUTES](ROUTES.md#route-registry) |
| Component(s) | None — redirect only |
| Component hierarchy | None |
| Hook / Composable / Service | None |
| Store / Context | None |
| Util(s) | None |
| Form / Validation | None |
| Feature flags | None |
| Key Decisions | Rendering: CSR; Auth dep: none; Optimistic update: N/A; Fail-posture: open (just redirects) |
| Call graph | 1. Page mounts → 2. `router.push('/resellers')` |
| Enforcement | Not enforced — relies on code review |
| Scope | `index.tsx` — redirect only; no UI renders here |

---

### 4.2 Reseller List

| Field | Value |
|---|---|
| Domain | Navigation |
| Page | `pages/resellers.tsx` → [ROUTES](ROUTES.md#route-registry) |
| Component(s) | `Layout`, `NavigationPanel`, `LoadingSkeleton` |
| Component hierarchy | `ResellersPage` → `Layout` → `NavigationPanel` + reseller cards |
| Hook / Composable / Service | `usePaginatedResellersList` → [DATA_LAYER](DATA_LAYER.md#fetch-layer-summary) |
| Store / Context | `PartnerContext` (partnerName display) |
| Util(s) | `formatTotalCount` in `utils/commonUtils.ts`, `invalidateResellers` in `utils/queryUtils.ts` |
| Form / Validation | None |
| Feature flags | None |
| Key Decisions | Rendering: CSR; Auth dep: none; Optimistic update: none; Fail-posture: open (inline error + "Try Again" button); adjacent pages prefetched |
| Call graph | 1. Page mounts → 2. `usePaginatedResellersList` → 3. `fetchResellersPage('/api/resellers?type=getAllResellers&...')` → 4. reseller cards render |
| Enforcement | `resellerId` guard in `/customers` prevents orphaned navigation |
| Scope | Lists and paginates resellers; navigates to `/customers?resellerId=<id>`; does NOT create/edit resellers |

---

### 4.3 Customer List

| Field | Value |
|---|---|
| Domain | Account Management |
| Page | `pages/customers.tsx` → [ROUTES](ROUTES.md#route-registry) |
| Component(s) | `Layout`, `NavigationPanel`, `LoadingSkeleton` |
| Component hierarchy | `CustomersPage` → `Layout` → `NavigationPanel` + customer cards |
| Hook / Composable / Service | `usePaginatedCustomersList` → [DATA_LAYER](DATA_LAYER.md#fetch-layer-summary) |
| Store / Context | None |
| Util(s) | `formatTotalCount`, `invalidateCustomers` |
| Form / Validation | None |
| Feature flags | None |
| Key Decisions | Rendering: CSR; Auth dep: `resellerId` query param (redirects to `/resellers` if missing); Optimistic update: none; Fail-posture: open (inline error + "Try Again") |
| Call graph | 1. Page mounts → 2. `usePaginatedCustomersList({ resellerId })` → 3. `fetchCustomersPage` OR `searchCustomers` depending on search mode → 4. customer cards render |
| Enforcement | `resellerId` guard: redirects to `/resellers` if missing |
| Scope | Lists and paginates customers for a reseller; navigates to `/customerdetails`; does NOT create customers (that is in `components/common/CustomerSelector`) |

---

### 4.4 Customer Details

| Field | Value |
|---|---|
| Domain | Account Management |
| Page | `pages/customerdetails.tsx` → [ROUTES](ROUTES.md#route-registry) |
| Component(s) | `Layout`, `NavigationPanel`, `AccountDetailsPanel`, `CustomerProductsOverviewPanel`, `PurchaseHistoryPanel`, `CartDetails`, `ThreeYearCommit`, `ClearCartConfirmationDialog`, `LoadingSkeleton` |
| Component hierarchy | `CustomerDetailsPage` → `Layout` → `Tabs` [products: `CustomerProductsOverviewPanel`, account: `AccountDetailsPanel`, history: `PurchaseHistoryPanel`] + cart dropdown + modals |
| Hook / Composable / Service | `fetchCustomerSubscriptions`, `fetchCustomerDetails`, `getResellerDetails` → [DATA_LAYER](DATA_LAYER.md#fetch-layer-summary) |
| Store / Context | `CartContext`, `PartnerContext` |
| Util(s) | `formatCustomerDiscountsForHeader`, `fetchCustomerSubscriptions` in `utils/customerDetailsUtils.ts`; `customerDetailKeys` in `utils/queryUtils.ts` |
| Form / Validation | 3YC enrollment form inside `ThreeYearCommit` component |
| Feature flags | None |
| Key Decisions | Rendering: CSR; Auth dep: `customerId` + `resellerId` query params + `router.isReady`; Optimistic update: none; Fail-posture: open (separate error messages per tab); subscriptions use staleTime=0 with 8s polling refetch; cart route guard active |
| Call graph | 1. Page mounts → 2. Three parallel queries (subscriptions, customer details, reseller details) → 3. Components render tab panels |
| Enforcement | Cart route guard via `useCartRouteGuard`; query params required |
| Scope | Shows product subscriptions, account info, purchase history for one customer; initiates 3YC enrollment; manages cart sidebar; does NOT place orders (checkout) |

---

### 4.5 Product Catalog Browse

| Field | Value |
|---|---|
| Domain | Catalog |
| Page | `pages/catalog.tsx` → [ROUTES](ROUTES.md#route-registry) |
| Component(s) | `Layout`, `NavigationPanel`, `CatalogFiltersComponent`, `CatalogProductCard`, skeleton cards |
| Component hierarchy | `CatalogPage` → `Layout` → `CatalogFiltersComponent` + `CatalogProductCard[]` + infinite scroll trigger |
| Hook / Composable / Service | `useInfiniteQuery` with `fetchPricelistPage` → [DATA_LAYER](DATA_LAYER.md#fetch-layer-summary) |
| Store / Context | `PartnerContext` (currencies, region), `CartContext` (cart state) |
| Util(s) | `getMarketSegmentCode`, `getProductType`, `getProductCloudCategory` in `utils/commonUtils.ts` + `utils/iconUtils.ts`; `prefetchAllMarketSegments`, `getPricelistQueryKey` in `utils/prefetchProducts.ts` |
| Form / Validation | None |
| Feature flags | None |
| Key Decisions | Rendering: CSR; Auth dep: PartnerContext must be loaded (enabled guard); Optimistic update: none; Fail-posture: open (inline error card); infinite scroll via IntersectionObserver; background prefetch of all market segments; client-side filter on fetched data (type, category, search) |
| Call graph | 1. Page mounts → 2. PartnerContext loads → 3. `useInfiniteQuery` fires `fetchPricelistPage(POST /api/pricelist)` → 4. `CatalogProductCard[]` renders → 5. IntersectionObserver triggers `fetchNextPage()` |
| Enforcement | `enabled: isPartnerDetailsReady && !!region && !!filters.currency` |
| Scope | Browse and add-to-cart for VIP MP products; market segment + type + category + search filters; does NOT manage customer assignment (that is in `FindOrCreateCustomer`) |

---

### 4.6 Cart Management

| Field | Value |
|---|---|
| Domain | Catalog |
| Page | Modal overlaid on `catalog.tsx` and `customerdetails.tsx` → [ROUTES](ROUTES.md#route-registry) |
| Component(s) | `CartDetails`, `FindOrCreateCustomer` |
| Component hierarchy | Overlay div → `CartDetails` + `FindOrCreateCustomer` (sequential modal flow) |
| Hook / Composable / Service | `useCreateOrderPreview` → [DATA_LAYER](DATA_LAYER.md#fetch-layer-summary) |
| Store / Context | `CartContext` |
| Util(s) | `cartItemsToLineItems`, `updateCartItemsWithSkuMapping`, `fetchOrderPreview` in `utils/cartUtils.ts` |
| Form / Validation | Customer creation form in `FindOrCreateCustomer` (react-hook-form + zod) |
| Feature flags | None |
| Key Decisions | Rendering: CSR; Auth dep: none for cart display, `customerId` required for preview; Optimistic update: none; Fail-posture: open (ErrorToast on preview failure); `updateCartItemsFromAPI` reconciles offer IDs after preview by SKU matching |
| Call graph | 1. User opens cart → 2. `CartDetails` renders from `CartContext` → 3. User clicks "Continue" → 4. `FindOrCreateCustomer` shown → 5. customer set → 6. `useCreateOrderPreview` fires → 7. cart updated with server prices |
| Enforcement | Market segment mixing blocked in `addProductToCart` |
| Scope | Cart item display, quantity adjustment, customer assignment, order preview; does NOT place the order (checkout does) |

---

### 4.7 Checkout / Order Review

| Field | Value |
|---|---|
| Domain | Ordering |
| Page | `pages/checkout.tsx` → [ROUTES](ROUTES.md#route-registry) |
| Component(s) | `Layout`, `NavigationPanel`, `CheckoutReviewDetails`, `CheckoutSummary`, `ErrorToast`, `ClearCartConfirmationDialog` |
| Component hierarchy | `CheckoutPage` → `Layout` → `CheckoutReviewDetails` (left) + `CheckoutSummary` (right) |
| Hook / Composable / Service | `useCreateOrder`, `getResellerDetails` → [DATA_LAYER](DATA_LAYER.md#fetch-layer-summary) |
| Store / Context | `CartContext` |
| Util(s) | `fetchOrderPreview`, `cartItemsToLineItems`, `updateCartItemsWithSkuMapping` in `utils/cartUtils.ts`; `generateExternalReferenceId` in `utils/commonUtils.ts` |
| Form / Validation | Promo code input (inline, no form library) |
| Feature flags | None |
| Key Decisions | Rendering: CSR; Auth dep: `customerInfoInCart.customerId` required (recalculation skipped if missing); Optimistic update: none; Fail-posture: open (ErrorToast on order failure, `createOrder.reset()` called); cart route guard active; auto-recalculation on mount |
| Call graph | 1. Page mounts → 2. `handleRecalculate()` fires (POST /api/orders?type=Preview) → 3. Cart items updated with server prices → 4. User clicks "Place Order" → 5. `useCreateOrder.mutate()` → 6. success: `clearCart()` + navigate to /orderConfirmation |
| Enforcement | Cart route guard; empty cart shows inline message; `customerInfoInCart.customerId` checked before placing |
| Scope | Final review and placement of a new order; promo code application; does NOT handle renewals (separate flow in customerdetails) |

---

### 4.8 Order Confirmation

| Field | Value |
|---|---|
| Domain | Ordering |
| Page | `pages/orderConfirmation.tsx` → [ROUTES](ROUTES.md#route-registry) |
| Component(s) | Custom header (no Layout wrapper), product list |
| Component hierarchy | `OrderConfirmedPage` → custom header + order summary card + confirmation message |
| Hook / Composable / Service | None |
| Store / Context | None — all data from URL query params |
| Util(s) | `getIconWithFallback` in `utils/iconUtils.ts`; `formatPrice` in `utils/commonUtils.ts` |
| Form / Validation | None |
| Feature flags | None |
| Key Decisions | Rendering: CSR; Auth dep: none; Optimistic update: N/A; Fail-posture: open (missing/malformed products param → empty list); data from query params only — no API calls on this page |
| Call graph | 1. Page mounts → 2. `router.query` parsed → 3. products JSON parsed → 4. confirmation card renders |
| Enforcement | Not enforced — data integrity relies on correct params from checkout |
| Scope | Display-only confirmation; "Continue" navigates to `/customerdetails`; does NOT create or modify any orders |

---

### 4.9 Partner Config Load

| Field | Value |
|---|---|
| Domain | Partner Config |
| Page | `pages/_app.tsx` (global provider) |
| Component(s) | `PartnerProvider` |
| Component hierarchy | `_app.tsx` → `PartnerProvider` wraps all pages |
| Hook / Composable / Service | `fetchPartnerDetails` → [DATA_LAYER](DATA_LAYER.md#fetch-layer-summary) |
| Store / Context | `PartnerContext` |
| Util(s) | None |
| Form / Validation | None |
| Feature flags | None |
| Key Decisions | Rendering: CSR; Auth dep: none; staleTime=24h gcTime=7d; retry=1; blocks catalog rendering until loaded |
| Call graph | 1. `_app.tsx` mounts → 2. `PartnerProvider` mounts → 3. `useQuery` fires `fetchPartnerDetails(GET /api/partnerDetails)` → 4. `partnerDetails` available to all pages |
| Enforcement | `isPartnerDetailsReady` guard on catalog pricelist query |
| Scope | Loads and caches partner configuration; does NOT manage per-user auth |

---

## §5 Cross-Reference Indexes

### Route → Capability

| Route | Capability |
|---|---|
| `/` | 4.1 Root Redirect |
| `/resellers` | 4.2 Reseller List |
| `/customers` | 4.3 Customer List |
| `/customerdetails` | 4.4 Customer Details |
| `/catalog` | 4.5 Product Catalog Browse + 4.6 Cart Management |
| `/checkout` | 4.7 Checkout / Order Review |
| `/orderConfirmation` | 4.8 Order Confirmation |

### Fetch Unit → Capabilities

| Fetch unit | Capabilities |
|---|---|
| `fetchPartnerDetails` | 4.9 Partner Config, 4.5 Catalog (indirectly) |
| `fetchResellersPage` / `searchResellers` | 4.2 Reseller List |
| `getResellerDetails` | 4.3 Customer List, 4.4 Customer Details, 4.7 Checkout |
| `fetchCustomersPage` / `searchCustomers` | 4.3 Customer List |
| `fetchCustomerDetails` | 4.4 Customer Details |
| `fetchCustomerSubscriptions` | 4.4 Customer Details |
| `fetchPricelistPage` | 4.5 Product Catalog Browse |
| `fetchProductPricing` | 4.4 Customer Details (renewal pricing) |
| `fetchOrderPreview` / `useCreateOrderPreview` | 4.6 Cart Management, 4.7 Checkout |
| `useCreateOrder` | 4.7 Checkout |

### Store → Capabilities It Gates

| Store / Context | Gates |
|---|---|
| `CartContext` | 4.6 Cart Management, 4.7 Checkout, 4.4 Customer Details (cart sidebar) |
| `PartnerContext` | 4.2 Reseller List (partnerName), 4.5 Catalog (currencies + region required), 4.4 Customer Details (primaryCurrency) |

### Browser Storage Keys

| Key | Store | Type | Description |
|---|---|---|---|
| `bridge_cart` | CartContext | `localStorage` | Cart items, quantities, and customer info in cart |

---

## §6 Cross-Cutting Modules

| Module | File | Used in |
|---|---|---|
| Layout wrapper | `components/Layout.tsx` | All pages except `orderConfirmation.tsx` |
| Breadcrumb nav | `components/NavigationPanel.tsx` | All pages |
| Loading skeleton | `components/LoadingSkeleton.tsx` | Customer list, Customer details |
| Error toast | `utils/ToastMessageUtils.tsx` (`ErrorToast`, `SuccessToast`) | Catalog, Checkout, Customer details |
| Cart details modal | `components/catalogCart/CartDetails.tsx` | Catalog, Customer details |
| Cart route guard | `hooks/useCartRouteGuard.ts` | Checkout, Customer details |
| Query key factories | `utils/queryUtils.ts` | All hooks |
| Common card styles | `styles/CommonCard.module.css` | Resellers, Customers, Catalog |
| Pino logger | `utils/logger.ts` (`createClientLogger`) | Multiple pages and hooks |

---

## §7 Type Inventory

| Type | Count |
|---|---|
| Pages | 7 |
| API route handlers | 9 |
| Contexts (stores) | 2 |
| TanStack Query hooks | 4 (`usePaginatedCustomersList`, `usePaginatedResellersList`, `useOrderOperations` × 2, `useProductPricing` × 2) |
| Fetch functions (`AccountApis.ts`) | 6 |
| Reusable components | ~30 |
| Zod models | 11 |
| Util files | 12+ |
| CSS Module files | 22+ |

---