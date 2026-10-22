---
name: surfboard-online-checkout
description: "Add online payments with Surfboard Payments: hosted Payment Page, self-hosted checkout, and shareable payment links. Covers redirect and return handling, order creation for web, payment methods per market, and checkout branding. Use for e-commerce checkout, a pay-by-link flow, or taking a card payment on a website."
---

# Online checkout

Read `surfboard-payments` first.

## Pick the right one

| Option | Card data touches your systems | Use when |
|---|---|---|
| **Payment Page** (hosted) | No | Default. Redirect to Surfboard, come back paid. |
| **Payment Link** | No | Invoices, remote sales, no checkout UI at all. |
| **Self-hosted** | No, but you own the page | You need full control of the checkout UI. |

Start with the hosted Payment Page unless the user has a specific reason not to. It is
the least code and the smallest PCI surface.

## The terminals already exist

**An online store is provisioned with a `PaymentPage` terminal and a
`MerchantInitiated` terminal when it is created.** List the store's terminals and take
the IDs. Do not try to register them. Agents get this wrong constantly and waste a
cycle on a registration call that was never needed.

Only `SelfHostedPage` needs registering, and only if you are collecting the first
payment on your own page. See `surfboard-terminals`.

## Hosted Payment Page

Create the order against the `PaymentPage` terminal, then redirect the customer to the
URL that comes back.

```
POST {SURFBOARD_API_URL}/orders
```

```json
{
  "terminal$id": "YOUR_PAYMENT_PAGE_TERMINAL_ID",
  "orderLines": [
    {
      "id": "SKU-001",
      "name": "Annual licence",
      "quantity": 1,
      "amount": { "regular": 129900, "total": 129900, "currency": "752" }
    }
  ],
  "totalOrderAmount": { "regular": 129900, "total": 129900, "currency": "752" },
  "controlFunctions": {
    "initiatePaymentsOptions": { "paymentMethod": "CARD" }
  }
}
```

Amounts are minor units. Currency is `"752"`, not `"SEK"`.

Three things the examples hide, because every documented example uses `quantity: 1`:

- **`amount.total` is the price of one unit**, not the line. The order total is
  `sum(total * quantity)`.
- **`amount.tax` is required on every line**, including zero-VAT lines. Omitting it
  returns `P_0001 ... reading 'vatValue'`.
- **Prices include tax.** `totalOrderAmount.total` must equal `regular`. Never add tax
  on top. See `surfboard-payments`.


Keep `orderId` against your own order record **before** redirecting. The customer may
close the tab, and the return URL is not a reliable delivery mechanism.

## Handling the return

The return URL tells you the customer came back. It does not tell you they paid.

**Confirm payment server-side before fulfilling anything.** Fetch the payment status,
or better, act on the webhook (`surfboard-webhooks`). Treat the redirect as a UI event
and the webhook as the source of truth. A customer who closes the tab after paying
must still get their order.

Handle these separately:

- **Success**: payment confirmed server-side, fulfil
- **Cancelled**: customer backed out, order still open
- **Failed**: declined, offer a retry
- **Never returned**: webhook fulfils it anyway

## Payment methods

Available methods vary by market. `CARD` is everywhere; `SWISH` is Sweden; `KLARNA`
and local wallets depend on the merchant's country. Do not hard-code a method list;
see `references/guides/payment-methods.md` and read it per store.

## Self-hosted checkout

You render the checkout; Surfboard still handles card entry through an embedded
component, so card data never reaches your server. Uses a per-order nonce in
`X-Surfboard-Nonce` rather than your key and secret.

`references/guides/self-hosted-checkout.md`.

## Payment links

No checkout UI at all: generate a URL, send it to the customer, get notified when it
is paid. Good for invoicing and remote sales. The Payment Page guide covers this: the
order call returns the link, which you can send instead of redirecting.

`references/guides/payment-page.md`.

## Branding

Logo, colours, and copy on the hosted page are configurable per partner, merchant,
store, and terminal, with the more specific level overriding the broader one.

`references/guides/partner-branding.md`.

## Split payouts and gift cards

To send a share of the payment to a third party, add it under
`controlFunctions.serviceProviders` on the order. The provider has to be onboarded and
linked to the merchant first: `references/guides/service-providers.md`. Gift cards and
promotions are issued through their own APIs: `references/guides/gift-cards-promotions.md`.

## Verify before reporting success

Create an order in Demo, open the payment page URL, complete a test card payment, and
confirm you observed the status server-side. Show the user the `orderId` and the final
status.

## Bundled guides

`references/guides/` holds: payment page, self-hosted checkout, payment links, create an
order, payment methods, partner branding, create order error codes, service providers
and split payouts, gift cards and promotions.
