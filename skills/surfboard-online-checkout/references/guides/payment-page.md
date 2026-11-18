---
title: "Payment Page"
source: https://www.surfboardpayments.com/developers/guides/payment-page
category: online
tags: [Online, API, Payment Page, Payment Link, Hosted Checkout, Orders]
generated: true
---

# Payment Page

> Take online payments on a Surfboard-hosted checkout page. Create an online store, verify its domains, create an order against the PaymentPage terminal it comes with, and redirect the customer or send them the link.

## Overview

The Payment Page is Surfboard's hosted checkout. You create an order against the store's `PaymentPage` terminal and the API returns a link to a page hosted by Surfboard. Redirect the customer to it from your own checkout, or send it by email, SMS or chat as a payment link. The customer pays there, is sent back to your site, and the card details never touch your infrastructure.

The order call is the same [Create Order](/developers/guides/create-an-order/) call you use in store. What changes is everything around it: the terminal must be an **online** terminal, that terminal must sit in an **online store**, and the store's domains must be verified before Surfboard will approve it. This guide walks the whole path once, then covers the `controlFunctions.online` block that shapes the page the customer lands on. If your online store is already approved, skip to [Step 4](#step-4-create-the-order).

If you want to render the card fields inside your own page instead of sending the customer away, use the [Self-Hosted Checkout](/developers/guides/self-hosted-checkout/) guide. If you want to charge a stored card from your backend with no customer present, see [Server-to-Server API](/developers/guides/server-to-server-api/).

## Prerequisites

1. A developer account at the [integration portal](https://integrate.surfboardpayments.com/signup)
2. A merchant that has completed onboarding and KYB
3. Control of the DNS, or of the web server, for the webshop domain you are going to register

Orders on the Payment Page are **customer-initiated transactions (CIT)**: the customer is present and completes the payment. **Merchant-initiated transactions (MIT)**, such as subscription renewals charged from your backend, can only be processed by a `MerchantInitiated` terminal. See [Server-to-Server API](/developers/guides/server-to-server-api/).

> **Demo environment:** only test cards work in demo. Real cards used in demo are voided automatically after 30 minutes and never settle.

## Step 1: Create an Online Store

Terminals live under stores, and an online terminal needs a store that carries an `onlineInfo` block. Create the store with the webshop details, or add `onlineInfo` to an existing physical store.

```json
POST /partners/:partnerId/merchants/:merchantId/stores
{
  "storeName": "Web Store",
  "email": "webstore@example.com",
  "phoneNumber": { "code": 46, "number": "701234567" },
  "address": "Drottninggatan 10",
  "city": "Stockholm",
  "zipCode": "103 16",
  "country": "SE",
  "onlineInfo": {
    "merchantWebshopURL": "https://shop.example.com",
    "termsAndConditionsURL": "https://shop.example.com/terms",
    "privacyPolicyURL": "https://shop.example.com/privacy"
  }
}
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "storeId": "81d64e7174dcb00b0f",
    "merchantId": "818712cdbcb670070e",
    "name": "Web Store",
    "merchantUrlDomainVerificationKey": "499470649f03b53fa1175659d4389743974710260b7f410313487e6062b3d559"
  },
  "message": "Store Created Successfully"
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `onlineInfo.merchantWebshopURL` | Yes | The URL of the merchant's webshop. Verified by DNS. |
| `onlineInfo.termsAndConditionsURL` | Yes | The webshop's terms and conditions. Must include the refund policy. Shown on the Payment Page. |
| `onlineInfo.privacyPolicyURL` | Yes | The webshop's privacy policy. Shown on the Payment Page. |
| `onlineInfo.paymentPageHostURL` | No | The URL that hosts the payment page. Only required for [Self-Hosted Checkout](/developers/guides/self-hosted-checkout/) (SDK mode), not for the Payment Page. Verified by DNS when present. |
| `onlineInfo.googlePayMerchantId` | No | Your Google Pay merchant identifier, used to enable Google Pay on the online store. Not needed for the Payment Page. |

To turn an existing physical store into an online store instead, send the same `onlineInfo` block to [Update Store Details](https://developers.surfboardpayments.com/api/stores). The response carries the same verification keys:

```json
PUT /partners/:partnerId/merchants/:merchantId/stores/:storeId
{
  "onlineInfo": {
    "merchantWebshopURL": "https://shop.example.com",
    "termsAndConditionsURL": "https://shop.example.com/terms",
    "privacyPolicyURL": "https://shop.example.com/privacy"
  }
}
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "merchantUrlDomainVerificationKey": "499470649f03b53fa1175659d4389743974710260b7f410313487e6062b3d559"
  },
  "message": "Store data updated successfully"
}
```

`paymentPageUrlDomainVerificationKey` is only returned when you send a `paymentPageHostURL`.

> **Online info can only be set once.** Get these URLs right before you send the call; they cannot be edited afterwards through the same route.

Terms, privacy policy and contact details must also be visible on the webshop itself. This is an acquiring requirement, not a Surfboard preference.

## Step 2: Verify the Domains

The response carries one verification key per URL. Publish each as a **TXT record** on the matching domain, or, if you can't edit the domain's DNS, serve it as plain text at `https://<your-domain>/.well-known/sb-domain-verification.txt` (see [Online Store Domains](/developers/guides/online-store-domains/#publish-the-verification-key) for both formats). Then ask Surfboard to check it:

