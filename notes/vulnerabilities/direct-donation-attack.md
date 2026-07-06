# Direct Donation Attack / Balance Manipulation

## 一句话理解

Direct Donation Attack 是指攻击者直接把 ERC20 token 转到合约地址，污染：

```solidity
token.balanceOf(address(this))
```

如果协议把这个实时余额当作可信的内部账本，就可能导致 share、价格、奖励、抵押率或池子比例被错误计算。

中文可以叫：

- 直接捐赠攻击
- Donation Attack
- 余额注入攻击

核心问题不是“有人给合约送钱”，而是协议错误假设：

```text
合约地址里的 token 余额 == 协议内部真实记账资产
```

但 ERC20 的转账由 token 合约控制，接收方通常无法拒绝一次普通的 ERC20 `transfer`。

## 漏洞分类

更通用的分类：

- Accounting Manipulation
- Erroneous Accounting
- Unsafe Balance-Based Accounting

根据影响场景，也可能被描述成：

- Direct Donation Attack
- Balance Manipulation
- Vault Inflation Attack
- Exchange Rate Manipulation
- Balance-based Price Manipulation

所以比较稳的写法是：

```text
Direct Donation Attack / Balance Manipulation
Category: Erroneous Accounting / Unsafe Balance-Based Accounting
```

## 典型问题

危险模式是协议直接用当前余额参与核心计算：

```solidity
function totalAssets() public view returns (uint256) {
    return asset.balanceOf(address(this));
}

function deposit(uint256 amount) external {
    uint256 shares = amount * totalSupply() / totalAssets();
    asset.safeTransferFrom(msg.sender, address(this), amount);
    _mint(msg.sender, shares);
}
```

如果攻击者能先直接转入 token：

```solidity
asset.transfer(address(vault), donation);
```

那么 `totalAssets()` 会被抬高，后续 share 计算、exchange rate 或价格就可能被干扰。

重点：`balanceOf(address(this))` 是外部可影响的状态，不一定等于协议自己通过受控流程记录的资产数量。

## 常见场景

### Vault / shares

Vault 里常见公式：

```solidity
shares = amount * totalSupply / totalAssets;
```

如果 `totalAssets` 直接来自：

```solidity
asset.balanceOf(address(this))
```

攻击者可以通过直接转 token 抬高 `totalAssets`，改变 share price。新用户可能收到更少 shares，或者攻击者结合 redeem / arbitrage 提取价值。

在 ERC4626 或类 ERC4626 vault 里，这类问题常被叫：

- Donation Attack
- Vault Inflation Attack

### AMM / pool

如果协议直接用 raw balance 算价格：

```solidity
price = token0.balanceOf(address(pool)) / token1.balanceOf(address(pool));
```

攻击者可以直接转入其中一种 token，扭曲池子比例。

成熟 AMM 一般不会把 raw balance 当作唯一价格来源，而是维护内部 reserve：

```solidity
reserve0
reserve1
```

并通过受控函数更新 reserve。

### Reward / staking

奖励或质押合约如果这样算：

```solidity
rewardPerToken = rewardToken.balanceOf(address(this)) / totalStaked;
```

攻击者直接转入 reward token 后，可能改变奖励分配逻辑。

这仍然属于 donation attack / accounting manipulation，只是影响对象从 share price 变成 reward accounting。

## 攻击路径

1. 协议在核心逻辑中使用 `token.balanceOf(address(this))`。
2. 攻击者直接 `transfer` token 到协议合约地址。
3. 合约的 raw token balance 被抬高，但内部业务状态没有对应更新。
4. 后续 deposit、withdraw、redeem、swap、reward、borrow 或 liquidation 使用被污染的余额。
5. 攻击者让用户拿到错误份额，扭曲价格，影响奖励，或通过套利提取价值。

## 影响

可能影响：

- share mint / burn
- exchange rate
- LP price
- AMM pool ratio
- reward distribution
- collateral value
- borrow / liquidation calculation
- accounting invariant

