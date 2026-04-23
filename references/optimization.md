# Dune SQL Optimization

Patterns that reduce credit burn and query runtime. Applied in order of impact.

## 1. Partition pruning (biggest win)

Most Dune tables are partitioned by `block_date` or `block_time`. Without a time filter, the engine scans everything.

### Always do this
```sql
WHERE block_time >= NOW() - INTERVAL '7' DAY
```

### For cross-chain tables — filter chain too
Cross-chain tables (`tokens.transfers`, `dex.trades`, `evms.erc20_evt_transfers`, `nft.trades`, `stablecoins.transfers`, `labels.addresses`) scan all chains unless you narrow:
```sql
WHERE
  blockchain = 'ethereum'
  AND block_time >= NOW() - INTERVAL '7' DAY
```

### Time filters in JOINs
Push time filter into `ON` clause when joining two time-partitioned tables:
```sql
FROM ethereum.transactions t
JOIN ethereum.logs l
  ON l.tx_hash = t.hash
  AND l.block_date = t.block_date   -- partition alignment
WHERE t.block_time >= NOW() - INTERVAL '7' DAY
```

## 2. Column selection

Dune uses **columnar storage**. `SELECT *` reads all columns from disk even if you only use two. This directly scales credit cost.

### Don't
```sql
SELECT * FROM tokens_ethereum.transfers WHERE ...
```

### Do
```sql
SELECT block_time, tx_hash, amount_usd, "from", "to"
FROM tokens_ethereum.transfers
WHERE ...
```

Exception: quick exploration (`SELECT * FROM ... LIMIT 10`) is fine — the LIMIT caps the scan.

## 3. Prefer decoded/curated tables

| Avoid (raw, expensive) | Prefer (decoded, cheap) |
|------------------------|-------------------------|
| `ethereum.logs` filtered by `topic0` | `uniswap_v3_ethereum.Pool_evt_Swap` |
| `ethereum.transactions` filtered by calldata | Protocol function call tables |
| `ethereum.traces` | Any decoded event/call table |
| `tokens_ethereum.transfers` for trades | `dex.trades` (already has USD values) |
| `prices.usd` when you need current | `prices.usd_latest` |
| `tokens_*.transfers` aggregated daily | `tokens_*.balances_daily` if you want balances |

Raw tables exist for edge cases where no decoded table is available.

## 4. CTE over correlated subqueries

### Slow (correlated)
```sql
SELECT
  tx_hash,
  (SELECT price FROM prices.usd
   WHERE contract_address = t.token_address
   AND minute <= t.block_time
   ORDER BY minute DESC LIMIT 1) AS price
FROM tokens_ethereum.transfers t
```

### Fast (CTE + join)
```sql
WITH tx AS (
  SELECT tx_hash, token_address, block_time
  FROM tokens_ethereum.transfers
  WHERE block_time >= NOW() - INTERVAL '7' DAY
)
SELECT tx.tx_hash, p.price
FROM tx
LEFT JOIN prices.usd p
  ON p.contract_address = tx.token_address
  AND p.blockchain = 'ethereum'
  AND p.minute = date_trunc('minute', tx.block_time)
```

## 5. Window functions instead of self-joins

Rolling metrics (7-day avg, cumulative sum, rank per chain) should use window functions — self-joins on the same large table are exponentially more expensive.

### Slow
```sql
SELECT a.day, AVG(b.volume)
FROM daily a
JOIN daily b ON b.day BETWEEN a.day - 6 AND a.day
GROUP BY a.day
```

### Fast
```sql
SELECT day, AVG(volume) OVER (ORDER BY day ROWS 6 PRECEDING) AS rolling_7d
FROM daily
```

## 6. LIMIT always (except when you really need the full set)

- Exploration: `LIMIT 100`
- Visualization: `LIMIT 1000` for time series, `LIMIT 50` for rankings
- Sanity check: `LIMIT 1` just to see column shape

**`ORDER BY` without `LIMIT` on a large table is especially expensive** — the engine must sort everything.

## 7. Use `sample_count` for distributions

When you want "rough shape of the data, not all of it":
```
mcp__dune__getExecutionResults(execution_id=..., sample_count=10000)
```
Returns a uniform sample, dramatically cheaper than fetching all rows.

## 8. Incremental queries (for scheduled workloads)

If you rerun the same query daily, process only new blocks:
```sql
WHERE block_time > (SELECT MAX(block_time) FROM @my_query_output)
```

Dune materialized views + incremental patterns can be **100× cheaper** than full re-scans for schedules.

## 9. Large tables that need extra care

These scan huge data even with moderate filters — always narrow aggressively:

| Table | Recommended filter |
|-------|--------------------|
| `ethereum.logs`, `arbitrum.logs`, etc. | Time ≤ 7 days AND `contract_address` or `topic0` |
| `ethereum.transactions` | Time ≤ 7 days AND `to`/`from` OR calldata hash |
| `ethereum.traces` | Time ≤ 1 day AND contract — AVOID if possible |
| `solana.transactions` | Time ≤ 1 day — Solana is enormous |
| `dex_solana.trades` | Time ≤ 7 days AND token or trader address |

Call `getTableSize` before writing queries on unfamiliar large tables.

## 10. Avoid these anti-patterns

- **`SELECT *` on `*.transactions`/`*.logs`** — scans all columns
- **No time filter** on partitioned tables — full scan
- **`DISTINCT` on large sets** without filter — expensive
- **`ORDER BY` without `LIMIT`** — full sort
- **String operations (`LIKE '%abc%'`)** on address columns — impossible to index
- **Correlated subqueries per row** — CTE+JOIN instead
- **Joining raw tables when decoded exists** — always prefer curated
- **N+1 fetching of results** (see below) — the most expensive mistake

## 11. N+1 result fetching — biggest hidden credit drain

**Antipattern:**
```python
# DON'T — 2000 wallet addresses, one fetch each
for wallet in wallets:
    execute_query(query_id, params={"wallet": wallet})  # 15 credits ×
    results = get_results(execution_id, limit=10)       # 10 credits ×
# = 25 × 2000 = 50,000 credits
```

**Fix:**
```python
# DO — one query over all wallets, paginated fetch
execute_query(query_id, params={"wallets": "0xa,0xb,..."})   # 15 credits once
# SQL uses: WHERE wallet IN (SELECT wallet FROM UNNEST(ARRAY[...]))
for page in paginate_results(execution_id, limit=32000):     # few fetches
    process(page)
# = 15 + (few fetches) ≈ tens of credits total
```

**Rule of thumb:** every read of the result set costs credits (datapoints × rows). Minimize the number of fetch calls. Use the largest acceptable `limit` per page and follow `next_uri`/`X-Dune-Next-Uri` to paginate — don't call in a tight loop with tiny limits.

## Source

- https://docs.dune.com/query-engine/writing-efficient-queries
- https://docs.dune.com/query-engine/query-executions
