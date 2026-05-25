# Vault（Naturelab）：Midas 预言机 + 滞后 `exchangePrice` 铸份额可被操纵

| 字段 | 内容 |
|------|------|
| **Target** | `0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3`（Ethereum Mainnet） |
| **Contract** | **`Vault`**（`VaultYieldBasic` + `StrategyMorphoBlue`；ERC4626 份额金库） |
| **Severity** | **High**（在 Midas / Morpho 预言机可被短时操纵前提下） |
| **Category** | Price manipulation / Custom oracle + Vault share pricing |
| **Reporter** | 赵志杰 |
| **Date** | 2026-05-19 |
| **Status** | Draft |

**语言：** 中文 · [English](../en/eth_0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3_Vault.md)

---

## Summary

本地址为 **Naturelab `Vault`**（约 10k 行 verified 工程，含 Morpho Blue 策略），**不是** QU!D/Vogue/Rover/Zap/MasterChef 模板。

金库份额铸造依赖 **`vaultState.exchangePrice`（再平衡员周期性更新的滞后净值价）**，而 **`MF_ONE` 存款**在入账时将抵押品按 **`IMidasOracle.lastAnswer()`** 实时折算为 USDC 名义金额，再调用 **`previewDeposit`**。二者不同步时，攻击者若在 **`lastAnswer` 被拉高** 且 **`exchangePrice` 尚未上调** 的窗口存入 `MF_ONE`，可按 **偏低的份额单价** 铸出 **过多份额**，稀释其他 LP。

`updateExchangePrice()` 的 **`MAX_PRICE_CHANGE_RATE`（1%）校验在源码中被注释掉**，再平衡员单次可将 `exchangePrice` 锚定到含预言机输入的 `underlyingTvl()`，若预言机可被同区块操纵，则 NAV 更新与份额体系均可被污染。策略层 **`getNetAssets()`** 亦通过同一 Midas 价将 `MF_ONE` 抵押品折算进 USDC 口径，并叠加 **Morpho `MORPHO_ORACLE.price()`** 头寸估值。

**Etherscan：** https://etherscan.io/address/0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3

**与 Vogue 关系：** **不同协议、不同源码树**；勿套用 `slot0` / `distributeBatchSwaps` / `swapAndLiquify` 叙事。

**PM 摘要：** `pm_likelihood_tier=极高`，`exploit_probability_pct=88`，`pm_vuln_count=3`。

---

## Affected scope

| 组件 | 地址（源码常量） | 角色 |
|------|------------------|------|
| **`Vault`** | `0x71ea…`（Target） | 用户存取款、份额铸造 |
| **`MIDAS_ORACLE`** | `0x8D51DBC85cEef637c97D02bdaAbb5E274850e68C` | `lastAnswer()` → MF_ONE 美元价 |
| **`StrategyMorphoBlue`** | 策略实例（部署时注册） | Morpho 借贷；`getNetAssets()` |
| **`MORPHO_ORACLE`** | `0x0cB1928EcA8783F05a07D9Ae2AfB33f38BFBEb78` | Morpho 市场抵押定价 |
| **`RedeemOperator`** | `vaultParams.redeemOperator` | 批量赎回、`confirmWithdrawal` |

**直接受影响函数：**

| 角色 | 函数 |
|------|------|
| 用户触发 | **`optionalDeposit` / `deposit`**（含 `MF_ONE` 路径） |
| 定价输入 | **`IMidasOracle.lastAnswer`**、`underlyingTvl()` |
| 份额铸造（火灾现场） | **`optionalDepositShares` → `previewDeposit` → `_mint`** |
| NAV 更新 | **`updateExchangePrice`**（`onlyRebalancer`） |
| 赎回 | **`optionalRedeem`**（`redeemOperator` + `_fairExchangePrice`） |

---

## Root cause

### 1. 双轨定价：滞后 `exchangePrice` vs 瞬时 `lastAnswer`

ERC4626 标准 `totalAssets()` 被覆写为 **缓存价**，而非实时 NAV：

```solidity
// VaultYieldBasic.sol — totalAssets（约 10009–10014 行）
function totalAssets() public view override returns (uint256) {
    if (block.timestamp - vaultState.lastUpdatePriceTime > vaultParams.maxPriceUpdatePeriod) {
        revert Errors.PriceNotUpdated();
    }
    return vaultState.exchangePrice * totalSupply() / PRECISION;
}
```

