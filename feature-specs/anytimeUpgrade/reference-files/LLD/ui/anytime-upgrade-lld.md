# UI LLD — Anytime Upgrade

**Feature:** `anytimeUpgrade`
**Source:** `feature-specs/anytimeUpgrade/experience-card/anytime-upgrade.md`
**Backend LLD:** `docs/ai-kit/LLD/backend/anytime-upgrade-lld.md`
**Target repo:** `/Users/ankitasaha/Documents/openSource/bridge`
**Design source:** Figma — `https://www.figma.com/design/c3rdN5VOmVVkwTMkb6pMYj/midterm` *(visual specs derived from experience card; Figma not fetched at time of authoring)*

---

## Section 1: Summary

The Anytime Upgrade UI extends the customer details product listing and adds a dedicated order summary page. On the product listing, an **Upgrade** button appears next to each subscription that has available upgrade paths (determined by `useOfferSwitchPaths`). Pressing it opens an `UpgradePathsDialog` where the user selects a target product. The user is then routed to `/checkout/upgrade`, which previews the prorated price on load, supports quantity editing (unless `switchType=FULL_ONLY`), allows promo code application, and places the switch order on confirmation. On success the user lands on the existing `/orderConfirmation` page.

### Unit Inventory

| Unit | Action | Location |
|------|--------|----------|
| `/checkout/upgrade` | CREATE | `pages/checkout/upgrade.tsx` |
| `UpgradeReviewDetails` | CREATE | `components/checkout/upgrade/UpgradeReviewDetails.tsx` |
| `UpgradePathsDialog` | CREATE | `components/customerdetails/UpgradePathsDialog.tsx` |
| `useOfferSwitchPaths` | CREATE | `hooks/useOfferSwitchPaths.ts` |
| `usePreviewSwitchOrder` | CREATE | `hooks/useOrderOperations.ts` |
| `usePlaceSwitchOrder` | CREATE | `hooks/useOrderOperations.ts` |
| `ActiveProducts` | MODIFY | `components/customerdetails/ActiveProducts.tsx` |
| `CheckoutProductCard` | MODIFY | `components/checkout/CheckoutProductCard.tsx` |

---

## Section 2: Route Changes

### `/checkout/upgrade` — NEW at `pages/checkout/upgrade.tsx`

**Why:** The upgrade order summary is a distinct page with its own preview/place-order lifecycle, separate from the cart-based `/checkout` flow.

| Aspect | Detail |
|--------|--------|
| Entry point | `router.push('/checkout/upgrade?...')` called from `UpgradePathsDialog.onSelectUpgrade` |
| Exit — success | `/orderConfirmation` |
| Exit — cancel | Browser back button; no side effects |
| Auth guard | Global IMS wrapper — no additional per-page guard needed |
| Pre-req stores | None — all data flows from query params and API calls |

**Query params:**

| Param | Type | Required | Purpose |
|-------|------|----------|---------|
| `customerId` | `string` | Yes | Identifies the customer for the order |
| `resellerId` | `string` | Yes | Used for breadcrumb navigation and confirmation page |
| `sourceOfferId` | `string` | Yes | Current product offer ID (used for display name lookup) |
| `sourceSubscriptionId` | `string` | Yes | ID passed as `subscriptionId` in `cancellingItems` |
| `sourceCurrency` | `string` | Yes | `currencyCode` for all API calls |
| `targetOfferId` | `string` | Yes | Selected upgrade target offer ID |
| `switchType` | `string` | Yes | `FULL_ONLY` → quantity read-only; `PARTIAL_ALLOWED` → quantity editable |
| `quantity` | `string` | Yes | Current subscription quantity; used as default and max |

---

## Section 3: Component Specs

### New Components

#### `UpgradeReviewDetails` — CREATE at `components/checkout/upgrade/UpgradeReviewDetails.tsx`

**Why:** Left-column review panel specific to the upgrade flow; differs from `CheckoutReviewDetails` in that it shows a source → target product transition instead of cart contents.

**Props:**

| Prop | Type | Purpose |
|------|------|---------|
| `customerName` | `string` | Displayed in purchase disclosure |
| `resellerName` | `string` | Displayed in purchase disclosure |
| `sourceProductName` | `string` | Current product name |
| `targetProductName` | `string` | Target product name |

**Visual spec:**