```json
POST /partners/:partnerId/merchants/:merchantId/stores/:storeId/verify
{
  "domainType": "MERCHANT_WEBSHOP_URL"
}
```

If you also registered a `paymentPageHostURL` for Self-Hosted Checkout, repeat with `"domainType": "PAYMENT_PAGE_HOST_URL"`. Surfboard also re-checks automatically every six hours, so a record or file published late is picked up without another call.

Verification is what unlocks the online terminal types: until the webshop domain passes, there is nothing to register a terminal against. Once it passes, the store goes through an internal approval step at Surfboard. Poll the store to see where it stands:

```
GET /partners/:partnerId/merchants/:merchantId/stores/:storeId/online
```

Approval usually takes 30 minutes to an hour once the domains are verified. See [Online Store Domains](/developers/guides/online-store-domains/) for how verification and approval work, including sharing one domain across stores, and [Store Management](/developers/guides/store-management/) for the full store lifecycle.

## Step 3: Pick Up the Terminal You Already Have

An online terminal is a mode, not a device — and for payment links you do not have to create one. Creating the online store provisions two terminals by default: a `PaymentPage` terminal, which is the one this guide uses, and a `MerchantInitiated` terminal for backend charges against a stored token. List the store's terminals and take the ID:

```
GET /partners/:partnerId/merchants/:merchantId/stores/:storeId/terminals
```

```json
// Response
{
  "status": "SUCCESS",
  "data": [
    {
      "terminalId": "813ca2cb12ce400405",
      "terminalType": "PaymentPage",
      "terminalStatus": "ACTIVE",
      "storeId": "81d64e7174dcb00b0f"
    },
    {
      "terminalId": "813ca2cb12ce400406",
      "terminalType": "MerchantInitiated",
      "terminalStatus": "ACTIVE",
      "storeId": "81d64e7174dcb00b0f"
    }
  ],
  "message": "Terminals fetched successfully"
}
```

Store the `PaymentPage` `terminalId` against something identifiable in your system — it is the `terminal$id` every order in this guide is created against.

| Mode | Use it for | Provisioned with the store |
|------|------------|----------------------------|
| `PaymentPage` | The hosted Payment Page and payment links. This guide. | Yes |
| `MerchantInitiated` | Backend charges against a stored token, such as subscription renewals. | Yes |
| `SelfHostedPage` | Card fields rendered on your own page by the Online SDK. Returns a `publicKey` and `sdkUrl`. | No |
| `iFrame` | An embedded payment frame inside your site. | No |

