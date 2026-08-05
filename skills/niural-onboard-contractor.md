---
name: Onboard a contractor onto a pay-on-demand contract
description: >-
  Create a Niural pay-on-demand contract for a contractor, sign it as the
  employer, invite the contractor to counter-sign, and confirm the contract
  reached a signed state.
api: openapi/niural-public-api-openapi.yml
operations:
  - POST /authenticate
  - POST /contracts
  - PATCH /contracts/{contract-id}/sign
  - PATCH /contracts/{contract-id}/invite-contractor
  - GET /contracts/{contract-id}
generated: '2026-08-04'
method: generated
---

# Onboard a contractor onto a pay-on-demand contract

Use the sandbox host `https://api-sandbox.niural.com` until the flow is proven,
then switch the host to `https://api-live.niural.com`. Nothing else changes
between environments — Niural has no test-mode key prefix.

## Before you start

- You need a `client_id` and `client_secret` created in the Niural dashboard under
  **Organization → Developer → API Keys**. The secret is shown once.
- Every request needs `Accept: application/json` and `Content-Type: application/json`.
- Stay under **25 requests/second** or you will get a `429`
  (`rate-limits/niural-rate-limits.yml`).

## Step 1 — Get a bearer token

`POST /authenticate` with `{ "client_id": "...", "client_secret": "..." }`.

The response is `{"data": {"access_token", "refresh_token", "expires_in"}}`.
`expires_in` is **3600 seconds**. Send `Authorization: Bearer <access_token>` on
every subsequent call. When it expires, use the `refresh_token` rather than
re-sending the client secret.

## Step 2 — Create the contract

`POST /contracts`. The request body is a `oneOf`; for this flow use the
**Pay On Demand Contract Request** variant with `contract_type: "PAY_ON_DEMAND"`.
Supply at minimum `contract_title`, `contractor_type`
(`INDIVIDUAL` or `ENTITY`), the contractor's name and email, start date, job
title, and `scope_of_work`. You may attach a free-form `tags` object.

There is **no idempotency key on this API**. If the call times out, do NOT blindly
retry — call `GET /contracts` filtered by `contractor_id` first and check whether
the contract already exists.

Read `data.contract_id` from the response and carry it forward.

## Step 3 — Sign as the employer

`PATCH /contracts/{contract-id}/sign`. No body is required.

## Step 4 — Invite the contractor to sign

`PATCH /contracts/{contract-id}/invite-contractor`, optionally with
`{ "message_for_contractor": "..." }`.

The contract now sits at `CONTRACTOR_SIGN_PENDING` until the contractor acts.
There is no API operation that signs on the contractor's behalf.

## Step 5 — Confirm

`GET /contracts/{contract-id}` and read `contract_status`. Do not poll faster than
the rate limit allows; prefer subscribing to the `contract.status.updated` webhook
(`asyncapi/niural-webhooks.yml`) and reconciling with this call.

## Errors

`400` invalid request, `404` unknown contract, `422` validation error, `500`
server error — all returned as `{"error_code", "message"}`. `error_code` is a
free-form string with no published registry. See `errors/niural-problem-types.yml`.
