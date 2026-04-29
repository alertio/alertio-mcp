# Alertio MCP Server

Real-time DeFi lending risk data for AI agents. Scan any Ethereum wallet across Aave V3, Aave V4, Spark, Morpho Blue, and Euler V2 — no account required.

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

Scans a wallet address across all supported lending protocols and returns every active borrowing position with health factors and liquidation risk.

**Input**

| Parameter | Type   | Description                        |
|-----------|--------|------------------------------------|
| `wallet`  | string | Ethereum address (`0x...`, 42 chars) |

**Protocols scanned**

- Aave V3 — Ethereum, Base, Arbitrum, Polygon, Optimism, Avalanche, Scroll
- Aave V4 — Ethereum mainnet (Core, Prime, Plus hubs)
- Spark — Ethereum mainnet
- Morpho Blue — all markets
- Euler V2 — all vaults

**Example output**

```
Lending positions for 0xabc...1234

AAVE V3
  ethereum: HF 2.45 · $18K collateral · $7K debt [SAFE]
  base: HF 1.28 · $5K collateral · $3.8K debt [DANGER]

SPARK
  No active positions.

MORPHO BLUE
  wstETH/USDC: HF 1.84 · $42K collateral · $22K debt [AT RISK]

EULER V2
  No active positions.

Total active positions: 3
```

**Risk levels**

| Label       | Health factor        |
|-------------|----------------------|
| `SAFE`      | HF ≥ 2.0             |
| `AT RISK`   | 1.3 ≤ HF < 2.0       |
| `DANGER`    | 1.0 ≤ HF < 1.3       |
| `LIQUIDATED`| HF < 1.0             |

---

### `simulate_liquidation`

Simulates the impact of a collateral price drop on all active positions for a wallet. Identifies which positions would be liquidated and which move into a higher risk tier.

**Formula:** `newHF = currentHF × (1 − drop%)`

This assumes all collateral moves uniformly with the price drop. Results are conservative when collateral is a mix of volatile and stable assets; see the `?` tooltip in the Alertio dashboard for nuances.

**Input**

| Parameter             | Type   | Description                                    |
|-----------------------|--------|------------------------------------------------|
| `wallet`              | string | Ethereum address (`0x...`, 42 chars)           |
| `collateral_drop_pct` | number | Collateral price drop percentage (1–80)        |

**Example output**

```
Liquidation simulation for 0xabc...1234
Scenario: collateral drops 30%

  Aave V3 (ethereum): 2.45 → 1.72 [AT RISK] ← was SAFE
  Aave V3 (base): 1.28 → 0.90 [LIQUIDATED] ← was DANGER
  Morpho Blue wstETH/USDC: 1.84 → 1.29 [DANGER] ← was AT RISK

△ 2 position(s) at risk at -30%
⚠ 1 position(s) LIQUIDATED at -30%
```

## Example agent queries

- *"Is 0xabc...1234 at risk of liquidation?"*
- *"What are all the DeFi lending positions for 0x...?"*
- *"What happens to 0x...1234's positions if ETH drops 40%?"*
- *"Which of these wallets is closest to liquidation: 0xaaa..., 0xbbb...?"*

## Authentication

No API key required. Rate limited per IP.

## Source

Built on [alertio.io](https://alertio.io) — DeFi health factor monitoring and alerts.  
GitHub: [github.com/naskata/alertio](https://github.com/naskata/alertio)