The two default terminals exist from the moment the store does, but they cannot take a payment until the domains verify and the store is approved. The other two modes are registered when you need them, and a store can hold as many as you like:

```json
POST /merchants/:merchantId/stores/:storeId/online-terminals
{
  "onlineTerminalMode": "SelfHostedPage"
}
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "terminalId": "813ca2cb12ce400407",
    "registrationStatus": "REGISTERED"
  },
  "message": "Terminal registered successfully"
}
```

## Step 4: Create the Order

Now the familiar call. The `terminal$id` is the `PaymentPage` terminal from step 3, and the response carries the link.

```json
POST /orders
{
  "terminal$id": "813ca2cb12ce400405",
  "referenceId": "order-2026-0418",
  "customer": {
    "person": {
      "name": { "firstName": "John", "lastName": "Doe" },
      "email": "john@example.com",
      "phoneNumber": { "code": "46", "number": "768100190" }
    }
  },
  "orderLines": [
    {
      "id": "ITEM-001",
      "name": "Annual Subscription",
      "quantity": 1,
      "amount": {
        "regular": 99900,
        "total": 99900,
        "currency": "752",
        "tax": [{ "amount": 19980, "percentage": 25, "type": "VAT" }]
      }
    }
  ],
  "totalOrderAmount": {
    "regular": 99900,
    "total": 99900,
    "currency": "752",
    "tax": [{ "amount": 19980, "percentage": 25, "type": "VAT" }]
  },
  "controlFunctions": {
    "initiatePaymentsOptions": {
      "paymentMethod": "CARD",
      "amount": 99900
    },
    "online": {
      "paymentPageValidFor": "2h",
      "enforce3DSecure": true,
      "generateShortLink": true,
      "payButtonType": "PAY",
      "redirectUrl": "https://shop.example.com/thanks",
      "failureRedirectUrl": "https://shop.example.com/checkout/failed"
    }
  }
}
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "orderId": "8455c12f9fd0620a010b",
    "paymentPageLink": "https://pay.withsurfboard.com/8455c12f9fd0620a010b?pi=Dr4GoyMXF0zHvjca_Oa0vHgxcT-OD1qp7KdokyI7dkTwRwYJt8nkXyQm3bT6vqCfgraOw50Bf5uOp__3ckbMWOV6L9QbiTiSEFS3YmF4Eb8lr5pTWP2KFjm9Ukmd0000&add=IzFlNDBhZg==",
    "shortLinkUrl": "https://srfb.se/Iq4JfPgHL"
  },
  "message": "Order created successfully"
}
```

