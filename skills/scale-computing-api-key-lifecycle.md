---
name: scale-computing-api-key-lifecycle
description: Create, inventory and revoke SC//Fleet Manager API keys. Contains the one irreversible operation on this API — read the warning before revoking anything.
api: SC//Fleet Manager API
base_url: https://api.scalecomputing.com
operations:
  - ApiKeyController_findAll
  - ApiKeyController_create
  - ApiKeyController_remove
generated: '2026-08-26'
method: generated
source: openapi/scale-computing-fleet-manager-openapi.json
---

# API key lifecycle

## Authenticate

`api-key: <FLEET_MANAGER_API_KEY>`, with a role permitted to manage keys. Keys can also be created
interactively at `https://fleet.scalecomputing.com/organization/settings`.

## Inventory

`GET /api/v2/api-keys` (`ApiKeyController_findAll`), paginated with `offset`/`limit`.
Each `ApiKeyDto` gives `id`, `name`, `organizationId`, `tokenSuffix`, `role`, `createdAt` and
`lastUsed`. `tokenSuffix` is how you match a key to a caller **without** ever handling the secret;
`lastUsed` is how you tell a live integration from an abandoned one.

## Create

`POST /api/v2/api-keys` (`ApiKeyController_create`) with a `CreateApiKeyDto`:

```json
{ "name": "<descriptive name>", "role": "<role>", "isSystemToken": false }
```

- Success is HTTP **201**.
- **The secret is returned exactly once.** Treat the response body as a credential: never log it, never
  echo it into a transcript, hand it straight to the secret store.
- Roles are not enumerated in the spec. The spec's own example payload names a `VM Viewer` role, and
  the authoritative list is in the Fleet Manager User Guide. Ask for the role by name rather than
  guessing, and prefer the least-privileged role that satisfies the caller.
- **Not idempotent.** There is no idempotency key. A retried create mints a *second* key. If a create
  times out ambiguously, call `GET /api/v2/api-keys` and check for a key with your `name` before
  retrying.

## Revoke — irreversible

`DELETE /api/v2/api-keys/{id}` (`ApiKeyController_remove`).

**This cannot be undone.** There is no restore or undelete operation, no soft-delete, and no grace
period published anywhere. The secret was only ever shown at creation, so the key cannot be recreated
with the same value — recovery means minting a new key and re-credentialing every consumer of the old
one. Every integration still using that key breaks the moment the call succeeds.

Before revoking:

1. Confirm the `id` against `tokenSuffix` and `name` from the inventory — never revoke by position in
   a list.
2. Check `lastUsed`. A recently-used key has a live consumer you have not identified yet.
3. Get an explicit human confirmation naming the key. This is the one operation on this API where a
   wrong call has no remedy.

## Audit

`GET /api/v2/organization-activities` (`OrganizationActivityController_findAll`) records user and
system events for the organization; use it to confirm a key creation or revocation landed and to see
who did it (`createdByName`, `createdById`).
