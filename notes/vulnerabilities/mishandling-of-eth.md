# Mishandling of ETH

## 一句话理解

合约里的 ETH 余额不一定只会通过你写的 `deposit()` 进入。

如果代码假设 `address(this).balance` 一定等于内部记账变量，就可能被强制转账打破，导致提现、结算或分配逻辑 revert。

## 典型问题

```solidity
uint256 public totalDeposits;
mapping(address => uint256) public deposits;

function deposit() external payable {
    deposits[msg.sender] += msg.value;
    totalDeposits += msg.value;
}

function withdraw() external {
    assert(address(this).balance == totalDeposits); // bad

    uint256 amount = deposits[msg.sender];
    totalDeposits -= amount;
    deposits[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```

问题在于：

```solidity
assert(address(this).balance == totalDeposits);
```

这个检查假设合约余额只能通过 `deposit()` 增加。

但 ETH 可以被强制发送到合约，比如：

- `selfdestruct`
- beacon chain withdrawal
- 其他协议层或特殊路径转入

即使合约没有 `receive` 或 `fallback`，也不能保证不会收到 ETH。

## 攻击结果

攻击者强制给合约转入 ETH 后：

```text
address(this).balance > totalDeposits
```

于是严格相等检查失败，`withdraw()` revert，用户资金可能被卡住。

## Mitigation

- 不要用严格相等依赖 `address(this).balance == internalAccounting`
- 更合理的是检查余额是否足够：

```solidity
require(address(this).balance >= totalDeposits);
```

- 内部会计应该以自己的状态变量为准
- 对多余 ETH 要有处理策略，比如 sweep、ignore、或单独记录
- 能用 pull payment 时，不要用复杂的批量 push payment

## 审计时注意

看到下面情况要多看一眼：

- `address(this).balance == someAccounting`
- `assert(address(this).balance == ...)`
- 代码假设没有 `receive/fallback` 就不能收 ETH
- 提现、分红、结算逻辑依赖合约实际 ETH balance
- 多余 ETH 会不会导致 DoS 或资金卡死

## 我的理解

Mishandling of ETH 的核心是：真实 ETH 余额和内部记账可能不一致。

合约不能假设“只有我的函数能让 ETH 进来”。如果关键逻辑依赖严格相等，一点被强制转入的 ETH 就可能把流程卡死。
