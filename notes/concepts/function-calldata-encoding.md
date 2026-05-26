# Function Calldata Encoding

## 一句话理解

调用合约函数时，EVM 看到的不是函数名，而是一段 `calldata`。

这段 `calldata` 通常由两部分组成：

```text
4 bytes function selector + ABI encoded arguments
```

## calldata 是什么

交易或低级调用里有一个 `data` 字段。

如果是部署合约，`data` 里放的是合约 creation bytecode。

如果是调用函数，`data` 里放的是 ABI 编码后的函数调用信息，也就是 calldata。

EVM 根据 calldata 的前 4 bytes 判断要执行哪个函数。

## function selector

function selector 是函数签名 hash 的前 4 bytes。

函数签名只包含函数名和参数类型，不包含参数名。

```solidity
transfer(address,uint256)
```

selector 计算方式：

```solidity
bytes4 selector = bytes4(keccak256("transfer(address,uint256)"));
```

调用 `transfer(to, amount)` 时，calldata 大概是：

```text
0xa9059cbb + abi.encode(to, amount)
```

其中 `0xa9059cbb` 就是 ERC20 `transfer(address,uint256)` 的 selector。

## 参数编码

selector 后面接的是参数的 ABI 编码。

```solidity
abi.encodeWithSelector(
    IERC20.transfer.selector,
    to,
    amount
)
```

等价于：

```solidity
abi.encodeWithSignature(
    "transfer(address,uint256)",
    to,
    amount
)
```

更推荐 `abi.encodeWithSelector` 或 `abi.encodeCall`，因为它们更不容易写错函数签名字符串。

## low-level call

低级调用时，括号里的 `bytes` 就是要传给目标合约的 calldata。

```solidity
(bool success, bytes memory returnData) = token.call(
    abi.encodeWithSelector(IERC20.transfer.selector, to, amount)
);
```

如果括号里是空 bytes：

```solidity
recipient.call{value: amount}("");
```

意思是只发送 ETH，不指定函数。目标如果是合约，可能触发 `receive` 或 `fallback`。

## call 和 staticcall

常见低级调用：

- `call`: 可以修改状态，也可以发送 ETH
- `staticcall`: 只读调用，不允许修改状态
- `delegatecall`: 执行别的合约代码，但使用当前合约的 storage

例子：

```solidity
(bool ok, bytes memory data) = target.staticcall(
    abi.encodeWithSignature("balanceOf(address)", user)
);
```

## msg.value 和 calldata 是两件事

`value` 是这次调用附带多少 ETH。

`data` 是这次调用要执行什么函数、传什么参数。

```solidity
target.call{value: 1 ether}(
    abi.encodeWithSignature("deposit(uint256)", amount)
);
```

可以理解为：

```text
value: 1 ether
data:  deposit(uint256) 的 selector + amount 的 ABI 编码
```

## 审计时注意

- 低级 `call` 是否检查了 `success`
- calldata 是否使用了正确的 selector
- 函数签名字符串是否拼错，比如空格、类型不一致
- `value` 和函数参数是否被混淆
- 空 calldata 转账是否会触发目标合约的 `receive` / `fallback`
- 任意 calldata 调用是否可能让用户调用不该调用的函数

## 我的理解

Solidity 里的 `target.someFunction(x)` 最后也会变成 calldata。

EVM 不认识函数名，它只看 `data` 前 4 bytes 的 selector，然后用后面的 ABI 编码参数执行函数。低级 `call` 只是让我们手动构造这段 `data`。
