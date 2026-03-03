# blockchain_etf

本项目实现了一个基于区块链的 ETF 协议，支持多资产篮子的申购、赎回与自动调仓，并具备通用 Oracle/执行器架构。

项目设计详见 [`doc/PRD_Design.md`](doc/PRD_Design.md)。该设计文档包括：

- **目标与系统架构**：ETF 申购/赎回流程和调仓机制
- **关键模块职责**：ETFCore（核心主合约）、IPriceOracle（价格源接口）、UniV3TradeExecutor（执行器）
- **主要数据结构与数学模型**：NAV 计算、用户份额、公允估值方法
- **Oracle 策略**：支持主备喂价源及熔断机制，兼容 Chainlink/Pyth/UniV3 等
- **权限与风控**：多层权限控制、swap 最小输出、熔断与暂停
- **接口清单**：用户、Keeper（调仓）、治理接口示例
- **事件与参数建议**
- **审计重点与迭代路线**

> 推荐直接查阅 [`doc/PRD_Design.md`](doc/PRD_Design.md) 获取完整设计细节与规范。

如需贡献或了解实现细节，请仔细阅读对应文档。
