# UI_CODE_PATTERNS — Bridge

---

## Part A — General UI Standards

### Naming
- Components in **PascalCase**; file names match the component name exactly (`CatalogProductCard.tsx` exports `CatalogProductCard`).
- Hooks in **camelCase** with `use` prefix (`usePaginatedCustomersList`, `useCartRouteGuard`).
- No barrel re-exports (`index.ts` files) — components are imported by direct path.
- Utility functions in **camelCase**; utility files in **camelCase** (`commonUtils.ts`, `cartUtils.ts`).

### Component design
- Single responsibility — each component handles one concern.
- Props interface declared at the top of the file with explicit types (no inline `{prop: type}` in the signature).
- CSS Modules (`*.module.css`) for all component styling — no inline styles, no CSS-in-JS.
- Shared styles go in `styles/CommonCard.module.css` for cross-page card/grid patterns.

### Async patterns
- Data fetching belongs in TanStack Query hooks or `utils/AccountApis.ts` fetch functions — not in component render bodies.
- Always handle three states: `isLoading`, `error`, and the empty/no-data case.
- `useEffect` for side effects only (timers, observer setup, navigation guards) — not for derived data.

### TypeScript
- No `any`; when unavoidable (e.g. TanStack Query error type), annotate with `as Error | null`.
- Use Zod models in `models/` for runtime validation of API responses.
- Prefer inferred types from fetch responses over hand-written interfaces. If typing a response manually, keep it in `types/` or the relevant `models/` file.

### Accessibility
- `aria-label` on all interactive elements that lack visible text (cart buttons, search fields, card `role="button"` divs).
- `tabIndex={0}` + `onKeyDown` handler (Enter/Space) on non-`<button>` interactive elements — see customer/reseller card pattern.
- Semantic HTML where possible; `<h1>` on page titles, `<header>` on the order confirmation header.

### Performance
- Infinite scroll via `IntersectionObserver` on catalog page — do not use a "Load More" button pattern.
- Icon preloading via `iconPreloader.preloadProductFamilyIcons` for the first 20 visible catalog items.
- TanStack Query `prefetchQuery` / `prefetchInfiniteQuery` for adjacent pages in all paginated lists.
- `useMemo` for derived/filtered arrays that are expensive to recompute.

### Error handling
- **Expected errors** (API returned 4xx, empty results): shown inline in the page with a descriptive message and a "Try Again" button.
- **Mutation errors**: surface via `ErrorToast` component, not inline.
- **Unexpected errors**: no global error boundary defined; Next.js default error page handles uncaught render errors.

### Test principles
- Test behavior (what the user sees), not implementation (internal state).
- Mock only at system boundaries: network (`fetch`), storage (`localStorage`), time.
- Do not assert on internal React state — assert on rendered output.

---

## Part B — Project-Specific (Strict) Conventions

### B.1 Naming Conventions

| Layer | Convention | Example |
|---|---|---|
| Pages | PascalCase component, camelCase file | `CatalogPage` in `pages/catalog.tsx` |
| Components | PascalCase component + file | `CatalogProductCard` in `components/CatalogProductCard.tsx` |
| Hooks | camelCase with `use` prefix | `usePaginatedCustomersList` in `hooks/usePaginatedCustomersList.ts` |
| Context | PascalCase provider + `use` hook | `CartProvider` / `useCart` in `contexts/CartContext.tsx` |
| Utils | camelCase function, camelCase file | `getSKUFromOfferId` in `utils/commonUtils.ts` |
| Models (Zod) | PascalCase schema + inferred type | `CustomerDetails` in `models/CustomerDetails.ts` |
| Types | PascalCase interface/type | `CreateOrderRequest` in `types/order.ts` |
| CSS Modules | PascalCase file, camelCase classes | `CatalogPage.module.css`, `styles.pageHeader` |
| Constants | SCREAMING_SNAKE_CASE object keys | `CUSTOMER_API_TYPE.GET_ALL_CUSTOMERS` |
| API route files | camelCase | `pages/api/customers.ts` |

### B.2 File / Directory Structure