| Element | Spec |
|---------|------|
| Shell classes | Reuses `checkoutStyles.reviewCard`, `reviewHeaderRowJustified`, `reviewTitle`, `reviewList`, `reviewListItem`, `reviewIcon` — same shell as `CheckoutReviewDetails` |
| Product transition row | Flex row, `gap: 12px`; each badge: icon (32×32, `border-radius: 6px`) + "Licenses" label; `→` separator between badges |
| Custom CSS classes | `.productTransition`, `.productBadge`, `.transitionIcon`, `.transitionLabel`, `.arrow` in `UpgradeReviewDetails.module.css` |

**States / Interactions / Accessibility:**

| Aspect | Detail |
|--------|--------|
| States | Happy path only — renders immediately from props; parent resolves names before rendering; no loading or error state in this component |
| Interactions | None — display only |
| Accessibility | `alt` set to product name; `onError` fallback to generic icon; `<section>` landmark element |

---

#### `UpgradePathsDialog` — CREATE at `components/customerdetails/UpgradePathsDialog.tsx`

**Why:** Standalone dialog for selecting an upgrade target; keeps `ActiveProducts` from growing further.

**Props:**

| Prop | Type | Purpose |
|------|------|---------|
| `product` | `ProductToDisplay` | Source product (display name + `currencyCode`) |
| `upgradePaths` | `{ targetBaseOfferId: string; switchType: string }[]` | Available upgrade targets |
| `onSelectUpgrade` | `(targetOfferId: string, switchType: string) => void` | Called when user confirms selection |

**Visual spec:**

| Element | Spec |
|---------|------|
| Container | `Dialog size="L"` from `@react-spectrum/s2` |
| Option list | `RadioGroup`; each item: icon (40×40) + product name + offer ID |
| Selected state | `border-color: #1473e6`, `background: #f0f6ff` |
| Actions row | **Cancel** (secondary) + **Proceed** (accent, disabled until a selection is made) |
| CSS | `styles/customerdetails/UpgradePathsDialog.module.css` |

**States:**

| State | Behaviour |
|-------|-----------|
| Loading | `useProductNamesFromPricelist` pending → "Loading upgrade options..." text |
| Empty / error | Guarded upstream — `hasUpgradePath` prevents the dialog from opening with an empty list |
| Happy path | Radio list rendered with resolved product names |

**Interactions:**

| Action | Result |
|--------|--------|
| Radio select | `setSelectedOfferId` |
| **Proceed** | `onSelectUpgrade(selectedOfferId, switchType)` + `close()` |
| **Cancel** | `close()` |

**Accessibility:**

| Element | Attribute |
|---------|-----------|
| `RadioGroup` | `aria-label="Select upgrade target"` |
| Each `Radio` | `aria-label={item.name}` |

---

### Modified Components

#### `ActiveProducts` — MODIFY at `components/customerdetails/ActiveProducts.tsx`

**Why:** Needs an **Upgrade** button per product card, conditional on `hasUpgradePath`, and a `DialogTrigger` wrapping `UpgradePathsDialog`.

**Props interface changes:** None — all new state is internal.

**Render changes:**

| Change | Detail |
|--------|--------|
| New imports | `useRouter` from `next/router`; `DialogTrigger` from `@react-spectrum/s2`; `UpgradePathsDialog`; `useOfferSwitchPaths` |
| New hook call | `useOfferSwitchPaths(customerId, products.map(p => p.id))` called at component top |
| Button row | Replace single `<Button>Add licenses</Button>` with `<div className={styles.buttonRow}>` containing the existing Add licenses button plus a conditional `<DialogTrigger>` + **Upgrade** button (rendered only when `hasUpgradePath(product.id)` is true) |
| New handler | `onSelectUpgrade(targetOfferId, switchType)` — builds query params and calls `router.push('/checkout/upgrade?...')` |
| New CSS | `.buttonRow { display: flex; gap: 8px; }` added to `styles/customerdetails/ActiveProducts.module.css` |

---

#### `CheckoutProductCard` — MODIFY at `components/checkout/CheckoutProductCard.tsx`

**Why:** The upgrade order summary needs to enforce a quantity ceiling (`maxQuantity`) and optionally lock quantity editing when `switchType=FULL_ONLY`.

**New props (optional, non-breaking):**

| Prop | Type | Required | Default | Wired to |
|------|------|----------|---------|----------|
| `maxQuantity` | `number` | No | — | `NumberField.maxValue` |
| `isQuantityReadOnly` | `boolean` | No | `false` | `NumberField.isReadOnly` |

