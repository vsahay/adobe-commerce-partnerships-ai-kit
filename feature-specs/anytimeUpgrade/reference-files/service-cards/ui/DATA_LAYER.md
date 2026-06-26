# DATA_LAYER — Bridge

## Overview

| Property | Value |
|---|---|
| Data fetch library | TanStack Query v5 (`@tanstack/react-query@5.77.2`) + native `fetch` |
| HTTP client | Native `fetch` (no Axios on the client side; Axios is server-side only) |
| Auth injection | **Option B — store-internal:** Server-side controllers acquire an IMS OAuth token via `utils/imsTokenService.ts` before calling VIP MP APIs. The browser never touches the token. |
| Data freshness policy | See `STATE_MANAGEMENT.md §Cache Strategy` |
| Cache lifetime policy | See `STATE_MANAGEMENT.md §Cache Strategy` |
| Error surface | Inline state variable (`setErrorMessage` + `ErrorToast`) for mutations; inline error message in page JSX for query errors |
| Optimistic update posture | **Not used** — no optimistic updates; mutations wait for server confirmation |
| SSR pre-fetching | Not used — CSR only |

---

## Fetch Layer Summary

| Domain | Fetch unit | Endpoint | Cache key pattern | Auth injection |
|---|---|---|---|---|
| Partner contract | `fetchPartnerDetails` | GET `/api/partnerDetails` | `['partnerDetails']` | Server-side (IMS token in controller) |
| Customers — list | `fetchCustomersPage` | GET `/api/customers?type=getAllCustomers&...` | `['customers','list', resellerId, page, limit]` | Server-side |
| Customers — search | `searchCustomers` | GET `/api/search?type=customer&...` | `['customers','search', resellerId, term, page, limit]` | Server-side |
| Customer detail | `fetchCustomerDetails` | GET `/api/customers?type=getCustomer&customerId=...` | `['customerDetails','customer', customerId]` | Server-side |
| Resellers — list | `fetchResellersPage` | GET `/api/resellers?type=getAllResellers&...` | `['resellers','list', page, limit]` | Server-side |
| Resellers — search | `searchResellers` | GET `/api/search?type=reseller&...` | `['resellers','search', term, page, limit]` | Server-side |
| Reseller detail | `getResellerDetails` | GET `/api/resellers?type=getResellerDetails&id=...` | `['reseller', resellerId]` | Server-side |
| Pricelist | `fetchPricelistPage` | POST `/api/pricelist` | `['pricelist-infinite', marketSegmentCode, currency]` | Server-side |
| Subscription pricing | `fetchProductPricing` | POST `/api/pricelist` | `['subscriptionPricing', offerId, region, currency, priceListType]` | Server-side |
| Subscriptions | `fetchCustomerSubscriptions` | GET `/api/subscriptions?...` | `['customerDetails','subscriptions', customerId]` | Server-side |
| Order preview | `fetchOrderPreview` (via `cartUtils`) | POST `/api/orders?type=Preview` | No cache — mutation | Server-side |
| Create order | `useCreateOrder` | POST `/api/orders?type=NEW` | No cache — mutation | Server-side |

For freshness/lifetime settings see `STATE_MANAGEMENT.md §Cache Strategy`.

---

## Standard Fetch Pattern

```typescript
// utils/AccountApis.ts — fetchCustomersPage
export const fetchCustomersPage = async (
  resellerId: string,
  page: number,
  itemsPerPage: number
): Promise<PaginatedCustomersResponse> => {
  const offset = (page - 1) * itemsPerPage;
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 30000); // [2] dependency guard: 30s timeout

  try {
    // [1] Auth injection: none on client — server-side controller handles IMS token
    const url = `/api/customers?type=${CUSTOMER_API_TYPE.GET_ALL_CUSTOMERS}&resellerId=${resellerId}&offset=${offset}&limit=${itemsPerPage}`;

    const response = await fetch(url, { signal: controller.signal });
    const responseText = await response.text();

    if (!response.ok) {
      throw new Error(`Failed to fetch customers: ${response.status} ${responseText}`);
    }

    let data: unknown;
    try {
      data = JSON.parse(responseText);
    } catch (parseError) {
      throw new Error('Invalid API response format');
    }

    if (!data || typeof data !== 'object' || !Array.isArray((data as any).customers)) {
      throw new Error('Invalid response format: expected paginated customers response');
    }

    // [3] Return shape: { customers: CustomerMinimal[], totalCount, count, offset, limit, hasMore }
    return data as PaginatedCustomersResponse;
  } finally {
    clearTimeout(timeoutId);
  }
};
```

Used in TanStack Query:

