# LLD — Flexible Discounts (UI)

**Target repo:** bridge  
**Feature spec:** `feature-specs/flexible-discounts/experience-card/flexible-discounts.md`  
**Backend LLD:** `feature-specs/flexible-discounts/reference-files/LLD/backend/flexible-discounts-lld.md`  
**Service cards:** `feature-specs/flexible-discounts/reference-files/service-cards/ui/` (reference)

---

## §1 — Summary

**No Figma URL — LLD derived from experience card description only. All visual specs are approximate.**

This feature adds three surfaces for partner interaction with Adobe Flexible Discounts (flex discounts):

1. **Surface 1 — `/flexDiscounts` page:** A new full-page experience with a left-side filter panel and an infinite-scroll discount card grid. Partners can browse all discounts for their market, filter by country, segment, category, date range, status, and use-case tags, copy discount codes to clipboard, and inspect eligible products in a modal overlay.

2. **Surface 2 — Inline discovery in Checkout:** The existing `PromoCodeDialog` (opened via `PromoCodeButton` in `CheckoutProductCard`) is extended with a `FlexDiscountDiscoveryPanel` that surfaces offer-scoped discounts fetched from `/api/flex-discounts`. Partners click Apply on a row to insert the code into the text input; confirming triggers an order preview that reflects the discounted price.

3. **Surface 3 — Inline discovery in Edit Renewal:** The same `PromoCodeDialog` extension is used inside `EditRenewalDialog`, with `startDate=today` added to the fetch so only renewal-eligible discounts appear. Partners click Select to load the code, then Apply to confirm; on Save, the PATCH subscription call and renewal preview flow (already implemented) carry the code through to the backend.

The UI talks exclusively to the BFF via `/api/flex-discounts` (defined in the backend LLD). Order and subscription submission routes (`/api/orders`, `/api/subscriptions`) are unchanged — `flexDiscountCodes` fields already exist in those payloads.

---

## §2 — Data Flow

**Surface 1 — Partner browses the Flex Discounts page**

```
User action: Partner clicks "Flexible Discounts" in sidebar → /flexDiscounts loads
  → FlexDiscountsPage mounts; country = '' (empty text field), marketSegment defaults to PartnerContext.marketSegments[0]
  → one radio button rendered per entry in PartnerContext.marketSegments
  ← grid does not fetch until country is non-empty

User action: Partner types a country code in the Country text field (debounced 300ms)
  → debouncedCountry becomes non-empty → useFlexDiscounts({ country, marketSegment }) fires
  → GET /api/flex-discounts?type=getFlexDiscounts&country=US&market-segment=COM&include-eligible-reusable-discounts=true&limit=20&offset=0
  ← response: { flexDiscounts[], totalCount, count, offset, limit }
  ← hook merges pages into flat discounts array; exposes fetchNextPage, hasNextPage
  ← FlexDiscountsPage applies client-side filters (status, applicableTo)
  ← when filtered results == 0 AND hasNextPage == true: auto-calls fetchNextPage (continuous loading)
  ← component renders card grid OR ProgressCircle OR empty/error state

User action: Partner changes Country text field or Market Segment / Category / Date Range filters
  → server-side filters: query key changes → new useInfiniteQuery fires with updated params
  → client-side filters (Status, Applicable to): visibleDiscounts recomputed without a new fetch

User action: Partner scrolls near bottom of grid
  → IntersectionObserver callback fires → fetchNextPage() called
  → next batch merged into grid

User action: Partner clicks "N products eligible" button on a card
  → EligibleProductsPanel opens with that discount's qualification.baseOfferIds
  → useProductNamesFromPricelist called for each offerId to resolve names and prices
  ← panel renders product list; falls back to offerId if name unavailable
```

**Surface 2 — Partner applies a discount during checkout**

```
Context: checkout.tsx fetches customer detail on mount using customerId from CartContext
  → GET /api/customers?type=getCustomer&customerId=<id>
  ← customerCountry = customer.companyProfile.address.country
  ← discoveryParams = { country: customerCountry, marketSegment: product.marketSegment }
     passed to each CheckoutProductCard → PromoCodeButton → PromoCodeDialog

User action: Partner clicks "Apply discount code" on a CheckoutProductCard
  → PromoCodeButton.setIsDialogOpen(true)
  → PromoCodeDialog opens; FlexDiscountDiscoveryPanel mounts with country + marketSegment from discoveryParams
  → useFlexDiscounts({ country, marketSegment, 'offer-ids': offerId }) fires immediately
  → GET /api/flex-discounts?type=getFlexDiscounts&country=US&market-segment=COM&offer-ids=<offerId>&include-eligible-reusable-discounts=true&limit=10
  ← panel renders discount rows (name, reusable badge, value badges, date range, Apply button)
  (if customerCountry not yet resolved, panel is shown but fetch is gated — rows appear once country resolves)

User action: Partner clicks "Apply" on a discovery row
  → PromoCodeDialog.setPromoInput(discount.code) — code fills text input
  (partner can also type a code manually)

User action: Partner clicks "Apply" in dialog footer
  → PromoCodeDialog.onApply(offerId, code) fires → PromoCodeButton.handleApply → dialog closes
  ← CheckoutPage.handlePromoCodeApply sets items[i].discountCode = code
  ← CheckoutPage re-triggers fetchOrderPreview with flexDiscountCodes: [code] in lineItem
  → POST /api/orders?type=Preview
  ← if flexDiscounts[0].result == 'SUCCESS': updated price shown in checkout summary
  ← if result == 'FAILURE': price unchanged (no explicit error surfaced — spec behaviour)
```

