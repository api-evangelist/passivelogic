---
name: Manage PassiveLogic organizations and auth-group access
description: Invite people to an organization or to a specific site/view, accept or decline invitations, and remove members — the tenancy and access-control surface of the PassiveLogic platform.
api: openapi/passivelogic-rest-api-openapi.yml
base_url: https://passivelogic.com/api
operations:
  - postApiOrganizationInvite
  - postApiOrganizationJoin
  - postApiOrganizationDecline
  - postApiOrganizationDelete
  - postApiAuthgroupInvite
  - getApiAuthgroupJoin
  - postApiAuthgroupDecline
  - postApiAuthgroupRemoveByAuthGroupIDByUserID
  - getApiAuthWhoami
generated: '2026-08-04'
method: generated
source: openapi/passivelogic-rest-api-openapi.yml
---

# Manage PassiveLogic organizations and auth-group access

PassiveLogic has two nested tenancy concepts and they are easy to confuse:

- **Organization** — the account/company boundary. Membership is company-wide.
- **AuthGroup** — access to a specific **Site or View** inside an organization. The spec says this explicitly:
  an auth-group invite generates "an invitation link for the given Site or View (AuthGroup)".

Inviting someone to an organization does not give them a site. Granting a site does not make them a member of the
organization. Decide which one the human actually meant before you call anything.

## Confirm who you are first

`getApiAuthWhoami` — `GET /api/auth/whoami`. Returns `userID`. Every operation below acts as that user, and an API
key inherits its issuer's permissions, so check before you act.

## Organization membership

| Operation | Call | Body / params |
|---|---|---|
| `postApiOrganizationInvite` | `POST /api/organization/invite` | `InviteUserToOrganizationData` — sends an invitation link to an email address |
| `postApiOrganizationJoin` | `POST /api/organization/join` | `AcceptOrganizationInvite` — adds a user to an organization |
| `postApiOrganizationDecline` | `POST /api/organization/decline` | `DeclineOrganizationInvite` |
| `postApiOrganizationUpdate-image` | `POST /api/organization/update-image` | `OrganizationImageData` |
| `postApiOrganizationUpdate-logo` | `POST /api/organization/update-logo` | `OrganizationImageData` |

`postApiOrganizationInvite` returns `OrgInviteResponse`.

## Site / view access

| Operation | Call | Notes |
|---|---|---|
| `postApiAuthgroupInvite` | `POST /api/authgroup/invite` | `InviteUserToAuthGroupData`; returns `AuthGroupInviteResponse` |
| `getApiAuthgroupJoin` | `GET /api/authgroup/join` | Accepts an invitation. **Use the GET form** — `postApiAuthgroupJoin` is deprecated |
| `postApiAuthgroupDecline` | `POST /api/authgroup/decline` | |
| `postApiAuthgroupRemoveByAuthGroupIDByUserID` | `POST /api/authgroup/remove/{authGroupID}/{userID}` | Both path parameters are uuids; returns `AuthGroupRemovalResponse` |

## Destructive operations — stop and ask

These delete real tenant data and have no undo, no confirmation step, and no idempotency key:

- `postApiOrganizationDelete` — `POST /api/organization/delete`, body `DeleteOrganizationData` (`orgID` uuid).
  **Deletes an entire organization.** Never call this without an explicit, in-the-moment human instruction naming
  the organization.
- `deleteApiAccountUserDelete` — `DELETE /api/account/user/delete`. Deletes the calling user.
- `postApiAuthgroupRemoveByAuthGroupIDByUserID` — revokes a person's access to a site. Confirm the `userID` maps to
  the person the human named before calling; the API will not tell you afterwards.

## Retry discipline

There is no `Idempotency-Key` on any of these routes. If an invite call times out, **query state or ask a human** —
do not resend. A retried invite sends a second email; a retried remove is harmless but a retried delete is not
recoverable.

## Errors

`{"error": true, "reason": "<text>"}`. The spec declares only `200` responses, so branch on HTTP status and treat
`reason` as prose. See `errors/passivelogic-problem-types.yml`.
