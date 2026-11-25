### [High] Missing Access Control on setPassword Allows Unauthorized Password Overwrite

### Summary:
PasswordStore.setPassword lacks an ownership/caller check, allowing any external
account to overwrite `s_password`. Additionally, `s_password` (private in storage slot 1)
is readable directly from contract storage.

### Root Cause:
`setPassword(string memory newPassword)` in PasswordStore.sol updates `s_password`
without verifying `msg.sender == s_owner`. Marking `s_password` as private only restricts
Solidity compiler access and does not hide data stored in EVM storage slots.

### Impact:
- Any arbitrary caller can overwrite `s_password`, clobbering the owner's 
password and preventing the owner from retrieving their prior password
via `getPassword()`.

- Anyone querying contract storage (e.g., via `vm.load` or RPC state reads)
can inspect raw storage slots.

### Proof Of Concept:
In test/PasswordStoreExploit.t.sol:

1. test_Password_Store_Exploit uses `vm.prank(attacker)` to call `setPassword("password_set_by_attacker")`, then switches to newOwner and verifies via `getPassword()` and `assertEq` that the overwrite succceeded.

2. test_ReadStorageSlot uses `vm.load(address(passwordStore), bytes32(uint256(1)))`, and `console.logBytes32` to read slot 1 directly.


### Run with: 
`forge test --match-path test/PasswordStoreExploit.t.sol -vvvv`


### Recommended Mitigation
* Enforce caller authorization in `setPassword` (e.g., `if (msg.sender != s_owner) revert ...;` or an `onlyOwner` check) before updating `s_password`.
* Do not treat `private` state variables as confidential on public blockchains; avoid storing sensitive secrets in plaintext/reversible form in contract storage.

