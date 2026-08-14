---
name: surfboard-payments
description: "Router and shared conventions for integrating Surfboard Payments: in-store card terminals, Tap to Pay, hosted online checkout, recurring billing, and merchant onboarding. Use whenever the task mentions Surfboard, surfpay, SurfTouch, SurfPad, SurfPrint, SurfMini, CheckoutX, or asks to add card payments, a payment terminal, or a checkout to a codebase. Start here before any other surfboard-* skill."
---

# Surfboard Payments

Surfboard Payments is a Swedish payment institution, licensed by Finansinspektionen,
providing in-store, online, and mobile payments through one API across twelve European
markets.

**Read this file fully before writing any integration code.** The conventions below are
where integrations go wrong, and they are wrong in ways that pass local testing.

## Before you can call anything

The API needs credentials that only a human can issue, because getting them means
accepting terms on behalf of a company. That happens in the Developer Portal.

If the user does not have credentials yet, stop, ask them to do this, and wait:

1. Create a developer account at <https://developers.surfboardpayments.com/sign-up>
2. Open the console at <https://developers.surfboardpayments.com/console/api-keys>
3. Copy the **Demo** API key and secret, and the merchant ID

Then have them put the values in `.env` themselves. Do not ask them to paste
credentials into the chat, and do not read them back:

```
SURFBOARD_API_URL=          # from the console; differs between Demo and Live
SURFBOARD_API_KEY=
SURFBOARD_API_SECRET=
SURFBOARD_MERCHANT_ID=
SURFBOARD_TERMINAL_ID=      # every in-store or checkout integration needs one
SURFBOARD_PARTNER_ID=       # partner-scoped endpoints only: stores, onboarding,
                            # billing, logistics, branding, client auth tokens
```

If you write a `.env.example` as a template, **leave every value empty and keep it
that way.** It is tracked. Real values go only in `.env`. Say out loud that you have
created it, because the common accident is the user pasting keys from the console
into whichever file is open, and that file is usually the template.

The base URL is shown in the console alongside the keys and differs between Demo and
Live. It is not published in the documentation, so read it from configuration. Never
hard-code a host you inferred.

See `references/environments.md` for what Demo can and cannot do.

## Which skill to use

Work out the integration shape from the codebase before asking. A POS or till app is
in-store; an e-commerce checkout is online; a subscription product is server-to-server.

| The user wants | Skill |
|---|---|
| Card payments on a physical terminal, or Tap to Pay | `surfboard-in-store` |
| A checkout page, payment links, hosted or self-hosted | `surfboard-online-checkout` |
| Subscriptions, recurring charges, saved cards, B2B invoices | `surfboard-server-to-server` |
| To onboard merchants as a platform or partner | `surfboard-onboarding` |
| To register, configure, or ship terminals | `surfboard-terminals` |
| Event notifications, callbacks, order status pushes | `surfboard-webhooks` |
| Credentials, headers, browser or mobile tokens | `surfboard-auth` |
| To test, simulate, refund, capture, void, or cancel | `surfboard-testing` |
| To move from Demo to Live | `surfboard-go-live` |

Load one. Do not load all of them, because they overlap deliberately so that each stands alone.

**For anything in-store, settle the hardware question before you write code.**
`GET /terminals/:terminalId` and read `terminalType`. A `checkoutX` terminal is
SoftPOS: payments need a native app switch from a point-of-sale app running on the
same device, and no server-only integration can complete one. A browser-based POS
talking to a SoftPOS terminal will create real orders and never take a payment. See
`surfboard-in-store`.

## Conventions that break integrations

### Amounts are integers in the smallest currency unit

10.00 SEK is `1000`. Never send a decimal. A float here survives every local test and
surfaces in production at a hundredth of the intended price.

### Currencies are ISO 4217 *numeric* codes, as strings

SEK is `"752"`. EUR is `"978"`. NOK is `"578"`. DKK is `"208"`. **Not `"SEK"`.**
This is the single most common first-integration error. Countries, by contrast, are
alpha-2 uppercase: `"SE"`, `"NO"`.

```json
"amount": { "regular": 50000, "total": 50000, "currency": "752" }
```

### Check `status`, not the HTTP code

Every response is the same envelope:

```json
{ "status": "SUCCESS", "data": {}, "message": "Order created successfully" }
```

`status` is `SUCCESS` or `ERROR`. Branch on that. Log `message`, but never branch on its
wording, which is not a contract. `data` is absent or null on errors.

**The failure carries a 2xx, and on order creation it is usually 201.** Not 200, not
4xx. A client that special-cases 200 and otherwise trusts the HTTP code will treat a
201 as success and then propagate that 201 into its own error path. Read the envelope
first, and map any 2xx-carrying-an-error onto a server-side failure of your own.

### Orders, payments, and receipts are not merchant-scoped

The merchant travels in the `MERCHANT-ID` **header**, never in the path.

```
POST   /orders                      correct
GET    /orders/:orderId/status      correct
POST   /merchants/:merchantId/orders   404, and the 404 does not say why
```

