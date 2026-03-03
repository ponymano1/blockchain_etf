# ETF 协议详细设计文档（Core + 通用 Oracle + UniV3 执行器）

> 版本：v2（采用通用 Oracle，可插拔）
>
> 目标：在保持 `ETFCore` 简洁与可审计的前提下，实现 5~10 个代币的指数化持仓、申购赎回与自动调仓。

---

## 1. 设计目标与原则

## 1.1 目标

- 用户以 USDT 申购 ETF 份额，按净值铸造份额。
- 用户赎回 ETF 份额，按净值兑回 USDT。
- Keeper 定期或触发式调仓，维持等权（或配置权重）。
- 支持 5~10 个 token 的篮子组合。

## 1.2 核心原则

1. **`ETFCore` 是唯一记账中心**：NAV、份额、资产托管都在 Core。
2. **交易执行外置**：通过 `UniV3TradeExecutor` 与 DEX 交互。
3. **价格读取抽象**：Core 只依赖 `IPriceOracle`，可切换 Chainlink/Pyth/UniTWAP。
4. **双层风控**：Oracle 估值 + DEX 偏差校验 + 滑点限制 + 冷却限制。

---

## 2. 系统架构

```text
┌─────────────────────────────────────────────────────────────────┐
│                            参与者                                │
│ User(申购/赎回)   Keeper(调仓)   Governor/Timelock(治理)         │
└───────────────┬──────────────────┬──────────────────┬───────────┘
                │                  │                  │
                v                  v                  v
┌─────────────────────────────────────────────────────────────────┐
│                           ETFCore                               │
│  - 资产托管(USDT + basket tokens)                               │
│  - shares mint/burn                                             │
│  - NAV计算、调仓规划、权限/风控                                  │
│  - 仅依赖 IPriceOracle + ITradeExecutor                         │
└───────────────┬──────────────────────────────┬──────────────────┘
                │                              │
                v                              v
┌────────────────────────────┐      ┌────────────────────────────┐
│ IPriceOracle               │      │ UniV3TradeExecutor         │
│  ├─ PrimaryOracleAdapter   │      │  - onlyCore                │
│  └─ FallbackOracleAdapter  │      │  - swapExactInputSingle    │
└───────────────┬────────────┘      └──────────────┬─────────────┘
                │                                   │
                v                                   v
      Chainlink / Pyth / TWAP                 UniswapV3Router/Pool
```

---

## 3. 合约与模块职责

## 3.1 ETFCore（核心主合约）

- 持有所有资产。
- 计算净值 \( NAV \) 与份额价格。
- 执行 `deposit/redeem/rebalance` 流程。
- 读取 Primary/Fallback Oracle，并做新鲜度与偏差检查。
- 通过 `ITradeExecutor` 调用 DEX 交易。

## 3.2 IPriceOracle（通用接口）

统一价格读取接口，Core 不关心具体来源。

建议接口：

- `getPriceE18(base, quote) -> (priceE18, updatedAt, ok)`
- `getBatchPriceE18(bases[], quote) -> (pricesE18[], updatedAts[], oks[])`

可实现：

- `ChainlinkOracleAdapter`
- `PythOracleAdapter`
- `UniV3TwapOracleAdapter`（可作为 fallback）

## 3.3 UniV3TradeExecutor

- 仅允许 Core 调用（`onlyCore`）。
- 做 token/router/feeTier 白名单校验。
- 执行 Uniswap V3 单跳（必要时扩展多跳）。
- 强制 `amountOutMinimum`、`deadline`。

---

## 4. 数据结构设计

## 4.1 TokenConfig

```text
token: address
enabled: bool
targetWeightBps: uint16     // 等权可统一设置，或按配置
driftBps: uint16            // 单token偏离阈值
maxSlippageBps: uint16      // 交易滑点上限
maxTradeSizeUsdt: uint256   // 单次交易最大规模
feeTier: uint24             // UniV3 fee
pool: address               // 对应token/USDT池（用于执行器或校验）
```

## 4.2 OracleConfig

```text
primaryOracle: address      // IPriceOracle
fallbackOracle: address     // IPriceOracle (可选)
maxStaleness: uint32        // 最大价格陈旧秒数
maxDeviationBps: uint16     // 主/备价最大允许偏差
```

## 4.3 VaultConfig

