# Dune Analytics — Documentation Summary

Собрано из официальных доков на 2026-04-23. Источники указаны в конце.

## Pricing & Credits

### Plan tiers
| Plan | Monthly credits | API rate limit | Storage | Webhooks |
|------|-----------------|----------------|---------|----------|
| Free | 2,500 | 40/min | 100 MB | 1 |
| Analyst | — | — | 1 GB | 5 |
| Plus | — | 200/min | 15 GB | 5 |
| Enterprise | Custom | Custom | Unlimited | Custom |

### Export cost (download results)
**Это ключевое — где кредиты сгорают на скачивании:**
| Plan | Credits per MB exported |
|------|-------------------------|
| Free | 20 |
| Analyst | 10 |
| Plus | 2 |

### Write operations (upload tables)
- 3 credits/GB (все тарифы кроме Enterprise)
- Минимум 1 credit per write operation

### Engine tiers (compute)
| Engine | Timeout | Concurrency | Baseline cost | Use for |
|--------|---------|-------------|---------------|---------|
| Small | 2 min | 3 max | минимум | простые, exploratory |
| Medium (default) | нет явного | unlimited | ~10 credits base | обычные dashboards |
| Large | нет явного | unlimited | ~20 credits base | сложные, хочется быстро |

- Фактическая стоимость зависит от CPU-секунд и объёма данных.
- **Failed executions ARE charged.**
- Popular dashboards выполняются на community cluster за 0 кредитов.
- Scheduled executions не могут использовать Small.

### Datapoints billing
- Формула: `credits = (rows × columns) / 1000` для результатов
- Default limit per result fetch: **250,000 datapoints** (защита от случайного слива)
- Max result size (internal): **32 GB**, для больших — `allow_partial_results=true`

## API Endpoints

### Base: `https://api.dune.com/api/v1/`
### Auth: header `X-DUNE-API-KEY: <key>` или query param `?api_key=<key>`

### Execute Query
`POST /query/{query_id}/execute`
- Body:
  - `performance`: `"medium"` (default) | `"large"`
  - `query_parameters`: `{ "key": "value" }` (partial OK, unmapped используют defaults)
- Response: `{ execution_id, state }`
- `state` values: `QUERY_STATE_PENDING` (ждёт слота), `QUERY_STATE_EXECUTING` (работает)
- 402 error → превышен лимит billing cycle

### Get Execution Results (JSON)
`GET /execution/{execution_id}/results`
- Params:
  - `limit`, `offset` — пагинация
  - `sample_count` — униформная выборка (несовместим с limit/offset/filters)
  - `filters` — WHERE-like expression
  - `sort_by` — ORDER BY-like
  - `columns` — CSV список нужных колонок (экономит datapoints)
  - `allow_partial_results` — для > 32 GB
- Response metadata:
  - `row_count` (текущая страница), `total_row_count` (всё)
  - `result_set_bytes` (страница), `total_result_set_bytes` (всё)
  - `datapoint_count` (для биллинга)
  - `execution_time_millis`
  - `next_offset`, `next_uri` — для пагинации

### Get Execution Results (CSV)
`GET /execution/{execution_id}/results/csv`
- Same params
- Пагинация — через headers: `X-Dune-Next-Offset`, `X-Dune-Next-Uri`

### Get Latest Query Result
`GET /query/{query_id}/results`
- То же, но берёт последнее выполнение (не запускает новое)

### Query Execution Limits
- **Timeout:** 30 минут hard max
- Invalid `offset` → empty response БЕЗ ошибки (важно: проверять `total_row_count`)
- Server может override `limit` если слишком большой (например 500k → 30k) и вернёт корректный `next_offset`

## MCP Server

### Config
- URL: `https://api.dune.com/mcp/v1`
- Auth: `x-dune-api-key` header или `?api_key=...`
- OAuth 2.0 поддерживается (PKCE S256)

### Setup Claude Code
```bash
claude mcp add --scope user --transport http dune https://api.dune.com/mcp/v1 --header "x-dune-api-key: <key>"
```

### 20 инструментов

**Discovery (5):**
- `searchDocs` — поиск по Dune docs
- `searchTables` — поиск таблиц по protocol/chain/category/schema
- `searchTablesByContractAddress` — decoded event/call таблицы для контракта
- `listBlockchains` — список сетей + количество таблиц
- `getTableSize` — оценка объёма скана

**Query lifecycle (5):**
- `createDuneQuery`, `getDuneQuery`, `updateDuneQuery`
- `executeQueryById` — возвращает execution_id
- `getExecutionResults` — статус + данные

**Visualization (5):**
- `generateVisualization`, `getVisualization`, `updateVisualization`, `deleteVisualization`, `listQueryVisualizations`

**Dashboard (4):**
- `createDashboard`, `getDashboard`, `updateDashboard`, `archiveDashboard`

**Account (1):**
- `getUsage` — проверка израсходованных кредитов в текущем billing period

### Ограничение MCP
- Codex: default 60s timeout → обрывает долгие запросы, нужно `tool_timeout_sec` 300+.

## Optimization (Writing Efficient Queries)

### Partition pruning (критично)
- **Всегда** добавлять фильтр по `block_time` или `block_date` в WHERE
- Для cross-chain таблиц (`tokens.transfers`, `dex.trades`, `evms.erc20_evt_transfers`) — фильтровать ещё по `blockchain`
- Time-фильтр в JOIN... ON clause, не только в WHERE

### Column selection
- Dune использует columnar storage — `SELECT col1, col2` дешевле `SELECT *`
- Никогда не `SELECT *` на `transactions`/`logs`/`traces`

### JOIN
- По indexed колонкам: `tx_hash + block_date`
- Time filter — в ON, если возможно

### Patterns
- **CTE вместо nested subqueries** — читаемее и дешевле
- **Window functions вместо correlated subqueries** (rolling averages, block ranges)
- **Всегда LIMIT** для exploration
- **ORDER BY без LIMIT** — дорого на больших

### Advanced
- Предпочитать curated/decoded tables (e.g. `uniswap_v3_ethereum.*`) сырым `ethereum.logs`
- Materialized views — для повторяющихся вычислений
- **Incremental queries** — до 100x дешевле для scheduled workloads (обрабатывать только новое)

## Pre-execution cost estimation

**Нет точного способа** — "You'll see costs after execution" (официальный FAQ).

Но можно:
- `getTableSize` перед запросом — получить оценку объёма скана
- Оценить примерно: datapoints = rows × cols, credits ≈ datapoints/1000 + compute
- Cost ceilings: per-query execution limit + monthly account cap (настраиваются в UI)

## Caching

- Результаты кэшируются
- `expires_at` в response — сколько они валидны
- Повторный `getExecutionResults` по тому же `execution_id` — бесплатно
- `getLatestQueryResult` использует кэш если запрос не менялся

## Sources

- https://docs.dune.com/api-reference/overview/billing
- https://docs.dune.com/api-reference/overview/faq
- https://docs.dune.com/api-reference/overview/introduction
- https://docs.dune.com/api-reference/agents/mcp
- https://docs.dune.com/api-reference/executions/endpoint/execute-query
- https://docs.dune.com/api-reference/executions/endpoint/get-query-result
- https://docs.dune.com/api-reference/executions/endpoint/get-execution-result-csv
- https://docs.dune.com/api-reference/executions/pagination
- https://docs.dune.com/query-engine/query-executions
- https://docs.dune.com/query-engine/writing-efficient-queries
- https://docs.dune.com/learning/how-tos/credit-system
- https://dune.com/pricing
