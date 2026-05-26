# abi.encode vs abi.encodePacked

## 一句话理解

`abi.encode` 是标准 ABI 编码，信息完整、可以安全解码。

`abi.encodePacked` 是紧凑编码，结果更短，但会丢掉一些边界信息，多个动态类型一起用时可能产生碰撞。

## abi.encode

`abi.encode(...)` 会按照 Solidity ABI 规范编码数据。

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

特点：

- 编码结果更短
- 不按标准 ABI 的完整格式保存所有边界信息
- 通常不能直接用 `abi.decode` 解码
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

`abi.encodePacked` 像直接把字符串拼起来：省空间，但边界可能丢失。做 hash 时尤其要小心。
