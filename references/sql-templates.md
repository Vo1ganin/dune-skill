# SQL Templates for Dune Analytics

Copy and adapt these templates. All use DuneSQL (Trino dialect).
Replace `0xYOUR_ADDRESS`, `0xTOKEN_ADDRESS`, etc. with actual values.

---

## 1. Wallet activity — all token transfers in/out

```sql
SELECT
  block_time,
  token_symbol,
  CASE
    WHEN "from" = 0xYOUR_ADDRESS THEN 'OUT'
    ELSE 'IN'
  END AS direction,
  amount,
  amount_usd,
  "from",
  "to",
  tx_hash
FROM tokens_ethereum.transfers
WHERE
  ("from" = 0xYOUR_ADDRESS OR "to" = 0xYOUR_ADDRESS)
  AND block_time >= NOW() - INTERVAL '30' DAY
ORDER BY block_time DESC
LIMIT 200
```

---

## 2. Wallet ETH balance changes (native transfers)

```sql
SELECT
  block_time,
  hash AS tx_hash,
  CASE
    WHEN "from" = 0xYOUR_ADDRESS THEN -CAST(value AS DOUBLE) / 1e18
    ELSE CAST(value AS DOUBLE) / 1e18
  END AS eth_delta,
  "from",
  "to",
  success
FROM ethereum.transactions
WHERE
  ("from" = 0xYOUR_ADDRESS OR "to" = 0xYOUR_ADDRESS)
  AND block_time >= NOW() - INTERVAL '30' DAY
  AND value > UINT256 '0'
ORDER BY block_time DESC
LIMIT 200
```

---

## 3. Top token holders (current snapshot)

```sql
SELECT
  address,
  amount,
  amount_usd,
  ROUND(100.0 * amount / SUM(amount) OVER (), 4) AS pct_supply
FROM tokens_ethereum.balances_daily
WHERE
  token_address = 0xTOKEN_ADDRESS
  AND day = (SELECT MAX(day) FROM tokens_ethereum.balances_daily)
ORDER BY amount DESC
LIMIT 100
```

---

## 4. Token price history (daily close)

```sql
SELECT
  date_trunc('day', minute) AS day,
  AVG(price) AS avg_price_usd,
  MIN(price) AS low_usd,
  MAX(price) AS high_usd
FROM prices.usd
WHERE
  contract_address = 0xTOKEN_ADDRESS
  AND blockchain = 'ethereum'
  AND minute >= NOW() - INTERVAL '90' DAY
GROUP BY 1
ORDER BY 1
```

---

## 5. Current token price (fast)

```sql
SELECT symbol, price AS price_usd, blockchain
FROM prices.usd_latest
WHERE
  contract_address = 0xTOKEN_ADDRESS
  AND blockchain = 'ethereum'
```

---

## 6. DEX volume by protocol (last 7 days)

```sql
SELECT
  project,
  version,
  COUNT(*) AS trade_count,
  SUM(amount_usd) AS volume_usd,
  COUNT(DISTINCT taker) AS unique_traders
FROM dex.trades
WHERE
  blockchain = 'ethereum'
  AND block_time >= NOW() - INTERVAL '7' DAY
GROUP BY 1, 2
ORDER BY volume_usd DESC
LIMIT 20
```

---

## 7. DEX volume for a specific token (daily)

```sql
SELECT
  date_trunc('day', block_time) AS day,
  SUM(amount_usd) AS volume_usd,
  COUNT(*) AS trade_count
FROM dex.trades
WHERE
  (token_bought_address = 0xTOKEN_ADDRESS OR token_sold_address = 0xTOKEN_ADDRESS)
  AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

---

## 8. Daily active users of a DEX pool

```sql
SELECT
  date_trunc('day', evt_block_time) AS day,
  COUNT(*) AS swap_count,
  COUNT(DISTINCT sender) AS unique_wallets,
  SUM(ABS(CAST(amount0 AS DOUBLE))) AS volume_token0_raw
FROM uniswap_v3_ethereum.Pool_evt_Swap
WHERE
  contract_address = 0xPOOL_ADDRESS
  AND evt_block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1
```

---

## 9. NFT collection — recent sales

```sql
SELECT
  block_time,
  token_id,
  seller,
  buyer,
  price_usd,
  amount_original AS price_eth,
  currency_symbol,
  project AS marketplace,
  tx_hash
FROM nft.trades
WHERE
  nft_contract_address = 0xCOLLECTION_ADDRESS
  AND block_time >= NOW() - INTERVAL '30' DAY
ORDER BY block_time DESC
LIMIT 100
```

---

## 10. NFT collection — volume and floor price by day

```sql
SELECT
  date_trunc('day', block_time) AS day,
  COUNT(*) AS sales_count,
  SUM(price_usd) AS volume_usd,
  MIN(price_usd) AS floor_usd,
  AVG(price_usd) AS avg_price_usd,
  COUNT(DISTINCT seller) AS unique_sellers
FROM nft.trades
WHERE
  nft_contract_address = 0xCOLLECTION_ADDRESS
  AND block_time >= NOW() - INTERVAL '90' DAY
