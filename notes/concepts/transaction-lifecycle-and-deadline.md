# Transaction Lifecycle and Deadline

## 一句话理解

用户签名并广播交易，不等于合约已经执行。

交易真正执行是在它被打包进区块时，EVM 才会按照当时的链上状态运行函数。

所以交易从发出到执行之间，可能已经过了几秒、几十秒，甚至更久；池子价格、储备、nonce、gas、MEV 环境都可能变化。

## 一笔交易经历什么

假设用户提交：

```solidity
deposit(
    10 ether,
    minimumLiquidityTokensToMint,
    maximumPoolTokensToDeposit,
    deadline
)
```

大致流程是：

```text
用户签名交易
    ↓
广播给节点
    ↓
进入 public mempool
    ↓
等待验证者 / builder 选择
    ↓
被打包进某个区块
    ↓
EVM 此时才执行 deposit()
    ↓
成功或 revert
```

关键点：合约代码不是在交易进入 mempool 时执行，而是在区块执行阶段才执行。

## 等待期间可能发生什么

交易等待打包时，链上状态不会为这笔交易静止。

可能发生：

- AMM 的 WETH reserve 变化
- Pool Token reserve 变化
- 其他用户先 swap、deposit、withdraw
- LP Token 兑换比例变化
- 预言机价格变化
- 用户 nonce 被前面的交易卡住
- gas price 太低，交易长时间不被打包
- MEV 搜索者观察、排序、夹击或构造 bundle

所以用户在前端看到的报价，只是签名当时的估算；真正执行时会重新读取当时的链上状态。

例如：

```solidity
uint256 wethReserves = i_wethToken.balanceOf(address(this));
uint256 poolTokenReserves = i_poolToken.balanceOf(address(this));
```

这些值读取的是执行时的余额，不是用户点击确认时的余额。

## Deadline 是什么

`deadline` 表示：用户只允许这笔交易在某个时间点之前执行；超过这个时间，即使交易还在链上有效，也必须 revert。

常见写法：

```solidity
modifier revertIfDeadlinePassed(uint64 deadline) {
    if (block.timestamp > deadline) {
        revert TSwapPool__DeadlinePassed(deadline);
    }
    _;
}
```

然后用于入口函数：

```solidity
function deposit(...)
    external
    revertIfDeadlinePassed(deadline)
    revertIfZero(wethToDeposit)
{
    ...
}
```

如果用户在 12:00 发交易，并设置：

```text
deadline = 12:20
```

那么：

- 12:05 被打包：可以执行
- 12:19 被打包：可以执行
- 12:21 被打包：revert
- 第二天被打包：revert

检查的是执行区块的 `block.timestamp`，不是用户签名时间，也不是交易进入 mempool 的时间。

## Deadline 和滑点保护的区别

滑点参数保护的是“成交结果”。

例如：

- `minimumLiquidityTokensToMint`：最少必须获得多少 LP Token
- `maximumPoolTokensToDeposit`：最多允许付出多少 Pool Token

`deadline` 保护的是“成交时间”。

它们解决的是不同问题：

| 参数 | 保护什么 |
| --- | --- |
| `minimumLiquidityTokensToMint` | 最少收到多少 LP Token |
| `maximumPoolTokensToDeposit` | 最多付出多少 Pool Token |
| `deadline` | 最晚允许什么时候执行 |

比较完整的用户意图是：

```text
我愿意在未来 20 分钟内，
最多投入 10,100 个 Pool Token，
并且至少获得 9.9 个 LP Token。
```

少了 deadline，就变成：

```text
无论过多久，只要还能满足价格边界，就可以执行。
```

这可能不是用户当前的真实意愿。

## 没有 Deadline 的风险

假设用户提交交易时，池子状态是：

```text
100 WETH
100,000 TOKEN
```

用户愿意存入：

```text
10 WETH
最多 10,100 TOKEN
最低 9.9 LP Token
```

但交易长时间没有被打包。

等待期间，其他人 swap 后，池子变成：

```text
100 WETH
101,000 TOKEN
```

