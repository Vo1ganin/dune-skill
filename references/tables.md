# Dune Table Reference

This file lists the most important Dune tables by use case.
Table naming pattern: `{namespace}_{chain}.{table}` or `{namespace}.{table}` for cross-chain.

---

## Token Transfers (ERC-20)

**`tokens_{chain}.transfers`** — decoded ERC-20 transfers, best for wallet/token activity
| Column | Type | Description |
|--------|------|-------------|
| block_time | TIMESTAMP | when the transfer happened |
| block_number | BIGINT | block number |
| tx_hash | VARBINARY | transaction hash |
| evt_index | INT | log index within tx |
| token_address | VARBINARY | ERC-20 contract address |
| token_symbol | VARCHAR | e.g. 'USDC', 'WETH' |
| token_decimals | INT | decimal places |
| `"from"` | VARBINARY | sender address (quoted — reserved word) |
| `"to"` | VARBINARY | receiver address (quoted — reserved word) |
| amount | DOUBLE | human-readable amount (already divided by decimals) |
| amount_raw | UINT256 | raw on-chain amount |
| amount_usd | DOUBLE | USD value at time of transfer |
| blockchain | VARCHAR | e.g. 'ethereum' |

Chains: `tokens_ethereum`, `tokens_arbitrum`, `tokens_optimism`, `tokens_base`, `tokens_polygon`, `tokens_bnb`

**Warning:** Always filter by `token_address` or `"from"`/`"to"` address — this table is huge.

---

## Raw Transactions

**`{chain}.transactions`** — every transaction on the chain
| Column | Type | Description |
|--------|------|-------------|
| block_time | TIMESTAMP | tx timestamp |
| block_number | BIGINT | |
| hash | VARBINARY | tx hash |
| `"from"` | VARBINARY | sender (quoted) |
| `"to"` | VARBINARY | recipient / contract |
| value | UINT256 | ETH/native token sent (in wei) |
| gas_used | BIGINT | |
| gas_price | UINT256 | in wei |
| success | BOOLEAN | true if not reverted |
| data | VARBINARY | input calldata |
| nonce | BIGINT | |

Chains: `ethereum.transactions`, `arbitrum.transactions`, `optimism.transactions`, `base.transactions`, `polygon.transactions`, `bnb.transactions`

**ETH value in ether:** `CAST(value AS DOUBLE) / 1e18`

---

## Raw Logs (Events)

**`{chain}.logs`** — raw event logs, use only if no decoded table exists
| Column | Type | Description |
|--------|------|-------------|
| block_time | TIMESTAMP | |
| block_number | BIGINT | |
| tx_hash | VARBINARY | |
| contract_address | VARBINARY | emitting contract |
| topic0 | VARBINARY | event signature hash |
| topic1 | VARBINARY | first indexed param |
| topic2 | VARBINARY | second indexed param |
| topic3 | VARBINARY | third indexed param |
| data | VARBINARY | non-indexed params (ABI-encoded) |

Chains: `ethereum.logs`, `arbitrum.logs`, `optimism.logs`, etc.

**Tip:** Prefer decoded tables over raw logs — they're cheaper and have human-readable columns.

---

## Token Prices

**`prices.usd`** — token prices in USD by minute
| Column | Type | Description |
|--------|------|-------------|
| minute | TIMESTAMP | price timestamp (per minute) |
| contract_address | VARBINARY | token contract |
| symbol | VARCHAR | token symbol |
| price | DOUBLE | USD price |
| decimals | INT | |
| blockchain | VARCHAR | |

**`prices.usd_latest`** — latest price only (much faster)
| Column | Type | Description |
|--------|------|-------------|
| contract_address | VARBINARY | |
| symbol | VARCHAR | |
| price | DOUBLE | current USD price |
| blockchain | VARCHAR | |

**Usage tip:** Join `prices.usd` on `minute = date_trunc('minute', block_time)` and `blockchain = 'ethereum'` etc.

---

## DEX Trades

**`dex.trades`** — cross-chain decoded DEX trades (Uniswap, Curve, Sushi, etc.)
| Column | Type | Description |
|--------|------|-------------|
| block_time | TIMESTAMP | |
| block_number | BIGINT | |
| tx_hash | VARBINARY | |
| blockchain | VARCHAR | 'ethereum', 'arbitrum', etc. |
| project | VARCHAR | 'uniswap', 'curve', 'sushiswap' |
| version | VARCHAR | '2', '3', 'v3' |
| token_bought_symbol | VARCHAR | |
| token_sold_symbol | VARCHAR | |
| token_bought_address | VARBINARY | |
| token_sold_address | VARBINARY | |
| amount_usd | DOUBLE | trade size in USD |
| token_bought_amount | DOUBLE | |
| token_sold_amount | DOUBLE | |
| taker | VARBINARY | wallet that traded |
| maker | VARBINARY | LP / pool |
| project_contract_address | VARBINARY | pool address |

