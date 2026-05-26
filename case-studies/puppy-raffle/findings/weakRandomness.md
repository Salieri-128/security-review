### [H-3] Weak randomness in `PuppyRaffle::selectWinner` allows attackers to influence the winner and NFT rarity

**Description:** `PuppyRaffle::selectWinner` uses on-chain values such as `msg.sender`, `block.timestamp`, and `block.difficulty` to generate randomness for choosing the raffle winner and determining the puppy NFT rarity.

```javascript
uint256 winnerIndex =
    uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
address winner = players[winnerIndex];

// We use a different RNG calculate from the winnerIndex to determine rarity
uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100;
```

These values are not secure sources of randomness. The EVM is deterministic, so any value derived only from chain data can be predicted or influenced. A caller can choose when to call `selectWinner`, use different caller addresses, or simulate outcomes off-chain before sending a transaction.

Since the NFT rarity is also generated from similar on-chain values, an attacker can try to influence not only who wins, but also which rarity NFT is minted.

**Impact:** A malicious user or validator can gain an unfair advantage in the raffle by influencing or predicting the winner selection. This breaks the fairness of the raffle and can cause honest players to lose funds or rewards.

If rare NFTs have higher value, weak randomness can also let attackers target higher-value outcomes.

**Proof of Concept:**

The randomness is computed from values available during transaction execution:

- `msg.sender` can be controlled by the caller
- `block.timestamp` can be influenced by block producers within allowed limits
- `block.difficulty` / `block.prevrandao` is not intended as secure application randomness

An attacker can simulate the winner calculation and only call `selectWinner` when the result is favorable.

```js
function testWeakRandomnessCanBePredicted() public {
    address attacker = makeAddr("attacker");
    uint256 playersLength = 4;

    uint256 predictedWinnerIndex = uint256(
        keccak256(abi.encodePacked(attacker, block.timestamp, block.difficulty))
    ) % playersLength;

    uint256 predictedRarity = uint256(
        keccak256(abi.encodePacked(attacker, block.difficulty))
    ) % 100;

    // If predictedWinnerIndex or predictedRarity is favorable, attacker calls selectWinner.
    // If not, attacker can wait, use another caller address, or try another block.
}
```

The Meebits case study shows the same class of issue in practice: an attacker repeatedly attempted mints and reverted unfavorable results until receiving a rare NFT.

**Recommended Mitigation:**

Do not generate randomness from only on-chain values such as `block.timestamp`, `block.prevrandao`, `block.difficulty`, `blockhash`, or `msg.sender`.

Use a verifiable off-chain randomness source such as Chainlink VRF. The raffle should request randomness, wait for the VRF callback, and only then select the winner and NFT rarity from the verified random value.

Alternatively, consider a well-designed commit-reveal scheme, but make sure it handles non-reveal behavior, deadlines, and last-revealer advantage.
