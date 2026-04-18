---
fheip: 0004
title: ACL lifecycle — transient scope, persistent revocation, cross-call forwarding
description: Formalize the lifecycle of `FHE.allowTransient`, introduce `FHE.revokeAllow` for persistent grants, and standardize cross-contract ACL forwarding so multi-contract FHE systems compose predictably without reinventing the pattern in every dapp.
author: 0xOucan (@0xOucan)
discussions-to: TBD
status: Draft
type: Standards Track
category: Core
created: 2026-04-17
requires: FHE.sol / ACL.sol (cofhe-contracts)
---

## Abstract

CoFHE's ACL provides `allowThis`, `allow(address)`, `allowTransient`, `allowGlobal`, `allowPublic`, but leaves three critical behaviors unspecified: (a) exactly when transient ACL expires and whether re-granting is idempotent, (b) how to revoke a persistent grant, and (c) how ACL propagates across contract boundaries when a ciphertext handle is returned upward or passed as calldata to a child contract. Today every multi-contract FHE system reinvents cross-call forwarding (`FHE.allowTransient(handle, address(next))` sprinkled at every boundary) and no contract can revoke a compromised viewer. This FHEIP pins down transient semantics, adds `FHE.revokeAllow` to the Core, and defines a canonical "forward transient to caller" helper.

## Motivation

Three real issues:

### Transient lifetime is under-specified

Docs say "transaction-scoped." Consumers have to guess:

- Does `allowTransient(h, A)` persist across reverts within the same tx?
- Is calling `allowTransient(h, A)` twice idempotent, or does the second call cost full gas?
- When exactly does it clear — at tx end, at the end of the top-level external call, on `selfdestruct`?

These choices change whether reentrancy-sensitive code is safe. Without a spec, every wrapper contract has to black-box-test.

### No persistent revocation

`FHE.allow(handle, viewer)` grants permanent decryption rights. There is no matching `revokeAllow`. A compromised viewer key today means rotating the entire ciphertext handle (re-encrypt all state), because there's no way to tell the threshold network "that 20-byte address is no longer authorized on this ctHash." Passkey wallets hit this the moment a device is lost.

### Cross-call forwarding is manual

When ledger `L` calls vault `V` which calls wrapped token `W`, the ciphertext handle flowing through must have **transient ACL granted explicitly to each next contract at each boundary**. Miss a single `FHE.allowTransient(amount, address(W))` and the inner call reverts with an unhelpful ACL error. The pattern is recapitulated in every multi-contract system; there is no canonical helper.

### Use cases

- Multi-contract FHE systems (ledger → vault → wrapper → underlying)
- Passkey wallets needing viewer-key revocation on device loss
- Relayer-submitted FHE ops where ACL binds to the relayer and needs immediate forwarding
- Composable DeFi on FHE (lending markets calling into AMMs calling into wrappers)

### Non-goals

- Changing the ACL's address-based identity model.
- Adding per-grant expiry timestamps (covered separately by permit standards).
- Solving key rotation at the threshold-network level (covered by FHEIP-0005 / FHEIP-0006).

## Specification

### 1. Transient ACL lifetime — formal spec

```solidity
library FHE {
    /// Grants `viewer` transient decryption ACL on `ctHash`.
    /// Scope: from the start of the top-level external transaction (via tstore)
    ///        through tx end, regardless of revert/success in nested frames.
    /// Idempotency: subsequent calls within the same tx are O(1) no-ops.
    function allowTransient(bytes32 ctHash, address viewer) internal;
}
```

Normative behavior:

- Transient ACL MUST persist across nested call reverts within the same top-level tx (i.e., it uses `TSTORE` / transient storage with the EVM semantics of EIP-1153).
- Transient ACL MUST clear at the top-level tx boundary. Implementations MUST NOT carry transient state across txs via any side channel.
- Calling `allowTransient(h, A)` when `(h, A)` is already set MUST be a no-op and MUST consume only the base gas of a transient-storage warm check — no re-writes, no events.
- `allowTransient(h, A)` MUST emit no event. (Persistent grants SHOULD emit — see §2.)

### 2. Persistent ACL revocation

```solidity
library FHE {
    /// Revokes a persistent `allow` grant. Only the contract that granted
    /// (stored as the `requester` in ACL state) may revoke.
    function revokeAllow(bytes32 ctHash, address viewer) internal;
}

// ACL contract emits:
event AllowGranted (bytes32 indexed ctHash, address indexed viewer, address indexed requester);
event AllowRevoked (bytes32 indexed ctHash, address indexed viewer, address indexed requester);
```

Normative behavior:

- `revokeAllow(h, A)` MUST succeed iff `msg.sender` matches the `requester` recorded when the grant was made. Otherwise revert with `ACLUnauthorized()`.
- Revocation propagates to the threshold-network verifier: a decryption request submitted *after* the revoke tx confirms MUST be rejected. (See FHEIP-0005 for the wire mechanics.)
- Re-granting a revoked handle/viewer pair is permitted and MUST emit `AllowGranted` anew.
- `AllowGranted` / `AllowRevoked` MUST be emitted by the ACL contract (not the caller), so any indexer subscribed to a single address can track all grants/revokes chain-wide.

### 3. Cross-call forwarding helper

```solidity
library FHE {
    /// Grants transient ACL on `ctHash` to `msg.sender`'s caller (i.e. one
    /// frame up the call stack). Useful when a callee is about to return
    /// a ciphertext handle that the immediate caller will use in its own
    /// FHE ops.
    function forwardTransientToCaller(bytes32 ctHash) internal;

    /// Shorthand for `allowTransient(ctHash, next) ; <external call>`.
    /// Emits no event.
    function allowTransientFor(bytes32 ctHash, address next) internal;
}
```

