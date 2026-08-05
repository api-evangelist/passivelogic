---
name: Authenticate to PassiveLogic and issue an API key
description: Get a working credential for the PassiveLogic / Quantum API — confirm who you are, mint a scoped, expiring PL API key for unattended use, and revoke it when you are done.
api: openapi/passivelogic-rest-api-openapi.yml
base_url: https://passivelogic.com/api
operations:
  - getApiAuthWhoami
  - getApiAuthApi-keyGenerate
  - deleteApiAuthApi-key
  - postApiAuthLogout
  - getApiUtilExternalauthconfig
generated: '2026-08-04'
method: generated
source: openapi/passivelogic-rest-api-openapi.yml, https://quantumalliance.org/documentation/
---

# Authenticate to PassiveLogic and issue an API key

PassiveLogic has no anonymous data access. Every Quantum query needs either a session JWT or a PL API key, and the
key is what you want for anything unattended. Interactive login is not an API call — it redirects to an external
Keycloak identity provider — so this skill assumes you already have a browser session, or that a human has given
you a key.

## Before you start

- Base URL: `https://passivelogic.com/api` (an on-premises Hive controller serves the identical surface on its own host).
- There is no test mode and no sandbox key prefix. Anything you do here happens on the real account. See
  `sandbox/passivelogic-sandbox.yml` for the public demo building datasets, which are the safe things to query.
- **No operation in this API is idempotent.** There is no `Idempotency-Key` header. Never blind-retry a POST or
  DELETE; re-read state first.

## Step 1 — find out where identity lives

`getApiUtilExternalauthconfig` — `GET /api/util/externalauthconfig`. Anonymous. Returns the account-management URI
and the full OpenID Connect configuration of the external identity provider (observed:
`https://login.passivelogic.com/realms/prod`). Use this rather than hard-coding the issuer; PassiveLogic can move it.

## Step 2 — confirm you are authenticated

`getApiAuthWhoami` — `GET /api/auth/whoami`, with one of:

- `PL-API-KEY: <key>`
- `X-PL-AUTH: <jwt>`

Returns `WhoAmIResponse`: `userID` (uuid, required), `firstName`, `lastName`, `emailVerified`, `eulaAccepted`.

**Read the response body, not just the status.** Unauthenticated callers get `404 {"error":true,"reason":"Not Found"}`
here, not a `401`. A 404 from this route means "not signed in", not "route missing".

If `eulaAccepted` is false, the account has not accepted the EULA and other calls may fail. Hand that back to a
human — `postApiAccountEulaAccept` is flagged deprecated in the spec and should not be called by an agent.

## Step 3 — mint a key

`getApiAuthApi-keyGenerate` — `GET /api/auth/api-key/generate`

| Query parameter | Type | Notes |
|---|---|---|
| `expireSeconds` | integer (int64), optional | Seconds until expiry. The spec's own example is `2678400` (31 days). Omit and you get the default period — set it explicitly. |
| `role` | string, optional | Role of the new key. Valid values come from the `QuantumRole` enum: `SuperUser`, `PrivilegedUser`, `StandardUser`, `ReadOnlyUser`. The spec's example is `ReadOnlyUser`. |

Returns `GenerateApiKeyResponse`: `key` (string) and `expiration` (int64).

Two rules:

1. **Ask for `ReadOnlyUser` unless you have a written reason not to.** A key inherits the permissions of the user
   who issued it, so a key minted by an admin is an admin key by default. `role` is the only lever you have.
2. **Always pass `expireSeconds`.** The key is a bearer credential with no other scoping.

Send the key as `PL-API-KEY: <key>`. The spec's description text still shows the older
`Authorization: PL-API-KEY <key>` form — that scheme is explicitly marked `DEPRECATED - PL API Key`. Use the
`PL-API-KEY` header.

## Step 4 — revoke when finished

`deleteApiAuthApi-key` — `DELETE /api/auth/api-key`, body `RemoveApiKeyRequest`. Validates the key belongs to the
authenticated user and removes it.

Do not use `deleteApiAuthApi-keyRemove` or `postApiAuthApi-keyRemove` (`/api/auth/api-key/remove`) — both are
deprecated.

`postApiAuthLogout` — `POST /api/auth/logout` ends a session JWT; it does not revoke API keys.

## Errors

Failures return `{"error": true, "reason": "<text>"}` with a human string and no machine-readable code. The
published spec declares only `200` responses, so switch on the HTTP status and treat `reason` as a log line, not a
branch condition. See `errors/passivelogic-problem-types.yml`.

## Do not call

- `getApiAuthLogin`, `postApiAuthRegister`, `postApiAuthRegisterInitiate`, `postApiAuthPasswordReset`,
  `postApiAuthPasswordChange`, `postApiAuthEmailChange`, `postApiAuthMagic-link-generate` — all deprecated in favour
  of the Keycloak-hosted flows at `/app/login/*`.
- `deleteApiAccountUserDelete` — deletes the calling user. Never call this on a human's behalf without an explicit,
  in-the-moment instruction.
