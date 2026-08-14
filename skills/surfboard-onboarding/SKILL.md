---
name: surfboard-onboarding
description: "Onboard merchants onto Surfboard Payments as a platform or partner: create merchant, KYB application URL, poll application status, create stores, and provision terminals. Covers the merchants and logistics webhooks, account and service provider management, and partner branding. Use when a software platform needs to sign up its own customers to take payments."
---

# Merchant onboarding

Read `surfboard-payments` first. This is the partner flow: you are onboarding your
customers, not yourself.

## The sequence

```
create merchant  →  KYB URL  →  merchant completes KYB  →  poll/webhook
      →  create store  →  provision terminals  →  live
```

Partner-level endpoints carry `partnerId` in the path and usually need no `MERCHANT-ID`
header. Sending one anyway, when it does not match the path, fails in a way that reads
like a permissions problem.

## 1. Create the merchant

`POST` the merchant's business details and get back a merchant ID plus a **KYB URL**.

KYB, Know Your Business, is a regulatory requirement, not a form you can skip or
prefill your way around. Surfboard verifies business details, beneficial owners,
signatories, and PEP status. Neither you nor an agent can approve it.

## 2. Hand the KYB URL to the merchant

Send them the URL. They complete it themselves. This takes minutes to days depending on
the business, so **never build a flow that blocks on it.** Treat merchant onboarding as
an asynchronous state machine in your own product.

## 3. Wait for completion

Two ways to find out:

- **Webhook**: `merchant.application.completed`. Use this. See `surfboard-webhooks`.
- **Polling**: check application status. Fine for a back-office view, wrong as the
  primary mechanism.

Model at least: `created`, `kyb_sent`, `kyb_in_progress`, `approved`, `rejected`. A
rejected application is a real outcome that needs a path in your UI.

## 4. Create a store

A store belongs to a merchant and is where terminals live. Physical stores need an
address; online stores do not.

**An online store arrives with a `PaymentPage` and a `MerchantInitiated` terminal
already provisioned.** Fetch them rather than registering them.

`references/guides/store-management.md`.

## 5. Provision terminals

Physical hardware is ordered, shipped, and then registered against a store with a
registration code. See `surfboard-terminals`, including logistics and returns.

## Branding as you go

Partner, merchant, store, and terminal branding is hierarchical: the most specific
level wins. Set partner-level defaults once and override per merchant only where it
matters. `references/guides/partner-branding.md`.

## Getting this right

- **Idempotency.** Merchant creation retried after a timeout must not create two
  merchants. Key on your own tenant identifier.
- **Store the merchant ID against your tenant** the moment you get it, before the KYB
  round trip. If you lose it you cannot recover the link automatically.
- **Do not collect KYB data yourself.** Surfboard collects it. You are handing over a
  URL, not building a form.
- **Rejection is not an error.** It is an outcome. Give the merchant a route to fix and
  resubmit.

## Verify before reporting success

In Demo, create a merchant, confirm the KYB URL comes back, create a store, and list
its terminals. Show the user the merchant ID, store ID, and terminal IDs.

## Bundled guides

`references/guides/` holds: merchant onboarding, store management, account and service
provider management, partner branding.
