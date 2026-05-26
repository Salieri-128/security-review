# DoS: Denial of Service

## 一句话理解

DoS 是指攻击者通过某种方式，让某个函数、交易、流程或协议步骤无法正常执行。

在智能合约里，它通常不是让整个链停掉，而是让协议中的某个关键功能失效，比如：

- 用户无法进入某个活动
- 无法完成分红
- 无法提款
- 无法结算
- 无法继续执行协议流程

## 为什么重要

很多协议功能依赖“某个函数一定能执行成功”。如果攻击者能让这个函数持续 revert，或者让它的 gas 成本高到超过区块 gas limit，那么这个功能就会被卡死。

这类问题可能导致资金无法分配、用户无法参与、协议无法推进到下一阶段。

## 常见原因 1：无限制数组 + loop

如果合约里存在一个不断增长的 array，并且某些函数会遍历整个 array，就要特别小心。

危险组合：

- array 没有长度限制
- 加入 array 的成本很低
- 函数里有 `for` loop 遍历 array
- loop 的 gas 消耗会随着 array 变大而增加

攻击者可以用很多地址反复加入 array，把 array 撑得很大。最后任何需要遍历这个 array 的函数都可能因为 gas 太高而无法执行。

## 例子

```solidity
contract DoS {
    address[] entrants;

    function enter() public {
        // Check for duplicate entrants
        for (uint256 i; i < entrants.length; i++) {
            if (entrants[i] == msg.sender) {
                revert("You've already entered!");
            }
        }
        entrants.push(msg.sender);
    }
}
```

这个例子的问题是：每次 `enter` 都要遍历全部 `entrants` 来检查重复。

当 `entrants` 很小时没有问题；但如果攻击者创建大量账户进入，`entrants.length` 会不断变大，后面的用户调用 `enter` 需要消耗越来越多 gas，最终可能无法进入。

## 常见原因 2：依赖外部合约调用

另一个常见 DoS 来源是函数里对外部地址或外部合约发起调用。

如果外部合约：

- 不存在对应函数
- 没有 `receive` 或 payable `fallback`
- 主动 revert
- 消耗大量 gas
- 是恶意合约

那么当前函数也可能被一起 revert，导致整个流程失败。

典型场景：

一个 AMM 或质押池统一给 LP 分红。如果分红函数用 loop 给每个 LP 转账，只要其中一个 LP 是不能接收 ETH 的恶意合约，转账时 revert，就可能导致整个分红函数失败。

## 攻击路径

### Array DoS

1. 找到一个可以低成本加入的 array。
2. 确认某个关键函数会遍历这个 array。
3. 用大量地址或多次操作扩大 array。
4. 让 loop 的 gas 消耗越来越高。
5. 最终使关键函数超过 gas limit 或变得不可用。

### External Call DoS

1. 找到协议中依赖外部地址接收 ETH 或执行回调的函数。
2. 部署一个会 revert 的恶意合约。
3. 让恶意合约成为接收方或参与者。
4. 当协议统一处理支付、分红、结算时，恶意合约触发 revert。
5. 整个流程失败。

## 影响

- 用户无法参与协议
- 奖励或分红无法发放
- 提款或结算被卡住
- 协议流程无法继续
- 资金可能长时间锁定在合约中

## 缓解方式

### 对 array 和 loop

- 避免在关键函数中遍历无限增长的 array
- 给 array 设置合理上限
- 使用 mapping 做 O(1) 查询，比如检查是否已经加入
- 把大循环拆成多次执行，使用分页或批处理
- 避免让攻击者能低成本扩大需要遍历的数据结构

例如，上面的重复检查可以改成：

```solidity
mapping(address => bool) hasEntered;
address[] entrants;

function enter() public {
    if (hasEntered[msg.sender]) {
        revert("You've already entered!");
    }

    hasEntered[msg.sender] = true;
    entrants.push(msg.sender);
}
```

### 对外部调用

- 避免在 loop 中直接 push payment
- 使用 pull payment，让用户自己领取资金
- 外部调用失败时，不要让整个系统关键流程被卡死
- 对外部调用做好失败处理
- 重要流程中谨慎使用 `call`
- 检查接收方是否可能是恶意合约

## 审计时注意

看到下面几类代码要重点检查：

- `for` loop 遍历 storage array
- array 可以被任意用户低成本 push
- 函数 gas 消耗随用户数量线性增长
- 在 loop 中转账或调用外部合约
- 分红、退款、结算、清算等批量处理逻辑
- 关键流程依赖某个外部地址必须成功接收 ETH

## 我的理解

DoS 的核心不是“偷钱”，而是“让事情做不成”。

审计时我应该关注两类问题：第一，数据结构会不会被攻击者无限放大，导致 loop 跑不动；第二，关键流程会不会因为某个外部合约 revert 而整体失败。
