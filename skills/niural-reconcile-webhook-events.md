---
name: Reconcile webhook events against the REST API
description: >-
  Verify Niural webhook signatures, deduplicate at-least-once deliveries, and run
  the reconciliation sweep Niural itself recommends because delivery is not
  guaranteed.
api: openapi/niural-public-api-openapi.yml
operations:
  - POST /authenticate
  - GET /contracts
  - GET /invoices
  - GET /transactions
generated: '2026-08-04'
method: generated
---

# Reconcile webhook events against the REST API

Niural states plainly that webhook delivery **is not guaranteed** and that an
endpoint **may receive the same event more than once**. Treat the webhook as a
latency optimisation over the REST API, never as the system of record.

## Verify every delivery

Each POST carries `X-Niural-Signatures` (a comma-separated list) and
`X-Niural-Timestamp`. Compute
`HMAC-SHA256(signing_key, "{raw_body}.{timestamp}")` as a hex digest and accept
the request only if that digest appears in the header list.

The list is plural because Niural keeps up to **4 previous signing keys valid for
24 hours** after a rotation. Never accept an unsigned request.

## Deduplicate

The envelope is `{"meta": {"event_type", "organization_id", "tracking_id"},
"resource": {...}}`. Key your dedupe log on `meta.tracking_id` and drop anything
you have already processed. Acknowledge with `200 OK` first, then process
asynchronously.

## The three events

- `contract.status.updated` — reconcile with `GET /contracts`
- `invoice.status.updated` — reconcile with `GET /invoices`
- `transaction.status.updated` — reconcile with `GET /transactions`

The events table and the sample payloads disagree on casing
(`Invoice.status.updated` vs `invoice.status.updated`,
`transactions.` vs `transaction.`). Match `meta.event_type` case-insensitively.

Subscriptions currently only support **All Events** — there is no server-side
filter, so your handler must switch on `meta.event_type` itself.

## The reconciliation sweep

1. `POST /authenticate` for a bearer token.
2. Walk each collection with cursor pagination: send `limit`, then follow
   `next_cursor` until it is `null`. Default page size is 20.
   - `GET /contracts` filters: `contract_type`, `contractor_id`, `contract_status`
   - `GET /invoices` filters: `contract_type`, `contract_id`, `invoice_status`
   - `GET /transactions` filters: `transaction_status`
3. Diff against the state your webhook handler recorded; replay anything missing.

Stay under **25 requests/second**. There are no `RateLimit-*` response headers, so
pace the sweep yourself rather than reacting to a `429`.
