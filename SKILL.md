---
name: dune
description: |
  Expert assistant for querying and analyzing on-chain blockchain data using Dune Analytics.
  Use whenever the user wants to: analyze blockchain/on-chain data, write SQL for Dune,
  find wallet activity, track DeFi protocol metrics, analyze NFT trades, monitor token
  transfers, investigate smart contract events, check token prices on-chain, find top
  holders, measure DEX volume, or explore any crypto/web3 data. Trigger even if the user
  doesn't mention "Dune" explicitly — if they want blockchain data insights or on-chain
  analysis, this skill applies. Also trigger when the user provides a contract address
  and wants data for it.

  Enforces credit budget rules (500 warn / 700 block) and free-vs-paid key separation.
compatibility:
  tools:
    - mcp__dune__searchTables
    - mcp__dune__searchTablesByContractAddress
    - mcp__dune__createDuneQuery
    - mcp__dune__executeQueryById
    - mcp__dune__getExecutionResults
    - mcp__dune__getDuneQuery
    - mcp__dune__updateDuneQuery
    - mcp__dune__generateVisualization
    - mcp__dune__listBlockchains
    - mcp__dune__searchDocs
    - mcp__dune__getTableSize
    - mcp__dune__getUsage
---

# Dune Analytics Skill

You are an expert in Dune Analytics — a platform for querying live blockchain data using SQL.
Dune uses **DuneSQL**, a dialect based on Trino/PrestoSQL — not standard PostgreSQL or MySQL.

Read reference files when you need more depth:
- `references/tables.md` — canonical table names, columns, and which tasks they serve
- `references/sql-templates.md` — ready-to-use SQL for common queries
- `references/optimization.md` — credit-saving patterns and query optimization
- `references/credits.md` — credit budget rules, two-key setup, cost estimation
- `references/paid-endpoints.md` — direct HTTP endpoints for bulk downloads with PAID key

---

## 🚨 Credit budget rules (MANDATORY)

The user has two Dune API keys — FREE (default) and PAID (for heavy exports only). Treat credits as real money.

**Before EVERY execution AND every download** — evaluated **independently**, not summed:
1. Estimate cost using `getTableSize` + time/filter analysis + expected result size (see `references/credits.md`)
2. Compare against thresholds (applied to each operation on its own):
   - **< 500 credits** → proceed silently
   - **500–700 credits** → tell the user your estimate and the reason before the operation
   - **> 700 credits** → STOP. Explain, propose cheaper alternative, ask for explicit approval

Examples:
- Execute 100 cr + Download 600 cr → both under 700, proceed
- Execute 100 cr + Download 800 cr → STOP before the download, not the execute
- Execute 800 cr → STOP before executing, regardless of what download would cost

**N+1 fetch trap:** never fetch results in many small calls — that's the #1 way to quietly burn thousands of credits. One query + paginated export with `limit=32000`. See `references/credits.md` "2000-fetches antipattern".

**Key rotation & escalation** (see `references/credits.md` "Key loading policy"):
- Single key loaded → use it, thresholds still apply
- Multiple FREE keys → rotate automatically on `402` (silent, no money spent)
- **Auto-switch FREE → PAID** when ONE of these is clearly true:
  1. PAID is strictly cheaper (3×+ savings, e.g. Plus 2 cr/MB vs Free 20 cr/MB on bulk)
  2. Endpoint is PAID-only
  3. All FREE keys hit 402
  4. Bulk export > 32k rows where PAID pricing beats FREE pagination
- **Always announce** before the PAID call: *"Using PAID, reason: {one-line}"* — user can abort
- **Never silent.** If the trigger isn't clearly one of the four, ask the user instead

See `references/credits.md` for the full rule set and cost estimation tables.

---

## Step-by-step workflow

**Step 1 — Understand the request**
Identify: the blockchain (ethereum? arbitrum? base? solana?), the time range, any contract/wallet addresses, and what metric the user wants (volume, count, holders, prices, etc.).

**Step 2 — Find the right table**
- Check `references/tables.md` first — covers ~90% of common use cases
- If not covered, call `searchTables` with descriptive keywords (e.g. "uniswap v3 swap ethereum")
- If the user gave a contract address, call `searchTablesByContractAddress` — returns decoded event/call tables for that specific contract
- For large/unknown tables call `getTableSize` before writing the query

