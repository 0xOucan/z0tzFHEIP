---
fheip: 0002
title: Solvency-checked FHE debit primitive
description: Add `FHE.trySub` returning both the transferred ciphertext and a decryptable `ebool` success flag, plus a standard `ConfidentialDebitResult` event, so confidential ledgers can surface insolvency without leaking the balance comparison on-chain.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2026-04-17
requires: FHE.sol (cofhe-contracts), FHERC-20 (fhenix-confidential-contracts)
---

## Abstract

FHE cannot revert on `balance < amount` without leaking the comparison, so every confidential debit today is implemented as `FHE.select(balance >= amount, amount, 0)` — on insolvency the contract silently transfers 0 and the caller has no way to distinguish "paid `amount`" from "paid nothing." This FHEIP defines `FHE.trySub(a, b) → (euint64 diff, ebool ok)` as a first-class primitive and a standard `ConfidentialDebitResult(bytes32 accountKey, bytes32 encSuccessFlag, bytes32 encAmount)` event so downstream code (wallets, indexers, composed contracts) can reveal the outcome off-chain via the existing `FHE.allow` viewer path without a second round-trip. This event is a companion to FHEIP-0003's `ConfidentialDebit` (which logs *that* a debit happened with directional metadata); `ConfidentialDebitResult` reports *whether the solvency gate passed* for that debit.

## Motivation

Every confidential-accounting contract reimplements the solvency gate by hand:

```solidity
ebool ok = FHE.lte(amount, balance);
euint64 transferred = FHE.select(ok, amount, FHE.asEuint64(0));
euint64 newBalance  = FHE.sub(balance, transferred);
```

Downstream consumers — wrappers, composed lending markets, wallets — only see the `transferred` ciphertext. They have no cheap way to know whether `ok` was true. Options today:

- **Ignore the flag.** The current path. Produces silent-zero bugs.
- **Pre-decrypt the balance.** Adds a round-trip and leaks the balance to whoever holds the viewer key even when the debit would have succeeded.
- **Revert on insolvency.** Requires decrypting the comparison on-chain synchronously — impossible for FHE contracts.

The missing piece is a way for the debit itself to emit a ciphertext boolean the existing viewer key already has decryption rights to.

### Use cases

- Confidential ledgers (private payroll, encrypted auction settlement, ledger-based wallets)
- FHERC-20 wrappers that compose with lending / DEX contracts
- Batch spending where partial success must be distinguishable from unanimous zero

### Non-goals

- Adding a plaintext-revert path; the comparison leak is real and worth avoiding.
- Solving amount *masking* (gas-timing side channels) — orthogonal.
- Changing the `euint`/`ebool` type system.

## Specification

### 1. `FHE.trySub` primitive

```solidity
library FHE {
    /// @notice Solvency-checked subtraction.
    /// @return diff `a - b` if `b <= a`, else FHE.asEuint64(0)
    /// @return ok   ciphertext boolean: true iff `b <= a`
    function trySub(euint64 a, euint64 b) internal returns (euint64 diff, ebool ok);
    // Overloads: euint8, euint16, euint32, euint128.
}
```

Semantics:

- `diff` and `ok` are fresh ciphertext handles allocated for this call.
- `trySub` MUST NOT cost more gas than `FHE.lte + FHE.select + FHE.sub` invoked separately. Compliant implementations MAY fuse the three.
- After `trySub` returns, the caller contract has **transient** ACL on both handles. Persistent ACL requires explicit `FHE.allowThis` / `FHE.allow(viewer)` by the caller.

### 2. `ConfidentialDebitResult` standard event

Confidential-ledger contracts that perform FHE debits and want to expose the solvency outcome SHOULD emit:

```solidity
event ConfidentialDebitResult(
    bytes32 indexed accountKey,   // implementation-defined account identifier
    bytes32          encSuccessFlag, // ebool handle — equals `ok` from trySub
    bytes32          encAmount       // euint64 handle — equals `diff`
);
```

This event is distinct from FHEIP-0003's `ConfidentialDebit`: `ConfidentialDebit` logs that a debit occurred along with source/destination/op metadata for history reconstruction, while `ConfidentialDebitResult` logs whether the solvency gate passed. A contract MAY emit both, one, or neither based on its privacy and indexer requirements.

Rules:

- `accountKey` is implementation-defined (EOA padded to bytes32, a ledger id, a vault slot). The indexed slot lets scanners filter by account without decryption.
- Non-indexed ciphertext fields let an indexer scan without holding any viewer key.
- Contracts MUST grant persistent viewer ACL on `encSuccessFlag` to every address that already holds viewer ACL on the debit's source balance. This keeps the "who can see" surface consistent.

