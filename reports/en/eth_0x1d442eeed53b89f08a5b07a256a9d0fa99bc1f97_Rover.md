# Rover / QU!D: Manipulable Uniswap V3 slot0 Pricing Causes LP Share and Treasury Misallocation

| Field | Value |
|-------|-------|
| **Report ID** | ETH-PM-2026-002 |
| **Target** | `0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97` (Ethereum Mainnet) |
| **Contract** | `Rover` (V3 NFPM liquidity wrapper; QU!D stack) |
| **Severity** | **High** |
| **Category** | Price manipulation / Spot AMM |
| **Reporter** | Zhao Zhijie (赵志杰) |
| **Date** | 2026-05-19 |
| **Status** | Draft |

**Language:** English · [中文](../zh-CN/eth_0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97_Rover.md)

---

## Summary

`Rover` manages a fixed **WETH/USDC Uniswap V3 `POOL`** and NFPM positions. All critical paths read **`IUniswapV3Pool(POOL).slot0()`** → `Rover.getPrice()` for USDC-denominated ETH pricing used in `_swap`, `deposit`/`withdraw`, and liquidity served to `Aux`/`Amp`.

There is **no TWAP or Chainlink**. The fatal chain is **`fetch`/`deposit` → `_repackNFT` → `_swap` (`exactInput`, `amountOutMinimum = 0`) → `NFPM.mint` (`amount0Min/amount1Min = 0`)**. After spot manipulation, attackers can mint excess `positions[msg.sender].liq`, extract surplus ETH via `withdraw` + `Amp.get`, or drain WETH/USDC when `Aux._clearSwaps` calls `V3.take` / `V3.withdrawUSDC`.

**Etherscan:** https://etherscan.io/address/0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97

## Verified deployments

Byte-identical verified source (0 line diff). Reuse this report; bind PoC to target `POOL` / `ROUTER` / `NFPM` / `AUX`.

| Address | Notes |
|---------|-------|
| `0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97` | **Primary target** |
| `0x491c3a404ff9564a4a09cf777c833c4d138801e6` | Isomorphic |
| `0x1d8227d7cdabce4ac80182857bc0de28645fae53` | Isomorphic |
| `0xb9ae20b2ed85e508c29881f24aa1308dd3751aff` | Isomorphic |

## Affected scope

**Rover functions:** `getPrice`, `fetch`, `repackNFT`, `deposit`, `withdraw`, `_swap`, `take`, `depositUSDC`, `withdrawUSDC`.

**Cross-contract:** `Aux._clearSwaps` → `V3.take` / `withdrawUSDC`; `Amp.leverETH` uses `V3.repackNFT()` + `getPrice()`.

## Root cause

### 1. Spot pricing without manipulation resistance

```solidity
function getPrice(uint160 sqrtRatioX96) public view returns (uint price) {
    uint256 ratioX128 = FullMath.mulDiv(casted, casted, 1 << 64);
  // derives USDC/WETH from sqrtPriceX96 only
}
```

### 2. Trigger vs sink mapping

| Tax-token template | Rover implementation |
|--------------------|----------------------|
| `_transfer` trigger | `fetch` / `deposit` / `repackNFT` |
| `swapAndLiquify` | `_repackNFT` → `_swap` → `NFPM.mint` |
| `swapTokensForEth` (min=0) | `ISwapRouter.exactInput(..., amountOutMinimum: 0)` |
| `addLiquidity` (min=0) | `NFPM.mint(..., amount0Min: 0, amount1Min: 0)` |

```solidity
// _swap — fatal: amountOutMinimum = 0
usdc += ISwapRouter(ROUTER).exactInput(
    ISwapRouter.ExactInputParams(..., targetETH, 0)
) * 1e12;

// _repackNFT — fatal: amount0Min/amount1Min = 0
NFPM.mint(INonfungiblePositionManager.MintParams({
    ...
    amount0Min: 0,
    amount1Min: 0,
    ...
}));
```

**No** `_transfer` / `swapAndLiquify` in this verified deployment.

## Attack scenario

1. Flash-loan manipulate `Rover.POOL` reserves.
2. Same block: victim `deposit` or public `repackNFT` → `_swap` at bad Router rates → `NFPM.mint` at bad ratio.
3. Attacker reverse-swaps for **slippage revenue**; optionally deposits at distorted price for excess shares.

Atomic in one block: distort → trigger Rover path → arbitrage.

## Impact

| Dimension | Assessment |
|-----------|------------|
| **Financial** | Rover LP shares, WETH/USDC treasury, `Amp` leverage positions |
| **Integrity** | `totalShares` vs `liquidityUnderManagement` distorted |
| **Availability** | Repeatable until TWAP / min-out added |

**High severity:** Permissionless pricing + zero min-out Router/NFPM sinks.

## Proof of concept

1. Fork mainnet; set `target` to any Rover address above; bind `POOL`/`ROUTER`/`NFPM`.
2. Flash-loan dump WETH in `POOL` (>1% vs Chainlink).
3. Same block: `Rover.deposit{value: X}()` or trigger `Aux.clearSwaps`.
4. Compare shares / withdrawals vs fair-price baseline.

## Recommendations

1. TWAP via `OracleLibrary.consult(POOL, secondsAgo)` or Chainlink for share pricing.
2. Non-zero `amountOutMinimum` / NFPM `amount*Min` from TWAP quotes.
3. Decouple unconditional `repack` from deposit/withdraw; spot vs TWAP check before repack.
4. Cap `AMP.get` top-up in `withdraw` with price sanity checks.
5. TWAP bounds before `Aux` calls `V3.*`.

## References

- Etherscan verified source (primary + isomorphic addresses)
- Related: [Vogue report](eth_0x0bafc174bf07ec3d0eafdee06df7d07377108c0c_Vogue.md) (V4 batch path on same stack)
- Core: `src/Rover.sol`, `src/Aux.sol`, `src/Amp.sol`

## Disclosure timeline

| Date | Event |
|------|-------|
| 2026-05-19 | Audit complete; draft report |
| TBD | Platform submission |
