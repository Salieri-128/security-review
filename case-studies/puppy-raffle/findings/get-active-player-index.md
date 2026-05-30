### [L-1] `getActivePlayerIndex` Returns `0` for Both a Missing Player and the First Player

**Description:** `PuppyRaffle::getActivePlayerIndex` returns the player's index if found, and returns `0` if the player is not found.

```javascript
function getActivePlayerIndex(address player) external view returns (uint256) {
    for (uint256 i = 0; i < players.length; i++) {
        if (players[i] == player) {
            return i;
        }
    }
    return 0;
}
```

Since array indexes start at `0`, the function returns the same value for two different cases:

- the player is active at index `0`
- the player is not active

**Impact:** Frontends, scripts, or integrators cannot reliably distinguish between "player not found" and "player found at index 0". This can lead to confusing user experience and incorrect follow-up calls to `refund`.

The core `refund` authorization still checks `players[playerIndex] == msg.sender`, so this is a correctness and integration issue rather than a direct fund-draining vulnerability.

**Proof of Concept:**

```js
function testGetActivePlayerIndexAmbiguous() public {
    address[] memory players = new address[](1);
    players[0] = playerOne;
    puppyRaffle.enterRaffle{value: entranceFee}(players);

    assertEq(puppyRaffle.getActivePlayerIndex(playerOne), 0);
    assertEq(puppyRaffle.getActivePlayerIndex(address(999)), 0);
}
```

**Recommended Mitigation:** Return both a boolean and the index, or revert when the player is not active.

```diff
- function getActivePlayerIndex(address player) external view returns (uint256) {
+ function getActivePlayerIndex(address player) external view returns (bool, uint256) {
    for (uint256 i = 0; i < players.length; i++) {
        if (players[i] == player) {
-           return i;
+           return (true, i);
        }
    }
-   return 0;
+   return (false, 0);
}
```
