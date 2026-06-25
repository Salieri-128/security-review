# Foundry Test: Address, Balance, Sender

## 一句话理解

Foundry 测试里要分清三件事：谁是地址、谁有钱、谁在调用函数。

## makeAddr

`makeAddr("name")` 用来生成一个固定的测试地址。

```solidity
address attacker = makeAddr("attacker");
```

它只是一个地址，不是合约，也不会自动有 ETH。Foundry 会给它加 label，所以 trace 里更容易看懂。

## vm.deal

`vm.deal(address, amount)` 用来设置某个地址的 ETH 余额。

```solidity
vm.deal(attacker, 2 ether);
```

注意：给谁 `deal`，谁才有钱。给 `attackerAddress` 钱，不代表攻击合约地址也有钱。

## vm.prank

`vm.prank(address)` 只影响下一次外部调用的 `msg.sender`。

```solidity
vm.prank(attackerAddress);
attacker.attack{value: 1 ether}();
```

在 `attack()` 里面：

```solidity
msg.sender == attackerAddress;
```

下一次调用结束后，`msg.sender` 就会恢复成原来的调用者。

## vm.startPrank / vm.stopPrank

如果要连续伪装多次调用，用 `vm.startPrank(address)` 和 `vm.stopPrank()`。

`vm.startPrank(address)` 会让后续所有外部调用的 `msg.sender` 都变成这个地址，直到执行 `vm.stopPrank()`。

```solidity
vm.startPrank(attackerAddress);

target.deposit(1 ether);
target.withdraw(0.5 ether);
target.claimReward();

vm.stopPrank();
```

在这三次调用里：

```solidity
msg.sender == attackerAddress;
```

这在需要同一个 user 连续执行多步操作时很常见，例如先 `approve`，再 `deposit`，最后 `withdraw`。

```solidity
vm.startPrank(user);
token.approve(address(vault), amount);
vault.depositToken(token, amount);
vault.withdrawToken(token);
vm.stopPrank();
```

注意：写了 `vm.startPrank()` 之后要记得 `vm.stopPrank()`，否则后面的外部调用还会继续用伪装的 `msg.sender`。

## vm.expectRevert

`vm.expectRevert()` 用来测试下一次调用应该 revert。

```solidity
vm.expectRevert();
puppyRaffle.refund(0);
```

它要写在会 revert 的调用前面，并且只影响下一次调用。

如果要检查具体错误信息：

```solidity
vm.expectRevert("PuppyRaffle: Only the player can refund");
puppyRaffle.refund(0);
```

如果是 custom error，通常检查 selector：

```solidity
vm.expectRevert(MyContract.NotOwner.selector);
myContract.onlyOwnerFunction();
```

## 测试合约本身也有余额

测试合约通常长这样：

```solidity
contract PuppyRaffleTest is Test {}
```

在 Foundry 测试环境里，继承 `Test` 的测试合约默认有足够 ETH。所以测试合约可以直接发起带 `value` 的调用：

```solidity
puppyRaffle.enterRaffle{value: entranceFee * 4}(players);
```

这里的钱来自测试合约，也就是 `address(this)`。

## 资金来源

谁发起这次调用，钱就从谁的余额里扣。

```solidity
puppyRaffle.enterRaffle{value: 4 ether}(players);
```

如果这行代码由测试合约直接调用，资金流是：

```text
Test contract -4 ETH
PuppyRaffle  +4 ETH
```

如果是外部用户调用攻击合约：

```solidity
vm.deal(attackerAddress, 2 ether);
vm.prank(attackerAddress);
attacker.attack{value: 1 ether}();
```

资金流是：

```text
attackerAddress          -1 ETH
Attack contract address  +1 ETH
```

如果攻击合约内部再调用目标合约并附带 `value`，钱就从攻击合约余额里扣。

## 速记

- `makeAddr`: 造一个测试地址
- `vm.deal`: 给某个地址设置 ETH 余额
- `vm.prank`: 修改下一次调用的 `msg.sender`
- `vm.startPrank`: 开始持续修改后续外部调用的 `msg.sender`
- `vm.stopPrank`: 停止持续 prank，恢复正常调用者
- `vm.expectRevert`: 期待下一次调用 revert
- `address(this)`: 当前测试合约地址
- `{value: ...}`: 谁调用，钱从谁那里扣

## 我的理解

测试里最容易混的是“地址”和“合约余额”。`makeAddr` 只是造地址，`deal` 才是给余额，`prank` / `startPrank` 只是换 `msg.sender`。

`expectRevert` 是提前告诉 Foundry：下一次调用 revert 才是正确结果。

当合约内部继续带 `value` 调用别的合约时，必须保证当前这个合约地址本身有钱。
