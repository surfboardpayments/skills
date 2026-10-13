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

For flows where the final capture amount may differ from the authorized amount, use `authMode: "PRE-AUTH"`. This automatically enables `delayCapture`:

```json
{
  "controlFunctions": {
    "delayCapture": true,
    "authMode": "PRE-AUTH",
    "initiatePaymentsOptions": {
      "paymentMethod": "CARD"
    }
  }
}
```

## Step 2: Capture the Payment

Once ready to finalize, call the capture endpoint with the `paymentId`:

```json
POST /payments/:paymentId/capture
{}
```

For `PRE-AUTH` orders, you can specify a different capture amount:

```json
POST /payments/:paymentId/capture
{
  "amount": 45000
}
```

```json
// Response
{
  "status": "SUCCESS",
  "message": "Payment captured successfully"
}
```

> The `amount` field is only valid for `PRE-AUTH` orders. For standard `delayCapture`, send an empty body to capture the full authorized amount.

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
