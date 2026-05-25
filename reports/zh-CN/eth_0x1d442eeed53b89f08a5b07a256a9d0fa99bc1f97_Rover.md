# Rover / QU!D 协议：Uniswap V3 现货 `slot0` 定价被操纵导致 LP 份额与金库资产错配

| 字段 | 内容 |
|------|------|
| **Target（主实例）** | `0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97`（Ethereum Mainnet） |
| **同构 Rover 部署** | **`0x491c3a…`**、**`0x1d8227…`**、**`0xb9ae20…`**（见下表） |
| **Contract** | **`Rover`**（Uniswap V3 流动性包装 / MasterChef 式份额；与 `Aux`、`Vogue`、`Amp` 等同栈集成） |
| **Severity** | **High** |
| **Category** | Price manipulation / Spot oracle（可操纵 AMM 现货价） |
| **Reporter** | 赵志杰 |
| **Date** | 2026-05-19 |
| **Status** | Draft |

**语言：** 中文 · [English](../en/eth_0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97_Rover.md)

---

## Summary

本地址部署的是 **`Rover`** 合约：围绕固定 **`POOL`（WETH/USDC Uniswap V3）** 管理 NFPM 头寸与 LP 份额。所有关键路径通过 **`IUniswapV3Pool(POOL).slot0()`** 读取瞬时 `sqrtPriceX96`，再经 **`Rover.getPrice()`** 换算为「USDC 计价的 ETH 价格」，用于 **`_swap` 定比`、`deposit`/`withdraw` 铸烧份额、以及向 `Aux`/`Amp` 提供流动性**。

该设计**未使用 TWAP / Chainlink**。致命链为 **`fetch`/`deposit` → `_repackNFT` → `_swap`（Router `amountOutMinimum = 0`）→ `NFPM.mint`（`amount0Min/amount1Min = 0`）**——对应税费代币类报告中的 `swapAndLiquify` + 无滑点卖币/加池。在现货价被闪电贷砸盘或拉盘后，攻击者可：

- 以失真价格 mint 过多 `positions[msg.sender].liq` 份额；
- 在 `withdraw` 时按错误 `price` 合成 `expected` 并从 **`Amp`** 补差提取超额 ETH；
- 在 `Aux._clearSwaps` 调用 **`V3.take` / `V3.withdrawUSDC`** 时，按操纵价向协议 **转出 WETH/USDC**。

同栈 **`Aux.getPrice`** 在 `sqrtPriceX96 == 0` 时亦回退同一类 V3 `slot0` 读价，故 Rover 与 Vogue 批次清算存在**交叉风险**（本报告以 **Rover 部署实例** 为主）。

**Etherscan（主实例）：** https://etherscan.io/address/0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97

---

## 已验证部署实例（Rover 组件）

下列地址 verified 源码**逐字节同构**（0 行差异），机理与本报告相同，**无需另写报告**。提交时将 **Target** 换为对应地址，并从该实例读取 `POOL` / `ROUTER` / `NFPM` / `AUX` 等构造参数。

| 地址 | 说明 |
|------|------|
| **`0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97`** | **主实例（本报告 Target）** |
| **`0x491c3a404ff9564a4a09cf777c833c4d138801e6`** | 同构部署 |
| **`0x1d8227d7cdabce4ac80182857bc0de28645fae53`** | 同构部署 |
| **`0xb9ae20b2ed85e508c29881f24aa1308dd3751aff`** | 同构部署 |

**源码：** `Etherscan verified source (see contract address above)`

---

## Affected scope

| 组件 | 与本地址关系 |
|------|----------------|
| **`Rover`** (`0x1d442…`) | **本报告 Target**；持有 V3 NFT 头寸、`totalShares` / `positions` |
| **`Aux`** | `onlyUs` 调用方；`setAux` 后通过 `V3.take`、`V3.withdrawUSDC`、`V3.depositUSDC`、`V3.repackNFT` 调度 Rover |
| **`Amp`** | `leverETH` / `leverUSD` / 平仓逻辑依赖 `V3.getPrice(V3.repackNFT())` |
| **`Vogue` / `VogueCore`** | 同仓库源码一并验证；批次 swap 与 V4 池逻辑见 `Vogue.md` |

**Rover 合约内直接受影响函数：**

- `getPrice` / `fetch` / `repackNFT` — 定价与再平衡入口  
- `deposit` / `withdraw` — 用户 LP 进出  
- `_swap` — 内部定比 + Router 成交（零滑点下限）  
- `take` / `depositUSDC` / `withdrawUSDC` — 仅 `Aux`/`Amp` 可调用，清算时外流资产  

---

## Root cause

### 1. 现货定价：`slot0` → `getPrice`，无抗操纵机制

`slot0().sqrtPriceX96` 为**单区块瞬时价**，可被同池大额 `swap` 任意扭曲。`getPrice` 仅做平方换算，**不与 TWAP、预言机或跨池中位数比对**。

```solidity
// Rover.sol — getPrice（约 10616–10627 行）
function getPrice(uint160 sqrtRatioX96) public view returns (uint price) {
    uint256 ratioX128 = FullMath.mulDiv(casted, casted, 1 << 64);
    if (token1isWETH)
        price = FullMath.mulDiv(1 << 128, WAD * 1e12, ratioX128);
    else
        price = FullMath.mulDiv(ratioX128, WAD * 1e12, 1 << 128);
}
```

### 2. 漏洞核心函数（触发源 vs 火灾现场）

> **对照说明：** 税费代币报告里常见「`_transfer` → `swapAndLiquify` → `swapTokensForEth` + `addLiquidity`」。**本地址 verified 源码无上述函数**；**等价致命链**为「`fetch`/`deposit` 触发 → `_repackNFT` → `_swap`（Router 卖出）→ `NFPM.mint`（加流动性）」。下表为平台审稿用的**直接对应**。

| 泛化称谓（税费代币类） | 本合约 `Rover.sol` 中的实际实现 |
|------------------------|----------------------------------|
| 外层触发 `_transfer` | **`fetch`** / **`deposit`** / **`repackNFT`**（读 `slot0` 后进入再平衡） |
| 内层 `swapAndLiquify` | **`_repackNFT`**：先 `_swap` 再 `NFPM.mint` |
| `swapTokensForEth`（卖币换 ETH，无 `amountOutMin`） | **`_swap`** 内 **`ISwapRouter.exactInput(..., amountOutMinimum: 0)`** |
| `addLiquidity`（无 `amountAMin`/`amountBMin`） | **`NFPM.mint` / `increaseLiquidity`** 且 **`amount0Min: 0, amount1Min: 0`** |

#### 2.1 触发源：读操纵价并进入再平衡

```solidity
// Rover.sol — fetch（约 10514–10520 行）：仅“拉闸”，真正失血在 _repackNFT 内
function fetch(address beneficiary) public returns (Deposit memory, uint, uint160) {
    (uint160 sqrtPrice, int24 tick,,,,,) = IUniswapV3Pool(POOL).slot0();
    LAST_TICK = tick;
    uint price = getPrice(sqrtPrice);
    _repackNFT(0, 0, price);  // ← 进入下方“火灾现场”
    return (LP, price, sqrtPrice);
}

