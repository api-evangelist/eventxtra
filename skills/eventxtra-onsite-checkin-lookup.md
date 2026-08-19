---
name: Look up an EventX attendee at the door
description: Resolve a scanned QR code token or a spoken short code to an attendee record, confirm their ticket, and correct their details on the spot.
api: openapi/eventxtra-public-api-openapi.json
base_url: https://esaas-api.eventx.io
operations: [issueJwt, getAttendeeByCheckInQRCodeToken, getAttendeeByShortCode, getAttendeeById, patchAttendeeById, patchAttendeeTicketByAttendeeId, getAllTicketClassByEventId]
generated: '2026-08-13'
method: generated
source: openapi/eventxtra-public-api-openapi.json + https://eventx-hq.gitbook.io/knowledge-base/api-doc/attendee
---

# Look up an EventX attendee at the door

The on-site path: something is scanned or read out, and you need the attendee
behind it in one call.

## Steps

1. **Authenticate** — `issueJwt` (`POST /public-api/v1/auth`). Cache the
   `accessToken` for the door session; re-issue when `expiresIn` elapses.

2. **Resolve a scanned badge/ticket QR code** — `getAttendeeByCheckInQRCodeToken`
   `GET /public-api/v1/event/{eventId}/attendee/qr-code-token/{qrCodeToken}`.

3. **Resolve a spoken or typed short code** — `getAttendeeByShortCode`
   `GET /public-api/v1/event/{eventId}/attendee/short-code/{shortCode}`.
   Use this as the fallback when a QR will not scan.

4. **Read the full record if you only have an id** — `getAttendeeById`
   `GET /public-api/v1/event/{eventId}/attendee/{attendeeId}`. Relevant fields
   for the door: `registration_status`, `attendance_type`, `role_tags`,
   `ticket_class_id` / `ticket_class_name`, `checked_in_at` and
   `check_in_method`.

5. **Fix a detail at the desk** — `patchAttendeeById`
   `PATCH /public-api/v1/event/{eventId}/attendee/{attendeeId}` for name,
   contact or custom-field corrections.

6. **Upgrade or change a ticket** — `patchAttendeeTicketByAttendeeId`
   `PATCH /public-api/v1/event/{eventId}/attendee/{attendeeId}/attendee-ticket`
   to change the ticket class and add-ons. Read the available classes first with
   `getAllTicketClassByEventId`.

## Rules

- **This API does not perform the check-in.** There is no check-in *write*
  operation in the Public API — `attendee-check-in` and `attendee-check-out`
  exist only as webhook actions emitted by EventX's own check-in apps and
  portal. Use these operations for lookup and correction; perform the actual
  check-in through the EventX Check-in App, then observe it arriving on your
  webhook (see the *Subscribe to EventX attendee webhooks* skill).
- **`checked_in_at` is your source of truth for "already in".** A non-null value
  means this badge has been used; challenge a second scan rather than waving it
  through.
- **A 500 may mean "not found".** The contract declares no 404, so an unknown
  QR token surfaces the same way as a server fault. Read `error.code` /
  `error.message` from the `{ error: {...} }` body before telling someone they
  are not registered.
- **Do not surface `magic_link_url`** on a door screen or printout — it
  authenticates the attendee into the event web app.
- **No rate-limit headers exist.** A scanner loop that retries hard has no
  signal to back off on; add your own client-side pacing.