`MF_ONE` 存款则将代币 **按 Midas 即时价** 折算后进入 `previewDeposit`：

```solidity
// Vault.sol — optionalDepositShares（约 8405–8417 行）
function optionalDepositShares(address _token, uint256 _assets) internal view returns (uint256 shares_) {
    if (_token == MF_ONE) {
        int256 price_ = IMidasOracle(MIDAS_ORACLE).lastAnswer();
        if (price_ < 0) revert Errors.PriceBelowZero();
        _assets = _assets * uint256(price_) / 1e20;
    }
    // ...
    shares_ = previewDeposit(_assets);  // 分母：exchangePrice * totalSupply，非实时 NAV
    shares_ = shares_ * (10000 - instantFee) / 10000;
}
```

**间接清算：** 份额单价由 **`exchangePrice`（滞后的「净值汇率」）** 决定，而 **`_assets` 分子由 `lastAnswer`（瞬时预言机）** 放大；在预言机 **严重偏离 / 异常扭曲** 且 `exchangePrice` 未同步上调时，铸份额成本被人为压低。

### 2. 漏洞核心函数（触发源 vs 火灾现场）

> **无** `_transfer` / `swapAndLiquify` / Uniswap `slot0`。同构链为：**`optionalDeposit`（触发）→ `optionalDepositShares` + `previewDeposit`（火灾现场）**；NAV 侧为 **`updateExchangePrice` ← `underlyingTvl()`**。

| 泛化称谓 | 本合约实现 |
|----------|------------|
| 外层触发 | **`optionalDeposit` / `deposit`**（`PriceNotUpdated` 时间门控） |
| 定价读取 | **`IMidasOracle.lastAnswer()`**、`underlyingTvl()` |
| 份额铸造型 Sink | **`previewDeposit` / `_mint`**（基于滞后 `exchangePrice`） |
| 加池/ Router 零滑点 | **不适用**（用户路径无 `swapExact*`） |

#### 2.1 触发源：存款入口 — `if` 仅校验价格更新时间

```solidity
// Vault.sol — optionalDeposit（约 8420–8445 行，节选）
function optionalDeposit(address _token, uint256 _assets, address _receiver, address _referral)
    public payable override returns (uint256 shares_)
{
    if (depositPaused) revert Errors.NotSupportedYet();
    if (_token != USDC && _token != MF_ONE) revert Errors.InvalidAsset();
    IERC20(_token).safeTransferFrom(msg.sender, address(this), _assets);
    shares_ = optionalDepositShares(_token, _assets);  // → 火灾现场
    if (vaultParams.maxPriceUpdatePeriod < block.timestamp - vaultState.lastUpdatePriceTime) {
        revert Errors.PriceNotUpdated();
    }
    _mint(_receiver, shares_);
}
```

**说明：** 允许在 **`lastUpdatePriceTime` 有效但 `exchangePrice` 低于真实 NAV** 时存款；**`MF_ONE` 分支用实时预言机放大 `_assets`**，是主要套利缝。

#### 2.2 火灾现场：`previewDeposit` — 用滞后单价铸份额（Mechanism）

`optionalDepositShares` 对 `MF_ONE` 先将数量折成 USDC 名义资产，再经 ERC4626 `previewDeposit` 铸份额。记 `PRECISION = 10^6`（部署构造参数，源码中亦出现 `1e18` 尺度换算），在供给量充足时可抽象为：

$$
\text{Assets}_{\text{Midas}}
= \frac{Q_{\text{MF\_ONE}} \cdot \text{lastAnswer}}{10^{20}}
$$

$$
\text{TotalAssets}_{\text{Stale}}
= \frac{\text{ExchangePrice}_{\text{Stale}} \cdot \text{TotalSupply}}{10^{6}}
$$

$$
\text{Shares}
\approx \frac{\text{Assets}_{\text{Midas}} \times 10^{18}}{\text{ExchangePrice}_{\text{Stale}}}
$$

