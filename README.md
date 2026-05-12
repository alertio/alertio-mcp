# Alertio MCP Server

Real-time DeFi risk data for AI agents. Scan lending positions across EVM, Solana, and TON chains, simulate liquidations, monitor stablecoin pegs, and inspect Aave market utilization — no account or API key required.

## Connection

```json
{
  "alertio": {
    "url": "https://alertio.io/api/mcp"
  }
}
```

For stdio-only clients (older versions of Claude Desktop), use `mcp-remote`:

```json
{
  "alertio": {
    "command": "npx",
    "args": ["-y", "mcp-remote", "https://alertio.io/api/mcp"]
  }
}
```

## Tools

### `scan_wallet`

Scans an EVM wallet address across all supported EVM lending protocols and returns every active borrowing position with health factors and liquidation risk.

**Supported protocols**

| Protocol  | Chains                                                      |
|-----------|-------------------------------------------------------------|
| Aave V3   | Ethereum, Base, Arbitrum, Polygon, Optimism, Avalanche, Scroll |
| Aave V4   | Ethereum (Core, Prime, Plus hubs)                           |
| Spark     | Ethereum                                                    |
| Morpho    | Ethereum, Base, Arbitrum                                    |
| Euler V2  | Ethereum                                                    |
| Moonwell  | Base                                                        |

**Input**

| Parameter | Type   | Description                          |
|-----------|--------|--------------------------------------|
| `wallet`  | string | EVM address (`0x…`, 42 chars)        |

**Risk levels**

| Label        | Health factor  |
|--------------|----------------|
| `SAFE`       | HF ≥ 2.0       |
| `AT RISK`    | 1.3 ≤ HF < 2.0 |
| `DANGER`     | 1.0 ≤ HF < 1.3 |
| `LIQUIDATED` | HF < 1.0       |

**Example output**

```
Lending positions for 0xabc...1234

AAVE V3
  ethereum: HF 2.45 · $18K collateral · $7K debt [SAFE]
  base: HF 1.28 · $5K collateral · $3.8K debt [DANGER]

AAVE V4
  No active positions.

SPARK
  No active positions.

MORPHO
  ethereum wstETH/USDC: HF 1.84 · $42K collateral · $22K debt [AT RISK]

EULER V2
  No active positions.

MOONWELL
  Base: HF 1.61 · $3.2K debt [SAFE]

Total active positions: 3
```

---

### `scan_solana_wallet`

Scans a Solana wallet address across Kamino and Marginfi lending protocols.

**Supported protocols**

| Protocol | Chain  |
|----------|--------|
| Kamino   | Solana |
| Marginfi | Solana |

**Input**

| Parameter | Type   | Description                          |
|-----------|--------|--------------------------------------|
| `wallet`  | string | Solana base58 address (32–44 chars)  |

**Example output**

```
Solana lending positions for 7xKX...AbCD

KAMINO
  Solana: HF 1.72 · $12K collateral · $6K debt [AT RISK]

MARGINFI
  No active borrow position.

Total active positions: 1
```

---

### `scan_ton_wallet`

Scans a TON wallet address on the EVAA lending protocol.

**Supported protocols**

| Protocol | Chain |
|----------|-------|
| EVAA     | TON   |

**Input**

| Parameter | Type   | Description                                               |
|-----------|--------|-----------------------------------------------------------|
| `wallet`  | string | TON friendly address (`EQ…` or `UQ…`, exactly 48 chars)  |

**Example output**

```
TON lending positions for EQD...wxYZ

EVAA
  TON: HF 2.10 · $8K collateral · $3K debt [SAFE]

Total active positions: 1
```

---

### `simulate_liquidation`

Simulates the impact of a collateral price drop on all active EVM positions for a wallet (Aave V3/V4, Spark, Morpho, Euler V2, Moonwell). Identifies which positions would be liquidated and which move into a higher risk tier.

**Formula:** `newHF = currentHF × (1 − drop%)`

Assumes all collateral moves uniformly with the price drop. Conservative when collateral is a mix of volatile and stable assets.

**Input**

