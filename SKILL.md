---
name: pharos-rwa-yield-router
description: >
  Scans all yield-bearing protocols on Pharos Network (ELFi RWA lending, Morpho isolated vaults,
  Faroo liquid staking) and returns a risk-adjusted APY comparison ranked by suitability for the
  user's capital and risk tolerance. Reads live on-chain data via cast call — never executes
  transactions. Also reads live token prices from Chainlink oracles (PROS/USD, ETH/USD, BTC/USD,
  USDC/USD) to show USD-denominated projected returns.
  Invoke whenever a user asks: "where should I put my USDC on Pharos?", "which protocol has the
  best yield?", "compare ELFi vs Morpho", "what's the APY on Pharos?", "safest yield on Pharos",
  "best passive income on Pharos", "how much would I earn in 90 days?", "show me DeFi opportunities
  on Pharos", or any question about yield, APY, staking, or capital deployment on Pharos Network.
  This skill is read-only and safe to call at any time without confirmation.
version: 0.1.0
requires:
  anyBins:
    - cast
---

# Pharos RWA Yield Intelligence Router

Scans Pharos's entire yield ecosystem and returns a ranked, risk-adjusted comparison so agents
can recommend optimal capital deployment. All data is read live from the chain — no stale estimates.

## Prerequisites

1. **Install Foundry** — check with `which cast` or `cast --version`.
   If missing:
   ```bash
   curl -L https://foundry.paradigm.xyz | bash
   source ~/.zshenv && foundryup
   ```
2. **Network config** — read `assets/networks.json`. Default: `atlantic-testnet`.
   ```bash
   RPC=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
   ```
3. **Protocol addresses** — read `assets/protocols.json`.
   Addresses marked `PLACEHOLDER_*` must be filled from Pharos Discord `#deployed-contracts`
   before those protocols can be queried. Skip unavailable protocols gracefully.
4. No private key required — this skill is **read-only**.

## Capability Index

| User Need | Capability | Detailed Instructions |
|-----------|-----------|----------------------|
| Full yield scan across all protocols | Read APY from ELFi, Morpho, Faroo, sort by risk-adjusted APY | → `references/yield-scan.md#full-scan` |
| Get current PROS/ETH/BTC price in USD | Chainlink oracle `latestAnswer()` | → `references/yield-scan.md#price-lookup` |
| Read ELFi lending APY | `cast call` on Aave V3-style pool | → `references/yield-scan.md#elfi-apy` |
| Read Morpho vault APY | `cast call` on ERC-4626 vault | → `references/yield-scan.md#morpho-apy` |
| Read Faroo staking APY | `cast call` on liquid staking contract | → `references/yield-scan.md#faroo-apy` |
| Calculate projected return for amount + days | Simple interest formula | → `references/yield-scan.md#projected-return` |
| Filter by risk tolerance (safe/moderate/aggressive) | Risk multiplier filter | → `references/risk-model.md` |
| Recommend best protocol for user's amount | Rank by risk-adjusted APY, filter by min deposit | → `references/yield-scan.md#full-scan` |

## Risk-Adjusted APY

Raw APY alone is misleading. A 12% yield on private credit is worse than 6% on government
treasuries for most users because the underlying can default. This skill adjusts APY by the
quality of the underlying asset. Full methodology in `references/risk-model.md`.

Quick reference:

| Asset Class | Multiplier | Example |
|-------------|-----------|---------|
| Government Treasury | 0.95 | ELFi T-Bills |
| Investment Grade Bonds | 0.85 | ELFi IG |
| Real Estate | 0.80 | ELFi RE pools |
| Crypto-Backed Lending | 0.70 | Morpho USDC |
| Native Staking | 0.50 | Faroo stPHRS |

`risk_adjusted_apy = nominal_apy × multiplier`

## Output Format

```
🔍 Pharos Yield Scan — <TIMESTAMP>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Rank  Protocol           Class               APY     Risk-Adj  TVL       Min
──────────────────────────────────────────────────────────────────────────
 #1   ELFi - T-Bills     Gov. Treasury      6.8%    6.46%     $12.4M    $100
 #2   Morpho USDC        Crypto Lending     5.2%    3.64%     $8.1M     $1
 #3   Faroo stPHRS       Native Staking     9.1%    4.55%     $5.3M     $10
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⭐ Best for <AMOUNT> <ASSET>: <PROTOCOL> @ <RISK_ADJ_APY>% risk-adjusted APY
   Reasoning: <one sentence>
   Projected <DAYS>d return: ~<AMOUNT * APY * DAYS/365> <ASSET>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Data: block <BLOCK> | <N> protocols scanned | <M> unavailable
```

## Decision Flow for Agents

1. Extract from user's message: **amount**, **asset** (default: USDC), **risk tolerance**
   (conservative / moderate / aggressive), **liquidity need** (immediate = no lockup).
2. Run the full scan: `references/yield-scan.md#full-scan`.
3. Apply risk tolerance filter: conservative → treasury/investment_grade only; moderate → include
   crypto_lending; aggressive → include native staking.
4. Filter out protocols with `min_deposit > user_amount`.
5. Rank by risk-adjusted APY.
6. Present the ranked table and the top recommendation with projected return.
7. Offer next steps:
   - "Want me to deposit to [protocol]?" → call `pharos-tx-guardrail` first, then execute.
   - "Want to compare specific protocols?" → run targeted queries.

## Important Rules

- This skill is **read-only**. It never executes deposits or transactions.
- If ALL protocol RPC calls fail → report "RPC connectivity issue", never fabricate APY data.
- If a protocol address is `PLACEHOLDER_*` → skip it gracefully, note it in output.
- APY data is point-in-time. Always show the block number so the user knows data freshness.
- Utilization > 90% → flag as liquidity risk regardless of APY.
- Never recommend a protocol whose address is unverified (treat as unknown risk class).

## Error Handling

| Error | Handling |
|-------|----------|
| Protocol address is PLACEHOLDER | Skip protocol, note "address not yet configured" |
| `cast call` fails for a protocol | Skip that protocol, note "data unavailable" |
| All protocols fail | Report RPC connectivity issue, do not guess APYs |
| ERC-4626 APY needs 2 snapshots | Report current price-per-share, note APY requires historical data |
| `cast` not installed | Show Foundry installation instructions and stop |
