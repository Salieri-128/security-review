### [M-2] Smart Contract Winner Can Block Winner Selection by Reverting ETH Transfers

**Description:** `PuppyRaffle::selectWinner` sends the prize pool directly to the winner with a low-level call and reverts if the transfer fails.

```javascript
(bool success,) = winner.call{value: prizePool}("");
require(success, "PuppyRaffle: Failed to send prize pool to winner");
_safeMint(winner, tokenId);
```

If the selected winner is a smart contract that cannot receive ETH, or intentionally reverts in `receive` or `fallback`, the entire `selectWinner` transaction reverts. Since winner selection depends on the current `players` array and the random index, this can prevent that raffle round from completing whenever the reverting contract is selected.

**Impact:** A malicious entrant can use a contract that reverts on ETH receipt to block winner selection when it is chosen as the winner. This can keep participant funds locked in the raffle and prevent the protocol from starting the next round.

Even an innocent smart contract wallet without an ETH-receiving function can cause the same failure.

**Proof of Concept:**

```js
contract RevertingWinner {
    function enter(PuppyRaffle puppyRaffle) external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: msg.value}(players);
    }

    receive() external payable {
        revert("cannot receive ETH");
    }
}
```

When `RevertingWinner` is selected, `winner.call{value: prizePool}("")` returns `false`, and `selectWinner` reverts.

**Recommended Mitigation:** Use a pull-payment pattern. Instead of sending ETH inside `selectWinner`, record the amount owed to the winner and let the winner claim it in a separate function.

```diff
+ mapping(address => uint256) public winnings;

function selectWinner() external {
    ...
    previousWinner = winner;
-   (bool success,) = winner.call{value: prizePool}("");
-   require(success, "PuppyRaffle: Failed to send prize pool to winner");
+   winnings[winner] += prizePool;
    _safeMint(winner, tokenId);
}

+ function claimWinnings() external {
+     uint256 amount = winnings[msg.sender];
+     require(amount > 0, "PuppyRaffle: No winnings");
+     winnings[msg.sender] = 0;
+     (bool success,) = msg.sender.call{value: amount}("");
+     require(success, "PuppyRaffle: Failed to claim winnings");
+ }
```

This prevents a single recipient from blocking the protocol's core state transition.