// Rover.sol — deposit（约 10817–10845 行）：用户入口同样经 fetch → _swap → _repackNFT
function deposit(uint amount) external nonReentrant payable {
    (Deposit memory LP, uint price, uint160 sqrtPrice) = fetch(msg.sender);
    // ...
    (amount, in_dollars) = _swap(amount + msg.value, 0, price);
    // ...
    _repackNFT(amount0, amount1, price);
}
```

#### 2.2 火灾现场 A：`_swap` — 等价于无保护的 `swapTokensForEth`

`_swap` 在扭曲的 `price` 下计算 `kp`/`targetETH`，并通过 **Uniswap V3 SwapRouter** 成交，但 **`amountOutMinimum` 硬编码为 0**（任意恶劣输出均不 revert）：

```solidity
// Rover.sol — _swap（约 10780–10811 行，致命特征）
usdc += ISwapRouter(ROUTER).exactInput(
    ISwapRouter.ExactInputParams(
        abi.encodePacked(address(weth), POOL_FEE, USDC),
        address(this), block.timestamp,
        targetETH,
        0   // ← amountOutMinimum = 0，无滑点保护
    )
) * 1e12;

eth += ISwapRouter(ROUTER).exactInput(
    ISwapRouter.ExactInputParams(
        abi.encodePacked(USDC, POOL_FEE, address(weth)),
        address(this), block.timestamp,
        toSwap,
        0   // ← amountOutMinimum = 0
    )
);
```

#### 2.3 火灾现场 B：`_repackNFT` → `NFPM.mint` — 等价于无保护的 `addLiquidity`

再平衡在 `_swap` 之后，将剩余 WETH/USDC **加回 V3 头寸**，但 mint 参数 **不设 `amount0Min` / `amount1Min`**：

```solidity
// Rover.sol — _repackNFT（约 10583–10596 行，致命特征）
(wethAmount, usdcAmount) = _swap(wethAmount, usdcAmount, price);

