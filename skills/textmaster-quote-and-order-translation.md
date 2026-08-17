---
name: Quote and order a translation project
description: >-
  The marquee TextMaster flow. Price work without committing money, create a project, batch-attach
  documents, finalize so translation-memory analysis can re-price it, then launch and debit the
  prepaid credit wallet — with the guardrails this API needs because it publishes no idempotency key.
api: openapi/textmaster-api-v1-openapi.yml
operations:
  - 'GET /v1/public/languages'
  - 'GET /v1/public/expertises'
  - 'GET /v1/clients/projects/quotation'
  - 'POST /v1/clients/projects'
  - 'POST /v1/clients/projects/{project_id}/batch/documents'
  - 'PUT /v1/clients/projects/{project_id}/finalize'
  - 'POST /v1/clients/projects/{project_id}/async_launch'
  - 'GET /v1/clients/projects/{project_id}'
scopes:
  - 'project:quote'
  - 'project:write'
  - 'project:launch'
consequence: physical
human_in_the_loop: recommended
generated: '2026-08-17'
method: generated
source: >-
  https://developer.textmaster.com/quick-start +
  https://developer.textmaster.com/overview/workflow +
  https://developer.textmaster.com/guides/integrator-best-practices +
  https://developer.textmaster.com/webhooks-and-events/events +
  openapi/textmaster-api-v1-openapi.yml
---

# Quote and order a translation project

**This flow spends money.** `PUT .../finalize` and `POST .../async_launch` debit the client's
prepaid credit wallet, and the API provides **no idempotency key**. Read the "Spend safety" section
before automating the launch step.

Prerequisites: an OAuth token carrying `project:quote`, `project:write` and (only if you intend to
launch) `project:launch`. See `skills/textmaster-authenticate-oauth-app.md`.

Model in one line: a **Project** holds the commercial and linguistic parameters; **Documents** hold
the content; launching the project publishes its documents to human authors.

## Step 1 — resolve the vocabulary (unauthenticated)

- `GET /v1/public/languages` — valid source and target language codes.
- `GET /v1/public/expertises` — subject-matter expertises. Drill into
  `GET /v1/public/expertises/{expertise_id}/sub_expertises` for the second level.
- `GET /v1/public/categories` — content categories (a closed enum, `C001`–`C010`).

Do not guess these values. Expertise and language level both affect price and author matching.

## Step 2 — get a quotation before creating anything

`GET /v1/clients/projects/quotation` (scope `project:quote`)

Query parameters, taken verbatim from the spec:

```
project[activity_name]
project[language_from]
project[language_to]
project[options][expertise]
project[options][language_level]
project[options][quality]
project[options][priority]
project[options][specific_attachment]
project[total_word_count]
currency_code
```

This is a true dry run: it creates nothing and debits nothing. TextMaster publishes **no rate card
anywhere**, so this operation *is* the price list. Always quote before ordering, and surface the
number to a human if you are acting on their behalf.

Note the `locale` parameter on this operation is marked `[DEPRECATED]` in the spec. Do not use it.

## Step 3 — create the project

`POST /v1/clients/projects` (scopes `project:write` / `project:manage`)

Creating a project does **not** spend money. Key body fields:

- the same language pair, activity, category and options you quoted;
- `auto_launch` — **leave this `false`** (the default) for any automated flow. With `true`, the
  project launches itself as soon as async work finishes, which removes your chance to check the
  final price.
- `external_id` — **use this.** It is your own correlation field, and since the API has no
  idempotency key it is the only way to recognise a project you already created. Put a
  deterministic key from your own system here.
- `callback` — register project-level webhooks now, at creation time. See
  `skills/textmaster-subscribe-to-project-events.md`.
- `author_should_use_rich_text` — copywriting projects only.

Keep the returned `id`. Ids are bare 24-character hex strings with no type prefix, so track which
id is which yourself.

## Step 4 — attach documents in batches

`POST /v1/clients/projects/{project_id}/batch/documents` (scopes `project:write` / `project:manage`)

**Use the batch operation, not the single-document create.** TextMaster terminates any request that
takes more than 30 seconds, and the provider's own worked example shows that large creates hit that
wall. Their guidance: batches of **about 10 documents or fewer**, splitting further for large
content.

Two more rules from the best-practices guide:

- **Prefer file URLs over inline plain text.** Either host the file yourself and pass a publicly
  reachable URL (TextMaster copies it), or call `POST /v1/clients/upload_properties` for signed
  upload properties and push the file to TextMaster's temporary store. Unlinked uploads are deleted
  after 60 days.
