# Invariant Testing

## What

Invariant testing checks whether an important system property stays true across many
sequences of actions.

Unlike fuzz testing a single function call, invariant testing lets Foundry call a set
of target functions repeatedly in different orders with different inputs.

## Why It Matters

Many smart contract bugs appear only after several actions happen together.

For example:

- deposit, withdraw, deposit again
- borrow, repay, liquidate
- mint, transfer, burn
- update price, rebalance, redeem

Invariant tests help check whether the system still obeys its core rules after these
changing states.

## Typical Pattern

```solidity
contract VaultInvariantTest is StdInvariant, Test {
    Vault vault;
    Handler handler;

    function setUp() public {
        vault = new Vault();
        handler = new Handler(vault);
        targetContract(address(handler));
    }

    function invariant_TotalAssetsMatchAccounting() public {
        assertEq(vault.totalAssets(), handler.ghost_totalDeposits());
    }
}
```

## Handler Pattern

A handler contract wraps calls into the system under test.

Handlers are useful because they can:

- constrain inputs into valid ranges
- set up realistic callers
- track ghost variables for expected accounting
- avoid meaningless reverts
- model real user flows more clearly

## Key Ideas

- An invariant should describe something that must always be true.
- Foundry generates sequences of calls, not just one call.
- `targetContract()` tells Foundry which contract to call during invariant runs.
- Ghost variables track expected behavior when the protocol itself does not expose it.
- Good invariants focus on core safety properties, not implementation details.

## Good Invariant Examples

- Total user balances should not exceed total assets.
- A vault should not become insolvent.
- A user should not withdraw more than their deposited amount plus valid yield.
- Token supply should equal the sum of tracked balances.
- Protocol fees should never exceed the configured maximum.
- Collateralized positions should not bypass liquidation rules.

## Common Mistakes

- Writing an invariant that only checks a weak property.
- Letting random calls constantly revert instead of exploring meaningful states.
- Forgetting to model multiple users.
- Trusting ghost variables without checking that the handler updates them correctly.
- Testing only happy-path functions and missing dangerous external entry points.

## Audit Use

Invariant testing is useful when I want to ask:

- What must always be true for this protocol to be solvent?
- Can any sequence of valid user actions break accounting?
- Can an attacker combine functions in an unexpected order?
- Are shares, assets, debt, collateral, and rewards always consistent?
- Does the protocol remain safe after many state transitions?

## My Understanding

Fuzz testing asks: "Does this function behave correctly for many inputs?"

Invariant testing asks: "Can the system ever reach a broken state after many actions?"

For audits, invariant testing feels closer to attacker thinking because real exploits
often come from combining valid actions in a harmful sequence.
