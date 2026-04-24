# Dune Credits — Budget Rules & Cost Management

## Credit budget thresholds (HARD RULES)

The user has a limited credit budget and pays real money for the PAID key. Thresholds are applied **independently** to execution and to download — not summed.

| Estimated credits (per operation) | Action |
|-----------------------------------|--------|
| `< 500` | Proceed silently |
| `500 – 700` | Report estimate + reason before the operation, wait for user acknowledgement |
| `> 700` | STOP. Propose a cheaper alternative (tighter filter, smaller window, curated table, sample_count, columns filter). Execute only with explicit user approval |

**Independent** means:
- Execute = 100 credits + Download = 600 credits → both under threshold, proceed
- Execute = 100 credits + Download = 800 credits → STOP on the download only
- Execute = 800 credits → STOP before executing, regardless of download

Evaluate each operation against its own budget. Do not sum them.

## ⚠️ THE 2000-FETCHES ANTIPATTERN (most common credit killer)

**Real case:** user's friend paid `15 (execute) + 2000 × 10 (fetches) = 20,015 credits` because they fetched the result in 2000 tiny API calls instead of one paginated export.

**Why:** every `getExecutionResults` / `/results` call charges credits for the datapoints returned. Calling it 2000 times with tiny `limit=10` costs 2000× the per-call overhead.

**The rule:** once a query has executed, read the result in as few fetches as possible.
- CSV export: `limit=32000` per page, follow `X-Dune-Next-Uri` headers
- JSON fetch: `limit=10000` per page, follow `next_uri`
- Re-reading cached results: use `getLatestQueryResult` once, not a fetch per row

**If you see yourself planning a loop like `for id in ids: fetch(id)`:** stop. Rewrite as one query with `WHERE id IN (...)` and one paginated fetch. If you're iterating rows the wrong way, the cost multiplier is massive.

## Key loading policy (works with 1, 2, or N keys)

The skill adapts to whatever keys are configured. On every new session, check which env vars exist:

| Env var | Role |
|---------|------|
| `DUNE_API_KEY_FREE` (or MCP default key) | Primary free key |
| `DUNE_API_KEY_FREE_2`, `DUNE_API_KEY_FREE_3`, … | Additional free keys for rotation |
| `DUNE_API_KEY_PAID` | Paid key for bulk/enterprise |

### Modes

**Single-key mode** — only one key loaded:
- All operations use that key
- Same 500/700 thresholds apply
- No auto-switch (nowhere to go)