```
bridge/
├── components/               # Reusable UI components
│   ├── Layout.tsx            # Page shell (header + sidebar + main)
│   ├── Header.tsx
│   ├── Sidebar.tsx
│   ├── NavigationPanel.tsx   # Breadcrumb navigation
│   ├── LoadingSkeleton.tsx   # Generic skeleton loader
│   ├── CatalogFilters.tsx    # Catalog filter sidebar
│   ├── CatalogProductCard.tsx
│   ├── UserProfileModal.tsx
│   ├── catalogCart/          # Cart-related components used in catalog + customer details
│   │   ├── CartDetails.tsx
│   │   └── FindOrCreateCustomer.tsx
│   ├── checkout/             # Checkout-specific components
│   │   ├── CheckoutReviewDetails.tsx
│   │   ├── CheckoutProductCard.tsx
│   │   └── CheckoutSummary.tsx
│   ├── common/               # Shared feature components
│   │   ├── CustomerSelector.tsx
│   │   ├── ResellerSelector.tsx
│   │   ├── PromoCodeButton.tsx
│   │   └── ...
│   ├── customerdetails/      # Customer details tab components
│   │   ├── AccountDetailsPanel.tsx
│   │   ├── CustomerProductsOverviewPanel.tsx
│   │   ├── PurchaseHistoryPanel.tsx
│   │   └── ClearCartConfirmationDialog.tsx
│   ├── renewal/              # Renewal workflow components
│   └── ThreeYearCommit/      # 3YC enrollment components
├── contexts/                 # React Context providers
│   ├── CartContext.tsx
│   └── PartnerContext.tsx
├── controllers/              # Server-side business logic (called by API routes)
│   ├── customerController.ts
│   ├── resellerController.ts
│   ├── orderController.ts
│   └── ...
├── hooks/                    # TanStack Query + custom React hooks
│   ├── usePaginatedCustomersList.ts
│   ├── usePaginatedResellersList.ts
│   ├── useOrderOperations.ts
│   ├── useProductPricing.ts
│   ├── useCartRouteGuard.ts
│   ├── useDebounce.ts
│   └── useToastState.ts
├── models/                   # Zod schemas (runtime validation + TypeScript types)
│   ├── Customer.ts
│   ├── CustomerDetails.ts
│   ├── Order.ts
│   ├── PriceList.ts
│   ├── Catalog.ts
│   └── ...
├── pages/                    # Next.js pages (file-based routes)
│   ├── _app.tsx              # Provider setup
│   ├── _document.tsx         # HTML shell
│   ├── index.tsx             # Redirect to /resellers
│   ├── resellers.tsx
│   ├── customers.tsx
│   ├── catalog.tsx
│   ├── checkout.tsx
│   ├── customerdetails.tsx
│   ├── orderConfirmation.tsx
│   └── api/                  # Next.js API routes (thin proxies to controllers)
│       ├── customers.ts
│       ├── resellers.ts
│       ├── pricelist.ts
│       ├── orders.ts
│       ├── subscriptions.ts
│       ├── search.ts
│       ├── recommendation.ts
│       ├── partnerDetails.ts
│       └── env.ts
├── styles/                   # CSS Modules
│   ├── globals.css
│   ├── CommonCard.module.css # Shared card/grid/pagination styles
│   └── *.module.css          # Page/component-specific styles
├── types/                    # TypeScript type definitions (non-Zod)
├── utils/                    # Pure helpers and API wrappers
│   ├── AccountApis.ts        # Client-side fetch functions → /api/* routes
│   ├── apiError.ts
│   ├── cartUtils.ts
│   ├── commonUtils.ts
│   ├── constants.ts          # HTTP methods, API type strings, route constants
│   ├── imsTokenService.ts    # Server-side only — IMS OAuth
│   ├── logger.ts
│   ├── prefetchProducts.ts   # Pricelist prefetch utilities
│   ├── queryUtils.ts         # TanStack Query key factories
│   └── ToastMessageUtils.tsx
├── lib/                      # Static data files
│   └── product_merchandising.json
└── tests/                    # Jest tests (integration + unit)
```

### B.3 Data Fetch Library Patterns

**Standard query (paginated list):**

