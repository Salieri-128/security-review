# Weird ERC20

## 一句话理解

Weird ERC20 是指表面兼容 ERC20 接口，但行为和协议默认想象不一样的 token。

漏洞根因是：协议把外部 token 当成标准、可信、静态的资产。

但真实世界里的 ERC20 可能：

- 转账扣手续费
- 不返回 bool
- transfer 时触发回调
- balance 不经过 transfer 也会变化
- 被 admin 升级、暂停或拉黑
- approve 行为不符合常见习惯

所以 ERC20 不只是“资产”，它也是一个外部合约。

## 为什么重要

很多 DeFi 协议默认：

```solidity
token.transferFrom(user, address(this), amount);
```

意味着：

- 用户少了 `amount`
- 协议收到了 `amount`
- 调用返回 `true`
- 不会重入
- token balance 不会自己变
- token 逻辑以后也不会变

这些假设在真实 token 里不一定成立。

TSwap、AMM、Vault、Lending、Staking 都依赖 token balance 和 accounting。一旦 token 行为非标准，就可能破坏：

- reserve
- share
- LP mint
- swap output
- withdraw
- repay / liquidation

## 常见类型

### Fee-on-transfer

这种 token 转账时会扣手续费。

比如协议调用：

```solidity
token.transferFrom(user, pool, 100);
```

协议以为池子收到了 `100`，但实际可能只收到 `98`，剩下的 `2` 被 token 扣走。

这对 AMM 特别危险，因为 swap、mint LP、更新 reserve 经常依赖：

```text
amountIn == actualReceived
```

如果协议直接用用户传入的 `amount` 记账，攻击者就可能用更少的实际输入换出过多资产。

更稳的做法是使用 balance delta：

```solidity
uint256 beforeBal = token.balanceOf(address(this));
token.safeTransferFrom(msg.sender, address(this), amount);
uint256 actualReceived = token.balanceOf(address(this)) - beforeBal;

balances[msg.sender] += actualReceived;
```

重点：不要相信参数 `amount`，要相信合约实际收到的数量。

### Missing return values / returns false

标准 ERC20 的 `transfer` / `transferFrom` 应该返回 `bool`，但很多真实 token 不返回值，比如 USDT。

危险写法：

```solidity
require(token.transfer(to, amount), "transfer failed");
```

如果 token 不返回 bool，这类代码可能直接 revert。

通常应该使用 OpenZeppelin `SafeERC20`：

```solidity
token.safeTransfer(to, amount);
token.safeTransferFrom(from, to, amount);
```

但 `SafeERC20` 不是万能的。它能处理很多返回值兼容问题，但不能自动解决 fee-on-transfer、rebasing、blocklist、upgradeable 等所有风险。

### Reentrant token

不要默认 ERC20 transfer 只是简单改 balance。

某些 token 在 transfer 过程中可能触发 hook 或回调，例如 ERC777 风格 token。也就是说：

```solidity
token.transferFrom(user, address(this), amount);
```

本质上是一次外部 call，中间可能重新进入当前协议。

危险模式：

```solidity
function deposit(uint256 amount) external {
    token.transferFrom(msg.sender, address(this), amount);
    balances[msg.sender] += amount;
}
```

审计时要把 token interaction 当作外部调用检查，结合 CEI 和 `nonReentrant`。

### Rebasing token

普通 token 的 balance 通常只会因为 transfer、mint、burn 改变。

但 rebasing token 可以在没有 transfer 的情况下改变余额。协议如果缓存了 token balance、reserve 或 totalAssets，就可能用到过期数据。

重点关注：

- AMM reserve
- Vault totalAssets
- Lending pool liquidity
- Staking reward accounting

看到 `storedBalance`、`reserve0`、`reserve1`、`totalAssets` 这类缓存值时，要问：如果底层 token balance 自己变了，协议还能正常吗？

### Upgradeable token

USDC、USDT 这类 token 是可升级的。

