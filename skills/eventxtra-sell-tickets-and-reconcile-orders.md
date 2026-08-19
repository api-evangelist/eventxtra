---
name: Quote, place and reconcile EventX ticketing orders
description: Read an event's payment config and registration form, quote a basket, create the order, poll its status, then reconcile against orders and invoices.
api: openapi/eventxtra-public-api-openapi.json
base_url: https://esaas-api.eventx.io
operations: [issueJwt, getAllTicketClassByEventId, getAllTicketClassAddOnByEventId, getPaymentConfigByEventId, getRegFormByEventIdAndId, getOrderQuotation, createOrder, getOrderStatus, getPaginatedOrderByEventId, getOrderById, getOneOrderInvoice, invoiceDetails]
generated: '2026-08-13'
method: generated
source: openapi/eventxtra-public-api-openapi.json + https://eventx-hq.gitbook.io/knowledge-base/api-doc/registration-order
---

# Quote, place and reconcile EventX ticketing orders

The ticketing flow spans two path families: the organizer-side
`/public-api/v1/event/{eventId}/…` reads and the registration-side
`/public-api/v1/reg/event/{eventId}/…` order operations.

## Steps

1. **Authenticate** — `issueJwt` (`POST /public-api/v1/auth`).

2. **Read what is for sale** — `getAllTicketClassByEventId`
   (`GET /public-api/v1/event/{eventId}/ticket-class/all`) and
   `getAllTicketClassAddOnByEventId`
   (`GET /public-api/v1/event/{eventId}/ticket-class-add-on/all`).
   New ticket classes are created with `createTicketClass`
   (`POST /public-api/v1/event/{eventId}/ticket-class`).

3. **Read the payment configuration** — `getPaymentConfigByEventId`
   `GET /public-api/v1/reg/event/{eventId}`. Payment is Stripe-backed on this
   surface (`stripeAccountId` / `stripeId` appear in the configuration and order
   schemas).

4. **Read the rendered registration form** — `getRegFormByEventIdAndId`
   `GET /public-api/v1/reg/event/{eventId}/{eventRegFormId}` — returns the form
   with its tickets and fields. Create one with `createRegForm`
   (`POST /public-api/v1/event/{eventId}/registration-form`).

5. **Quote before you commit** — `getOrderQuotation`
   `POST /public-api/v1/reg/event/{eventId}/quotation` returns the price for a
   basket **without creating an order**. Always quote first and show the total
   before step 6.

6. **Create the order** — `createOrder`
   `POST /public-api/v1/reg/event/{eventId}`.

7. **Poll the order status** — `getOrderStatus`
   `GET /public-api/v1/reg/event/{eventId}/{orderId}/status`.

8. **Reconcile** — `getPaginatedOrderByEventId`
   (`GET /public-api/v1/event/{eventId}/order`) and `getOrderById`
   (`GET /public-api/v1/event/{eventId}/order/{orderId}`).
   Invoices: `getOneOrderInvoice`
   (`GET /public-api/v1/event/{eventId}/invoice/{orderId}`) and `invoiceDetails`
   (`POST /public-api/v1/event/{eventId}/invoice/details`).

## Rules

- **`createOrder` is money-moving and NOT replay-safe.** EventX documents no
  idempotency key of any kind. If `createOrder` times out or returns a 500, do
  **not** retry it. Reconcile first — page `getPaginatedOrderByEventId` (sorted
  `-createdAt`) or, if you captured an id, call `getOrderStatus` — and only
  re-issue once you have positively established that no order exists.
- **A 500 does not mean the charge failed.** The contract declares only 200 /
  204 / 500, with no 402, 409 or 422, so a payment decline, a validation error
  and a genuine server fault are indistinguishable by status. Read
  `error.code` and `error.message` from the `{ error: {...} }` body, then verify
  against the order list before telling anyone their payment failed.
- **Quote and order are separate calls with no lock between them.** Prices and
  availability can move; re-quote if the buyer pauses.
- **Platform fee differs by plan** — 5% on Essentials, 0% on Enterprise. Do not
  compute a net figure without knowing the organizer's plan.