**FREE + PAID** — both loaded (user's typical setup):
- Default = FREE
- Auto-switch to PAID (with announce) when any of:
  1. Operation is **strictly cheaper** on PAID (e.g. Plus plan 2 cr/MB vs Free 20 cr/MB → 3×+ savings)
  2. Endpoint is **PAID-only** (some enterprise features, queries endpoint on Plus+)
  3. FREE key returned `402` (quota exhausted) and task can't wait for monthly reset
  4. Bulk export > 32k rows where pagination cost on FREE exceeds PAID export

In all four cases: **announce before executing**, reason stated, user can abort with ctrl+c or a reply. **Never silent.**

**Multi-FREE + PAID (rotation mode)** — several FREE + one PAID:
- Use `FREE`. On `402`, rotate to `FREE_2`, `FREE_3`, etc. — free rotation is automatic and silent (no money spent)
- When ALL FREE keys hit 402, apply the auto-switch rule above (announce + proceed on PAID unless user aborts)

### Auto-switch algorithm (pseudo-code)

```
keys = [FREE, FREE_2, FREE_3, ...]   # available free keys
for key in keys:
    try:
        return call(endpoint, auth=key)
    except HTTP402:
        continue                      # try next free, silent

# All free exhausted OR one of the auto-switch triggers fires:
if PAID available:
    reason = determine_reason()       # "all FREE exhausted" | "PAID-only endpoint"
                                      # | "bulk export, PAID is 10x cheaper" | ...
    announce(f"Using PAID key, reason: {reason}. Estimated cost: X credits.")
    return call(endpoint, auth=PAID)  # proceed unless user interrupts
else:
    raise error("Out of free quota, no PAID key configured")
```

The announcement pattern is always: `"Using PAID, reason: <concrete justification>"`. If the reason can't be stated in one line, it's not a clear auto-switch — ask the user instead.

### Why separate keys instead of one shared

1. **Safety** — routine exploration (`searchTables`, `getTableSize`, small queries) can't accidentally spend PAID budget
2. **Auditability** — spend shows up on separate Dune accounts, easy to see what each key burned
3. **Explicit consent** — every PAID operation requires a conscious decision, not quiet drift

### When using PAID (always)

1. **Announce before executing**: *"Using PAID key. Reason: bulk export of N MB ≈ M credits on Plus tier (vs ~10× more on FREE)"*
2. **Never silent.** User must see the announcement and can abort before or during.
3. **500/700 thresholds still apply** — PAID does not exempt. 700+ credits still stops, regardless of which key.
4. **Log downloads** so user can audit spend.
5. If you're unsure whether the auto-switch trigger is clearly met — **ask the user** instead of guessing.

## Cost model (what consumes credits)

### 1. Query execution
Depends on CPU-seconds and data scanned:
- **Small engine:** 2 min timeout, 3 concurrent, cheapest. Use for exploratory queries.
- **Medium engine (default):** ~10 credits baseline, scales with resources. Use for standard queries.
- **Large engine:** ~20 credits baseline, ~2× compute of Medium. Use only when query is slow on Medium and user needs speed.

**Failed executions ARE charged** — validate SQL before executing. A typo in a 5-minute query costs the same as a successful one.

**Popular public dashboards** run free on community cluster. Your own queries do NOT benefit from this.

### 2. Export (download results) — biggest silent cost

| Plan | Credits per MB exported |
|------|-------------------------|
| Free | 20 |
| Analyst | 10 |
| Plus | 2 |

**This stacks with execution cost.** A query that scans 10 GB might cost 50 credits to execute, but if you export 500 MB of results:
- Free plan: `50 + 500×20 = 10,050` credits
- Plus plan: `50 + 500×2 = 1,050` credits

Moral: on Free, be surgical about what you download.

### 3. Datapoints formula (for result fetching)
`credits = (rows × columns) / 1000`

Default cap per API response: **250,000 datapoints** to prevent runaway spend. You can request more with pagination.

### 4. Write operations (creating tables / inserting data)
3 credits/GB written, minimum 1 credit per operation. Rarely relevant for analytics.

## Pre-execution cost estimation

Dune doesn't give exact cost upfront. Best practices:

### Step 1: size the scan
```
mcp__dune__getTableSize(table="dex_solana.trades")
```
Returns table size. Apply filters proportionally:
- Time filter `7 DAY` on a 180-day table ≈ scan 4% of total
- `blockchain = 'ethereum'` on cross-chain ≈ scan 30-50% depending on chain share

### Step 2: estimate output datapoints
`expected_rows × columns_selected` → apply `/1000` for credits

### Step 3: estimate export MB
`rows × avg_row_bytes / 1024² ≈ MB`, then multiply by plan rate

### Step 4: sum and apply threshold
Execution estimate + export estimate → compare against 500 / 700 thresholds.

### Check remaining budget
```
mcp__dune__getUsage()
```
Returns current billing-period credit usage. Run this if user asks about balance or before a big export.

## Cheap-by-design patterns

These patterns dramatically reduce credit burn. Prefer them when the user's request allows:

1. **Curated tables over raw**: `uniswap_v3_ethereum.Pool_evt_Swap` is vastly cheaper than filtering `ethereum.logs` by topic.
2. **`prices.usd_latest`** instead of `prices.usd` when you only need current price.
3. **`sample_count` parameter** when the user wants a rough distribution, not all rows.
4. **Incremental queries** — scheduled jobs processing only new blocks since last checkpoint can be 100× cheaper than full re-scans.
5. **Tight time windows** — default to 7 days, expand only if needed.
6. **`columns=a,b,c` query param** on result fetch — reduces datapoints billed.
7. **`LIMIT 1000`** for exploration — sanity-check the query shape before removing the limit.

## Rate limits

| Plan | Requests per minute |
|------|---------------------|
| Free | 40 |
| Plus | 200 |
| Enterprise | Custom |

Exceeded → 429 response. Back off, don't retry in a tight loop.

## Source

- https://docs.dune.com/api-reference/overview/billing
- https://docs.dune.com/query-engine/query-executions
- https://docs.dune.com/learning/how-tos/credit-system
- https://docs.dune.com/api-reference/overview/faq
