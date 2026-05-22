# Fuzz Testing

## What

Fuzz testing lets Foundry generate many random inputs for a test function.

Instead of writing only one fixed example, the test describes a property that should
hold across many possible values.

## Why It Matters

Manual unit tests usually check the cases I already thought about. Fuzz tests help
search for edge cases I did not think to write by hand, such as:

- zero values
- maximum values
- unusual caller balances
- unexpected input combinations
- boundary conditions around limits, fees, or precision

## Typical Pattern

```solidity
function testFuzz_Deposit(uint256 amount) public {
    amount = bound(amount, 1, 100 ether);

    vm.deal(user, amount);
    vm.prank(user);
    vault.deposit{value: amount}();

    assertEq(vault.balanceOf(user), amount);
}
```

## Key Ideas

- Parameters in a test function become fuzzed inputs.
- Use `bound()` to keep generated values inside meaningful ranges.
- Use `vm.assume()` to discard invalid cases, but do not overuse it.
- The test should check a property, not just one expected example.
- Good fuzz tests often target accounting, permissions, state transitions, and math.

## Common Mistakes

- Testing with unbounded values that make the scenario unrealistic.
- Using too many `vm.assume()` filters, which can cause Foundry to reject too many runs.
- Only asserting that the transaction does not revert.
- Recreating the same simple unit test with a random number but no deeper property.

## Audit Use

Fuzzing is useful when I want to ask:

- Does this function work for all valid input sizes?
- Can accounting break around zero, dust amounts, or very large values?
- Can a user mint, withdraw, repay, or redeem more than they should?
- Does rounding always favor the intended side?
- Do state changes stay consistent after many input combinations?

## My Understanding

Fuzz testing is like turning one example-based test into a small search engine.

I still need to define the important property. Foundry only helps explore inputs; it
does not know what security means unless I express the invariant or expected behavior
in assertions.
