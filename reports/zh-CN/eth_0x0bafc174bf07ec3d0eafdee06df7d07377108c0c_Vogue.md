# Vogue / QU!D 协议：内部 Uniswap V4 现货价格被操纵导致兑换与 LP 会计失真

| 字段 | 内容 |
|------|------|
| **Target（主实例）** | `0x0bafc174bf07ec3d0eafdee06df7d07377108c0c`（Ethereum Mainnet） |
| **同构 Vogue 部署** | `0xb3c368…`、`0x19f4b121…`、`0x9743de…`、`0x32956d…` 等（见 **§已验证部署实例**） |
| **Contract** | `Vogue`（协议核心还包括 `Aux`、`VogueCore`、`Basket`、`Rover` 等，见下文） |
| **Severity** | **High** |
| **Category** | Price manipulation / Spot oracle（可操纵现货价） |
| **Reporter** | 赵志杰 |
| **Date** | 2026-05-19 |
| **Status** | Draft |

**语言：** 中文 · [English](../en/eth_0x0bafc174bf07ec3d0eafdee06df7d07377108c0c_Vogue.md)

---

## Summary

Vogue（QU!D 篮子协议的一部分）在结算用户兑换、批量清算（ASS）、LP 存取款及杠杆逻辑时，**统一使用来自协议自建 Uniswap V4 池（`VogueCore.VANILLA`，mock ETH/USD）的瞬时 `sqrtPriceX96`**，经 `Aux.getPrice()` 转成美元计价，而**未使用 TWAP、Chainlink 等抗操纵预言机**，也未与外部主池（如 WETH/USDC Uniswap V3）价格做有效锚定校验。

攻击者可在单区块内通过闪电贷或自有资金，在 V4 池上执行大额 `swap`，拉高/压低内部现货价，再触发受害者的 `Aux.swap()`、`clearSwaps()`、`deposit()` / `withdraw()` 或 `leverETH()` / `leverUSD()`，使受害者按失真价格成交或分配，从而**窃取兑换输出、稀释 LP、或不当占用 `POOLED_ETH` / `POOLED_USD` 储备**。小额兑换路径（&lt; 约 $5000）甚至**明确不做防三明治保护**，进一步放大 MEV/抢跑风险。

## Affected scope

| 组件 | 角色 |
|------|------|
| **`Vogue`** (`0x0bafc174bf07ec3d0eafdee06df7d07377108c0c`) | 部署入口；LP、批量 swap 队列、`distributeBatchSwaps` |
| **`Aux`** | 用户-facing `swap()`、`clearSwaps()`、价格 `getPrice()` |
| **`VogueCore`** | Uniswap V4 `poolManager.swap` / `batchSwap`；维护 `POOLED_ETH` / `POOLED_USD` |
| **`Rover`** | 辅助 V3 流动性与 `getPrice()`（同样基于 `slot0` 现货） |

**主要受影响函数（用户或清算者可触发）：**

- `Aux.swap()` — 即时兑换（小额）与排队（大额）
- `Aux.clearSwaps()` — 公开可调的批量清算
- `Vogue.distributeBatchSwaps()` — 按操纵价比例分配批次输出
- `Vogue.deposit()` / `withdraw()` — 使用 `AUX.getPrice(sqrtPriceX96)` 配对流动性
- `Aux.leverETH()` / `Aux.leverUSD()` — 杠杆开仓依赖同一价格
- `VogueCore.swap()` / `batchSwap()` — 内部 V4 池成交，价格由池状态决定

**Etherscan（主实例）：**  
https://etherscan.io/address/0x0bafc174bf07ec3d0eafdee06df7d07377108c0c

---

## 已验证部署实例（Vogue 组件）

下列地址经 Etherscan verified 源码比对，**价格操纵机理与本报告相同，无需另写报告**。提交时将 **Target** 换为对应地址即可（链上 `POOL` / `Aux` 等构造参数以该实例为准）。

| 地址 | 说明 |
|------|------|
| **`0x0bafc174bf07ec3d0eafdee06df7d07377108c0c`** | **主实例（本报告 Target）** |
| **`0xb3c368bc4033361c2d9a2568d32ea8dd585b7fec`** | 与主实例源码同构（0 行差异） |
| **`0x19f4b1215456f706475e0ee408d216ebb181129a`** | 同源变体（`VogueCore` 等同栈；`Amp.leverETH` 等无 `onlyUs`） |
| **`0x9743de2355d3840d52529503d496bf33ff7a9793`** | 与 `0x19f4b121…` 完全相同 |
| **`0x32956d93910feb9dfb80e965cec248bda58eb5fe`** | 扩展变体（11 个 `src/` 文件补丁差异；核心 PM 面同构） |