（工业界记账抽象式，突出 **分子 / 分母剪刀差**；链上 `totalAssets = \text{exchangePrice} \cdot \text{totalSupply} / 10^{6}`，`constructor` 中 `PRECISION = 10^{6}`，与上式在量纲归一后等价。严格实现为 `shares = assets · (totalSupply + offset) / (totalAssets + 1)`，见 `ERC4626Upgradeable.previewDeposit`。）

| 符号 | 含义 | 数据源 |
|------|------|--------|
| $\text{Assets}_{\text{Midas}}$ | 被 **`lastAnswer` 瞬时抬高** 的入账名义资产 | `IMidasOracle.lastAnswer()` |
| $\text{ExchangePrice}_{\text{Stale}}$ | **滞后、未随预言机同步** 的份额净值单价 | `vaultState.exchangePrice` |

当 $\text{lastAnswer}$ 相对真实市场 **瞬时异常扭曲** 偏高，而 $\text{ExchangePrice}_{\text{Stale}}$ 仍锚定旧 NAV 时，同一笔 `MF_ONE` 存款可得 **过大 $\text{Shares}$**，稀释既有持有人。

#### 2.3 NAV 更新：`underlyingTvl` 与被注释的涨跌幅上限

```solidity
// Vault.sol — underlyingTvl（约 8352–8360 行）
function underlyingTvl() public override returns (uint256) {
    uint256 usdcBal_ = IERC20(USDC).balanceOf(address(this));
    uint256 totalStrategy_ = totalStrategiesAssets();
    uint256 mFONEBal_ = IERC20(MF_ONE).balanceOf(address(this));
    int256 price_ = IMidasOracle(MIDAS_ORACLE).lastAnswer();
    if (price_ < 0) revert Errors.PriceBelowZero();
    return totalStrategy_ + usdcBal_ + (mFONEBal_ * uint256(price_) / 1e20) - vaultState.revenue;
}

// VaultYieldBasic.sol — updateExchangePrice（约 9909–9919 行，涨跌幅检查已注释）
// if (vaultState.exchangePrice - oldExchangePrice_ > oldExchangePrice_ * MAX_PRICE_CHANGE_RATE / 1e4) {
//     revert Errors.IncorrectState();
// }
```

> **本质说明：** 再平衡员调用 **`updateExchangePrice`** 时，**`newExchangePrice = underlyingTvl() * PRECISION / totalSupply`** 完全信任 **`lastAnswer` 与 `getNetAssets()`**；**1%/ 步长限制被注释**，单次更新可吸收 **被污染的 NAV**。策略 **`convertMFONEtoUSDC`** 与 **`getRatio`** 同样依赖 Midas / Morpho 预言机。

```solidity
// StrategyMorphoBlue.sol — convertMFONEtoUSDC（约 9331–9335 行）
function convertMFONEtoUSDC(uint256 _amountIn) internal view returns (uint256 _amountOut) {
    int256 rawPrice_ = IMidasOracle(MIDAS_ORACLE).lastAnswer();
    if (rawPrice_ < 0) revert Errors.PriceBelowZero();
    _amountOut = (_amountIn * uint256(rawPrice_)) / 1e20;
}
```

**结论：** 用户侧 **资金稀释** 发生在 **`optionalDepositShares` → `previewDeposit`**；协议侧 **NAV/汇率跳变** 发生在 **`updateExchangePrice` + 可操纵 `underlyingTvl()`**。

### 3. 赎回路径（次要，操作员信任）

`optionalRedeem` 由 **`redeemOperator`** 传入 **`_fairExchangePrice`** 结算（约 10120–10121 行），`confirmWithdrawal` 要求其与 **`exchangePrice()` ± `exchangePriceRate`（默认 300 bps）** 一致。若操作员被误导或恶意，可造成赎回端错付；**主操纵面仍为存款铸份额 + 预言机污染 NAV**。

---

## 攻击路径演练

**前提：** 攻击者可短时抬高 **`MIDAS_ORACLE.lastAnswer()`**（取决于 Midas 预言机实现，非本仓库源码）；金库 **`exchangePrice` 未在同区块被 `updateExchangePrice` 同步**。

### 第 1 步：准备

监控 **`optionalDeposit(MF_ONE, …)`**、`updateExchangePrice` 间隔，以及 `remainingUpdateTime() > 0` 的存款窗口。

### 第 2 步：污染定价输入

