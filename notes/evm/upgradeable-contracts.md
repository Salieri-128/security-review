# Upgradeable Contracts

## 一句话理解

普通智能合约部署后代码不可修改。

Upgradeable contract 的核心做法是：用户一直和 proxy 合约交互，状态存在 proxy 里，逻辑放在 implementation 合约里。升级时部署新的 implementation，再让 proxy 指向新的 implementation。

## Proxy Pattern

可升级合约通常拆成两部分：

- Proxy: 用户交互的地址，保存 storage 和 ETH/token 状态
- Implementation: 逻辑合约，保存业务代码

用户调用 proxy，proxy 在 `fallback` 中把调用转发给 implementation。

升级时：

1. 部署新的 implementation
2. 更新 proxy 里保存的 implementation 地址
3. 用户继续调用同一个 proxy 地址
4. proxy 的状态保留，但执行的是新逻辑

## delegatecall 是关键

proxy 不是普通 `call` implementation，而是用 `delegatecall`。

`delegatecall` 的特点：

- 执行的是 implementation 的代码
- 读写的是 proxy 的 storage
- `msg.sender` 保持为原始调用者
- `msg.value` 保持为原始调用附带的 ETH

简单理解：

```text
User -> Proxy
        Proxy delegatecall Implementation
        Implementation code runs on Proxy storage
```

所以 implementation 看起来像在操作自己的变量，实际上改的是 proxy 对应 slot 里的数据。

## 为什么 storage layout 很重要

因为 implementation 的代码会按自己的变量顺序读写 storage slot。

如果升级前后 storage layout 不一致，就可能读错或覆盖 proxy 里的旧数据。

危险升级：

```solidity
// V1
uint256 balance; // slot 0
address owner;   // slot 1

// V2 bad
address owner;   // slot 0, wrong
uint256 balance; // slot 1, wrong
```

V2 会把原来的 `balance` 当成 `owner`，把原来的 `owner` 当成 `balance`，状态直接错乱。

更安全的升级方式：

```solidity
// V2 better
uint256 balance; // slot 0
address owner;   // slot 1
uint256 fee;     // slot 2, append new variable at the end
```

## EIP-1967 Slot

proxy 自己也需要保存 implementation 地址。

如果把它存在普通 slot，比如 slot `0`，就很容易和 implementation 的状态变量冲突。

所以常见 proxy 使用 EIP-1967 规定的特殊 slot 保存 implementation 地址：

```text
bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1)
```

常见值：

```text
0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
```

这个 slot 足够特殊，正常 Solidity 状态变量不会自动占到这里，从而减少 storage collision。

## initializer

implementation 合约通常不能依赖 constructor 初始化 proxy 的状态。

constructor 只会初始化 implementation 自己的 storage，不会初始化 proxy 的 storage。

所以可升级合约通常使用 `initialize()`：

```solidity
function initialize(address owner_) external initializer {
    owner = owner_;
}
```

注意：

- `initialize` 只能调用一次
- implementation 本身也要防止被别人初始化
- 忘记初始化可能导致 owner 为空或被抢占

## 常见风险

- upgrade 函数没有 access control
- storage layout 升级不兼容
- proxy slot 和 implementation slot 冲突
- initializer 可以被重复调用
- implementation 未初始化，被攻击者接管
- 新 implementation 引入恶意逻辑
- `delegatecall` 到不可信地址
- proxy fallback 转发逻辑错误

## 审计时注意

- 用户实际交互的是 proxy 还是 implementation
- implementation 地址存在哪个 slot
- 谁可以升级 implementation
- 升级权限是否有 timelock、multisig 或 governance
- V1 到 V2 是否只在末尾新增变量
- 是否改了变量顺序、类型、继承顺序
- initializer 是否只能执行一次
- 是否使用 OpenZeppelin 等成熟 proxy 实现

## 我的理解

可升级合约不是“修改了原合约代码”，而是让 proxy 换一个新的逻辑合约。

proxy 像一个固定入口和状态仓库，implementation 像可替换的大脑。`delegatecall` 让新的大脑操作旧的身体，所以 storage layout 必须对齐，升级权限也必须非常严格。
