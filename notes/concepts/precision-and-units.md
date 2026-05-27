# Precision and Units in Solidity

## 一句话理解

EVM 只认识整数，没有浮点数，也没有默认 `1e18` 精度。

`1e18` 只是 ETH、很多 ERC20 和 DeFi 协议里常用的人为放大倍数。

## 三个概念不要混

### EVM / uint256

```solidity
uint256 a = 2;
```

就是整数 `2`，不会自动变成 `2e18`。

### ETH

EVM 里 ETH 用 wei 记录。

```solidity
1 ether == 1e18 wei
```

所以 `uint256 amount = 1` 是 `1 wei`，不是 `1 ETH`。

### ERC20 decimals

`decimals()` 只是显示约定，不会自动影响合约计算。

18 decimals 的 ERC20 里：

```text
1 token = 1e18 smallest units
```

所以 `_mint(user, 1)` 只是 mint `1` 个最小单位，不是 1 个完整 token。

## Precision Loss

Solidity 整数除法会向下取整：

```solidity
5 / 2 == 2
```

没有 `2.5`。

常见问题是先除后乘：

```solidity
reward = userAmount / totalAmount * rewardPool;
```

如果 `userAmount < totalAmount`，前半段可能直接变成 `0`。

通常应该先乘后除：

```solidity
reward = userAmount * rewardPool / totalAmount;
```

## WAD / RAY

DeFi 常见缩放精度：

- `WAD = 1e18`
- `RAY = 1e27`

它们只是协议自己约定的 scaling factor，不是 EVM 默认规则。

## 审计时注意

- 不要默认所有 token 都是 18 decimals
- 不要把 `1` 当成 `1 ETH` 或 `1 token`
- 注意先除后乘
- 注意 WAD、RAY、token decimals 混用
- 注意 rounding dust 和 rounding 方向

## 我的理解

`uint256 a = 2` 就是 `2`。

如果表示 ETH，它是 `2 wei`。

如果表示 18 decimals ERC20，它是 `0.000000000000000002 token`。

`1e18` 是人为放大倍数，不是 Solidity 自动精度。
