---
name: relex-ingest-transaction-batch
description: Push a batch of retail transaction records (sales, deliveries, goods received,
  inventory events) into the RELEX planning platform through the RELEX Data API, and then
  CONFIRM the batch actually landed — because a 200 on the write only means accepted.
api: RELEX Data API
version: '1.0'
generated: '2026-08-26'
method: generated
source: openapi/relex-data-api-openapi.json,
  https://www.relexsolutions.com/api/retail-restapi-example-customer.html
operations:
- PostSales
- PostDailySales
- PostDeliveries
- PostGoodsReceived
- PostInventoryEvents
- PostAdjustments
- PostBalances
- PostLostSales
- GetErrors
- GetVersion
- GetHealth
---

# Ingest a transaction batch into RELEX

Every operationId named here was read out of `openapi/relex-data-api-openapi.json`. Nothing is
invented. If an operation is not in that file, it does not exist for this customer.

## Before you start

You need a Client ID, a Client Secret and a RELEX Identity token endpoint URL. **These cannot be
self-issued.** RELEX provides them during the implementation project. If you do not have them,
stop — there is no trial, sandbox key or anonymous path.

You also need to know which environment you are writing to. UAT and production are different
hosts *and* different RELEX Identity authorities, and UAT may be running a newer API version.

- production: `https://{geography}.rest.relexsolutions.com/{customer}`
- UAT: `https://uat-{geography}.rest.relexsolutions.com/{customer}`

## Step 1 — Get an access token

POST to the RELEX Identity token endpoint with
`Content-Type: application/x-www-form-urlencoded` and a body of
`client_id=…&client_secret=…&grant_type=client_credentials`.

Keep `expires_in`. **Client Credentials issues no refresh token.** Either refresh before expiry or
handle a `401` by re-running this step. Do not treat a 401 as a data error.

## Step 2 — Check you are pointed at the right contract

`GetVersion` (`GET /meta/version`) returns the API version for this environment. It is the sum of
a shared Core version and this customer's Data model version, so it will not match another
customer's number. `GetHealth` (`GET /meta/health`) is the liveness check.

Do this once per run, not per batch.

## Step 3 — Build the batch

The Data API is **batch-oriented**. One request carries a *list* of records in the body's `data`
array, and `POST` is an **upsert** — new records are created, matching records are updated, in the
same call.

In the body's `meta` section, always send:

- `batch_id` — a UUID you generate. **One batch_id per logically distinct request. Reuse the SAME
  batch_id on every retry of that request.** This is what makes retries safe.
- `timestamp` — a monotonically increasing integer. If several source systems write to one
  resource, they must share one timestamp source.

Hard limit: **1 MB per request.** Over that you get `413` with an RFC 7807 problem body. Split the
list and send the chunks as independent requests; each is processed separately.

Hard limit: **5 requests per second.** Over that you get `429`. There is no `Retry-After` header
and no `RateLimit-*` headers, so you cannot see your remaining budget — pace yourself at or below
5 rps and back off on 429 with your own schedule.

## Step 4 — Write

Pick the operation that matches the record type. The common transaction writes are:

| Operation | Path | What it carries |
|---|---|---|
| `PostSales` | `/data/transactions/sales` | Goods sold from stock; a customer return is modelled as a reversal record |
| `PostDailySales` | `/data/transactions/daily_sales` | Day-grain sales |
| `PostDeliveries` | `/data/transactions/deliveries` | Deliveries |
| `PostGoodsReceived` | `/data/transactions/goods_received` | Receipts |
| `PostInventoryEvents` | `/data/transactions/inventory_events` | Inventory movements |
| `PostAdjustments` | `/data/transactions/adjustments` | Stock adjustments |
| `PostBalances` | `/data/transactions/balances` | Stock balances |
| `PostLostSales` | `/data/transactions/lost_sales` | Lost sales |

Records are keyed on **business keys** — for almost all of these, `product` + `location` (+ a
date or timestamp). There is no surrogate id. Re-POSTing the same business key overwrites in
place.

Capture the `request_id` from the response. You need it for step 5.

## Step 5 — CONFIRM. Do not skip this.

**A `200` means the payload was accepted for ingestion. It does not mean the records were
processed.** Validation and processing failures appear afterwards, asynchronously, and only here:

`GetErrors` — `GET /meta/errors?request_id={request_id}`

Poll it. Because processing is asynchronous, the same query can return *more* errors on a later
call, so one empty result is not proof of success — poll until the result stops changing for your
batch. An empty result is `200 OK` with an empty `data` array.

If the error set is large, page it: `page` and `per_page` (max `10000`), and follow
`_links.next.href` until `next` is absent.

You can also sweep by time instead of by request: `start_timestamp` (inclusive) +
`end_timestamp` (exclusive). **The two modes are mutually exclusive** — do not send both.

## If something goes wrong

Errors are RFC 7807 `application/problem+json` with `type`, `title`, `status` and usually
`detail`.

- `400` — malformed payload, missing parameter, or the call was made over plain HTTP. HTTPS is
  mandatory.
- `401` — token missing, expired or revoked. Re-run step 1.
- `403` — usually the caller's egress IP is not on the RELEX IP allowlist for this environment.
- `404` — the resource is not in *this customer's* data model. Custom resources differ per
  customer.
- `413` — over 1 MB. Split and resend.
- `429` — over 5 rps. Back off.
- `500` / `503` — retryable.

## What you cannot undo

**There is no delete, void, cancel or reverse operation on any transaction resource, and no
dry-run mode anywhere in the contract.** If you write a wrong transaction you have two options,
both of which require knowing the data:

1. Re-POST the same business key with corrected values (upsert overwrites in place), or
2. Post a compensating record — the sales resource explicitly models a customer return as a
   reversal.

RELEX states no reversal window for either. Only 8 **master-data** resources have a `DELETE`
(campaigns, delivery schedules, batch sizes, pre-pack variants, product/location variants,
product replacements and references) — and deletion itself is not reversible.

Because of this, treat every transaction write as final and validate before you send.