`paymentPageLink` is the page to send the customer to. Treat it as opaque and pass it on whole: the query string carries the payment intent, and a link with a trimmed or re-encoded `pi` will not open. The one thing you can add is a `features` parameter, see [Customize the Page from the Link](#customize-the-page-from-the-link). `shortLinkUrl` appears only when you asked for it with `generateShortLink`, and is the one to put in an SMS.

Keep the `orderId`. Everything afterwards is keyed on it, and unlike an in-store order there is no `paymentId` yet: nothing has been attempted until the customer opens the page. The `paymentId` arrives with the first attempt, in the status response and in the webhook.

Line items, tax, adjustments, customer objects and the order-level calculation rules all behave exactly as they do in store. [Create an Order](/developers/guides/create-an-order/) covers them in full.

## Control Functions for Online Orders

Everything that shapes the page itself -- where the customer lands afterwards, how long the link lives, 3-D Secure, card saving, subscriptions -- goes in `controlFunctions.online`.

| Field | Type | Description |
|-------|------|-------------|
| `paymentPageValidFor` | string | How long the link works, as `<number><unit>` where the unit is `m`, `h` or `d` — for example `15m`, `2h`, `3d`. Defaults to one day. |
| `redirectUrl` | string | Where the customer lands after a successful payment. The `orderId` is appended as a query parameter. |
| `failureRedirectUrl` | string | Where the customer lands after a failure. Also carries the `orderId`. |
| `generateShortLink` | boolean | Returns `shortLinkUrl` alongside the full link. Default `false`. |
| `payButtonType` | string | The label on the button: `PAY`, `DONATE`, `BOOK`, `ORDER`, `CHECKOUT`, `CONTINUE`, `CONTRIBUTE`, `ADD_MONEY`, `RENT`, `SUPPORT`, `TIP`, `TOP_UP`. |
| `enforce3DSecure` | boolean | Force a 3-D Secure challenge where the issuer supports it. Leave it unset if you want frictionless payments to stay possible. |
| `relaxed3ds` | boolean | Ask for fewer customer details on card payments. See [Collect fewer customer details](#collect-fewer-customer-details). |
| `addressRequirements` | boolean | Ask for an address on the page. |
| `enforceTokenization` | boolean | Save the card for later use, overriding the terminal configuration. |
| `tokenisationIfPossible` | boolean | Tokenize when supported, but do not fail the payment if it is not. |
| `errorIfTokenizationFails` | boolean | Fail the flow when the card cannot be tokenized. |
| `subscription` | boolean | Mark the order as recurring-capable. |
| `recurring` | object | The subscription schedule. See [Recurring Orders](#recurring-orders). |
| `accountNameVerification` | string | Account-name verification mode applied during the payment. |
| `generateOnlineLinkWith` | string | Generate the link with a different terminal than the one the order was created against. |
| `currency` | string | Currency as a numeric ISO 4217 code, for example `752` for SEK. |
| `selfCardCharging` | boolean | Let the customer charge their own card. |

These sit directly on `controlFunctions`, next to `online`. They are not online-specific but matter here:

| Field | Type | Description |
|-------|------|-------------|
| `initiatePaymentsOptions` | object | Start the payment with a given `paymentMethod`, for example `CARD`. |
| `lockToPaymentMethods` | array | Restrict the page to the methods you list, e.g. `["CARD", "KLARNA"]`. |
| `authMode` | string | `AUTH` (default) charges the card when the customer pays. `PRE-AUTH` only holds the amount and sets `delayCapture` for you; you capture the final amount, lower or higher, later. |
| `delayCapture` | boolean | Authorize now, capture later. See [Capture a Payment](/developers/guides/capture-a-payment/). |
| `delayPayout` | string | Hold the payout for a period, as `<number><unit>`. |
| `cancelPreviousPendingOrder` | boolean | Cancel all previous pending orders when this one is created. |
| `callBackUrl` | string | Per-order webhook URL for this order and its payments. |

### Save the Card for Later

Tokenize the card during checkout and require 3-D Secure:

```json
"controlFunctions": {
  "initiatePaymentsOptions": {
    "paymentMethod": "CARD"
  },
  "online": {
    "enforceTokenization": true,
    "enforce3DSecure": true,
    "redirectUrl": "https://shop.example.com/thanks",
    "failureRedirectUrl": "https://shop.example.com/checkout/failed"
  }
}
```

### Collect Fewer Customer Details

By default a card payment asks for the customer's email, phone and full billing address. Turn on `relaxed3ds` and the page only needs email or phone, plus postal code and country code. Street address and city are not required:

```json
"controlFunctions": {
  "online": {
    "relaxed3ds": true,
    "redirectUrl": "https://shop.example.com/thanks",
    "failureRedirectUrl": "https://shop.example.com/checkout/failed"
  }
}
```

> **Warning:** Don't prefill the order with generated or placeholder customer details to skip the fields. The details go to the card issuer, who decides how to treat the payment, and fake data lowers 3-D Secure approval rates and goes against card scheme rules.

### Authorize Now, Capture on Shipping

Pre-authorize the amount and capture it when the order ships. `authMode: "PRE-AUTH"` turns on `delayCapture` for you:

```json
"controlFunctions": {
  "authMode": "PRE-AUTH",
  "lockToPaymentMethods": ["CARD"],
  "online": {
    "redirectUrl": "https://shop.example.com/thanks",
    "failureRedirectUrl": "https://shop.example.com/checkout/failed",
    "paymentPageValidFor": "30m"
  }
}
```

### Recurring Orders

For a subscription, add the `recurring` object inside `online` and mark the order as one:

```json
{
  "controlFunctions": {
    "online": {
      "subscription": true,
      "enforceTokenization": true,
      "recurring": {
        "subscriptionAmountType": "fixed",
        "frequency": "monthly",
        "numberOfPayments": 12,
        "uniqueReference": "sub-4471",
        "validation": "validated"
      }
    }
  }
}
```

The first payment is a customer-initiated transaction on the page, which is where the card is tokenized and 3-D Secure is satisfied. Every renewal after that is a merchant-initiated transaction against the stored token, charged from your backend through a `MerchantInitiated` terminal. [Recurring Payments](/developers/guides/recurring-payments/) has the renewal side.

| Field | Description |
|-------|-------------|
| `subscriptionAmountType` | `fixed` or `variable`. |
| `maxAmount` | Ceiling in minor units, for `variable` subscriptions only. |
| `frequency` | `daily`, `twiceWeekly`, `weekly`, `tenDays`, `fortNightly`, `monthly`, `everyTwoMonths`, `trimester`, `quarterly`, `twiceYearly`, `annually`, `unscheduled`. Required. |
| `numberOfPayments` | How many payments the schedule expects. |
| `uniqueReference` | Your reference for the recurring order. |
| `validation` | `validated` or `notValidated`. |

## Step 5: Confirm the Payment

The redirect back to your site tells you the customer finished, not that the money moved. Confirm server-side, either by polling or, better, by subscribing to the webhook.

```
GET /orders/:orderId/status
```

```json
// Response
{
  "status": "SUCCESS",
  "data": {
    "orderStatus": "PAYMENT_COMPLETED",
    "payments": [
      {
        "paymentId": "83a1ba3264bd500106",
        "paymentStatus": "PAYMENT_COMPLETED",
        "paymentMethod": "CARD",
        "amount": 99900
      }
    ]
  }
}
```

The states are the same as in store. An order sits in `PENDING` while the link is unused, and a failed or cancelled payment leaves it there, so the same `orderId` can be paid again without creating a new order.

| Order status | Meaning |
|--------------|---------|
| `PENDING` | The link has not been paid yet, or the last attempt failed or was cancelled. |
| `PAYMENT_COMPLETED` | Paid in full. The order is closed. |
| `PARTIAL_PAYMENT_COMPLETED` | Part of the total is paid. See [Partial Payments](/developers/guides/partial-payments/). |
| `PAYMENT_CANCELLED` | The order was cancelled. |

Subscribe to `order.paymentcompleted` and `order.paymentfailed` rather than polling on a timer. The payload carries the `orderId`, your `referenceId`, the amount and the transaction details, and each delivery carries a `metadata.eventId` you should use for idempotency — Surfboard retries a failed delivery twice, after 5 and 10 minutes.

```json
{
  "eventType": "order.paymentcompleted",
  "metadata": {
    "eventId": "831fc2f040bf405fff",
    "created": 1745821536443,
    "retryAttempt": 0,
    "terminalId": "813ca2cb12ce400405"
  },
  "data": {
    "orderId": "83a1ba32774149710b",
    "referenceId": "order-2026-0418",
    "paymentId": "83a1ba3264bd500106",
    "paymentStatus": "PAYMENT_COMPLETED",
    "paymentMethod": "CARD",
    "amount": "99900",
    "type": "PURCHASE"
  }
}
```

See [Webhooks](/developers/guides/webhooks-notifications/) for subscription and signature verification.

## Sending the Link

The link is a URL, so how it reaches the customer is your call:

- **Redirect** from your own checkout. The customer clicks "Pay", your backend creates the order, and you send the browser to `paymentPageLink`. After paying, the customer lands on your `redirectUrl` or `failureRedirectUrl`.
- **Email or SMS** for invoices, deposits and quotes. Use `shortLinkUrl` in an SMS and keep `paymentPageValidFor` short enough that a stale link cannot be paid by mistake.
- **QR code** printed or shown on screen, for pay-at-table and self-service.

Two things to hold on to. Set `paymentPageValidFor` deliberately — a link that lives for three days is a link someone can pay three days late, after you have cancelled the order. And never treat the arrival at `redirectUrl` as proof of payment: a customer can reach that URL by other means. The webhook and the status call are the record.

## Customize the Page from the Link

A few display options are set on the link itself rather than on the order. Append a `features` parameter to `paymentPageLink`, leaving the rest of the link untouched, as a comma-separated list of `key:value` pairs:

```
https://pay.withsurfboard.com/8455c12f9fd0620a010b?pi=...&add=...&features=method:SSWISH,show_footer:off,lang:en
```

| Option | What it does |
|--------|--------------|
| `method:<ID>` | Puts that payment method first in the list. This is the only ordering control. |
| `show_footer:off` | Hides the footer. |
| `show_top:off` | Hides the top block. |
| `minimal:on` | Hides the sidebar and the extras in the payment footer. The sidebar still shows on the success page. |
| `lang:<code>` | Sets the page's default language: `en`, `sv`, `da`, `fi` or `no`. |
| `address_autofill:off` | Turns off Google address autofill in the billing address form. |

This only works on the full `paymentPageLink`. The options can't be applied to `shortLinkUrl`, so if you want them in an SMS or a QR code, send the full link instead.

`method` takes the ID of any payment method the page supports, for example `CARD`, `KLARNA`, `SSWISH`, `SVIPPS` or `SMOBILEPAY`. New methods get their own ID as they are added to the page, so this list is not exhaustive. The method is only moved to the top if it is available for the order; otherwise the page keeps its default order, which puts the method the customer chose last time first and then card. To limit which methods appear at all, use `lockToPaymentMethods` on the order instead.

Unknown keys and unsupported `lang` values are ignored. The options apply when the page opens: if the customer picks another payment method or switches language themselves, their choice wins.

## Error Handling

Create-order failures come back as `status: "ERROR"` with an `OR_*` or `PS_*` code. The ones you will meet setting this up:

| Code | Cause |
|------|-------|
| `OR_0042` | Terminal not found. The `terminal$id` is wrong, or it belongs to another merchant. |
| `OR_0037` | The total does not reconcile with the line items. |
| `OR_0048` | Line items mix currencies. |

If the store's terminal list comes back without a `PaymentPage` entry, the store was created without `onlineInfo` — the two default terminals only come with an online store. A terminal that will not register, or one that is there but refuses a payment, usually means the store has not cleared domain verification or is still in approval. Check the store's online status before you look at the terminal call.

The [Create Order Error Codes](/developers/guides/create-order-error-codes/) reference lists the rest, including errors raised by the payment initiation that happens inside the same call.

## Next Steps

- [Self-Hosted Checkout](/developers/guides/self-hosted-checkout/) — keep the customer on your own page
- [Server-to-Server API](/developers/guides/server-to-server-api/) — charge a stored card with no customer present
- [Capture a Payment](/developers/guides/capture-a-payment/) — finalize a delayed-capture authorization
- [Refund an Order](/developers/guides/refund-an-order/) — return funds after settlement

## Reference

- [Stores API](https://developers.surfboardpayments.com/api/stores)
- [Terminals API](https://developers.surfboardpayments.com/api/terminals)
- [Create Order API](https://developers.surfboardpayments.com/api/orders)
- [Payments API](https://developers.surfboardpayments.com/api/payments)
- [Integration portal](https://integrate.surfboardpayments.com/)
