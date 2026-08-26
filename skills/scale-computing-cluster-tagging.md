---
name: scale-computing-cluster-tagging
description: Safely read and rewrite the tags on a Scale Computing SC//HyperCore cluster via the SC//Fleet Manager API. The write is an overwrite, so capture the prior value first.
api: SC//Fleet Manager API
base_url: https://api.scalecomputing.com
operations:
  - ClustersController_findOne
  - ClustersController_findAll
  - ClustersController_upsertClusterTags
generated: '2026-08-26'
method: generated
source: openapi/scale-computing-fleet-manager-openapi.json
---

# Cluster tagging

`POST /api/v2/clusters/{id}/tags` is one of only three write operations on this API, and its published
summary says what it does plainly: *"Overwrite the existing cluster tags with the provided..."*. It is
a **replace**, not an append. There is no server-side previous version to restore and no published
retention window, so the only reversal available is the one you prepare yourself.

## Authenticate

`api-key: <FLEET_MANAGER_API_KEY>` on every request. Tag writes need a role with write access; a
viewer role will not do it.

## Steps

1. **Capture the current state — always, before writing.**
   `GET /api/v2/clusters/{id}` (`ClustersController_findOne`) and store `ClusterDto.tags` verbatim.
   This stored value is your entire rollback capability. If you skip it, the write is irreversible.

2. **Compute the new tag set.** Merge, add or remove against the captured value locally. The payload
   shape is a comma-delimited **string**, not an array — the spec's own example is:

   ```json
   { "tags": "foo, bar, baz" }
   ```

   Note the contract defect here: the requestBody `$ref` in the published spec is empty
   (`"#/components/schemas/"`), so the inline example is the authoritative description of the payload.

3. **Write.** `POST /api/v2/clusters/{id}/tags` (`ClustersController_upsertClusterTags`).
   A success is HTTP **201** (not 200).

4. **Verify.** `GET /api/v2/clusters/{id}` again and confirm `tags` matches your intent.

5. **Roll back if wrong.** Re-post the string you captured in step 1.

## Safety rules

- **No idempotency.** This API has no `Idempotency-Key` header and no replay protection. A retry after
  an ambiguous timeout re-applies the whole tag set — which is safe here *only because* the operation
  is a full overwrite rather than an append. Do not generalise that to other operations.
- **No dry run.** There is no validate-only or preview parameter.
- **Batch carefully.** To retag many clusters, walk `GET /api/v2/clusters`
  (`ClustersController_findAll`) with `offset`/`limit`, capture every prior tag string first, then
  write one cluster at a time so a mid-run failure leaves a recoverable record.
- Errors: only HTTP 400 is declared. An unauthenticated call returns 500 rather than 401.