**源码：** `Etherscan verified source (see contract address above)`

---

## Root cause

### 1. 定价源为可操纵的 V4 池现货

`Aux.getPrice()` 将传入的 `sqrtPriceX96`（通常来自 `V4.repack()` → `VogueCore.poolStats()`，即 **V4 内部池 slot0**）换算为 ETH/USD 价格；仅在 `sqrtPriceX96 == 0` 时回退读取外部 `v3PoolWETH.slot0()`，正常运行几乎总是 **V4 内部价**。

```solidity
// Aux.sol — getPrice
function getPrice(uint160 sqrtPriceX96) public view returns (uint price) {
    if (sqrtPriceX96 == 0)
        (sqrtPriceX96,,,,,, ) = v3PoolWETH.slot0();
    // ... FullMath 由 sqrtPriceX96 推导 USDC per WETH ...
}
```

`Aux.swap()` 在每次兑换前调用 `V4.repack()` 取得当前 `sqrtPriceX96`，随后：

- **大额**（`sensitive`）：进入批次队列，由 `clearSwaps()` 清算；
- **小额**：**立即** `CORE.swap(sqrtPriceX96, ...)`，注释写明无三明治保护。

```solidity
// Aux.sol — swap（节选）
(uint160 sqrtPriceX96,,,) = V4.repack();
uint price = getPrice(sqrtPriceX96);
// ...
if (sensitive) {
    blockNumber = V4.pushSwap(zeroForOne, current, waitable);
} else {
    // Executes instantly, no batching, no sandwich protection...
    CORE.swap(sqrtPriceX96, msg.sender, zeroForOne, token, amount);
}
```

`VogueCore` 的 swap 使用同一 `sqrtPriceX96` 构造 `sqrtPriceLimitX96`（`paddedSqrtPrice`，仅 ±3% / ±10% 带宽），**限制的是相对该（可操纵）参考价的偏离，而非市场真实价格**。

```solidity
// VogueCore.sol — _handleSwap（节选）
BalanceDelta delta = poolManager.swap(VANILLA,
    IPoolManager.SwapParams({
        zeroForOne: forOne,
        amountSpecified: -int(amount),
        sqrtPriceLimitX96: VOGUE.paddedSqrtPrice(sqrtPriceX96, !forOne, 300)
    }), ZERO_BYTES);
```

### 2. 批量清算使用同一操纵价，且仅 2% 聚合容差

`clearSwaps()` 为 **external、无访问控制**；任何人可在操纵 V4 价格后调用。清算时用 `getPrice(sqrtPriceX96)` 计算应付 ETH/USD，并调用 `Vogue.distributeBatchSwaps()`。

批次分配仅要求**实际成交量**与「按 `getPrice` 算出的期望值」相差 ≤ **2%**（`delta <= swappedForUSD / 50`），**并不验证 `getPrice` 本身是否等于市场价格**：

```solidity
// Vogue.sol — distributeBatchSwaps（节选）
require(stdMath.delta(swappedForUSD, FullMath.mulDiv(
    forETH.total * 1e12, WAD, AUX.getPrice(sqrtPriceX96))) <= swappedForUSD / 50);
```

因此攻击者可在操纵价格的同时满足 2% 执行容差，使排队用户按错误比例获得 ETH/稳定币。

### 3. LP 与储备会计绑定失真价格

`deposit()` / `addLiquidityHelper()` 使用 `price = AUX.getPrice(sqrtPriceX96)` 决定 ETH/USD 配对及 `V4.modLP()` 数量；`POOLED_ETH` / `POOLED_USD` 在 `VogueCore._handleDelta` 中随 swap 更新。操纵价可导致：

- 攻击者低价「存入」获得过高份额或过高 `pooled_eth` 记账；
- 诚实 LP 提款时按错误比例承担损失。

### 4. 内部 V4 池与外部市场脱钩

`VogueCore.setup()` 用部署时 V3 `slot0` **一次性**初始化 V4 池，之后 V4 与主市场 WETH/USDC 池**独立演化**。低流动性 mock 池更易被单笔大额 `swap` 推动价格，而协议逻辑仍信任该价格指导主业务。

## 攻击路径演练

