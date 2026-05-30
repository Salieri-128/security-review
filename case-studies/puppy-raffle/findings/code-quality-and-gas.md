### [I-1] Code Quality and Gas Observations

**Description:** The contract contains several informational and gas-related issues that do not directly create a critical exploit path, but make the protocol harder to maintain, more expensive to use, or easier to misconfigure.

**Observations:**

1. Floating pragma

```javascript
pragma solidity ^0.7.6;
```

The floating pragma allows compilation with any compatible `0.7.x` compiler version. Different patch versions can have different compiler behavior or bug fixes.

Recommended mitigation:

```diff
- pragma solidity ^0.7.6;
+ pragma solidity 0.7.6;
```

2. `feeAddress` is not checked against the zero address

```javascript
constructor(uint256 _entranceFee, address _feeAddress, uint256 _raffleDuration) ERC721("Puppy Raffle", "PR") {
    entranceFee = _entranceFee;
    feeAddress = _feeAddress;
    raffleDuration = _raffleDuration;
    raffleStartTime = block.timestamp;
}

function changeFeeAddress(address newFeeAddress) external onlyOwner {
    feeAddress = newFeeAddress;
    emit FeeAddressChanged(newFeeAddress);
}
```

If `feeAddress` is set to `address(0)`, protocol fees can be sent to the zero address.

Recommended mitigation:

```diff
constructor(uint256 _entranceFee, address _feeAddress, uint256 _raffleDuration) ERC721("Puppy Raffle", "PR") {
+   require(_feeAddress != address(0), "PuppyRaffle: Zero fee address");
    entranceFee = _entranceFee;
    feeAddress = _feeAddress;
    raffleDuration = _raffleDuration;
    raffleStartTime = block.timestamp;
}

function changeFeeAddress(address newFeeAddress) external onlyOwner {
+   require(newFeeAddress != address(0), "PuppyRaffle: Zero fee address");
    feeAddress = newFeeAddress;
    emit FeeAddressChanged(newFeeAddress);
}
```

3. Unchanged state variables can be immutable or constant

`raffleDuration` is set only in the constructor and never changed. The puppy image URIs are also initialized once and then only copied into mappings.

```javascript
uint256 public raffleDuration;
string private commonImageUri = "ipfs://...";
string private rareImageUri = "ipfs://...";
string private legendaryImageUri = "ipfs://...";
```

Recommended mitigation: make `raffleDuration` immutable and convert fixed string values to constants where possible.

4. Magic numbers should be named constants

```javascript
uint256 prizePool = (totalAmountCollected * 80) / 100;
uint256 fee = (totalAmountCollected * 20) / 100;
```

The `80`, `20`, and `100` values encode payout logic but are not named. Constants make the logic easier to audit and update.

Recommended mitigation:

```diff
+ uint256 private constant PRIZE_POOL_PERCENTAGE = 80;
+ uint256 private constant FEE_PERCENTAGE = 20;
+ uint256 private constant PERCENTAGE_DENOMINATOR = 100;

- uint256 prizePool = (totalAmountCollected * 80) / 100;
- uint256 fee = (totalAmountCollected * 20) / 100;
+ uint256 prizePool = (totalAmountCollected * PRIZE_POOL_PERCENTAGE) / PERCENTAGE_DENOMINATOR;
+ uint256 fee = (totalAmountCollected * FEE_PERCENTAGE) / PERCENTAGE_DENOMINATOR;
```

5. Storage reads in loops can be cached

Functions that loop over `players` read `players.length` from storage on every iteration.

```javascript
for (uint256 i = 0; i < players.length; i++) {
    if (players[i] == msg.sender) {
        return true;
    }
}
```

Recommended mitigation:

```diff
+ uint256 playersLength = players.length;
- for (uint256 i = 0; i < players.length; i++) {
+ for (uint256 i = 0; i < playersLength; i++) {
```

6. Dead code can be removed

`_isActivePlayer` is defined but not used anywhere in the contract.

```javascript
function _isActivePlayer() internal view returns (bool) {
    for (uint256 i = 0; i < players.length; i++) {
        if (players[i] == msg.sender) {
            return true;
        }
    }
    return false;
}
```

Recommended mitigation: remove unused code or use it where active-player checks are needed.

7. Event parameters can be indexed

```javascript
event RaffleRefunded(address player);
event FeeAddressChanged(address newFeeAddress);
```

Indexing important addresses makes off-chain monitoring and filtering easier.

Recommended mitigation:

```diff
- event RaffleRefunded(address player);
- event FeeAddressChanged(address newFeeAddress);
+ event RaffleRefunded(address indexed player);
+ event FeeAddressChanged(address indexed newFeeAddress);
```

**Impact:** These issues increase maintenance cost, reduce off-chain observability, and can make the protocol more expensive or easier to misconfigure.

**Recommended Mitigation:** Apply the targeted mitigations above, then rerun tests, static analysis, and coverage checks to confirm behavior remains unchanged.