---

## Section 4: Data Fetch Unit Specs

### `useOfferSwitchPaths` — CREATE at `hooks/useOfferSwitchPaths.ts`

**Why:** Checks upgrade eligibility per subscription on product listing render; runs in parallel via `useQueries`.

**Backend LLD ref:** Section 3 — `pages/api/offer-switch-paths.ts` (CREATE)

**Fetch contract:**

| Field | Value |
|-------|-------|
| Endpoint | `GET /api/offer-switch-paths?customer-id=<id>&subscription-id=<id>` |
| Auth | None (handled server-side by `handlePrerequisites`) |
| Cache key | `['offerSwitchPaths', customerId, subscriptionId]` |
| Cache freshness | `staleTime: 5 min` |
| Cache lifetime | `gcTime: 15 min` |
| Dependency guard | `enabled: !!customerId && !!subscriptionId` |

**Response shape:**

| Field | Type | Notes |
|-------|------|-------|
| `productUpgrades[].sourceBaseOfferId` | `string` | Current product offer ID |
| `productUpgrades[].targetList[].targetBaseOfferId` | `string` | Upgrade target |
| `productUpgrades[].targetList[].sequence` | `number` | Display order (ascending) |
| `productUpgrades[].targetList[].switchType` | `string` | `FULL_ONLY` or `PARTIAL_ALLOWED` |

**Hook returns:**

| Method | Signature | Behaviour on error |
|--------|-----------|--------------------|
| `hasUpgradePath` | `(subscriptionId: string) => boolean` | Returns `false` — Upgrade button hidden (silent fail per experience card) |
| `getUpgradePathDetails` | `(subscriptionId: string) => { targetBaseOfferId, switchType }[]` | Returns `[]` — ordered by `sequence` on success |

---

### `usePreviewSwitchOrder` — CREATE in `hooks/useOrderOperations.ts`

**Why:** `useMutation` wrapper for the non-destructive price preview call.

**Backend LLD ref:** Section 3 — `pages/api/orders.ts` MODIFY, `PREVIEW_SWITCH` branch

**Fetch contract:**

| Field | Value |
|-------|-------|
| Endpoint | `POST /api/orders?type=PREVIEW_SWITCH&fetch-price=true` |
| Auth | None (handled server-side) |
| Cache key | N/A — mutation |

**Request shape:**

| Field | Type | Notes |
|-------|------|-------|
| `customerId` | `string` | |
| `externalReferenceId` | `string` | |
| `currencyCode` | `string` | |
| `lineItems[0].extLineItemNumber` | `1` | Fixed |
| `lineItems[0].offerId` | `string` | Target offer |
| `lineItems[0].quantity` | `number` | |
| `lineItems[0].flexDiscountCodes` | `string[]` | Optional |
| `cancellingItems[0].extLineItemNumber` | `2` | Fixed |
| `cancellingItems[0].referenceLineItemNumber` | `1` | Fixed |
| `cancellingItems[0].subscriptionId` | `string` | Source subscription |
| `cancellingItems[0].quantity` | `number` | |

**Response shape:**

| Field | Type | Used for |
|-------|------|----------|
| `pricingSummary[].totalLineItemPartnerPrice` | `number` | Estimated total display |
| `pricingSummary[].currencyCode` | `string` | |
| `lineItems[].pricing.partnerPrice` | `number` | |
| `lineItems[].pricing.netPartnerPrice` | `number` | |
| `lineItems[].pricing.lineItemPartnerPrice` | `number` | Per-item price |

**Outcomes:**

| Outcome | Behaviour |
|---------|-----------|
| Success | Updates `item` pricing state; sets `estimatedTotal`; clears `needsRecalculation`; enables Place order |
| Error | `switchOrderPreview.isError` → inline error message shown on upgrade page; Place order remains disabled |

---

### `usePlaceSwitchOrder` — CREATE in `hooks/useOrderOperations.ts`

**Why:** `useMutation` wrapper for the final switch order placement.

**Backend LLD ref:** Section 3 — `pages/api/orders.ts` MODIFY, `SWITCH` branch

**Fetch contract:**

| Field | Value |
|-------|-------|
| Endpoint | `POST /api/orders?type=SWITCH` |
| Auth | None (handled server-side) |

**Request shape:** Same as `usePreviewSwitchOrder` minus the `fetch-price` query param.

**Response shape:**

