# Experience Card: Flexible Discounts (Discovery, Checkout, Edit Renewal Preference)

- Full Discovery
- Inline Discovery & Application During Order Checkout
- Inline Discovery & Application During Renewal Preference Editing

figma-url:

---

## Overview

Adobe offers partners time-limited promotional discounts (Flexible Discounts, or "flex discounts") that can be applied to customer orders to reduce the price of eligible products. Discounts are scoped to a country and market segment, and come in two categories — **Standard** (broad promotional discounts) and **Intro** (introductory offers for new customers or first-time product adoption).

This kit launches with a focus on **Flexible Discounts for 3-Year Commitments (3YC)** — promotional pricing designed to incentivize new 3-year commitments or reward existing 3YC customers. Partners can target two groups: customers becoming 3YC-compliant through a new order, and existing 3YC subscribers (including those meeting minimum purchase quantity requirements). These promotions can be applied to the first term, the current term, or the full 3YC commitment duration, and are discoverable via the existing `GET /v3/flex-discounts` API and applied through standard `NEW` or `RENEWAL` order requests.

This experience covers three surfaces where partners interact with flex discounts:

1. **Flex Discounts Page** — a dedicated page where partners can browse and filter all available discounts for a given market.
2. **Promo Discovery in Checkout** — an inline panel inside the checkout promo code dialog that surfaces relevant discounts at the moment of purchase.
3. **Promo Discovery in Edit Renewal** — an inline panel inside the edit renewal dialog that surfaces renewal-eligible discounts per product when a partner is editing a customer's upcoming renewal.

---

## Goals

- Partners can quickly discover which discounts are available for a given country and market segment.
- Partners can narrow results to discounts relevant to a specific use case (new purchases, renewals, upgrades, 3YC).
- Partners can copy a discount code and understand exactly which products and pricing it applies to.
- During checkout, partners can discover and apply an eligible discount without leaving the order flow.
- During renewal editing, partners can discover and apply renewal-eligible discounts per product, and see auto-applied reusable discounts before confirming.

---

## Surface 1 — Flex Discounts Page

### Entry point

The partner arrives at this page by clicking **Flexible Discounts** in the sidebar navigation.

---

### Screen: Discount availability

The page is split into two areas: a **filter panel** on the left and a **discount card grid** on the right.

When the page loads, it immediately fetches all available discounts for the partner's country and market segment, including reusable discounts. The total number of matching discounts is shown above the grid (e.g. *"14 discount(s) available"*).

**Loading state:** A spinner is shown while results are being fetched. The grid does not render until both a country and market segment are set.

**Empty state:** If no discounts exist for the selected market, the page shows: *"No discounts available for [country] / [market segment]."*

**Error state:** If the request fails, an error message is shown with the failure reason.

---

### Filters

Partners can narrow results using the following filters.

**Filters that refresh results from Adobe (server-side):**

| Filter | Options                                    | Behaviour |
|--------|--------------------------------------------|-----------|
| Country | Text field — partner enters a country code | Changing country re-fetches all results |
| Market segment | Radio — COM/GOV/EDU                        | Changing segment re-fetches all results |
| Category | Radio — All / Standard / Intro             | Filters to one discount category or shows both |
| Discount code | Search field                               | Searches by exact code; results refresh after the partner stops typing (300 ms delay) |
| Date range | Start date + End date                      | Shows only discounts active within the specified window |

**Filters applied instantly without a new request (client-side):**

| Filter | Options | Logic                                                                                                                                                                                                                                                                                                              |
|--------|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Status | Radio — All / Active / Upcoming | **Active** — discount is currently live. **Upcoming** — discount exists but its start date is in the future.                                                                                                                                                                                                       |
| Applicable to | Checkboxes — New purchases / Renewals / 3YC customers / Anytime upgrades | Client-side substring match against `discount.name` — case-insensitive. Keywords per option: **New purchases** → `"Add Seats"` · **Renewals** → `"Renewal"` · **3YC customers** → `"3YC"` · **Anytime upgrades** → `"Switch"`. A discount matches if its name contains any keyword from any selected option. |

> **Note on status:** Adobe's API returns both live and future-dated discounts under the same "Active" status. The page uses the discount's start date to distinguish currently-live discounts from upcoming ones — this is handled automatically by the UI.

**Continuous loading:** When the applied filters produce no visible results but more discounts exist on the server, the page automatically loads the next batch and applies the filter again — repeating until matches are found or all discounts have been checked.

The grid grows as the partner scrolls down. More results load automatically as they near the bottom of the page.

---

### Discount card

Each discount is shown as a card. A card displays:

- **Discount value** — one badge per outcome value. Formats: *"20% off"*, *"$10 off · USD"*, *"$15.99 fixed price · USD"*
- **Reusable** badge — shown when the discount has a `discountLockEndDate` set.
- **Name and description** — the discount's title and eligibility details. If the description exceeds 3 lines, it is truncated with a **Read more** link; clicking it expands the full description inline.
- **Discount code** — a clickable chip the partner can tap to copy the code. After clicking, it briefly shows *"Copied!"* before reverting. If the browser blocks clipboard access, the copy fails silently.
- **Validity dates** — when the discount starts, ends, and (for reusable discounts) the date until which it can still be reused
- **Eligible products** — a button showing how many products this discount applies to. Clicking it opens the Eligible Products panel. If the discount applies to all products, the button reads *"All products eligible"* instead.

---

### Eligible products panel

When the partner clicks the eligible products button on a card, a modal opens showing:

