---
name: Pull listed real estate ESG datasets and compliance files
description: Discover ESGx Securities listed real estate reports, enumerate their company-level and building-level data sets, download each via its pre-signed URL, and retrieve the matching compliance files — the capital-markets read path.
api: openapi/measurabl-esgx-securities-openapi.yml
operations:
  - GET /insights/v0/listed_real_estate_reports
  - GET /insights/v0/listed_real_estate_reports/{id}
  - GET /insights/v0/listed_real_estate_reports/{id}/company_level
  - GET /insights/v0/listed_real_estate_reports/{id}/company_level/{data_set}
  - GET /insights/v0/listed_real_estate_reports/{id}/company_level/{data_set}/download
  - GET /insights/v0/listed_real_estate_reports/{id}/building_level
  - GET /insights/v0/listed_real_estate_reports/{id}/building_level/{data_set}
  - GET /insights/v0/listed_real_estate_reports/{id}/building_level/{data_set}/download
  - GET /insights/v0/listed_real_estate/compliance_files
  - GET /insights/v0/listed_real_estate/compliance_files/{id}
  - GET /insights/v0/listed_real_estate/compliance_files/{id}/download
---

# Pull listed real estate ESG datasets and compliance files

> Operations are referenced by **METHOD + path** — the published Measurabl specs declare no
> `operationId`. Paths verified in `openapi/_original/measurabl-esgx-securities-openapi.json` and
> `openapi/_original/measurabl-esgx-securities-compliance-files-openapi.json`.

This flow spans **two** published APIs that share the `/insights/v0` base path: ESGx Securities
(8 operations) and ESGx Securities Compliance Files (3 operations). Both are entirely read-only —
there is not a single write operation between them.

## 1. Authenticate

OAuth 2.0 client credentials at `https://api.measurabl.com/token`; send
`Authorization: Bearer <token>`. Entitlement is per dataset: a token that can read reports may
still get `403` on a specific data set.

## 2. List the reports

```
GET /insights/v0/listed_real_estate_reports
GET /insights/v0/listed_real_estate_reports/{id}
```

JSON:API collection — read ids from `data[].id`. Paginate with `page`/`size` and follow
`links.next`; a page past the end returns `404`.

## 3. Enumerate the data sets at each level

Every report exposes two levels, each with an identical three-operation shape:

```
GET /insights/v0/listed_real_estate_reports/{id}/company_level
GET /insights/v0/listed_real_estate_reports/{id}/building_level
```

These return the **available data sets** for that level. `{data_set}` in the next step is a name
taken from this response — never a guess.

```
GET /insights/v0/listed_real_estate_reports/{id}/company_level/{data_set}
GET /insights/v0/listed_real_estate_reports/{id}/building_level/{data_set}
```

## 4. Download

```
GET /insights/v0/listed_real_estate_reports/{id}/company_level/{data_set}/download
GET /insights/v0/listed_real_estate_reports/{id}/building_level/{data_set}/download
```

Both answer **`302`** with a `Location` header pointing at a **pre-signed URL**
("Export is downloaded via redirect to pre-signed URL").

Two rules for the redirect:

1. **Follow it, but strip your `Authorization` header.** On the pre-signed host the signature *is*
   the credential; sending a bearer token there is unnecessary and leaks it to a third-party origin.
2. **Do not cache the pre-signed URL.** Re-issue the `/download` call each time — pre-signed URLs
   expire.

## 5. Get the compliance files

The companion API, same base path:

```
GET /insights/v0/listed_real_estate/compliance_files
GET /insights/v0/listed_real_estate/compliance_files/{id}
GET /insights/v0/listed_real_estate/compliance_files/{id}/download
```

`{id}` returns metadata; `/download` again answers **`302`** to a pre-signed URL. Note the resource
is documented as "ESGx Securities (fka Listed Real Estate)" — the path still carries the old name
while the product has been renamed. Treat `listed_real_estate` as the stable path segment.

## Incremental sync

There is **no event, webhook or streaming surface** in the Measurabl platform. To detect new
reports or refreshed data sets you must **poll** `GET /insights/v0/listed_real_estate_reports` on a
schedule and diff against what you already hold. `GET` is limited to 1000 requests per endpoint per
5 minutes, which is ample for a daily or hourly poll.

## Error handling

JSON:API error objects in `application/vnd.api+json`. This pair of APIs declares a narrow set:

| Status | Meaning here |
|---|---|
| `403` | Authenticated but not entitled to this report or data set — terminal, do not retry |
| `404` | Unknown report id, level or data set name — re-enumerate from step 3 |
| `302` | **Not an error** — the intended download path |

Full catalog: `errors/measurabl-problem-types.yml`.

## Related artifacts

- `conventions/measurabl-conventions.yml` — redirect-download pattern, pagination
- `data-model/measurabl-data-model.yml` — report → data set → compliance file
- `lifecycle/measurabl-lifecycle.yml` — v0, no deprecation policy, no status page
