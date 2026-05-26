# fallback vs receive

## 一句话理解

`receive` 专门处理“空 calldata 的收 ETH”。

`fallback` 专门处理“函数不存在，或者 calldata 匹配不到函数”的情况，也可以在某些情况下收 ETH。

## receive

`receive()` 是合约收到 ETH 时的特殊函数。

触发条件：

- 调用时带 ETH
- `calldata` 为空
- 合约中定义了 `receive`

写法：

```solidity
receive() external payable {
    // receive ETH
}
```

注意：

- 必须是 `external payable`
- 没有 `function` 关键字
- 没有参数，也没有返回值
- 最常见用途是让合约可以直接收 ETH

## fallback

`fallback()` 是兜底函数。

触发条件：

- 调用的函数不存在
- 或者 calldata 无法匹配任何函数
- 如果没有 `receive`，空 calldata 收 ETH 时也可能触发 payable fallback

写法：

```solidity
fallback() external payable {
    // fallback logic
}
```

注意：

- 可以是 `payable`，也可以不是
- 如果希望 fallback 能收 ETH，必须加 `payable`
- 常见于代理合约，因为代理需要把未知函数调用转发给实现合约

## 对比

| 场景 | 触发函数 |
| --- | --- |
| 空 calldata + 发送 ETH + 有 receive | `receive` |
| 空 calldata + 发送 ETH + 没有 receive + fallback payable | `fallback` |
| 调用不存在的函数 | `fallback` |
| calldata 不为空，但匹配不到函数 | `fallback` |
| fallback 不 payable，却收到 ETH | revert |

## 小例子

```solidity
contract Example {
    event Received(address sender, uint256 amount);
    event FallbackCalled(address sender, uint256 amount, bytes data);

    receive() external payable {
        emit Received(msg.sender, msg.value);
    }

    fallback() external payable {
        emit FallbackCalled(msg.sender, msg.value, msg.data);
    }
}
```

如果用户直接转 ETH，且没有附带 calldata，会进入 `receive`。

如果用户调用一个不存在的函数，会进入 `fallback`。

## 审计时注意

- 合约是否本应接收 ETH？如果不应该，为什么 `receive` 或 payable `fallback` 存在？
- `fallback` 里有没有复杂逻辑、外部调用或 delegatecall？
- 代理合约的 `fallback` 是否正确转发 calldata 和 returndata？
- `receive` 或 `fallback` 收到 ETH 后，资金是否有提取路径？
- 是否有人能通过 fallback 绕过正常函数的限制？

## 我的理解

`receive` 更像“收款入口”。

`fallback` 更像“没找到函数时的兜底入口”。

如果合约写了这两个函数，审计时要重点确认：它为什么需要接收 ETH，收到后怎么处理，以及 fallback 有没有隐藏的转发或权限风险。