### 3. FHERC-20 transfer overloads

FHERC-20 SHOULD expose:

```solidity
function confidentialTransfer(address to, euint64 amount) returns (euint64 transferred, ebool ok);
function confidentialTransferFrom(address from, address to, euint64 amount) returns (euint64 transferred, ebool ok);
```

Implementations use `trySub` internally and MAY emit `ConfidentialDebitResult` for consumers that want solvency-outcome visibility. Existing single-return overloads remain for backwards compatibility and are defined as discarding the `ok` flag.

## Rationale

**Why a new primitive instead of a convention?** Codifying the three-op pattern saves gas (fused implementation), standardizes ACL grants on the returned handles, and gives the event schema a stable anchor.

**Why expose `ok` as a ciphertext and not a plaintext?** Revealing plaintext solvency on every debit would leak balance ordering to the caller. Emitting a ciphertext that the existing viewer key can decrypt off-chain preserves the privacy model: only the viewer sees solvency, just as the viewer already sees balances.

**Why `accountKey` of `bytes32` in the event?** ERC-20 `Transfer(from, to, value)` indexes EOAs. Confidential ledgers key on ledger IDs, vault slots, or other implementation-defined identifiers. `bytes32` is maximally general; implementations can left-pad an `address` to mimic ERC-20 when that's the natural key.

**Why SHOULD, not MUST, emit the event?** Contracts with no viewer concept (e.g. a fully-private sink with no decryption path) may legitimately skip it. Wallets treating absence as "opt-out" is preferable to forcing every FHE contract to pay for an unused event.

**Why a different event name from FHEIP-0003's `ConfidentialDebit`?** The two events answer different questions for different audiences. FHEIP-0003's event is a history-reconstruction primitive — it carries `source`, `destination`, and a `uint8 op` so any scanner can graph flows. This FHEIP's event is a solvency-introspection primitive — it carries the ebool success flag only viewers with the right ACL can decrypt. Collapsing them into one event would either force history indexers to decode ACL-restricted fields they cannot use or force viewers to pay for indexer metadata they do not need. Keeping them separate lets each audience subscribe to the event that matches its trust boundary.

## Backwards Compatibility

Fully additive. Existing contracts using the three-op pattern continue to work. Indexers unaware of `ConfidentialDebitResult` ignore it. The new primitive requires a CoFHE contracts release; dapps can detect availability via a build-time feature flag.

## Security Considerations

1. **ACL propagation.** Implementations MUST grant viewer ACL on `encSuccessFlag` whenever they grant ACL on the updated balance. A desynchronized grant (viewer sees flag but not balance, or vice versa) creates confusing UX and potential correlation attacks.
2. **Handle freshness.** A fresh ciphertext handle MUST be allocated per debit. Reusing a prior handle would let an indexer correlate debits across time.
3. **Indexer decryption posture.** Indexers MUST NOT assume they can decrypt `encSuccessFlag` — they can only if the indexer's address has been `FHE.allow`'d on that handle. Rendering "success/fail" in an explorer requires a viewer permit.
4. **Constant-time semantics.** `trySub` MUST run identical ciphertext ops regardless of plaintext outcome. An implementation that branches on the plaintext comparison would leak via gas.
5. **Composed reentrancy.** The ebool flag is purely informational post-execution. Contracts branching on a *decrypted* flag (via a threshold-network round-trip) must treat the decryption callback like any external call and enforce reentrancy guards.

## Reference Implementation

- The three-op pattern as currently deployed: [`Z0tzPrivateLedger.sol` lines 237-238](https://github.com/0xOucan/Z0tz/blob/main/contracts/contracts/ledger/Z0tzPrivateLedger.sol#L237-L238) — `ebool ok = FHE.lte(amount, from.balance); euint64 transferred = FHE.select(ok, amount, FHE.asEuint64(0));`.
- Real-world manifestation of the silent-zero bug this proposal prevents: a cross-chain cashout on the Z0tz V6.5 ledger transferred 0 USDC to the ephemeral stealth despite the source balance being sufficient. The downstream unshield + claim completed, every receipt reported success, but the stealth held 0. A pre-decrypt via viewer permit caught the case; the FHE primitive could not surface it.
- FHERC-20 current implementation at `fhenix-confidential-contracts/contracts/FHERC20.sol` performs the same `FHE.select(value.lte(balance), value, FHE.asEuint64(0))` pattern inline in `_update` around line 403.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