Only partner-level endpoints put an ID in the path, and they use `partnerId`:
stores, merchant onboarding, billing plans, logistics, branding, client auth tokens.
45 of the 157 endpoints are partner-scoped; none of them are orders, payments, or
receipts.

> Several long-form guides still show `POST /merchants/:merchantId/orders`. They are
> wrong. `api/ai/docs.json` is the authoritative index, and it says `/orders`.

### Prices are tax-inclusive

`totalOrderAmount.total` must **equal** `regular`. The `tax` array describes the VAT
*contained within* that price; it is never added on top.

```json
"totalOrderAmount": { "regular": 11576, "total": 11576, "currency": "752",
                      "tax": [{ "amount": 577, "percentage": 5.25, "type": "VAT" }] }
```

Sending `total = regular + tax` is rejected with `P_0001 ... Invalid total order price`.
If the system you are integrating quotes prices net of tax, as most US point-of-sale
systems do, **gross each unit up before building the order.** That is a real
transformation, not a field rename, and it is the kind of error that passes every
local test.

### `terminal$id` contains a dollar sign

It will break ORMs, query builders, and template languages that treat `$` specially.
Quote it. Identifiers generally are opaque hex strings. Store them as strings, never
parse them, never infer type or length from them.

### Pagination is header-based and fixed at 100

Request a page with the `X-PAGE-NUMBER` header; there is no page-size parameter.
Responses carry `x-page-number` and `x-total-items`. Past the last page you get
`SUCCESS` with an empty array, not an error.

**Terminate the loop on an empty `data` array or on having seen `x-total-items` rows,
never on a short page.** Only the final page is short, and assuming otherwise silently
truncates reports.

### Dates and durations

Timestamps are ISO 8601 (`2026-04-04T10:20:30+02:00`). Durations are `<number><unit>`
with unit `m`, `h`, or `d`: `15m`, `2h`, `3d`.

## The entity model

```
partner  →  merchant  →  store  →  terminal
```

- **Partner**: you, the software company. Partner-level endpoints carry `partnerId` in
  the path and often need no `MERCHANT-ID` header.
- **Merchant**: your customer, the business taking the money. Onboarded through KYB.
- **Store**: a physical or online location belonging to a merchant.
- **Terminal**: a device or a virtual endpoint. An online store is provisioned with
  `PaymentPage` and `MerchantInitiated` terminals **already created**. List them and
  take the IDs. Do not try to register them.

## Authentication headers

```
API-KEY:      YOUR_API_KEY
API-SECRET:   YOUR_API_SECRET
MERCHANT-ID:  YOUR_MERCHANT_ID
```

Where `MERCHANT-ID` is optional and you send it anyway, it must match the `:merchantId`
in the path. Anything running in a browser or on a customer's phone uses a short-lived
client token instead. See `surfboard-auth`. A key in client code is a key in public.

## Rules you do not break

- **Never handle card data.** No PAN, no CVV, no track data, in any code you write.
  Capture happens on a Surfboard hosted page, on the terminal, or in the SDK. This is
  what keeps the user out of PCI scope, and it is the whole reason the API is shaped
  the way it is.
- **Demo by default.** Build and verify against Demo. Moving to Live requires
  certification and a human. See `surfboard-go-live`.
- **Credentials to `.env`, never to source.** Add `.env` to `.gitignore`. Never print a
  secret to a terminal, a log, or the chat.
- **Never mix environments.** Live credentials against a Demo base URL, or the reverse,
  fails at registration or at the first transaction, and the error will not say why.

## Verify before reporting success

An integration is not done because the code compiles. Create an order, initiate a
payment, and fetch its status until it reaches a terminal state. Show the user the
order ID and the final status. If you have not seen a real response from the API, say
so plainly rather than implying it works.

## Finding more

Bundled with this skill: `references/guides/` holds: API conventions, payment lifecycle,
payment methods.

Beyond that, three authoritative developer sources. Use the `www` host on the second and
third, because the apex redirects and not every fetcher follows redirects.

| Question | Source |
|---|---|
| What does this endpoint take, what does this webhook send? | <https://ai.developers.surfboardpayments.com/llms.txt>, which links a `.md` per page |
| How does this flow fit together end to end? | <https://www.surfboardpayments.com/api/ai/guides.json>, 44 guides with full text inline |
| What endpoints exist at all? | <https://www.surfboardpayments.com/api/ai/docs.json>, 157 endpoints grouped by API |

Or search them through the MCP server: `npx -y @surfboardpayments/surf-mcp`
(`search_api_docs`, `search_webhook_docs`, `search_guides`, `read_doc`).

**Do not build against <https://www.surfboardpayments.com/llms.txt>.** That is the marketing
corpus: company facts, product specs, pricing. It is the right source for "what does
Surfboard sell" and the wrong one for "what does this endpoint accept".

## Record what you set up

When the integration works, write or update `AGENTS.md` in the project root with the
merchant ID, store ID, terminal IDs, environment, and the flow chosen. A future session
should not have to rediscover any of it.
