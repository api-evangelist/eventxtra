---
name: Authenticate against EventX and read events
description: Exchange an EventX apiToken for a bearer JWT and page through the events an organization hosts or manages.
api: openapi/eventxtra-public-api-openapi.json
base_url: https://esaas-api.eventx.io
operations: [issueJwt, getPaginatedEvent, getPaginatedHostingEvent, getEventById]
generated: '2026-08-13'
method: generated
source: openapi/eventxtra-public-api-openapi.json + https://eventx-hq.gitbook.io/knowledge-base/api-doc/auth
---

# Authenticate against EventX and read events

Every EventX Public API call except the token exchange needs a bearer JWT. This
skill gets one and uses it to read the event list.

## Before you start

- You need an `apiToken` issued from the EventX platform admin panel. The
  pricing page lists **Public API** as an Enterprise-plan capability, so a Free
  or Essentials account cannot obtain one.
- Base URL is `https://esaas-api.eventx.io`. Every path is prefixed
  `/public-api/v1/`.

## Steps

1. **Get a bearer token** — `issueJwt`
   `POST /public-api/v1/auth` with `{"apiToken": "<your api token>"}`.
   The response is `{ "data": { "accessToken", "userId", "expiresIn" } }`.
   `expiresIn` is a duration string; the token is short-lived, so cache it and
   re-issue when it expires rather than calling this on every request.

2. **Send the token on every later call.**
   Header: `Authorization: <accessToken>`. The spec declares this as an apiKey
   scheme named `Authorization` described as "Bearer Authorization with jwt
   token" — follow the docs' own example for whether to prefix `Bearer `.

3. **List events** — `getPaginatedEvent`
   `GET /public-api/v1/event`. Query parameters: `page` (min 1, default 1),
   `pageSize` (min 1, max 100, default 25), `pageEnd` (optional inclusive end
   page, max range 200), `sort` (default `-createdAt`; available sorts
   `createdAt`, `updatedAt`, `startsAt`, `endsAt`, `name`; a leading `-` is
   DESC), and `filter` (object filter supporting `$eq`, `$in`, `$nin`, `$gt`,
   `$gte`, `$lt`, `$lte`, `$exists`, including on `extensionData` keys).
   Response: `{ "dataList": [...], "pagination": { page, pageSize, pageCount,
   totalCount } }`. Page until `page === pageCount`.

4. **Narrow to events you host** — `getPaginatedHostingEvent`
   `GET /public-api/v1/event/hosting-event`, same pagination contract.

5. **Read one event** — `getEventById`
   `GET /public-api/v1/event/{eventId}`. Returns `{ "data": { ... } }` including
   `format` (`virtual` | `in-person` | `hybrid`), `entryType` (`public` |
   `by_invite_only`), `startsAt`/`endsAt`/`timezone`, `defaultLocale` and
   `supportedLocales` (en, ja, ko, vi, zh-HK, zh-CN, es, th, pt), and the
   free-form `extensionData` map.

## Rules

- **Failures are undifferentiated.** The contract declares only `200`, `204` and
  `500`. There is no declared `401`, `403`, `404` or `429`. On any non-2xx, read
  `error.code` and `error.message` from the `{ error: { code, message, status,
  meta } }` body — the HTTP status alone tells you nothing. Do not treat a 500
  as automatically retryable; it may be an expired token or a missing event.
- **There is no idempotency key.** Do not blind-retry writes.
- **No rate-limit headers.** The API publishes no `RateLimit-*` or `Retry-After`
  headers and no documented API throttle. Pace yourself conservatively.
