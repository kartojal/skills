# ERC4337 Attack Vectors

7 attack vectors specific to ERC4337 (Account Abstraction) implementations. For each: detection pattern (what to look for in code) and false-positive signals (what makes it NOT a vulnerability even if the pattern matches).

---

**1. validateUserOp Missing EntryPoint Caller Restriction**

- **Detect:** `validateUserOp(UserOperation calldata, bytes32, uint256)` is `public` or `external` without a guard that enforces `msg.sender == address(entryPoint)`. Anyone can call the validation function directly, bypassing the EntryPoint's replay and gas-accounting protections. Also check `execute` and `executeBatch` — they should be similarly restricted to the EntryPoint or the wallet owner.
- **FP:** Function starts with `require(msg.sender == address(_entryPoint), ...)` or uses an `onlyEntryPoint` modifier. Internal visibility used.

**2. validateUserOp Signature Not Bound to nonce or chainId**

- **Detect:** `validateUserOp` reconstructs the signed digest manually (not via `entryPoint.getUserOpHash(userOp)`) and omits `userOp.nonce` or `block.chainid` from the signed payload. Enables cross-chain replay (same signature valid on other chains sharing the contract address) or in-chain replay after the wallet state is reset. Pattern: `keccak256(abi.encode(userOp.sender, userOp.callData, ...))` without nonce/chainId.
- **FP:** Signed digest is computed via `entryPoint.getUserOpHash(userOp)` — EntryPoint includes sender, nonce, chainId, and entryPoint address. Custom digest explicitly includes `block.chainid` and `userOp.nonce`.

**3. Paymaster ERC-20 Payment Deferred to postOp Without Pre-Validation**

- **Detect:** `validatePaymasterUserOp` does not transfer tokens or lock funds — payment is deferred entirely to `postOp` via `safeTransferFrom`. Between validation and execution the user can revoke the ERC-20 allowance (or drain their balance), causing `postOp` to revert. The paymaster still owes the bundler its gas costs, losing deposit without collecting payment. Pattern: `postOp` contains `token.safeTransferFrom(user, address(this), cost)` with no corresponding lock in the validation phase.
- **FP:** Tokens are transferred or locked (e.g., via `transferFrom` into the paymaster) during `validatePaymasterUserOp` itself. `postOp` is used only to refund excess, never to collect initial payment.

**4. Paymaster Gas Penalty Undercalculation**

- **Detect:** Paymaster computes the prefund amount as `requiredPreFund + (refundPostopCost * maxFeePerGas)` without including the 10% penalty the EntryPoint applies to unused execution gas (`postOpUnusedGasPenalty`). When a UserOperation specifies a large `executionGasLimit` and uses little of it, the EntryPoint deducts a penalty the paymaster did not budget for, draining its deposit. Pattern: prefund formula lacks any reference to unused-gas penalty or `_getUnusedGasPenalty`.
- **FP:** Prefund calculation explicitly adds the unused-gas penalty: `requiredPreFund + penalty + (refundCost * price)`. Paymaster uses conservative overestimation that covers worst-case penalty.

**5. Banned Opcode in Validation Phase (Simulation-Execution Divergence)**

- **Detect:** `validateUserOp` or `validatePaymasterUserOp` references `block.timestamp`, `block.number`, `block.coinbase`, `block.prevrandao`, or `block.basefee`. Per ERC-7562, these opcodes are banned in the validation phase because their values can differ between bundler simulation (off-chain) and on-chain execution, causing ops that pass simulation to revert on-chain. The bundler pays gas for the failed inclusion.
- **FP:** Banned opcodes appear only in the execution phase (inside `execute`/`executeBatch` logic, not in validation). The entity using the banned opcode is staked and tracked under the ERC-7562 reputation system (reduces but does not eliminate risk).

**6. Counterfactual Wallet Initialization Parameters Not Bound to Deployed Address**

- **Detect:** Factory's `createAccount` uses `CREATE2` but the salt does not incorporate all initialization parameters (especially the owner address). An attacker can call `createAccount` with a different owner before the legitimate user, deploying a wallet they control to the same counterfactual address. Pattern: `salt` is a plain user-supplied value or only includes a partial subset of init data; `CREATE2` address can be predicted and front-run with different constructor args.
- **FP:** Salt is derived from all initialization parameters: `salt = keccak256(abi.encodePacked(owner, ...))`. Factory reverts if the account already exists. Initializer is called atomically in the same transaction as deployment.

**7. ERC-1271 isValidSignature Delegated to Untrusted or Arbitrary Module**

- **Detect:** `validateUserOp` or the wallet's `isValidSignature` implementation calls `isValidSignature(hash, sig)` on an externally-supplied or user-registered contract address without verifying that the contract is an explicitly whitelisted module or owner-registered guardian. A malicious module that always returns `0x1626ba7e` (ERC-1271 magic value) passes all signature checks. Pattern: `ISignatureValidator(module).isValidSignature(...)` where `module` comes from user input or an unguarded registry.
- **FP:** `isValidSignature` is only delegated to contracts in an owner-controlled whitelist or to the wallet owner's EOA address directly. Module registry has a timelock or guardian approval before a new module can validate signatures.

---
