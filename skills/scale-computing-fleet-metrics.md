---
name: scale-computing-fleet-metrics
description: Pull historical CPU, RAM and storage time-series across a Scale Computing fleet and export cluster/VM inventories as CSV. Read-only.
api: SC//Fleet Manager API
base_url: https://api.scalecomputing.com
operations:
  - MetricsController_findAll
  - ClustersController_downloadCsv
  - VmController_downloadCsv
  - ClustersController_findAll
  - VmController_findAll
generated: '2026-08-26'
method: generated
source: openapi/scale-computing-fleet-manager-openapi.json
---

# Fleet metrics and export

## Authenticate

`api-key: <FLEET_MANAGER_API_KEY>`. A read-only role is sufficient.

## Time-series metrics

`GET /api/v2/metrics` (`MetricsController_findAll`).

| Parameter | Required | Meaning |
|---|---|---|
| `metricType` | **yes** | which metric to query |
| `query` | no | filter for the metrics query |
| `from` | no | start of the window |
| `to` | no | end of the window |
| `offset` / `limit` | no | pagination; `limit` defaults to 20 |

The response is a `QueryResponseDto`: `pointData`, `entity`, `fields`, `label`, `colors` — a chart-
shaped projection rather than a resource collection.

**Two honest gaps to work around.** The spec does not enumerate the valid `metricType` values, and it
does not declare a format for `from`/`to` (the description only hints "if not using an specific
date/..."). Do not guess a vocabulary. Issue one probe call with a single candidate `metricType` and
read the 400 body, or read the value off the Fleet Manager UI, before running a bulk pull.

## Bulk inventory export

- `GET /api/v2/clusters/download` (`ClustersController_downloadCsv`) → `text/csv` of clusters.
- `GET /api/v2/vms/download` (`VmController_downloadCsv`) → `text/csv` of virtual machines.

These are the fastest way to snapshot the whole fleet — neither takes pagination parameters, so they
return the full set in one response. Use them for inventory and reconciliation; use the paginated
`GET /api/v2/clusters` and `GET /api/v2/vms` when you need the JSON field set
(`ClusterDto` / `VmDto`) rather than the CSV columns.

## Conventions

- Responses to the JSON collections are `{ meta: {offset, limit, total}, items: [...] }`.
- Only HTTP 400 is declared; an unauthenticated call returns 500, not 401.
- No rate limits are documented and no rate-limit headers are returned — pace bulk pulls yourself, and
  prefer the CSV endpoints over deep pagination when you need everything.
