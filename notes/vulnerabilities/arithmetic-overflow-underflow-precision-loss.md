# Arithmetic: Overflow, Underflow, Precision Loss

## 一句话理解

智能合约里的整数运算要注意三类问题：

- Overflow: 数字超过类型最大值
- Underflow: 数字低于类型最小值
- Precision loss: 整数除法或缩放顺序导致精度丢失

## Solidity 版本差异

Solidity `0.8.0` 之前，整数 overflow / underflow 默认不会 revert，而是会 wrap around。

例如 `uint8` 最大值是 `255`：

```solidity
uint8 x = 255;
x = x + 1; // old Solidity: x becomes 0
```

Solidity `0.8.0` 及之后，整数 overflow / underflow 默认会 revert。

但是如果代码放在 `unchecked` 里，仍然会关闭检查：

```solidity
unchecked {
    x = x + 1;
}
```

所以审计时要同时关注：

- 合约使用的 Solidity 版本
- 是否存在 `unchecked`
- 是否有不安全的类型转换

## Overflow

Overflow 是数字超过该类型能表示的最大值。

```solidity
contract Overflow {
    uint8 public count;

    function increment(uint8 amount) public {
        unchecked {
            count = count + amount;
        }
    }
}
```

如果 `count = 200`，再加 `60`，数学上应该是 `260`。

但 `uint8` 最大只能到 `255`，在 unchecked 或 Solidity 0.8 之前会 wrap，结果变成 `4`。

## Underflow

Underflow 是数字低于该类型能表示的最小值。

```solidity
contract Underflow {
    uint8 public count;

    function decrement() public {
        unchecked {
            count--;
        }
    }
}
```

如果 `count = 0`，再减 `1`，在 unchecked 或 Solidity 0.8 之前会变成 `255`。

## Unsafe Casting 也可能造成类似问题

即使 Solidity 0.8+ 默认检查加减乘除，显式类型转换也要小心。

例如把大的 `uint256` 转成小的 `uint64`：

```solidity
uint256 fee = 100 ether;
uint64 smallFee = uint64(fee);
```

如果 `fee` 超过 `uint64` 最大值，高位数据会被截断，结果不是原来的值。

Puppy Raffle 里这行就值得警觉：

```solidity
totalFees = totalFees + uint64(fee);
```

问题点是：为什么要把 `fee` 从更大的整数类型 cast 成 `uint64`？如果 `fee` 过大，可能导致费用记录错误。

## Precision Loss

Solidity 没有浮点数，整数除法会直接向下取整。

```solidity
contract PrecisionLoss {
    uint256 public moneyToSplitUp = 225;
    uint256 public users = 4;

    function shareMoney() public view returns (uint256) {
        return moneyToSplitUp / users;
    }
}
```

数学上 `225 / 4 = 56.25`。

但 Solidity 里结果是 `56`，小数部分会被丢掉。

## 需要注意的金额计算

```solidity
uint256 prizePool = (totalAmountCollected * 80) / 100;
uint256 fee = (totalAmountCollected * 20) / 100;
```

这种写法不是最危险的写法。它先乘后除，通常比先除后乘更好，因为可以减少 precision loss。

但它仍然需要检查 rounding dust。

例如：

```solidity
uint256 totalAmountCollected = 101;

uint256 prizePool = (totalAmountCollected * 80) / 100; // 80
uint256 fee = (totalAmountCollected * 20) / 100;       // 20
```

`prizePool + fee = 100`，还有 `1 wei` 没有被分配。如果协议没有处理这部分 dust，可能导致小额资金滞留或会计不一致。

更危险的是：

```solidity
uint256 fee = totalAmountCollected / 100 * 20;
```

如果 `totalAmountCollected` 不能被 `100` 整除，先除法会提前丢精度。

## Mitigation

- 使用 Solidity `0.8.0` 或更高版本
- 谨慎使用 `unchecked`
- 避免不必要的小类型，比如 `uint64`
- 显式 cast 前检查范围
- 先乘后除，减少整数除法带来的 precision loss
- 对金额、份额、费用计算写边界测试
- 对大数、极小值、最大值写 fuzz test

## 审计时注意

看到下面这些代码要多看一眼：

- `unchecked`
- `uint8`、`uint16`、`uint64` 等小整数类型
- `uint64(x)`、`uint128(x)` 等向下 cast
- 费用、分红、奖励、份额计算
- 先除后乘的公式
- Solidity 版本低于 `0.8.0`
- 使用旧版 SafeMath 或没有 SafeMath 的老合约

## 我的理解

Overflow 和 underflow 是数字超出类型边界的问题。老版本 Solidity 会 wrap，高版本默认 revert，但 `unchecked` 和 unsafe casting 仍然可能引入风险。

Precision loss 不是溢出，而是整数数学天然没有小数。审计时看到除法，就要想：这一步丢掉的小数会不会影响资金、份额或奖励分配。