GROUP BY 1
ORDER BY 1
```

---

## 11. Stablecoin flows — large transfers

```sql
SELECT
  block_time,
  symbol,
  "from",
  "to",
  amount,
  amount_usd,
  tx_hash
FROM stablecoins.transfers
WHERE
  blockchain = 'ethereum'
  AND amount_usd > 1000000  -- only transfers > $1M
  AND block_time >= NOW() - INTERVAL '7' DAY
ORDER BY amount_usd DESC
LIMIT 100
```

---

## 12. Wallet PnL — transfers enriched with USD prices

```sql
WITH wallet_transfers AS (
  SELECT
    t.block_time,
    t.tx_hash,
    t.token_address,
    t.token_symbol,
    t.amount,
    CASE WHEN t."from" = 0xYOUR_ADDRESS THEN -t.amount ELSE t.amount END AS net_amount,
    p.price AS price_usd_at_time
  FROM tokens_ethereum.transfers t
  LEFT JOIN prices.usd p
    ON p.contract_address = t.token_address
    AND p.blockchain = 'ethereum'
    AND p.minute = date_trunc('minute', t.block_time)
  WHERE
    (t."from" = 0xYOUR_ADDRESS OR t."to" = 0xYOUR_ADDRESS)
    AND t.block_time >= NOW() - INTERVAL '30' DAY
)
SELECT
  token_symbol,
  SUM(net_amount) AS net_balance_change,
  SUM(net_amount * COALESCE(price_usd_at_time, 0)) AS net_usd_flow
FROM wallet_transfers
GROUP BY token_symbol
ORDER BY ABS(net_usd_flow) DESC
```

---

## 13. Protocol TVL proxy — token balance in contract

```sql
SELECT
  date_trunc('day', block_time) AS day,
  SUM(CASE WHEN "to" = 0xCONTRACT_ADDRESS THEN amount ELSE 0 END) -
  SUM(CASE WHEN "from" = 0xCONTRACT_ADDRESS THEN amount ELSE 0 END) AS net_inflow,
  SUM(SUM(CASE WHEN "to" = 0xCONTRACT_ADDRESS THEN amount ELSE 0 END) -
      SUM(CASE WHEN "from" = 0xCONTRACT_ADDRESS THEN amount ELSE 0 END))
    OVER (ORDER BY date_trunc('day', block_time)) AS cumulative_balance
FROM tokens_ethereum.transfers
WHERE
  token_address = 0xTOKEN_ADDRESS
  AND ("from" = 0xCONTRACT_ADDRESS OR "to" = 0xCONTRACT_ADDRESS)
  AND block_time >= NOW() - INTERVAL '180' DAY
GROUP BY 1
ORDER BY 1
```

---

## 14. Gas usage analysis — top gas-consuming contracts

```sql
SELECT
  "to" AS contract_address,
  l.name AS contract_label,
  COUNT(*) AS tx_count,
  SUM(gas_used) AS total_gas,
  AVG(gas_used) AS avg_gas,
  SUM(CAST(gas_used AS DOUBLE) * CAST(gas_price AS DOUBLE) / 1e18) AS total_eth_fees
FROM ethereum.transactions t
LEFT JOIN labels.addresses l
  ON l.address = t."to" AND l.blockchain = 'ethereum'
WHERE
  block_time >= NOW() - INTERVAL '7' DAY
  AND success = TRUE
GROUP BY 1, 2
ORDER BY total_gas DESC
LIMIT 20
```

---

## 15. Aave liquidations — recent events

```sql
SELECT
  evt_block_time,
  reserve AS collateral_asset,
  user AS borrower,
  liquidationAmount AS liquidated_amount_raw,
  CAST(liquidationAmount AS DOUBLE) / 1e18 AS liquidated_amount,
  evt_tx_hash
FROM aave_v3_ethereum.Pool_evt_LiquidationCall
WHERE evt_block_time >= NOW() - INTERVAL '7' DAY
ORDER BY evt_block_time DESC
LIMIT 100
```

---

## 16. Address label enrichment

```sql
-- Enrich a list of wallets with known labels
SELECT
  w.address,
  COALESCE(l.name, 'Unknown') AS label,
  COALESCE(l.category, 'unknown') AS category
FROM (
  VALUES
    (0xADDRESS_1),
    (0xADDRESS_2),
    (0xADDRESS_3)
) AS w(address)
LEFT JOIN labels.addresses l
  ON l.address = w.address AND l.blockchain = 'ethereum'
```

---

## Tips for adapting templates

- **Change the chain:** Replace `_ethereum` suffix with `_arbitrum`, `_optimism`, `_base`, etc.
- **Change the time range:** Replace `INTERVAL '30' DAY` with any value
- **Add USD filter:** Add `AND amount_usd > 100` to skip dust transfers
- **Group by week/month:** Use `date_trunc('week', ...)` or `date_trunc('month', ...)`
- **Multiple tokens:** Use `WHERE token_address IN (0xADDR1, 0xADDR2)`