| Field | Type |
|-------|------|
| `orderId` | `string` |
| `status` | `string` |
| `lineItems[].offerId` | `string` |
| `lineItems[].quantity` | `number` |
| `lineItems[].subscriptionId` | `string` |

**Outcomes:**

| Outcome | Behaviour |
|---------|-----------|
| Success | `router.push('/orderConfirmation?...')` with `customerId`, `currencyCode`, `totalAmount`, `customerName`, `resellerId`, `products` |
| Error | `switchOrderSubmit.isError` → inline error; user stays on page; Place order button re-enables |

---

## Section 5: Data Flow

```
User action: Product listing renders
  → ActiveProducts mounts
  → useOfferSwitchPaths fires GET /api/offer-switch-paths per subscriptionId
  ← { productUpgrades[] } per subscription
  ← hasUpgradePath(id) = true → Upgrade button rendered on product card

User action: Clicks Upgrade button
  → DialogTrigger opens UpgradePathsDialog
  → useProductNamesFromPricelist fetches display names for each targetBaseOfferId
  ← Radio list rendered with resolved product names

User action: Selects target and clicks Proceed
  → onSelectUpgrade(targetOfferId, switchType) fires
  → router.push('/checkout/upgrade?customerId=...&targetOfferId=...&switchType=...&...')
  → UpgradeCheckoutPage mounts

User action: Page load (automatic)
  → useEffect fires → setItem initialised → runPreview()
  → usePreviewSwitchOrder.mutate({ ..., orderType implied PREVIEW_SWITCH })
  → POST /api/orders?type=PREVIEW_SWITCH
  ← { pricingSummary, lineItems[].pricing }
  ← estimatedTotal set; item.pricePerUnit updated; Place order enabled

User action: Changes quantity
  → setItem({ ...prev, quantity: newQty }); setNeedsRecalculation(true)
  ← Update price button activates; Place order disabled

User action: Clicks Update price
  → runPreview() with current item
  → POST /api/orders?type=PREVIEW_SWITCH
  ← estimatedTotal refreshed; needsRecalculation cleared

User action: Clicks Place order
  → usePlaceSwitchOrder.mutate({ ..., orderType implied SWITCH })
  → POST /api/orders?type=SWITCH
  ← { orderId, status }
  ← router.push('/orderConfirmation?...')
```

---

## Section 6: Acceptance Criteria Coverage

| Criterion (from experience card) | LLD section |
|-----------------------------------|-------------|
| Upgrade button shown only when upgrade paths exist | Section 3 — `ActiveProducts` MODIFY; `hasUpgradePath` gate |
| Upgrade button hidden if eligibility API fails | Section 4 — `useOfferSwitchPaths` error → `hasUpgradePath = false` |
| Dialog lists each upgrade path with product name and icon | Section 3 — `UpgradePathsDialog` CREATE |
| Targets sorted by `sequence` ascending | Section 4 — `useOfferSwitchPaths.getUpgradePathDetails`; targets returned from API already sequenced; UI maps them in order |
| Preview loaded on order summary page load | Section 4 — `usePreviewSwitchOrder`; Section 6 data flow step 4 |
| Estimated total from `pricingSummary[].totalLineItemPartnerPrice` | Section 4 — `usePreviewSwitchOrder` response shape |
| Quantity editable unless `FULL_ONLY` | Section 3 — `CheckoutProductCard` MODIFY `isQuantityReadOnly` prop |
| Update price re-runs preview with new quantity | Section 6 — data flow "User action: Clicks Update price" |
| Place order uses `orderType: SWITCH` | Section 4 — `usePlaceSwitchOrder`; backend LLD enforces via `SwitchOrderSchema` |
| Success → navigate to confirmation | Section 6 — `router.push('/orderConfirmation?...')` on success |
| Place order error → inline error, stay on screen | Section 4 — `switchOrderSubmit.isError` inline message; no navigation on error |

---

## Section 7: Out of Scope

| Item | Reason |
|------|--------|
| Revert switch order UI | Not described in experience card |
| Personalised recommendations upgrade path (`PersonalizedRecommendations.tsx`) | Reference implementation extends this component; experience card does not describe it |
| `OrderDetailsDialog` SWITCH order display | Not in experience card scope |
| Flex discount / promo context filtering (`ApplicableTo.ANYTIME_UPGRADES`) | `flexDiscountsUtils` does not exist in this codebase |

---
