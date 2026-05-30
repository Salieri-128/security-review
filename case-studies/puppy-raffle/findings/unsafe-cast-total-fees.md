### [H-4] Unsafe Cast of `fee` to `uint64` Can Corrupt Protocol Fee Accounting

**Description:** `PuppyRaffle::selectWinner` calculates `fee` as a `uint256`, then casts it down to `uint64` before adding it to `totalFees`.

```javascript
uint256 fee = (totalAmountCollected * 20) / 100;
totalFees = totalFees + uint64(fee);
```

If `fee` is greater than `type(uint64).max`, the explicit cast truncates the value instead of preserving the full fee amount. Solidity 0.7.6 also does not include built-in overflow checks, so `totalFees = totalFees + uint64(fee)` can wrap if the accumulated fees exceed the `uint64` range.

**Impact:** The protocol can record an incorrect fee amount. This can cause the fee recipient to withdraw less than the protocol actually earned, leave ETH stuck in the contract, and break fee accounting assumptions used by `withdrawFees`.

Because this accounting protects protocol revenue and is tied to ETH withdrawals, an incorrect cast can create direct financial loss for the protocol.

**Proof of Concept:**

The maximum value of `uint64` is:

```text
18,446,744,073,709,551,615
```

With an `entranceFee` of `1e18`, a sufficiently large number of players can make the 20% fee exceed that value.

```js
uint256 totalAmountCollected = players.length * entranceFee;
uint256 fee = (totalAmountCollected * 20) / 100;

// If fee > type(uint64).max, this stores fee % 2^64 instead of fee.
uint64 storedFee = uint64(fee);
```

For example, a fee of `20e18` is larger than `type(uint64).max`, so `uint64(20e18)` will not equal `20e18`.

**Recommended Mitigation:** Use `uint256` for `totalFees` so fee accounting uses the same width as ETH amounts.

```diff
- uint64 public totalFees = 0;
+ uint256 public totalFees = 0;
```

If storage packing is required, check the range before casting.

```diff
uint256 fee = (totalAmountCollected * 20) / 100;
+ require(fee <= type(uint64).max, "PuppyRaffle: Fee too large");
totalFees = totalFees + uint64(fee);
```

For Solidity versions before 0.8, also use SafeMath or upgrade the compiler so arithmetic overflow checks are enabled by default.
