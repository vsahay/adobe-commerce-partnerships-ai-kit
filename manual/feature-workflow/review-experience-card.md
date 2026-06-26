# Receiving the Experience Card

**Who does this:** Partners validates completeness
**Frequency:** Once per feature  
**Prerequisites:** Backend and UI Service Cards are complete and reviewed

---

## What the Experience Card Is

The experience card is a structured document written by **Adobe's Marketplace team** that defines the user journey for a feature. It describes what happens from the user's perspective: what they see, what they do, what data the app needs at each step, and what happens when things go wrong.

You do not write the experience card. Adobe provides it to you as part of the feature package alongside the API spec.

Your job at this step is to:
1. Confirm the experience card is complete and unambiguous before running any skills
2. Raise any gaps or questions with your Adobe partner contact before proceeding

Starting rest of the feature implementation with an incomplete experience card will produce LLDs that miss error states, omit steps, or describe UI behavior that does not match Adobe's intent.

---

## What a Complete Experience Card Looks Like

A well-formed experience card has all of the following. Use this as your validation checklist.

**Structure checklist:**

- [ ] An **overview** paragraph that describes what the feature does and why
- [ ] A **preconditions** section listing what must be true before the flow starts — required permissions, required session state, required prior actions
- [ ] A **numbered step sequence** — not bullet points, not prose, but discrete numbered steps
- [ ] For each step: a user action, the API call at that step (if any), a success state, and named error states
- [ ] An **edge cases** section covering scenarios that do not fit the main step sequence
- [ ] An **out of scope** section explicitly stating what this feature does not handle
- [ ] A **Figma design URL** with a node ID (or explicit note that no design is available)

**Completeness checklist — things most often missing:**

- [ ] Every step that involves an API call names the operation (e.g., "GET offer-switch-paths") — not just "fetch subscription data"
- [ ] Error states are specific: they name what the user sees, whether the user can retry, and whether existing data is preserved — not "show an error message"
- [ ] Preconditions include permission requirements, not just data requirements
- [ ] Edge cases cover multi-item scenarios (e.g., a customer with multiple subscriptions where some are eligible and some are not)
- [ ] The Figma URL includes the node ID (`?node-id=XX%3AYY`) — a URL without a node ID points to the whole file, not the right frame

---

## Worked Example — Anytime Upgrade

Below is the experience card for the Anytime Upgrade feature, as provided by Adobe. This is the format you will receive.

```markdown
# Experience Card — Anytime Upgrade

## Overview
A partner can upgrade a customer's subscription mid-term to a higher tier or greater
quantity. The upgrade is applied immediately with prorated pricing — the customer pays
only for the remaining term at the new tier.

## Preconditions
- Partner is authenticated with a valid session token
- Partner has permission to manage the customer's subscriptions
- The customer has at least one active subscription

## Figma Design
https://www.figma.com/file/xxxx/anytime-upgrade?node-id=12%3A345

## User Journey

### Step 1 — Check upgrade eligibility
**User action:** Partner navigates to the subscription management page for a customer.
**API call:** GET offer-switch-paths — retrieves all available upgrade paths for each
              active subscription.
**Success state:** The page displays each subscription with an "Upgrade" button if
                   upgrade paths exist for it.
**Error states:**
- No upgrade paths available: Show subscription with no upgrade button; optionally
  show a tooltip "No upgrade options available for this subscription."
- API error (5xx): Show an inline error banner; do not hide existing subscription data.

### Step 2 — View upgrade options
**User action:** Partner clicks "Upgrade" on a specific subscription.
**API call:** None — uses data already fetched in Step 1.
**Success state:** A panel or modal opens showing available upgrade tiers and quantities.
**Error states:** N/A — no new API call at this step.

### Step 3 — Preview upgrade pricing
**User action:** Partner selects an upgrade option (tier or quantity).
**API call:** POST orders with type PREVIEW_SWITCH — returns a price preview for the
              selected upgrade.
**Success state:** The panel updates to show the prorated price for the remaining term.
**Error states:**
- Ineligible combination: Show inline message "This combination is not available."
- API error: Show inline error; keep previous preview visible if one existed.

### Step 4 — Adjust quantity
**User action:** Partner changes the quantity in the upgrade panel.
**API call:** POST orders with type PREVIEW_SWITCH — same as Step 3, called again with
              new quantity.
**Success state:** Prorated price updates in place.
**Error states:** Same as Step 3.

### Step 5 — Place the upgrade
**User action:** Partner clicks "Confirm Upgrade."
**API call:** POST orders with type SWITCH — places the upgrade order.
**Success state:** Panel closes; subscription row updates to reflect new tier/quantity;
                   a success toast is shown.
**Error states:**
- Order failed: Show error in the panel; do not close it; allow the partner to retry.
- Session expired: Redirect to login with a message "Your session expired. Please log
  in again to complete the upgrade."

## Edge Cases
- If the customer has multiple subscriptions, each is assessed independently for
  eligibility. The UI must handle the case where some are eligible and some are not.
- Quantity must be at least 1 and at most the maximum defined in the offer-switch-path
  response. Enforce this in the UI before calling the preview API.

## Out of Scope
- Downgrade flows
- Cancellation flows
- Upgrading a subscription that is already in a pending state
```

---

## What to Do If the Experience Card Is Incomplete

If you run through the validation checklist and find gaps, do not proceed to rest of the feature implementation workflow steps. Incomplete inputs produce incomplete outputs.

**Contact your Adobe partner contact** with specific questions. Helpful framing:

- "Step 3 names a success state but no error state. What should the user see if the PREVIEW_SWITCH call returns a 400?"
- "The edge cases section does not address what happens if the customer has zero active subscriptions. Should the page be hidden, disabled, or show an empty state?"
- "The Figma URL does not include a node ID — can you share the specific frame?"

Be specific. "The experience card is missing some things" will result in a slower response than a precise list of gaps.

---

## After This Step

Confirm with your partner engineer that the API spec has also been received. Both documents are needed before running FW-3 and FW-4.

Once both documents are in hand and validated, proceed to **[Receiving the API Spec](review-api-spec.md)**.