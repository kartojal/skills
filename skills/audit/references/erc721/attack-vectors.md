# ERC721 Attack Vectors

11 attack vectors specific to ERC721 tokens. For each: detection pattern (what to look for in code) and false-positive signals (what makes it NOT a vulnerability even if the pattern matches).

---

**1. ERC721 Unsafe Transfer to Non-Receiver**

- **Detect:** `_transfer()` (unsafe) used instead of `_safeTransfer()`, or `_mint()` instead of `_safeMint()`, sending NFTs to contracts that may not implement `IERC721Receiver`. Tokens permanently locked in the recipient contract.
- **FP:** All transfer and mint paths use `safeTransferFrom` or `_safeMint`, which perform the `onERC721Received` callback check. Function is `nonReentrant` to prevent callback abuse.

**2. ERC721 onERC721Received Arbitrary Caller Spoofing**

- **Detect:** Contract implements `onERC721Received` and uses its parameters (`operator`, `from`, `tokenId`) to update state — recording ownership, incrementing counters, or crediting balances — without verifying that `msg.sender` is the expected NFT contract address. Anyone can call `onERC721Received(attacker, victim, fakeTokenId, "")` directly with fabricated parameters, fooling the contract into believing it received an NFT it never got. Pattern: `function onERC721Received(...) { credited[from][tokenId] = true; }` with no `require(msg.sender == nftContract)`.
- **FP:** `msg.sender` is validated against a known NFT contract address before any state update: `require(msg.sender == address(nft))`. The function is `view` or reverts unconditionally (acts as a sink only). State changes are gated on verifiable on-chain ownership (`IERC721(msg.sender).ownerOf(tokenId) == from`) before committing.

**3. ERC721 Approval Not Cleared in Custom Transfer Override**

- **Detect:** Contract overrides `transferFrom` or `safeTransferFrom` with custom logic — fee collection, royalty payment, access checks — but does not call `super._transfer()` or `super.transferFrom()` internally. OpenZeppelin's `_transfer` is the function that executes `delete _tokenApprovals[tokenId]`. Skipping it leaves the previous approved address permanently approved on the token under the new owner. Pattern: custom `transferFrom` that calls a bespoke `_transferWithFee(from, to, tokenId)` without the approval-clear step.
- **FP:** Custom override calls `super.transferFrom(from, to, tokenId)` or `super._transfer(from, to, tokenId)` internally, preserving OZ's approval clearing. Or explicitly calls `delete _tokenApprovals[tokenId]` / `_approve(address(0), tokenId, owner)` before returning.

**4. ERC721Enumerable Index Corruption on Burn or Transfer**

- **Detect:** Contract extends `ERC721Enumerable` and overrides `_beforeTokenTransfer` (OZ v4) or `_update` (OZ v5) without calling the corresponding `super` function. `ERC721Enumerable` maintains four interdependent index structures (`_ownedTokens`, `_ownedTokensIndex`, `_allTokens`, `_allTokensIndex`) that must be updated atomically on every mint, burn, and transfer. Skipping the super call leaves stale entries — `tokenOfOwnerByIndex` returns wrong token IDs, `ownerOf` for enumerable lookups resolves incorrectly, and `totalSupply` diverges from actual supply.
- **FP:** Override always calls `super._beforeTokenTransfer(from, to, tokenId, batchSize)` or `super._update(to, tokenId, auth)` as its first statement. Contract does not inherit `ERC721Enumerable` and tracks supply independently.

**5. EIP-2981 Royalty Signaled But Never Enforced**

- **Detect:** Contract implements `IERC2981.royaltyInfo(tokenId, salePrice)` and `supportsInterface(0x2a55205a)` returns `true`, advertising royalty support. However, the protocol's own transfer, listing, or settlement logic never calls `royaltyInfo()` and never routes payment to the royalty recipient. EIP-2981 is a signaling standard — it cannot force payment. Any marketplace that does not voluntarily query and pay royalties will bypass them entirely. Pattern: `royaltyInfo()` implemented, but `transferFrom` and all settlement paths contain no corresponding payment call.
- **FP:** Protocol's own marketplace or settlement contract reads `royaltyInfo()` and transfers the royalty amount to the recipient before or after completing the sale — enforced on-chain. Royalties are intentionally zero (`royaltyBps = 0`) and this is documented.

**6. ERC721A / Lazy Ownership — ownerOf Uninitialized in Batch Range**

- **Detect:** Contract uses ERC721A (or OpenZeppelin `ERC721Consecutive`) for gas-efficient batch minting. Ownership is stored lazily: only the first token of a consecutive run has its ownership struct written; all subsequent IDs in the range inherit it by binary search. Before any transfer occurs, `ownerOf(id)` for IDs in the middle of a batch may return `address(0)` or the batch-start owner depending on implementation version. Access control that calls `ownerOf(tokenId) == msg.sender` on freshly minted tokens without an explicit transfer may fail or return incorrect results. Pattern: `require(ownerOf(tokenId) == msg.sender)` in a staking or approval function called immediately after a batch mint.
- **FP:** Protocol always waits for an explicit `transferFrom` or `safeTransferFrom` before checking ownership (each transfer initializes the packed slot). Contract uses standard OZ `ERC721` where every mint writes `_owners[tokenId]` directly.

