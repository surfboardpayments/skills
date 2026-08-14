# Demo and Live

Surfboard has two environments. They are called **Demo** and **Live**, not sandbox and
production, though people use those words in conversation.

## Demo

Demo credentials are issued by the Developer Portal the moment an account is created.
No sales call, no ticket, no approval step.

| | Demo |
|---|---|
| Terminals | Payment page mode only |
| Cards | Test cards |
| Real cards | Voided automatically after 30 minutes. Never captured, never settled. |
| Credentials | <https://developers.surfboardpayments.com/console/api-keys> |

Demo is enough to build and verify an entire integration: create orders, initiate
payments, poll status, receive webhooks, issue refunds. What it will not do is drive
physical terminal hardware beyond payment page mode.

## Live

Live credentials are issued separately, after Surfboard certifies the integration. The
sequence is:

1. Sign the contract and receive approval
2. Complete an onboarding call to test and certify the integration
3. Receive production credentials
4. Update the base URL from Demo to Live
5. Start accepting live payments

An agent cannot obtain Live credentials, and must not try. See `surfboard-go-live`.

## The base URL changes between them

Both hosts come from the console, alongside the keys. They are not published in the
public documentation, so there is no host to infer or copy from an example. Read it
from configuration.

```
SURFBOARD_API_URL=      # from the console, differs per environment
SURFBOARD_API_KEY=
SURFBOARD_API_SECRET=
SURFBOARD_MERCHANT_ID=
```

## Never mix them

A test merchant against a production backend, or Live credentials against a Demo base
URL, fails at registration or at the first transaction, and the error message will not
tell you that mixing environments is the cause. If a previously working integration
starts failing at auth or at terminal registration, check this first.
