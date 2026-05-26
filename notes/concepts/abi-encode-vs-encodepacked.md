# abi.encode vs abi.encodePacked

## 一句话理解

`abi.encode` 是标准 ABI 编码，信息完整、可以安全解码。

`abi.encodePacked` 是紧凑编码，结果更短，但会丢掉一些边界信息，多个动态类型一起用时可能产生碰撞。

## abi.encode

`abi.encode(...)` 会按照 Solidity ABI 规范编码数据。

可以简单理解成：把数据放进一个个 32 bytes 的格子里。

例如 `uint256(1)` 会被编码成 32 bytes：

```text
0x0000000000000000000000000000000000000000000000000000000000000001
```

数字、地址、bool 这类静态类型会被补齐到 32 bytes。这样虽然更长，但每个数据的位置和大小都很清楚。

特点：

- 每个值通常按 32 bytes 对齐
- 动态类型会保存偏移量和长度
- 编码结果更长
- 可以用 `abi.decode` 正确解码回来
- 更适合函数调用参数、跨合约交互、需要还原原始数据的场景

例子：

```solidity
bytes memory data = abi.encode("hello", uint256(123));
(string memory text, uint256 num) = abi.decode(data, (string, uint256));
```

## abi.encodePacked

`abi.encodePacked(...)` 会把数据尽量紧凑地拼接在一起。

可以简单理解成：不再把每个数据都放进完整的 32 bytes 格子，而是尽量去掉 ABI 标准编码里的填充部分，然后把结果直接拼在一起。

例如较小的整数或短数据，在 packed 编码里不会保留完整的 32 bytes 填充：

```solidity
abi.encode(uint16(1));       // 32 bytes，前面补很多 0
abi.encodePacked(uint16(1)); // 2 bytes，只保留 uint16 本身的大小
```

对于 `string` / `bytes` 这种动态类型，`abi.encodePacked` 会直接拼接原始内容，不保存长度，也不保存边界。

特点：

- 编码结果更短
- 不按标准 ABI 的完整格式保存所有边界信息
- 通常不能直接用 `abi.decode` 解码，也很难可靠复原原始输入
- 常用于生成 hash，比如 `keccak256`

例子：

```solidity
bytes32 hash = keccak256(abi.encodePacked(msg.sender, amount));
```

## 重要风险：hash 碰撞

当 `abi.encodePacked` 同时编码多个动态类型时，可能出现不同输入得到相同编码结果。

例如：

```solidity
abi.encodePacked("ab", "c");
abi.encodePacked("a", "bc");
```

这两者都会变成类似 `"abc"` 的紧凑结果。如果再拿去 hash，就会得到相同的 hash。

问题的根源是：packed 编码之后只看到拼接结果，无法知道原来边界在哪里。看到 `"abc"` 时，不知道它原来是 `("ab", "c")`，还是 `("a", "bc")`。

这在签名校验、权限校验、唯一 ID 生成中可能变成安全问题。

## 怎么选

- 需要标准编码、未来要解码：用 `abi.encode`
- 要 hash 固定长度数据：可以用 `abi.encodePacked`
- 要 hash 多个动态类型：优先用 `abi.encode`
- 如果一定要 packed 动态类型，要手动加分隔符或长度，但更推荐直接用 `abi.encode`

## 审计时注意

看到下面这种代码要多看一眼：

```solidity
keccak256(abi.encodePacked(a, b));
```

如果 `a` 和 `b` 都可能是 `string`、`bytes`、数组等动态类型，就要考虑 packed 编码碰撞风险。

## 我的理解

`abi.encode` 像正规打包：每个东西的位置、长度都写清楚。

`abi.encodePacked` 像直接把内容压缩后拼起来：省空间，但很多补零、长度、边界信息都没了，所以 packed 之后通常不能反推出原来的多个输入。做 hash 时尤其要小心。
