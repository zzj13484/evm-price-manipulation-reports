# Vault (Naturelab): Stale exchangePrice vs Live Midas Oracle Enables Share Over-Minting

| Field | Value |
|-------|-------|
| **Report ID** | ETH-PM-2026-005 |
| **Target** | `0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3` (Ethereum Mainnet) |
| **Contract** | `Vault` (`VaultYieldBasic` + `StrategyMorphoBlue`; ERC4626) |
| **Severity** | **High** (conditional on short-term Midas / Morpho oracle manipulation) |
| **Category** | Price manipulation / Custom oracle + vault share pricing |
| **Reporter** |
| **Date** | 2026-05-19 |
| **Status** | Draft |

**Language:** English · [中文](../zh-CN/eth_0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3_Vault.md)

---

## Summary

Naturelab **`Vault`** mints shares using **`vaultState.exchangePrice`** (rebalancer-updated, lagging NAV), while **`MF_ONE` deposits** scale collateral by **`IMidasOracle.lastAnswer()`** in real time before **`previewDeposit`**.

When **`lastAnswer` spikes** but **`exchangePrice` has not caught up**, attackers deposit `MF_ONE` and mint **too many shares**, diluting other LPs. **`MAX_PRICE_CHANGE_RATE` (1%) checks are commented out** in `updateExchangePrice()`. Strategy **`getNetAssets()`** also values `MF_ONE` via Midas and Morpho **`MORPHO_ORACLE.price()`**.

**Etherscan:** https://etherscan.io/address/0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3

**Not applicable:** Uniswap `slot0`, QU!D batch swaps, tax-token `swapAndLiquify`.

## Affected scope

| Component | Address (from source) | Role |
|-----------|----------------------|------|
| `Vault` | Target | Deposits / share mint |
| `MIDAS_ORACLE` | `0x8D51DBC85cEef637c97D02bdaAbb5E274850e68C` | `lastAnswer()` for MF_ONE |
| `StrategyMorphoBlue` | Deployed strategy | `getNetAssets()` |
| `MORPHO_ORACLE` | `0x0cB1928EcA8783F05a07D9Ae2AfB33f38BFBEb78` | Morpho collateral pricing |

**Functions:** `optionalDeposit`/`deposit` → `optionalDepositShares` → `previewDeposit` → `_mint`; `updateExchangePrice` (rebalancer).

## Root cause

### Dual-track pricing

`totalAssets()` uses **cached** exchange price:

```solidity
function totalAssets() public view override returns (uint256) {
    if (block.timestamp - vaultState.lastUpdatePriceTime > vaultParams.maxPriceUpdatePeriod) {
        revert Errors.PriceNotUpdated();
    }
    return vaultState.exchangePrice * totalSupply() / PRECISION;
}
```

`MF_ONE` deposit scales assets by **live oracle** then calls `previewDeposit`:

```solidity
function optionalDepositShares(address _token, uint256 _assets) internal view returns (uint256 shares_) {
    if (_token == MF_ONE) {
        int256 price_ = IMidasOracle(MIDAS_ORACLE).lastAnswer();
        _assets = _assets * uint256(price_) / 1e20;
    }
    shares_ = previewDeposit(_assets);  // denominator uses stale exchangePrice
}
```

**Scissors effect:** numerator inflated by `lastAnswer`; denominator anchored to stale `exchangePrice` → excess shares.

### NAV update

`underlyingTvl()` feeds `updateExchangePrice`; **`MAX_PRICE_CHANGE_RATE` guard commented out** — single rebalancer update can jump NAV if oracle is manipulated same block.

No Router zero min-out on user deposit path; exploit is **share mispricing**, not AMM swap slippage.

## Attack scenario

1. Manipulate or exploit stale **`lastAnswer`** upward (or deposit during oracle spike before rebalancer update).
2. Call `optionalDeposit(MF_ONE, ...)` while `exchangePrice` still low.
3. Receive inflated shares; later redeem at fair NAV or sell shares to victims.

Requires oracle manipulation window or lag between oracle and `exchangePrice` update.

## Impact

| Dimension | Assessment |
|-----------|------------|
| **Financial** | Existing LPs diluted; attacker captures NAV spread |
| **Integrity** | ERC4626 share price invariant broken across oracle vs cached price |
| **Availability** | Repeatable while dual-track pricing persists |

**High severity (conditional):** No admin role for deposit; impact scales with oracle manipulability and rebalancer lag.

## Proof of concept

1. Fork mainnet; read `exchangePrice`, `lastAnswer`, `totalSupply`.
2. Simulate elevated `lastAnswer` with unchanged `exchangePrice`.
3. Compare `optionalDepositShares(MF_ONE, Q)` vs fair-NAV mint amount.
4. Optional: model post-deposit redeem profit.

## Recommendations

1. Unify pricing: `previewDeposit` must use same oracle/NAV source as asset conversion.
2. Re-enable **`MAX_PRICE_CHANGE_RATE`** on `updateExchangePrice`.
3. Revert deposits when `lastAnswer` deviates from cached `exchangePrice` beyond ε.
4. Use TWAP or multi-source oracle for `MF_ONE`; cap single-block deposit size.
5. Rebalancer must atomically update `exchangePrice` when oracle moves beyond threshold.

## References

- Etherscan verified source
- Contrast: [Vogue](eth_0x0bafc174bf07ec3d0eafdee06df7d07377108c0c_Vogue.md) (AMM spot), [MasterChef](eth_0x7f6229786703f01a8bfcede5f94c36179c467acd_MasterChef.md) (Router min-out)

## Disclosure timeline

| Date | Event |
|------|-------|
| 2026-05-19 | Audit; draft report |
| TBD | Platform submission |
