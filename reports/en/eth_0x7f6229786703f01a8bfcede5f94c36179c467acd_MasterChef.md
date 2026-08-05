# MasterChef (SushiMasterChef): Zero-Slippage V2 Router Swap of Excess LP Is Manipulable

| Field | Value |
|-------|-------|
| **Report ID** | ETH-PM-2026-003 |
| **Target** | `0x7f6229786703f01a8bfcede5f94c36179c467acd` (Ethereum Mainnet) |
| **Contract** | `MasterChef` (`contracts/yzz/SushiMasterChef.sol`) |
| **Severity** | **High** |
| **Category** | Price manipulation / Spot AMM (Uniswap V2) |
| **Reporter** |
| **Date** | 2026-05-19 |
| **Status** | Draft |

**Language:** English · [中文](../zh-CN/eth_0x7f6229786703f01a8bfcede5f94c36179c467acd_MasterChef.md)

---

## Summary

`MasterChef` tracks LP staking with internal `powerPerPrice` (time-monotonic accounting). The **exploitable PM path** is in **`reward()`**: when on-contract LP exceeds liabilities implied by `powerPerPrice`, **`excessLp`** is sold via **Uniswap V2 Router** for reward token `token` with **`amountOutMin = 0`**.

An attacker manipulates V2 pair reserves **before** `reward()` executes, forcing the chef to sell `excessLp` at a terrible rate, **under-updating `accSushiPerShare`** and diluting all stakers; the attacker captures slippage via same-block reverse swap. Triggers: public **`updatePrice`** (calls `reward` every 300s), or user **`deposit`/`withdraw`**.

**Etherscan:** https://etherscan.io/address/0x7f6229786703f01a8bfcede5f94c36179c467acd

## Verified deployments

| Address | Notes |
|---------|-------|
| `0x7f6229786703f01a8bfcede5f94c36179c467acd` | **Primary target** |
| `0xdae1acc21ed8e26beb311edeb70e1ae5e27e8a0b` | ~23 line diff; **`reward()` zero min-out unchanged** |

## Affected scope

| Component | Role |
|-----------|------|
| `MasterChef` | Holds pool `lpToken`; unlimited Router approve |
| `uniswapRouter` | `0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D` (V2) |
| `buyToken` | Default USDT |
| `token` | Reward ERC20; `accSushiPerShare` updated from buy amount |

**Functions:** `updatePrice`, `deposit`, `withdraw` (triggers) → **`reward`** (sink) → **`swapExactTokensForTokens(..., 0)`**.

**Not the main PM surface:** `powerPerPrice` is time-driven internal accounting — **not** flash-loan manipulable.

## Root cause

### Trigger vs sink

| Template | Implementation |
|----------|----------------|
| Outer trigger | `deposit` / `withdraw` / `updatePrice` |
| Composite settlement | `reward()` computes `excessLp` then swaps |
| Zero min-out sink | `swapExactTokensForTokens(excessLp, 0, path, ...)` |

```solidity
function reward(uint256 _pid) public {
    // ...
    if (excessLp > 0 && address(uniswapRouter) != address(0)) {
        uint256[] memory rewardAmounts = uniswapRouter.swapExactTokensForTokens(
            excessLp,
            0,   // amountOutMin = 0
            swapPathToUse,
            address(this),
            block.timestamp + 300
        );
        totalRewardTokensBought = rewardAmounts[rewardAmounts.length - 1];
    }
    if (totalRewardTokensBought > 0 && pool.totalPower > 0) {
        pool.accSushiPerShare = pool.accSushiPerShare.add(
            totalRewardTokensBought.mul(1e12).div(pool.totalPower)
        );
    }
}
```

V2 implied price follows **instant reserves**; large swaps skew the rate within one block. No `getAmountsOut`-based floor is enforced.

## Attack scenario

1. Wait until `excessLp > 0` (`balance > totalRefundBalance`).
2. Front-run: manipulate V2 pool on `lpToken → buyToken → token` path.
3. Trigger `updatePrice(_pid)` or `deposit` → **`reward`** sells at min-out 0.
4. Reverse swap; repay flash loan; profit = slippage extracted from chef + staker dilution.

Single-block atomic closure.

## Impact

| Dimension | Assessment |
|-----------|------------|
| **Financial** | Stakers lose `pendingSushi`; protocol `excessLp` sold at bad rates |
| **Integrity** | Reward accrual decoupled from fair market buy cost |
| **Availability** | Repeatable until min-out / TWAP added |

**High severity:** Public `reward`/`updatePrice`; `excessLp` grows with deposits.

## Proof of concept

1. Fork mainnet; read `_pid` `lpToken`, `token`, `buyToken`, chef balance.
2. Ensure `excessLp > 0`.
3. Manipulate V2 reserves; same block call `updatePrice(_pid)`.
4. Compare `RewardDistributed.rewardAmount` vs unmanipulated fork.

## Recommendations

1. Set `amountOutMin` from `getAmountsOut` minus slippage bps, or TWAP/Chainlink floor.
2. Optional access control / timelock on `reward` (does not replace min-out).
3. Decouple `updatePrice` accounting from buyback execution with pre-trade sanity checks.
4. Alert on correlated large V2 swaps and `excessLp` sales.

## References

- Etherscan verified source (primary + `0xdae1acc…`)
- Contrast: [Zap](eth_0x7d3a6d1085fe898965cbc0b47a5a652965438cac_Zap.md) (Curve), [Rover](eth_0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97_Rover.md) (V3)

## Disclosure timeline

| Date | Event |
|------|-------|
| 2026-05-19 | PM scan + audit; draft report |
| TBD | Vendor contact |
