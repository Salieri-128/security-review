# Reentrancy

## 一句话理解

Reentrancy 是指合约在关键状态更新之前，先调用了外部合约，导致外部合约可以在状态还没改变时再次调用回来。

最典型的场景是 `withdraw` 或 `refund`：钱已经转出去了，但用户余额还没归零，攻击合约就通过 `receive` / `fallback` 再次调用提款函数。

## 为什么会发生

核心原因是：合约主动把控制权交给了外部地址。

危险顺序通常是：

```solidity
function withdraw() external {
    uint256 amount = balances[msg.sender];

    (bool ok, ) = msg.sender.call{value: amount}("");
    require(ok);

    balances[msg.sender] = 0;
}
```

问题在于，`call` 发生在 `balances[msg.sender] = 0` 之前。

如果 `msg.sender` 是攻击合约，收到 ETH 时会触发它的 `receive` 或 `fallback`，然后它可以再次调用 `withdraw()`。由于余额还没清零，第二次进入时读到的还是原来的余额。

## 攻击流程

```text
Attacker.attack()
  -> Bank.withdraw()
       amount = balances[Attacker]
       -> send ETH to Attacker
            -> Attacker.receive()
                 -> Bank.withdraw() again
                      amount is still unchanged
```

这不是普通的函数内部递归，而是通过外部调用形成的跨合约递归调用链。

## 影响

- 攻击者可以重复提款或重复退款
- 合约内部记账状态没有及时更新
- 最坏情况下，合约中的 ETH 或 token 被全部取走
- 依赖余额、份额、奖励、退款状态的逻辑都会被破坏

## 常见危险信号

看到下面的模式要重点检查：

- 外部调用发生在状态更新之前
- 使用 `call{value: amount}("")` 给用户转 ETH
- `withdraw`、`refund`、`claim`、`redeem`、`unstake` 等函数
- 函数依赖 `balances[msg.sender]`、`shares[msg.sender]` 等用户状态
- 外部调用目标可能是合约
- 函数没有 reentrancy lock

## Mitigation 1: CEI Pattern

CEI 是 Checks-Effects-Interactions：

- Checks: 先检查条件
- Effects: 再修改状态
- Interactions: 最后才和外部合约交互

修复后的顺序：

```solidity
function withdraw() external {
    uint256 amount = balances[msg.sender];
    if (amount == 0) revert();

    balances[msg.sender] = 0;

    (bool ok, ) = msg.sender.call{value: amount}("");
    require(ok);
}
```

这样即使攻击合约在收到 ETH 后再次调用 `withdraw()`，它的余额也已经是 `0`。

## Mitigation 2: Reentrancy Lock

另一种方式是在函数执行期间加锁，保证同一个受保护函数不能被重复进入。

简化理解：

```solidity
bool locked;

modifier noReentrant() {
    require(!locked, "reentrant call");
    locked = true;
    _;
    locked = false;
}
```

OpenZeppelin 提供了 `ReentrancyGuard`，常用 modifier 是 `nonReentrant`：

```solidity
function withdraw() external nonReentrant {
    // withdraw logic
}
```

注意：加锁不能代替良好的状态更新顺序。更好的习惯是优先使用 CEI，再根据风险加 `nonReentrant`。

## 审计时注意

检查外部调用时，要问：

- 外部调用之前，关键状态是否已经更新？
- 如果接收方是合约，它能不能通过 `receive` / `fallback` 调用回来？
- 第二次进入时，余额、份额、领取状态是否还是旧值？
- 有没有跨函数重入，例如从 `withdraw` 回调进 `claim`？
- `nonReentrant` 是否覆盖了所有相关入口？

## 我的理解

Reentrancy 的重点不是“函数里写了递归”，而是“状态还没改，就先把控制权交给了外部合约”。

外部合约拿到控制权后，可以调用回来，让原合约在旧状态下重复执行某些操作。最典型结果就是重复提款、重复退款，严重时把合约资金全部取走。

现在这类基础重入已经不算新鲜漏洞，但只要代码里有外部调用，尤其是提款和退款逻辑，还是必须检查 CEI 和 reentrancy guard。
