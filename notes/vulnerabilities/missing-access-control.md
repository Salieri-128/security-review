# Missing Access Control

## What

Access control bugs happen when a privileged function can be called by an
unauthorized user. The code may say "only owner" in comments or documentation,
but the EVM only enforces checks that are actually implemented.

## Why it matters

Many smart contract functions are dangerous if anyone can call them: changing
ownership, setting configuration, upgrading contracts, withdrawing funds, pausing
the protocol, minting tokens, or changing sensitive state.

## Root cause

The function changes privileged state but does not check `msg.sender`.

In the PasswordStore case, `setPassword` was documented as owner-only behavior,
but the function was `external` and had no owner check, so any address could
change the stored password.

## Exploit path

1. Find a function that mutates important state.
2. Compare the function behavior with comments, NatSpec, docs, and expected roles.
3. Call the function from a non-owner address.
4. Confirm the protected state changes successfully.

## Impact

Unauthorized users can perform privileged actions. In PasswordStore, anyone could
replace the password, breaking the intended owner-only control.

## Mitigation

Add explicit authorization checks to every privileged function. Common patterns:

- `onlyOwner`
- role-based access control
- custom `if (msg.sender != owner) revert ...`

Tests should include negative cases proving that unauthorized callers revert.

## Example

```solidity
function setPassword(string memory newPassword) external {
    if (msg.sender != s_owner) {
        revert PasswordStore__NotOwner();
    }

    s_password = newPassword;
    emit SetNewPassword();
}
```

## My understanding

When reviewing a function, I should first ask: "Who should be allowed to call
this?" Then I should verify that the code enforces that answer, not just that the
docs claim it. State-changing functions are especially important to test with
non-privileged callers.
