---
name: surfboard-webhooks
description: Receive and verify Surfboard Payments webhooks: notification subscriptions, the event catalog, signature verification, idempotency and replay handling, retries, and local tunnelling. Use when payment status must reach the backend without polling, or when debugging events that are missing, duplicated, or arriving out of order.
---

# Webhooks

Read `surfboard-payments` first.

Webhooks are how payment status reaches you in production. Polling is fine while
building; it is not fine at scale, and it cannot tell you about a customer who closed
the tab after paying.

## Subscribe

Notification subscriptions are created through the API and configured in the Console
under the developer account settings. Register the events you want and the URL to
receive them at.

`references/guides/notification-subscriptions.md`.

## Events worth handling

| Event | Meaning |
|---|---|
| `order.payment.completed` | Paid. Fulfil here, not on the redirect. |
| `order.payment.cancelled` | Customer backed out. |
| `order.payment.failed` | Declined. Offer a retry. |
| `merchant.application.completed` | KYB finished. See `surfboard-onboarding`. |

The full catalog is in `references/guides/webhooks-notifications.md`, and searchable
through the MCP server's `search_webhook_docs`.

## Four rules

**1. Verify the signature before you trust the body.** An unverified webhook endpoint
is an unauthenticated endpoint that changes payment state. Verify first, parse second.

**2. Be idempotent.** Delivery is at-least-once. The same event will arrive twice, and
the second copy must be a no-op. Key on the event ID and record what you have already
processed. Do not infer it from your own state.

**3. Return 2xx fast, then work.** Acknowledge, queue, process. A slow handler looks
like a failed handler and triggers a retry, which is how one payment becomes three
fulfilment emails.

**4. Do not assume ordering.** Events can arrive out of order. Reconcile against the
payment's actual status rather than assuming the sequence you expected.

## Handler skeleton

```ts
// Express. The shape matters more than the framework.
import express from 'express'

const app = express()

// Raw body: signature verification runs over the exact bytes sent,
// so this must come before any JSON parsing middleware.
app.post('/webhooks/surfboard', express.raw({ type: '*/*' }), async (req, res) => {
  if (!verifySignature(req.headers, req.body)) return res.sendStatus(401)

  const event = JSON.parse(req.body.toString())

  if (await alreadyProcessed(event.id)) return res.sendStatus(200) // at-least-once
  await enqueue(event)                                            // work happens elsewhere

  res.sendStatus(200)
})
```

```python
# FastAPI
from fastapi import FastAPI, Request, Response

app = FastAPI()

@app.post("/webhooks/surfboard")
async def surfboard(request: Request):
    raw = await request.body()          # verify over raw bytes, before parsing
    if not verify_signature(request.headers, raw):
        return Response(status_code=401)

    event = json.loads(raw)
    if await already_processed(event["id"]):
        return Response(status_code=200)

    await enqueue(event)
    return Response(status_code=200)
```

## Developing locally

Surfboard cannot reach `localhost`. Tunnel it with ngrok, Cloudflare Tunnel, or your
platform's preview URL, then point the subscription at the public URL. Remember to
change it back before shipping.

## When events do not arrive

1. Is the subscription pointing at the right URL, in the right environment?
2. Is the endpoint publicly reachable and returning 2xx? Retries stop eventually.
3. Are you verifying against the raw body, or a re-serialised one? Re-serialising
   changes the bytes and every signature fails.
4. Is a body-parsing middleware consuming the stream before your handler sees it?

## Bundled guides

`references/guides/` holds: webhooks, notification subscriptions.
