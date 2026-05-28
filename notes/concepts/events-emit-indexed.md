# Events, Emit, Indexed

## 一句话理解

`event` 是定义日志格式，`emit` 是真正写出一条日志。

事件主要给链下系统看，比如前端、Etherscan、indexer、The Graph。

## event

`event` 只是声明一种事件类型，不会执行任何逻辑。

```solidity
event Transfer(address indexed from, address indexed to, uint256 amount);
```

可以理解成定义一个日志模板：

```text
Transfer(from, to, amount)
```

## emit

`emit` 才是真正把事件写进交易 logs。

```solidity
emit Transfer(msg.sender, to, amount);
```

这不会修改 storage，只是留下链下可查询的日志。

## event 不等于 storage

事件存在 transaction logs 里，不存在合约 storage 里。

合约一般不能读取历史 event。

简单区分：

- `storage`: 合约自己保存和读取状态
- `event logs`: 链下系统查询、监听、展示

## indexed

`indexed` 让字段更容易被链下过滤。

```solidity
event Transfer(address indexed from, address indexed to, uint256 amount);
```

这样前端或 indexer 可以方便地查：

- from 是某个地址的 Transfer
- to 是某个地址的 Transfer

一般一个 event 最多有 3 个 `indexed` 参数。

## 审计时注意

- 关键状态变化最好 emit event，比如 owner、fee、关键参数修改
- event 参数要准确，不能 from/to 写反
- 不要把 event 当成链上权限或状态依据
- 链上逻辑应该看 storage，不应该依赖 logs

## 我的理解

`event` 是“定义会记录什么类型的事情”。

`emit` 是“这件事真的发生了，记一笔日志”。

`indexed` 是“让链下更容易按这个字段搜索”。
