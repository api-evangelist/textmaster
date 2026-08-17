---
name: Subscribe to TextMaster project and document events
description: >-
  Register webhook callbacks at account, project or document level, and handle the 19 documented
  events correctly given at-least-once, unordered delivery, a 30-second receiver budget and no HMAC
  signature header.
api: openapi/textmaster-api-v1-openapi.yml
operations:
  - 'GET /v1/clients/users/me'
  - 'PUT /v1/clients/users/{user_id}'
  - 'POST /v1/clients/projects'
  - 'POST /v1/clients/projects/{project_id}/documents'
scopes:
  - 'user:write'
  - 'project:write'
generated: '2026-08-17'
method: generated
source: >-
  https://developer.textmaster.com/webhooks-and-events/webhooks +
  https://developer.textmaster.com/webhooks-and-events/events +
  https://developer.textmaster.com/webhooks-and-events/webhooks/creating-webhooks +
  https://developer.textmaster.com/webhooks-and-events/webhooks/securing-webhooks +
  https://developer.textmaster.com/webhooks-and-events/webhooks/troubleshooting-webhooks +
  openapi/textmaster-api-v1-openapi.yml
---

# Subscribe to TextMaster project and document events

Webhooks are how you learn that work progressed. The provider is unambiguous about this:

> "Always prefer using webhooks over HTTP polling for reliability."

The reason is payload size — pulling finished documents over the API risks the 30-second timeout,
whereas webhooks push the content notification to you.

## There is no webhook resource

This is the thing to internalise before writing code. TextMaster has **no** `/webhooks` endpoint.
You subscribe by writing a `callback` object onto an existing resource, keyed by event name:

```json
{ "callback": { "<event_name>": { "url": "https://..." } } }
```

Three scopes of subscription:

| Level | Operation | Scopes |
|---|---|---|
| Account (global) | `PUT /v1/clients/users/{user_id}` | `user:write` / `user:manage` |
| Project | `POST /v1/clients/projects` (or `PUT /v1/clients/projects/{project_id}`) | `project:write` / `project:manage` |
| Document | `POST /v1/clients/projects/{project_id}/documents`, the batch create, or `PUT .../documents/{document_id}` | `project:write` / `project:manage` |

Because these are ordinary schema fields rather than an OpenAPI `webhooks` block, a client generated
from the spec will give you a settable field and no hint that the keys are a closed vocabulary. The
vocabulary is below.

## Step 1 — get your user id

`GET /v1/clients/users/me` (scope `user:read` / `user:manage`) → the `id` you need for the path of
`PUT /v1/clients/users/{user_id}`. There is no `/users/me` write alias.

## Step 2 — register a global callback

```shell
curl "https://api.textmaster.com/v1/clients/users/USER_ID" \
  -X PUT \
  -H "Authorization: Bearer ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
        "user": {
          "callback": {
            "word_count_finished": {
              "url": "https://example.com/payload?token=REPLACE_WITH_YOUR_SECRET"
            }
          }
        }
      }'
```

For local development the provider's own tutorial uses ngrok (`./ngrok http 4567`) and registers the
resulting `*.ngrok.io` URL.

## The 19 events

### Project events (7)

| Event | Meaning |
|---|---|
| `project_in_progress` | Launched and available to be claimed by an author. **Also the launch-success signal** — retry a launch only if this has not arrived in ~30 minutes. |
| `project_finalized` | All documents attached and TM/PEMT analysis completed successfully. |
| `project_not_launched` | An `auto_launch` project could not launch — "often due to the client account not having enough credits on its wallet." **Your insufficient-funds alarm.** |
| `project_canceled` | Project canceled. |
| `project_tm_completed` | TM analysis done and **the project cost has been updated**. |
| `project_tm_diff_completed` | "Force Exact Matches" analysis done and **the cost has been updated**. |
| `project_in_review` | All document work submitted; ready for QC or client review. |

⚠️ The spec accepts **both** `project_canceled` and `project_cancelled` as callback keys; the events
documentation uses the single-l form. Tolerate both on receipt.

### Document status-change events (10)

`waiting_assignment`, `in_progress`, `in_review`, `incomplete`, `completed`, `paused`, `canceled`,
`quality_control`, `copyscape`, `counting_words`