**前提：** 攻击者持有闪电贷或大额 WETH/稳定币；协议定价依赖可被动结的 AMM 现货（`Aux.getPrice` ← `sqrtPriceX96` / `slot0`），且 V4 内部池 `VANILLA` 流动性相对主市场更薄。

### 第 1 步：准备

监听 mempool 中即将执行的 `Aux.swap()`（敏感批次）、`Aux.clearSwaps()`，或 `Vogue.deposit()` / `Vogue.withdraw()` / `Aux.redeem()` 等会读取当前价格的交易。

### 第 2 步：污染 AMM 瞬时储备（砸盘 / 拉盘）

攻击者在同一区块内向 **V3 参考池或 V4 内部 `VANILLA` 池** 大额 `swap`，污染瞬时储备。

**表述修正：** 合约**未**显式实现 `price = (reserve1 * 1e18) / reserve0`。这是 **间接清算**：结算由 **`Aux.getPrice`（读 `slot0` / V4 状态）** 与 **Router / `VogueCore.swap` 在缺乏有效最小输出保护时隐式采用的瞬时兑换比率** 共同决定。

可抽象为被污染的池内比率（示意）：

$$
\frac{\text{Reserve}_{\text{稳定币}}}{\text{Reserve}_{\text{ETH}}}
$$

砸盘时该比率 **严重偏离 / 恶意扭曲** 公允价（勿写作「安全偏离」）。`sqrtPriceX96` 与储备比例单调对应，操纵池子即操纵 `getPrice` 输入。

> **说明：** `0x0bafc…` **无** `getVoguePrice()`、**无** `_transfer(to==pair)` → `swapAndLiquify`、**无** Tax/Liquidity Fee 收纳换币。定价入口为 **`Aux.getPrice`** / **`VogueCore.swap`**（`src/Aux.sol`、`src/VogueCore.sol`）。

### 第 3 步：间接清算 + 滑点保护不足（业务层收割）

**间接清算型**漏洞：在污染比率下，**原生代码未对 swap/批次分配设置有效下限**（如 `paddedSqrtPrice` 仅相对已操纵价带宽、`distributeBatchSwaps` 仅 2% 执行容差），被迫完成财务结算。

**受害的具体函数（非泛化「deposit / liquidation」）：**

- **`Aux._clearSwaps` → `VogueCore.batchSwap` → `Vogue.distributeBatchSwaps`**  
- **`Aux.swap`（即时）→ `VogueCore.swap`**  
- **`Vogue.deposit` / `addLiquidityHelper`**

详见 **「资金外流受灾函数」** 表。**不是** `_transfer` 被动触发的 `swapAndLiquify` 手续费滑点流失。

### 第 4 步：还原

攻击者反向交易恢复池价，偿还闪电贷，保留价差。

**变种：** `Aux.swap` 小额路径（&lt; 约 $5000 等值）走即时 `CORE.swap`，源码注释标明 **无 sandwich 保护**，便于同区块夹攻。

---

## 资金外流受灾函数（已验证源码，非泛化描述）

下列函数在实现中 **直接调用 `Aux.getPrice(sqrtPriceX96)` 或在其后立即按 `price` 转账/铸份额/清算批次**。价格被操纵时，这些路径会造成 **真实资产错配外流**（而非仅「潜在风险」）：

| 函数 | 源文件（约行） | 资金后果 |
|------|----------------|----------|
| **`Aux.getPrice`** | `src/Aux.sol` ~7726 | 定价中枢；`sqrtPriceX96==0` 时读 `v3PoolWETH.slot0()`，否则用 V4 `repack` 价 |
| **`Aux._clearSwaps`** | `src/Aux.sol` ~7834 | 批量清算：用 `price` 拆分 `POOLED_ETH/USD`、调用 `V3.take` / `wethVault.deposit`，再 `CORE.batchSwap` |
| **`Vogue.distributeBatchSwaps`** | `src/Vogue.sol` ~11148 | 按 `AUX.getPrice(sqrtPriceX96)` 比例分配批次；`FullMath.mulDiv(..., price, WAD)` 结算后 **`_sendETH`**、**`AUX.take(..., USDC)`** 转出资产 |
| **`Aux.swap`**（即时分支） | `src/Aux.sol` ~7777–7817 | `V4.repack()` 后立刻 **`CORE.swap(sqrtPriceX96, ...)`**，用户按扭曲价成交 |
| **`Vogue.deposit`** | `src/Vogue.sol` ~11250 | `price = AUX.getPrice(sqrtPriceX96)` → **`addLiquidityHelper`** → **`V4.modLP`**，错误价稀释其它 LP |
| **`Vogue.withdraw`** | `src/Vogue.sol` ~11203 | 提款与 **`V4.modLP` / `_sendETH`**，按失真池状态结算 |
| **`Aux._take` / `take`** | `src/Aux.sol` ~8102–8159 | 从 ERC4626 稳定币金库 **`redeem`** 至用户；赎回量依赖 `get_metrics` / 定价上下文 |
| **`Aux.redeem`** | `src/Aux.sol` ~7979 | 销毁 QUID 后 **`QUID.turn` + `_take`** 支付稳定币，可多付/少付 |
| **`Aux.leverETH` / `leverUSD`** | `src/Aux.sol` ~7916–7974 | 杠杆开仓用 `getPrice` 计算抵押与借款额 |
| **`VogueCore._handleSwap` / `_handleBatchSwap`** | `src/VogueCore.sol` ~11618–11661 | 内部 V4 成交，更新 **`POOLED_ETH` / `POOLED_USD`**，限额仅相对已操纵的 `sqrtPriceX96` |