- The discount name, code, and category
- The discount value badges
- A list of each eligible product with its name, standard price, and discounted price

If product names cannot be loaded, the offer IDs are shown in their place.

The panel closes when the partner clicks outside it or uses the close action.

---

## Surface 2 — Promo Discovery in Checkout

### Entry point

When a partner is placing a **new purchase** , each product card in the checkout flow includes a promo code field. Clicking on it opens a dialog where the partner can either type a code directly or browse available discounts using the inline discovery panel.

---

### Screen: Promo code dialog with inline discovery

The dialog shows two sections stacked vertically:
- A **promo code input** at the top where the partner can type or paste a code
- An **inline discovery panel** below it, listing discounts relevant to the product being purchased

When the dialog opens, discounts are fetched via `GET /v3/flex-discounts` — scoped to the customer's country and market segment, filtered by the product's offer ID.

**Loading state:** A small spinner is shown while discounts are being fetched.

**Empty / error state:** If no relevant discounts are found, the panel shows: *"No discounts available for this product."*

---

### Discount list

The panel shows a compact list of discount rows. Each row displays:

- Discount name and (when present) a **Reusable** badge
- Discount value badges alongside the date range
- An **Apply** button that inserts the code; disabled when the discount has no code

A **Load more** button at the bottom of the list fetches the next page of offer-scoped results.

**Pagination model:** same offset-based model as Surface 1

---

### Applying a discount

When the partner clicks **Apply** on a row, the discount code is inserted into the promo code input field in the dialog.

---

### Order preview with promo code

Once a code is entered or selected, an order preview is triggered immediately. The code is sent in the line item as `flexDiscountCodes: [code]`. The preview API validates the code and returns one of two outcomes per line item:

- **Success** — `flexDiscounts: [{ code, result: "SUCCESS" }]`. The pricing in the checkout updates to reflect the discounted price.
- **Failure** — `flexDiscounts: [{ code, result: "FAILURE" }]`. The code is treated as invalid; the discount is not applied and pricing remains unchanged.

The partner sees the updated (discounted) price in the order summary before confirming. If the code is invalid, no error is surfaced explicitly — the pricing simply does not change.

---

### Placing the order

When the partner confirms the order, the line item is submitted with `flexDiscountCodes: [code]` included. This applies to all checkout contexts:

- **New purchase** — `flexDiscountCodes` sent in the new order line item
- **Renewal** — `flexDiscountCodes` sent inside `autoRenewal.flexDiscountCodes` on the subscription update request

---

## Surface 3 — Promo Discovery in Edit Renewal

### Entry point

When a partner opens the **Edit Renewal** dialog for a customer, each product in the renewal list shows an **Apply discount code** button. Clicking it opens the promo code dialog for that specific product.

---

### Screen: Promo code dialog with inline discovery

The dialog is identical in structure to Surface 2 — a **promo code input** at the top with the **inline discovery panel** below it — but is filtered to discounts relevant to the renewal context.

When the dialog opens, discounts are fetched via `GET /v3/flex-discounts` — scoped to the customer's country and market segment, filtered by the product's offer ID, with `startDate` set to today.

**Loading state:** A spinner is shown while discounts are being fetched.

**Empty / error state:** If no renewal-eligible discounts are found, the panel shows: *"No discounts available for this product."*

---

### Discount list

Each row in the discovery panel displays:

- Discount name and a **Reusable** badge when applicable
- Discount value badges (percentage off, fixed discount, or fixed price) alongside the validity date range
- A **Select** button that fills the promo code input with that discount's code

A **Load more** button at the bottom fetches the next page of offer-scoped results.

---

### Applying a discount

When the partner clicks **Select** on a row, the discount code is inserted into the promo code input. The partner then clicks **Apply** in the dialog to confirm the selection.

Once applied:
- The code is stored against that product in the renewal order state (`product.discountCode`)
- The product's displayed price is cleared once a discount code is applied
- When the partner saves changes, `PATCH /v3/customers/{customerId}/subscriptions/{subscriptionId}` is called first (in parallel for all changed subscriptions) to persist the updated auto-renewal settings and discount codes; once those complete, a renewal preview is triggered and the updated pricing is shown

---


### Renewal preview with promo code

When the partner saves their changes in the dialog, a renewal preview is triggered:

```
POST /v3/customers/<customer-id>/orders
{
  "orderType": "PREVIEW_RENEWAL",
  "lineItems": [
    {
      "offerId": "<offer-id>",
      "quantity": <quantity>,
      "flexDiscountCodes": ["<code>"]   // omitted if no code applied
    }
  ]
}
```

The preview API validates the code per line item and returns one of two outcomes:

- **Success** — `flexDiscounts: [{ code, result: "SUCCESS" }]`. The product's displayed price updates to the discounted value.
- **Failure** — `flexDiscounts: [{ code, result: "FAILURE" }]`. The code badge is shown in a negative (red) state, and pricing remains unchanged.

The partner sees updated pricing for every product before confirming the renewal.

---

### Saving the renewal

When the partner confirms, each product's applied discount code is submitted inside `autoRenewal.flexDiscountCodes` on the subscription update request:

```
PATCH /v3/customers/<customer-id>/subscriptions/<subscription-id>
{
  "autoRenewal": {
    "enabled": true,
    "renewalQuantity": <quantity>,
    "flexDiscountCodes": ["<code>"]   // omitted if no code applied
  }
}
```

Partners can edit multiple products in a single dialog session; each product carries its own independent discount code. Codes applied to one product do not affect others.

---