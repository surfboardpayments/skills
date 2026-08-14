# Surfboard Payments agent skills

Ten agent skills for integrating [Surfboard Payments](https://surfboardpayments.com)
into a codebase. In-store card terminals, online checkout, recurring billing, and
merchant onboarding, through one API across twelve European markets.

Surfboard Payments is a Swedish payment institution licensed by Finansinspektionen.

## Install

```bash
npx -y skills add surfboardpayments/skills --skill '*' --yes
```

Works with any agent that reads the [Agent Skills](https://github.com/vercel-labs/skills)
format: Claude Code, Cursor, Codex, Gemini CLI, Windsurf, Copilot.

Or give your agent the whole setup in one line:

```
fetch https://surfpay.ai to add payments
```

That installs these skills, connects the documentation, and walks the integration
through to a verified test transaction.

## The skills

Start with `surfboard-payments`. It is the router, and it carries the conventions
every other skill assumes.

| Skill | For |
|---|---|
| `surfboard-payments` | Router and shared conventions. Read this first. |
| `surfboard-auth` | Credentials, headers, client auth tokens for browser and mobile |
| `surfboard-in-store` | Card, Swish, and Klarna on a physical terminal |
| `surfboard-online-checkout` | Hosted Payment Page, self-hosted checkout, payment links |
| `surfboard-server-to-server` | Recurring billing, subscriptions, Merchant Initiated Transactions |
| `surfboard-webhooks` | Subscriptions, signature verification, idempotency, replay |
| `surfboard-onboarding` | Onboarding merchants as a platform partner, KYB, stores |
| `surfboard-terminals` | Device registration, configuration, logistics, SoftPOS SDKs |
| `surfboard-testing` | Demo environment, test cards, refunds, captures, voids, error codes |
| `surfboard-go-live` | Certification, PCI scope, markets, the production checklist |

Load one worker skill, not all of them. They overlap deliberately so each stands alone.

## What is in a skill

```
surfboard-in-store/
├── SKILL.md                    hand-written
└── references/guides/          generated
    ├── create-an-order.md
    ├── create-order-error-codes.md
    └── ...
```

**`SKILL.md` is hand-written.** It is the judgement: what to do in what order, what
goes wrong, and what "done" looks like. Each one carries a complete, runnable example.

**`references/guides/` is generated** from
[`api/ai/guides.json`](https://www.surfboardpayments.com/api/ai/guides.json), which
publishes all 44 Surfboard integration guides with their full content. A skill bundles
the guides it cites, so it cannot drift from the documentation. Do not edit those files
by hand; they are overwritten on every refresh.

## Getting credentials

Demo credentials are issued by the Developer Portal as soon as you sign up. No sales
call and no approval step.

1. Create an account at <https://developers.surfboardpayments.com/sign-up>
2. Open <https://developers.surfboardpayments.com/console/api-keys>
3. Copy the Demo API key, API secret, merchant ID, and base URL into `.env`

```
SURFBOARD_API_URL=
SURFBOARD_API_KEY=
SURFBOARD_API_SECRET=
SURFBOARD_MERCHANT_ID=
```

Your agent should never ask you to paste these into a chat, and the skills tell it not
to. Live credentials are separate, and are issued to a person after certification.

## What your agent will not do

The skills set hard boundaries, because this is payments:

- **No cardholder data.** No code touching PAN, CVV, or track data. Card capture happens
  on a Surfboard hosted page, on the terminal, or in the SDK. That boundary is what keeps
  your systems out of PCI scope.
- **Demo by default.** Every call targets the Demo environment. A real card used there is
  voided automatically after 30 minutes; it is never captured and never settles.
- **No Live access.** Going live needs certification and a person.
- **No secrets in source.** Credentials go to `.env`, never to a log or the chat.

## Documentation

The skills are grounded in the developer corpora, not the marketing site.

| Question | Source |
|---|---|
| What does this endpoint take, what does this webhook send? | <https://ai.developers.surfboardpayments.com/llms.txt> |
| How does this flow fit together end to end? | <https://www.surfboardpayments.com/api/ai/guides.json> |
| What endpoints exist at all? | <https://www.surfboardpayments.com/api/ai/docs.json> |

There is also an MCP server for searching the same material live:

```json
{
  "mcpServers": {
    "surfboard": { "command": "npx", "args": ["-y", "@surfboardpayments/surf-mcp"] }
  }
}
```

Tools: `search_api_docs`, `search_webhook_docs`, `search_guides`, `read_doc`.

## Conventions that break first integrations

The router skill covers these in full. The short version, because they are worth knowing
before you read anything else:

- **Amounts are integers in the smallest currency unit.** 10.00 SEK is `1000`. A decimal
  here passes local testing and surfaces in production at a hundredth of the price.
- **Currencies are ISO 4217 numeric codes, as strings.** SEK is `"752"`, not `"SEK"`.
- **Check `status` in the response envelope, not the HTTP code.** A 200 can carry
  `"status": "ERROR"`.
- **`terminal$id` contains a dollar sign.** Quote it. It breaks ORMs and template engines.
- **Pagination is fixed at 100 per page** via the `X-PAGE-NUMBER` header. Terminate on an
  empty `data` array, never on a short page.

## Refreshing

The generated references come from the live guides corpus. To rebuild them, run the
generator in the [surfpay.ai](https://surfpay.ai) project and sync the result here.

## Issues

Corrections to `SKILL.md` are welcome as pull requests. For anything wrong in a bundled
guide, fix the guide on the developer site instead; the reference here is a copy and will
be overwritten.

Integration support: <https://developers.surfboardpayments.com>
