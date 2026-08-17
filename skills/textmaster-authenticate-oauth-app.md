---
name: Authenticate against the TextMaster API
description: >-
  Register an OAuth App, run the authorization-code flow, request only the scopes you need, and
  verify the resulting token. Also covers the legacy signature strategy, which is acceptable for
  test scripts only.
api: openapi/textmaster-api-v1-openapi.yml
operations:
  - 'GET /ping'
  - 'GET /test'
  - 'GET /v1/clients/users/me'
scopes:
  - 'user:read'
  - 'user:email'
generated: '2026-08-17'
method: generated
source: >-
  https://developer.textmaster.com/overview/authentication +
  https://developer.textmaster.com/apps/about-oauth-apps +
  https://developer.textmaster.com/apps/building-oauth-apps/scopes-for-oauth-apps +
  openapi/textmaster-api-v1-openapi.yml
---

# Authenticate against the TextMaster API

Base URL: `https://api.textmaster.com` — sandbox: `https://api.textmasterstaging.com`.
All requests are HTTPS and all bodies are JSON. Note there are no operationIds in this API's
OpenAPI, so every step below names a METHOD and PATH.

## Choose the right strategy

| Strategy | Use for | Provider's own guidance |
|---|---|---|
| OAuth 2.0 authorization code | everything in production | Recommended |
| Signature (`Apikey` + `Date` + `Signature`) | throwaway test scripts | "TextMaster discourages using the signature strategy to authenticate production applications" |

Do not build an integration on the signature strategy. Signatures expire 5 minutes after creation
and there is no scope model behind them — a signature carries the account's full authority.

## Step 0 — confirm reachability (no credentials needed)

`GET /ping` → `200 {"message":"Textmaster API at your service"}`

Use this as your health check. It is unauthenticated on both production and sandbox.

## Step 1 — register an OAuth App

This is a human step in the web application; it cannot be automated.

1. Sign in at `https://app.textmaster.com/sign_in`.
2. Top-bar navigation → **API & Loop**.
3. **View OAuth Applications** → **create a new OAuth application**.
4. Set a callback URL. For local or CLI testing use `urn:ietf:wg:oauth:2.0:oob`, which is what
   TextMaster's own published Postman environment uses.
5. Copy the `client_id` and `client_secret`.

## Step 2 — request only the scopes you need

There are 22 scopes. Full reference with descriptions: `scopes/textmaster-scopes.yml`.

Request them space-separated, URL-encoded as `%20`:

```
https://app.textmaster.com/oauth/authorize?
  client_id=YOUR_CLIENT_ID&
  redirect_uri=YOUR_CALLBACK&
  response_type=code&
  scope=project:read%20project:quote%20user:read
```

Scope selection rules that matter:

- **Never request `project:launch` unless the integration is meant to spend money.** It "grants
  access to launch projects and debit the client's account." Read-and-quote integrations do not
  need it.
- `project:manage` implicitly includes `project:launch` and `project:quote`. If you want to quote
  but not spend, ask for `project:read project:write project:quote` — not `project:manage`.
- `user:manage` implicitly includes `user:email`. Only ask for `user:email` if you actually need
  the private email address.
- `public` is the default scope if none is provided, and covers the `/v1/public/*` reference data.

## Step 3 — exchange the code

`POST https://api.textmaster.com/oauth/token` with the authorization code, `client_id`,
`client_secret` and the same `redirect_uri` you registered.

Refresh tokens are supported; the refresh URL is the same token endpoint.

Failure modes are RFC 6749 shaped — see `errors/textmaster-problem-types.yml`:

- `invalid_client` — wrong `client_id` / `client_secret`.
- `invalid_grant` — code expired/reused, **or** `redirect_uri` mismatch. Both produce the same
  message, so check the redirect URI first; it is the more common cause.

## Step 4 — read the GRANTED scopes, not the requested ones

The token response carries a `scope` attribute. Users can grant less than you asked for, and can
edit token scopes after the flow completes. Read the granted set, store it, and degrade
functionality explicitly rather than discovering the gap as a 403 later.

## Step 5 — call the API

Send the token in the `Authorization` header — the provider's stated preference:

```shell
curl https://api.textmaster.com/v1/clients/users/me \
  -H "Authorization: Bearer ACCESS-TOKEN"
```

`GET /v1/clients/users/me` (scopes `user:read` / `user:manage`) is the whoami call. Do this first
in any integration: it validates the token **and** returns the `id` you need for
`PUT /v1/clients/users/{user_id}`, which is how webhooks are registered
(see `skills/textmaster-subscribe-to-project-events.md`).

## Interpreting auth failures

Both 401 and 403 return the same body — `{"error":"invalid_token","error_description":"The access
token is invalid"}` — so branch on the STATUS, not the payload:

- **401** — the token itself is bad: missing, malformed, expired or revoked. Refresh it.
- **403** — the token is fine but lacks the scope this operation requires. Look up the operation's
  `security[]` block in the spec, or `scopes/textmaster-scopes.yml`, and re-run the authorization
  flow for the missing scope. Refreshing will not help.

## Legacy signature strategy (test only)

```shell
export APIKEY=yourApikey
export APISECRET=yourApiSecret
export DATE=$(date -u +"%Y-%m-%d %H:%M:%S")
export SIGNATURE=$(echo -n $APISECRET$DATE | openssl sha1 | sed 's/.*= //')

curl "https://api.textmaster.com/test" \
  -H "Apikey: $APIKEY" \
  -H "Date: $DATE" \
  -H "Signature: $SIGNATURE"
```

`GET /test` echoes back whether the key is valid, the date well-formatted and the signature
correct — a genuinely useful onboarding check. Note the `Date` header format is
`YYYY-MM-DD HH:MM:SS` (space-separated, no `Z`), which is NOT the ISO 8601 form the rest of the API
uses for timestamps.

## What is not available

- **No OIDC.** There is no `/.well-known/openid-configuration` and no `id_token`. "Login with
  TextMaster" is plain OAuth 2.0 delegation.
- **No OAuth discovery metadata.** RFC 8414 and RFC 9728 are both unimplemented (verified: 404 on
  every host). Hard-code the endpoints from the docs; you cannot discover them.
- **PKCE is undocumented.** The provider's own Postman collection uses plain
  client_id/client_secret, so do not assume a code challenge is required.

## Housekeeping

Delete OAuth Apps you stop using (`Managing OAuth Apps` → `Deleting an OAuth App`). Each live app
is a standing grant against the account.
