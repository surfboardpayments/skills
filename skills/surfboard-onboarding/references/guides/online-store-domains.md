---
title: "Online Store Domains"
source: https://www.surfboardpayments.com/developers/guides/online-store-domains
category: online
tags: [Online, Domain Verification, Stores, Approval, Apple Pay, CORS]
generated: true
---

# Online Store Domains

> Verify the domains of an online store, share a domain across stores and merchants, and know what store approval unlocks for payments, the Online SDK and Apple Pay.

## Overview

Every online store registers one or two URLs: the merchant's webshop (`merchantWebshopURL`) and, for [Self-Hosted Checkout](/developers/guides/self-hosted-checkout/), the page the Online SDK runs on (`paymentPageHostURL`). Before the store can take a payment, Surfboard has to confirm you control those domains and then approve the store for online payments.

This guide covers that path from a new domain to a live store: publishing the verification key, how far one verification reaches, sharing a domain between stores and merchants, approval, and what approval switches on. It ends with the problems you see when a domain is not live yet.

The path in short:

1. Create the online store with its URLs. See [Create an Online Store](/developers/guides/store-management/#create-an-online-store).
2. Publish the verification key for each domain, as a DNS TXT record or an HTTPS file.
3. Trigger verification, or wait for the automatic check.
4. Surfboard approves the store for online payments.
5. Payments, the Online SDK and Apple Pay start working on the domain.

## Prerequisites

- An online store, created with `onlineInfo` (see [Store Management](/developers/guides/store-management/))
- Control of the DNS, or of the web server, for each domain you registered on the store
- API credentials (API key and API secret)

## How Verification Works

Verification is tied to the **domain**, not to the store.

- **One key per domain.** The key belongs to the domain name. If the webshop and payment page URLs of a store are on the same domain, [Fetch Store Domains](https://developers.surfboardpayments.com/api/stores) returns the same key for both.
- **Once per partner.** A domain only has to be verified once under your partner account. When you register an already verified domain on another store, under the same merchant or a different one, it is verified automatically. There is nothing to publish or trigger again.
- **Exact host.** The domain is the host of the URL you registered. `www.shop.example.com` and `shop.example.com` are different hosts, and wildcards are not supported.

This is what makes it practical to host checkout for many merchants: verify `pay.example.com` once, and every store you point at it is covered.

## Publish the Verification Key

The keys are returned when the store is created, and you can fetch them again at any time with [Fetch Store Domains](https://developers.surfboardpayments.com/api/stores):

```
GET /partners/:partnerId/merchants/:merchantId/stores/:storeId/online
```

```json
{
  "status": "SUCCESS",
  "data": {
    "onlineStoreStatus": "INITIATED",
    "domains": [
      {
        "status": "ACTIVE",
        "type": "PAYMENT_PAGE_HOST_URL",
        "name": "pay.example.com",
        "verificationKey": "7fd242dd862a9f9f74ae11d62216f5729a5e5600e00b8d44300db41b4efcce68",
        "isDomainVerified": false
      },
      {
        "status": "ACTIVE",
        "type": "MERCHANT_WEBSHOP_URL",
        "name": "shop.example.com",
        "verificationKey": "7fd242dd862a9f9f74ae11d62216f5729a5e5600e00b8d44300db41b4efcce68",
        "isDomainVerified": false
      }
    ]
  },
  "message": "Online store domain details fetched successfully."
}
```

Each entry in `domains` carries the domain `name` to verify and its `verificationKey`. Publish each key in one of two ways. You only need one.

### Option A: DNS TXT record

Add the verification key as a **TXT record** in the domain's DNS configuration.

<!-- TODO: confirm with the platform team which record name the check looks up (the registered host itself, e.g. pay.example.com, or the root domain) and whether the value includes the sb-domain-verification-key= prefix. Partners have asked for both. -->

### Option B: HTTPS file

If you can serve pages on the domain but can't edit its DNS, serve the key from a well-known file instead. This suits platforms that host a merchant's custom domain while the DNS stays with the merchant's own provider: you can publish the file for every merchant automatically, with no action from them.

Serve the file on the same host as the URL you registered on the store:

```
https://<your-domain>/.well-known/sb-domain-verification.txt
```

If the store's URL is `https://www.example.com`, serve it at `https://www.example.com/.well-known/sb-domain-verification.txt`. Serving it on the root domain, `https://example.com/.well-known/sb-domain-verification.txt`, also works.

- The URL you serve it on must return `200` directly over HTTPS, with no redirect. If your root domain redirects to `www`, serve the file on the `www` host.
- Serve it as plain text (`Content-Type: text/plain`), not JSON.
- The body is the verification key with its `sb-domain-verification-key=` prefix.
- If the store has two different keys on the same domain, put both in the same file, one per line.

```
sb-domain-verification-key=7fd242dd862a9f9f74ae11d62216f5729a5e5600e00b8d44300db41b4efcce68
```

Check it before you trigger verification:

```bash
curl -i https://www.example.com/.well-known/sb-domain-verification.txt
```

The response should be `200` with `content-type: text/plain` and the key in the body.

## Trigger Verification

Surfboard checks pending domains automatically every 6 hours, through either method, so a store waiting for verification is picked up without further action once the record or file is live. To verify right away, call [Verify Store Domain](https://developers.surfboardpayments.com/api/stores):

```
POST /partners/:partnerId/merchants/:merchantId/stores/:storeId/verify
```

```json
{
  "domainType": "MERCHANT_WEBSHOP_URL"
}
```

Use `MERCHANT_WEBSHOP_URL` for the webshop domain or `PAYMENT_PAGE_HOST_URL` for the payment page domain. If the store has both, call it once for each.

Fetch Store Domains shows the result per domain in `isDomainVerified`.

## Sharing a Domain Between Stores

Several stores can use the same domain, under the same merchant or under different merchants of your partner. A common setup is one checkout host, for example `pay.example.com`, serving many merchants.

What is fixed is the other direction: each store has one `paymentPageHostURL`, and the Online SDK only works on that exact host. If a merchant takes payments on more than one domain, for example their own webshop and a white-label site, create one online store per domain. See [One Store per Checkout Domain](/developers/guides/self-hosted-checkout/#one-store-per-checkout-domain).

## Store Approval

Once a store's domains are verified, Surfboard reviews the store for online payments. When verification is done, approval usually takes **30 minutes to an hour**.

If you register the exact same URL as an already approved store under the **same merchant**, the new store is approved automatically.

<!-- TODO: confirm whether a store of a different merchant under the same partner, using the exact same URL as an approved store, is also approved automatically. -->

Follow the status with Fetch Store Domains (`onlineStoreStatus`) or [Fetch Store Details](https://developers.surfboardpayments.com/api/stores) (`onlineOnboardingStatus`):

| Status | Meaning |
|--------|---------|
| `INITIATED` | Waiting for domain verification or approval |
| `APPROVED` | The store can take online payments |
| `FAILED` | The store was not approved. Contact Surfboard support |

## What Approval Unlocks

Approval is the switch for everything online on the store's domains:

- **Payments.** The store's online terminals exist from the moment the store is created, but they refuse payments until the store is approved. This applies to the Payment Page, payment links, the Online SDK and merchant-initiated charges.
- **Online SDK requests.** Surfboard adds the store's approved domains to the allowed origins for the Online SDK automatically. You don't need to request this.
- **Apple Pay.** Surfboard registers the domain with Apple Pay as part of approval. Your only part is hosting the domain association file on the payment page domain. See [Apple Pay](/developers/guides/self-hosted-checkout/#apple-pay).

Google Pay on your own website is the exception: Google approves your domains under your own Google Pay merchant ID. See [Google Pay](/developers/guides/self-hosted-checkout/#google-pay).

## Troubleshooting

**Online SDK calls fail with a CORS error.** The browser blocks the SDK's requests from your page before they reach Surfboard. The page's domain is not in the allowed origins yet, which means the store is not approved. Check `onlineStoreStatus`, and check that the page's host matches the store's `paymentPageHostURL` exactly, including `www`.

**Verification doesn't pass.** For the HTTPS file, run the `curl` check above: a redirect, a JSON content type or a missing prefix will fail it. For a TXT record, DNS changes can take a while to propagate. The automatic check runs every 6 hours, so you can also just wait.

**Orders are refused, or the terminal won't take a payment.** The store has not cleared verification or approval. Check its status before you look at the order call.

**The Apple Pay button doesn't show, or the payment fails.** Check that the store is approved, then check the domain association file with `curl -i https://<your-domain>/.well-known/apple-developer-merchantid-domain-association`. It has to return `200` on the exact host the SDK runs on, with no redirect. An approved store with a `404` on that path is the most common cause.

## API Quick Reference

| Operation | Method | Endpoint |
|-----------|--------|----------|
| Fetch store domains | GET | `/partners/:partnerId/merchants/:merchantId/stores/:storeId/online` |
| Verify domain | POST | `/partners/:partnerId/merchants/:merchantId/stores/:storeId/verify` |
| Fetch store details | GET | `/partners/:partnerId/merchants/:merchantId/stores/:storeId` |