- A project cannot hold more than **99,999** documents.

Per-document `callback` maps can be set here too, for document-level status events.

## Step 5 — finalize, then RE-READ the price

`PUT /v1/clients/projects/{project_id}/finalize` (scopes `project:launch` / `project:manage`)

Finalization runs translation-memory and/or PEMT analysis. **This changes the cost.** The provider
says so twice, in the descriptions of the `project_tm_completed` and `project_tm_diff_completed`
events: the project's "cost has been updated accordingly."

So after finalize:

1. Wait for `project_tm_completed` (and `project_tm_diff_completed` if Force Exact Matches is on),
   or poll `GET /v1/clients/projects/{project_id}`.
2. Compare the current cost against the Step 2 quotation.
3. If it moved materially, stop and confirm with a human before launching.

Never cache the Step 2 quote and launch against it.

## Step 6 — launch (this spends)

`POST /v1/clients/projects/{project_id}/async_launch` (scopes `project:launch` / `project:manage`)

Use the **async** variant. The synchronous sibling
`PUT /v1/clients/projects/{project_id}/launch` exists but risks the 30-second timeout on any
realistic project, and the docs direct integrators to async endpoints.

Completion is signalled by the `project_in_progress` event — "triggered when a project has launched
and is made available to be claimed by an author."

## Spend safety — read this before automating Step 6

There is no `Idempotency-Key` on this API (zero matches for "idempoten" in the whole 246KB spec).
The provider's documented reconciliation pattern is time-and-event based:

> "Launch process can safely be retried if this event [`project_in_progress`] is not received in a
> reasonable time (more than 30 minutes)."

Therefore:

1. **Do not blind-retry a launch that timed out.** A 30-second client timeout does not mean the
   launch failed.
2. On an ambiguous response, `GET /v1/clients/projects/{project_id}` and read the status.
3. Only retry the launch if `project_in_progress` has not arrived after ~30 minutes.
4. Handle `project_not_launched` — it fires when an `auto_launch` project "cannot be launched, often
   due to the client account not having enough credits on its wallet." This is your
   insufficient-funds signal and it arrives by webhook, not as an API error.
5. Consider gating `project:launch` behind a human approval in your own system. It is the single
   scope in this API that moves money, which makes it a clean place to put a confirmation step.

## Step 7 — watch it through the workflow

`GET /v1/clients/projects/{project_id}` returns status, cost, progress and document counts.

To find work needing attention across many projects, use the filter endpoint rather than paging
everything — `GET /v1/clients/projects/filter` accepts MongoDB-style selectors:

```shell
curl -G \
  --data-urlencode 'where={"status":{"$in":["in_progress","in_review"]}}' \
  --data-urlencode 'order=-created_at' \
  https://api.textmaster.com/v1/clients/projects/filter \
  -H "Authorization: Bearer ACCESS_TOKEN"
```

Supported operators: `$gt` `$gte` `$lt` `$lte` `$in` `$nin` `$ne` `$or` `$regex`. An unsupported
selector returns 422.

Retrieving finished content is covered in
`skills/textmaster-collect-completed-translations.md`.

## Mid-flight controls

All take `project:write` / `project:manage`:

- `PUT /v1/clients/projects/{project_id}/pause` / `.../resume`
- `PUT /v1/clients/projects/{project_id}/cancel`
- `PUT /v1/clients/projects/{project_id}/archive` / `.../unarchive`
- `POST /v1/clients/projects/{project_id}/duplicate` — **hazardous without an idempotency key**;
  guard it with your own `external_id` check.
- `PUT /v1/clients/projects/{project_id}/activate_tm_options` — turns on translation-memory options,
  which re-prices the project.

## Test it first

Point `baseUrl` at `https://api.textmasterstaging.com` — a full parallel environment that "behaves
the same way as the production environment." It needs its own account and its own OAuth app; no
shared test credentials are published, and there are no magic test values. See
`sandbox/textmaster-sandbox.yml`.

Careful with the interactive Swagger console at `https://app.textmaster.com/api-docs/index.html`:
it loads the production spec, whose only server is `https://api.textmaster.com`. "Try it out" on
the launch operation fires at **production**.

## Error handling

- 422 → read the field-keyed `errors` object: `{"errors":{"author_id":["must be filled"]}}`.
- 403 → missing scope, not a bad token.
- Quote `X-Request-Id` from the response headers when contacting support@textmaster.com.
- Full catalogue: `errors/textmaster-problem-types.yml`.
