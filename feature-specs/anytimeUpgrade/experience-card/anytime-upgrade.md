# Reference Workflow: Anytime Upgrade
last-updated: 2026-04-21

figma-url: https://www.figma.com/design/c3rdN5VOmVVkwTMkb6pMYj/midterm?node-id=0-1&p=f&t=LjNBinF9wa5P7Itx-0

---

## Overview

The Anytime Upgrade feature allows a partner to upgrade a customer's active subscription
to a higher-tier product before the subscription's anniversary date. The upgrade is
executed as a switch order — the existing subscription is cancelled and replaced with
the new one, with prorated pricing applied for the remaining term.

---

## Preconditions

- The customer has at least one active subscription.
- The partner has a valid session with an active org context.
- Upgrade eligibility is determined per subscription, not per customer.

---

## Step 1 — Determine upgrade eligibility per subscription

**Trigger:** The product listing screen renders.

**Action:** For each active subscription displayed, check whether upgrade paths exist.

**API call:**
```
GET /v3/offer-switch-paths
  subscriptionId = <subscription's ID>
  customerId     = <customer's ID>
```

**Data used from response:**
- `productUpgrades` — array of `{ sourceBaseOfferId, targetList[] }`
- If `productUpgrades` is empty or absent → no upgrade available for this subscription

**UI outcome:**
- If upgrade paths exist → show an **Upgrade** button on that product card
- If no upgrade paths exist → no button shown; product card renders as normal

**Error handling:**
- If the eligibility API call fails → do not show the Upgrade button; fail silently (log the error)

---

## Step 2 — Display available upgrade paths

**Trigger:** User presses the **Upgrade** button on a product card.

**Data already available (no new API call needed):**
- `productUpgrades` from Step 1 response — the target offers for this subscription
- `sourceBaseOfferId` — the current product's offer ID
- `subscriptionId`, `customerId`, `currentQuantity`, `currencyCode`

**UI outcome:**
- Show a popup dialog listing each upgrade path
- For each target offer, display: product name, product icon, and offer ID
- Resolve product names from the offer ID using the product catalogue (not the raw offer ID string)
- Sort targets by `sequence` ascending

**User action:** User selects one upgrade target.

**Data captured on selection:**
- `targetOfferId` — the selected target offer ID

**Error handling:**
- If no upgrade paths available → show "No upgrade options available for this product."
- Provide a way to dismiss/close without taking action.

---

## Step 3 — Redirect to Order Summary Page and Preview the upgrade order

**Trigger:** User selects an upgrade target and navigates to order summary page.

**Purpose:** Fetch a price preview before the user commits to the order.
This is a non-destructive call — it does not place an order.

**API call:**
```
POST /v3/customers/{customerId}/orders?fetch-price=true
Body:
  orderType:      "PREVIEW_SWITCH"
  currencyCode:   <from subscription>
  lineItems:      [{ extLineItemNumber: 1, offerId: targetOfferId, quantity: currentQuantity }]
  cancellingItems:[{ extLineItemNumber: 1, referenceLineItemNumber: 1,
                     subscriptionId: sourceSubscriptionId, quantity: currentQuantity }]
```

**Data used from response:**
- `pricingSummary[].totalLineItemPartnerPrice` — estimated total cost for the partner
- `lineItems[].pricing` — per-item pricing breakdown

**UI outcome:**
- Show order summary: source product → target product (with icons and names)
- Show quantity field (editable, default = current subscription quantity)
- Show estimated total from the preview response
- Show an **Update price** button — re-runs the preview call with updated quantity
- Show a **Place order** button

**Error handling:**
- If preview call fails → show inline error; keep the screen open with a retry option
- While preview is loading → show a loading indicator in the price area; disable Place order button

---

## Step 4 — Update price (optional, user-triggered)

**Trigger:** User changes quantity and presses **Update price**.

**Action:** Re-run the same preview API call from Step 3 with the new quantity.

**UI outcome:**
- Refresh the displayed total price and offerId
- If call fails → show inline error

---

## Step 5 — Place the upgrade order

**Trigger:** User presses **Place order**.

**API call:**
```
POST /v3/customers/{customerId}/orders
Body:
  orderType:      "SWITCH"
  currencyCode:   <from subscription>
  lineItems:      [{ extLineItemNumber: 1, offerId: targetOfferId, quantity: <current quantity> }]
  cancellingItems:[{ extLineItemNumber: 1, referenceLineItemNumber: 1,
                     subscriptionId: sourceSubscriptionId, quantity: <current quantity> }]
```

**Data used from response:**
- `orderId` — confirms the order was placed
- `status` — order status

**UI outcome — success:**
- Show a success confirmation (toast or inline message)
- Navigate back to the customer detail screen

**UI outcome — error:**
- Show an inline error message with the failure reason
- Keep the user on the order summary screen
- Offer a retry option (re-enable Place order button)
- Do not navigate away

---

## Data flow summary

```
Product listing screen
  └── [on render, per subscription]
        GET /v3/offer-switch-paths?subscriptionId=...&customerId=...
          → productUpgrades[]
          → show Upgrade button if paths exist

User presses Upgrade
  └── open upgrade paths panel
        display targetList[] from productUpgrades
        [user selects target]
          → navigate to order summary screen
            carrying: customerId, subscriptionId, sourceOfferId,
                      targetOfferId, quantity, currencyCode,
                      customerName, resellerId

Order summary screen
  └── [on load]
        POST /v3/customers/{id}/orders (PREVIEW_SWITCH)
          → pricingSummary → display estimated total

  └── [user changes qty → Update price]
        POST /v3/customers/{id}/orders (PREVIEW_SWITCH, new qty)
          → refresh estimated total

  └── [user confirms → Place order]
        POST /v3/customers/{id}/orders (SWITCH)
          → success → navigate to customer detail
          → error   → show inline error, stay on screen
```

---

## Edge cases

| Scenario | Behaviour |
|---|---|
| No upgrade paths for a subscription | No Upgrade button shown; silent API failure also hides button |
| Preview API fails on load | Inline error on order summary screen; Place order disabled |
| User changes quantity below 1 | Quantity field enforces minimum of 1 |
| Place order fails | Inline error shown; user stays on order summary; retry available |
| User navigates back mid-flow | No order is placed; no side effects |

---