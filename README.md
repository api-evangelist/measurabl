# Measurabl

ESG data and sustainability management platform for commercial real estate — utility and waste data
collection, carbon accounting, green building certifications and ratings, local performance
ordinance compliance, CRREM transition risk, and portfolio/fund-level reporting.

- Website: https://www.measurabl.com/
- API docs: https://api.measurabl.com/api-docs/
- Release notes: https://api.measurabl.com/release_notes
- Help center: https://support.measurabl.com/hc/en-us

## API surface

Five OpenAPI 3.0.1 documents, 110 operations, all JSON:API (`application/vnd.api+json`) over
OAuth 2.0 client credentials at `https://api.measurabl.com/token`.

| API | Base path | Ops |
|---|---|---|
| Core | `/core/v0` | 57 |
| ESGx Buildings (Insights) | `/insights/v0` | 39 |
| ESGx Securities | `/insights/v0` | 8 |
| ESGx Securities Compliance Files | `/insights/v0` | 3 |
| Partner | `/partners/v0` | 3 |

API access is not self-serve: it requires Premium Tier entitlement and credentials issued by a
Measurabl Customer Delivery Manager. There is no sandbox.

See `llms/measurabl-llms.txt` for the full machine-readable summary, including known gaps.