**典型资金归零/枯竭机制（对应本合约，而非泛化「deposit/liquidation」）：**

1. **`distributeBatchSwaps`**：批次 ETH 侧用户应得 USD 输出为 `mulDiv(swappedForUSD, userAmt, forETH.total)`，而 `swappedForUSD` 与 **`getPrice` 锚定**；砸盘使 `price` 极低时，协议仍可能通过 2% 容差检查，但用户 **按失真比例** 领取稳定币，攻击者通过前置/后置套利抽走金库差额。  
2. **`_clearSwaps` → `_take` / `wethVault` / `V3.withdrawUSDC`**：清算逻辑用同一 `price` 决定从 **`wethVault`、稳定币 vault、V3 Rover** 调出多少资产完成批次，错误价导致 **多付稳定币或少收 ETH**。  
3. **`Vogue.withdraw` + `_sendETH`**：LP 退出时按操纵后的池状态与 **`modLP`** 结算，可 **超额提取** `wethVault` 中 ETH。

若对比「税费代币」类 Vogue 实现：该类合约在 **`_transfer(..., to == pair)`** 卖出分支调用 **`getVoguePrice()`** 做回购/分红折算，会把合约内 **USDT 等储备** 按错误比例送出。**本地址部署的 verified 源码无此模式**；上表为 **`0x0bafc…` 上实际存在的受害路径**。

## Impact

| 维度 | 说明 |
|------|------|
| **Financial** | 通过 **`distributeBatchSwaps`、`_clearSwaps`、`deposit`/`withdraw`、`redeem`、`_take`** 等路径，可导致 **ETH、USDC 及稳定币金库（ERC4626 vault）** 以错误汇率流出；批次用户、即时 `swap` 用户、`autoManaged` LP 及 `leverETH`/`leverUSD` 头寸均可蒙受可量化损失。 |
| **Integrity** | 破坏协议「按美元公允价值」记账与分配不变量；`POOLED_*` 与 `totalShares` 关系可被短时扭曲。 |
| **Availability** | `clearSwaps()` 可被用于配合操纵的定时清算，不直接导致停机，但可反复利用。 |

**Severity 定为 High 的理由：** 无需特权账户；价格操纵为链上公开可验证模式；影响核心兑换与 LP 资金；在流动性不足时单笔闪电贷攻击即可获利。

## Proof of concept

**状态：** 概念验证思路（Foundry fork 主网可实施）；完整 PoC 脚本待补充。

**复现要点：**

1. Fork 主网，加载已部署 `Vogue` / `Aux` / `VogueCore` 地址（从链上 `setup` 事件或存储槽解析关联地址）。
2. 记录 `VogueCore.poolStats()` 的 `sqrtPriceX96` 与 `Aux.getPrice()`。
3. 闪电贷 WETH → 循环 `Aux.swap()` 小额或单笔大额影响 V4 池 → 再次读取价格，确认相对 Chainlink / V3 TWAP 偏离 &gt; 1%。
4. 在同一区块模拟受害者 `Aux.swap(token, forETH, amount, waitable)` 或触发 `clearSwaps()`，对比公平价格下 `balanceOf` 差异。

```solidity
// PoC 骨架（伪代码）
function test_price_manipulation() public {
    uint160 priceBefore = poolStats_sqrtPrice();
    manipulateV4Pool(/* push ETH price down */);
    uint160 priceAfter = poolStats_sqrtPrice();
    assertGt(relativeDeviation(priceBefore, priceAfter), 0.01e18);

    uint victimOut = victimSwap(...);
    uint fairOut = quoteFairMarket(...);
    assertLt(victimOut, fairOut * 99 / 100);
}
```

