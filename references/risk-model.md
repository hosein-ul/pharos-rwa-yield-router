# Risk Scoring Methodology

## Philosophy

Raw APY is misleading. A 12% APY on native token staking is worse for most users than 6% on
US Treasury-backed assets because the underlying can lose significant value. This model adjusts
APY by the quality of the underlying asset backing the yield.

The goal is to answer: **what APY would a risk-neutral investor require to be indifferent
between this yield opportunity and a risk-free alternative?**

---

## Risk Multipliers

| Asset Class | Multiplier | Pharos Protocols | Rationale |
|-------------|-----------|------------------|-----------|
| `government_treasury` | **0.95** | (none yet on Pharos) | US/AAA sovereign debt. Near-zero default risk. |
| `investment_grade` | **0.85** | **TermMax** (fixed-rate) | Predictable returns, audited, lockup until maturity. |
| `real_estate` | **0.80** | **AquaFlux**, **Zona** | Tokenized property/RWA. Tranche or collateral structure adds risk. |
| `crypto_lending` | **0.70** | **Morpho Blue** | Crypto-collateralized P2P lending. Liquidation risk in volatility. |
| `private_credit` | **0.60** | **R25 Axil** (7-day, 6-month) | Emerging-market consumer credit. Curated/audited but defaults possible. |
| `native_staking` | **0.50** | **Native PROS staking**, **Faroo** (when live) | Native token price risk. APY denominated in volatile PROS. |
| `novel_protocol` | **0.40** | Unknown new | No track record. Smart contract + underlying risk. |

---

## Formula

```
risk_adjusted_apy = nominal_apy × risk_multiplier
```

### Example

ELFi T-Bills at 6.8% nominal APY:
```
risk_adjusted_apy = 6.8 × 0.95 = 6.46%
```

Faroo stPHRS at 9.1% nominal APY:
```
risk_adjusted_apy = 9.1 × 0.50 = 4.55%
```

Even though Faroo has higher nominal APY, ELFi T-Bills rank higher on risk-adjusted basis.

---

## Risk Tolerance Filters

Apply before ranking to exclude protocols beyond user's risk appetite:

| User Preference | Include Asset Classes |
|-----------------|----------------------|
| `conservative` | `government_treasury`, `investment_grade` |
| `moderate` | above + `real_estate`, `crypto_lending` |
| `aggressive` | all classes including `native_staking`, `private_credit` |

---

## Additional Risk Flags

These are warnings shown alongside the APY table — they do not modify the multiplier,
but the agent should always call them out explicitly.

| Flag | Condition | Label |
|------|-----------|-------|
| Liquidity Risk | Protocol utilization > 90% | ⚠️ High utilization — withdrawals may be slow |
| New Protocol | Protocol deployed < 30 days ago | ⚠️ New protocol — limited track record |
| Low Liquidity | TVL < $100,000 | ⚠️ Small pool — exit may be difficult |
| Unverified Contract | Source code not verified on explorer | 🔴 Unverified — cannot audit logic |
| Placeholder Address | Protocol not yet configured | ℹ️ Address pending — run `pharos-tx-guardrail` before depositing |

---

## Utilization Rate

For Aave V3 / ELFi-style pools:
```bash
# Total borrowed
cast call $POOL "getTotalVariableDebt(address)(uint256)" $ASSET --rpc-url $RPC
# Total supplied
cast call $POOL "getTotalLiquidity(address)(uint256)" $ASSET --rpc-url $RPC

utilization = total_debt / total_liquidity
```

Flag if `utilization > 0.90`.

For ERC-4626 vaults (Morpho):
```bash
ASSETS=$(cast call $VAULT "totalAssets()(uint256)" --rpc-url $RPC)
IDLE=$(cast call $VAULT "idle()(uint256)" --rpc-url $RPC)
deployed = ASSETS - IDLE
utilization = deployed / ASSETS
```

---

## Projected Return Formula

```
projected_return = principal × (nominal_apy / 100) × (days / 365)
```

Simple interest approximation — sufficient for display. Use nominal APY (not risk-adjusted)
for the actual return number, since risk-adjusted APY is a ranking tool, not an expected return.

---

## Protocol Classification

When adding a new protocol, assign an asset class using these criteria:

1. **What is the yield-generating activity?** (lending, staking, RWA, trading fees)
2. **What is the underlying collateral?** (crypto, real estate, government bonds, nothing)
3. **How liquid is it?** (instant withdrawal, lock-up, or redemption queue)
4. **Is the contract verified and audited?**

If uncertain, use the more conservative multiplier.
