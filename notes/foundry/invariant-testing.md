# Invariant Testing

## 1. Stateful Fuzz 的理念

Invariant testing 检查的是：系统经过一连串随机操作后，某个核心规则是否仍然成立。

这里最容易误解的是：stateful fuzz 不是随机执行 invariant test 函数里的代码。真正的流程是：

```text
setUp()
  部署合约
  准备测试数据
  设置 fuzz target

Foundry stateful fuzz
  随机调用 target contract 的 public / external 函数
  多次调用发生在同一个状态上
  状态会持续累积

invariant function
  执行收尾操作
  检查最终规则是否仍然成立
```

随机调用由 Foundry 根据 `targetContract()` 自动执行；invariant 函数只负责最后的检查逻辑。

Stateless fuzz 和 stateful fuzz 的区别：

```text
stateless fuzz:
  deposit(random amount) -> assert -> reset
  deposit(random amount) -> assert -> reset
  deposit(random amount) -> assert -> reset

stateful fuzz:
  deposit -> withdraw -> deposit -> deposit -> withdraw -> invariant check
```

Stateless fuzz 随机的是“单次测试函数的参数”，每一轮都是干净状态。Stateful fuzz 随机的是“一连串函数调用”，这些调用会共享并改变同一个合约状态。

Stateful fuzz 更适合检查这类问题：

- 用户经过多次 deposit / withdraw 后，是否还能完整取回资产。
- 协议经过 borrow / repay / liquidate 后，accounting 是否仍然正确。
- 多个合法操作组合后，系统是否会进入坏状态。

## 2. Open Stateful Fuzz

Open stateful fuzz 是最直接的写法：把真实协议合约本身交给 Foundry，让 Foundry 随机调用它的外部函数。

假设协议是一个 ERC20 vault，核心函数是：

```solidity
depositToken(token, amount)
withdrawToken(token)
```

协议声称的不变量是：

```text
Users must always be able to withdraw the exact balance amount.
```

也就是：用户存进去多少，最后应该能完整取回来。

一个简洁的 open stateful fuzz 例子：

```solidity
contract OpenInvariantTest is StdInvariant, Test {
    HandlerStatefulFuzzCatches vault;
    MockUSDC mockUSDC;
    YieldERC20 yieldERC20;
    address user = makeAddr("user");
    uint256 startingAmount = 100 ether;

    function setUp() public {
        mockUSDC = new MockUSDC();
        yieldERC20 = new YieldERC20();
        vault = new HandlerStatefulFuzzCatches();

        mockUSDC.mint(user, startingAmount);
        yieldERC20.mint(user, startingAmount);

        vm.startPrank(user);
        mockUSDC.approve(address(vault), startingAmount);
        yieldERC20.approve(address(vault), startingAmount);
        vm.stopPrank();

        targetContract(address(vault));
    }

    function statefulFuzz_testInvariantBreaks() public {
        vm.startPrank(user);
        vault.withdrawToken(mockUSDC);
        vault.withdrawToken(yieldERC20);
        vm.stopPrank();

        assertEq(mockUSDC.balanceOf(address(vault)), 0);
        assertEq(yieldERC20.balanceOf(address(vault)), 0);
        assertEq(mockUSDC.balanceOf(user), startingAmount);
        assertEq(yieldERC20.balanceOf(user), startingAmount);
    }
}
```

执行顺序是：

```text
Foundry 先随机调用 vault 的函数很多次
  depositToken(randomToken, randomAmount)
  withdrawToken(randomToken)
  depositToken(randomToken, randomAmount)
  ...

然后才执行 statefulFuzz_testInvariantBreaks()
  user 尝试取出所有支持的 token
  assert vault 没有残留
  assert user 回到初始余额
```

Invariant 函数里主动调用 `withdrawToken` 是有意义的，因为这里要检查的是“用户最终能不能取回资产”。如果 vault accounting 记录用户有 100 USDC，但 vault 实际只剩 90 USDC，只有真的执行 withdraw，才可能暴露问题。

Open stateful fuzz 的弊端是：输入太开放。

Foundry 随机调用：

```solidity
depositToken(IERC20 token, uint256 amount)
```

时，`token` 参数可能是任意随机地址。但这个 vault 实际只支持：

```text
mockUSDC
yieldERC20
```

结果大量调用都会因为 `UnsupportedToken()` revert。测试可能表面上 pass，但其实 Foundry 大部分时间都在喂无意义参数，没有真正探索核心路径。

## 3. Handler 控制的 Stateful Fuzz

Handler 的作用是把随机输入变成“有意义的随机操作”。

不是让 Foundry 直接随机调用真实协议：

```text
vault.depositToken(random address, random amount)
```

而是让 Foundry 随机调用 Handler：

```text
handler.depositMockUSDC(amount)
handler.depositYieldERC20(amount)
handler.withdrawMockUSDC()
handler.withdrawYieldERC20()
```

