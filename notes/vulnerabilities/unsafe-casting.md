# Unsafe Casting

## 一句话理解

Unsafe casting 是把一个大类型强制转成小类型时，没有检查数值范围，导致高位数据被截断，结果变小或变成错误值。

## 典型例子

```solidity
uint256 fee = 20e18;
uint64 smallFee = uint64(fee);
```

`uint64` 最大值是：

```text
18446744073709551615
```

但 `20e18` 是：

```text
20000000000000000000
```

超过 `uint64` 能表示的范围，cast 后不会得到原来的 `20e18`，而是被截断成另一个值。

## uint256 转 uint64 的直觉

`uint256` 转成 `uint64` 时，可以理解成只保留最后 64 bits。

高位数据会被丢掉，低 64 位留下来。

数学效果类似：

```text
uint64(x) == x % 2^64
```

所以如果 `x` 大于 `type(uint64).max`，cast 后的值会变成一个更小的错误值。

## Puppy Raffle 里的问题

```solidity
totalFees = totalFees + uint64(fee);
```

这里 `fee` 原本是 `uint256`，但被强制转成 `uint64`。

如果 `fee` 大于 `type(uint64).max`，记录到 `totalFees` 里的值会变错，协议可能少记录大量 fees。

## 和 overflow 的区别

Overflow 通常是运算超过类型上限。

Unsafe casting 是类型转换时超过目标类型上限。

它们结果都可能表现得像 wrap/truncation，但触发点不同。

## Mitigation

- 尽量避免把大整数 cast 成小整数
- 如果必须 cast，先检查范围

```solidity
require(fee <= type(uint64).max, "fee too large");
totalFees = totalFees + uint64(fee);
```

- 更简单的方式：直接让 `totalFees` 使用 `uint256`

## 审计时注意

看到下面这些写法要警觉：

- `uint64(x)`
- `uint128(x)`
- `uint8(x)`
- `int256` 转 `uint256`
- 金额、fees、shares、balances 被 cast 成更小类型

## 我的理解

Solidity 0.8+ 会检查普通加减乘除的 overflow，但显式 cast 仍然要自己小心。

看到“大类型 -> 小类型”，第一反应应该是：这个值有没有可能超过目标类型最大值？
