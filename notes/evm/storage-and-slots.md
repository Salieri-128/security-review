# EVM Storage and Slots

## 一句话理解

Storage 是合约的永久存储区，可以把它想成一个巨大的 key-value 表。

每个 storage slot 是 32 bytes，slot 编号从 `0` 开始。合约的状态变量会按照声明顺序放进这些 slot 里。

## 基础规则

```solidity
contract Example {
    uint256 favoriteNumber; // slot 0
    bool active;            // slot 1, if not packed with another small value
}
```

`uint256` 正好是 32 bytes，所以会占满一个 slot。

小于 32 bytes 的类型可能被 Solidity 打包到同一个 slot 里，比如 `uint128`、`uint64`、`bool`、`address` 等。

```solidity
contract Packed {
    uint128 a; // slot 0
    uint128 b; // slot 0
    uint256 c; // slot 1
}
```

## constant 和 immutable

`constant` 和 `immutable` 通常不占用普通 storage slot。

它们的值会被放进合约 bytecode 或在部署时写入代码相关的位置，而不是像普通状态变量一样存在 slot 里。

```solidity
uint256 public constant X = 123;
```

## 动态数组

动态数组本身会占一个 slot，里面存的是数组长度。

数组元素不直接从这个 slot 后面开始存，而是从：

```text
keccak256(slot)
```

这个位置开始连续存储。

```solidity
contract Example {
    uint256[] nums; // slot 0 stores nums.length
}
```

如果 `nums` 在 slot `0`：

```text
nums.length -> slot 0
nums[0]     -> keccak256(0)
nums[1]     -> keccak256(0) + 1
nums[2]     -> keccak256(0) + 2
```

## mapping

`mapping` 本身也会占一个 slot，但这个 slot 不存长度，也不能遍历 key。

它的作用更像一个命名空间，用来参与计算每个 key 对应 value 的位置。

```solidity
contract Example {
    mapping(address => uint256) balances; // slot 0
}
```

如果 `balances` 在 slot `0`，某个 key 的 value 存在：

```text
keccak256(abi.encode(key, slot))
```

例如：

```text
balances[user] -> keccak256(abi.encode(user, uint256(0)))
```

这也是为什么 mapping 不能直接知道有哪些 key：链上只存每个 key hash 后的位置，不维护 key 列表。

## nested mapping

嵌套 mapping 是一层一层 hash。

```solidity
mapping(address => mapping(uint256 => bool)) approved; // slot 0
```

位置可以理解为：

```text
innerSlot = keccak256(abi.encode(user, uint256(0)))
approved[user][id] = keccak256(abi.encode(id, innerSlot))
```

## string 和 bytes

`string` 和 `bytes` 是动态类型，但它们有特殊存储规则。

如果内容很短，长度不超过 31 bytes，数据和长度会放在同一个 slot 里：

- 数据放在高位 bytes
- 最低位 byte 存 `length * 2`

如果内容长度是 32 bytes 或更长，slot 里存：

```text
length * 2 + 1
```

真实数据从：

```text
keccak256(slot)
```

开始存储，和动态数组类似。

简单理解：

- 短 `string` / `bytes`: 数据直接塞进自己的 slot
- 长 `string` / `bytes`: slot 存长度标记，真实数据存在 `keccak256(slot)` 开始的位置

## storage, memory, calldata

状态变量存在 `storage`，会永久保存在链上。

函数里的临时变量通常在 `memory` 或 stack 中，函数执行结束后就没了。

动态类型作为函数参数或返回值时，经常需要显式写 `memory` 或 `calldata`，比如：

```solidity
function setName(string memory name) external {
    // name is temporary memory data
}
```

## 审计时注意

- `private` 状态变量仍然可以通过 storage slot 被读到
- 合约升级时，storage layout 不能随便改变
- proxy 合约尤其要关注 slot 冲突
- mapping 的 key 不可枚举，想遍历必须额外维护数组
- 动态数组长度存在主 slot，元素位置从 `keccak256(slot)` 开始
- 短 string/bytes 和长 string/bytes 的存储方式不同

## 我的理解

普通状态变量像按顺序放进编号柜子里，每个柜子 32 bytes。

动态数组和 mapping 不能简单顺序存，因为大小不确定，所以 Solidity 用 `keccak256` 给它们的数据找新的存储位置。

理解 slot 的意义很重要：它能解释为什么 `private` 不是真的隐藏，也能解释 proxy 升级为什么不能乱改变量顺序。
