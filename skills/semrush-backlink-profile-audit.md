---
name: semrush-backlink-profile-audit
description: >-
  Audit a domain's backlink profile with the Semrush Backlinks API v4 — pull aggregate
  metrics, walk the individual links, and identify referring domains and toxic-signal
  patterns, while staying inside Semrush's rate limits and API unit budget.
api: Semrush Backlinks API v4
generated: '2026-08-13'
method: generated
source: https://developer.semrush.com/api/v4/seo/backlinks/
base_url: https://api.semrush.com
operations:
  - GET /apis/v4/backlinks/v1/overview
  - GET /apis/v4/backlinks/v1/summary
  - GET /apis/v4/backlinks/v1/links
  - GET /apis/v4/backlinks/v1/ref-domains
  - GET /apis/v4/backlinks/v1/anchors
  - GET /apis/v4/backlinks/v1/score-profile
  - GET /apis/v4/backlinks/v1/competitors
grounding: >-
  Endpoints, parameters and unit costs are read verbatim from the Semrush v4 Backlinks API
  reference. Semrush publishes no OpenAPI for this API, so there are no operationIds to
  bind to — bind to the method and path.
---

# Audit a backlink profile with Semrush

## Before you start

- You need a **v4 API key**. A v3 key will not authorize a v4 endpoint.
- Every request costs **API units** against your account balance. Check it first — the
  balance call itself is free:
  `GET http://www.semrush.com/users/countapiunits.html?key=<key>`
- Semrush caps **10 requests/second and 10 concurrent requests per account**, across all of
  its APIs. There are **no rate-limit response headers** — you will not be warned before a
  429, so throttle on your own clock.
- Semrush's Terms of Service (§3.3) forbid caching API-derived data for more than one month
  without written consent. Set a TTL accordingly.

## Authorization

Pass the key in the header (recommended):

```
Authorization: Apikey <YOUR_V4_API_KEY>
```

Never put it in a query string for v4 calls — that path exists only for the legacy v3
Standard API and leaks the credential into logs and referrers. The key also grants access to
your unit balance, so a leak is a spend risk.

## Step 1 — Get the aggregate picture

```
GET https://api.semrush.com/apis/v4/backlinks/v1/overview?url=example.com&scope=ROOT_DOMAIN
```

Required: `url` (max 2000 chars, protocol optional) and `scope`
(`ROOT_DOMAIN` | `SUBDOMAIN` | `SUBFOLDER` | `PAGE`).

Cost: **45 API units per request.**

Use `fields` to return only what you need — for example
`fields=backlinks_count,domains_count,score`. Use `format=csv` if you are loading a
warehouse; `json` is the default.

Read `backlinks_count`, `domains_count`, `follows_count`, `ips_count` and
`ip_class_c_count` from the response. A high `backlinks_count` against a low
`domains_count` is the first signal of a narrow, possibly manipulated profile.

## Step 2 — Establish the trend

```
GET https://api.semrush.com/apis/v4/backlinks/v1/summary?url=example.com&scope=ROOT_DOMAIN&fields=score,backlinks_count,domains_count&limit=12&date_from=2024-01&date_to=2024-12
```

`limit` defaults to 12 and caps the number of history points returned; if the target has
less history than you ask for, only the available points come back — that is not an error.

A sudden step change in `domains_count` between two adjacent months is where to look first.

## Step 3 — Walk the individual links

```
GET https://api.semrush.com/apis/v4/backlinks/v1/links?url=example.com&scope=ROOT_DOMAIN&limit=100&offset=0
```

Paging is **limit/offset**. There is no cursor, no next-page token, and no total count —
increment `offset` by `limit` and stop when a short page comes back.

Use the `filter` parameter for anything beyond equality. The syntax is `Field Operator Value`:

```
?filter=page_score > 50 AND (source_url LIKE 'https://blog.%' OR is_sponsored IN (false))
```

Operators: `>` `>=` `<` `<=`, `LIKE` `CONTAINS` `STARTS_WITH` `ENDS_WITH` `WORD_MATCH`,
`IN` `NOT_IN`, `AND` `OR`, `HAS` `HAS_ANY` `HAS_ALL`. URL-encode `#`, `&`, `*`, `+`, `-`,
`:`, `|`, `%` and `/` per the Semrush encoding table.

Per-link fields worth pulling: `domain_score`, `page_score`, `anchor`, `first_seen_at`,
`is_nofollow`, `is_sponsored`, `is_ugc`, `is_lost`, `is_new`, `is_sitewide_main`,
`external_links_count`, `ip_address`.

## Step 4 — Profile the sources

- `GET /apis/v4/backlinks/v1/ref-domains` — referring domains with `domain_score`,
  `is_follow`, `is_lost`, `is_new`, `country`.
- `GET /apis/v4/backlinks/v1/anchors` — anchor-text distribution with `backlinks_count`
  and `domains_count`. A single exact-match anchor dominating the distribution is the
  classic over-optimization signal.
- `GET /apis/v4/backlinks/v1/score-profile` — the distribution of source authority.
- `GET /apis/v4/backlinks/v1/competitors` — domains sharing referring domains with the
  target, via `common_refdomains`.

## Step 5 — Handle failure correctly

Every response carries the same envelope:

```json
{ "meta": { "success": false, "status_code": 429, "request_id": "..." },
  "error": { "code": 429, "message": "...", "retryable": true, "details": {} } }
```

- **Branch on `error.retryable`.** It is the only retry guidance Semrush gives.
- Retry on `409`, `429`, `499`, `500`, `503`, `504` with exponential backoff. There is no
  `Retry-After` header — pick your own interval.
- Do **not** retry `400`, `401`, `403`, `501`.
- On `401`, first check you are using a v4 key against a v4 endpoint. That mismatch is the
  most common cause since keys became version-scoped on 2026-07-15.
- Always log `meta.request_id`; it is what Semrush Support will ask for.

## Cost discipline

- Set `fields` on every call. You pay for what comes back.
- Historical data costs several times what live data costs — on the Standard API, 50 units
  per line versus 10.
- Estimate before you loop: rows × unit cost × targets. A thousand links across a hundred
  domains runs into millions of units.

## Related artifacts

- `conventions/semrush-conventions.yml` — pagination, filtering, tracing, metering
- `errors/semrush-problem-types.yml` — full status-code catalog
- `rate-limits/semrush-rate-limits.yml` — every published limit
- `authentication/semrush-authentication.yml` — key model and OAuth flows
