---
name: Run ESGx energy and carbon estimates for a set of buildings
description: Submit buildings to the Measurabl ESGx Buildings (Insights) API, poll the asynchronous estimate job to JOB_SUCCESS, read absolute and intensity estimates, and export a batch to CSV via a pre-signed download.
api: openapi/measurabl-esgx-buildings-openapi.yml
operations:
  - POST /insights/v0/energy_estimates
  - GET /insights/v0/energy_estimates/{id}
  - GET /insights/v0/energy_estimates
  - POST /insights/v0/carbon_estimates
  - GET /insights/v0/carbon_estimates/{id}
  - POST /insights/v0/energy_estimate_batches/upload
  - GET /insights/v0/energy_estimate_batches
  - GET /insights/v0/energy_estimate_batches/{id}
  - GET /insights/v0/energy_estimate_batches/{id}/energy_estimates
  - POST /insights/v0/energy_estimate_batches/export
  - GET /insights/v0/exports/{id}
  - GET /insights/v0/exports/{id}/download
---

# Run ESGx energy and carbon estimates for a set of buildings

> Operations are referenced by **METHOD + path** because the published Measurabl OpenAPI documents
> declare **no `operationId`**. Every method and path below was verified verbatim in
> `openapi/_original/measurabl-esgx-buildings-openapi.json`.

This is the **create-then-poll** flow. Estimates are asynchronous jobs, not synchronous responses —
the ESGx Buildings Quick Start spells this out and the API returns a record whose `status` you must
poll until it reads `JOB_SUCCESS`.

## 1. Authenticate

Same OAuth 2.0 client-credentials token as every other Measurabl surface:

```
POST https://api.measurabl.com/token
```

Send `Authorization: Bearer <token>`. Base path for this API is `/insights/v0` on
`https://api.measurabl.com`. Note the published spec declares **no `servers` array**, so the base
URL cannot be resolved from the document alone — `overlays/measurabl-esgx-buildings-overlay.yaml`
records it.

## 2. Submit the buildings

```
POST /insights/v0/energy_estimates
```

Buildings can be identified three ways, each with its own request-body schema in the spec:

| Schema | Use when |
|---|---|
| `request_body_msr_building_id` | The building already exists in Measurabl |
| `request_body_building_custom_id` | You key on your own building identifier |
| `request_body_coordinates_with_client_metadata` | You only have lat/long |
| `request_body_building_info_with_client_metadata` | You have address/characteristics |

`timePeriod` is **required** on the estimate operations and is a closed enum:
`T-12`, `2025`, `2024`, `2023`, `2022`, `2021`.

Carbon estimates are the identical shape at `POST /insights/v0/carbon_estimates`.

**Watch the payload size.** Oversized submissions are rejected with **`413`** — the spec's own
description is "too many estimates in request". The numeric cap is not published, so split large
sets and back off on `413` rather than guessing.

## 3. Poll to JOB_SUCCESS

```
GET /insights/v0/energy_estimates/{id}
```

Take the `id` from the first estimate record and poll until `status` reads `JOB_SUCCESS`. Results
then appear in the **`absoluteEstimates`** and **`intensityEstimates`** fields.

Poll on a backoff. `GET` is capped at 1000 requests per endpoint per 5 minutes, which is generous,
but a tight poll loop across many estimate ids shares that single endpoint budget.

You can also filter the collection server-side:

```
GET /insights/v0/energy_estimates?status=JOB_SUCCESS
```

`status` is an enum whose only declared value is `JOB_SUCCESS`.

## 4. For volume, use batches

```
POST /insights/v0/energy_estimate_batches/upload      # CSV upload creates the batch
GET  /insights/v0/energy_estimate_batches             # list batches
GET  /insights/v0/energy_estimate_batches/{id}        # poll the batch
GET  /insights/v0/energy_estimate_batches/{id}/energy_estimates   # read the results
```

The same five-operation shape repeats for every batch family — swap `energy_estimate_batches` for
`carbon_estimate_batches`, `certification_lookup_batches` or `ordinance_lookup_batches`.

**CRREM is the exception.** The six `/insights/v0/crrem_lookups*` and
`/insights/v0/crrem_lookup_batches*` paths are published as **empty path items with no HTTP
operations declared at all**. The CRREM transition-risk capability is visible in the spec but not
callable from it. Do not attempt to construct those calls — ask Measurabl support for the contract.

## 5. Export and download

```
POST /insights/v0/energy_estimate_batches/export   # creates an Export
GET  /insights/v0/exports/{id}                     # 200 "Export is not ready yet" while generating
GET  /insights/v0/exports/{id}/download            # 302 -> pre-signed URL
```

Two things to handle:

- `GET /insights/v0/exports/{id}` returns **`200` with "Export is not ready yet"** — a `200` does
  **not** mean ready. Poll until the export reports complete before downloading.
- `GET /insights/v0/exports/{id}/download` answers **`302`** with a `Location` header pointing at a
  **pre-signed URL**. Follow the redirect, and do **not** send your `Authorization` header to the
  pre-signed host — the signature is the credential there.
- Exporting a batch whose records span different time periods fails with **`422`** ("Export record
  has batch records with different time periods"). Normalize `timePeriod` across the batch first.

## Error handling

JSON:API error objects in `application/vnd.api+json` — not RFC 9457. Beyond the shared statuses,
this API adds:

| Status | Meaning here |
|---|---|
| `413` | Too many estimates / certification lookups / ordinance lookups in one request |
| `422` | Batch records span different time periods — cannot export |
| `403` | Authenticated but not entitled to this dataset |

Full catalog: `errors/measurabl-problem-types.yml`.

## Related artifacts

- `conventions/measurabl-conventions.yml` — the async create-then-poll and redirect-download patterns
- `data-model/measurabl-data-model.yml` — batch → result records → export
- `rate-limits/measurabl-rate-limits.yml` — per-endpoint 5-minute windows