(ID, liquidityUnderManagement,,) = NFPM.mint(
    INonfungiblePositionManager.MintParams({
        // ...
        amount0Desired: token1isWETH ? usdcAmount : wethAmount,
        amount1Desired: token1isWETH ? wethAmount : usdcAmount,
        amount0Min: 0,   // ← 无 amountAMin 保护
        amount1Min: 0,   // ← 无 amountBMin 保护
        recipient: address(this),
        deadline: block.timestamp
    })
);

// increaseLiquidity 同样为 0, 0（约 10601–10603 行）
NFPM.increaseLiquidity(..., amount0, amount1, 0, 0, block.timestamp);
```

**结论：** 外层 `fetch`/`deposit` 只负责在**已被操纵的 `slot0`** 上打开开关；**资金实质流失**发生在 **`_swap` 的零最小输出 Router 成交** 与 **`NFPM.mint` 的零最小流动性保护** 的组合中。

### 3. `fetch` / `deposit` 与上层清算的耦合

`deposit` / `withdraw` 首行均 `fetch(msg.sender)` → 读当前 `slot0` 并 `_repackNFT`。攻击者在受害交易前砸盘，受害者的份额计算与 `_swap` 均基于**已扭曲的 slot0**。

### 4. 对 QU!D 栈上层的流动性出口

`Aux._clearSwaps`（约 7849–7908 行，同仓库 `src/Aux.sol`）在批次清算时：

- `gotForETH = V3.withdrawUSDC(...)`  
- `pooled_eth = V3.take(pooled_usd)`  
- `V3.depositUSDC(..., price)`  

均使用 **`Rover.getPrice` 语境下的 V3 池状态** 或 `repackNFT()` 刷新后的价格，使 **Rover 金库中的 WETH/USDC** 在清算环节以错误比例流向 `Aux` / 用户。

---

## 攻击路径演练

**前提：** 攻击者可闪电贷操纵 Rover 绑定的 **`POOL`（WETH/USDC 0.3%）** 储备；`repackNFT()` 为 **public**（`nonReentrant`），可被第三方触发读价。

### 第 1 步：准备

监听 `Rover.deposit` / `Rover.withdraw`，或链上 `Aux.clearSwaps` 即将调用 `V3.take` / `V3.withdrawUSDC` 的交易。

### 第 2 步：污染 AMM 瞬时储备（砸盘）

攻击者向 **`Rover.POOL`（WETH/USDC Uniswap V3）** 大额 `swap`，改变池内瞬时储备，使后续结算采用的兑换比率被污染。

**重要表述修正：** 本合约**源码中并未**显式编写 `price = (reserve1 * 1e18) / reserve0` 一类公式。财务结果来自：

1. **`Rover.getPrice(sqrtPriceX96)`** 读取 **`POOL.slot0()`** 的瞬时价（与储备比例单调对应，属**间接**定价）；  
2. **`ISwapRouter.exactInput`** 在 **`amountOutMinimum = 0`** 时，按 Router 路径上的**瞬时储备兑换比率**成交（等价于被迫接受被污染的比例）。

砸盘时，可抽象为池内瞬时比率（示意，token 顺序以 `token1isWETH` 为准）：

$$
\frac{\text{Reserve}_{\text{quote}}}{\text{Reserve}_{\text{base}}}
\quad\text{（例如 USDC/WETH 侧）}
$$

在攻击方向上：**用于定价/兑换的「分子侧」资产储备锐减、「分母侧」储备激增**，该比率被 **恶意扭曲、严重偏离** 公允市场（可极端偏低或偏高）。**并非**「瞬间安全偏离」——应为 **严重偏离 / 恶意扭曲**。

> **与税费代币报告的区分：** 本地址 **无** `_transfer` → `swapAndLiquify` → `swapExactTokensForTokens`；**无** 收纳 Tax/Liquidity Fee 后再自动换币的流程。受害的是 **LP 再平衡与 Router/NFPM 清算**，见第 3 步。

### 第 3 步：间接清算 + 零滑点路由（单区块原子性）

这是 **间接清算型** 价格操纵：合约通过 **外部 Uniswap V3 Router / NFPM** 完成兑换与加池，在 **原生代码层面未对最小输出做有效防御**（`amountOutMinimum = 0`、`amount0Min/amount1Min = 0`），从而在污染比率下被迫成交。

**受害的具体路径（非泛化「deposit / liquidation」）：**

| 步骤 | 实际函数 | 流失资产 |
|------|----------|----------|
| 触发 | **`fetch`** / **`deposit`** / 公开 **`repackNFT`** | 读被污染的 `slot0` |
| 火灾现场 A | **`_swap` → `ISwapRouter.exactInput(..., 0)`** | 协议 WETH/USDC 在 Router 上 **滑点流失** |
| 火灾现场 B | **`_repackNFT` → `NFPM.mint(..., amount0Min:0, amount1Min:0)`** | 剩余资产 **贱配进头寸** |
| 上层清算 | **`Aux._clearSwaps` → `V3.take` / `withdrawUSDC`** | 本实例金库 **WETH/USDC 被提走** |

**物理过程（砸盘 WETH，与 `swapAndLiquify` 仅经济学同构）：**

1. **抢跑**污染 `POOL` 储备 → Router 隐含比率极差。  
2. **受害 tx** 进入 **`_repackNFT`**（或 `deposit` 内联的 `_swap`）：  
   - 阶段一：Router **`exactInput` 且 `amountOutMin=0`**，被迫按污染比率卖出/买入，**换回的对侧资产极少**；  
   - 阶段二：**`NFPM.mint` 零 `amount*Min`**，大量单边资产以错误比例注入流动性。  
3. **Slippage Revenue**：攻击者在**同一区块**内反向 `swap` 套利。

**原子性：** 污染池子、触发 `Rover` 路径、反向套利，可压在 **同一 `block.number`** 内完成。

### 第 4 步：还原

反向交易恢复 `POOL` 价格，偿还闪电贷，保留 Slippage Revenue 净利润。

---

## 资金外流受灾函数（已验证源码）

| 函数 | 源文件（约行） | 资金后果 |
|------|----------------|----------|
| **`Rover.getPrice`** | `src/Rover.sol` ~10616 | 定价中枢；输入为可操纵的 `sqrtPriceX96` |
| **`Rover.fetch`** | ~10514 | 每次存取款前 `slot0` + `_repackNFT` |
| **`Rover.repackNFT`** | ~10607 | 公开触发；读 `slot0` 后按 `price` 调 `_swap` / NFPM |
| **`Rover._swap`** | ~10717 | 用 `price` 定比；**Router `exactInput(..., 0)`** 无最少输出 |
| **`Rover.deposit`** | ~10817 | `fetch` → `_swap` → 按失真 `sqrtPrice` **铸 `newShares`** |
| **`Rover.withdraw`** | ~10921 | `expected = ethAmount + mulDiv(usdAmount, price, WAD)`；不足时 **`AMP.get(expected - ethAmount)`** 补 ETH 发给用户 |
| **`Rover.take`** | ~10859 | `onlyUs`：`repackNFT` → 减流动性 → **USDC→WETH swap(0 min)** → **`weth.transfer(msg.sender)`**（`Aux` 提款） |
| **`Rover.withdrawUSDC`** | ~10895 | `onlyUs`：减流动性 → 可能 **WETH→USDC swap(0 min)** → **`USDC.transfer(msg.sender)`** |
| **`Rover.depositUSDC`** | ~10883 | `onlyUs`：按传入 **`price`** 做 `_swap` 增流动性 |
| **`Amp.leverETH` / 平仓** | `src/Amp.sol` ~7141+ | `V3.repackNFT()` + `V3.getPrice()` 决定借贷与平仓规模 |

**典型损失机制：**

1. **`_repackNFT` 两阶段泄漏（核心）**：砸盘 → `_swap` 贱卖（`exactInput` min=0）→ `NFPM.mint` 贱配（`amount*Min=0`）→ 同块反向套利抽取 **Slippage Revenue**。  
2. **`deposit`（攻击者 LP）**：在阶段 1 之后存入，以失真 `price` 铸出过多 `liq`，稀释诚实 LP。  
3. **`withdraw` + `AMP.get`**：拉盘后 `expected` 膨胀，从 `Amp` 侧多补 ETH。  
4. **`Aux._clearSwaps` → `V3.take` / `withdrawUSDC`**：清算环节从 Rover 金库按操纵价提出 WETH/USDC。

---

## Impact

| 维度 | 说明 |
|------|------|
| **Financial** | `Rover` LP（`positions`）、通过 `Aux` 清算抽走的 **WETH/USDC**，以及 `Amp` 杠杆头寸均可损失；Router **零最小输出** 放大可提取价值。 |
| **Integrity** | `totalShares` 与 `liquidityUnderManagement` 比例、`ETH_FEES`/`USD_FEES` 分配可被短时扭曲。 |
| **Availability** | 不直接停机；可重复利用直至添加预言机/TWAP 或滑点下限。 |

**Severity：High** — 无需特权；定价与成交逻辑均依赖可操纵 `slot0`；与主栈资金（`Aux` 批次、`wethVault`）硬耦合。

---

## Proof of concept

**状态：** 概念验证思路；完整 Foundry fork 待补充。

1. Fork 主网，将 **`target`** 设为下表任一 Rover 地址，并绑定该实例的 `POOL` / `ROUTER` / `NFPM`。  
2. 闪电贷 WETH，在 `POOL` 上 `swap` 使 `slot0` 偏离 Chainlink &gt; 1%。  
3. 同一区块：  
   - 调用 `Rover.deposit{value: X}()`，记录 `positions[attacker].liq`；或  
   - 先操纵再 `Aux.clearSwaps()`，观测 `V3.withdrawUSDC` / `take` 转出量。  
4. 与未操纵时 `getPrice` 下公平份额/转出量对比。

```solidity
// 骨架
function test_rover_slot0_manipulation() public {
    uint p0 = Rover(target).getPrice(pool.slot0());
    flashLoanDumpWETHInPool(pool);
    uint p1 = Rover(target).getPrice(pool.slot0());
    assertLt(p1, p0 * 99 / 100); // 砸盘价下跌
    Rover(target).deposit{value: 1 ether}();
    // 断言 attacker liq 相对公平价格偏高
}
```

---

## Recommendations

1. **定价改用 TWAP**：`OracleLibrary.consult(POOL, secondsAgo)` 或 Chainlink ETH/USD，禁止单独 `slot0` 决定份额。  
2. **所有 `exactInput` / NFPM mint 设置非零 `amountOutMinimum` / `amount*Min`**，基于 TWAP 推导。  
3. **`fetch` / `repackNFT` 解耦**：存取款不应在单 tx 内无条件 `repack`；或 repack 前校验 spot 与 TWAP 偏差。  
4. **`withdraw` 中 `AMP.get` 补差**：增加价格健全性检查，防止操纵 `expected`。  
5. **`Aux` 调用 `V3.*` 前**：强制 TWAP 边界（与 `Vogue.md` 建议一致）。

---

## References

- 源码（主实例）：`Etherscan verified source`  
- 同构实例：`eth_0x491c3a404ff9564a4a09cf777c833c4d138801e6.txt`、`eth_0x1d8227d7cdabce4ac80182857bc0de28645fae53.txt`、`eth_0xb9ae20b2ed85e508c29881f24aa1308dd3751aff.txt`  
- 核心：`src/Rover.sol`、`src/Aux.sol`（`V3.*` 调用）、`src/Amp.sol`  
- 同栈 V4 问题：`../en/eth_0x0bafc174bf07ec3d0eafdee06df7d07377108c0c_Vogue.md`  
- PM 审计：`internal PM audit metadata`（上述 Rover：高，70%，`pm_vuln_count` 8）

---

## Platform checklist

- [x] 标题与 Summary 一致  
- [x] 地址与链正确  
- [ ] PoC 可独立复现（待补）  
- [x] Impact 与 High 定级匹配  
- [x] 已列出具体受害函数与行号范围  
