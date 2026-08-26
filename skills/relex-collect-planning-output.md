---
name: relex-collect-planning-output
description: Receive RELEX webhook notifications that planning output is ready (sales forecasts,
  order proposals, workload driver forecasts), verify the signature, and pull the actual records —
  the webhook carries metadata only.
api: RELEX Data API
version: '1.0'
generated: '2026-08-26'
method: generated
source: openapi/relex-data-api-openapi.json,
  https://www.relexsolutions.com/api/retail-restapi-example-customer.html
operations:
- list_GetSalesForecasts
- get_GetSalesForecasts
- list_GetOrderProposals
- get_GetOrderProposals
- list_GetWorkloadDriverForecastsDay
- get_GetWorkloadDriverForecastsDay
- list_GetWorkloadDriverForecastsQuarter
- get_GetWorkloadDriverForecastsQuarter
---

# Collect planning output from RELEX

RELEX pushes a notification when planning output becomes available, then you pull it. This skill
covers both halves. Every operationId below was read out of
`openapi/relex-data-api-openapi.json`.

## The shape of it

> "Only the `metadata` is sent to the endpoint. To get the actual `data` subsequent request to the
> `rest-api` should be executed." — RELEX Data API reference

So a webhook is a pointer, never a record. Two steps, always.

## Step 1 — Receive and verify the webhook

RELEX delivers through **Svix**. Each POST to your endpoint carries:

- `SVIX-ID` — unique message identifier. Use it to de-duplicate redeliveries.
- `SVIX-TIMESTAMP` — seconds since epoch.
- `SVIX-SIGNATURE` — space-delimited Base64 signatures, HMAC-SHA256.

Verify with the Svix SDK (RELEX's recommendation) or manually. You need the signing secret RELEX
issued you.

Two configuration facts that break receivers if you miss them:

- **Disable CSRF protection on this endpoint.** Webhooks are server-initiated; CSRF only protects
  client-initiated requests.
- If you are behind a firewall, allowlist Svix's published static source IP ranges
  (https://docs.svix.com/receiving/source-ips).

Return any `2xx` **within 15 seconds**. If you do not, RELEX retries on a published schedule:
immediately, 5s, 5m, 30m, 2h, 5h, 10h, then 10h again. Acknowledge fast and process
asynchronously — do not do the pull in the webhook handler.

## Step 2 — Read the notification body

```json
{
  "data": [
    { "id": "…", "resource": "sales_forecasts", "timestamp": "2024-04-03T07:15:22.567Z",
      "url": "/data/transactions/sales_forecasts/…" }
  ],
  "meta": { "created_at": "…", "type": "data.transactions.notify" }
}
```

`meta.type` follows `data.<namespace>.notify`; the namespace is `transactions` for standard
planning output and `custom` for customer-specific resources. Each `data[]` entry names the
resource, gives you the `id`, and gives you the relative `url` to fetch.

## Step 3 — Pull the records

Authenticate exactly as for any Data API call: OAuth 2.0 Client Credentials against RELEX
Identity, `Authorization: Bearer <jwt>`, no refresh token.

| What | List | By id |
|---|---|---|
| Sales forecasts | `list_GetSalesForecasts` | `get_GetSalesForecasts` |
| Order proposals | `list_GetOrderProposals` | `get_GetOrderProposals` |
| Workload driver forecasts (day) | `list_GetWorkloadDriverForecastsDay` | `get_GetWorkloadDriverForecastsDay` |
| Workload driver forecasts (quarter) | `list_GetWorkloadDriverForecastsQuarter` | `get_GetWorkloadDriverForecastsQuarter` |

Prefer the by-id form when you have a webhook — you were handed the exact `id`, and it is the only
place in the contract where a RELEX record has a surrogate identifier.

**These list endpoints are not paginated.** RELEX states it "imposes no limits on return payloads;
the payload size can be arbitrarily large". Stream the response rather than buffering it, and be
ready for a large body. Only `/meta/errors` implements `page`/`per_page`.

Read requests count against the same 5 requests-per-second limit. `GET` is idempotent and has no
side effects, so retrying a read is always safe.

## If you are polling instead of receiving webhooks

The list operations work standalone. But you will not know when new output landed, and there is no
`If-Modified-Since`, ETag or cursor documented — so polling means repeatedly pulling an unbounded
payload against a 5 rps budget. Take the webhook if you can get it.