拉高 **`lastAnswer`**（或压低，用于其他组合套利），使 $\text{Assets}_{\text{Midas}}$ 相对 $\text{ExchangePrice}_{\text{Stale}}$ **严重偏离** 或 **瞬时异常扭曲**。

**表述规范：** 非 AMM `reserve` 公式，而是 **预言机 `lastAnswer` → 份额铸造型间接清算**。**禁止**使用「瞬间**安全**偏离」等措辞。

### 第 3 步：间接清算 + 份额铸造型套利（单区块或短窗口）

| 步骤 | 函数 | 后果 |
|------|------|------|
| 触发 | **`optionalDeposit(MF_ONE, …)`** | 转入 MF_ONE |
| 火灾现场 | **`optionalDepositShares` → `previewDeposit` → `_mint`** | $\text{Shares}$ 因剪刀差过大 |
| 可选后续 | **`updateExchangePrice`**（再平衡员） | 滞后价追平，攻击者份额已铸出 |

**物理过程（Mechanism）：**

1. **抬高** `lastAnswer` → $\text{Assets}_{\text{Midas}}$ 虚高。  
2. 在 $\text{ExchangePrice}_{\text{Stale}}$ 仍偏低时调用 **`optionalDeposit(MF_ONE)`**，按  
   $\text{Shares} \approx \dfrac{\text{Assets}_{\text{Midas}} \times 10^{18}}{\text{ExchangePrice}_{\text{Stale}}}$  
   铸出 **超额份额**（分母滞后、分子瞬时，形成定价剪刀差）。  
3. 其他 LP **份额被稀释**；攻击者可在预言机恢复前进入 Naturelab 原生赎回队列套现价差。

### 第 4 步：还原与收割（Naturelab 赎回路径）

预言机恢复、再平衡员 **`updateExchangePrice`** 后，攻击者已持有过高份额。套现需走本合约 **非标准 ERC4626 `redeem`** 流程，宜在 PoC 中直接调用：

| 路径 | 用户入口（`Vault.sol`） | 操作员侧 |
|------|-------------------------|----------|
| 慢赎 | **`requestRedeemSlow`** | **`IRedeemOperator.registerWithdrawal(..., false, epoch)`** → **`confirmWithdrawal`** |
| 快赎 | **`requestRedeemRapid`** | **`registerWithdrawal(..., true, epoch)`**（受 `redeemInstantLimit` 约束）→ **`confirmWithdrawal`** |
| 链上结算 | — | **`optionalRedeem(token, shares, _fairExchangePrice, …)`**（仅 `redeemOperator`；`_fairExchangePrice` 须落在 `exchangePrice ± exchangePriceRate`） |

**收割闭环：** 存款阶段用 Midas 价放大分子铸份额 → 赎回阶段经 **`requestRedeem*` + `optionalRedeem`** 按操作员确认的 `_fairExchangePrice` 取回 USDC（或 **`cancelRedeem`** 回滚排队份额）→ 在 **`underlyingTvl` 重定价** 前完成套利，稀释诚实 **`deposit` / `optionalDeposit(USDC)`** 持有人。

---

## 资金外流 / 稀释受灾函数

| 函数 | 源文件（约行） | 后果 |
|------|----------------|------|
| **`Vault.optionalDeposit` / `optionalDepositShares`** | `Vault.sol` ~8405+ | **过量铸份额** |
| **`VaultYieldBasic.totalAssets` / `previewDeposit`** | ~10009+ | 分母用滞后 `exchangePrice` |
| **`Vault.underlyingTvl`** | ~8352 | NAV 含可操纵 `lastAnswer` |
| **`VaultYieldBasic.updateExchangePrice`** | ~9889 | 单次大幅调价（涨跌幅检查注释） |
| **`StrategyMorphoBlue.getNetAssets`** | ~9214 | 策略资产计入 NAV |
| **`StrategyMorphoBlue.getRatio`** | ~9172 | Morpho 健康度用 `MORPHO_ORACLE.price()` |

---

## Impact