```text
usdt: address
checkInterval: uint32
cooldown: uint32
globalDriftBps: uint16
priceMoveTriggerBps: uint16
minTradeUsdt: uint256
maxTurnoverBps: uint16
rebalanceDeadlineSec: uint32
```

---

## 5. 关键数学定义

- 统一精度：价格与 NAV 用 `1e18`（USDT内部可按 6 位做换算）。
- 组合净值：
  \[
  NAV = USDT\_{bal} + \sum_i \left( balance_i \times price_i \right)
  \]
- 份额价格：
  \[
  sharePrice = \frac{NAV}{totalSupply}
  \]
- 申购份额（非首笔）：
  \[
  sharesOut = usdtIn \times \frac{totalSupply}{NAV\_{before}}
  \]

---

## 6. Oracle 策略（主备 + 熔断）

## 6.1 读取顺序

1. 先读 Primary。
2. Primary 不可用（失败或 stale）则读 Fallback。
3. 若两者都可用，则检查偏差：
   \[
   deviationBps = \frac{|P*{primary}-P*{fallback}|}{P\_{primary}} \times 10000
   \]
4. 偏差超 `maxDeviationBps` -> 熔断（禁止调仓；可按策略限制申赎）。

## 6.2 使用建议

- NAV 与份额计算：优先 Primary。
- 交易前 minOut 计算：取 `min(primary, fallback)` 的保守价。

---

## 7. 权限模型

- `DEFAULT_ADMIN_ROLE`: 角色与关键地址管理（建议多签+timelock）
- `GOVERNOR_ROLE`: 配置参数、增删 token、切换 Oracle
- `KEEPER_ROLE`: `executeRebalance`
- `PAUSER_ROLE`: `pause/unpause`

约束：

- `TradeExecutor.swap...` 必须 `onlyCore`
- 所有配置修改应有范围检查（bps、interval、地址非零）

---

## 8. 流程与调用图

## 8.1 静态调用关系图

```text
User      -> ETFCore.deposit/redeem/preview
Keeper    -> ETFCore.checkRebalance/executeRebalance
Governor  -> ETFCore.updateConfig/setOracle/addToken/pause

ETFCore   -> IPriceOracle(primary/fallback)
ETFCore   -> UniV3TradeExecutor.swapExactInputSingle
Executor  -> UniswapV3Router.exactInputSingle
```

## 8.2 申购（Deposit）

```text
User                     ETFCore                     IPriceOracle(P/F)
 | deposit(usdtIn,minShares) |
 |--------------------------->|
 | transferFrom(USDT)         |
 | navBefore = _calcNav()     |
 |--------------------------->| getBatchPriceE18
 |<---------------------------| prices
 | sharesOut = formula        |
 | require(sharesOut>=min)    |
 | mint(user, sharesOut)      |
 | emit Deposit               |
 |<---------------------------|
```

## 8.3 赎回（Redeem）

```text
User                     ETFCore                     Oracle              Executor
 | redeem(shares,minUsdt)  |
 |------------------------->|
 | nav = _calcNav()         |
 |------------------------->| getBatchPriceE18
 |<-------------------------| prices
 | burn(shares)             |
 | usdtOut = formula        |
 | if USDT不足:             |
 |   build sell list        |
 |--------------------------------------------------------------->| swap token->USDT
 |<---------------------------------------------------------------| amountOut
 | require(usdtOut>=min)    |
 | transfer USDT            |
 | emit Redeem              |
 |<-------------------------|
```

## 8.4 调仓检查（Check）

```text
Keeper                  ETFCore                  Oracle(P/F)
 | checkRebalance()       |
 |----------------------->|
 | interval/cooldown 检查 |
 |----------------------->| getBatchPriceE18
 |<-----------------------| prices
 | 计算权重偏离/价格波动   |
 | return need + preview  |
 |<-----------------------|
```

## 8.5 执行调仓（Execute）

```text
Keeper                  ETFCore                  Oracle(P/F)              Executor
 | executeRebalance()     |
 |----------------------->|
 | require keeper/notPaused/cooldown
 |----------------------->| getBatchPriceE18
 |<-----------------------| prices
 | 计算 NAV、目标持仓
 | trade plan: 先卖后买
 | loop sell/buy:
 |--------------------------------------------------------------->| swap...
 |<---------------------------------------------------------------| amountOut
 | 更新lastRebalanceTime/price
 | emit Rebalanced
 |<-----------------------|
```