### Additional document events (2)

| Event | Meaning |
|---|---|
| `word_count_finished` | Word counting on a document completed. |
| `support_message_created` | A new message on a document's support thread — pair with `GET /v1/clients/projects/{project_id}/documents/{document_id}/support_messages`. |

Three further keys (`in_creation`, `in_extra_review`, `complete`) appear in the spec's callback
schemas but are **not** in the documented event tables. Do not depend on them firing. In particular
`complete` and `completed` both appear as accepted keys; only `completed` is documented.

Subscribe only to the events you handle — the provider notes this "limits the number of HTTP
requests to your server."

## Step 3 — secure the receiver

**There is no signature header today.** TextMaster states an intent — "In the future, TextMaster will
use your secret token to create a hash signature of each payload" — but as of now you cannot
cryptographically verify a delivery.

The documented pattern is a shared secret carried in the callback URL you register:

```shell
ruby -rsecurerandom -e 'puts SecureRandom.hex(20)'
```

…then register `https://example.com/payload?token=<that value>` and compare on receipt. Use a
**different token per user** of your service, so one leak does not compromise the rest.

Be honest with yourself about the strength of this: a query parameter is logged by proxies and web
servers by default, and it authenticates the endpoint rather than the payload. Serve the receiver
over HTTPS (TextMaster verifies certificates and logs SSL errors in the delivery response) and treat
payload contents as untrusted input regardless.

An IP allowlist is possible but **the provider recommends against it** ("our public IP addresses are
subject to changes"). If you use one anyway, the published addresses are:

- Production: `104.155.57.91`, `104.155.91.236`, `35.205.172.93`, `34.140.71.130`
- Sandbox: `34.76.154.26`, `34.76.94.225`, `34.76.144.86`, `35.241.160.58`

## Step 4 — build the receiver correctly

Four delivery properties dictate the design:

1. **At-least-once.** "We guarantee delivery of webhook at least once but webhooks can be delivered
   more than once to your server." Your handler **must be idempotent** — dedupe on the resource id
   plus event name plus status.
2. **Unordered.** "We also cannot guarantee the order in which webhooks are delivered. Your server
   should handle receiving events out of order." Do not build a state machine that assumes
   `waiting_assignment` arrives before `in_progress`. Re-read the resource instead of inferring
   state from arrival order.
3. **30-second budget.** The connection stays open at most 30 seconds. Acknowledge immediately and
   do the real work in a background job (the docs name Sidekiq, RQ and RabbitMQ). "Note that even
   with a background job running, TextMaster still expects your server to respond within 30
   seconds."
4. **20 retries with exponential backoff.** Failures are retried up to 20 times, each attempt
   appearing as a new delivery in the log. A flapping receiver will be hammered for a long time.

Branch on the **`X-TextMaster-Event`** request header rather than parsing the body to work out what
arrived. The provider's guidance: "Ensure that your application explicitly checks the type and action
of an event before doing any webhook processing", because new event types get added over time.

## Step 5 — respond usefully

Status codes carry meaning back to the user, who can inspect your responses in TextMaster's delivery
log:

- `200` — processed.
- `201` / `202` — received but not processed (e.g. a payload for a stale project). Use these
  deliberately; the docs suggest exactly this.
- `500` — reserve for catastrophic failure. It triggers the retry budget.

Put a clear, informative message in the response body. Users read it, and "we will sometime populate
the response body with error messages on un-expected exceptions received from your server."

## Step 6 — debug from the delivery log

The application shows recent deliveries with the full outbound request (headers and JSON payload)
and your server's response (status, headers, body), per attempt. This is the only observability
surface for the event pipeline; the retention period is not published.

## What does not exist

- **No AsyncAPI document.** `api.textmaster.com/asyncapi.yaml` → 404, and the developer portal's
  58-page index has no event-spec page. The full catalogue captured here is
  `asyncapi/textmaster-event-surface.yml`.
- **No event-poll endpoint.** If you do not run a receiver, real-time status is simply unavailable
  to you — your only option is polling the project and document resources.
- **No event replay API.** Redelivery happens only through the retry budget.