| 维度 | 说明 |
|------|------|
| **Financial** | 诚实 USDC/`MF_ONE` 存款人被 **份额稀释**；NAV 更新滞后放大套利窗 |
| **Integrity** | `exchangePrice` 与 Midas 现货价脱钩；注释掉的 **1% 步长** 限制削弱再平衡安全阀 |
| **Availability** | 预言机长期不更新会 `PriceNotUpdated` 暂停存款（设计内），但不消除窗口套利 |

**Severity：High** — 在预言机可操纵假设下，无需特权即可 **`optionalDeposit`**；PM 88% / 极高。

---

## Proof of concept（思路级）

1. Fork 主网，绑定 `Vault`、`MIDAS_ORACLE`、`MF_ONE`。  
2. 记录 `exchangePrice()`、`lastAnswer()`、`previewDeposit(convertMFONE(1e18))`。  
3. 模拟抬高 `lastAnswer`（或 fork 操纵交易），在同一 `lastUpdatePriceTime` 窗口调用 **`optionalDeposit(MF_ONE, …)`**。  
4. 断言攻击者 `shares` > 公平 NAV 下铸份额；对比仅 USDC `deposit` 基准。  
5. （可选）走 **`requestRedeemRapid` → `confirmWithdrawal` → `optionalRedeem`** 验证赎回端能否完成套利闭环。

---

## Recommendations

1. **统一定价源：** `previewDeposit` 对 `MF_ONE` 使用与 **`updateExchangePrice` 相同的 NAV 公式**（或 Chainlink/TWAP 双源取 min）。  
2. **恢复并强制执行 `MAX_PRICE_CHANGE_RATE`**；预言机与 `exchangePrice` 偏差 > ε 时 revert 存款。  
3. **`MF_ONE` 存款：** 要求再平衡员先 **`updateExchangePrice`** 再开放 `MF_ONE`，或采用 **commit-reveal / 时间加权价**。  
4. **Morpho / Midas 预言机：** 审计 `0x8D51…` / `0x0cB19…` 是否现货池；改用 TWAP 或熔断。  
5. **`optionalRedeem`：** 链上绑定 `previewRedeem(exchangePrice)` 下限，降低 operator 单点风险。

---

## 第二部分：Datalog 规则优化（静态分析 / PM 检测）

| 规范术语 | 本合约映射 |
|----------|------------|
| **TaintedPricingRead** | `IMidasOracle.lastAnswer`, `IOracle(MORPHO_ORACLE).price` |
| **StaleSharePriceDenominator** | `totalAssets() = exchangePrice * totalSupply / PRECISION` |
| **CompositeSink** | `optionalDepositShares` → `previewDeposit` → `_mint` |
| **DisabledGuard** | 注释掉的 `MAX_PRICE_CHANGE_RATE` revert |

```text
rule vault_midas_oracle_share_mint_sink:
  EntryPoint("Vault", {"optionalDeposit", "deposit"}) &
  BranchPredicate("token == MF_ONE") &
  TaintedPricingRead(
    Callee: "IMidasOracle",
    Selector: "lastAnswer()",
    RetFlowInto: "optionalDepositShares._assets"
  ) &
  ShareMintSink(
    Callee: "ERC4626",
    Selector: "previewDeposit(uint256)",
    DenominatorUses: "vaultState.exchangePrice"
  )
  -> Vuln("oracle_stale_exchange_price_mint", severity=High)

rule vault_exchange_price_step_guard_disabled:
  Function("VaultYieldBasic.updateExchangePrice") &
  CommentedOutRevert("MAX_PRICE_CHANGE_RATE")
  -> Vuln("nav_update_no_rate_limit", severity=Medium)
```

**符号约定：** 规则名以 `rule` 声明（**无** Souffle 不支持的行首 `.rule` 前缀）；`&` 为合取，`->` 为结论箭头。

**勿**将 Morpho `liquidate` 接口命中单独标为 Router 零滑点；本合约主风险为 **预言机 + 份额定价双轨**。

---

## References

- 源码：`pm_audit_tier_high_and_extreme/eth_0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3.txt`
- PM 元数据：`eth_0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3.meta.txt`
- 对比（不同协议）：`Vogue.md`、`MasterChef.md`、`Zap.md`

---

## Disclosure timeline

| 日期 | 事件 |
|------|------|
| 2026-05-19 | PM 扫描 + 源码审计 |
| （待填） | 厂商联系 |
| （待填） | 修复验证 |