Normative behavior:

- `forwardTransientToCaller(h)` grants transient ACL to the caller of the current external frame. In a call stack `A → B → C`, when `C` executes `forwardTransientToCaller(h)`, the grant goes to `B`.
- `forwardTransientToCaller` MUST revert with `NoExternalCaller()` if the current frame is the top-level tx origin (no parent call frame).
- `allowTransientFor(h, next)` is semantically equivalent to `allowTransient(h, next)` but is marked as a forwarding intent, useful for static analyzers.

### 4. Self-call ACL-binding pattern

When a contract needs `msg.sender` to be itself for ACL purposes inside a nested FHE op (e.g., a relayer-called function that must verify an `InEuintN` input), the canonical pattern is:

```solidity
function spend(InEuint64 calldata input) external {
    // ... auth checks ...
    euint64 amount = this.verifyAmount(input);  // self-call flips msg.sender
    // ... use amount ...
}

function verifyAmount(InEuint64 calldata input) external returns (euint64) {
    if (msg.sender != address(this)) revert NotSelf();
    return FHE.asEuint64(input);
}
```

Implementations SHOULD expose a shared helper library exporting this pattern as a single `FHE.asEuintNSelf(input)` call. (See FHEIP-0009 for the dedicated self-call primitive.)

## Rationale

**Why use EIP-1153 transient storage for transient ACL?** TSTORE-backed semantics give us the exact "tx-scoped, reset-free" behavior with minimum gas. Implementing transient ACL as regular storage would require explicit clearing and increase gas per grant by ~20k.

**Why limit `revokeAllow` to the original requester?** The requester is the only party entitled to revoke. Viewer self-revoke is not defined here because the viewer signing a revocation op is outside FHE.sol's scope (it belongs in the SDK / threshold-network layer). A consumer wanting viewer self-revoke can build it on top of the primitive by having the viewer call a contract method that forwards to `revokeAllow`.

**Why `AllowGranted` / `AllowRevoked` at the ACL contract, not the consumer?** A single indexer address (`ACL.address`) captures every grant/revoke across the chain. Emitting at the consumer contract would fragment indexing across thousands of deployments.

**Why `forwardTransientToCaller` instead of implicit propagation?** Implicit propagation would make ACL state opaque: every cross-contract call would silently modify ACL without the author's knowledge, making security analysis impossible. Explicit forwarding is a small syntactic cost for a major clarity win.

## Backwards Compatibility

- Transient ACL §1: the spec codifies what current implementations already do (TSTORE-backed). Any deviation in the current implementation is a bug to fix, not a break.
- Revocation §2: purely additive. Existing `allow`-only dapps keep working.
- Forwarding §3: purely additive. Existing manual `allowTransient(h, A)` patterns keep working; the helper is a stylistic shortcut.

## Security Considerations

1. **Transient re-grant across revert boundaries.** If contract `A` calls `B`, `B` grants transient ACL, `B` reverts, then `A` grants transient ACL — per this spec both grants remain until tx end. Implementations MUST NOT use the `B` revert to clear transient state. Consumers relying on revert-clear behavior (inherited from some storage patterns) must be updated.
2. **Revocation race.** Between a revocation tx and a pending decryption request, the threshold network may already have the request in-flight. FHEIP-0005 addresses this by including the revocation block number in the decryption-authorization path; standalone FHEIP-0004 alone does NOT guarantee perfect freshness.
3. **Unauthorized revocation via `requester` impersonation.** `revokeAllow` MUST verify `msg.sender == recorded_requester`. Implementations MUST NOT allow the viewer itself to revoke on behalf of the requester.
4. **Forwarding to the wrong address.** `forwardTransientToCaller` trusts `msg.sender`'s caller without question. A malicious contract in the middle of the call stack could redirect forwarded ACL to itself. Consumers with trust boundaries at mid-stack SHOULD use explicit `allowTransientFor(h, knownGoodAddress)` instead.
5. **Event firehose.** High-activity ACL deployments will emit many `AllowGranted` / `AllowRevoked` events. Indexers SHOULD subscribe with account-key filters where the ABI supports them; the spec uses indexed topics precisely to enable this.
6. **Idempotent `allowTransient` as a cache.** Because re-grants are no-ops, consumers can safely call `allowTransient` defensively without gas concerns. This is intentional: it lets wrapper contracts grant without first checking, simplifying composable code.

## Reference Implementation

- Current ACL contract: `cofhe-contracts/contracts/internal/host-chain/contracts/detereministic-tm/DeterministicACL.sol` — `allow` at line 89, `allowTransient` at line 153, `allowedOnBehalf` at line 203, `allowedTransient` at line 223. No `revokeAllow`. The upstream path preserves the `detereministic-tm` directory name as spelled in the repo.
- Z0tz's manual cross-call forwarding: [`Z0tzPrivateLedgerVault.sol` line 186](https://github.com/0xOucan/Z0tz/blob/main/contracts/contracts/ledger/Z0tzPrivateLedgerVault.sol#L186) — `FHE.allowTransient(encAmount, address(wrappedToken))` immediately before calling `wrappedToken.confidentialTransfer(...)`, without which the nested FHE op reverts on ACL check.
- Z0tz's self-call pattern: [`Z0tzPrivateLedger.sol` lines 303-306](https://github.com/0xOucan/Z0tz/blob/main/contracts/contracts/ledger/Z0tzPrivateLedger.sol#L303-L306) — an external `verifyAmount(InEuint64)` function guarded by `msg.sender == address(this)` so the inner `FHE.asEuint64(input)` call binds ACL to the ledger rather than to whichever relayer invoked `spend`.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