## Recommendations

1. **不要使用 V4 内部池 `slot0` 作为唯一定价源。** 对结算、批次分配、LP 铸份额使用：
   - Uniswap V3 **TWAP**（`OracleLibrary`）或 Chainlink 等外部预言机；或
   - 要求 `getPrice` 同时读取 `v3PoolWETH` TWAP 与 V4 价，偏差超过阈值（如 0.5%–1%）则 revert。
2. **即时 swap 路径增加防护：** 强制最小批次延迟、或与主池 TWAP 绑定的 `sqrtPriceLimitX96`；对小额也启用 ASS 或 commit-reveal。
3. **`clearSwaps()` 增加价格健全性检查：** 清算前断言 `getPrice(sqrtPriceX96)` 与 TWAP/Chainlink 偏差 &lt; ε；考虑限制调用频率或仅允许受信 keeper（权衡去中心化）。
4. **`distributeBatchSwaps`：** 除 2% 执行滑点外，增加**预言机边界**校验；按用户子订单分别限价，而非仅全局聚合检查。
5. **监控与熔断：** 单区块内 `sqrtPriceX96` 变化超过阈值时暂停 `swap`/`clearSwaps`。
6. **部分部署（如 `0x19f4b121…` / `0x9743de…`）：** `Amp.leverETH` / `leverUSD` 恢复 `onlyUs`，与 `0x0bafc…` 对齐。

## 第二部分：Datalog 规则优化（静态分析 / PM 检测）

| 规范术语 | 含义 |
|----------|------|
| **Control-flow branch → Post-dominated External Sink** | `Aux.swap` 内 `sensitive` 分支不单独报；需追到 `CORE.swap` / `_clearSwaps` |
| **TaintedPricingRead(`getPrice`, `slot0`, `repack`)** | 污染定价输入 |
| **CompositeSink(`_clearSwaps`)** | 剪枝 `path_explosion_possible_fp` |
| **DeployVariantMissingModifier(`onlyUs`)** | 如 `0x19f4b121…` 部署上 `Amp.leverETH` 跨实例差异 |

```text
rule quid_vogue_v4_batch_or_swap_sink:
  EntryPoint("Aux", {"swap", "clearSwaps"}) &
  PostDominatedBy(InternalCallee, {"_clearSwaps", "VogueCore.swap"}) &
  TaintedPricingRead("Aux.getPrice", "sqrtPriceX96") &
  (
    ExternalSink("VogueCore", "batchSwap") &
    ExternalSink("Vogue", "distributeBatchSwaps", ArgAnchor="AUX.getPrice")
  |
    ExternalSink("VogueCore", "swap", SlippageBandOnly=true)
  )
  -> Vuln("indirect_settlement_v4_spot", severity=High)

rule quid_amp_missing_only_us_variant:
  DeployAddress({"0x19f4b1215456f706475e0ee408d216ebb181129a",
                 "0x9743de2355d3840d52529503d496bf33ff7a9793"}) &
  Function("Amp.leverETH", ModifierMissing="onlyUs")
  -> Vuln("access_control_expanded_price_sink", severity=Medium)
```

**符号约定：** 规则名以 `rule` 声明（无行首 `.rule`）；`&` 为合取，`->` 为结论。

## References

- 本地已验证源码（主实例）：`Etherscan verified source`
- 同构 / 变体实例：`eth_0xb3c368bc4033361c2d9a2568d32ea8dd585b7fec.txt`、`eth_0x9743de2355d3840d52529503d496bf33ff7a9793.txt`、`eth_0x19f4b1215456f706475e0ee408d216ebb181129a.txt`、`eth_0x32956d93910feb9dfb80e965cec248bda58eb5fe.txt`
- 关键文件：`src/Aux.sol`、`src/Vogue.sol`、`src/VogueCore.sol`、`src/Amp.sol`
- 同栈 Rover：`../en/eth_0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97_Rover.md`
- 内部 PM 审计：`internal PM audit metadata`（上述 Vogue 部署多为：高，70%，`pm_vuln_count` 4–7）

---

## Platform checklist

- [x] 标题与 Summary 一致
- [x] 地址与链正确
- [ ] PoC 可独立复现（待补 Foundry fork 测试）
- [x] Impact 与 High 定级匹配
- [x] 修复建议可执行