**Surface 3 — Partner applies a discount during renewal editing**

```
Context: customerdetails.tsx already has customer loaded via customerDetailKeys.customer(id)
  ← customer.companyProfile.address.country passed to EditRenewalDialog as country prop
  ← customer.marketSegment passed to EditRenewalDialog as marketSegment prop
  ← EditRenewalDialog passes country + marketSegment + isRenewalContext=true to each PromoCodeButton → PromoCodeDialog

User action: Partner clicks "Apply discount code" on a product in EditRenewalDialog
  → PromoCodeButton.setIsDialogOpen(true)
  → PromoCodeDialog opens; FlexDiscountDiscoveryPanel mounts with country + marketSegment + startDate
  → useFlexDiscounts({ country, marketSegment, 'offer-ids': offerId, 'start-date': today }) fires
  → GET /api/flex-discounts?type=getFlexDiscounts&country=US&market-segment=COM&offer-ids=<offerId>&include-eligible-reusable-discounts=true&start-date=<today>
  ← panel renders renewal-eligible discount rows with "Select" button instead of "Apply"

User action: Partner clicks "Select" on a discovery row
  → PromoCodeDialog.setPromoInput(discount.code) — code fills input

User action: Partner clicks "Apply" in dialog
  → onApply(offerId, code) → EditRenewalDialog.handlePromoCodeApply(offerId, code)
  ← product.discountCode = code; product.pricePerUnit = 0; product.lineItemTotal = 0

User action: Partner clicks "Save Changes" in EditRenewalDialog
  → Promise.allSettled over updateRenewalSubscription(product, customerId) per changed product
  → PATCH /api/subscriptions with { autoRenewal: { enabled, renewalQuantity, flexDiscountCodes: [code] } }
  → fetchRenewalPreviewAndUpdateProducts fires
  → POST /api/orders?type=PREVIEW_RENEWAL with lineItems[].flexDiscountCodes: [code]
  ← if flexDiscounts[0].result == 'SUCCESS': product price updates to discounted value
  ← if result == 'FAILURE': code badge shown in negative (red) variant; price unchanged
  ← EditRenewalDialog shows updated EstimatedTotal per currency
```

---

## §3 — Change Summary

### Overview Table

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `pages/flexDiscounts.tsx` | CREATE | Route | New Flex Discounts page (Surface 1) |
| `components/flexDiscounts/FlexDiscountCard.tsx` | CREATE | Component | Discount card displayed in the grid |
| `components/flexDiscounts/FlexDiscountsFilterPanel.tsx` | CREATE | Component | Left-side filter panel |
| `components/flexDiscounts/EligibleProductsPanel.tsx` | CREATE | Component | Modal overlay showing eligible products |
| `components/flexDiscounts/FlexDiscountDiscoveryPanel.tsx` | CREATE | Component | Inline discovery panel inside PromoCodeDialog (Surfaces 2 & 3) |
| `hooks/useFlexDiscounts.ts` | CREATE | Data Fetch | Paginated/infinite TanStack Query hook for `/api/flex-discounts` |
| `types/flexDiscount.ts` | CREATE | Util | TypeScript interfaces for FlexDiscount response shapes |
| `styles/flexDiscounts/FlexDiscountsPage.module.css` | CREATE | Style | Page layout and filter panel styles |
| `styles/flexDiscounts/FlexDiscountCard.module.css` | CREATE | Style | Card layout, badge, and copy-chip styles |
| `styles/flexDiscounts/FlexDiscountDiscoveryPanel.module.css` | CREATE | Style | Inline discovery panel row styles |
| `components/Layout/Sidebar.tsx` | MODIFY | Component | Add Flex Discounts nav item; extend `ActivePage` union type |
| `components/common/PromoCodeDialog.tsx` | MODIFY | Component | Add `country`, `marketSegment`, `isRenewalContext` props; always render `FlexDiscountDiscoveryPanel` |
| `components/common/PromoCodeButton.tsx` | MODIFY | Component | Add `country`, `marketSegment`, `isRenewalContext` props; thread to `PromoCodeDialog` |
| `components/checkout/CheckoutProductCard.tsx` | MODIFY | Component | Add `country`, `marketSegment` props; thread to `PromoCodeButton` |
| `components/checkout/CheckoutSummary.tsx` | MODIFY | Component | Add `customerCountry` prop; pass `customerCountry` and `product.marketSegment` to each `CheckoutProductCard` |
| `pages/checkout.tsx` | MODIFY | Route | Fetch customer country; add `marketSegment` to cart item mapping; pass `customerCountry` to `CheckoutSummary` |
| `components/renewal/EditRenewalDialog.tsx` | MODIFY | Component | Accept `country` and `marketSegment` props; pass both plus `isRenewalContext` to each `PromoCodeButton` |
| `components/customerdetails/CustomerProductsOverviewPanel.tsx` | MODIFY | Component | Extract `customer.companyProfile.address.country`; pass `country` and `marketSegment` to `EditRenewalDialog` |
| `utils/constants.ts` | MODIFY | Util | Add `FLEX_DISCOUNT_API_TYPE` constants |

