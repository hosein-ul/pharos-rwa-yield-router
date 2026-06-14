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
version: 0.3.0
requires:
  anyBins:
    - cast
---

# Pharos RWA Yield Intelligence Router (v0.3.0)

Scans Pharos's yield ecosystem and returns a ranked, risk-adjusted comparison so agents can
recommend optimal capital deployment. All data is read live from on-chain — no stale estimates,
no fabricated APYs. **The skill is HONEST about what's working today** — Pharos mainnet is
new, and not all listed protocols have active markets yet.

## What Changed in v0.3.0

Following deep activity verification (2026-06-13):
- **Reality check**: most "deployed" Pharos protocols have zero on-chain activity
- **R25 Axil**: LIVE — verified $847K USDC TVL on the 7-day vault
- **Zona**: LIVE — Aave V3 reserves are active, USDC supply rate is readable
- **Morpho Blue**: deployed but ZERO CreateMarket events — no active markets yet
- **TermMax**: deployed but ZERO factory events — no vaults exist yet
- **AquaFlux**: deployed but ABI for APY view functions not in public docs
- **Native PROS staking**: precompile not at standard slots (0x1000/0x1001/0x0800)
- **Faroo**: mainnet "coming soon"; testnet stPROS works

## What Changed in v0.2.0

- **Removed ELFi** — not deployed on Pharos at all (Arbitrum & Base only)
- **Removed fake Morpho address** — canonical `0xBBBB...` is NOT used; Pharos has a distinct
  Morpho deployment at `0x18573fA1...`
- **Default network → `mainnet`** — capital lives there

## Prerequisites

1. **Install Foundry**:
   ```bash
   which cast || (curl -L https://foundry.paradigm.xyz | bash && foundryup)
   ```
2. **Network config** — read `assets/networks.json`. Default: `mainnet` (chain 1672).
3. **Protocol catalog** — read `assets/protocols.json` (already verified via eth_getCode).
4. **Full ecosystem registry** — `assets/ecosystem.json` (cross-reference for context).
5. No private key required — read-only skill.

## Pharos Mainnet Protocols — Honest Status

| Protocol | Asset Class | Risk Mult | Status | Agent Can Read? |
|----------|-------------|-----------|--------|-----------------|
| **R25 — Axil 7-day** | private_credit | 0.60 | ✅ ACTIVE ($847K TVL) | ✅ Yes — full ERC-4626 |
| **R25 — Axil 6-month** | private_credit | 0.60 | ✅ ACTIVE | ✅ Yes — full ERC-4626 |
| **Zona** | real_estate | 0.80 | ✅ ACTIVE | ✅ Yes — Aave V3 standard |
| **Faroo (testnet)** | native_staking | 0.50 | ✅ ACTIVE (testnet only) | ✅ Yes — ERC-4626 |
| **Faroswap** | lp_yield | 0.65 | ✅ ACTIVE | 🟡 Swap quotes via DODO API |
| **Morpho Blue** | crypto_lending | 0.70 | ⏳ Deployed, NO markets | ❌ Nothing to read yet |
| **TermMax** | investment_grade | 0.85 | ⏳ Deployed, NO vaults | ❌ Nothing to read yet |
| **AquaFlux** | real_estate | 0.80 | ⏳ Deployed, ABI gap | ❌ ABI not in public docs |
| **Native PROS Staking** | native_staking | 0.50 | 🟡 Active, no precompile addr | ❌ Need Pharos staking docs |
| Faroo (mainnet) | native_staking | 0.50 | ⏳ Coming Soon | ❌ Empty config |

Full per-protocol detail in `assets/protocols.json`.

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
🔍 Pharos Yield Scan — <TIMESTAMP> (block <BLOCK>, mainnet 1672)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ACTIVE PROTOCOLS (live data):
Rank  Protocol              Class            APY    Risk-Adj  TVL      Min   Lockup
 #1   R25 Axil 7d           private credit   8.5%   5.10%     $847K    $1    7d queue
 #2   R25 Axil 6m           private credit   11.2%  6.72%     ...      $1    180d
 #3   Zona USDC supply      real estate      3.6%   2.88%     $620K    $1    none
 #4   Faroo stPROS (test)   native staking   N/A    —         testnet  —     —

DEPLOYED BUT NOT YET ACTIVE:
  • Morpho Blue — 0 markets created yet
  • TermMax — 0 vaults created yet
  • AquaFlux — ABI not publicly documented yet
  • Native PROS staking — precompile address not published yet
  • Faroo (mainnet) — "Coming Soon" per app config
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⭐ Best for <AMOUNT> <ASSET>: <PROTOCOL> @ <RISK_ADJ_APY>%
   Reasoning: <one sentence>
   Projected <DAYS>d return: ~<RETURN> <ASSET>
   ⚠️ Lockup: <describe>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Data: block <BLOCK> | 4 protocols scanned | 5 not yet active
```

**Always show both sections** — users need to know the full landscape, including what's coming.

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
