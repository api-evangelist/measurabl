---
name: Ingest utility meter data into a Measurabl building
description: Authenticate against the Measurabl Core API, locate a portfolio and building, create or find its energy/water meter, and push consumption readings — the primary write path for keeping a building's ESG data current.
api: openapi/measurabl-core-openapi.yml
operations:
  - GET /core/v0/portfolios
  - GET /core/v0/portfolios/{portfolio_id}/buildings
  - GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters
  - POST /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters
  - POST /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters/{meter_id}/readings
  - GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters/{meter_id}/readings
  - GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters/{meter_id}/readings/{meter_reading_id}/bill
---

# Ingest utility meter data into a Measurabl building

> Operations are referenced by **METHOD + path** because the published Measurabl OpenAPI documents
> declare **no `operationId` on any of their 110 operations**. Every method and path below was
> verified verbatim in `openapi/_original/measurabl-core-openapi.json`. Do not invent operationIds.

## Before you start

- **There is no sandbox.** Measurabl publishes no test environment. Every call in this skill hits
  live customer data. Work against a portfolio you are authorized to write to, and prefer reading
  before writing.
- **There is no idempotency contract.** No `Idempotency-Key` header or parameter exists anywhere in
  the Measurabl surface. A retried `POST .../readings` will create a **second** reading. Before
  retrying a write that timed out (`408`) or failed at the gateway (`502`/`503`), re-read the
  collection and check whether the record landed.

## 1. Authenticate

OAuth 2.0 **client credentials** against a single token endpoint:

```
POST https://api.measurabl.com/token
grant_type=client_credentials&client_id=<key>&client_secret=<secret>
```

The key and secret are issued per organization by a Measurabl Customer Delivery Manager and require
Premium Tier entitlement — they cannot be self-served. The flow declares an **empty scopes map**, so
authorization is entitlement-based, not scope-based: a token either can reach a portfolio or gets
`403`. Send the result as `Authorization: Bearer <token>` on every request.

See `authentication/measurabl-authentication.yml` and `scopes/measurabl-scopes.yml`.

## 2. Find the portfolio

```
GET /core/v0/portfolios
```

Every other Core API path is rooted at a `portfolio_id`, so this is always step one — the API's own
Quick Start says so. Responses are **JSON:API** (`application/vnd.api+json`): read ids from
`data[].id`, not from a bare array.

## 3. Find the building

```
GET /core/v0/portfolios/{portfolio_id}/buildings
```

Paginate with `page` (default `1`) and `size` (default `25`, **max 200**). Follow `links.next` and
stop when it is null — requesting a page past the last page returns **`404`**, not an empty
collection.

Narrow with the RSQL-style `filter` parameter, which accepts any attribute the endpoint exposes:

```
?filter=yearbuilt==1969
?filter=updatedAt=ge=2022-11-01T00:00
```

## 4. Find or create the meter

```
GET  /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters
POST /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters
```

Always list first. Creating a duplicate meter silently splits a building's consumption history
across two records, and there is no idempotency key to prevent it. Waste streams are a separate
resource — use `.../waste_meters` and `.../waste_meters/{waste_meter_id}/readings` instead.

## 5. Push the reading

```
POST /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters/{meter_id}/readings
```

The request body is a JSON:API document (`data.type` + `data.attributes`), sent as
`Content-Type: application/vnd.api+json`.

**Rate limits apply per endpoint over a rolling 5-minute window**, and the meter endpoints get the
raised write allowance precisely because they carry bulk ingestion:

| Method class | Endpoint group | Limit per endpoint / 5 min |
|---|---|---|
| GET | any | 1000 |
| POST / PATCH / DELETE | energy & water meters, waste meters | 200 |
| POST / PATCH / DELETE | everything else | 100 |

No `X-RateLimit-*` or `Retry-After` headers are published, so you cannot read your remaining budget
off a response — pace writes to stay under 200 per 5 minutes per endpoint and treat `429` as the
only signal. See `rate-limits/measurabl-rate-limits.yml`.

## 6. Verify, and pull the bill if one exists

```
GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters/{meter_id}/readings
GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/meters/{meter_id}/readings/{meter_reading_id}/bill
```

The bill operation answers **`303`** with a `Location` header pointing at where the utility bill
lives. Follow the redirect; do not expect a body on the `303` itself.

## Error handling

Errors are **JSON:API error objects**, not RFC 9457 `application/problem+json`. The envelope is a
top-level `errors` array whose members carry `id`, `status`, `code`, `title`, `detail` and
`source.pointer`. The `id` member is the only correlation handle Measurabl exposes, and only on
failures — capture it before retrying.

| Status | Meaning here | Do this |
|---|---|---|
| `400` | Malformed JSON:API document or bad `filter` syntax | Fix the document/`filter`; do not retry unchanged |
| `401` | Token expired or invalid | Re-run the client-credentials grant, resend |
| `403` | Authenticated but not entitled to this portfolio/building | Stop — a retry cannot fix entitlement |
| `404` | Bad id, **or a page past the last page** | Check ids; clamp `page` to `links.last` |
| `408` | Server-side timeout | Retry with a smaller `size` or a narrower `filter` |
| `429` | Rate limit exceeded | Back off; the window is 5 minutes |
| `500` / `502` / `503` | Server/gateway error | Backoff retry — **re-read before re-POSTing** |

Full catalog: `errors/measurabl-problem-types.yml`.

## Related artifacts

- `conventions/measurabl-conventions.yml` — pagination, filtering, media type, async patterns
- `data-model/measurabl-data-model.yml` — portfolio → building → meter → reading → bill
- `lifecycle/measurabl-lifecycle.yml` — v0 across every surface, no deprecation policy