再由 Handler 内部调用真实协议。

一个简洁的 Handler 例子：

```solidity
contract Handler is Test {
    HandlerStatefulFuzzCatches vault;
    IERC20 mockUSDC;
    IERC20 yieldERC20;
    address user;

    constructor(
        HandlerStatefulFuzzCatches _vault,
        IERC20 _mockUSDC,
        IERC20 _yieldERC20,
        address _user
    ) {
        vault = _vault;
        mockUSDC = _mockUSDC;
        yieldERC20 = _yieldERC20;
        user = _user;
    }

    function depositMockUSDC(uint256 amount) public {
        _deposit(mockUSDC, amount);
    }

    function depositYieldERC20(uint256 amount) public {
        _deposit(yieldERC20, amount);
    }

    function withdrawMockUSDC() public {
        vm.prank(user);
        vault.withdrawToken(mockUSDC);
    }

    function withdrawYieldERC20() public {
        vm.prank(user);
        vault.withdrawToken(yieldERC20);
    }

    function _deposit(IERC20 token, uint256 amount) internal {
        amount = bound(amount, 0, token.balanceOf(user));

        vm.startPrank(user);
        token.approve(address(vault), amount);
        vault.depositToken(token, amount);
        vm.stopPrank();
    }
}
```

测试里再把 fuzz target 设置成 Handler，并限制可调用的函数 selector：

```solidity
contract HandlerInvariantTest is StdInvariant, Test {
    HandlerStatefulFuzzCatches vault;
    Handler handler;
    MockUSDC mockUSDC;
    YieldERC20 yieldERC20;
    address user = makeAddr("user");
    uint256 startingAmount = 100 ether;

    function setUp() public {
        mockUSDC = new MockUSDC();
        yieldERC20 = new YieldERC20();
        vault = new HandlerStatefulFuzzCatches();
        handler = new Handler(vault, mockUSDC, yieldERC20, user);

        mockUSDC.mint(user, startingAmount);
        yieldERC20.mint(user, startingAmount);

        bytes4[] memory selectors = new bytes4[](4);
        selectors[0] = handler.depositYieldERC20.selector;
        selectors[1] = handler.depositMockUSDC.selector;
        selectors[2] = handler.withdrawMockUSDC.selector;
        selectors[3] = handler.withdrawYieldERC20.selector;

        targetSelector(FuzzSelector({
            addr: address(handler),
            selectors: selectors
        }));
        targetContract(address(handler));
    }

    function statefulFuzz_testInvariantBreaks() public {
        vm.startPrank(user);
        vault.withdrawToken(mockUSDC);
        vault.withdrawToken(yieldERC20);
        vm.stopPrank();

        assertEq(mockUSDC.balanceOf(address(vault)), 0);
        assertEq(yieldERC20.balanceOf(address(vault)), 0);
        assertEq(mockUSDC.balanceOf(user), startingAmount);
        assertEq(yieldERC20.balanceOf(user), startingAmount);
    }
}
```

这里最关键的是同时写：

```solidity
targetSelector(FuzzSelector({
    addr: address(handler),
    selectors: selectors
}));
targetContract(address(handler));
```

它们控制的东西不一样：

- `targetContract(address(handler))`: 控制 Foundry 随机调用哪个合约。
- `targetSelector(...)`: 控制 Foundry 在这个合约里随机调用哪些函数。

只写 `targetSelector()` 不够。它只是说“对于 handler 这个地址，我希望 fuzz 这些 selector”，但不一定会把其他部署出来的合约从 fuzz target 集合里排除。

如果 `setUp()` 里部署了：

```solidity
yieldERC20 = new YieldERC20();
mockUSDC = new MockUSDC();
vault = new HandlerStatefulFuzzCatches();
handler = new Handler(...);
```

但没有明确：

```solidity
targetContract(address(handler));
```

Foundry 可能会随机调用测试中部署出来的其他合约，例如：

```text
MockUSDC.transferFrom(address,address,uint256)
YieldERC20.approve(address,uint256)
YieldERC20.transfer(address,uint256)
```

这不是我想测的路径。`transferFrom` 需要 allowance，但随机 sender、随机 from、随机 to、随机 amount 基本一定会失败，于是测试会因为 ERC20 的无意义随机调用 revert。

最终想要的路径是：

```text
Foundry
  -> 只能调用指定 selectors
Handler
  -> 把输入限制到 mockUSDC / yieldERC20 和合理 amount
Vault
  -> 执行真实协议逻辑
Invariant function
  -> user 尝试取回全部资产
  -> assert 最终状态
```

一句话总结：

```text
targetContract 控制“调用哪个合约”。
targetSelector 控制“调用这个合约里的哪些函数”。
Handler 控制“随机输入怎样变成有效业务操作”。
```
