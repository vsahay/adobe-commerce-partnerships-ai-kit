# Receiving the API Spec

**Who does this:** Partners validates completeness  
**Frequency:** Once per feature  
**Prerequisites:** Backend Service card is complete and reviewed; Feature Experience card has been received and validated

---

## What the API Spec Is

The API spec is a precise, structured description of every Adobe VIPMP Partners API endpoint the feature will call. It is written by **Adobe's Marketplace team** and provided to you as part of the feature package alongside the experience card.

You do not write the API spec. Your job at this step is to validate that it is complete and internally consistent before the skills use it to generate LLDs.

The AI generates the backend LLD by reading your API spec alongside your service cards. If the API spec is incomplete or ambiguous, the LLD will fill in the blanks with plausible-but-wrong assumptions. Catching gaps here costs minutes; catching them in code review costs hours.

---

## What a Complete API Spec Looks Like

A well-formed API spec covers every endpoint the feature uses, not just the primary one. Use this as your validation checklist.

**Structure checklist — for each operation:**

- [ ] HTTP method and full path are specified (e.g., `GET /v3/offer-switch-paths`)
- [ ] All query parameters are listed with name, type, required/optional, and description
- [ ] All request headers are listed — typically `Authorization: Bearer <token>` and `X-Api-Key`
- [ ] Request body is specified (for POST/PUT/PATCH) with every field, its type, and whether it is required
- [ ] Response body (200 OK) is specified with every field the feature will use — actual JSON structure, not a prose description
- [ ] All error responses are listed with status code, error code, meaning, and expected UI behavior

**Consistency checklist — cross-check against the experience card:**

- [ ] Every step in the experience card that says "API call: X" has a corresponding operation in the API spec
- [ ] The data returned by each operation is sufficient to power the UI state described in the experience card (e.g., if Step 3 shows a "prorated price," that field exists in the response)
- [ ] The error codes in the API spec align with the error states in the experience card — no error state is described in the experience card for which there is no corresponding API error code, and vice versa
- [ ] If an endpoint has multiple modes controlled by a field (e.g., `type: PREVIEW_SWITCH` vs `type: SWITCH`), each mode has its own documented request/response shape

---

## Worked Example — Anytime Upgrade

Below is the API spec for the Anytime Upgrade feature, as provided by Adobe. This is the format you will receive.

### Authentication

All requests require:
- `Authorization: Bearer <access_token>` — obtained from the partner auth flow
- `X-Api-Key: <api_key>` — your partner API key

---

### Operation 1 — Get Upgrade Paths

**Purpose:** Retrieves all available upgrade paths for a customer's subscriptions. Called when the subscription management page loads (Step 1 of the experience card).

**HTTP Method and Path:**
```
GET /v3/offer-switch-paths
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `customerId` | string | Yes | The customer's Adobe ID |
| `subscriptionId` | string | Yes | The subscription to retrieve upgrade paths for |

**Request Headers:**

| Header | Value |
|---|---|
| `Authorization` | `Bearer <access_token>` |
| `X-Api-Key` | `<api_key>` |

**Response (200 OK):**
```json
{
  "offerSwitchPaths": [
    {
      "subscriptionId": "string — ID of the source subscription",
      "currentOfferId": "string — offer ID of the current subscription tier",
      "switchPaths": [
        {
          "targetOfferId": "string — offer ID of the upgrade target",
          "targetOfferName": "string — display name of the upgrade target",
          "minQuantity": "number — minimum quantity for this offer",
          "maxQuantity": "number — maximum quantity for this offer"
        }
      ]
    }
  ]
}
```

**Error Responses:**

| Status | Code | Meaning | UI behavior |
|---|---|---|---|
| 404 | `CUSTOMER_NOT_FOUND` | Customer ID does not exist | Show page-level error banner |
| 403 | `FORBIDDEN` | Partner lacks access to this customer | Redirect to customer list with error message |
| 500 | — | Server error | Show page-level error banner; preserve any data already loaded |

---

### Operation 2 — Preview or Place Upgrade Order

**Purpose:** Used for two sub-operations depending on the `type` field. Called to preview pricing (Steps 3–4 of the experience card) and to confirm the order (Step 5).

**HTTP Method and Path:**
```
POST /v3/orders
```

**Request Headers:**

| Header | Value |
|---|---|
| `Authorization` | `Bearer <access_token>` |
| `X-Api-Key` | `<api_key>` |
| `Content-Type` | `application/json` |

**Request Body — Preview:**
```json
{
  "type": "PREVIEW_SWITCH",
  "customerId": "string — customer's Adobe ID",
  "subscriptionId": "string — ID of the subscription being upgraded",
  "targetOfferId": "string — offer ID of the upgrade target",
  "quantity": "number — desired quantity after upgrade"
}
```

**Request Body — Place Order:**
```json
{
  "type": "SWITCH",
  "customerId": "string",
  "subscriptionId": "string",
  "targetOfferId": "string",
  "quantity": "number"
}
```

**Response (200 OK — Preview):**
```json
{
  "previewOrder": {
    "proratedTotal": "number — amount due now, in cents",
    "currency": "string — ISO 4217 currency code",
    "remainingTermDays": "number — days left in current billing term",
    "newMonthlyPrice": "number — monthly price after upgrade, in cents"
  }
}
```

**Response (200 OK — Place Order):**
```json
{
  "orderId": "string — ID of the placed order",
  "status": "string — COMPLETED | PENDING",
  "subscription": {
    "subscriptionId": "string",
    "offerId": "string — new offer ID after upgrade",
    "quantity": "number — new quantity"
  }
}
```

**Error Responses:**

| Status | Code | Meaning | UI behavior |
|---|---|---|---|
| 400 | `INVALID_QUANTITY` | Quantity outside min/max bounds | Show inline error in the upgrade panel |
| 400 | `INELIGIBLE_SWITCH` | This subscription/offer combination cannot be switched | Show inline error; do not close panel |
| 409 | `PENDING_ORDER_EXISTS` | A prior order for this subscription is pending | Inform user; disable confirm button |
| 500 | — | Server error | Show inline error; allow retry |

---

## What to Do If the API Spec Is Incomplete

If any item in the validation checklist is missing, contact your Adobe partner contact before proceeding.

Common gaps and how to describe them:

- "Operation 2 documents the PREVIEW_SWITCH response but not the SWITCH response. Can you add the SWITCH 200 response shape?"
- "The error table for Operation 1 does not have a UI behavior column — can you specify what should happen for each error?"
- "The experience card Step 3 says the preview shows a 'prorated total,' but the response shape does not include that field. Is it `proratedTotal` or something else?"

Do not make assumptions to fill gaps. An assumption that is wrong will propagate through the LLD into generated code.

---

## After This Step

Both the experience card and API spec are validated. You are ready to generate the backend LLD.

Proceed to **[Generating the Backend LLD](generate-backend-lld.md)**.