**7. setApprovalForAll Grants Permanent Unlimited Operator Access**

- **Detect:** Protocol requires users to call `nft.setApprovalForAll(protocol, true)` to enable staking, escrow, or any protocol-managed transfer. This grants the operator irrevocable, time-unlimited control over every current and future token the user holds in that collection. No expiry, no per-token scoping, and no per-amount limit. If the approved operator contract is exploited, upgraded maliciously, or its admin key is compromised, an attacker can drain all tokens from all users who granted approval in a single sweep. Pattern: `require(nft.isApprovedForAll(msg.sender, address(this)), "must approve")` at the entry point of a staking or escrow function.
- **FP:** Protocol uses individual `approve(address(this), tokenId)` before each transfer, requiring per-token user action. Operator is an immutable non-upgradeable contract with a formally verified transfer function. Protocol provides an on-chain `revokeAll()` helper users are trained to call after each interaction.

**8. ERC721 transferFrom with Unvalidated `from` Parameter**

- **Detect:** Custom ERC721 overrides `transferFrom(from, to, tokenId)` and verifies that `msg.sender` is the owner or approved, but does not verify that `from == ownerOf(tokenId)`. An attacker who is an approved operator for `tokenId` can call `transferFrom(victim, attacker, tokenId)` with a fabricated `from` address — the approval check passes for the operator, the token moves, but `from` was not the actual owner and may not be the intended origin for accounting, event logging, or protocol-level state. Pattern: `require(isApprovedOrOwner(msg.sender, tokenId))` without a subsequent `require(from == ownerOf(tokenId))`.
- **FP:** `super.transferFrom()` or OZ's `_transfer(from, to, tokenId)` called internally — OZ's `_transfer` explicitly checks `from == ownerOf(tokenId)` and reverts with `ERC721IncorrectOwner` if not. Custom override includes an explicit `require(ownerOf(tokenId) == from)` before transfer logic.

**9. NFT Staking / Escrow Records msg.sender Instead of ownerOf**

- **Detect:** Staking or escrow contract accepts an ERC721 via `nft.transferFrom(msg.sender, address(this), tokenId)` and records `depositor[tokenId] = msg.sender`. An operator (approved but not the owner) can call `stake(tokenId)` — the transfer succeeds because the operator holds approval, but `msg.sender` is the operator, not the real owner. The real owner loses their NFT; the operator is credited as depositor and receives all staking rewards and the right to unstake. Pattern: `depositor[tokenId] = msg.sender` without cross-checking against `nft.ownerOf(tokenId)` before the transfer.
- **FP:** Contract reads `address realOwner = nft.ownerOf(tokenId)` before accepting the transfer and records `depositor[tokenId] = realOwner`. Or requires `require(nft.ownerOf(tokenId) == msg.sender, "not owner")` so operators cannot stake on others' behalf.

**10. ERC721Consecutive (EIP-2309) Balance Corruption with Single-Token Batch**

- **Detect:** Contract uses OpenZeppelin's `ERC721Consecutive` extension (OZ < 4.8.2) and mints a batch of exactly one token via `_mintConsecutive(to, 1)`. A bug in that version fails to increment the recipient's balance for size-1 batches. `balanceOf(to)` returns 0 despite ownership being assigned. When the owner later calls `transferFrom`, the internal balance decrement underflows (reverts in checked math, or wraps in unchecked), leaving the token in a frozen state or causing downstream accounting errors in any contract that relies on `balanceOf` for reward distribution or collateral checks.
- **FP:** OZ version ≥ 4.8.2 used (patched via GHSA-878m-3g6q-594q). Batch size is always ≥ 2. Contract uses standard `ERC721._mint` (non-consecutive) where every mint writes the balance mapping directly.

**11. ERC721 / ERC1155 Type Confusion in Dual-Standard Marketplace**

- **Detect:** Marketplace or aggregator handles both ERC721 and ERC1155 in a shared `buy` or `fill` function using a type flag, but the `quantity` parameter required for ERC1155 amount is also accepted for ERC721 without validation that it equals 1. Price is computed as `price * quantity`. An attacker passes `quantity = 0` for an ERC721 listing — price calculation yields zero, NFT transfers successfully, payment is zero. Root cause of the TreasureDAO exploit (March 2022, $1.4M): `buyItem(listingId, 0)` for an ERC721 listing passed all checks and transferred the NFT for free.
- **FP:** ERC721 branch explicitly `require(quantity == 1)` before any price arithmetic. Separate code paths for ERC721 and ERC1155 with no shared quantity parameter. Price computed independently of quantity for ERC721 listings.

---
