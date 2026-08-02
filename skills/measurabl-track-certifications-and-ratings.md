---
name: Track green building certifications and ratings across a portfolio
description: Read the certification types available to a portfolio, create and maintain certifications on buildings, assign them to spaces through the JSON:API relationships resource, and roll certifications and ratings up to portfolio level.
api: openapi/measurabl-core-openapi.yml
operations:
  - GET /core/v0/portfolios/{portfolio_id}/certification_types
  - GET /core/v0/portfolios/{portfolio_id}/certifications
  - GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications
  - POST /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications
  - PATCH /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications/{certification_id}
  - DELETE /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications/{certification_id}
  - PATCH /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications/{certification_id}/relationships
  - GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/spaces/{space_id}/certifications
  - GET /core/v0/portfolios/{portfolio_id}/ratings
  - GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/ratings
  - GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/spaces/{space_id}/ratings
---

# Track green building certifications and ratings across a portfolio

> Operations are referenced by **METHOD + path** — the published Measurabl specs declare no
> `operationId`. All paths verified in `openapi/_original/measurabl-core-openapi.json`.

Certifications are **writable**. Ratings are **read-and-delete only** — there is no `POST` or
`PATCH` for ratings anywhere in the spec, because ratings are produced by Measurabl rather than
supplied by the customer. Plan around that asymmetry.

## 1. Read the certification types available to the portfolio

```
GET /core/v0/portfolios/{portfolio_id}/certification_types
```

This is the reference list scoped to the portfolio. Resolve the type you intend to record here
before creating a certification; do not hard-code type identifiers.

## 2. Create a certification on a building

```
POST /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications
```

JSON:API request document, `Content-Type: application/vnd.api+json`.

**No idempotency.** Re-posting after a timeout creates a duplicate certification. Always
`GET .../certifications` first and re-check after any `408`/`502`/`503` before retrying.

This endpoint is in the "everything else" write class: **100 requests per endpoint per 5 minutes**.

## 3. Maintain it

```
PATCH  /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications/{certification_id}
DELETE /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications/{certification_id}
GET    /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications/{certification_id}
```

`DELETE` answers **`204`** with no body ("Certification Deleted Successfully").

## 4. Assign a certification to a space

```
PATCH /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/certifications/{certification_id}/relationships
```

This is a **JSON:API relationships resource**, not an attribute update: the body is a linkage
document, and the operation assigns the certification to a specified space or list of spaces. It
answers `204` ("Certification Assigned Successfully to the Specified Space").

The same relationships pattern appears twice more in the Core API and behaves identically:

- `PATCH .../funds/{fund_id}/relationships` — assign existing buildings to a fund
- `PATCH .../meters/{meter_id}/relationships` — assign an existing meter to a space

Read the result back from the space side:

```
GET /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/spaces/{space_id}/certifications
```

## 5. Roll up

```
GET /core/v0/portfolios/{portfolio_id}/certifications   # every certification in the portfolio
GET /core/v0/portfolios/{portfolio_id}/ratings          # every rating in the portfolio
```

These portfolio-wide collections are the reporting path — use them instead of fanning out per
building, which burns the per-endpoint GET budget far faster.

Paginate with `page`/`size` (default `25`, **max 200**) and follow `links.next`. A page past the
last page returns **`404`**, not an empty set.

Filter with the RSQL-style `filter` parameter on any attribute the endpoint exposes, e.g.
`?filter=updatedAt=ge=2026-01-01T00:00` to pull only what changed since a checkpoint. This is the
correct incremental-sync pattern for Measurabl — there is no events, webhook or streaming surface
anywhere in the platform, so **polling with an `updatedAt` filter is the only change-detection
mechanism available.**

## 6. Ratings are read-only

```
GET    /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/ratings
GET    /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/ratings/{rating_id}
DELETE /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/ratings/{rating_id}
GET    /core/v0/portfolios/{portfolio_id}/buildings/{building_id}/spaces/{space_id}/ratings
```

There is no create or update operation for ratings. If a rating is wrong, the only API-side action
is `DELETE`.

## Error handling

JSON:API error objects (`application/vnd.api+json`), not RFC 9457. `403` on a portfolio you are not
entitled to is terminal — do not retry. Full catalog: `errors/measurabl-problem-types.yml`.

## Related artifacts

- `data-model/measurabl-data-model.yml` — certification/rating attachment to buildings and spaces
- `conventions/measurabl-conventions.yml` — pagination, RSQL filtering, relationships resources
- `rate-limits/measurabl-rate-limits.yml`
