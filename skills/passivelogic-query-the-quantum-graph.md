---
name: Query the Quantum digital twin graph
description: Run GraphQL queries against the PassiveLogic Quantum ontology — buildings, floors, zones, surfaces, equipment, components and properties — including how to discover the schema when introspection is gated.
api: openapi/passivelogic-rest-api-openapi.yml
base_url: https://passivelogic.com/api
operations:
  - postApiGraphql
  - getApiGraphql
  - getApiGraphqlSubscribe
  - getApiUtilQuantumversion
generated: '2026-08-04'
method: generated
source: openapi/passivelogic-rest-api-openapi.yml, https://quantumalliance.org/documentation/
---

# Query the Quantum digital twin graph

The REST API is the perimeter; GraphQL is the product. Everything about a building — its floors, zones, surfaces,
equipment, components, properties, behaviours and time series — lives in the Quantum graph and is reachable only
through `/api/graphql`.

## Step 1 — pin the schema version

`getApiUtilQuantumversion` — `GET /api/util/quantumversion`. Anonymous. Returns the Quantum ontology schema version
as **bare text**, not JSON (observed `0.28.0`). Do not `JSON.parse()` it.

Record this alongside any query you cache or any result you store. It moves independently of the HTTP API version
train (`/api`, `/api/v0.19`, `/api/v0.20`).

## Step 2 — send the query

`postApiGraphql` — `POST /api/graphql`, body `GraphQLRequest`, response `GraphQLResult`. This is the form to use.

`getApiGraphql` — `GET /api/graphql` also exists, taking the query as a `query` query-string parameter plus
`operationName` / `variables`. Use it only for short reads; long documents will hit URL limits and end up in logs.

Authenticate with `PL-API-KEY: <key>` (see the *Authenticate and issue an API key* skill) or `X-PL-AUTH: <jwt>`.

Anonymous calls return `401 {"error":true,"reason":"Could not find hive or user."}` — the wording is about tenancy
resolution, not a malformed request. If you see it, your credential is missing or is not bound to a hive/site.

## Step 3 — discovering the schema

Introspection works, but only for an authenticated caller. Anonymous `{__schema{queryType{name}}}` is refused.

Three ways in, in order of preference:

1. **Introspect with your own credential** once you have one — the standard GraphQL introspection document works.
2. **Quantum Insights** (`https://quantumalliance.org/insights/`) — the query builder has an embedded GraphiQL
   interface and interactive documentation of the type system, plus three public demonstration buildings to query
   against. This is the fastest way for a human to hand you the exact field names.
3. **`vocabulary/passivelogic-quantum-object-types.yml`** in this repository — the 89 Quantum node types transcribed
   from the published OpenAPI (`Building`, `Floor`, `Zone`, `Surface`, `Equipment`, `EquipmentComponent`, `Property`,
   `Quanta`, `Behavior`, `System`, `Subsystem`, `ConnectionNode`, …). This gives you the nouns, not the fields.

**Never guess a field name.** The type system is not published anonymously, so a query written from memory will
fail. Introspect or ask.

## Step 4 — the traversal shape

The Quantum documentation models a query as a **From → To** traversal with optional chaining and a Where filter.
The published worked examples are:

| Question | Traversal | Filter |
|---|---|---|
| How many floors per building? | `Building → Floor` | — |
| How many zones per building? | `Building → Zone` | — |
| Which zones have unlocked properties? | `Zone → Property` | `Property.Locked = false` |
| Where are the sites? | `Site` | — |
| Surfaces per zone, by floor | `Building → Floor → Zone → Surface` | chained |

Start from `Site` or `Building` and walk down. Quantum is a physics-based ontology: a component's identity is its
actor type, quanta type, behaviours and properties — so if you are looking for "the pump", you are looking for an
`Equipment` with a transport actor role, not for a string named "pump".

## Step 5 — live data

`getApiGraphqlSubscribe` — `GET /api/graphqlSubscribe` upgrades to a WebSocket for GraphQL subscriptions
(`graphql-ws` / `graphql-transport-ws`). `/api/quantumsync` carries Quantum graph replication. Neither has a
published message schema — see `asyncapi/passivelogic-quantum-events.yml`. Do not call `/api/datasync`; it is the
deprecated legacy route.

## Rules

- No pagination exists in REST and none is documented for GraphQL — **ask for exactly the fields you need**. On a
  454,000-square-foot warehouse an unbounded traversal is a very large document.
- No documented rate limits, no `Retry-After`. Back off exponentially on your own initiative.
- Mutations are not idempotent and there is no idempotency key. Re-read before you re-send.
- GraphQL failures come back as a standard `errors[]` array (`message`, `locations`, `path`) inside a `200`, not as
  the REST `{"error":true,"reason":...}` envelope. Check `errors` even on a 200.
