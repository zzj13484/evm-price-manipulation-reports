# PM 审计补充：负样本对照与治理组件（非价格操纵专报）

| 日期 | 2026-05-19 |
| **用途** | 静态分析 / Datalog **Ground Truth**；**不**作为可向漏洞平台提交的 PM 价格操纵报告 |

**语言：** 中文 · [English](../en/PM_Audit_Supplement_Controls.md)

---

## 1. YOLOToken — 负样本（False Positive Control）

| 字段 | 内容 |
|------|------|
| **地址** | `0xef8a4e034736c148ca591bc1b897f2729d38dbbe` |
| **对照部署** | `0x8c05a211d26baab73af5cd74cc2a4fa71f56d32a`（verified 源码 **1 行差异**，同 `ERC20WithYOLOMerkleTree`） |
| **PM 扫描** | 高 / 76% / `pm_vuln_count=3`（易因 `custom_price_function` 等关键词误命中） |

### 审计结论（价格操纵）

**不存在** 闪电贷 / AMM 现货 / 借贷清算型 **价格操纵** 面：

- **无** 外部 Uniswap/Curve `swap`、`slot0`、`get_dy`、零滑点 Router 清算；
- **无** 动态份额 NAV 与瞬时池价双轨铸币（cf. `Vault_0x71ea…`）；
- 核心为 OZ v5 **`_update`** 对隐私池地址的拦截：

```solidity
// ERC20WithYOLOMerkleTree — _update（节选）
if (from == SHIELDED_POOL || to == SHIELDED_POOL) {
    revert ShieldedPoolTransferBlocked();
}
```

### Datalog 剪枝建议（降误报）

规则应要求 **External Sink 至 Pair/Router/Oracle 且影响资产定价** 才报 `price_manipulation`；对仅含 **内部 `revert` 截断** 的 ERC20 钩子 **不报**。

```text
rule yolo_shielded_pool_fp_exclude:
  ContractName("YOLOToken") &
  InternalRevertOnly("ShieldedPoolTransferBlocked") &
  NotExists(ExternalPricingSink)
  -> Suppress("oracle_stale_exchange_price_mint")
```

**Ground Truth 标签：** `NEGATIVE_CONTROL / NO_PM_PRICE_MANIPULATION`

---

## 2. Box（QU!D 注册表）— 治理特权，非闪电贷 PM

| 字段 | 内容 |
|------|------|
| **地址** | `0xe3b6d810411774b1805e578f1fbc83f8ce9be323` |
| **合约** | **`Box`**（`src/Box.sol`；FundingModule / Timelock / Shutdown / Recover） |
| **PM 扫描** | 极高 / 92% / `pm_vuln_count=2` |

### 审计结论（价格操纵）

**不适合** 归类为与 MasterChef/Rover 同类的 **单区块现货三明治 PM**：

| 维度 | 说明 |
|------|------|
| **用户闪电贷面** | Box **不**承载 `swapExactTokensForTokens(..., 0)` 或 V3 `slot0` 铸份额逻辑；兑换经 **`ISwapper` + `maxSlippage` + NAV 缓存**（`flash and swap operations` 防操纵注释），属 **治理/运营** 路径 |
| **真实风险** | **中心化特权**：注册/下线 FundingModule、Timelock、`Shutdown`/`Recover`；Guardian/Admin 私钥泄露 → 资金路由被重定向 |
| **建议报告类型** | **Access control / Governance / Centralization**（另立项）；**非** 本目录下 PM 价格操纵模板 |

**Ground Truth 标签：** `GOVERNANCE_RISK / NOT_SPOT_PM_SINK`

---

## 3. 本批已并入主报告的同构实例（无需新稿）

| 类型 | 新增地址 | 并入 |
|------|----------|------|
| MasterChef | `0xdae1acc21ed8e26beb311edeb70e1ae5e27e8a0b` | [MasterChef](../en/eth_0x7f6229786703f01a8bfcede5f94c36179c467acd_MasterChef.md) |
| Rover | `0xb9ae20b2ed85e508c29881f24aa1308dd3751aff` | [Rover](../en/eth_0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97_Rover.md) |

---

## References

- `eth_0xef8a4e034736c148ca591bc1b897f2729d38dbbe.txt`
- `eth_0x8c05a211d26baab73af5cd74cc2a4fa71f56d32a.txt`
- `eth_0xe3b6d810411774b1805e578f1fbc83f8ce9be323.txt`
