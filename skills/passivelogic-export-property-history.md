---
name: Export Quantum property history as CSV
description: Pull historical time-series values for Quantum properties out of PassiveLogic as CSV, and know exactly what the export does and does not tell you.
api: openapi/passivelogic-rest-api-openapi.yml
base_url: https://passivelogic.com/api
operations:
  - getApiExportPropertyHistory
  - postApiGraphql
  - getApiUtilQuantumversion
generated: '2026-08-04'
method: generated
source: openapi/passivelogic-rest-api-openapi.yml
---

# Export Quantum property history as CSV

`getApiExportPropertyHistory` — `GET /api/export/property/history` is the one bulk-data route in the PassiveLogic
REST API. It returns time-series values for Quantum properties as CSV text.

## Response shape

`text/plain`, CSV, with these columns exactly as documented in the spec:

```
Time,Equipment name,Property name,Unit,Value
```

`Time` is an ISO 8601 string. Values are stored in SI units in the published demonstration datasets — confirm the
`Unit` column rather than assuming.

## Authentication

`PL-API-KEY: <key>` or `X-PL-AUTH: <jwt>`. A `ReadOnlyUser` key is sufficient and is what you should use.

## The gap you must plan around

The published OpenAPI declares **no parameters** for this operation, while its own description says it returns
history "from the given properties". So the selection mechanism — which properties, which time window — is real but
undocumented in the machine-readable contract.

Do not guess parameter names. Instead:

1. Identify the property nodes you want through GraphQL first (`postApiGraphql`, traversing
   `Site → Building → Floor → Zone → Equipment → EquipmentComponent → Property`), and hold their UUIDs.
2. Ask PassiveLogic support, or read the request the Portfolio web application issues, for the exact query
   parameters. Record what you learn back into this repository rather than into a private script.
3. If you cannot determine the parameters, pull the history through GraphQL instead — the `History` and
   `EventHistory` node types exist in the ontology.

## Before you export

- Call `getApiUtilQuantumversion` and record the schema version with the extract. A CSV without its schema version
  is not reproducible.
- There is no pagination and no documented size cap. Bound your request by time window, not by row count, and
  stream the response rather than buffering it.
- There is no `Retry-After` and no documented rate limit. If an export fails, back off exponentially — do not hammer.

## Related

- `postApiSiteCopy` (`POST /api/site/copy`) copies a whole project between organizations. Note its documented
  behaviour: **history is NOT copied unless `startDate` and `endDate` are provided**, the copy runs in the
  background, and **there is no indication of success**. Treat it as fire-and-forget and verify by querying the
  target organization afterwards. It requires `targetOrganizationID` and `siteID` (both uuid) as query parameters
  and returns an `ObjectIdentifiers` old-ID → new-ID remap.
