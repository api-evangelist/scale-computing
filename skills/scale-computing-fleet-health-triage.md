---
name: scale-computing-fleet-health-triage
description: Survey the health of a Scale Computing SC//HyperCore fleet and triage the open cluster conditions, using the SC//Fleet Manager API. Read-only.
api: SC//Fleet Manager API
base_url: https://api.scalecomputing.com
operations:
  - ClustersController_findAll
  - ClustersController_findOne
  - ClusterConditionsController_findAll
  - ClusterConditionsController_findOne
  - VmController_findAll
generated: '2026-08-26'
method: generated
source: openapi/scale-computing-fleet-manager-openapi.json
---

# Fleet health triage

Answer "what is wrong in my fleet right now, and where" without touching any state. Every operation
in this skill is a GET.

## Authenticate

Send an `api-key` header on every request. Keys are created in Fleet Manager at
`https://fleet.scalecomputing.com/organization/settings` and are **role-restricted** — a read-only
role is sufficient for this entire skill, and is what you should ask for.

```
api-key: <FLEET_MANAGER_API_KEY>
```

## Steps

1. **List the fleet.** `GET /api/v2/clusters` (`ClustersController_findAll`).
   Paginate with `offset` and `limit` (`limit` defaults to 20). Read `meta.total` from the `PageDto`
   envelope to know how many pages to walk; the items are in `items`.
   Rank on `healthScore`, `healthState`, `onlineStatus` and `lastCheckin`. A stale `lastCheckin` on an
   otherwise-healthy cluster is the signal that the cluster stopped reporting, not that it is fine.

2. **Pull the open conditions.** `GET /api/v2/conditions` (`ClusterConditionsController_findAll`).
   These are the issues the HyperCore clusters themselves reported. Sort by `severityLevel`
   (numeric) rather than `severity` (label) so ordering is stable, and group by `cluster`.
   Use `setDate` to separate a new fault from one that has been open for weeks.

3. **Expand the worst offenders.** For each cluster at the top of both lists,
   `GET /api/v2/clusters/{id}` (`ClustersController_findOne`) for the full `ClusterDto` — including
   `updatesAvailable`, `updatesAvailableOptions` and `updatesStatus`, which tell you whether the fault
   is one a pending HyperCore update already addresses.
   For a specific condition, `GET /api/v2/conditions/{id}` (`ClusterConditionsController_findOne`).

4. **Correlate to workloads.** `GET /api/v2/vms` (`VmController_findAll`), filter to the affected
   `clusterId`, and report which VMs are exposed: `state`, `ips`, `cores`, `memory`, `drivePercent`.
   `drivePercent` near capacity on a cluster already reporting a storage condition is the pairing
   worth escalating first.

## Conventions that apply

- **Pagination** is offset/limit on every collection; responses are `{ meta: {offset, limit, total}, items: [...] }`.
- **Filtering** is a single opaque `query` string per collection. There is no field-level filter and no
  sort parameter — do the sorting client-side.
- **Errors**: the spec declares HTTP 400 and nothing else. In practice a call with no `api-key` returns
  **HTTP 500**, not 401 (observed 2026-08-26). Do not treat a 500 from this API as retryable until you
  have confirmed the key is present and valid.
- **Rate limits** are undocumented and no rate-limit headers are returned. Pace your pagination
  conservatively and do not parallelise page fetches aggressively.

## Do not

- Do not call `POST /api/v2/clusters/{id}/tags` from this skill. It **overwrites** the cluster's whole
  tag set; see the tagging skill.
- Do not call anything under `/api/v1/` — that is an undocumented control plane with destructive,
  unsummarised operations.
