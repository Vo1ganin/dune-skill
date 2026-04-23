# Dune Direct HTTP API (for bulk downloads with PAID key)

The MCP server handles most workflows. For bulk downloads exceeding MCP result limits, use the direct HTTP API with the PAID key.

## When to use direct HTTP instead of MCP

- Need > 30k rows of results (MCP/server will clamp large `limit`)
- Need paginated CSV export
- Running an ETL script outside Claude
- Plan-specific enterprise features (queries endpoint on Plus+, data transformations on Enterprise)

## Authentication

```bash
export DUNE_API_KEY_PAID="dkey_..."
```

Header: `X-DUNE-API-KEY: $DUNE_API_KEY_PAID`
Or query param: `?api_key=$DUNE_API_KEY_PAID`

**Always announce in chat before switching to PAID key:**
> "Switching to PAID key. Reason: exporting ~N MB of results."

## Base URL
`https://api.dune.com/api/v1`

## Core endpoints

### Execute query
```http
POST /query/{query_id}/execute
```
Body:
```json
{
  "performance": "medium",
  "query_parameters": { "wallet": "0x...", "days": 30 }
}
```
Response: `{ "execution_id": "01HKZJ...", "state": "QUERY_STATE_PENDING" }`

### Poll execution status
```http
GET /execution/{execution_id}/status
```
States: `QUERY_STATE_PENDING` → `QUERY_STATE_EXECUTING` → `QUERY_STATE_COMPLETED` (or `FAILED`).
Poll every 5–15s. Hard timeout 30 min.

### Get results (JSON)
```http
GET /execution/{execution_id}/results?limit=10000&offset=0
```
Response:
```json
{
  "result": {
    "rows": [...],
    "metadata": {
      "row_count": 10000,
      "total_row_count": 128400,
      "result_set_bytes": 2450000,
      "total_result_set_bytes": 31_400_000,
      "datapoint_count": 80000,
      "next_offset": 10000,
      "next_uri": "https://api.dune.com/..."
    }
  }
}
```

### Get results (CSV) — for big exports
```http
GET /execution/{execution_id}/results/csv?limit=32000&offset=0
```
Pagination via **response headers**:
- `X-Dune-Next-Offset: 32000`
- `X-Dune-Next-Uri: https://api.dune.com/...`

### Get latest query result (cached)
```http
GET /query/{query_id}/results
```
Returns the most recent execution's results — does NOT trigger a new run. Fast, cheap (still charges datapoints for the fetch, but no execution cost).

## Useful query params on results endpoints

| Param | Purpose |
|-------|---------|
| `limit` | Max rows this page (server may clamp) |
| `offset` | Starting row (0-indexed) |
| `columns=a,b,c` | Return only these columns — reduces datapoints billed |
| `filters` | WHERE-like expression |
| `sort_by` | ORDER BY-like expression |
| `sample_count` | Uniform sample instead of full (mutually exclusive with limit/offset/filters) |
| `allow_partial_results=true` | Enable for results > 32 GB |

## Pagination pattern (CSV, Python)

```python
import requests, os

API = "https://api.dune.com/api/v1"
KEY = os.environ["DUNE_API_KEY_PAID"]
headers = {"X-DUNE-API-KEY": KEY}

exec_id = "01HKZJ..."
url = f"{API}/execution/{exec_id}/results/csv?limit=32000&offset=0"
page = 0

while url:
    r = requests.get(url, headers=headers)
    r.raise_for_status()
    with open(f"page_{page}.csv", "wb") as f:
        f.write(r.content)
    # Pagination via headers on CSV endpoint
    next_uri = r.headers.get("X-Dune-Next-Uri")
    url = next_uri  # None when done
    page += 1
```

## Pagination pattern (JSON, Python)

```python
url = f"{API}/execution/{exec_id}/results?limit=10000&offset=0"
all_rows = []
while url:
    r = requests.get(url, headers=headers).json()
    all_rows.extend(r["result"]["rows"])
    url = r["result"]["metadata"].get("next_uri")
```

## Gotchas

- **Invalid `offset` → empty response, no error.** Always check `total_row_count` against accumulated rows to detect gaps.
- **Server may clamp `limit`.** If you pass 500000, server might return 30000 with a `next_offset` of 30000. Trust the server's next-offset.
- **Failed executions are charged.** Validate SQL before `executeQueryById`.
- **32 GB total result ceiling.** Bigger → `allow_partial_results=true` or narrow the query.
- **Datapoints default cap 250,000** per fetch — explicit pagination needed for larger.

## Cost reminder

Every row you download costs `credits_per_MB × size_MB` (see `credits.md`). Before starting a paginated export:
1. Note `total_row_count` and `total_result_set_bytes` from the first page
2. Compute `total_MB × plan_rate` → total export credits
3. If > 700 credits → STOP and report to user before continuing

## SDKs (alternative)

- Python: `pip install dune-client` — has auto-pagination
- TypeScript: `@duneanalytics/client-sdk`

Both handle polling and pagination automatically but give less control over per-page cost.

## Source

- https://docs.dune.com/api-reference/executions/endpoint/execute-query
- https://docs.dune.com/api-reference/executions/endpoint/get-query-result
- https://docs.dune.com/api-reference/executions/endpoint/get-execution-result-csv
- https://docs.dune.com/api-reference/executions/pagination
