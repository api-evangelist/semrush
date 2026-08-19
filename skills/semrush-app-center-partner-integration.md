---
name: semrush-app-center-partner-integration
description: >-
  Authenticate an App Center partner app to Semrush server-to-server, read a viewer's
  purchase and subscription status, and send partner notification events through the Hermes
  Partner API.
api: Semrush AppCenter API
generated: '2026-08-13'
method: generated
source: openapi/_original/semrush-openapi.yml
base_url: https://api.semrush.com
operations:
  - POST /app-center-api/v2/jwt-token/          # overlay operationId issueAppCenterJwt
  - POST /apis/v4/app-center/v2/partner/viewer-status   # overlay operationId getAppCenterViewerStatus
  - GET /apis/v4/hermes/v0/subscriptions        # overlay operationId getHermesSubscriptions
  - GET /apis/v4/hermes/v0/user/{user_id}/subscription/{id}  # overlay operationId getHermesUserSubscriptionStatus
  - POST /apis/v4/hermes/v0/event               # overlay operationId createHermesEvent
  - GET /apis/v4/hermes/v0/event/{id}           # overlay operationId getHermesEventStatus
grounding: >-
  Every operation is a real path in the Semrush AppCenter OpenAPI published at
  github.com/semrush/app-center-openapi and held in this repo. The upstream spec declares no
  operationIds; the ids referenced above are assigned by overlays/ and are not upstream values.
---

# Integrate an App Center partner app with Semrush

This is the **partner/billing** surface, not the marketing-data surface. It answers
"is this Semrush user entitled to my app?" — it returns no keyword, backlink or traffic data.

## Step 1 — Exchange for an access token

```
POST https://api.semrush.com/app-center-api/v2/jwt-token/
Content-Type: application/json

{ "jwt": "<your signed partner JWT>" }
```

Response: `{ "jwt": "<access token>" }`.

**Handle 403 specially.** This operation returns `403 Access denied` as `text/html`, not
JSON — the one operation in the spec that breaks the meta/error envelope. Check
`Content-Type` before parsing, or your JSON decoder will throw on the error path instead of
reporting it.

`400` and `401` return the JSON error shape with `message`, `message_code` and `code`.

## Step 2 — Authorize every later call

```
Authorization: Bearer <jwt from step 1>
```

The `jwtIssuerToken` scheme in the spec is HTTP bearer with `bearerFormat: JWT`. Cache the
token and re-issue on `401`; Semrush documents no refresh endpoint for this surface.

## Step 3 — Read viewer status

```
POST https://api.semrush.com/apis/v4/app-center/v2/partner/viewer-status
Content-Type: application/json

{ "user_id": 123456 }
```

Despite the `POST` verb this is a **read** — it retrieves purchases and subscriptions and
mutates nothing. It is safe to retry. Do not let generic agent tooling classify it as a
write because of the method.

Response is the standard envelope:
`{ "meta": { "success": true, "status_code": 200, "request_id": "..." }, "data": { ... } }`.

Note that `data` is typed as an untyped `object` in the spec, so you must read the App Center
documentation for the payload shape rather than generate a model from the contract.

## Step 4 — Work the Hermes subscription surface

```
GET https://api.semrush.com/apis/v4/hermes/v0/subscriptions
GET https://api.semrush.com/apis/v4/hermes/v0/user/{user_id}/subscription/{id}
```

Both are reads returning the meta/data envelope.

## Step 5 — Send a notification event

```
POST https://api.semrush.com/apis/v4/hermes/v0/event
Content-Type: application/json

{
  "type": "<event type>",
  "id": "<your event id>",
  "user_id": 123456,
  "data": "<payload string>",
  "attachments": [ { "name": "...", "mime": "...", "content": "..." } ]
}
```

This is the only **write** in the surface, and it is **not idempotent**. Semrush documents no
idempotency key and no de-duplication window for it. Supply your own `id`, record it before
you send, and check status before re-sending rather than blind-retrying — a naive retry on a
timeout can deliver the notification twice.

Poll the result:

```
GET https://api.semrush.com/apis/v4/hermes/v0/event/{id}
```

The event carries `eventAction` objects with `id`, `ts`, `type`, `finished` and a nested
`notification` describing `channel`, `subscription_id` and `recipient_id`. Treat
`finished: true` as the terminal condition.

## Rate limits

10 requests/second and 10 concurrent requests per account, shared with every other Semrush
API you call. No rate-limit headers are returned.

## Related artifacts

- `openapi/semrush-jwt-issuer-api-openapi.yml`, `openapi/semrush-partner-service-api-openapi.yml`,
  `openapi/semrush-hermes-partner-api-api-openapi.yml`
- `overlays/` — operationIds, consequence classes and the 403 content-type exception
- `errors/semrush-problem-types.yml`
- `authentication/semrush-authentication.yml`
