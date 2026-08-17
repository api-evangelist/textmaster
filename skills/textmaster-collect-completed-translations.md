---
name: Collect completed translations and reconcile spend
description: >-
  Retrieve finished content from completed documents, mint human review links, confirm completion,
  use MongoDB-style filters to find work needing attention, and reconcile transactions and invoices
  against launched projects.
api: openapi/textmaster-api-v1-openapi.yml
operations:
  - 'GET /v1/clients/projects/{project_id}/documents'
  - 'GET /v1/clients/projects/{project_id}/documents/{document_id}'
  - 'GET /v1/clients/projects/{project_id}/documents/filter'
  - 'PUT /v1/clients/projects/{project_id}/documents/{document_id}/complete'
  - 'POST /v1/clients/projects/{project_id}/documents/{document_id}/review_url'
  - 'GET /v1/clients/transactions'
  - 'GET /v1/clients/invoices'
scopes:
  - 'project:read'
  - 'project:write'
  - 'transaction:read'
generated: '2026-08-17'
method: generated
source: >-
  https://developer.textmaster.com/guides/integrator-best-practices +
  https://developer.textmaster.com/overview/filters +
  https://developer.textmaster.com/overview/workflow +
  openapi/textmaster-api-v1-openapi.yml
---

# Collect completed translations and reconcile spend

The retrieval half of the workflow. Ordering is covered in
`skills/textmaster-quote-and-order-translation.md`.

## Prefer being pushed over pulling

The provider's guidance is explicit: subscribe to the document `completed` event and let TextMaster
tell you the work is ready, rather than polling. The stated reason is payload size — pulling finished
documents over the API risks the 30-second timeout. Set up the receiver first
(`skills/textmaster-subscribe-to-project-events.md`), then use the operations below to fetch on
notification.

## Step 1 — find documents that need attention

`GET /v1/clients/projects/{project_id}/documents/filter` (scopes `project:read` / `project:manage`)

Use the filter endpoint rather than paging the whole collection:

```shell
curl -G \
  --data-urlencode 'where={"status":{"$in":["completed","in_review"]}}' \
  --data-urlencode 'order=-updated_at' \
  https://api.textmaster.com/v1/clients/projects/PROJECT_ID/documents/filter \
  -H "Authorization: Bearer ACCESS_TOKEN"
```

Operators: `$gt` `$gte` `$lt` `$lte` `$in` `$nin` `$ne` `$or` `$regex` (PCRE, flags `i`/`m`/`x`).
`order` is a comma-separated field list; a `-` prefix sorts descending. An unsupported selector
returns **422**.

The plain list (`GET /v1/clients/projects/{project_id}/documents`) pages at 100 items with
`page` (1-based) and `per_page` (max 100, not honoured on every endpoint). If you are getting fewer
results than you expect, you are on page 1 — this is the single most common integrator mistake
according to the provider's own troubleshooting page.

## Step 2 — retrieve the finished content

`GET /v1/clients/projects/{project_id}/documents/{document_id}` (scopes `project:read` /
`project:manage`)

The finished translation is behind the **`author_work`** field, typed in the spec as
`oneOf: object | string`. When it is a string it is a **URL** to the content.

**Do not parse or construct that URL.** From the best-practices guide:

> "For the stability of your app, you shouldn't try to parse this data or try to guess and construct
> the format of future URLs. Your app is liable to break if we decide to change the URL."

Follow it as given. Also follow redirects — the API "uses HTTP redirection where appropriate" and
every redirect sets `Location`; receiving one is not an error.

Other fields worth reading on the document:

- `status` — the authoritative state; do not infer it from webhook arrival order.
- `completion`, `progress`, `written_words` — quantitative progress.
- `satisfaction` — nullable client rating.
- `reference` — the client-supplied reference you set at creation; your correlation handle.
- `can_post_message_to_author` — whether the support thread is open.

## Step 3 — send content for human review (optional)

`POST /v1/clients/projects/{project_id}/documents/{document_id}/review_url` (scopes `project:write`
/ `project:manage`)

Mints a review URL a human can open to inspect and comment on the delivered work. This is the right
move when an agent is in the loop but a person owns acceptance — hand off the link rather than
auto-accepting.

Related, if the reviewer needs to talk to the translator:

- `GET /v1/clients/projects/{project_id}/documents/{document_id}/support_messages` (`discussion:read`)
- `POST /v1/clients/projects/{project_id}/documents/{document_id}/support_messages` (`discussion:write`)

The counterparty is a human author expecting a human; the `support_message_created` event fires when
they reply.

## Step 4 — confirm completion

`PUT /v1/clients/projects/{project_id}/documents/{document_id}/complete` (scopes `project:write` /
`project:manage`)

Or, for many at once,
`POST /v1/clients/projects/{project_id}/batch/documents/complete` — batch it for the same
30-second-timeout reason you batch creates.

This is a state transition on paid work. There is no idempotency key, so make your caller
idempotent: re-read the document status before completing, and treat "already completed" as success
rather than retrying blindly.

## Step 5 — reconcile the spend

Under `transaction:read` (or `transaction:manage` / `transaction:write`):

- `GET /v1/clients/transactions` — movements against the prepaid credit wallet.
- `GET /v1/clients/invoices` — issued invoices.
- `GET /v1/clients/receipts` — payment receipts.
- `GET /v1/clients/negotiated_contracts` — bespoke commercial terms on the account.

**Known limitation, and it matters:** none of these schemas carries a `project_id` or `document_id`.
The API models no foreign key from a financial record back to the work that produced it. You cannot
join spend to a project through the API — reconciliation has to be done on `issued_at` timestamps and
amounts, correlated against the launch times you recorded yourself.

Practical mitigation: log every `async_launch` call with its project `id`, your `external_id`, the
project cost read immediately post-finalize, and the response `X-Request-Id`. That local ledger is
what makes reconciliation possible later. See `data-model/textmaster-data-model.yml`.

## Cost changes to watch for

Two events re-price a project after creation — `project_tm_completed` and
`project_tm_diff_completed`, both documented as updating the cost "accordingly". If your
reconciliation compares against the original quotation you will see phantom discrepancies. Compare
against the cost read after finalization instead.

## Cleanup

`DELETE /v1/clients/projects/{project_id}/documents/{document_id}` (scopes `project:write` /
`project:manage`) removes a document. Only meaningful before launch; after launch the work is in
flight with a human author.

Uploaded files that were never linked to a document are deleted automatically after **60 days** from
the temporary store.

## Error handling

- **403** — missing scope, not a bad token. `transaction:read` is separate from `project:read`; an
  integration that can read projects cannot necessarily read invoices.
- **404** — `{"errors":{"base":["Resource not found."]}}`. Check the id; ids are bare 24-hex strings
  with no type prefix, so passing a `project_id` where a `document_id` belongs produces exactly
  this.
- **422** — field-keyed validation messages, or an unsupported filter selector.
- **Timeout at 30 seconds** — your page or batch is too large. Split it.
- No rate limit is published and no `429` is declared anywhere in the spec, but "intentionally
  ignoring repeated validation errors may result in the suspension of your app for abuse." Back off
  on repeated 4xx rather than looping.

Full catalogue: `errors/textmaster-problem-types.yml`.
