# Private Data Stored On Chain

## What

`private` in Solidity only limits access from other contracts at the language level.
It does not make the value secret. Contract storage, calldata, transaction input,
events, and historical state can still be inspected from the chain.

## Why it matters

If a protocol stores a password, key, seed, secret answer, or other sensitive value
on chain, any user can read it with RPC tools or block explorers. The protocol may
look private from Solidity code, but the secret is already public data.

## Root cause

Confusing Solidity visibility with real confidentiality.

In the PasswordStore case, the password was stored in `s_password` and marked
`private`, but the raw storage slot could still be read directly from chain.

## Exploit path

1. Identify the storage slot that contains the sensitive value.
2. Read the slot with a tool such as `cast storage`.
3. Decode the returned bytes into the original string or value.

## Impact

The secret is disclosed to everyone. Any feature depending on that value staying
hidden is broken.

## Mitigation

Do not store plaintext secrets on chain. If a value must be referenced on chain,
store a commitment, hash, or encrypted value, and keep the secret material off
chain. Also avoid sending secrets in transaction input, because calldata is public.

## Example

```bash
cast storage <CONTRACT_ADDRESS> <SLOT> --rpc-url <RPC_URL>
cast parse-bytes32-string <STORAGE_VALUE>
```

## My understanding

`private` means "other Solidity contracts cannot read this variable directly";
it does not mean "users cannot see this value." During review, any variable that
looks like a password, private key, secret, salt, or hidden answer should trigger
the question: "Could this be read from public chain data?"