## 8.6 Oracle 主备路径图

```text
ETFCore
  |
  |-- read Primary -> ok && fresh ? ----yes----> use Primary
  |                      |
  |                      no
  |                      v
  |-- read Fallback -> ok && fresh ? ---yes----> use Fallback
  |                      |
  |                      no
  |                      v
  |----------------------> revert / circuit break
```

---

## 9. 核心函数清单（建议）

## 9.1 用户接口

- `deposit(uint256 usdtIn, uint256 minSharesOut)`
- `redeem(uint256 sharesIn, uint256 minUsdtOut)`
- `previewDeposit(uint256 usdtIn)`
- `previewRedeem(uint256 sharesIn)`

## 9.2 Keeper 接口

- `checkRebalance() returns (bool need, RebalancePreview memory p)`
- `executeRebalance(RebalanceParams calldata p)`

## 9.3 治理接口

- `addToken(TokenConfig calldata cfg)`
- `updateToken(address token, TokenConfig calldata cfg)`
- `removeToken(address token)`
- `setOracle(address primary, address fallback, OracleConfig calldata cfg)`
- `setVaultConfig(VaultConfig calldata cfg)`
- `pause()/unpause()`

---

## 10. 风控规则（必须实现）

1. **最小输出保护**：每笔 swap 需 `amountOutMinimum`。
2. **陈旧价格拒绝**：`block.timestamp - updatedAt <= maxStaleness`。
3. **主备偏差熔断**：超阈值则禁止调仓。
4. **单轮换手上限**：`turnover <= NAV * maxTurnoverBps / 10000`。
5. **冷却时间**：`block.timestamp - lastRebalanceTime >= cooldown`。
6. **最小交易额**：过滤 dust trades。
7. **暂停开关**：极端行情可暂停调仓（赎回策略由治理决定）。

---

## 11. 事件设计

- `Deposit(address user, uint256 usdtIn, uint256 sharesOut)`
- `Redeem(address user, uint256 sharesIn, uint256 usdtOut)`
- `RebalanceChecked(bool need, uint256 nav, uint256 driftBps)`
- `RebalanceTrade(address tokenIn, address tokenOut, uint256 amountIn, uint256 amountOut)`
- `Rebalanced(uint256 navBefore, uint256 navAfter, uint256 turnoverUsdt)`
- `OracleSwitched(address primary, address fallback)`
- `CircuitBreakerTriggered(bytes32 reason)`
- `ConfigUpdated(bytes32 key)`
- `Paused(address by)`
- `Unpaused(address by)`

---

## 12. 默认参数建议（MVP）

- 资产数：6（范围 5~10）
- `checkInterval = 1h`
- `cooldown = 4h`
- `maxStaleness = 15m`
- `priceMoveTriggerBps = 300`
- `driftBps = 200`
- `globalDriftBps = 700`
- `maxDeviationBps = 150`
- `maxSlippageBps = 80`
- `minTradeUsdt = 100 USDT`
- `maxTurnoverBps = 2500`

---

## 13. 审计重点清单

1. share mint/burn 是否严格由 Core NAV 决定。
2. Oracle fallback 与熔断状态机是否可绕过。
3. Executor `onlyCore` 与白名单是否完备。
4. 所有 swap 是否均有 `minOut + deadline`。
5. rebalance 循环中的精度、重入、资产卡死风险。
6. pause 状态下 deposit/redeem 的策略一致性。
7. 治理参数是否有边界限制与 timelock。

---

## 14. 迭代路线

1. 完成 Core + Oracle 接口 + 只读 NAV。
2. 接入 Executor，打通单笔交易。
3. 实现 checkRebalance（预览与原因码）。
4. 实现 executeRebalance（先串行先卖后买）。
5. 加入熔断、pause、timelock。
6. 测试网实盘参数校准后主网上线。

---

## 15. 结论

该设计相比“仅 UniV3 询价”更通用、更稳健：

- Core 与 Oracle/DEX 解耦，便于升级；
- 可使用行业通用喂价（Primary）；
- 结合 DEX 价格做偏差校验，降低异常报价风险；
- 保持 ETF 份额公平性与审计可读性。

```


```
