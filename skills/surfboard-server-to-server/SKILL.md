---
name: surfboard-server-to-server
description: Recurring and subscription billing with Surfboard Payments: Merchant Initiated Transactions, tokenising a card on the first customer payment, then charging it from your backend. Covers enforceTokenization, token storage and reuse, the two terminals required, and B2B invoicing. Use for subscriptions, retainers, usage billing, saved cards, or any charge without the customer present.
---

# Server-to-server and recurring payments

Read `surfboard-payments` first. This is the highest-value and most error-prone flow, so
work through it in order.

## Two stages, always

**A merchant cannot charge a card the customer has never presented.** The first payment
is customer-initiated and tokenises the card. Every later charge uses that token.

```
stage 1: customer pays  →  card tokenised
stage 2: you charge the token, from your backend, no customer present
```

Skipping stage 1 is the mistake that gets made here. There is no endpoint that takes a
card number.

## You need two terminals

| Terminal type | Stage | How you get it |
|---|---|---|
| `PaymentPage` or `SelfHostedPage` | 1, the first customer payment | PaymentPage already exists on an online store. Fetch it. |
| `MerchantInitiated` | 2, every later charge | Already exists on an online store. Fetch it. |

An online store is provisioned with both. **List the store's terminals and take the two
IDs.** Only `SelfHostedPage` needs registering, and only if you are collecting the first
payment on your own page.

## Stage 1: tokenise on the first payment

```
POST {SURFBOARD_API_URL}/merchants/:merchantId/orders
```

```json
{
  "terminal$id": "YOUR_PAYMENT_PAGE_TERMINAL_ID",
  "orderLines": [
    {
      "id": "SUB-001",
      "name": "Monthly subscription",
      "quantity": 1,
      "amount": { "regular": 29900, "total": 29900, "currency": "752" }
    }
  ],
  "totalOrderAmount": { "regular": 29900, "total": 29900, "currency": "752" },
  "controlFunctions": {
    "enforceTokenization": true,
    "initiatePaymentsOptions": { "paymentMethod": "CARD" }
  }
}
```

`enforceTokenization: true` is what makes the card reusable. Without it stage 2 has
nothing to charge, and you will not find out until the first renewal.

When the payment completes, store the token against your customer record. Store the
token, not the card. You will never see the card.

## Stage 2: charge from your backend

Create an order against the `MerchantInitiated` terminal, referencing the stored token.
No customer, no redirect, no page.

Exact request shape: `references/guides/server-to-server-api.md`.

## Getting this right in production

- **Idempotency.** A retried renewal must not double-charge. Key each charge on your own
  billing period identifier and check before charging again.
- **Declines are normal.** Expired cards, insufficient funds, and issuer blocks are
  routine at renewal. Build a retry schedule and a dunning path; do not treat a decline
  as a system error.
- **Card expiry.** Tokens outlive cards. Have a route for the customer to re-present a
  card, which means running stage 1 again.
- **Never store a PAN.** If a design requires you to, the design is wrong.
- **Amounts are minor units, currency is numeric.** `29900` and `"752"`.

## B2B invoices

Invoice-based billing rather than card-on-file is a separate flow.
`references/guides/b2b-invoices.md`.

## Verify before reporting success

Run stage 1 in Demo to completion, confirm a token came back, then run one stage 2
charge against it and show the user both order IDs. An untested recurring integration
fails at the first renewal, in production, a month after everyone stopped looking.

## Bundled guides

`references/guides/` holds: server-to-server API, recurring payments, tokens, B2B invoices.
