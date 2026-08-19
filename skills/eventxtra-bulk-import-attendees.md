---
name: Bulk import and maintain EventX attendees
description: Define an event's custom fields, bulk upsert attendees into it, then read them back — and know the difference between removing and deleting.
api: openapi/eventxtra-public-api-openapi.json
base_url: https://esaas-api.eventx.io
operations: [issueJwt, getAllCustomFieldByEventId, bulkUpsertCustomFieldByEventId, bulkUpsertAttendeeByEventId, getPaginatedAttendeeByEventId, getAttendeeById, patchAttendeeById, getAllAttendeeTagByEventId, bulkRemoveAttendeeByEventId, bulkDeleteAttendeeByEventId]
generated: '2026-08-13'
method: generated
source: openapi/eventxtra-public-api-openapi.json + https://eventx-hq.gitbook.io/knowledge-base/api-doc/attendee
---

# Bulk import and maintain EventX attendees

Load an attendee list into an EventX event, keep it in sync, and read it back.

## Steps

1. **Authenticate** — `issueJwt` (`POST /public-api/v1/auth`). See the
   *Authenticate against EventX and read events* skill.

2. **Inspect the event's question library** — `getAllCustomFieldByEventId`
   `GET /public-api/v1/event/{eventId}/custom-field/all`. Custom field answers
   on an attendee are keyed against these definitions, so read them before you
   import anything that carries custom data.

3. **Create or update custom fields if needed** — `bulkUpsertCustomFieldByEventId`
   `PUT /public-api/v1/event/{eventId}/custom-field/bulk-upsert`.
   Fields can carry conditional logic (`targetQuestionId`) and choice options
   (`questionOptionId`).

4. **Upsert the attendees** — `bulkUpsertAttendeeByEventId`
   `PUT /public-api/v1/event/{eventId}/attendee/bulk-upsert`.
   This is the import path — it creates new attendees and updates matching ones
   in a single call.

5. **Read the list back** — `getPaginatedAttendeeByEventId`
   `GET /public-api/v1/event/{eventId}/attendee`. Beyond the standard `page` /
   `pageSize` / `sort` / `filter`, this operation adds `q` (free text),
   `customFieldSearch`, `customFields`, `attendeeIds`, `isIgnoreCase` and
   `createdAt$gt` — use `createdAt$gt` to poll for newly created attendees
   instead of re-paging the whole list.

6. **Read or patch one attendee** — `getAttendeeById` /
   `patchAttendeeById` (`GET`/`PATCH
   /public-api/v1/event/{eventId}/attendee/{attendeeId}`). To move someone
   between ticket classes or change their add-ons use
   `patchAttendeeTicketByAttendeeId`
   (`PATCH .../attendee/{attendeeId}/attendee-ticket`).

7. **List the tags in play** — `getAllAttendeeTagByEventId`
   `GET /public-api/v1/event/{eventId}/attendee-tag/all`.

## Rules

- **`bulk-remove` and `bulk-delete` are not the same operation.**
  `bulkRemoveAttendeeByEventId` (`DELETE .../attendee/bulk-remove`) removes
  attendees *from the event*. `bulkDeleteAttendeeByEventId` (`DELETE
  .../attendee/bulk-delete`) **permanently deletes** them. Never map a generic
  "delete" intent onto the second one without explicit confirmation — it is
  irreversible and the API offers no undo.
- **No idempotency key exists.** A timed-out bulk upsert cannot be safely
  replayed by the protocol. Re-read with `getPaginatedAttendeeByEventId` before
  retrying.
- **Every failure is a 500.** Read `error.code` / `error.message` from the body.
- **Attendee data is personal data.** EventX is GDPR-compliant and ISO 27001
  certified; treat exported attendee records (`email`, `contact_no`,
  `job_title`, `organization`) accordingly and do not hold them longer than the
  task requires.
