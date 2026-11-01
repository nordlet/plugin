---
name: developer-webhooks
description: Receive Nordlet events in your own system - webhook subscriptions and the event catalogue, the delivery envelope and headers, verifying the HMAC signature and the timestamp, retries, pausing after failures, the delivery log and redelivery. Use when an integrator asks to be notified when an invoice is issued or paid, a run is approved or stock changes, to debug a missing or failed delivery, or to verify a signature.
---

# Webhooks in Nordlet

Calls go through `call_operation` with the path in camelCase as `operationId` (`webhooks/subscriptions/create` → `webhooksSubscriptionsCreate`); the API key needs the `webhooks` scopes. Conventions are in the `nordlet` skill. Reference: https://nordlet.com/en/docs/guides/api-conventions/#webhooks and the `x-webhooks` section of the OpenAPI document at https://nordlet.com/en/docs/api/, which describes each event's payload.

## 1. Subscribe

`webhooks/subscriptions/create` with `url` (https, resolving to a public network; 422 otherwise), `events` (an array of event names, or `["*"]` for every event; an unknown name is 400) and optional `secret`. The signing secret is generated when not given, and this call is the only one that returns it: store it at once.

- `webhooks/subscriptions/list` shows the subscriptions with their last delivery state (filters `url`, `isActive`).
- `webhooks/subscriptions/update` with `id` changes `url`, `events` or `isActive`. After 5 deliveries in a row have failed the subscription is paused and the company owners get an e-mail; `isActive: true` clears `pausedAt` and `consecutiveFailures` and switches it back on.
- `webhooks/subscriptions/delete` removes it.

Events: `agreement.invoice_generated`, `bank_feed.synced`, `document_capture.peppol_received`, `filing.failed`, `filing.rejected`, `goods_receipt.posted`, `intercompany.invoice_mirrored`, `item.created`, `item.deleted`, `item.updated`, `lead.converted`, `lead.created`, `partner_inquiry.created`, `payroll_run.approved`, `payroll_run.reversed`, `pos_report.created`, `price_list.updated`, `purchase_invoice.paid`, `purchase_invoice.registered`, `purchase_order.approved`, `purchase_order.received`, `recurring_invoice.generated`, `refund_liability.actual`, `refund_liability.trued_up`, `report.completed`, `report.failed`, `revenue_recognition.modified`, `revenue_recognition.posted`, `sale_invoice.einvoice_sent`, `sale_invoice.issued`, `sale_invoice.paid`, `sale_invoice.peppol_delivered`, `sale_invoice.peppol_failed`, `sale_invoice.peppol_rejected`, `sale_invoice.peppol_sent`, `sale_invoice.quote_sent`, `sale_invoice.sent`, `sales_order.created`, `sales_order.fulfilled`, `settlement.imported`, `settlement.posted`, `settlement.updated`, `stock.changed`, `stock.reorder_needed`, `vat_review.opened`, `vat_review.resolved`.

## 2. What arrives

Events are written to an outbox in the same database transaction as the change: a rolled-back document never emits an event and no event is lost.

Each delivery is a `POST` to the URL with a JSON body `{ id, version, companyId, eventType, createdAt, data }`: `version` is the envelope version (currently 1), `companyId` the company the event belongs to (one receiver can serve several companies), `data` the event's payload. Headers:

| Header | Content |
|---|---|
| `x-nordlet-timestamp` | unix seconds, the time the delivery was signed |
| `x-nordlet-signature-v2` | `t=<timestamp>,v1=<hex HMAC-SHA256 of "<timestamp>.<raw body>">` with the subscription secret |
| `x-nordlet-signature` | `sha256=<hex HMAC-SHA256 of the raw body>`, kept for older receivers |
| `x-nordlet-delivery` | the delivery id, the same on every retry of one delivery |

A non-2xx answer or a redirect is retried after 30 seconds, 2 minutes, 10 minutes, 30 minutes, 1 hour, 3 hours, 6 hours and 12 hours, about 24 hours in all, then the delivery is marked `failed`. One server sends a delivery at a time, so several API instances do not send it twice.

## 3. Verify

Before trusting a payload:

1. Compute HMAC-SHA256 over `<x-nordlet-timestamp>.<raw request body>` with the secret and compare it with `v1` of `x-nordlet-signature-v2` using a constant-time comparison. Sign the raw bytes as received, not a re-serialised JSON.
2. Reject a delivery whose timestamp is more than 5 minutes away from the receiver's clock. The timestamp is inside the signed text, so a captured delivery cannot be replayed later with a fresh timestamp.
3. Store the `x-nordlet-delivery` ids already processed and ignore a repeat within that window; retries carry the same id.
4. Answer with a 2xx status; any other status or a redirect is retried.

## 4. Deliveries and redelivery

`webhooks/deliveries/list` lists the company's deliveries newest first with filters `status`, `eventType`, `subscriptionId`, and shows the attempt count and the last error of each. `webhooks/deliveries/redeliver` with `id` sends one again now: its status goes back to `pending` with the attempts and the last error cleared. 409 when the subscription is inactive; reactivate it first.

When an event seems missing: check that the subscription's `events` includes it, that `isActive` is true (a paused subscription delivers nothing), and read the delivery row for the receiver's answer. `audit/list` (filters `action`, `entity`, `entityId`) shows whether the change itself was recorded.
