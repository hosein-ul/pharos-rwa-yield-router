---
name: pharos-rwa-yield-router
description: >
  Scans all yield-bearing protocols on Pharos Pacific Mainnet (chain 1672) and returns a
  risk-adjusted APY comparison ranked by suitability for the user's capital and risk tolerance.
  Covers: Morpho Blue (P2P lending), TermMax (fixed-rate lending), AquaFlux (RWA structured
  yield with tri-token tranching), Zona (Aave V3 fork for RWA collateral), R25 Axil Consumer
  Credit Vaults (7-day and 6-month tokenized credit), Native PROS staking, and Faroo liquid
  staking (when live on mainnet — currently testnet-only).
  Reads live on-chain data via cast call — never executes transactions. Also reads live token
  prices from Chainlink Push Engine oracles (PROS/USD, ETH/USD, BTC/USD, USDC/USD) to show
  USD-denominated projected returns.
  Invoke whenever a user asks: "where should I put my USDC on Pharos?", "which protocol has
  the best yield?", "compare Morpho vs TermMax vs Zona", "what's the APY on Pharos?", "safest
  yield on Pharos?", "best RWA yield on Pharos?", "fixed rate vs variable on Pharos?", "how
  much would I earn in 90 days?", "Axil consumer credit", "AquaFlux tranches", or any question
  about yield, APY, staking, or capital deployment on Pharos.
  This skill is read-only and safe to call at any time.
version: 0.2.0
requires:
  anyBins:
    - cast
---

# Pharos RWA Yield Intelligence Router (v0.2.0)

Scans Pharos's entire yield ecosystem and returns a ranked, risk-adjusted comparison so agents
can recommend optimal capital deployment. All data is read live from on-chain — no stale estimates,
no fabricated APYs.

## What Changed in v0.2.0

Following ecosystem verification (2026-06-13):
- **Removed ELFi** — not deployed on Pharos at all (Arbitrum & Base only)
- **Removed fake Morpho address** — canonical 0xBBBB... is NOT used; Pharos has a distinct
  Morpho deployment at `0x18573fA1...`
- **Added 5 verified mainnet protocols**: Morpho, TermMax, AquaFlux, Zona, R25 (Axil Credit)
- **Default network changed to `mainnet`** — that's where actual capital lives. Atlantic
  testnet is kept only for testing (Faroo testnet stPROS).

## Prerequisites

1. **Install Foundry**:
   ```bash
   which cast || (curl -L https://foundry.paradigm.xyz | bash && foundryup)
   ```
2. **Network config** — read `assets/networks.json`. Default: `mainnet` (chain 1672).
3. **Protocol catalog** — read `assets/protocols.json` (already verified via eth_getCode).
4. **Full ecosystem registry** — `assets/ecosystem.json` (cross-reference for context).
5. No private key required — read-only skill.

## Verified Pharos Mainnet Protocols

| Protocol | Type | Asset Class | Risk Mult | Status |
|----------|------|-------------|-----------|--------|
| **Morpho Blue** | P2P lending | crypto_lending | 0.70 | ✅ DEPLOYED |
| **TermMax** | Fixed-rate lending | investment_grade | 0.85 | ✅ DEPLOYED |
| **AquaFlux** | RWA structured (tranched) | real_estate | 0.80 | ✅ DEPLOYED |
| **Zona** | Aave V3 fork (RWA collateral) | real_estate | 0.80 | ✅ DEPLOYED |
| **R25 — Axil 7-day** | Consumer credit vault | private_credit | 0.60 | ✅ DEPLOYED |
| **R25 — Axil 6-month** | Consumer credit vault | private_credit | 0.60 | ✅ DEPLOYED |
| **Native PROS Staking** | L1 staking | native_staking | 0.50 | ✅ DEPLOYED |
| Faroo Liquid Staking | Liquid staking | native_staking | 0.50 | ⏳ TESTNET ONLY |

## Capability Index

| User Need | Capability | Reference |
|-----------|-----------|-----------|
| Scan all yield protocols / "best APY" | Read all protocols, rank by risk-adjusted APY | → `references/yield-scan.md#full-scan` |
| Get live token price in USD | Chainlink Push Engine `latestAnswer()` | → `references/yield-scan.md#price-lookup` |
| Read Morpho Blue market APY | Iterate markets via `market(bytes32)` | → `references/yield-scan.md#morpho-apy` |
| Read TermMax fixed-rate APY | `cast call MarketViewer` | → `references/yield-scan.md#termmax-apy` |
| Read AquaFlux tranche yields | Per-token preview functions | → `references/yield-scan.md#aquaflux-apy` |
| Read Zona supply APY | Aave V3 `getReserveData()` → liquidityRate | → `references/yield-scan.md#zona-apy` |
| Read R25 Axil vault APY | ERC-4626 `previewRedeem()` snapshot diff | → `references/yield-scan.md#r25-apy` |
| Compare RWA strategies for $X capital | Apply risk filter + min deposit + project return | → `references/yield-scan.md#full-scan` |
| Filter by risk tolerance | conservative / balanced / aggressive | → `references/risk-model.md` |
| Calculate projected return | Simple interest formula | → `references/yield-scan.md#projected-return` |

## Output Format

```
🔍 Pharos Yield Scan — <TIMESTAMP> (block <BLOCK>, mainnet)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Rank  Protocol              Type          Class            APY    Risk-Adj  TVL      Min
─────────────────────────────────────────────────────────────────────
 #1   TermMax — 30d fixed   fixed-rate    inv. grade      5.4%   4.59%     $4.2M    $10
 #2   Morpho — USDC/WPROS   p2p lending   crypto lending  4.1%   2.87%     $3.8M    $1
 #3   R25 — Axil 7d         consumer cr.  private credit  9.2%   5.52%     $1.1M    $1
 #4   AquaFlux — Y token    structured    real estate     8.0%   6.40%     $850k    $50
 #5   Zona — USDC supply    aave fork     real estate     3.6%   2.88%     $620k    $1
 #6   Native PROS staking   l1 staking    native staking  6.5%   3.25%     —        —
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⭐ Best for <AMOUNT> <ASSET>: <PROTOCOL> @ <RISK_ADJ_APY>%
   Reasoning: <one sentence>
   Projected <DAYS>d return: ~<RETURN> <ASSET>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Decision Flow

1. Extract from user: **amount**, **asset** (USDC/PROS/USDT/WETH), **risk tolerance**, **time horizon**.
2. Run the scan against mainnet (chain 1672).
3. Apply risk filter:
   - `conservative` → investment_grade, native_staking only
   - `balanced` → above + crypto_lending, real_estate
   - `aggressive` → all classes including private_credit
4. Filter by min deposit and lockup matching user's needs.
5. Rank by risk-adjusted APY.
6. Present the table + top recommendation + projected return.
7. Offer next steps: deposit via `pharos-tx-guardrail` first, then execute.

## Critical Rules

- This skill is **read-only**.
- If a protocol's read fails, mark "data unavailable" — never fabricate APY.
- Always show block number for data freshness.
- Native PROS staking has no ERC-20 contract — handle differently (validator delegation flow).
- R25 vaults have lockup — always disclose to user before recommending.
- Faroo on mainnet is "coming soon"; flag clearly if user asks about Faroo.

## References

- Full protocol catalog & status: `assets/protocols.json`
- Complete Pharos ecosystem context: `assets/ecosystem.json`
- Risk model methodology: `references/risk-model.md`
- Per-protocol cast commands: `references/yield-scan.md`
