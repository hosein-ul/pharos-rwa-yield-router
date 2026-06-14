# Yield Scan — Reference (v0.3.0)

All operations read-only. Default network: Pacific Mainnet (chain 1672).

> **⚠️ Pharos RPC Quirk**: `eth_getLogs` is limited to **1000 blocks per query**. To enumerate
> historical events, chunk queries into 1000-block windows from the contract's deployment block.

> **Setup**:
> ```bash
> RPC=$(jq -r '.networks[] | select(.name=="mainnet") | .rpcUrl' assets/networks.json)
> BLOCK=$(cast block-number --rpc-url $RPC)
> USDC=$(jq -r '."mainnet"[] | select(.symbol=="USDC") | .address' assets/tokens.json)
> ```

---

## Table of Contents
1. [Full Scan](#full-scan)
2. [Price Lookup](#price-lookup)
3. [R25 Axil APY](#r25-apy) — ✅ FULLY WORKING
4. [Zona APY](#zona-apy) — ✅ FULLY WORKING
5. [Faroo (testnet) APY](#faroo-testnet-apy) — ✅ FULLY WORKING
6. [Morpho APY](#morpho-apy) — ⏳ NO ACTIVE MARKETS YET
7. [TermMax APY](#termmax-apy) — ⏳ NO ACTIVE VAULTS YET
8. [AquaFlux](#aquaflux) — ❌ ABI NOT PUBLISHED
9. [Faroswap LP APY](#faroswap-apy) — 🟡 API ONLY
10. [Projected Return](#projected-return)

---

## Full Scan

```
1. Read assets/protocols.json — focus on entries in _summary_for_agents.fully_executable_now
2. For each fully-executable protocol:
   a. Read APY using the documented cast call
   b. Read TVL
   c. risk_adjusted_apy = nominal_apy × risk_multiplier
3. Read USD prices via Chainlink (see Price Lookup)
4. Apply user filters
5. Rank and present
6. For protocols in _summary_for_agents.partial_executable / not_yet_usable / abi_gap:
   append a "Coming Soon" or "Not Yet Active" footnote so the user knows the FULL Pharos
   yield landscape — but DO NOT include them in the ranked APY table.
```

---

## Price Lookup

```bash
FEED=$(jq -r '."mainnet".feeds."<PAIR>"' assets/oracles.json)
cast call $FEED "latestAnswer()(int256)" --rpc-url $RPC
```

| Pair | Feed Address |
|------|-------------|
| PROS/USD | `0x9356C87a48F913d11C87a0d4b8cD16CD04624BF3` |
| ETH/USD  | `0x092ff0175Be8B2e83Ca5740d3EB13C6225901fa7` |
| BTC/USD  | `0x6BFcd14b164de6c8C4dA2d065d511055A589EB20` |
| USDC/USD | `0x8d08eA83A55ad1e805b5660F5eC76C99C6aF5eaf` |

Parse: `price_usd = raw_int256 / 1e18`

---

## R25 APY ✅

R25 vaults are ERC-4626. APY = change in `previewRedeem(1e18)` over time.

### Live verification (2026-06-13)
```
totalAssets() on VRPCW = 0x000000000000000000000000c54be08e7d
                      = 847,884,484,733 raw USDC
                      = $847,884.48 USD TVL
```

### Read commands

```bash
VAULT_7D="0x1c2bc8b553d9a7e61f7531a3a4bf2162f4569268"
VAULT_6M="0xee26bb0989691735c997dfdc49a4a607f75e190b"

# Current price-per-share (returns USDC raw, 6 decimals — but result is uint256)
cast call $VAULT_7D "previewRedeem(uint256)(uint256)" 1000000000000000000 --rpc-url $RPC

# Total value locked
cast call $VAULT_7D "totalAssets()(uint256)" --rpc-url $RPC
# Convert: TVL_USDC = result / 1e6

# User balance
cast call $VAULT_7D "balanceOf(address)(uint256)" $WALLET --rpc-url $RPC
cast call $VAULT_7D "convertToAssets(uint256)(uint256)" $SHARES --rpc-url $RPC
```

### APY calculation

```
pps_now = previewRedeem(1e18) / 1e18  # shares→USDC ratio
# Save snapshot to disk for next time
# Next call:
pps_then = saved_snapshot
seconds_elapsed = block_timestamp_now - block_timestamp_then
apy = (pps_now / pps_then) ^ (31536000 / seconds_elapsed) - 1
apy_pct = apy * 100
```

### No-snapshot fallback

If this is the first call (no saved snapshot), report **current TVL** and the published R25 APY:

```bash
curl -s "https://app.r25.xyz/api/vaults" 2>/dev/null
```

Or direct user to R25 app for published yield.

### Lockup disclosure (MANDATORY)

Always tell the user:
- **VRPCW (7-day)**: withdrawals queue for 7 days after request
- **VRPCS (6-month)**: 180-day lockup; not for short-term capital

---

## Zona APY ✅

Aave V3 fork. Standard reading pattern.

### Live verification (2026-06-13)
```
getReserveData(USDC) returned 962-char tuple with non-zero values across all fields.
```

### Read commands

```bash
POOL="0xda464e68208A3083Eb65FE5c522a72AeD1C1372a"

cast call $POOL \
  "getReserveData(address)((uint256,uint128,uint128,uint128,uint128,uint128,uint40,uint16,address,address,address,address,uint128,uint128,uint128))" \
  $USDC --rpc-url $RPC
```

### Tuple field reference (Aave V3 ReserveData)

| Index | Field | Type |
|-------|-------|------|
| 0 | configuration | uint256 |
| 1 | liquidityIndex | uint128 |
| 2 | currentLiquidityRate | uint128 ← **supply APY in RAY** |
| 3 | variableBorrowIndex | uint128 |
| 4 | currentVariableBorrowRate | uint128 |
| 5 | currentStableBorrowRate | uint128 |
| 6 | lastUpdateTimestamp | uint40 |
| 7 | id | uint16 |
| 8 | aTokenAddress | address |
| 9 | stableDebtTokenAddress | address |
| 10 | variableDebtTokenAddress | address |
| 11 | interestRateStrategyAddress | address |
| 12 | accruedToTreasury | uint128 |
| 13 | unbacked | uint128 |
| 14 | isolationModeTotalDebt | uint128 |

### APY conversion

`currentLiquidityRate` is in RAY (1e27), represents rate-per-second × 1e9.

```
rate_per_sec = liquidityRate / 1e27
apy = (1 + rate_per_sec)^31536000 - 1
apy_pct = apy * 100
```

### Get aToken TVL

```bash
ATOKEN=$(cast call $POOL "getReserveData(address)((...))" $USDC --rpc-url $RPC | grep -oE '0x[a-fA-F0-9]{40}' | head -1)
cast call $ATOKEN "totalSupply()(uint256)" --rpc-url $RPC
# Convert: TVL_USDC = result / 1e6
```

---

## Faroo (testnet) APY ✅

Atlantic testnet only. ERC-4626 style.

```bash
RPC_TESTNET=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
FAROO="0xc9A0B63d91c2A808dD631d031f037944fedDaA12"

cast call $FAROO "totalAssets()(uint256)"  --rpc-url $RPC_TESTNET
cast call $FAROO "totalSupply()(uint256)" --rpc-url $RPC_TESTNET
cast call $FAROO "decimals()(uint8)"      --rpc-url $RPC_TESTNET
```

Snapshot `totalAssets/totalSupply` ratio for APY computation.

---

## Morpho APY ⏳

**Status as of 2026-06-13**: Morpho Blue is deployed at `0x18573fA1...` but has **ZERO events**
— no markets exist yet. The skill should report this clearly and skip Morpho from active ranking.

### When markets are created

```bash
MORPHO="0x18573fA18fd17dDfD790B4a5B5b2977aad3b4Efb"

# Enumerate markets via events (1000-block windows!)
DEPLOY_BLOCK=<get from deployment tx>
END_BLOCK=$(cast block-number --rpc-url $RPC)
for ((from=$DEPLOY_BLOCK; from < $END_BLOCK; from+=1000)); do
  to=$((from + 999))
  cast logs --from-block $from --to-block $to \
    --address $MORPHO \
    "CreateMarket(bytes32,(address,address,address,address,uint256))" \
    --rpc-url $RPC
done

# For each marketId (bytes32), read market state:
cast call $MORPHO \
  "market(bytes32)((uint128,uint128,uint128,uint128,uint128,uint128))" \
  $MARKET_ID --rpc-url $RPC
# Returns: (totalSupplyAssets, totalSupplyShares, totalBorrowAssets, totalBorrowShares, lastUpdate, fee)
```

### APY from utilization

```
utilization = totalBorrowAssets / totalSupplyAssets
# Read borrow rate from IRM
IRM="0xD5E02889C13230458506CC842347c4E62F8cDF3a"
borrow_rate = IRM.borrowRateView(marketParams, marketState) / 1e18  # per-second rate
supply_rate = borrow_rate × utilization × (1 - fee/1e18)
apy = (1 + supply_rate)^31536000 - 1
```

---

## TermMax APY ⏳

**Status as of 2026-06-13**: Factories deployed but **NO factory events** — no vaults/markets
exist yet. Skill should report this clearly.

### When vaults are created

```bash
VIEWER="0x57400bc0486b174972909D05A4097B7067ab761F"
VFAC="0x5316b0d2Ee13C81E243226D6BB93CF29FBf95837"

# Discover vaults via VaultFactoryV2 events (chunk 1000 blocks each)
# Or use TermMax app: https://app.termmax.com (filter Pharos)

# Once you have a vault address:
cast call $VAULT "apr()(uint256)" --rpc-url $RPC
# OR use the viewer for full info:
cast call $VIEWER "getVaultInfo(address)((...))" $VAULT --rpc-url $RPC
```

VaultInfo struct includes `apr` field directly. See:
https://github.com/term-structure/termmax-contract-v2/blob/main/contracts/v2/router/TermMaxViewer.sol

### Per-market read

```bash
cast call $MARKET "name()(string)" --rpc-url $RPC
# Returns e.g. "Termmax Market:USDC-24-Dec"
```

---

## AquaFlux ❌

**Status**: Core contract deployed, but **view functions for APY are not in public docs**.

Per AquaFlux docs (confirmed via `?ask=` query):
- No documented on-chain getter for APY per tranche
- Yield = coupon share + protocol fee + rewards - expected loss (waterfall)
- Tri-token model: P (Principal), C (Collateral), S (Senior yield)

### Workaround

```bash
# Report tri-token components to user
echo "AquaFlux Core: 0x0da9... (proxy → impl 0x3a36...)"
echo "AMM PoolManager: 0x2A92..."
echo "For live APY: https://app.aquaflux.pro"
```

Skill should mention AquaFlux exists with its tri-token system, but DEFER APY reading to the
official app until ABI is published.

---

## Faroswap APY 🟡

DEX (DODO fork). LP APY is not a single view function — needs pool discovery + 24h fee data.

### Use DODO route service for quotes

```bash
curl -s "https://api.dodoex.io/route-service/v2/widget/getdodoroute" \
  -G \
  --data-urlencode "chainId=1672" \
  --data-urlencode "fromTokenAddress=$USDC" \
  --data-urlencode "toTokenAddress=$WPROS" \
  --data-urlencode "fromAmount=1000000000" \
  --data-urlencode "slippage=1"
```

For LP APY: skip in v0.3.0 — direct user to Faroswap app.

---

## Projected Return

```
projected_return = principal × (nominal_apy / 100) × (days / 365)
```

Use **nominal APY** (not risk-adjusted). Risk-adjusted is for ranking.

### Example

$5,000 USDC into R25 7-day vault at 8.5% nominal APY for 30 days:
```
return = 5000 × 0.085 × (30/365) = $34.93 USDC
```

---

## Error Handling

| Error | Action |
|-------|--------|
| `Block range too large` (>1000 blocks) | Chunk query into 1000-block windows |
| Protocol returns empty (`0x`) | Mark as "no activity yet"; do not fabricate APY |
| Tuple parse fails | Try Aave V3 strict tuple; if still fails, fall back to PoolDataProvider simpler view |
| RPC unreachable | Retry 3x with 5s delay; then report "Pharos mainnet unreachable" |
| All protocols fail | Report connectivity issue, do not fabricate any values |