严重程度取决于 raw balance 是否进入了资金相关的核心计算。

如果它只用于展示 UI 或非关键统计，可能只是低风险；如果它决定 mint shares、redeem assets、价格、抵押率或奖励分配，就可能是高风险。

## Mitigation

- 不要盲目信任 `token.balanceOf(address(this))` 作为内部账本
- 维护内部记账变量，比如 `totalManagedAssets`、`accountedBalance`、`reserve0`、`reserve1`
- 只通过受控协议流程更新内部记账
- 对 vault 使用成熟 ERC4626 实现，并考虑 virtual shares / virtual assets / minimum shares
- 对 AMM 使用内部 reserve，而不是直接用 raw balance 算价格
- 对 reward / staking 逻辑，用明确的 reward accounting，而不是把当前余额直接当作待分配奖励
- 如果需要处理多余 token，提供受控的 sweep、skim 或 sync 机制，并明确谁能调用、如何影响 accounting

内部记账例子：

```solidity
uint256 public totalManagedAssets;

function deposit(uint256 amount) external {
    asset.safeTransferFrom(msg.sender, address(this), amount);
    totalManagedAssets += amount;
}

function totalAssets() public view returns (uint256) {
    return totalManagedAssets;
}
```

实际项目里还要注意 fee-on-transfer token。更稳的方式可能是用 balance delta 记录实际收到的数量：

```solidity
uint256 beforeBalance = asset.balanceOf(address(this));
asset.safeTransferFrom(msg.sender, address(this), amount);
uint256 received = asset.balanceOf(address(this)) - beforeBalance;

totalManagedAssets += received;
```

但要区分两件事：

- balance delta 可以确认这次受控操作实际收到多少 token
- 直接把整个 `balanceOf(address(this))` 当总账本仍然可能被 donation 污染

## 审计时注意

看到下面代码要多看一眼：

- `token.balanceOf(address(this))`
- `asset.balanceOf(address(this))`
- `balanceOf(pool)`
- `balanceOf(vault)`
- `totalAssets()` 直接返回 raw token balance
- share / LP / reward / price / collateral 计算依赖 raw balance
- `address(this).balance == internalAccounting` 的 ERC20 版本
- 没有内部 reserve、accounted balance 或 total managed assets
- 有 `sync()`、`skim()`、`sweep()` 但权限或会计影响不清楚

审计问题可以这样问：

```text
如果攻击者现在直接转 1 wei / 1 token / 巨量 token 到合约，
下一次 deposit、redeem、swap、claim、liquidate 会发生什么？
```

## 和 Mishandling of ETH 的关系

它和 ETH 强制转账问题很像：

- ETH 场景：`address(this).balance` 可能被强制转入污染
- ERC20 场景：`token.balanceOf(address(this))` 可能被直接 transfer 污染

共同根因都是：

```text
外部真实余额不一定等于协议内部记账
```

区别是 ERC20 donation 不需要 `selfdestruct` 这类特殊路径。只要 token 支持普通 `transfer`，任何人通常都能把 token 发到合约地址。

## 现在还存在吗

存在。

原因是 ERC20 的基础行为没有变：合约通常不能阻止别人直接把 token 转到自己的地址。

成熟协议一般会通过内部 accounting、reserve、adapter、safe ERC4626 模式来规避这个坑。但很多手写 vault、staking pool、reward distributor、AMM-like 合约、简单 DeFi 合约仍然可能直接用 raw `balanceOf(address(this))` 做关键计算。

所以这个风险不会因为时间消失，只能靠协议设计避免错误依赖。

## 我的理解

Direct Donation Attack 的本质不是攻击者“捐了资产”，而是攻击者让协议看到一个不属于内部业务流程的余额变化。

如果协议把 `balanceOf(address(this))` 当作可信账本，攻击者就能通过直接转账改变这个账本的外观。审计时我应该把 raw token balance 当成外部可控输入，而不是天然可信的协议状态。
