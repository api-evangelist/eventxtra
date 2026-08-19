---
name: Subscribe to EventX attendee webhooks
description: Register an HTTPS endpoint against an EventX event to receive attendee create/update/delete/check-in/check-out deliveries, and manage the subscription.
api: openapi/eventxtra-public-api-openapi.json
base_url: https://esaas-api.eventx.io
operations: [issueJwt, createEventWebhook, getAllEventWebhook, patchEventWebhookById, deleteEventWebhookById]
generated: '2026-08-13'
method: generated
source: openapi/eventxtra-public-api-openapi.json + https://eventx-hq.gitbook.io/knowledge-base/api-doc/event-webhook
---

# Subscribe to EventX attendee webhooks

EventX pushes attendee lifecycle events to a URL you register per event. This is
the mechanism behind its Zapier and CRM (Salesforce / HubSpot / Marketo) syncs.

## Steps

1. **Authenticate** — `issueJwt` (`POST /public-api/v1/auth`).

2. **Create the subscription** — `createEventWebhook`
   `POST /public-api/v1/event/{eventId}/webhook` with your endpoint URL and the
   actions you want. The action enum is exactly:
   `attendee-create`, `attendee-update`, `attendee-delete`,
   `attendee-check-in`, `attendee-check-out`. There are no other actions.

3. **Verify it registered** — `getAllEventWebhook`
   `GET /public-api/v1/event/{eventId}/webhook/all`.

4. **Adjust it** — `patchEventWebhookById`
   `PATCH /public-api/v1/event/{eventId}/webhook/{webhookId}`.

5. **Remove it** — `deleteEventWebhookById`
   `DELETE /public-api/v1/event/{eventId}/webhook/{webhookId}`. This is a **hard
   delete** — the subscription is permanently removed, not disabled.

## Handling a delivery

EventX sends an HTTP `POST` with a JSON body shaped:

```
{
  "config":   { "event_id", "webhook_id", "action", "endpoint_url" },
  "attendee": { "id", "first_name", "last_name", "firstName", "lastName",
                "name", "email", "registration_status",
                "virtual_attendance_status", "first_entered_web_app_at",
                "magic_link_url", "attendance_type", "role_tags",
                "ticket_class_id", "ticket_class_name", "job_title",
                "organization", "city", "country", "area_code", "contact_no",
                "source", "created_at", "update_at", "attended_at",
                "registered_at", "checked_in_at", "custom_fields",
                "check_in_method" }
}
```

**Respond with any 2xx status to acknowledge receipt.**

## Rules

- **Deliveries are unsigned.** EventX documents no signature header, shared
  secret or timestamp, so you cannot cryptographically prove a POST came from
  EventX. Use an unguessable endpoint path, restrict by source where you can,
  and treat the payload as untrusted input — re-read the attendee through
  `getAttendeeById` before acting on anything consequential.
- **Retry behaviour is undocumented.** Assume at-least-once delivery is not
  guaranteed and that duplicates are possible. Deduplicate on
  `config.webhook_id` + `attendee.id` + `config.action` + `attendee.update_at`,
  and reconcile periodically with `getPaginatedAttendeeByEventId` using
  the `createdAt$gt` filter rather than relying on webhooks alone.
- **Field-name quirks are real, not typos in this doc.** The payload carries
  both `first_name` and `firstName`, and the updated timestamp is `update_at`
  (not `updated_at`). Parse defensively.
- **`magic_link_url` is a credential.** It authenticates the attendee into the
  event web app. Never log it, forward it, or store it in a system where it
  would be visible to anyone but that attendee.
