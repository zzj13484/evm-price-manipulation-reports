# Vogue / QU!D: Manipulable Uniswap V4 Spot Price Causes Swap and LP Accounting Distortion

| Field | Value |
|-------|-------|
| **Report ID** | ETH-PM-2026-001 |
| **Target** | `0x0bafc174bf07ec3d0eafdee06df7d07377108c0c` (Ethereum Mainnet) |
| **Contract** | `Vogue` (protocol stack: `Aux`, `VogueCore`, `Basket`, `Rover`, `Amp`) |
| **Severity** | **High** |
| **Category** | Price manipulation / Spot oracle |
| **Reporter** | Zhao Zhijie (赵志杰) |
| **Date** | 2026-05-19 |
| **Status** | Draft |

**Language:** English · [中文](../zh-CN/eth_0x0bafc174bf07ec3d0eafdee06df7d07377108c0c_Vogue.md)

---

## Summary

Vogue (part of the QU!D basket protocol) settles user swaps, batch liquidations (ASS), LP deposits/withdrawals, and leverage using the **instantaneous `sqrtPriceX96` from an internal Uniswap V4 pool** (`VogueCore.VANILLA`, mock ETH/USD), converted to USD via `Aux.getPrice()`. The design does **not** use TWAP, Chainlink, or meaningful anchoring to external WETH/USDC markets.

An attacker can flash-loan or use capital to move the V4 spot price within one block, then trigger victims via `Aux.swap()`, `clearSwaps()`, `deposit()`/`withdraw()`, or `leverETH()`/`leverUSD()`, causing mispriced settlement, LP dilution, or improper use of `POOLED_ETH` / `POOLED_USD`. Small swaps (< ~$5,000 equivalent) execute **immediately without sandwich protection** per source comments.

## Verified deployments

Same PM mechanism applies to all listed addresses; submit with the target deployment address and on-chain constructor wiring.

| Address | Notes |
|---------|-------|
| `0x0bafc174bf07ec3d0eafdee06df7d07377108c0c` | **Primary target** |
| `0xb3c368bc4033361c2d9a2568d32ea8dd585b7fec` | Byte-identical source |
| `0x19f4b1215456f706475e0ee408d216ebb181129a` | Variant (`Amp.leverETH` lacks `onlyUs`) |
| `0x9743de2355d3840d52529503d496bf33ff7a9793` | Identical to `0x19f4b121…` |
| `0x32956d93910feb9dfb80e965cec248bda58eb5fe` | Extended variant (11 patched `src/` files) |

**Etherscan:** https://etherscan.io/address/0x0bafc174bf07ec3d0eafdee06df7d07377108c0c

## Affected scope

| Component | Role |
|-----------|------|
| `Vogue` | LP, batch swap queue, `distributeBatchSwaps` |
| `Aux` | User `swap()`, `clearSwaps()`, `getPrice()` |
| `VogueCore` | V4 `poolManager.swap` / `batchSwap`; `POOLED_ETH` / `POOLED_USD` |
| `Rover` | Auxiliary V3 liquidity; also `slot0`-based pricing |

**Key functions:** `Aux.swap`, `Aux.clearSwaps`, `Vogue.distributeBatchSwaps`, `Vogue.deposit`/`withdraw`, `Aux.leverETH`/`leverUSD`, `VogueCore.swap`/`batchSwap`.

## Root cause

### 1. Pricing from manipulable V4 spot

`Aux.getPrice()` converts `sqrtPriceX96` (from V4 `repack()` / `poolStats()`) to USD. In normal operation this is almost always the **internal V4 price**, not TWAP.

```solidity
function getPrice(uint160 sqrtPriceX96) public view returns (uint price) {
    if (sqrtPriceX96 == 0)
        (sqrtPriceX96,,,,,, ) = v3PoolWETH.slot0();
    // FullMath derives USDC per WETH from sqrtPriceX96
}
```

`Aux.swap()` calls `V4.repack()` then either queues (large) or **immediately** `CORE.swap(sqrtPriceX96, ...)` for small amounts (no sandwich protection).