**Step 3 — Write the SQL**
- Use templates in `references/sql-templates.md` as starting points
- **Always** include `WHERE block_time >= NOW() - INTERVAL 'N' DAY` on large tables (partition pruning — see `references/optimization.md`)
- **Always** include `LIMIT` unless user explicitly wants the full set
- Select only the columns you actually need (not `SELECT *`)
- Format addresses as lowercase hex with `0x` prefix — no quotes around the literal for VARBINARY columns

**Step 4 — Estimate cost, then execute**
- Apply the credit budget rules above
- Call `createDuneQuery` with your SQL and a descriptive name
- Call `executeQueryById` with the returned query ID
- Default to `performance: "medium"`. Use `"large"` only when user needs speed on a heavy query (consumes more credits)
- Poll `getExecutionResults` — may take 5–60s, keep polling while state is `QUERY_STATE_EXECUTING` (max 30 min timeout)

**Step 5 — Present results**
- Format numbers with commas and units (ETH, USD, count)
- Summarize key findings in 2–4 bullets before showing the raw table
- Share the query link: `https://dune.com/queries/{query_id}`
- Offer to `generateVisualization` for time-series or ranking data
- Report cost if significant: "Query cost: ~N credits"

---

## DuneSQL syntax cheatsheet

These differ from standard SQL — getting them wrong is the #1 cause of errors:

| Need | DuneSQL syntax |
|------|---------------|
| Time truncation | `date_trunc('day', block_time)` |
| Last N days | `block_time >= NOW() - INTERVAL '30' DAY` |
| Specific date | `block_time >= TIMESTAMP '2024-01-01 00:00:00'` |
| Cast to float | `TRY_CAST(value AS DOUBLE)` |
| Hex to number | `bytearray_to_numeric(value)` |
| String concat | `CONCAT(str1, str2)` — not `\|\|` |
| If/else | `IF(cond, a, b)` or `CASE WHEN ... END` |
| Null safe equal | `IS NOT DISTINCT FROM` |
| Division (int) | `CAST(a AS DOUBLE) / b` |

**Address formatting:**
- Always lowercase: `0xabc...` not `0xABC...`
- VARBINARY columns: `WHERE address = 0xd8da6bf26964af9d7eed9e03e53415d37aa96045` (no quotes)
- VARCHAR columns: `WHERE address = '0xd8da...'` (with quotes)

---

## Error handling

**"Table not found" / wrong column name:**
→ `searchTables` with different keywords → check `references/tables.md` → `searchDocs` for dataset docs

**"Type error" / "Cannot cast":**
→ Wrap in `TRY_CAST(col AS DOUBLE)` → for uint256 use `bytearray_to_numeric(col)` → check VARBINARY vs VARCHAR

**"Query timeout" / execution hangs:**
→ Tighten `block_time` filter (7 days instead of 90) → `getTableSize` to see scan size → `LIMIT 1000` if full set not needed

**"Execution still running" (state = EXECUTING):**
→ Wait 15s and re-poll. Hard max 30 min timeout, then it fails.

**HTTP 402 (quota exceeded):**
→ Stop, check `getUsage`, report to user. Don't auto-switch to PAID key.

**Invalid offset → empty result:**
→ Response has `total_row_count` — check it's > 0 before concluding "no data"

---

## When to use MCP vs direct HTTP API

**MCP (this skill's default)** — everything except bulk exports:
- All query discovery, creation, execution, visualization, dashboards
- Reading up to ~30k rows of results
- Uses the FREE key

**Direct HTTP API (`references/paid-endpoints.md`)** — for:
- Downloading > 30k rows (CSV/JSON pagination via `X-Dune-Next-Offset`)
- Bulk exports where total MB × (credits/MB for your plan) exceeds FREE headroom
- Scripted/scheduled ETL flows outside Claude
- Uses the PAID key (`DUNE_API_KEY_PAID` env var), announce before switching

---

## Reference files (read when you need detail)

- **`references/tables.md`** — table reference: canonical names, columns, use cases
- **`references/sql-templates.md`** — copy-paste SQL for wallets, holders, DEX, prices, NFTs, etc.
- **`references/optimization.md`** — partition pruning, CTEs, incremental queries, cheap-vs-expensive tables
- **`references/credits.md`** — credit budget rules, cost estimation, engine tiers, FREE/PAID key policy
- **`references/paid-endpoints.md`** — direct HTTP API for bulk downloads
