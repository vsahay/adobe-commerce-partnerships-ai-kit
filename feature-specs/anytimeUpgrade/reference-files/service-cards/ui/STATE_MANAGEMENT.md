# STATE_MANAGEMENT — Bridge

## Overview

| Property | Value |
|---|---|
| State library | React Context API (built-in) |
| Number of stores / contexts | 2 (CartContext, PartnerContext) |
| Server state | TanStack Query v5 — managed per-hook, no global store |
| Initialization chain enforcement | Manual: `CartContext` hydrates from localStorage on mount; `PartnerContext` fires a query on mount |
| Persisted contexts | CartContext → `localStorage` key `bridge_cart` |

---

## 1. CartContext

**File:** `contexts/CartContext.tsx`  
**Purpose:** Manages the shopping cart (items, quantities, customer assignment) with localStorage persistence.

### State shape

```typescript
interface CartContextType {
  cartItemIdToQuantityMap: { [offerId: string]: number };
  cartItemIdToCartItemDetailsMap: { [offerId: string]: Omit<CartItem, 'quantity'> };
  customerInfoInCart: {
    customerId?: string;
    customerName?: string;
    resellerId?: string;
    resellerName?: string;
    makeAvailable?: 'now' | 'atRenewal';
  };
  showCartModal: boolean;
}

interface CartItem {
  id: string;           // UUID assigned at add-time
  offerId: string;      // VIP MP offer ID (may change after preview — use SKU for matching)
  productName: string;
  pricePerUnit: number;
  quantity: number;
  currency: string;
  productFamily?: string;
  lineItemTotal?: number;    // Set after order preview
  marketSegment?: string;    // COM | EDU | GOV
  discountCode?: string;     // Promo code if applied
}
```

### Actions / mutations

| Function | Signature | Effect |
|---|---|---|
| `addToCart` | `(offerId, productName, pricePerUnit, currency, marketSegment?, productFamily?, quantity?)` | Adds or increments item; generates UUID on first add |
| `removeFromCart` | `(offerId: string)` | Removes item from both maps |
| `updateQuantity` | `(offerId: string, quantity: number)` | Updates quantity; calls `removeFromCart` if ≤ 0 |
| `clearCart` | `()` | Empties cart and clears customerInfoInCart |
| `getTotalItems` | `() => number` | Computed sum of all quantities |
| `setCustomerInfoInCart` | `(info: CustomerInfoInCart)` | Sets customer/reseller assignment |
| `clearCustomerInfoInCart` | `()` | Clears customer assignment only |
| `setShowCartModal` | `(show: boolean)` | Toggles cart modal visibility |
| `updateCartItemsFromAPI` | `(updatedItems: CartItem[])` | Reconciles offer IDs after order preview — matches by SKU (first 8 chars of offerId) |

### Initialization

```typescript
// contexts/CartContext.tsx — useEffect on mount
useEffect(() => {
  const savedCart = loadCartFromLocalStorage();   // reads from localStorage key 'bridge_cart'
  setCartItemIdToQuantityMap(savedCart.cartItemIdToQuantityMap);
  setCartItemIdToCartItemDetailsMap(savedCart.cartItemIdToCartItemDetailsMap);
  setCustomerInfoState(savedCart.customerInfoInCart);
  setIsInitialized(true);   // guards the save effect
}, []);
```

Initialization is **synchronous** (JSON.parse from localStorage). The `isInitialized` flag prevents a false save on the first render before hydration completes.

### Storage persistence

| Key | Storage | Contents | Duration | Version | Migration |
|---|---|---|---|---|---|
| `bridge_cart` | `localStorage` | `{ cartItemIdToQuantityMap, cartItemIdToCartItemDetailsMap, customerInfoInCart }` | Until `clearCart()` | None | None — corrupt data returns empty cart |

### Consumer pattern

```typescript
// Any component within CartProvider
import { useCart } from '../contexts/CartContext';

const { cartItemIdToQuantityMap, addToCart, getTotalItems } = useCart();
// useCart() throws if called outside CartProvider
```

### Rules

- Cart items are keyed by `offerId`. After an order preview, offer IDs may change; `updateCartItemsFromAPI` reconciles by matching the first 8 characters (SKU). Do not compare full offer IDs across pre/post-preview state.
- All items in a cart must share the same `marketSegment`. Mixing segments is blocked in the Catalog page with an error toast.
- `clearCart()` also clears `customerInfoInCart`. Call `clearCustomerInfoInCart()` to clear only the customer without affecting cart items.