---

## Protocol-Specific: Uniswap V3

**`uniswap_v3_{chain}.Pool_evt_Swap`**
| Column | Type | Description |
|--------|------|-------------|
| evt_block_time | TIMESTAMP | |
| evt_tx_hash | VARBINARY | |
| contract_address | VARBINARY | pool address |
| sender | VARBINARY | |
| recipient | VARBINARY | |
| amount0 | INT256 | token0 delta (negative = sold) |
| amount1 | INT256 | token1 delta |
| sqrtPriceX96 | UINT256 | price as sqrt ratio |
| liquidity | UINT256 | pool liquidity |
| tick | INT | current tick |

Chains: `uniswap_v3_ethereum`, `uniswap_v3_arbitrum`, `uniswap_v3_optimism`, `uniswap_v3_base`, `uniswap_v3_polygon`

**`uniswap_v3_{chain}.Factory_evt_PoolCreated`** — pool creation events
**`uniswap_v3_{chain}.Pool_evt_Mint`** — liquidity added
**`uniswap_v3_{chain}.Pool_evt_Burn`** — liquidity removed

---

## NFT Trades

**`nft.trades`** — cross-chain NFT sales (OpenSea, Blur, LooksRare, etc.)
| Column | Type | Description |
|--------|------|-------------|
| block_time | TIMESTAMP | |
| blockchain | VARCHAR | |
| project | VARCHAR | 'opensea', 'blur', 'looksrare' |
| token_id | VARCHAR | NFT token ID |
| nft_contract_address | VARBINARY | collection contract |
| collection | VARCHAR | collection name |
| seller | VARBINARY | |
| buyer | VARBINARY | |
| price_usd | DOUBLE | sale price in USD |
| amount_original | DOUBLE | sale price in native currency |
| currency_symbol | VARCHAR | 'ETH', 'WETH', etc. |
| tx_hash | VARBINARY | |

---

## Token Holders / Balances

**`tokens_{chain}.balances_daily`** — daily token balance snapshots per address
| Column | Type | Description |
|--------|------|-------------|
| day | DATE | snapshot date |
| blockchain | VARCHAR | |
| address | VARBINARY | holder address |
| token_address | VARBINARY | |
| symbol | VARCHAR | |
| amount | DOUBLE | balance (human readable) |
| amount_usd | DOUBLE | USD value |
| last_updated | TIMESTAMP | |

**Note:** For "top holders right now", filter `WHERE day = (SELECT MAX(day) FROM tokens_ethereum.balances_daily)`

---

## Protocol-Specific: Aave V3

**`aave_v3_{chain}.Pool_evt_Supply`** — deposits
**`aave_v3_{chain}.Pool_evt_Withdraw`** — withdrawals
**`aave_v3_{chain}.Pool_evt_Borrow`** — borrows
**`aave_v3_{chain}.Pool_evt_Repay`** — repayments
**`aave_v3_{chain}.Pool_evt_LiquidationCall`** — liquidations

Common columns: `evt_block_time`, `evt_tx_hash`, `reserve` (asset address), `user`, `amount`

Chains: `aave_v3_ethereum`, `aave_v3_arbitrum`, `aave_v3_optimism`, `aave_v3_base`, `aave_v3_polygon`

---

## Stablecoins

**`stablecoins.transfers`** — cross-chain stablecoin transfers (USDC, USDT, DAI, etc.)
| Column | Type | Description |
|--------|------|-------------|
| block_time | TIMESTAMP | |
| blockchain | VARCHAR | |
| symbol | VARCHAR | 'USDC', 'USDT', 'DAI' |
| `"from"` | VARBINARY | |
| `"to"` | VARBINARY | |
| amount | DOUBLE | human-readable |
| amount_usd | DOUBLE | |
| tx_hash | VARBINARY | |

---

## Cross-chain Labels

**`labels.addresses`** — known address labels (exchanges, protocols, whales)
| Column | Type | Description |
|--------|------|-------------|
| address | VARBINARY | |
| blockchain | VARCHAR | |
| name | VARCHAR | human label (e.g. 'Binance 14') |
| category | VARCHAR | 'cex', 'defi', 'nft', 'whale' |
| contributor | VARCHAR | who submitted the label |
| source | VARCHAR | |
| created_at | TIMESTAMP | |

Useful for enriching wallet addresses with names.

---

## Supported Blockchains (common)

`ethereum`, `arbitrum`, `optimism`, `base`, `polygon`, `bnb`, `avalanche_c`, `gnosis`, `fantom`, `celo`, `solana`, `zksync`, `scroll`, `linea`, `mantle`

Call `listBlockchains` for the full current list.