---

### `pages/flexDiscounts.tsx` — CREATE

Entry point for Surface 1. Uses `Layout` wrapper with `activePage="flexDiscounts"`.

**State variables:**

| Name | Type | Default | Re-fetches? |
|------|------|---------|-------------|
| `country` | `string` | `''` (partner types) | Yes — debounced 300ms |
| `marketSegment` | `string` | `PartnerContext.marketSegments[0]` | Yes |
| `category` | `'ALL' \| 'STANDARD' \| 'INTRO'` | `'ALL'` | Yes |
| `discountCode` | `string` | `''` | Yes — debounced 300ms |
| `startDate` | `string` | `''` | Yes |
| `endDate` | `string` | `''` | Yes |
| `statusFilter` | `'ALL' \| 'ACTIVE' \| 'UPCOMING'` | `'ALL'` | No — client-side |
| `applicableTo` | `string[]` | `[]` | No — client-side |
| `selectedDiscount` | `FlexDiscount \| null` | `null` | No |

**PartnerContext read:**

```typescript
const { marketSegments } = usePartnerDetails();
// useState initialized as: useState<string>(marketSegments[0] ?? '')
```

`marketSegments` is passed to `FlexDiscountsFilterPanel`; the panel renders one radio button per entry. The selected `marketSegment` state lives in the page.

**Data fetch:**

```typescript
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
  isLoading,
  error,
} = useFlexDiscounts(
  {
    country: debouncedCountry,
    marketSegment,
    categories: category !== 'ALL' ? category : undefined,
    'flex-discount-code': debouncedCode || undefined,
    'start-date': startDate || undefined,
    'end-date': endDate || undefined,
  },
  { enabled: !!debouncedCountry && !!marketSegment }
);

const allDiscounts: FlexDiscount[] = data?.pages.flatMap(p => p.flexDiscounts) ?? [];
```

**Client-side filtering:**

```typescript
const today = new Date().toISOString().split('T')[0];

const visibleDiscounts = allDiscounts.filter(d => {
  if (statusFilter === 'ACTIVE' && d.startDate > today) return false;
  if (statusFilter === 'UPCOMING' && d.startDate <= today) return false;
  if (applicableTo.length > 0) {
    const keywords: Record<string, string> = {
      'new-purchases': 'Add Seats',
      renewals: 'Renewal',
      '3yc': '3YC',
      upgrades: 'Switch',
    };
    const name = d.name?.toLowerCase() ?? '';
    const matches = applicableTo.some(k => name.includes(keywords[k]?.toLowerCase() ?? ''));
    if (!matches) return false;
  }
  return true;
});
```

**Continuous loading effect:** When `visibleDiscounts.length === 0` and `hasNextPage` and not currently fetching, call `fetchNextPage()`. Use a `useEffect` dependency on `[visibleDiscounts.length, hasNextPage, isFetchingNextPage]`.

**Infinite scroll:** Attach an `IntersectionObserver` to a sentinel `<div>` at the bottom of the grid. On intersection, call `fetchNextPage()` if `hasNextPage && !isFetchingNextPage`.

**Loading state:** Render `<ProgressCircle aria-label="Loading discounts" isIndeterminate />` when `isLoading`.

**Empty state:** `"No discounts available for {country} / {marketSegment}."` when `visibleDiscounts.length === 0 && !isLoading && !hasNextPage`.

**Error state:** Inline error `<div>` with `error.message`.

**Count display:** `"{allDiscounts.length} discount(s) available"` above the grid (updates as pages load).

**Layout:** Two-column flex: `FlexDiscountsFilterPanel` (fixed width ~280px) + scrollable card grid (flex-grow). Grid uses CSS grid with auto-fill columns (~340px min). Accessible via `<main>` landmark.

**EligibleProductsPanel wiring:**

State declaration:
```typescript
const [selectedDiscount, setSelectedDiscount] = useState<FlexDiscount | null>(null);
```

JSX usage (inside the card grid map and below it):
```tsx
<FlexDiscountCard
  discount={discount}
  onEligibleProductsClick={setSelectedDiscount}
/>

<EligibleProductsPanel
  isOpen={selectedDiscount !== null}
  onClose={() => setSelectedDiscount(null)}
  discount={selectedDiscount}
/>
```

---

### `components/flexDiscounts/FlexDiscountCard.tsx` — CREATE

Renders a single discount as a card. All visual specs are approximate (no Figma).

**Props:**

| Prop | Type | Required |
|------|------|----------|
| `discount` | `FlexDiscount` | Yes |
| `onEligibleProductsClick` | `(d: FlexDiscount) => void` | Yes |

**Rendered elements:**

- **Value badges:** One `<Badge fillStyle="bold" variant="informative">` per outcome value. Format:
    - `PERCENTAGE_DISCOUNT`: `"{value}% off"`
    - `FIXED_DISCOUNT`: `"${value} off · {currency}"`
    - `FIXED_PRICE`: `"${value} fixed price · {currency}"`