| Parameter             | Type   | Description                             |
|-----------------------|--------|-----------------------------------------|
| `wallet`              | string | EVM address (`0x…`, 42 chars)           |
| `collateral_drop_pct` | number | Collateral price drop percentage (1–80) |

**Example output**

```
Liquidation simulation for 0xabc...1234
Scenario: collateral drops 30%

  Aave V3 (ethereum): 2.45 → 1.72 [AT RISK] ← was SAFE
  Aave V3 (base): 1.28 → 0.90 [LIQUIDATED] ← was DANGER
  Morpho wstETH/USDC: 1.84 → 1.29 [DANGER] ← was AT RISK
  Moonwell (Base): 1.61 → 1.13 [DANGER] ← was SAFE

△ 2 position(s) at risk at -30%
⚠ 1 position(s) LIQUIDATED at -30%
```

---

### `get_stablecoin_pegs`

Returns current USD peg data for the top stablecoins (USDT, USDC, DAI, FRAX, crvUSD, etc.) sourced from DeFiLlama. A deviation above 0.5% is flagged as a depeg signal.

**Input**

| Parameter | Type   | Required | Description                                             |
|-----------|--------|----------|---------------------------------------------------------|
| `symbol`  | string | No       | Filter to a specific coin, e.g. `"USDC"`. Omit for all.|

**Example output**

```
Stablecoin peg status (USD target = $1.00)

  USDT     $1.0001  (+0.010%) · mcap $143.2B
  USDC     $1.0000  (+0.000%) · mcap $61.4B
  DAI      $0.9998  (-0.020%) · mcap $5.1B
  FRAX     $0.9971  (-0.290%) · mcap $643M  △ watch
  crvUSD   $0.9942  (-0.580%) · mcap $89M   ⚠ DEPEG

⚠ crvUSD showing depeg signal (>0.5% deviation)
```

---

### `get_market_utilization`

Returns Aave V3 lending market utilization rates across 7 chains. High utilization (above the market's optimal rate) causes borrow APY to spike. Useful for assessing rate risk, borrowing capacity, and whether a market is frozen or paused.

**Input**

| Parameter | Type   | Required | Description                                                    |
|-----------|--------|----------|----------------------------------------------------------------|
| `asset`   | string | No       | Filter by symbol, e.g. `"USDC"` or `"WETH"`. Omit for all.   |
| `chain`   | string | No       | One of: `ethereum`, `base`, `arbitrum`, `polygon`, `optimism`, `avalanche`, `scroll`. Omit for all. |

**Example output**

```
Aave V3 market utilization

  Asset     Chain       Util%   Optimal  Borrow APY  Supply APY  Status
  ─────────────────────────────────────────────────────────────────────
  USDC      ethereum    94.2%      90%      12.40%       9.80%  HIGH UTIL △
  USDT      arbitrum    87.3%      90%       5.10%       3.90%  OK
  WETH      base        42.1%      80%       2.30%       0.90%  OK

△ Above optimal utilization: USDC (ethereum) 94.2%
```

---

## Example agent queries

**EVM wallets**
- *"Is 0xabc…1234 at risk of liquidation on Aave or Morpho?"*
- *"Show me all lending positions for 0x…1234"*
- *"What happens to 0x…1234's positions if ETH drops 40%?"*
- *"Which of these wallets is closest to liquidation: 0xaaa…, 0xbbb…?"*
- *"Does 0x…1234 have any Moonwell borrows on Base?"*

**Solana wallets**
- *"Check Kamino and Marginfi health for 7xKX…AbCD"*
- *"Is this Solana wallet at risk of liquidation on Kamino?"*

**TON wallets**
- *"What's the EVAA health factor for EQD…wxYZ?"*

**Stablecoins**
- *"Is USDC currently depegged?"*
- *"Which stablecoins are showing depeg signals right now?"*

**Markets**
- *"What's the USDC utilization on Aave Ethereum — is there room to borrow?"*
- *"Are any Aave markets frozen or above optimal utilization?"*

---

## Authentication

No API key required. Rate limited per IP.

## Source

Built on [alertio.io](https://alertio.io) — real-time DeFi monitoring and alerts across EVM, Solana, and TON.