```typescript
// hooks/usePaginatedCustomersList.ts
const { data, isLoading, error, isFetching, isPlaceholderData } =
  useQuery<PaginatedCustomersResponse>({
    queryKey: [...getCustomerListKey(resellerId, currentPage, itemsPerPage)],
    queryFn: () => fetchCustomersPage(resellerId, currentPage, itemsPerPage),
    enabled: enabled && !isSearchMode && !!resellerId,
    placeholderData: previousData => previousData,
    staleTime: 5 * 60 * 1000,
    gcTime: 10 * 60 * 1000,
  });
```

**Standard mutation:**

```typescript
// hooks/useOrderOperations.ts
export function useCreateOrder() {
  return useMutation({
    mutationFn: async (orderData: CreateOrderRequest) => {
      const response = await fetch('/api/orders?type=NEW', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(orderData),
      });
      if (!response.ok) {
        const error = new Error(`Request failed with status ${response.status}`) as any;
        error.status = response.status;
        throw error;
      }
      return response.json();
    },
    retry: (failureCount, error: any) => {
      if (failureCount >= 3) return false;
      if (error.status >= 500) return true;
      return false;
    },
    retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 10000),
  });
}
```

### B.4 State Library Patterns

**Store read:**

```typescript
// Any component inside CartProvider
const { cartItemIdToQuantityMap, getTotalItems } = useCart();
const totalItems = getTotalItems();
```

**Store write / action call:**

```typescript
const { addToCart, setShowCartModal } = useCart();

addToCart(offerId, productName, price, currency, marketSegmentCode, productFamily, 1);
setShowCartModal(true);
```

### B.5 Design System

**Import pattern:**

```typescript
import { Button, SearchField, Tabs, TabList, Tab, TabPanel } from '@react-spectrum/s2';
import ShoppingCart from '@react-spectrum/s2/icons/ShoppingCart';
import Search from '@react-spectrum/s2/icons/Search';
import ChevronRight from '@react-spectrum/s2/icons/ChevronRight';
import InfoCircle from '@react-spectrum/s2/icons/InfoCircle';
```

**Component usage:**

| Scenario | Spectrum component |
|---|---|
| Buttons | `Button` with `variant="accent"` / `"secondary"`, `fillStyle="outline"` |
| Search inputs | `SearchField` |
| Tab navigation | `Tabs` + `TabList` + `Tab` + `TabPanel` |
| Loading indicator | `ProgressCircle` |
| Page heading | `Heading` |
| Icons | Named imports from `@react-spectrum/s2/icons/*` |

**A11y provided by library:** ARIA roles, keyboard navigation, focus management for interactive Spectrum components.  
**A11y the project must add manually:** `aria-label` on non-Spectrum interactive elements (card `div` with `role="button"`, icon-only buttons), `tabIndex` + `onKeyDown` on clickable `div` elements.

### B.6 Form Patterns

⚠ NOT CAPTURED — No form components using `react-hook-form` + `zod` were found in the pages read during this scan. `react-hook-form` and `zod` are present in `package.json` and Zod models exist in `models/`, but the forms using them (likely in customer creation flows within `components/common/`) were not fully read. Check `components/common/CustomerSelector.tsx` and `utils/customerFormValidation.ts` for the pattern.

### B.7 Error Handling in Components

**Query error display:**

```tsx
// pages/customers.tsx
} : error ? (
  <div className={`${styles.stateMessage} ${styles.errorMessage}`}>
    <div>
      <h3>Unable to load customers</h3>
      <p>{error?.message || 'An unexpected error occurred'}</p>
      <Button variant="secondary" onPress={() => invalidateCustomers(queryClient)}>
        Try Again
      </Button>
    </div>
  </div>
```

**Mutation error display (ErrorToast):**

```tsx
// pages/checkout.tsx
const [showErrorToast, setShowErrorToast] = useState(false);
const [errorMessage, setErrorMessage] = useState('');

// In onError callback:
setErrorMessage(error?.message || 'Failed to create order. Please try again.');
setShowErrorToast(true);

// In JSX:
<ErrorToast
  show={showErrorToast}
  message={errorMessage}
  onClose={() => setShowErrorToast(false)}
/>
```