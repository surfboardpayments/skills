---
title: "Getting Started"
source: https://www.surfboardpayments.com/developers/guides/getting-started
category: in-store
tags: [Getting Started, Developer Portal, API Keys, Sandbox, Onboarding]
generated: true
---

# Getting Started

> The first hour of every Surfboard integration: create the developer account, invite your team, take the API keys, learn the demo and live environments, and pick the path of guides for what you are building.

## Overview

Everything around the integration itself starts in the [integration portal](https://integrate.surfboardpayments.com/): your partner account, your team, the Demo API keys and webhooks, and a step-by-step path for what you are building. This guide gets you through that once, so every other guide can assume you have it.

It is the same list we send a new partner by email before the first call. If you would rather go through it together, [book a startup call](/contact/) and we will set it up with you.

## Step 1: Create the Account and Invite Your Team

Sign up at the [integration portal](https://integrate.surfboardpayments.com/signup). There is no approval queue: the account is live immediately, with a demo environment attached. Say what you are building and the portal lays out the steps for it, each one ready to hand to a coding agent.

Then invite the developers who will work on the integration: the **+** next to **Team** in the portal's account panel.

> **Use one email domain.** Team members are mapped to your partner ID by their email domain, so everyone should sign up with the same one, for example `@yourcompany.com`. If you use external contractors with other domains, tell us before they sign up and we will make sure they land in the same account.

The account is a partner account. Your Demo `partnerId` is shown on the portal's **Demo API key** step and is what you will pass when onboarding merchants and creating stores later.

## Step 2: Take the Keys and Try the API

Create an API key and secret on the portal's first step, **Demo API key**. The secret is shown once, together with the base URL to call. Every request carries them as headers, with the merchant you are acting for as a third:

```
API-KEY:      YOUR_API_KEY
API-SECRET:   YOUR_API_SECRET
MERCHANT-ID:  YOUR_MERCHANT_ID
```

The same API key and secret also log you in to the [Partner Portal](https://partner.surfboardpayments.com/). The login screen asks for an email and a password: enter the API key as the email and the API secret as the password. The portal is a GUI built on the same APIs, so anything you do there can also be done with an API call. It is useful while you build, for checking what a call actually created: merchants, stores, transactions and payment methods.

Keys expire. When a call that worked yesterday returns `401 Unauthorized` today with nothing else changed, generate a new pair before you debug anything else.

Then try the API before you write code:

- **[Playground](https://developers.surfboardpayments.com/playground/)**: run real requests against the demo environment from the browser.
- **[API reference](https://developers.surfboardpayments.com/references/api/orders/create-order)**: every endpoint and payload. Create Order is the one to read first; the platform is orders-first, and the payment is initiated in the same call.
- **[Webhook reference](https://developers.surfboardpayments.com/references/webhooks/merchants/application-completed)**: every event you can subscribe to.
- **Postman collection**: [download it](/files/Surfboard_Payments_API.postman_collection.json) and drop your keys into the environment.
- **MCP server**: `npx -y @surfboardpayments/surf-mcp` gives your coding agent the whole reference as tools. Setup is on the [MCP page](/developers/mcp/).

Read [API Conventions](/developers/guides/api-conventions/) once. Amounts are integers in the smallest currency unit, currencies are numeric ISO 4217 codes as strings, prices are tax-inclusive, and order endpoints are not merchant-scoped in the path. Each of those catches a first integration at least once.

## Step 3: Know the Environments

You start in the demo environment. Live credentials, and a different base URL, come after certification.

| Environment | Terminals | Cards |
|-------------|-----------|-------|
| **Demo** | All hardware terminals, the Terminal Tester app, Mobile Checkout | Live cards can be used. Transactions are voided immediately after payment. |
| **Live** | All hardware terminals, Mobile Checkout | Live cards. Transactions are settled and paid out. |

For in-store testing, the Terminal Tester app on Android simulates payments with built-in success and failure cards.

The base URL is issued to your account rather than published, and it changes between demo and live. It comes with the key when you create one in the portal. Keep host, key and secret in configuration:

```
SURFBOARD_API_URL=
SURFBOARD_API_KEY=
SURFBOARD_API_SECRET=
SURFBOARD_MERCHANT_ID=
```

Never mix environments in one flow. A demo merchant against the live host, or live keys against the demo host, fails at the first call and the error will not say why.

## Step 4: Pick Your Path

The guides are written one topic at a time. The paths put them in reading order for a kind of build, and each path can be copied as one markdown file for your coding agent:

- [Food & Beverage](/developers/guides/paths/food-and-beverage/): restaurant and café POS with tips, split bills and online orders.
- [Retail](/developers/guides/paths/retail/): store POS with a product catalog, a Swedish cash register, returns and gift cards.
- [Hospitality](/developers/guides/paths/hospitality/): deposits online, pre-authorisation at check-in, stored cards and invoices.
- [SoftPOS only](/developers/guides/paths/softpos/): Tap to Pay on iPhone and Android with no hardware.
- [Online store](/developers/guides/paths/online-store/): e-commerce checkout on your own page or a hosted payment page, with Apple Pay, Swish and Klarna.

If none of them is your build, the assistant on the [guides page](/developers/guides/) answers questions across all of them, and the [Developer Guides](/developers/guides/) index lists every guide by topic.

## Step 5: Certification and Go Live

When the integration works end to end in demo:

1. Sign the contract and receive approval.
2. Complete an onboarding call where we test and certify the integration together. Some products carry their own certification on top, such as the cash register self-certification in [Build Your Own ECR](/developers/guides/electronic-cash-register/).
3. Receive production credentials.
4. Switch the base URL from demo to production.
5. Start accepting live payments.

Branding is worth doing before the first merchant sees the product. [Partner Branding](/developers/guides/partner-branding/) sets your colours, fonts and logo on the terminal screens, receipts and payment pages.

## Reference

- [Integration portal](https://integrate.surfboardpayments.com/)
- [Sign up](https://integrate.surfboardpayments.com/signup)
- [Playground](https://developers.surfboardpayments.com/playground/)
- [API Conventions](/developers/guides/api-conventions/)
- [Book a startup call](/contact/)