当交易之后被打包时，`deposit()` 会使用新的 reserve 重新计算需要投入的 Pool Token 和能 mint 的 LP Token。

如果结果仍然勉强满足：

```text
poolTokensToDeposit <= maximumPoolTokensToDeposit
liquidityTokensToMint >= minimumLiquidityTokensToMint
```

交易就会成功，即使它已经远远晚于用户本来能接受的时间窗口。

这就是“过期意图继续执行”的问题。

## MEV 相关风险

交易进入 public mempool 后，搜索者可以看到交易参数，例如：

- 用户准备 deposit 多少 WETH
- 最多接受多少 Pool Token 支出
- 最少接受多少 LP Token
- deadline 是多少

如果合约没有检查 deadline，交易的有效窗口会变长。

搜索者或 builder 可能有更多机会：

- 观察交易
- 在交易前改变池子储备
- 让交易靠近滑点边界执行
- 在交易后反向操作获利

如果函数本身已经有滑点边界，攻击者不能无限改变价格，否则交易会 revert。但缺少 deadline 仍然会扩大交易被延迟执行和被 MEV 利用的窗口。

## Deadline 不能保证立刻执行

`deadline = block.timestamp + 10 minutes` 不表示交易一定会在 10 分钟内被打包。

它只表示：

```text
如果超过 10 分钟才执行，合约主动 revert。
```

交易仍然可能：

- 一直留在 mempool
- 被钱包替换或取消
- 超时后被打包，但执行 revert
- 因为前序 nonce 卡住而无法执行

deadline 是合约层面的有效期检查，不是区块链调度机制。

## 为什么不能检查进入 mempool 的时间

合约不知道交易什么时候进入 mempool。

EVM 能看到的是执行环境，例如：

- `block.timestamp`
- `block.number`
- `msg.sender`
- `msg.value`
- `tx.gasprice`

但它看不到：

- 交易第一次广播时间
- 交易进入某个节点 mempool 的时间
- 交易在 mempool 等待了多久

而且 mempool 不是全球统一的。不同节点看到交易的时间不同，私有交易甚至可能完全不进入 public mempool。

所以用户传入一个绝对时间戳 `deadline`，合约执行时用 `block.timestamp > deadline` 判断是否过期，是最自然的设计。

## 常见 Bug

最容易出现的问题不是没有 deadline 参数，而是参数存在但没有使用：

```solidity
function deposit(..., uint64 deadline) external {
    // completely ignores deadline
}
```

前端可能正常传：

```javascript
deadline = Math.floor(Date.now() / 1000) + 1200;
```

用户以为交易 20 分钟后失效，但合约完全无视这个限制。

正确做法是把 deadline 接入 modifier：

```solidity
function deposit(
    uint256 wethToDeposit,
    uint256 minimumLiquidityTokensToMint,
    uint256 maximumPoolTokensToDeposit,
    uint64 deadline
)
    external
    revertIfDeadlinePassed(deadline)
    revertIfZero(wethToDeposit)
    returns (uint256 liquidityTokensToMint)
{
    ...
}
```

## 审计时注意

看到用户交易会依赖当前价格、储备或兑换比例时，要检查：

- 函数是否有 deadline 参数
- deadline 参数是否真的被使用
- 检查逻辑是否是 `block.timestamp > deadline`
- deadline 是否覆盖 swap、deposit、withdraw、remove liquidity 等入口
- 是否同时有滑点保护和 deadline
- 滑点保护是否使用执行时的真实 reserve / balance
- 交易延迟执行后，用户意图是否仍然成立
- public mempool 中的参数是否会被 MEV 利用
- 前端传了 deadline，不代表合约一定检查了 deadline

## 我的理解

滑点参数防止“成交得太差”，deadline 防止“成交得太晚”。

交易签名和广播只是表达用户意图；真正改变链上状态的是区块执行阶段。审计时要站在执行时的状态看代码，而不是站在用户点击确认时的状态看代码。

如果合约声明了 `deadline` 却没有检查，它就是一种假安全：用户和前端以为交易会过期，但协议实际上允许过期意图继续执行。