- **Reusable badge:** `<Badge variant="positive">Reusable</Badge>` when `discount.discountLockEndDate` is set.

- **Name + description:** `<Heading level={3}>{discount.name}</Heading>`. Description is truncated at 3 lines via CSS (`-webkit-line-clamp: 3; overflow: hidden`). A local `isExpanded: boolean` state controls a "Read more" / "Show less" `<button>` below the description.

- **Discount code chip:** A `<button>` styled as a chip showing `discount.code`. On click:
    1. `navigator.clipboard.writeText(code).catch(() => {})` — silent failure on clipboard block
    2. Local `copied: boolean` state → `true` → show `"Copied!"` text for 1500ms → reset

- **Validity dates:** `"Valid {startDate} – {endDate}"`. Include `"Reusable until {discountLockEndDate}"` if set.

- **Eligible products button:** If `discount.qualification?.baseOfferIds?.length > 0`, render `<Button variant="secondary">{N} products eligible</Button>` that calls `onEligibleProductsClick(discount)`. If no `baseOfferIds` or empty, render `"All products eligible"` as plain text.

---

### `components/flexDiscounts/FlexDiscountsFilterPanel.tsx` — CREATE

Left-side filter panel for Surface 1. All filter state is owned by `FlexDiscountsPage` and passed as controlled props.

**Props:**

| Prop | Type |
|------|------|
| `country` | `string` |
| `onCountryChange` | `(v: string) => void` |
| `marketSegment` | `string` |
| `onMarketSegmentChange` | `(v: string) => void` |
| `marketSegments` | `string[]` |
| `category` | `'ALL' \| 'STANDARD' \| 'INTRO'` |
| `onCategoryChange` | `(v: 'ALL' \| 'STANDARD' \| 'INTRO') => void` |
| `discountCode` | `string` |
| `onDiscountCodeChange` | `(v: string) => void` |
| `startDate` | `string` |
| `onStartDateChange` | `(v: string) => void` |
| `endDate` | `string` |
| `onEndDateChange` | `(v: string) => void` |
| `statusFilter` | `'ALL' \| 'ACTIVE' \| 'UPCOMING'` |
| `onStatusFilterChange` | `(v: 'ALL' \| 'ACTIVE' \| 'UPCOMING') => void` |
| `applicableTo` | `string[]` |
| `onApplicableToChange` | `(v: string[]) => void` |

**UI elements:**

- Country: `<TextField label="Country code" value={country} onChange={onCountryChange} />` (plain input; debounce handled in page)
- Market segment: one radio button per entry in `marketSegments` (sourced from `PartnerContext.marketSegments`) via `<div role="radiogroup">` + `<input type="radio">` styled elements or Spectrum equivalents; defaults to first entry on mount
- Category: three radio buttons (`All`, `Standard`, `Intro`)
- Discount code: `<SearchField>` with `{...({ placeholder: 'Search by code' } as any)}` TS workaround
- Date range: two `<TextField type="date">` inputs for start and end
- Status: three radio buttons (`All`, `Active`, `Upcoming`) — labeled as client-side filter
- Applicable to: four checkboxes (`New purchases`, `Renewals`, `3YC customers`, `Anytime upgrades`) — labeled as client-side filter

Section headings use `<Heading level={4}>` to distinguish server-side vs client-side filter groups.

---

### `components/flexDiscounts/EligibleProductsPanel.tsx` — CREATE

Modal overlay listing eligible products for a selected discount. Uses `createPortal` to `document.body` following the existing `PromoCodeDialog` pattern.

**Props:**

| Prop | Type | Required |
|------|------|----------|
| `isOpen` | `boolean` | Yes |
| `onClose` | `() => void` | Yes |
| `discount` | `FlexDiscount \| null` | Yes |

**Behaviour:**

- Backdrop click → `onClose()`
- Render: discount name, code chip, category badge, value badges at top
- Product list: for each `offerId` in `discount.qualification.baseOfferIds`, call `useProductNamesFromPricelist` to resolve name and standard price. Show: product name, standard price, and computed discounted price.
- If `useProductNamesFromPricelist` returns no name for an offer, display the raw `offerId` in its place.
- `role="dialog"`, `aria-modal="true"`, `aria-labelledby="eligible-products-title"`

**Runtime price calculation** — apply outcome values to the standard price client-side (no preview call needed):

| Outcome type | Calculation |
|---|---|
| `PERCENTAGE_DISCOUNT` | `discountedPrice = standardPrice * (1 - value / 100)` |
| `FIXED_DISCOUNT` | `discountedPrice = standardPrice - value` |
| `FIXED_PRICE` | `discountedPrice = value` |

When multiple outcome values exist for a product (e.g. country-specific values), match on `outocmeValue.country === product.country` if present; otherwise use the first value. Display both the standard price (struck through) and the discounted price.

---

### `components/flexDiscounts/FlexDiscountDiscoveryPanel.tsx` — CREATE

Inline panel rendered inside `PromoCodeDialog` (below the promo code text input) for Surfaces 2 & 3. Not shown on the Flex Discounts page.

