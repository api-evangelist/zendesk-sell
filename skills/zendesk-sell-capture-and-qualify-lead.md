---
name: Capture and qualify a Zendesk Sell lead
description: >-
  Create or match a lead in Zendesk Sell, then promote it into a contact and an open deal,
  using only operations that exist in the Sell OpenAPI held in this repo.
api: openapi/zendesk-sell-leads-api-openapi.yml
operations: [listLeads, createLead, upsertLead, getLead, upsertContact, createDeal]
generated: '2026-08-13'
method: generated
source: >-
  openapi/ specs in this repo plus conventions/zendesk-sell-conventions.yml,
  errors/zendesk-sell-problem-types.yml and rate-limits/zendesk-sell-rate-limits.yml
---

# Capture and qualify a Zendesk Sell lead

**Before you start.** Zendesk Sell retires on 2027-08-31 (announced 2025-09-09). Do not build
new long-lived automation on it; see `lifecycle/zendesk-sell-lifecycle.yml`.

## Setup

- Base URL: `https://api.getbase.com`
- Auth: `Authorization: Bearer <access_token>` — OAuth 2.0. A dashboard-minted personal token
  never expires; a token from `/oauth2/token` expires after **one hour** and must be
  refreshed. Writing requires the `write` scope; a missing scope returns **403
  `insufficient_scope`**.
- **Always send a `User-Agent` header.** Sell rejects requests without one: 400
  `invalid_user_agent`. Most default HTTP clients omit it.
- `Accept: application/json` and `Content-Type: application/json`.
- Every request and response body is wrapped: send `{"data": {...}}`, read `{"data": {...},
  "meta": {...}}`. Collections return `{"items": [{"data": ..., "meta": ...}], "meta": ...}`.

## Steps

1. **Check whether the lead already exists** — `listLeads` (`GET /v2/leads`) filtered on the
   attributes you have, e.g. `?email=jane@example.com&per_page=100`. Paginate with `page` and
   `per_page` (max 100). There is no cursor and no total count.
2. **Create or match** — prefer `upsertLead` (`POST /v2/leads/upsert`) when you have a stable
   matching attribute; it creates or updates in one call. Use `createLead`
   (`POST /v2/leads`) only when you know the record is new.
   - There is **no idempotency key** on this API. If a `createLead` call times out, do not
     blind-retry: re-run step 1 first, or switch to `upsertLead`. See
     `conventions/zendesk-sell-conventions.yml`.
3. **Read back** — `getLead` (`GET /v2/leads/{id}`) to confirm the stored state before acting
   on it. Custom data lives under `custom_fields`; the field must already be defined in the
   Sell account or the write is rejected with a resource error `unknown`.
4. **Promote to a contact** — `upsertContact` (`POST /v2/contacts/upsert`). Set
   `is_organization` to `true` for a company record, `false` for a person.
5. **Open the deal** — `createDeal` (`POST /v2/deals`) with `contact_id` set to the contact
   from step 4, plus `name`, `value`, `currency`, and `stage_id` for the target pipeline
   stage. `owner_id` assigns the rep.

## Error handling

Errors come back as `{"errors":[{"error":{"code","message","details","resource","field"}}],
"meta":{"http_status","logref"}}` — **not** RFC 9457 problem+json.

- `422` with `resource` + `field` (JSON Pointer): validation. Codes are `missing`, `blank`,
  `invalid_type`, `incorrect_value`, `already_exists`, `unknown`. Fix the payload; retrying
  unchanged will fail identically.
- `401 unauthorized`: token missing, malformed or expired — refresh and retry once.
- `429 rate_limit_exceeded`: the ceiling is 36,000 requests/hour per token (10/sec). **No
  rate-limit headers are returned**, so back off exponentially rather than reading a reset
  value.
- Always log `meta.logref` (equal to the `X-Request-Id` response header) — it is the only
  identifier Zendesk support can trace.
