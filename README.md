# Dune Analytics Skill

SQL-driven blockchain analytics on [Dune Analytics](https://dune.com) (100+ chains) for Claude Code.

## What it does

- Writes DuneSQL (Trino dialect) queries with proper partition pruning
- Creates / executes / polls Dune queries via MCP (20-tool catalog)
- Enforces credit budget — **500 credits → warn, 700 → block** per operation (execute and download evaluated independently)
- Manages FREE (default) and PAID keys with automatic rotation on `402`

## When it triggers

Any request involving blockchain data analytics, on-chain queries, wallet activity, DEX volume, NFT sales, token prices, protocol metrics — even without the word "Dune". Also triggers on bare contract addresses when the user wants associated data.

## Files

| File | Purpose |
|------|---------|
| [`SKILL.md`](SKILL.md) | Workflow, hard rules, syntax cheatsheet |
| [`references/tables.md`](references/tables.md) | Canonical table names, columns, use cases |
| [`references/sql-templates.md`](references/sql-templates.md) | 16 ready-to-use SQL templates |
| [`references/optimization.md`](references/optimization.md) | Partition pruning, CTE patterns, anti-patterns (incl. "N+1 fetching") |
| [`references/credits.md`](references/credits.md) | Budget rules, cost model, multi-key rotation |
| [`references/paid-endpoints.md`](references/paid-endpoints.md) | Direct HTTP for bulk exports (Python pagination patterns) |

## Key rules

1. **Estimate before executing** — use `getTableSize` and expected row count to project credits
2. **500 / 700 thresholds** applied independently to execute and to download
3. **Never `SELECT *`** on `*.transactions` / `*.logs` (columnar storage)
4. **Always time-filter** on partitioned tables (`block_time >= NOW() - INTERVAL 'N' DAY`)
5. **Multi-FREE rotation auto, FREE→PAID needs user consent**

## Quick example

```
> "Find top holders of USDC on Ethereum"

Skill:
  1. Identifies table: tokens_ethereum.balances_daily
  2. Estimates: filtered to 1 day × 1 token = ~50 credits
  3. Proceeds silently (< 500)
  4. Returns top 100 with percentages
```

## Setup

Set `DUNE_API_KEY_FREE` (required) and optionally `DUNE_API_KEY_PAID` in `.env`. If you have multiple free keys (for rotation), set `DUNE_API_KEY_FREE_2`, `DUNE_API_KEY_FREE_3`, etc.

The MCP server is configured at `https://api.dune.com/mcp/v1`. Setup details in [`INSTALL.md`](INSTALL.md).
