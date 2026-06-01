# Read-only Reentrancy

## 一句话理解

Read-only reentrancy 是一种“只读重入”。

攻击者不一定要重入修改状态，而是在目标合约状态还没更新完时，重入调用 view/read 函数，读到旧状态或中间状态。

## 和普通 reentrancy 的区别

普通 reentrancy:

```text
外部调用 -> 攻击合约回调 -> 再次调用修改状态的函数 -> 重复取钱/重复操作
```

Read-only reentrancy:

```text
外部调用 -> 攻击合约回调 -> 调用 view/read 函数 -> 读到未更新的状态
```

它看起来只是读数据，但如果第三方协议、oracle、AMM、借贷协议依赖这个读数，就可能被错误价格或错误状态影响。

## 为什么会发生

核心原因还是没有遵守 CEI。

比如：

```solidity
function removeLiquidity(...) external nonReentrant {
    // calculate amounts

    token.safeTransfer(recipient, amount); // external call

    reserves = newReserves; // state update happens after external call
}
```

即使 `removeLiquidity` 有 `nonReentrant`，攻击者可能不能再次进入 `removeLiquidity`。

但如果 `getReserves()` 没有检查 reentrancy guard，攻击合约仍然可以在 callback/fallback 里调用：

```solidity
well.getReserves();
```

这时读到的可能还是旧 reserves。

## Beanstalk Wells 案例

在 Beanstalk Wells finding 中，`removeLiquidity` 会先转出 token，再设置新的 reserve values。

如果 recipient 是带 callback 的合约，或者使用 ERC777 这类带 hook 的 token，外部调用期间可以触发 callback。

callback 里可以反向调用：

```solidity
(bool success, bytes memory result) = msg.sender.call(
    abi.encodeWithSignature("getReserves()")
);

uint256[] memory reserves = abi.decode(result, (uint256[]));
```

这里 `msg.sender` 是触发 callback 的 Well 合约，所以这行是在 callback 中读 Well 的 `getReserves()`。

问题是：此时 `removeLiquidity` 还没更新 reserves，所以返回值可能是旧数据。

## 影响

- oracle / pump 读取到旧 reserves
- 第三方协议基于错误状态计算价格
- AMM、借贷、清算、抵押率等逻辑被影响
- 目标协议本身可能有 `nonReentrant`，但集成方仍然可能受害

所以 read-only reentrancy 的危险点常常不在“本合约能不能被再次提款”，而在“别人会不会信任这个 view 函数返回值”。

## Mitigation

- 遵守 CEI：先更新状态，再做外部调用
- view/read 函数也可以检查 reentrancy guard
- 对外暴露的读函数不要返回中间状态
- 外部集成方不要盲目信任可能处于 reentrant window 的读数
- 对 oracle / reserve / price 相关函数尤其谨慎

例如：

```solidity
function getReserves() external view returns (uint256[] memory) {
    if (_reentrancyGuardEntered()) {
        revert("reentrant read");
    }
    return reserves;
}
```

## 审计时注意

看到下面情况要多看一眼：

- 外部调用发生在状态更新之前
- `nonReentrant` 只保护写函数，没有保护关键 read 函数
- view 函数返回 reserve、price、share、oracle 数据
- callback/hook token，例如 ERC777
- AMM remove/add liquidity、swap、oracle update
- 第三方协议会读取该合约的 view 函数做决策

## 我的理解

Read-only reentrancy 的关键不是“重入后修改状态”，而是“重入后读取还没更新完的状态”。

它通常发生在外部调用窗口里：目标合约已经做了一部分操作，但还没有把关键状态写回去。攻击合约利用 fallback/callback 反向调用 view 函数，就能看到旧值或中间值。

所以 view 函数不一定天然安全。只要它返回的数据会被其他协议用于价格、抵押、清算或分配，就要考虑 read-only reentrancy。
