---
name: Invoice a contract and settle it with a transaction
description: >-
  Raise an invoice against an existing Niural contract, price the payment with an
  estimate, and initiate the transaction that settles it.
api: openapi/niural-public-api-openapi.yml
operations:
  - POST /authenticate
  - POST /invoices
  - GET /payment-methods
  - POST /transaction-estimates
  - POST /transactions
  - GET /transactions/{transaction-id}
generated: '2026-08-04'
method: generated
---

# Invoice a contract and settle it with a transaction

**This skill moves real money on the live host.** Niural publishes no
idempotency-key mechanism, so `POST /transactions` is not safe to retry blind.
Treat it as a one-shot, confirm-by-read operation.

## Step 1 — Get a bearer token

`POST /authenticate` with the client credentials; use
`Authorization: Bearer <access_token>` for everything below.

## Step 2 — Create the invoice

`POST /invoices` (`operationId: createInvoice`). Required fields:
`contract_id`, `currency`, `invoice_title`, `amount`, `issued_date`, `due_date`.
Optional: `cycle_start_date`, `cycle_end_date`, and a free-form `tags` object.

Take `invoice_id` from the response — it is a prefixed string like
`AUTOM-4EWB7EEM`, not a UUID.

If the request times out, reconcile with `GET /invoices`
(`operationId: listInvoices`) filtered by `contract_id` before creating again.

## Step 3 — Choose a payment method

`GET /payment-methods` supports `limit`, `next_cursor`, `status` and `is_primary`.
Take the `payment_account_id` you intend to debit.

## Step 4 — Estimate before you pay

`POST /transaction-estimates` with `transfer_type` and `invoice_ids` (both
required). This returns the amount and fee breakdown without moving money. It is
safe to retry. Always run it before step 5 so the fee is known and can be
confirmed by a human if the agent is operating with a spend ceiling.

## Step 5 — Initiate the transaction

`POST /transactions` with `transfer_type` and `invoice_ids` (required), plus
`payment_account_id`, optional `crypto_payment_details`, and optional `tags`.

Record the returned transaction id immediately.

## Step 6 — Confirm settlement

`GET /transactions/{transaction-id}` returns `fee_amount`, `fee_breakdown`,
`sub_total_amount` and `associated_invoices`. Subscribe to the
`transaction.status.updated` webhook rather than tight-polling; Niural itself
recommends a periodic reconciliation job because webhook delivery is not
guaranteed.

For an on-chain settlement, attach the proof with
`PATCH /transactions/{transaction-id}` supplying `payment_hash` and `chain_id`.

## Errors and limits

`400` / `404` / `422` / `500` per `errors/niural-problem-types.yml`. `429` at 25
requests/second. Nothing here is transactional across steps — an invoice created
in step 2 stays created even if step 5 fails.
