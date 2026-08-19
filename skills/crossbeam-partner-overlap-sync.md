---
name: crossbeam-partner-overlap-sync
description: >-
  Pull Crossbeam partner overlaps and ecosystem signals into your own system over the REST
  Partner API — authenticate with OAuth 2.0, resolve the organization header, page the overlaps
  and signals endpoints with cursors, and resume from a bookmark on the next run. Use when
  someone asks how to sync Crossbeam overlaps into a CRM, warehouse or internal app, how to
  page or bookmark the Crossbeam API, how to fetch partner deal signals, or how to backfill
  webhook signals that were missed.
api: https://developers.crossbeam.com/
contract: postman/crossbeam-partner-api.postman_collection.json
base_url: https://api.crossbeam.com
operations:
  - GET /v1/users/me
  - GET /v1/partners
  - GET /v1/partner-populations
  - GET /v1/overlaps/accounts
  - GET /v1/overlaps/leads
  - GET /v1/signals/accounts
  - GET /v1/signals/leads
  - GET /v1/signals/own-deals
scopes: [openid, read:partnerships, read:reports, read:populations, offline_access]
---

# Sync Crossbeam overlaps and signals into your system

Every path, parameter and scope below comes from Crossbeam's own first-party Partner API
collection (`postman/crossbeam-partner-api.postman_collection.json`, published at
https://developers.crossbeam.com/). Crossbeam publishes no OpenAPI, so there are no
operationIds to cite — the operation identity is the method and path.

> **Gate check first.** The REST Partner API requires a Custom Integration app and is
> positioned for Enterprise. Signals webhooks are Enterprise-only. Say so before recommending
> this path.

## 1. Get a token

Create the app at https://app.crossbeam.com/integrations (Custom section) to obtain a
`client_id` and `client_secret`, then run the standard three-legged authorization-code flow:

- Authorization URL: `https://auth.crossbeam.com/authorize?audience=https://api.getcrossbeam.com`
- Token URL: `https://auth.crossbeam.com/oauth/token`

Request `openid` plus the scopes for the endpoints you will call, **plus `offline_access`** —
access tokens expire after 24 hours and a sync job must refresh, not re-prompt:

```
scope=openid read:partnerships read:reports read:populations offline_access
```

Refresh with `grant_type=refresh_token` against the same token URL.

## 2. Resolve the organization header — do this before anything else

```
GET /v1/users/me
```

Take `organization.uuid` from the `authorizations` array. **Every subsequent call needs two
headers**, and omitting the second is the single most common 4xx on this API:

```
Authorization: Bearer <access_token>
Xbeam-Organization: <organization-uuid>
```

## 3. Page the data with cursors, not offsets

`GET /v1/overlaps/accounts`, `GET /v1/overlaps/leads`, `GET /v1/signals/accounts`,
`GET /v1/signals/leads`, `GET /v1/signals/own-deals`, `GET /v1/records/accounts` and
`GET /v1/audit-logs` all page the same way:

- Request: `limit` (default 25, maximum 1000 — 1000 default on audit-logs) and `cursor`.
- Response: a top-level `items` array plus a `pagination` object containing `next_href` and
  `next_cursor`.
- **Follow `pagination.next_href`.** It already carries the cursor and every other query
  parameter you called with; rebuilding the URL yourself is how filters get silently dropped.
- Stop when `next_href` is absent.

The `/search` variants (`/v1/overlaps/accounts/search`, `/v1/overlaps/leads/search`) are **not
paged** — do not look for a cursor on them.

## 4. Bookmark the cursor and resume incrementally

The overlaps and signals endpoints are bookmarkable: persist the last `next_cursor` from a
completed run and pass it as `cursor` on the next run to collect only what has changed. This is
the difference between a nightly sync that costs one page and one that re-reads the whole
ecosystem — which matters commercially, because **API reads draw down your plan's unique
record-export allowance** (5,000 / 25,000 / 100,000+ by tier).

## 5. Filter server-side

Do not list-then-filter. Use the parameters the endpoints accept:

- Signals: `record_id` and `population_id`, both repeatable —
  `?record_id=abc123&record_id=def456`.
- Overlaps search: `term` (fuzzy prefix), `record_id`, `domain` / `email` (exact), and
  repeatable `partner-population-ids[]`.
- Accounts search: `term`, `record_id`, `domain`, `opportunity_id`, `contact_id`.

## 6. Prefer webhooks for "tell me when"

If the requirement is real-time reaction rather than a periodic pull, use Signals webhooks
(Enterprise-only) instead of polling `/v1/signals/*`. Verify every delivery: HMAC-SHA256 over
body + `X-Crossbeam-Timestamp`, base64-encoded, compared constant-time against
`X-Crossbeam-Signature-256`, and reject timestamps older than 300 seconds. Return 200 or 201
with no body. **Make handlers idempotent** — Crossbeam retries on 408, 429, 500, 502, 503 and
504, and a signal can arrive more than once. After retries are exhausted, recover the missed
message by pulling `/v1/signals/*`. Group signal rows by `event_id` to reconstruct the
aggregated event; `signal_id` is unique per row.

## 7. What you cannot rely on

- **No published rate limits.** Crossbeam says limits live in its security policy and are
  subject to change; no numbers, no `RateLimit-*` headers, no documented 429 contract. Back off
  conservatively on your own schedule.
- **No error catalog and no problem+json.** Plan for bare HTTP status codes with no stable body
  shape.
- **No deprecation policy or Sunset headers.** Nothing will warn you before an endpoint changes.
- **Read-only in practice.** `write:activity-timeline` is the only write scope, and no timeline
  endpoint appears in the published collection.