---

## 2. PartnerContext

**File:** `contexts/PartnerContext.tsx`  
**Purpose:** Fetches and exposes partner contract details (currencies, region, market segments) from `/api/partnerDetails`. Cached aggressively since this data changes rarely.

### State shape

```typescript
interface PartnerContextType {
  partnerDetails: PartnerDetails | undefined;  // Raw API response
  partnerName: string;
  availableCurrencies: Currency[];             // All currencies from contract
  region: string;                              // Derived from first currency's priceRegion
  regionCurrencies: Currency[];                // Currencies filtered to match region
  marketSegments: string[];                    // Unique VIPMP market segment codes
  isLoading: boolean;
  error: Error | null;
}
```

### Initialization

```typescript
// contexts/PartnerContext.tsx
const { data: partnerDetails, isLoading, error } = useQuery({
  queryKey: ['partnerDetails'],
  queryFn: fetchPartnerDetails,   // GET /api/partnerDetails
  staleTime: 24 * 60 * 60 * 1000, // 24 hours
  gcTime:    7 * 24 * 60 * 60 * 1000, // 7 days
  retry: 1,
});
```

Initialization is **asynchronous**. Pages that require partner data (Catalog, Resellers) guard renders on `isLoading`.

### Computed / derived values

| Property | Derivation |
|---|---|
| `availableCurrencies` | `partnerDetails.currencies` |
| `region` | `availableCurrencies[0].priceRegion` |
| `regionCurrencies` | `availableCurrencies` filtered by `currency.priceRegion === region` |
| `marketSegments` | Unique `marketSegment` values from programs where `programType === 'VIPMP'` |

### Consumer pattern

```typescript
import { usePartnerDetails } from '../contexts/PartnerContext';

const { regionCurrencies, marketSegments, isLoading } = usePartnerDetails();
// usePartnerDetails() throws if called outside PartnerProvider
```

### Rules

- `partnerName` is an empty string until the query resolves; do not use it as a loading check — use `isLoading`.
- Catalog page guards the infinite query with `enabled: isPartnerDetailsReady && !!region && !!filters.currency`. Never fire price list queries before partner details are loaded.

---

## Initialization Order & Boot Dependency Chain

```
_app.tsx mounts
│
├─ QueryClientProvider (TanStack Query — must be outermost)
│   │
│   ├─ PartnerContext (fires GET /api/partnerDetails on mount)
│   │   └─ Provides: partnerName, currencies, region, marketSegments
│   │
│   └─ CartContext (hydrates from localStorage on mount)
│       └─ Provides: cartItemIdToQuantityMap, customerInfoInCart
│
└─ Pages render
    └─ Catalog: waits for PartnerContext.isLoading === false before firing pricelist query
       └─ Guard expression: enabled: isPartnerDetailsReady && !!region && !!filters.currency
```

Enforcement is **manual guard** — pages check `isLoading` or derived readiness flags, not a centralized boot orchestrator.

---

## Cross-Store Dependencies

| Dependent | Requires | Required field | Behavior while waiting |
|---|---|---|---|
| Catalog page (pricelist query) | PartnerContext | `region`, `availableCurrencies` | Shows "Loading partner information…" skeleton; pricelist query disabled |
| Checkout page (order preview) | CartContext | `customerInfoInCart.customerId` | Recalculation skipped; `handleRecalculate` returns early |
| CustomerDetails page (subscriptions) | Next.js router | `customerId` query param | Query disabled via `enabled: !!customerIdString && router.isReady` |

---

## Cache Strategy

TanStack Query cache settings by domain:

| Domain | staleTime | gcTime | Notes |
|---|---|---|---|
| Partner details | 24 hours | 7 days | Rarely changes; intentionally long |
| Pricelist (infinite) | 5 minutes | 10 minutes | Invalidated on currency change |
| Subscription pricing | 10 minutes | 30 minutes | Relatively stable within a session |
| Customers (list) | 5 minutes | 10 minutes | Prefetches adjacent pages |
| Customers (search) | 5 minutes | 10 minutes | |
| Resellers (list) | 5 minutes | 10 minutes | |
| Customer details | 0 (always fresh) | 0 | Stale-time 0 — always refetched on mount |
| Subscriptions | 0 (always fresh) | 0 | Stale-time 0 + 8s polling refetch on details page |
| Reseller details | 2 minutes | 5 minutes | |
| Checkout reseller lookup | 10 minutes | 10 minutes | |