**Props:**

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `country` | `string` | Yes | Customer's country (from customer record — not a text field) |
| `marketSegment` | `string` | Yes | Customer's market segment |
| `offerId` | `string` | Yes | Filters results to this offer |
| `startDate` | `string \| undefined` | No | ISO date string; set to today for renewal context |
| `onSelectCode` | `(code: string) => void` | Yes | Called when partner clicks Apply/Select on a row |
| `isRenewalContext` | `boolean` | No | When true, row CTA label is "Select"; otherwise "Apply" |

**Data fetch:**

```typescript
const { data, fetchNextPage, hasNextPage, isFetchingNextPage, isLoading, error } =
  useFlexDiscounts(
    { country, marketSegment, 'offer-ids': offerId, 'start-date': startDate },
    { enabled: !!country && !!marketSegment && !!offerId }
  );

const discounts = data?.pages.flatMap(p => p.flexDiscounts) ?? [];
```

**Loading state:** `<ProgressCircle size="S" aria-label="Loading discounts" isIndeterminate />`

**Empty / error state:** `"No discounts available for this product."`

**Discount row elements:**
- Discount name
- Reusable badge (`<Badge variant="positive">Reusable</Badge>`) when `discountLockEndDate` set
- Value badges (same format as `FlexDiscountCard`)
- Validity date range (compact: `"{startDate} – {endDate}"`)
- CTA button: `<Button variant="secondary" size="S" isDisabled={!discount.code} onPress={() => onSelectCode(discount.code!)}>` — label: `"Select"` if `isRenewalContext`, else `"Apply"`

**Load more button:** `<Button variant="secondary" size="S" onPress={() => fetchNextPage()} isDisabled={isFetchingNextPage}>` rendered when `hasNextPage`. Text: `"Load more"`.

---

### `hooks/useFlexDiscounts.ts` — CREATE

TanStack `useInfiniteQuery` wrapper for `GET /api/flex-discounts?type=getFlexDiscounts`.

**Query key:**
```typescript
['flexDiscounts', params.country, params.marketSegment, params['categories'], params['offer-ids'],
 params['flex-discount-code'], params['start-date'], params['end-date']]
```

**Config:**

| Field | Value |
|-------|-------|
| staleTime | 2 minutes |
| gcTime | 5 minutes |
| retry | 2 |
| initialPageParam | 0 |
| getNextPageParam | `(lastPage) => lastPage.offset + lastPage.count < lastPage.totalCount ? lastPage.offset + lastPage.count : undefined` |

**Fetch function:**

```typescript
const PAGE_SIZE = 20; // 10 for discovery panel — callers override via params.limit

async function fetchFlexDiscounts(params: FlexDiscountsQueryParams, offset: number) {
  const qs = new URLSearchParams({
    type: FLEX_DISCOUNT_API_TYPE.GET_FLEX_DISCOUNTS,
    'market-segment': params.marketSegment,
    country: params.country,
    'include-eligible-reusable-discounts': 'true',
    limit: String(params.limit ?? PAGE_SIZE),
    offset: String(offset),
    ...(params.categories && { categories: params.categories }),
    ...(params['offer-ids'] && { 'offer-ids': params['offer-ids'] }),
    ...(params['flex-discount-code'] && { 'flex-discount-code': params['flex-discount-code'] }),
    ...(params['start-date'] && { 'start-date': params['start-date'] }),
    ...(params['end-date'] && { 'end-date': params['end-date'] }),
  });

  const response = await fetch(`/api/flex-discounts?${qs}`);
  if (!response.ok) throw new Error('Failed to fetch flex discounts');
  return response.json() as Promise<FlexDiscountsApiResponse>;
}
```

**Exported signature:**

```typescript
export function useFlexDiscounts(
  params: FlexDiscountsQueryParams,
  options?: { enabled?: boolean }
): UseInfiniteQueryResult<InfiniteData<FlexDiscountsApiResponse>, Error>
```

`FlexDiscountsQueryParams` and `FlexDiscountsApiResponse` are defined in `types/flexDiscount.ts`.

---

### `types/flexDiscount.ts` — CREATE

```typescript
export interface FlexDiscountOutcomeValue {
  country?: string;
  currency?: string;
  value: number;
}

export interface FlexDiscountOutcome {
  type: 'FIXED_PRICE' | 'FIXED_DISCOUNT' | 'PERCENTAGE_DISCOUNT';
  discountValues: FlexDiscountOutcomeValue[];
}

export interface FlexDiscountQualification {
  baseOfferIds: string[];
}

export interface FlexDiscount {
  id?: string;
  category?: string;
  code?: string;
  name?: string;
  description?: string;
  startDate?: string;
  endDate?: string;
  discountLockEndDate?: string;
  status?: string;
  qualification?: FlexDiscountQualification;
  outcomes?: FlexDiscountOutcome[];
}

export interface FlexDiscountsApiResponse {
  limit: number;
  offset: number;
  count: number;
  totalCount: number;
  flexDiscounts: FlexDiscount[];
}

export interface FlexDiscountsQueryParams {
  country: string;
  marketSegment: string;
  limit?: number;
  categories?: string;
  'offer-ids'?: string;
  'flex-discount-code'?: string;
  'start-date'?: string;
  'end-date'?: string;
}

```

---

### `styles/flexDiscounts/FlexDiscountsPage.module.css` — CREATE

