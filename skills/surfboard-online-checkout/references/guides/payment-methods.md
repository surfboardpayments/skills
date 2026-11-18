---
title: "Payment Methods"
source: https://www.surfboardpayments.com/developers/guides/payment-methods
category: online
tags: [Online, API, Payment Methods, In-Store]
generated: true
---

# Payment Methods

> Activate, deactivate, and list payment methods for a merchant. Manage card, Swish, Klarna, AMEX, Vipps, MobilePay, and more via the API or Partner Portal.

## Overview

Card payments are enabled by default for all merchants. Use the Payment Methods API to activate or deactivate additional methods like Swish, Klarna, AMEX, Vipps, MobilePay, B2B invoicing, and account-to-account transfers. Payment methods can be set at the merchant or store level.

## Available Payment Methods

| Method | Parameter | Description |
|--------|-----------|-------------|
| Card | `card` | Visa, Mastercard (enabled by default) |
| AMEX | `amex` | American Express |
| Swish | `swish` | Swedish mobile payments |
| Klarna | `klarna` | Buy now, pay later |
| B2B Invoice | `b2binv` | B2B invoice payments |
| Account-to-Account | `acc2acc` | Bank transfer |
| Vipps | `svipps` | Norwegian mobile payments |
| MobilePay | `smobilepay` | Danish mobile payments |

> **Note:** Payout timing varies by payment method. Klarna, account-to-account and Epassi pay out at T+7. MobilePay, Vipps and Swish online with phone number (`SSWISH`) pay out at T+3. American Express usually pays out at T+3, but this can differ in some markets. Swish in-store with QR code (`NSWISH`) pays out at T+1. See [When Payouts Arrive](/developers/guides/settlements-reporting/#when-payouts-arrive).

## Payment Methods in an Online Checkout

In an online checkout, each method has its own setup beside the checkout code. Activation is done here. The domain steps are done on the store.

| Method | How it is switched on | What you do in the checkout |
|--------|-----------------------|-----------------------------|
| Card | On by default | Send the customer's email, phone and billing address, or use `relaxed3ds` to collect less. |
| Apple Pay | Registered by Surfboard for the domain when the online store is approved | Host the domain association file on the checkout domain. See [Apple Pay](/developers/guides/self-hosted-checkout/#apple-pay). |
| Google Pay | Your own Google Pay merchant ID, approved by Google for your domains | Set `onlineInfo.googlePayMerchantId` when you create the store. See [Google Pay](/developers/guides/self-hosted-checkout/#google-pay). |
| Swish | Activate `swish` for the merchant | Send the customer's phone number. |
| Klarna | Activate `klarna` for the merchant | Send email, phone and billing address, and a shipping address for physical goods. |

Apple Pay and Google Pay are not in the activation request below. They ride on the store's domain setup, so the place to look when a wallet button doesn't show is [Online Store Domains](/developers/guides/online-store-domains/#what-approval-unlocks), not this API.

### Testing in the Demo Environment

All of these can be tested in demo. You don't have to wait for production.

| Method | In demo |
|--------|---------|
| Card | Works as soon as the checkout does. |
| Apple Pay | Works once the store is approved and the domain association file is hosted on the checkout domain. Test in Safari. |
| Klarna | Activate it for the test merchant yourself, with the [activation request](#activate-payment-methods) or in the Partner Portal, as you would in production. |
| Swish | Ask Surfboard to set up Swish test credentials for your test merchant. This step is not automated, so activating `swish` on its own is not enough in demo. |

If Apple Pay fails on an approved store, the association file is almost always the cause. Check it the way Apple does:

```bash
curl -i https://shop.example.com/.well-known/apple-developer-merchantid-domain-association
```

The response should be `200`, with no redirect and the file unmodified. A `404` here means the button will fail no matter what the store's status is.

## Activate Payment Methods

Enable one or more payment methods for a merchant in a single request. Set each method to `true` to activate it.

```
POST /merchants/:merchantId/payment-methods
```

### Request

```json
{
  "card": true,
  "swish": true,
  "klarna": true,
  "b2binv": true
}
```

### Response

```json
{
  "status": "SUCCESS",
  "data": [
    {
      "method": "card",
      "paymentMethodId": "pm_abc123",
      "status": "SUCCESS"
    },
    {
      "method": "swish",
      "paymentMethodId": "pm_def456",
      "status": "SUCCESS"
    }
  ],
  "message": "Payment methods activated successfully"
}
```

Each activated method returns a `paymentMethodId` you'll need for fetching details or deactivating later.

### Via Partner Portal

1. Log in to the **Partner Portal** > **Merchants** > select the merchant > **Payment Methods**
2. Select the payment methods you want to enable
3. Click **Activate Selected Methods**

## List All Payment Methods

Retrieve all available payment methods and their activation status for a merchant.

```
GET /merchants/:merchantId/payment-methods
```

### Response

```json
{
  "status": "SUCCESS",
  "data": [
    {
      "paymentMethodId": "pm_abc123",
      "paymentMethod": "CARD"
    },
    {
      "paymentMethodId": "pm_def456",
      "paymentMethod": "SWISH"
    }
  ],
  "message": "Payment methods retrieved successfully"
}
```

## Get Payment Method Details

Fetch the configuration and status of a specific payment method.

```
GET /merchants/:merchantId/payment-methods/:paymentMethodId
```

### Response

```json
{
  "status": "SUCCESS",
  "data": {
    "paymentMethodId": "pm_abc123",
    "paymentMethod": "CARD",
    "status": "ACTIVATED",
    "acquirerMID": "12345678"
  },
  "message": "Payment method retrieved successfully"
}
```

The `status` field is either `ACTIVATED` or `DEACTIVATED`. For AMEX, the response includes an `amexMID`. For card payments, it includes an `acquirerMID`.

## Deactivate a Payment Method

Remove a payment method from a merchant using its `paymentMethodId`.

```
DELETE /merchants/:merchantId/payment-methods/:paymentMethodId
```

### Response

```json
{
  "status": "SUCCESS",
  "message": "Payment method deactivated successfully"
}
```

### Via Partner Portal

1. Log in to the **Partner Portal** > **Merchants** > select the merchant > **Payment Methods**
2. Find the method to remove and click the **Delete** icon

## Store-Level Configuration

You can scope payment methods to a specific store by including `storeId` in the activation request:

```json
{
  "storeId": "YOUR_STORE_ID",
  "swish": true,
  "klarna": true
}
```

This lets different stores under the same merchant accept different payment methods.
