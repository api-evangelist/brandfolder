---
name: Subscribe to Brandfolder asset events
description: Register a webhook for asset create/update/delete events on a Brandfolder, test delivery first, and manage the subscription lifecycle.
api: openapi/brandfolder-openapi-original.yml
operations:
  - opIdApiV4WebhooksSendPost
  - opIdApiV4WebhooksPost
  - opIdApiV4WebhooksGet
  - opIdApiV4WebhooksByIdGet
  - opIdApiV4WebhooksByIdDelete
generated: '2026-08-13'
method: generated
source: openapi/brandfolder-openapi-original.yml (webhooks tag)
---

# Subscribe to Brandfolder asset events

## The version trap — read this first

**The webhook operations do not run on v4.** They are served from
`https://brandfolder.com/api/v1`, and the OpenAPI carries path-level `servers[]`
overrides saying so on all three webhook paths. If you build the URL from the v4
base like every other call in this API, every webhook request fails.

```
https://brandfolder.com/api/v1/webhooks
```

`Content-Type: application/json` is a **required** header on these operations —
the spec declares it as an explicit header parameter with a single-value enum,
not merely as the body media type. Send it.

## 1. Prove your receiver works before you subscribe

`opIdApiV4WebhooksSendPost` → `POST /webhooks/send`

The docs are explicit that you should do this first. It queues a fake
notification to your `callback_url` and answers `202` with
`{"message": "Added demo webhook to queue", "status": "ok"}`.

A `202` means Brandfolder accepted the request, **not** that your endpoint
received anything. If nothing arrives within a few minutes, the `callback_url`
is the problem. It must:

- begin with `https://`
- resolve on a public domain — not `localhost`, not behind a firewall, not
  behind authentication
- accept `POST` and answer `2xx`

Optional `asset_key` and `organization_key` attributes let you shape the test
payload.

## 2. Create the subscription

`opIdApiV4WebhooksPost` → `POST /webhooks`

Required attributes, all four:

| Attribute | Value |
|---|---|
| `event_type` | `asset.create`, `asset.update` or `asset.delete` |
| `resource_type` | `brandfolder` — the only supported value |
| `resource_key` | the Brandfolder ID to watch |
| `callback_url` | your HTTPS receiver |

There is **one event type per subscription**. Watching all three lifecycle
events on one Brandfolder means three separate subscriptions.

You cannot subscribe at collection, section or organization scope. Watching a
whole organization means enumerating its Brandfolders
(`list-brandfolders` → `GET /brandfolders`) and subscribing to each.

## 3. Manage them

- `opIdApiV4WebhooksGet` → `GET /webhooks` — list active subscriptions.
- `opIdApiV4WebhooksByIdGet` → `GET /webhooks/{webhook_id}` — fetch one.
- `opIdApiV4WebhooksByIdDelete` → `DELETE /webhooks/{webhook_id}` — stop delivery.

## Receiver hardening — the gap you must cover yourself

**Brandfolder publishes no webhook signature, no shared secret and no verification
header.** Nothing in the payload lets you prove a delivery came from Brandfolder.
Anyone who learns your `callback_url` can post to it.

Compensate on your side:

- Use an unguessable, high-entropy path in the `callback_url`.
- Treat the payload as a *hint*, not as truth: on receipt, call back into the v4
  API (`opIdApiV4AssetsByIdGet` → `GET /assets/{asset_id}`) and act on what the
  API returns.
- Expect redelivery and out-of-order arrival — no retry policy or ordering
  guarantee is published — so make your handler idempotent on `asset_id` +
  event.

## Errors on the webhook endpoints

- `400` — invalid payload, or the required `Content-Type` header is missing.
- `403` — permission denied on the referenced Brandfolder.
- `404` — the webhook does not exist, or the requester does not own it.
- `409` — **Subscription Already Exists.** This is the closest thing this API has
  to idempotency: a duplicate create is rejected rather than duplicated. Treat
  409 on `POST /webhooks` as success, not as an error to retry.
