# Yield Scan — Reference

All operations are read-only. No private key required.

> **Setup**: Load network config before any command:
> ```bash
> RPC=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
> BLOCK=$(cast block-number --rpc-url $RPC)
> ```

---

## Table of Contents
1. [Full Scan](#full-scan)
2. [Price Lookup](#price-lookup)
3. [ELFi APY](#elfi-apy)
4. [Morpho APY](#morpho-apy)
5. [Faroo APY](#faroo-apy)
6. [TVL Query](#tvl-query)
7. [Projected Return](#projected-return)

---

## Full Scan

Run all protocol queries, skip any that are unavailable, rank by risk-adjusted APY.

### Step-by-step

```
1. Load RPC from assets/networks.json
2. Record current block with: cast block-number --rpc-url $RPC
3. For each protocol in assets/protocols.json:
   a. Skip if address contains "PLACEHOLDER"
   b. Read APY using the appropriate method (see sections below)
   c. Read TVL (see TVL Query)
   d. Apply risk multiplier from assets/risk-model.md
   e. Calculate risk_adjusted_apy = nominal_apy × multiplier
4. Sort available protocols by risk_adjusted_apy (descending)
5. Apply user filters (asset, min deposit, risk tolerance, liquidity)
6. Present ranked table and top recommendation
```

---

## Price Lookup

Reads live USD prices from Chainlink Push Engine oracles deployed on Pharos.

### Command Template

```bash
FEED=$(jq -r '."atlantic-testnet".feeds."<PAIR>"' assets/oracles.json)
cast call $FEED "latestAnswer()(int256)" --rpc-url $RPC
```

### Available Pairs (Atlantic Testnet)

| Pair | Feed Address |
|------|-------------|
| PROS/USD | `0x67488Fac9Bc4174a53a485b11F2066498Cd34b3A` |
| ETH/USD  | `0xCd47D1843f3D6313836303fE1434BA26D257d500` |
| BTC/USD  | `0x82d0e03ea6d94120B92EA4Ea236DcFA273D42994` |
| USDC/USD | `0xDF6afcf662345Ea29ceACa6DA06141d828c516EA` |
| USDT/USD | `0x2f7796B346d01a3f2264Ff0D93dDdFF8680b8B66` |
| WBTC/USD | `0x6F24f8bDeF2870aCa886fb3Fbc04919B0B46F993` |

### Output Parsing

- Result is `int256` with **18 decimal places**.
- `price_usd = raw_result / 1e18`
- Example: raw `1800000000000000000000` → `$1800.00`

### Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| Empty result | Feed contract unreachable | Skip USD conversion, show native amounts only |
| `execution reverted` | Wrong feed address | Check assets/oracles.json for the correct address |

### Agent Guidelines

1. Always show timestamp or block number alongside prices to indicate freshness.
2. For native PHRS prices on testnet, use the PROS/USD feed (same asset, different network name).
3. Round USD values to 2 decimal places for display.

---

## ELFi APY

ELFi is an Aave V3 fork. Read the supply APY from the lending pool's reserve data.

### Prerequisites

Load the ELFi pool address:
```bash
ELFI_POOL=$(jq -r '."atlantic-testnet".elfi.pools.lending_pool' assets/protocols.json)
USDC_ADDR=$(jq -r '."atlantic-testnet"[] | select(.symbol=="USDC") | .address' assets/tokens.json)
```

If `ELFI_POOL` contains "PLACEHOLDER", skip this protocol and note it as unavailable.

### Step 1 — Read reserve data (Aave V3 pattern)

```bash
cast call $ELFI_POOL \
  "getReserveData(address)((uint256,uint128,uint128,uint128,uint128,uint128,uint40,uint16,address,address,address,address,uint128,uint128,uint128))" \
  $USDC_ADDR \
  --rpc-url $RPC
```

### Output Parsing

The return tuple's 4th field (index 3, 0-based) is the `liquidityRate` in RAY units (1e27):

```
tuple = (config, liquidityIndex, variableBorrowIndex, currentLiquidityRate, ...)
currentLiquidityRate = tuple[3]  (uint128, in RAY = 1e27)
```

Convert to APY:
```
rate_per_second = liquidityRate / 1e27
apy = (1 + rate_per_second / 31536000)^31536000 - 1  # compound annually
apy_pct = apy * 100
```

### Fallback — Try simpler supply rate function

```bash
cast call $ELFI_POOL \
  "getSupplyRate(address)(uint256)" \
  $USDC_ADDR \
  --rpc-url $RPC
```

Convert: `rate / 1e27 → decimal → compound → %`

### Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| `execution reverted` | USDC not listed as a reserve | Try with USDT address |
| Empty result | Pool address wrong or not deployed | Mark ELFi as "unavailable" |
| Tuple parse fails | ABI mismatch (different Aave version) | Try fallback `getSupplyRate` |

### Agent Guidelines

1. Try the full `getReserveData` call first.
2. On failure, try `getSupplyRate`.
3. On both failing, mark ELFi as "data unavailable" — do not estimate.
4. Show the liquidityRate in both raw RAY and converted APY%.
5. Also query TVL for the USDC reserve (see TVL Query section).

---

## Morpho APY

Morpho vaults implement ERC-4626. APY is computed from price-per-share change over time.

### Prerequisites

```bash
MORPHO_VAULT=$(jq -r '."atlantic-testnet".morpho.vaults.usdc_vault' assets/protocols.json)
```

If `MORPHO_VAULT` contains "PLACEHOLDER", skip and note unavailable.

### Step 1 — Read current share price

```bash
# Total assets in the vault (in USDC, 6 decimals)
cast call $MORPHO_VAULT "totalAssets()(uint256)" --rpc-url $RPC

# Total shares outstanding (in vault token, 18 decimals)
cast call $MORPHO_VAULT "totalSupply()(uint256)" --rpc-url $RPC

# Price per share: preview how much USDC 1e18 shares redeem for
cast call $MORPHO_VAULT "previewRedeem(uint256)(uint256)" 1000000000000000000 --rpc-url $RPC
```

### Output Parsing

```
price_per_share = previewRedeem(1e18) / 1e6   # USDC has 6 decimals
tvl_usdc = totalAssets / 1e6
```

### APY Calculation Limitation

ERC-4626 APY requires **two price snapshots** separated by time:
```
apy = (price_now / price_then)^(365 / days_elapsed) - 1
```

On first call, you only have one snapshot. Options:
1. Show the current price-per-share and note "APY calculation requires a second measurement".
2. If The Graph or an indexer is available, query historical prices.
3. If the vault has a `totalAssets()` that reflects accrued interest, estimate from block timestamps.

### Fallback — Check for explicit APY function

```bash
# Some vaults expose APY directly
cast call $MORPHO_VAULT "apr()(uint256)" --rpc-url $RPC
cast call $MORPHO_VAULT "currentApy()(uint256)" --rpc-url $RPC
```

### Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| Empty `totalAssets` | Vault not deployed or wrong address | Mark Morpho as unavailable |
| Division by zero | Vault has no shares yet (empty) | Show TVL=0, APY=N/A |

### Agent Guidelines

1. Always report current TVL even if APY cannot be computed.
2. If APY requires two snapshots, clearly say so and offer to check again later.
3. Show the price-per-share as a progress metric (e.g., "1 share = 1.0237 USDC").

---

## Faroo APY

Faroo is a liquid staking protocol for PHRS. Users stake PHRS and receive stPHRS.

### Prerequisites

```bash
FAROO=$(jq -r '."atlantic-testnet".faroo.contracts.staking' assets/protocols.json)
```

If `FAROO` contains "PLACEHOLDER", skip and note unavailable.

### Step 1 — Try common APY getter methods

Try each in order until one succeeds:

```bash
# Method 1 — direct APY in basis points or percentage
cast call $FAROO "getAPY()(uint256)" --rpc-url $RPC

# Method 2 — annual staking APY
cast call $FAROO "getStakingAPY()(uint256)" --rpc-url $RPC

# Method 3 — reward rate per second (needs manual annualization)
cast call $FAROO "rewardRate()(uint256)" --rpc-url $RPC

# Method 4 — annualized yield
cast call $FAROO "annualizedYield()(uint256)" --rpc-url $RPC
```

### Output Parsing

The return encoding depends on the contract. Common patterns:

| Raw value range | Interpretation | Conversion |
|-----------------|---------------|-----------|
| 100–10000 | Basis points | `apy_pct = raw / 100` |
| 1e16–1e20 | 18-decimal percentage | `apy_pct = raw / 1e18 * 100` |
| 1e25–1e27 | RAY-encoded rate | `apy_pct = raw / 1e25` |

Try each interpretation and show whichever produces a plausible result (0–200%).

### Step 2 — Read total staked (TVL)

```bash
cast call $FAROO "totalAssets()(uint256)" --rpc-url $RPC
# or
cast call $FAROO "totalSupply()(uint256)" --rpc-url $RPC
```

### Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| All 4 APY methods fail | Contract not deployed or ABI differs | Mark Faroo as unavailable |
| Result is 0 | Staking not yet active | Show 0% APY with note |

### Agent Guidelines

1. Try all 4 APY methods before marking unavailable.
2. Show raw value and interpreted APY so user can verify.
3. Display the exchange rate if available: `cast call $FAROO "exchangeRate()(uint256)"`.

---

## TVL Query

Read total value locked in a protocol.

### For Aave V3 / ELFi pools

```bash
# Total supplied (all asset types combined)
cast call $ELFI_POOL "totalDeposits()(uint256)" --rpc-url $RPC
# or per-asset
cast call $ELFI_POOL "getReserveData(address)(...)" $USDC_ADDR --rpc-url $RPC
# Read aTokenAddress from tuple, then query its totalSupply
```

### For ERC-4626 vaults (Morpho)

```bash
cast call $VAULT "totalAssets()(uint256)" --rpc-url $RPC
```

Divide by token decimals (USDC = 6, PHRS = 18).

### For liquid staking (Faroo)

```bash
cast call $FAROO "totalAssets()(uint256)" --rpc-url $RPC
```

---

## Projected Return

Simple interest calculation for display purposes.

### Formula

```
projected_return = principal × (nominal_apy / 100) × (days / 365)
```

### Example

User wants to put $5,000 USDC into ELFi T-Bills at 6.8% APY for 90 days:
```
return = 5000 × (6.8 / 100) × (90 / 365)
       = 5000 × 0.068 × 0.2466
       = $83.85 USDC
```

### Agent Guidelines

1. Always use nominal APY for projected return (not risk-adjusted, which is a ranking tool).
2. Show both the return amount and the final portfolio value (principal + return).
3. Add a disclaimer that APY is current rate and may change.
4. If the user specifies a token, show the return in that token and convert to USD using Chainlink.
