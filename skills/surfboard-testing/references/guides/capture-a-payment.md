---
title: "Capture a Payment"
source: https://www.surfboardpayments.com/developers/guides/capture-a-payment
category: online
tags: [Online, API, Payments, Capture, In-Store]
generated: true
---

# Capture a Payment

> Finalize a previously authorized payment by capturing funds. Covers delay capture and pre-authorization flows with step-by-step API examples.

## Overview

When you create an order with `delayCapture: true` or `authMode: "PRE-AUTH"`, funds are authorized but not immediately captured. This lets you verify inventory, confirm fulfillment, or adjust the final amount before charging the customer.

This guide walks through the full capture flow: create an authorized order, capture the payment, and verify the result.

## Delay Capture or Pre-Authorization?

Both authorize the card now and charge it later. The difference is what you can capture:

| | `delayCapture: true` | `authMode: "PRE-AUTH"` |
|---|---|---|
| At checkout | The full amount is authorized | The amount is held on the card |
| What you capture | Exactly the authorized amount | The final amount: the same, lower, or higher |
| Capture body | Empty `{}` | `{ "amount": ... }`, or empty for the held amount |
| Typical use | Capture at shipment | Hotel stays, car rental, bar tabs |
| `delayCapture` | You set it | Set for you |

## When to Use Delay Capture

| Scenario | Description |
|----------|-------------|
| **E-commerce fulfillment** | Authorize at checkout, capture at shipment |
| **Pre-authorization** | Hold a variable amount (e.g., hotel deposit), capture actual charge later |
| **Service bookings** | Authorize upfront, capture after service delivery |
| **Digital products** | Authorize, verify access, then capture |

## Step 1: Create an Order with Delay Capture

Create an order with `delayCapture: true` in `controlFunctions`:

```json
POST /orders
{
  "terminal$id": "YOUR_TERMINAL_ID",
  "orderLines": [
    {
      "id": "ITEM-001",
      "name": "Nike Shoes",
      "quantity": 1,
      "amount": {
        "regular": 50000,
        "total": 50000,
        "currency": "752",
        "tax": [
          { "amount": 10000, "percentage": 25, "type": "VAT" }
        ]
      }
    }
  ],
  "totalOrderAmount": {
    "regular": 50000,
    "total": 50000,
    "currency": "752",
    "tax": [
      { "amount": 10000, "percentage": 25, "type": "VAT" }
    ]
  },
  "controlFunctions": {
    "delayCapture": true,
    "initiatePaymentsOptions": {
      "paymentMethod": "CARD"
    }
  }
}
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "orderId": "83ac302f7c5130810b",
    "paymentId": "83ac302f24bfb00b06"
  },
  "message": "Order created successfully"
}
```

Store the `paymentId` -- you need it to capture.

### Webhooks at Authorization

When the customer pays, you receive `order.paymentinitiated`, sometimes `order.paymentprocessed`, and then `order.paymentcompleted`. For a delayed-capture order, `order.paymentcompleted` means the payment is **authorized**, not captured. No webhook is sent when the payment is later captured, so use the capture status call in [Step 3](#step-3-check-capture-status) to follow the capture.

### Payment Methods

`delayCapture` is accepted for every payment method, but not every method can hold funds:

| Payment method | Behavior with `delayCapture` |
|----------------|------------------------------|
| **Card** | Authorized at checkout, captured when you call capture |
| **Apple Pay / Google Pay** | Same as card |
| **Klarna** | Supports delayed capture and partial capture |
| **Swish** | No separate authorization step. The payment settles when the customer approves it and the flag has no effect |

### How Long an Authorization Lasts

Surfboard does not limit how long after authorization you can capture. The limit is set by the card scheme and acquirer, typically around **7 days** for cards, so capture well within that window.

Surfboard never captures or voids an authorization automatically. If you neither capture nor [void](/developers/guides/payment-lifecycle#void-a-payment) it, the authorization lapses at the issuer and the funds are released back to the customer. Void authorizations you no longer intend to capture rather than letting them lapse.

### Pre-Authorization Mode

When the final amount isn't known at checkout, for example a hotel stay where the minibar is added at check-out, hold an amount with `authMode: "PRE-AUTH"` and capture the real total later. Set it on `controlFunctions`, spelled `PRE-AUTH` with a hyphen. `delayCapture` is turned on for you, so you don't need to send it:

```json
{
  "controlFunctions": {
    "authMode": "PRE-AUTH",
    "initiatePaymentsOptions": {
      "paymentMethod": "CARD"
    }
  }
}
```

**What the customer and you see after they pay:**

- The order and payment read `PAYMENT_COMPLETED`, the same as a normal sale. For a pre-authorized order this means *held*, not charged. Nothing is charged until you capture.
- The transaction is a `PURCHASE` with `settlementStatus: "NOT_SETTLED"` until the capture is processed.

**Changing the amount:** once the customer has paid, the order itself can't be edited (Update Order answers `OR_0041`). Send the final amount with the capture instead, as shown in Step 2.

**Not charging at all:** [void the payment](/developers/guides/payment-lifecycle#void-a-payment) before you capture. That releases the whole hold.

## Step 2: Capture the Payment

Once ready to finalize, call the capture endpoint with the `paymentId`:

```json
POST /payments/:paymentId/capture
{}
```

For `PRE-AUTH` orders, send the final amount in the smallest currency unit. It can differ from the held amount in either direction:

- **Lower**, e.g. 300 SEK on a 500 SEK hold: only 300 SEK is charged.
- **Higher**, e.g. 650 SEK on a 500 SEK hold, for extras added at check-out.

```json
POST /payments/:paymentId/capture
{
  "amount": 45000
}
```

Leave the body empty (`{}`) to capture exactly the held amount.

```json
// Response
{
  "status": "SUCCESS",
  "message": "Payment captured successfully"
}
```

> The `amount` field is only valid for `PRE-AUTH` orders. For an order with only `delayCapture`, send an empty body to capture the full authorized amount.

A `SUCCESS` response means the capture request was accepted and sent on. For cards, captures are processed asynchronously in batches, and the acquirer can still decline the capture afterwards, so a successful response is not yet a completed capture.

## Step 3: Check Capture Status

There is no webhook for a completed or failed capture. Poll the capture status instead, starting about 30 minutes after the capture request:

```json
GET /payments/:paymentId/capture
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "captureStatus": "SUCCESS"
  }
}
```

**Possible `captureStatus` values:** `PENDING` | `SUCCESS` | `ERROR`

- **`PENDING`** -- the capture has not been processed yet. Check again later.
- **`ERROR`** -- the capture did not go through. It is safe to send `POST /payments/:paymentId/capture` again for a capture that never completed.
- If a capture stays `PENDING` well beyond the normal processing time, contact Surfboard support with the `paymentId`.

## Reference

- [Create Order API](https://developers.surfboardpayments.com/api/orders)
- [Payments API](https://developers.surfboardpayments.com/api/payments)
- [Create an Order](/developers/guides/create-an-order)
- [Payment Lifecycle](/developers/guides/payment-lifecycle)