```css
.pageContainer {
  display: flex;
  height: 100%;
  gap: 24px;
  padding: 24px;
  overflow: hidden;
}

.filterPanel {
  width: 280px;
  flex-shrink: 0;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.filterSection {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.filterLabel {
  font-size: 12px;
  font-weight: 600;
  color: #505050;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.gridArea {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.countLabel {
  font-size: 14px;
  color: #505050;
}

.discountGrid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 16px;
}

.loadingState {
  display: flex;
  justify-content: center;
  padding: 40px;
}

.emptyState {
  padding: 40px;
  text-align: center;
  color: #505050;
}

.errorState {
  padding: 24px;
  color: #e34850;
}

.sentinel {
  height: 1px;
}
```

---

### `styles/flexDiscounts/FlexDiscountCard.module.css` — CREATE

```css
.card {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.badgeRow {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  align-items: center;
}

.nameRow {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.description {
  font-size: 14px;
  color: #505050;
  overflow: hidden;
  display: -webkit-box;
  -webkit-box-orient: vertical;
}

.descriptionCollapsed {
  -webkit-line-clamp: 3;
}

.descriptionExpanded {
  -webkit-line-clamp: unset;
}

.readMoreButton {
  background: none;
  border: none;
  color: #0066cc;
  cursor: pointer;
  font-size: 14px;
  padding: 0;
  font-family: inherit;
}

.codeChip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: #f5f5f5;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  padding: 4px 10px;
  font-size: 13px;
  font-family: monospace;
  cursor: pointer;
  color: #131313;
  transition: background 0.1s;
}

.codeChip:hover {
  background: #ebebeb;
}

.codeChipCopied {
  color: #12805c;
}

.datesRow {
  font-size: 13px;
  color: #767676;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.footerRow {
  display: flex;
  justify-content: flex-end;
}
```

---

### `styles/flexDiscounts/FlexDiscountDiscoveryPanel.module.css` — CREATE

```css
.panel {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding-top: 16px;
  border-top: 1px solid #e0e0e0;
  max-height: 280px;
  overflow-y: auto;
}

.panelHeading {
  font-size: 13px;
  font-weight: 600;
  color: #505050;
  margin-bottom: 4px;
}

.row {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 10px 0;
  border-bottom: 1px solid #f0f0f0;
}

.rowHeader {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.rowName {
  font-size: 14px;
  font-weight: 600;
  color: #131313;
}

.rowMeta {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  align-items: center;
}

.rowDates {
  font-size: 12px;
  color: #767676;
}

.loadingState {
  display: flex;
  justify-content: center;
  padding: 16px;
}

.emptyState {
  font-size: 14px;
  color: #767676;
  text-align: center;
  padding: 12px;
}

.loadMoreRow {
  display: flex;
  justify-content: center;
  padding-top: 8px;
}
```

---

### `components/Layout/Sidebar.tsx` — MODIFY

Two changes:

1. **Extend `ActivePage` type:** Add `'flexDiscounts'` to the union.

```typescript
// Before:
export type ActivePage = 'resellers' | 'catalog' | 'customers' | 'checkout';

// After:
export type ActivePage = 'resellers' | 'catalog' | 'customers' | 'checkout' | 'flexDiscounts';
```

2. **Add nav item to `navigationItems` array** after the existing `catalog` entry:

```typescript
{
  id: 'flexDiscounts' as ActivePage,
  label: 'Discounts',
  icon: (isActive: boolean) => (
    <div className={`${styles.largeIcon} ${isActive ? styles.iconColorActive : styles.iconColorInactive}`}>
      <Offer />  {/* import Offer from '@react-spectrum/s2/icons/Offer' */}
    </div>
  ),
  path: '/flexDiscounts',
},
```

Add the import at the top:
```typescript
import Offer from '@react-spectrum/s2/icons/Offer';
```

> **Note:** If `Offer` is not available in `@react-spectrum/s2/icons`, use `Tag` or `Discount` — check available icons with `import * as Icons from '@react-spectrum/s2/icons'`. The exact icon is approximate.

---

### `components/common/PromoCodeDialog.tsx` — MODIFY

Add `country`, `marketSegment`, and `isRenewalContext` props. Always render `FlexDiscountDiscoveryPanel` between the code input and the error text — discovery is not conditional.

**New props added to `PromoCodeDialogProps`:**

```typescript
country: string;
marketSegment: string;
isRenewalContext?: boolean;
```

**New import:**
```typescript
import { FlexDiscountDiscoveryPanel } from '../flexDiscounts/FlexDiscountDiscoveryPanel';
```

**New handler inside the component:**

```typescript
const handleDiscoverySelect = (code: string) => {
  setPromoInput(code);
};
```

**JSX change** — add discovery panel between `codeInput` div and `errorText` div (always rendered):

```tsx
<FlexDiscountDiscoveryPanel
  country={country}
  marketSegment={marketSegment}
  offerId={offerId}
  startDate={isRenewalContext ? new Date().toISOString().split('T')[0] : undefined}
  onSelectCode={handleDiscoverySelect}
  isRenewalContext={isRenewalContext}
/>
```

**Dialog width change:** Update `.dialog` max-width in `PromoCode.module.css` from `400px` to `520px` unconditionally — the discovery panel is always present so the wider size is always appropriate.