`VogueCore` uses the same `sqrtPriceX96` with `paddedSqrtPrice` (±3% / ±10% bands) — limiting deviation **relative to the already manipulable reference**, not fair market price.

### 2. Public batch settlement with 2% execution tolerance only

`clearSwaps()` is **external, permissionless**. `distributeBatchSwaps` checks actual fill vs expectation within **2%**, but **does not validate** that `getPrice(sqrtPriceX96)` matches external market price.

### 3. LP accounting tied to distorted price

`deposit()` / `addLiquidityHelper()` use `AUX.getPrice(sqrtPriceX96)` for pairing and `V4.modLP()`.

### 4. Internal V4 pool decoupled from main market

V4 is initialized once from V3 `slot0` at deploy; thereafter evolves independently. Thin mock liquidity is easier to move than main WETH/USDC.

## Attack scenario

1. **Prepare:** Monitor mempool for `Aux.swap`, `clearSwaps`, `deposit`, `withdraw`, `redeem`.
2. **Distort price:** Large swap on V4 `VANILLA` (or reference V3 pool) in the same block.
3. **Trigger settlement:** Victim swap or public `clearSwaps()` runs at manipulated `getPrice`.
4. **Revert pool / arbitrage:** Reverse trade, repay flash loan, keep slippage revenue.

**Fund outflow paths (verified):** `Aux._clearSwaps` → `VogueCore.batchSwap` → `Vogue.distributeBatchSwaps`; instant `Aux.swap` → `VogueCore.swap`; `Vogue.deposit`/`withdraw`; `Aux.redeem`/`_take`; `Aux.leverETH`/`leverUSD`.

This deployment has **no** tax-token `_transfer` → `swapAndLiquify` pattern.

## Impact

| Dimension | Assessment |
|-----------|------------|
| **Financial** | ETH, USDC, and ERC4626 vault assets can exit at wrong rates via batch/instant swap, LP, and redeem paths |
| **Integrity** | Dollar-denominated accounting and `POOLED_*` / share invariants break under spot manipulation |
| **Availability** | No shutdown; repeatable while spot pricing remains unprotected |

**High severity:** No privileged role required; core protocol funds affected; flash-loan viable when liquidity is thin.

## Proof of concept

**Status:** Conceptual — Foundry mainnet fork recommended.

1. Fork mainnet; bind deployed `Vogue` / `Aux` / `VogueCore`.
2. Record `poolStats().sqrtPriceX96` and `Aux.getPrice()`.
3. Flash-loan WETH; manipulate V4 pool; confirm >1% deviation vs Chainlink / V3 TWAP.
4. Same block: victim `Aux.swap` or `clearSwaps()`; compare output vs fair quote.

```solidity
function test_price_manipulation() public {
    uint160 priceBefore = poolStats_sqrtPrice();
    manipulateV4Pool(/* push price */);
    uint160 priceAfter = poolStats_sqrtPrice();
    assertGt(relativeDeviation(priceBefore, priceAfter), 0.01e18);
    uint victimOut = victimSwap(...);
    uint fairOut = quoteFairMarket(...);
    assertLt(victimOut, fairOut * 99 / 100);
}
```

## Recommendations

1. Do **not** use V4 internal `slot0` as sole price source; use V3 TWAP / Chainlink with deviation bounds.
2. Add sandwich protection on instant swap path; bind limits to main-pool TWAP.
3. `clearSwaps()`: oracle sanity check before settlement; consider keeper + rate limits.
4. `distributeBatchSwaps`: per-user min-out, not only 2% aggregate tolerance.
5. Circuit breaker on large single-block `sqrtPriceX96` moves.
6. Deployments `0x19f4b121…` / `0x9743de…`: restore `onlyUs` on `Amp.leverETH` / `leverUSD`.

## References

- Etherscan verified source for primary and isomorphic addresses
- Related: [Rover report](eth_0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97_Rover.md) (same QU!D stack, V3 path)
- Key files: `src/Aux.sol`, `src/Vogue.sol`, `src/VogueCore.sol`, `src/Amp.sol`

## Disclosure timeline

| Date | Event |
|------|-------|
| 2026-05-19 | PM scan + manual audit; draft report |
| TBD | Platform submission |
| TBD | Fix verification |
