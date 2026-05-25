# PM Audit Supplement: Negative Controls and Governance (Non-PM Reports)

| Date | 2026-05-19 |
| **Purpose** | Static analysis / Datalog **ground truth** — **not** submittable PM price-manipulation reports |

**Language:** English · [中文](../zh-CN/PM_Audit_Supplement_Controls.md)

---

## 1. YOLOToken — negative control (false positive)

| Field | Value |
|-------|-------|
| **Address** | `0xef8a4e034736c148ca591bc1b897f2729d38dbbe` |
| **Reference deploy** | `0x8c05a211d26baab73af5cd74cc2a4fa71f56d32a` (1-line verified diff) |
| **PM scan** | High / 76% / `pm_vuln_count=3` |

### Audit conclusion (price manipulation)

**No** flash-loan / AMM spot / lending-liquidation **price manipulation** surface:

- No external Uniswap/Curve `swap`, `slot0`, `get_dy`, or zero-slippage router settlement
- No dual-track vault share mint vs spot pool (cf. Vault report)
- Core logic is OZ v5 **`_update`** blocking transfers to/from shielded pool:

```solidity
if (from == SHIELDED_POOL || to == SHIELDED_POOL) {
    revert ShieldedPoolTransferBlocked();
}
```

**Ground truth label:** `NEGATIVE_CONTROL / NO_PM_PRICE_MANIPULATION`

---

## 2. Box (QU!D registry) — governance risk, not spot PM

| Field | Value |
|-------|-------|
| **Address** | `0xe3b6d810411774b1805e578f1fbc83f8ce9be323` |
| **Contract** | `Box` (`src/Box.sol`) |
| **PM scan** | Very high / 92% / `pm_vuln_count=2` |

### Audit conclusion (price manipulation)

**Not** classifiable as single-block spot sandwich PM like MasterChef/Rover:

| Dimension | Notes |
|-----------|-------|
| **User flash-loan surface** | No `swapExactTokensForTokens(..., 0)` or V3 `slot0` share mint; swaps via **`ISwapper` + `maxSlippage` + NAV cache** on governance/ops paths |
| **Real risk** | **Centralization:** register/shutdown FundingModules, Timelock, Recover — key compromise redirects funds |
| **Suggested report type** | Access control / governance (separate track) |

**Ground truth label:** `GOVERNANCE_RISK / NOT_SPOT_PM_SINK`

---

## 3. Isomorphic instances merged into main reports

| Family | Address | Merged into |
|--------|---------|-------------|
| MasterChef | `0xdae1acc21ed8e26beb311edeb70e1ae5e27e8a0b` | [MasterChef report](../en/eth_0x7f6229786703f01a8bfcede5f94c36179c467acd_MasterChef.md) |
| Rover | `0xb9ae20b2ed85e508c29881f24aa1308dd3751aff` | [Rover report](../en/eth_0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97_Rover.md) |
