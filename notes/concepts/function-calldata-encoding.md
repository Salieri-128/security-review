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

## abi.encodeWithSignature

`abi.encodeWithSignature(...)` 用函数签名和参数生成 calldata。

没有参数：

```solidity
abi.encodeWithSignature("getReserves()")
```

生成的 calldata 只有：

```text
bytes4(keccak256("getReserves()"))
```

也就是 `getReserves()` 的 function selector。

有参数：

```solidity
abi.encodeWithSignature("transfer(address,uint256)", to, amount)
```

生成：

```text
transfer(address,uint256) selector + abi encoded to + abi encoded amount
```

注意：函数签名字符串只写类型，不写参数名。

## call 的返回值

低级 `call` 返回两个值：

```solidity
(bool success, bytes memory result) = target.call(data);
```

- `success`: 调用是否成功
- `result`: 目标函数返回的 ABI encoded bytes

如果目标函数返回 `uint256[]`：

```solidity
uint256[] memory reserves = abi.decode(result, (uint256[]));
```

如果目标函数返回多个值：

```solidity
(uint256 a, address b) = abi.decode(result, (uint256, address));
```

`result` 只是原始 bytes，必须按正确返回类型 `abi.decode`。

## msg.sender.call 调的是谁

在某个函数里写：

```solidity
msg.sender.call(abi.encodeWithSignature("getReserves()"));
```

意思是：对当前 `msg.sender` 这个地址发起低级调用。

如果这段代码在 fallback 里：

```solidity
fallback() external payable {
    (bool success, bytes memory result) = msg.sender.call(
        abi.encodeWithSignature("getReserves()")
    );
}
```

那么 `msg.sender` 是触发 fallback 的那个外部合约。

调用链可以理解成：

```text
Well -> MockCallbackRecipient.fallback()
          msg.sender == Well
          msg.sender.call("getReserves()") -> Well.getReserves()
```

注意：这不是调用自己的 `getReserves()`。

如果要调用自己，才是：

```solidity
address(this).call(abi.encodeWithSignature("getReserves()"));
```

## 高级调用 vs 低级 call

如果有 interface，更推荐高级调用：

```solidity
uint256[] memory reserves = IWell(msg.sender).getReserves();
```

低级 call：

```solidity
(bool success, bytes memory result) = msg.sender.call(
    abi.encodeWithSignature("getReserves()")
);
```

低级 call 更灵活，但缺点是：

- 编译器不检查函数是否存在
- 签名字符串写错不会编译报错
- 返回值是 bytes，需要手动 decode
- 必须自己检查 `success`

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
- `call` 返回的 `result` 是否按正确类型 decode
- `value` 和函数参数是否被混淆
- 空 calldata 转账是否会触发目标合约的 `receive` / `fallback`
- 任意 calldata 调用是否可能让用户调用不该调用的函数
- fallback 里使用 `msg.sender.call(...)` 是否会反向调用外部调用者

## 我的理解

Solidity 里的 `target.someFunction(x)` 最后也会变成 calldata。

EVM 不认识函数名，它只看 `data` 前 4 bytes 的 selector，然后用后面的 ABI 编码参数执行函数。

低级 `call` 只是让我们手动构造这段 `data`，并把返回值当作 bytes 拿回来，再自己 `abi.decode`。
