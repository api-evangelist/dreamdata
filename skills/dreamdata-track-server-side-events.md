---
name: Send server-side events to Dreamdata
description: Authenticate against the Dreamdata Event Tracking API and POST a batch of Segment-compatible identify/track/page events from a backend.
api: https://api.dreamdata.cloud/v1/batch
grounding: https://developer.dreamdata.io/server-side/server-side-tracking/
operations:
  - POST /v1/batch
generated: '2026-08-13'
method: generated
source: >-
  https://developer.dreamdata.io/server-side/server-side-tracking/ +
  https://developer.dreamdata.io/client-side/api/ +
  conventions/dreamdata-conventions.yml
---

# Send server-side events to Dreamdata

Dreamdata publishes no OpenAPI. This skill is grounded in the one documented HTTP
operation on the ingestion surface — `POST https://api.dreamdata.cloud/v1/batch` — and in the
event shapes the provider prints in its server-side tracking guide. Do not invent other
endpoints; there are none published.

## 1. Get the key

The source API key lives in the Dreamdata app under **Data Platform > Sources > Server Side
Analytics APIs**. It is account-scoped and cannot be minted through an API.

## 2. Authenticate

HTTP Basic, **API key as the username, empty password** — i.e. base64 of `"<apiKey>:"`.

```
curl -u "<apiKey>:" -X POST https://api.dreamdata.cloud/v1/batch \
  -H 'Content-Type: application/json' \
  -d @batch.json
```

## 3. Build the batch

One request carries an envelope plus a `batch` array. Envelope fields: `messageId` (unique per
batch) and `sentAt` (ISO 8601). Every event in the array needs:

- `type` — one of `identify`, `track`, `page`, `group`, `alias`
- `messageId` — a unique UUID for that event
- `userId` **or** `anonymousId` (at least one)
- `timestamp` — ISO 8601, when the event occurred

A batch may mix types. The canonical pattern from the docs: a form fill sends an `identify`
(you now know who they are) **and** a `track`; an anonymous button click sends only a `track`.

```json
{
  "messageId": "6f47f5a0-4118-42de-9e15-95cf9ed2a426",
  "sentAt": "2026-02-06T11:11:14.000Z",
  "batch": [
    {
      "type": "identify",
      "messageId": "6ad2be86-6d78-43bd-8a5b-846ddbc042c2",
      "userId": "019mr8mf4r",
      "traits": { "email": "laura@example.com" },
      "context": { "ip": "24.5.68.47", "library": { "name": "customer", "version": "v1" } },
      "timestamp": "2026-02-06T14:24:14.000Z"
    },
    {
      "type": "track",
      "messageId": "3f8236bc-c010-43bb-a341-cd30b5228e9d",
      "userId": "019mr8mf4r",
      "event": "form_submitted",
      "timestamp": "2026-02-06T11:11:14.000Z"
    }
  ]
}
```

## 4. Identity rules

- `userId` when the person is authenticated or you hold a CRM/database id for them.
- `anonymousId` when they are not — and for a purely server-side integration **you** generate
  and persist it (e.g. a UUID in the session). Dreamdata does not mint one for you server-side.
- `group` carries the company (`groupId`) — this is what makes the model account-based.
- `alias` links an anonymous identity to a known `userId` via `previousId`.

## 5. Limits and failure modes you must respect

- **500kb per request.** Split larger batches. The status code returned on violation is not
  documented — treat any non-2xx as a hard failure and split before retrying.
- **No published rate limits and no rate-limit headers.** There is no `Retry-After` or
  `RateLimit-*` signal to read (`rate-limits/dreamdata-rate-limits.yml`). Back off
  conservatively on your own schedule.
- **No documented error catalog for this endpoint.** There is no problem+json body and no
  status table (`errors/dreamdata-error-codes.yml`). Log the raw response.
- **Idempotency is NOT guaranteed.** Every event carries a unique `messageId` per the Segment
  spec, but Dreamdata does not document deduplication of retried batches and ships no
  `Idempotency-Key` header. Assume a blind retry can double-count, and prefer resending with
  the *same* `messageId` values over regenerating them.

## 6. Prefer the SDK where it fits

`@dreamdata/analytics` (Node.js) and `github.com/dreamdata-io/analytics-go` (Go) handle
batching, retries and flush-on-shutdown. Note the currency finding in
`packages/dreamdata-packages.yml`: the Node package's latest release is 6.0.0 from 2022 and its
upstream repo is archived, so for a new integration weigh calling `/v1/batch` directly.