---

### `components/common/PromoCodeButton.tsx` — MODIFY

Thread `country`, `marketSegment`, and `isRenewalContext` to `PromoCodeDialog`.

**New props added to `PromoCodeButtonProps`:**

```typescript
country: string;
marketSegment: string;
isRenewalContext?: boolean;
```

**In JSX**, add to the `<PromoCodeDialog>` element:

```tsx
country={country}
marketSegment={marketSegment}
isRenewalContext={isRenewalContext}
```

---

### `components/checkout/CheckoutProductCard.tsx` — MODIFY

Thread `country` and `marketSegment` to `PromoCodeButton`.

**New props:**

```typescript
country: string;
marketSegment: string;
```

**In JSX**, add to `<PromoCodeButton>`:

```tsx
country={country}
marketSegment={marketSegment}
isRenewalContext={false}
```

---

### `pages/checkout.tsx` — MODIFY

Fetch the customer's country; add `marketSegment` to the cart item mapping; pass `customerCountry` down to `CheckoutSummary`.

**New inline query** — `customerId` is already declared later in the file as `const customerId = customerInfoInCart.customerId`, so use a different binding here to avoid a block-scoped conflict:

```typescript
const cartCustomerId = customerInfoInCart.customerId;

const { data: customerData } = useQuery({
  queryKey: ['customer', cartCustomerId],
  queryFn: async () => {
    const res = await fetch(`/api/customers?type=getCustomer&customerId=${cartCustomerId}`);
    if (!res.ok) throw new Error('Failed to fetch customer');
    return res.json();
  },
  enabled: !!cartCustomerId && router.isReady,
  staleTime: 10 * 60 * 1000,
  gcTime: 10 * 60 * 1000,
});

const customerCountry: string = customerData?.companyProfile?.address?.country ?? '';
```

**Add `marketSegment` to the cart item mapping `useEffect`** — the existing mapping omits it, so `product.marketSegment` would be `undefined` and the discovery panel fetch would never fire:

```typescript
// Inside the useEffect that builds checkoutItems:
return {
  id: (index + 1).toString(),
  offerId: offerId,
  productName: details?.productName || offerId,
  quantity: quantity,
  pricePerUnit: details?.pricePerUnit || 0,
  currency: details?.currency || '',
  productFamily: details?.productFamily,
  discountCode: details?.discountCode,
  marketSegment: details?.marketSegment,  // ← add this line
};
```

**Pass `customerCountry` to `CheckoutSummary`** — `CheckoutProductCard` is not rendered directly in this page; it is rendered inside `CheckoutSummary`. Pass the country there and let `CheckoutSummary` thread it per-product:

```tsx
<CheckoutSummary
  // ... existing props ...
  customerCountry={customerCountry}
/>
```

`customerCountry` defaults to `''` while the fetch is in progress. `FlexDiscountDiscoveryPanel`'s fetch is gated on `!!country && !!marketSegment && !!offerId` — so no fetch fires until both values resolve.

---

### `components/checkout/CheckoutSummary.tsx` — MODIFY

Add `customerCountry` prop and thread it with each product's `marketSegment` to `CheckoutProductCard`.

**New prop added to `CheckoutSummaryProps`:**

```typescript
customerCountry?: string;
```

**Destructure with default:**

```typescript
customerCountry = '',
```

**In the product render loop**, pass `country` and `marketSegment` to `CheckoutProductCard`:

```tsx
<CheckoutProductCard
  key={product.id}
  product={product}
  // ... existing props ...
  country={customerCountry}
  marketSegment={product.marketSegment ?? ''}
/>
```

---

### `components/renewal/EditRenewalDialog.tsx` — MODIFY

Accept `country` and `marketSegment` props; pass both plus `isRenewalContext` flat to each `PromoCodeButton`. `startDate` is computed inside `PromoCodeDialog` from `isRenewalContext`.

**New props added to `EditRenewalDialogProps`:**

```typescript
country: string;
marketSegment: string;
```

**In the product render loop**, update `<PromoCodeButton>`:

```tsx
<PromoCodeButton
  offerId={product.offerId}
  productName={productName}
  appliedCode={product.discountCode}
  onApply={handlePromoCodeApply}
  onRemove={handlePromoCodeRemove}
  isLoading={isUpdatingPrices}
  disabled={isSaving}
  country={country}
  marketSegment={marketSegment}
  isRenewalContext={true}
/>
```

---

### `components/customerdetails/CustomerProductsOverviewPanel.tsx` — MODIFY

> **Note:** The LLD originally targeted `pages/customerdetails.tsx`, but `EditRenewalDialog` is rendered inside `CustomerProductsOverviewPanel`, not in the page itself. Modify this component instead.

The component already receives the `customer` prop of type `CustomerDetails`. Extract country from `companyProfile.address.country` (correctly typed — no cast required) and pass it alongside the existing `marketSegment` to `EditRenewalDialog`.

**Add country extraction** (alongside the existing `marketSegment` line):

```typescript
const marketSegment = customer?.companyProfile?.marketSegment;
const customerCountry: string = customer?.companyProfile?.address?.country ?? '';
```

**Update `<EditRenewalDialog>` JSX:**