```typescript
// hooks/usePaginatedCustomersList.ts
const { data } = useQuery<PaginatedCustomersResponse>({
  queryKey: [...getCustomerListKey(resellerId, currentPage, itemsPerPage)],
  queryFn: () => fetchCustomersPage(resellerId, currentPage, itemsPerPage),
  enabled: enabled && !isSearchMode && !!resellerId,  // [2] dependency guard
  placeholderData: previousData => previousData,
  staleTime: STALE_TIME,
  gcTime: GC_TIME,
});
```

---

## Standard Mutation Pattern

```typescript
// hooks/useOrderOperations.ts — useCreateOrder
export function useCreateOrder() {
  return useMutation({
    // [1] payload → async POST call
    mutationFn: async (orderData: CreateOrderRequest) => {
      const response = await fetch('/api/orders?type=NEW', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(orderData),
      });

      if (!response.ok) {
        const errorText = await response.text();
        let errorMessage = `Request failed with status ${response.status}`;
        try {
          const errorData = JSON.parse(errorText);
          errorMessage = JSON.stringify(errorData);
        } catch {
          errorMessage = errorText || errorMessage;
        }
        const error = new Error(errorMessage) as any;
        error.status = response.status;
        throw error;
      }

      return response.json();
    },

    // [2] Cache update after success: none — caller clears cart and navigates to /orderConfirmation
    // No query invalidation needed because the order list is not displayed on the same page.

    // [3] Error surface: thrown error caught by onError callback in pages/checkout.tsx
    retry: (failureCount, error: any) => {
      if (failureCount >= 3) return false;
      if (error.status >= 500) return true;
      if (error.message?.includes('fetch') || error.message?.includes('network')) return true;
      return false;
    },
    retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 10000),
  });
}
```

Called in `pages/checkout.tsx`:

```typescript
createOrder.mutate(orderData, {
  onSuccess: () => {
    clearCart();
    router.push(`/orderConfirmation?${queryParams.toString()}`);
  },
  onError: (error: any) => {
    setErrorMessage(error?.message || 'Failed to create order. Please try again.');
    setShowErrorToast(true);
    createOrder.reset();
  },
});
```

---

## Auth Token Injection — Option B (Store-internal / server-side)

The browser client never holds an IMS token. The token is acquired and used exclusively on the server:

```
Browser → fetch('/api/orders', ...) → pages/api/orders.ts → orderController.ts
                                                                    ↓
                                           utils/imsTokenService.ts → POST IMS_TOKEN_URL
                                                                    ↓
                                           controller adds: Authorization: Bearer <token>
                                                                    ↓
                                           fetch(PARTNER_API_BASE_URL + '/...', { headers })
```

`imsTokenService.ts` handles token acquisition (client credentials flow) and is called inside each controller before the external API call.

---

## Error Handling Posture

**Query errors** — shown inline in page JSX:

```tsx
// Example from pages/customers.tsx
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

**Mutation errors** — displayed via `ErrorToast` component:

```tsx
// Example from pages/checkout.tsx
<ErrorToast
  show={showErrorToast}
  message={errorMessage}
  onClose={() => setShowErrorToast(false)}
/>
```

---

## Loading States

Skeleton cards (CSS module skeletons) for list pages:

```tsx
// pages/customers.tsx
{isLoading && !isPlaceholderData ? (
  <LoadingSkeleton count={itemsPerPage} />
) : ...}
```

Inline skeleton cards for the catalog:

```tsx
// pages/catalog.tsx
{isLoading ? (
  Array.from({ length: 10 }).map((_, index) => (
    <div key={index} className={`${styles.catalogSkeletonCard} ${commonStyles.skeleton}`}>
      <div className={styles.catalogSkeletonIcon} />
      <div className={styles.catalogSkeletonTitle} />
      ...
    </div>
  ))
```

`placeholderData: previousData => previousData` is used on paginated queries to keep showing the previous page while the next page loads (stale-while-revalidate UX).

---

## Adding a New Data Fetch — Checklist

1. **Identify capability** — determine which page/component needs this data
2. **Name the fetch unit** — add a function to `utils/AccountApis.ts` (or create a domain-specific util) following the `fetchXxx` naming convention
3. **Define cache key** — add a key generator to `utils/queryUtils.ts` using array format `['domain', 'operation', ...params]`
4. **Set dependency guard** — set `enabled: !!requiredParam && router.isReady` in the query options
5. **Set freshness/lifetime** — choose `staleTime` and `gcTime` based on data volatility and add to `STATE_MANAGEMENT.md §Cache Strategy`
6. **Validate response shape** — add a Zod model in `models/` or at minimum validate the shape in the fetch function before returning
7. **Add to Fetch Layer Summary** — add a row to the table in this file (`DATA_LAYER.md`)
8. **Add to UI_MODULE_INDEX.md** — add the fetch unit to the relevant capability's row in §4
