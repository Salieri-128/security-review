# Puppy Raffle Security Review

Puppy Raffle is the second full audit practice case from Cyfrin Updraft's Smart Contract Security course. It is a raffle protocol where users pay an entrance fee, join a player list, may request refunds before the raffle ends, and a later `selectWinner` call chooses a winner, pays the prize pool, mints an NFT, and accrues protocol fees.

This case study records the main security findings, proof-of-concept notes, and code quality observations I learned while auditing the protocol.

## Scope

- Repository: `cyfrin/4-puppy-raffle-audit`
- Primary contract: `src/PuppyRaffle.sol`
- Solidity version: `^0.7.6`
- Main assets at risk:
  - User entrance fees
  - Winner prize pool
  - Protocol fees
  - Fairness of winner and rarity selection

## Findings

| ID | Severity | Finding |
| --- | --- | --- |
| H-1 | High | [Denial of Service in `enterRaffle`](./findings/denialOfService.md) |
| H-2 | High | [Reentrancy in `refund` allows an attacker to drain the contract](./findings/reentrancy.md) |
| H-3 | High | [Weak randomness in `selectWinner` allows attackers to influence the winner and NFT rarity](./findings/weakRandomness.md) |
| H-4 | High | [Unsafe cast of `fee` to `uint64` can corrupt protocol fee accounting](./findings/unsafe-cast-total-fees.md) |
| M-1 | Medium | [Strict balance check in `withdrawFees` can be broken by forced ETH](./findings/withdrawFees-force-eth.md) |
| M-2 | Medium | [A smart contract winner can block winner selection by reverting ETH transfers](./findings/smart-contract-winner-dos.md) |
| L-1 | Low | [`getActivePlayerIndex` returns `0` for both a missing player and the first player](./findings/get-active-player-index.md) |
| I-1 | Informational | [Code quality and gas observations](./findings/code-quality-and-gas.md) |

## Key Lessons

- External calls must be treated as control handoff. In `refund`, sending ETH before clearing state creates a direct reentrancy path.
- On-chain values such as `block.timestamp`, `block.difficulty`, and `msg.sender` are not safe randomness sources for value-bearing raffles.
- Internal accounting should not depend on strict equality with `address(this).balance`, because ETH can be forced into a contract.
- Downcasting token amounts or fee values is dangerous unless the range is checked before the cast.
- Push payments to arbitrary winners can turn winner selection into a denial-of-service vector. Pull payments are usually safer.
- Gas and information findings still matter because they make the codebase easier to reason about and reduce long-term operational risk.
