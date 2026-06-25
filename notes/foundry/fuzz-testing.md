# Fuzz Testing

## 一句话理解

Fuzz testing 是让 Foundry 用很多随机输入反复运行同一个测试函数，检查某个性质是否一直成立。

它不是只测一个固定例子，而是把一个测试变成“小型搜索器”。

## Stateless Fuzz

Foundry 里最常见的 fuzz test 是 stateless fuzz。

意思是：每次运行测试时，Foundry 都会生成一组新的随机参数，但每一轮测试都是干净状态。

例如：

```solidity
function testFuzz_Deposit(uint256 amount) public {
    amount = bound(amount, 1, 100 ether);

    vm.deal(user, amount);
    vm.prank(user);
    vault.deposit{value: amount}();

    assertEq(vault.balanceOf(user), amount);
}
```

Foundry 会做很多轮：

```text
run 1: amount = 1
  set up clean state
  deposit(1)
  assert

run 2: amount = 50 ether
  set up clean state
  deposit(50 ether)
  assert

run 3: amount = random value
  set up clean state
  deposit(random value)
  assert
```

重点是：每一轮都会重新开始，不会把上一轮的状态带到下一轮。

## 和 Stateful Fuzz 的区别

Stateless fuzz：

```text
deposit(random amount) -> assert -> reset
deposit(random amount) -> assert -> reset
deposit(random amount) -> assert -> reset
```

Stateful fuzz：

```text
deposit -> withdraw -> deposit -> deposit -> withdraw -> invariant check
```

区别在于：

- stateless fuzz 随机的是“单次测试函数的参数”
- stateful fuzz 随机的是“目标合约的一连串函数调用”
- stateless fuzz 每轮是干净状态
- stateful fuzz 会在同一个状态上连续操作，最后再检查 invariant

## 为什么重要

手写单元测试通常只覆盖我已经想到的例子。

Fuzz test 可以帮我搜索一些没手写出来的边界情况，比如：

- 0
- 极大值
- 很小的 dust amount
- 不同余额下的调用
- fee / precision / rounding 附近的边界
- 多个参数之间的异常组合

但 Foundry 只负责生成输入，不负责理解安全性。真正重要的是：测试里要写清楚“什么性质必须成立”。

## bound

`bound(value, min, max)` 用来把随机值限制在有意义的范围内。

```solidity
amount = bound(amount, 1, 100 ether);
```

这表示 `amount` 仍然是随机的，但只会落在 `1` 到 `100 ether` 之间。

如果不限制范围，Foundry 可能生成非常极端的值，让测试变成不现实的场景。

## vm.assume

`vm.assume(condition)` 用来丢弃不满足条件的 fuzz case。

```solidity
vm.assume(amount > 0);
```

如果 `amount == 0`，这一轮输入会被跳过。

注意：不要过度使用 `vm.assume()`。如果过滤条件太多，Foundry 会浪费很多轮输入，甚至很难找到有效 case。

能用 `bound()` 表达的范围，通常优先用 `bound()`。

## 好的 Fuzz Test 应该检查什么

好的 fuzz test 不只是检查“没有 revert”，而是检查一个性质。

例如：

- deposit 后用户份额应该增加正确数量
- withdraw 不能超过用户可取余额
- mint 后总供应量和用户余额应该一致
- rounding 不应该让用户凭空多拿资产
- 权限函数不能被非授权地址调用成功

## 常见误区

- 输入完全不限制，导致测试场景不现实。
- `vm.assume()` 用太多，大量随机输入被丢弃。
- 只断言“不 revert”，但没有检查结果是否正确。
- 把普通单元测试的数字换成随机数，却没有表达更深的性质。
- 把 stateless fuzz 和 stateful fuzz 混在一起理解。

## 我的理解

Stateless fuzz 问的是：

```text
这个函数在很多不同输入下，是否都满足我写的断言？
```

Stateful fuzz 问的是：

```text
系统经过一连串随机操作后，是否仍然满足核心不变量？
```

所以 fuzz testing 的关键不是“随机”，而是“属性”。Foundry 可以帮我探索输入，但我要负责定义什么行为才算正确。