```tsx
<EditRenewalDialog
  isOpen={isEditRenewalDialogOpen}
  onClose={() => setIsEditRenewalDialogOpen(false)}
  subscriptions={subscriptions}
  customerId={customerId}
  anniversaryDate={anniversaryDate}
  country={customerCountry}
  marketSegment={marketSegment ?? ''}
/>
```

---

### `utils/constants.ts` — MODIFY

Add after the last `*_API_TYPE` block (after `SUBSCRIPTION_API_TYPE`):

```typescript
export const FLEX_DISCOUNT_API_TYPE = {
  GET_FLEX_DISCOUNTS: 'getFlexDiscounts',
  GET_CUSTOMER_FLEX_DISCOUNTS: 'getCustomerFlexDiscounts',
} as const;
```

---

## §4 — Design Decisions

| Decision | Why | Trade-off | Enforcement |
|----------|-----|-----------|-------------|
| Single `useFlexDiscounts` hook for all three surfaces | All surfaces call the same BFF endpoint with the same response shape; code deduplication. | The hook's query key includes all filter params, so calling it with different params (page vs. dialog) creates separate cache entries — correct isolation with no extra logic. | Naming convention: all callers import from `hooks/useFlexDiscounts.ts`. |
| `country` fetched via inline `useQuery` in `checkout.tsx` rather than adding it to `CartContext` | `CartContext` has `customerId` but not `country`; adding it there would require propagating through the add-to-cart flow. The inline query uses the same pattern as the reseller breadcrumb in `customers.tsx`. | Extra GET `/api/customers` call on the checkout page; mitigated by 10min staleTime (data likely already cached from `customerdetails.tsx`). | Pattern matches the existing inline `useQuery` for reseller detail in `customers.tsx`. |
| Discovery panel fetch gated on `!!country` (shows placeholder until country resolves) | Country arrives asynchronously in checkout. Rather than hiding the panel, show it immediately and let the hook's `enabled` guard prevent a premature fetch. | Partner sees the panel frame instantly; discount rows appear ~1–2s later once customer data resolves. | `enabled: !!country && !!marketSegment && !!offerId` in `useFlexDiscounts`. |
| Continuous loading is implemented in `FlexDiscountsPage` (not in the hook) | The hook should remain a general data-fetching primitive; auto-advance-on-empty-results is specific to the full-page browse surface. Dialog panels use explicit "Load more" instead. | `useEffect` watching `visibleDiscounts.length` could loop if many server pages all fail the client filter. Guard with a `maxAutoLoadPages` counter (suggest: 10). | Convention only — document in the component. |

---

## §5 — Acceptance Criteria Coverage

| Goal (from experience card) | Covered by |
|-----------------------------|-----------|
| Partners can quickly discover which discounts are available for a given country and market segment | `pages/flexDiscounts.tsx` + `hooks/useFlexDiscounts.ts` + `utils/constants.ts` |
| Partners can narrow results to discounts relevant to a specific use case (new purchases, renewals, upgrades, 3YC) | `pages/flexDiscounts.tsx` (client-side `applicableTo` filter) + `components/flexDiscounts/FlexDiscountsFilterPanel.tsx` |
| Partners can copy a discount code and understand exactly which products and pricing it applies to | `components/flexDiscounts/FlexDiscountCard.tsx` (copy chip) + `components/flexDiscounts/EligibleProductsPanel.tsx` |
| During checkout, partners can discover and apply an eligible discount without leaving the order flow | `components/flexDiscounts/FlexDiscountDiscoveryPanel.tsx` + `components/common/PromoCodeDialog.tsx` (MODIFY) + `pages/checkout.tsx` (MODIFY) |
| During renewal editing, partners can discover and apply renewal-eligible discounts per product, and see auto-applied reusable discounts before confirming | `components/flexDiscounts/FlexDiscountDiscoveryPanel.tsx` + `components/renewal/EditRenewalDialog.tsx` (MODIFY) + `pages/customerdetails.tsx` (MODIFY) |

All five acceptance criteria are covered. Two notes:

- **"See auto-applied reusable discounts before confirming":** The experience card implies that reusable discounts (those with `discountLockEndDate`) may be automatically applied. The current LLD surfaces reusable discounts in the discovery panel with a "Reusable" badge. If auto-application (pre-filling the code without partner action) is required, that behaviour is not specified in sufficient detail to implement here — mark as a follow-up clarification with product.

- **"Filtering by date range":** Covered by the `startDate`/`endDate` filter fields in `FlexDiscountsFilterPanel`. These are server-side filters forwarded to the BFF.

---

## §6 — Out of Scope

- **Customer-scoped reusable discounts (`GET /api/flex-discounts?type=getCustomerFlexDiscounts`):** The backend LLD defines this operation. The experience card does not describe a dedicated surface for it, and the inline discovery panels use the market-scoped operation. This endpoint is available for future use.
- **BFF-level caching of discount results:** Per backend LLD §7 — not implemented.
- **Rate limiting / quota enforcement:** Per backend LLD §7 — handled upstream.
- **Test files:** Integration and component test skeletons for new hooks and components are not included in this LLD. Follow patterns in `tests/` directory using `jest.mock` for TanStack Query and contexts.