---
title: "Self-Hosted Checkout"
source: https://www.surfboardpayments.com/developers/guides/self-hosted-checkout
category: online
tags: [Online, SDK, JavaScript, Self-Hosted, Checkout]
generated: true
---

# Self-Hosted Checkout

> Embed a payment form directly in your web app with the Surfboard Online SDK. Full UI control with Surfboard handling PCI compliance.

## Overview

The Self-Hosted Checkout (Online SDK) lets you embed payment fields directly in your website while Surfboard handles PCI compliance and payment processing. You get full control over the look and feel of your checkout page.

Surfboard renders secure input fields inside your page using the Online SDK. You control the layout, branding, and customer experience.

## Prerequisites

1. Create a developer account at the [Developer Portal](https://developers.surfboardpayments.com/sign-up)
2. Complete onboarding (merchant and store setup)
3. Register a terminal with the type set to **SelfHostedPage**
4. Note the terminal's `publicKey` from the registration response

### One Store per Checkout Domain

The SDK sends the host of the page it runs on, and Surfboard checks it against the store's `onlineInfo.paymentPageHostURL`. The check is an exact hostname match: `www.shop.example.com` and `shop.example.com` are different hosts, and wildcards are not supported.

If a merchant takes payments on more than one domain, for example their own webshop and a white-label site, create one online store per domain, each with its own `paymentPageHostURL` and its own SelfHostedPage terminal.

Each `paymentPageHostURL` has to pass domain verification before the store can take payments. If you host checkout on merchants' custom domains but their DNS sits with their own provider, you don't need them to add a TXT record: serve the verification key from `https://<your-domain>/.well-known/sb-domain-verification.txt` instead. See [Domain Verification](/developers/guides/store-management#domain-verification) for the file format.

> **Tip:** You can retrieve the terminal's public key later using the [Fetch Terminal by ID API](https://developers.surfboardpayments.com/api/terminals). It is returned as `terminalPublicKey` in the response.

## Initializing the SDK

To initialize the SDK, you need three parameters:

1. **`publicKey`** -- From terminal registration
2. **`orderId`** -- From the [Create Order API](https://developers.surfboardpayments.com/api/orders) response
3. **`nonce`** -- From the Create Order response (serves as access control)

```javascript
// Set up error handling
SurfboardOnlineSDK.errorCallback((code, message) => {
    console.error(`Error [${code}]: ${message}`);
});

// Listen for payment status changes
SurfboardOnlineSDK.paymentStatusCallback = function (data) {
    // data.paymentStatus: 'PAYMENT_INITIATED' | 'PAYMENT_COMPLETED' |
    //   'PAYMENT_CANCELLED' | 'PAYMENT_FAILED' | 'PAYMENT_PROCESSING'
    console.log("Payment status:", data.paymentStatus);
};

// Initialize
SurfboardOnlineSDK.initialiseOnlineSDK({
    publicKey: "YOUR_PUBLIC_KEY",
    orderId: "YOUR_ORDER_ID",
    nonce: "YOUR_NONCE",
});
```

To re-initialize for a different order without a full page reload:

```javascript
SurfboardOnlineSDK.remountOnlineSDK({
    publicKey: "YOUR_PUBLIC_KEY",
    orderId: "NEW_ORDER_ID",
    nonce: "NEW_NONCE",
});
```

### Error Codes

| Error Code | Message | Category |
|------------|---------|----------|
| -- | Surfboard SDK cannot function in the given environment | FATAL |
| -- | Surfboard SDK initialisation failed | FATAL |
| -- | Public key validation failed | FATAL |
| -- | Invalid Order ID | FATAL |
| -- | Invalid Nonce | FATAL |
| 401 | Invalid or Expired Link | FATAL |

## Available Data Objects

After successful initialization, the SDK exposes data objects on `SurfboardOnlineSDK`:

- **`order`** -- Order details, line items, and payment methods
- **`merchant`** -- Merchant name and organization number
- **`branding`** -- Colors, fonts, logos for your checkout styling
- **`store`** -- Store contact info, privacy policy, and terms URLs
- **`paymentMethods`** -- Supported payment methods for this terminal
- **`customer`** -- Customer details, saved cards, and addresses

> **Warning:** You are required to display the store contact information, privacy policy, and terms and conditions on your payment page.

## Payment Flow

### Updating Customer Information

Most payment methods require customer information. Provide it via the SDK or include it when creating the order.

| Payment Method | Required Fields |
|----------------|----------------|
| Card | Email, Phone, Address (email or phone, postal code and country code with [`relaxed3ds`](#collecting-fewer-details-with-relaxed-3-d-secure)) |
| Klarna | Email, Phone, Address, Shipping Address (physical goods) |
| Apple Pay | Email, Name, Phone, Postal Address |

```javascript
await SurfboardOnlineSDK.order.addCustomerInformation({
    name: "Jane Doe",
    email: "jane@example.com",
    phone: { countryCode: "+46", number: "701234567" },
    billingAddress: {
        addressLine1: "Main Street 1",
        city: "Stockholm",
        postalCode: "11122",
        countryCode: "SE",
    },
});
```

> **Tip:** Include the customer address in the Create Order API request when you have it. This pre-fills the address so the customer does not need to enter it manually.

#### Collecting fewer details with relaxed 3-D Secure

If you don't want to ask for a full billing address on card payments, set `relaxed3ds` to `true` in `controlFunctions.online` when you [create the order](https://developers.surfboardpayments.com/api/orders):

```json
POST /orders
{
  ...,
  "controlFunctions": {
    "online": {
      "relaxed3ds": true
    }
  }
}
```

With `relaxed3ds` on, a card payment only needs email **or** phone, plus postal code and country code. Street address and city aren't needed:

```javascript
await SurfboardOnlineSDK.order.addCustomerInformation({
    email: "jane@example.com",
    billingAddress: {
        postalCode: "11122",
        countryCode: "SE",
    },
});
```

> **Warning:** Don't fill the customer fields with generated or placeholder data, including a fixed set of details per country. The details are passed to the card issuer, who decides how to treat the payment. Fake details lower 3-D Secure approval rates and go against card scheme rules. Use `relaxed3ds` instead.

### Card Payments

Mount the card input fields in your page:

```html
<div id="card-details"></div>
```

```javascript
SurfboardOnlineSDK.mount({
    mountCardWidget: "card-details",
});

// When the customer clicks "Pay":
await SurfboardOnlineSDK.order.initiatePayments("CARD");
```

### Swish Payments

```javascript
const paymentAttempt = await SurfboardOnlineSDK.order.initiatePayments("NSWISH");

// For mobile: redirect to Swish app
const redirectUrl = paymentAttempt.getSwishAppRedirectUrl("https://your-site.com/callback");

// For web: display QR code
const qrData = paymentAttempt.getSwishQRData;
```

### Apple Pay

```html
<div id="apple-pay"></div>
```

```javascript
SurfboardOnlineSDK.mount({
    mountApplePayWidget: "apple-pay",
});
// Payment is initiated automatically when the customer clicks the Apple Pay button
```

#### Domain verification

Apple Pay only shows on domains that host Surfboard's domain association file:

1. Download the [domain association file](https://firebasestorage.googleapis.com/v0/b/firebase-test-2e49.appspot.com/o/online-docs%2Fhost.txt?alt=media&token=a83e08cf-e178-4052-8829-dc9c7df414e5).
2. Host it at `https://<your-domain>/.well-known/apple-developer-merchantid-domain-association`.
3. Serve it as `text/plain`, publicly accessible, with no redirects.

The same file works for every merchant, in test and production. There is no separate production file, and you don't need your own Apple Pay certificates.

### Google Pay

When the SDK runs on your own website, Google Pay uses your own Google Pay merchant ID, and Google has to approve the domains your checkout runs on. Until that's done, pressing the Google Pay button fails with error `OR_BIBED_11`, which means Google doesn't recognise the page's domain for the merchant ID in use.

To set it up:

1. Sign up at the [Google Pay & Wallet Console](https://pay.google.com/business/console) and complete your business profile. This gives you a Google Pay merchant ID.
2. Go to **Google Pay API** → **Get started** / **Integrate with your website** and add the domains where your checkout runs.
3. Choose **Gateway** as the integration type, with `surfboard` as the gateway ID. Upload the screenshots of your checkout flow that Google asks for, and submit for approval.
4. Once Google approves you for production, set your merchant ID as `onlineInfo.googlePayMerchantId` on the store. The SDK picks it up automatically.

```json
POST /partners/:partnerId/merchants/:merchantId/stores
{
  ...,
  "onlineInfo": {
    "merchantWebshopURL": "https://shop.example.com",
    "paymentPageHostURL": "https://shop.example.com/checkout",
    "termsAndConditionsURL": "https://shop.example.com/terms",
    "privacyPolicyURL": "https://shop.example.com/privacy",
    "googlePayMerchantId": "YOUR_GOOGLE_PAY_MERCHANT_ID"
  }
}
```

> **Warning:** Online info can only be set once, whether you send it when [creating the store](https://developers.surfboardpayments.com/api/stores) or when turning a physical store into an online store with [Update Store Details](https://developers.surfboardpayments.com/api/stores). If you want Google Pay, get your Google Pay merchant ID before you send `onlineInfo`.

### Klarna

```javascript
await SurfboardOnlineSDK.order.addCustomerInformation({
    phone: { countryCode: "+46", number: "701234567" },
    name: "Jane Doe",
    email: "jane@example.com",
    billingAddress: {
        city: "Stockholm",
        postalCode: "11122",
        countryCode: "SE",
        addressLine1: "Main Street 1",
    },
});

await SurfboardOnlineSDK.order.initiatePayments("KLARNA");
```

## Styling the Card Fields

You can style the card input fields the SDK renders so they match the rest of your checkout. Each field and its label has its own class, which you can override in your own stylesheet:

| Field | Input class | Label class |
|-------|-------------|-------------|
| Card number | `sb-field-card` | `sb-field-label-card` |
| Cardholder name | `sb-field-name` | `sb-field-label-name` |
| Expiry date | `sb-field-expiryDate` | `sb-field-label-expiryDate` |
| CVC | `sb-field-cvc` | `sb-field-label-cvc` |

```css
.sb-field-card,
.sb-field-name,
.sb-field-expiryDate,
.sb-field-cvc {
    border: 1px solid #d0d5dd;
    border-radius: 8px;
    padding: 12px;
    font-family: inherit;
}

.sb-field-label-card,
.sb-field-label-name,
.sb-field-label-expiryDate,
.sb-field-label-cvc {
    font-size: 14px;
    color: #344054;
}
```

The `branding` data object gives you the merchant's colors, fonts, and logos if you want to style the fields from it.

> **Note:** The Apple Pay and Google Pay buttons can't be styled. Apple and Google own those buttons and render them according to their own guidelines.

## Payment Error Codes

| Code | Message | Category |
|------|---------|----------|
| ON_009 | Phone number required for Swish payment | Non Fatal |
| ON_010 | Payment method not supported for this order | Non Fatal |
| ON_011 | Payment already completed | Non Fatal |
| ON_012 | Error initiating payment -- retry | Non Fatal |
| ON_013 | Unknown error -- page reload may help | Non Fatal |
| ON_016 | Invalid card details | Non Fatal |
| ON_017 | Email required for this payment | Non Fatal |
| ON_018 | Billing address required for this payment | Non Fatal |

## Reference

- [React Sample App](https://github.com/surfboardpayments/react-next-online-sdk) for a complete working example
- [Create Order API](https://developers.surfboardpayments.com/api/orders)
- [Fetch Terminal API](https://developers.surfboardpayments.com/api/terminals)
- [Developer Portal](https://developers.surfboardpayments.com/)
