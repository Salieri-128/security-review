# Foundry 常用 Assertion

Foundry 的标准断言函数由 `forge-std/Test.sol` 提供，用于在测试中验证实际结果是否符合预期；断言失败时，当前测试会失败。

| 断言 | 含义 | 典型用途 |
| --- | --- | --- |
| `assertEq(a, b)` | `a == b` | 比较余额、状态变量、返回值等是否相等。 |
| `assertNotEq(a, b)` | `a != b` | 验证两个值不应相同。 |
| `assertGt(a, b)` | `a > b` | 验证值严格大于下界。 |
| `assertGe(a, b)` | `a >= b` | 验证值不小于下界。 |
| `assertLt(a, b)` | `a < b` | 验证值严格小于上界。 |
| `assertLe(a, b)` | `a <= b` | 验证值不大于上界。 |
| `assertTrue(x)` | `x == true` | 验证任意布尔条件成立。 |
| `assertFalse(x)` | `x == false` | 验证任意布尔条件不成立。 |

```solidity
function testDeposit() public {
    uint256 amount = 1 ether;

    vm.prank(alice);
    vault.deposit{value: amount}();

    assertEq(vault.balanceOf(alice), amount);
    assertGe(address(vault).balance, amount);
    assertTrue(vault.balanceOf(alice) > 0);
}
```

使用时优先选择最能表达意图的断言：

- 检查精确结果用 `assertEq`。
- 检查边界或单调性用 `assertGt`、`assertGe`、`assertLt`、`assertLe`。
- 检查复合条件或权限结果用 `assertTrue`、`assertFalse`。
- 对涉及 rounding、手续费或价格精度的场景，通常不应机械使用 `assertEq`，而应断言允许的上下界。
