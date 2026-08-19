---
name: Move a Zendesk Sell deal through the pipeline
description: >-
  Read, update and close deals in Zendesk Sell — stage changes, value updates and safe
  deletion — grounded in the deal operations published in this repo's OpenAPI.
api: openapi/zendesk-sell-deals-api-openapi.yml
operations: [listDeals, getDeal, createDeal, updateDeal, upsertDeal, deleteDeal, getContact]
generated: '2026-08-13'
method: generated
source: >-
  openapi/zendesk-sell-deals-api-openapi.yml plus conventions/, errors/ and
  agentic-access/zendesk-sell-agentic-access.yml
---

# Move a Zendesk Sell deal through the pipeline

**Before you start.** Zendesk Sell retires 2027-08-31. Deletions here are irreversible and the
API offers no undo.

## Setup

Base URL `https://api.getbase.com`, `Authorization: Bearer <token>`, `write` scope for any
mutation, and a non-empty `User-Agent` header (400 `invalid_user_agent` without it). Wrap
bodies in `data`.

## Steps

1. **Find the deal** — `listDeals` (`GET /v2/deals`). Filter with query parameters
   (`?contact_id=1`, `?ids=1,2,3`), sort with `sort_by=name:desc`, page with
   `page`/`per_page` (max 100). For complex boolean filters or aggregations, use the separate
   Search API (`https://developer.zendesk.com/api-reference/sales-crm/search/introduction`) —
   it is not part of these specs.
2. **Read current state** — `getDeal` (`GET /v2/deals/{id}`). Capture `stage_id` and `value`
   before changing anything so the change is reversible by hand.
   - Note the documented breaking change: **deal decimal values are serialized as JSON
     strings**. Parse `value` as a decimal string, never as a float.
3. **Confirm the counterparty** — `getContact` (`GET /v2/contacts/{id}`) on the deal's
   `contact_id` when the action depends on who the deal is with.
4. **Advance the stage** — `updateDeal` (`PUT /v2/deals/{id}`) with the new `stage_id`. Send
   only the fields you intend to change; this is a targeted update, not a full replace.
5. **Bulk import path** — `upsertDeal` (`POST /v2/deals/upsert`) creates or updates against
   your matching attributes. Use it instead of `createDeal` in any retryable job, because the
   API publishes no idempotency key and a retried `createDeal` produces a duplicate deal.
6. **Deletion is a last resort** — `deleteDeal` (`DELETE /v2/deals/{id}`) returns `204` and is
   permanent. Treat it as a human-confirmation step: in
   `agentic-access/zendesk-sell-agentic-access.yml` every write operation is classified
   `acting`, and destructive CRM writes should not run unattended.

## Error handling

- `404 not_found` — the deal id does not exist or the token's user cannot see it. Sell scopes
  are account-wide (`read`, `write`, `profile`), so visibility is a record-ownership question,
  not a scope question.
- `422` with `resource`/`field` — validation; `incorrect_value` on `stage_id` means the stage
  does not belong to that pipeline.
- `429 rate_limit_exceeded` — 36,000 req/hour per token, no headers, exponential backoff.
- Log `meta.logref` from every failure.
