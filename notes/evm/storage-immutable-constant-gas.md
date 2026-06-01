# Solidity Data Locations and Gas

## 一句话理解

Solidity 里的数据不都存在一个地方。

不同数据位置决定了它能不能修改、能活多久、以及读写 gas 成本。

重点记：

- `storage`: 永久状态，读写贵
- `memory`: 函数执行中的临时数据，可修改
- `calldata`: 外部调用传入的数据，只读，便宜
- `immutable`: 部署时确定，写进 runtime bytecode，读便宜
- `constant`: 编译时确定，直接内联进 bytecode，读便宜

## Storage

普通 state variable 存在 storage slot 里。

```solidity
address public s_owner;
```

特点：

- 永久保存在链上
- 可以修改
- 每个 slot 是 32 bytes
- 读取用 `SLOAD`
- 写入用 `SSTORE`
- gas 贵

大概记忆：

- cold `SLOAD`: 约 2100 gas
- warm `SLOAD`: 约 100 gas
- `SSTORE`: 更贵，可能几千到两万 gas

适合保存真正的合约状态，比如 balances、owner、config、mapping。

## Memory

`memory` 是函数执行期间的临时区域。

```solidity
function f(string memory name) public pure {}
```

特点：

- 函数执行结束后消失
- 可以修改
- 比 storage 便宜
- 常用于临时数组、字符串、struct

适合函数内部临时计算。

## Calldata

`calldata` 是外部调用传进来的原始参数数据。

```solidity
function f(string calldata name) external {}
```

这里的意思是：`name` 不会先被复制到 `memory`，函数会直接从 calldata 区域读取传入参数。

特点：

- 只读
- 不能修改
- 不会复制到 memory 时更省 gas
- 常用于 external 函数的动态类型参数

如果只是读取外部传入的数组、字符串、bytes，优先考虑 `calldata`。

例如：

```solidity
function enterRaffle(address[] calldata players) external {
    // read players directly from calldata
}
```

如果函数里需要修改这个数组，就不能直接改 `calldata`，需要复制到 `memory`。

## Immutable

`immutable` 在 constructor 里赋值一次，部署后不能改。

```solidity
address public immutable i_owner;

constructor() {
    i_owner = msg.sender;
}
```

它不存普通 storage slot。

constructor 结束后，Solidity 会把值嵌入 runtime bytecode。

读取时更像：

```text
PUSH20 <owner address>
```

而不是：

```text
SLOAD(slot)
```

所以 `immutable` 读取通常比 storage 便宜。

适合：

- 部署时确定
- 之后永远不改
- 经常被读取

例如 owner、token、router、price feed、entrance fee。

不适合需要后续修改的变量，比如可转移 owner、可调整 fee、可升级 router。

## Constant

`constant` 必须在编译时就能确定。

```solidity
uint256 public constant FEE = 100;
```

特点：

- 编译时确定
- 不能依赖 `msg.sender`、constructor 参数等部署时数据
- 编译器直接内联进 bytecode
- 读取便宜

可以：

```solidity
uint256 public constant MINIMUM_USD = 5e18;
```

不可以：

```solidity
address public constant OWNER = msg.sender;
```

因为 `msg.sender` 只有部署时才知道，这种应该用 `immutable`。

## Code

`code` 是合约部署后的 runtime bytecode。

特点：

- 部署后不可变
- 用户调用合约时执行的是 runtime code
- `immutable` 和 `constant` 的值最终都和 code 关系很近

部署时其实有两段代码：

- creation code: 部署时执行，跑 constructor
- runtime code: constructor 后返回，最终留在链上

普通 storage variable 是 constructor 里 `SSTORE` 到 storage。

`immutable` 是 constructor 得到值后，把 runtime code 里的占位符替换成真实值。

## Logs

`event` 写入的是 transaction logs。

```solidity
emit Transfer(from, to, amount);
```

logs 给链下系统读取，比如前端、Etherscan、indexer。

特点：

- 不是 storage
- 合约内部一般不能读取历史 event
- 适合记录链下需要追踪的状态变化

## Stack

EVM 执行时还会用 stack。

特点：

- 很临时
- opcode 执行时使用
- Solidity 开发中通常不用直接管理

知道它存在即可，日常审计更常关注 storage、memory、calldata。

## 对比表

| 类型 | 生命周期 | 是否可改 | 位置 | gas 直觉 |
| --- | --- | --- | --- | --- |
| `storage` | 永久 | 可以 | storage slot | 贵 |
| `memory` | 函数执行期间 | 可以 | memory | 中等/便宜 |
| `calldata` | 本次外部调用 | 只读 | calldata | 便宜 |
| `immutable` | 部署后固定 | 不可以 | runtime bytecode | 读便宜 |
| `constant` | 编译后固定 | 不可以 | bytecode 内联 | 读便宜 |
| `logs` | 交易日志 | 不给合约改 | receipt/logs | 给链下看 |

## storage vs immutable vs constant

最常见 gas 对比：

```solidity
address public s_owner;
address public immutable i_owner;
uint256 public constant FEE = 100;
```

`s_owner`:

```text
SLOAD(slot) -> expensive
```

`i_owner`:

```text
embedded value in runtime bytecode -> cheaper
```

`FEE`:

```text
compile-time value -> directly inlined
```

所以如果一个 state variable 只在 constructor 赋值，之后不再修改，可以建议改成 `immutable`。

这通常是 gas optimization，不是安全漏洞。

## 审计时注意

- state variable 默认是 storage，读写贵
- external 函数的只读动态参数可考虑 `calldata`
- 只在 constructor 赋值且不再修改的变量可考虑 `immutable`
- 编译期固定值用 `constant`
- 需要后续修改的变量不能用 `immutable`
- 不要把 event logs 当成合约状态
- proxy / upgradeable contract 特别关注 storage layout

## 我的理解

`storage` 是合约的状态数据库，永久但贵。

`memory` 是函数里的草稿纸，用完就丢。

`calldata` 是外部调用带来的只读输入。

`immutable` 是部署时写进合约代码里的值。

`constant` 是编译时就写死的值。

gas 差异的本质不是语法，而是数据放在 EVM 的不同区域，读取方式不同。
