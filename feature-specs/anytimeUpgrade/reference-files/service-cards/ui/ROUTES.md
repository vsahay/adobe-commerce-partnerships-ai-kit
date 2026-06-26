# ROUTES — Bridge

## Overview

| Property | Value |
|---|---|
| Router | Next.js file-based (Pages Router — `pages/` directory) |
| Auth guard mechanism | None — no authentication guard at the UI layer; auth is entirely server-side (IMS token in API routes) |
| Auth guard file | N/A |
| Root redirect | `/` → `/resellers` (via `router.push` in `pages/index.tsx`) |
| Unauthenticated redirect | N/A — UI pages are not auth-protected |
| Layout hierarchy | All pages use `<Layout>` wrapper (`components/Layout.tsx`) except `orderConfirmation.tsx` which has its own header |

---

## Route Registry

| URL pattern | Page file | Path params | Query params | Auth required | Store dependencies before render |
|---|---|---|---|---|---|
| `/` | `pages/index.tsx` | — | — | No | None |
| `/resellers` | `pages/resellers.tsx` | — | — | No | PartnerContext (partnerName display) |
| `/customers` | `pages/customers.tsx` | — | `resellerId` (required) | No | None |
| `/catalog` | `pages/catalog.tsx` | — | — | No | PartnerContext (`availableCurrencies`, `region` must be loaded before pricelist query fires) |
| `/checkout` | `pages/checkout.tsx` | — | — | No | CartContext (`customerInfoInCart.customerId` must be set; empty cart redirects via guard) |
| `/customerdetails` | `pages/customerdetails.tsx` | — | `customerId` (required), `resellerId` (required) | No | CartContext (for cart modal) |
| `/orderConfirmation` | `pages/orderConfirmation.tsx` | — | `customerId`, `currencyCode`, `totalAmount`, `customerName`, `resellerId`, `products` (JSON) | No | None — all data from query params |

---

## Auth Guard Pattern

There is no route-level auth guard. The application relies entirely on server-side authentication: Next.js API routes call `utils/imsTokenService.ts` to acquire an IMS token before proxying requests to the VIP MP API.

The only navigation guards present are **cart route guards** (`hooks/useCartRouteGuard.ts`) which show a confirmation dialog when a user tries to leave the checkout or customer details page with items in the cart.

```typescript
// hooks/useCartRouteGuard.ts usage in pages/checkout.tsx
const cartRouteGuard = useCartRouteGuard({
  router,
  getTotalItems,
  clearCart,
  shouldAllowNavigation: (url: string) => {
    if (url.includes('/orderConfirmation')) return true;
    if (!customerInfoInCart.customerId) return false;
    const urlParams = new URLSearchParams(url.split('?')[1] || '');
    const targetCustomerId = urlParams.get('customerId');
    return targetCustomerId === customerInfoInCart.customerId && url.includes('/customerdetails');
  },
});
```

**Rule for new pages:** No auth guard is needed at the page level. If the page requires a cart, use `useCartRouteGuard`. If the page requires query params, redirect early:

```typescript
// Pattern used in pages/customers.tsx
useEffect(() => {
  if (!resellerId && router.isReady) {
    router.push('/resellers');
  }
}, [resellerId, router.isReady, router]);
```

---

## Navigation Patterns

**Programmatic navigation:**

```typescript
// Standard push with query params
router.push({
  pathname: '/customerdetails',
  query: {
    resellerId: reseller.resellerId,
    customerId: customer.customerId,
  },
});

// After order creation — passes data via query string (no router state)
const queryParams = new URLSearchParams({
  customerId: customerInfoInCart.customerId || '',
  currencyCode: getOrderCurrency() || '',
  totalAmount: estimatedTotal?.toString() || '0',
  customerName: customerInfoInCart.customerName || '',
  resellerId: customerInfoInCart.resellerId || '',
  products: JSON.stringify(orderProducts),
});
router.push(`/orderConfirmation?${queryParams.toString()}`);
```

**State-passing strategy:** Query string parameters (no router state). Data not suited for query strings (cart items) lives in CartContext which is persisted to localStorage.

**Back navigation:** Standard browser back via breadcrumb `<NavigationPanel>` links. No custom back-navigation logic.

---

## Route Constants

Partial — only one route is centralised:

```typescript
// utils/constants.ts
export const ROUTES = {
  CUSTOMER_DETAILS: '/customerdetails',
} as const;

export const buildCustomerDetailsUrl = (customerId: string, resellerId: string): string => {
  return `${ROUTES.CUSTOMER_DETAILS}?customerId=${customerId}&resellerId=${resellerId}`;
};
```

All other route strings (`/resellers`, `/customers`, `/catalog`, `/checkout`, `/orderConfirmation`) are inline string literals scattered across components and pages. **Known inconsistency** — these should be centralised in `ROUTES` but are not currently.

---

## Error Routes

| Scenario | Handling |
|---|---|
| 404 — page not found | Next.js default 404 page (no custom `pages/404.tsx` found) |
| Auth failure | N/A — no client-side auth |
| Missing required query param | Inline redirect (e.g. `/customers` without `resellerId` → push to `/resellers`) |
| Empty cart on `/checkout` | Inline empty state: `<div>No items in cart.</div>` |
| API/render error | Inline error message with "Try Again" button (query errors); `ErrorToast` (mutation errors) |

---