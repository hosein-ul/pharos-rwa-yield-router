# Yield Scan — Reference (v0.2.0, mainnet)

All operations are read-only. Default network: Pacific Mainnet (chain 1672).

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
3. [Morpho APY](#morpho-apy)
4. [TermMax APY](#termmax-apy)
5. [AquaFlux APY](#aquaflux-apy)
6. [Zona APY](#zona-apy)
7. [R25 Axil APY](#r25-apy)
8. [Native PROS Staking](#native-staking)
9. [Projected Return](#projected-return)

---

## Full Scan

```
1. Read assets/protocols.json — iterate only entries with status="DEPLOYED"
2. For each protocol:
   a. Read APY using the method documented per protocol below
   b. Read TVL (totalAssets() for ERC-4626; protocol-specific for others)
   c. risk_adjusted_apy = nominal_apy × risk_multiplier (from protocols.json)
3. Read USD prices for all relevant tokens from Chainlink (see Price Lookup)
4. Apply user filters (asset, min deposit, risk tolerance, lockup)
5. Sort by risk_adjusted_apy descending
6. Present ranked table + top recommendation + projected return
```

---

## Price Lookup

```bash
FEED=$(jq -r '."mainnet".feeds."<PAIR>"' assets/oracles.json)
cast call $FEED "latestAnswer()(int256)" --rpc-url $RPC
```

**Pacific Mainnet feeds** (Chainlink Push Engine, 18 decimals):

| Pair | Feed Address |
|------|-------------|
| PROS/USD | `0x9356C87a48F913d11C87a0d4b8cD16CD04624BF3` |
| ETH/USD  | `0x092ff0175Be8B2e83Ca5740d3EB13C6225901fa7` |
| BTC/USD  | `0x6BFcd14b164de6c8C4dA2d065d511055A589EB20` |
| USDC/USD | `0x8d08eA83A55ad1e805b5660F5eC76C99C6aF5eaf` |
| USDT/USD | `0x84B06e38C70DD1f0039bA25E017CAe7cFcDE53b0` |

Parse: `price_usd = raw_int / 1e18`.

---

## Morpho APY

Pharos Morpho deployment is at `0x18573fA18fd17dDfD790B4a5B5b2977aad3b4Efb` (DIFFERENT from canonical).

### Discover active markets

Morpho Blue uses `bytes32` marketId derived from market params. Active markets are emitted via
`CreateMarket` events. To enumerate:

```bash
MORPHO=$(jq -r '."mainnet".morpho.contracts.morpho_blue' assets/protocols.json)

# Get all CreateMarket events
cast logs --from-block 0 --to-block latest \
  --address $MORPHO \
  "CreateMarket(bytes32,(address,address,address,address,uint256))" \
  --rpc-url $RPC
```

### Read market state and compute APY

For each active marketId:
```bash
cast call $MORPHO \
  "market(bytes32)((uint128,uint128,uint128,uint128,uint128,uint128))" \
  $MARKET_ID --rpc-url $RPC
# Returns: (totalSupplyAssets, totalSupplyShares, totalBorrowAssets, totalBorrowShares,
#           lastUpdate, fee)
```

Then query IRM for current rate:
```bash
IRM=$(jq -r '."mainnet".morpho.contracts.adaptive_curve_irm' assets/protocols.json)
cast call $IRM "borrowRateView((bytes32,uint128,uint128,uint128,uint128,uint128,uint128))(uint256)" $MARKET_PARAMS --rpc-url $RPC
# Returns rate per second, scaled by 1e18
```

Convert to APY:
```
utilization = totalBorrowAssets / totalSupplyAssets
supply_rate = borrow_rate × utilization × (1 - fee/1e18)
apy = (1 + supply_rate / 1e18) ^ (365 × 86400) - 1
apy_pct = apy × 100
```

### Fallback — use Morpho's API if on-chain enumeration is too slow

```bash
curl -s "https://blue-api.morpho.org/graphql" \
  -X POST -H "Content-Type: application/json" \
  -d '{"query":"{ markets(where: { chainId_in: [1672] }) { items { uniqueKey state { supplyApy borrowApy utilization } } } }"}'
```

Note: as of 2026-06-13 Morpho API may not yet support chainId 1672. Verify before using.

---

## TermMax APY

Fixed-rate lending. Each market has a defined fixed rate, not a floating APY.

### Discover active markets

```bash
TERMMAX_FACTORY=$(jq -r '."mainnet".termmax.contracts.factory_v2' assets/protocols.json)
VIEWER=$(jq -r '."mainnet".termmax.contracts.market_viewer' assets/protocols.json)

# MarketViewer exposes view functions for all markets
cast call $VIEWER "getAllMarkets()(address[])" --rpc-url $RPC
```

### Read per-market fixed rate

For each market address:
```bash
cast call $MARKET "fixedRate()(uint256)" --rpc-url $RPC
cast call $MARKET "maturity()(uint256)" --rpc-url $RPC
cast call $MARKET "underlying()(address)" --rpc-url $RPC
cast call $MARKET "totalAssets()(uint256)" --rpc-url $RPC
```

Convert: `apy_pct = fixedRate / 1e16` (assuming WAD scaling, 1e18 = 100%).

Verify scaling with a known market — try `fixedRate / 1e16` first; if result is implausible
(>200%), try `fixedRate / 1e18 × 100`.

---

## AquaFlux APY

Tri-token model: Yield Token (Y) / Principal Token (P) / Collateral Token (C). Each tranche
has different return characteristics.

```bash
CORE=$(jq -r '."mainnet".aquaflux.contracts.core_proxy' assets/protocols.json)
FACTORY=$(jq -r '."mainnet".aquaflux.contracts.token_factory' assets/protocols.json)

# Enumerate all token sets
cast call $FACTORY "getAllTokenSets()(address[])" --rpc-url $RPC
# or
cast call $CORE "getAllPositions()(...)" --rpc-url $RPC
```

For each Y/P/C set:
```bash
# Read tranche yield from oracle or position state
cast call $CORE "getYieldTokenAPY(address)(uint256)" $Y_TOKEN --rpc-url $RPC
cast call $CORE "getPositionInfo(address)((...))" $POSITION --rpc-url $RPC
```

If specific getter unavailable, fall back to ERC-4626 pattern on the Y token if it implements it.

**Note**: AquaFlux ABI not fully documented publicly. Read the AquaFlux Core contract on
explorer to identify view functions, or fall back to API:
```bash
curl -s "https://api.aquaflux.pro/tranches?chain=1672"
```

---

## Zona APY

Zona is an Aave V3 fork. Standard Aave reading pattern.

```bash
POOL=$(jq -r '."mainnet".zona.contracts.pool_proxy' assets/protocols.json)
DATA_PROVIDER=$(jq -r '."mainnet".zona.contracts.pool_data_provider' assets/protocols.json)
USDC=$(jq -r '."mainnet"[] | select(.symbol=="USDC") | .address' assets/tokens.json)

# Aave V3 getReserveData returns a long tuple
cast call $POOL \
  "getReserveData(address)((uint256,uint128,uint128,uint128,uint128,uint128,uint40,uint16,address,address,address,address,uint128,uint128,uint128))" \
  $USDC --rpc-url $RPC
```

Tuple field index 3 (`currentLiquidityRate`) is supply APY in RAY (1e27).

```
liquidity_rate = rate_ray / 1e27
apy = (1 + liquidity_rate / (365 × 86400)) ^ (365 × 86400) - 1
apy_pct = apy × 100
```

Alternative simpler view (if data provider exposes it):
```bash
cast call $DATA_PROVIDER \
  "getReserveData(address)((uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint256,uint40))" \
  $USDC --rpc-url $RPC
# liquidityRate is at index 5
```

---

## R25 Axil APY

R25 vaults are ERC-4626. APY needs two `previewRedeem(1e18)` snapshots over time.

```bash
VAULT_7D=$(jq -r '."mainnet".r25_axil.vaults.vrpcw_7day.address' assets/protocols.json)
VAULT_6M=$(jq -r '."mainnet".r25_axil.vaults.vrpcs_6month.address' assets/protocols.json)

# Current price per share (returns USDC, 6 decimals)
cast call $VAULT_7D "previewRedeem(uint256)(uint256)" 1000000000000000000 --rpc-url $RPC
# Total assets in USDC
cast call $VAULT_7D "totalAssets()(uint256)" --rpc-url $RPC
```

### Real-time APY (requires snapshot)

```
pps_now  = previewRedeem(1e18) / 1e6   # USDC has 6 decimals
pps_then = previously saved snapshot
days_elapsed = (block_now - block_then) / blocks_per_day
apy = (pps_now / pps_then) ^ (365 / days_elapsed) - 1
```

If no historical snapshot exists, fall back to R25's published yield:
```bash
curl -s "https://app.r25.xyz/api/vaults?chain=1672"
```

**Lockup disclosure**: ALWAYS tell the user about the lockup before recommending:
- VRPCW: 7-day withdrawal lock
- VRPCS: 6-month withdrawal lock

---

## Native Staking

Native PROS staking does not use an ERC-20 contract. Delegation typically goes through a
precompile or system contract. As of 2026-06-13 the exact contract address/ABI is not
publicly documented; consult `assets/ecosystem.json` for current Pharos staking guidance.

For now, report Native PROS staking as "DEPLOYED — delegation via Pharos validator UI" and
do not provide a `cast call` template. Once Pharos publishes the staking precompile, add it.

---

## Projected Return

```
projected_return = principal × (nominal_apy / 100) × (days / 365)
```

Use **nominal APY**, not risk-adjusted. Risk-adjusted is for ranking; nominal is the actual
expected return.

### Example

$10,000 USDC into TermMax 30d fixed at 5.4%:
```
return_30d = 10000 × 0.054 × (30 / 365) = $44.38 USDC
```

---

## Error Handling

| Error | Action |
|-------|--------|
| Protocol RPC call fails | Mark protocol "data unavailable"; continue with others |
| All protocol reads fail | Report "RPC connectivity issue"; do not fabricate APY |
| Address contains "COMING_SOON" / "TESTNET_ONLY" | Skip with status note |
| Unknown ABI / call reverts | Try alternate view function from `_note` field |
| APY result implausible (negative, >1000%) | Try alternate scaling factor (1e16 vs 1e18 vs 1e27) |