这意味着 token 今天没有手续费，不代表以后没有；今天不会重入，不代表以后不会；今天 transfer 正常，不代表未来逻辑不变。

这种风险不一定是协议当前代码里有 bug，而是协议依赖了外部 token 的当前行为。

常见防御：

- 重要资产使用 allowlist
- 监控 token implementation 是否变化
- 用 adapter / wrapper 隔离 token 行为
- upgrade 后暂停或冻结相关市场

### Blocklist / pausable token

USDC、USDT 这类 token 有 blocklist。协议地址如果被拉黑，可能无法转入或转出。

影响可能包括：

- AMM pool 资产卡死
- Vault 用户无法 withdraw
- Lending market 无法还款、清算或提款

Pausable token 也类似。admin 一旦 pause，相关 transfer 可能全部失败。

这类风险通常不能完全靠代码解决，需要风险披露、allowlist、emergency pause / withdraw 等机制。

### Approval quirks

有些 token 不允许从非零 allowance 直接改成另一个非零 allowance，USDT 是经典例子。

危险假设：

```solidity
token.approve(spender, newAmount);
```

更稳的做法通常是先清零再设置：

```solidity
token.safeApprove(spender, 0);
token.safeApprove(spender, newAmount);
```

或者根据场景使用 `safeIncreaseAllowance` / `safeDecreaseAllowance`。

审计时还要注意：

- approve 到 zero address 可能 revert
- approve `0` 可能 revert
- approve 很大数可能 revert

### Zero transfer / zero approval revert

有些 token 对 `0` value transfer 或 approval 会 revert。

如果协议在循环中统一转账：

```solidity
token.safeTransfer(user, amount);
```

而 `amount` 可能是 `0`，就可能因为某个 weird token 导致整个流程 DoS。

更稳的习惯：

```solidity
if (amount > 0) {
    token.safeTransfer(user, amount);
}
```

## 简短例子

- Fee-on-transfer：`transferFrom(user, pool, 100)`，pool 实际只收到 `98`。
- Balancer STA：transfer fee 曾被用于 drain Balancer pool。
- USDT：missing return value，并且 approve 行为非标准。
- USDC / USDT：可升级，也有 blocklist。
- ERC777 / imBTC：transfer hook 引入重入风险。

## 审计时注意

看到协议和 ERC20 交互时，要重点检查：

- 是否直接相信用户传入的 `amount`
- `transferFrom` 后是否使用 balance delta 计算实际收到数量
- mint、swap、deposit、repay 是否基于实际收到的 token
- transfer 后 reserve 是否和真实 balance 同步
- token transfer 是否可能重入
- 是否缓存 token balance、reserve、totalAssets
- token 是否可升级、可暂停、可 blocklist
- `amount == 0` 时是否仍然调用 token
- approve 是否兼容 USDT 类行为
- token revert 是否会导致核心流程 DoS
- 协议是否有 token allowlist 或 adapter

## Mitigation

常见缓解方式：

- 使用 `SafeERC20` 处理返回值兼容问题
- 对收入 token 使用 balance delta，而不是相信参数 `amount`
- 把所有 token call 当作外部调用，遵循 CEI
- 高风险入口加 `nonReentrant`
- 对支持的 token 做 allowlist
- 对可升级、可暂停、可 blocklist 的 token 做风险披露和监控
- 对特殊 token 使用 wrapper 或 adapter
- 跳过 `amount == 0` 的 transfer / approve
- 避免长期依赖过期的 cached balance

## 我的理解

Weird ERC20 的核心不是某一个固定漏洞，而是一组“协议错误相信 token 标准行为”的问题。

审计时我不能只看协议自己的 accounting，还要问：如果 token 少转了、没返回值、重入了、余额自己变了、被暂停了、被升级了，这套 accounting 还成立吗？

对 TSwap / AMM 来说，最重要的点是：用户传入的 `amount` 不等于池子实际收到的 `amount`。只要 swap、mint LP 或 reserve 更新用了错误的数量，就可能被 fee-on-transfer token 破坏。
