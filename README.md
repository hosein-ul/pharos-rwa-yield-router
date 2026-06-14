# pharos-rwa-yield-router

**RWA Yield Intelligence Router for Pharos Network AI Agents.**

Scans Pharos's yield ecosystem, reads live APY from on-chain protocols, applies risk-adjusted multipliers per asset class, and returns a ranked recommendation for idle capital deployment — all read-only, no transactions executed.

[![Pharos Network](https://img.shields.io/badge/Pharos-Mainnet%201672-6B4FFF?style=flat-square)](https://pharos.xyz)
[![Hackathon](https://img.shields.io/badge/AI%20Agent%20Carnival-Phase%201-00C2A8?style=flat-square)](https://dorahacks.io/hackathon/pharos-phase1/)
[![TVL](https://img.shields.io/badge/Active%20TVL-%247.7M%2B%20USDC-green?style=flat-square)](https://app.r25.xyz)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)

---

## What it does

When a user asks "where should I put my USDC on Pharos?", this skill:

1. Reads live APY from each active protocol via `cast call`
2. Multiplies by a risk-adjustment factor per asset class
3. Filters by user's risk tolerance, min deposit, and lockup needs
4. Returns a ranked table + best recommendation + projected return

```
🔍 Pharos Yield Scan — 2026-06-13 (block 10,073,874, mainnet 1672)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ACTIVE PROTOCOLS:
Rank  Protocol              Class            APY    Risk-Adj  TVL       Lockup
 #1   R25 Axil 7-day        private credit   8.5%   5.10%     $892K     7d queue
 #2   R25 Axil 6-month      private credit   11.2%  6.72%     $6.83M    180d
 #3   Zona USDC supply      real estate      3.6%   2.88%     $620K     none

DEPLOYED, NOT YET ACTIVE:
  • Morpho Blue — 0 markets created yet
  • TermMax — 0 vaults created yet
  • AquaFlux — ABI gap in public docs
  • Faroo mainnet — coming soon
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⭐ Best for $5,000 USDC (balanced): R25 Axil 7-day @ 5.10% risk-adjusted
   Projected 30d return: ~$34.93 USDC
   ⚠️ Lockup: 7-day withdrawal queue
```

---

## Risk model

Each asset class carries a risk multiplier that discounts the nominal APY:

| Asset Class | Multiplier | Protocols |
|-------------|-----------|-----------|
| investment_grade | **0.85** | TermMax (when active) |
| real_estate | **0.80** | AquaFlux, Zona |
| crypto_lending | **0.70** | Morpho Blue |
| private_credit | **0.60** | R25 Axil vaults |
| native_staking | **0.50** | Native PROS staking, Faroo |

`risk_adjusted_apy = nominal_apy × multiplier`

---

## Protocol status (verified 2026-06-13 via eth_getCode + eth_getLogs)

| Protocol | Address | Status | Readable? |
|----------|---------|--------|-----------|
| R25 Axil 7-day | `0x1c2bc8b5...569268` | ✅ $892K TVL | ✅ Full ERC-4626 |
| R25 Axil 6-month | `0xee26bb09...e190b` | ✅ $6.83M TVL | ✅ Full ERC-4626 |
| Zona Pool | `0xda464e68...1372a` | ✅ Aave V3 reserves active | ✅ Standard |
| Morpho Blue | `0x18573fA1...4Efb` | ⏳ 0 markets | ❌ Not yet |
| TermMax Factory | `0xEDC206E6...8395` | ⏳ 0 vaults | ❌ Not yet |
| AquaFlux Core | `0x0da98a84...127` | ⏳ ABI gap | ❌ Not yet |
| Faroo (testnet) | `0xc9A0B63d...A12` | ✅ Atlantic testnet only | ✅ |

> ⚠️ **Pharos RPC quirk**: `eth_getLogs` is limited to 1000 blocks per query. Chunk historical scans accordingly.

> ⚠️ **Morpho address**: Pharos uses `0x18573fA1...` — NOT the canonical `0xBBBB...FFCb` used on other chains.

---

## Installation

### Pharos Skill Engine / Claude Code

```bash
git clone https://github.com/hosein-ul/pharos-rwa-yield-router \
  ~/.claude/skills/pharos-rwa-yield-router
```

### Anvita Flow

Submit `https://github.com/hosein-ul/pharos-rwa-yield-router` in the Skill Hub.

### Manual / Any AI Agent

```bash
git clone https://github.com/hosein-ul/pharos-rwa-yield-router
# Point your agent to pharos-rwa-yield-router/SKILL.md as the entry point
```

**Prerequisite — Foundry:**

```bash
curl -L https://foundry.paradigm.xyz | bash && foundryup
cast --version
```

---

## File structure

```
pharos-rwa-yield-router/
├── SKILL.md                  ← Agent entry point
├── assets/
│   ├── networks.json         ← RPC URLs and chain IDs
│   ├── tokens.json           ← ERC-20 token addresses
│   ├── oracles.json          ← Chainlink Push Engine feeds
│   ├── protocols.json        ← Per-protocol addresses + executability status
│   └── ecosystem.json        ← Full Pharos Port ecosystem registry
├── references/
│   ├── yield-scan.md         ← Per-protocol cast commands (R25, Zona, Morpho, TermMax...)
│   └── risk-model.md         ← Risk multiplier methodology + formulas
└── evals/
    └── evals.json            ← 4 test scenarios
```

---

## Live read commands (quick start)

```bash
RPC="https://rpc.pharos.xyz"

# R25 Axil 7-day vault TVL
cast call 0x1c2bc8b553d9a7e61f7531a3a4bf2162f4569268 \
  "totalAssets()(uint256)" --rpc-url $RPC

# Zona USDC supply APY (Aave V3 — field index 2 = currentLiquidityRate in RAY)
cast call 0xda464e68208A3083Eb65FE5c522a72AeD1C1372a \
  "getReserveData(address)((uint256,uint128,uint128,uint128,uint128,uint128,uint40,uint16,address,address,address,address,uint128,uint128,uint128))" \
  0xc879c018db60520f4355c26ed1a6d572cdac1815 --rpc-url $RPC

# PROS/USD price from Chainlink (18 decimals)
cast call 0x9356C87a48F913d11C87a0d4b8cD16CD04624BF3 \
  "latestAnswer()(int256)" --rpc-url $RPC
```

---

## Networks

| Network | Chain ID | RPC |
|---------|----------|-----|
| Pacific Mainnet | 1672 | `https://rpc.pharos.xyz` |
| Atlantic Testnet | 688689 | `https://atlantic.dplabs-internal.com` |

Default: mainnet.

---

## This skill is part of the Pharos AI Agent Carnival — Phase 1

[Hackathon page](https://dorahacks.io/hackathon/pharos-phase1/) · [Pharos Network](https://pharos.xyz) · [Companion skill: pharos-tx-guardrail](https://github.com/hosein-ul/pharos-tx-guardrail)
