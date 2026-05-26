# Weak Randomness

## 一句话理解

EVM 是确定性的状态机，链上没有真正的随机数。

如果随机数完全由链上数据计算出来，那么在交易被打包执行时，结果就已经可以被预测、影响或反复尝试。

## 为什么会发生

常见的伪随机写法：

```solidity
uint256 random = uint256(
    keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))
) % players.length;
```

这些值看起来很多变，但都不是安全随机源：

- `block.timestamp`: 验证者可以在一定范围内影响，攻击者也可以等待有利时间
- `block.prevrandao`: 不是给应用层安全随机设计的，也存在偏置和可预测性问题
- `msg.sender`: 调用者可以换地址或提前部署合约地址
- `blockhash`: 过去的 block hash 是公开的，未来的 block hash 也可能被出块者影响

## 攻击方式

### 预测结果

如果攻击者知道随机数公式，就可以在链下模拟结果。

当公式使用的输入都能提前知道或控制时，攻击者可以只在结果有利时发送交易。

### 反复 reroll

攻击者可以部署合约调用目标函数，如果结果不好就 revert。

因为 revert 会回滚状态，攻击者可以反复尝试，直到得到想要的结果。

Meebits 的案例就是类似问题：攻击者反复 mint，不喜欢结果就 revert，最终获得稀有 NFT 并卖出高价。

## 影响

- 抽奖 winner 可以被操纵
- NFT 稀有度可以被 reroll
- 游戏、盲盒、奖励分配失去公平性
- 攻击者可以用成本换概率优势，普通用户处于劣势
- 如果奖励价值高，协议可能出现严重经济损失

## Mitigation

### Chainlink VRF

更推荐使用 Chainlink VRF 这类可验证随机数服务。

VRF 的核心好处是：随机数在链下生成，同时带有可验证证明，合约在链上验证证明后再使用结果。

### Commit-Reveal

另一种方式是 commit-reveal：

1. 用户先提交 `hash(secret, salt)`，不暴露真实值
2. 之后进入 reveal 阶段，用户公开 `secret` 和 `salt`
3. 合约验证 hash 是否匹配
4. 用 reveal 的数据共同生成结果

这种方式要注意超时、用户不 reveal、最后一人优势等边界问题。

## 审计时注意

看到下面这些代码要重点检查：

- `keccak256(abi.encodePacked(...)) % n`
- 使用 `block.timestamp` 做随机数
- 使用 `block.prevrandao` / `block.difficulty` 做随机数
- 使用 `msg.sender`、`tx.origin` 做随机数
- NFT rarity、抽奖 winner、游戏结果、奖励分配
- 用户能否通过 revert 或重复调用 reroll 结果
- 结果是否在同一笔交易里生成并立即使用

## 我的理解

链上随机数最大的问题是：链上所有东西最终都要被所有节点确定性地算出同一个结果。

所以只要随机数完全来自链上，就不能把它当成真正不可预测的随机数。安全随机需要外部可信机制，比如 VRF，或者设计良好的 commit-reveal。
