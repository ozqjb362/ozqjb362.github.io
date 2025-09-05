---
layout:     post
title:      DeFi 经典案例：手把手教你本地复现 Compound 去中心化借贷银行
date:       2025-09-05
header-img: img/post-bg-desk.jpg
catalog: true
---

> 关键词：Compound 部署教程、去中心化借贷、DeFi 复现、BSC 测试网、智能合约部署、cToken 初始化、清算机制、收益率模型

只要你有一台 Mac + MetaMask + Remix，十分钟就能把整条 **去中心化借贷协议**跑起来。下文不会堆砌艰深名词，而是按搭建积木的思路，把“银行”从 0 到 1 拼完整，并在常见卡点处穿插 FAQ，减少踩坑时间。

---

## 先做三件事

1. **把工具开好**  
   macOS（本文用 10.15.6）+ Chrome 装最新版 MetaMask → 网络切 BSC Testnet → 领测试 BNB。
2. **源码抓下来**  
   GitHub 搜索 `compound-protocol`，clone 代码后只保留 `contracts` 目录，其余统统干掉，减少 Remix 编译压力。
3. **打开编译优化**  
   Remix → Solidity 编译器 → Enable optimization（推荐 10 万次）。

👉 一键获取复合收益率模拟器，提前算清算风险：[](https://okxdog.com/)

---

## 一、先把“股权”建好：COMP 代币

Compound 用 COMP 做激励，所以它必须是第一条部署记录：  
- 可选 1：直接 fork Compound 自带的 `Comp.sol`，改 `name`/`symbol`/`totalSupply`。  
- 可选 2：直接用一个通用的 ERC20 占位。  

最后拿到地址，记作 `COMP`，后面 `Comptroller` 会引用。

---

## 二、把“会计部”装上去：Comptroller + Unitroller

该模块决定了「谁的抵押值多少钱、能借多少」。  
- **Unitroller.sol** 是透明的代理壳，要最先部署；示例地址：  
  ```
  0x268e3eF4380DA46e55B77a6263810910722a875E
  ```
- **ComptrollerG7.sol** 是真正逻辑，示例地址：  
  ```
  0x67006E2110119Abfd40b2A743A85b4d3bF8967b9
  ```

> 绑定流程（缺一不可）：  
> 1. Unitroller 调用 `_setPendingImplementation(Comptroller 地址)`  
> 2. Comptroller 调用 `_become(Unitroller 地址)`

完成后外部永远把 **Unitroller 地址** 当作 Comptroller 用。  

---

## 三、为资产定价：PriceOracle

DeFi 的命根是喂价。  
- 测试环境用 `SimplePriceOracle.sol`，一次部署，随时手动 `setUnderlyingPrice`。  
- 部署示例：  
  ```
  0x5991199a9aB1801A229a2E49a042471eDE997a21
  ```

---

## 四、决定“存款利率”：Jump 模型

Compound 采用分档利率（JumpRateModelV2）。  
部署一次可复用，关键 5 个参数：

| 参数 | 样例值（18 位精度） |
|---|---|
| baseRatePerYear | 0 |
| multiplierPerYear | 7% → 70000000000000000 |
| jumpMultiplierPerYear | 3 倍 → 3e18 |
| kink | 75% → 0.75e18 |
| owner | yourAddress |

生成 2 份实例：  
一份给 **cToken ERC20**，一份给 **cToken ETH**。  
👉 想深究各项利率含义？点这里[](https://okxdog.com/)

---

## 五、铸造“存款凭证”：cToken 体系

### 针对 ERC20 资产的 cToken

1. 随便发一个测试代币（示例：USDT 改造），地址记作 `underlying`  
2. 部署逻辑实现 `CErc20Delegate.sol`：示例  
   ```
   0xc176eD65274b2a2d422126d597Be715fc97d2e98
   ```
3. 部署可代理的存款凭证 `CErc20Delegator.sol`：  
   调用参数清单  
   - underlying：测试代币地址  
   - comptroller：Unitroller 地址  
   - interestRateModel：刚部署的 Jump 模型地址  
   - initialExchangeRateMantissa：1 * 10^18（1:1）  
   - name / symbol / decimals：按习惯起名即可  
   示例结果：  
   ```
   0x209C9b6a0Ec37b91d0758514070A8439B14B9B3c
   ```

### 针对主币 ETH 的 cToken

直接部署 `CEther.sol`，参数同上（去掉 underlying）。  
示例：  
```
0xf3feeab27E8B8b71ED92040be19E5aA80baf9B01
```

---

## 六、开闸放水：把市场真正连起来

| 动作 | 对应函数 | 关键值 | 备注 |
|---|---|---|---|
| 先在 Oracle 定价 | setUnderlyingPrice | cUSD = $1, cETH = $2000 | 随时可改 |
| 设置保证金系数 _setReserveFactor | CErc20: 10% / CEther: 20% | 决定了项目方抽成 |
| 设置抵押限额 _setCollateralFactor | 0.6 * 1e18 → 60% | 低于此值触发清算 |
| 注册市场 _supportMarket | 依次填入 cUSD、cETH | 只需一次即可 |

全部成功后，你的 Comptroller 现在能识别两种可借/存资产。

---

## 七、开启“流动性挖矿”：COMP 激励

官方通过 `_setCompSpeed(cToken, compSpeed)` 给市场发奖励：  
- `compSpeed` 含义：每区块分配给**供应商或借款人**的 COMP（单位 weis）。*注意*：总数需再×2（供应+借款）。  
- 体验阶段可写死 1 wei，看前端是否正确显示收益。

---

## 八、五段测试脚本，把流程跑一遍

### 1. 入金 mint

- 用户 Alice `approve` 测试代币 → 调用 `cUSD.mint(200e18)`  
- 获得 200 cUSD；Comptroller 自动算出可借资金 120 美元（200 * 0.6）

### 2. 借款 borrow

- Alice 借出 50 USDT → `totalBorrows += 50`，池子现金剩 150  
- 利用率产生，利率立刻大于 0

### 3. 还款 repay

- Alice 再转入 50 USDT 赎回债务，协议立即把 `totalBorrows` 清零  
- 注意利率指数已更新，实际欠款会稍大于 50

### 4. 取回 redeem

- Alice 可 `redeem(部分 cUSD)` 或 `redeemUnderlying(指定底层资产数量)`  
- Comptroller 先算她是否还有足够抵押，若不足直接 revert

### 5. 清算 liquidation（负流动性场景）

- Alice 打包 Uni + USDT 两个市场，存 Uni 借 USDT  
- 当 Uni 价格下跌 → 账户流动性成负数  
- 清算者 Bob 调用 `liquidateBorrow`，先还欠款 USDT，再收走 Uni cToken 并拿到 bonus（清算激励 8%）

---

## FAQ 高频疑问

**Q1：为什么先部署 Unitroller，再绑定 Comptroller？**  
A：代理-实现分离让后期升级逻辑不再需要用户迁移资产。设置流程的两步验证也避免了黑客一次性把代理指向恶意逻辑。

**Q2：Jump Model 的 kink 设 75% 会不会太高？**  
A：75% 是主网目前常用值；测试网可调至 45% 观察利率曲线突变点。

**Q3：Oracle 只有三个函数，Mainnet 能这么用吗？**  
A：绝不能！主网需 Chainlink、Pyth 或其他具有主权保护的喂价。

**Q4：CompSpeed 填 0 是否就放弃了治理激励？**  
A：对，用户不会收到 COMP 但协议仍可运作，适合只做技术验证的场景。

**Q5：我把 `CErc20` 和 `CEther` ReserveFactor 颠倒填写会怎样？**  
A：只是平台抽成不同，不影响用户安全，可在 `updateJumpRateModel` 时修正。

**Q6：清算人监测流动性负值的逻辑在哪里？**  
A：写在 Comptroller 的 `getAccountLiquidity()`，返回值如果小于 0 即可触发。区块浏览器直接读就行。

---

## 写在最后

把上面八个环节全部走完，你就拥有了 **本地可迭代的 Compound Fork**。下一步把前端用 Ether.js 接一下，或者写机器人做利率/清算套利，都是天然延伸。  
👉 [立刻进阶：获取 Compound 收益全景表 + 清算闪电盈利脚本](https://okxdog.com/)