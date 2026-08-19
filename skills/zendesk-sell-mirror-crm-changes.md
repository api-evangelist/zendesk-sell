---
name: Mirror Zendesk Sell records into a local store
description: >-
  Keep a local copy of Sell leads, contacts and deals current — a paginated backfill over the
  Core API followed by incremental catch-up on the pull-based Firehose stream.
api: openapi/zendesk-sell-contacts-api-openapi.yml
operations: [listLeads, listContacts, listDeals, getLead, getContact, getDeal]
generated: '2026-08-13'
method: generated
source: >-
  openapi/ specs in this repo, events/zendesk-sell-events.yml and
  conventions/zendesk-sell-conventions.yml
---

# Mirror Zendesk Sell records into a local store

Zendesk Sell **pushes nothing**. There are no webhooks. Everything below is a pull, and that
constraint drives the whole design. Sell retires 2027-08-31 — a mirror is also the export path
customers will need.

## Step 1 — Backfill with the Core API

Use `listLeads` (`GET /v2/leads`), `listContacts` (`GET /v2/contacts`) and `listDeals`
(`GET /v2/deals`).

- Page with `page` (1-based) and `per_page` (max 100). There is no cursor, no `next` link and
  no total count, so iterate until a short/empty `items` array.
- Sort deterministically (`sort_by=id:asc`) so concurrent writes do not shuffle records
  between pages.
- Budget against the ceiling: **36,000 requests/hour per token, 10/second**. At 100 records a
  page that is 3.6M records/hour theoretical, but there are no rate-limit headers, so pace
  yourself and treat every `429 rate_limit_exceeded` as a signal to back off exponentially.
- Fetch single records with `getLead`, `getContact`, `getDeal` only for repair, not in the
  hot loop.

## Step 2 — Catch up incrementally on Firehose

The Firehose API (documented, not in the OpenAPI held here) is the change feed:

- `GET https://api.getbase.com/v3/:resource/stream` — e.g. `/v3/deals/stream`, with the same
  `Authorization: Bearer` token.
- Required `position` (`top`, `tail`, or an opaque position string you carry forward) plus
  optional `limit` (1-100, default 25). Call it iteratively to drain the queue.
- Each event carries `data` (current snapshot) and `meta` with `event_id`, `event_type`
  (`created`/`updated`/`deleted`), `event_cause`, `event_time`, `sequence` and, on updates,
  `previous`.
- **Retention is 72 hours.** If your consumer is down longer than that, you must re-run
  step 1; there is no replay beyond the window.
- `sequence` is monotonic **per resource id**, so use it to discard out-of-order updates for a
  record. It is not a global ordering.
- 22 resource streams exist (contacts, deals, leads, notes, tasks, calls, appointments, line
  items, orders, products, sources, stages, tags, users, visits, documents and more) — see
  `events/zendesk-sell-events.yml`.

## Step 3 — Or use the Sync API

`https://api.getbase.com/v2/sync` is Sell's stateful synchronization protocol: it hands you the
changed set and you acknowledge it, so you never track changes yourself. It is documented as
**eventually consistent** (delivery guaranteed, ordering not) and is a **premium, separately
priced** offering — confirm entitlement before building on it.

## Rules

- Persist `meta.logref` / `X-Request-Id` alongside failures for support escalation.
- Treat `event_type: deleted` as a tombstone; Sell will not resend it after 72 hours.
- Reconcile decimal money fields as strings (`value` on deals), never as floats.
- All timestamps are ISO 8601 UTC; `9999-12-31T00:00:00Z` means "forever".
